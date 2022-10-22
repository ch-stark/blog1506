# Guide to Cluster Landing Zones for Hybrid and Multi-cloud Architectures (Part 2)

## Overview

In this installment we look at extending the hybrid cloud architecture introduced [previously](https://cloud.redhat.com/blog/a-guide-to-cluster-landing-zones-for-hybrid-and-multi-cloud-architectures) to support stateful application workloads in addition to stateless ones already covered. We also demonstrate how the architecture deals with disaster scenarios such as the catastrophic failure of a cloud provider. We make extensive useage of various tools within the Red Hat Advanced Cluster Management toolbox and show how these can work together to deliver a robust solution.

## Cluster Landing Zone

In order to support stateful workloads such as databases we need to extend the original multi-tenancy operating model to now include DBAs in addition to application teams and cluster administrators (SREs). The revised model looks as per this diagram.


This extension is required because granting application teams direct access to Policy resources to provision databases and their underpinning resources would be considered an anti-pattern given that the Policy Controller runs with elevated privileges. Separating application concerns from database and cluster concerns by team is a common enterprise practice which the multi-tnancy operating model easily enables.



The following diagram depicts the cluster landing zone that we are going to build and the components that will be deployed.


For simplicity, the configuration of global load balancers to enable users external to the cluster to continue operation is not shown. Postgres was chosen as the database backend given it's popularity. Similar results should be achievable using other database products.

Before we look at the configuration of the networking layer which is the foundation for inter-cluster connectivty, let's quickly cover the cluster deployment itself and the usage of the MachinePool API made available with Red Hat Advanced Cluster Management (RHACM) that enables the SRE team to isolate the needs of different workloads, in this case application (frontend) and database (backend). In this [blog](https://cloud.redhat.com/blog/securing-ingress-controllers-on-a-managed-openshift-cluster-using-red-hat-advanced-cluster-management) we cover setting up a ClusterPool as a kind of templating engine for spawning clusters on-demand by submitting a ClusterClaim object.

Let's assume we have defined one ClusterPool per cloud provider with an initial pool size of zero. The abbreviated specification is as follows:

	apiVersion: hive.openshift.io/v1
	kind: ClusterPool
	metadata:
	  name: 'red-cluster-pool-1'
	  namespace: 'red-cluster-pool'
	  labels:
	    cloud: 'AWS'
	    region: 'ap-southeast-1'
	    vendor: OpenShift
	    cluster.open-cluster-management.io/clusterset: 'red-cluster-set'
	spec:
	  size: 0
	  ...  
	---
	apiVersion: hive.openshift.io/v1
	kind: ClusterPool
	metadata:
	  name: 'red-cluster-pool-2'
	  namespace: 'red-cluster-pool'
	  labels:
	    cloud: 'GCP'
	    region: 'asia-southeast1'
	    vendor: OpenShift
	    cluster.open-cluster-management.io/clusterset: 'red-cluster-set'
	spec:
	  size: 0
	...  
	apiVersion: hive.openshift.io/v1
	kind: ClusterPool
	metadata:
	  name: 'red-cluster-pool-3'
	  namespace: 'red-cluster-pool'
	  labels:
	    cloud: 'Azure'
	    region: 'southeastasia'
	    vendor: OpenShift
	    cluster.open-cluster-management.io/clusterset: 'red-cluster-set'
	spec:
	  size: 0
	...  

Note that all three ClusterPools are linked into a common ManagedClusterSet defined as follows:

	apiVersion: cluster.open-cluster-management.io/v1beta1
	kind: ManagedClusterSet
	metadata:
	  name: red-cluster-set
	spec: {}

As will be shown later the ManagedClusterSet is key to enabling cross-cluster connectivity and workload distribution.

Finally here are the cluster claims that when submitted result in clusters being spawned by the Hive API in the underlying cloud provider substrate with an amalgamated set of labels based on what was defined for the ClusterPool and the ClusterClaim.

	apiVersion: hive.openshift.io/v1
	kind: ClusterClaim
	metadata:
	  name: red-cluster-1
	  namespace: red-cluster-pool
	  labels:
	    env: dev
	spec:
	  clusterPoolName: red-cluster-pool-1
	---
	apiVersion: hive.openshift.io/v1
	kind: ClusterClaim
	metadata:
	  name: red-cluster-2
	  namespace: red-cluster-pool
	  labels:
	    env: dev
	spec:
	  clusterPoolName: red-cluster-pool-2
	---
	apiVersion: hive.openshift.io/v1
	kind: ClusterClaim
	metadata:
	  name: red-cluster-3
	  namespace: red-cluster-pool
	  labels:
	    env: dev
	spec:
	  clusterPoolName: red-cluster-pool-3

At this juncture we should have three clusters operational each with 3 control plane nodes and 3 worker nodes assuming default settings were used for the install config. Because these clusters will be running both application (frontend) and database (backend) workloads it is good practice to segregate these types of workloads. To do so we will use the MachinePool API to construct an additional set of worker nodes with appropriate labels and taints so that only Postgresql components are allowed. For brevity only the MachinePool configuraiton for GCP is shown (rinse and repeat for the other cloud providers and substitute instance types accordingly).

	apiVersion: hive.openshift.io/v1
	kind: MachinePool
	metadata:
	  name: '{{ (lookup "hive.openshift.io/v1" "ClusterClaim" "red-cluster-pool" "red-cluster-2").spec.namespace }}-backend-worker'
	  namespace: '{{ (lookup "hive.openshift.io/v1" "ClusterClaim" "red-cluster-pool" "red-cluster-2").spec.namespace }}'
	spec:
	  clusterDeploymentRef:
	    name: '{{ (lookup "hive.openshift.io/v1" "ClusterClaim" "red-cluster-pool" "red-cluster-2").spec.namespace }}'
	  name: backend-worker
	  labels:
	    node-role.kubernetes.io/backend: ""
	  taints:
	  - effect: NoSchedule
	    key: postgresql
	  - effect: NoSchedule
	    key: pgpool
	  platform:
	    gcp:
	      osDisk: {}
	      type: n1-standard-4
	  replicas: 3

This might look complex because it is using templating functions to resolve the identity of dynamically generated cluster names when using ClusterClaims but is actually no different than how PersistentVolumeClaims generate dyanmic names for Perstistent Volumes which most users are familiar with. The templates are intended to be processed by the [policyGenerator plugin](https://github.com/stolostron/policy-generator-plugin) as part of an automated policy-driven workflow operated by the SRE team. For more details abou the policyGenerator plugin please refer to this [blog](https://cloud.redhat.com/blog/generating-governance-policies-using-kustomize-and-gitops).

	$ oc get machinepools -A
	NAMESPACE                      NAME                                          POOLNAME         CLUSTERDEPLOYMENT              REPLICAS
	red-cluster-pool-aws-lt87l     red-cluster-pool-aws-lt87l-backend-worker     backend-worker   red-cluster-pool-aws-lt87l     3
	red-cluster-pool-aws-lt87l     red-cluster-pool-aws-lt87l-worker             worker           red-cluster-pool-aws-lt87l     3
	red-cluster-pool-azure-g76vj   red-cluster-pool-azure-g76vj-backend-worker   backend-worker   red-cluster-pool-azure-g76vj   3
	red-cluster-pool-azure-g76vj   red-cluster-pool-azure-g76vj-worker           worker           red-cluster-pool-azure-g76vj   3
	red-cluster-pool-gcp-x5mmj     red-cluster-pool-gcp-x5mmj-backend-worker     backend-worker   red-cluster-pool-gcp-x5mmj     3
	red-cluster-pool-gcp-x5mmj     red-cluster-pool-gcp-x5mmj-worker             worker           red-cluster-pool-gcp-x5mmj     3

From the RHACM UI it is also possible to verify that the machines have been incorporated successfully into the cluster from the Infrastructure/Cluster/Nodes view. 

## Network Connectivity

Next we will enable layer-3 network connectivity, using Submariner which is part of RHACM, between all of the clusters. This is only possible if all clusters are bound to the same managed cluster set. Layer-3 connectivity enables tunneling of both TCP and UDP protocols which is important for technologies such as Postgres and it's middleware proxy, PgPool-II, which use both to establish heartbeat signalling for quorum. Installing Submariner is outside the scope of this article though (please refer to the [documentation](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.6/html/add-ons/add-ons-overview#submariner). Suffice it is to note that each cluster participating as a member should be configured with a non-overlapping cluster and service CIDRs. This can be defined inside the ClusterPool install-config secret. For example the install-config referenced by the ClusterPool for GCP looks as follows:

	apiVersion: v1
	metadata:
	  name: 'gcp-install-config'
	baseDomain: # your domain
	controlPlane:
	  architecture: amd64
	  hyperthreading: Enabled
	  name: master
	  replicas: 3
	  platform:
	    gcp:
	      type: n1-standard-4
	compute:
	- hyperthreading: Enabled
	  architecture: amd64
	  name: 'worker'
	  replicas: 3 
	  platform:
	    gcp:
	      type: n1-standard-4
	networking:
	  networkType: OVNKubernetes
	  clusterNetwork:
	  - cidr: 10.132.0.0/14
	    hostPrefix: 23
	  machineNetwork:
	  - cidr: 10.0.0.0/16
	  serviceNetwork:
	  - 172.31.0.0/16
	platform:
	  gcp:
	    projectID: # your project 
	    region: asia-southeast1
	pullSecret: ""
	sshKey: |-
	    ssh-rsa # your public key

The addresses for the clusterNetwork and serviceNetwork would be bumped for each ClusterPool so as to ensure non-overlapping CIDRS for connectivity. Check the Submariner add-on UI within RHACM to ensure that all is well with inter-cluster networkconnectivity before proceeding with the next steps.

As a final note it is possible to setup Submariner using policy-driven templating similar to what is shown above with ClusterClaims as part of your provisioning policies set. This would enable automated workflows.

## Postgres Setup 

With the clusters established and inter-connected using secure VXLAN tunnels established by Submariner, the final step is to setup Postgres in a highly-available configuration. For quorum to function properly it is required to have a minimum of three independent failure domains which in this case will be achievable at both a zonal and cloud level, as we don't wish to unecessarily failover to another cloud provider unless all zones are impaired due to a cascading fault. Thus the end result will be that there are a total of nine copies of Postgres running with one being the master servicing reads and writes and configured with streaming replication to the other eight standby replicas services reads and potential failover targets. A detailed introspective of this type of architecture can be found in the [PgPool-II documentation](https://www.pgpool.net/docs/43/en/html/example-cluster.html). PgPool-II is middleware for Posttgres that enables transparent failover from an application perspective amongst other functions such as connection pooling and will need to be deployed to make high-available work in the manner described.

The process of installing Postgresql will not be covered here, as the intent is to describe how to configure Postgresql so that instances can communicate across the cross-cluster network and failover to each other. For simplicity only one instance per cluster per cloud provider will be deployed. This can be extended to three instances per cluster per cloud provider for a production setup.

## Conclusion

In this article we have shown how to leverage cluster landing zones in a hybrid cloud architecture to achieve higher levels of availability and mainteance windows that encompass cluster restarts without any noticeable impact on service performance than what was previously considered possible when running workloads on a single cloud provider with multiple available zones configured.
