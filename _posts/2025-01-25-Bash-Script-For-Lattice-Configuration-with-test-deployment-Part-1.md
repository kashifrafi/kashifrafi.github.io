---
title: Bash Script For Lattice Configuration With Test Deployment Part-1
dates: 2024-01-25
categories: [AWS, VPC, Lattice, Test Deployment]
tags: [AWS, VPC, Lattice, Test Deployment]
---

# Automated Guide to VPC Lattice Setup and East-West Traffic Testing in Amazon EKS

In today’s cloud-centric world, modern applications demand robust, secure, and scalable solutions for service-to-service communication. As organizations increasingly adopt microservices architectures, managing traffic between services—often referred to as *east-west traffic*—becomes a critical challenge. Amazon Elastic Kubernetes Service (EKS) provides a powerful platform for running Kubernetes workloads, and when paired with Amazon VPC Lattice, it offers an advanced, streamlined approach to traffic management. VPC Lattice is a fully managed service that simplifies connectivity between services across Virtual Private Clouds (VPCs), accounts, and compute environments, ensuring secure and efficient communication without the complexity of traditional networking setups.

This comprehensive guide walks you through an automated Bash script designed to deploy the AWS Gateway API Controller for EKS using VPC Lattice. The script automates essential configuration steps, including setting up IAM policies, creating service accounts, installing the Gateway API Controller via Helm, and establishing GatewayClass and Gateway resources for traffic routing. Beyond setup, it includes testing mechanisms to verify *north-south* (external-to-cluster) and *east-west* (service-to-service) connectivity using HTTPRoutes and DNS-based routing. By the end of this guide, you’ll have a fully functional VPC Lattice configuration integrated with your EKS cluster, ready to handle sophisticated traffic patterns.

The objectives of this script are clear and practical:

1. **Deploy the AWS Gateway API Controller**: Use Helm to install and configure the controller, enabling VPC Lattice integration with EKS.
2. **Establish Traffic Routing**: Set up GatewayClass and Gateway resources to define how traffic enters and moves within your cluster.
3. **Test North-South Connectivity**: Validate external access to your services using HTTPRoutes.
4. **Verify East-West Communication**: Ensure seamless service-to-service interactions with DNS-based routing.

This guide assumes a basic familiarity with AWS, Kubernetes, and command-line tools, but it provides enough detail to assist both novice and experienced users. Let’s dive into the process and explore how this script simplifies a potentially complex setup.

---

## Why VPC Lattice and EKS?

Before diving into the script, it’s worth understanding why VPC Lattice is a game-changer for EKS users. Traditional Kubernetes networking relies on tools like Ingress controllers or service meshes, which can be effective but often require significant configuration overhead. VPC Lattice abstracts much of this complexity by operating at the application layer (Layer 7), offering features like path-based routing, authentication, and observability out of the box. When integrated with EKS via the AWS Gateway API Controller, it allows you to define networking policies declaratively using Kubernetes-native resources like `Gateway` and `HTTPRoute`, aligning with modern Infrastructure-as-Code practices.

For east-west traffic—communication between microservices within your cluster or across VPCs—VPC Lattice provides a centralized control plane. This eliminates the need for intricate service discovery mechanisms or manual VPC peering configurations, making it ideal for multi-cluster or hybrid environments. The script we’re about to explore automates this integration, saving time and reducing the risk of human error.

---

## Prerequisites

To successfully execute this script, ensure you have the following in place:

- **An EKS Cluster**: An existing cluster provisioned in your AWS account, ideally created with `eksctl` for compatibility with the script’s commands.
- **Command-Line Tools**: Install `aws-cli` (AWS Command Line Interface), `kubectl` (Kubernetes CLI), `eksctl` (EKS management tool), and `helm` (Kubernetes package manager).
- **AWS Permissions**: Access to an AWS account with sufficient privileges to manage IAM roles, policies, EKS clusters, and VPC resources.
- **Git**: Installed to clone the AWS Gateway API Controller repository.

With these prerequisites met, you’re ready to proceed with the automated setup.

---

## Detailed Walkthrough of the Script

The Bash script provided below automates the entire process, from environment setup to connectivity testing. Below, I’ve expanded the explanations for each section to provide deeper insight into what’s happening and why each step matters.

```bash
#!/bin/bash

# Define Variables (Modify these as per your setup)
CLUSTER_NAME="xxxxxxxxxxxxx" # Replace with your actual EKS cluster name
AWS_REGION="ap-south-1"      # Set to your AWS region, e.g., us-east-1 or eu-west-2
AWS_ACCOUNT_ID="xxxxxxxxxxxxxxx" # Your 12-digit AWS account ID
VPC_ID="vpc-xxxxxxxxxxxxxxxxx"   # The VPC ID where your EKS cluster resides

# Download the recommended inline policy for the controller installation
echo "Downloading recommended inline policy..."
curl https://raw.githubusercontent.com/aws/aws-application-networking-k8s/main/files/controller-installation/recommended-inline-policy.json -o recommended-inline-policy.json
# This policy defines the permissions needed for the Gateway API Controller to interact with VPC Lattice.

# Create the IAM policy
echo "Creating IAM policy..."
aws iam create-policy \
    --policy-name VPCLatticeControllerIAMPolicy-eks-2 \
    --policy-document file://recommended-inline-policy.json
# The policy is named uniquely to avoid conflicts; adjust the suffix if needed.

# Get the Policy ARN
VPCLatticeControllerIAMPolicyArn=$(aws iam list-policies --query 'Policies[?PolicyName==`VPCLatticeControllerIAMPolicy-eks-2`].Arn' --output text)
# This ARN will be attached to a service account later.

# Apply the controller installation manifest for the namespace
echo "Applying the controller installation manifest..."
kubectl apply -f https://raw.githubusercontent.com/aws/aws-application-networking-k8s/main/files/controller-installation/deploy-namesystem.yaml
# Creates the `aws-application-networking-system` namespace for the controller.

# Enable OIDC provider for EKS
echo "Enabling OIDC provider..."
eksctl utils associate-iam-oidc-provider --cluster $CLUSTER_NAME --approve --region $AWS_REGION
# OIDC enables IAM roles for Kubernetes service accounts, a key security feature.

# Create IAM Service Account for the pod
echo "Creating IAM service account..."
eksctl create iamserviceaccount \
    --cluster=$CLUSTER_NAME \
    --namespace=aws-application-networking-system \
    --name=gateway-api-controller \
    --attach-policy-arn=$VPCLatticeControllerIAMPolicyArn \
    --override-existing-serviceaccounts \
    --region $AWS_REGION \
    --approve
# Links the IAM policy to a Kubernetes service account for pod-level permissions.

# Login to AWS ECR Public and install the controller using Helm
echo "Login to AWS ECR Public and installing Helm chart..."
aws ecr-public get-login-password --region us-east-1 | helm registry login --username AWS --password-stdin public.ecr.aws

# Install the Gateway API Controller with Helm
helm install gateway-api-controller \
    oci://public.ecr.aws/aws-application-networking-k8s/aws-gateway-controller-chart \
    --version=v1.0.6 \
    --set=serviceAccount.create=false \
    --namespace aws-application-networking-system \
    --set=awsRegion=$AWS_REGION \
    --set=clusterVpcId=$VPC_ID \
    --set=awsAccountId=$AWS_ACCOUNT_ID \
    --set=clusterName=$CLUSTER_NAME \
    --set=log.level=info
# Deploys the controller, integrating it with your EKS cluster and VPC.

# Wait for the Pod to be in RUNNING state
echo "Waiting for the gateway-api-controller pod to be in RUNNING state..."
kubectl wait --namespace aws-application-networking-system \
  --for=condition=ready pod -l app.kubernetes.io/instance=gateway-api-controller \
  --timeout=300s  # Ensures the controller is fully operational.

# Wait for 3 minutes after Helm installation
echo "Waiting for 3 minutes after Helm installation..."
sleep 180  # Allows time for the controller to stabilize.

# Apply the GatewayClass manifest
echo "Creating GatewayClass..."
kubectl apply -f https://raw.githubusercontent.com/aws/aws-application-networking-k8s/main/files/controller-installation/gatewayclass.yaml
# Defines the GatewayClass, a blueprint for Gateway resources.

# Check if the Git repository exists, if not, clone it
GIT_REPO_DIR="aws-application-networking-k8s"
if [ ! -d "$GIT_REPO_DIR" ]; then
  echo "Cloning AWS Gateway API Controller repository..."
  git clone https://github.com/aws/aws-application-networking-k8s.git
else
  echo "Repository already cloned. Skipping git clone."
fi

# Navigate to the repository directory
cd $GIT_REPO_DIR

# Update the Helm chart for default service network
echo "Updating Helm chart for service network..."
aws ecr-public get-login-password --region us-east-1 | helm registry login --username AWS --password-stdin public.ecr.aws

helm upgrade gateway-api-controller \
    oci://public.ecr.aws/aws-application-networking-k8s/aws-gateway-controller-chart \
    --version=v1.0.6 \
    --reuse-values \
    --namespace aws-application-networking-system \
    --set=defaultServiceNetwork=my-hotel
# Configures a default service network for traffic routing.

# Wait for 3 minutes after updating the Helm chart
echo "Waiting for 3 minutes after updating Helm chart..."
sleep 180

# Test Deployment and North-South Connectivity Testing

# Apply the "my-hotel" gateway configuration
echo "Creating the 'my-hotel' gateway..."
kubectl apply -f files/examples/my-hotel-gateway.yaml
# Sets up a Gateway for external traffic.

# Verify that the Gateway was created successfully
echo "Verifying Gateway creation..."
kubectl get gateway

# Apply the HTTPRoute configurations
echo "Applying HTTPRoute configurations..."
kubectl apply -f files/examples/parking.yaml
kubectl apply -f files/examples/review.yaml
kubectl apply -f files/examples/rate-route-path.yaml

# Apply additional HTTPRoute for inventory
echo "Applying inventory HTTPRoute..."
kubectl apply -f files/examples/inventory-ver1.yaml
kubectl apply -f files/examples/inventory-route.yaml

# Wait for 3 minutes after applying inventory HTTPRoute
echo "Waiting for 3 minutes after applying inventory HTTPRoute..."
sleep 180

# Check DNS names of HTTPRoutes
echo "Fetching DNS names for HTTPRoutes..."
ratesFQDN=$(kubectl get httproute rates -o json | jq -r '.metadata.annotations."application-networking.k8s.aws/lattice-assigned-domain-name"')
inventoryFQDN=$(kubectl get httproute inventory -o json | jq -r '.metadata.annotations."application-networking.k8s.aws/lattice-assigned-domain-name"')

# Display DNS names
echo -e "Rates FQDN: $ratesFQDN\nInventory FQDN: $inventoryFQDN"

# Verify service-to-service communication
echo "Verifying service-to-service communication..."
kubectl exec deploy/inventory-ver1 -- curl -s $ratesFQDN/parking $ratesFQDN/review
kubectl exec deploy/parking -- curl -s $inventoryFQDN

```

---

## Expanded Explanation of Key Steps

### 1. Variable Definition
The script begins by defining critical variables like `CLUSTER_NAME`, `AWS_REGION`, `AWS_ACCOUNT_ID`, and `VPC_ID`. These must be customized to match your environment, ensuring the script targets the correct resources. Incorrect values here can lead to failures, so double-check them before running.

### 2. IAM Policy Creation
The script downloads a predefined IAM policy from the AWS GitHub repository and creates it in your account. This policy grants the Gateway API Controller permissions to manage VPC Lattice resources, such as service networks and targets. Naming the policy uniquely (e.g., `VPCLatticeControllerIAMPolicy-eks-2`) avoids conflicts with existing policies.

### 3. Namespace and Service Account
The `aws-application-networking-system` namespace isolates the controller’s resources, while the IAM service account ties AWS permissions to Kubernetes pods via OIDC. This adheres to the principle of least privilege, enhancing security.

### 4. Helm Installation
Using Helm to deploy the Gateway API Controller ensures a reproducible, version-controlled setup. The `--set` flags configure the controller with your cluster’s specifics, and the three-minute wait periods (`sleep 180`) allow for stabilization, which is critical in distributed systems like Kubernetes.

### 5. GatewayClass and Gateway
The `GatewayClass` defines a template for Gateway resources, which VPC Lattice uses to expose services. The `my-hotel` Gateway example demonstrates how external traffic can enter your cluster, a key step for north-south connectivity.

### 6. HTTPRoutes and Testing
The script applies HTTPRoute configurations for services like `parking`, `review`, and `inventory`, enabling path-based routing. The final `kubectl exec` commands test east-west communication by curling DNS names assigned by VPC Lattice, proving that services can communicate seamlessly.

---

## Conclusion

This Bash script automates the integration of VPC Lattice with EKS, providing a robust foundation for modern traffic management. By deploying the AWS Gateway API Controller, configuring Gateway resources, and testing connectivity, you’ve established a scalable, secure networking layer for your Kubernetes workloads. This guide offers a detailed narrative to complement the automation, ensuring you understand each step’s purpose and impact. For further customization or troubleshooting, consult the [AWS VPC Lattice Documentation](https://docs.aws.amazon.com/vpc-lattice/).
