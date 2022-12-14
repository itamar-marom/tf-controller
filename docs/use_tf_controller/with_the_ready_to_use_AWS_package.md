## Use TF-controller with the ready-to-use AWS package

This document describes how to use the Weave TF-controller with the ready-to-use AWS package.
It requires TF-controller v0.13+ to run the example.

## What is a package?

A package is a collection of primitive Terraform modules that are bundled into an OCI image.
You can think of a TF-controller's package as a thin wrapper around a Terraform module provider,
and a TF-controller primitive module as a thin wrapper around a Terraform resource or a root module.

We will provide a set of ready-to-use packages for the most popular cloud providers.
Currently, we ship the package for AWS only.

## AWS Package

To provide the out-of-the-box experience, the AWS Package is installed by default when you installed the TF-controller.
Unlike other IaC implementation, our package model is designed to be very lightweight as a package is just a set of TF files in the form of OCI. 
Packages would not put any burden to your cluster. However, you can opt this package out by setting `awsPackage.install: false` in your Helm chart values.

If you run `flux get sources oci` you should see the AWS package installed in your cluster listed as `aws-package`.

```shell
flux get sources oci
NAME          REVISION                    SUSPENDED   READY   MESSAGE                                                                                                         
aws-package   v4.38.0-v1alpha11/6033f3b   False       True    stored artifact for digest 'v4.38.0-v1alpha11/6033f3b'
```

## A step-by-step tutorial

This section describes how to use the AWS package to provision an S3 bucket with ACL using the TF-controller.

### Create a KinD local cluster

If you don't have a Kubernetes cluster, you can create a KinD cluster with the following command:

```shell
kind create cluster
```

### Install Flux

After you have a Kubernetes cluster, you can install Flux with the following command:

```shell
flux install
```

### Install TF-controller

Then, you can install the TF-controller with the following command:

```shell
kubectl apply -f https://raw.githubusercontent.com/weaveworks/tf-controller/main/docs/release.yaml
```

### Setup AWS credentials

To provision AWS resources, you need to provide the AWS credentials to your Terraform objects.
You can do this by creating a secret with the AWS credentials and reference it in each of your Terraform objects.

```shell

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: aws-credentials
  namespace: flux-system
type: Opaque
stringData:
  AWS_ACCESS_KEY_ID: Axxxxxxxxxxxxxxxxxxx
  AWS_SECRET_ACCESS_KEY: qxxxxxxxxxxxxxxxxxxxxxxxxx
  AWS_REGION: us-east-1 # the region you want
```

To apply the secret, run the following command:

```shell
kubectl apply -f aws-credentials.yaml
```

### Setup AWS Bucket and ACL

Now, you can create two Terraform objects, one for an S3 bucket, another one for ACL.
Please note that we are using GitOps dependencies to make sure the ACL is created after the bucket is created.
You can read more about the GitOps dependencies in the [GitOps dependencies](./with_GitOps_dependency_management.md) document.

```shell

```yaml
---
apiVersion: infra.contrib.fluxcd.io/v1alpha1
kind: Terraform
metadata:
  name: aws-s3-bucket
  namespace: flux-system
  labels:
    tf.weave.works/composite: s3-bucket
spec:
  path: aws_s3_bucket
  values:
    bucket: my-tf-controller-test-bucket
    tags:
      Environment: Dev
      Name: My bucket
  sourceRef:
    kind: OCIRepository
    name: aws-package
  approvePlan: auto
  retryInterval: 10s
  interval: 2m
  destroyResourcesOnDeletion: true
  writeOutputsToSecret:
    name: aws-s3-bucket-outputs
    outputs:
    - arn
    - bucket
  runnerPodTemplate:
    spec:
      envFrom:
      - secretRef:
          name: aws-credentials
---
apiVersion: infra.contrib.fluxcd.io/v1alpha1
kind: Terraform
metadata:
  name: example-bucket-acl
  namespace: flux-system
  labels:
    tf.weave.works/composite: s3-bucket
spec:
  path: aws_s3_bucket_acl
  values:
    acl: private
    bucket: ${{ .aws_s3_bucket.bucket }}
  sourceRef:
    kind: OCIRepository
    name: aws-package
  approvePlan: auto
  retryInterval: 10s
  interval: 3m
  dependsOn:
  - name: aws-s3-bucket
  readInputsFromSecrets:
  - name: aws-s3-bucket-outputs
    as: aws_s3_bucket
  destroyResourcesOnDeletion: true
  runnerPodTemplate:
    spec:
      envFrom:
      - secretRef:
          name: aws-credentials
```
