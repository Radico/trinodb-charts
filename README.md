
Trino Community Kubernetes Helm Charts
===========
[![CI/CD](https://github.com/trinodb/charts/actions/workflows/ci-cd.yaml/badge.svg?branch=main)](https://github.com/trinodb/charts/actions/workflows/ci-cd.yaml)

Fast distributed SQL query engine for big data analytics that helps you explore your data universe


## Usage

[Helm](https://helm.sh) must be installed to use the charts.
Please refer to Helm's [documentation](https://helm.sh/docs/) to get started.

Once Helm is set up properly, add the repo as follows:

```console
helm repo add trino https://trinodb.github.io/charts/
```

You can then run `helm search repo trino` to see the charts.

Then you can install chart using:

```console
helm install my-trino trino/trino --version 0.25.0
```

Also, you can check the manifests using:

```console
helm template my-trino trino/trino --namespace <YOUR_NAMESPACE>
```

## Simon fork: building and publishing

This fork tracks upstream chart 1.45.0 with a small set of Simon patches on top,
published as chart version `1.45.0+simon` (Trino 482). Helm stores the `+` as `_`
in the OCI tag.

### Building

```console
cd charts/trino
helm package .
```

### Publishing

#### AWS ECR

```console
cd charts/trino
aws ecr get-login-password \
     --region us-east-1 | helm registry login \
     --username AWS \
     --password-stdin <account>.dkr.ecr.us-east-1.amazonaws.com/<repo>
helm push trino-1.45.0+simon.tgz oci://<account>.dkr.ecr.us-east-1.amazonaws.com
```

```console
helm install trino-blue oci://<account>.dkr.ecr.us-east-1.amazonaws.com/trino --version 1.45.0+simon
```

See [the values migration notes](./charts/trino/MIGRATION-0.13-to-1.45.md) when
moving a release from the 0.13 fork.

## Documentation

You can find documentation about the chart [here](./charts/trino/README.md).

## Development

To test the chart, install it into a Kubernetes cluster. Use `kind` to create a
Kubernetes cluster running in a container, and `chart-testing` to install the
chart and run [tests](charts/trino/templates/tests).

```console
brew install helm kind chart-testing
kind create cluster
ct install
```

To run tests with specific values:
```console
ct install --helm-extra-set-args "--set image.tag=450"
```

Use the `test.sh` script to run a suite of tests, with different chart values.
If some of the tests fail, use the `-s` flag to skip cleanup and inspect the
resources installed in the Kubernetes cluster. Use `-n` to use a specific
namespace, not a randomly generated one. Use `-t` to run only selected tests.
See the command help (`-h`) for a list of available tests.

Example:
```console
./test.sh -n trino -s -t default
```

The documentation is automatically generated from the chart files. Install a
git hook to have it automatically updated when committing changes. Make sure
you [install the pre-commit binary](https://pre-commit.com/#install), then run:

```console
pre-commit install
pre-commit install-hooks
```
