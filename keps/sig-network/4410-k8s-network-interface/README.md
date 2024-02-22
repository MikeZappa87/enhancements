# KEP-4410: Kubernetes Networking reImagined

> **NOTE**: for the initial PR we've removed a lot of the templated text and
> aimed to keep this first iteration small and easier to consume. We are only
> focusing on the "What" and "Why" (e.g. motivation, goals, user stories) for
> this iteration so that we can build consensus on those first before we add
> any of the "How".

<!-- toc -->
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

KNI or Kubernetes Networking Interface, is an effort to take a second look at Kubernetes networking and evaluate what are the pain points, and what can we improve. At its core, KNI will be a foundational network API specific for Kubernetes that will provide flexibility and extensibility to solve basic and the most advanced and opinionated networking use cases.  
 
## Motivation

Kubernetes networking is an area of complexity and multiple layers which has created several challenges and areas of improvement. These challenges include deployment of the CNI plugins, troubleshooting networking issues and development of new functionality. 

Currently networking happens in three layers of the stack:

1. Kubernetes itself by means of kube-proxy or another controller based solution, to configure networking in the host node network namespace itself (typically workload-to-workload routing). 
2. The container runtime, which is responsible creating Linux network namespaces for pods.
3. And CNI plugins and the OCI runtime. The OCI runtime does additional network setup for network-namespaced pods by invoking binary CNI plugins that reside on the node filesystem. These plugins may configure either networking within the pod network namespace, or configure additional node-namespace-level networking, or both.

All of the communication and coordination required for this happens through non-network specific APIs, and in many cases non-Kubernetes APIs, which to the reader of the code makes it hard to determine where ‘networking’ is happening, and to the operator makes pod networking configuration hard to manage. Having "pod networking" concerns spread across several layers presents an issue when needing to troubleshoot issues, as operators need to check several areas, several configs, and several telemetry sources (kubectl logs, CNI execution logs, etc). This becomes more of an effort as multiple uncoordinated processes (CNI node agents, CNI plugins, etc) are making changes to the same resources - such as the network namespace of either the host node, or the pod, or the shared node-level CNI config (e.g. /etc/cni/net.d). The KNI aims at reducing the complexity by consolidating the networking into a single layer and having a uniform process for both namespaced and kernel isolated pods through a gRPC API. Leveraging gRPC will allow users the ability to migrate away from the current execution model that the CNI currently leverages.   

The next challenge is the deployment of the CNI plugins that provide the network setup and teardown of the pod. Virtually all CNI providers use plugins, and operators can choose to chain multiple plugins, which provides useful lifecycle hooks for customizing pod networking. The idiomatic way to deploy a workload in Kubernetes is that everything should be in a pod; however the CNI reliance on a shared node-global config file and deployed "plugin" binaries on the host node file system in practice results in the need to run privileged daemonset "plugin installers" that copy files to the host node, and do filesystem watches on binaries and configs. This typically makes collision detection, self-repair, and uninstall among uncoordinating "installer" daemonsets complex and error-prone. Since all existing K8s network plugins are running as daemonsets we will take this approach as well, where all the dependencies are packaged into the container image thus adopting a well known approach.

Another area that KNI will improve is ‘network readiness’. Currently the container runtime is involved with providing both network and runtime readiness with the Status CRI RPC. The container runtime defines the network as ‘ready’ by the presence of the CNI network configuration in the host file system. The more recent CNI specification does include a status verb, however this is still bound by the current limitations, files on disk and execution model. The KNI will provide an RPC that can be implemented so the kubelet will call the KNI via gRPC. 

KNI aims to help the community and other proposals in the Kubernetes ecosystem. We will do this by providing necessary information via the gRPC service. We should be the API that provides the “what networks are available on this node” so that another effort can make the kube-scheduler aware of networks. We should also provide IPAM status as a common issue, is that the IPAM runs out of assignable IP addresses and pods are no longer able to be scheduled on that node until intervention. We should provide visibility into this so that we can indicate “no more pods” as setting the node to not ready will evict the healthy pods. While the future state of KNI could aim to propose changes to the kube-scheduler, it's not a part of our initial work and instead should try to assist other efforts such as DRA/device plugin to provide the information they need. 

The community may ask for more features, as we are taking a bold approach to reimagining Kubernetes networking by reducing the amount of layers involved in networking. We should prioritize feature parity with the current CNI model and then capture future work. KNI aims to be the foundational network api that is specific for Kubernetes and should make troubleshooting easier, deploying more friendly and innovate faster while reducing the need to make changes to core Kubernetes. 

### Goals

- Design a cool looking t-shirt
- Provide a RPC for the Attachment and Detachment of interface[s] for a Pod
- Provide a RPC for the Querying of Pod network information (interfaces, network namespace path, ip addresses, routes, ...)
- Provide a RPC to prevent additional scheduling of pods if IPAM is out of IP addresses without evicting running pods
- Provide a RPC to indicate network readiness for the node (no more CNI network configuration files in host file system)
- Provide a RPC to provide the user the ability to query what networks are on a node
- Consolidate K8s networking to a single layer without involving the container/oci runtimes
- KNI should provide the RPC's required to establish feature parity with current CNI [ADD, DEL]
- Provide documentation, examples, troubleshooting and FAQ's for KNI.
- Decouple the Pod and Node Network setup, while continuing to provide guarantees about the readiness of both.
- Provide garbage collection to ensure no resources created during pod setup such as Linux bridges, ebpf programs, 
allocated IP addresses are left behind after pod deletion
- Improve the current IP handling for pods (PodIP) to be handle multiple IP addresses and 
a field to identify the IP address family (IPV4 vs IPV6)
- Provide backwards compatibility for the existing CNI approach and migration a path to fully adopt KNI
- Provide a uniform approach for network setup/teardown for both virtualized (kata) and non-virtualized (runc) 
runtimes including kubevirt. This could eliminate the high and low level runtimes from the networking path
- Provide a reference implementation of the KNI network runtime
- Provide the ability to have all the dependencies packaged in the container image (no more CNI binaries in the host file system)
..- No more downloading CNI binaries via initContainers/Mounting /etc/cni/net.d or /opt/cni/bin
- Provide the ability to use native k8s resources for configuration such as a ConfigMap's instead of configuration files in host file system
- Eliminate the need to exec binaries and replace with gRPC
- Make troubleshooting easier by having network runtime logs accessible via kubectl logs
- Improve network pod startup time

### Non-Goals

1. Any changes to the kube-scheduler 
2. Any specific implementation other than the reference implementation. However we should ensure the KNI-API is flexible enough to support

## Proposal

The proposal of this KEP is to design and implement the KNI-API and make necessary changes to the CRI-API and container runtimes. The scope should be kept to a minimum and we should target feature parity. 

### User Stories

#### As a service mesh

There may be multiple daemonsets from multiple vendors installing multiple CNI plugins on the same node, and any model we adopt must reckon with this as a first-class concern.

At a minimum it's important for me that any CNI/KNI extension support meets the following basic criteria:

- I am able to validate that KNI is "ready" on a node, and that node-level networking is "configured"
- I am able to subscribe or register an extension with KNI from an in-cluster daemonset, and have guarantees TOCTOU errors will not [silently unregister my extension](https://github.com/istio/istio/issues/46848).
- I have a guarantee that if the above things are true, my extension will be invoked for every pod creation after that point, and that if my extension (or any other extension) fails during invocation, the pod will not be scheduled on the node.
- I am able to get things like interfaces, netns paths, and assigned IPs for that pod, as context for my invocation.

A key part of what current CNI provides is a framework for different vendors to independently add extensions (CNI plugins) to a node, and have guarantees about when in the lifecycle of pod creation those are invoked, and what context they have from other plugins.

#### As a service mesh operator

- I have guarantees that pods will not be scheduled on a node until all networking configuration and all extensions I have installed successfully complete.
- I am able to add and remove vendor-specific networking extensions to running nodes (via daemonsets or some other mechanism).
- I am able to easily upgrade KNI on nodes.
- I am able to easily upgrade any vendor networking extensions.

We are constantly adding these user stories, please join the community sync to discuss. 

### Notes/Constraints/Caveats

## Constraints

1. Guarantee the pod interface is setup and in a healthy state before containers are started (ephemeral, init, regular)

## Notes

Additional Information/Diagrams: https://docs.google.com/document/d/1Gz7iNtJNMI-zKJhaOcI3aflPCx3etJ01JMxzbtvruKk/edit?usp=sharing

Changes to the pod specification will require hard evidence. 

The specifics of "Network Readiness" is an implementation detail. We need to provide this RPC to the user. 

Since the network runtime can be run separated from the container runtime, you can package everything into a pod and not need to have binaries on disk. This allows the CNI plugins to be isolated in the pod and the pod will never need to mount /opt/cni/bin or /etc/cni/net.d. This offers a potentially more ability to control execution. Keep in mind CNI is the implementation however when this is used chaining is still available.

## Ongoing Considerations

### KNI Implementations PULL instead of PUSH?

The original KNI POC provides a gRPC API for callbacks which (in the POC) are
added to the Kubelet during `Pod` tasks to callout to the KNI implementation to
get `Pod` networking configured. This is pretty straightforward, and the
initial POC actually showed very good performance characteristics, but it has
a couple of potential downsides:

1. the synchronous nature of callbacks makes it harder to avoid deadlocks
2. in some extremely heavy use cases with lots of `Pods` rapidly deploying and
   tearing down, this could be a potential scalability bottleneck.

Additionally, we intend to create Kubernetes APIs for networks and their
configurations, which means that the Kubelet and other component would operate
as something of a middleman consuming Kubernetes APIs via watch mechanisms and
converting them to gRPC calls, and then being responsible for the status,
e.t.c.

As such we've been actively considering whether it might make sense for at
least some of the functionality in KNI (such as domain/namespace
creation/deletion, and then interface attachment/detachment) be done by the KNI
directly via the KNI Kubernetes APIs.

For a simplified example a KNI implementation might watch `Pods` and wait for
kubelet to reach a state (via the status) where it indicates its ready to hand
off for network setup. The KNI implementation does it's work, and then updates
the `Pod` indicating the network setup is complete and the `Pods` containers
are then created.

There are downsides to this approach as well, one in particularly being that it
makes the provision of hookpoints for the KNI a lot more complicated for the
implementations. For now we've added this to our ongoing considerations section
as something to come back to, discuss and review.
