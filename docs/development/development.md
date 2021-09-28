# Developer Guide
This document will walk you through how to set up a local development environment, build Akri component containers, and test Akri using your newly built containers. It also includes instructions on running Akri locally, naming guidelines, and points to our documentation on extending Akri with new Discovery Handlers and brokers.

> Note: different tools are needed depending on what parts of Akri you are developing. This document aims to make that clear.

## Table of Contents
- [Requirements](#requirements)
- [Build and Test Akri's Components](#Build-and-test-Akri's-Rust-components)
- [Running Akri's Components Locally](#Running-Akri's-Components-Locally)
- [Building Akri Containers](#building-akri-containers)
- [Installing Akri with newly built containers](#Installing-Akri-with-newly-built-containers)
- [Useful Helm commands](#useful-Helm-commands)
- [Testing with Debug Echo Discovery Handler](#Testing-with-Debug-Echo-Discovery-Handler)
- [Discovery Handler and Broker Development](#Discovery-Handler-and-Broker-Development)
- [Developing Akri's non-Rust components](#developing-akri's-non-rust-components)
- [Naming Guidelines](Naming-Guidelines)

## Requirements
### Linux Environment
To develop, you'll **need a Linux environment** whether on amd64 or arm64v8. We recommend using an Ubuntu VM; however, WSL2 should work for building and testing (but has not been extensively tested).

### Tools for developing Akri's Rust components
The majority of Akri is written in Rust. To install Rust and Akri's component's depencies, run Akri's setup script:
```sh
./build/setup.sh
```
If you previously installed Rust ensure you are using the v1.54.0 toolchain that Akri's build system uses:
```sh
sudo curl https://sh.rustup.rs -sSf | sh -s -- -y --default-toolchain=1.54.0
rustup default 1.54.0
cargo version
``` 

## Build and test Akri's Rust components 
1. Clone [Akri](https://github.com/deislabs/akri) and navigate to the repo's top folder

1. To install Rust and Akri's component's depencies, run Akri's setup script:
    ```sh
    ./build/setup.sh
    ```

    If you previously installed Rust ensure you are using the v1.54.0 toolchain that Akri's build system uses:
    ```sh
    sudo curl https://sh.rustup.rs -sSf | sh -s -- -y --default-toolchain=1.54.0
    rustup default 1.54.0
    cargo version
    ``` 

1. Build Controller, Agent, Discovery Handlers, and udev broker
    ```sh
    cargo build
    ```

    > Note: To build a specific component, use the `-p` paramenter along with the [workspace member](https://github.com/deislabs/akri/blob/main/Cargo.toml). For example, to only build the Agent, run `cargo build -p agent`

1. To run all unit tests:
    ```sh
    cargo test
    ```

    > Note: To test a specific component, use the `-p` paramenter along with the [workspace member](https://github.com/deislabs/akri/blob/main/Cargo.toml). For example, to only test the Agent, run `cargo test -p agent`

## Running Akri's components locally
To locally run Akri's Agent, Controller, and Discovery Handlers as part of a Kubernetes cluster, follow these steps:

1.  Create or provide access to a valid cluster configuration by setting `KUBECONFIG` (can be done in the command line) ...
    for the sake of this, the config is assumed to be in `$HOME/.kube/config`. Reference Akri's [cluster setup instructions](https://docs.akri.sh/user-guide/cluster-setup) if needed.
    
1.  Build the repo with all default features by running `cargo build`

1.  Run the desired component by navigating to the appropriate directory and using `cargo run`

    Run the **Controller** locally with info-level logging and using `8081` to serve Akri's metrics (for Prometheus integration):

    ```sh
    cd akri/controller
    RUST_LOG=info METRICS_PORT=8081 KUBECONFIG=$HOME/.kube/config cargo run
    ```

    > `METRICS_PORT` can be set to any value as it is only used if Prometheus is enabled. Just ensure that the Controller and Agent 
    > use different ports if they are both running. 

    Run the **Agent** locally with info-level logging, debug echo enabled for testing, and a metrics port of `8082`. The Agent must be run privileged in order to connect to the kubelet. Specify the user path to cargo `$HOME/.cargo/bin/cargo` so you do not have to re-install cargo for the sudo user: 
    
    ```sh
    cd akri/agent
    sudo -E DEBUG_ECHO_INSTANCES_SHARED=true ENABLE_DEBUG_ECHO=1 RUST_LOG=info METRICS_PORT=8082 KUBECONFIG=$HOME/.kube/config DISCOVERY_HANDLERS_DIRECTORY=~/tmp/akri AGENT_NODE_NAME=myNode HOST_CRICTL_PATH=/usr/bin/crictl HOST_RUNTIME_ENDPOINT=/var/run/dockershim.sock HOST_IMAGE_ENDPOINT=/var/run/dockershim.sock $HOME/.cargo/bin/cargo run
    ```
    By default, the Agent does not have embedded Discovery Handlers. To allow embedded Discovery Handlers in the
    Agent, turn on the `agent-full` feature and the feature for each Discovery Handler you wish to embed -- Debug echo
    is always included if `agent-full` is turned on. For example, to run the Agent with OPC UA, ONVIF, udev, and
    debug echo Discovery Handlers add the following to the above command: `--features "agent-full udev-feat
    opcua-feat onvif-feat"`.

    > Note: The environment variables `HOST_CRICTL_PATH`, `HOST_RUNTIME_ENDPOINT`, and `HOST_IMAGE_ENDPOINT` are for
    > slot-reconciliation (making sure Pods that no longer exist are not still claiming Akri resources). The values of
    > these vary based on Kubernetes distribution. The above is for vanilla Kubernetes. For MicroK8s, use
    > `HOST_CRICTL_PATH=/usr/local/bin/crictl HOST_RUNTIME_ENDPOINT=/var/snap/microk8s/common/run/containerd.sock
    > HOST_IMAGE_ENDPOINT=/var/snap/microk8s/common/run/containerd.sock` and for K3s, use
    > `HOST_CRICTL_PATH=/usr/local/bin/crictl HOST_RUNTIME_ENDPOINT=/run/k3s/containerd/containerd.sock
    > HOST_IMAGE_ENDPOINT=/run/k3s/containerd/containerd.sock`.

    To run **Discovery Handlers** locally, simply navigate to the Discovery Handler under `akri/discovery-handler-modules/` and run using `cargo run`, setting where the Discovery Handler socket should be created in the `DISCOVERY_HANDLERS_DIRECTORY` variable. The discovery handlers must be run privileged in order to connect to the Agent. For example, to run the ONVIF Discovery Handler locally:
    ```sh
    cd akri/discovery-handler-modules/onvif-discovery-handler/
    sudo -E RUST_LOG=info DISCOVERY_HANDLERS_DIRECTORY=~/tmp/akri AGENT_NODE_NAME=myNode $HOME/.cargo/bin/cargo run
    ```
    To run the [debug echo Discovery Handler](#testing-with-debug-echo-discovery-handler), an environment variable,
    `DEBUG_ECHO_INSTANCES_SHARED`, must be set to specify whether it should register with the Agent as discovering
    shared or unshared devices. Run the debug echo Discovery Handler to discover mock unshared devices like so:
    ```sh
    cd akri/discovery-handler-modules/debug-echo-discovery-handler/
    RUST_LOG=info DEBUG_ECHO_INSTANCES_SHARED=false DISCOVERY_HANDLERS_DIRECTORY=~/tmp/akri AGENT_NODE_NAME=myNode $HOME/.cargo/bin/cargo run
    ```

## Building Akri's Containers
`Makefile` has been created to help with the more complicated task of building the Akri components and containers for the various supported platforms.

### Tools for building Akri's Rust containers
In order to cross-build Akri's Rust code for both ARM and x64 containers, several tools are leveraged.
- The [Cross](https://github.com/rust-embedded/cross) tool can be installed with this command: `cargo install cross`.
- `qemu` can be installed with:
  ```sh
  sudo apt-get install -y qemu qemu qemu-system-misc qemu-user-static qemu-user binfmt-support
  ```

  For `qemu` to be fully configured on Ubuntu 18.04, after running apt-get install, run these commands:
  ```sh
    sudo mkdir -p /lib/binfmt.d
    sudo sh -c 'echo :qemu-arm:M::\\x7fELF\\x01\\x01\\x01\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x02\\x00\\x28\\x00:\\xff\\xff\\xff\\xff\\xff\\xff\\xff\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\xfe\\xff\\xff\\xff:/usr/bin/qemu-arm-static:F > /lib/binfmt.d/qemu-arm-static.conf'
    sudo sh -c 'echo :qemu-aarch64:M::\\x7fELF\\x02\\x01\\x01\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x02\\x00\\xb7\\x00:\\xff\\xff\\xff\\xff\\xff\\xff\\xff\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\x00\\xfe\\xff\\xff\\xff:/usr/bin/qemu-aarch64-static:F > /lib/binfmt.d/qemu-aarch64-static.conf'
    sudo systemctl restart systemd-binfmt.service
  ```

### Establish a container repository
Containers for Akri are currently hosted in `ghcr.io/deislabs/akri` using the new [GitHub container registry](https://github.blog/2020-09-01-introducing-github-container-registry/). Any container repository can be used for private containers. If you want to enable GHCR, you can follow the [getting started guide](https://docs.github.com/en/free-pro-team@latest/packages/getting-started-with-github-container-registry).

To build containers, log into the desired repository:
```sh
CONTAINER_REPOSITORY=<repo>
sudo docker login $CONTAINER_REPOSITORY
```

### Build intermediate containers
To ensure quick builds, we have created a number of intermediate containers that rarely change.

By default, `Makefile` will try to create containers with tag following this format: `<repo>/$USER/<component>:<label>` where

* `<component>` = rust-crossbuild | opencv-base
* `<repo>` = `devcaptest.azurecr.io`
  * `<repo>` can be overridden by setting `REGISTRY=<desired repo>`
* `$USER` = the user executing `Makefile` (could be `root` if using sudo)
  * `<repo>/$USER` can be overridden by setting `PREFIX=<desired container path>`
* `<label>` = the label is defined in [../build/intermediate-containers.mk](https://github.com/deislabs/akri/blob/main/build/intermediate-containers.mk)

#### Rust cross-build containers
These containers are used by the `cross` tool to crossbuild the Akri Rust code.  There is a container built for each supported platform and they contain any required dependencies for Akri components to build.  The dockerfile can be found here: build/containers/intermediate/Dockerfile.rust-crossbuild-*
  ```sh
  # To make all of the Rust cross-build containers:
  PREFIX=$CONTAINER_REPOSITORY make rust-crossbuild
  # To make specific platform(s):
  PREFIX=$CONTAINER_REPOSITORY BUILD_AMD64=1 BUILD_ARM32=1 BUILD_ARM64=1 make rust-crossbuild
  ```
After building the cross container(s), update [Cross.toml](https://github.com/deislabs/akri/blob/main/Cross.toml) to point to your intermediate container(s).

#### .NET OpenCV containers
These containers allow the ONVIF broker to be created without rebuilding OpenCV for .NET each time.  There is a container built for AMD64 and it is used to crossbuild to each supported platform.  The dockerfile can be found here: build/containers/intermediate/Dockerfile.opencvsharp-build.
  ```sh
  # To make all of the OpenCV base containers:
  PREFIX=$CONTAINER_REPOSITORY make opencv-base
  # To make specific platform(s):
  PREFIX=$CONTAINER_REPOSITORY BUILD_AMD64=1 BUILD_ARM32=1 BUILD_ARM64=1 make opencv-base
  ```

### Build Akri component containers
By default, `Makefile` will try to create containers with tag following this format: `<repo>/$USER/<component>:<label>` where

* `<component>` = controller | agent | etc
* `<repo>` = `devcaptest.azurecr.io`
  * `<repo>` can be overridden by setting `REGISTRY=<desired repo>`
* `$USER` = the user executing `Makefile` (could be `root` if using sudo)
  * `<repo>/$USER` can be overridden by setting `PREFIX=<desired container path>`
* `<label>` = v$(cat version.txt)
  * `<label>` can be overridden by setting `LABEL_PREFIX=<desired label>`

**Note**: Before building these final component containers, make sure you have built any necessary [intermediate containers](./#build-intermediate-containers). In particular, to build any of the rust containers (the Controller, Agent, or udev broker), you must first [build the cross-build containers](./#rust-cross-build-containers).

```sh
# To make all of the Akri containers:
PREFIX=$CONTAINER_REPOSITORY make akri
# To make a specific component:
PREFIX=$CONTAINER_REPOSITORY make akri-controller
PREFIX=$CONTAINER_REPOSITORY make akri-udev
PREFIX=$CONTAINER_REPOSITORY make akri-onvif
PREFIX=$CONTAINER_REPOSITORY make akri-opcua-monitoring
PREFIX=$CONTAINER_REPOSITORY make akri-anomaly-detection
PREFIX=$CONTAINER_REPOSITORY make akri-streaming
PREFIX=$CONTAINER_REPOSITORY make akri-agent
# To make an Agent with embedded Discovery Handlers, turn on the `agent-full` feature along with the 
# feature for any Discovery Handlers that should be embedded.
PREFIX=$CONTAINER_REPOSITORY BUILD_SLIM_AGENT=0 AGENT_FEATURES="agent-full onvif-feat opcua-feat udev-feat" make akri-agent-full 

# To make a specific component on specific platform(s):
PREFIX=$CONTAINER_REPOSITORY BUILD_AMD64=1 BUILD_ARM32=1 BUILD_ARM64=1 make akri-streaming

# To make a specific component on specific platform(s) with a specific label:
PREFIX=$CONTAINER_REPOSITORY LABEL_PREFIX=latest BUILD_AMD64=1 BUILD_ARM32=1 BUILD_ARM64=1 make akri-streaming
```

**NOTE:** If your docker install requires you to use `sudo`, this will conflict with the `cross` command.  This flow has helped:
```sh
sudo -s
source /home/$SUDO_USER/.cargo/env

# run make commands that crossbuild the Rust

exit
```

### More information about Akri build
For more detailed information about the Akri build infrastructure, review the [Akri Container building document](./building.md)

## Installing Akri with newly built containers
When installing Akri using helm, you can set the `imagePullSecrets`, `image.repository` and `image.tag` [Helm values](https://github.com/deislabs/akri/blob/main/deployment/helm/values.yaml) to point to your newly created containers. For example, to install Akri with with custom Controller and Agent containers, run the following, specifying the `image.tag` version to reflect [version.txt](https://github.com/deislabs/akri/blob/main/version.txt):
```bash
kubectl create secret docker-registry <your-secret-name> --docker-server=ghcr.io  --docker-username=<your-github-alias> --docker-password=<your-github-token>
helm repo add akri-helm-charts https://deislabs.github.io/akri/
helm install akri akri-helm-charts/akri-dev \
    --set imagePullSecrets[0].name="<your-secret-name>" \
    --set agent.image.repository="ghcr.io/<your-github-alias>/agent" \
    --set agent.image.tag="v<akri-version>-amd64" \
    --set controller.image.repository="ghcr.io/<your-github-alias>/controller" \
    --set controller.image.tag="v<akri-version>-amd64"
```

More information about the Akri Helm charts can be found in the [user guide](../user-guide/getting-started.md#understanding-akri-helm-charts).

## Useful Helm Commands
### Helm Package
If you make changes to anything in the [helm folder](https://github.com/deislabs/akri/tree/main/deployment/helm), you will probably need to create a new Helm chart for Akri. This can be done using the [`helm package`](https://helm.sh/docs/helm/helm_package/) command. To create a chart using the current state of the Helm templates and CRDs, run (from one level above the Akri directory) `helm package akri/deployment/helm/`. You will see a tgz file called `akri-<akri-version>.tgz` at the location where you ran the command. Now, install Akri using that chart:
```sh
helm install akri akri-<akri-version>.tgz \
    --set useLatestContainers=true
```
### Helm Template
When you install Akri using Helm, Helm creates the DaemonSet, Deployment, and Configuration yamls for you (using the values set in the install command) and applies them to the cluster. To inspect those yamls before installing Akri, you can use [`helm template`](https://helm.sh/docs/helm/helm_template/). 
For example, you will see the image in the Agent DaemonSet set to `image: "ghcr.io/<your-github-alias>/agent:v<akri-version>-amd64"` if you run the following:
```sh
helm template akri deployment/helm/ \
  --set imagePullSecrets[0].name="<your-secret-name>" \
  --set agent.image.repository="ghcr.io/<your-github-alias>/agent" \
  --set agent.image.tag="v<akri-version>-amd64"
```

### Helm Get Manifest
Run the following to inspect an already running Akri installation in order to see the currently applied yamls such as the Configuration CRD, Instance CRD, protocol Configurations, Agent DaemonSet, and Controller Deployment:
```sh
helm get manifest akri | less
```

### Helm Upgrade
To modify an Akri installation to reflect a new state, you can use [`helm upgrade`](https://helm.sh/docs/helm/helm_upgrade/). See the [Customizing an Akri Installation document](../user-guide/customizing-an-akri-installation.md) for further explanation. 

## Testing with Debug Echo Discovery Handler
In order to kickstart using and debugging Akri, a debug echo Discovery Handler has been created. See its
[documentation](./debugging.md) to start using it.

## Discovery Handler and Broker Development
Akri was made to be easily extensible as Discovery Handlers and brokers can be implemented in any language and deployed in their own Pods. Reference the [Discovery Handler development](handler-development.md) and [broker Pod development](broker-development.md) documents to get started, or if you prefer to learn by example, reference the [extending Akri walkthrough](./development-walkthrough).

## Developing Akri's non-Rust components
This document focuses on developing Akri's Rust components; however, Akri has several non-Rust components. Reference their respective READMEs in [Akri's source code](https://github.com/deislabs/akri) for instructions on developing.
- Several [sample brokers](https://github.com/deislabs/akri/tree/main/samples/brokers) and [applications](https://github.com/deislabs/akri/tree/main/samples/apps) for demo purposes.
- A [certificate generator](https://github.com/deislabs/akri/tree/main/samples/opcua-certificate-generator) for testing and using Akri's OPC UA Discovery Handler
- Python script for running [end-to-end integration tests](https://github.com/deislabs/akri/blob/main/test/run-end-to-end.py).
- Python script for [testing Akri's Configuration validation webhook](https://github.com/deislabs/akri/blob/main/test/run-webhook.py).

## Naming Guidelines

One of the [two hard things](https://martinfowler.com/bliki/TwoHardThings.html) in Computer Science is naming things. It is proposed that Akri adopt naming guidelines to make developers' lives easier by providing consistency and reduce naming complexity.

Akri existed before naming guidelines were documented and may not employ the guidelines summarized here. However, it is hoped that developers will, at least, consider these guidelines when extending Akri.

### General Principles

+ Akri uses English
+ Akri is written principally in Rust, and Rust [naming](https://rust-lang.github.io/api-guidelines/naming.html) conventions are used
+ Types need not be included in names unless ambiguity would result
+ Shorter, simpler names are preferred

### Akri Discovery Handlers

Various Discovery Handlers have been developed: `debug_echo`, `onvif`, `opcua`, `udev`

Guidance:

+ `snake_case` names
+ (widely understood) initializations|acronyms are preferred

### Akri Brokers

Various Brokers have been developed: `onvif-video-broker`, `opcua-monitoring-broker`, `udev-video-broker`

Guidance:

+ Broker names should reflect Discovery Handler (Protocol) names and be suffixed `-broker`
+ Use Programming language-specific naming conventions when developing Brokers in non-Rust languages

> **NOTE** Even though the initialization of [ONVIF](https://en.wikipedia.org/wiki/ONVIF) includes "Video", the specification is broader than video and the broker name adds specificity by including the word (`onvif-video-broker`) in order to effectively describe its functionality.

### Kubernetes Resources

Various Kubernetes Resources have been developed:

+ CRDS: `Configurations`, `Instances`
+ Instances: `akri-agent-daemonset`, `akri-controller-deployment`, `akri-onvif`, `akri-opcua`, `akri-udev`

Guidance:

+ Kubernetes Convention is that resources (e.g. `DaemonSet`) and CRDs use (upper) CamelCase
+ Akri Convention is that Akri Kubernetes resources be prefixed `akri-`, e.g. `akri-agent-daemonset`
+ Names combining words should use hypens (`-`) to separate the words e.g. `akri-debug-echo`

> **NOTE** `akri-agent-daemonset` contradicts the general principle of not including types, if it had been named after these guidelines were drafted, it would be named `akri-agent`.
>
> Kubernetes' resources are strongly typed and the typing is evident through the CLI e.g. `kubectl get daemonsets/akri-agent-daemonset` and through a resource's `Kind` (e.g. `DaemonSet`). Including such types in the name is redundant.
