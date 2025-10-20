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

### Phase 2: Application Deployment and Load Balancing
#### Step 4: Build and Deploy Artifact
Build the artifact on a computer using MAVEN, then push the artifact to S3 bucket using AWS CLI. Finally, the artifact is pulled from the s3 bucket using IAM roles and deployed in the Tomcat Server.
* Create S3 bucket.

* In IAM, create user, attach S3 full access policy, download and save the access keys and password.

* Create IAM roles: In IAM roles click create roles, under AWS service select EC2, for the permission select Amazon S3 full acess.

* Apply the role to the Tomcat instance.

* Using VS code/ go to src > main > resource > application.properties files, replace db01, mc01 and rmq01 with the mapnames from step 3 for in this projject I replaced db01 to db01.multitier.in

* check version of maven using `mvn -version` then build the artifact using `mvn install`, a new folder called target will be made.

* configure the AWS CLI using the saved Accesskeys and password

* Copy the artifact from the target folder to S3 bucket using aws `s3 cp target/vprofile-v2.war s3://bucketname/`

* Deploy the artifact to the Tomcat server, by connecting  the server through ssh, install aws cli using snap install aws-cli -- classic then copy articat from s3 bucket to a temp folder using aws   `s3 cp s3://bucketname /vprofile-v2 war /tmp/`
stop the tomcat service using systemctl stop tomcat10 . Remove and replace the ROOT with the artifact. `rm -r /var/lib/tomcat10/webapps/ROOT`
`cp /tmp/vprofile-v2.war /var/lib/tomcat/webapps/ROOT.war`
restart tomcat using `systemctl start tomcat10`.

#### Step 5 Load Balancer and DNS 
* Create a target group - use 8080 for HTTP because tomcat uses port 8080 rather than 80, in the advanced health check overide the port 80 to 8080 as well. add the tomcat instance to the target group and create it.

* Create the Load Balancer: select the application load balancer, select all availability zones, Use the Load Balancer security group initially created, add the target group just created, add https listener for secured connection and select the certificate created .

* Copy the DNS name of the load balancer, go to route 53, create a CNAME record in the domain hosted zone and map the load balancer DNS name to the domain name.

### Phase 3: Automation and Verification

#### Step 6: Autoscaling Group
* The first step of creating an Autoscaling group is to create an AMI from the instance you want to auto scale and in this project it is the tomcat instance. Select the instance, click create AMI, name it and click create.

* The next step is to create a Launch Template. Navigate to the Template section and click on create template, select the AMI just created, then choose the instance type, the key pair and the application security group created in step 1. Add the IAM role to the launch template.

* The final step Autoscaling is creating the Autoscaling group: click on create autoscale group, select the launch template just created, pick the availability zones. Attach it to the load balancer created in step 5 and the target group created as well. Turn on ELB healthcheck, select the desired amount of instance, the minimum and maximum instance as well. Select the metric used to scale out or in, In this project I used average CPU utilization. Add notification -SNS Topic, to get notification on the scaling.

#### Step 7: Validate

* Verify Auto Scaling Group
Confirm that the **Auto Scaling Group (ASG)** is actively managing and maintaining the desired number of application instances.

* Test Auto Scaling by stressing CPU

* Confirm load balancing across Tomcat nodes

* Access the Application
Use the assigned **domain name** to access the deployed application.

* Authenticate
Log in with the following credentials:
  - **Username:** `admin_vp`
  - **Password:** `admin_vp`

* Functional Validation
Navigate through the application and confirm that all features and services are functioning as expected.  
Ensure page responses, database connectivity, and load balancing are working properly.



### Cleanup Procedure

* Delete Auto Scaling Group
Remove the **Auto Scaling Group** to stop automatic management of instances.

* Terminate Backend Instances
Delete all **backend EC2 instances** created during deployment.

* Remove Load Balancer
Delete the **Load Balancer** associated with the application to release allocated resources.

* Clean Up DNS and Hosted Zones
  - Delete the **DNS records** associated with the domain.  
  - Remove the **Hosted Zones** from Route 53.

* Deregister AMI: 
Deregister the **Amazon Machine Image (AMI)** created for the deployment.

* Delete Snapshots
Permanently delete any **snapshots** associated with the AMI or instances to free up storage.


## 🧠 Lessons Learned

* Automation matters — EC2 User Data simplifies instance setup

* Scaling becomes effortless — AWS ASG removes manual VM management

* Security is layered — SGs, IAM, and SSL certificates protect every tier

* DNS management — Route 53 provides simple yet powerful control

## 🔮 Future Improvements

* Re-Architecting Web App on AWS Cloud [PAAS & SAAS]

* Implement CI/CD with AWS CodePipeline & CodeDeploy

* Containerize the architecture using ECS or EKS

* Add centralized monitoring via CloudWatch Dashboards

## 📚 References

* [AWS EC2 Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/concepts.html)

* [AWS S3 Documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html)

* [AWS Load Balancer Docs](https://docs.aws.amazon.com/autoscaling/ec2/userguide/autoscaling-load-balancer.html)

* [Route 53 Docs](https://docs.aws.amazon.com/route53/)

## 🧑🏽‍💻 Author

### Voke Ogigbah
Cloud Engineer | Systems & Security | AWS | Automation

[💼 LinkedIn](https://www.linkedin.com/in/voke-ogigbah-015b5871/)

[📝 Medium](https://medium.com/@vokeogigbah)

[💻 Portfolio](vokeogigbah.com)