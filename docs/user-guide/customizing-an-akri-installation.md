# Customizing an Akri Installation

The [ONVIF](../discovery-handlers/onvif.md), [udev](../discovery-handlers/udev.md), and [OPC UA](../discovery-handlers/opc-ua.md) Configurations documentation explains how to deploy Akri and utilize a specific Discovery Handler using Helm (more information about the Akri Helm charts can be found in the [user guide](getting-started.md#understanding-akri-helm-charts)). This documentation elaborates upon them, covering the following: 

1. Starting Akri without any Configurations 
1. Generating, modifying and applying a Configuration
1. Deploying multiple Configurations 
1. Modifying a deployed Configuration
1. Adding another Configuration to a cluster
1. Modifying a broker
1. Deleting a Configuration from a cluster
1. Applying Discovery Handlers

## Starting Akri without any Configurations

To install Akri without any protocol Configurations, run this:


```bash
helm repo add akri-helm-charts https://project-akri.github.io/akri/
helm install akri akri-helm-charts/akri
```

This will deploy the Akri Controller and deploy Akri Agents.

## Generating, modifying and applying a Configuration

Helm allows us to parametrize the commonly modified fields in our Configuration templates and we have provided many (to see them, run `helm inspect values akri-helm-charts/akri`). For more advanced Configuration changes that are not aided by our Helm chart, we suggest creating a Configuration file using Helm and then manually modifying it.

For example, to create an ONVIF Configuration file, run the following. (To instead create a udev Configuration, substitute `onvif.configuration.enabled` with `udev.configuration.enabled` and add a udev rule. For OPC UA, substitute with `opcua.configuration.enabled`.)

```bash
helm template akri akri-helm-charts/akri \
    --set onvif.configuration.enabled=true \
    --set onvif.configuration.brokerPod.image.repository=nginx \
    --set rbac.enabled=false \
    --set controller.enabled=false \
    --set agent.enabled=false > configuration.yaml
```

Note, that for the broker pod image, nginx was specified. Insert your broker image instead or remove the broker pod image from the installation command to generate a Configuration without a broker PodSpec or ServiceSpecs. Once you have modified the yaml file, you can apply the new Configuration to the cluster with standard kubectl like this:

```bash
kubectl apply -f configuration.yaml
```

{% hint style="info" %}
When modifying the Configuration, do not remove the resource request and limit `{{PLACEHOLDER}}`. The Controller inserts the request for the discovered device/Instance here.
{% endhint %}

The following sections explain some of the ways the configuration.yaml could be modified to customize settings/fields that cannot be set with Akri's Helm Chart.

#### Modifying the brokerPodSpec

The `brokerPodSpec` property is a full [PodSpec](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#podspec-v1-core) and can be modified as such. For example, to allow the master Node to have a protocol broker Pod scheduled to it, modify the Configuration, ONVIF in this case, like so:

```yaml
spec:
  brokerPodSpec:
    containers:
    - name: akri-onvif-video-broker
      image: "ghcr.io/project-akri/akri/onvif-video-broker:latest-dev"
      resources:
        limits:
          "{{PLACEHOLDER}}" : "1"
    tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
```

#### Modifying the brokerJobSpec

The `brokerJobSpec` property is a full [JobSpec](https://v1-18.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#jobspec-v1-batch) and can be modified as such. Akri's Helm chart enables modifying the `capacity`, `parallelism`, and `backoffLimit` fields of the JobSpec. Other fields of the JobSpec and the PodSpec within the JobSpec can be specified in a similar manner as described in the [modifying the PodSpec section](#Modifying-the-brokerPodSpec).

#### Modifying instanceServiceSpec or configurationServiceSpec

The `instanceServiceSpec` and `configurationServiceSpec` properties are full [ServiceSpecs](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#servicespec-v1-core) and can be modified as such. The simplest reason to modify either might be to specify different ports (perhaps 8085 and 8086):

```yaml
spec:
  instanceServiceSpec:
    ports:
    - name: grpc
      port: 8085
      targetPort: 8083
  configurationServiceSpec:
    ports:
    - name: grpc
      port: 8086
      targetPort: 8083
```

{% hint style="info" %}
The simple properties of `instanceServiceSpec` and `configurationServiceSpec` (like name, port, targetPort, and protocol) can be set using Helm's `--set` command, e.g.`--set onvif.instanceService.targetPort=90`.
{% endhint %}

## Deploying multiple Configurations using `helm install`

If you want your end application to consume frames from both IP cameras and locally attached cameras, Akri can be installed from the start with both the ONVIF and udev Configurations like so:

```bash
helm repo add akri-helm-charts https://project-akri.github.io/akri/
helm install akri akri-helm-charts/akri \
    --set onvif.configuration.enabled=true \
    --set udev.configuration.enabled=true \
    --set udev.configuration.discoveryDetails.udevRules[0]='KERNEL=="video[0-9]*"\, ENV{ID_V4L_CAPABILITIES}==":capture:"'
```

{% hint style="info" %}
You must specify a udev rule to successfully build the udev Configuration.
{% endhint %}

You can confirm that both a `akri-onvif` and `akri-udev` Configuration have been created by running:

```bash
kubectl get akric
```

Each Configuration could also have been deployed via separate Helm installations:

```bash
helm install udev-config akri-helm-charts/akri \
 --set controller.enabled=false \
 --set agent.enabled=false \
 --set rbac.enabled=false \
 --set udev.configuration.enabled=true  \
 --set udev.configuration.discoveryDetails.udevRules[0]='KERNEL=="video[0-9]*"\, ENV{ID_V4L_CAPABILITIES}==":capture:"'

helm install onvif-config akri-helm-charts/akri \
 --set controller.enabled=false \
 --set agent.enabled=false \
 --set rbac.enabled=false \
 --set onvif.configuration.enabled=true
```

## Modifying a deployed Configuration

An already deployed Configuration can be modified in one of two ways: 

1. Using the `helm upgrade` command 
2. [Generating, modifying and applying a custom Configuration](customizing-an-akri-installation.md#generating-modifying-and-applying-a-configuration)

Note: Only the broker properties and capacity of an applied configuration should be modified, for any other modification, you need to delete and reapply the Configuration.

### Using `helm upgrade`

A Configuration can be modified by using the `helm upgrade` command. It upgrades an existing release according to the values provided, only updating what has changed. Simply modify your `helm install` command to reflect the new **desired state** of Akri and replace `helm install` with `helm upgrade`. Using the ONVIF protocol implementation as an example, say you want to set the capacity of discovered cameras to 3:

```bash
helm upgrade akri akri-helm-charts/akri \
    --set onvif.configuration.enabled=true \
    --set onvif.configuration.brokerPod.image.repository=<your broker image name> \
    --set onvif.configuration.brokerPod.image.tag=<your broker image tag> \
    --set onvif.configuration.capacity=3
```

Note that the command is not simply `helm upgrade --set onvif.configuration.capacity=3`; rather, it includes all the old settings along with the new one. Also, note that we assumed you specified a broker pod image in your original installation command, so that brokers were deployed to utilize discovered cameras.

Helm will create a new ONVIF Configuration and apply it to the cluster. When the Agent sees that a Configuration has been updated, it updates all Instances associated with that new Configuration.

## Adding another Configuration to a cluster

Another Configuration can be added to an existing Akri installation using `helm upgrade` or via a new Helm installation.

### Adding additional Configurations using `helm upgrade`

Another Configuration can be added to the cluster by using `helm upgrade`. For example, if you originally installed just the ONVIF Configuration and now also want to discover local cameras via udev, as well, simply run the following:

```bash
helm upgrade akri akri-helm-charts/akri \
    --set onvif.enabled=true \
    --set udev.enabled=true \
    --set udev.udevRules[0]='KERNEL=="video[0-9]*"\, ENV{ID_V4L_CAPABILITIES}==":capture:"'
```

### Adding additional Configurations via new Helm installations

The udev Configuration could also have been applied via a new Helm installation like so:

```bash
helm install udev-config akri-helm-charts/akri \
 --set controller.enabled=false \
 --set agent.enabled=false \
 --set rbac.enabled=false \
 --set udev.configuration.enabled=true  \
 --set udev.configuration.discoveryDetails.udevRules[0]='KERNEL=="video[0-9]*"\, ENV{ID_V4L_CAPABILITIES}==":capture:"'
```

## Modifying a broker

Currently, to modify a broker (be it a Job or Pod), you need to delete and re-apply the Configuration.

## Deleting a Configuration from a cluster

If an operator no longer wants Akri to discover devices defined by a Configuration, they can delete the Configuration and all associated broker pods will automatically be brought down. This can be done with `helm upgrade`, `helm delete`, or kubectl.

### Deleting a Configuration using `helm upgrade`

A Configuration can be deleted from a cluster using `helm upgrade`. For example, if both ONVIF and udev Configurations have been installed in a cluster, the udev Configuration can be deleted by only specifying the ONVIF Configuration in a `helm upgrade` command like the following:

```bash
helm upgrade akri akri-helm-charts/akri \
    --set onvif.enabled=true
```

### Deleting a Configuration using `helm delete`

If the Configuration was applied in its own Helm installation (named `udev-config` in this example), the Configuration can be deleted by deleting the installation.

```bash
helm delete udev-config
```

### Deleting a Configuration using kubectl

A configuration can also be deleted using kubectl. To list all applied Configurations, run `kubectl get akric`. If both udev and ONVIF Configurations have been applied with capacities of 5. The output should look like the following:

```bash
NAME                CAPACITY   AGE
akri-onvif          5          3s
akri-udev           5          16m
```

To delete the ONVIF Configuration and bring down all ONVIF broker pods, run:

```bash
kubectl delete akric akri-onvif
```

## Installing Discovery Handlers

The Agent discovers devices via Discovery Handlers. Akri supports an Agent image that includes all supported Discovery Handlers. This Agent will be used if `agent.full=true`, like so:

```bash
helm install akri akri-helm-charts/akri \
  --set agent.full=true
```

By default, a slim Agent without any embedded Discovery Handlers is deployed and the required Discovery Handlers can be deployed as DaemonSets by specifying `<discovery handler name>.discovery.enabled=true` when installing Akri. For example, Akri is installed with the OPC UA and ONVIF Discovery Handlers like so:

```bash
helm install akri akri-helm-charts/akri \
  --set opcua.discovery.enabled=true \
  --set onvif.discovery.enabled=true
```

