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

## Step 2: Update Security Groups for VPC Lattice

Add Prefix for IPv4 and IPv6
Authorize security group ingress for IPv4 and IPv6:
# IPv4
PREFIX_LIST_ID=$(aws ec2 describe-managed-prefix-lists --query "PrefixLists[?PrefixListName=='com.amazonaws.$AWS_REGION.vpc-lattice'].PrefixListId" | jq -r '.[]')
aws ec2 authorize-security-group-ingress --group-id $CLUSTER_SG --ip-permissions "PrefixListIds=[{PrefixListId=${PREFIX_LIST_ID}}],IpProtocol=-1"

# IPv6
PREFIX_LIST_ID_IPV6=$(aws ec2 describe-managed-prefix-lists --query "PrefixLists[?PrefixListName=='com.amazonaws.$AWS_REGION.ipv6.vpc-lattice'].PrefixListId" | jq -r '.[]')
aws ec2 authorize-security-group-ingress --group-id $CLUSTER_SG --ip-permissions "PrefixListIds=[{PrefixListId=${PREFIX_LIST_ID_IPV6}}],IpProtocol=-1"

## Step 3: Create and Attach IAM Policy

Download the recommended inline policy and create it in AWS IAM:
curl https://raw.githubusercontent.com/aws/aws-application-networking-k8s/main/files/controller-installation/recommended-inline-policy.json -o recommended-inline-policy.json

aws iam create-policy \
    --policy-name VPCLatticeControllerIAMPolicy \
    --policy-document file://recommended-inline-policy.json

export VPCLatticeControllerIAMPolicyArn=$(aws iam list-policies --query 'Policies[?PolicyName==`VPCLatticeControllerIAMPolicy`].Arn' --output text)
