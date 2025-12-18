# Configuration-level Resources

This document discusses the motivation and potential implementation for supporting "Configuration-level resources" in Akri. Currently, Akri only supports creating a Kubernetes resource (i.e. device plugin) for each individual device. Since each device in Akri is represented as an Instance custom resource, these are called Instance-level resources. A Configuration-level resource is a resource that represents all of the devices discovered via a Configuration, i.e. all IP cameras.

Currently, with only Instance-level resources, specific devices must be [requested as resources](../docs/user-guide/requesting-akri-resources.md). For example, a Pod could be deployed that uses both `ip-camera-UNIQUE_ID1` and `ip-camera-UNIQUE_ID2` by adding the following resource request to the `PodSpec`:

```yaml
resources:
  requests:
    akri.sh/onvif-camera-UNIQUE_ID1: "1"
    akri.sh/onvif-camera-UNIQUE_ID2: "1"
```

With Configuration-level resources, instead of needing to know the specific Instances to request, two generic cameras could be requested and the Agent will behind the scenes do the work of selecting which Instances to reserve:

```yaml
resources:
  requests:
    akri.sh/onvif-camera: "2"
```

Furthermore, with Configuration-level resources, users could develop their own deployment strategies rather than relying on the Akri Controller to deploy Pods to discovered devices. Higher level Kubernetes objects (Deployments, ReplicaSets, DaemonSets, etc.) can be deployed requesting resources, and Pods will only be run on Nodes that have those resources available.

## Terms

| Term                              | Definition                                                                                                                                                                                                                                                                                                                                                                                                   |
| --------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `capacity`                        | Maximum amount of containers that can use a device. Defines the number of usage slots a device's device plugin advertises to the kubelet                                                                                                                                                                                                                                                                     |
| Configuration-level (CL) resource | An extended resource that is created by the Agent for each Configuration. This equates to the creation of a Kubernetes device plugin for each Configuration. It enables requesting any of a set of (Configuration) resources, such as any IP camera, thermometer, etc. Resources are given the Configuration name. A request to use a CL resource is mapped to an IL resource by the Agent.                  |
| Instance-level (IL) resource      | An extended resource that is created by the Agent for each discovered device (i.e. Instance), such as a specific IP camera, thermometer, etc. This equates to the creation of a Kubernetes device plugin for each Instance. Resources are named in the format <Configuration Name>-<HASH>, where the input to the one-way hash is a descriptor of the Instance (and node name for local non-shared devices). |

## Implementation

To support Configuration-level resources, the Agent will create an additional device plugin for each
Configuration (that initiates discovery). No changes are needed to the Controller or Discovery Handlers.

The device plugin created for each Configuration will contain `capacity` \* `number of
instances` slots. Each slot will map to a "real" slot of an Instance device plugin. For example,
after deploying a `onvif` Configuration with a `capacity` of `2`, the `NodeSpec` of a node that
could see both cameras would be:

```yaml
Capacity:
  akri.sh/akri-onvif: 4
  akri.sh/akri-onvif-8120fe: 2
  akri.sh/akri-onvif-a19705: 2
```

Now, a Pod could be deployed that requests 2 cameras by requesting 2 `akri.sh/akri-onvif` resources.

```yaml
apiVersion: "apps/v1"
kind: Pod
metadata:
  name: onvif-broker
spec:
  containers:
    - name: nginx
      image: "nginx:latest"
      resources:
        requests:
          "akri.sh/akri-onvif": "2"
        limits:
          "akri.sh/akri-onvif": "2"
```

The Kubernetes scheduler will schedule the Pod to the Node since the resources is available.
The kubelet on that Node will make an `allocate` call to the `akri.sh/akri-onvif` device plugin in the Akri Agent.
The Agent will then reserve slots in the associated Instance device plugins (`akri.sh/akri-onvif-8120fe` and `akri.sh/akri-onvif-a19705`).
When selecting the slots from Instances, the Agent will preference unique Instances; for example, in the above Pod, the Agent will make sure 1 slot of each `akri-onvif-8120fe` and `akri-onvif-a19705` is reserved for each Pod.

There are two implementation options in the case where there are not enough unique Instances to meet the requested number of Configuration-level resources. One scenario where this could happen is if a Pod requests 3 cameras when only 2 exist with a capacity of 2 each. In this case, the Configuration-level camera resource shows as having a quantity of 4, despite there being two cameras. In this case, the kubelet will try to schedule the Pod, thinking there are enough resources. The Agent could either allocate 2 spots on one camera and one on the other or deny the allocation request. The latter is the preferred approach as it is more consistent and ensures the workload is getting the number of unique devices it expects. After failing an `allocate` request from the kubelet, the Pod will be in an `UnexpectedAdmissionError` state until another camera comes online and it can be successfully scheduled.

The Agent implements the latter approach that denies `allocate` requests that request more than available cameras.
It manages the number of available slots reported to kubelet to prevent kubelet from scheduling Pod that requests invalid number of resources.

### Device Plugin behavior

The Agent exposes Configuration-level and Instance-level resources to kubelt as device plugins. The CL and IL device plugins have
different behaviors but use almost the same set of information. Originally, the IL device plugin is implemented in
the `DevicePluginService` class. To support Configuration-level resources, the behavior part of device plugin is extracted out
from `DevicePluginService` and introduce the `ConfiguratinDevicePlugin` and `InstanceDevicePlugin` classes to represent the behavior of
CL and IL device plugin. The `DevicePluginService` holds the common data for device plugins to operate and
a `device_plugin_behavior` field for the desired behavior (CL or IL).
CL/IL specific data and behavior is defined in `ConfiguratinDevicePlugin` and `InstanceDevicePlugin` class.

The Rust code in Agent for `DevicePluginService`, `ConfiguratinDevicePlugin` and `InstanceDevicePlugin` look like below

```yaml
pub enum DevicePluginBehavior {
    Configuration(ConfigurationDevicePlugin),
    Instance(InstanceDevicePlugin),
}


pub struct DevicePluginService {
    /// Instance CRD name
    pub instance_name: String,
    /// Instance's Configuration
    pub config: ConfigurationSpec,

    /// ... data used by both CL and IL device plugin

    /// Enum object that defines the behavior of the device plugin
    pub device_plugin_behavior: DevicePluginBehavior,
}
```

### Configuration Device Plugin

The `ConfiguratinDevicePlugin` defines the behavior of the configuration-level resources, combined with `DevicePluginService`, it forms a device plugin that advertises the configuration-level resources to the kubelet.

Similar to the Agent creates a `DevicePluginService` with `InstanceDevicePlugin` behavior for each discovered Instance. The Agent creates a
`DevicePluginService` with `ConfigurationDevicePlugin` behavior for each Configuration when the Configuration is applied to the cluster. All
`DevicePluginService` share the same structure that has `list_and_watch` and `allocate` for kubelet to call. The actual behavior of
`list_and_watch` and `allocate` is defined in `ConfigurationDevicePlugin` and `InstanceDevicePlugin` for Configuration and Instance respectively.
The CL and IL device plugin need to coordinate to sync up available resources so when a resource is claimed by CL (or IL) device plugin, the
other device plugin is notified and re-calculate the available resources for itself. The `list_and_watch_message_sender` in
`DevicePluginService` is used for notifying available resource change within a device plugin. For notifying resource change across device
plugins, a copy of `list_and_watch_message_sender` for each `DevicePluginService` are saved in `InstanceConfig`, where
`usage_update_message_sender` holds the message sender of CL device plugin and IL device plugin's `list_and_watch_message_sender` is saved in
each instance's `InstanceInfo`.
For IL device plugin, the `list_and_watch` is notified when available IL resource changed, which include

- the instance is deleted
- an IL allocate failed
- a CL allocate succeeded.

For CL device plugin, the `list_and_watch` is notified when

- the configuration is deleted
- an instance slot is added or deleted
- a CL allocate failed
- an IL allocate succeeded

The following diagram shows the notification flows for the device plugins.

![](../media/list-and-watch-notification.svg)

### Configuration Device Plugin dynamic virtual device ids

The Instance device plugins report available resources using fixed virtual device id. Instance device plugin construct the virtual device id by
appending an index to the Instance name. For example, for an Instance device plugin, if the instance name is `akri.sh/akri-onvif-8120fe` and
the `capacity` is 2, Instance device plugin reports 2 virtual devices `akri-onvif-8120fe-0` and `akri-onvif-8120fe-1`. This works fine for
Instance device plugin but is not flexible for Configuration device plugin as we want to minimize the possibility that kubelet issues `allocate`
requests that request more than available cameras. Instead of using fixed virtual device id, Configuration device plugins expose available
resources using dynamic virtual device ids to provide maximum device usage flexibility.

Using the same example that two cameras are discovered for a Configuration (`akri.sh/akri-onvif-8120fe` and `akri.sh/akri-onvif-a19705`) and the
`capacity` is 2. The Configuration device plugin calculates available resources and, instead of reporting fixed id like `akri-onvif-8120fe-0`,
the Configuration device plugin reports virtual device id "0", "1", ... as 'place holder' in `list_and_watch`. The actual device slot to be used
is determined when `allocate` is called.

To avoid kubelet issues `allocate` requests that requests 3 cameras when only 2 cameras exist with a capacity of 2 each. The Configuration
device plugin only report resources already being claimed by the Configuration device plugin and +1 for each camera that still has at least one
free slot. For our example, if all slots from 2 cameras are free, the Configuration device plugin reports 2 virtual devices "0" and "1" even
there are actually 4 slots available to use. In a different case, if `akri-onvif-8120fe-1` had been claimed by the Configuration device plugin
as virtual device id "4" and all other 3 slots are free, the Configuration device plugin reports virtual device id "0", "1", and "4" as
available virtual device ids. The virtual device id "4" maps to `akri-onvif-8120fe-1` and id "0" and "1" can be mapped to
`akri-onvif-8120fe-0`, `akri-onvif-a19705-0`, or `akri-onvif-a19705-1` later when `allocate` is called. By managing the number of available
virtual devices, we can reduce the chances that kubelet issues `allocate` requests more than available cameras exist.

Note that there is still chances that all slots have been claimed by Configuration device plugin. For example, "0": `akri-onvif-8120fe-0`, "1":
`akri-onvif-8120fe-1`, "2":`akri-onvif-a19705-0`, "3":`akri-onvif-a19705-1`. In this case, Configuration device plugin reports all 4 virtual
devices are available, and it's possible that kubelet requests "0" and "1" which map to the same camera. In that case, the Configuration device
plugin denies the allocation request and the Pod will be in an `UnexpectedAdmissionError` state. The Configuration device plugin calculates
virtual device availability periodically and reduces the number of available virtual devices when the reconciler detects slots not in-use and
set the slot usage to free. In our example, assume "3":`akri-onvif-a19705-1` is the only slot being used. The reconciler set slots
`akri-onvif-8120fe-0`,
`akri-onvif-8120fe-1`, and `akri-onvif-a19705-0` to free. The Configuration device plugin then reduces the device availability to "0", "1" and "3".
If kubelet retries to claim "0" and "1", the Configuration device plugin will allow it by mapping "0" to `akri-onvif-8120fe-0` or `akri-onvif-8120fe-1`, "1" to `akri-onvif-a19705-0`.

The Configuration device plugin reports "0", "1", ... as virtual device ids in `list_and_watch` and determines the actual device slot to be used
when `allocate` is called. The algorithm to map virtual device ids to actual device slot works on the allocation requests on a per-container
basis that:

- For a given container request, ensure allocated devices are unique instances
- If a virtual device id was claimed before, use the previous allocation information.
- If a virtual device id has being claimed, pick a device slot from the instance that has the most free slots.

For example, assume "3":`akri-onvif-a19705-1` is the only slot being used and other slots are all free. Configuration device plugin
`list_and_watch` reports virtual device id "0", "1" and "3" are available (1 free from `akri-onvif-8120fe`, 1 free from `akri-onvif-a19705` and
1 previously claimed from `akri-onvif-a19705`).

- If kubelet requests "0" and "3" in a container request of an `allocate` call, the mapping is "0": `akri-onvif-8120fe-0` or
  `akri-onvif-8120fe-1` (since `akri-onvif-8120fe` has the most free slots between instance `akri-onvif-8120fe` (2) and `akri-onvif-a19705` (1))
  and "3":`akri-onvif-a19705-1` (previously claimed).
- If kubelet requests "0", "1" and "3" in a container request of an `allocate` call, Configuration device plugin denies the request as there is
  only 2 cameras available to allocate.
- If kubelet request "0" and "1" in a container request and "3" in a different container request of an `allocate` call, the Configuratin device
  plugin maps "0": `akri-onvif-8120fe-0` or `akri-onvif-8120fe-1` (the most free slots) and "1":`akri-onvif-a19705-0` (unique instance) to
  container request 1 and "3":`akri-onvif-a19705-1` (previously claimed) to container request 2.

### Maintaining Device Usage

The `Instance.deviceUsage` in Akri Instances and Akri slot annotation are extended to support Configuration device plugin.
The `Instance.deviceUsage` may look like this:

```yaml
deviceUsage:
  my-resource-00095f-0: ""
  my-resource-00095f-1: ""
  my-resource-00095f-2: ""
  my-resource-00095f-3: "node-a"
  my-resource-00095f-4: ""
```

where empty string means the slot is free and non-empty string indicates the slot is used (by the node). To support Configuration device plugin,
we need a way to distinguish if a slot is used by a Configuration device plugin or Instance device plugin. The `Instance.deviceUsage` format is
extended hold the additional information, the deviceUsage can be a "<node_name>" (for Instance) or a "C:<virtual_device_id>:<node_name>" (for
Configuration). For example, the `Instance.deviceUsage` shows the slot `my-resource-00095f-2` is used by virtual device id "0" of the
Configuration device plugin on `node-b`. The slot `my-resource-00095f-3` is used by Instance device plugin on `node-a`. The other 3 slots are
free.

```yaml
deviceUsage:
  my-resource-00095f-0: ""
  my-resource-00095f-1: ""
  my-resource-00095f-2: "C:0:node-b"
  my-resource-00095f-3: "node-a"
  my-resource-00095f-4: ""
```

The format of Akri slot annotation is extended in a similar manner, too. The Akri slot annotation looked like this:

```yaml
"akri.agent.slot-my-resource-00095f-3": "my-resource-00095f-3"
```

To support Configuration device plugin, the Akri slot annotation can be a "<slot_name>" (for Instance) or a "C:<virtual_device_id>:<slot_name>" (for Configuration). The example below shows the pod uses the slot `my-resource-00095f-2` claimed by virtual device id "0" of the Configuration device plugin. The pod also uses the slot `my-resource-00095f-3` which is claimed by Instance device plugin.

```yaml
"akri.agent.slot-my-resource-00095f-2": "C:0:my-resource-00095f-2"
"akri.agent.slot-my-resource-00095f-3": "my-resource-00095f-3"
```

## Deployment Strategies with Configuration-level resources

The Akri Agent and Discovery Handlers enable device discovery and Kubernetes resource creation: they discover devices, create Kubernetes resources to represent the devices, and ensure only `capacity` containers are using a device at once via the device plugin framework. The Akri Controller eases device use. If a broker is specified in a Configuration, the Controller will automatically deploy Kubernetes Pods or Jobs to discovered devices. Currently the Controller only supports two deployment strategies: either deploying a non-terminating Pod (that Akri calls a "broker") to each Node that can see a device or deploying a single Job to the cluster for each discovered device. There are plenty of scenarios that do not fit these two strategies such as a ReplicaSet like deployment of n number of Pods to the cluster. With Configuration-level resources, users could easily achieve their own scenarios without the Akri Controller, as selecting resources is more declarative. A user specifies in a resource request how many OPC UA servers are needed rather than needing to delineate the exact ones already discovered by Akri, as explained in Akri's current documentation on [requesting Akri resources](../docs/user-guide/requesting-akri-resources.md).

For example, with Configuration-level resources, the following Deployment could be applied to a cluster:

```yaml
apiVersion: "apps/v1"
kind: Deployment
metadata:
  name: onvif-broker-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      name: onvif-broker
  template:
    metadata:
      labels:
        name: onvif-broker
    spec:
      containers:
        - name: nginx
          image: "nginx:latest"
          resources:
            requests:
              "akri.sh/akri-onvif": "2"
            limits:
              "akri.sh/akri-onvif": "2"
```

Pods will only be successfully scheduled to a Node and run if the resources exist and are available. In the case of the
above scenario, if there were two cameras on the network like in the [implementation scenario](#implementation), two
Pods would be deployed to the cluster. If there are not enough resources, say there is only one camera on the network,
the two Pods will be left in a `Pending` state until another is discovered. This is the case with any deployment on
Kubernetes where there are not enough resources. However, `Pending` Pods do not use up cluster resources.

Currently, the Akri Controller handles deployment of Pods to avoid Pods being scheduled to Nodes without resources. This
is another advantage of using the Akri Controller rather than applying Deployments, DaemonSets, etc. In conjunction with
this work on supporting Configuration level resources, the Akri's Controller could support a new deployment strategy for
using these CL resources. The Controller also takes care of bringing down Pods when resources are no longer available,
while Pods from manually created Deployments would continue to run even if the resources are no longer there.
