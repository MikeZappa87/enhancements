# KEP-4410: Kubernetes Networking reImagined

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories (Optional)](#user-stories-optional)
    - [Story 1](#story-1)
    - [Story 2](#story-2)
<!-- /toc -->

## Summary

This proposal is to design and implement the KNI [Kubernetes Networking Interface] or better known as Kubernetes Networking reImagined. KNI is a foundational network API that is specific to Kubernetes. KNI will provide the users the ability to make Kubernetes networking completely pluggable. 

## Motivation

Kubernetes networking has traditionally been challenging to understand for users
interacting with the Kubernetes API, and there has been considerable flexibility
in how Container Network Interfaces (CNIs) set up networking within clusters.
This has resulted in a scenario where things like pod networking (including pod
to pod networking) is opaque to users, with different implementations taking
markedly different approaches. This fragmentation and issues with the API have
negatively impacted adoption in sectors such as telecommunications. Our goal is
to transform Kubernetes networking by making networks and their components
actual resources within the Kubernetes API. This will allow for the development
of shared functionalities and their integration into the API. We anticipate that
this new approach will enhance support for areas that are currently struggling,
facilitate the development and promotion of common features, and better define
and accommodate advanced functionalities and potential areas for expansion.

### Goals

1. Design a cool looking t-shirt
2. Make it possible to specify networks and network configurations via the Kubernetes API, to some reasonable degree
3. Provide highly focused networking APIs that are composable
3. Support multi-network attachment for `Pods`
4. Support `Pod` to `Pod` network semantics with Kubernetes API specification
5. Enable egress (Telcos/others need to strongly define/control traffic routes)
6. Provide APIs that are flexible, encompassing both "core" and "extended" features
7. Provide APIs with extension points for implementation-specific behaviors
8. Provide integration with on premise or cloud systems to provide network status
9. Provide a reference implementation
10. Establish feature parity with current [ADD, DEL]
11. Decouple Node and Pod network setup
12. Ensure that the network runtime is consolidated inside of a Pod

> **Note**: The concepts of "core" and "extended" features is inspired by the
> Gateway API project. "Core" features are something every implementation MUST
> support. "Extended" features are things that implementations MAY support (but
> at least 2 known implementations actively need).

### Non-Goals

1. Any changes to the kube-scheduler 
2. Any specific implementation other than the reference implementation. However we should ensure the KNI-API is flexible enough to support

## Proposal

> **Note**: You'll notice that much of the proposal is currently missing. It is
> our intention to fill it (and all the other sections) in as part of later
> iterations, but for these early iterations we want to make sure we're laser
> focused on building consensus among the community on the "What" and "Why",
> without yet focusing on the "How". So for now we're just focused on user
> stories to support the "Why", and later PRs will propose the actual
> implementation to accomplish it.

### User Stories (Optional)

#### Story 1

#### Story 2
