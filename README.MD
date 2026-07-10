# Terraform Module for Creating Cloudbees CI Operation Center and Client Controller in AWS EC2

# Overview

This Terraform module is designed to deploy Cloudbees CI Operation Center and Client Controller on AWS EC2 instances. It provides a flexible and customizable setup for running Cloudbees CI in a cloud environment.

# Features

- Deploys Cloudbees CI Operation Center on AWS EC2 instances.
- Deploys Cloudbees CI Client Controller on AWS EC2 Auto Scaling Group.
- Automatically generate ec2 key pair for accessing the EC2 instances.
- Automatically configures security groups.
- Automatically install wildcard license file (`license.key` and `license.cert`).
- Automatically configures VPC and subnet configurations.
- Automatically configure ALB for Operation Center and Client Controller.

# Requirements

- AWS account with necessary permissions to create EC2 instances, security groups, and VPCs.
- Predefined hosted zone in Route53 for DNS configuration.
- Terraform installed on your local machine or CI/CD environment.
- Cloudbees CI license file (`license.key` and `license.cert`) placed in the `secrets` directory.
- Terraform version 0.12 or higher.
- AWS provider version 3.0 or higher.

# Usage

- Copy .auto.tfvars.example to .auto.tfvars and fill in the required variables.
- Create hosted zone in Route53 for DNS configuration. and update the .auto.tfvars file
- Copy wildcard license file (`license.key` and `license.cert`) to the `secrets` directory.
- Run `terraform init` to initialize the module.
- Run `terraform plan` to see the execution plan.
- Run `terraform apply` to create the resources defined in the module.
- Run `terraform destroy` to remove the resources created by the module.

# Verify

List resources that have been created (see also default tags in `.auto.tfvars`)

```
aws resourcegroupstaggingapi get-resources \
  --tag-filters Key=cb-user,Values=<YOUR_USRENAME> \
  --region <YOUR_REGION>
```

```
aws resourcegroupstaggingapi get-resources \
  --tag-filters Key=cb-environment,Values=<YOUR_ENVIRONMENT> \
  --region  <YOUR_REGION>
```

# Connect

```
ssh -i "<YOUR_LOCAL_PEM>.pem" ec2-user@ec2-<YOUR_INSTANCE>.compute-1.amazonaws.com
```

# Architecture

The diagram below shows the AWS infrastructure provisioned by this Terraform module.

```mermaid
graph TB
    subgraph Internet["🌐 Internet"]
        User(["👤 User / Browser"])
    end

    subgraph DNS["Route 53"]
        R53_OC["A Record\noc.&lt;domain&gt; → ALB"]
        R53_CM["A Record\ncm.&lt;domain&gt; → ALB"]
        R53_Val["CNAME Records\nACM DNS Validation"]
    end

    subgraph ACM["AWS Certificate Manager"]
        CERT["Wildcard TLS Certificate\noc.domain + cm.domain\nDNS Validated"]
    end

    subgraph VPC["AWS VPC  (10.0.0.0/16)"]
        IGW["🌐 Internet Gateway"]

        subgraph AZ_A["Availability Zone A  –  Subnet-0 (10.0.0.0/24)"]
            OC["🖥 EC2 – Operation Center\nt3.medium  :8888\ncloud-init/oc.tpl\nCasC via jenkins.yaml"]
        end

        subgraph AZ_B["Availability Zone B  –  Subnet-1 (10.0.1.0/24)"]
            ASG["⚙️ Auto Scaling Group (min/max 2)\nLaunch Template → t3.medium\ncloud-init/cm.tpl"]
            CM1["🖥 EC2 – Client Controller 1\n:8080"]
            CM2["🖥 EC2 – Client Controller 2\n:8080"]
            EFS_MT["EFS Mount Target\nNFS :2049"]
        end

        subgraph ALB_Layer["Application Load Balancer  (public, multi-AZ)"]
            ALB["☁️ ALB\nHTTPS :443"]
            HTTPS_L["HTTPS Listener\n:443"]
            OC_TG["Target Group\ncb-oc-tg  :8888\nHealth: /whoAmI/api/json"]
            CM_TG["Target Group\ncb-cm-tg  :8080\nSticky sessions (lb_cookie)\nHealth: /whoAmI/api/json"]
        end

        subgraph SGs["Security Groups"]
            ALB_SG["alb-sg\nIngress: 443/tcp 0.0.0.0/0"]
            JEN_SG["jenkins-sg\nIngress: 8888+8080 from ALB\n22/tcp 0.0.0.0/0\nIntra-VPC all TCP"]
            EFS_SG["efs-sg\nIngress: 2049 from jenkins-sg"]
        end

        EFS["🗄 EFS File System\n(bursting throughput)\nPersistent JENKINS_HOME for CM"]
    end

    subgraph Keys["Key Management"]
        TLS_KEY["tls_private_key (RSA-4096)\n→ .pem saved locally"]
        KP["AWS Key Pair"]
    end

    %% User traffic flow
    User -->|"HTTPS oc.domain"| R53_OC
    User -->|"HTTPS cm.domain"| R53_CM
    R53_OC -->|"Alias"| ALB
    R53_CM -->|"Alias"| ALB

    %% ALB → TG routing (host-header rules)
    ALB --> HTTPS_L
    HTTPS_L -->|"Host: oc.domain  prio 100"| OC_TG
    HTTPS_L -->|"Host: cm.domain  prio 101"| CM_TG
    OC_TG -->|":8888"| OC
    CM_TG -->|":8080"| CM1
    CM_TG -->|":8080"| CM2

    %% ASG manages CM instances
    ASG -.->|"Manages"| CM1
    ASG -.->|"Manages"| CM2

    %% EFS
    CM1 -->|"NFS mount"| EFS_MT
    CM2 -->|"NFS mount"| EFS_MT
    EFS_MT --> EFS

    %% Security Groups applied
    ALB --- ALB_SG
    OC --- JEN_SG
    CM1 --- JEN_SG
    CM2 --- JEN_SG
    EFS --- EFS_SG

    %% Internet Gateway
    IGW --> ALB

    %% TLS cert attached to listener
    CERT -->|"ssl_certificate"| HTTPS_L

    %% Route53 ACM validation
    R53_Val -.->|"Validates"| CERT

    %% Key pair used by EC2 instances
    TLS_KEY --> KP
    KP -->|"SSH"| OC
    KP -->|"SSH"| ASG
```

**Key data flows:**

| Path | Protocol | Port |
|------|----------|------|
| User → Route53 → ALB | HTTPS | 443 |
| ALB → Operation Center (OC) | HTTP | 8888 |
| ALB → Client Controller (CM) | HTTP | 8080 |
| CM instances → EFS | NFS | 2049 |
| Admin → EC2 (SSH) | SSH | 22 |

**Component summary:**

| Component | Resource | Count |
|-----------|----------|-------|
| VPC | `aws_vpc` | 1 |
| Public Subnets | `aws_subnet` | 2 (multi-AZ) |
| Internet Gateway | `aws_internet_gateway` | 1 |
| Application Load Balancer | `aws_lb` | 1 (shared) |
| Operation Center EC2 | `aws_instance` | 1 (t3.medium) |
| Client Controller ASG | `aws_autoscaling_group` | 1 (min/max 2) |
| EFS File System | `aws_efs_file_system` | 1 |
| ACM TLS Certificate | `aws_acm_certificate` | 1 (DNS validated) |
| Route53 Records | `aws_route53_record` | 2 + validation CNAMEs |

---

# TODOS

- [ ] Add Agent VM (SSH)
- [ ] Connect Agent VM to Controller (Permanent SSH Agent)
- [ ] Add test Jobs/scripts and scenarios
- [ ] CasC Items review
