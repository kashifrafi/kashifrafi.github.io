---
title: AWS Load Balancer Controller for EKS Cluster
dates: 2024-12-17
categories: [AWS, EKS, NLB Controller, TG Binding]
tags: [AWS, VPC, EKS, NLB Controller, TG Binding]
---

# Configuring AWS Load Balancer Controller for EKS Clusters

## Overview

The AWS Load Balancer Controller is a Kubernetes controller that automates the provisioning and management of AWS load balancers, including **Application Load Balancers (ALB)** and **Network Load Balancers (NLB)**, for Kubernetes services. It integrates seamlessly with AWS services to dynamically create and configure load balancers, manage target groups, and route traffic to Kubernetes pods.

This guide covers the step-by-step configuration of the AWS Load Balancer Controller on an EKS cluster, along with a practical example to test the setup.

---

## Key Concepts

1. **Load Balancer Types**:
   - **ALB**: Best for HTTP/HTTPS traffic with advanced routing.
   - **NLB**: Ideal for low-latency TCP/UDP traffic.

2. **Annotations**:
   - Service annotations define how AWS load balancers behave (e.g., internal vs. external).

3. **IAM Role**:
   - Grants the controller permission to interact with AWS APIs.

4. **Subnet Tags**:
   - Required tags must be added to EKS subnets for proper resource provisioning:
     - **Key**: `kubernetes.io/role/internal-elb`
     - **Value**: `1`

## Prerequisites

Before you begin, ensure the following:

1. **EKS Cluster**: A functioning EKS cluster is required.
2. **Helm**: Install Helm to deploy the AWS Load Balancer Controller.
3. **IAM Permissions**: Ensure administrative privileges for creating roles and policies.
4. **Tagged Subnets**: Add the following tag to all EKS subnets:
   - **Key**: `kubernetes.io/role/internal-elb`
   - **Value**: `1`

## Step-by-Step Configuration

---
### Step 1: Create IAM Role

1. Download the IAM policy:
```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.7.2/docs/install/iam_policy.json
```
2. Create the IAM policy
 ```bash
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```
3. Create the IAM role and associate it with the EKS cluster
 ```bash
eksctl create iamserviceaccount \
    --cluster=my-cluster \
    --namespace=kube-system \
    --name=aws-load-balancer-controller \
    --role-name AmazonEKSLoadBalancerControllerRole \
    --attach-policy-arn arn:aws:iam::111122223333:policy/AWSLoadBalancerControllerIAMPolicy \
    --approve
```
---
### Step 2: Install AWS Load Balancer Controller

1. Add the Helm chart repository:
 ```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks
```
2. Install the controller
 ```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
    -n kube-system \
    --set clusterName=my-cluster \
    --set serviceAccount.create=false \
    --set serviceAccount.name=aws-load-balancer-controller \
    --set region=region-code \
    --set vpcId=vpc-xxxxxxxx
```
---
### Step 3: Verify Installation
Check the status of the controller and its pods
 ```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
kubectl get pods -n kube-system
```
---
### Testing the Setup

1. Create a Sample nginx Deployment and LoadBalancer Service
 ```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx
          ports:
            - containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: external
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
    service.beta.kubernetes.io/aws-load-balancer-scheme: internal
    service.beta.kubernetes.io/aws-load-balancer-name: test-nlb
  spec:
    selector:
      app: nginx
    ports:
      - port: 80
        targetPort: 80
        protocol: TCP
    type: LoadBalancer
```

2. Apply the manifest
 ```bash
kubectl apply -f nginx.yaml
```
3. Verify the LoadBalancer service
 ```bash
kubectl get svc nginx-service
```
4. Test the LoadBalancer
 ```bash
curl <external-endpoint>
```
Cleanup
To remove all resources
 ```bash
kubectl delete -f nginx.yaml
aws elbv2 delete-target-group --target-group-arn <target-group-arn>
```
### Summary
This guide demonstrated how to configure and test the AWS Load Balancer Controller for Kubernetes services in an EKS cluster. By automating the creation and management of load balancers, the controller enhances the scalability and reliability of Kubernetes workloads while reducing operational overhead.

For more information, refer to the AWS Load Balancer Controller Documentation.
