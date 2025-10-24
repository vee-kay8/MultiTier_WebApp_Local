# 🏗️ Re-Architecting a Multi-Tier Web Application on AWS (Cloud-Native)

## 📘 Overview

This project demonstrates how a previously lift-and-shifted multi-tier web application was re-architected to fully leverage AWS cloud-native services.
The goal was to boost agility, scalability, and business continuity by transitioning from traditional virtual machine-based management to PaaS, SaaS, and managed services on AWS.

## 🎯 Objectives

Reduce operational overhead

Improve uptime and scaling

Eliminate manual provisioning and CapEx

Embrace Infrastructure as Code (IaC) and automation

Increase flexibility with PaaS/SaaS adoption


## 🧩 Architecture

### Legacy Architecture (Problem)
- Monolithic application running on manually managed EC2 instances.
- Components: Nginx Load Balancer, Tomcat VMs, MySQL on VM, Memcached on VM, RabbitMQ on VM.
- Challenges: Operational overhead, scaling difficulties, manual processes.

### 🏗️ High-Level Design


Cloud-Native re-architecture using AWS managed services.

Layers and Services:

| Layer                      | AWS Service                    | Description                                                              |
| -------------------------- | ------------------------------ | ------------------------------------------------------------------------ |
| **Application (Frontend)** | Elastic Beanstalk              | Hosts Tomcat-based web application with auto scaling and load balancing. |
| **Database (Backend)**     | Amazon RDS (MySQL)             | Managed relational database with automated backups and patching.         |
| **Cache Layer**            | Amazon ElastiCache (Memcached) | Improves response times and offloads DB reads.                           |
| **Message Queue**          | Amazon MQ (ActiveMQ)           | Provides reliable asynchronous messaging between services.               |
| **Storage**                | Amazon S3 / EFS                | Object and shared file storage for app assets and static content.        |
| **DNS + CDN**              | Route 53 + CloudFront          | Domain name resolution and global content delivery.                      |

### 🧾 Comparison: Lift-and-Shift vs Cloud-Native

| Component      | Lift & Shift (VM/EC2) | Cloud-Native AWS Service |
| -------------- | --------------------- | ------------------------ |
| App Hosting    | Tomcat on EC2         | Elastic Beanstalk        |
| Load Balancing | Nginx / ELB           | Built-in Beanstalk ELB   |
| Auto Scaling   | Manual                | Automated (Beanstalk)    |
| Storage        | NFS / Local Disk      | EFS / S3                 |
| Database       | MySQL on EC2          | RDS (MySQL)              |
| Caching        | Memcached on EC2      | ElastiCache (Memcached)  |
| Message Queue  | RabbitMQ on EC2       | Amazon MQ (ActiveMQ)     |
| DNS            | Local DNS             | Route 53                 |
| CDN            | None                  | CloudFront               |


## ⚙️ Flow of Execution
### Step 1: Base Infrastructure Setup
1.  **Create a Key Pair:** Create a key pair in the EC2 console for SSH access to Beanstalk instances if needed.
2.  **Create Security Groups (SGs):**
    *   `SG-Backend`: To be used by RDS, ElastiCache, and Amazon MQ. Initially, allow traffic on their respective ports (3306, 11211, 5671/61617) from your own IP for setup.
    *   `SG-ElasticBeanstalk`: Created automatically by Beanstalk.
    * Update the `SG-Backend` to allow the traffic from `SG-ElasticBeanstalk` after it has been created

### Step 2: Provision Backend Services
1.  **Create RDS Instance:**
    *   Engine: MySQL.
    *   Create Parameter group
    *   Create Subnet group
    *   Create DB Instances, ensure no public instance.
    *   Configure in a private subnet (recommended).
    *   Assign the `SG-Backend` security group.
    *   Note the endpoint, username, and password.
2.  **Create ElastiCache Cluster:**
    *   Engine: Memcached.
    *   Create Parameter group, use memcached 1.6
    *   Create Subnet group
    *   Create memcached, use 1.6 and default port 11211
    *   Assign the `SG-Backend` security group.
    *   Note the configuration endpoint.
3.  **Create Amazon MQ Broker:**
    *   Engine: RabbitMQ.
    *   Assign the `SG-Backend` security group.
    *   Ensure Private access
    *   Give Username & Password and take note of it.
4.  **DB Initialization:**
    *  Launch an instance, in the same VPC with the RDS.
    *  Connect to the instance through ssh
    *  Install git and mysql client `apt update && apt install mysql-client git -y`
    *  Allow the security group of this instance to be connected to the backend security group using mysql.
    *  Clone the source code `git clone https://github.com/vee-kay8/MultiTier_WebApp.git` 
    *  Get the endpoint of the RDS, the username and the password and login
    *  Deploy the schema: `mysql -h "endpoint" -u "username" -p "password" accounts < src/main/resources/db_backup.sql`
    *  log in to confirm: `mysql -h "endpoint" -u "username" -p "password" accounts`
    *  `show tables;` `exit`
    *  Delete Instance

### Step 3: Configure & Deploy the Application Tier
1.  **Create Elastic Beanstalk Environment:**
    *   Platform: Tomcat.
    *   Upload your initial application version (can be a placeholder).
    *   Beanstalk will automatically create an ELB and Auto-Scaling Group.
2.  **Update Security Groups:**
    *   Edit `SG-Backend` to allow inbound traffic from `SG-ElasticBeanstalk` on the necessary ports. This locks down the backend to only the app tier.
3.  **Initialize the Database:**
    *   Launch a temporary EC2 instance (a "jump box") in a public subnet with the `SG-Backend` SG.
    *   SSH into it and connect to the RDS endpoint using a MySQL client.
    *   Run your database schema creation scripts.
    *   **Terminate the instance after this step.**

### Step 4: Application Configuration & Deployment
1.  **Update Beanstalk Load Balancer:**
    *   In the Beanstalk console, add a listener to the ELB for port 443 (HTTPS) and attach your ACM certificate.
2.  **Update Application Health Check:**
    *   Change the health check path from `/` to a meaningful endpoint like `/login`.
3.  **Build and Deploy Final Artifact:**
    *   Rebuild your application WAR/JAR file, configuring it with the endpoints and credentials for RDS, ElastiCache, and Amazon MQ (use environment properties in Beanstalk for credentials!).
    *   Deploy this final artifact to your Beanstalk environment.

### Step 5: Global Delivery & DNS
1.  **Create a CloudFront Distribution:**
    *   Set the Beanstalk environment's URL as the origin.
    *   Use the same ACM certificate for custom SSL.
2.  **Configure Route 53:**
    *   Create a public hosted zone for your domain (e.g., `myapp.com`).
    *   Create an **A record** that aliases your domain (e.g., `www.myapp.com`) to the CloudFront distribution.

## Verification & Testing

1.  Access your application via the Route 53 URL (e.g., `https://www.myapp.com`).
2.  Test core functionalities: login, data retrieval, etc.
3.  Trigger the Auto-Scaling group by applying load (e.g., using a stress testing tool) to verify scaling policies.
4.  Check CloudWatch logs in the Beanstalk console for any errors.

## Cleanup

To avoid incurring charges, remember to delete all created resources:
*   Elastic Beanstalk Environment (will delete EC2 instances, ELB)
*   RDS Instance
*   ElastiCache Cluster
*   Amazon MQ Broker
*   CloudFront Distribution
*   Route 53 Hosted Zone


🧪 Testing & Validation

Verify health status in Elastic Beanstalk dashboard

Check DB connection logs

Confirm cache hits via ElastiCache metrics

Validate message flow in ActiveMQ console

Perform end-to-end functional testing through the app UI

🧹 Cleanup

To avoid incurring unnecessary charges:

Terminate the Elastic Beanstalk environment

Delete RDS, ElastiCache, and Amazon MQ instances

Delete CloudFront distribution

Remove Route 53 records

Remove unused key pairs and security groups

📸 Screenshots

Elastic Beanstalk environment dashboard

RDS instance configuration

Route 53 DNS setup

CloudFront distribution settings

🧠 Key Learnings

Migrating to AWS PaaS drastically reduces maintenance overhead.

Managed services improve uptime and scalability with minimal ops effort.

Infrastructure automation accelerates deployment and disaster recovery.

🔗 References

AWS Elastic Beanstalk Documentation

Amazon RDS Documentation

Amazon ElastiCache Documentation

Amazon MQ Documentation

AWS CloudFront Documentation

👨‍💻 Author

Voke Ogigbah
Cloud Engineer | DevOps Enthusiast | AWS Practitioner
🔗 LinkedIn
 | Medium
 | GitHub

database
9F90cHsunWDCtLzdMGZP
admin

amazonmq
voke
Favour888888