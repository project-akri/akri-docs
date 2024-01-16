# End to end test framework

This documentation covers the details of Akri's end to end testing workflow, which runs Akri on several Kubernetes distributions
and versions each time a commit is pushed to a PR or a PR is merged.

It will explain how it works, how the CI runs it, how to run it locally and how to write new tests.

The end to end test framework is based on Python and [pytest](https://docs.pytest.org/), the dependencies are managed using [Poetry](https://python-poetry.org/). It aims to test different scenarios directly in a Kubernetes instance.

## How tests are done

The test suite assumes a working, clean Kubernetes cluster (can be single node) and also needs a working kube config and `helm` client.

For every test suite, Akri will be installed (and uninstalled at the end of it) using helm with the needed discovery handlers.

All tests suite and cases can be run independently in any order.

## The CI Test K3s, Kubernetes (Kubeadm) and MicroK8s Workflow

File:
[`akri/.github/workflows/run-test-cases.yml`](https://github.com/project-akri/akri/blob/main/.github/workflows/run-test-cases.yml)

A GitHub workflow that:

+ runs Python pytest-based end-to-end tests;
+ through 4 different Kubernetes versions: 1.24, 1.25, 1.26, 1.27;
+ on 3 different Kubernetes distros: [K3s](https://k3s.io), [Kubernetes (Kubeadm)](https://kubernetes.io/docs/reference/setup-tools/kubeadm/), [MicroK8s](https://microk8s.io).

### Jobs|Steps

The workflow comprises two jobs (`build-containers` and `test-cases`).

#### `build-containers`

`build-containers` builds container images for Akri 'controller', 'agent' and discovery handlers based upon the commit that triggers the
workflow. Once build, these images are shared across the `test-cases` job, using GitHub Action
[upload-artifact](https://github.com/actions/upload-artifact).

When not running in a PR context, this is skipped

#### `test-cases`

`test-cases` uses a GitHub
[strategy](https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions#jobsjob_idstrategy) to run
its steps across the different Kubernetes distros and versions summarized at the top of this document.

New Kubernetes distro versions may be added to the job by adding entries to `jobs.test-cases.strategy.matrix.kube`. Each
array entry must include:

|Property|Description|
|--------|-----------|
|`runtime`|The Kubernetes distribution name (`k3s`, `k8s` or `microk8s`)|
|`version`|A distro-specific unique identifier for the Kubernetes version|

Notes:

+ `runtime` is used by subsequent steps as a way to determine the distro, e.g. `startsWith(matrix.kube.runtime, 'k3s')`
+ `version` is used by each distro to determine which binary, snap etc. to install. Refer to each distro's documentation
  to determine the value required

##### Distro installation and Akri container images insertion

Each distro has an installation step and a step to import the Akri images created by the `build-containers` job.

The installation steps are identified by:

```YAML
if: startsWith(matrix.kube.runtime, ${DISTRO})
```

The installation instructions map closely with the installation instructions provided for the distro.

The container image import steps are identified by:

```YAML
if: (startsWith(github.event_name, 'pull_request')) && (startsWith(matrix.kube.runtime, ${DISTRO}))
```

##### Tests

Of all the steps, only one is needed to run the Python end-to-end script.

The scripts arguments contains all needed information for it to run (see [Run the tests locally](#run-the-tests-locally) for details on arguments).

stdout|stderr from the script can be logged to the workflow.

##### Upload logs

Once the end-to-end script is complete, the workflow uses the GitHub Action
[upload-artifact](https://github.com/actions/upload-artifact) again to upload `/tmp/log` and so that these remain available (for download) once the workflow completes.

## Run the tests locally

In order to run the test suite on your computer, you need:

+ [Python ≥3.10](https://wiki.python.org/moin/BeginnersGuide/Download)
+ [Poetry](https://python-poetry.org/docs/#installation)
+ [Helm client](https://helm.sh/docs/intro/install/)

You also need a clean Kubernetes cluster, one can be easily created using [k3d](https://k3d.io/).

All further commands are expected to be run from the `/test/e2e/` directory.

To install the dependencies run `poetry install`.

To run all the tests run `poetry run pytest -v --distribution ${DISTRO}`, with `DISTRO` either `k3s`, `k8s`, or `microk8s`.

To run specific test suite, add the suite file as argument to the pytest command (e.g add `test_webhook.py`).

To run a specific test in a test suite add the fully quialified test name as argument to the pytest command (e.g add `test_webhook::test_valid_configuration_accepted`).

You can specify multiple tests or test suites in you pytest command.

By default, the tests will run on latest `akri-dev` chart.

There are other options in addition to `--distribution` that affect the test run:

| Option | Description |
| ------ | ----------- |
| `--distribution` | Specify the target distribution for the tests, can be one of `k3s`, `k8s` or `microk8s` |
| `--release` | Use `akri` chart instead of `akri-dev` |
| `--test-version` | Version of the chart to use |
| `--use-local` | Use local chart (i.e `/deployment/helm` and local images) |
| `--local-tag` | When using local images, the tag used by images (by default will look for `pr` tag) |

## Technical details and writing new tests

All end to end tests suites are located in `/test/e2e/`, as we use pytest to run those, a test suite file must be named `test_${SUITE}.py`.
Within these files every `test_*` functions will run independently, fixtures are here to help in setting up and tearing down the test evironment.

Every fixture will get set up when first used in scope (everything before the `yield` is executed), and teared down after last use in scope (everything after the `yield` is executed).
Useful scopes for our usecases are `session` for the entire time of the pytest run, `module` for a specific suite and `function` for a specific test.
An `autouse` fixture will get automatically added without explicitely asking for it.

Akri will be installed thanks to the `autouse` fixture `install_akri`, the scope of this fixture is `module`, meaning it will get installed and uninstalled
for every test suite. This fixture is configured by the module variable `discovery_handlers` that must be set to a list of discovery handlers to enable.

A test passes if the function return; a test fails if the function raises an exception.

Be careful when writing new tests, the test order is **not** guarenteed, so make sure your test reverts any modification.
