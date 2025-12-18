# Configuration-level Resources

Akri supports creating a Kubernetes resource (i.e. device plugin) for each individual device. Since each device in Akri is represented as an Instance custom resource, these are called Instance-level resources. Instance-level resources are named in the format `<configuration-name>-<instance-id>`. Akri also creates a Kubernetes Device Plugin for a Configuration called Configuration-level resource. A Configuration-level resource is a resource that represents all of the devices discovered via a Configuration. With Configuration-level resources, instead of needing to know the specific Instances to request, resources could be requested by the Configuration name and the Agent will do the work of selecting which Instances to reserve. The example below shows a deployment that requests the resource at Configuration level and would deploy a nginx broker to each discovered device respectively.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: onvif-camera-broker-deployment
  labels:
    app: onvif-camera-broker
spec:
  replicas: 1
  selector:
    matchLabels:
      app: onvif-camera-broker
  template:
    metadata:
      labels:
        app: onvif-camera-broker
    spec:
      containers:
        - name: onvif-camera-broker
          image: nginx
          resources:
            limits:
              akri.sh/onvif-camera: "2"
            requests:
              akri.sh/onvif-camera: "2"
```

With Configuration-level resources, users could use higher level Kubernetes objects (Deployments, ReplicaSets, DaemonSets, etc.) or develop their own deployment strategies, rather than relying on the Akri Controller to deploy Pods to discovered devices.

### Maintaining Device Usage

The [in-depth resource sharing doc](resource-sharing-in-depth.md) describes how the `Configuration.capacity` and `Instance.deviceUsage` are used to achieve resource sharing between nodes.
The same data is used to achieve sharing the same resource between Configuration-level and Instance-level resources.

The `Instance.deviceUsage` in Akri Instances is extended to support Configuration device plugin.
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
the `Instance.deviceUsage` format is extended to hold the additional information, the deviceUsage can be a "<node_name>" (for Instance) or a "C:<virtual_device_id>:<node_name>" (for
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

## Deployment Strategies with Configuration-level resources

The Akri Agent and Discovery Handlers enable device discovery and Kubernetes resource creation: they discover devices, create Kubernetes resources to represent the devices, and ensure only `capacity` containers are using a device at once via the device plugin framework. The Akri Controller eases device use. If a broker is specified in a Configuration, the Controller will automatically deploy Kubernetes Pods or Jobs to discovered devices. Currently the Controller only supports two deployment strategies: either deploying a non-terminating Pod (that Akri calls a "broker") to each Node that can see a device or deploying a single Job to the cluster for each device discovered. There are plenty of scenarios that do not fit these two strategies such as a ReplicaSet like deployment of n number of Pods to the cluster. With Configuration-level resources, users could easily achieve their own scenarios without the Akri Controller, as selecting resources is more declarative. A user specifies in a resource request how many OPC UA servers are needed rather than needing to delineate the exact ones already discovered by Akri, as explained in Akri's current documentation on [requesting Akri resources](../docs/user-guide/requesting-akri-resources.md).

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
above scenario, if there were two cameras on the network, two Pods would be deployed to the cluster. If there are not
enough resources, say there is only one camera on the network,
the two Pods will be left in a `Pending` state until another is discovered. This is the case with any deployment on
Kubernetes where there are not enough resources. However, `Pending` Pods do not use up cluster resources.
