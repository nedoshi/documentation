---
title: "Creating a ROSA HCP cluster with custom KMS key"
date: 2026-02-06
tags: ["AWS", "ROSA", "HCP", "KMS", "Encryption"]
authors:
  - Nerav Doshi
---

# Creating a ROSA HCP cluster with custom KMS key

This guide walks you through deploying a Red Hat OpenShift Service on AWS (ROSA) with Hosted Control Planes (HCP) using a customer-managed AWS KMS key. The KMS key can be used to encrypt:

- Worker node root volumes
- etcd database (control plane encryption)
- PersistentVolumes (via custom StorageClass)

> **Tip:** For official documentation, see [Creating ROSA HCP clusters using a custom AWS KMS encryption key](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws/4/html/install_clusters/rosa-hcp-creating-cluster-with-aws-kms-key).

> **Note:** This guide is specifically for **ROSA with Hosted Control Planes (HCP)**. For ROSA Classic, see [Creating a ROSA cluster in STS mode with custom KMS key](/rosa/kms/).

## Prerequisites

- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html) installed and configured
- [ROSA CLI](https://console.redhat.com/openshift/downloads) v1.2.0 or higher
- [OpenShift CLI](https://console.redhat.com/openshift/downloads) (`oc`)
- AWS account with ROSA enabled
- Red Hat account linked to AWS via the ROSA console

### Verify Prerequisites

#### Verify ROSA CLI version (must be 1.2.0+)
```rosa version```

#### Verify AWS CLI is configured
```
aws sts get-caller-identity
```

#### Verify ROSA login
```
rosa whoami
```

#### Verify ROSA is enabled in your AWS account
```
rosa verify quota
rosa verify permissions
```
### Set Environment Variables
Set the following environment variables to use throughout this guide:
# AWS Configuration
export AWS_REGION=us-east-1
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Cluster Configuration
export CLUSTER_NAME=my-rosa-hcp
export MACHINE_CIDR=10.0.0.0/16

# Role Prefixes
export ACCOUNT_ROLES_PREFIX=ManagedOpenShift
export OPERATOR_ROLES_PREFIX=${CLUSTER_NAME}

# Verify
echo "AWS Account: ${AWS_ACCOUNT_ID}"
echo "Region: ${AWS_REGION}"
echo "Cluster Name: ${CLUSTER_NAME}"
