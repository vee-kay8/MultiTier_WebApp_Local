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
1.  **Create IAM Roles for the Beanstalk:**
    *   Select AWS service and for service select EC2 Instances
    *   Attach the following Policies
        *   AdminstratorAccess-AWSElasticBeanstalk
        *   AWSElasticBeanstalkCustomPlatformforEC2Role
        *   AWSElasticBeanstalkRoleSNS
        *   AWSElasticBeanstalkWebTier
    *   Give it a name "`multitier-beanrole`" and save.
2.  **Create Amazon Elastic Beaqnstalk Application**<br>
    Click on create Application
    *   Configure Environment
        *   select Web server environment
        *   give application a name "`multitier-beanapp`"
        *   give Environment name "`multitier-beanapp-prod`"
        *   add a unique domain name "`multitier.us-east-1.elasticbeanstalk.com`"
        *   Select `Tomcat` as platform and `Tomcat 10 with Correto 21 running on 64bit Amazon Linux 2023` as platform branch
        *   select `Sample application` for now and for Presets select `Custom configuration`
    *   Configure Service access
        *   select `aws-elasticbeanstalk-service-role` as the service role.
        *   select the IAM role created "`multitier-beanrole`" as the EC2 instance profile
        *    select existing keypair
    *   Set up networking, database, and tags.
        *   select the default vpc
        *   tick the Public IP address Activated box
        *   select all the Availabiliy zones in the instance subnets
        *   don't select anything in the database part because we already created a databse.
        *   create the appropraiate tags
    *   Configure instance traffic and scaling
        *   change Root volume type to `General purpose3(SSD)`
        *   dont select anything in security group so the app creates it's own security group and we can edit it.
        *   select `Load balanced` in the auto scaling section
        *   select min and max instances based on your projec. In this project I selected min 2, and max 4 instances.
        *   instance type, I selected T2 micro
        *   for scaling trigger, I selected `NetworkOut`
        *   for process, edit and add stickiness
    *   Configure updates, monitoring and logging
        *   Select `Rolling` as the Deployment policy and `50%` Deployment batch size
    *   Review
        * Submit
3.  **Build & Deploy Artifact**
    *  In VS-code, navigate the project folder, src>main>application.properties and update the backend information
        *    replace db01 with the RDS end point that was noted previously also replace the username and password as well.
        *   replace mc01 with the Elasticache endpoint, the portnumber repains the same
        *   replace the rabbitmq address from rmq01 to the endpoint end point of the amazonMQ and change the port number to 5671, update the username and password too.
        *   crosscheck and make sure every detail is accurate, else the application wont work.
    *   Go to default terminal and run the following commands to build
        *   `mvn -version` to see the version of maven installed and java installed as well, it should be maven 3.9.9 and java 17.0.12
        *   `mvn install` to build the artifact in the local machine.
    *   Go to Beanstalk environment and click upoload, navigate to the .war file (artifact), select it, give it a version  and click upload. 
    *   After it is successful, click on the domain and it will take you to the login page but this page is not secure. to secure it we will need to:
        *   navigate to configuration then click on edit on "Instance traffic and scaling"
        *   go to listeners under load balancer and add listener
        *   select HTTPs protocol and port 443, and select the registered SSL certificate and save it
        *   Apply the changes.
    *   Copy the URL and go to Route53, add a new CNAME record and map it to the registered domain.
    *   Go to web browser and put in the mapped domain, observe that the connection is now secured
    *   log in and verify everything is working.

### Step 4: Global Delivery
1.  **Create a CloudFront Distribution:**
    <br> Navigate to cloudfront and click on create distribution
    *   Give the Disribution a name, select single website or app as Distribution type
    *   Specify Origin
        *   select Elastic Load balancer
        *   browse and select the loadbalancer Beanstalk created.
        *   Customise Origin setting
            *   Protocol = match viewer
    *   In this project i didnt enable the WAF but in production WAF shouuld be enabled
    *   deploy
    *   Add domain
        *   Give Domain name we are going to use to acccess the website.
        *   select the certificate
        *   add domain
    *   Select Distribution domain name, go to route 53, create a CNAME record and map the distribution domain name to the website domain name.
    *   Verify by going to the website and inspect the page 


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

## 🧠 Key Learnings

Migrating to AWS PaaS drastically reduces maintenance overhead.

Managed services improve uptime and scalability with minimal ops effort.

Infrastructure automation accelerates deployment and disaster recovery.

## 🔗 References

[AWS Elastic Beanstalk Documentation](https://docs.aws.amazon.com/elastic-beanstalk/)

[Amazon RDS Documentation](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Welcome.html)

[Amazon ElastiCache Documentation](https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/WhatIs.html)


[AWS CloudFront Documentation](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Introduction.html)

## 👨‍💻 Author

Voke Ogigbah
Cloud Engineer | DevOps Enthusiast | AWS Practitioner

[💼 LinkedIn](https://www.linkedin.com/in/voke-ogigbah-015b5871/)

[📝 Medium](https://medium.com/@vokeogigbah)

[💻 Portfolio](vokeogigbah.com)
