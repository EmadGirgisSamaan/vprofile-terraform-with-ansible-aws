# 🚀 End-to-End Enterprise AWS Environment Provisioning

[![Terraform](https://img.shields.io/badge/Terraform-844FBA?style=for-the-badge&logo=terraform&logoColor=white)](https://www.terraform.io/)
[![Ansible](https://img.shields.io/badge/Ansible-EE0000?style=for-the-badge&logo=ansible&logoColor=white)](https://www.ansible.com/)
[![AWS](https://img.shields.io/badge/AWS-232F3E?style=for-the-badge&logo=amazon-aws&logoColor=white)](https://aws.amazon.com/)
[![Linux](https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black)](https://www.linux.org/)
[![Git](https://img.shields.io/badge/Git-F05032?style=for-the-badge&logo=git&logoColor=white)](https://git-scm.com/)

## 📖 Project Overview
This project demonstrates a comprehensive, production-ready approach to **Infrastructure as Code (IaC)** and **Configuration Management**. Developed as part of the Senior Academy DevOps Advanced Diploma, it automates the provisioning of a secure, multi-tier AWS environment using **Terraform**, followed by automated OS hardening and multi-service deployment using **Ansible**. 

The architecture is designed to eliminate manual server setup, enforce immutable configuration standards, prevent environment drift, and establish a repeatable, end-to-end deployment pipeline.

---

## 🏗️ Architecture Diagram

```mermaid
graph TD
    User((End User)) --> ALB["Application Load Balancer<br/>Port 80"]

    subgraph VPC ["AWS VPC 10.0.0.0/16"]
        direction TB
        
        subgraph Public ["Public Subnet 10.0.1.0/24"]
            ALB --> App["Application Server<br/>Nginx Reverse Proxy"]
        end

        subgraph Private ["Private Subnet 10.0.2.0/24"]
            App --> DB[("MariaDB Server<br/>Port 3306")]
            App --> MQ["RabbitMQ Server<br/>Port 5672"]
            App --> Cache[("Memcached Server<br/>Port 11211")]
        end

        IGW["Internet Gateway"] -.-> Public
        NAT["NAT Gateway"] -.-> Private
    end

    classDef public fill:#d4edda,stroke:#28a745,stroke-width:2px;
    classDef private fill:#f8d7da,stroke:#dc3545,stroke-width:2px;
    class App,ALB,IGW public;
    class DB,MQ,Cache,NAT private;
