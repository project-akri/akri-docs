# Building Akri Containers

Building Akri containers, whether locally or in the automated CI builds, leverages the same set of Dockerfiles. In order to help with local development, a set of `Makefile` exists.

The Makefiles are using `docker buildx` behind the scenes, ensure you have it installed, if you want to build for foreign architectures, you must also ensure you have correctly set up your docker builder to do so (see [Docker buildx documentation](https://github.com/docker/buildx#building-multi-platform-images))

In essence, Akri components can be thought of as:

1. Runtime components
    1. Rust code: containers based on Rust code are built using `Cargo cross` and subsequent `docker build` commands include the cross-built binaries.
        > Note: For Rust code, `build/Dockerfile.*` does NOT run `cargo build`, instead they simply copy cross-built binaries into the container
    2. Other code: these containers can be .NET or python or whatever else ... the `build/Dockerfile.*` must do whatever building is required.
2. Intermediate components: these containers are used as part of the build process and are not used in production explicitly

## Akri components

The Akri core components are the containers that provide Akri's functionality. They include the agent, the controller, the webhook and the discovery handlers. All of these are written in Rust.

The samples containers are a set of brokers and applications written either in .NET, python or Rust that are used in this documentation examples.

All components are built with a `make` command. These are the supporting Makefiles:

* `Makefile`: this provides a single point of entry to build any Akri component
* `build/akri-containers.mk`: this provides the build and push functionality for Akri core containers
* `build/samples.mk`: this provides the build and push functionality for containers used in the samples and documentation
* `build/intermediate-container.mk`: this provides the build and push functionality for the opcvsharp base container

### Configurability

The makefiles allow for several configurations:

* PUSH: if set, the make commands will push the built container images to the registry
* LOAD: if set, the make command will load the built container images into the local docker daemon
* PLATFORMS: space separated list of architectures to build for (default to local architecture in LOAD mode, and to `"amd64 arm64 arm/v7"` otherwise)
* REGISTRY: allows configuration of the container registry (defaults to imaginary: devcaptest.azurecr.io)
* UNIQUE\_ID: allows configuration of container registry account (defaults to $USER)
* PREFIX: allows configuration of container registry path for containers
* LABEL\_PREFIX: allows configuration of container labels

### Local development usage

For a local build, some typical patterns are:

* `make akri`: build akri core container images for all architectures (build only, no push nor load)
* `make akri PLATFORMS=arm64`: build akri core containers for ARM64 (build only, no push nor load)
* `make akri PREFIX=ghcr.io/myaccount PUSH=1`: builds all of the Akri core containers and stores them in a container registry, `ghcr.io/myaccount`.
* `make akri PREFIX=ghcr.io/myaccount LABEL_PREFIX=local PUSH=1`: builds all of the Akri containers and stores them in a container registry, `ghcr.io/myaccount` with labels set to `local`.
* `make akri PREFIX=ghcr.io/myaccount PLATFORMS=amd64`: builds all of the Akri containers for AMD64 and stores them in a container registry, `ghcr.io/myaccount`.
* `make akri-controller PREFIX=ghcr.io/myaccount PUSH=1`: builds the Akri controller container for all platforms and stores them in a container registry, `ghcr.io/myaccount`.
* `make akri LOAD=1`: build akri core containers for the local architecture and load them into the docker daemon

### make targets

Here is the list of supported make targets:

* `all`: builds all core samples and intermediate container images
* `push`: shortcut for `all PUSH=1`
* `load`: shortcut for `all LOAD=1`
* `akri`: builds all core container images
* `samples`: builds all samples container images
* `akri-<component>`: builds the container image for this specific core component, core components are currently one of these: agent, agent-full, controller, webhook-configuration, debug-echo-discovery-handler, onvif-discovery-handler, opcua-discovery-handler, udev-discovery-handler
* `<sample-name>`: builds this specific sample container image, can be one of: opcua-monitoring-broker, onvif-video-broker, akri-udev-video-broker, anomaly-detection-app, video-streaming-app
* `opencv-base`: see [opencvsharp-build](#opencvsharp-build)

### Adding a new component

To add a new Rust-based component, follow these steps:

1. Add the new component to `build/akri-containers.mk` as a dependency to the `akri` target
1. Add the new component to the list of components to build in the `build-others` job of `.github/workflows/build-rust-containers.yml`

## Intermediate components

These are the intermediate components:

* [opencvsharp-build](https://github.com/orgs/project-akri/packages/container/package/akri%2Fopencvsharp-build)

### opencvsharp-build

This container is used by the [onvif-video-broker](https://github.com/orgs/project-akri/packages/container/package/akri%2Fonvif-video-broker) as part of its build process. The main purpose of this container is to prevent each build from needing to build the OpenCV C\# platform. This container can be built locally for all platforms using this command:

```bash
make opencv-base
```

If a change needs to be made to this container, 2 pull requests are needed.

1. Create PR with desired `opencvsharp-build` changes (new dependencies, etc) AND update `BUILD_OPENCV_BASE_VERSION` in `build/intermediate-containers.mk`. This PR is intended to create the new version of `opencvsharp-build` (not to use it).
1. After 1st PR is merged and the new version of `opencvsharp-build` is pushed to ghcr.io/akri, create PR with any changes that will leverage the new version of `opencvsharp-build` AND update `USE_OPENCV_BASE_VERSION` in `build/samples.mk`. This PR is intended to **use** the new version of `opencvsharp-build`.

## Automated builds usage

The automated CI builds are using several jobs and leverages the docker build-push action, but it is equivalent to:

```bash
# Build and push all images on ghcr.io/project-akri using v<version>-dev label
make push PREFIX="ghcr.io/project-akri" LABEL_PREFIX="v$(cat version.txt)-dev" 
```

## Build and run Akri without a Container Registry

For development and/or testing, it can be convenient to run Akri without a Container Registry. For example, the Akri CI tests that validate pull requests build Akri components locally, store the containers only in local docker, and configure Helm to only use the local docker containers.

There are two steps to this. For the sake of this demonstration, only the local architecture version of the agent and controller will be built, but this method can be extended to any and all components:

1. Build:

```bash
    # PREFIX can be anything, as long as it matches what is specified in the Helm command
    PREFIX=no-container-registry
    # LABEL_PREFIX can be anything, as long as it matches what is specified in the Helm command
    LABEL_PREFIX=dev
    # Build and load the controller and the agent
    make akri-controller akri-agent LOAD=1
```

1. Runtime

   ```bash
    # Specify pullPolicy as Never
    # Specify repository as $PREFIX/<component>
    # Specify tag as $LABEL_PREFIX
    helm install akri ./deployment/helm \
        $AKRI_HELM_CRICTL_CONFIGURATION \
        --set agent.image.pullPolicy=Never \
        --set agent.image.repository="$PREFIX/agent" \
        --set agent.image.tag="$LABEL_PREFIX" \
        --set controller.image.pullPolicy=Never \
        --set controller.image.repository="$PREFIX/controller" \
        --set controller.image.tag="$LABEL_PREFIX"
   ```
