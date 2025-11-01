# function-deletion-protection

A Crossplane Composition Function that adds deletion protection to resources by creating `ClusterUsage` or `Usage` objects when resources are labeled with `protection.fn.crossplane.io/block-deletion: "true"`.

The Usage will block deletion requests 

This function requires Crossplane version 2.0 or higher, which includes the new `protection.crossplane.io` API Group.

## Overview

This function monitors resources in a composition for the `protection.fn.crossplane.io/block-deletion` label and creates corresponding ClusterUsage objects to prevent accidental deletion. It can protect:

- Composite resources (XRs) when labeled
- Composed resources when labeled. If a Composed resources is protected, the parent Composite will also be protected.

Resources can be labeled outside of the Composition using `kubectl label`. The function will check if either the
desired or observed state is labeled:

```yaml
apiVersion: ec2.aws.upbound.io/v1beta1
kind: VPC
metadata:
  labels:
    protection.fn.crossplane.io/block-deletion: "true"
  name: my-vpc
```

The function will generate a `ClusterUsage`

```yaml
apiVersion: protection.crossplane.io/v1beta1
kind: ClusterUsage
metadata:
  name: vpc-my-vpc-fn-protection
spec:
  of:
    apiVersion: ec2.aws.upbound.io/v1beta1
    kind: VPC
    resourceRef:
      name: my-vpc
  reason: created by function-deletion-protection via label protection.fn.crossplane.io/block-deletion

```

If a resource is Cluster-scoped, a `ClusterUsage` will be generated. If Namespaced a `Usage` will be created in the Resource's namespace.

## Installation

The function can be installed in a Crossplane [Composition Pipeline](https://docs.crossplane.io/latest/composition/compositions/).

The only setting is `cacheTTL`, which configures the alpha Function Response Cache.

```yaml
    - step: protect-resources
      functionRef:
        name: crossplane-contrib-function-protection
      input:
        apiVersion: protection.fn.crossplane.io/v1beta1
        kind: Input
        cacheTTL: 10m
```

## Building

To build the Docker image for both arm64 and amd64 and save the results
in a `tar` file, run:

```shell
export VERSION=0.1.3
# Build the function's runtime image
$ docker buildx build --platform linux/amd64,linux/arm64 . --output type=oci,dest=function-deletion-protection-runtime-v${VERSION}.tar
```

Next, build the Crossplane Package:

```shell
export VERSION=0.1.3
crossplane xpkg build -f package --embed-runtime-image-tarball=function-deletion-protection-runtime-v${VERSION}.tar -o function-deletion-protection-v${VERSION}.xpkg
```
