# DOST PAGASA — AWS Infrastructure

> Philippine Atmospheric, Geophysical and Astronomical Services Administration

## Architecture

```
VPC
├── Public Subnets (shared)
│   ├── Bastion Host
│   └── ALB (prod only)
├── Private Subnets - Dev
│   └── 1 standalone EC2 (t3.small)
├── Private Subnets - Staging
│   └── 1 standalone EC2 (t3.medium)
├── Private Subnets - Prod
│   └── ASG 2-10 EC2s (t3.large) behind ALB + WAF
├── ElastiCache Valkey (shared, encrypted)
├── NAT Gateway (shared)
└── Site-to-Site VPN → DOST PAGASA on-premises
```

## CloudFormation Stacks

| Stack | Template | Type | Description |
|-------|----------|------|-------------|
| dost-pagasa-network | network.yaml | Shared | VPC, subnets, NAT, VPN, bastion |
| dost-pagasa-security-{env} | security.yaml | Per-env | Security groups + WAF |
| dost-pagasa-certificates | certificates.yaml | Shared | ACM certificate |
| dost-pagasa-cache | cache.yaml | Shared | ElastiCache Valkey cluster |
| dost-pagasa-storage | storage.yaml | Shared | S3 + CloudFront CDN |
| dost-pagasa-compute-{env} | compute.yaml | Per-env | EC2 / ALB + ASG |

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
On-premises → Bastion Host → EC2 instances
               (bastion-key)   (env-key)
```

## Tags

All resources are tagged with:

| Tag | Value |
|-----|-------|
| Agency | DOST-PAGASA |
| Locality | Quezon City |
| Project | DOST-PAGASA |
| Environment | dev / staging / prod |

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
