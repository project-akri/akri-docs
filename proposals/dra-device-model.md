# DRA as Akri's Device Model

This document discusses adopting [Dynamic Resource Allocation (DRA)][1] as the way Akri models
and offers devices to Kubernetes. Today Akri uses the device plugin framework. DRA went
[GA in Kubernetes 1.34][2] and is where device management in Kubernetes is heading.

Adopting DRA is not a new idea for Akri. It was proposed in September 2023 and written up as
[akri-docs#88][3]. Through 2024 it was deferred behind the agent refactor and the configuration
split. The last recorded discussion was November 2024.

Two things have changed since then. The DRA design Akri planned against no longer exists, and
the node locality problem raised with the DRA maintainers now has an answer in the API. This
document revisits the plan against the DRA that shipped.

It supersedes [akri-docs#88][3]. It does not supersede [akri-docs#106][4] (split configuration)
or [akri-docs#75][5] (arbitrary broker resources), which contain no DRA-specific content and
remain valid.

## Background

Akri models a discovered device as follows:

- **Configuration**: a namespaced `akri.sh/v0` resource naming a Discovery Handler and its discovery details.
- **Agent**: a per-node DaemonSet that drives Discovery Handlers. For each discovered device it creates an `Instance` and registers a device plugin with the kubelet, using device plugin API `v1beta1`.
- **Resource**: devices are advertised as an extended resource named `akri.sh/<config>`. Sharing is bounded by `capacity` and tracked in the `Instance.device_usage` slot map.
- **Broker**: a workload requests the device with an integer resource limit.

A workload requests a camera today like this:

```yaml
resources:
  requests:
    akri.sh/onvif-camera-8120fe: "1"
```

On `main` the Agent already builds a CDI device for each discovered device, exposed as
`Instance.cdi_name`, while still advertising through the device plugin API.

The device plugin API limits Akri in four ways:

- **Devices are counted, not described**: Discovery Handlers report properties such as resolution, codec, protocol and location. These can only become broker environment variables. They are not selectable. A user cannot request an ONVIF camera in building-A that supports H.264. They can only request one `akri.sh/onvif`.
- **Resources are node-local**: KEP-4381 names Akri here. Projects like Akri "have to work around that by reporting the same network-attached resource on all nodes that it could get attached to and then updating resource availability on all of those nodes when resources get used".
- **Sharing is hand-rolled**: `capacity` and `device_usage` are reconciled by Akri itself, including when a node disappears.
- **Claims carry no parameters**: there is no first-class way for a request to carry device-specific configuration.

DRA addresses all four.

## Prior Work

The design and its history are recorded in the developer meeting notes and in three proposal
pull requests, all still open.

| Date       | What happened                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| ---------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 2023-09-05 | DRA proposed as the successor to the device plugin model. The design introduced `BrokerTemplate`, `DiscoveryConfiguration` and `PropertyFilter`, a new controller alongside a rename of the existing one, and an agent that registers one resource plugin per Discovery Handler. The notes also record the intent to "get involved with SIG and Intel folks that started this KEP and have Akri included as a reference implementation", and that "breaking up the configuration would help this as well and lead us on a path to DRA". Participants included Nicolas Belouin, Kate Goldenring, Johnson Shih and Yu Jin Kim. |
| 2023-08-31 | The design written up as [akri-docs#88][3], building on [akri-docs#75][5].                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| 2023-12-07 | The configuration split written up as [akri-docs#106][4].                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| 2024-01-09 | Sequencing agreed. The agent refactor and the configuration split were "must have" prerequisites.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| 2024-02-06 | "first split config, get merged then get DRA one merged".                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| 2024-03-05 | A DRA-mode prototype hit a "bug with deallocating resources with DRA mode" where "kubelet is crashing". DRA was judged unable to land soon.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| 2024-11-05 | The direction was discussed with the DRA maintainers. They were "focused on the GPU story" and on "linking the device to the node". The idea floated was "maybe we could change the node field to be a node selector".                                                                                                                                                                                                                                                                                                                                                                                                       |

DRA does not appear on the [published roadmap][6], which still lists only additional Discovery
Handlers and broker deployment strategies. Putting it back on the roadmap is part of what this
document asks for.

## What Changed in DRA

The 2023 design predates structured parameters, which reached GA in 1.34. Most of what stopped
Akri has been resolved upstream.

| Blocker                                                                                                                                 | Status in `resource.k8s.io/v1`                                                                                                                                                                                                                         |
| --------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| A vendor control plane controller was required for allocation and node selection. Akri's prototype crashed the kubelet on deallocation. | The controller was removed before GA. The scheduler allocates directly from published `ResourceSlice` objects. `PodSchedulingContext`, the type that handshake used, no longer exists. The deallocation path that crashed the kubelet is gone with it. |
| Devices are network-attached, not node-local. DRA looked node-tied and GPU-centric.                                                     | Resolved, and in GA. `ResourceSliceSpec` carries `nodeName`, `nodeSelector` and `allNodes`, none of them feature-gated. The node selector idea from November 2024 shipped.                                                                             |
| A `PropertyFilter` resource was needed to filter instances on a claim.                                                                  | Not needed. CEL selectors on `DeviceClass` and on claim requests select over device attributes directly.                                                                                                                                               |
| A `ResourceClass` carried driver parameters.                                                                                            | Replaced by `DeviceClass`.                                                                                                                                                                                                                             |
| `capacity` and the `device_usage` slot map were hand-rolled sharing.                                                                    | Partly addressed. `Device.capacity` is GA. Allowing several claims to share one device needs `allowMultipleAllocations`, which is alpha. This is the least settled part of the story.                                                                  |
| The configuration split was a prerequisite.                                                                                             | Still true and still unmerged. [akri-docs#106][4] remains the first step.                                                                                                                                                                              |

The result is that GA DRA is smaller than the design Akri planned against. It removes one
controller and one custom resource, and it replaces the slot map with API we do not maintain.

### Feature Maturity

Not all of this is GA. As of Kubernetes 1.37:

- **GA**: `ResourceSlice`, `DeviceClass`, `ResourceClaim`, `ResourceClaimTemplate`, CEL device selectors, `Device.capacity`, and the `nodeName`, `nodeSelector` and `allNodes` scoping that resolves the node locality problem.
- **Beta in 1.37**: `sharedCounters` for partitionable devices, and `extendedResourceName` on `DeviceClass`.
- **Alpha**: `allowMultipleAllocations` and consumable capacity, `partitionTypeAttribute`, `skipNodeOperations`, `perDeviceNodeSelection`.

Everything the core of this proposal depends on is GA. Device sharing is the exception. A first
DRA implementation would either restrict itself to exclusive device allocation or take an opt-in
dependency on an alpha gate.

## Terms

| Term                    | Definition                                                                                                                                                                      |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `DeviceClass`           | A cluster-scoped category of devices that a claim can select from. Carries CEL selectors and driver configuration.                                                              |
| `ResourceSlice`         | A published inventory of devices and their attributes, authored by the driver. Scoped by node name, node selector, or all nodes.                                                |
| `ResourceClaim`         | A workload's request for one or more devices, selected by CEL over device attributes.                                                                                           |
| `ResourceClaimTemplate` | A template from which a `ResourceClaim` is generated per pod.                                                                                                                   |
| Kubelet plugin          | The per-node driver component the kubelet calls to prepare and unprepare devices. This is the DRA analogue of the device plugin. It is not the same thing as CSI's node plugin. |
| Structured parameters   | The GA DRA model, in which the scheduler allocates from published attributes and no vendor controller is required.                                                              |

## Proposed Direction

### Publishing devices

The Agent publishes what a Discovery Handler finds as a `ResourceSlice`. Discovery Handler
properties become device attributes. Network-attached cameras reachable from two nodes are
published once, with a node selector, rather than duplicated per node.

Attributes are kept uniform across the devices in a pool. A selector that references an
attribute some devices do not carry will not evaluate against them.

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceSlice
metadata:
  name: akri-onvif-0
spec:
  driver: onvif.akri.sh
  pool:
    name: akri-onvif
    generation: 1
    resourceSliceCount: 1
  nodeSelector:
    nodeSelectorTerms:
      - matchExpressions:
          - key: kubernetes.io/hostname
            operator: In
            values: [node-1, node-2]
  devices:
    - name: camera-8120fe
      attributes:
        ipAddress:
          string: "10.0.0.12"
        resolution:
          string: "1920x1080"
        codec:
          string: "H.264"
        location:
          string: "building-a"
        firmware:
          version: "2.4.1"
    - name: camera-4d91c7
      attributes:
        ipAddress:
          string: "10.0.0.17"
        resolution:
          string: "3840x2160"
        codec:
          string: "H.265"
        location:
          string: "building-a"
        firmware:
          version: "3.1.0"
    - name: camera-a7f302
      attributes:
        ipAddress:
          string: "10.0.1.23"
        resolution:
          string: "1280x720"
        codec:
          string: "H.264"
        location:
          string: "building-b"
        firmware:
          version: "2.4.1"
```

### Selecting devices

A `DeviceClass` groups everything a Discovery Handler publishes.

```yaml
apiVersion: resource.k8s.io/v1
kind: DeviceClass
metadata:
  name: akri-onvif
spec:
  selectors:
    - cel:
        expression: device.driver == "onvif.akri.sh"
```

A workload then selects on attributes. This is the request that is not expressible today.

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaimTemplate
metadata:
  name: onvif-building-a
spec:
  spec:
    devices:
      requests:
        - name: camera
          exactly:
            deviceClassName: akri-onvif
            selectors:
              - cel:
                  expression: |
                    device.attributes["onvif.akri.sh"].location == "building-a" &&
                    device.attributes["onvif.akri.sh"].codec == "H.264"
```

Of the three cameras above, only `camera-8120fe` matches. `camera-4d91c7` is in the right
building but reports H.265, and `camera-a7f302` reports H.264 from the wrong building. Today
all three are interchangeable behind a single `akri.sh/onvif` resource.

### Mapping

| Akri today                              | DRA                                                                                                                   |
| --------------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| `Configuration`                         | Split per [akri-docs#106][4]. `DiscoveryConfiguration` stays Akri-side and informs one or more `DeviceClass` objects. |
| `Instance`                              | A device entry in a `ResourceSlice`, with Discovery Handler properties as attributes.                                 |
| Per-device device plugin server         | A DRA kubelet plugin that publishes slices and handles `NodePrepareResources` and `NodeUnprepareResources`.           |
| `capacity` and `device_usage`           | `Device.capacity`, and `allowMultipleAllocations` where sharing is needed.                                            |
| Device reported on every candidate node | One `ResourceSlice` with a node selector.                                                                             |
| `akri.sh/<config>: "1"`                 | A `ResourceClaim` selecting on attributes. Prepare returns the CDI device IDs Akri already computes.                  |
| Controller broker and Service lifecycle | Largely unchanged. Workloads reference claims instead of resource limits.                                             |

Because the Agent already produces CDI devices, returning CDI IDs from `NodePrepareResources` is
a short step from `main`.

We propose `WorkloadConfiguration` rather than the `BrokerTemplate` of [akri-docs#88][3],
following [akri-docs#106][4]. The reasoning there still holds. What Akri schedules "can be much
more than a mediator", so broker is the wrong word.

## Implementation Options

A DRA driver needs kubelet plugin registration, `ResourceSlice` publishing and reconciliation,
the prepare and unprepare gRPC lifecycle, claim fetching, and API version negotiation.
Kubernetes ships [`k8s.io/dynamic-resource-allocation`][7] for this. It is Go. Akri is Rust, and
no equivalent exists.

Three options:

- **Write the driver in Go**: uses the official library, which is maintained upstream and tracks DRA API changes for us. It makes Akri a two-language project. The driver also needs discovery results, and those live in the Rust Agent, so this means either a second component with an interface between them or moving discovery into Go.
- **Hand-roll the plumbing in the Agent**: keeps Akri in one language and adds no dependency. Akri then owns registration, slice reconciliation, the gRPC lifecycle and version negotiation, and tracks DRA API evolution itself, indefinitely.
- **Build on a Rust DRA library**: the same as the previous option, with the plumbing factored into a crate instead of living in the Agent. Akri stays Rust and stops carrying that code. I have been prototyping [`kube-dra`][8] for this, and it is currently the only Rust DRA library that exists. It is at an early stage and still a work in progress.

## Migration

- **Additive first**: DRA as an alternative device exposure path, opt-in per `DiscoveryConfiguration`. The device plugin path stays unchanged and remains the default.
- **Version floor**: `resource.k8s.io/v1` is GA in 1.34. DRA support is conditional on cluster version, and the device plugin path stays for clusters below the floor.
- **A possible bridge**: `DeviceClass.spec.extendedResourceName` lets a `DeviceClass` satisfy a pod's extended resource request. Akri already advertises `akri.sh/<config>` as an extended resource. Existing manifests may keep working unchanged while the backend moves to DRA. The gate is beta in 1.37. This is worth validating early, since it would avoid a hard cutover to claims.
- **One handler first**: prove the model end-to-end on debug-echo, which needs no hardware, before broadening.
- **No forced deprecation**: this document does not propose removing the device plugin path.

## Non-goals

- Deprecating or removing the device plugin path.
- Committing Akri to a specific driver implementation or library.
- A complete design. This is a direction to agree on first.

## Open Questions

1. Do we want DRA to become Akri's primary device model, and on what timeline?
2. Which implementation option do we take? Is there appetite to co-invest in a Rust DRA library the wider ecosystem could reuse?
3. Does `Instance` survive? [akri-docs#88][3] kept it because "the Instance resource is all about available resource tracking that is explicitly not covered by DRA". That is no longer true, since `ResourceSlice` is that tracking. If `Instance` shrinks or disappears, what does the workload controller watch?
4. How do `WorkloadConfiguration` and `ResourceClaim` compose? Does Akri generate claims on the user's behalf, or does the user reference them directly?
5. Do we accept exclusive allocation at first, or take a dependency on the alpha consumable capacity gate to preserve `capacity` semantics?
6. Is the DRA scheduler and kubelet path acceptable on resource-constrained edge nodes?
7. Do we still want the 2023 intent of Akri as a DRA reference implementation, now that KEP-4381 already cites Akri as motivation?

## References

- KEP-4381 names Akri in its motivation and lists "Support node-local and network-attached resources" as a goal. [KEP-4381][9]
- Field maturity above is taken from the `+featureGate=` and `+k8s:beta(since:)` markers in [`resource/v1/types.go`][10].
- Akri implementation details: device plugin `v1beta1` in `agent/proto/pluginapi.proto` and `agent/src/plugin_manager/`, `device_usage` and `cdi_name` in `shared/src/akri/instance.rs`, CDI in `agent/src/device_manager/cdi.rs`, and `API_VERSION` in `shared/src/akri/mod.rs`.
- Meeting notes: [archived][11], [current][12].

[1]: https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/
[2]: https://kubernetes.io/blog/2025/09/01/kubernetes-v1-34-dra-updates/
[3]: https://github.com/project-akri/akri-docs/pull/88
[4]: https://github.com/project-akri/akri-docs/pull/106
[5]: https://github.com/project-akri/akri-docs/pull/75
[6]: https://docs.akri.sh/community/roadmap
[7]: https://github.com/kubernetes/dynamic-resource-allocation
[8]: https://github.com/nubicle/kube-dra
[9]: https://github.com/kubernetes/enhancements/blob/master/keps/sig-node/4381-dra-structured-parameters/README.md
[10]: https://github.com/kubernetes/kubernetes/blob/master/staging/src/k8s.io/api/resource/v1/types.go
[11]: https://hackmd.io/@akri/rJED-1EB6
[12]: https://hackmd.io/UUqW3_GgQDimQ5b23rS9rg
