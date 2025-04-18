---
title: Amazon VPC Lattice
dates: 2024-12-12
categories: [AWS, VPC, EKS, VPC Lattice, service discovery]
tags: [AWS, VPC, EKS, VPC Lattice, service discovery]
---

# Amazon VPC Lattice: Simplifying Application Networking  

Amazon VPC Lattice is a fully managed application networking service built directly into AWS infrastructure. It helps connect, secure, and monitor your services across multiple AWS accounts and virtual private clouds (VPCs).  

With **Amazon EKS**, you can use **VPC Lattice** through the **AWS Gateway API Controller**, which follows standard Kubernetes Gateway API specifications. This setup simplifies cross-cluster connectivity while maintaining consistent Kubernetes semantics.  

---

## Key Features of Amazon VPC Lattice  

### 1. Service Directory  
The **Service Directory** is an account-level catalog where all your services are organized in one place. It provides a centralized view of:  
- Your services.  
- Services shared with you.  

For instance, a service might route traffic on port 80 (HTTP) but could use different rules (like path or query parameters) to send traffic to various targets, such as:  
- Kubernetes pods.  
- AWS Lambda functions.  

### 2. Service Network  
A **Service Network** is a logical layer that connects services across different VPCs and accounts. It simplifies networking by:  
- Automatically handling **service discovery** and **connectivity**.  
- Managing access and observability policies.  
- Supporting HTTP, HTTPS, and gRPC protocols.  

When a VPC is part of a service network, clients in that VPC can automatically discover available services via DNS and communicate through VPC Lattice. **AWS Resource Access Manager (RAM)** is used to control which accounts and VPCs can communicate within the network.

### 3. Service Policies  
Service policies let you manage observability, access, and traffic control. These policies help:  
- Control who can access your services.  
- Define how traffic is handled.  
- Assign IAM roles for authorizing requests (similar to S3 or IAM policies).  

Policies can be applied at both the service and service network levels for consistent and secure access rules.  

### 4. Services in VPC Lattice  
A **service** in VPC Lattice is any independently deployable unit (like a Kubernetes pod, virtual machine, or Lambda function) that performs a specific task.  

Each service has a configuration that includes:  
- **Listeners**: Define the port and protocol (e.g., HTTP/1.1, HTTP/2, HTTPS, or gRPC) the service accepts traffic on.  
- **Rules**: Specify conditions for routing traffic (e.g., priority, path, or query-based routing).  
- **Target Groups**: Collections of resources (like EC2 instances, IP addresses, Lambda functions, or Kubernetes pods) that receive traffic based on the rules.  

For Kubernetes workloads, VPC Lattice uses the **AWS Gateway Controller** to direct traffic to services and pods efficiently.  

---

## How Amazon VPC Lattice Works  

1. **Enable Your VPC**: Add your VPC to a service network.  
2. **Service Discovery**: Clients in the VPC automatically find services using DNS.  
3. **Route Traffic**: All inter-application traffic flows through VPC Lattice.  
4. **Control Access**: Use IAM-based policies for traffic authorization and management.  

---

## Benefits of Amazon VPC Lattice  
- **Simplified Networking**: No need to manage complex network configurations.  
- **Cross-VPC Communication**: Easy connectivity between services across VPCs and accounts.  
- **Secure**: Apply consistent security policies using IAM.  
- **Versatile**: Supports various compute types, including containers, virtual machines, and serverless. 

By leveraging Amazon VPC Lattice, you can streamline application networking while maintaining high security and performance across your AWS environment.  

![image](https://github.com/user-attachments/assets/f142983d-7709-40e3-9c47-0d1ec7ffb52d)


Relationship between VPC Lattice and Kubernetes
As a Kubernetes user, you can have a very Kubernetes-native experience using the VPC Lattice APIs. The following figure illustrates how VPC Lattice objects connect to Kubernetes Gateway API  objects:
![image](https://github.com/user-attachments/assets/1704eed8-61b2-4552-aab2-cb8d7029e779)



## Amazon VPC Lattice architecture patterns and best practices

https://d1.awsstatic.com/events/Summits/reinvent2023/NET326_Amazon-VPC-Lattice-architecture-patterns-and-best-practices.pdf
![image](https://github.com/user-attachments/assets/e6a80bfb-d49a-4bf0-a323-00c3752a084e)




