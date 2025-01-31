# Kubelet endpoint for pod resource assignment

## Table of Contents

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
    - [Device aware CNI plugin](#device-aware-cni-plugin)
    - [Topology aware scheduling](#topology-aware-scheduling)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Proposed API](#proposed-api)
  - [Test Plan](#test-plan)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha](#alpha)
    - [Alpha to Beta Graduation](#alpha-to-beta-graduation)
    - [Beta to G.A Graduation](#beta-to-ga-graduation)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature enablement and rollback](#feature-enablement-and-rollback)
  - [Rollout, Upgrade and Rollback Planning](#rollout-upgrade-and-rollback-planning)
  - [Monitoring requirements](#monitoring-requirements)
  - [Dependencies](#dependencies)
  - [Scalability](#scalability)
  - [Troubleshooting](#troubleshooting)
- [Implementation History](#implementation-history)
- [Alternatives](#alternatives)
  - [Add v1alpha1 Kubelet GRPC service, at <code>/var/lib/kubelet/pod-resources/kubelet.sock</code>, which returns a list of <a href="https://github.com/kubernetes/kubernetes/blob/master/pkg/kubelet/apis/cri/runtime/v1alpha2/api.proto#L734">CreateContainerRequest</a>s used to create containers.](#add-v1alpha1-kubelet-grpc-service-at--which-returns-a-list-of-createcontainerrequests-used-to-create-containers)
  - [Add a field to Pod Status.](#add-a-field-to-pod-status)
  - [Use the Kubelet Device Manager Checkpoint file](#use-the-kubelet-device-manager-checkpoint-file)
  - [Add a field to the Pod Spec:](#add-a-field-to-the-pod-spec)
<!-- /toc -->

## Release Signoff Checklist

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [X] (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements](https://github.com/kubernetes/enhancements/issues/606)
- [X] (R) KEP approvers have approved the KEP status as `implementable`
- [X] (R) Design details are appropriately documented
- [X] (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input
- [X] (R) Graduation criteria is in place
- [X] (R) Production readiness review completed
- [X] Production readiness review approved
- [X] "Implementation History" section is up-to-date for milestone
- ~~ [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io] ~~
- [X] Supporting documentation e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[kubernetes/website]: https://git.k8s.io/website

## Summary

This document presents the kubelet endpoint which allows third party consumers to inspect the mapping between devices and pods.

## Motivation

[Device Monitoring](https://docs.google.com/document/d/1NYnqw-HDQ6Y3L_mk85Q3wkxDtGNWTxpsedsgw4NgWpg/edit?usp=sharing) in Kubernetes is expected to be implemented out of the kubernetes tree.

For the metrics to be relevant to cluster administrators or Pod owners, they need to be able to be matched to specific containers and pod (e.g: GPU utilization for pod X).
As such the external monitoring agents need to be able to determine the set of devices in-use by containers and attach pod and container metadata to the metrics.

### Goals

* Deprecate and remove current device-specific knowledge from the kubelet, such as [accelerator metrics](https://github.com/kubernetes/kubernetes/blob/master/pkg/kubelet/apis/stats/v1alpha1/types.go#L229)
* Enable external device monitoring agents to provide metrics relevant to Kubernetes

## Proposal

### User Stories

* As a _Cluster Administrator_, I provide a set of devices from various vendors in my cluster.  Each vendor independently maintains their own agent, so I run monitoring agents only for devices I provide.  Each agent adheres to to the [node monitoring guidelines](https://docs.google.com/document/d/1_CdNWIjPBqVDMvu82aJICQsSCbh2BR-y9a8uXjQm4TI/edit?usp=sharing), so I can use a compatible monitoring pipeline to collect and analyze metrics from a variety of agents, even though they are maintained by different vendors.
* As a _Device Vendor_, I manufacture devices and I have deep domain expertise in how to run and monitor them. Because I maintain my own Device Plugin implementation, as well as Device Monitoring Agent, I can provide consumers of my devices an easy way to consume and monitor my devices without requiring open-source contributions. The Device Monitoring Agent doesn't have any dependencies on the Device Plugin, so I can decouple monitoring from device lifecycle management. My Device Monitoring Agent works by periodically querying the `/devices/<ResourceName>` endpoint to discover which devices are being used, and to get the container/pod metadata associated with the metrics:

![device monitoring architecture](https://user-images.githubusercontent.com/3262098/43926483-44331496-9bdf-11e8-82a0-14b47583b103.png)

#### Device aware CNI plugin

As soon as this interface has been introduced it was used by CNI plugins like [kuryr-kubernetes](https://review.opendev.org/#/c/651580/) in tandem with [intel-sriov-device-plugin](https://github.com/intel/sriov-network-device-plugin) to correctly define which devices were assigned to the pod. Since intel-sriov-device-plugin provides pci address of the device as a device id, CNI plugin can make correct assumption about the exact device. When CNI plugin knows concrete device, in most cases it's a VF of SR-IOV, it puts it into network namespace of the container. It allows to use a device from appropriate NUMA node.

#### Topology aware scheduling

This interface can be used to track down allocated resources with information about the NUMA topology of the worker node in general way.
This interface can be used to the available resources on the worker node. The kubelet is the best source of information because it manages concrete resources assignment. The information can then be used in NUMA aware scheduling.


### Risks and Mitigations

This API is read-only, which removes a large class of risks. The aspects that we consider below are as follows:
- What are the risks associated with the API service itself?
- What are the risks associated with the data itself?

| Risk                                                      | Impact        | Mitigation |
| --------------------------------------------------------- | ------------- | ---------- |
| Too many requests risk impacting the kubelet performances | High          | Implement rate limiting and or passive caching, follow best practices for gRPC resource management. |
| Improper access to the data | Low | Server is listening on a root owned unix socket. This can be limited with proper pod security policies. |


## Design Details

### Proposed API

We propose to add a new gRPC service to the Kubelet. This gRPC service would be listening on a unix socket at `/var/lib/kubelet/pod-resources/kubelet.sock` and return information about the kubelet's assignment of devices to containers.

This gRPC service returns information about
- the devices which kubelet knows about from the device plugins.
- the kubelet's assignment of devices and cpus with NUMA id to containers.
The GRPC service obtains this information from the internal state of the kubelet's Device Manager and CPU Manager respectively.
The GRPC Service exposes these endpoints:
- `List`, which returns a single PodResourcesResponse, enabling monitor applications to poll for resources allocated to pods and containers on the node.
- `GetAllocatableResources`, which returns AllocatableResourcesResponse, enabling monitor applications to learn about the the allocatable resources available on the node.
- `Watch`, which returns a stream of PodResourcesResponse, enabling monitor applications to be notified of new resource allocation, release or resource allocation updates, using the `action` field in the response.

This is shown in proto below:
```protobuf
// PodResourcesLister is a service provided by the kubelet that provides information about the
// node resources consumed by pods and containers on the node
service PodResourcesLister {
    rpc List(ListPodResourcesRequest) returns (ListPodResourcesResponse) {}
    rpc GetAllocatableResources(AllocatableResourcesRequest) returns (AllocatableResourcesResponse) {}
    rpc Watch(WatchPodResourcesRequest) returns (stream WatchPodResourcesResponse) {}
}

message AllocatableResourcesRequest {}

// AvailableResourcesResponses contains informations about all the devices known by the kubelet
message AllocatableResourcesResponse {
    repeated ContainerDevices devices = 1;
    repeated int64 cpu_ids = 2;
}

// ListPodResourcesRequest is the request made to the PodResources service
message ListPodResourcesRequest {}

// ListPodResourcesResponse is the response returned by List function
message ListPodResourcesResponse {
    repeated PodResources pod_resources = 1;
}

// WatchPodResourcesRequest is the request made to the Watch PodResourcesLister service
message WatchPodResourcesRequest {}

enum WatchPodAction {
    ADDED = 0;
    DELETED = 1;
}

// WatchPodResourcesResponse is the response returned by Watch function
message WatchPodResourcesResponse {
    WatchPodAction action = 1;
    string uid = 2;
    repeated PodResources pod_resources = 3;
}

// PodResources contains information about the node resources assigned to a pod
message PodResources {
    string name = 1;
    string namespace = 2;
    repeated ContainerResources containers = 3;
}

// ContainerResources contains information about the resources assigned to a container
message ContainerResources {
    string name = 1;
    repeated ContainerDevices devices = 2;
    repeated int64 cpu_ids = 3;
}

// Topology describes hardware topology of the resource
message TopologyInfo {
	repeated NUMANode nodes = 1;
}

// NUMA representation of NUMA node
message NUMANode {
	int64 ID = 1;
}

// ContainerDevices contains information about the devices assigned to a container
message ContainerDevices {
    string resource_name = 1;
    repeated string device_ids = 2;
    TopologyInfo topology = 3;
}
```

When new `HintProvider`s will be added to the `kubelet`, ContainerDevices will be updated accordingly to expose
the resources managed by the new `HintProvider`.
ContainerDevices reflects information obtained from device plugins. For example, device plugin could describe
resource which spread across NUMA nodes. Also it's possible to have a device located on the different NUMA nodes.
ContainerDevices could represent it as following:
```json
[
    {
        resource_name: "res1",
        device_ids: [ "dev1", "dev2" ],
        topology: [ { ID: 0 }, ],
    },
    {
        resource_name: "res1",
        device_ids: [ "dev3", "dev4" ],
        topology: [ { ID: 1 } ]
    },
    {
        resource_name: "res2",
        device_ids: [ "dev5" ],
        topology: [ { ID: 1 }, { ID: 2 } ]
    },
]
```

Using the `Watch` endpoint, client applications can be notified of the pod resource allocation changes as soon as possible.
However, the state of a pod will not be sent up until the first resource allocation change, which is the pod deletion in the worst case.
Client applications who need to have the complete resource allocation picture thus need to consume both `List` and `Watch` endpoints.

The `resourceVersion` found in the responses of both APIs allows client applications to identify the most recent information.
The `resourceVersion` value is updated following the same semantics of pod `resourceVersion` value, and the implementation
may use the same value from the corresponding pods.
To keep the implementation simple as possible, the kubelet does *not* store any historical list of changes.

In order to make sure not to miss any updates, client application can:
1. call the `Watch` endpoint to get a stream of changes.
2. call the `List` endpoint to get the state of all the pods in the node.
3. reconcile updates using the `resourceVersion`.

In order to make the resource accounting on the client side, safe and easy as possible the `Watch` implementation
will guarantee ordering of the event delivery in such a way that the capacity invariants are always preserved, and the value
will be consistent after each event received - not only at steady state.
Consider the following scenario with 10 devices, all allocated: pod A with device D1 allocated gets deleted, then
pod B starts and gets device D1 again. In this case `Watch` will guarantee that `DELETE` and `ADDED` events are delivered
in the correct order.

To properly evaluate the amount of allocatable compute resources on a node, client applications can use the `GetAlloctableResources` endpoint.
Applications can use `List` and `Watch` to learn about the current allocation of these resources, and thus the current amount of free (unallocated)
compute resources.
Applications are expected to run `GetAlloctableResources` each time they expect a change in the resources
availability (e.g. CPUs onlined/offlined, devices added/removed). We anticipate these changes to happen very rarely.
Client applications should not expect any ordering in the return value.

### Test Plan

Given that the API allows observing what device has been associated to what container, we need to be testing different configurations, such as:
* Pods without devices assigned to any containers.
* Pods with devices assigned to some but not all containers.
- Pods with devices assigned to init containers.
- ...

We have identified two main ways of testing this API:
- Unit Tests which won't rely on gRPC. They will test different configurations of pods and devices.
- Node E2E tests which will allow us to test the service itself. The Tests will cover the both a list-only client and a list-and-watch client.

E2E tests are explicitly not written because they would require us to generate and deploy a custom container.
The infrastructure required is expensive and it is not clear what additional testing (and hence risk reduction) this would provide compare to node e2e tests.

### Graduation Criteria

#### Alpha
- [X] Implement the new service API.
- [X] [Ensure proper e2e node tests are in place](https://testgrid.k8s.io/sig-node-kubelet#node-kubelet-serial&include-filter-by-regex=DevicePluginProbe).

#### Alpha to Beta Graduation
- [X] Demonstrate that the endpoint can be used to replace in-tree GPU device metrics in production environments (NVIDIA, sig-node April 30, 2019).

#### Beta to G.A Graduation
- [X] Multiple real world examples ([Multus CNI](https://github.com/intel/multus-cni)).
- [X] Allowing time for feedback (2 years).
- [X] [Start Deprecation of Accelerator metrics in kubelet](https://github.com/kubernetes/kubernetes/pull/91930).
- [X] Risks have been addressed.

### Upgrade / Downgrade Strategy

With gRPC the version is part of the service name.
Old versions and new versions should always be served and listened by the kubelet.

To a cluster admin upgrading to the newest API version, means upgrading Kubernetes to a newer version as well as upgrading the monitoring component.

To a vendor changes in the API should always be backwards compatible.

Downgrades here are related to downgrading the plugin

### Version Skew Strategy

Kubelet will always be backwards compatible, so going forward existing plugins are not expected to break.

## Production Readiness Review Questionnaire
### Feature enablement and rollback

* **How can this feature be enabled / disabled in a live cluster?**
  - [X] Feature gate (also fill in values in `kep.yaml`).
    - Feature gate name: `KubeletPodResources`.
    - Components depending on the feature gate: N/A.

* **Does enabling the feature change any default behavior?** No
* **Can the feature be disabled once it has been enabled (i.e. can we rollback the enablement)?** Yes, through feature gates.
* **What happens if we reenable the feature if it was previously rolled back?** The service recovers state from kubelet.
* **Are there any tests for feature enablement/disablement?** No, however no data is created or deleted.

### Rollout, Upgrade and Rollback Planning

* **How can a rollout fail? Can it impact already running workloads?** Kubelet would fail to start. Errors would be caught in the CI.
* **What specific metrics should inform a rollback?** Not Applicable, metrics wouldn't be available.
* **Were upgrade and rollback tested? Was upgrade->downgrade->upgrade path tested?** Not Applicable.
* **Is the rollout accompanied by any deprecations and/or removals of features,  APIs, fields of API types, flags, etc.?** No.

### Monitoring requirements
* **How can an operator determine if the feature is in use by workloads?**
  - Look at the `pod_resources_endpoint_requests_total` metric exposed by the kubelet.
  - Look at hostPath mounts of privileged containers.
* **What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?**
  - [X] Metrics
    - Metric name: `pod_resources_endpoint_requests_total`
    - Components exposing the metric: kubelet

* **What are the reasonable SLOs (Service Level Objectives) for the above SLIs?** N/A or refer to Kubelet SLIs.
* **Are there any missing metrics that would be useful to have to improve observability if this feature?** No.


### Dependencies

* **Does this feature depend on any specific services running in the cluster?** Not aplicable.

### Scalability

* **Will enabling / using this feature result in any new API calls?** No.
* **Will enabling / using this feature result in introducing new API types?** No.
* **Will enabling / using this feature result in any new calls to cloud provider?** No.
* **Will enabling / using this feature result in increasing size or count of the existing API objects?** No.
* **Will enabling / using this feature result in increasing time taken by any operations covered by [existing SLIs/SLOs][]?** No. Feature is out of existing any paths in kubelet.
* **Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components?** In 1.18, DDOSing the API can lead to resource exhaustion. It is planned to be addressed as part of G.A.
Feature only collects data when requests comes in, data is then garbage collected. Data collected is proportional to the number of pods on the node.

### Troubleshooting

* **How does this feature react if the API server and/or etcd is unavailable?**: No effect.
* **What are other known failure modes?** No known failure modes
* **What steps should be taken if SLOs are not being met to determine the problem?** N/A

[supported limits]: https://git.k8s.io/community//sig-scalability/configs-and-limits/thresholds.md
[existing SLIs/SLOs]: https://git.k8s.io/community/sig-scalability/slos/slos.md#kubernetes-slisslos

## Implementation History

- 2018-09-11: Final version of KEP (proposing pod-resources endpoint) published and presented to sig-node.  [Slides](https://docs.google.com/presentation/u/1/d/1xz-iHs8Ec6PqtZGzsmG1e68aLGCX576j_WRptd2114g/edit?usp=sharing)
- 2018-10-30: Demo with example gpu monitoring daemonset
- 2018-11-10: KEP lgtm'd and approved
- 2018-11-15: Implementation and e2e test merged before 1.13 release: kubernetes/kubernetes#70508
- 2019-04-30: Demo of production GPU monitoring by NVIDIA
- 2019-04-30: Agreement in sig-node to move feature to beta in 1.15
- 2020-06-17: Agreement in sig-node to move feature to G.A in 1.19 or 1.20
- 2020-07-06: Add Topology and cpus into PodResources interface
- 2020-09-02: Add the GetAllocatableResources endpoint
- 2020-10-01: KEP extended with Watch API
- 2020-11-02: Agreement in sig-node to implement Watch API in the 1.21 cycle

## Alternatives

### Add v1alpha1 Kubelet GRPC service, at `/var/lib/kubelet/pod-resources/kubelet.sock`, which returns a list of [CreateContainerRequest](https://github.com/kubernetes/kubernetes/blob/master/pkg/kubelet/apis/cri/runtime/v1alpha2/api.proto#L734)s used to create containers.
* Pros:
  * Reuse an existing API for describing containers rather than inventing a new one
* Cons:
  * It ties the endpoint to the CreateContainerRequest, and may prevent us from adding other information we want in the future
  * It does not contain any additional information that will be useful to monitoring agents other than device, and contains lots of irrelevant information for this use-case.
* Notes:
  * Does not include any reference to resource names.  Monitoring agentes must identify devices by the device or environment variables passed to the pod or container.

### Add a field to Pod Status.
* Pros:
  * Allows for observation of container to device bindings local to the node through the `/pods` endpoint
* Cons:
  * Only consumed locally, which doesn't justify an API change
  * Device Bindings are immutable after allocation, and are _debatably_ observable (they can be "observed" from the local checkpoint file).  Device bindings are generally a poor fit for status.

### Use the Kubelet Device Manager Checkpoint file
* Allows for observability of device to container bindings through what exists in the checkpoint file
  * Requires adding additional metadata to the checkpoint file as required by the monitoring agent
* Requires implementing versioning for the checkpoint file, and handling version skew between readers and the kubelet
* Future modifications to the checkpoint file are more difficult.

### Add a field to the Pod Spec:
* A new object `ComputeDevice` will be defined and a new variable `ComputeDevices` will be added in the `Container` (Spec) object which will represent a list of `ComputeDevice` objects.
```golang
// ComputeDevice describes the devices assigned to this container for a given ResourceName
type ComputeDevice struct {
	// DeviceIDs is the list of devices assigned to this container
	DeviceIDs []string
	// ResourceName is the name of the compute resource
	ResourceName string
}

// Container represents a single container that is expected to be run on the host.
type Container struct {
    ...
	// ComputeDevices contains the devices assigned to this container
	// This field is alpha-level and is only honored by servers that enable the ComputeDevices feature.
	// +optional
	ComputeDevices []ComputeDevice
	...
}
```
* During Kubelet pod admission, if `ComputeDevices` is found non-empty, specified devices will be allocated otherwise behaviour will remain same as it is today.
* Before starting the pod, the kubelet writes the assigned `ComputeDevices` back to the pod spec.  
  * Note: Writing to the Api Server and waiting to observe the updated pod spec in the kubelet's pod watch may add significant latency to pod startup.
* Allows devices to potentially be assigned by a custom scheduler.
* Serves as a permanent record of device assignments for the kubelet, and eliminates the need for the kubelet to maintain this state locally.
