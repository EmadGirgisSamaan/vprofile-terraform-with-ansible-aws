# vprofile: Terraform + Ansible on AWS

Provisions a VPC, ALB, NAT gateway and 4 EC2 instances (app01, db01, rmq01, mc01) with Terraform, then configures them with Ansible from app01.

## Requirements
- Terraform, AWS CLI (configured), an AWS account that allows VPC, Elastic IP and NAT Gateway

## Setup
1. Generate SSH keys (private keys are git-ignored):

   mkdir -p keys
   ssh-keygen -t ed25519 -f keys/admin -N ""
   ssh-keygen -t ed25519 -f keys/managed -N ""

2. Replace `203.0.113.10` in `main.tf` with your own public IP:

   curl -s https://checkip.amazonaws.com

3. Deploy:

   terraform init
   terraform plan -out=tfplan
   terraform apply tfplan
   terraform output website

## Cleanup
NAT Gateway and ALB are billed hourly. Always run:

   terraform destroy
