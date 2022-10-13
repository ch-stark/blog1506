# Cluster Landing Zone for Hybrid and Multi-cloud Architecture

Cluster landing zones provide a framework in which common organizational challenges with respect to network segmentation, failure domains and team boundaries when designing cluster deployment topologies for hybrid and multi-cloud architectures can be addressed. This is augmented with an operating model that provides tools and guidance on how to deal with day-to-day operational issues related to cluster lifecycling, application rollout in a multi-tenanted environment, governance, observability and disaster recovery. This article deals with the former as best practices pertaining to the latter is covered by multiple other articles on this blog site covering Red Hat Advanced Cluster Management for Kubernetes (RHACM).

Let's dig a little deeper into what common design challenges enterprises face when migrating to cloud and deploying clusters at scale. Other than the obvious application-level concerns, the next layer down deals with the more challenging aspects of organisational context that manifest themselves as design demarcation points or boundaries, and inform the entprise architect how the cluster landing zone should be translated to the needs of the organization.

- network (internal-facing vs external-facing)
- environment (production vs non-production)
- location (on-premises vs cloud)
- team (often business unit or system aligned)
- domain (business support systems vs product platforms)

Call these trust boundaries if you will as their purpose is to establish an administrative span of control in line with well-estalished architectural principles based on separation of concerns and least privilege amongst others. These boundaries and architectural principles on which they are based need to be reflected in the design of a cluster landing zone and are contributing factors to why a hybrid cloud architecture is distinct from a multi-cloud architecture. 

Modern architectures based around hybrid cloud (in which basic infrastructure resources (compute, storage, network) from one or more cloud providers are logically coalesced into a single addressable unit) and multi-cloud (in which infrastructure resources of one or more cloud providers remain logically separated) are often abstracted using self-managing infrastructure orchestration platforms such as OpenShift with a view towards enabling container-oriented workloads to be run resiliently, securely and portably across multiple clouds.  

What both hybrid and multi-cloud architectures have in common is the segregation of the control plane from the data plane. The control plane (hub in RHACM) centralises all cluster management functions and governance, and enables distribution of workloads down to the data plane (managed clusters in RHACM). The distinction lies primarily in the data plane and whether clusters spread across multiple cloud providers (including on-premises) are coalesced into logical unit (hybrid model) or kept logically disparate (multi-cloud model). Coalescing clusters together enables cross-cluster communications using multi-cluster services APIs implemented by technologies such as Submariner on OpenShift, making these services even more highly-available. 

The cluster landing zone model for a hybrid cloud architecture is shown in the following diagram.

Cluster lifecycle operations are handled by IT Operations / SRE team who is given the cluster-admin role. All operations targeting either the hub or managed clusters are defined using ACM policies and versioned in git. A collection of sample policies can be found here:

The hub also supports multi-tenancy for application teams to rollout their applications to a subset of clusters encapsulated inside of a managed cluster set as defined by the SRE team to which the GitOps engine (ArgoCD) is bound. There is one instance per team and it is possible within the instance to further sub-divide responsibilities (e.g., production vs non-production environments) to specific team members using the ArgoCD project API. In order to limit what the application team can do on the hub they are given access to a very limited set of APIs (placements and applicationsets) with no ability to run pods on the hub itself or view the contents of other namespaces. Thus the hub services as an application launcher and obviates the need to deploy an ArgoCD instance on each managed cluster.

From an application team perspective they do not need to know the details of their cluster, such as cluster name or which managed cluster set the cluster resides in. From a usage perspective it sufficies to correctly label their clusters when they request for these to be provisioned and the SRE simply needs to provide them with a bill-of-materials that resembles what is shown inside the boxes in the diagram. Based on this the applicaiton team can define their own placements and integrate these into their applicationset defintions as shown in the following two examples:

This in turn would be referred to in an ArgoCD ApplicationSet for deployment like this:   

If the application should only be deployed to clusters on AWS the placement could be refined further like this:

At no time was the name of the cluster itself referrenced thus adhering to the "cattle-not-pets" paradigm of cloud-native computing.

As an side, some advantages of this solution is that it makes a "cluster-per-team" approach a feasible proposition which has inherenet benefits such as reduced blast radius, decoupled maintenance windows, ad hoc restart and bespoke scaling and upgrades. The downsides is the increased cost which can be mitigated via solutions such as HyperShift.

For a multi-cloud topology From an application team perspective the multi-cloud multi-tenancy model stays the same, what changes is that clusters managed by distinct hubs are not able to be logically coalesced together into a cross-cluster communications domain enabled by Submariner which operate at OSI layer 3. In order to overcome this limitation, OSI layer 7 based solutions such as federated service meshes can be considerd as an option. From an SRE perspective what changes is that a top-level hub (hub-of-hubs architecture) may be required to ensure consistency and compliance across multiple clouds as shown in the following diagram.  


The multi-cloud model does have the flexibility to support more complex organizational structures such as when dealing with multiple legal entities or divisions, where decentralised IT ownership is more common. It also deals with with very assymetric cloud architectures in which each hub employs a bespoke operating model but there is still a need for some form of over-arching governance for compliance and reporting required.

Conclusion

Cluster landing zones are useful to model and implement cluster topologies based on well-defined boundaries within the enterprise and to enable multi-tenancy for application teams using tools such as ArgoCD that integrate with APIs surfaced by ACM that expose cluster grouping and addressability. Using multi-cluster management tools such as RHACM to implement and operate a cluster landing zone makes for a more pleasnat hybrid/multi-cloud adoption journey compared to not having these in place or a manual home-grown approach.
