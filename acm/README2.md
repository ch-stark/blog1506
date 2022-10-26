# Guide to Cluster Landing Zones for Hybrid and Multi-cloud Architectures (Part 2)

## Overview

In this blog we look at how to realize the cluster landing zone for hybrid cloud that was introduced in the previous [blog](https://cloud.redhat.com/blog/a-guide-to-cluster-landing-zones-for-hybrid-and-multi-cloud-architectures) with full support for both stateless and stateful workloads in a cloud-agnostic manner, including testing of failover and failback. To this end we will make extensive use of various tools within the Red Hat Advanced Cluster Management (RHACM) toolbox to compose our solution.

## Extending the Cluster Landing Zone

In the previous blog we defined a multi-tenanted operating model on the hub for individual application teams to manage stateless workloads using ArgoCD and SREs to perform cluster administration using policies for all cluster lifecycle operations. In this blog we extend this model to include DBAs who also need to perform operations on clusters such as provisioning databases. This will be accomplished via policies given their flexible and powerful nature. The revised cluster landing zone model is depicted in the following diagram.

<p align="center">
  <img src="https://github.com/jwilms1971/blog/blob/main/acm/Cluster%20Landing%20Zones%20-%20Hybrid-cloud%20advanced.png">
  <em>Diagram 1. Cluster landing zone for a hybrid cloud architecture</em>
</p>

Projects within ArgoCD will need to be configured by SREs to allow DBAs to deploy policies to a specific namespace (dba-policies) with restrictions placed on the kinds of resources (Policies, Placements, ConfigMaps) that can be deployed to here as per principle of least privilege. Via RBAC our DBAs are not able to edit the contents of any other namespace on the hub.

The diagram introduces MachinePools which are hub-side constructs that define and scale a set of workers either via the RHACM console or via YAML. We will be using the default pool for running stateless workloads and a separate machine pool (composed of a single node) for running a stateful workload (PostgreSQL). PostgreSQL is a common open-source database used in many cloud-native projects and has in-built replication and failover capabilities which we will be exploring in this blog.

## Deploying the Cluster Landing Zone

The next set of instructions are intended to be executed by the SREs as they pertain to cluster configuration. The starting point here is a hub cluster with no managed clusters in existence. The YAML files shown here should go into a policy set such as openshift-provisioning as shown in the diagram.

We start by defining an empty ManagedClusterSet that serves as a logical grouping for clusters spawned from one or more cloud providers.

	apiVersion: cluster.open-cluster-management.io/v1beta1
	kind: ManagedClusterSet
	metadata:
	  name: red-cluster-set
	spec: {}

We create a namespace (dba-policies) and bind this to the managed cluster set so that any policies written to here can be executed against the clusters in the managed cluster set (the policy controller checks for this binding). Because we also need to generate some common configuration information that needs to be shared across all of the clusters hosting PostgreSQL servers this needs to be stored on the hub cluster itself as a ConfigMap and thus we also need to be bind the dba-policies namespace to the hub.

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

ClusterPools encapsulate details of the underlying cloud provider infrastructure and assign any clusters spawned from it to a managed cluster set. For brevity, detailed specification of referenced resources are not shown and the example shown is for AWS. A corresponding ClusterPool for GCP needs to be defined with a non-overlapping network address space which is a Submariner pre-requisite.

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

To spawn a cluster we need to submit a ClusterClaim (which is similar in concept to how a PersistentVolumeClaim results in the creation of a PersistentVolume). The following cluster claims must be submitted and contain custom labels that later on will help with mapping policies to managed clusters.

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

The resulting set of clusters are as follows.

	$ oc get managedclusters -A
	NAME                           HUB ACCEPTED   MANAGED CLUSTER URLS                                           JOINED   AVAILABLE   AGE
	local-cluster                  true           https://api.hub-cluster-1.aws.jwilms.net:6443                  True     True        5h38m
	red-cluster-pool-aws-1-4thqz   true           https://api.red-cluster-pool-aws-1-4thqz.aws.jwilms.net:6443   True     True        4h37m
	red-cluster-pool-gcp-1-8chvh   true           https://api.red-cluster-pool-gcp-1-8chvh.gcp.jwilms.net:6443   True     True        4h45m

Note that cluster names are dynamically generated when using ClusterClaims. This should not be an issue provided applications are not exposed using the cluster domain name, i.e., applications should use the [appsDomain feature](https://docs.openshift.com/container-platform/4.11/networking/ingress-operator.html#nw-ingress-configuring-application-domain_configuring-ingress), or deploy a secondary ingress controller to decouple the application plane from the administration plane.

By default each cluster spins up with three worker nodes which will be used for hosting stateless workloads. In the next step we add a machine pool for hosting stateful workloads which typically have their own performance needs. Each machine pool will have only a single worker node since we will be configuring PostgreSQL in an active/standby configuration across two clusters in separate cloud providers.

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
	
This YAML file uses policy templating constructs to map a known quantity (the name of the cluster claim) to an unknown quantity (the name of the cluster). See the following [blog](https://cloud.redhat.com/blog/applying-policy-based-governance-at-scale-using-templates) for more details as well this [blog](https://cloud.redhat.com/blog/generating-governance-policies-using-kustomize-and-gitops) which introduces the Policy Generator tool which processes YAML files and turns them into policies.

After having applied the generated policies to our clusters their machine configuration as seen from the hub is as follows.

	$ oc get machinepools -A
	NAMESPACE                      NAME                                          POOLNAME         CLUSTERDEPLOYMENT              REPLICAS
	red-cluster-pool-aws-1-4thqz   red-cluster-pool-aws-1-4thqz-backend-worker   backend-worker   red-cluster-pool-aws-1-4thqz   1
	red-cluster-pool-aws-1-4thqz   red-cluster-pool-aws-1-4thqz-worker           worker           red-cluster-pool-aws-1-4thqz   3
	red-cluster-pool-gcp-1-8chvh   red-cluster-pool-gcp-1-8chvh-backend-worker   backend-worker   red-cluster-pool-gcp-1-8chvh   1
	red-cluster-pool-gcp-1-8chvh   red-cluster-pool-gcp-1-8chvh-worker           worker           red-cluster-pool-gcp-1-8chvh   3

With the compute resources in place we next turn our attention to the network. What needs to happen here is basically a "flattening" of the networks between the clusters so that services and pods in one cluster have direct line-of-sight to services and pods in other clusters. This hybrid network connectivity happens at layer-3 of the OSI model and supports TCP and UDP protocols across IPsec tunnels making it a very flexible and high-performance solution. Two gateways are deployed per cluster for high-availability. Please refer to the [documentation](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.6/html/add-ons/add-ons-overview#preparing-selected-hosts-to-deploy-submariner) for prerequisites and more details on Submariner.

To configure Submariner we submit the following YAML as per the example for AWS.


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

Check the status of Submariner network connectivity from the RHACM console before continuing.


<p align="center">
  <img src="https://github.com/jwilms1971/blog/blob/main/acm/Screenshot from 2022-10-26 12-54-17.png">
  <em>Diagram 2. Submariner setup for two clusters spanning two cloud providers</em>
</p>

## Deploying PostgreSQL

All of the steps shown so far are intended to be executed by SREs who are responsible for the cluster fleet including capacity and networking. The next set of steps are intended to be executed by our DBAs using RHACM policies which will be mapped into the dba-policies namespace via the hub cluster OpenShift GitOps instance. From here they will be picked up by the policy controller and applied to the clusters based on placement resources defined later.

Depending on whether you are using Operators or Helm charts to install PostgreSQL the steps may vary and are well-covered by the respective providers. The main considerations for deploying PostgreSQL in a hybrid cloud environment include tuning of timeouts related to network latency and high-performance block storage. Other factors related to a production setup are discussed later. In this setup we will be focusing on connecting PostgreSQL in an active/standby configuration to a hybrid network established by Submariner such that the active primary instance runs on one cluster and the passive standby instance runs on another cluster all controlled by a replication manager and PgPool for client load-balancing and connection pooling.

To complete the integration of PostgreSQL with the "flattened" network that Submariner establishes between clusters requires some additional configuration. The intention is that this configuraiton is saved in YAML files that are then processed by the policy generator tool as a post-installation step as will be shown later.

The first thing that is required is to export the headless service of each PostgreSQL statefulset to the other clusters so that the PostgreSQL servers can communicate to each other just like if they were deployed co-resident within a single cluster.

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

Next we need to set the environment variables of each PostgreSQL server to use these global service names. This requires that we first define a common list of values that can be shared between clusters as they will need to know each other's identities and settings. This information must be staged on the hub (preferably using a policy that executes on the hub) so that downstream polices that configure PostgreSQL on a managed cluster can reference this information using the hub template construct as shown below.

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

The next set of YAML references these pre-populated values using the hub template construct and a fromConfigMap function as these policies themselves are meant to be evaluated on the managed clusters. Both the statefulset for PostgreSQL and deployment for PgPool need to be patched. Also note that replicas are now defined because it only makes sense to do so now and not at installation time (they should be set to zero).

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

Repeat for the standby PostgreSQL server represented by the prefix pg-2 adjusting environment variables accordingly.

Here is the contents of the policy generator configuration for ingesting these YAML files which themselves are stored inside various subdirectories. Note that the policies are all loaded with the disabled flag set to true as the intention is to enable them from within the RHACM console once the DBA is satisfied with the state of the clusters and has completed installing a baseline PostgreSQL environment on each.

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

The policy generator leverages placements to map policies to clusters. An example placement resource is as follows.

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

We are using a mix of custom and auto-generated labels to evaluate the clusters in scope for having the policy applied to it. At no point do we ever refer to the name of the cluster which is dynamically generated and will change if the cluster were to be re-created. Also note that we include a filter for clusterSets as it is possible that our DBA may be managing clusters across multiple teams using similar labels and thus this filter helps to ensure the policies target only clusters belonging to the red team.

## Testing

Once our policies have been enabled from the RHACM console, the PostgreSQL statefulset and PgPool deployment will be patched with settings contained in the ConfigMap that are pulled from the hub by the policy controller. In our setup we have limited the number of PgPool replicas to one for simplification. In a production setup the number of PgPool replicas should match the scaling needs of the application and uptime SLA.

At this stage the resources deployed to one of the clusters looks as per the following output. 

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

The last two endpointslices composed with cluster names identify the IP addresses of the local and remote PostgreSQL server pods and were created by Submariner.

The next step is to test for the catastrophic loss of a cloud provider. This can be simulated using the RHACM hibernate feature to stop a running cluster resulting in the loss of network communication between the primary and standby PostgreSQL servers. After a number of failed re-connection attempts, this triggers the replication manager to promote the standby into a primary role. Before we pull the plug on the cluster let's take a look at how the replication manager and PgPool view system health.

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

The results are as expected and we can now login to the RHAC console and hibernate the cluster running the primary PostgreSQL server which based on the output above is red-cluster-pool-aws-1-4thqz.

<p align="center">
  <img src="https://github.com/jwilms1971/blog/blob/main/acm/Screenshot from 2022-10-26 09-37-46.png">
  <em>Diagram 3. Hibernating a cluster to simulate the catastrophic loss of a cloud provider</em>
</p>

After a short while the standby PostgreSQL server is promoted to primary which can be seen in the following output obtained after the standby comes back online as the new primary.

	$ repmgr -f /opt/bitnami/repmgr/conf/repmgr.conf service status
	ID   | Name                            | Role    | Status    | Upstream | repmgrd | PID | Paused? | Upstream last seen
	-----+---------------------------------+---------+-----------+----------+---------+-----+---------+--------------------
	1000 | pg-1-postgresql-ha-postgresql-0 | primary | - failed  | ?        | n/a     | n/a | n/a     | n/a                
	1001 | pg-2-postgresql-ha-postgresql-0 | primary | * running |          | running | 1   | no      | n/a                
	
	WARNING: following issues were detected
	  - unable to  connect to node "pg-1-postgresql-ha-postgresql-0" (ID: 1000)

PgPool is also able to continue servicing clients by sending their queries to the new primary.

Assuming our cloud provider doesn't stay offline indefinitely we need to prepare for the eventual restoration of service. In the configuration above we defined an environment variable to identify the primary PostgreSQL server.

	- name: REPMGR_PRIMARY_HOST
          value: '{{hub fromConfigMap "" "pg-config" (printf "hostname0") hub}}'

If we bring back online the cluster hosting the original primary with the original configuration it will continue to think that it still is the primary resulting in a split-brain situation. To avoid this we simply edit the policies in our git repo and update them to reflect the new reality in our cluster and let the policy controller propagate these changes when the cluster comes online. For the avoidance of doubt the change to the statefulset required is as follows.

	- name: REPMGR_PRIMARY_HOST
	  value: '{{hub fromConfigMap "" "pg-config" (printf "hostname1") hub}}'

Do note that making this change to the policy managing the configuration of the new primary will trigger a restart so again this must all be co-ordinated as part of a pre-flight service restoration checklist to minimize impact. After completing these changes, the next step is to resume the cluster itself which will then patch and start the former primary as a standby server.

Note that there is no need to update the PgPool deployment as there is no defined primary setting in its list of configurable parameters.

Once the cluster starts and the former PostgreSQL primary server is started as a standby it joins the replication cluster as shown in the following output.

	$ repmgr -f /opt/bitnami/repmgr/conf/repmgr.conf service status
	ID   | Name                            | Role    | Status    | Upstream                        | repmgrd | PID | Paused? | Upstream last seen
	-----+---------------------------------+---------+-----------+---------------------------------+---------+-----+---------+--------------------
	1000 | pg-1-postgresql-ha-postgresql-0 | standby |   running | pg-2-postgresql-ha-postgresql-0 | running | 1   | no      | 1 second(s) ago    
	1001 | pg-2-postgresql-ha-postgresql-0 | primary | * running |                                 | running | 1   | no      | n/a                

The view from PgPool is as follows.

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

Notice that PgPool correctly reports the new primary and standby roles in the column pg_role but reports the old values in the role column. In practice this does not appear to matter as read/write queries are both processed correctly. After restarting PgPool both sets of values are correctly aligned. This concludes the test.

Some parting thoughts:

* For a production environment we could add a third standby (on a separate cloud provider) or a witness node (on a HyperShift cluster) to establish quorum so as to avoid various split-brain scenarios and for strong fencing which is considered a best practice. Doing so would also enable a transition to a solution with a RTO approaching zero.

* We can make use of the fact that cloud providers in the same region typically have low network latency (<10ms) between them. Thus the local PostgreSQL cluster can be configured for synchronous replication enabling a RPO of zero. This could be augmented with asynchronous replication to remote PostgreSQL clusters located in another region for protection against a large-scale disaster.

* Deploy a global load-balancer that spans across multiple cluster ingress endpoints and is itself not dependent on the availability of any of the underlying cloud providers so as to avoid a shared fate situation.

## Conclusion

In this blog we have shown how tools from the RHACM toolbox can be used together to build non-trivial open hybrid cloud architectures with higher levels of availability and scaleability than what is possible with a multi-cloud architecture. This in turn is underpinned by an extensible multi-tenant operating model that forms the basis of a cluster landing zone allowing multiple teams to securely access, operatate and deploy resources across clouds in a frictionless manner.
