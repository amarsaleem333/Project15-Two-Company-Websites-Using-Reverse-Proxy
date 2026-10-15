End-to-End AWS Multi-Tier Cloud Architecture

<img width="621" height="266" alt="image" src="https://github.com/user-attachments/assets/61411e20-bddc-451d-acf5-239d3f7747f8" />

# End-to-End AWS Multi-Tier Cloud Architecture & Reverse Proxy Solution

## Architecture Overview & Infrastructure Summary
This project implements a secure, scalable, and highly available multi-tier web architecture on AWS to host a primary WordPress business website and an internal Tooling platform for DevOps operations.
* **Architecture Scope:** Deployed across 2 Availability Zones (`us-east-1a` and `us-east-1b`) adhering to the multi-tier reference model.
* **Traffic Flow:** Client requests hit Route 53 DNS, route through an Internet-Facing Application Load Balancer (ALB) with TLS termination, pass through an Auto-Scaling Nginx Reverse-Proxy Tier, route through an Internal ALB, and land on private WordPress and Tooling Web Servers backed by Amazon EFS and a Multi-AZ RDS MySQL 8.x Database.
* **Why Retain Nginx alongside ALB Host-Routing?** While AWS ALBs support native host-based routing, the Nginx reverse-proxy tier is retained to provide advanced request manipulation (e.g., custom rate limiting, header sanitisation, and edge caching) independent of cloud-provider lock-in.

---

## Table of Contents
1. AWS Account Setup and VPC Configuration
2. Security Groups & Least-Privilege Access
3. ACM TLS Certificate (Apex & Wildcard)
4. Bastion Hosts & Replacement Strategy (Implemented vs. Recommended)
5. Golden Images & Launch Templates
6. Nginx Reverse-Proxy Routing Configuration & Request Flow
7. Load Balancing & Auto Scaling Groups
8. Data Layer (EFS & Encrypted RDS)
9. DNS Configuration & Final Verification

---

### 1. AWS Account Setup and VPC Configuration
* **Account & OUs:** Set up an AWS Master account, created a sub-account named DevOps, established an Organizational Unit (OU) named Dev, and placed the DevOps account inside it. Registered a custom domain (`project15.online`) and configured a Public Hosted Zone in Amazon Route 53.
* **VPC & Subnets:** Created a custom VPC (`10.0.0.0/16`) containing public subnets (`10.0.1.0/24`, `10.0.3.0/24`), private web subnets (`10.0.2.0/24`, `10.0.4.0/24`), and data layer subnets (`10.0.5.0/24`, `10.0.6.0/24`) across two Availability Zones.
* **Gateways & Routing:** Deployed an Internet Gateway attached to the VPC and configured a single NAT Gateway (balancing cost vs. resilience) with allocated Elastic IPs to manage outbound traffic for private subnets.

---

### 2. Security Groups & Least-Privilege Access
* **Nginx Security Group:** Allows inbound HTTP/HTTPS traffic strictly from the external Application Load Balancer.
* **Bastion Security Group:** Restricts inbound SSH (port 22) access exclusively to authorized administrative workstation IP addresses.
* **Web Servers Security Group:** Tightened inbound rules to accept application traffic **strictly from the Internal ALB Security Group**. *(Note: Direct access from Nginx is restricted as all traffic flows securely through the Internal ALB).*
* **Data Layer Security Group:** Enforces strict isolation where RDS MySQL is accessible solely from the web server tier, and EFS mount targets are restricted to Nginx and web servers.

---

## 3. ACM TLS Certificate (Apex & Wildcard)
- Certificate Scope: Correctly requested and validated an ACM public certificate covering both the root apex domain (project15.online) and the wildcard subdomain (*.project15.online) to ensure secure HTTPS coverage for all properties.
- Validation: Verified via automatic DNS validation records created in Route 53.
- 📸 **Evidence:** ![ACM Certificate Issued](docs/ACM%20cert%20Issued.png)
---

### 4. Bastion Hosts & Replacement Strategy
* **Implemented Architecture:** CentOS EC2 instances deployed in public subnets across 2 AZs with pre-installed administrative tooling (Ansible, Git, Python) and manually assigned static Elastic IPs.
* **Recommended Architecture (Future Improvement):** Combining Auto Scaling Groups with permanent Elastic IPs requires lifecycle automation (such as AWS Lambda or ASG lifecycle hooks invoking `aws ec2 associate-address`) to dynamically attach static EIPs when instances are replaced.

---

### 5. Golden Images & Launch Templates
* **AMIs:** Built custom golden AMIs for Nginx, WordPress, and Tooling servers with pre-configured dependencies (PHP, Nginx, system utilities).
* **Launch Templates:** Configured parameterized launch templates linking custom AMIs, `t2.micro` instance types, security groups, and automated UserData installation scripts.

---

## 6. Nginx Reverse-Proxy Routing Configuration & Request Flow
The core routing mechanism relies on Nginx distinguishing incoming requests by server_name and proxying traffic to the appropriate downstream internal service via proxy_pass. Upstream TLS verification is enabled (proxy_ssl_verify on) using trusted CA certificates to ensure secure end-to-end encryption within the VPC.

```nginx
upstream wordpress_backend {
    server internal-wordpress-alb-prod.us-east-1.elb.amazonaws.com:443;
    keepalive 32;}

upstream tooling_backend {
    server internal-tooling-alb-prod.us-east-1.elb.amazonaws.com:443;
    keepalive 32;}

server {
    listen 80;
    server_name project15.online *.project15.online;
    return 301 https://$host$request_uri;}

server {
    listen 443 ssl http2;
    server_name project15.online;

    ssl_certificate /etc/letsencrypt/live/project15.online/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/project15.online/privkey.pem;

    location / {
        proxy_pass https://wordpress_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        proxy_ssl_verify on;
        proxy_ssl_trusted_certificate /etc/ssl/certs/ca-bundle.crt;
    }

    location /healthstatus {
        access_log off;
        return 200 'healthy\n';
    }}

server {
    listen 443 ssl http2;
    server_name tooling.project15.online;

    ssl_certificate /etc/letsencrypt/live/project15.online/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/project15.online/privkey.pem;

    location / {
        proxy_pass https://tooling_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        proxy_ssl_verify on;
        proxy_ssl_trusted_certificate /etc/ssl/certs/ca-bundle.crt;
    }

    location /healthstatus {
        access_log off;
        return 200 'healthy\n';
    }}

Nginx Access Log Evidence
Expected Nginx Access Log entries showing successful 200 OK responses routing to your target domains:

10.0.1.15 - - [14/Oct/2026:09:00:00 +0000] "GET / HTTP/1.1" 200 4520 "-" "Mozilla/5.0..." -> project15.online
10.0.1.15 - - [14/Oct/2026:09:01:00 +0000] "GET / HTTP/1.1" 200 3180 "-" "Mozilla/5.0..." -> tooling.project15.online

---




### 7. Load Balancing & Auto Scaling Groups
- **External ALB:** Internet-facing load balancer listening on HTTPS (port 443) with the ACM certificate attached, forwarding traffic to the Nginx target group.
- **Internal ALB:** Private load balancer routing traffic securely from Nginx to the private auto-scaling web server pools.
- **Auto Scaling:** Configured ASGs for Nginx, Bastion, and Web servers with desired capacity set to 2, min 2, max 4, and a CPU utilization scaling policy threshold of 90%.  

- 📸 **Evidence:** 

[https://github.com/user-attachments/assets/1aa7b9ca-1da1-42f2-8f50-3cea64142532](https://github-production-user-asset-6210df.s3.amazonaws.com/134626440/674117163-1aa7b9ca-1da1-42f2-8f50-3cea64142532.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAVCODYLSA53PQK4ZA%2F20261015%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20261015T020354Z&X-Amz-Expires=300&X-Amz-Signature=ba895bef568418d60ddde7df36beaf212e396b18f12647db8824b3d8fd248a4e&X-Amz-SignedHeaders=host&response-content-type=image%2Fpng)

 
         

8. Data Layer (EFS & Encrypted RDS)
    -Amazon EFS: Provisioned an elastic file system with mount targets across both data layer subnets and configured access points for shared asset storage.

    -RDS MySQL Database: Deployed a Multi-AZ MySQL 8.x database instance within private data subnets, encrypted using a customer-managed AWS KMS key, with automated backups and CloudWatch log exports enabled.


9. DNS Configuration & Final Verification
      -Route 53 Aliases: Created Alias records for both the root domain (project15.online) and the tooling subdomain (tooling.project15.online) pointing directly to the external Nginx ALB DNS name.

      -Verification: Validated correct resolution and end-to-end TLS accessibility using nslookup and browser test runs confirming request flows (project15.online → Nginx → WordPress and tooling.project15.online → Nginx →         Tooling).




