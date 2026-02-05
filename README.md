# Multi-Tier Secure Web Infrastructure on AWS
# Project Overview
This project demonstrates a highly available, secure, and scalable web architecture deployed on AWS. By leveraging an Application Load Balancer (ALB) and AWS Web Application Firewall (WAF), the infrastructure ensures that incoming traffic is both distributed efficiently across multiple EC2 nodes and filtered against malicious intent.

# Key Features:
1. **Layer 7 Protection**: Integrated AWS WAF to mitigate SQL Injection (SQLi) and implement Geo-blocking.

2. **High Availability**: Traffic distribution across multiple EC2 instances using a Target Group.

3. **Network Isolation**: Tiered Security Group strategy to ensure EC2 instances are not directly exposed to the public internet.

4. **Automated Deployment**: Bootstrapped web server configuration using Bash scripting (User Data).

# Architecture
The traffic flow follows a structured path to ensure maximum security:

- **User Request**: Hits the ALB Entry point.

- **Protection Layer (WAF)**: Inspects headers and payloads "(Blocks non-India IP ranges & SQLi patterns)".

- **Orchestration Layer (ALB)**: Performs health checks and routes traffic to healthy instances.

- **Compute Layer (EC2)**: Apache servers process requests within a restricted security perimeter.

# Tech Stack
- **Cloud**: **AWS** (EC2, VPC, ALB, WAF)

- **Web Server**: Apache (HTTPD) on Amazon Linux 2023

- **Security**: AWS Managed Rules, Custom Geo-matching Rules

- **Scripting**: Bash (Automation)

# Deployment Steps
1. **Security Configuration**
Two distinct Security Groups were created to enforce traffic isolation:

- "**ALB-SG**: Allows HTTP (80) and HTTPS (443) from `0.0.0.0/0`.

- **Web-Server-SG**: Restricts HTTP (80) access only to traffic originating from the ALB-SG, and SSH (22) for administrative access from a specific Admin IP.

2. **Compute Provisioning**
- Launched two EC2 instances (**Web-Server-Alpha** & **Web-Server-Beta**) using the following specifications:

- AMI: Amazon Linux 2023

- Instance Type: t2.micro

- User Data: Automated the installation of Apache and created a dynamic index.html that displays the specific Instance ID and Availability Zone for load balancing verification.

3. **Load Balancing & Health Checks**
- Created a Target Group (**WebApp-Target-Group**) using the HTTP protocol on port 80.

- Deployed an Application Load Balancer (**WebApp-ALB**) to distribute traffic across the target group.

4. "WAF Implementation"
- **Deployed a Web ACL (WebApp-WAF-ACL)**: associated with the ALB including:

- **AWSManagedRulesSQLiRuleSet**: To protect against common database injection attacks.

- **Custom Geo-Block (Block-Non_India)**: A rule-based filter to restrict access to traffic originating from outside India.

# Troubleshooting: Resolving the 504 Gateway Timeout
During implementation, the ALB returned a **504 Gateway Timeout**, marking instances as `Unhealthy`.

* **The Issue:** I identified a **Security Group Mismatch**. The Web Server was configured to a stale SG ID, causing the ALB's health checks to be dropped.
* **The Resolution:** 1. Performed a **Bypass Test** by temporarily opening the EC2 to `0.0.0.0/0` to confirm Apache was running.
  2. Corrected the Inbound Rule to point exactly to the **ALB-SG ID**.
  3. Verified the Health Check Path matched the root directory (`/`).
* **The Lesson:** This reinforced the importance of **Security Group Nesting** and referencing IDs rather than IPs for a dynamic, secure "chain of trust."

# Testing & Validation
To verify the integrity of the setup, the following tests were performed:

- Load Balancing: Repeatedly refreshing the ALB DNS confirmed traffic toggling between Alpha and Beta instances.

- SQLi Protection: Attempted common SQL injection strings in the URL; AWS WAF successfully returned a 403 Forbidden error.

- Geo-blocking: Accessed the site via a VPN (simulating a non-India origin); the WAF successfully blocked the connection.

# User Data Script
The servers were bootstrapped using the following script to provide real-time metadata:

```bash
#!/bin/bash
# Update and install Apache
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd

# Fetch Instance Metadata (IMDSv2)
TOKEN=$(curl -X PUT "[http://169.254.169.254/latest/api/token](http://169.254.169.254/latest/api/token)" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
ID=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" -s [http://169.254.169.254/latest/meta-data/instance-id](http://169.254.169.254/latest/meta-data/instance-id))
ZONE=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" -s [http://169.254.169.254/latest/meta-data/placement/availability-zone](http://169.254.169.254/latest/meta-data/placement/availability-zone))

# landing page
cat <<EOF > /var/www/html/index.html
<!DOCTYPE html>
<html>
<body style="font-family: Arial; text-align: center; background-color: #f0f2f5;">
    <div style="margin-top: 100px; display: inline-block; background: white; padding: 30px; border-radius: 8px; border-top: 5px solid #2563eb;">
        <h2>Cloud Node Status: <span style="color: #16a34a;">Active</span></h2>
        <p>This request was handled by:</p>
        <code style="background: #eee; padding: 5px; font-size: 1.2em;">$ID ($ZONE)</code>
        <p style="color: #666; font-size: 0.9em; margin-top: 20px;">Secure Traffic via ALB + WAF</p>
    </div>
</body>
</html>
EOF
```
