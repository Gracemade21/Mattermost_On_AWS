## Mattermost_On_AWS
This guide walks you through setting up a secure, self-hosted Mattermost chat application on AWS EC2. The architecture includes:  A public subnet hosting the Mattermost web application.  A private subnet hosting the MySQL database.  A NAT gateway to allow the private subnet to access the internet securely.
# Deploying Mattermost and MySQL on AWS EC2

This guide walks you through setting up a secure, self-hosted Mattermost chat application on AWS EC2. The architecture includes:

- A **public subnet** hosting the Mattermost web application.
- A **private subnet** hosting the MySQL database.
- A **NAT instance** to allow the private subnet to access the internet securely.

## 📌 Prerequisites

- An active AWS account.
- Basic knowledge of AWS services (VPC, EC2, Subnets, sql database, Security Groups).
- SSH access to EC2 instances.

## 🗺️ Architecture Overview
![image](https://github.com/user-attachments/assets/a4d139de-869c-4c2e-b66d-eab501617489)

The deployment consists of:

- A Virtual Private Cloud (VPC) with CIDR block `10.0.0.0/16`.
- A public subnet (`10.0.1.0/24`) for the Mattermost server and NAT instance.
- A private subnet (`10.0.2.0/24`) for the MySQL database.
- An Internet Gateway attached to the VPC.
- Route tables configured for public and private subnets.
- Security groups to control inbound and outbound traffic.

## 🛠️ Deployment Steps

### 1. Create a VPC

- Navigate to the VPC dashboard in AWS.
- Create a new VPC with CIDR block `10.0.0.0/16`.
- Enable DNS hostnames for the VPC.

### 2. Create Subnets

- **Public Subnet**:
  - CIDR block: `10.0.1.0/24`.
  - Enable auto-assign public IPv4 addresses.

- **Private Subnet**:
  - CIDR block: `10.0.2.0/24`.
  - Do not enable auto-assign public IPv4 addresses.

### 3. Set Up Internet Gateway

- Create an Internet Gateway.
- Attach it to your VPC.

### 4. Configure Route Tables

- **Public Route Table**:
  - Associate it with the public subnet.
  - Add a route: Destination `0.0.0.0/0`, Target: Internet Gateway.

- **Private Route Table**:
  - Associate it with the private subnet.
  - Add a route: Destination `0.0.0.0/0`, Target: NAT instance (to be created).

### 5. Launch EC2 Instances

- **NAT Instance**:
  - Launch in the public subnet.
  - Assign an Elastic IP.
  - Configure as a NAT instance.

- **MySQL Database Instance**:
  - Launch in the private subnet.
  - Use Ubuntu 20.04 LTS AMI.

- **Mattermost Application Instance**:
  - Launch in the public subnet.
  - Use Ubuntu 20.04 LTS AMI.

### 6. Install and Configure MySQL

- SSH into the MySQL instance:
  ```bash
  ssh -i MySQL.pem ubuntu@<private-IP>
