<span>**1.1. AWS Account Setup and VPC Configuration**:Phase 1: Preparation & Network Architecture.</span>

- <span>**Create Sub-Accounts & OUs:** Set up an AWS Master account, create a sub-account named </span>`DevOps`<span>, create an Organizational Unit (OU) named </span>`Dev`<span>, and move the </span>`DevOps`<span> account inside it. Register your custom domain name via a domain registrar and set up a Public Hosted Zone in Amazon Route 53.</span>

- <span>**VPC & Subnet Setup:** Create a custom VPC (</span>`10.0.0.0/16`<span>) and provision public subnets (</span>`10.0.1.0/24`<span>, </span>`10.0.3.0/24`<span>), private web subnets (</span>`10.0.2.0/24`<span>, </span>`10.0.4.0/24`<span>), and data layer subnets (</span>`10.0.5.0/24`<span>, </span>`10.0.6.0/24`<span>) across two Availability Zones.</span>

- <span>**Gateways & Routing:** Create an Internet Gateway and attach it to your VPC. Allocate 3 Elastic IPs, assign one to a NAT Gateway placed in a public subnet, and configure separate Route Tables for public and private subnets.</span>

- **Verification:** Confirm that public subnets route internet-bound traffic through the Internet Gateway and private subnets route traffic through the NAT Gateway.

<span>**2.2. Define Security Group Rules**:Phase 2: Security Groups Setup.</span>

- <span>**Nginx Security Group:** Create a security group allowing inbound HTTP/HTTPS (ports 80 and 443) strictly from the external Application Load Balancer.</span>

- <span>**Bastion Security Group:** Restrict inbound SSH (port 22) access to your specific workstation public IP address.</span>

- <span>**Web Servers Security Group:** Configure rules to permit inbound traffic exclusively from the Nginx instances and the internal Application Load Balancer.</span>

- <span>**Data Layer Security Group:** Configure the EFS and RDS security groups so that database access is restricted solely to the web servers, while EFS mount targets accept connections from both Nginx and web servers.</span>

- **Verification:** Check the inbound rules for each security group in the EC2 Console to ensure least-privilege access is enforced.

<span>**3.3. Request and Validate ACM TLS Certificate**:Phase 3: Security & Certificates.</span>

- <span>**ACM Certificate Request:** Navigate to the AWS Certificate Manager (ACM), request a public wildcard certificate for your domain name (e.g., </span>`*.project15.online`<span>), and select DNS validation.</span>

- <span>**DNS Validation:** Use the **Create records in Route 53** feature inside ACM to automatically provision the required CNAME validation records.</span>

- <span>**Verification:** Confirm that the certificate status changes from pending validation to **Issued**.</span>

<span>**4.4. Provision Bastion Hosts and Elastic IPs**:Phase 4: Compute & Bastion Hosts.</span>

- <span>**Launch Bastion EC2 Instances:** Deploy CentOS/AlmaLinux EC2 instances for your Bastion hosts across your availability zones. Install necessary packages (</span>`python`<span>, </span>`ntp`<span>, </span>`net-tools`<span>, </span>`vim`<span>, </span>`wget`<span>, </span>`telnet`<span>, </span>`epel-release`<span>, </span>`htop`<span>, </span>`git`<span>, </span>`ansible`<span>).</span>

- <span>**Allocate Elastic IPs:** Assign a dedicated Elastic IP to each Bastion EC2 instance to maintain permanent external management access.</span>

- <span>**Verification:** Verify that each Elastic IP is successfully associated with its respective Bastion instance ID in the EC2 console.</span>

<span>**5.5. Build Golden Images and Launch Templates**:Phase 5: Golden Images & Launch Templates.</span>

- <span>**Configure Golden Images:** Spin up base instances for Nginx, WordPress, and Tooling web servers, install required dependencies (including PHP for web servers), and create custom AMIs out of them.</span>

- <span>**Create Launch Templates:** Build distinct launch templates for Nginx, Bastion, and Web servers referencing your custom AMIs, instance types (</span>`t2.micro`<span>), and user data installation scripts.</span>

- <span>**Verification:** Check the **AMIs** and **Launch Templates** consoles to verify that all configurations point to the correct image IDs and userdata scripts.</span>

<span>**6.6. Set Up Load Balancers and Auto Scaling Groups**:Phase 6: Load Balancing & Auto Scaling.</span>

- <span>**External ALB (Nginx):** Create an internet-facing Application Load Balancer listening on HTTPS port 443, attach your ACM certificate, and target your Nginx instances.</span>

- <span>**Internal ALB (Web Servers):** Create an internal Application Load Balancer within the private subnets to route traffic safely to the private WordPress and Tooling web servers.</span>

- <span>**Auto Scaling Groups (ASGs):** Deploy ASGs for Nginx, Bastion, and web servers with a desired capacity of 2, minimum of 2, and maximum of 4, configuring CPU utilization scaling policies (90%).</span>

- <span>**Verification:** Ensure all target groups show registered instances with a **Healthy** status.</span>

<span>**7.7. Configure Amazon EFS and Multi-AZ RDS MySQL**:Phase 7: Data Layer (EFS & RDS).</span>

- <span>**Amazon EFS Setup:** Create an EFS file system, provision mount targets across all availability zones in your data layer subnets, and create an EFS access point.</span>

- <span>**KMS Key & DB Subnet Group:** Create a customer-managed KMS key for database encryption and configure a database subnet group using your private data subnets.</span>

- <span>**RDS MySQL Deployment:** Deploy a Multi-AZ MySQL 8.x RDS database instance, attach your DB security group, ensure public accessibility is disabled, and enable encryption using your KMS key.</span>

- <span>**Verification:** Confirm that the EFS mount targets and the RDS database instance display an **Available** status.</span>

<span>**8.8. Finalize Route 53 DNS Routing**:Phase 8: DNS Configuration & Final Verification.</span>

- <span>**Create Alias Records:** Navigate to your Route 53 hosted zone and create **Alias** records for both the root domain (</span>`project15.online`<span>) and your tooling subdomain (</span>`tooling.project15.online`<span>) pointing directly to your external Nginx ALB DNS name.</span>

- <span>**Verification:** Run </span>`nslookup project15.online`<span> and </span>`nslookup tooling.project15.online`<span> from your terminal to verify proper DNS resolution to the external ALB.</span>



