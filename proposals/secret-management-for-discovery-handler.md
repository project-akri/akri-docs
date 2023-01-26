# Support Akri Agent to pass K8s Secrets to Discovery Handlers
The proposal is to have Agent retrieve secrets or other data and pass it to DH (Discovery Handler) through the 
```DiscoveryHandler``` service (the secret/data will be part of the ```DiscoverRequest```).  This additional data, named 
‘discover property’ will in the format of string key-value pair list.  Once DH received the discovery properties and 
discovery details, it should be able to perform discovery tasks.  The Agent will watch any discovery property changes 
from the discovery property data source and re-issue a ```DiscoverRequest``` if any of the data changed.

The Discovery properties will be specified as part of the Akri configuration, here is an example.  The 
```discoveryProperties``` section is newly added for specifying the secret information. Current plan is to support 
both k8s ```secret``` and k8s ```configMap```. The ```discoveryProperties``` is optional.  If 
```discoveryProperties``` is not specified, it falls back to what we have currently, just ```discoveryDetails``` 
passed to DH.

The key-value pair in ```discoveryProperties``` is a list and can be as many entries as needed.

The key-value pair can be optional. The ```optional``` attribute in the ```discoveryProperties``` entry is default to 
```false```, it means if the key doesn't exist in the ```secret``` (or ```configMap```), the Configuration deployment 
will fail.  If ```optional``` is ```true``` and the secret key doesn't exist, Agent will not add the entry to the list 
passed to DH, the Configuration deployment will success.

## Secret data changes after deploying the Akri Configuration
Current plan Akri Agent will not monitor the secret data changes after the Akri Configuration is deployed.  If the 
content of ```discoveryProperties``` changed from the data sources after Configuration deployed, users need to 
manually update the Akri Configuration.  If necessary, we can make improvement to have Akri Agent monitors the secret 
data source and re-issue discoverRequest if the data source changed.  For example, username and password are stored in 
a k8s secret and are specified in the ```discoverProperties``` as mandatory in an Akri Configuration.  After the 
configuration deployed, Agent can watch any change to these two properties in k8s secret and re-issue discoverRequest 
if they changed.  If the mandatory property is deleted from the data store, the configuration is considered invalid, 
and Akri Agent will revoke the discoveryRequest from DH.

```yaml
apiVersion: akri.sh/v0
kind: Configuration
metadata:
  name: akri-onvif-video
spec:
  discoveryHandler:
    name: onvif
    discoveryDetails: |+
        ipAddresses:
          action: Exclude
          items:
          - 10.0.0.1
          - 10.0.0.2
        macAddresses:
          action: Exclude
          items: []
        scopes:
          action: Include
          items:
          - onvif://www.onvif.org/name/GreatONVIFCamera
          - onvif://www.onvif.org/name/AwesomeONVIFCamera
        discoveryTimeoutSeconds: 2
    discoveryProperties:
      - name: dev-common-data
        value: “plain text data”
      - name: dev-auth-username
        valueFrom:
          secretKeyRef:
            name: mysecret
            key: username
            optional: false
      - name: dev-auth-password
        valueFrom:
          secretKeyRef:
            name: mysecret
            key: password
            optional: true
      - name: dev-configData1
        valueFrom:
          configMapKeyRef:
            name: myconfigMap
            key: configOption1
            optional: false
      - name: dev-configData2
        valueFrom:
          configMapKeyRef:
            name: myconfigMap
            key: configOption2
            optional: true
  brokerSpec:
    brokerPodSpec:
      containers:
      - name: akri-onvif-video-broker
        image: "ghcr.io/project-akri/akri/onvif-video-broker:latest-dev"
        imagePullPolicy: Always
        resources:
          limits:
            "{{PLACEHOLDER}}" : "1"
  instanceServiceSpec:
    ports:
    - name: grpc
      port: 80
      targetPort: 8083
  configurationServiceSpec:
    ports:
    - name: grpc
      port: 80
      targetPort: 8083
  brokerProperties: {}
  capacity: 5
```

## K8s Secret Encryption
K8s Secrets values are encoded as base64 strings and are stored unencrypted by default.  Cluster owner should 
configure the cluster to encrypt the Secret objects.  Akri accesses the Secret objects via K8s resource APIs and 
relies on the cluster owner to encrypt Secret objects and arrange access permission properly to ensure Secret objects 
are secured and reduce the risk of accidental exposure.  See [Good practices for Kubernetes Secrets | Kubernetes](https://kubernetes.io/docs/concepts/security/secrets-good-practices/) 
for good practices to enhance the security of K8s Secrets.

## Store K8s Secret in the Cloud
Depends on the cluster configuration, the sensitive data might be stored in the cloud and cluster owner may want to 
pull secrets from a cloud backed key management service (e.g., Azure KeyVault, AWS, or Hashicorp Vault). Cluster owner 
can enable the secret-store-csi-driver on their cluster to mount their existing KMS backed secrets/certs as a native 
k8s secret.  For example, Azure Key Vault provider for [Secrets Store CSI Driver](https://nam06.safelinks.protection.outlook.com/?url=https%3A%2F%2Fgithub.com%2Fkubernetes-sigs%2Fsecrets-store-csi-driver&data=05%7C01%7Cjshih%40microsoft.com%7Cff3661f4074b4df3db0d08dac9ac531d%7C72f988bf86f141af91ab2d7cd011db47%7C1%7C0%7C638044039217629223%7CUnknown%7CTWFpbGZsb3d8eyJWIjoiMC4wLjAwMDAiLCJQIjoiV2luMzIiLCJBTiI6Ik1haWwiLCJXVCI6Mn0%3D%7C3000%7C%7C%7C&sdata=d4eUzlxUsy69fru9Ebe8yYOAo8WHD81HoMMiR1BzFU8%3D&reserved=0) allows you to get secret contents stored in an [Azure Key Vault](https://nam06.safelinks.protection.outlook.com/?url=https%3A%2F%2Fdocs.microsoft.com%2Fen-us%2Fazure%2Fkey-vault%2Fgeneral%2Foverview&data=05%7C01%7Cjshih%40microsoft.com%7Cff3661f4074b4df3db0d08dac9ac531d%7C72f988bf86f141af91ab2d7cd011db47%7C1%7C0%7C638044039217629223%7CUnknown%7CTWFpbGZsb3d8eyJWIjoiMC4wLjAwMDAiLCJQIjoiV2luMzIiLCJBTiI6Ik1haWwiLCJXVCI6Mn0%3D%7C3000%7C%7C%7C&sdata=Et9doGA1CEgONnsPwnlGu2%2BY6dIsDgexTHQYXCSLoZQ%3D&reserved=0) instance and use the Secrets Store CSI driver interface to mount them into Kubernetes pods.  Akri relies on the proper setup to access Secrets backed by cloud storage through K8s resource APIs.
