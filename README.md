# Cluster Landing Zone for Hybrid and Multi-cloud Architecture

Cluster landing zones provide a framework in which common organizational challenges with respect to network segmentation, failure domains and team boundaries when designing cluster deployment topologies for hybrid and multi-cloud architectures can be addressed. This is augmented with an operating model that provides tools and guidance on how to deal with day-to-day operational issues related to cluster lifecycling, application rollout in a multi-tenanted environment, governance, observability and disaster recovery. This article deals with the former as best practices pertaining to the latter is covered by multiple other articles on this blog site covering Red Hat Advanced Cluster Management for Kubernetes (RHACM).

Let's dig a little deeper into what common design challenges enterprises face when migrating to cloud and deploying clusters at scale. Other than the obvious application-level concerns, the next layer down deals with the more challenging aspects of organisational context that manifest themselves as design demarcation points or boundaries, and inform the entprise architect how the cluster landing zone should be translated to the needs of the organization.

- network (internal-facing vs external-facing)
- environment (production vs non-production)
- location (on-premises vs cloud)
- team (often product aligned)
- domain (business support systems vs product platforms)

Call these trust boundaries if you will as their purpose is to establish an administrative span of control in line with well-estalished architectural principles based on separation of concerns and least privilege amongst others. These boundaries and architectural principles on which they are based need to be reflected in the design of a cluster landing zone and are contributing factors to why a hybrid cloud architecture is distinct from a multi-cloud architecture. 

Modern computing architectures based around hybrid cloud adoption (defined here as a logical coalescing of cloud provider infrastructure into a single addressable unit) and multi-cloud (defined here as a disparate set of cloud provider infrastructure) are often abstracted using self-managing infrastructure orchestration platforms such as OpenShift with a view to enabling container-oriented workloads to run in a resilient, secure and portably manner underpinned by self-service and automation, to a varying degree. 

What both hybrid and multi-cloud architectures have in common is the segregation of the control plane from the data plane. The control plane (hub in RHACM numenclamenture) centralises all of the cluster management lifecycle functions and higher-level governance controls. It also facilitates distribution of user workloads down to the data plane (managed clusters in RHACM numenclamenture). The distinction between the two models ends there and whereas hybrid cloud architecture coalesce clusters hosted on separate cloud providers into a logically addressable unit managed by single hub (as shown in the first diagram below), the multi-cloud model (shown in the second diagram) operates these as disjoint entities managed by separate hubs. This may be an architectural nicety, but the impact of this is that only with the hybrid cloud model is it possible to run multi-cluster services using technologies such as Submariner which require clusters to be defined within a managed cluster set.

From an operational model the diagrams show that cluster provisioning activities including hardening and security are performed by IT Operations / SRE team across the fleet as would be expected with such a centralised model. In addition to that (but not mandatory) is the possibility of leveraging the hub for multi-tenanted application delivery.

For each application team a set of managed clusters is provisioned by the SRE team and bound to a team-specific namespace that hosts the GitOps engine (ArgoCD). The namespace is bound (via RHACM) to the managed cluster set that encapsulates the managed clusters that were provisioned. This ensures that even if the application team mis-configures their workflow and accidentally target another teams clusters that the effect is nil as only operations against the bound managed cluster set are allowed to go through. Furthermore the applicaiton team cannot run any containers inside this namespace, via Kubernetes RBAC roles assigned to the team they would only have access to placement and application set resources (more details below). If an application team wishes to further sub-divide responsibilities between members of the team, say the one team is allowed to deploy to development clusters but not production clusters then ArgoCD offers RBAC within the tool itself to achieve this finer level of segregation.




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
