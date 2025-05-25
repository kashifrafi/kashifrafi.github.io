---
title: Bash Script For Lattice Configuration With Test Deployment Part-2
dates: 2024-01-31
categories: [AWS, VPC, Lattice, Test Deployment]
tags: [AWS, VPC, Lattice, Test Deployment]
---

### Automated Guide to VPC Lattice Setup and East-West & North South Traffic Testing in Amazon EKS 
Modern applications require seamless and secure service-to-service communication, especially when operating at scale in the cloud. Amazon Elastic Kubernetes Service (EKS) combined with AWS VPC Lattice provides an efficient way to implement scalable, east-west communication and streamline traffic management between services.

This script is a step-by-step guide to deploying the AWS Gateway API Controller for EKS using VPC Lattice. It simplifies the setup process by automating key configurations, such as IAM policies, service accounts, and Gateway API installations, ensuring a smooth integration with your Kubernetes environment.

By following this script, you'll:

1. Set up the AWS Gateway API Controller with Helm.
2. Deploy GatewayClass and Gateway configurations for traffic routing.
3. Establish and verify north-south connectivity using HTTPRoutes.
4. Test service-to-service communication with DNS-based routing.

### Configuring AWS Load Balancer Controller for EKS Clusters

The AWS Load Balancer Controller is a Kubernetes controller that automates the creation and management of AWS load balancers, such as Application Load Balancers (ALB) and Network Load Balancers (NLB), for Kubernetes services. It simplifies traffic routing, enhances scalability, and provides fault tolerance for your applications.
Why Use AWS Load Balancer Controller?
1. Automates the creation and configuration of load balancers for Kubernetes services.
2. Integrates seamlessly with AWS services like IAM, security groups, and VPCs.
3. Scales dynamically to handle traffic changes and pod scaling.
4. Reduces manual effort and ensures efficient use of resources.

```bash
#!/bin/bash

# ================================
# Set Environment Variables
# ================================
export CLUSTER_NAME="xxxxxxxxx"
export AWS_REGION="xxxxxxx"  # Update with your AWS region
export AWS_ACCOUNT_ID="xxxxxxxxxx"
export VPC_ID="xxxxxxx"  # Update with your VPC ID
export NLB_ARN="arn:aws:elasticloadbalancing:ap-south-1:$AWS_ACCOUNT_ID:loadbalancer/net/icp-prod-nlb-eks1/34b0a324d1f40832"
export NLB_DNS="xxxxxxxxxx-34b0a324d1f40832.elb.ap-south-1.amazonaws.com"
export POLICY_NAME="AWSLoadBalancerControllerIAMPolicy-xxxx"  # Updated policy name
export IAM_ROLE_NAME="AmazonEKSLoadBalancerControllerRole-xxxxx"
export SUBNET_TAG_KEY="kubernetes.io/role/internal-elb"
export SUBNET_TAG_VALUE="1"

# ================================
# Download Recommended Inline Policy
# ================================
echo "Downloading recommended inline policy..."
curl -s https://raw.githubusercontent.com/aws/aws-application-networking-k8s/main/files/controller-installation/recommended-inline-policy.json \
  -o recommended-inline-policy.json

# ================================
# Create IAM Policy if Not Exists
# ================================
echo "Checking for existing IAM policy..."
VPCLatticeControllerIAMPolicyArn=$(aws iam list-policies --query 'Policies[?PolicyName==`VPCLatticeControllerIAMPolicy-eks-3`].Arn' --output text)
if [[ -z "$VPCLatticeControllerIAMPolicyArn" ]]; then
  echo "Creating IAM policy..."
  aws iam create-policy \
    --policy-name VPCLatticeControllerIAMPolicy-eks-3 \
    --policy-document file://recommended-inline-policy.json
  VPCLatticeControllerIAMPolicyArn=$(aws iam list-policies --query 'Policies[?PolicyName==`VPCLatticeControllerIAMPolicy-eks-3`].Arn' --output text)
else
  echo "IAM policy already exists: $VPCLatticeControllerIAMPolicyArn"
fi

# ================================
# Enable OIDC Provider if Not Associated
# ================================
echo "Checking OIDC provider association..."
OIDC_ASSOCIATED=$(eksctl utils associate-iam-oidc-provider --cluster $CLUSTER_NAME --region $AWS_REGION --approve --verbose 2>&1 | grep "IAM OIDC provider is already associated")
if [[ -z "$OIDC_ASSOCIATED" ]]; then
  echo "Associating IAM OIDC provider..."
  eksctl utils associate-iam-oidc-provider \
    --cluster $CLUSTER_NAME \
    --approve \
    --region $AWS_REGION
else
  echo "IAM OIDC provider is already associated."
fi

# ================================
# Create IAM Service Account if Not Exists
# ================================
echo "Checking IAM service account..."
if eksctl get iamserviceaccount --cluster=$CLUSTER_NAME | grep -q "gateway-api-controller"; then
  echo "IAM service account already exists."
else
  echo "Creating IAM service account..."
  eksctl create iamserviceaccount \
    --cluster=$CLUSTER_NAME \
    --namespace=aws-application-networking-system \
    --name=gateway-api-controller \
    --attach-policy-arn=$VPCLatticeControllerIAMPolicyArn \
    --override-existing-serviceaccounts \
    --region $AWS_REGION \
    --approve
fi

# ================================
# Install Gateway API Controller Using Helm
# ================================
echo "Checking if Gateway API Controller is already installed..."

if helm list -n aws-application-networking-system | grep -q "gateway-api-controller"; then
  echo "Gateway API Controller is already installed. Skipping installation."
else
  echo "Installing Gateway API Controller using Helm..."
  aws ecr-public get-login-password --region us-east-1 | helm registry login --username AWS --password-stdin public.ecr.aws

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

  echo "Waiting for the gateway-api-controller pod to be ready..."
  kubectl wait --namespace aws-application-networking-system \
    --for=condition=ready pod -l app.kubernetes.io/instance=gateway-api-controller \
    --timeout=300s

fi

# ================================
# Apply the GatewayClass manifest
# ================================
echo "Checking if GatewayClass already exists..."
if kubectl get gatewayclass lattice &>/dev/null; then
  echo "GatewayClass already exists. Skipping creation."
else
  echo "Creating GatewayClass..."
  kubectl apply -f https://raw.githubusercontent.com/aws/aws-application-networking-k8s/main/files/controller-installation/gatewayclass.yaml
fi

# ================================
# Clone Repository if Not Present
# ================================
GIT_REPO_DIR="aws-application-networking-k8s"
if [ ! -d "$GIT_REPO_DIR" ]; then
  echo "Cloning AWS Gateway API Controller repository..."
  git clone https://github.com/aws/aws-application-networking-k8s.git
else
  echo "Repository already cloned. Skipping clone step."
fi

cd $GIT_REPO_DIR

# ================================
# Update Helm Chart for Service Network
# ================================
echo "Checking if Helm chart needs an update..."
if helm list -n aws-application-networking-system | grep -q "gateway-api-controller"; then
  echo "Updating Helm chart to configure the default service network..."
  aws ecr-public get-login-password --region us-east-1 | helm registry login --username AWS --password-stdin public.ecr.aws

  helm upgrade gateway-api-controller \
    oci://public.ecr.aws/aws-application-networking-k8s/aws-gateway-controller-chart \
    --version=v1.0.6 \
    --reuse-values \
    --namespace aws-application-networking-system \
    --set=defaultServiceNetwork=xxxxxxxxxxxxxxxx

  echo "Waiting for the gateway-api-controller pod to be ready..."
  kubectl wait --namespace aws-application-networking-system \
    --for=condition=ready pod -l app.kubernetes.io/instance=gateway-api-controller \
    --timeout=300s
# echo "Waiting for 2 minutes to ensure the Helm update is complete..."
# sleep 120
else
  echo "Gateway API Controller is not installed or does not require an update. Skipping Helm upgrade."
fi

# ================================
# Apply Gateway and HTTPRoute Configurations
# ================================
echo "Applying Gateway and HTTPRoute configurations..."

apply_if_not_exists() {
  local resource_type=$1
  local resource_name=$2
  local file_path=$3

  if kubectl get "$resource_type" "$resource_name" &>/dev/null; then
    echo "$resource_type $resource_name already exists. Skipping apply."
  else
    echo "Applying $resource_type from $file_path..."
    kubectl apply -f "$file_path"
  fi
}

apply_if_not_exists gateway my-hotel-gateway /aws-application-networking-k8s/files/examples/my-hotel-gateway.yaml
apply_if_not_exists httproute parking /aws-application-networking-k8s/files/examples/parking.yaml
apply_if_not_exists httproute review /aws-application-networking-k8s/files/examples/review.yaml
apply_if_not_exists httproute inventory-ver1 /aws-application-networking-k8s/files/examples/inventory-ver1.yaml

echo "Waiting for 2 minutes after applying HTTPRoute configurations..."
sleep 120

apply_if_not_exists httproute rate-route-path /aws-application-networking-k8s/files/examples/rate-route-path.yaml
apply_if_not_exists httproute inventory-route /aws-application-networking-k8s/files/examples/inventory-route.yaml

echo "Waiting for 2 minutes after applying HTTPRoute configurations..."
sleep 120


# ================================
# Fetch DNS Names for HTTPRoutes
# ================================
echo "Fetching DNS names for HTTPRoutes..."
ratesFQDN=$(kubectl get httproute rates-2 -o json | jq -r '.metadata.annotations."application-networking.k8s.aws/lattice-assigned-domain-name"')
inventoryFQDN=$(kubectl get httproute inventory-2 -o json | jq -r '.metadata.annotations."application-networking.k8s.aws/lattice-assigned-domain-name"')

if [[ -z "$ratesFQDN" || "$ratesFQDN" == "null" ]]; then
  echo "Rates FQDN not found!"
else
  echo "Rates FQDN: $ratesFQDN"
fi

if [[ -z "$inventoryFQDN" || "$inventoryFQDN" == "null" ]]; then
  echo "Inventory FQDN not found!"
else
  echo "Inventory FQDN: $inventoryFQDN"
fi

# ================================
# Test Service-to-Service Communication
# ================================
echo "Testing service-to-service communication..."

# ================================
# Test Service-to-Service Communication
# ================================
echo "Testing service-to-service communication..."
kubectl exec deploy/inventory-ver1-1 -- curl -s $ratesFQDN/parking
kubectl exec deploy/inventory-ver1-1 -- curl -s $ratesFQDN/review
kubectl exec deploy/parking-1 -- curl -s $inventoryFQDN

echo "East-West service to service communication tested successfully"

# ================================
# Configuration of AWS ELB Controller
# ================================


# Step 1: Tag EKS cluster subnets
echo "Tagging EKS cluster subnets..."
SUBNETS=$(aws eks describe-cluster --name $CLUSTER_NAME --region $AWS_REGION --query "cluster.resourcesVpcConfig.subnetIds" --output text)
for SUBNET in $SUBNETS; do
    echo "Tagging subnet $SUBNET with key $SUBNET_TAG_KEY and value $SUBNET_TAG_VALUE..."
    aws ec2 create-tags --resources $SUBNET --tags Key=$SUBNET_TAG_KEY,Value=$SUBNET_TAG_VALUE --region $AWS_REGION
done

# Step 2: Create IAM Policy if not exists
echo "Checking if IAM Policy exists..."
POLICY_ARN="arn:aws:iam::$AWS_ACCOUNT_ID:policy/$POLICY_NAME"
POLICY_EXISTS=$(aws iam list-policies --query "Policies[?PolicyName=='$POLICY_NAME']" --output json | jq 'length')

if [ "$POLICY_EXISTS" -eq 0 ]; then
    echo "Creating IAM Policy..."
    curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.7.2/docs/install/iam_policy.json
    aws iam create-policy --policy-name $POLICY_NAME --policy-document file://iam_policy.json
else
    echo "IAM Policy '$POLICY_NAME' already exists. Skipping creation."
fi

# Step 3: Create IAM Role if not exists
echo "Checking if IAM Role exists..."
ROLE_EXISTS=$(aws iam get-role --role-name $IAM_ROLE_NAME --query "Role.RoleName" --output text 2>/dev/null)

if [ "$ROLE_EXISTS" == "$IAM_ROLE_NAME" ]; then
    echo "IAM Role '$IAM_ROLE_NAME' already exists. Skipping creation."
else
    echo "Creating IAM service account using eksctl..."
    eksctl create iamserviceaccount \
      --cluster=$CLUSTER_NAME \
      --namespace=kube-system \
      --name=aws-load-balancer-controller \
      --role-name $IAM_ROLE_NAME \
      --attach-policy-arn=$POLICY_ARN \
      --approve
fi

# Step 4: Install AWS Load Balancer Controller
echo "Checking if AWS Load Balancer Controller is already installed..."
if helm list -n kube-system | grep -q aws-load-balancer-controller; then
    echo "AWS Load Balancer Controller already exists. Skipping installation."
else
    echo "Installing AWS Load Balancer Controller..."
    helm repo add eks https://aws.github.io/eks-charts
    helm repo update eks
    helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
      -n kube-system \
      --set clusterName=$CLUSTER_NAME \
      --set serviceAccount.create=false \
      --set serviceAccount.name=aws-load-balancer-controller \
      --set region=$AWS_REGION \
      --set vpcId=$VPC_ID
fi

# Step 5: Verify Installation
echo "Verifying AWS Load Balancer Controller installation..."
while true; do
  STATUS=$(kubectl get deployment -n kube-system aws-load-balancer-controller -o jsonpath='{.status.conditions[?(@.type=="Available")].status}')
  if [ "$STATUS" == "True" ]; then
    echo "AWS Load Balancer Controller is up and running."
    break
  else
    echo "Waiting for AWS Load Balancer Controller to be ready..."
    sleep 10
  fi
done

kubectl get pods -n kube-system
echo "AWS Load Balancer Controller setup completed."

# ================================
# Check if NGINX Deployment Exists
# ================================
echo "Checking if NGINX Deployment exists..."
if kubectl get deployment nginx-new &>/dev/null; then
  echo "NGINX Deployment already exists, skipping creation."
else
  echo "Creating NGINX Deployment and ClusterIP Service..."
  cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-new
  labels:
    app: nginx-new
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-new
  template:
    metadata:
      labels:
        app: nginx-new
    spec:
      containers:
        - name: nginx-new
          image: nginx
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service1-new
  labels:
    app: nginx-new
spec:
  selector:
    app: nginx-new
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
  type: ClusterIP
EOF
  echo "NGINX Deployment and Service created successfully."
fi

# ================================
# Wait for NGINX Pods to be Ready
# ================================
echo "Waiting for NGINX pods to be ready..."
kubectl wait --for=condition=ready pod -l app=nginx-new --timeout=300s

# ================================
# Check if Target Group Exists
# ================================
TG_NAME="tg-nginx-new"
TG_ARN=$(aws elbv2 describe-target-groups --names "$TG_NAME" --query 'TargetGroups[0].TargetGroupArn' --output text 2>/dev/null)

if [[ -n "$TG_ARN" ]]; then
  echo "Target Group $TG_NAME already exists. ARN: $TG_ARN"
else
  echo "Creating Target Group..."
  TG_ARN=$(aws elbv2 create-target-group \
    --name "$TG_NAME" \
    --protocol TCP \
    --port 6070 \
    --vpc-id "$VPC_ID" \
    --target-type ip \
    --query 'TargetGroups[0].TargetGroupArn' \
    --output text)
  echo "Target Group created. ARN: $TG_ARN"
fi

# ================================
# Check if Listener Exists
# ================================
LISTENER_EXISTS=$(aws elbv2 describe-listeners --load-balancer-arn "$NLB_ARN" --query "Listeners[?Port==\`6070\`]" --output text 2>/dev/null)

if [[ -n "$LISTENER_EXISTS" ]]; then
  echo "Listener on port 6070 already exists, skipping creation."
else
  echo "Creating Listener for port 6070..."
  aws elbv2 create-listener \
    --load-balancer-arn "$NLB_ARN" \
    --protocol TCP \
    --port 6070 \
    --default-actions Type=forward,TargetGroupArn="$TG_ARN"
  echo "Listener created on port 6070."
fi

# ================================
# Check if Target Group Binding Exists
# ================================
if kubectl get targetgroupbinding tgb-nginx &>/dev/null; then
  echo "Target Group Binding already exists, skipping creation."
else
  echo "Creating Target Group Binding..."
  cat <<EOF | kubectl apply -f -
apiVersion: elbv2.k8s.aws/v1beta1
kind: TargetGroupBinding
metadata:
  name: tgb-nginx-new
spec:
  serviceRef:
    name: nginx-service1-new
    port: 80
  targetGroupARN: $TG_ARN
EOF
  echo "Target Group Binding created."
fi

# ================================
# Final Verification
# ================================
echo "Setup completed. Verifying services and resources..."
if curl -s "$NLB_DNS:6070"; then
  echo "Curl test successful. All tasks finished successfully."
else
  echo "Curl test failed. Please check service connectivity."
fi
```

### Clean up 

```bash
# =====================================================================
cleanup
# =====================================================================

cat nsew_v3_cleanup.sh
#!/bin/bash

# ================================
# Set Environment Variables
# ================================
export CLUSTER_NAME="xxxxxxxxxxxx"
export AWS_REGION="xxxxxxxxxxx"
export AWS_ACCOUNT_ID="4xxxxxxxxx"
export VPC_ID="vpc-xxxxxxxxxx"
export NLB_ARN="arn:aws:elasticloadbalancing:ap-south-1:$AWS_ACCOUNT_ID:loadbalancer/net/icp-prod-nlb-eks1/34b0a324d1f40832"
export POLICY_NAME="AWSLoadBalancerControllerIAMPolicy-xxxxxxxxx"
export IAM_ROLE_NAME="AmazonEKSLoadBalancerControllerRole-xxxxxxxx"
export TG_NAME="tg-nginx-new"
export TG_ARN=$(aws elbv2 describe-target-groups --names "$TG_NAME" --query 'TargetGroups[0].TargetGroupArn' --output text 2>/dev/null)

# ================================
# Delete Target Group Binding
# ================================
echo "Deleting Target Group Binding..."
kubectl delete targetgroupbinding tgb-nginx-new --ignore-not-found

# ================================
# Delete Listener
# ================================
LISTENER_ARN=$(aws elbv2 describe-listeners --load-balancer-arn "$NLB_ARN" --query "Listeners[?Port==\`6070\`].ListenerArn" --output text 2>/dev/null)
if [[ -n "$LISTENER_ARN" ]]; then
  echo "Deleting Listener on port 6070..."
  aws elbv2 delete-listener --listener-arn "$LISTENER_ARN"
fi

# ================================
# Delete Target Group
# ================================
if [[ -n "$TG_ARN" ]]; then
  echo "Deleting Target Group $TG_NAME..."
  aws elbv2 delete-target-group --target-group-arn "$TG_ARN"
fi

# ================================
# Delete NGINX Deployment and Service
# ================================
echo "Deleting NGINX Deployment and Service..."
kubectl delete deployment nginx-new --ignore-not-found
kubectl delete service nginx-service1-new --ignore-not-found

# ================================
# Uninstall AWS Load Balancer Controller
# ================================
echo "Uninstalling AWS Load Balancer Controller..."
helm uninstall aws-load-balancer-controller -n kube-system --ignore-not-found

# ================================
# Delete IAM Service Account
# ================================
echo "Deleting IAM Service Account..."
eksctl delete iamserviceaccount --cluster=$CLUSTER_NAME --namespace=kube-system --name=aws-load-balancer-controller --wait

# ================================
# Delete IAM Role
# ================================
echo "Deleting IAM Role..."
aws iam delete-role --role-name $IAM_ROLE_NAME 2>/dev/null

# ================================
# Delete IAM Policy
# ================================
POLICY_ARN="arn:aws:iam::$AWS_ACCOUNT_ID:policy/$POLICY_NAME"
echo "Deleting IAM Policy..."
aws iam delete-policy --policy-arn "$POLICY_ARN" 2>/dev/null

# ================================
# Uninstall Gateway API Controller
# ================================
echo "Uninstalling Gateway API Controller..."
helm uninstall gateway-api-controller -n aws-application-networking-system --ignore-not-found

# ================================
# Delete Gateway and HTTPRoute Configurations
# ================================

echo "Patching the httproute and gateway configurations..."

  kubectl patch httproute inventory-2 -p '{"metadata":{"finalizers":null}}' --type=merge
  kubectl patch httproute rates-2 -p '{"metadata":{"finalizers":null}}' --type=merge
  kubectl patch gateway icp-prod-latticeservicenetwork-1 -p '{"metadata":{"finalizers":null}}' --type=merge
echo "Deleting HTTPRoute configurations..."

  kubectl delete httproute rates-2 --ignore-not-found
  kubectl delete httproute inventory-2 --ignore-not-found
  kubectl delete deploy parking --ignore-not-found
  kubectl delete deploy review --ignore-not-found
  kubectl delete deploy inventory-ver1 --ignore-not-found
  kubectl delete svc review-service1 --ignore-not-found
  kubectl delete svc parking-service1 --ignore-not-found
  kubectl delete svc inventory-ver1-service1 --ignore-not-found
  kubectl delete gateway icp-prod-latticeservicenetwork-1 --ignore-not-found
echo "HTTPRoute/Deployment/Service/Gateway configurations deleted."

# comment out below command if you want to delete through manifest
# kubectl delete -f /aws-application-networking-k8s/files/examples/my-hotel-gateway.yaml --ignore-not-found
# kubectl delete -f /aws-application-networking-k8s/files/examples/parking.yaml --ignore-not-found
# kubectl delete -f /aws-application-networking-k8s/files/examples/review.yaml --ignore-not-found
# kubectl delete -f /aws-application-networking-k8s/files/examples/inventory-ver1.yaml --ignore-not-found
# echo "Waiting for 30 seconds for the pod cleanup"
# sleep 30
# echo "Deleting additional HTTPRoute Configurations..."
# kubectl delete -f /aws-application-networking-k8s/files/examples/rate-route-path.yaml --ignore-not-found
# kubectl delete -f /aws-application-networking-k8s/files/examples/inventory-route.yaml --ignore-not-found

# ================================
# Delete IAM Policy for VPCLatticeController
# ================================
echo "Deleting VPCLatticeController IAM Policy..."
aws iam delete-policy --policy-arn "arn:aws:iam::$AWS_ACCOUNT_ID:policy/VPCLatticeControllerIAMPolicy-eks-3" 2>/dev/null

# ================================
# Cleanup Completed
# ================================
echo "Cleanup completed successfully."

```
<!-- LikeBtn.com BEGIN -->
<span class="likebtn-wrapper" data-i18n_like="Like" data-identifier="item_1"></span>
<script>(function(d,e,s){if(d.getElementById("likebtn_wjs"))return;a=d.createElement(e);m=d.getElementsByTagName(e)[0];a.async=1;a.id="likebtn_wjs";a.src=s;m.parentNode.insertBefore(a, m)})(document,"script","//w.likebtn.com/js/w/widget.js");</script>
<!-- LikeBtn.com END -->
