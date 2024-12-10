---
title: VPC Lattice Setup
dates: 2024-10-12
categories: [AWS, VPC, EKS, VPC Lattice]
tags: [AWS, VPC, EKS, VPC Lattice]
---


# Setting Up Amazon VPC Lattice with AWS Gateway API Controller for EKS

## Introduction

This guide provides a comprehensive overview of configuring Amazon VPC Lattice with AWS Gateway API Controller for Elastic Kubernetes Service (EKS). It includes configuring security groups, creating IAM roles, installing the controller using Helm, and enabling service-to-service communication.

---

## Prerequisites

Before proceeding, ensure the following:

- An existing EKS cluster.
- Tools installed: `aws-cli`, `kubectl`, `eksctl`.
- Access to AWS account with necessary permissions.

---

## Step 1: Set AWS Environment Variables

Start by setting your AWS region and cluster name as environment variables:

```bash
export AWS_REGION=<cluster_region>
export CLUSTER_NAME=<cluster_name>
```

If you created the cluster with `eksctl`, retrieve the Security Group ID:

```bash
CLUSTER_SG=$(aws eks describe-cluster --name $CLUSTER_NAME --output json | jq -r '.cluster.resourcesVpcConfig.clusterSecurityGroupId')
```

---

## Step 2: Update Security Groups for VPC Lattice

### Add Prefix for IPv4 and IPv6

Authorize security group ingress for IPv4 and IPv6:

```bash
# IPv4
PREFIX_LIST_ID=$(aws ec2 describe-managed-prefix-lists --query "PrefixLists[?PrefixListName=='com.amazonaws.$AWS_REGION.vpc-lattice'].PrefixListId" | jq -r '.[]')
aws ec2 authorize-security-group-ingress --group-id $CLUSTER_SG --ip-permissions "PrefixListIds=[{PrefixListId=${PREFIX_LIST_ID}}],IpProtocol=-1"

# IPv6
PREFIX_LIST_ID_IPV6=$(aws ec2 describe-managed-prefix-lists --query "PrefixLists[?PrefixListName=='com.amazonaws.$AWS_REGION.ipv6.vpc-lattice'].PrefixListId" | jq -r '.[]')
aws ec2 authorize-security-group-ingress --group-id $CLUSTER_SG --ip-permissions "PrefixListIds=[{PrefixListId=${PREFIX_LIST_ID_IPV6}}],IpProtocol=-1"
```

---

## Step 3: Create and Attach IAM Policy

Download the recommended inline policy and create it in AWS IAM:

```bash
curl https://raw.githubusercontent.com/aws/aws-application-networking-k8s/main/files/controller-installation/recommended-inline-policy.json -o recommended-inline-policy.json

aws iam create-policy     --policy-name VPCLatticeControllerIAMPolicy     --policy-document file://recommended-inline-policy.json

export VPCLatticeControllerIAMPolicyArn=$(aws iam list-policies --query 'Policies[?PolicyName==`VPCLatticeControllerIAMPolicy`].Arn' --output text)
```

---

## Step 4: Apply Namespace Manifest

Apply the namespace manifest for the controller:

```bash
kubectl apply -f https://raw.githubusercontent.com/aws/aws-application-networking-k8s/main/files/controller-installation/deploy-namesystem.yaml
```

---

## Step 5: Create IAM Service Account for Pod-Level Permissions

### Associate OIDC Provider

```bash
eksctl utils associate-iam-oidc-provider --cluster $CLUSTER_NAME --approve --region $AWS_REGION
```

### Create IAM Service Account

```bash
eksctl create iamserviceaccount     --cluster=$CLUSTER_NAME     --namespace=aws-application-networking-system     --name=gateway-api-controller     --attach-policy-arn=$VPCLatticeControllerIAMPolicyArn     --override-existing-serviceaccounts     --region $AWS_REGION     --approve
```

---

## Step 6: Install Gateway API Controller Using Helm

Log in to AWS Public ECR and install the Gateway API Controller:

```bash
aws ecr-public get-login-password --region us-east-1 | helm registry login --username AWS --password-stdin public.ecr.aws

helm install gateway-api-controller     oci://public.ecr.aws/aws-application-networking-k8s/aws-gateway-controller-chart     --version=v1.0.6     --set=serviceAccount.create=false     --namespace aws-application-networking-system     --set=awsRegion=$AWS_REGION     --set=clusterVpcId=<your_cluster_vpc_id>     --set=awsAccountId=<your_account_id>     --set=clusterName=$CLUSTER_NAME     --set=log.level=info
```

---

## Step 7: Verify Controller Status

Check if the Gateway API Controller pods are running:

```bash
kubectl --namespace aws-application-networking-system get pods -l "app.kubernetes.io/instance=gateway-api-controller"
```

---

## Step 8: Configure VPC Lattice GatewayClass

Apply the GatewayClass manifest:

```bash
kubectl apply -f https://raw.githubusercontent.com/aws/aws-application-networking-k8s/main/files/controller-installation/gatewayclass.yaml
```

---

## Step 9: Setup Service-to-Service Communication

### Clone the AWS Gateway API Controller Repository

```bash
git clone https://github.com/aws/aws-application-networking-k8s.git
cd aws-application-networking-k8s
```

### Create Kubernetes Gateway and Routes

```bash
kubectl apply -f files/examples/my-hotel-gateway.yaml
kubectl apply -f files/examples/parking.yaml
kubectl apply -f files/examples/review.yaml
```

### Verify Connectivity

Check connectivity between services:

```bash
kubectl exec deploy/inventory-ver1 -- curl -s $ratesFQDN/parking $ratesFQDN/review
```

---

## Conclusion

You have successfully set up Amazon VPC Lattice with AWS Gateway API Controller for your EKS cluster. This configuration enables secure and efficient service-to-service communication. For more details, refer to the [AWS Documentation](https://docs.aws.amazon.com/).
