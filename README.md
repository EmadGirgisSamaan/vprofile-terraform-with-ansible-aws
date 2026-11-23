# vprofile on AWS: Terraform + Ansible

Infrastructure-as-Code project that provisions a multi-tier AWS environment with **Terraform** and configures the whole application stack automatically with **Ansible**, in a single `terraform apply`.

> **Status:** Terraform configuration validated (`terraform validate`, `terraform plan`: 28 resources). Full deployment is pending a successful run on an AWS account without restrictive SCPs.

## Architecture

```
                    Internet
                       |
              +--------v--------+
              |       ALB       |  public subnets (us-east-1a / 1b)
              |  HTTP :80       |  allowed only from your IP
              +--------+--------+
                       | :8080
              +--------v--------+
              |  app01 (Tomcat) |  public subnet 10.0.1.0/24
              |  Ansible control|  SSH allowed only from your IP
              +----+---+---+----+
                   |   |   |
        :3306      |   |   |  :5672        :11211
   +---------------+   |   +---------------+
   |                   |                   |
+--v------+       +----v-----+       +-----v----+
|  db01   |       |  rmq01   |       |   mc01   |   private subnet
| MariaDB |       | RabbitMQ |       | Memcached|   10.0.2.0/24
+---------+       +----------+       +----------+
                       |
                 NAT Gateway (outbound only)
```

| Host | Role | Private IP | Subnet |
|---|---|---|---|
| app01 | Tomcat 9 + Ansible controller | 10.0.1.10 | Public |
| db01 | MariaDB | 10.0.2.40 | Private |
| rmq01 | RabbitMQ | 10.0.2.30 | Private |
| mc01 | Memcached | 10.0.2.20 | Private |

## What gets created

- 1 VPC, 3 subnets (2 public, 1 private), Internet Gateway, NAT Gateway, route tables
- 4 EC2 instances (Ubuntu 22.04, `t3.micro`, encrypted gp3 volumes, IMDSv2 required)
- 1 Application Load Balancer with a target group and `/login` health check
- Security groups following least privilege (backend ports reachable only from app01)
- 2 EC2 key pairs (`admin` for app01, `managed` for the backend servers)

## How it works

1. Terraform creates the network, servers and load balancer.
2. `user_data` installs Ansible (in a Python venv) and required collections on app01.
3. A `terraform_data` resource connects to app01 over SSH, uploads the playbook, inventory and the `managed` key, then runs `ansible-playbook` from app01.
4. Ansible configures all four servers over the private network:
   - Generates database and RabbitMQ passwords on the controller (never stored in the repo)
   - Installs and hardens MariaDB, RabbitMQ and Memcached (private listeners only)
   - Builds the application from a pinned Git revision with Maven and deploys it to Tomcat 9
   - Waits for the `/login` page to return HTTP 200

## Security highlights

- Private keys and Terraform state are git-ignored; only public keys are committed
- Backend servers have no public IPs and are reachable only through app01
- ALB and SSH access are restricted to a single `/32` admin IP
- Secrets are generated at deploy time and handled with `no_log`
- RabbitMQ `guest` user removed, MariaDB anonymous users and test DB removed
- EBS encryption enabled, IMDSv2 enforced

## Prerequisites

- [Terraform](https://developer.hashicorp.com/terraform/install) >= 1.7
- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) configured with credentials (`aws sts get-caller-identity` should succeed)
- An AWS account that allows creating a VPC, Elastic IP, NAT Gateway and ALB
- `ssh-keygen` (included with OpenSSH / Git Bash)

## Quick start

**1. Clone the repository**

```bash
git clone https://github.com/EmadGirgisSamaan/vprofile-terraform-with-ansible-aws.git
cd vprofile-terraform-with-ansible-aws
```

**2. Generate SSH keys** (private keys are git-ignored)

```bash
mkdir -p keys
ssh-keygen -t ed25519 -f keys/admin -N ""
ssh-keygen -t ed25519 -f keys/managed -N ""
```

**3. Set your public IP** in `main.tf` (replace the placeholder `203.0.113.10`)

```bash
MYIP=$(curl -s https://checkip.amazonaws.com)
sed -i "s#203.0.113.10/32#${MYIP}/32#g" main.tf
```

**4. Deploy**

```bash
terraform init
terraform plan -out=tfplan
terraform apply tfplan
```

**5. Open the application**

```bash
terraform output website
```

The first run takes roughly 15 to 25 minutes because Ansible builds the application with Maven.

## Outputs

| Output | Description |
|---|---|
| `website` | ALB URL of the application |
| `app_public_ip` | Public IP of app01 (SSH: `ssh -i keys/admin ubuntu@<ip>`) |
| `target_group_arn` | ARN of the Tomcat target group |

## Cleanup

The NAT Gateway, Elastic IP and ALB are billed hourly. Destroy everything when you are done:

```bash
terraform destroy
```

## Troubleshooting

| Problem | Cause / fix |
|---|---|
| `no file exists at ./keys/admin.pub` | Run step 2 to generate the keys |
| `explicit deny in a service control policy` | The AWS account belongs to an Organization that blocks VPC / EIP / key pair creation. Use an unrestricted account |
| Cannot reach the site or SSH | Your public IP changed. Update the `/32` in `main.tf` and run `terraform apply` |
| Maven build is slow or runs out of memory | Use `t3.small` for app01 |
| ALB target unhealthy | SSH to app01 and check `systemctl status tomcat9` |

## Project structure

```
.
├── main.tf          # AWS infrastructure and Ansible trigger
├── playbook.yml     # Configuration of all four servers
├── inventory.ini    # Ansible inventory (static private IPs)
├── ansible.cfg      # Ansible settings
├── keys/            # Public keys only (private keys are git-ignored)
└── README.md
```

## Notes

- The sample application (vprofile) comes from an external training repository, pinned to a specific commit in `playbook.yml`. This project covers the infrastructure and automation around it.
- This is a lab setup: single AZ for the backend, HTTP only, no autoscaling or backups.

## Possible improvements

- HTTPS on the ALB with ACM
- Remote Terraform state (S3 + DynamoDB locking)
- Variables for region, instance types and admin IP
- Multi-AZ backend and managed services (RDS, ElastiCache, Amazon MQ)
- CI with `terraform validate`, `tflint` and `trivy`
