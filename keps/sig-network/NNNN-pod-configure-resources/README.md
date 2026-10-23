# KEP-NNNN: DRA Pod Resource Configuration After Sandbox Creation

Discussion draft — updated October 23, 2026. RPC names, feature gate, API fields, and
lifecycle rules below are proposals, not existing Kubernetes interfaces. KEP number,
authors, reviewers, target releases, and final SIG ownership are TBD. Proposed owning
SIG: SIG Network; participating SIG: SIG Node. Sponsorship and reviewer assignments
remain to be confirmed.

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Startup ordering](#startup-ordering)
  - [DRA API: protobuf changes](#dra-api-protobuf-changes)
  - [Driver opt-in and compatibility](#driver-opt-in-and-compatibility)
    - [Illustrative driver startup option](#illustrative-driver-startup-option)
    - [Required support handshake](#required-support-handshake)
    - [Driver helper and kubelet implementation responsibilities](#driver-helper-and-kubelet-implementation-responsibilities)
  - [Network namespace access and CRI dependency](#network-namespace-access-and-cri-dependency)
    - [CRI API: protobuf changes](#cri-api-protobuf-changes)
  - [User namespace considerations](#user-namespace-considerations)
  - [Retries, partial failure, and recovery](#retries-partial-failure-and-recovery)
  - [Teardown and claim sharing](#teardown-and-claim-sharing)
  - [Open Questions for SIG Discussion](#open-questions-for-sig-discussion)
  - [Test Plan](#test-plan)
    - [Prerequisite testing updates](#prerequisite-testing-updates)
    - [Unit tests](#unit-tests)
    - [Integration tests](#integration-tests)
    - [e2e tests](#e2e-tests)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha](#alpha)
    - [Beta](#beta)
    - [GA](#ga)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature Enablement and Rollback](#feature-enablement-and-rollback)
        - [How can this feature be enabled / disabled in a live cluster?](#how-can-this-feature-be-enabled--disabled-in-a-live-cluster)
        - [Does enabling the feature change any default behavior?](#does-enabling-the-feature-change-any-default-behavior)
        - [Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?](#can-the-feature-be-disabled-once-it-has-been-enabled-ie-can-we-roll-back-the-enablement)
        - [What happens if we reenable the feature if it was previously rolled back?](#what-happens-if-we-reenable-the-feature-if-it-was-previously-rolled-back)
        - [Are there any tests for feature enablement/disablement?](#are-there-any-tests-for-feature-enablementdisablement)
  - [Rollout, Upgrade and Rollback Planning](#rollout-upgrade-and-rollback-planning)
        - [How can a rollout or rollback fail? Can it impact already running workloads?](#how-can-a-rollout-or-rollback-fail-can-it-impact-already-running-workloads)
        - [What specific metrics should inform a rollback?](#what-specific-metrics-should-inform-a-rollback)
        - [Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?](#were-upgrade-and-rollback-tested-was-the-upgrade-downgrade-upgrade-path-tested)
        - [Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?](#is-the-rollout-accompanied-by-any-deprecations-andor-removals-of-features-apis-fields-of-api-types-flags-etc)
  - [Monitoring Requirements](#monitoring-requirements)
        - [How can an operator determine if the feature is in use by workloads?](#how-can-an-operator-determine-if-the-feature-is-in-use-by-workloads)
        - [How can someone using this feature know that it is working for their instance?](#how-can-someone-using-this-feature-know-that-it-is-working-for-their-instance)
        - [What are the reasonable SLOs (Service Level Objectives) for the enhancement?](#what-are-the-reasonable-slos-service-level-objectives-for-the-enhancement)
        - [What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?](#what-are-the-slis-service-level-indicators-an-operator-can-use-to-determine-the-health-of-the-service)
        - [Are there any missing metrics that would be useful to have to improve observability of this feature?](#are-there-any-missing-metrics-that-would-be-useful-to-have-to-improve-observability-of-this-feature)
  - [Dependencies](#dependencies)
        - [Does this feature depend on any specific services running in the cluster?](#does-this-feature-depend-on-any-specific-services-running-in-the-cluster)
  - [Scalability](#scalability)
        - [Will enabling / using this feature result in any new API calls?](#will-enabling--using-this-feature-result-in-any-new-api-calls)
        - [Will enabling / using this feature result in introducing new API types?](#will-enabling--using-this-feature-result-in-introducing-new-api-types)
        - [Will enabling / using this feature result in any new calls to the cloud provider?](#will-enabling--using-this-feature-result-in-any-new-calls-to-the-cloud-provider)
        - [Will enabling / using this feature result in increasing size or count of the existing API objects?](#will-enabling--using-this-feature-result-in-increasing-size-or-count-of-the-existing-api-objects)
        - [Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?](#will-enabling--using-this-feature-result-in-increasing-time-taken-by-any-operations-covered-by-existing-slisslos)
        - [Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components?](#will-enabling--using-this-feature-result-in-non-negligible-increase-of-resource-usage-cpu-ram-disk-io--in-any-components)
        - [Can enabling / using this feature result in resource exhaustion of some node resources (PIDs, sockets, inodes, etc.)?](#can-enabling--using-this-feature-result-in-resource-exhaustion-of-some-node-resources-pids-sockets-inodes-etc)
  - [Troubleshooting](#troubleshooting)
        - [How does this feature react if the API server and/or etcd is unavailable?](#how-does-this-feature-react-if-the-api-server-andor-etcd-is-unavailable)
        - [What are other known failure modes?](#what-are-other-known-failure-modes)
        - [What steps should be taken if SLOs are not being met to determine the problem?](#what-steps-should-be-taken-if-slos-are-not-being-met-to-determine-the-problem)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Infrastructure Needed](#infrastructure-needed)
<!-- /toc -->

## Release Signoff Checklist

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [ ] (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [ ] (R) KEP approvers have approved the KEP status as `implementable`
- [ ] (R) Design details are appropriately documented
- [ ] (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input (including test refactors)
  - [ ] e2e Tests for all Beta API Operations (endpoints)
  - [ ] (R) Ensure GA e2e tests meet requirements for [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md)
  - [ ] (R) Minimum Two Week Window for GA e2e tests to prove flake free
- [ ] (R) Graduation criteria is in place
  - [ ] (R) [all GA Endpoints](https://github.com/kubernetes/community/pull/1806) must be hit by [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md) within one minor version of promotion to GA
- [ ] (R) Production readiness review completed
- [ ] (R) Production readiness review approved
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [ ] Supporting documentation—e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[kubernetes/website]: https://git.k8s.io/website

## Summary

Introduce an optional DRA node RPC, provisionally named `NodeConfigurePodResources`,
that kubelet invokes after CRI `RunPodSandbox` succeeds and before creating any workload
container in that sandbox (CRI `CreateContainer`). This gives DRA drivers a Kubernetes-managed opportunity to
finish configuration that depends on the pod sandbox, particularly its network
namespace.

This KEP targets DRA resources configured for the pod as a whole and associated with its
sandbox. Container-specific resource configuration and per-container lifecycle
operations are out of scope and may be addressed by a future KEP. This scope does not
change the core Pod API for CPU or memory requests.

Keep `NodePrepareResources` as the existing claim-level preparation operation. The new
operation is scoped to a particular pod and sandbox instance, because a prepared
ResourceClaim can outlive a sandbox or be used by multiple pods.

Pair configuration with `NodeUnconfigurePodResources` for sandbox-scoped cleanup. A
configuration failure fails the sandbox startup attempt: kubelet attempts rollback,
stops the sandbox, and retries startup with backoff.

Linux networking is the initial use case. The interface should allow future
sandbox-dependent resource configuration without becoming a general-purpose runtime hook
system. A standardized CRI mechanism for exposing a usable network namespace reference
is a prerequisite for portable Linux networking support.

## Motivation

Kubelet currently prepares DRA resources before sandbox creation. Drivers cannot
complete operations requiring the sandbox network namespace at that point. The current
ordering is visible in [kubelet's runtime
manager](https://github.com/kubernetes/kubernetes/blob/master/pkg/kubelet/kuberuntime/kuberuntime_manager.go).

Network resource drivers may need to move an allocated interface into a pod namespace,
configure a device-backed interface, or apply namespace-local settings before
applications run. CDI hooks can cover basic networking cases: moving a network device
into the pod network namespace, assigning or restoring its IP address, and bringing
the interface up. That does not necessarily make the network ready for the workload.
Additional work, such as IP address allocation, routing and policy configuration, or
coordination with an external network service, still needs a component to complete
the setup. Drivers can combine DRA with runtime hooks such as NRI to perform that
work, but must coordinate it with the DRA claim and sandbox lifecycle.

CDI does not define a complete network provisioning workflow. Extending a hook to
finish setup leaves the driver responsible for orchestration and for reconciling
container-scoped hook execution with pod-wide network state: configuring the interface
once, avoiding repeated moves as containers start, and cleaning up when the sandbox
is replaced. A DRA-native operation would make that sandbox-scoped coordination
explicit between kubelet and the driver.

It would also surface network configuration failures before kubelet calls
`CreateContainer` for any workload container. With setup deferred to a workload
container's CDI hook, kubelet must first complete `RunPodSandbox` and attempt container
creation or startup before that hook can report a failure. The proposed RPC still
requires a successfully created sandbox, but establishes a pod-wide configuration
barrier before any workload container creation is attempted.

NRI provides runtime-driven pod sandbox lifecycle callbacks and can avoid those
container-hook limitations.
[KEP-5304](https://github.com/kubernetes/enhancements/blob/master/keps/sig-node/5304-dra-attributes-downward-api/README.md)
describes deferred network metadata produced through an NRI sandbox hook. [NRI's
API](https://github.com/containerd/nri/blob/main/pkg/api/api.proto) includes pod sandbox
lifecycle events.

However, a driver using NRI primarily to bridge this timing gap must also deploy and
configure an NRI integration, correlate runtime callbacks with prepared DRA claims,
and coordinate recovery across the two interfaces. NRI already uses RPCs; the benefit
of this proposal is making the operation part of the DRA lifecycle under kubelet's
control.

A dedicated DRA RPC would carry explicit pod, sandbox, and claim context directly to
the driver after sandbox creation. Its synchronous response would give kubelet a
defined success or failure boundary before creating workload containers. Paired
configuration and unconfiguration operations would let kubelet coordinate timeouts,
rollback, retries, and sandbox replacement with the driver that owns the resource.
Drivers still need idempotent operations and recovery from partial failures, but
would implement them through one DRA contract rather than correlate DRA preparation
with runtime hooks and separately published device specifications.

This proposal reduces the need for CDI hooks or NRI for this specific sandbox-scoped
network setup. CDI remains useful for container device injection, and NRI continues
to serve other runtime integration needs.

### Goals

- Configure resources for the pod as a whole, with configuration and cleanup scoped to its sandbox lifetime.
- Provide a synchronous DRA operation after successful sandbox creation and before workload container creation.
- Make the sandbox's network namespace accessible through a defined, supported contract.
- Distinguish claim-level preparation from pod-and-sandbox-level configuration.
- Define recovery from timeouts, driver or kubelet restarts, and sandbox replacement.
- Preserve existing behavior for drivers that do not require the operation.

### Non-Goals

- Configure resources for individual containers, select different resource configurations per container, or add per-container configure/unconfigure RPCs. These can be considered in a separate KEP.
- Change existing container-specific device injection performed through claim preparation and the container runtime.
- Replace primary CNI networking or configure resources before CNI executes.
- Introduce arbitrary hooks for every container lifecycle transition.
- Change scheduling, allocation, or the ResourceClaim sharing model.
- Support dependencies between different drivers' configuration operations in the initial version.
- Promise host-accessible Linux namespaces for VM-based sandboxes or Windows runtimes.

## Proposal

Kubelet will coordinate sandbox-dependent DRA configuration after successful sandbox
creation CRI `RunPodSandbox` and before workload container creation CRI `CreateContainer`. Participating drivers receive pod
metadata and sandbox context, and a paired unconfiguration operation handles
sandbox-scoped cleanup. Both operations apply to pod-level resources; they do not
receive a target container or run separately for each container. Failed configuration
causes rollback of the startup attempt and a retry with backoff. Drivers that do not opt
in keep their existing lifecycle.

### User Stories

1. **As a DRA network driver developer**, I want to configure allocated network devices after the pod's network namespace exists, so that applications can use them from their first container start.
2. **As a cluster administrator**, I want to deploy DRA drivers that perform sandbox-dependent configuration through kubelet, so that I can support those drivers without also configuring an NRI integration.
3. **As a workload author**, I want my init and workload containers to start only after their allocated resources are configured, so that they don't need startup scripts or retries to handle incomplete device setup.

### Risks and Mitigations

Namespace access gives a driver authority over pod networking and may require host
privileges. The kubelet supplies runtime-derived context only for resources assigned to
that driver. Drivers validate identities and never treat workload annotations as trusted
namespace paths.

Drivers must avoid altering primary CNI interfaces or routes outside their documented
ownership. Independent driver operations must not depend on invocation order. The first
version should use a predictable sequential invocation strategy; ordering is not a
dependency API.

The new barrier adds pod startup latency and makes driver availability part of startup
readiness. Checkpoint growth, hung drivers, leaked host resources, and unsafe device
reuse require explicit bounds and recovery tests. Kubelet should not report that the pod
is ready to start containers before this configuration has completed; the precise
condition integration requires review.

## Design Details

### Startup ordering

```text
Allocate and reserve ResourceClaims
  -> NodePrepareResources (claim-level, existing)
  -> CRI RunPodSandbox
  -> CRI PodSandboxStatus (typed sandbox context)
  -> NodeConfigurePodResources (pod/sandbox-level, proposed)
  -> CreateContainer / StartContainer for init and application containers
```

Kubelet must require successful configuration by every participating driver before
creating a workload container. This applies to init containers, restartable init
containers, regular containers, and subsequently added ephemeral containers. The
runtime's infrastructure processes may already exist; the guarantee concerns workload
containers managed by kubelet.

Configuration runs for each new sandbox instance, including replacement sandboxes for
the same pod. It does not run for each container restart in a successfully configured
sandbox.

After kubelet restart, reconciliation must also enforce this barrier when reusing an
existing sandbox. Placing the call only inside the sandbox creation branch would be
insufficient.

### DRA API: protobuf changes

**API boundary:** kubelet calls the DRA node plugin. These additions belong to
`staging/src/k8s.io/kubelet/pkg/apis/dra/<version>/api.proto` in kubernetes/kubernetes.
The target DRA API version remains TBD. They do not belong to `runtime.v1` or cri-api.

The RPC declarations below are additions inside the existing `DRAPlugin` service; the
messages are package-level additions. `Claim` and `Device` refer to existing DRA
messages. Exact protobuf schema and API versioning require review.

```proto
rpc NodeConfigurePodResources(NodeConfigurePodResourcesRequest)
    returns (NodeConfigurePodResourcesResponse);
rpc NodeUnconfigurePodResources(NodeUnconfigurePodResourcesRequest)
    returns (NodeUnconfigurePodResourcesResponse);

message NodeConfigurePodResourcesRequest {
  // The ResourceClaims to configure for the pod.
  repeated Claim claims = 1;
  // The sandbox being configured. Required.
  Sandbox sandbox = 2;
  // Selected pod identity and metadata. Required.
  // Deliberately excludes the pod spec and status.
  PodMetadata pod = 3;
}

message PodMetadata {
  // Pod identity. All three fields are required.
  string uid = 1;
  string namespace = 2;
  string name = 3;
  // Pod labels at configuration time.
  map<string, string> labels = 4;
  // Drivers should only use annotations they are authorized and
  // configured to consume.
  map<string, string> annotations = 5;
}

message NodeConfigurePodResourcesResponse {
  // Results keyed by claim UID. Every requested claim must have an entry.
  // A missing entry fails the sandbox startup attempt and triggers rollback
  // of all attempted configuration. A subsequent startup attempt uses
  // a new sandbox.
  map<string, NodeConfigurePodResourceResponse> claims = 1;
}

message NodeConfigurePodResourceResponse {
  // Additional devices for pod-scoped configuration. This response does not
  // define container-specific assignments or per-container runtime changes.
  // A claim and its requests may each have zero or more devices.
  repeated Device devices = 1;
  // If non-empty, configuration failed. Devices are ignored in that case.
  string error = 2;
}

// Runtime context for one pod sandbox instance.
message Sandbox {
  // Runtime-assigned sandbox ID. Required. Stable across retries for this
  // instance; changes when the sandbox is replaced.
  string id = 1;
  // Networking context, when available.
  SandboxNetworking networking = 2;
}

message SandboxNetworking {
  // Path to the sandbox's Linux network namespace.
  // Must be accessible from the driver's mount namespace.
  // May be absent during unconfiguration if the namespace is gone.
  string netns_path = 1;
}

message NodeUnconfigurePodResourcesRequest {
  // Claims included in the attempted configuration, including unknown outcomes.
  repeated Claim claims = 1;
  // Sandbox whose configuration must be reverted. ID is required;
  // networking context may no longer be available.
  Sandbox sandbox = 2;
}

message NodeUnconfigurePodResourcesResponse {
  // Results keyed by claim UID. Every requested claim must have an entry,
  // including claims with no remaining configuration.
  map<string, NodeUnconfigurePodResourceResponse> claims = 1;
}

message NodeUnconfigurePodResourceResponse {
  // If non-empty, cleanup failed and must be retried.
  string error = 1;
}
```

The request batches relevant claims for one pod and one sandbox per driver. Claim
identity includes UID, avoiding confusion when a claim name is reused. Drivers obtain
allocation details through their existing DRA mechanisms. Kubelet must preserve the
pod's applicable request selection when determining which resources are used; any
additional request-selection fields must be settled during API review.

The configuration response can return additional devices using the existing `Device`
message. These results are scoped to this pod sandbox, not to every pod sharing the
claim. Kubelet must retain successful results for that sandbox and use them only after
all required drivers succeed. Failed attempts discard their device results during
rollback. The use of these results for pod-scoped runtime configuration, and merge rules
with claim-preparation results, including deduplication of CDI identifiers and request
selection, require API review. This field must not introduce new container-specific
injection or assignment semantics; any such extension requires a separate KEP.

`Sandbox` uses independent message fields rather than a `oneof`, allowing networking and
future CPU or memory context to coexist. CPU and memory fields are deferred until
concrete use cases define their contents. Absent `networking` means that networking
context is unavailable. An empty `netns_path` must never implicitly select the host
namespace; drivers requiring namespace access must reject missing context during
configuration. Unconfiguration must work from the sandbox ID even after the namespace
disappears.

The field numbers above assume an unreleased API. If a schema with `netns_path = 1` and
`id = 2` has already shipped, those field numbers must be preserved and networking
context added using a new field number.

Labels and annotations are a snapshot at configuration time, not trusted authorization
inputs. Changes to this metadata do not implicitly trigger reconfiguration of an already
configured sandbox.

### Driver opt-in and compatibility

Existing DRA drivers MUST continue to work without code, configuration, or deployment
changes, whether the proposed `DRAPodSandboxConfiguration` kubelet feature gate is
disabled or enabled. Enabling the gate MUST NOT cause kubelet to probe existing drivers
with the new RPCs. Existing prepare/unprepare message semantics remain unchanged.

Use explicit driver-wide opt-in for the initial implementation. An opted-in driver
receives configuration and unconfiguration calls for pod sandboxes using its prepared
claims. It may return successful results without changes for claims that need no
sandbox-specific work. No per-claim configuration flag is proposed in the existing
preparation response.

| Kubelet feature gate | Driver | Behavior |
| --- | --- | --- |
| Disabled | Existing, not opted in | Existing prepare/unprepare lifecycle |
| Enabled | Existing, not opted in | Existing prepare/unprepare lifecycle |
| Enabled | Opted in, support established | Configure/unconfigure lifecycle |
| Disabled or unsupported | Requires configuration | Reject dependent preparation |

A driver requiring configuration MUST positively establish that kubelet supports and
has enabled the lifecycle before allowing dependent preparation to succeed. A new
registration field that an older kubelet can silently ignore is insufficient. The
acknowledgment mechanism, whether registration capabilities or explicit API version
negotiation, remains subject to design review. This KEP does not assume that such an
acknowledgment already exists. Failure must affect dependent workloads, not unrelated
legacy drivers.

If required runtime context is unavailable, or a required RPC returns `Unimplemented`,
kubelet MUST NOT start workload containers. Deployment guidance must restrict dependent
workloads to compatible nodes; registration alone does not inform scheduling. Disabling
the gate requires draining dependent workloads and completing cleanup as specified in
the downgrade strategy.

#### Illustrative driver startup option

Expose explicit opt-in through a driver helper startup option. The following API is
illustrative; it is not an existing option or a finalized signature:

```go
helper, err := kubeletplugin.Start(
    ctx,
    driver,
    kubeletplugin.DriverName(driverName),
    kubeletplugin.PodResourceConfiguration(true),
)
```

Omitting the option is equivalent to `false` and preserves the existing lifecycle.
Implementing the optional interfaces alone does not enable the feature. With the option
enabled, helper startup must validate that the driver implements both
`PodResourceConfigurer` and `PodResourceUnconfigurer`; otherwise startup returns an
error identifying the missing interface. A driver that does not use this helper must
provide equivalent opt-in and support verification through the agreed wire protocol.

#### Required support handshake

The intended handshake is:

1. The opted-in driver advertises that it requires pod resource configuration during
   plugin registration.
2. Kubelet verifies that it supports the lifecycle and that
   `DRAPodSandboxConfiguration` is enabled.
3. Kubelet explicitly acknowledges acceptance of that requirement.
4. Only after acknowledgment may the helper allow dependent claim preparation to
   succeed. Before acknowledgment, preparation returns an actionable, retryable error;
   it must not wait indefinitely inside an RPC for registration to complete.

Ordinary registration success does not count as acknowledgment: an older kubelet may
ignore a new field. Absence of acknowledgment must never be interpreted as acceptance.
Once accepted, kubelet invokes configure after sandbox creation and unconfigure during
rollback or teardown; `Unimplemented` is an error for this participating driver.

| Driver startup configuration | Result |
| --- | --- |
| Option omitted or false | Existing lifecycle; no new RPCs |
| Option true, either optional interface missing | Helper startup fails |
| Option true, kubelet explicitly accepts | New configuration lifecycle |
| Option true, gate disabled or no acknowledgment | Dependent preparation is blocked |

The wire representation of the requirement and acknowledgment remains unresolved. The
final design must define registration/reconnection scope and invalidation of acceptance
when kubelet restarts or support changes. Acknowledgment from a prior kubelet must not
silently authorize operation after a downgrade. This section specifies the required
behavior, not a completed plugin-registration protocol extension.

#### Driver helper and kubelet implementation responsibilities

The driver-side helper at
`staging/src/k8s.io/dynamic-resource-allocation/kubeletplugin/draplugin.go` wraps driver
implementations and serves the DRA gRPC API; it is not the kubelet-side RPC client.
The implementation spans these components:

| Component | Responsibility |
| --- | --- |
| DRA protobuf API | Define the new RPCs and messages. |
| Driver helper (`kubeletplugin/draplugin.go`) | Expose optional callbacks and explicit opt-in, serve handlers, and participate in support verification. |
| Kubelet plugin registration and RPC client | Establish opt-in/support and invoke RPCs only for participating drivers. |
| Kubelet DRA manager and runtime lifecycle | Enforce the feature gate, startup barrier, checkpoints, rollback, and cleanup. |
| CRI API and runtime | Expose typed sandbox context for kubelet to pass to the driver. |

The existing public Go `DRAPlugin` interface MUST NOT gain mandatory configure or
unconfigure methods. The POC implements this separation using two optional helper
interfaces: `PodResourceConfigurer`, providing `ConfigurePodResources`, and
`PodResourceUnconfigurer`, providing `UnconfigurePodResources`. Existing drivers do not
need to implement either interface. Preserve this optional-interface approach in the
upstream design, with explicit driver opt-in disabled by default. Existing driver
implementations must still compile against the updated helper and retain existing
behavior. Final interface signatures and registration option names remain subject to
API review.

The POC currently discovers support by invoking the new RPCs and treating gRPC
`Unimplemented` as an unsupported operation that can be skipped. This is prototype
behavior, not the proposed upstream compatibility contract. The upstream design
requires explicit driver opt-in and positive acknowledgment of kubelet support before
a driver relies on configuration. Implementing the optional interfaces alone does not
establish that acknowledgment. Once configuration is required, `Unimplemented` must
fail dependent startup rather than silently skip the configuration barrier. The
negotiation mechanism remains an open design question; it is not implemented by the
POC's fallback behavior.

Compatibility coverage must include an unchanged existing driver built against the
updated helper; existing drivers with the gate enabled and disabled; and a new driver
requiring configuration with the gate disabled or an older kubelet. Tests must verify
that legacy drivers receive no new RPC calls and that unsupported dependent preparation
cannot silently succeed.

### Network namespace access and CRI dependency

Kubelet should retrieve sandbox context through the existing CRI `PodSandboxStatus` RPC
after `RunPodSandbox` succeeds. No additional CRI RPC is proposed.

Containerd already exposes namespace information through verbose status, but as
implementation-specific JSON. Its [sandbox
controller](https://github.com/containerd/containerd/blob/main/internal/cri/server/podsandbox/sandbox_status.go)
populates `SandboxInfo` when `verbose` is true, marshals it to JSON, and returns the
string under `info["info"]`. The [SandboxInfo
type](https://github.com/containerd/containerd/blob/main/internal/cri/server/podsandbox/types/sandbox_info.go)
includes `runtimeSpec` and `sandboxMetadata`. The latter uses [versioned sandbox
metadata](https://github.com/containerd/containerd/blob/main/internal/cri/store/sandbox/metadata.go)
containing `NetNSPath`. In that representation, the path is at
`sandboxMetadata.Metadata.NetNSPath` after decoding `response.info["info"]`. The OCI
runtime specification can also contain namespace paths, but is not guaranteed to be
populated for an incompletely created sandbox.

Consumers must **unmarshal** the JSON string to read these fields. This demonstrates
that containerd has the information; it does not establish a portable CRI contract. A
stored path also does not prove the namespace is still usable.

#### CRI API: protobuf changes

**API boundary:** kubelet calls the container runtime. These changes belong to
`staging/src/k8s.io/cri-api/pkg/apis/runtime/v1/api.proto` in kubernetes/kubernetes,
mirrored to `kubernetes/cri-api/pkg/apis/runtime/v1/api.proto`. They are separate from
the DRA API additions above.

The proposed extension adds `networking` to the existing
`runtime.v1.LinuxPodSandboxStatus` and introduces
`runtime.v1.LinuxPodSandboxNetworking`. Existing `PodSandboxStatus.linux = 6` and
`PodSandboxStatusResponse.status = 1` remain unchanged. No new CRI RPC or DRA message
import is required. The typed field is returned when available regardless of
`PodSandboxStatusRequest.verbose`.

The following shows the amended Linux status message and its new nested context type; it
is a proposal, not a field already present upstream:

```proto
// Proposed CRI extension, separate from the DRA Sandbox message.
message LinuxPodSandboxStatus {
  Namespace namespaces = 1; // Existing field; unchanged.
  // Linux networking context, when available.
  LinuxPodSandboxNetworking networking = 2; // Proposed new field.
}

message LinuxPodSandboxNetworking {
  // Host path to the live sandbox network namespace.
  // Empty when no usable path is available; never implies host networking.
  string netns_path = 1;
}
```

Kubelet maps the two APIs explicitly:

```text
CRI PodSandboxStatusResponse.status.id
  -> DRA Sandbox.id
CRI PodSandboxStatusResponse.status.linux.networking.netns_path
  -> DRA Sandbox.networking.netns_path
```

The runtime supplies a host path; kubelet and driver deployment must ensure the driver
can access the same namespace reference. The CRI and DRA networking messages are
distinct types even though they currently carry similar information.

The exact placement, field numbers, and capability negotiation require CRI review. The
[current CRI
schema](https://github.com/kubernetes/cri-api/blob/master/pkg/apis/runtime/v1/api.proto)
exposes namespace options but no typed namespace path. Existing fields retain their
meanings; older runtimes can omit the new context and older clients can ignore it.
Existing workloads must not require it. Kubelet translates supported CRI context into
DRA `Sandbox.networking`; dependent configuration fails explicitly when required context
is missing. Existing verbose information remains unchanged for diagnostic consumers.
Runtime-specific JSON parsing may support prototypes, but is not the proposed upstream
kubelet integration.

The proposed host filesystem path representation requires a documented lifetime and a
mount arrangement that makes the reference accessible to the driver. A path in the
runtime's mount namespace is not automatically usable inside a driver DaemonSet. An
opaque sandbox ID alone does not solve namespace access, and a process ID alone
introduces lifetime and reuse hazards.

The runtime must keep the referenced namespace valid while the sandbox is live and
kubelet is performing configuration. Kubelet must serialize its configuration and
teardown operations for a sandbox; drivers must still tolerate unexpected runtime
failure or namespace disappearance. Linux namespace references must not be fabricated
for runtimes where the namespace is unavailable on the host.

Initially, namespace-dependent configuration is unsupported for host-network pods and
runtimes without the required context. Required configuration must produce a clear error
in these cases. Supporting host networking later requires an explicit safety contract,
since namespace modifications would affect the node.

### User namespace considerations

Pod user namespaces do not change RPC ordering, but affect the privileges needed to
enter and configure the network namespace. A namespace path alone grants no authority.
Drivers must have the capabilities required in the relevant user namespaces and must not
assume that workload UIDs match host UIDs.

Drivers must support their advertised operations for pods using user namespaces, or
return an explicit configuration error when unsupported. A host-privileged driver need
not enter the workload user namespace merely to configure its network namespace.
Operations that create files or change ownership may additionally require UID/GID
mapping information; adding that context is deferred until a concrete use case requires
it.

Configuration and cleanup tests must cover `hostUsers: false`, insufficient driver
privileges, and namespace disappearance. See the [Linux setns permission
rules](https://man7.org/linux/man-pages/man2/setns.2.html) and [Kubernetes user
namespace
documentation](https://kubernetes.io/docs/concepts/workloads/pods/user-namespaces/).

### Retries, partial failure, and recovery

Delivery is at least once. Drivers must make configuration idempotent for `(pod UID,
sandbox ID, claim UID)` and handle a retry after an unknown outcome. They must not
attach a device twice or overwrite unrelated configuration.

Kubelet tracks attempted configuration and completion per driver, claim, and sandbox in
a durable checkpoint. Cleanup intent is persisted before each configuration call, and
success is checkpointed before crossing the container creation barrier. After a crash, a
sandbox with incomplete or uncertain configuration is rolled back. A fully configured
sandbox can be reused after reconciliation; its completion does not authorize a
replacement sandbox.

Any required configuration failure fails the entire sandbox startup attempt. Failures
include RPC errors, timeouts, missing claim results, and per-claim errors. A transport
error or timeout does not establish whether the driver applied changes. No workload
container may be created in the failed sandbox, and kubelet must persist the rollback
decision before proceeding so a restart cannot accidentally reuse it.

The initial implementation treats sandbox creation and required DRA configuration as one
startup attempt:

```text
CRI RunPodSandbox succeeds
  -> NodeConfigurePodResources fails or times out
  -> NodeUnconfigurePodResources for every driver whose configuration was attempted
  -> CRI StopPodSandbox / RemovePodSandbox
  -> Retry sandbox creation and configuration with backoff
```

Rollback includes drivers that succeeded and the driver whose call failed or had an
unknown outcome. Drivers that were never called need no rollback. Cleanup must cover all
claims included in each attempted call, even when no successful response was received.
Cleanup attempts before sandbox shutdown are bounded; pending cleanup is retained for
reconciliation as described below.

This is analogous to a sandbox startup failure from the pod's perspective, but kubelet
must explicitly coordinate DRA rollback because CRI cannot undo driver side effects.
Prepared claims remain prepared while the pod still needs them. A failed sandbox attempt
does not itself trigger claim-level `NodeUnprepareResources`.

Kubelet must discover a sandbox after an uncertain `RunPodSandbox` result and reconcile
it before allowing workload container creation. Cleanup of an old attempt must not
affect a replacement sandbox. Startup retries may proceed only when overlapping resource
use is safe; unresolved cleanup must prevent reuse of affected resources.

A timed-out configuration call may continue executing in the driver. For the same
sandbox association, drivers must serialize configuration and unconfiguration, or
provide an equivalent durable reconciliation mechanism, so a late configuration
completion cannot recreate state after cleanup. Once unconfiguration has completed,
stale configuration calls for that sandbox must not reapply changes. The implementation
must specify persistence and garbage collection of this protection across driver
restarts.

Deletion cancels startup and proceeds with teardown. An unresponsive configuration RPC
must not indefinitely block pod termination.

### Teardown and claim sharing

Cleanup is a design requirement, not an assumption that namespace destruction undoes
every side effect. A driver can create host-side state, and claim-level
`NodeUnprepareResources` is too coarse to identify which sandbox association ended for a
shared claim. The [existing DRA
protocol](https://github.com/kubernetes/kubernetes/blob/master/staging/src/k8s.io/kubelet/pkg/apis/dra/v1/api.proto)
identifies claims in its prepare and unprepare requests.

The setup operation is paired with an idempotent `NodeUnconfigurePodResources` RPC
identifying the same pod, sandbox, and claims. Unconfiguration must succeed when
configuration was absent, partial, or already removed, unless remaining cleanup actually
fails. This gives sandbox-scoped side effects an explicit owner and lifetime.

Normal ordering should be:

```text
Stop workload containers
  -> NodeUnconfigurePodResources
  -> CRI StopPodSandbox / RemovePodSandbox
  -> NodeUnprepareResources when the claim is no longer needed on this node
```

Kubelet records cleanup intent before invoking configuration, including calls whose
outcome is uncertain. Cleanup is attempted for partial setup as well as success. Drivers
must tolerate the namespace already being gone, and must not detach resources still used
by another sandbox or pod.

Teardown attempts before sandbox stop are bounded. Failure must not prevent CRI from
stopping the sandbox and reclaiming its networking. Kubelet retains pending cleanup
records and retries namespace-independent cleanup afterward. Device reuse must be fenced
until safe cleanup is confirmed; the exact interaction with claim release and driver
reconciliation is an alpha implementation requirement.

Claim-level unprepare is not a substitute for sandbox-scoped rollback: a shared claim or
a claim needed by a replacement sandbox may still be in use.

### Open Questions for SIG Discussion

1. Should sandbox context be a prerequisite CRI KEP or part of this enhancement? Which namespace representation is portable and safe?
2. How should drivers persist and garbage-collect protection against late configuration calls after unconfiguration?
3. What capability and version negotiation mechanism integrates best with current DRA registration?
4. How does the checkpoint represent request selections, partial progress, and pending cleanup without duplicating claim state?
5. How are devices fenced against reuse while obsolete sandbox cleanup is pending?
6. How should sandbox-scoped device results be applied, merged with claim-preparation
results, and persisted without introducing container-level configuration semantics?
7. What namespace and sharing constraints should the Linux alpha explicitly enforce?

### Test Plan

- Verify setup happens after successful `RunPodSandbox` and before any workload `CreateContainer`, including init and ephemeral containers.
- Exercise timeout after successful driver work, duplicate calls, partial results, plugin restart, and kubelet crashes around every checkpoint transition.
- Verify configuration failure rolls back every attempted driver, prevents workload container creation, and retries with a new sandbox ID while retaining needed claim preparation.
- Verify late completion of a timed-out configuration call cannot recreate state after unconfiguration, including across driver restart.
- Verify an unconfiguration failure permits bounded sandbox shutdown while retaining cleanup intent and blocking unsafe device reuse.
- Reconcile an existing sandbox after kubelet restart and an uncertain sandbox creation outcome.
- Replace a sandbox and verify old completion cannot authorize containers in the new sandbox.
- Delete pods during configuration; verify bounded termination and eventual cleanup.
- Test multiple drivers, request selection, shared claims, and teardown of only one sharing pod.
- Validate namespace reference accessibility and lifetime with representative supported CRI runtimes.
- Verify explicit failures for unavailable context and host networking, and unchanged operation for legacy drivers.
- Demonstrate a Linux network driver completing namespace-dependent setup without an NRI plugin.
- Confirm cleanup errors cannot allow unsafe resource reuse or leak checkpoint records indefinitely.

#### Prerequisite testing updates

TBD with component owners. Existing test refactors and baseline coverage have not been
assessed.

#### Unit tests

Cover capability checks, response validation, checkpoint transitions, and rollback
decisions. Package coverage measurements are TBD.

#### Integration tests

Exercise kubelet, fake CRI, and driver interactions for ordering, partial results,
timeouts, restarts, and failed cleanup. Test locations and CI jobs are TBD.

#### e2e tests

Validate a Linux networking driver with supported runtimes, first-container device
availability, sandbox replacement, and compatibility scenarios. Implemented tests and CI
evidence are pending.

### Graduation Criteria

#### Alpha

Feature disabled by default behind a proposed `DRAPodSandboxConfiguration` kubelet gate.
Implement capability negotiation, typed runtime context, startup barrier, checkpoints,
retries, and sandbox-scoped cleanup. Provide a Linux networking reference driver and
automated lifecycle tests. Resolve the API and cleanup questions below before declaring
the KEP implementable.

#### Beta

Demonstrate interoperability across supported runtime implementations, operational
recovery under restart and replacement, documented upgrade behavior, bounded overhead,
and feedback from network driver maintainers. Agree on defaults and API stability based
on those results.

#### GA

Complete sustained production validation, compatibility and conformance expectations for
supported combinations, and resolution of remaining lifecycle issues. Target versions
are TBD.

### Upgrade / Downgrade Strategy

Upgrading must preserve existing behavior for drivers that do not opt in. Install
supporting runtime and driver versions, enable the kubelet feature gate, and restrict
dependent workloads to compatible nodes before using the feature.

Downgrading to a kubelet that does not support pod resource configuration requires
terminating dependent pods and completing sandbox-scoped cleanup before the downgrade.
Preserving running dependent pods across such a downgrade is not supported. An older
kubelet cannot invoke the new unconfiguration RPC or configure replacement sandboxes,
even when the driver still implements the older DRA interfaces.

The supported procedure is:

1. Stop placing dependent workloads on the node.
2. Drain dependent workloads while the supporting kubelet and driver are still running.
3. Complete unconfiguration and all pending sandbox cleanup before replacing those components. A failed cleanup blocks this supported downgrade procedure until resolved.
4. Downgrade kubelet and, if needed, the runtime; retain a DRA driver version compatible with the older kubelet's DRA API.
5. Keep workloads requiring this feature off the node until support is restored.

Workloads that do not use this feature retain existing behavior. Persisted cleanup
records must not be silently discarded. Checkpoint format compatibility and safeguards
against an unplanned downgrade remain implementation requirements. Add upgrade,
downgrade, and reupgrade tests, including failed cleanup and placement constraints,
before graduation.

This follows the compatibility principle in [KEP-4381's downgrade
strategy](https://github.com/kubernetes/enhancements/blob/master/keps/sig-node/4381-dra-structured-parameters/README.md#upgrade--downgrade-strategy):
drivers support interfaces used by older kubelets. This KEP additionally requires prior
sandbox cleanup because older interfaces do not provide the new sandbox lifecycle
operations.

### Version Skew Strategy

CRI context additions must use new protobuf field numbers and remain optional for
existing workloads. Older clients can ignore added fields; older runtimes can omit them.
A new kubelet with an existing driver retains existing behavior. Required configuration
fails explicitly when the runtime lacks needed context.

A driver must establish kubelet support before relying on the new RPC. An old kubelet
cannot enforce a hook it does not know about, so deployment constraints must prevent
dependent workloads from landing on unsupported nodes. DRA capability negotiation and
source compatibility for driver helper interfaces remain TBD. No control-plane
allocation or scheduling API change is currently proposed; automatic compatibility-aware
placement is not defined by this draft.

## Production Readiness Review Questionnaire

Preliminary answers follow. Individual questions marked TBD require completion and
review before implementable status.

### Feature Enablement and Rollback

Enablement requires the proposed kubelet feature gate plus explicit driver and runtime
support. Existing drivers retain their current behavior. To disable support, stop
placing dependent workloads on affected nodes and terminate or migrate those workloads
first. Replacement sandboxes still require configuration. Downgrade to an unsupported
kubelet requires terminating dependent pods and completing cleanup as described above.
Exact behavior for reenablement, checkpoint compatibility, and feature-gate transition
tests is TBD.

###### How can this feature be enabled / disabled in a live cluster?

TBD. See the preliminary answer above; detailed validation is pending.

###### Does enabling the feature change any default behavior?

TBD. See the preliminary answer above; detailed validation is pending.

###### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

TBD. See the preliminary answer above; detailed validation is pending.

###### What happens if we reenable the feature if it was previously rolled back?

TBD. See the preliminary answer above; detailed validation is pending.

###### Are there any tests for feature enablement/disablement?

TBD. See the preliminary answer above; detailed validation is pending.

### Rollout, Upgrade and Rollback Planning

Roll out runtime and driver support before enabling dependent workloads on compatible
nodes. Monitor startup failures, configuration latency, and pending cleanup. No existing
API removal is proposed. Upgrade, downgrade, and reupgrade validation has not been
performed; the implementation and associated tests are pending.

###### How can a rollout or rollback fail? Can it impact already running workloads?

TBD. See the preliminary answer above; detailed validation is pending.

###### What specific metrics should inform a rollback?

TBD. See the preliminary answer above; detailed validation is pending.

###### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?

TBD. See the preliminary answer above; detailed validation is pending.

###### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?

TBD. See the preliminary answer above; detailed validation is pending.

### Monitoring Requirements

Proposed signals include low-cardinality RPC latency/error metrics and
pending-configuration/cleanup counts, plus pod events identifying the driver and stage.
Keep pod, claim, and sandbox UIDs in diagnostic logs rather than metric labels. Metric
names, event reasons, per-pod success visibility, and SLOs are TBD.

###### How can an operator determine if the feature is in use by workloads?

TBD. See the preliminary answer above; detailed validation is pending.

###### How can someone using this feature know that it is working for their instance?

TBD. See the preliminary answer above; detailed validation is pending.

###### What are the reasonable SLOs (Service Level Objectives) for the enhancement?

TBD. See the preliminary answer above; detailed validation is pending.

###### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?

TBD. See the preliminary answer above; detailed validation is pending.

###### Are there any missing metrics that would be useful to have to improve observability of this feature?

TBD. See the preliminary answer above; detailed validation is pending.

### Dependencies

Dependent workloads require a supporting DRA driver and a CRI runtime exposing usable
sandbox context. Driver unavailability or missing context prevents startup. Cleanup
failures can delay safe resource reuse. Runtime failure may destroy the namespace before
driver cleanup; drivers must support cleanup after namespace loss. Minimum versions and
the context capability contract are TBD.

###### Does this feature depend on any specific services running in the cluster?

TBD. See the preliminary answer above; detailed validation is pending.

### Scalability

Work and checkpoint state scale with participating pod-sandbox-driver-claim
associations. Measure startup latency, checkpoint size, and recovery under
representative node density and driver outages. Numerical limits, benchmark results, and
resource overhead are TBD.

###### Will enabling / using this feature result in any new API calls?

TBD. See the preliminary answer above; detailed validation is pending.

###### Will enabling / using this feature result in introducing new API types?

TBD. See the preliminary answer above; detailed validation is pending.

###### Will enabling / using this feature result in any new calls to the cloud provider?

TBD. See the preliminary answer above; detailed validation is pending.

###### Will enabling / using this feature result in increasing size or count of the existing API objects?

TBD. See the preliminary answer above; detailed validation is pending.

###### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?

TBD. See the preliminary answer above; detailed validation is pending.

###### Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components?

TBD. See the preliminary answer above; detailed validation is pending.

###### Can enabling / using this feature result in resource exhaustion of some node resources (PIDs, sockets, inodes, etc.)?

TBD. See the preliminary answer above; detailed validation is pending.

### Troubleshooting

Inspect pod events for the failing driver and stage, then correlate kubelet and driver
logs by pod, sandbox, and claim identity. Recovery reconciles persisted associations
against active sandboxes and desired pods, rolls back incomplete or uncertain
configuration, and retries pending cleanup. Operator procedures and diagnostic examples
require implementation validation.

###### How does this feature react if the API server and/or etcd is unavailable?

TBD. See the preliminary answer above; detailed validation is pending.

###### What are other known failure modes?

TBD. See the preliminary answer above; detailed validation is pending.

###### What steps should be taken if SLOs are not being met to determine the problem?

TBD. See the preliminary answer above; detailed validation is pending.

## Implementation History

- 2026-10-08: Initial discussion draft. No enhancement issue, upstream PR, or implementation submitted.
- 2026-10-23: Adopted paired configuration/unconfiguration RPCs and rollback of the sandbox startup attempt on configuration failure, including partial success and unknown outcomes.

## Drawbacks

The new startup barrier adds latency and makes participating driver availability a
prerequisite for container creation. Failed attempts recreate sandboxes and repeat
primary network setup. Durable rollback and cleanup tracking add kubelet and driver
complexity, and network namespace access requires supporting runtime implementations.

## Alternatives

**Continue using NRI.** Reuses existing runtime lifecycle integration, but keeps the
second integration for drivers that need only a DRA sandbox configuration stage.
Drivers must correlate runtime callbacks with prepared claims and coordinate recovery
across DRA and NRI instead of using a kubelet-managed DRA operation.

**Call `NodePrepareResources` a second time after sandbox creation.** Blurs claim and
sandbox lifetimes and does not supply pod identity or namespace context in the current
interface.

**Use a CNI integration.** Appropriate for primary network provisioning and work needed
before sandbox creation succeeds. DRA configuration after `RunPodSandbox` cannot satisfy
a CNI operation that itself requires the DRA setup to complete first.

**Use CDI hooks.** Can move network devices into the pod network namespace, but ties
the operation to container OCI hooks and requires publishing device specifications
and deploying hook binaries. Basic interface setup may suffice for simple cases;
more complete network provisioning still needs driver-specific orchestration in the
hook or another component. The integration must reconcile container hook execution
with pod-wide setup, retries, and sandbox-scoped cleanup. A dedicated DRA RPC makes
that coordination explicit between kubelet and the resource-owning driver.

**Expose a generic kubelet hook framework.** Broader than the identified need and
introduces a larger policy and lifecycle surface.

## Infrastructure Needed

No new shared infrastructure is currently proposed. Runtime interoperability CI and a
reference driver will need test environments; ownership and provisioning are TBD.
