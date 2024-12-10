---
title: VPC Lattice Setup
dates: 2024-10-12
categories: [AWS, VPC, EKS, VPC Lattice]
tags: [AWS, VPC, EKS, VPC Lattice]
---

# Setting Up Amazon VPC Lattice for EKS with AWS Gateway API Controller

## Set AWS Region and Cluster Name as Environment Variables

Set the AWS region and cluster name as environment variables. Refer to the Amazon VPC Lattice FAQs for a list of supported regions.

```bash
export AWS_REGION=<cluster_region>
export CLUSTER_NAME=<cluster_name>
CLUSTER_SG=<your_node_security_group>

If you created the cluster with the eksctl create cluster --name $CLUSTER_NAME --region $AWS_REGION command, you can use this command to export the Security Group ID:
CLUSTER_SG=$(aws eks describe-cluster --name $CLUSTER_NAME --output json | jq -r '.cluster.resourcesVpcConfig.clusterSecurityGroupId')

Prefix Addition for IPV4 and IPV6
To allow traffic from the correct prefix list for IPv4 and IPv6, run the following commands:
# IPv4
PREFIX_LIST_ID=$(aws ec2 describe-managed-prefix-lists --query "PrefixLists[?PrefixListName=='com.amazonaws.$AWS_REGION.vpc-lattice'].PrefixListId" | jq -r '.[]')
aws ec2 authorize-security-group-ingress --group-id $CLUSTER_SG --ip-permissions "PrefixListIds=[{PrefixListId=${PREFIX_LIST_ID}}],IpProtocol=-1"

# IPv6
PREFIX_LIST_ID_IPV6=$(aws ec2 describe-managed-prefix-lists --query "PrefixLists[?PrefixListName=='com.amazonaws.$AWS_REGION.ipv6.vpc-lattice'].PrefixListId" | jq -r '.[]')
aws ec2 authorize-security-group-ingress --group-id $CLUSTER_SG --ip-permissions "PrefixListIds=[{PrefixListId=${PREFIX_LIST_ID_IPV6}}],IpProtocol=-1"

Creating and Attaching the IAM Policy
Download and create the IAM policy for VPC Lattice Controller.

curl https://raw.githubusercontent.com/aws/aws-application-networking-k8s/main/files/controller-installation/recommended-inline-policy.json -o recommended-inline-policy.json
aws iam create-policy \
    --policy-name VPCLatticeControllerIAMPolicy \
    --policy-document file://recommended-inline-policy.json

export VPCLatticeControllerIAMPolicyArn=$(aws iam list-policies --query 'Policies[?PolicyName==`VPCLatticeControllerIAMPolicy`].Arn' --output text)


markdown
Copy code
# Setting Up Amazon VPC Lattice for EKS with AWS Gateway API Controller

## Set AWS Region and Cluster Name as Environment Variables

Set the AWS region and cluster name as environment variables. Refer to the Amazon VPC Lattice FAQs for a list of supported regions.

```bash
export AWS_REGION=<cluster_region>
export CLUSTER_NAME=<cluster_name>
CLUSTER_SG=<your_node_security_group>
If you created the cluster with the eksctl create cluster --name $CLUSTER_NAME --region $AWS_REGION command, you can use this command to export the Security Group ID:

bash
Copy code
CLUSTER_SG=$(aws eks describe-cluster --name $CLUSTER_NAME --output json | jq -r '.cluster.resourcesVpcConfig.clusterSecurityGroupId')
Prefix Addition for IPV4 and IPV6
To allow traffic from the correct prefix list for IPv4 and IPv6, run the following commands:

bash
Copy code
# IPv4
PREFIX_LIST_ID=$(aws ec2 describe-managed-prefix-lists --query "PrefixLists[?PrefixListName=='com.amazonaws.$AWS_REGION.vpc-lattice'].PrefixListId" | jq -r '.[]')
aws ec2 authorize-security-group-ingress --group-id $CLUSTER_SG --ip-permissions "PrefixListIds=[{PrefixListId=${PREFIX_LIST_ID}}],IpProtocol=-1"

# IPv6
PREFIX_LIST_ID_IPV6=$(aws ec2 describe-managed-prefix-lists --query "PrefixLists[?PrefixListName=='com.amazonaws.$AWS_REGION.ipv6.vpc-lattice'].PrefixListId" | jq -r '.[]')
aws ec2 authorize-security-group-ingress --group-id $CLUSTER_SG --ip-permissions "PrefixListIds=[{PrefixListId=${PREFIX_LIST_ID_IPV6}}],IpProtocol=-1"
Creating and Attaching the IAM Policy
Download and create the IAM policy for VPC Lattice Controller.

bash
Copy code
curl https://raw.githubusercontent.com/aws/aws-application-networking-k8s/main/files/controller-installation/recommended-inline-policy.json -o recommended-inline-policy.json
aws iam create-policy \
    --policy-name VPCLatticeControllerIAMPolicy \
    --policy-document file://recommended-inline-policy.json

export VPCLatticeControllerIAMPolicyArn=$(aws iam list-policies --query 'Policies[?PolicyName==`VPCLatticeControllerIAMPolicy`].Arn' --output text)
Apply the Namespace Manifest
Apply the Kubernetes manifest for the namespace.
kubectl apply -f https://raw.githubusercontent.com/aws/aws-application-networking-k8s/main/files/controller-installation/deploy-namesystem.yaml

You can also use AWS IAM Roles for Service Accounts (IRSA) to assign the necessary permissions via a ServiceAccount.

Create IAM OIDC Provider
To associate the IAM OIDC provider for your cluster, run:
eksctl utils associate-iam-oidc-provider --cluster $CLUSTER_NAME --approve --region $AWS_REGION

Create IAM Service Account for Pod-Level Permission
Create an IAM service account to attach the policy:
eksctl create iamserviceaccount \
    --cluster=$CLUSTER_NAME \
    --namespace=aws-application-networking-system \
    --name=gateway-api-controller \
    --attach-policy-arn=$VPCLatticeControllerIAMPolicyArn \
    --override-existing-serviceaccounts \
    --region $AWS_REGION \
    --approve

Clean Up IAM Service Account (Optional)
To delete the IAM service account if needed, run:
eksctl delete iamserviceaccount \
    --cluster=$CLUSTER_NAME \
    --namespace=aws-application-networking-system \
    --name=gateway-api-controller \
    --region=$AWS_REGION \
    --wait
Install the Controller Using Helm
Log in to the AWS Public ECR and install the controller using the following Helm chart:
aws ecr-public get-login-password --region us-east-1 | helm registry login --username AWS --password-stdin public.ecr.aws
helm install gateway-api-controller \
    oci://public.ecr.aws/aws-application-networking-k8s/aws-gateway-controller-chart \
    --version=v1.0.6 \
    --set=serviceAccount.create=false \
    --namespace aws-application-networking-system \
    --set=awsRegion=ap-south-1 \                                                      # Provide the region here
    --set=clusterVpcId=vpc-0727d00ef47b05913 \                     # Provide the cluster VPC ID
    --set=awsAccountId=345594602021 \                                       # Provide the AWS account ID
    --set=clusterName=icp-dev-eks \                                                 # Provide the Cluster Name
    --set=log.level=info # use "debug" for debug level logs


Note: The following values are required to ensure the Helm chart works properly, as we are using Fargate compute. Otherwise, the controller pod will enter a CrashLoopBackOff state:

awsRegion
clusterVpcId
awsAccountId
clusterName
Check Controller Status
After installing the Helm chart, verify that the controller is running with the following command:
kubectl --namespace aws-application-networking-system get pods -l "app.kubernetes.io/instance=gateway-api-controller"

Create Amazon VPC Lattice GatewayClass
Apply the GatewayClass manifest:
kubectl apply -f https://raw.githubusercontent.com/aws/aws-application-networking-k8s/main/files/controller-installation/gatewayclass.yaml

Set Up Service-to-Service Communication with VPC Lattice
Clone the AWS Gateway API Controller repository:
git clone https://github.com/aws/aws-application-networking-k8s.git
cd aws-application-networking-k8s


markdown
Copy code
# Setting Up Amazon VPC Lattice for EKS with AWS Gateway API Controller

## Set AWS Region and Cluster Name as Environment Variables

Set the AWS region and cluster name as environment variables. Refer to the Amazon VPC Lattice FAQs for a list of supported regions.

```bash
export AWS_REGION=<cluster_region>
export CLUSTER_NAME=<cluster_name>
CLUSTER_SG=<your_node_security_group>
If you created the cluster with the eksctl create cluster --name $CLUSTER_NAME --region $AWS_REGION command, you can use this command to export the Security Group ID:

bash
Copy code
CLUSTER_SG=$(aws eks describe-cluster --name $CLUSTER_NAME --output json | jq -r '.cluster.resourcesVpcConfig.clusterSecurityGroupId')
Prefix Addition for IPV4 and IPV6
To allow traffic from the correct prefix list for IPv4 and IPv6, run the following commands:

bash
Copy code
# IPv4
PREFIX_LIST_ID=$(aws ec2 describe-managed-prefix-lists --query "PrefixLists[?PrefixListName=='com.amazonaws.$AWS_REGION.vpc-lattice'].PrefixListId" | jq -r '.[]')
aws ec2 authorize-security-group-ingress --group-id $CLUSTER_SG --ip-permissions "PrefixListIds=[{PrefixListId=${PREFIX_LIST_ID}}],IpProtocol=-1"

# IPv6
PREFIX_LIST_ID_IPV6=$(aws ec2 describe-managed-prefix-lists --query "PrefixLists[?PrefixListName=='com.amazonaws.$AWS_REGION.ipv6.vpc-lattice'].PrefixListId" | jq -r '.[]')
aws ec2 authorize-security-group-ingress --group-id $CLUSTER_SG --ip-permissions "PrefixListIds=[{PrefixListId=${PREFIX_LIST_ID_IPV6}}],IpProtocol=-1"
Creating and Attaching the IAM Policy
Download and create the IAM policy for VPC Lattice Controller.

bash
Copy code
curl https://raw.githubusercontent.com/aws/aws-application-networking-k8s/main/files/controller-installation/recommended-inline-policy.json -o recommended-inline-policy.json
aws iam create-policy \
    --policy-name VPCLatticeControllerIAMPolicy \
    --policy-document file://recommended-inline-policy.json

export VPCLatticeControllerIAMPolicyArn=$(aws iam list-policies --query 'Policies[?PolicyName==`VPCLatticeControllerIAMPolicy`].Arn' --output text)
Apply the Namespace Manifest
Apply the Kubernetes manifest for the namespace.

bash
Copy code
kubectl apply -f https://raw.githubusercontent.com/aws/aws-application-networking-k8s/main/files/controller-installation/deploy-namesystem.yaml
You can also use AWS IAM Roles for Service Accounts (IRSA) to assign the necessary permissions via a ServiceAccount.

Create IAM OIDC Provider
To associate the IAM OIDC provider for your cluster, run:

bash
Copy code
eksctl utils associate-iam-oidc-provider --cluster $CLUSTER_NAME --approve --region $AWS_REGION
Create IAM Service Account for Pod-Level Permission
Create an IAM service account to attach the policy:

bash
Copy code
eksctl create iamserviceaccount \
    --cluster=$CLUSTER_NAME \
    --namespace=aws-application-networking-system \
    --name=gateway-api-controller \
    --attach-policy-arn=$VPCLatticeControllerIAMPolicyArn \
    --override-existing-serviceaccounts \
    --region $AWS_REGION \
    --approve
Clean Up IAM Service Account (Optional)
To delete the IAM service account if needed, run:

bash
Copy code
eksctl delete iamserviceaccount \
    --cluster=$CLUSTER_NAME \
    --namespace=aws-application-networking-system \
    --name=gateway-api-controller \
    --region=$AWS_REGION \
    --wait
Install the Controller Using Helm
Log in to the AWS Public ECR and install the controller using the following Helm chart:

bash
Copy code
aws ecr-public get-login-password --region us-east-1 | helm registry login --username AWS --password-stdin public.ecr.aws
helm install gateway-api-controller \
    oci://public.ecr.aws/aws-application-networking-k8s/aws-gateway-controller-chart \
    --version=v1.0.6 \
    --set=serviceAccount.create=false \
    --namespace aws-application-networking-system \
    --set=awsRegion=ap-south-1 \                                                      # Provide the region here
    --set=clusterVpcId=vpc-0727d00ef47b05913 \                     # Provide the cluster VPC ID
    --set=awsAccountId=345594602021 \                                       # Provide the AWS account ID
    --set=clusterName=icp-dev-eks \                                                 # Provide the Cluster Name
    --set=log.level=info # use "debug" for debug level logs
Note: The following values are required to ensure the Helm chart works properly, as we are using Fargate compute. Otherwise, the controller pod will enter a CrashLoopBackOff state:

awsRegion
clusterVpcId
awsAccountId
clusterName
Check Controller Status
After installing the Helm chart, verify that the controller is running with the following command:

bash
Copy code
kubectl --namespace aws-application-networking-system get pods -l "app.kubernetes.io/instance=gateway-api-controller"
Create Amazon VPC Lattice GatewayClass
Apply the GatewayClass manifest:

bash
Copy code
kubectl apply -f https://raw.githubusercontent.com/aws/aws-application-networking-k8s/main/files/controller-installation/gatewayclass.yaml
Set Up Service-to-Service Communication with VPC Lattice
Clone the AWS Gateway API Controller repository:

bash
Copy code
git clone https://github.com/aws/aws-application-networking-k8s.git
cd aws-application-networking-k8s
Set Up In-Cluster Service-to-Service Communication
Important: Before proceeding, manually create the Service Network, VPC Association, and Service Association in the AWS console, then continue as described.

The AWS Gateway API Controller needs a VPC Lattice service network to operate. You can update the chart configurations to specify the default service network:
aws ecr-public get-login-password --region us-east-1 | helm registry login --username AWS --password-stdin public.ecr.aws
helm upgrade gateway-api-controller \
oci://public.ecr.aws/aws-application-networking-k8s/aws-gateway-controller-chart \
--version=v1.0.6 \
--reuse-values \
--namespace aws-application-networking-system \
--set=defaultServiceNetwork=icp-dev-latticeservicenetwork

Create the Kubernetes Gateway
Before applying the file, change the name to icp-dev-latticeservicenetwork:
kubectl apply -f files/examples/my-hotel-gateway.yaml

Verify the gateway status:
kubectl get gateway

Output:
NAME       CLASS                ADDRESS   PROGRAMMED   AGE
my-hotel   amazon-vpc-lattice               True      7d12h

Create Kubernetes HTTPRoute
Create HTTP routes for rates, inventory, and other services:
kubectl apply -f files/examples/parking.yaml
kubectl apply -f files/examples/review.yaml
kubectl apply -f files/examples/rate-route-path.yaml
kubectl apply -f files/examples/inventory-ver1.yaml
kubectl apply -f files/examples/inventory-route.yaml

Check HTTPRoute DNS Name
Get the DNS name for the HTTPRoute:
kubectl get httproute

Output:
NAME        HOSTNAMES      AGE
inventory        		       51s
rates               		       6m11s	

Check VPC Lattice generated DNS address:
kubectl get httproute inventory -o yaml
kubectl get httproute rates -o yaml

Store the DNS names into variables:
ratesFQDN=$(kubectl get httproute rates -o json | jq -r '.metadata.annotations."application-networking.k8s.aws/lattice-assigned-domain-name"')
inventoryFQDN=$(kubectl get httproute inventory -o json | jq -r '.metadata.annotations."application-networking.k8s.aws/lattice-assigned-domain-name"')

echo "$ratesFQDN \n$inventoryFQDN"

Verify Service-to-Service Communication
Check connectivity from the inventory-ver1 service to the parking and review services:
kubectl exec deploy/inventory-ver1 -- curl -s $ratesFQDN/parking $ratesFQDN/review

Check connectivity from the parking service to the inventory-ver1 service:
kubectl exec deploy/parking -- curl -s $inventoryFQDN



