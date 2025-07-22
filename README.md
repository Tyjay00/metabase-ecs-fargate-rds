# Metabase Deployment on AWS ECS Fargate with RDS PostgreSQL
---

## 1. Introduction

This document provides an overview of the steps taken to successfully deploy Metabase, an open-source business intelligence tool, on Amazon Web Services (AWS) using Elastic Container Service (ECS) with the Fargate launch type.

---

## 2. Core Components Utilized

The deployment architecture involves several key AWS services:

* **Amazon Elastic Container Service (ECS) with Fargate**: A fully managed container orchestration service that allows running Docker containers without managing underlying servers. **Fargate** eliminates the need to provision, configure, and scale clusters of virtual machines.
* **Amazon Relational Database Service (RDS) for PostgreSQL**: A managed database service that simplifies the setup, operation, and scaling of a relational database. It provides high availability, automated backups, and patching. In this deployment, **RDS PostgreSQL** serves as the application database for Metabase itself (storing Metabase's configuration, users, dashboards, etc.).
* **Application Load Balancer (ALB)**: A Layer 7 load balancer that distributes incoming application traffic across multiple targets, such as ECS tasks. It enables advanced routing features and integrates seamlessly with ECS.
* **Amazon Virtual Private Cloud (VPC) & Subnets**: A logically isolated section of the AWS Cloud where AWS resources are launched. The deployment utilized the default **VPC** and its subnets for network isolation and resource placement.
* **Security Groups**: Act as virtual firewalls that control inbound and outbound traffic for instances and other resources within a VPC. They are crucial for implementing the principle of least privilege.

---

## 3. Deployment Steps Summary

The deployment process involved setting up each component and configuring their interactions, with a strong emphasis on network security.

### 3.1. Database Setup (Amazon RDS PostgreSQL)

* **RDS Instance Creation**: An RDS PostgreSQL instance (`metabase-db`) was provisioned within the chosen VPC and subnets.
* **Public Accessibility**: Crucially, the RDS instance was configured as **not publicly accessible** to prevent direct internet access to the database.
* **The default security group of the VPC** was identified and its rules were configured specifically for the RDS instance. Initially, it had no inbound rules allowing public access to port 5432. The necessary inbound rule was added later to allow traffic only from the Metabase ECS tasks.

<img width="1906" height="907" alt="1rds-instance" src="https://github.com/user-attachments/assets/ea365ddf-7594-42db-8a44-e68de8864492" />


### 3.2. Load Balancer Setup (Application Load Balancer)

* **ALB Creation (`metabase-lb`)**: An Application Load Balancer was created as an Internet-facing load balancer within the same VPC and selected subnets.
* **Default VPC Security Group Configuration for ALB**: The **default security group of the VPC** was associated with the ALB and its inbound rules were configured.
    * **Inbound Rules**: Configured to allow public **HTTP** traffic on **Port 80** (from `0.0.0.0/0`). An **HTTPS** listener on **Port 443** (from `0.0.0.0/0`) was also recommended for production-grade security, requiring an SSL/TLS certificate.
* **Listener and Target Group Configuration**:
    * An HTTP listener on **Port 80** was set up on the ALB.
    * A new target group (`metabase-tg`) was created. This target group was configured to forward traffic to **IP addresses** (suitable for Fargate) on **Port 3000** using HTTP protocol. This is because Metabase containers internally listen on port 3000.
    * Health checks were configured for the target group using the `/api/health` endpoint to ensure only healthy Metabase tasks receive traffic.

<img width="1886" height="906" alt="image" src="https://github.com/user-attachments/assets/347d0a39-aa5d-4278-9e50-bfa392994166" />


### 3.3. ECS Cluster and Task Definition

* **ECS Cluster Creation (`metabase-cluster`)**: An ECS cluster was created with the **Fargate** launch type compatibility.
* **Metabase Task Definition (`metabase-task-definition`)**: A task definition was created to define the Metabase container.
    * **Container Image**: The official `metabase/metabase:latest` Docker image was specified.
    * **Port Mapping**: A crucial port mapping was defined, exposing **Container port: 3000**. This is the port Metabase listens on inside the container.
    * **Environment Variables**: Environment variables (`MB_DB_TYPE`, `MB_DB_HOST`, `MB_DB_PORT`, `MB_DB_DBNAME`, `MB_DB_USER`, `MB_DB_PASS`) were configured to enable Metabase to connect to its RDS PostgreSQL application database.

<img width="1907" height="906" alt="meta-cluster-running" src="https://github.com/user-attachments/assets/bc32fee6-e052-43bc-bc4a-18c93559a74a" />

---

<img width="1892" height="912" alt="3task-port3000" src="https://github.com/user-attachments/assets/fbef2ceb-d048-4079-8b69-78b3d32e7221" />


### 3.4. ECS Service Creation

* **ECS Service Creation (`metabase-service`)**: An ECS service was created within the `metabase-cluster`, using the `metabase-task-definition`.
* **Networking Configuration**: The service was deployed into the same VPC and subnets as the ALB and RDS.
* **Dedicated ECS Task Security Group (`metabase-ecs-tasks-sg`)**: A new security group was created for the ECS tasks. This security group was initially configured in the service creation wizard and then refined post-creation.
* **Load Balancer Integration**: The ECS service was explicitly configured to use the existing `metabase-alb` and its `metabase-tg` target group. This ensures that the ALB automatically registers and deregisters Metabase tasks as they start and stop.

<img width="1895" height="912" alt="5meta-task-running" src="https://github.com/user-attachments/assets/10860062-0850-4f56-bcb0-06dffc40c56b" />

---

## 4. Security Group Configuration Details

The security group configurations are critical for ensuring secure and functional communication between components. In this deployment, the **default security group of the VPC** was leveraged and its rules were modified to facilitate the required traffic flow.

* **Default Security Group (Associated with ALB, ECS Tasks, and RDS)**:
    * **Inbound Rule for ALB Traffic**:
        * Type: HTTP (TCP 80), Source: `0.0.0.0/0` (Allows public web access)
        * Type: HTTPS (TCP 443), Source: `0.0.0.0/0` (Recommended for secure public access)
    * **Inbound Rule for ECS Task Traffic (from ALB)**:
        * Type: Custom TCP (TCP 3000), Source: *Self-referencing the default security group ID*. This rule ensures that the Application Load Balancer (also in the default SG) can send traffic to the Metabase containers on port 3000.
    * **Inbound Rule for RDS Traffic (from ECS Tasks)**:
        * Type: PostgreSQL (TCP 5432), Source: *Self-referencing the default security group ID*. This is the only inbound rule needed for the RDS instance, ensuring only the Metabase ECS tasks (also in the default SG) can connect to the database. Crucially, no public access (`0.0.0.0/0`) was allowed on this port.
    * **Outbound Rule**: The default security group typically allows all outbound traffic. In this configuration, this general outbound rule was maintained to enable the ALB to forward requests to ECS tasks, and for ECS tasks to initiate connections to RDS and external services.
---

<img width="1892" height="908" alt="6sg" src="https://github.com/user-attachments/assets/db9986ac-bb8b-4bfd-85b3-c007cf0d1846" />


## 5. Outcome and Next Steps

Upon successful deployment, the "Welcome to Metabase" setup screen was accessible via the Application Load Balancer's DNS name. This confirmed that:

* The Metabase container was running.
* The Load Balancer was correctly routing traffic to the Metabase task.
* Metabase successfully connected to and initialized its application database on the RDS PostgreSQL instance.

<img width="1881" height="952" alt="metabase-login" src="https://github.com/user-attachments/assets/fb32a5f6-d0fa-4609-bc3f-a5174050f35e" />

---

<img width="1887" height="962" alt="meta-login" src="https://github.com/user-attachments/assets/64980249-9d03-4321-b912-0727ddb644c0" />

---

Next steps for the task involve:

* Implementing HTTPS on the ALB for secure access to Metabase.
* Creating dedicated security groups
  
