# 🏗️ Lift and Shift: Migrating a Multi-Tier Application from Vagrant to AWS Cloud ☁️
## 📘 Overview

This project demonstrates how I lifted and shifted a locally hosted multi-tier web application (previously built using Vagrant and VirtualBox) to the AWS Cloud, using an Infrastructure-as-a-Service (IaaS) approach.

The goal was to modernize the setup, automate deployments, and simplify scaling by replacing local virtualization with AWS-managed services while keeping the same application architecture.

## 🚀 Scenario

In traditional on-prem or virtualized environments, multiple teams (sysadmins, virtualization, monitoring, operations) often manage workloads manually across physical or virtual machines.

This leads to:

* Complex infrastructure management

* Difficult scaling (up/down)

* High upfront CapEx and ongoing OpEx

* Time-consuming manual provisioning

* Limited automation

__Objective:__

This documentation details the process of migrating a legacy multi-tier application (previously hosted on Vagrant/VMs) to Amazon Web Services (AWS) using a "Lift and Shift" strategy. The goal is to establish a flexible, scalable, and cost-effective Infrastructure-as-a-Service (IaaS) foundation using modern Infrastructure-as-Code (IaC) principles.

## 🧩 Solution Overview

By leveraging AWS IaaS services, we replace manual infrastructure management with automated, scalable, and cost-effective solutions.
| Old Setup (Vagrant)    | AWS Cloud Equivalent            |
| ---------------------- | ------------------------------- |
| Local Virtual Machines | EC2 Instances                   |
| Nginx Load Balancer    | Elastic Load Balancer (ELB)     |
| Manual Scaling         | Auto Scaling Groups             |
| Local DNS              | Route 53 Private Hosted Zone    |
| Manual Configs         | Automated via User Data Scripts |
| Local Artifacts        | S3 Buckets for Build Storage    |

## 🧱 AWS Architecture

__Components:__

* EC2 Instances: Host each service (Tomcat, RabbitMQ,     MySQL, Memcached)

* S3: Store build artifacts for deployment

* ELB: Load balancing for Tomcat instances

* Route 53: DNS mapping for internal and public endpoints

* ACM: SSL certificate for HTTPS

* Auto Scaling: Scale Tomcat EC2 instances dynamically

__Architecture Diagram:__

   Users
     │
     ▼
  Route 53  ──►  AWS ELB (HTTPS)
                     │
         ┌───────────┴───────────┐
         │                       │
     Tomcat EC2 (App)       Tomcat EC2 (App)
         │                       │
      RabbitMQ EC2        Memcached EC2
             │
         MySQL EC2 (DB)

## ⚙️ Prerequisites

Before deploying, ensure you have:

* An active AWS Account

* IAM User with admin or EC2 privileges

* AWS CLI installed and configured

* SSH Key Pair for instance access

* Source code or build artifact (WAR file for Tomcat app)
## 🔧 Step-by-Step Setup Instructions
### Phase 1: Preparation and Base Infrastructure
#### Step 1: Create Security Groups and Key Pair
Create a Key Pair: Generate a secure key pair for SSH access to EC2 instances.

Create and Define 3 security groups:
* The first Securtiy Group is for the Load Balancer and it listens to the users for HTTP and HTTPS. (Inbound rule: allow HTTP and HTTPS from anywhere )

* The second Security Group is for the Tomcat Apache App instance and it listens to the Load Balancer Security Group. (Inbound rule: allow port 8080 connection from the Load balancer Security group, also allow ssh connection to Tomcat instance)

* The Third Security Group  is for the Backend services and it listens to the App security group (Inbound rule: allow Mysql service port 3306, Memcached port 11211, Rabbitmq port 5672 from the App SG. also allow ssh connections to the backend instances and allow connections between backend services)

#### Step 2: Launch Instances
Create 4 Instances, one for the Tomcat Apaches service and the other three for the backend services. The bash scripts used to  launch the instances are in the userdata folder in the Repo

* Create the first backend Instance for the mysql service, use Amazon Linux as the OS image, use the Backend Security group that was created and the mysql.sh script.

*  Create the second backend Instance for the memcached service, use Amazon Linux as the OS image, use the Backend Security group that was created and the memcache.sh script.

*  Create the third backend Instance for the rabbitmq service, use Amazon Linux as the OS image, use the Backend Security group that was created and the rabbitmq.sh script.

* Create the fourth Instance, this is for the Tomcat Apache Application, use the ubuntu 24 OS image, the Application security group and the tomcat_ubuntu.sh script.

* ssh into the various instances and check the system status of all the service  installed through the scripts. ensure they are all running.

#### Step 3: Update IP to name mapping in route 53
In this next we are going to use route 53 to create a Private DNS service to map the usernames ogf the services to the private address of their instances.
* Go to create hosted zones in route 53, name it whatever you want ( in this project i used multitier.in), select private hosted zone and click create hosted zone.

* Click create records, use A records, get the private address from the instances and map them to the name you give(For example, db01.multitier.in to 172.31.28.5,) 

* Repeat creating records for all the backend services. the tomcat service is optional because i am going to use a load balancer that is connected to it

### Phase 2 Application Deployment and Load Balancing
 



