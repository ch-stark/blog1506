# Guide to Cluster Landing Zones for Hybrid and Multi-cloud Architectures (Part 2)

## Overview

In the previous [blog](https://cloud.redhat.com/blog/a-guide-to-cluster-landing-zones-for-hybrid-and-multi-cloud-architectures) the concept of establishing a cluster landing zone to enable hybrid and multi-cloud architectures was explained. In this blog we are going to look at how to realize the cluster landing zone for hybrid cloud to leverage the purported benefits of higher availabilty by distributing a backend database (Postgres) across multiple clusters hosted on different cloud providers thus mitigating the infrequent but yet still ever prevelant risk of one cloud provider experiencing a bad hair day. Other than risk mitigation, this also enables non-impacting operational maintenance windows as well as cluster reboots and rebuilds.

## Cluster Landing Zone

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

At this juncture we should have three clusters operational each with 3 control plane nodes and 3 worker nodes assuming default settings were used for the install config. Because these clusters will be running both application (frontend) and database (backend) workloads it is good practice to segregate these types of workloads. To do so we will use the MachinePool API to construct an additional set of worker nodes with appropriate labels and taints so that only backend workloads that match the taint can be hosted. For brevity only the MachinePool configuraiton for GCP is shown (just rinse and repeat for the other cloud provider substituting machine types accordingly).

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
	    key: postgres
	  platform:
	    gcp:
	      osDisk: {}
	      type: n1-standard-4
	  replicas: 3

This looks complex because it is using templating functions to resolve the identity of the dynamically generated name of the cluster that was provisioned by Hive in response to a ClusterClaim. The templates are intended to be processed by the [policyGenerator plugin](https://github.com/stolostron/policy-generator-plugin) as part of an automated policy-driven workflow operated by the SRE team. Alternatively, a manual lookup and substitution can be done. The net result is a cluster with two machine pools per cloud provider.

$ oc get machinepools -A
NAMESPACE                      NAME                                          POOLNAME         CLUSTERDEPLOYMENT              REPLICAS
red-cluster-pool-aws-lt87l     red-cluster-pool-aws-lt87l-backend-worker     backend-worker   red-cluster-pool-aws-lt87l     3
red-cluster-pool-aws-lt87l     red-cluster-pool-aws-lt87l-worker             worker           red-cluster-pool-aws-lt87l     3
red-cluster-pool-azure-g76vj   red-cluster-pool-azure-g76vj-backend-worker   backend-worker   red-cluster-pool-azure-g76vj   3
red-cluster-pool-azure-g76vj   red-cluster-pool-azure-g76vj-worker           worker           red-cluster-pool-azure-g76vj   3
red-cluster-pool-gcp-x5mmj     red-cluster-pool-gcp-x5mmj-backend-worker     backend-worker   red-cluster-pool-gcp-x5mmj     3
red-cluster-pool-gcp-x5mmj     red-cluster-pool-gcp-x5mmj-worker             worker           red-cluster-pool-gcp-x5mmj     3

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
	  - cidr: 10.136.0.0/14
	    hostPrefix: 23
	  machineNetwork:
	  - cidr: 10.0.0.0/16
	  serviceNetwork:
	  - 172.32.0.0/16
	platform:
	  gcp:
	    projectID: # your project 
	    region: asia-southeast1
	pullSecret: ""
	sshKey: |-
	    ssh-rsa # your public key

The addresses for the clusterNetwork and serviceNetwork would be bumped for each ClusterPool so as to ensure non-overlapping CIDRS for connectivity. Check the Submariner add-on UI within RHACM to ensure that all is well with inter-cluster networkconnectivity before proceeding with the next steps.

## Postgres Setup 





 

 




Cluster landing zones provide a framework in which common organizational challenges with respect to network segmentation, failure domains, team ownership and data sovereignty when designing cluster deployment topologies for hybrid and multi-cloud architectures can be addressed. This is augmented with an operating model that provides tools and practical guidance on how to deal with day-to-day operational issues related to cluster lifecycling, application rollout, governance, observability and disaster recovery. This article treats primarily with the former set of concerns and touches on best practices for the latter as and when needed. There are also many other excellent blog articles that provide more detailed guidance and should be consulted. Whilst the nature of this blog is architectural it is underpinned with a specific set of tooling in mind that provides multi-cluster management and GitOps capabilities, namely Red Hat Advanced Cluster Management for Kubernetes (RHACM) and OpenShift GitOps (ArgoCD).

## Designing for Hybrid and Multi-cloud Architectures

Let's dig a little deeper into what common design challenges enterprises face when migrating to cloud and deploying clusters at scale. Other than the obvious application-level concerns, the next layer down deals with more challenging aspects of organisational context that manifest themselves as design demarcation points or trust boundaries. These typically  inform the entprise architect of how the cluster landing zone should be segmented and shaped to the needs of the organization. A list of common boundaries include any of the following:

- network zone
- operational environment mode
- infrastructure provider
- geography
- business unit
- application team

From an operational  perspective the purpose of trust boundaries are to establish an administrative span of control within the enterpise and are often based on well-estalished enterprise architecture principles such as separation of concerns and least privilege amongst others such as data sovereignty. These boundaries and principles need to be reflected in the design of a cluster landing zone and are major contributing factor to how many hubs and clusters are present in the final design.

Modern computing architectures based around hybrid cloud (defined here as a logical coalescing of infrastructure from one or more cloud providers) and multi-cloud (defined here as disparate infrastructure from one or more cloud providers) are within the context of OpenShift abstracted into cloud-agnostic platforms for the purpose of running container workloads reliably and seamlessly.

What both hybrid and multi-cloud architectures have in common is the segregation of the control plane from the data plane. The control plane (referred to as the hub in RHACM) centralises all of the cluster lifecycle management functions and governance controls. It also facilitates distribution of user workloads down to the data plane (referred to as managed clusters in RHACM). The distinction between the two models ends there as hybrid cloud architecture coalesce managed clusters hosted on multiple cloud providers into a logical unit (referred to as a managed clusterset in RHACM) and manages these with a single hub that has visibility across all cloud providers. Managed cluster sets are also present in a multi-cloud architecture but only contain managed clusters hosted on one cloud provider as typically the boundary of the hub is the cloud provider itself, as per the diagrams below. Note that in a multi-cloud architecture there might be a top-level hub (hub-of-hubs) to provide some level of cross-cloud consistency and governance, but this is optional.  The multi-cloud model does have more flexibility in terms of support for more complex organizational structures such as when dealing with multiple legal entities or mergers, or in shops where decentralised IT ownership is still practiced. On the other hand the hybrid cloud model better supports new Kubernetes features such as the multi-cluster services API that enables load-balancing of services across multiple clusters in a managed cluster set and thus across cloud providers, providing evef higher levels of resiliency.

<p align="center">
  <img src="https://github.com/jwilms1971/blog/blob/main/acm/RHACM%20Operating%20Model%20-%20Hybrid-cloud.png">
  <em>Diagram 1. Cluster landing zone for a hybrid cloud architecture</em>
</p>

<p align="center">
  <img src="https://github.com/jwilms1971/blog/blob/main/acm/RHACM%20Operating%20Model%20-%20Multi-cloud.png">
  <em>Diagram 2. Cluster landing zone for a multi-cloud architecture</em>
</p>

From an operational perspective the diagrams show that all provisioning activities including cluster hardening and configuration are performed by the IT Operations/SRE team via a policy-driven/GitOps approach from the hub itself. This is described in greater detail in other blogs on this site. In addition to that (but not mandatory) is the possibility of leveraging the hub for multi-tenanted application delivery.

For each application team a set of managed clusters is provisioned by the SRE which are bound to a team-specific namespace that hosts the GitOps engine (ArgoCD). The namespace is bound to the managed cluster set thus limiting the scope of what is visible to ArogCD. This ensures that even if the application team mis-configures their workflow and accidentally target clusters belonging to another team, the net effect of this would be nil as only operations against the bound managed cluster set are possible via this scope control. As an aside, if the application team wishes to further segregate the responsibilities for deploying applications to specific clusters between team members this can be achieved using ArgoCD projects.

Placements and application sets are the main APIs that the application will need to understand to manage their deployment workflow. An example is shown below. Let's say that the red team wish to deploy their application to production only. Here are the resources they would need to define to achieve this.

	apiVersion: cluster.open-cluster-management.io/v1beta1
	kind: Placement
	metadata:
	  name: placement-red-clusters-prod
	  namespace: red-gitops
	spec:
	  predicates:
	    - requiredClusterSelector:
	      labelSelector:
	        matchExpressions:
	          - {key: env, operator: In, values: ["prod"]}
	---
	apiVersion: argoproj.io/v1alpha1
	kind: ApplicationSet
	metadata:
	  name: my-app
	  namespace: red-gitops
	spec:
	  generators:
	    - clusterDecisionResource:
	      configMapRef: acm-placement
	      labelSelector:
	        matchLabels:
	          cluster.open-cluster-management.io/placement: placement-red-clusters-prod
	      requeueAfterSeconds: 180
	  template:
	    metadata:
	      name: 'my-app-{{name}}'
	    spec:
	      project: default
	      source:
	        repoURL: # valid URL
	        targetRevision: main
	        path: # valid path
	      destination:
	        namespace: my-app
	        server: '{{server}}'
	      syncPolicy:
	        syncOptions:
	          - CreateNamespace=true
	          - PruneLast=true
	      automated:
	        prune: true
	        selfHeal: true

And that is everything. Behind the scenes controllers work to evaluate the placement into a valid list of clusters based on the managed cluster set that is bound to the ArgoCD instance. This list will then be handed over to ArgoCD to generate one application per cluster. Based on this code and assuming it is applied to the hybrid cloud architecture above the net result is two applications deployed: one to the production cluster in AWS and the other to the production cluster in GCP.

A more complex placement example leveraging both labels and cluster claims that select the red team's non-production cluster in AWS only is as follows:

	apiVersion: cluster.open-cluster-management.io/v1beta1
	kind: Placement
	metadata:
	  name: placement-red-clusters-nonprod
	  namespace: red-gitops
	spec:
	  predicates:
	  - requiredClusterSelector:
	      labelSelector:
	        matchExpressions:
	          - {key: env, operator: NotIn, values: ["prod"]}
	      claimSelector:
	        matchExpressions:
	          - {key: platform.open-cluster-management.io, operator: In, values: ["AWS"]}

The net result of this (assuming the same applicaiton set was updated the new placement name) would be that there are two  applications deployed, one to the staging cluster and the other to the development cluster, both in AWS only.

Unlike labels which are typically user-defined and should be specified at provisioning time (they can be changed later), cluster claims are auto-generated by RHACM and provide a rich set of semantics that can be used to distinguish clusters and cloud providers. The upshot is that at no time does the application team need to specify the cluster name which is important when using a dynamic cluster provisioning strategy which was described in this prior [blog](https://cloud.redhat.com/blog/securing-ingress-controllers-on-a-managed-openshift-cluster-using-red-hat-advanced-cluster-management).

As an side, some advantages of this architecture is that it facilitates a "cluster-per-team" approach which has some inherenet benefits such as reduced blast radius, decoupled maintenance windows, ad hoc restart along with bespoke scaling and cluster version/configuration. The downsides include increased infrastructure cost which can be mitigated via solutions such as HyperShift. The centralised control plane model in which all administrative and application activity is initiated from the hub has been long established in IT and removes the need to integrate managed clusters with identity systems and track user activity. RHACM search capabilities also support this hands-off model as all objects can be searched for from the hub UI.

## Conclusion

Cluster landing zones are a useful model for understanding and implementing hybrid and multi-cloud topologies based on well-defined trust boundaries and architectural principles. With a carefully considered multi-tenancy model and using integrated tooling such as ArgoCD the hub can also serve the needs of application teams. Using the various multi-cluster lifecycle, governance, and observability tools shipped with RHACM enables the SRE to fully implement and operate a cluster landing zone tailored to the needs of adaptive enterprise that is able to better evolve as new technologies and practices emerge.
