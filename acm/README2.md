# Guide to Cluster Landing Zones for Hybrid and Multi-cloud Architectures (Part 2)

## Overview

In this installment we look at extending the cluster landing zone for hybrid cloud that was introduced [previously](https://cloud.redhat.com/blog/a-guide-to-cluster-landing-zones-for-hybrid-and-multi-cloud-architectures) to support stateful application workloads. We also demonstrate how the architecture deals with disaster scenarios such as the catastrophic failure of a cloud provider. We make extensive useage of various capabilities within the Red Hat Advanced Cluster Management (RHACM) toolbox and show how these can work together to deliver a robust and compelling solution which enables a reference multi-tenancy operating model.

## Extending the Cluster Landing Zone

In order to run stateful workloads such as databases across a hybrid cloud envvironment we need to extend the original cluster landing zone to cater for DBAs as -class tenants in addition to application teams and cluster administrators (SREs). The revised cluster landing zone looks as per the following diagram.

<p align="center">
  <img src="https://github.com/jwilms1971/blog/blob/main/acm/RHACM%20Operating%20Model%20-%20Hybrid-cloud%20extended.png">
  <em>Diagram 1. Cluster landing zone for a hybrid cloud architecture</em>
</p>

This modification is required because the alternative approach of granting application teams access to Polices in order to configure databases themselves is considered an anti-pattern in most organizations with respect to the principle of separation of concerns. DBAs can leverage the default OpenShift GitOps (ArgoCD) instance by configuring AppProjects to manage permissions so that DBAs can only deploy to the dba-policies namespace on the hub.

The diagram introduces MachinePools which are logical groupings of cloud provider machine instances that are defined externally to the cluster itself and can be scaled manually or automatically. For our purposes we are using these to segregate disparate workloads that have distinct profiles, in this case frontend web applications from backend databases. 

The discussion continues with a focus on the configuration of the database for a hybrid cloud setup as details around deploying applications were presented in the previous installment. Also out of scope but worth noting is that it is recommended to deploy global load balancers targeting the ingress endpoint of each cluster and that these should be deployed in a manner that avoids a shared fate situation.

The database shown in the diagram is PostgreSQL and was selected due to it's familiarity and built-in replication with failover capabilities. In addition PgPool is used to provide middleware functions including load-balancing and connection pooling for client applications accessing the database. There are multiple Operators and Helm charts available for installing PostgreSQL and the focus in this article is on what is required to configure the system in the context of hybrid cloud and to support the kind of high availability scenarios we are trying to achieve. Note that in order to leverage auto-failover and recovery it is required to deploy a PostgreSQL server on at least three nodes to establish quorum. In practice it is recommended to deploy a PostgreSQL server on each node in each availability zone to protect against both zonal and cloud provider failure. For readability purposes, a single PostgreSQL server per cloud provider on one node in each machine pool is being configured.

## Deploying the Cluster Landing Zone

We start of with defining a ManagedClusterSet that serves as a logical grouping mechanism for referencing clusters spawned from all cloud provider infrastructure.

	apiVersion: cluster.open-cluster-management.io/v1beta1
	kind: ManagedClusterSet
	metadata:
	  name: red-cluster-set
	spec: {}

Next we create and bind the dba-policies namespaces to the red-cluster-set so that policies written to this namespace can be deployed to the clusters referenced by a managed cluster set.

	apiVersion: cluster.open-cluster-management.io/v1beta1
	kind: ManagedClusterSetBinding
	metadata:
	  name: red-cluster-set
	  namespace: dba-policies
	spec:
	  clusterSet: red-cluster-set

ClusterPools encapsulate the underlying cloud provider infrastructure and cluster configuration and assign any clusters spawned to the red-cluster-set. An example is shown for AWS.

	apiVersion: hive.openshift.io/v1
	kind: ClusterPool
	metadata:
	  name: 'red-cluster-pool-aws-1'
	  namespace: 'red-cluster-pool'
  	labels:
	  cloud: 'AWS'
	  region: 'ap-southeast-1'
	  vendor: OpenShift
	  cluster.open-cluster-management.io/clusterset: 'red-cluster-set'
	spec:
	  size: 0
	  runningCount: 0
	  skipMachinePools: false
	  baseDomain: aws.example.com
	  installConfigSecretTemplateRef:
	    name: red-cluster-pool-aws-install-config-1
	  imageSetRef:
	    name: img4.11.2-x86-64-appsub
	  pullSecretRef:
	    name: red-cluster-pool-aws-pull-secret
	  platform:
	    aws:
	      credentialsSecretRef:
	        name: red-cluster-pool-aws-creds
	      region: ap-southeast-1

To spawn a cluster we need to submit a ClusterClaim (this is similar in concept to how a PersistentVolumeClaim results in the creation of a PersistentVolume). The following cluster claims were submitted and reference ClusterPools mapped to cloud providers and specific regions. Convenience labels (e.g., env=dev, postgresql=pg-X) are defined for placing workloads on the correct clutser by the Policy Generator tool. 

        apiVersion: hive.openshift.io/v1
        kind: ClusterClaim
        metadata:
          name: red-cluster-1
          namespace: red-cluster-pool
          labels:
	    env: dev
	    postgresql: pg-1
        spec:
          clusterPoolName: red-cluster-pool-aws-1
        ---
        apiVersion: hive.openshift.io/v1
        kind: ClusterClaim
        metadata:
          name: red-cluster-2
          namespace: red-cluster-pool
          labels:
	    env: dev
	    postgresql: pg-2
        spec:
          clusterPoolName: red-cluster-pool-azure-1
        ---
        apiVersion: hive.openshift.io/v1
        kind: ClusterClaim
        metadata:
          name: red-cluster-3
          namespace: red-cluster-pool
          labels:
	    env: dev
	    postgresql: pg-3
        spec:
          clusterPoolName: red-cluster-pool-gcp-1

After some time the resulting set of clusters spawned are as follows.

	$ oc get managedclusters
	NAME                             HUB ACCEPTED   MANAGED CLUSTER URLS                                                JOINED   AVAILABLE   AGE
	local-cluster                    true           https://api.hub-cluster-1.aws.example.com:6443                      True     True        80m
	red-cluster-pool-aws-1-lt87l     true           https://api.red-cluster-pool-aws-1-lt87l.aws.example.com:6443       True     True        32m
	red-cluster-pool-azure-1-g76vj   true           https://api.red-cluster-pool-azure-1-g76vj.azure.example.com:6443   True     True        33m
	red-cluster-pool-gcp-1-x5mmj     true           https://api.red-cluster-pool-gcp-1-x5mmj.gcp.example.com:6443       True     True        33m

Note that cluster names are dynamically generated in line with the "cattle not pets" paradigm adopted by Kubernetes. This is beneficial because it allows for rapid cluster rebuilds provided we have decoupled our application endpoints from the cluster administration endpoint and are using a restore-from-code approach.

At this stage we have a general purpose application cluster with three control plane nodes and three worker nodes. In the next step we begin to customize the cluster to support both stateless and stateful workloads. To this end we configure an additional MachinePool for each cluster with a node in each availability zone. The following Policy template example is for AWS.

	apiVersion: hive.openshift.io/v1
	kind: MachinePool
	metadata:
	  name: '{{ (lookup "hive.openshift.io/v1" "ClusterClaim" "red-cluster-pool" "red-cluster-1").spec.namespace }}-backend-worker'
	  namespace: '{{ (lookup "hive.openshift.io/v1" "ClusterClaim" "red-cluster-pool" "red-cluster-1").spec.namespace }}'
	spec:
	  clusterDeploymentRef:
	    name: '{{ (lookup "hive.openshift.io/v1" "ClusterClaim" "red-cluster-pool" "red-cluster-1").spec.namespace }}'
	  name: backend-worker
	  labels:
	    node-role.kubernetes.io/backend: ""
	  taints:
	  - effect: NoSchedule
	    key: postgresql
	  platform:
	    aws:
	      rootVolume:
	        iops: 100
	        size: 120 
	        type: gp3
	      type: m6i.xlarge
	  replicas: 3
	
Note that we use the known static cluster claim name in order to derive the dynamically generated cluster name. This piece of YAML cannot be submitted directly to the API Server as it is using Go templating and  needs to first be processed by the Policy Generator tool into a valid Policy document that in turn can be applied to the system via the Policy controller. For more details on this tool please refer to [here](https://cloud.redhat.com/blog/generating-governance-policies-using-kustomize-and-gitops).

The outcome of applying this Policy is that each cluster now has two machine pools: one for running application workloads (default workers) and one for database workloads (backend workers) with the underlying machines distributed equally across all availability zones by default.

	$ oc get machinepools -A
	NAMESPACE                        NAME                                            POOLNAME         CLUSTERDEPLOYMENT                REPLICAS
	red-cluster-pool-aws-1-lt87l     red-cluster-pool-aws-1-lt87l-backend-worker     backend-worker   red-cluster-pool-aws-1-lt87l     3
	red-cluster-pool-aws-1-lt87l     red-cluster-pool-aws-1-lt87l-worker             worker           red-cluster-pool-aws-1-lt87l     3
	red-cluster-pool-azure-1-g76vj   red-cluster-pool-azure-1-g76vj-backend-worker   backend-worker   red-cluster-pool-azure-1-g76vj   3
	red-cluster-pool-azure-1-g76vj   red-cluster-pool-azure-1-g76vj-worker           worker           red-cluster-pool-azure-1-g76vj   3
	red-cluster-pool-gcp-1-x5mmj     red-cluster-pool-gcp-1-x5mmj-backend-worker     backend-worker   red-cluster-pool-gcp-1-x5mmj     3
	red-cluster-pool-gcp-1-x5mmj     red-cluster-pool-gcp-1-x5mmj-worker             worker           red-cluster-pool-gcp-1-x5mmj     3

The next step is to setup layer-3 network connectivity between the clusters across the cloud providers so that technologies such as PostgreSQL and PgPool that leverage both TCP and UDP protocols can communicate seamlessly without requiring NAT. To do so requires that we "flatten" the network between clusters so that pods and services in one cluster have line-of-sight access to pods and services in all of the other clusters (provided that the pods and services are hosted in the same namespace which effectively establishes a trust boundary). Submariner is the tool from the RHACM toolbox that can be used to establish a secure tunnel overlay between the clusters and cloud providers to enable this kind of hybrid network connectivity.

Assuming that the preprequisites as documented [here](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.6/html/add-ons/add-ons-overview#preparing-selected-hosts-to-deploy-submariner) have been met then the next step is to enable the Submariner add-on in RHACM, configure the broker and gateway nodes for each cluster which will result in VXLAN tunnels being established between the gateway nodes for passing IPsec traffic. An example of deploying this for clusters hosted on AWS using Policy templates is as follows.

	apiVersion: addon.open-cluster-management.io/v1alpha1
	kind: ManagedClusterAddOn
	metadata:
	  name: submariner
	  namespace: '{{ (lookup "hive.openshift.io/v1" "ClusterClaim" "red-cluster-pool" "red-cluster-1").spec.namespace }}'
	spec:
	  installNamespace: submariner-operator
	---
	apiVersion: submariner.io/v1alpha1
	kind: Broker
	metadata:
	  name: submariner-broker
	  namespace: red-cluster-set-broker
	spec:
	  globalnetEnabled: false
	---
	apiVersion: submarineraddon.open-cluster-management.io/v1alpha1
	kind: SubmarinerConfig
	metadata:
	  name: submariner
	  namespace: '{{ (lookup "hive.openshift.io/v1" "ClusterClaim" "red-cluster-pool" "red-cluster-1").spec.namespace }}'
	spec:
	  gatewayConfig:
	    gateways: 2
	    aws:
	      instanceType: c5d.large
	  IPSecNATTPort: 4500
	  NATTEnable: true
	  cableDriver: libreswan
	  credentialsSecret:
	    name: '{{ (lookup "hive.openshift.io/v1" "ClusterClaim" "red-cluster-pool" "red-cluster-1").spec.namespace }}-aws-creds'

## Deploying PostgreSQL

So far all of the steps shown are intended to be performed by the SREs who are responsible for cluster fleet capacity and networking. The next set of steps are intended to be performed by the DBAs using Policy templates that are deployed via OpenShift GitOps into the dba-policies namespace from where they are picked up by the Policy controller and executed on the clusters bound by the managed cluster set. DBAs will only need RBAC permissions for Policies (API group policy.open-cluster-management.io) and Placements (API group cluster.open-cluster-management.io) to perform their role.

Depending on whether you are using Operators or Helm charts to install PostgreSQL the steps may vary and are covered in great detail in many other public sources. Considerations for deploying PostgreSQL in a hybrid cloud architecture include tuning of timeouts and high-performance block storage. There is also some post-installation configuration that needs to take place to integrate PostgreSQL with the hybrid cloud network that Submariner has established for us and then testing the failover mechanism in the event of catastrophic loss. As PostgreSQL is deployed using StatefulSets and exposed using a headless service, the underlying endpoints (in this case identified by the prefix of a single PostgreSQL server pod per cluster) needs to be made discoverable across all of the clusters participating in the hybrid cloud network. The following YAML (which can be processed by the Policy Generator shown later) will instruct Submariner to create a globally addressable service endpoint on the clusterset.local domain. See the Submariner [documentation](https://submariner.io/) for more details.

	apiVersion: multicluster.x-k8s.io/v1alpha1
	kind: ServiceExport
	metadata:
	  name: pg-1-postgresql-ha-postgresql-headless
	  namespace: database
	---
	apiVersion: multicluster.x-k8s.io/v1alpha1
	kind: ServiceExport
	metadata:
	  name: pg-2-postgresql-ha-postgresql-headless
	  namespace: database
	---
	apiVersion: multicluster.x-k8s.io/v1alpha1
	kind: ServiceExport
	metadata:
	  name: pg-3-postgresql-ha-postgresql-headless
	  namespace: database

Now that the global service names have been defined we need to configure each PostgreSQL server and PgPool instance to communicate with each other using these names. The first step to doing this is to define a common set of configuration properties on the hub so that these can be distributed to each of the managed clusters as a second step. Again the following YAML is intended to be processed by the Policy Generator.

	apiVersion: v1
	kind: ConfigMap
	metadata:
	  name: pg-config
	  namespace: dba-policies
	data:
	  nodeid0: '1000'
	  nodeid1: '1001'
	  nodeid2: '1002'
	  nodename0: 'pg-1-postgresql-ha-postgresql-0'
	  nodename1: 'pg-2-postgresql-ha-postgresql-0'
	  nodename2: 'pg-3-postgresql-ha-postgresql-0'
	  hostname0: 'pg-1-postgresql-ha-postgresql-0.{{- (lookup "hive.openshift.io/v1" "ClusterClaim" "red-cluster-pool" "red-cluster-1").spec.namespace -}}.pg-1-postgresql-ha-postgresql-headless.database.svc.clusterset.local'
	  hostname1: 'pg-2-postgresql-ha-postgresql-0.{{- (lookup "hive.openshift.io/v1" "ClusterClaim" "red-cluster-pool" "red-cluster-2").spec.namespace -}}.pg-2-postgresql-ha-postgresql-headless.database.svc.clusterset.local'
	  hostname2: 'pg-3-postgresql-ha-postgresql-0.{{- (lookup "hive.openshift.io/v1" "ClusterClaim" "red-cluster-pool" "red-cluster-3").spec.namespace -}}.pg-3-postgresql-ha-postgresql-headless.database.svc.clusterset.local'

The next step is to configure the PostgreSQL server and PgPool instance on each cluster to use these hostnames and ids for communication and data replication. The following YAML accomplishes this by patching the configuration of the installed PostgreSQL statefulset and PgPool deployment with values extracted from the configmap resource. An example is shown for the PostgreSQL server and PgPool on AWS identified by the prefix pg-1 as per the diagram above.

	apiVersion: apps/v1
	kind: StatefulSet
	metadata:
	  name: pg-1-postgresql-ha-postgresql
	  namespace: database
	spec:
	  replicas: 1
	  selector:
	    matchLabels:
	      app.kubernetes.io/component: postgresql
	      app.kubernetes.io/instance: pg-1
	  template:
	    metadata:
	      labels:
	        app.kubernetes.io/component: postgresql
	        app.kubernetes.io/instance: pg-1
	    spec:
	      containers:
	      - env:
	        - name: REPMGR_NODE_ID
	          value: '{{hub fromConfigMap "" "pg-config" (printf "nodeid0") hub}}'
	        - name: REPMGR_NODE_NAME
	          value: '{{hub fromConfigMap "" "pg-config" (printf "nodename0") hub}}'
	        - name: REPMGR_PARTNER_NODES
	          value: '{{hub fromConfigMap "" "pg-config" (printf "hostname0") hub}},{{hub fromConfigMap "" "pg-config" (printf "hostname1") hub}},{{hub fromConfigMap "" "pg-config" (printf "hostname2") hub}}'
	        - name: REPMGR_PRIMARY_HOST
	          value: '{{hub fromConfigMap "" "pg-config" (printf "hostname0") hub}}'
	        - name: REPMGR_NODE_NETWORK_NAME
	          value: '{{hub fromConfigMap "" "pg-config" (printf "hostname0") hub}}'
	        image: # postgresql container image
	        name: postgresql
	      initContainers:
	      - image: # postgresql container image
	        name: init-chmod-data
	---
	apiVersion: apps/v1
	kind: Deployment
	metadata:
	  name: pg-1-postgresql-ha-pgpool
	  namespace: database
	spec:
	  replicas: 1
	  selector:
	    matchLabels:
	      app.kubernetes.io/component: pgpool
	      app.kubernetes.io/instance: pg-1
	    spec:
	  template:
	    metadata:
	      labels:
	        app.kubernetes.io/component: pgpool
	        app.kubernetes.io/instance: pg-1
	    spec:
	      containers:
	      - env:
	        - name: PGPOOL_BACKEND_NODES
                  value: '0:{{hub fromConfigMap "" "pg-config" (printf "hostname0") hub}}:5432,1:{{hub fromConfigMap "" "pg-config" (printf "hostname1") hub}}:5432,2:{{hub fromConfigMap "" "pg-config" (printf "hostname2") hub}}:5432'
	        image: # pgpool container image
	        name: pgpool

After running the above through the Policy Generator tool a PostgreSQL server and PgPool instance will be started on each cluster. Review the logs of the primary PostgreSQL server (pg-1) to ensure this worked correctly and that the replication manager has started and is accepting connections from the standby servers.

	LOG:  starting PostgreSQL 14.5 on x86_64-pc-linux-gnu, compiled by gcc (Debian 10.2.1-6) 10.2.1 20210110, 64-bit
	LOG:  listening on IPv4 address "0.0.0.0", port 5432
	LOG:  listening on IPv6 address "::", port 5432
	LOG:  listening on Unix socket "/tmp/.s.PGSQL.5432"
	LOG:  database system was shut down at 2022-10-23 01:53:52 GMT
	LOG:  database system is ready to accept connections
	 done
	server started
	postgresql-repmgr 01:54:42.46 INFO  ==> ** Starting repmgrd **
	[NOTICE] repmgrd (repmgrd 5.3.2) starting up
	INFO:  set_repmgrd_pid(): provided pidfile is /tmp/repmgrd.pid
	[NOTICE] starting monitoring of node "pg-1-postgresql-ha-postgresql-0" (ID: 1000)
	[NOTICE] monitoring cluster primary "pg-1-postgresql-ha-postgresql-0" (ID: 1000)
	[NOTICE] new standby "pg-2-postgresql-ha-postgresql-0" (ID: 1001) has connected
	[NOTICE] new standby "pg-3-postgresql-ha-postgresql-0" (ID: 1002) has connected

Connect to PgPool and verify the status of all PostgreSQL servers.

	postgres=# show pool_nodes;
	node_id |                                                             hostname                                                              | port | status | pg_status | lb_weight |  role   | pg_role | select_cnt | load_balance_node | replication_delay | replication_state | replication_sync_state | last_status_change  
	--------+-----------------------------------------------------------------------------------------------------------------------------------+------+--------+-----------+-----------+---------+---------+------------+-------------------+-------------------+-------------------+------------------------+---------------------
	0       | pg-1-postgresql-ha-postgresql-0.red-cluster-pool-aws-1-lt87l.pg-1-postgresql-ha-postgresql-headless.database.svc.clusterset.local | 5432 | up     | up        | 0.333333  | primary | primary | 7       | false             | 0                 |                   |                        | 2022-10-23 02:17:57
	1       | pg-2-postgresql-ha-postgresql-0.red-cluster-pool-azure-1-g76vj.pg-2-postgresql-ha-postgresql-headless.database.svc.clusterset.local | 5432 | up     | up        | 0.333333  | standby | standby | 15      | false             | 0                 |                   |                        | 2022-10-23 02:17:57
	2       | pg-3-postgresql-ha-postgresql-0.red-cluster-pool-gcp-1-x5mmj.pg-3-postgresql-ha-postgresql-headless.database.svc.clusterset.local | 5432 | up     | up        | 0.333333  | standby | standby | 10      | true              | 0                 |                   |                        | 2022-10-23 02:17:57
	(3 rows)

It is also worth reviewing the service endpointslices created by Submariner on each cluster as ultimately these are critical for sucessful communication between all of the components. Here is an example:

	$ oc get endpointslices
	NAME                                                                    ADDRESSTYPE   PORTS     ENDPOINTS    AGE
	pg-1-postgresql-ha-pgpool-545ps                                         IPv4          5432      10.131.0.31  93m
	pg-1-postgresql-ha-postgresql-headless-red-cluster-pool-aws-1-lt87l     IPv4          5432      10.131.2.9   93m
	pg-1-postgresql-ha-postgresql-headless-zwrqj                            IPv4          5432      10.131.2.9   93m
	pg-1-postgresql-ha-postgresql-ktczz                                     IPv4          5432      10.131.2.9   93m
	pg-2-postgresql-ha-postgresql-headless-red-cluster-pool-azure-1-g76vj   IPv4          5432      10.139.2.6   93m
	pg-3-postgresql-ha-postgresql-headless-red-cluster-pool-gcp-1-x5mmj     IPv4          5432      10.135.2.5   93m

And here is what the the Policy Generation configuration wrapper looks like that was used by the DBA (the configuraiton of the Policy Generation to build the cluster by the SRE team is out of scope for this installment).

	apiVersion: policy.open-cluster-management.io/v1
	kind: PolicyGenerator
	metadata:
	  name: policy-postgresql-provisioning
	placementBindingDefaults:
	  name: binding-policy-postgresql-provisioning
	policyDefaults:
	  standards:
	    - NIST SP 800-53
	  categories:
	    - CM Configuration Management
	  controls: 
	    - CM-2 Baseline Configuration
	  namespace: dba-policies
	  complianceType: musthave
	  remediationAction: enforce
	  severity: low
	policies:
	- name: policy-generate-postgresql-config
	  manifests:
	    - path: input-hub-clusters/postgresql/
	  policySets:
	    - policyset-hub-clusters
	- name: policy-patch-postgresql-red-clusters-aws
	  manifests:
	    - path: input-standalone-clusters/red/aws/postgresql/
	  policySets:
	     - policyset-red-standalone-clusters-aws
	- name: policy-patch-postgresql-red-clusters-azure
	  manifests:
	    - path: input-standalone-clusters/red/azure/postgresql/
	  policySets:
	     - policyset-red-standalone-clusters-gcp
	- name: policy-patch-postgresql-red-clusters-gcp
	  manifests:
	    - path: input-standalone-clusters/red/gcp/postgresql/
	  policySets:
	     - policyset-red-standalone-clusters-gcp
	policySets:
	  - description: This policy set is focused on PostgreSQL components for the ACM hub.
	    name: policyset-hub-clusters
	    placement:
	      placementPath: input/placement-hub-clusters.yaml
	  - description: This policy set is focused on PostgreSQL components for managed OpenShift clusters on AWS.
	    name: policyset-red-standalone-clusters-aws
	    placement:
	      placementPath: input/placement-red-standalone-clusters-aws.yaml
	  - description: This policy set is focused on PostgreSQL components for managed OpenShift clusters on Azure.
	    name: policyset-red-standalone-clusters-azure
	    placement:
	      placementPath: input/placement-red-standalone-clusters-azure.yaml
	  - description: This policy set is focused on PostgreSQL components for managed OpenShift clusters on GCP.
	    name: policyset-red-standalone-clusters-gcp
	    placement:
	      placementPath: input/placement-red-standalone-clusters-gcp.yaml

An example placement for targeting development clusters in AWS labeled for PostgreSQL deployment is as follows:

	apiVersion: cluster.open-cluster-management.io/v1beta1
	kind: Placement
	metadata:
	  name: placement-red-standalone-clusters-aws
	  namespace: dba-policies
	spec:
	  clusterSets:
	    - red-cluster-set
	  predicates:
	  - requiredClusterSelector:
	      labelSelector:
	        matchExpressions:
	          - {key: vendor, operator: In, values: ["OpenShift"]}
                  - {key: name, operator: NotIn, values: ["local-cluster"]}
                  - {key: env, operator: In, values: ["dev"]}
                  - {key: postgresql, operator: In, values: ["pg-1"]}


Note that two of these labels (env and postgresql) were defined in the ClusterClaim. The other two are auto-populated by Hive as part of the provisioning process and ensure we are only deploying to OpenShift clusters and not on the hub itself. Also note that we explictely filter on the red-cluster-set in case the dba-policies namespace is bound to other managed cluster sets in future.

## Testing

Now that everything has been setup we can test for two common scenarios. The first is maintenance windows during which clusters are rebooted either due to an upgrade or because it is requested, and the other is unexpected catastraphic loss either due to an incident or misconfiguraiton in the cloud provider infrastructure that causes cascading failures. Both events are handled similarly from a PostgreSQL perspective in terms of triggering a failover to another standby server in the event that it is the cluster hosting the primary server that is affected. The difference arises beween the fact that when a cluster needs to be rebooted this is done from the RHACM console by placing the cluster in and out of hibernation. When the cluster comes out of hibernation it has the same name as before and the PostgreSQL server and PgPool can rely upon this fact during their starting cycle. In the event of a catastrophic failure the entire cluster could be lost and thus the process would entail submitting a ClusterClaim for the affected cluster using the same ClusterClaim name as the original. However because ClusterClaims result in the dynamic generation of a cluster name which will most likely be different this time the global service names that PostgreSQL and PgPool rely upon need to be updated. This would be tedious to do if we didn't use RHACM Policies and the Policy Generator tool which can do all this for us automatically provided we have enabled enforcing mode on the policy. The following shows how this works out in practice based on the configuration and status shown above which servers as the starting point for the test case.

When the surviving PostgreSQL standby servers detect the loss of the primary PostgreSQL server due to a catastrophic loss or reboot one of them automatically is protomoted to become the new primary PostgreSQL server to which the other standby server connects. PgPool continues to function as before directing queries to the surviving servers as shown in the following output.

When the cluster which was destroyed due to a catastrophic loss is restored is restored with a new name after SRE submit the ClusterClaim (that is the only step required to be performed and why ClusterClaims can contribute significantly to achieving very low RTO targets). MachinePools and Submariner configuration will automatically be triggered post-build provided these policies are left enabled and in enforcing mode. Once the cluster is available the SRE team can inform the DBA team who perform PostgreSQL installation after which the post-installation configuration is automatically performed if the polices are left enabled and in enforcing mode. Note in the output of PgPool that the new cluster names are automatically picked up and configured across all of the surviving PostgreSQL instances.








## Conclusion

In this installment we have extended the original cluster landing zone for hybrid cloud to support both stateless and stateful workloads and by distributing these workloads across multiple cloud providers we can protect ourselves from failures limited to a single node or availability zone, as well as larger-scale events such as the catastrophic loss of a cloud provider or region due to cascading failures or misconfiguration errors. The dynamic cluster generation capabilities of RHACM along with Policy templating and Submariner for hybrid network connectivity are key to building out this capability and enabling it within the context of a multi-tenanted operating model for managing the cluster fleet.
