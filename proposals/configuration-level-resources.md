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

| Term | Definition |
| --- | --- |
| `capacity`  | Maximum amount of containers that can use a device. Defines the number of usage slots a device's device plugin advertises to the kubelet |
| Configuration-level (CL) resource | An extended resource that is created by the Agent for each Configuration. This equates to the creation of a Kubernetes device plugin for each Configuration. It enables requesting any of a set of (Configuration) resources, such as any IP camera, thermometer, etc. Resources are given the Configuration name. A request to use a CL resource is mapped to an IL resource by the Agent. |
| Instance-level (IL) resource | An extended resource that is created by the Agent for each discovered device (i.e. Instance), such as a specific IP camera, thermometer, etc. This equates to the creation of a Kubernetes device plugin for each Instance. Resources are named in the format <Configuration Name>-<HASH>, where the input to the one-way hash is a descriptor of the Instance (and node name for local non-shared devices). |

## Implementation

To support Configuration-level resources, the Agent will create an additional device plugin for each
Configuration (that initiates discovery). No changes are needed to the Discovery Handlers.  For the Controller, changes might be required to support the Configuration device plugin.

On a node that exposes device plugin for Configuration, there are two different policies to decide how the Configuration device plugin selects slots
Users can set the value of `Configuration.uniqueDevices` to specify which policy to use.
| Policy |Configuration.uniqueDevices| Definition |
|--- | --- | --- |
| Per-Instance | true | `1` slot per instance, total `1` * `number of instances` slots on a node.|
| Per-DeviceUsage | false | `capacity` slots per instance, total `capacity` * `number of instances` slots on a node.|

`Configuration.uniqueDevices` is default to true if not specified in Configuration.

When a slot is allocated, each slot will map to a "real" deviceUage slot in an Instance custom resource.  The total number of slots
allocated by CL and IL resources for an Instance is restricted by `Instance.deviceUsage`.
 See [Resource Sharing between CL and IL device plugins](#resource-sharing-between-cl-and-il-device-plugins) for more details.

For example,
after deploying a `onvif` Configuration with a `capacity` of `2`, the `NodeSpec` of a node that
could see both cameras would be:

```yaml
Capacity:
  akri.sh/akri-onvif:         4
  akri.sh/akri-onvif-8120fe:  2
  akri.sh/akri-onvif-a19705:  2
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
How the Agent select slots to reserve is based on the value of `Configuration.uniqueDevices`. 

 When `Configuration.uniqueDevices` is true, the Agent will select max 1 slot for each Instance. For example, in the above Pod, the Agent will make sure 1 slot of each `akri-onvif-8120fe` and `akri-onvif-a19705` is reserved for each Pod.

 When `Configuration.uniqueDevices` is false, the Agent will select slots based on deviceUsage availability.  In the same example, if both deviceUsage slots of `akri-onvif-8120fe` are reserved by other nodes, the Agent will selecting both slots from `akri-onvif-a19705`, both maps to the same instance.

  With different slot allocation policies, users can select policy that meet their requirement.  For example, for devices that act as data source (e.g., ip cameras), users might want to set `Configuration.uniqueDevices` to true so a workload won't get the same image from a camera.  For devices that provide services (e.g. GPU or devices that provide computing capacity), set `Configuration.uniqueDevices` to false will help to maximize the device sharing.

The `Configuration.uniqueDevices` policy provides two different behaviors in the case where there are not enough unique Instances to meet the requested number of Configuration-level resources. One scenario where this could happen is if a Pod requests 3 cameras when only 2 exist with a capacity of 2 each.
 In this case, if the `Configuration.uniqueDevices` is true, the Configuration-level camera resource shows as having a quantity of 2 on all nodes that has access to two cameras. The kubelet will see there is not enough resource to start the pod and put the pod in Pending state waiting for one more camera resource exposed from Configuration device plugin.
 
 In the same example but the `Configuration.uniqueDevices` is false, the Configuration-level camera resource shows as having a quantity of 4 on all nodes that has access to two cameras.  The kubelet will try to schedule the Pod, thinking there are enough resources. The Agent will allocate 2 spots on one camera and one on the other.

## Resource sharing between CL and IL device plugins

When a slot is allocated, each slot will map to a "real" deviceUage slot in an Instance custom resource.  The total number of slots
allocated by CL and IL resources for an Instance is restricted by `Instance.deviceUsage`.
 Here is an example to show how CL and IL device plugins coordinate to each other to achieve resource sharing.
 In the example, `capacity` is 2 and there are two nodes `node-a` and `node-b` in the cluster that both discover two onvif cameras.

 **`Configuration.uniqueDevices`: true**

 The resource availability for the cluster:
```yaml
Capacity:
  akri.sh/akri-onvif:         4
  akri.sh/akri-onvif-8120fe:  2
  akri.sh/akri-onvif-a19705:  2

deviceUsage:
  akri.sh/akri-onvif-8120fe-0: ""
  akri.sh/akri-onvif-8120fe-1: ""

deviceUsage:
  akri.sh/akri-onvif-a19705-0: ""
  akri.sh/akri-onvif-a19705-1: ""
```

 The resource exposed by `node-a`:
```yaml
CL device plugin:
  akri.sh/akri-onvif-8120fe:  "Healthy" ; free deviceUsage slots available
  akri.sh/akri-onvif-a19705:  "Healthy" ; free deviceUsage slots available

IL device plugin1:
  akri.sh/akri-onvif-8120fe-0: "Healthy" ; free
  akri.sh/akri-onvif-8120fe-1: "Healthy" ; free

IL device plugin2:
  akri.sh/akri-onvif-a19705-0: "Healthy" ; free
  akri.sh/akri-onvif-a19705-1: "Healthy" ; free
```

 The resource exposed by `node-b`:
```yaml
CL device plugin:
  akri.sh/akri-onvif-8120fe:  "Healthy" ; free deviceUsage slots available
  akri.sh/akri-onvif-a19705:  "Healthy" ; free deviceUsage slots available

IL device plugin1:
  akri.sh/akri-onvif-8120fe-0: "Healthy" ; free
  akri.sh/akri-onvif-8120fe-1: "Healthy" ; free

IL device plugin2:
  akri.sh/akri-onvif-a19705-0: "Healthy" ; free
  akri.sh/akri-onvif-a19705-1: "Healthy" ; free
```

Assume Kubernetes shcedule a pod that request `2` CL devices to `node-a`.
 After the pod scheduled to the node, 2 slots are reserved from the CL device plugin.  On `node-a`, the CL resource availability is `2` (but allocated) and 
 the IL resource availability is `1`, respectively.  The cluster and node states become:

 The resource availability for the cluster:
```yaml
Capacity:
  akri.sh/akri-onvif:         4
  akri.sh/akri-onvif-8120fe:  2
  akri.sh/akri-onvif-a19705:  2

deviceUsage:
  akri.sh/akri-onvif-8120fe-0: "node-a"
  akri.sh/akri-onvif-8120fe-1: ""

deviceUsage:
  akri.sh/akri-onvif-a19705-0: ""
  akri.sh/akri-onvif-a19705-1: "node-a"
```

 Resources exposed by `node-a`:
```yaml
CL device plugin:
  akri.sh/akri-onvif-8120fe:  "Healthy" ; reserved by self
  akri.sh/akri-onvif-a19705:  "Healthy" ; reserved by self

IL device plugin1:
  akri.sh/akri-onvif-8120fe-0: "Unhealthy" ; reserved by CL device plugin
  akri.sh/akri-onvif-8120fe-1: "Healthy" ; free

IL device plugin2:
  akri.sh/akri-onvif-a19705-0: "Healthy" ; free
  akri.sh/akri-onvif-a19705-1: "Unhealthy" ; reserved by CL device plugin
```

 Resources exposed by `node-b`:
```yaml
CL device plugin:
  akri.sh/akri-onvif-8120fe:  "Healthy" ; one free deviceUsage slot available
  akri.sh/akri-onvif-a19705:  "Healthy" ; one free deviceUsage slot available

IL device plugin1:
  akri.sh/akri-onvif-8120fe-0: "Unhealthy" ; reserved by other node
  akri.sh/akri-onvif-8120fe-1: "Healthy" ; free

IL device plugin2:
  akri.sh/akri-onvif-a19705-0: "Healthy" ; free
  akri.sh/akri-onvif-a19705-1: "Unhealthy" ; reserved by other node
```

Assume Kubernetes schedule another pod that request `1` IL resource device `akri-onvif-a19705` to `node-b`.
 The state is listed below, and there is still one free slot that is available to be used.

```yaml
Capacity:
  akri.sh/akri-onvif:         4
  akri.sh/akri-onvif-8120fe:  2
  akri.sh/akri-onvif-a19705:  2

deviceUsage:
  akri.sh/akri-onvif-8120fe-0: "node-a"
  akri.sh/akri-onvif-8120fe-1: ""

deviceUsage:
  akri.sh/akri-onvif-a19705-0: "node-b"
  akri.sh/akri-onvif-a19705-1: "node-a"
```

 Resource exposed by `node-a`:
```yaml
CL device plugin:
  akri.sh/akri-onvif-8120fe:  "Healthy" ; reserved by self
  akri.sh/akri-onvif-a19705:  "Healthy" ; reserved by self

IL device plugin1:
  akri.sh/akri-onvif-8120fe-0: "Unhealthy" ; reserved by CL device plugin
  akri.sh/akri-onvif-8120fe-1: "Healthy" ; free

IL device plugin2:
  akri.sh/akri-onvif-a19705-0: "Unhealthy" ; reserved by other node
  akri.sh/akri-onvif-a19705-1: "Unhealthy" ; reserved by CL device plugin
```

 Resource exposed by `node-b`:
```yaml
CL device plugin:
  akri.sh/akri-onvif-8120fe:  "Healthy" ; one free deviceUsage slot available
  akri.sh/akri-onvif-a19705:  "Unhealthy" ; no free slot, reserved by other node and IL device plugin

IL device plugin1:
  akri.sh/akri-onvif-8120fe-0: "Unhealthy" ; reserved by other node
  akri.sh/akri-onvif-8120fe-1: "Healthy" ; free

IL device plugin2:
  akri.sh/akri-onvif-a19705-0: "Healthy" ; reserved by self
  akri.sh/akri-onvif-a19705-1: "Unhealthy" ; reserved by other node
```

On the other hand, if the resources are claimed by the IL device plugins, with the same example (2 nodes, 2 cameras, capacity = 2)
 the deviceUsage will be updated as following:
 
 Initiali state
```yaml
Capacity:
  akri.sh/akri-onvif:         4
  akri.sh/akri-onvif-8120fe:  2
  akri.sh/akri-onvif-a19705:  2

deviceUsage:
  akri.sh/akri-onvif-8120fe-0: ""
  akri.sh/akri-onvif-8120fe-1: ""

deviceUsage:
  akri.sh/akri-onvif-a19705-0: ""
  akri.sh/akri-onvif-a19705-1: ""
```

 The resource exposed by `node-a`:
```yaml
CL device plugin:
  akri.sh/akri-onvif-8120fe:  "Healthy" ; free deviceUsage slots available
  akri.sh/akri-onvif-a19705:  "Healthy" ; free deviceUsage slots available

IL device plugin1:
  akri.sh/akri-onvif-8120fe-0: "Healthy" ; free
  akri.sh/akri-onvif-8120fe-1: "Healthy" ; free

IL device plugin2:
  akri.sh/akri-onvif-a19705-0: "Healthy" ; free
  akri.sh/akri-onvif-a19705-1: "Healthy" ; free
```

 The resource exposed by `node-b`:
```yaml
CL device plugin:
  akri.sh/akri-onvif-8120fe:  "Healthy" ; free deviceUsage slots available
  akri.sh/akri-onvif-a19705:  "Healthy" ; free deviceUsage slots available

IL device plugin1:
  akri.sh/akri-onvif-8120fe-0: "Healthy" ; free
  akri.sh/akri-onvif-8120fe-1: "Healthy" ; free

IL device plugin2:
  akri.sh/akri-onvif-a19705-0: "Healthy" ; free
  akri.sh/akri-onvif-a19705-1: "Healthy" ; free
```

 Assume Kubernetes shcedule one pod for each discovered instance. `2` IL devices are allocated to `node-a`.
 After the pod scheduled to the node, 2 slots are reserved for the IL device plugins.  The IL resource availability is `1 (2 - 1)` respectively and 
 the CL resource availability is `2`.

```yaml
Capacity:
  akri.sh/akri-onvif:         4
  akri.sh/akri-onvif-8120fe:  2
  akri.sh/akri-onvif-a19705:  2

deviceUsage:
  akri.sh/akri-onvif-8120fe-0: "node-a"
  akri.sh/akri-onvif-8120fe-1: ""

deviceUsage:
  akri.sh/akri-onvif-a19705-0: ""
  akri.sh/akri-onvif-a19705-1: "node-a"
```

 Resources exposed by `node-a`:
```yaml
CL device plugin:
  akri.sh/akri-onvif-8120fe:  "Healthy" ; one free deviceUsage slot available
  akri.sh/akri-onvif-a19705:  "Healthy" ; one free deviceUsage slot available

IL device plugin1:
  akri.sh/akri-onvif-8120fe-0: "Healthy" ; reserved by self
  akri.sh/akri-onvif-8120fe-1: "Healthy" ; free

IL device plugin2:
  akri.sh/akri-onvif-a19705-0: "Healthy" ; free
  akri.sh/akri-onvif-a19705-1: "Healthy" ; reserved by self
```

 Resources exposed by `node-b`:
```yaml
CL device plugin:
  akri.sh/akri-onvif-8120fe:  "Healthy" ; one free deviceUsage slot
  akri.sh/akri-onvif-a19705:  "Healthy" ; one free deviceUsage slot

IL device plugin1:
  akri.sh/akri-onvif-8120fe-0: "Unhealthy" ; reserved by other node
  akri.sh/akri-onvif-8120fe-1: "Healthy" ; free

IL device plugin2:
  akri.sh/akri-onvif-a19705-0: "Healthy" ; free
  akri.sh/akri-onvif-a19705-1: "Unhealthy" ; reserved by other node
```

 The `Configuration.uniqueDevices` is true in previous example, if `Configuration.uniqueDevices` is false, the flow is exactly the same except 
 different number of devices (instances * capacity = 2 * 2) reported by the Configuration device plugins on those two nodes and the CL device plugin decides resource availability based on the same rule used by IL device plugin.

 **`Configuration.uniqueDevices`: false**

 The resource availability for the cluster:
```yaml
Capacity:
  akri.sh/akri-onvif:         4
  akri.sh/akri-onvif-8120fe:  2
  akri.sh/akri-onvif-a19705:  2

deviceUsage:
  akri.sh/akri-onvif-8120fe-0: ""
  akri.sh/akri-onvif-8120fe-1: ""

deviceUsage:
  akri.sh/akri-onvif-a19705-0: ""
  akri.sh/akri-onvif-a19705-1: ""
```

 The resource exposed by `node-a`:
```yaml
CL device plugin:
  akri.sh/akri-onvif-8120fe-0: "Healthy" ; free
  akri.sh/akri-onvif-8120fe-1: "Healthy" ; free
  akri.sh/akri-onvif-a19705-0: "Healthy" ; free
  akri.sh/akri-onvif-a19705-1: "Healthy" ; free

IL device plugin1:
  akri.sh/akri-onvif-8120fe-0: "Healthy" ; free
  akri.sh/akri-onvif-8120fe-1: "Healthy" ; free

IL device plugin2:
  akri.sh/akri-onvif-a19705-0: "Healthy" ; free
  akri.sh/akri-onvif-a19705-1: "Healthy" ; free
```

 The resource exposed by `node-b`:
```yaml
CL device plugin:
  akri.sh/akri-onvif-8120fe-0: "Healthy" ; free
  akri.sh/akri-onvif-8120fe-1: "Healthy" ; free
  akri.sh/akri-onvif-a19705-0: "Healthy" ; free
  akri.sh/akri-onvif-a19705-1: "Healthy" ; free

IL device plugin1:
  akri.sh/akri-onvif-8120fe-0: "Healthy" ; free
  akri.sh/akri-onvif-8120fe-1: "Healthy" ; free

IL device plugin2:
  akri.sh/akri-onvif-a19705-0: "Healthy" ; free
  akri.sh/akri-onvif-a19705-1: "Healthy" ; free
```

Assume Kubernetes shcedule a pod that request `2` CL devices to `node-a`. Devices from the same instance might be selected.

 The resource availability for the cluster:
```yaml
Capacity:
  akri.sh/akri-onvif:         4
  akri.sh/akri-onvif-8120fe:  2
  akri.sh/akri-onvif-a19705:  2

deviceUsage:
  akri.sh/akri-onvif-8120fe-0: "node-a"
  akri.sh/akri-onvif-8120fe-1: "node-a"

deviceUsage:
  akri.sh/akri-onvif-a19705-0: ""
  akri.sh/akri-onvif-a19705-1: ""
```

 The resource exposed by `node-a`:
```yaml
CL device plugin:
  akri.sh/akri-onvif-8120fe-0: "Healthy" ; reserved by self
  akri.sh/akri-onvif-8120fe-1: "Healthy" ; reserved by self
  akri.sh/akri-onvif-a19705-0: "Healthy" ; free
  akri.sh/akri-onvif-a19705-1: "Healthy" ; free

IL device plugin1:
  akri.sh/akri-onvif-8120fe-0: "Unhealthy" ; reserved by CL device plugin
  akri.sh/akri-onvif-8120fe-1: "Unhealthy" ; reserved by CL device plugin

IL device plugin2:
  akri.sh/akri-onvif-a19705-0: "Healthy" ; free
  akri.sh/akri-onvif-a19705-1: "Healthy" ; free
```

 The resource exposed by `node-b`:
```yaml
CL device plugin:
  akri.sh/akri-onvif-8120fe-0: "Unhealthy" ; reserved by other node
  akri.sh/akri-onvif-8120fe-1: "Unhealthy" ; reserved by other node
  akri.sh/akri-onvif-a19705-0: "Healthy" ; free
  akri.sh/akri-onvif-a19705-1: "Healthy" ; free

IL device plugin1:
  akri.sh/akri-onvif-8120fe-0: "Unhealthy" ; reserved by other node
  akri.sh/akri-onvif-8120fe-1: "Unhealthy" ; reserved by other node

IL device plugin2:
  akri.sh/akri-onvif-a19705-0: "Healthy" ; free
  akri.sh/akri-onvif-a19705-1: "Healthy" ; free
```

## Deployment Strategies with Configuration-level resources

The Akri Agent and Discovery Handlers enable device discovery and Kubernetes resource creation: they discovers devices, creates Kubernetes resources to represent the devices, and ensure only `capacity` containers are using a device at once via the device plugin framework. The Akri Controller eases device use. If a broker is specified in a Configuration, the Controller will automatically deploy Kubernetes Pods or Jobs to discovered devices. Currently the Controller only supports two deployment strategies: either deploying a non-terminating Pod (that Akri calls a "broker") to each Node that can see a device or deploying a single Job to the cluster for each discovered device. There are plenty of scenarios that do not fit these two strategies such as a ReplicaSet like deployment of n number of Pods to the cluster. With Configuration-level resources, users could easily achieve their own scenarios without the Akri Controller, as selecting resources is more declarative. A user specifies in a resource request how many OPC UA servers are needed rather than needing to delineate the exact ones already discovered by Akri, as explained in Akri's current documentation on [requesting Akri resources](../docs/user-guide/requesting-akri-resources.md).

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


Pods will only be successfully scheduled to a Node and run if the resources exists and are available. In the case of the
above scenario, if there were two cameras on the network like in the [implementation scenario](#implementation), two
Pods would be deployed to the cluster. If there are not enough resources, say there is only one camera on the network,
the two Pods will be left in a `Pending` state until another is discovered. This is the case with any deployment on
Kubernetes where there are not enough resources. However, `Pending` Pods do not use up cluster resources.

Currently, the Akri Controller handles deployment of Pods to avoid Pods being scheduled to Nodes without resources. This
is another advantage of using the Akri Controller rather than applying Deployments, DaemonSets, etc. In conjunction with
this work on supporting Configuration level resources, the Akri's Controller could support a new deployment strategy for
using these CL resources. The Controller also takes care of bringing down Pods when resources are no longer available,
while Pods from manually created Deployments would continue to run even if the resources are no longer there.
