# End-to-End Enterprise AWS Environment Provisioning with Terraform & Ansible

![Terraform](https://img.shields.io/badge/IaC-Terraform-7B42BC?logo=terraform&logoColor=white)
![Ansible](https://img.shields.io/badge/Config%20Mgmt-Ansible-EE0000?logo=ansible&logoColor=white)
![AWS](https://img.shields.io/badge/Cloud-AWS-FF9900?logo=amazonaws&logoColor=white)
![Nginx](https://img.shields.io/badge/Web-Nginx-009639?logo=nginx&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-blue)

> **Terraform builds the infrastructure. Ansible deploys and configures the application.**
> A production-style, multi-tier AWS environment provisioned and configured entirely as code, with zero manual steps.

*Case Study Project 01 — DevOps Workshop Advanced Diploma, Senior Academy.*

---

## Table of Contents

1. [Overview](#1-overview)
2. [Architecture](#2-architecture)
3. [Tech Stack](#3-tech-stack)
4. [Repository Structure](#4-repository-structure)
5. [Prerequisites](#5-prerequisites)
6. [Infrastructure Layer (Terraform)](#6-infrastructure-layer-terraform)
7. [Configuration Layer (Ansible)](#7-configuration-layer-ansible)
8. [Deployment Workflow](#8-deployment-workflow)
9. [Validation & Acceptance Criteria](#9-validation--acceptance-criteria)
10. [Security Design](#10-security-design)
11. [Idempotency & Repeatability](#11-idempotency--repeatability)
12. [Troubleshooting](#12-troubleshooting)
13. [Future Improvements](#13-future-improvements)
14. [Cleanup](#14-cleanup)
15. [Author](#15-author)

---

## 1. Overview

This project combines **Infrastructure as Code (IaC)** with **automated Configuration Management** to build a complete AWS environment for hosting a scalable, multi-tier web application.

- **Terraform** (HCL) provisions the network topology (VPC, subnets, IGW, route tables, security groups), four Linux EC2 instances, and an Application Load Balancer.
- **Ansible** playbooks then configure the operating systems, harden them, and deploy Nginx, the application, MariaDB, RabbitMQ, and Memcached.

### Goals

- Eliminate manual server setup and prevent environment drift.
- Enforce clear separation between **provisioning** and **configuration**.
- Keep all backend services isolated in a private subnet.
- Make the whole environment reproducible from version-controlled code.

---

## 2. Architecture

```
                         Internet
                            │
                     HTTP :80 / HTTPS :443
                            │
                  ┌─────────▼──────────┐
                  │ Internet Gateway   │
                  └─────────┬──────────┘
 ┌──────────────────────────▼────────────────────────────────────┐
 │ VPC  10.0.0.0/16                                              │
 │                                                               │
 │  ┌─────────────────────────────────────────────────────────┐  │
 │  │ Public Subnet  10.0.1.0/24                              │  │
 │  │   ┌──────────────────────┐    ┌──────────────────────┐  │  │
 │  │   │ Application Load     │───▶│ App Server (EC2)     │  │  │
 │  │   │ Balancer (ALB) :80   │    │ Nginx + Application  │  │  │
 │  │   └──────────────────────┘    └───┬──────┬───────┬───┘  │  │
 │  └───────────────────────────────────┼──────┼───────┼──────┘  │
 │                                      │3306  │5672   │11211    │
 │  ┌───────────────────────────────────▼──────▼───────▼──────┐  │
 │  │ Private Subnet  10.0.2.0/24                             │  │
 │  │   ┌────────────┐   ┌────────────┐   ┌────────────────┐  │  │
 │  │   │ MariaDB    │   │ RabbitMQ   │   │ Memcached      │  │  │
 │  │   │ (EC2)      │   │ (EC2)      │   │ (EC2)          │  │  │
 │  │   └────────────┘   └────────────┘   └────────────────┘  │  │
 │  └─────────────────────────────────────────────────────────┘  │
 └───────────────────────────────────────────────────────────────┘
```

### Request Flow

1. A user opens the **ALB DNS name** in a browser.
2. The ALB forwards HTTP traffic to the **Application Server** (Target Group).
3. **Nginx** acts as a reverse proxy and forwards requests to the application process.
4. The application talks to **MariaDB**, **RabbitMQ**, and **Memcached** over the private network only.

### Network Plan

| Component        | CIDR / Value   | Notes                                      |
|------------------|----------------|--------------------------------------------|
| VPC              | `10.0.0.0/16`  | Custom VPC                                 |
| Public Subnet    | `10.0.1.0/24`  | ALB + Application Server                   |
| Private Subnet   | `10.0.2.0/24`  | MariaDB, RabbitMQ, Memcached               |
| Internet Gateway | —              | Attached to the VPC                        |
| Outbound (private) | NAT Gateway or approved alternative | For package installs and updates |

### EC2 Instances

| # | Role           | Subnet  | Service Port | Managed By |
|---|----------------|---------|--------------|------------|
| 1 | Application    | Public  | 80 (Nginx)   | Ansible role `app` |
| 2 | MariaDB        | Private | 3306         | Ansible role `mariadb` |
| 3 | RabbitMQ       | Private | 5672         | Ansible role `rabbitmq` |
| 4 | Memcached      | Private | 11211        | Ansible role `memcached` |

---

## 3. Tech Stack

| Layer                  | Technology                                   |
|------------------------|----------------------------------------------|
| Infrastructure as Code | HashiCorp Terraform (HCL)                    |
| Configuration Mgmt     | Ansible (playbooks & roles)                  |
| Cloud                  | AWS — VPC, EC2, ALB, Security Groups         |
| Remote Access          | AWS Systems Manager (SSM), keyless execution |
| Web Server             | Nginx (reverse proxy)                        |
| Data / Messaging / Cache | MariaDB, RabbitMQ, Memcached               |
| OS                     | Linux (Ubuntu / RHEL)                        |
| Version Control        | Git / GitHub                                 |
| Scripting              | Bash                                         |

---

## 4. Repository Structure

> Suggested layout — adjust to match your implementation.

```
.
├── terraform/
│   ├── main.tf              # Provider and module wiring
│   ├── variables.tf
│   ├── outputs.tf           # Instance IPs, ALB DNS name
│   ├── vpc.tf               # VPC, subnets, IGW, route tables, NAT
│   ├── security_groups.tf   # Dedicated SG per instance
│   ├── ec2.tf               # 4 EC2 instances
│   ├── alb.tf               # ALB, listener, target group, health checks
│   └── terraform.tfvars.example
├── ansible/
│   ├── ansible.cfg
│   ├── inventory/
│   │   ├── hosts.ini        # Static inventory (app, mariadb, rabbitmq, memcached)
│   │   └── aws_ec2.yml      # Dynamic inventory (EC2 metadata & tags)
│   ├── group_vars/
│   ├── site.yml             # Main playbook
│   └── roles/
│       ├── common/          # OS updates, base config, hardening
│       ├── app/             # Application + Nginx reverse proxy
│       ├── mariadb/         # DB install, DB/user creation
│       ├── rabbitmq/        # Broker, user, vhost, permissions
│       └── memcached/       # Cache install and tuning
├── docs/
│   └── architecture.png
├── .gitignore
└── README.md
```

---

## 5. Prerequisites

### Knowledge

- Linux administration (Ubuntu / RHEL)
- Networking fundamentals (IP addressing, CIDR, subnets, routing)
- Basic AWS services (VPC, EC2, Security Groups)
- Shell scripting and Git workflows

### Tooling

| Tool          | Purpose                         |
|---------------|---------------------------------|
| Terraform     | Provision AWS resources         |
| Ansible       | Configure and deploy            |
| AWS CLI       | Credentials and verification    |
| Session Manager plugin | SSM access to instances |
| Git           | Version control                 |

### AWS Requirements

- An AWS account with permissions for VPC, EC2, ELB, IAM (instance profile for SSM).
- AWS credentials configured locally:

```bash
aws configure
aws sts get-caller-identity   # verify
```

---

## 6. Infrastructure Layer (Terraform)

Terraform provisions:

- **Network:** custom VPC, public/private subnets, Internet Gateway, public/private route tables, NAT Gateway (or approved alternative).
- **Compute:** four Linux EC2 instances, one per service.
- **Security Groups:** one dedicated group per instance.
- **Load Balancing:** ALB, HTTP listener on port 80, Target Group, health checks.

### Security Groups

| Security Group | Inbound Rule                          | Source                  |
|----------------|---------------------------------------|-------------------------|
| `app-sg`       | TCP 80                                | ALB security group      |
| `mariadb-sg`   | TCP 3306                              | `app-sg` only           |
| `rabbitmq-sg`  | TCP 5672 (management UI restricted)   | `app-sg` only           |
| `memcached-sg` | TCP 11211                             | `app-sg` only           |

### Usage

```bash
cd terraform
terraform init
terraform validate
terraform plan -out=tfplan
terraform apply tfplan
```

Useful outputs:

```bash
terraform output alb_dns_name
terraform output
```

---

## 7. Configuration Layer (Ansible)

Ansible configures all four instances using a modular role-based structure and a single orchestrating playbook (`site.yml`).

### Inventory Groups

| Group       | Hosts                |
|-------------|----------------------|
| `app`       | Application Server   |
| `mariadb`   | MariaDB Server       |
| `rabbitmq`  | RabbitMQ Server      |
| `memcached` | Memcached Server     |

Dynamic inventory is supported through AWS EC2 metadata and tags. Connectivity can use AWS SSM for secure, keyless access.

### Roles

| Role        | Responsibilities |
|-------------|------------------|
| `common`    | Refresh package repos, install base packages, apply security updates, enable services on boot, basic OS hardening |
| `app`       | Install Nginx and runtime, deploy source code, set environment variables, configure reverse proxy, enable and start services, verify HTTP |
| `mariadb`   | Install server, bind to private interface, create database and dedicated user, assign privileges, verify connectivity from the app |
| `rabbitmq`  | Install server, create app user and virtual host, set permissions, accept authorized connections, verify connectivity |
| `memcached` | Install, listen on the private interface only, set memory allocation, use TCP 11211, verify connectivity |

### Usage

```bash
cd ansible
ansible-inventory -i inventory/aws_ec2.yml --graph   # confirm discovery
ansible all -i inventory/aws_ec2.yml -m ping         # confirm connectivity
ansible-playbook -i inventory/aws_ec2.yml site.yml
```

Run a single role via tags (if tagged):

```bash
ansible-playbook -i inventory/aws_ec2.yml site.yml --tags mariadb
```

---

## 8. Deployment Workflow

### Phase 1 — Infrastructure Provisioning (Terraform)

1. Write the HCL configuration.
2. Define the VPC, subnets, IGW, route tables, and security groups.
3. Define the ALB and Target Group.
4. Define the four EC2 instances.
5. `terraform init`
6. `terraform plan`
7. `terraform apply`
8. Verify that all resources were created.

**Expected result:** four running EC2 instances and all required networking components.

### Phase 2 — Configuration & Deployment (Ansible)

1. Collect the provisioned instance information.
2. Generate or update the inventory.
3. Run the main playbook.
4. Apply OS updates and base hardening.
5. Install and configure MariaDB, RabbitMQ, Memcached.
6. Deploy the application and configure Nginx.
7. Enable and start all services.
8. Validate application connectivity to all backends.

**Expected result:** all instances configured automatically, application and dependencies deployed.

---

## 9. Validation & Acceptance Criteria

Use this checklist to sign off the project.

### A. Automated Infrastructure Provisioning

- [ ] One custom VPC
- [ ] One public subnet and one private subnet
- [ ] One Internet Gateway and required route tables
- [ ] Dedicated security groups
- [ ] Four Linux EC2 instances
- [ ] One ALB and one Target Group
- [ ] No manual configuration through the AWS Console

### B. Automated Service Deployment

- [ ] Main playbook configures all four instances
- [ ] Nginx, application, MariaDB, RabbitMQ, Memcached installed and running
- [ ] No manual software installation

### C. Application Accessibility

- [ ] Application opens via the ALB public DNS name
- [ ] Requests reach the Application Server
- [ ] Nginx forwards requests to the application

```bash
curl -I http://<ALB_DNS_NAME>
```

### D. Backend Service Connectivity

| Service   | Check                                           |
|-----------|-------------------------------------------------|
| MariaDB   | App connects; read and write operations succeed |
| RabbitMQ  | App connects; a test message is published and consumed |
| Memcached | App connects; values can be stored and retrieved |

Example manual checks from the Application Server:

```bash
# MariaDB
mysql -h <MARIADB_PRIVATE_IP> -u <app_user> -p -e "SHOW DATABASES;"

# RabbitMQ
nc -zv <RABBITMQ_PRIVATE_IP> 5672

# Memcached
echo "stats" | nc <MEMCACHED_PRIVATE_IP> 11211
```

### E. Security Validation

- [ ] MariaDB reachable only from authorized app instances
- [ ] RabbitMQ reachable only from authorized app instances
- [ ] Memcached reachable only from authorized app instances
- [ ] Backend instances have **no public IP**
- [ ] Security groups enforce the required restrictions

### F. Idempotency & Repeatability

- [ ] Re-running the playbook introduces no unnecessary changes
- [ ] `terraform plan` after apply shows no unexpected changes
- [ ] Everything is maintained in version-controlled code

---

## 10. Security Design

- **Network isolation:** databases, brokers, and caches live in a private subnet with no public IPs.
- **Least privilege:** backend security groups accept traffic only from the application security group, on a single port each.
- **Restricted administration:** administrative access is limited; SSM is used for keyless remote execution instead of open SSH.
- **Hardening:** baseline OS hardening is applied by the `common` role.
- **Secrets:** never commit credentials. Use Ansible Vault, SSM Parameter Store, or AWS Secrets Manager.

```bash
ansible-vault create group_vars/all/vault.yml
ansible-playbook -i inventory/aws_ec2.yml site.yml --ask-vault-pass
```

Recommended `.gitignore` entries:

```
*.tfstate
*.tfstate.*
.terraform/
tfplan
terraform.tfvars
*.pem
vault.yml
```

---

## 11. Idempotency & Repeatability

| Tool      | How it is verified                                         |
|-----------|------------------------------------------------------------|
| Terraform | `terraform plan` returns **No changes** after a successful apply |
| Ansible   | A second playbook run reports `changed=0`                  |

```bash
ansible-playbook -i inventory/aws_ec2.yml site.yml   # run twice, expect changed=0 on the second
```

Tip: store Terraform state remotely (S3 + DynamoDB locking) when working in a team.

---

## 12. Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| ALB returns 502/503 | Target unhealthy, Nginx or app not running | Check target group health and `systemctl status nginx <app>` |
| Private instances cannot install packages | Missing NAT Gateway or route | Verify private route table and NAT |
| Ansible cannot reach hosts | Wrong inventory, SSM or key setup | Run `ansible all -m ping`; check SSM agent and IAM instance profile |
| App cannot reach MariaDB | Security group or bind address | Confirm SG allows 3306 from `app-sg` and MariaDB binds to the private IP |
| RabbitMQ auth errors | Missing user, vhost, or permissions | Re-run the `rabbitmq` role and verify with `rabbitmqctl list_permissions` |
| Terraform shows drift | Manual console changes | Revert the change or import it into state |

---

## 13. Future Improvements

- Horizontal scaling: Auto Scaling Group behind the ALB (architecture already supports adding app instances).
- HTTPS on the ALB with ACM certificates.
- Remote Terraform state with locking.
- CI/CD pipeline (GitHub Actions) running `terraform plan` and `ansible-lint` on every PR.
- Monitoring and logging (CloudWatch, Prometheus/Grafana).
- Multi-AZ deployment and managed services (RDS, Amazon MQ, ElastiCache) for production.

---

## 14. Cleanup

Remove all resources to avoid AWS charges:

```bash
cd terraform
terraform destroy
```

---

## 15. Author

**Name:** Emad Gerges Samaan
**Role:** DevOps Engineer
**LinkedIn:** [_link_](https://www.linkedin.com/in/emad-gerges-samaan-gad/)

Built as part of the **Senior Academy — DevOps Workshop Advanced Diploma**.

