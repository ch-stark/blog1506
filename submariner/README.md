# Guide to Cluster Landing Zones for Hybrid and Multi-cloud Architectures (Part 2)

## Overview

In this installment we look at extending the cluster landing zone for hybrid cloud that was introduced [previously](https://cloud.redhat.com/blog/a-guide-to-cluster-landing-zones-for-hybrid-and-multi-cloud-architectures) to support stateful application workloads. We also demonstrate how the architecture deals with disaster scenarios such as the catastrophic failure of a cloud provider. We make extensive useage of various capabilities within the Red Hat Advanced Cluster Management (RHACM) toolbox and show how these can work together to deliver a robust and compelling solution which enables a reference multi-tenancy operating model.

## Extending the Cluster Landing Zone

In order to run stateful workloads such as databases across a hybrid cloud envvironment we need to extend the cluster landing zone to support DBAs as first-class tenants in addition to application teams and cluster administrators (SREs). The revised cluster landing zone looks as per the following diagram.


This extension is required because the alternative approach of granting application teams access to Polices in order to configure databases (described in more detail later) is considered an anti-pattern given that the Policy controller runs with elevated privileges and thus could be used to undermine the safeguards put in place by the cluster administrators. Separating application concerns from database and cluster administrative concerns by team boundary is a common practice which the multi-tenancy operating model being proposed here supports. Note that in the diagram we show DBAs writing policies into their own namespace which are processed by the cluster-wide Policy controller.

The diagram introduces MachinePools which are defined on the hub and used for segregating fronted (application) workloads from backend (database) workloads for a managed cluster. This is done for multiple reasons including to avoid outages due to competition for resources caused by disparate workloads or misconfiguration errors, reduced impact of cluster upgrades on application/database availability when nodes reboot (assuming workloads are spread across all nodes in a machine pool and that machines are distributed across availability zones), the ability to select machine types that better match the profile of a given workload and support bespoke tuning.

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

To spawn a cluster we need to submit a ClusterClaim (this is similar in concept to how a PersistentVolumeClaim results in the creation of a PersistentVolume). The following cluster claims were submitted and reference ClusterPools mapped to cloud providers and specific regions. Convenience labels (e.g., env=dev) are defined for sorting clusters into subsets which was explained in the previous blog. Note that we do not need to add labels to identify the origin cloud provider as these are auto-generated for us as part of a pre-defined label set provided by RHACM.

        apiVersion: hive.openshift.io/v1
        kind: ClusterClaim
        metadata:
          name: red-cluster-1
          namespace: red-cluster-pool
          labels:
            env: dev
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
	
Note that we use the known static cluster claim name in order to derive the dynamically generated cluster name. This piece of YAML cannot be submitted directly to the API Server as it is using Go templating and  needs to first be processed by the Policy Generator tool into a valid Policy document that in turn can be applied to the system via the Policy controller. For more details on this tool please refer to [here](https://cloud.redhat.com/blog/generating-governance-policies-using-kustomize-and-gitops) and [here](https://github.com/stolostron/policy-generator-plugin/blob/main/docs/policygenerator.md).

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

Assuming that the preprequisites as documented [here](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.6/html/add-ons/add-ons-overview#preparing-selected-hosts-to-deploy-submariner) have been met then the next step is to enable the Submariner add-on in RHACM and configure the broker and gateway nodes for each cluster which in turn will establish VXLAN tunnels for passing IPsec traffic. An example of doing this for clusters hosted on AWS using Policy templates is as follows.

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

So far all of the steps shown are intended to be performed by the SRE team who administer the cluster fleet. The next set of steps are intended to be performed by the DBA.

***


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

Installation of Postgresql can be achieved using Operators or Helm charts and the details of this process are well-covered elsewhere. The post-installation configuraiton however is of particular importance and is where Policies can be used to drive conformance to an internal baseline designed for hybrid cloud deployment.



With the clusters established and inter-connected using secure VXLAN tunnels established by Submariner, the final step is to setup Postgres in a highly-available configuration. For quorum to function properly it is required to have a minimum of three independent failure domains which in this case will be achievable at both a zonal and cloud level, as we don't wish to unecessarily failover to another cloud provider unless all zones are impaired due to a cascading fault. Thus the end result will be that there are a total of nine copies of Postgres running with one being the master servicing reads and writes and configured with streaming replication to the other eight standby replicas services reads and potential failover targets. A detailed introspective of this type of architecture can be found in the [PgPool-II documentation](https://www.pgpool.net/docs/43/en/html/example-cluster.html). PgPool-II is middleware for Posttgres that enables transparent failover from an application perspective amongst other functions such as connection pooling and will need to be deployed to make high-available work in the manner described.

The process of installing Postgresql will not be covered here, as the intent is to describe how to configure Postgresql so that instances can communicate across the cross-cluster network and failover to each other. For simplicity only one instance per cluster per cloud provider will be deployed. This can be extended to three instances per cluster per cloud provider for a production setup.

## Conclusion

In this article we have shown how to leverage cluster landing zones in a hybrid cloud architecture to achieve higher levels of availability and mainteance windows that encompass cluster restarts without any noticeable impact on service performance than what was previously considered possible when running workloads on a single cloud provider with multiple available zones configured.
