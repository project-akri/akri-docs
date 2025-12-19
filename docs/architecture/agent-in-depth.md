# Agent

The Akri Agent executes on all worker Nodes in the cluster. It is primarily tasked with:

1. Handling resource availability changes
2. Enabling resource sharing

These two tasks enable Akri to find configured resources (leaf devices), expose them to the Kubernetes cluster for workload scheduling, and allow resources to be shared by multiple Nodes.

## Handling resource availability changes

The first step in handling resource availability is determining what resources (leaf devices) to look for. This is accomplished by finding existing Configurations and watching for changes to them.

Once the Akri Agent understands what resources to look for (via `Configuration.discovery_handler`), it will [find any resources that are visible](agent-in-depth.md#resource-discovery).

For each resource that is found:

1. An Instance is created and uploaded to etcd
2. A connection with the kubelet is established according to the Kubernetes Device Plugin framework. This connection is used to convey availability changes to the kubelet. The kubelet will, in turn, expose these availability changes to the Kubernetes scheduler.

Each protocol will periodically reassess what resources are visible and update both the Instance and the kubelet with the current availability.

This process allows Akri to dynamically represent resources that appear and disappear.

## Enabling resource sharing

To enable resource sharing, the Akri Agent creates and updates the `Instance.deviceUsage` map and communicates with kubelet. The `Instance.deviceUsage` map is used to coordinate between Nodes. The kubelet communication allows Akri Agent to communicate any resource availability changes to the Kubernetes scheduler.

For more detailed information, see the [in-depth resource sharing doc](resource-sharing-in-depth.md).

Akri Agent also exposes all discovered resources at Configuration level. Configuration level resources can be referred by the name of Configuration so Configuration name can be used to request resources without the need to know the specific Instances id to request. Agent will behind the scenes do the work of selecting which Instances to reserve.

For more detailed information about Configuration level resource, see the [Configuration-level resources doc](configuration-level-resource-in-depth.md).

## Resource discovery

The Agent discovers resources via Discovery Handlers (DHs). A Discovery Handler is anything that implements the `DiscoveryHandler` service defined in [`discovery.proto`](https://github.com/project-akri/akri/blob/main/discovery-utils/proto/discovery.proto). In order to be utilized, a DH must register with the Agent, which hosts the `Registration` service defined in [`discovery.proto`](https://github.com/project-akri/akri/blob/main/discovery-utils/proto/discovery.proto). The Agent maintains a list of registered DHs and their connectivity statuses, which is either `Waiting`, `Active`, or `Offline(Instant)`. When registered, a DH's status is `Waiting`. Once a Configuration requesting resources discovered by a DH is applied to the Akri-enabled cluster, the Agent will create a connection with the DH requested in the Configuration and set the status of the DH to `Active`. If the Agent is unable to connect or loses a connection with a DH, its status is set to `Offline(Instant)`. The `Instant` marks the time at which the DH became unresponsive. If the DH has been offline for more than 5 minutes, it is removed from the Agent's list of registered Discovery Handlers. If a Configuration is deleted, the Agent drops the connection it made with all DHs for that Configuration and marks the DHs' statuses as `Waiting`. Note, while probably not commonplace, the Agent allows for multiple DHs to be registered for the same protocol. IE: you could have two udev DHs running on a node on different sockets.

The Agent's registration service defaults to running on the socket `/var/lib/akri/agent-registration.sock` but can be Configured with Helm. While Discovery Handlers must register with this service over UDS, the Discovery Handler's service can run over UDS or an IP based endpoint.

Supported Rust DHs each have a [library](https://github.com/project-akri/akri/tree/main/discovery-handlers) and a [binary implementation](https://github.com/project-akri/akri/tree/main/discovery-handler-modules). This allows them to either be run within the Agent binary or in their own Pod.

Reference the [Discovery Handler development document](../development/handler-development.md) to learn how to implement a Discovery Handler.

## Passing additional properties to Discovery Handlers

In addition to the `discoveryDetails` in Configuration that sets details for narrowing the Discovery Handlers' search, the `discoveryProperties` can be used to pass additional information to Discovery Handler. One of scenarios that can leverage `discoveryProperties` is to pass credential data to Discovery Handlers to perform
authenticated resource discovery.
It is common for a device to require authentication in order to access its properties. The Discovery Handler then need these credentials to properly discover and filter the device. The credential data can be placed in
`discoverProperties`, if it is specified in Configuration, Agent reads the content and generate a list of string key-value pair properties and pass the list to
Discovery Handler along with `discoveryDetails`.

Agent supports plain text, K8s `secret` and `configMap` in the schema of `discoverProperties`. An example below shows how each type of property is specified in `discoveryProperties`. The `name` of property is required and needs to be in C_IDENTIFIER format. The value can be specified by `value` or `valueFrom`. For value specified by `valueFrom`, it can be from `secret` or `configMap`. The `optional` attribute is default to `false`, it means if the data doesn't exist (in the `secret` or `configMap`), the Configuration deployment
will fail. If `optional` is `true`, Agent will ignore the entry if the data doesn't exist, and pass all exist properties to Discovery Handler, the Configuration deployment will success.

```yaml
discoveryProperties:
  - name: property_from_plain_text
    value: “plain text data”
  - name: property_from_secret
    valueFrom:
      secretKeyRef:
        name: mysecret
        namespace: mysecret-namespace
        key: secret-key
        optional: false
  - name: property_from_configmap
    valueFrom:
      configMapKeyRef:
        name: myconfigMap
        namespace: myconfigmap-namespace
        key: configmap-key
        optional: true
```

For the example above, with the content of secret and configMap.

```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
  namespace: mysecret-namespace
type: Opaque
stringData:
  secret-key: "secret1"
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: myconfigMap
  namespace: myconfigmap-namespace
data:
  configmap-key: "configmap1"
```

Agent read all properties and pass the string key-value pair list to Discovery Handle.

```yaml
"property_from_plain_text": “plain text data”
"property_from_secret": "secret1"
"property_from_configmap": "configmap1"
```
