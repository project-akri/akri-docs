# Kubernetes Cluster Setup

Before deploying Akri, you must have a Kubernetes cluster (v1.16 or higher) running with `kubectl` and `Helm` installed. Akri is Kubernetes native, so it should run on most Kubernetes distributions. This document provides cluster setup instructions for the three Kubernetes distributions that all of our end-to-end tests run on.

{% hint style="info" %}
Note: All nodes must be Linux on amd64, arm64v8, or arm32v7.
{% endhint %}

## Install Kubernetes Distribution

{% tabs %}
{% tab title="Kubernetes" %}
1. Reference [Kubernetes documentation](https://kubernetes.io/docs/tasks/tools/) for instructions on how to install Kubernetes. See Akri's [release notes](https://github.com/project-akri/akri/releases) to see what versions of Kubernetes Akri has been tested on.

2. Install Helm for deploying Akri.

   ```bash
    sudo apt install -y curl
    curl -L https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
   ```

{% hint style="info" %}
Note: To enable workloads on a single-node cluster, remove the master taint.

```bash
kubectl taint nodes --all node-role.kubernetes.io/master-
```
{% endhint %}
{% endtab %}

{% tab title="K3s" %}


1. Install [K3s](https://k3s.io/). The following will install the latest K3s version. Reference Akri's [release notes](https://github.com/project-akri/akri/releases) to see what versions of K3s Akri has been tested on.

   ```bash
      curl -sfL https://get.k3s.io | sh -
   ```

   > Note: Optionally specify a version with the `INSTALL_K3S_VERSION` env var as follows: `curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.21.5+k3s1 sh -`

2. Grant admin privilege to access kube config.

   ```bash
    sudo addgroup k3s-admin
    sudo adduser $USER k3s-admin
    sudo usermod -a -G k3s-admin $USER
    sudo chgrp k3s-admin /etc/rancher/k3s/k3s.yaml
    sudo chmod g+r /etc/rancher/k3s/k3s.yaml
    su - $USER
   ```

3. Check K3s status.

   ```bash
    kubectl get node
   ```

4. Install Helm.

   ```bash
    export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
    sudo apt install -y curl
    curl -L https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
   ```

5. If desired, add nodes to your cluster by running the K3s installation script with the `K3S_URL` and `K3S_TOKEN` environment variables. See [K3s installation documentation](https://rancher.com/docs/k3s/latest/en/quick-start/#install-script) for more details.
{% endtab %}

{% tab title="MicroK8s" %}


1. Install [MicroK8s](https://microk8s.io/docs). The following will install the latest MicroK8s version. Add `--channel=$VERSION/stable` to specify as specific Kubernetes version. Reference Akri's [release notes](https://github.com/project-akri/akri/releases) to see what versions of MicroK8s Akri has been tested on.

   ```bash
    snap install microk8s --classic
   ```

2. Grant admin privilege for running MicroK8s commands.

   ```bash
    sudo usermod -a -G microk8s $USER
    sudo chown -f -R $USER ~/.kube
    su - $USER
   ```

3. Check MicroK8s status.

   ```bash
    microk8s status --wait-ready
   ```

4. Enable CoreDNS, Helm and RBAC for MicroK8s.

   ```bash
    microk8s enable dns helm3 rbac
   ```

5. If you don't have an existing `kubectl` and `helm` installations, add aliases. If you do not want to set an alias, add `microk8s` in front of all `kubectl` and `helm` commands.

   ```bash
    alias kubectl='microk8s kubectl'
    alias helm='microk8s helm3'
   ```

6. By default, MicroK8s does not allow Pods to run in a privileged context. None of Akri's components run privileged; however, if your custom broker Pods do in order to access devices for example, enable privileged Pods like so:

   ```bash
    echo "--allow-privileged=true" >> /var/snap/microk8s/current/args/kube-apiserver
    microk8s.stop
    microk8s.start
   ```

7. If desired, reference [MicroK8's documentation](https://microk8s.io/docs/clustering) to add additional nodes to the cluster.
{% endtab %}
{% endtabs %}

## Configure `crictl`
Akri depends on `crictl` to track some Pod information. In order to use it, the Agent must know where the container runtime socket lives. This can be configured with Akri's Helm chart either directly by setting `agent.host.containerRuntimeSocket` or indirectly by specifying the Kubernetes distribution that is being used (`kubernetesDistro=k3s|microk8s|k8s`). If a distribution is specified, then the appropriate default will be used. If no distribution or runtime is specified, the `k8s` default is used.

Akri recommends setting this choice as an `AKRI_HELM_CRICTL_CONFIGURATION` environment variable and then adding the variable to each Akri installation like so:
```sh
  helm install akri akri-helm-charts/akri \
     $AKRI_HELM_CRICTL_CONFIGURATION
```
The following are the recommended settings based on Kubernetes distribution.
{% tabs %}
{% tab title="Kubernetes" %}
To use the default standard Kubernetes container runtime socket `/var/run/dockershim.sock`, set `k8s` as the distribution.
```sh
export AKRI_HELM_CRICTL_CONFIGURATION="--set kubernetesDistro=k8s"
```
{% endtab %}

{% tab title="K3s" %}
To use the default K3s container runtime socket `/run/k3s/containerd/containerd.sock`, set `k3s` as the distribution. 

```bash
export AKRI_HELM_CRICTL_CONFIGURATION="--set kubernetesDistro=k3s"
```
{% endtab %}

{% tab title="MicroK8s" %}
To use the default MicroK8s container runtime socket `/var/snap/microk8s/common/run/containerd.sock`, set `microk8s` as the distribution. 

```bash
export AKRI_HELM_CRICTL_CONFIGURATION="--set kubernetesDistro=microk8s"
```
{% endtab %}
{% tab title="Other" %}
A specific container runtime socket can be set in the `agent.host.containerRuntimeSocket` value. 

```bash
export AKRI_HELM_CRICTL_CONFIGURATION="--set agent.host.containerRuntimeSocket=/container/runtime.sock"
```
{% endtab %}
{% endtabs %}

