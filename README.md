# Highly-Available-Web-Architecture-on-AWS-ALB-ASG-Bastion-NAT-
Project2

-----------------------

📌 Overview

This project demonstrates how to build a highly available, auto-scaled web application architecture on AWS using:

Application Load Balancer (ALB)

Auto Scaling Group (ASG)

Launch Template with User Data

Bastion Host for secure SSH access

Private/Public Subnet design

NAT Gateway for outbound internet access

Security Groups and NACL configuration

The architecture was built from scratch and includes real-world troubleshooting scenarios encountered during implementation.

-----------------------

🏗 Architecture

```bash
Internet
    ↓
Application Load Balancer (Public Subnets)
    ↓
Target Group(Logical Resource)
    ↓
Auto Scaling Group
    ↓
EC2 Web Servers (Private Subnets, Amazon Linux 2023)

Management Access:
Local PC → Bastion Host → ASG Instances

Outbound Access:
Private Subnet → NAT Gateway → Internet Gateway
```

🧱 VPC Design

2 Public Subnets (Multi-AZ)

2 Private Subnets (Multi-AZ)

Internet Gateway attached to VPC

NAT Gateway in Public Subnet

Route Tables properly associated

🔐 Security Design
Security Groups

SG-ALB

Inbound: HTTP (80) from 0.0.0.0/0

Outbound: All traffic

SG-WEB

Inbound: HTTP (80) from SG-ALB

Inbound: SSH (22) from SG-Bastion

Outbound: All traffic

SG-Bastion

Inbound: SSH (22) from my public IP

Outbound: All traffic

⚙️ Launch Template User Data
```bash
#!/bin/bash

yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "Web Server from $(hostname)" > /var/www/html/index.html
```
This script installs Apache automatically and starts the service during instance launch.

-----------------------

🧠 Major Troubleshooting & Lessons Learned

This project involved multiple real-world infrastructure issues. Below are the key challenges and how they were resolved.

❌ 1. SSH Key Issues (.ppk vs .pem)

Problem:
Could not SSH into instances using PuTTY and SCP.

Cause:
Different key pairs were used for Bastion and ASG instances.

Fix:

Used proper .pem file for each instance.

Configured SSH ProxyJump in Windows OpenSSH:

Host bastion
  HostName <bastion-public-ip>
  User ec2-user
  IdentityFile C:\path\Project1_Web.pem

Host asg
  HostName <asg-private-ip1>
  User ec2-user
  IdentityFile C:\path\Project2-LT.pem
  ProxyJump bastion

Host asg1
  HostName <asg-private-ip2>
  User ec2-user
  IdentityFile C:\path\Project2-LT.pem
  ProxyJump bastion

-----------------------

❌ 2. "Permission denied (publickey)"

Cause: Wrong key file used when connecting.

Fix: Ensure correct key pair matches the instance.

-----------------------

❌ 3. ALB 503 Error

Meaning:
No healthy targets in Target Group.

Cause:
ASG instances were not properly registered or unhealthy.

-----------------------

❌ 4. Target Group Unhealthy – Web Server Not Running

Symptom:

Health checks failed
curl: Failed to connect

Root Cause:
User Data failed during yum install httpd.

Why?
ASG instances were in private subnet without internet access.

-----------------------

❌ 5. NAT Gateway Timeout (Critical Issue)

Even after creating a NAT Gateway, curl -4 ifconfig.me returned no response.

Real Cause #1:

Route table still pointed to a deleted NAT Gateway.

0.0.0.0/0 → nat-xxxx (Deleted)

Fixed by updating route to the correct NAT:

0.0.0.0/0 → nat-0d7f6f70861561b4f

Real Cause #2:

ASG instances were mistakenly launched in a Public Subnet without public IP.

Even though IGW was attached, instances had no public IP → no outbound access.

Fix:

Modified ASG subnet configuration to use Private Subnets only

Terminated existing instances to force recreation

Confirmed instances launched in Private Subnet

-----------------------

✅ Final Result

ALB successfully routes traffic to ASG instances

Health checks pass

Page refresh shows different hostnames (Load balancing confirmed)

Bastion-based SSH access working

Private instances access internet via NAT

Infrastructure fully operational

-----------------------

📈 What This Project Demonstrates

VPC architecture design

Public vs Private subnet strategy

NAT Gateway design and troubleshooting

ALB + Target Group configuration

Auto Scaling Group behavior

Launch Template and User Data automation

Advanced debugging of AWS networking

Real-world infrastructure problem solving

-----------------------

🧠 Key Takeaways

Route Tables must point to the correct NAT (Deleted NAT causes silent failures)

Public subnet without public IP ≠ internet access

Instance Refresh does not move existing instances automatically

Always verify actual subnet association

Debugging AWS networking requires step-by-step traffic flow analysis

-----------------------

🚀 Future Improvements

HTTPS with ACM certificate

Auto Scaling policies based on CPU

Sticky sessions configuration

CloudWatch monitoring and alarms

CI/CD integration
