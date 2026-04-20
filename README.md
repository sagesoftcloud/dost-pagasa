# DOST PAGASA — AWS Infrastructure

> Philippine Atmospheric, Geophysical and Astronomical Services Administration

## Project Details

| | |
|---|---|
| **AWS Account** | 305366321337 |
| **Region** | ap-southeast-1 (Singapore) |
| **VPC CIDR** | 10.2.0.0/16 |
| **On-Premises** | 10.100.100.0/24 (Sangfor NGAF M5800) |
| **VPN Public IP** | 122.55.201.79 |
| **Domains** | pagasa.dost.gov.ph, www.pagasa.dost.gov.ph, bagong.pagasa.dost.gov.ph |

## Architecture

```
VPC: 10.2.0.0/16
├── Public Subnets (shared): 10.2.1.0/24, 10.2.2.0/24
│   ├── Bastion Host
│   └── ALB (prod only)
├── Private Subnets - Dev: 10.2.10.0/24, 10.2.11.0/24
│   └── 1 standalone EC2 (t3.small)
├── Private Subnets - Staging: 10.2.20.0/24, 10.2.21.0/24
│   └── 1 standalone EC2 (t3.medium)
├── Private Subnets - Prod: 10.2.30.0/24, 10.2.31.0/24
│   └── ASG 2-10 EC2s (t3.large) behind ALB + WAF
├── ElastiCache Valkey (shared, encrypted)
├── NAT Gateway (shared)
└── Site-to-Site VPN → DOST PAGASA on-premises
```

## CloudFormation Stacks

| Stack | Template | Type | Description |
|-------|----------|------|-------------|
| dost-pagasa-network | network.yaml | Shared | VPC, subnets, NAT, VPN, bastion |
| dost-pagasa-security-dev | security.yaml | Per-env | Dev security groups + WAF |
| dost-pagasa-security-staging | security.yaml | Per-env | Staging security groups + WAF |
| dost-pagasa-security-prod | security.yaml | Per-env | Prod security groups + WAF |
| dost-pagasa-certificates | certificates.yaml | Shared | ACM certificate (needs DNS validation) |
| dost-pagasa-cache | cache.yaml | Shared | ElastiCache Valkey cluster |
| dost-pagasa-storage | storage.yaml | Shared | S3 + CloudFront CDN |
| dost-pagasa-compute-dev | compute.yaml | Per-env | Dev EC2 instance |
| dost-pagasa-compute-staging | compute.yaml | Per-env | Staging EC2 instance |
| dost-pagasa-compute-prod | compute.yaml | Per-env | Prod ALB + ASG |

## Deploy Order

```
1. network
2. security-dev, security-staging, security-prod
3. cache (requires security-prod)
4. storage
5. compute-dev, compute-staging, compute-prod
```

## How to Deploy

1. Go to **GitHub → Actions → Deploy Infrastructure**
2. Click **Run workflow**
3. Select the stack from the dropdown
4. Click **Run workflow**

Each stack deploys independently. Only changed resources get updated.

## SSH Access

```
On-premises (122.55.201.79) → Bastion Host → EC2 instances
                                (bastion-key)   (env-key)
```

| Key Pair | Used By |
|----------|---------|
| dost-pagasa-bastion-key | Bastion host |
| dost-pagasa-dev-key | Dev EC2 |
| dost-pagasa-staging-key | Staging EC2 |
| dost-pagasa-prod-key | Prod EC2 (ASG) |

## Tags

All resources are tagged with:

| Tag | Value |
|-----|-------|
| Agency | DOST-PAGASA |
| Locality | Quezon City |
| Project | DOST-PAGASA |
| Environment | dev / staging / prod |

## HTTPS

Currently HTTP-only. To enable HTTPS:
1. Deploy `certificates` stack
2. DOST PAGASA DNS team adds CNAME records to .gov.ph DNS
3. Wait for ACM validation
4. Add `AcmCertificateArn` to compute-prod parameters
5. Redeploy compute-prod

## Security

- Security groups isolated per environment
- WAF with AWS Managed Rules (OWASP, SQLi, IP reputation, rate limiting)
- EC2 SSH restricted to bastion host only
- Bastion SSH restricted to on-premises IP only
- ElastiCache encrypted at rest and in transit
- S3 buckets private with public access blocked
- VPC Flow Logs enabled (365-day retention)
- EC2 IMDSv2 enforced, EBS encrypted
- CloudFront with Origin Access Control
