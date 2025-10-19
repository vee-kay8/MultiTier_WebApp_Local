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

