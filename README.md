# ABOUT

In this hands-on project, I designed and deployed a secure, scalable web application on AWS built entirely from scratch using :
VPC, public/private subnets, NAT Gateways, Application Load Balancer (ALB), and AWS Systems Manager (SSM) Session Manager for secure remote management.
There were no bastion hosts and no public SSH and it is 100% managed through AWS best practices.

This build is part of my 12-Week AWS Hands-On Challenge, where each week focuses on mastering core AWS services while implementing real-world, security-first infrastructure designs.

---

# Objectives

 Understand AWS core services: IAM, EC2, VPC, and ALB
 Build a secure, highly available architecture across two Availability Zones
 Replace traditional SSH with SSM Session Manager for secure, auditable access
 Deploy a static web app behind an Application Load Balancer
 Configure NAT Gateways and route tables for private outbound access
This goal is to architect for security, automation, and scalability.

---

# High-Level Architecture

“Secure Web App on AWS with ALB, NATGW, and SSM”
The architecture implements defense in depth where every component (from IAM users to EC2 instances) follows least privilege access and AWS security best practices.

![Architecture diagram](https://i.postimg.cc/wTfBMh9W/EC2-VPC-WEB-drawio-4.png)


---

# How the Architecture Works

Below is the breakdown of the traffic and security flow that powers the application:

### 1️⃣ Inbound Traffic

 Users access the app through the ALB DNS endpoint.
 The ALB, hosted in public subnets, receives HTTP/HTTPS requests.
 It forwards the requests to EC2 instances in private subnets, ensuring no direct internet exposure.

Result: The application is public, but the servers remain private.

---

### 2️⃣ Load Balancing Across AZs

 The ALB distributes requests across EC2 instances in multiple Availability Zones.
 If one zone fails, the application continues to serve traffic from the other, ensuring fault tolerance and high availability.

---

### 3️⃣ Outbound Traffic (Private → Internet)

 EC2 instances in private subnets access the internet via NAT Gateways (in public subnets).
 This setup allows outbound updates/downloads without exposing the instances to inbound internet traffic.

---

### 4️⃣ Secure Management with SSM (No Port 22)

 Instead of SSH, administrators connect via AWS Systems Manager Session Manager.
 Sessions are IAM-controlled, encrypted, and logged in CloudTrail.
 No public IPs, no open ports, no inbound exposure.

---

### 5️⃣ Identity and Access Control

 IAM users are protected with Multi-Factor Authentication (MFA).
 Permissions follow the Principle of Least Privilege (PoLP).
 EC2 instances use IAM roles (not embedded credentials) for SSM communication and resource access.

---

### 6️⃣ Network Segmentation & Routing

 Public Subnets → route through Internet Gateway (IGW)
 Private Subnets → route through NAT Gateway only
  This separation ensures total isolation between public-facing and private resources.

---

#  Step 1: Setting Up IAM and Security

### Create IAM User

 Go to IAM → Users → Add user
 Enable programmatic/console access
 Assign least-privilege permissions
 Enable MFA for both the IAM user and root account
 ![USERS](https://i.postimg.cc/ZKJ4fP3z/Screenshot-2025-10-21-120358.png)
![USERS](https://i.postimg.cc/bwzHLvqs/Screenshot-2025-10-21-120632.png)


### Create IAM Role for EC2 (SSM Access)

 Policy: `AmazonSSMManagedInstanceCore`
 Role name: `ec2_ssm_role`
  This enables your EC2 instances to communicate securely with Systems Manager.
![ROLES](https://i.postimg.cc/yN7LtPBm/Screenshot-2025-10-21-122116.png)
![ROLES](https://i.postimg.cc/kXrzwWVF/Screenshot-2025-10-21-122338.png)
![ROLES](https://i.postimg.cc/BQsNXpLT/Screenshot-2025-10-21-122540.png)
![ROLES](https://i.postimg.cc/028d8NhP/Screenshot-2025-10-21-122621.png)


---

# 🌐 Step 2: Build the VPC and Network Layer

### Create Custom VPC
Set up an Elastic IP
![Elastic IP](https://i.postimg.cc/Jndvw4NG/Screenshot-2025-10-21-141044.png)
 Name: `12WeekChallenge-Week-VPC`
 CIDR: `10.0.0.0/16`
 This defines the private address space for your application environment.
![VPC](https://i.postimg.cc/LXVd1XTb/Screenshot-2025-10-21-123955.png)
![VPC](https://i.postimg.cc/nzh23hZn/Screenshot-2025-10-21-123452.png)


### Attach Internet Gateway

 Path: VPC → Internet Gateways → Create → Attach to VPC
  This allows internet connectivity for your public subnets and NAT Gateways.
![IGW](https://i.postimg.cc/RFJkgdvz/Screenshot-2025-10-21-124652.png)
![IGW](https://i.postimg.cc/9XJkysdG/Screenshot-2025-10-21-125310.png)

NB: Without attaching the IGW, even public instances won’t have internet access.

---

# Step 4: Create NAT Gateways and Configure Route Tables

### NAT Gateways

 Create one NAT Gateway per AZ for high availability.
 Each NAT is deployed in a public subnet and attached to an Elastic IP (EIP).
![NGW](https://i.postimg.cc/T3pJ53Gw/NGW-AZ1.png)
![NGW](https://i.postimg.cc/fW8d7Q8b/NGW-AZ2.png)

### Route Tables

 Public Route Table: routes `0.0.0.0/0` → Internet Gateway
 Private Route Table: routes `0.0.0.0/0` → NAT Gateway
![RT](https://i.postimg.cc/T1NQBLfd/route-table.png)

Subnet Associations:
![Sub Ass](https://i.postimg.cc/xCtPbJkJ/RTN4.png)
 Public RT → Public Subnets
 Private RT → Private Subnets

Result:
 Public resources access the internet directly.
 Private resources access the internet indirectly (via NAT).


At this stage, your VPC should contain:

 2 Public Subnets
 2 Private Subnets
 1 Internet Gateway
 2 NAT Gateways
 Separate Route Tables for each subnet type

This forms the foundation of a secure AWS network architecture.

---

#  Step 5: Configure EC2 Instances (Private Subnets)

### Launch EC2 Instances

 AMI: Ubuntu 22.04 LTS
 Type: t3.micro
 Subnet: Private (no public IP)
 IAM Role: `ec2-ssm-role`
 Security Group: Allow HTTP (80) only from ALB
 ![ec2](https://i.postimg.cc/FRy0jf7R/Screenshot-2025-10-21-132459.png)
![ec2](https://i.postimg.cc/d1Yytq4Q/Screenshot-2025-10-21-132530.png)

### User Data Script

This script automates Apache installation and web page setup at launch:

```bash
#!/bin/bash
apt update -y
apt install -y apache2
systemctl enable apache2
systemctl start apache2
cat > /var/www/html/index.html <<'EOF'
<html>
  <head><title>Simple Web App</title></head>
  <body style="font-family: Arial; text-align: center; margin-top: 50px;">
    <h1>Project: Basic EC2 Web App</h1>
    <p>This project is part of the <strong>12 Week AWS Workshop Challenge</strong>.</p>
    <p>Instance ID: $(curl -s http://169.254.169.254/latest/meta-data/instance-id)</p>
    <p>Launched by: Samuel Nartey Otuafo | <a href="https://www.linkedin.com/in/samuelnartey/">Follow on LinkedIn</a></p>
    <p>Powered by AWS User Group Yaounde</p>
  </body>
</html>
EOF
```

---

# Step 6: Configure the Application Load Balancer (ALB)
From the EC2 window, Find Load balancer
![Alb](https://i.postimg.cc/13YzjyL7/Screenshot-2025-10-21-133608.png)

Select ALB
![Alb](https://i.postimg.cc/c155C2GH/Screenshot-2025-10-21-133646.png)
  
 Scheme: Internet-facing
 Listeners: HTTP (80)
 VPC: Your custom VPC
 Subnets: Both public subnets
 Security Group: Allow HTTP (80) inbound

Create a Target Group 
![Target Group](https://i.postimg.cc/fR6vhQXn/Screenshot-2025-10-21-135331.png)
(type: Instance. Protocol: HTTP) and register your EC2 instances.
The ALB now distributes traffic across both private instances.

Result:
 High availability across AZs
 Secure, centralized entry point
 Simplified scaling and monitoring

---

#  Step 7: Test and Validate

### Start an SSM Session

 Go to Systems Manager → Session Manager → Start Session
 Select your EC2 instance
 ![SSM](https://i.postimg.cc/3NtcjrRv/INSTANCE-CONNECT.jpg)
 
 Verify Apache service:

```bash
sudo systemctl status apache2
```
![Apache](https://i.postimg.cc/mDtjx3Zn/CTL.jpg)

### Test via Load Balancer

Copy your ALB DNS name (EC2 → Load Balancers → Description → DNS name) and open it in your browser:

```bash
curl http://<your-load-balancer-dns>
```

You should see your web app’s HTML output, confirming successful deployment! 

---

#  Key Takeaways

  Built a multi-AZ, fault-tolerant web app
  Used NAT Gateways for secure private subnet internet access
  Eliminated SSH by using SSM Session Manager
  Followed AWS security best practices and least privilege principles

---

# 💬 Final Thoughts

This project strengthened my understanding of network design, secure access control, and automation on AWS.
Every service, from IAM to ALB was built manually, ensuring deep practical understanding of how AWS infrastructure works together securely and efficiently.

If you’re also learning AWS, start small but go deep understanding of the “why” behind every component will make you a far stronger cloud engineer. 

---

Would you like me to make this **LinkedIn post-ready version** (with emojis, hashtags, and short formatting for public feed — not README)?
That version would be perfect to publish on your profile.
