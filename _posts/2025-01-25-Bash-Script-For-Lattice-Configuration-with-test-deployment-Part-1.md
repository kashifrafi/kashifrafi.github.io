---
title: Bash Script For Lattice Configuration With Test Deployment-Part-1
dates: 2024-01-25
categories: [AWS, VPC, Lattice, Test Deployment]
tags: [AWS, VPC, Lattice, Test Deployment]
---

### Automated Guide to VPC Lattice Setup and East-West Traffic Testing in Amazon EKS 
Modern applications require seamless and secure service-to-service communication, especially when operating at scale in the cloud. Amazon Elastic Kubernetes Service (EKS) combined with AWS VPC Lattice provides an efficient way to implement scalable, east-west communication and streamline traffic management between services.

This script is a step-by-step guide to deploying the AWS Gateway API Controller for EKS using VPC Lattice. It simplifies the setup process by automating key configurations, such as IAM policies, service accounts, and Gateway API installations, ensuring a smooth integration with your Kubernetes environment.

By following this script, you'll:

1. Set up the AWS Gateway API Controller with Helm.
2. Deploy GatewayClass and Gateway configurations for traffic routing.
3. Establish and verify north-south connectivity using HTTPRoutes.
4. Test service-to-service communication with DNS-based routing.

---

```bash

!/bin/bash

# Define Variables (Modify these as per your setup)
CLUSTER_NAME="xxxxxxxxxxxxx" # Your cluster name
AWS_REGION="ap-south-1"  # Modify as per your region
AWS_ACCOUNT_ID="xxxxxxxxxxxxxxx" # Your account id
VPC_ID="vpc-xxxxxxxxxxxxxxxxx"  # Modify with your VPC ID

# Download the recommended inline policy for the controller installation
echo "Downloading recommended inline policy..."
curl https://raw.githubusercontent.com/aws/aws-application-networking-k8s/main/files/controller-installation/recommended-inline-policy.json -o recommended-inline-policy.json

# Create the IAM policy
echo "Creating IAM policy..."
aws iam create-policy \
    --policy-name VPCLatticeControllerIAMPolicy-eks-2 \
    --policy-document file://recommended-inline-policy.json

# Get the Policy ARN
VPCLatticeControllerIAMPolicyArn=$(aws iam list-policies --query 'Policies[?PolicyName==`VPCLatticeControllerIAMPolicy-eks-2`].Arn' --output text)

# Apply the controller installation manifest for the namespace
echo "Applying the controller installation manifest..."
kubectl apply -f https://raw.githubusercontent.com/aws/aws-application-networking-k8s/main/files/controller-installation/deploy-namesystem.yaml

# Enable OIDC provider for EKS
echo "Enabling OIDC provider..."
eksctl utils associate-iam-oidc-provider --cluster $CLUSTER_NAME --approve --region $AWS_REGION

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

# Wait for the Pod to be in RUNNING state
echo "Waiting for the gateway-api-controller pod to be in RUNNING state..."
kubectl wait --namespace aws-application-networking-system \
  --for=condition=ready pod -l app.kubernetes.io/instance=gateway-api-controller \
  --timeout=300s  # Timeout after 5 minutes

# Wait for 3 minutes after Helm installation
echo "Waiting for 3 minutes after Helm installation..."
sleep 180  # Sleep for 180 seconds (3 minutes)

# Apply the GatewayClass manifest
echo "Creating GatewayClass..."
kubectl apply -f https://raw.githubusercontent.com/aws/aws-application-networking-k8s/main/files/controller-installation/gatewayclass.yaml

# Check if the Git repository exists, if not, clone it
GIT_REPO_DIR="aws-application-networking-k8s"  # Directory to check for Git clone
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

# Wait for 3 minutes after updating the Helm chart
echo "Waiting for 3 minutes after updating Helm chart..."
sleep 180  # Sleep for 180 seconds (3 minutes)

# Test Deployemnt and North-South Connectivity Testing

# Apply the "my-hotel" gateway configuration
echo "Creating the 'my-hotel' gateway..."
kubectl apply -f /prod-eks-2/test/aws-application-networking-k8s/files/examples/my-hotel-gateway.yaml

# Verify that the Gateway was created successfully
echo "Verifying Gateway creation..."
kubectl get gateway

# Apply the HTTPRoute configurations
echo "Applying HTTPRoute configurations..."
kubectl apply -f /aws-application-networking-k8s/files/examples/parking.yaml
kubectl apply -f /aws-application-networking-k8s/files/examples/review.yaml
kubectl apply -f /aws-application-networking-k8s/files/examples/rate-route-path.yaml

# Apply additional HTTPRoute for inventory
echo "Applying inventory HTTPRoute..."
kubectl apply -f /aws-application-networking-k8s/files/examples/inventory-ver1.yaml
kubectl apply -f /aws-application-networking-k8s/files/examples/inventory-route.yaml

# Wait for 3 minutes after applying inventory HTTPRoute
echo "Waiting for 3 minutes after applying inventory HTTPRoute..."
sleep 180  # Sleep for 180 seconds (3 minutes)

# Check DNS names of HTTPRoutes
echo "Fetching DNS names for HTTPRoutes..."
ratesFQDN=$(kubectl get httproute rates -o json | jq -r '.metadata.annotations."application-networking.k8s.aws/lattice-assigned-domain-name"')
inventoryFQDN=$(kubectl get httproute inventory -o json | jq -r '.metadata.annotations."application-networking.k8s.aws/lattice-assigned-domain-name"')

# Display DNS names
echo -e "Rates FQDN: $ratesFQDN\nInventory FQDN: $inventoryFQDN"

# Verify service-to-service communication from inventory to parking and review services
echo "Verifying service-to-service communication..."
kubectl exec deploy/inventory-ver1 -- curl -s $ratesFQDN/parking $ratesFQDN/review
kubectl exec deploy/parking -- curl -s $inventoryFQDN
```
