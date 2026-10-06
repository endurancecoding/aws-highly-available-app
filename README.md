# AWS Highly Available Web Application

A highly available web application architecture built on AWS using **Amazon VPC, Application Load Balancer, Amazon EC2, Amazon RDS for MySQL, AWS Systems Manager, IAM, NAT Gateway, and AWS CloudFormation**.

The project demonstrates how to deploy a multi-tier web application across multiple Availability Zones while keeping the application servers and database private.

---

## Project Overview

The application uses a public **Application Load Balancer (ALB)** to distribute HTTP traffic across two private EC2 application servers located in separate Availability Zones.

The EC2 instances run Apache HTTP Server and do not have public IP addresses. They use a NAT Gateway for outbound Internet access when required.

The MySQL database runs on **Amazon RDS** inside dedicated private database subnets. Database access is restricted so that only the application servers can connect to it over TCP port 3306.

**AWS Systems Manager Session Manager** provides secure administrative access to the private EC2 instances without requiring SSH, public IP addresses, or inbound port 22.

---

## Architecture

![image alt](https://github.com/endurancecoding/aws-highly-available-app/blob/cff119363a616a301883ccfc18ad139eeaa97c2c/diagram/Highly-available-app-%20architectural-diagram.png)

### Architecture Flow

```text
                         Internet
                            |
                            v
                 +----------------------+
                 |  Application Load    |
                 |      Balancer        |
                 +----------+-----------+
                            |
                 +----------+----------+
                 |                     |
                 v                     v
        +----------------+    +----------------+
        |  Private EC2   |    |  Private EC2   |
        |  App Server 1  |    |  App Server 2  |
        |     AZ-1       |    |     AZ-2       |
        +--------+-------+    +--------+-------+
                 |                     |
                 +----------+----------+
                            |
                            v
                    +---------------+
                    | Private RDS   |
                    |    MySQL      |
                    +---------------+
```

---

## Network Architecture

The project uses a custom VPC with separate public, application, and database subnet tiers.

| Resource             | CIDR           | Purpose                |
| -------------------- | -------------- | ---------------------- |
| VPC                  | `10.0.0.0/16`  | Main network           |
| Public Subnet 1      | `10.0.1.0/24`  | ALB / NAT Gateway      |
| Public Subnet 2      | `10.0.2.0/24`  | ALB                    |
| Private App Subnet 1 | `10.0.11.0/24` | EC2 application server |
| Private App Subnet 2 | `10.0.12.0/24` | EC2 application server |
| Private DB Subnet 1  | `10.0.21.0/24` | RDS                    |
| Private DB Subnet 2  | `10.0.22.0/24` | RDS                    |

The application and database tiers are distributed across two Availability Zones.

---

## AWS Services Used

| AWS Service                   | Purpose                                                   |
| ----------------------------- | --------------------------------------------------------- |
| **Amazon VPC**                | Provides the isolated network environment                 |
| **Internet Gateway**          | Provides Internet connectivity for public resources       |
| **NAT Gateway**               | Allows private EC2 instances to make outbound connections |
| **Application Load Balancer** | Distributes HTTP traffic between application servers      |
| **Amazon EC2**                | Hosts the web application                                 |
| **Amazon RDS for MySQL**      | Provides the relational database                          |
| **AWS Systems Manager**       | Provides secure access to private EC2 instances           |
| **AWS IAM**                   | Controls permissions for EC2 and Systems Manager          |
| **AWS CloudFormation**        | Deploys the infrastructure as code                        |
| **Elastic IP**                | Provides a stable public IP for the NAT Gateway           |

---

## Security Architecture

The project uses security groups to enforce communication between application tiers.

### Internet → ALB

The Application Load Balancer accepts HTTP traffic on port `80` from the Internet.

```text
Internet
   |
   | HTTP :80
   v
ALB Security Group
```

### ALB → EC2

The application servers accept HTTP traffic only from the ALB security group.

The EC2 instances do **not** have public IP addresses.

```text
ALB Security Group
        |
        | HTTP :80
        v
Application Security Group
```

### EC2 → RDS

The database accepts MySQL traffic only from the application-server security group.

```text
Application Security Group
          |
          | MySQL :3306
          v
Database Security Group
```

The RDS database is configured with:

* `PubliclyAccessible: false`
* Private database subnets
* No direct Internet access
* MySQL access restricted to the application tier

---

## Systems Manager Access

The EC2 instances are private and do not expose SSH to the Internet.

Instead, an IAM role containing:

```text
AmazonSSMManagedInstanceCore
```

is attached to the EC2 instances through an IAM instance profile.

This allows administrators to connect to the private servers through **AWS Systems Manager Session Manager**.

No public IP or inbound SSH port is required.

### Verification

A Session Manager connection was successfully established to the private application server.

This confirmed that the EC2 instances could be securely administered while remaining private.

**Screenshot:**

![image alt](https://github.com/endurancecoding/aws-highly-available-app/blob/cff119363a616a301883ccfc18ad139eeaa97c2c/screenshots/ssm-private-servers.png)


---

## Application Servers

Two Amazon Linux 2023 EC2 instances are deployed into separate private application subnets.

The EC2 User Data automatically:

1. Updates the operating system.
2. Installs Apache HTTP Server.
3. Enables Apache at boot.
4. Starts Apache.
5. Retrieves the instance Availability Zone.
6. Retrieves the EC2 instance ID.
7. Creates a server-specific HTML page.

Each server identifies itself so that load balancing can be observed.

Example:

```text
Highly Available AWS Application

Server: APP-SERVER-1
Availability Zone: us-east-1a
Instance ID: i-xxxxxxxx
```
![image alt](https://github.com/endurancecoding/aws-highly-available-app/blob/cff119363a616a301883ccfc18ad139eeaa97c2c/screenshots/app-alb-server1-deployed-application.png)

The second server displays its own server identity.
![image alt](https://github.com/endurancecoding/aws-highly-available-app/blob/cff119363a616a301883ccfc18ad139eeaa97c2c/screenshots/app-alb-server2-deployed-application.png)

---

## Application Load Balancer

An internet-facing Application Load Balancer distributes HTTP requests across the two EC2 instances.

The target group uses:

```text
Protocol: HTTP
Port: 80
Health Check Path: /
```

Only healthy EC2 instances receive traffic.

### Target Health

Both application servers were verified as healthy targets.

**Screenshot:**
![image alt](https://github.com/endurancecoding/aws-highly-available-app/blob/cff119363a616a301883ccfc18ad139eeaa97c2c/screenshots/target-group.png)

---

## Database

The project uses **Amazon RDS for MySQL**.

The database is deployed into dedicated private database subnets and is not publicly accessible.

Database configuration includes:

```text
Engine: MySQL
Instance Class: db.t3.micro
Storage: 20 GB gp3
Publicly Accessible: No
Database Name: appdb
```

The database security group only permits MySQL traffic from the application-server security group.

**Screenshot:**
![image alt](https://github.com/endurancecoding/aws-highly-available-app/blob/cff119363a616a301883ccfc18ad139eeaa97c2c/screenshots/rds-database.png)

---

## Connectivity Verification

Connectivity from a private EC2 instance to the RDS database was tested over TCP port `3306`.

The test successfully established a connection to:

```text
ha-app-database.c47ecmmmwjg2.us-east-1.rds.amazonaws.com:3306
```

This verified the intended network and security-group relationship:

```text
Private EC2
     |
     | TCP 3306
     v
Private RDS MySQL
```

**Screenshot:**
![image alt](https://github.com/endurancecoding/aws-highly-available-app/blob/cff119363a616a301883ccfc18ad139eeaa97c2c/screenshots/ec2-rds-connectivity.png)

> The database password is intentionally not included in this repository.

---

## High Availability Verification

The ALB distributes traffic between the two application servers.

Each server displays its own identity, making it possible to observe which backend handled a request.

For example:

```text
Server: APP-SERVER-1
```
**screenshot**
![image alt](https://github.com/endurancecoding/aws-highly-available-app/blob/cff119363a616a301883ccfc18ad139eeaa97c2c/screenshots/app-alb-server1-deployed-application.png)

and:

```text
Server: APP-SERVER-2
```
![image alt](https://github.com/endurancecoding/aws-highly-available-app/blob/cff119363a616a301883ccfc18ad139eeaa97c2c/screenshots/app-alb-server2-deployed-application.png)

Refreshing the ALB endpoint can result in responses from either healthy backend.

**Screenshot:**
![image alt](https://github.com/endurancecoding/aws-highly-available-app/blob/cff119363a616a301883ccfc18ad139eeaa97c2c/screenshots/load-balancer-two-servers.png)

---

## Infrastructure as Code

The infrastructure is defined in:

```text
infrastructure/template.yaml
```

CloudFormation manages the infrastructure including:

* VPC
* Internet Gateway
* Subnets
* Route Tables
* NAT Gateway
* Security Groups
* EC2 Instances
* IAM Role
* IAM Instance Profile
* Application Load Balancer
* Target Group
* ALB Listener
* RDS Subnet Group
* RDS MySQL Database

The stack was deployed using:

```bash
aws cloudformation deploy \
  --template-file infrastructure/template.yaml \
  --stack-name aws-ha-app-network \
  --region us-east-1 \
  --capabilities CAPABILITY_NAMED_IAM
```

---

## Project Structure

```text
aws-highly-available-app/
│
├── infrastructure/
│   └── template.yaml
│
├── screenshots/
│   ├── 01-vpc-created.png
│   ├── 02-vpc-subnets.png
│   ├── 03-security-groups.png
│   ├── 04-ec2-application-servers.png
│   ├── 05-application-load-balancer.png
│   ├── 06-target-group-healthy.png
│   ├── 07-deployed-application.png
│   ├── 08-rds-database.png
│   ├── 09-ssm-private-server.png
│   ├── 10-ec2-to-rds-connectivity.png
│   └── 11-load-balancer-two-servers.png
│
├── diagram/
│   └── architecture.png
│
└── README.md
```

---

## Screenshots

### 1. VPC

Shows the deployed VPC and its network configuration.

![image alt](https://github.com/endurancecoding/aws-highly-available-app/blob/cff119363a616a301883ccfc18ad139eeaa97c2c/screenshots/vpc-created-dashboard.png)

### 2. Subnets

Shows the public, private application, and private database subnets.

![image alt](https://github.com/endurancecoding/aws-highly-available-app/blob/cff119363a616a301883ccfc18ad139eeaa97c2c/screenshots/all-vpc-subnets.png)

### 3. Security Groups

Shows the security groups controlling communication between the ALB, EC2, and RDS tiers.

![image alt](https://github.com/endurancecoding/aws-highly-available-app/blob/cff119363a616a301883ccfc18ad139eeaa97c2c/screenshots/security-groups.png)

### 4. EC2 Application Servers

Shows the two private EC2 application servers.

![image alt](https://github.com/endurancecoding/aws-highly-available-app/blob/cff119363a616a301883ccfc18ad139eeaa97c2c/screenshots/ec2-application-servers.png)

### 5. Application Load Balancer

Shows the deployed internet-facing Application Load Balancer.

![image alt](https://github.com/endurancecoding/aws-highly-available-app/blob/cff119363a616a301883ccfc18ad139eeaa97c2c/screenshots/load-balancer.png)

### 6. Deployed Application

Shows the web application being served through the ALB.

![image alt](https://github.com/endurancecoding/aws-highly-available-app/blob/cff119363a616a301883ccfc18ad139eeaa97c2c/screenshots/app-alb-server2-deployed-application.png)

### 7. Systems Manager Session

Shows secure access to a private EC2 instance through Session Manager.

![image alt](https://github.com/endurancecoding/aws-highly-available-app/blob/cff119363a616a301883ccfc18ad139eeaa97c2c/screenshots/ssm-private-servers.png)

---

## What I Learned

This project strengthened my understanding of:

* Designing AWS VPC architectures across multiple Availability Zones
* Public vs. private subnet design
* Route tables and routing
* NAT Gateway functionality
* Application Load Balancer configuration
* Target groups and health checks
* EC2 security groups
* Security-group-to-security-group communication
* Private RDS deployments
* Systems Manager Session Manager
* IAM roles and instance profiles
* Infrastructure as Code with CloudFormation
* Multi-tier AWS application architecture
* Verifying connectivity between application and database tiers

---

## Future Improvements

Possible improvements include:

* HTTPS using AWS Certificate Manager
* Custom domain using Route 53
* EC2 Auto Scaling Group
* Multi-AZ RDS deployment
* AWS Secrets Manager for database credentials
* CloudWatch monitoring and alarms
* CI/CD deployment with GitHub Actions
* Docker containerization
* Blue/green or rolling deployments

---

## Cleanup

This project contains billable AWS resources, particularly the **NAT Gateway and Amazon RDS**.

After completing testing and capturing the required screenshots, the CloudFormation stack can be deleted:

```bash
aws cloudformation delete-stack \
  --stack-name aws-ha-app-network \
  --region us-east-1
```

Check the CloudFormation stack status afterward to confirm that the resources have been removed.

---

## Conclusion

This project demonstrates a production-inspired AWS architecture where the public entry point is separated from the application and database layers.

The final architecture provides:

* Public access through an Application Load Balancer
* Two application servers across Availability Zones
* Private EC2 instances
* Private RDS database
* Security-group-based tier isolation
* Secure administrative access through Systems Manager
* Infrastructure managed through CloudFormation

The project combines **networking, compute, load balancing, databases, IAM, security, Systems Manager, and Infrastructure as Code** into one end-to-end AWS deployment.
