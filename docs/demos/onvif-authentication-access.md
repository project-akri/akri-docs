### Passing credentials to Onvif Discovery Handler to discover authenticated devices
Make sure you have at least one Onvif camera that is reachable so Onvif discovery handler can discovery your Onvif camera. To test accessing Onvif with credentials, make sure your Onvif camera is authentication-enabled. **Write down the username and password**, they are required in the flow below.

#### Preparation

Add Akri helm chart repo and set the environment variable `AKRI_HELM_CRICTL_CONFIGURATION` to proper value.

```bash
# add akri helm charts repo
helm repo add akri-helm-charts https://project-akri.github.io/akri/
# ensure helm repos are up-to-date
helm repo update
```
Set up the Kubernetes distribution being used, here we use 'k8s', make sure to replace it with a value that matches the Kubernetes distribution you used.

See [the cluster setup steps](../user-guide/cluster-setup.md#configure-crictl) for information on how to set the crictl configuration variable `AKRI_HELM_CRICTL_CONFIGURATION`

 ```bash
export AKRI_HELM_CRICTL_CONFIGURATION="--set kubernetesDistro=k8s"
```

#### Acquire Onvif camera's device uuid
In real product scenarios, the device uuids are acquired directly from the vendors or already known before installing Akri Configuration.
If you already know the device uuids, you can skip this and go to the next step.

First use the following helm chart to deploy an Akri Configuration and see if your camera is discovered.

```bash
helm install akri akri-helm-charts/akri-dev \
   $AKRI_HELM_CRICTL_CONFIGURATION \
   --set onvif.discovery.enabled=true \
   --set onvif.configuration.name=akri-onvif \
   --set onvif.configuration.enabled=true \
   --set onvif.configuration.capacity=3 \
   --set onvif.configuration.brokerPod.image.repository="nginx" \
   --set onvif.configuration.brokerPod.image.tag="stable-alpine"
```

Here is the result of running the installation command above on a cluster with 1 control plane and 2 work nodes.  There is one Onvif camera connects to the network, thus 1 pods running on each node.

```bash=
$ kubectl get nodes,akric,akrii,pods
NAME           STATUS   ROLES           AGE   VERSION
node/kube-01   Ready    control-plane   22d   v1.26.1
node/kube-02   Ready    <none>          22d   v1.26.1
node/kube-03   Ready    <none>          22d   v1.26.1

NAME                               CAPACITY   AGE
configuration.akri.sh/akri-onvif   3          62s

NAME                                 CONFIG       SHARED   NODES                   AGE
instance.akri.sh/akri-onvif-029957   akri-onvif   true     ["kube-03","kube-02"]   48s

NAME                                              READY   STATUS    RESTARTS   AGE
pod/akri-agent-daemonset-gnwb5                    1/1     Running   0          62s
pod/akri-agent-daemonset-zn2gb                    1/1     Running   0          62s
pod/akri-controller-deployment-56b9796c5-wqdwr    1/1     Running   0          62s
pod/akri-onvif-discovery-daemonset-wcp2f          1/1     Running   0          62s
pod/akri-onvif-discovery-daemonset-xml6t          1/1     Running   0          62s
pod/akri-webhook-configuration-75d9b95fbc-wqhgw   1/1     Running   0          62s
pod/kube-02-akri-onvif-029957-pod                 1/1     Running   0          48s
pod/kube-03-akri-onvif-029957-pod                 1/1     Running   0          48s
```
Get the device uuid from the Akri Instance. Below is an example, the Onvif discovery handler discovers the camera and expose the device's uuid. **Write down the device uuid for later use**. Note that in real product scenarios, the device uuids are acquired directly from the vendors or already known before installing Akri Configuration.

```bash=
$ kubectl get akrii akri-onvif-029957 -o yaml | grep ONVIF_DEVICE_UUID
    ONVIF_DEVICE_UUID: 3fa1fe68-b915-4053-a3e1-ac15a21f5f91
```

#### Set up Kubernetes secrets
Now we can set up the credential information to Kubernetes Secret. Replace the device uuid and the values of username/password with information of your camera.

```bash
cat > /tmp/onvif-auth-secret.yaml<< EOF
---
apiVersion: v1
kind: Secret
metadata:
  name: onvif-auth-secret
type: Opaque
stringData:
  device_credential_list: |+
    [ "credential_list" ]
  credential_list: |+
    {
        "3fa1fe68-b915-4053-a3e1-ac15a21f5f91" :
            {
                "username" : "camuser",
                "password" : "HappyDay"
            }
    }
EOF

# add the secret to cluster
kubectl apply -f /tmp/onvif-auth-secret.yaml

```
#### Upgrade the Akri configuration
Upgrade the Akri Configuration to include the secret information and the sample video broker container.

```bash
helm upgrade akri akri-helm-charts/akri-dev \
   --install \
   $AKRI_HELM_CRICTL_CONFIGURATION \
   --set onvif.discovery.enabled=true \
   --set onvif.configuration.enabled=true \
   --set onvif.configuration.capacity=3 \
   --set onvif.configuration.discoveryProperties[0].name=device_credential_list \
   --set onvif.configuration.discoveryProperties[0].valueFrom.secretKeyRef.name=onvif-auth-secret \
   --set onvif.configuration.discoveryProperties[0].valueFrom.secretKeyRef.namesapce=default \
   --set onvif.configuration.discoveryProperties[0].valueFrom.secretKeyRef.key=device_credential_list \
   --set onvif.configuration.discoveryProperties[0].valueFrom.secretKeyRef.optoinal=false \
   --set onvif.configuration.brokerPod.image.repository="ghcr.io/project-akri/akri/onvif-video-broker" \
   --set onvif.configuration.brokerPod.image.tag="latest-dev" \
   --set onvif.configuration.brokerPod.image.pullPolicy="Always" \
   --set onvif.configuration.brokerProperties.CREDENTIAL_DIRECTORY="/etc/credential_directory" \
   --set onvif.configuration.brokerProperties.CREDENTIAL_CONFIGMAP_DIRECTORY="/etc/credential_cfgmap_directory" \
   --set onvif.configuration.brokerPod.volumeMounts[0].name="credentials" \
   --set onvif.configuration.brokerPod.volumeMounts[0].mountPath="/etc/credential_directory" \
   --set onvif.configuration.brokerPod.volumeMounts[0].readOnly=true \
   --set onvif.configuration.brokerPod.volumes[0].name="credentials" \
   --set onvif.configuration.brokerPod.volumes[0].secret.secretName="onvif-auth-secret"
```

With the secret information, the Onvif discovery handler is able to discovery the Onvif camera and the video broker is up and running
```bash=
$ kubectl get nodes,akric,akrii,pods
NAME           STATUS   ROLES           AGE   VERSION
node/kube-01   Ready    control-plane   22d   v1.26.1
node/kube-02   Ready    <none>          22d   v1.26.1
node/kube-03   Ready    <none>          22d   v1.26.1

NAME                               CAPACITY   AGE
configuration.akri.sh/akri-onvif   3          18m

NAME                                 CONFIG       SHARED   NODES                   AGE
instance.akri.sh/akri-onvif-029957   akri-onvif   true     ["kube-03","kube-02"]   22s

NAME                                              READY   STATUS    RESTARTS   AGE
pod/akri-agent-daemonset-bq494                    1/1     Running   0          18m
pod/akri-agent-daemonset-c2rng                    1/1     Running   0          18m
pod/akri-controller-deployment-56b9796c5-rtm5q    1/1     Running   0          18m
pod/akri-onvif-discovery-daemonset-rbgwq          1/1     Running   0          18m
pod/akri-onvif-discovery-daemonset-xwjlp          1/1     Running   0          18m
pod/akri-webhook-configuration-75d9b95fbc-cr6bc   1/1     Running   0          18m
pod/kube-02-akri-onvif-029957-pod                 1/1     Running   0          22s
pod/kube-03-akri-onvif-029957-pod                 1/1     Running   0          22s

# dump the logs from sample video broker
$ kubectl logs kube-02-akri-onvif-029957-pod
[Akri] ONVIF request http://192.168.1.145:2020/onvif/device_service http://www.onvif.org/ver10/device/wsdl/GetService
[Akri] ONVIF media url http://192.168.1.145:2020/onvif/service
[Akri] ONVIF request http://192.168.1.145:2020/onvif/service http://www.onvif.org/ver10/media/wsdl/GetProfiles
[Akri] ONVIF profile list contains: profile_1
[Akri] ONVIF profile list contains: profile_2
[Akri] ONVIF profile list profile_1
[Akri] ONVIF request http://192.168.1.145:2020/onvif/service http://www.onvif.org/ver10/media/wsdl/GetStreamUri
[Akri] ONVIF streaming uri list contains: rtsp://192.168.1.145:554/stream1
[Akri] ONVIF streaming uri rtsp://192.168.1.145:554/stream1
[VideoProcessor] Processing RTSP stream: rtsp://----:----@192.168.1.145:554/stream1
info: Microsoft.Hosting.Lifetime[0]
      Now listening on: http://[::]:8083
info: Microsoft.Hosting.Lifetime[0]
      Application started. Press Ctrl+C to shut down.
info: Microsoft.Hosting.Lifetime[0]
      Hosting environment: Production
info: Microsoft.Hosting.Lifetime[0]
      Content root path: /app
Ready True
Adding frame from rtsp://----:----@192.168.1.145:554/stream1, Q size: 1, frame size: 862986
Adding frame from rtsp://----:----@192.168.1.145:554/stream1, Q size: 2, frame size: 865793
Adding frame from rtsp://----:----@192.168.1.145:554/stream1, Q size: 2, frame size: 868048
Adding frame from rtsp://----:----@192.168.1.145:554/stream1, Q size: 2, frame size: 869655
Adding frame from rtsp://----:----@192.168.1.145:554/stream1, Q size: 2, frame size: 871353
```
#### Deploying the sample video streaming application
Deploy the sample video streaming application Instructions described from the step 4 of [camera demo](https://docs.akri.sh/demos/usb-camera-demo#inspecting-akri)

Deploy a video streaming web application that points to both the Configuration and Instance level services that were automatically created by Akri.

Copy and paste the contents into a file and save it as `akri-video-streaming-app.yaml`
```bash
cat > /tmp/akri-video-streaming-app.yaml<< EOF
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: akri-video-streaming-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: akri-video-streaming-app
  template:
    metadata:
      labels:
        app: akri-video-streaming-app
    spec:
      serviceAccountName: akri-video-streaming-app-sa
      containers:
      - name: akri-video-streaming-app
        image: ghcr.io/project-akri/akri/video-streaming-app:latest-dev
        imagePullPolicy: Always
        securityContext:
          runAsUser: 1000
          allowPrivilegeEscalation: false
          runAsNonRoot: true
          readOnlyRootFilesystem: true
          capabilities:
            drop: ["ALL"]
        env:
        # Streamer works in two modes; either specify the following commented
        # block of env vars to explicitly target cameras (update the <id>s for
        # your specific cameras) or 
        # specify a Akri configuration name to pick up cameras automatically
        # - name: CAMERAS_SOURCE_SVC
        #   value: "akri-udev-video-svc"
        # - name: CAMERA_COUNT
        #   value: "2"
        # - name: CAMERA1_SOURCE_SVC
        #   value: "akri-udev-video-<id>-svc"
        # - name: CAMERA2_SOURCE_SVC
        #   value: "akri-udev-video-<id>-svc"
        - name: CONFIGURATION_NAME
          value: akri-onvif
---
apiVersion: v1
kind: Service
metadata:
  name: akri-video-streaming-app
  namespace: default
  labels:
    app: akri-video-streaming-app
spec:
  selector:
    app: akri-video-streaming-app
  ports:
  - name: http
    port: 80
    targetPort: 5000
  type: NodePort
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: akri-video-streaming-app-sa
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: akri-video-streaming-app-role
rules:
- apiGroups: [""]
  resources: ["services"]
  verbs: ["list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: akri-video-streaming-app-binding
roleRef:
  apiGroup: ""
  kind: ClusterRole
  name: akri-video-streaming-app-role
subjects:
  - kind: ServiceAccount
    name: akri-video-streaming-app-sa
    namespace: default
EOF
```

Deploy the video stream app

```bash
kubectl apply -f /tmp/akri-video-streaming-app.yaml
```
 
Determine which port the service is running on. **Save this port number for the next step**:
```bash
kubectl get service/akri-video-streaming-app --output=jsonpath='{.spec.ports[?(@.name=="http")].nodePort}' && echo
```

SSH port forwarding can be used to access the streaming application. Open a new terminal, enter your ssh command to to access your machine followed by the port forwarding request. The following command will use port 50000 on the host. Feel free to change it if it is not available. Be sure to replace `<streaming-app-port>` with the port number outputted in the previous step.

```bash=
ssh someuser@<machine IP address> -L 50000:localhost:<streaming-app-port>
```
Navigate to http://localhost:50000/ using browser. The large feed points to Configuration level service, while the bottom feed points to the service for each Instance or camera.

#### Clean up
Close the page http://localhost:50000/ from the browser

Delete the sample streaming application resources
```bash
kubectl delete -f /tmp/akri-video-streaming-app.yaml
```

Delete the Secret information
```bash
kubectl delete -f /tmp/onvif-auth-secret.yaml
```

Delete deployment and Akri installation to clean up the system.
```bash
helm delete akri
kubectl delete crd configurations.akri.sh
kubectl delete crd instances.akri.sh
```

