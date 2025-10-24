# Cloud-Native Re-architecting of a Multi-Tier Web Application on AWS

## Project Overview


This project demonstrates the re-architecting of a legacy "lift-and-shifted" web application into a fully cloud-native, scalable, and highly available system on AWS. We migrated from a manually managed infrastructure on EC2 instances to a PaaS/SaaS model using managed services to reduce operational overhead and improve agility.

## Architecture & Rationale

### Legacy Architecture (Problem)
- Monolithic application running on manually managed EC2 instances.
- Components: Nginx Load Balancer, Tomcat VMs, MySQL on VM, Memcached on VM, RabbitMQ on VM.
- Challenges: Operational overhead, scaling difficulties, manual processes.

### Target Cloud-Native Architecture (Solution)
![Architecture Diagram](link/to/your/architecture-diagram.png)

*Replace with a diagram from your AWS Architecture Icons toolkit.*

**Services Used:**
*   **Frontend & Application Tier:** AWS Elastic Beanstalk (for Tomcat, Auto-Scaling, and integrated ELB)
*   **Storage:** Amazon EFS (for shared file storage) / Amazon S3 (for static assets)
*   **Database Tier:** Amazon RDS (MySQL)
*   **Caching Tier:** Amazon ElastiCache (for Memcached)
*   **Message Queue:** Amazon MQ (for RabbitMQ compatibility)
*   **Content Delivery & DNS:** Amazon CloudFront, Route 53

## Prerequisites

1.  An AWS Account with appropriate permissions.
2.  AWS CLI configured on your machine.
3.  An existing Application WAR/JAR file (the artifact).
4.  An SSL Certificate provisioned in AWS Certificate Manager (ACM).

## Flow of Execution & Setup Instructions

### Step 1: Base Infrastructure Setup
1.  **Create a Key Pair:** Create a key pair in the EC2 console for SSH access to Beanstalk instances if needed.
2.  **Create Security Groups (SGs):**
    *   `SG-Backend`: To be used by RDS, ElastiCache, and Amazon MQ. Initially, allow traffic on their respective ports (3306, 11211, 5671/61617) from your own IP for setup.
    *   `SG-ElasticBeanstalk`: Created automatically by Beanstalk.

### Step 2: Provision Backend Services
1.  **Create RDS Instance:**
    *   Engine: MySQL.
    *   Configure in a private subnet (recommended).
    *   Assign the `SG-Backend` security group.
    *   Note the endpoint, username, and password.
2.  **Create ElastiCache Cluster:**
    *   Engine: Memcached.
    *   Assign the `SG-Backend` security group.
    *   Note the configuration endpoint.
3.  **Create Amazon MQ Broker:**
    *   Engine: RabbitMQ.
    *   Assign the `SG-Backend` security group.
    *   Note the connection URL and credentials.

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

## References

*   [AWS Elastic Beanstalk Documentation](https://docs.aws.amazon.com/elasticbeanstalk/)
*   [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)