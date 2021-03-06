---
title: "Creating a Cluster Pool"
description: How to create a cluster pool from which a job can claim a cluster.
---

This document describes how to set up cluster pools backed by custom cloud platform accounts owned by users or teams that want their jobs to use clusters provisioned with these custom accounts.

## Prerequisites

* Verify that the cloud platform you want to use is supported by [Hive](https://github.com/openshift/hive#supported-cloud-providers) (the cluster pool mechanism back-end).
* [Configure the cloud account](https://docs.openshift.com/container-platform/4.7/installing/installing-preparing.html): Hive does not require more configuration of the cloud account than installing an OpenShift cluster. Make sure that the account has quotas for the pools to contain the desired number of clusters.

## Create a cluster pool

In the openshift/release repository, create a folder in the [`clusters/hive/pools`](https://github.com/openshift/release/tree/master/clusters/hive/pools) directory that will contain manifests of all your pools (see [openshift-ci](https://github.com/openshift/release/tree/master/clusters/hive/pools/openshift-ci) as an example). Then place the manifests for the new `ClusterPool` there.

```yaml
apiVersion: hive.openshift.io/v1
kind: ClusterPool
metadata:
  name: ci-ocp-4-7-amd64-aws-us-east-1
  namespace: cvp-cluster-pool
  labels: # product, version, architecture, cloud, owner are used to filter out a pool when a job claims a cluster
    product: ocp
    version: "4.7"
    architecture: amd64
    cloud: aws
    owner: cvp
    region: us-east-1
spec:
  baseDomain: hive.example.org # the base domain to install the cluster
  imageSetRef:
    name: ocp-4.7.0-amd64 # the name of the imageSet which determines the image to install the cluster
  installConfigSecretTemplateRef:
    name: install-config-aws-us-east-1 # the name of the secret with an installation config for the installer
  skipMachinePools: true
  platform:
    aws:
      credentialsSecretRef:
        name: hive-aws-credentials # the name of the secret with the credentials of the cloud account
      region: us-east-1
  pullSecretRef:
    name: pull-secret # the name of the pull secret used by installer
  size: 1 # the number of clusters that Hive should keep provisioned and waiting for use.
  maxSize: 10 # the maximum number of clusters that have not been destroyed.
```

Hive maintains the number of clusters in the pool as specified by its `size`. A provisioned cluster will be [hibernating](https://github.com/openshift/hive/blob/master/docs/hibernating-clusters.md) after staying idled for sometime and can be waken up if a job claims it. Hive removes it from the pool once it is claimed and creates a new cluster to maintain the pool's `size`. The cluster will be destroyed after 4 hours since it is claimed. All clusters before destruction consume the quotas of the cloud account. If several jobs claim a cluster simultaneously, Hive will feed all of them up until the number of living clusters reaches the pool's `maxSize`. The meaning of other fields in the pool specification is available at [Hive's documentation](https://pkg.go.dev/github.com/openshift/hive/apis@master/hive/v1#ClusterPool).

It is suggested to put the relevant manifests to the cluster pool in the same `namespace`. Hive [replaces](https://github.com/openshift/hive/blob/master/docs/clusterpools.md#install-config-template) `baseDomain` with the value taken from the pool's spec and `metadata.name` with a random string in the installation config from `secret/install-config-aws-us-east-1` and then passes it onto the OpenShift installer. Customizations such as the number of workers and the type of instances can be made there. A `ClusterImageSet` which a cluster level CRD defines the production and its version of the installed cluster.

If any manifest contains sensitive information, e.g., `secret/hive-aws-credentials`, see [how to add a secret to CI](/docs/how-tos/adding-a-new-secret-to-ci/).

When those manifests for a pool are applied on the cluster `hive`, the cluster pool `cvp-cluster-pool` can be used to [claim a cluster for tests](/docs/architecture/ci-operator/#testing-with-a-cluster-from-a-cluster-pool).
