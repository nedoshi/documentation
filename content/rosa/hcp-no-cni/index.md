---
date: '2024-10-29'
title: ROSA HCP with AWS CNI Plugin
tags: ["AWS", "ROSA"]
authors:
  - Nerav Doshi
---
## Overview

You can set up Red Hat OpenShift Service on AWS (ROSA) with hosted control planes (HCP) without a Container Network Interface (CNI) and then adding the AWS CNI plugin.

This document will guide you through setting up AWS CNI Plugin.


## Prerequisites
* [Create a ROSA with HCP cluster without CNI ](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws/4/html/install_rosa_with_hcp_clusters/rosa-hcp-cluster-no-cli#rosa-hcp-no-cni-cluster-creation)
* [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)
* [OpenShift Command Line Interface (CLI)](https://mirror.openshift.com/pub/openshift-v4/clients/ocp/)
* [jq](https://stedolan.github.io/jq/download/)
* [wget](https://www.hostinger.com/tutorials/wget-command-examples/#How_to_install_wget)

## Deploy a ROSA HCP cluster without CNI
To create a new ROSA HCP cluster with external authentication enabled, ensure that you add the `--no-cni` flag to your `rosa create cluster` command. An example command is included below:

```bash
rosa create cluster --cluster-name=<cluster_name> --sts --mode=auto \
    --hosted-cp --operator-roles-prefix <operator-role-prefix> \
    --oidc-config-id <ID-of-OIDC-configuration> \
    --external-auth-providers-enabled \
    --subnet-ids=<public-subnet-id>,<private-subnet-id> --no-cni
```

Once the ROSA HCP cluster has been created successfully, you clusterw hould be in `ready` state, check the status of the cluster and run the following command:

```bash
rosa describe cluster --cluster=<cluster_name>
```
After sucessful installation

{{% alert state="warning" %}}
Important: When you first log in to the cluster after it reaches ready status, the nodes will still be in the not ready state until you install your own CNI plugin. After CNI installation, the nodes will change to ready.
{{% /alert %}}

## Create a Dedicated IAM Role for AWS CNI Plugin

#### 1. Create the IAM Policy
First, create the policy document that defines the permissions needed by AWS CNI:


```bash
cat > aws-cni-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:AssignPrivateIpAddresses",
        "ec2:AttachNetworkInterface",
        "ec2:CreateNetworkInterface",
        "ec2:DeleteNetworkInterface",
        "ec2:DescribeInstances",
        "ec2:DescribeNetworkInterfaces",
        "ec2:DescribeSubnets",
        "ec2:DescribeTags",
        "ec2:DescribeVpcs",
        "ec2:DetachNetworkInterface",
        "ec2:ModifyNetworkInterfaceAttribute",
        "ec2:UnassignPrivateIpAddresses",
        "ec2:CreateTags"
      ],
      "Resource": "*"
    }
  ]
}
EOF
```
#### 2. Create a policy in AWS

```bash
aws iam create-policy \
  --policy-name AmazonHCP_CNI_Policy \
  --policy-document file://aws-cni-policy.json
```
example output:
```
----------------------------------------------------------------------------------------------
|                                        CreatePolicy                                        |
+--------------------------------------------------------------------------------------------+
||                                          Policy                                          ||
|+--------------------------------+---------------------------------------------------------+|
||  Arn                           |  arn:aws:iam::xxxxxxxxxxxx:policy/AmazonHCP_CNI_Policy  ||
||  AttachmentCount               |  0                                                      ||
||  CreateDate                    |  2025-04-24T18:59:50Z                                   ||
||  DefaultVersionId              |  v1                                                     ||
||  IsAttachable                  |  True                                                   ||
||  Path                          |  /                                                      ||
||  PermissionsBoundaryUsageCount |  0                                                      ||
||  PolicyId                      |  xxxxxxxxxxxxxxxxxxxxx                                  ||
||  PolicyName                    |  AmazonHCP_CNI_Policy                                   ||
||  UpdateDate                    |  2025-04-24T18:59:50Z                                   ||
|+--------------------------------+---------------------------------------------------------+|
```
Save the ARN from the output for use in the next steps

#### 3: Create Trust Relationship Document

Create a trust relationship document that allows the service to assume this role:

```bash
# Get your OpenShift cluster's OIDC provider URL
OIDC_PROVIDER=$(rosa describe cluster --name my-hcp-cluster --output json | jq -r '.aws.sts.oidc_endpoint_url' | sed 's|https://||')

# Get your AWS account ID
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

cat > trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::${ACCOUNT_ID}:oidc-provider/${OIDC_PROVIDER}"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "${OIDC_PROVIDER}:sub": "system:serviceaccount:amazon-vpc-cni:aws-node"
        }
      }
    }
  ]
}
EOF
```
#### 4: Create the IAM Role
Create the role with the trust relationship:

```bash
aws iam create-role \
  --role-name AmazonHCP_CNI_Role \
  --assume-role-policy-document file://trust-policy.json
```

#### 5: Attach the Policy to the Role

```bash
aws iam attach-role-policy \
  --role-name AmazonHCP_CNI_Role \
  --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/AmazonHCP_CNI_Policy
```

## Install and configure AWS CNI plugin

#### 1: Log into your cluster 
```bash
oc login <api-server-url> -u <username> -p <password>
```

#### 2: Create namespace for AWS CNI:
```bash
oc new-project amazon-vpc-cni
```

#### 3: Download the AWS VPC CNI plugin:

```bash
wget https://raw.githubusercontent.com/aws/amazon-vpc-cni-k8s/master/config/master/aws-k8s-cni.yaml
```
Replace all AWS CNI components (ServiceAccount, ConfigMap, ClusterRoleBinding, DaemonSet) are consistently deployed in the amazon-vpc-cni namespace

```bash
sed -i 's/namespace: kube-system/namespace: amazon-vpc-cni/g' aws-k8s-cni.yaml
```
For mac users use

```bash
sed -i '' 's/namespace: kube-system/namespace: amazon-vpc-cni/g' aws-k8s-cni.yaml
```

#### 4: Apply AWS VPC CNI plugin:

```bash
oc apply -f aws-k8s-cni.yaml
```

example output:

```
oc apply -f aws-k8s-cni.yaml

customresourcedefinition.apiextensions.k8s.io/eniconfigs.crd.k8s.amazonaws.com created
customresourcedefinition.apiextensions.k8s.io/policyendpoints.networking.k8s.aws created
serviceaccount/aws-node created
configmap/amazon-vpc-cni created
clusterrole.rbac.authorization.k8s.io/aws-node unchanged
clusterrolebinding.rbac.authorization.k8s.io/aws-node unchanged
daemonset.apps/aws-node created
```

#### 5: Create service account

```bash
oc create serviceaccount aws-node -n amazon-vpc-cni 
```

example output:

```
oc create serviceaccount aws-node -n amazon-vpc-cni 
serviceaccount/aws-node created
```

Annotate the service account with the IAM role ARN

```bash
oc annotate serviceaccount -n amazon-vpc-cni aws-node \
  eks.amazonaws.com/role-arn=arn:aws:iam::${ACCOUNT_ID}:role/AmazonHCP_CNI_Role
```
#### 6: Creat a custome SCC and bind it

```yaml
cat << EOF | oc apply -f -
apiVersion: security.openshift.io/v1
kind: SecurityContextConstraints
metadata:
  name: aws-node-scc
allowHostNetwork: true
allowHostPorts: true
allowPrivilegedContainer: true
runAsUser:
  type: RunAsAny
seLinuxContext:
  type: RunAsAny
fsGroup:
  type: RunAsAny
supplementalGroups:
  type: RunAsAny
users:
- system:serviceaccount:kube-system:aws-node
EOF
```
