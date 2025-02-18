---
title: VAWS Application Migration Service
dates: 2025-02-18
categories: [AWS, Migration]
tags: [AWS, Migration]
---

# Understanding AWS Application Migration Service: Architecture and Network Setup

Migrating applications to the cloud can be a complex and daunting task, especially when dealing with diverse source environments, compatibility issues, and the need for minimal downtime. AWS Application Migration Service (AWS MGN) simplifies this process by offering a flexible, reliable, and highly automated lift-and-shift solution. In this blog post, we’ll dive into the architecture and network setup of AWS Application Migration Service, exploring how it streamlines the migration of physical, virtual, or cloud servers to AWS.

---

## What is AWS Application Migration Service?

AWS Application Migration Service is designed to help businesses migrate their applications and databases to AWS with minimal disruption. It supports a wide range of operating systems, including common Windows and Linux distributions, and can handle mission-critical applications like SAP, Oracle, SQL Server, and even custom-built applications.

The service uses block-level replication to continuously synchronize data from your source servers to AWS, ensuring that your applications remain up-to-date during the migration process. This approach eliminates compatibility issues, reduces performance disruptions, and shortens cutover windows, making it an ideal solution for large-scale migrations.

---

## AWS Application Migration Service Architecture

Let’s break down the architecture of AWS Application Migration Service to understand how it works:

### 1. Source Environment
The source environment consists of the servers you want to migrate. These can be physical, virtual, or cloud-based servers. In the example below, the source environment includes two servers:
- The top server has two disks attached.
- The bottom server has three disks attached.

### 2. AWS Region
On the right side of the architecture is the AWS Region where the servers will be migrated. Subnets are pre-defined in this region to facilitate the migration process.

### 3. AWS Replication Agent
To begin the migration, you’ll need to install the AWS Replication Agent on your source servers. This lightweight agent performs the following tasks:
- Authenticates with the Application Migration Service API endpoint using TLS 1.3 encryption.
- Registers the source server with the service.
- Automatically provisions resources in the staging area subnet.

The agent can be installed without requiring a reboot, ensuring minimal disruption to your operations.

### 4. Staging Area Subnet
The staging area subnet is a critical component of the migration process. It uses low-cost compute and storage resources to keep your data in sync with AWS. Key resources in the staging area subnet include:
- **Replication Servers**: Lightweight Amazon EC2 instances that handle data replication.
- **Staging Volumes**: Low-cost Amazon EBS volumes that store replicated data.
- **Amazon EBS Snapshots**: Incremental snapshots of the replicated data.

For every disk in your source environment, AWS Application Migration Service creates an identically sized EBS volume in the staging area subnet. In our example, the five source disks result in five EBS volumes attached to the replication servers.

### 5. Data Replication
Once the agent is installed and the staging area resources are provisioned, data replication begins. The data is encrypted (using AES-256 encryption) and transferred directly from the source servers to the replication servers. You can use private connectivity options like AWS Direct Connect or a VPN to secure the replication path.

The service automatically scales the staging area resources based on the number of concurrently replicating servers and disks, eliminating the need for manual maintenance.

---

## Key Features of AWS Application Migration Service

1. **Ephemeral Replication Servers**: Replication servers are temporary resources that are automatically rotated by the service. They use t3.small Linux instances by default, with T3 Unlimited pricing enabled.
2. **Efficient Data Handling**: One replication server can handle up to 15 concurrently replicating disks. Data is compressed and encrypted in transit and can also be encrypted at rest using Amazon EBS encryption.
3. **Continuous Replication**: After the initial sync, the service continuously replicates data changes from the source servers to the staging area.

---

## Launching Migrated Instances

Once replication is in progress, you can configure the launch settings, which define how and where your migrated instances will be launched. These settings include:
- Subnet and security group configurations.
- Instance type and volume type.
- Tags for resource management.

When the source server is marked as “Ready for Testing,” you can launch test or cutover instances. During the launch process, the service spins up a conversion server to make necessary changes to drivers, network settings, and operating system licenses. Once the conversion is complete, the instance is ready for use on AWS.

---

## Network Connectivity Requirements

To ensure a smooth migration, you’ll need to set up the following network connections:

1. **Source Servers to Application Migration Service API Endpoints**: Allow connectivity on Port 443 for authentication, configuration, and monitoring.
2. **Source Servers to Amazon S3**: Enable connectivity for the AWS Replication Agent to retrieve software components.
3. **Source Servers to Staging Area Subnet**: Allow connectivity on Port 1500 for continuous data replication.
4. **Staging Area Subnet to AWS Services**: Ensure replication and conversion servers can communicate with Application Migration Service endpoints and Amazon S3.

---

## Conclusion

AWS Application Migration Service is a powerful tool for simplifying and accelerating your cloud migration journey. By leveraging its automated, block-level replication capabilities, you can migrate applications and databases to AWS with minimal downtime and disruption. Whether you’re moving physical, virtual, or cloud-based servers, AWS Application Migration Service provides the flexibility and reliability you need to ensure a successful migration.

Ready to take the next step? Explore AWS Application Migration Service today and unlock the full potential of the cloud for your business.

---

Let me know if you’d like to add or modify anything in this blog post!
