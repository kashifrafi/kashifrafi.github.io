---
title: CF EC2 with tools installed
dates: 2024-01-25
categories: [AWS, EC2, EKSCTL, KUBECTL, HELM, AWSCLI]
tags: [AWS, EC2, EKSCTL, KUBECTL, HELM, AWSCLI]
---


# CloudFormation Template to Provision an EC2 Instance with Essential Tools

This CloudFormation template provisions an EC2 instance in a specific VPC and subnet. It installs essential tools such as Git, kubectl, eksctl, Helm, and configures AWS CLI with hardcoded credentials.

## Features

- Provisions an **Amazon Linux 2** EC2 instance.
- Installs the following tools:
  - Git
  - kubectl
  - eksctl
  - Helm
- Configures the AWS CLI with pre-defined credentials.
- Creates a Security Group allowing all ingress and egress traffic.

---

## CloudFormation Template

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: "CloudFormation template to provision an EC2 instance in a specific VPC and subnet, install Git, kubectl, eksctl, Helm, and configure AWS CLI with hardcoded values."

Parameters:
  InstanceType:
    Description: "EC2 instance type"
    Type: String
    Default: "t3.micro"
    AllowedValues:
      - t2.micro
      - t3.micro
      - t3.small
    ConstraintDescription: "Must be a valid EC2 instance type."

Resources:
  EC2Instance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: !Ref InstanceType
      ImageId: ami-xxxxxxxxxxxxxxx # Amazon Linux 2 AMI (update as per region)
      SubnetId: subnet-xxxxxxxxxxxxxx # your subnet
      SecurityGroupIds:
        - !Ref InstanceSecurityGroup
      UserData:
        Fn::Base64: !Sub |
          #!/bin/bash
          yum update -y

          # Install Git
          yum install git -y

          # Install kubectl
          curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.31.2/2024-11-15/bin/linux/amd64/kubectl
          curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.31.2/2024-11-15/bin/linux/amd64/kubectl.sha256
          sha256sum -c kubectl.sha256
          install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
          chmod +x kubectl
          mkdir -p ~/.local/bin
          mv ./kubectl ~/.local/bin/kubectl

          # Install eksctl
          curl --silent --location https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz | tar xz -C /tmp
          mv /tmp/eksctl /usr/local/bin

          # Install Helm
          curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
          chmod 700 get_helm.sh
          ./get_helm.sh

          # Configure AWS CLI with hardcoded values
          mkdir -p /root/.aws
          cat <<EOT > /root/.aws/credentials
          [default]
          aws_access_key_id = YOUR_ACCESS_KEY_ID
          aws_secret_access_key = YOUR_SECRET_ACCESS_KEY
          EOT

          cat <<EOT > /root/.aws/config
          [default]
          region = us-west-2
          output = json
          EOT

          echo "AWS CLI configured with hardcoded values."

  InstanceSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: "Allow HTTP and HTTPS traffic"
      VpcId: vpc-xxxxxxxxxxxxxx # your VPC id
      SecurityGroupIngress:
        - IpProtocol: "-1"
          CidrIp: 0.0.0.0/0
      SecurityGroupEgress:
        - IpProtocol: "-1"
          CidrIp: 0.0.0.0/0

Outputs:
  InstanceId:
    Description: "The instance ID of the EC2 instance"
    Value: !Ref EC2Instance

  PublicIP:
    Description: "Public IP address of the EC2 instance"
    Value: !GetAtt EC2Instance.PublicIp
```
