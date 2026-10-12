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



## 📑 Table of Contents
- [About](#-about)
- [How It Works](#-how-it-works)
- [Real-World Use Cases](#-real-world-use-cases)
- [Features](#-features)
- [Installation](#-installation)
- [Quick Start](#-quick-start)
- [Usage](#-usage)
- [Commands](#-commands)
- [Flags & Options](#-flags--options)
- [Configuration File](#-configuration-file)
- [Examples](#-examples)
- [Shell Completion](#-shell-completion)
- [Contributing](#-contributing)
- [License](#-license)

---



## 🧾 About
> **Tagline** — One sentence describing the tool.

2–3 sentences: what this CLI does, what pain it eliminates, and who should install it.

Example:
> `cli-tool-name` is a zero-config code scaffolding tool for TypeScript monorepos. Instead of copying boilerplate between projects, you run one command and get a fully configured workspace — with linting, testing, and CI pre-wired — in under 10 seconds.

---



## ⚙️ How It Works
Describe what happens when a command is run:

```
$ cli-tool-name run
        │
        ▼
  Load config (.cli-tool.json or defaults)
        │
        ▼
  Validate inputs (args + flags)
        │
        ▼
  Execute pipeline (steps run in order)
        │
        ▼
  Write output / print result to stdout
```

---



## 💼 Real-World Use Cases
- **Backend Developers:** Scaffold a new microservice with one command instead of copying config from an existing service.
- **DevOps Engineers:** Automate repetitive environment setup scripts across multiple machines without maintaining bash scripts per OS.
- **Open Source Maintainers:** Run cross-repo tasks (version bumps, changelog generation) from a single familiar command.
- **Teams with Standards:** Enforce consistent project structure by distributing a shared config file as an npm package.

---



## ✨ Features
- ⚡ Blazing fast execution
- 🔧 Zero-config defaults with full customization
- 🌈 Rich terminal output with color support
- 📄 JSON and plain-text output modes
- 🔌 Plugin-friendly architecture
- 🔤 Shell completion for bash, zsh, and fish

---



## 📦 Installation
```bash
# npm
npm install -g cli-tool-name

# Homebrew
brew install username/tap/cli-tool-name

# Binary (all platforms)
curl -sSL https://install.example.com | sh
```

**Requirements:** Node.js ≥ 18

---



## 🚀 Quick Start
```bash
# 1. Initialize config in current project
cli-tool-name init

# 2. Run the primary action
cli-tool-name run

# 3. Check version
cli-tool-name --version
```

---



## 💡 Usage



### Basic workflow
```bash
# Initialize once per project
cli-tool-name init

# Run against a target directory
cli-tool-name run ./src

# List available actions
cli-tool-name list

# Preview changes without applying them
cli-tool-name run --dry-run
```



### Piping and scripting
```bash
# Output as JSON for scripting
cli-tool-name list --output json | jq '.targets[]'

# Use in a Makefile or CI step
cli-tool-name build --config .cli-tool.prod.json
```

---



## 📚 Commands
| Command                      | Description                        |
| ---------------------------- | ---------------------------------- |
| `cli-tool-name init`         | Initialize config in current dir   |
| `cli-tool-name run [target]` | Execute the main workflow          |
| `cli-tool-name list`         | List available targets             |
| `cli-tool-name config`       | View or edit configuration         |
| `cli-tool-name upgrade`      | Upgrade to the latest version      |

---



## 🏳️ Flags & Options
| Flag         | Alias | Type      | Description                       | Default          |
| ------------ | ----- | --------- | --------------------------------- | ---------------- |
| `--output`   | `-o`  | `string`  | Output format (`json` / `text`)  | `text`           |
| `--verbose`  | `-v`  | `boolean` | Enable verbose logging            | `false`          |
| `--dry-run`  | `-d`  | `boolean` | Preview without executing         | `false`          |
| `--config`   | `-c`  | `string`  | Path to config file               | `.cli-tool.json` |

---



## ⚙️ Configuration File
Create `.cli-tool.json` at your project root:

```json
{
  "$schema": "https://cli-tool-name.dev/schema.json",
  "output": "json",
  "targets": ["src", "lib"],
  "ignore": ["node_modules", "dist"],
  "plugins": ["@cli-tool/plugin-example"]
}
```

---



## 💡 Examples
```bash
# Run verbosely on a specific path
cli-tool-name run ./src --verbose

# Dry-run a destructive operation
cli-tool-name clean --dry-run

# Output JSON and pipe to jq
cli-tool-name list --output json | jq '.targets'
```

---



## 🔤 Shell Completion
```bash
# Bash
cli-tool-name completion bash >> ~/.bashrc

# Zsh
cli-tool-name completion zsh >> ~/.zshrc

# Fish
cli-tool-name completion fish > ~/.config/fish/completions/cli-tool-name.fish
```

---



## 🤝 Contributing
```bash
git clone https://github.com/username/cli-tool.git
cd cli-tool
npm install
npm run dev -- --help
```

Please read [CONTRIBUTING.md](CONTRIBUTING.md) before opening a PR.

---



## 📝 License
Distributed under the MIT License. See [LICENSE](LICENSE) for more information.