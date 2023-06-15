# Guide to Cluster Landing Zone for Hybrid and Multi-cloud Architecture (Part 3)

## Overview

Release 2.8 of Red Hat Advanced Cluster Management (RHACM) introduces support for multi-hub federation. This new capability can be leveraged to enable new multi-tenancy patterns based on previous patterns introduced in parts <a href="https://cloud.redhat.com/blog/a-guide-to-cluster-landing-zones-for-hybrid-and-multi-cloud-architectures" rel="nofollow">1</a> and <a href="https://cloud.redhat.com/blog/guide-to-cluster-landing-zones-for-hybrid-and-multi-cloud-architectures-part-2" rel="nofollow">2</a>.

## Addressing complex multi-tenancy requirements

Before Global Hub the only way to segment a managed cluster fleet was from a single hub and the lead architect had to be make a choice at design time, as to whether the operator's perspective (typically linked to execution environment) or a developer's perspective was the primary driver for segmentation. It was difficult to accomodate the needs of both parties due to the fact that a single cluster can only be bound to a single managed cluster set at a given point in time.

With the introduction of multi-hub federation it is now possible to segment the cluster fleet at the operational layer first, and susbsequently at the team layer, as shown in the diagram below. This is a much more natural pattern that reflects how many customers operate their IT systems, particular in regulated enterprise which enforce the Separation of Concerns principle. The global hub establishes a federated view (single pane-of-glass) across the managed cluster fleet for governance purposes, whilst allowing administrative delegation of a subset of clusters to operational teams based on environment boundaries.

<p align="center">
  <img src="https://github.com/jwilms1971/blog/blob/main/acm/Cluster%20Landing%20Zone%20-%20Pattern%203c.png">
</p>

Note that as in previous patterns, it is possible to have managed clusters sets that are either dedicated or shared between application development teams. Either way, appropriate guardrails should be put in place by the global hub and delegated cluster administrators to prevent any undesirable interaction patterns. Tools such as Gatekeeper (included with RHACM), Kyverno and others can help in this space, along with appropriate RHACM Policies to harden the cluster security posture which can be found <a href="https://github.com/open-cluster-management-io/policy-collection/tree/main/policygenerator/policy-sets/community/" rel="nofollow">here</a>.


## Conclusion

Multi-hub federation offers a new way of segmenting the fleet of managed clusters to support segmentation of the cluster fleet along multiple boundaries. The right choice of multi-tenancy pattern will depend very much on a customer's operating model and desired level of complexity. In this series of blogs we have demonstrated several multi-tenancy patterns and how RHACM can be used to implementt these. 
