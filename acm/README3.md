# Guide to Cluster Landing Zone for Hybrid and Multi-cloud Architecture (Part 3)

## Overview

Release 2.8 of Red Hat Advanced Cluster Management (RHACM) introduces support for multi-hub federation, which is referred to as Global Hub in the documentation. This new capability can be leveraged to extend the multi-tenancy patterns introduced in parts <a href="https://cloud.redhat.com/blog/a-guide-to-cluster-landing-zones-for-hybrid-and-multi-cloud-architectures" rel="nofollow">1</a> and <a href="https://cloud.redhat.com/blog/guide-to-cluster-landing-zones-for-hybrid-and-multi-cloud-architectures-part-2" rel="nofollow">2</a>.

Note that currently multi-hub federation is limited to Governance, which is sufficient for the purposes of establishing multi-tenancy across the managed cluster fleet as RHACM Policies are the primary technical enabler for implementing this. 

## Addressing complex multi-tenancy requirements

Before Global Hub the only way to segment a managed cluster fleet was from a single hub and a choice had to be made at design time, as to whether the operator's perspective (typically linked to environment) or a developer's perspective was the primary driver for segmentation. It was difficult to accomodate the needs of both parties due to the fact that a single cluster can only be bound to a single managed cluster set at a given point in time.

With the introduction of Global Hub it is now possible to segment the cluster fleet at the operational layer first, and susbsequently at the team layer, as shown in the diagram below. This is much more natural pattern that reflects how many customers in regulated enterprise operate today. The Global Hub establishes a federated view (single pane-of-glass) across the managed cluster fleet, whilst allowing delegation of a set of clusters to operational teams based on the principle of Separation of Concerns.

<p align="center">
  <img src="https://github.com/jwilms1971/blog/blob/main/acm/Cluster%20Landing%20Zone%20-%20Pattern%203b.png">
</p>

Note that as in previous patterns, it is possible to have managed clusters sets that are either dedicated or shared between teams. Either way, appropriate guardrails should be put in place to prevent any undesirable interaction patterns. Tools such as Gatekeeper (included with RHACM), Kyverno and others can help with this.

The hub administrator is responsible for defining the RHACM Policies that establish not only the global hub but alse each of the federated hub members. The hub administrator then delegates control of each hub member to a separate operations team responsible for the environment-based clusters that the hub manages. The local hub administrators in turn define the right mix of managed cluster sets based on user requirements, and bind these to dedicated ArgoCD instances that teams can use for deploying their applications from the hub. Application development teams have highly restrictive access to the hub (typically to a single namespace and specific APIs only). Application development teams have also access to the RHACM UI so that they can leverage Search capabilities to find resources deployed. In the event of breakglass localised accounts on the managed clusters can be temporarily allowed to support launching of ephemeral containers for debugging purposes.


## Conclusion

Global Hub offers a new way of segmenting the fleet of managed clusters using a federated multi-hub approach. The right choice of multi-tenancy pattern will depend very much on a customer's operating model and regulatory concerns that need to be fulfilled. In this series of blogs we have demonstrated several patterns to highlight the flexibility of RHACM in achieving this.
