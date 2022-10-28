# Guide to Cluster Landing Zones for Hybrid and Multi-cloud Architectures (Part 2)

## Overview

In this blog we look at how to realize a cluster landing zone for our hybrid cloud architecture that was introduced in the previous [blog](https://cloud.redhat.com/blog/a-guide-to-cluster-landing-zones-for-hybrid-and-multi-cloud-architectures). In this blog We will extend the multi-tenanted operating model underpinning our solution so that it can run both stateless and stateful workloads in a "cloud agnostic" manner. To this end we will make use of various tools within the Red Hat Advanced Cluster Management for Kubernetes (RHACM) toolbox to achieve a fully integrated and frictionless implementation.

## Extending the Cluster Landing Zone

In the previous blog we defined a multi-tenanted operating model for the hub site whereby we centralized management functions for both cluster admninistrators (SREs) and application teams so that they could independently provision and manage cluster lifecycles and application lifecycles. We now extend this to include DBAs who will provision stateful backend workloads that support frontend stateless workloads that application teams look after. The revised cluster landing zone operating model is depicted in the following diagram.

<p align="center">
  <img src="https://github.com/jwilms1971/blog/blob/main/acm/Cluster%20Landing%20Zones%20-%20Hybrid-cloud%20advanced.png">
  <em>Diagram 1. Cluster landing zone operating model for a hybrid cloud architecture</em>
</p>

In order to share the main GitOps engine located on the hub, SREs will need to configure Projects within ArgoCD to restrict DBAs to specific namespaces (dba-policies) and include restrictions on the kinds of resources that can be deployed to here, i.e., Policies, Placements, and ConfigMaps.

The diagram above shows how workers for each OpenShift cluster are segregated into distinct MachinePools which can be auto-scaled manually or automatically via the RHACM console. We will be using the default worker MachinePool for running stateless frontend workloads, and a separate MachinePool composed of a single worker node, for running stateful backend workloads. A PostgreSQL server will be deployed to this node in an active/standby configuration whereby the primary PostgreSQL server runs on AWS and the standby PostgreSQL server runs on GCP with replication manager configured and fronted by PgPool for client-side load-balancing and connection pooling.

## Deploying the Cluster Landing Zone

The next set of instructions are intended to be executed by our SREs as they pertain to cluster provisioning and configuration. The starting point here is a hub cluster with no managed clusters in existence. The YAML files shown here should go into a policy set aligned to provisioning tasks as shown in the diagram.

We start by defining an empty ManagedClusterSet that serves as a logical grouping for any clusters spawned from one or more cloud providers.

	apiVersion: cluster.open-cluster-management.io/v1beta1
	kind: ManagedClusterSet
	metadata:
	  name: red-cluster-set
	spec: {}

We create a namespace (dba-policies) and bind this to the ManagedClusterSet so that any policies written here can only be executed against clusters in the ManagedClusterSet. For more details on how ManagedClusterSets and namespace bindings work together please refer to [here](https://open-cluster-management.io/concepts/managedclusterset/#what-is-managedclusterset). Because we also need to generate some common configuration information that will be shared across all clusters hosting PostgreSQL, this needs to be stored on the hub cluster itself as a ConfigMap in the dba-policies namespaces. Thus we also need to allow any policies written to dba-policies to propagate to the hub which is identified by the default ManagedClusterSet.

	apiVersion: cluster.open-cluster-management.io/v1beta1
	kind: ManagedClusterSetBinding
	metadata:
	  name: red-cluster-set
	  namespace: dba-policies
	spec:
	  clusterSet: red-cluster-set
	---
	apiVersion: cluster.open-cluster-management.io/v1beta1
	kind: ManagedClusterSetBinding
	metadata:
	  name: default
	  namespace: dba-policies
	spec:
	  clusterSet: default

ClusterPools encapsulate details of the underlying cloud provider infrastructure and assign any clusters spawned from here into a ManagedClusterSet. The example shown is for AWS and for brevity detailed specifications such as the install config are omitted. A corresponding ClusterPool resource for GCP also needs to be defined with a non-overlapping network address space which is a Submariner pre-requisite.

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
	  baseDomain: # your domain name
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

To spawn a cluster we need to submit a ClusterClaim (similar in concept to how a PersistentVolumeClaim results in the creation of a PersistentVolume). The following ClusterClaims must be submitted and have custom labels that later on will help with mapping policies to managed clusters.

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
          clusterPoolName: red-cluster-pool-gcp-1

The resulting set of clusters is as follows.

	$ oc get managedclusters -A
	NAME                           HUB ACCEPTED   MANAGED CLUSTER URLS                                           JOINED   AVAILABLE   AGE
	local-cluster                  true           https://api.hub-cluster-1.aws.jwilms.net:6443                  True     True        5h38m
	red-cluster-pool-aws-1-4thqz   true           https://api.red-cluster-pool-aws-1-4thqz.aws.jwilms.net:6443   True     True        4h37m
	red-cluster-pool-gcp-1-8chvh   true           https://api.red-cluster-pool-gcp-1-8chvh.gcp.jwilms.net:6443   True     True        4h45m

Note that cluster names are dynamically generated by Hive. This should not be an issue provided applications are not being published using a cluster domain name in their URL. To avoid this applications can use the [appsDomain feature](https://docs.openshift.com/container-platform/4.11/networking/ingress-operator.html#nw-ingress-configuring-application-domain_configuring-ingress), or deploy a secondary ingress controller to decouple the application plane from the administration plane.

By default each cluster spins up with three worker nodes which will be used for hosting stateless frontend workloads. In the next step we add a machine pool for hosting stateful backend workloads which typically have their own performance needs and lifecycle. Each machine pool will have only a single worker node since we will be configuring PostgreSQL in an active/standby configuration across two clusters hosted across separate cloud providers.

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
	  replicas: 1
	
This YAML file uses templating to map a known quantity (the name of the ClusterClaim) to an unknown quantity (the name of the cluster). See the following [blog](https://cloud.redhat.com/blog/applying-policy-based-governance-at-scale-using-templates) for more details as well this [blog](https://cloud.redhat.com/blog/generating-governance-policies-using-kustomize-and-gitops) which introduces the Policy Generator tool which can processes YAML template files and turn them into policies that RHACM can then apply.

The MachinePool configuration as seen from the hub is as follows.

	$ oc get machinepools -A
	NAMESPACE                      NAME                                          POOLNAME         CLUSTERDEPLOYMENT              REPLICAS
	red-cluster-pool-aws-1-4thqz   red-cluster-pool-aws-1-4thqz-backend-worker   backend-worker   red-cluster-pool-aws-1-4thqz   1
	red-cluster-pool-aws-1-4thqz   red-cluster-pool-aws-1-4thqz-worker           worker           red-cluster-pool-aws-1-4thqz   3
	red-cluster-pool-gcp-1-8chvh   red-cluster-pool-gcp-1-8chvh-backend-worker   backend-worker   red-cluster-pool-gcp-1-8chvh   1
	red-cluster-pool-gcp-1-8chvh   red-cluster-pool-gcp-1-8chvh-worker           worker           red-cluster-pool-gcp-1-8chvh   3

With compute resources in place we next turn our attention to the cross-cluster network. What needs to happen here is basically a "flattening" of the networks between the managed clusters so that services and pods in one cluster have a direct line-of-sight to services and pods located in other clusters. This hybrid network connectivity happens at layer-3 of the OSI model and supports both TCP and UDP protocols across IPsec tunnels making it a very flexible, secure and a high-performance solution that is mostly transparent. Two gateways are deployed per cluster for high-availability. Please refer to the [documentation](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.6/html/add-ons/add-ons-overview#preparing-selected-hosts-to-deploy-submariner) for prerequisites and more details on Submariner.

To configure Submariner we submit the following YAML as per this example for AWS, again using templating to derive cluster names.


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

Check the status of the Submariner network from the RHACM console before continuing.

<p align="center">
  <img src="https://github.com/jwilms1971/blog/blob/main/acm/Screenshot from 2022-10-26 12-54-17.png">
  <em>Diagram 2. Submariner setup for two clusters spanning two cloud providers</em>
</p>

## Deploying PostgreSQL

All of the steps shown so far are intended to be executed by SREs who are responsible for the cluster fleet including configuration, capacity and networking. The next set of steps are intended to be executed by our DBAs using RHACM Policies which will be written to the dba-policies namespace via the OpenShift GitOps instance running on the hub. From here they will be picked up by the Policy controller and applied to all managed clusters based on Placement resources defined later.

Depending on whether you are using Operators or Helm charts to install PostgreSQL the steps may vary and are well-covered by the respective distribution providers. The main considerations for deploying PostgreSQL in a hybrid cloud environment include tuning of timeouts related to network latency and high-performance block storage. Other factors related to a production setup are discussed later. In our setup we will be focusing on deploying PostgreSQL in an active/standby configuration into a hybrid network established by Submariner, such that the active primary instance runs on a cluster in AWS and the passive standby instance runs on a cluster in GCP. Both PostgreSQL servers are part of a replication cluster and fronted by PgPool for load-balancing and connection pooling.

To complete the integration of PostgreSQL with the hybrid network that Submariner has created requires specific configuration for PostgreSQL to connect to services on the clusterset.local domain instead of the usual cluster.local domain. This domain is created for us by Submariner when we generate ServiceExport resources for our headless service that gets created by the StatefulSet resource that deploys PostgreSQL.

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

Next we need to set environment variables for each PostgreSQL server to use these clusterset.local service names. This must be generated and stored on the hub as this is the only system which has knowledge of all the managed clusters. This will then be referenced by subsequent YAML that target specific managed clusters.

	apiVersion: v1
	kind: ConfigMap
	metadata:
	  name: pg-config
	  namespace: dba-policies
	data:
	  nodeid0: '1000'
	  nodeid1: '1001'
	  nodename0: 'pg-1-postgresql-ha-postgresql-0'
	  nodename1: 'pg-2-postgresql-ha-postgresql-0'
	  hostname0: 'pg-1-postgresql-ha-postgresql-0.{{- (lookup "hive.openshift.io/v1" "ClusterClaim" "red-cluster-pool" "red-cluster-1").spec.namespace -}}.pg-1-postgresql-ha-postgresql-headless.database.svc.clusterset.local'
	  hostname1: 'pg-2-postgresql-ha-postgresql-0.{{- (lookup "hive.openshift.io/v1" "ClusterClaim" "red-cluster-pool" "red-cluster-2").spec.namespace -}}.pg-2-postgresql-ha-postgresql-headless.database.svc.clusterset.local'

The following YAML extracts the values from the ConfigMap and puts them into the correct environment variables in the right order. These YAML files will be used by RHACM Policies to patch PostgreSQL servers and PgPool post-installation.

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
	          value: '{{hub fromConfigMap "" "pg-config" (printf "hostname0") hub}},{{hub fromConfigMap "" "pg-config" (printf "hostname1") hub}}'
	        - name: REPMGR_PRIMARY_HOST
	          value: '{{hub fromConfigMap "" "pg-config" (printf "hostname0") hub}}'
	        - name: REPMGR_NODE_NETWORK_NAME
	          value: '{{hub fromConfigMap "" "pg-config" (printf "hostname0") hub}}'
	        image: # set this to your postgresql container image
	        name: postgresql
	      initContainers:
	      - image: # set this to your postgresql container image
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
                  value: '0:{{hub fromConfigMap "" "pg-config" (printf "hostname0") hub}}:5432,1:{{hub fromConfigMap "" "pg-config" (printf "hostname1") hub}}:5432'
	        image: # set this to you pgpool container image
	        name: pgpool

Repeat the above for the standby PostgreSQL server represented by the prefix pg-2, adjusting environment variables accordingly.

Here is the contents of the Policy Generator configuration file that will be used to ingest all of the files above stored in their respective subdirectories depending on which type of cluster the policy should be executed on. Note that all policies will be loaded into RHACM in a disabled state so that they can be enabled only after the DBA has finished with the baseline installation of PostgreSQL on each cluster. In our case this was performed using a Helm chart with replicas set to zero.

	apiVersion: policy.open-cluster-management.io/v1
	kind: PolicyGenerator
	metadata:
	  name: policy-postgresql-provisioning
	placementBindingDefaults:
	  name: binding-policy-postgresql-provisioning
	policyDefaults:
	  namespace: dba-policies
	  complianceType: musthave
	  remediationAction: enforce
	  severity: low
	policies:
	- name: policy-generate-postgresql-config-hub
	  manifests:
	    - path: input-hub-clusters/postgresql/
	  disabled: true
	  policySets:
	    - policyset-postgresql-hub-clusters
	- name: policy-patch-postgresql-red-clusters-aws-1
	  manifests:
	    - path: input-standalone-clusters/red/aws-1/postgresql/
	  policySets:
	     - policyset-postgresql-red-standalone-clusters-aws-1
	  disabled: true
	- name: policy-patch-postgresql-red-clusters-gcp-1
	  manifests:
	    - path: input-standalone-clusters/red/gcp-1/postgresql/
	  policySets:
	     - policyset-postgresql-red-standalone-clusters-gcp-1
	  disabled: true
	policySets:
	  - description: This policy set is focused on PostgreSQL components for the ACM hub.
	    name: policyset-postgresql-hub-clusters
	    placement:
	      placementPath: input/placement-hub-clusters.yaml
	  - description: This policy set is focused on PostgreSQL components for managed OpenShift clusters on AWS.
	    name: policyset-postgresql-red-standalone-clusters-aws-1
	    placement:
	      placementPath: input/placement-red-standalone-clusters-aws-1.yaml
	  - description: This policy set is focused on PostgreSQL components for managed OpenShift clusters on GCP.
	    name: policyset-postgresql-red-standalone-clusters-gcp-1
	    placement:
	      placementPath: input/placement-red-standalone-clusters-gcp-1.yaml

The Policy Generator leverages Placements to map Policies to clusters. An example Placement resource is as follows.

	apiVersion: cluster.open-cluster-management.io/v1beta1
	kind: Placement
	metadata:
	  name: placement-red-standalone-clusters-aws-1
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

Here we are using a mix of custom and auto-generated labels to identify clusters in scope. At no point do we ever refer to the dynamically generated name of the cluster given that this name will change whenever the cluster is rebuilt. Also note that we include a filter for clusterSets here as it is possible that our DBA may be managing clusters for multiple teams using similar labels, and thus the filter helps to ensure that policies target only clusters belonging to the red team.

## Simulating Cloud Provider Failure

Once our policies have been enabled via the RHACM console triggering the Policy controller to patch the PostgreSQL statefulset and PgPool deployment with the settings from the ConfigMap which will be transferred to the managed clusters. In our setup we have limited the number of PgPool replicas to just one for simplification. In a production setup the number of PgPool replicas should match the scaling needs of the application and uptime SLA requirements.

At this stage the resources deployed to the AWS cluster look as per the following output. 

	$ oc get all,pvc,endpointslices
	NAME                                            READY   STATUS    RESTARTS   AGE
	pod/pg-1-postgresql-ha-pgpool-bf5b69cbb-h6pfl   1/1     Running   0          24m 
	pod/pg-1-postgresql-ha-postgresql-0             1/1     Running   0          24m 
	
	NAME                                             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
	service/pg-1-postgresql-ha-pgpool                ClusterIP   172.30.81.120   <none>        5432/TCP   24m
	service/pg-1-postgresql-ha-postgresql            ClusterIP   172.30.48.22    <none>        5432/TCP   24m
	service/pg-1-postgresql-ha-postgresql-headless   ClusterIP   None            <none>        5432/TCP   24m
	
	NAME                                        READY   UP-TO-DATE   AVAILABLE   AGE
	deployment.apps/pg-1-postgresql-ha-pgpool   1/1     1            1           24m
	
	NAME                                                   DESIRED   CURRENT   READY   AGE
	replicaset.apps/pg-1-postgresql-ha-pgpool-bf5b69cbb    1         1         1       24m 
	
	NAME                                             READY   AGE
	statefulset.apps/pg-1-postgresql-ha-postgresql   1/1     24m
	
	NAME                                                         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
	persistentvolumeclaim/data-pg-1-postgresql-ha-postgresql-0   Bound    pvc-66b68308-5e44-49a1-94e8-655fa0c8475d   8Gi        RWO            gp3-csi        24m 
	
	NAME                                                                                                 ADDRESSTYPE   PORTS   ENDPOINTS     AGE
	endpointslice.discovery.k8s.io/pg-1-postgresql-ha-pgpool-75wjr                                       IPv4          5432    10.128.2.28   24m
	endpointslice.discovery.k8s.io/pg-1-postgresql-ha-postgresql-d2rhs                                   IPv4          5432    10.130.2.6    24m
	endpointslice.discovery.k8s.io/pg-1-postgresql-ha-postgresql-headless-4fbxh                          IPv4          5432    10.130.2.6    24m
	endpointslice.discovery.k8s.io/pg-1-postgresql-ha-postgresql-headless-red-cluster-pool-aws-1-4thqz   IPv4          5432    10.130.2.6    24m
	endpointslice.discovery.k8s.io/pg-2-postgresql-ha-postgresql-headless-red-cluster-pool-gcp-1-8chvh   IPv4          5432    10.134.2.6    24m

The last two endpointslices composed with cluster names identify the IP addresses of the local and remote PostgreSQL servers on both AWS and GCP clusters and are the result of the ServiceExport resource.

The next step is to test for catastrophic loss of a cloud provider region which can happen due to a cascading failure, misconfiguration error or vulnerability exploit that affects the control plane logic. This can be simulated using the RHACM Hibernate feature to stop a running cluster resulting in loss of network connectivity between the primary and standby PostgreSQL servers. After a number of failed re-connection attempts, the decision logic within the PostgreSQL replication manager will promote the standby server into a primary role. Before we pull the plug on the OpenShift cluster on AWS, let's take a look at how the replication manager and PgPool currently view the health of the PostgreSQL servers.

	$ repmgr -f /opt/bitnami/repmgr/conf/repmgr.conf service status
	ID   | Name                            | Role    | Status    | Upstream                        | repmgrd | PID | Paused? | Upstream last seen
	-----+---------------------------------+---------+-----------+---------------------------------+---------+-----+---------+--------------------
	1000 | pg-1-postgresql-ha-postgresql-0 | primary | * running |                                 | running | 1   | no      | n/a                
	1001 | pg-2-postgresql-ha-postgresql-0 | standby |   running | pg-1-postgresql-ha-postgresql-0 | running | 1   | no      | 0 second(s) ago    

	postgres=# show pool_nodes;
	-[ RECORD 1 ]----------+----------------------------------------------------------------------------------------------------------------------------------
	node_id                | 0
	hostname               | pg-1-postgresql-ha-postgresql-0.red-cluster-pool-aws-1-4thqz.pg-1-postgresql-ha-postgresql-headless.database.svc.clusterset.local
	port                   | 5432
	status                 | up
	pg_status              | up
	lb_weight              | 0.500000
	role                   | primary
	pg_role                | primary
	select_cnt             | 111
	load_balance_node      | true
	replication_delay      | 0
	replication_state      | 
	replication_sync_state | 
	last_status_change     | 2022-10-26 00:55:34
	-[ RECORD 2 ]----------+----------------------------------------------------------------------------------------------------------------------------------
	node_id                | 1
	hostname               | pg-2-postgresql-ha-postgresql-0.red-cluster-pool-gcp-1-8chvh.pg-2-postgresql-ha-postgresql-headless.database.svc.clusterset.local
	port                   | 5432
	status                 | up
	pg_status              | up
	lb_weight              | 0.500000
	role                   | standby
	pg_role                | standby
	select_cnt             | 113
	load_balance_node      | false
	replication_delay      | 0
	replication_state      | 
	replication_sync_state | 
	last_status_change     | 2022-10-26 00:55:44

Everthing looks good and we can now login to the RHAC console and hibernate the cluster hosted on AWS which is running the primary PostgreSQL server.

<p align="center">
  <img src="https://github.com/jwilms1971/blog/blob/main/acm/Screenshot from 2022-10-26 09-37-46.png">
  <em>Diagram 3. Hibernating a cluster to simulate the catastrophic loss of a cloud provider</em>
</p>

After a short while the standby PostgreSQL server is promoted to the primary role which can be seen in the following output.

	$ repmgr -f /opt/bitnami/repmgr/conf/repmgr.conf service status
	ID   | Name                            | Role    | Status    | Upstream | repmgrd | PID | Paused? | Upstream last seen
	-----+---------------------------------+---------+-----------+----------+---------+-----+---------+--------------------
	1000 | pg-1-postgresql-ha-postgresql-0 | primary | - failed  | ?        | n/a     | n/a | n/a     | n/a                
	1001 | pg-2-postgresql-ha-postgresql-0 | primary | * running |          | running | 1   | no      | n/a                
	
	WARNING: following issues were detected
	  - unable to  connect to node "pg-1-postgresql-ha-postgresql-0" (ID: 1000)

PgPool is also able to continue servicing clients by sending their queries to the new primary.

Assuming our cloud provider doesn't stay offline indefinitely we need to prepare for the eventual restoration of the failed PostgreSQL server. In the configuration above we defined an environment variable to identify the primary PostgreSQL server which is configured identical on both clusters.

	- name: REPMGR_PRIMARY_HOST
          value: '{{hub fromConfigMap "" "pg-config" (printf "hostname0") hub}}'

If we bring back online the cluster hosting the failed PostgreSQL server it will continue to think that it still running as the primary server, resulting in a split-brain situation. To avoid this we update the YAML to reflect the new reality in our cluster and let the Policy controller propagate these changes when the cluster comes online. The changes required to the StatefulSet configuration for both PostgreSQL servers is to swap hostname0 with hostname1 for the environment variable REPMGR_PRIMARY_HOST as follows.

	- name: REPMGR_PRIMARY_HOST
	  value: '{{hub fromConfigMap "" "pg-config" (printf "hostname1") hub}}'

After making these changes, the next step is to resume the cluster which will patch and restart the former primary as a standby server. Note that there is no need to update the PgPool deployment configuration. 

Checking the output of the replication manager after both primary and standby servers have been updated and restarted shows the following.

	$ repmgr -f /opt/bitnami/repmgr/conf/repmgr.conf service status
	ID   | Name                            | Role    | Status    | Upstream                        | repmgrd | PID | Paused? | Upstream last seen
	-----+---------------------------------+---------+-----------+---------------------------------+---------+-----+---------+--------------------
	1000 | pg-1-postgresql-ha-postgresql-0 | standby |   running | pg-2-postgresql-ha-postgresql-0 | running | 1   | no      | 1 second(s) ago    
	1001 | pg-2-postgresql-ha-postgresql-0 | primary | * running |                                 | running | 1   | no      | n/a                

The output for PgPool is as follows.

	postgres=# show pool_nodes;
	-[ RECORD 1 ]----------+----------------------------------------------------------------------------------------------------------------------------------
	node_id                | 0
	hostname               | pg-1-postgresql-ha-postgresql-0.red-cluster-pool-aws-1-4thqz.pg-1-postgresql-ha-postgresql-headless.database.svc.clusterset.local
	port                   | 5432
	status                 | up
	pg_status              | up
	lb_weight              | 0.500000
	role                   | primary
	pg_role                | standby
	select_cnt             | 14
	load_balance_node      | false
	replication_delay      | 0
	replication_state      | 
	replication_sync_state | 
	last_status_change     | 2022-10-26 02:22:34
	-[ RECORD 2 ]----------+----------------------------------------------------------------------------------------------------------------------------------
	node_id                | 1
	hostname               | pg-2-postgresql-ha-postgresql-0.red-cluster-pool-gcp-1-8chvh.pg-2-postgresql-ha-postgresql-headless.database.svc.clusterset.local
	port                   | 5432
	status                 | up
	pg_status              | up
	lb_weight              | 0.500000
	role                   | standby
	pg_role                | primary
	select_cnt             | 11
	load_balance_node      | true
	replication_delay      | 83914888
	replication_state      | 
	replication_sync_state | 
	last_status_change     | 2022-10-26 02:22:34

Notice that PgPool correctly reports the new primary and standby roles in the column pg_role but reports the wrong values in the role column. In practice this does not appear to matter as read/write queries are routed correctly. After restarting PgPool both sets of values will be correctly aligned. 

Some parting thoughts:

* For a production environment we could add a third standby (on a separate cloud provider) or a witness node (on a HyperShift cluster) to establish quorum so as to avoid other possible split-brain scenarios and for strong fencinge. Doing so would move help us to achieve a RTO approaching zero.

* We can make use of the fact that cloud providers located in the same region typically have a network latency of under 10 milliseconds. Thus our local PostgreSQL clusters can be configured for synchronous replication achieving a RPO of zero. This could be augmented with asynchronous replication to remote PostgreSQL clusters located in another region for disaster recovery purposes.

* Deploy a global load-balancer that spans across multiple cluster ingress endpoints and is itself not dependent on the infrastructure of any of the underlying cloud providers.

## Conclusion

In this blog we have shown how tools from the RHACM toolbox can be used together to build non-trivial solutions using a open hybrid cloud architecture, thereby achieving higher levels of availability and scalability. This in turn is underpinned by an extensible multi-tenant operating model allowing multiple teams to securely access, operate and deploy resources across clusters and clouds in a frictionless manner.
