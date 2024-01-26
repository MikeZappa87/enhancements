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
2. Design and implement the KNI-API
3. Provide documentation, examples, troubleshooting and FAQ's for KNI.
   * we should provide a example network runtime
4. Provide an API that is flexible for experimentation and opinionated use cases
   * example extradata map[string] string
5. Provide integration with on premise or cloud systems to provide network status
6. Provide an API that provides networks available on the node
7. Determine the reference implementation
8. Establish feature parity with current [ADD, DEL]
9. Decouple Node and Pod network setup
10. Ensure that the network runtime is consolidated inside of a Pod

### Non-Goals

1. Any changes to the kube-scheduler 
2. Any specific implementation other than the reference implementation. However we should ensure the KNI-API is flexible enough to support

## Proposal


The proposal of this KEP is to design and implement the KNI-API and make necessary changes to the CRI-API and container runtimes. The scope should be kept to a minimum and we should target feature parity. 

### User Stories (Optional)

#### Story 1

#### Story 2
