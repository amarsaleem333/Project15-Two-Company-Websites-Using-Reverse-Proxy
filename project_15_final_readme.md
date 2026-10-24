# AWS Cloud Solution For 2 Company Websites Using A Reverse Proxy Technology (Project 15)

> **⚠️ WARNING:** This infrastructure setup is **NOT** covered by the AWS Free Tier. Ensure you **DELETE ALL** created resources immediately after finishing the project to avoid significant monthly costs. It is strongly recommended to set up an AWS Budget with spend notifications.

## Architecture Overview & Infrastructure Summary

This project implements a secure, scalable, and highly available multi-tier web architecture on AWS to host a primary WordPress business website (`project15.online`) and an internal Tooling platform (`tooling.project15.online`) for DevOps operations.

* **AWS Region & Availability Zones:** Deployed within **`us-east-2` (US East - Ohio)** across 3 Availability Zones (`us-east-2a`, `us-east-2b`, `us-east-2c`) to ensure multi-AZ fault tolerance.

* **Traffic & Request Flow:**

  1. Incoming DNS requests are resolved by **Amazon Route 53**.

  2. Public HTTPS (443) traffic hits an **Internet-Facing Application Load Balancer (External ALB)** where initial SSL/TLS is terminated.

  3. The External ALB forwards requests to an **Auto Scaling Nginx Reverse-Proxy Tier** running in public subnets.

  4. Nginx inspects the Host header and proxies requests via HTTPS to an **Internal Application Load Balancer**.

  5. The Internal ALB distributes traffic to private **WordPress** or **Tooling Web Servers** backed by **Amazon EFS** and an encrypted **RDS MySQL 8.x Database**.

* **Internal ALB Topology:** The production environment utilizes a single Internal ALB with host-based listener rules (routing `project15.online` and `tooling.project15.online` to their respective target groups) to optimize cost while maintaining distinct backend target groups.

* **End-to-End Security & TLS Strategy:** TLS is terminated at the External ALB using AWS Certificate Manager (ACM). Traffic between the External ALB, Nginx, Internal ALB, and Web Servers is re-encrypted in transit to enforce complete end-to-end encryption within the VPC.

* **Why Retain Nginx alongside ALB Host-Routing?** While AWS ALBs support native host-based routing, the Nginx reverse-proxy tier is retained to provide advanced request manipulation (e.g., custom rate limiting, header sanitisation, and edge caching) independent of cloud-provider lock-in.

## Architecture Diagram

## Table of Contents

 1. [Prerequisites & AWS Account Setup](#1-prerequisites--aws-account-setup)

 2. [Virtual Private Cloud (VPC) Setup](#2-virtual-private-cloud-vpc-setup)

 3. [Security Groups & Least-Privilege Access](#3-security-groups--least-privilege-access)

 4. [ACM TLS Certificate (Apex & Wildcard)](#4-acm-tls-certificate-apex--wildcard)

 5. [Compute Resources: Nginx Tier](#5-compute-resources-nginx-tier)

 6. [Compute Resources: Bastion Tier & Replacement Strategy](#6-compute-resources-bastion-tier--replacement-strategy)

 7. [Compute Resources: Webservers (WordPress & Tooling)](#7-compute-resources-webservers-wordpress--tooling)

 8. [Application Load Balancers (External & Internal)](#8-application-load-balancers-external--internal)

 9. [Nginx Reverse-Proxy Routing Configuration & Validation](#9-nginx-reverse-proxy-routing-configuration--validation)

10. [Data Layer (EFS & Encrypted RDS)](#10-data-layer-efs--encrypted-rds)

11. [DNS Configuration with Route 53 & End-to-End Verification](#11-dns-configuration-with-route-53--end-to-end-verification)

## 1. Prerequisites & AWS Account Setup

* **Master & Sub-Account Structure:**

  1. Created an **AWS Master (Root) Account**.

  2. Created an AWS Organization Unit (OU) named **`Dev`**.

  3. Created a sub-account named **`DevOps`** and moved it into the **`Dev`** OU.

  4. Logged into the `DevOps` sub-account to provision all project resources.

* **Domain Registration & Route 53 Hosted Zone:**

  1. Registered the custom domain `project15.online`.

  2. Created a Public Hosted Zone in AWS Route 53 and updated domain NS records accordingly.

* **Resource Tagging Standards:**

  * `Project`: `Project-15`

  * `Environment`: `dev`

  * `Automated`: `No`

## 2. Virtual Private Cloud (VPC) Setup

Configured a custom VPC in **`us-east-2`** according to the reference multi-tier architecture:

1. **VPC:** Created VPC with CIDR block `10.0.0.0/16`.

2. **Subnets:**

   * **Public Subnets (Nginx & Bastion):** `10.0.1.0/24` (`us-east-2a`), `10.0.3.0/24` (`us-east-2b`), `10.0.7.0/24` (`us-east-2c`)

   * **Private Web Subnets (WordPress & Tooling):** `10.0.2.0/24` (`us-east-2a`), `10.0.4.0/24` (`us-east-2b`), `10.0.8.0/24` (`us-east-2c`)

   * **Private Data Subnets (RDS & EFS):** `10.0.5.0/24` (`us-east-2a`), `10.0.6.0/24` (`us-east-2b`), `10.0.9.0/24` (`us-east-2c`)

3. **Internet Gateway:** Attached an Internet Gateway to the VPC.

4. **Elastic IPs & NAT Gateways:**

   * Allocated 3 Elastic IPs.

   * Configured NAT Gateway in the public subnet to grant outbound internet access to private subnets.

5. **Route Tables:**

   * **Public Route Table:** Associated with public subnets; default route `0.0.0.0/0` target set to Internet Gateway.

   * **Private Route Table:** Associated with private subnets; default route `0.0.0.0/0` target set to NAT Gateway.

## 3. Security Groups & Least-Privilege Access

* **External ALB SG (`external-alb-sg`):** Allows inbound `HTTP (80)` and `HTTPS (443)` from `0.0.0.0/0`.

* **Nginx SG (`nginx-sg`):** Allows inbound `HTTPS (443)` strictly from `external-alb-sg`.

* **Bastion SG (`bastion-sg`):** Allows inbound `SSH (22)` strictly from admin workstation public IP (`curl www.canhazip.com`).

* **Internal ALB SG (`internal-alb-sg`):** Allows inbound `HTTPS (443)` strictly from `nginx-sg`.

* **Web Servers SG (`webservers-sg`):** Allows inbound application traffic strictly from `internal-alb-sg`.

* **Data Layer SG (`datalayer-sg`):**

  * **RDS MySQL (Port 3306):** Inbound allowed strictly from `webservers-sg`.

  * **EFS Mount Target (Port 2049):** Inbound allowed from both `nginx-sg` and `webservers-sg`.

## 4. ACM TLS Certificate (Apex & Wildcard)

1. Navigated to AWS Certificate Manager (ACM) in `us-east-2`.

2. Requested a Public Certificate covering:

   * Apex domain: `project15.online`

   * Wildcard domain: `*.project15.online`

3. Validated ownership via CNAME records in Route 53 Public Hosted Zone.

### 📸 Evidence: ACM Certificate Details


*(Shows `project15.online` and `*.project15.online` active and issued in `us-east-2`)*

## 5. Compute Resources: Nginx Tier

1. **EC2 Provisioning:** Deployed CentOS/RHEL EC2 instances in public subnets across AZs.

2. **Software Dependencies Installed:**

   ```
   sudo yum update -y
   sudo yum install -y python ntp net-tools vim wget telnet epel-release htop nginx
   
   ```

3. **Golden AMI:** Created a base Nginx AMI from the provisioned instance.

4. **Launch Template:** Created Launch Template using the Nginx AMI, public subnet mapping, and `nginx-sg`.

5. **Target Group (`nginx-tg`):**

   * Target type: `Instances`

   * Protocol/Port: `HTTPS:443`

   * Health Check Path: `/healthstatus`

6. **Auto Scaling Group (`nginx-asg`):**

   * Spanning public subnets.

   * Attached to `nginx-tg`.

   * Capacity: Desired = 2, Min = 2, Max = 4.

   * Target Tracking Policy: Scale out if average CPU reaches 90%.

   * SNS notifications attached for ASG scaling events.

## 6. Compute Resources: Bastion Tier & Replacement Strategy

1. **Provisioning:** Deployed CentOS/RHEL EC2 instances per public subnet with Elastic IPs attached.

2. **Software Dependencies:**

   ```
   sudo yum update -y
   sudo yum install -y python ntp net-tools vim wget telnet epel-release htop ansible git
   
   ```

3. **Golden AMI & Launch Template:** Created Bastion AMI and configured Bastion Launch Templates.

4. **Target Group & ASG:** Target type `Instances`, TCP Port 22, Health checks active, Desired = 2, Min = 2, Max = 4.

5. **Architectural Note on EIP Association:** While manual static EIPs were attached during initial deployment, an ASG replacement strategy requires AWS Lambda or ASG Lifecycle Hooks invoking `aws ec2 associate-address` to dynamically reassign static EIPs when instances rotate.

## 7. Compute Resources: Webservers (WordPress & Tooling)

1. **Provisioning:** Deployed base EC2 instances in private web subnets for both WordPress and Tooling.

2. **Base Dependencies Installed:** `python`, `ntp`, `net-tools`, `vim`, `wget`, `telnet`, `epel-release`, `htop`, `php`.

3. **AMIs & Launch Templates:**

   * **WordPress Launch Template:** Configured with UserData script installing and bootstrapping WordPress.

   * **Tooling Launch Template:** Configured with UserData script installing Tooling code base.

4. **Target Groups:**

   * `wordpress-tg`: Protocol `HTTPS:443`, Health check path `/`

   * `tooling-tg`: Protocol `HTTPS:443`, Health check path `/`

5. **Auto Scaling Groups:** Individual ASGs configured for WordPress and Tooling in private web subnets (Desired: 2, Min: 2, Max: 4).

## 8. Application Load Balancers (External & Internal)

### External ALB (Internet-Facing)

* Deployed in Public Subnets across `us-east-2`.

* HTTPS Listener on Port 443 with ACM Certificate (`project15.online` / `*.project15.online`) attached.

* Default Action: Forward to `nginx-tg`.

### Internal ALB (Private)

* Deployed in Private Web Subnets across `us-east-2`.

* HTTPS Listener on Port 443.

* Listener Rules:

  * Host Header `project15.online` -> Forward to `wordpress-tg`

  * Host Header `tooling.project15.online` -> Forward to `tooling-tg`

### 📸 Evidence: ALB & Target Group Health

1. **External ALB Listener & ACM Certificate:**
   

2. **Nginx Target Group (Healthy Targets in External ALB):**
   

3. **Auto Scaling Group Target Spin Up:**
   

4. **Internal Target Groups (WordPress & Tooling Healthy Targets):**
   

## 9. Nginx Reverse-Proxy Routing Configuration & Validation

### Sanitized Active Nginx Configuration (`/etc/nginx/conf.d/project15.conf`)

```
# Upstream definition pointing to the Internal ALB
upstream internal_alb_backend {
    server internal-app-alb-123456789.us-east-2.elb.amazonaws.com:443;
    keepalive 32;
}

# Redirect HTTP to HTTPS
server {
    listen 80;
    server_name project15.online *.project15.online;
    return 301 https://$host$request_uri;
}

# WordPress Proxy Block
server {
    listen 443 ssl http2;
    server_name project15.online;

    ssl_certificate /etc/ssl/certs/project15.crt;
    ssl_certificate_key /etc/ssl/certs/project15.key;

    location /healthstatus {
        access_log off;
        return 200 'healthy\n';
    }

    location / {
        proxy_pass https://internal_alb_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_ssl_verify off;
    }
}

# Tooling Proxy Block
server {
    listen 443 ssl http2;
    server_name tooling.project15.online;

    ssl_certificate /etc/ssl/certs/project15.crt;
    ssl_certificate_key /etc/ssl/certs/project15.key;

    location /healthstatus {
        access_log off;
        return 200 'healthy\n';
    }

    location / {
        proxy_pass https://internal_alb_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_ssl_verify off;
    }
}

```

### Configuration Syntax Test (`nginx -t`)

```
$ sudo nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful

```

### Live Nginx Access Logs (`/var/log/nginx/access.log`)

```
10.0.1.12 - - [24/Oct/2026:14:10:05 +0000] "GET / HTTP/1.1" 200 4520 "-" "Mozilla/5.0" "project15.online"
10.0.3.44 - - [24/Oct/2026:14:10:12 +0000] "GET /login.php HTTP/1.1" 200 3180 "-" "Mozilla/5.0" "tooling.project15.online"

```

## 10. Data Layer (EFS & Encrypted RDS)

### Amazon Elastic File System (EFS)

1. Provisioned an EFS instance encrypted with AWS KMS.

2. Created Mount Targets in private data subnets (`10.0.5.0/24`, `10.0.6.0/24`, `10.0.9.0/24`) attached to `datalayer-sg`.

3. Created EFS Access Points for WordPress and Tooling file storage mounting.

### Amazon RDS MySQL

1. Created KMS key in AWS KMS for database encryption.

2. Created DB Subnet Group containing private data subnets.

3. Provisioned Multi-AZ MySQL 8.x DB Instance inside private data subnets.

4. Enforced strict network security: Public access set to **No**.

5. Enabled CloudWatch log exports (Error log, Slow query log) and automated backups.

## 11. DNS Configuration with Route 53 & End-to-End Verification

### Route 53 Records

* **`project15.online` (Apex):** Alias record pointing directly to External ALB DNS Name.

* **`tooling.project15.online` (Subdomain):** Alias record pointing directly to External ALB DNS Name.

### Terminal Curl Verification

```
# 1. WordPress Endpoint Test
$ curl -I -L https://project15.online

HTTP/1.1 200 OK
Server: nginx
Date: Sat, 24 Oct 2026 14:15:00 GMT
Content-Type: text/html; charset=UTF-8
Connection: keep-alive
X-Powered-By: PHP/8.1

# 2. Tooling Endpoint Test
$ curl -I -L https://tooling.project15.online

HTTP/1.1 200 OK
Server: nginx
Date: Sat, 24 Oct 2026 14:15:05 GMT
Content-Type: text/html; charset=UTF-8
Connection: keep-alive

```

### 📸 Live Application Screenshots (Valid HTTPS)

1. **WordPress Live Site (`https://project15.online`):**
   
   *(Displays live WordPress website operating under valid HTTPS)*

2. **Tooling Live Site (`https://tooling.project15.online`):**
   
   *(Displays live Tooling web application operating under valid HTTPS)*