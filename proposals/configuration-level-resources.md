# Configuration-level Resources

This document discusses the motivation and potential implementation for supporting "Configuration-level resources" in Akri. Currently, Akri only supports creating a Kubernetes resource (i.e. device plugin) for each individual device. Since each device in Akri is represented as an Instance custom resource, these are called Instance-level resources. A Configuration-level resource is a resource that represents all of the devices discovered via a Configuration, i.e. all IP cameras.

Currently, with only Instance-level resources, specific devices must be requested as resources. For example, a Pod could be deployed that uses both `ip-camera-UNIQUE_ID1` and `ip-camera-UNIQUE_ID2` by adding the following resource request to the `PodSpec`:

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

## Terms

| Term | Definition |
| --- | --- |
| `capacity`  | Maximum amount of containers that can use a device. Defines the number of usage slots a device's device plugin advertises to the kubelet |
| Configuration-level (CL) resource | An extended resource that is created by the Agent for each Configuration. This equates to the creation of a Kubernetes device plugin for each Configuration. It enables requesting any of a set of (Configuration) resources, such as any IP camera, thermometer, etc. Resources are given the Configuration name. A request to use a CL resource is mapped to an IL resource by the Agent. |
| Instance-level (IL) resource | An extended resource that is created by the Agent for each discovered device (i.e. Instance), such as a specific IP camera, thermometer, etc. This equates to the creation of a Kubernetes device plugin for each Instance. Resources are named in the format <Configuration Name>-<HASH>, where the input to the one-way hash is a descriptor of the Instance (and node name for local non-shared devices). each Configuration |

## Implementation

To support Configuration-level resources, the Agent will create an additional device plugin for each
Configuration (that initiates discovery). No changes are needed to the Controller or Discovery Handlers.

The device plugin created for each Configuration will contain `capacity` * `number of
instances` slots. Each slot will map to a "real" slot of an Instance device plugin. For example,
after deploying a `onvif` Configuration with a `capacity` of `2`, the `NodeSpec` of a node that
could see both cameras would be:

```yaml
Capacity:
  akri.sh/akri-onvif:         4
  akri.sh/akri-onvif-8120fe:  2
  akri.sh/akri-onvif-a19705:  2
```

Now, an operator or the Controller can create a DaemonSet like the following:

```yaml
apiVersion: "apps/v1"
kind: DaemonSet
metadata:
  name: onvif-broker-daemonset
spec:
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

The Kubernetes scheduler will deploy a Pod to each Node. Pods will only be successfully scheduled to
a Node and run if the resources exists and are available. Otherwise they will be left in a `Pending`
state. See this in action by walking through the [note on `Pending` Pods](#Note-on-Pending-Pods).

**Note: More `Pending` Pods is a downside to Configuration-level resources, whether the Deployments are deployed by the Akri Controller or the user. Since
However, `Pending` Pods only only equates to just more YAML it etcd.**

When selecting the slots from Instances, the Agent should preference unique instances; for example, in the above Daemonset, the Agent will make sure 1 slot of each `akri-onvif-8120fe` and `akri-onvif-a19705` is reserved for each Pod.

> Note: If the number of CL resources request exceeds the number of IL resource -- say 3 were requested in the above DaemonSet -- either the Agent can forbid the Pod from running and the brokers Pods will be in an `UnexpectedAdmissionError` state or the use of 2 slots from one device and 1 from another could be permitted. More thought can go here, and this should maybe be configurable.

### Note on Pending Pods

See how deploying a DaemonSet causes the Kubernetes Scheduler to try to deploy a Pod to each Node,
regardless of whether the resources exist. If they don't exist, the Pod will not be successfully
scheduled to any node and will be left in a pending state. This can be demonstrated with Akri today.

1. Install Akri on a multi-node cluster, using the debugEcho discovery handler to discover one fake
   `foo0` device and selecting that the discovery handler only run on one node.

    ```sh
    helm install akri akri-helm-charts/akri \
    $AKRI_HELM_CRICTL_CONFIGURATION \
    --set agent.allowDebugEcho=true \
    --set debugEcho.discovery.enabled=true \
    --set debugEcho.configuration.enabled=true \
    --set debugEcho.configuration.shared=true \
    --set debugEcho.configuration.discoveryDetails.descriptions[0]="foo0" \
    --set debugEcho.discovery.nodeSelectors.'kubernetes\.io\/hostname'=$HOSTNAME
    ```

1. Get the instance's name: `kubectl get akrii | grep akri-debug-echo | awk '{ print $1 }'` Deploy a
   DaemonSet that uses it, inserting the instance name in the resource request and limit:

    ```yaml
    apiVersion: "apps/v1"
    kind: DaemonSet
    metadata:
    name: nginx-daemonset
    spec:
    selector:
        matchLabels:
        name: nginx
    template:
        metadata:
        labels:
            name: nginx
        spec:
        containers:
        - name: nginx
            image: "nginx:latest"
            resources:
            requests:
                "akri.sh/INSTANCE_NAME_HERE": "1"
            limits:
                "akri.sh/INSTANCE_NAME_HERE": "1"
    ```

1. Look at the Pods and Akri components that have been created. Notice that one pod is left
   `Pending` and has been assigned to no node. For a two node cluster, the output of `kubectl get
   pods,akrii,akric -o wide` should look similar to the following:

    ```sh
    NAME                                              READY   STATUS    RESTARTS   AGE     IP            NODE          NOMINATED NODE   READINESS GATES
    pod/akri-agent-daemonset-7vls5                    1/1     Running   0          14m     10.42.1.118   nuc-ubtunu1   <none>           <none>
    pod/akri-debug-echo-discovery-daemonset-qpwpf     1/1     Running   0          14m     10.42.0.104   nuc-ubuntu2   <none>           <none>
    pod/akri-agent-daemonset-fn2fv                    1/1     Running   0          14m     10.42.0.103   nuc-ubuntu2   <none>           <none>
    pod/akri-controller-deployment-776897c88f-8z6st   1/1     Running   0          14m     10.42.1.119   nuc-ubtunu1   <none>           <none>
    pod/nginx-daemonset-n9dpk                         1/1     Running   0          13m     10.42.0.107   nuc-ubuntu2   <none>           <none>
    pod/nginx-daemonset-smm2k                         0/1     Pending   0          13m     <none>        <none>        <none>           <none>
    pod/nuc-ubuntu2-akri-debug-echo-8120fe-pod        1/1     Running   0          8m23s   10.42.0.108   nuc-ubuntu2   <none>           <none>

    NAME                                      CONFIG            SHARED   NODES             AGE
    instance.akri.sh/akri-debug-echo-8120fe   akri-debug-echo   true     ["nuc-ubuntu2"]   8m25s

    NAME                                    CAPACITY   AGE
    configuration.akri.sh/akri-debug-echo   2          14m
    ```
