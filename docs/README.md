# EKS Infrastructure Setup

Step-by-step guide to provision a production-grade EKS infrastructure on AWS using the CLI. All resources use a configurable `<PREFIX>` (e.g. `acme-prod`) for naming.

## Network Architecture

```
                         ┌─────────────────────────────────────┐
                         │           VPC  10.10.0.0/16         │
                         │                                     │
  Internet ──► IGW ──►   │  ┌──────────┐  ┌──────────┐         │
                         │  │ public-a │  │ public-b │  (ALB)  │
                         │  │ .1.0/24  │  │ .2.0/24  │         │
                         │  └────┬─────┘  └──────────┘         │
                         │       │                             │
                         │      NAT                            │
                         │       │                             │
                         │  ┌────▼─────┐  ┌──────────┐         │
                         │  │  app-a   │  │  app-b   │ (Nodes) │
                         │  │ .10.0/24 │  │ .20.0/24 │         │
                         │  └──────────┘  └──────────┘         │
                         │                                     │
                         │  ┌──────────┐  ┌──────────┐         │
                         │  │  infra   │  │ infra-2  │ (Jenkins│
                         │  │ .30.0/24 │  │ .40.0/24 │  RDS)   │
                         │  └──────────┘  └──────────┘         │
                         └─────────────────────────────────────┘
```

## Core Steps (Required)

| # | Document | Description |
|---|---|---|
| 1 | [01-vpc-and-networking.md](01-vpc-and-networking.md) | VPC, subnets, IGW, NAT, route tables, S3 endpoint, K8s tags |
| 2 | [02-security-groups.md](02-security-groups.md) | ALB, Jenkins, Node, and Control Plane security groups |
| 3 | [03-iam-roles.md](03-iam-roles.md) | EKS cluster role, node role, Jenkins role with instance profile |
| 4 | [04-eks-cluster.md](04-eks-cluster.md) | EKS cluster, access config, node group, OIDC, ALB controller |
| 5 | [05-jenkins.md](05-jenkins.md) | Jenkins EC2, ACM cert, ALB with HTTPS, tool installation |

## Optional Steps

| # | Document | Description |
|---|---|---|
| 6 | [06-ecr.md](06-ecr.md) | ECR repositories with lifecycle policies |
| 7 | [07-rds.md](07-rds.md) | Second infra subnet, RDS MySQL instance, DB initialization |
| 8 | [08-app-dns-and-cert.md](08-app-dns-and-cert.md) | ACM certificate for application domains |

## Naming Convention

All resources follow the pattern `<PREFIX>-<resource>`:

| Resource | Name Pattern |
|---|---|
| VPC | `<PREFIX>-vpc` |
| Subnets | `<PREFIX>-public-a`, `<PREFIX>-app-a`, `<PREFIX>-infra` |
| EKS Cluster | `<PREFIX>-eks` |
| Node Group | `<PREFIX>-ng-app` |
| IAM Roles | `<PREFIX>-eks-cluster-role`, `<PREFIX>-node-role`, `<PREFIX>-jenkins-role` |
| Security Groups | `<PREFIX>-alb-sg`, `<PREFIX>-jenkins-sg`, `<PREFIX>-node-sg` |
| Jenkins | `<PREFIX>-jenkins`, `<PREFIX>-jenkins-alb`, `<PREFIX>-jenkins-tg` |
| ECR | `<PREFIX>-<app-name>` |
| RDS | `<PREFIX>-dyo-db` |

## Subnet Layout

| Subnet | CIDR | AZ | Type | Purpose |
|---|---|---|---|---|
| `<PREFIX>-public-a` | 10.10.1.0/24 | a | Public | ALB |
| `<PREFIX>-public-b` | 10.10.2.0/24 | b | Public | ALB |
| `<PREFIX>-app-a` | 10.10.10.0/24 | a | Private | EKS worker nodes |
| `<PREFIX>-app-b` | 10.10.20.0/24 | b | Private | EKS worker nodes |
| `<PREFIX>-infra` | 10.10.30.0/24 | a | Private | Jenkins, RDS |
| `<PREFIX>-infra-2` | 10.10.40.0/24 | b | Private | RDS (multi-AZ requirement) |

## Key Design Decisions

- **Environment file (`eks_env.sh`)** — each step sources this file for variables from prior steps and appends its own variables at the end; only manually-entered values (PREFIX, REGION, domain names) require `export`
- **Private EKS endpoint only** — no public API access; all cluster management goes through Jenkins (via SSM)
- **Single NAT Gateway** — cost-optimized for non-HA requirements; place in public-a
- **S3 Gateway Endpoint** — avoids NAT costs for ECR image pulls
- **IRSA** — IAM Roles for Service Accounts used for AWS Load Balancer Controller
- **Jenkins on Docker** — runs `jenkins/jenkins:lts-jdk21` with Docker socket mounted for Docker-in-Docker builds
- **TLS 1.3** — ALB uses `ELBSecurityPolicy-TLS13-1-2-2021-06`
- **Immutable ECR tags** — prevents image tag overwriting
