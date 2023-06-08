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

There are two different policies to decide the number of slots the device plugin created for each Configuration will contain.
Users can set the value of `Configuration.uniqueDevices` to specify which policy to use.
| Policy |Configuration.uniqueDevices| Definition |
|--- | --- | --- |
| Per-Instance | true | `1` slot per instance, total `1` * `number of instances` slots.|
| Per-DeviceUsage | false | `capacity` slots per instance, total `capacity` * `number of instances` slots. |

`Configuration.uniqueDevices` is default to true if not specified in Configuration.

When a slot is allocated, each slot will map to a "real" deviceUage slot in an Instance custom resource.  The total number of slots
allocated by CL and IL resources for an Instance is restricted by `Instance.deviceUsage`.
 See [Resource Sharing between CL and IL device plugins](#resource-sharing-between-cl-and-il-device-plugins) for more details.

For example,
after deploying a `onvif` Configuration with a `capacity` of `2`, the `NodeSpec` of a node that
could see both cameras would be:

**`Configuration.uniqueDevices`: true**
```yaml
Capacity:
  akri.sh/akri-onvif:         2
  akri.sh/akri-onvif-8120fe:  2
  akri.sh/akri-onvif-a19705:  2
```

**`Configuration.uniqueDevices`: false**
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

 When `Configuration.uniqueDevices` is false, the Agent will select slots based on deviceUsage availability.  In the same example, if both deviceUsage slots of `akri-onvif-8120fe` are reserved, the Agent will selecting both slots from `akri-onvif-a19705`, both maps to the same instance.

  With different slot allocation policies, users can select policy that meet their requirement.  For example, for devices that act as data source (e.g., ip cameras), users might want to set `Configuration.uniqueDevices` to true so a workload won't get the same image from a camera.  For devices that provide services (e.g. GPU or devices that provide computing capacity), set `Configuration.uniqueDevices` to false will help to maximize the device sharing.

There are two implementation options in the case where there are not enough unique Instances to meet the requested number of Configuration-level resources. One scenario where this could happen is if a Pod requests 3 cameras when only 2 exist with a capacity of 2 each. In this case, the Configuration-level camera resource shows as having a quantity of 4, despite there being two cameras. In this case, the kubelet will try to schedule the Pod, thinking there are enough resources. The Agent could either allocate 2 spots on one camera and one on the other or deny the allocation request. The latter is the preferred approach as it is more consistent and ensures the workload is getting the number of unique devices it expects. After failing an `allocateRequest` from the kubelet, the Pod will be in an `UnexpectedAdmissionError` state until another camera comes online and it can be successfully scheduled.

## Resource sharing between CL and IL device plugins

When a slot is allocated, each slot will map to a "real" deviceUage slot in an Instance custom resource.  The total number of slots
allocated by CL and IL resources for an Instance is restricted by `Instance.deviceUsage`.
 Here is an example to show how CL and IL device plugins coordinate to each other to achieve resource sharing.
 In the example, `capacity` is 2 and there are two nodes `node-a` and `node-b` in the cluster that both discover two onvif cameras.
 The `Configuration.uniqueDevices` is true in this example, if `Configuration.uniqueDevices` is false, the flow is exactly the same instead
 the capacity of `akri-onvif` is `4 ( 2 * 2)`.

 **`Configuration.uniqueDevices`: true**
```yaml
Capacity:
  akri.sh/akri-onvif:         2
  akri.sh/akri-onvif-8120fe:  2
  akri.sh/akri-onvif-a19705:  2
```

```yaml
deviceUsage:
  akri.sh/akri-onvif-8120fe-0: ""
  akri.sh/akri-onvif-8120fe-1: ""

deviceUsage:
  akri.sh/akri-onvif-a19705-0: ""
  akri.sh/akri-onvif-a19705-1: ""
```

Assume Kubernetes shcedule a pod that request `2` CL devices to to `node-a`.
 After the pod scheduled to the node, 2 slots are reserved from the CL device plugin.  The CL resource capacity is `0 (2 - 2)` and 
 the IL resource capacity is `1`, respectively.  The capacity and deviceUsage become:

```yaml
Capacity:
  akri.sh/akri-onvif:         0
  akri.sh/akri-onvif-8120fe:  1
  akri.sh/akri-onvif-a19705:  1
```

```yaml
deviceUsage:
  akri.sh/akri-onvif-8120fe-0: "node-a"
  akri.sh/akri-onvif-8120fe-1: ""

deviceUsage:
  akri.sh/akri-onvif-a19705-0: ""
  akri.sh/akri-onvif-a19705-1: "node-a"
```

Assume Kubernetes schedule another pod that request `1` IL resource device `akri-onvif-a19705` to `node-b`.
 The capacity and deviceUsage is listed below, and there is still one free slot that is available to be used.

```yaml
Capacity:
  akri.sh/akri-onvif:         0
  akri.sh/akri-onvif-8120fe:  1
  akri.sh/akri-onvif-a19705:  0
```

```yaml
deviceUsage:
  akri.sh/akri-onvif-8120fe-0: "node-a"
  akri.sh/akri-onvif-8120fe-1: ""

deviceUsage:
  akri.sh/akri-onvif-a19705-0: "node-b"
  akri.sh/akri-onvif-a19705-1: "node-a"
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
