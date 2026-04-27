# DOST PAGASA — AWS Infrastructure

> Philippine Atmospheric, Geophysical and Astronomical Services Administration

## Architecture

```
VPC (10.2.0.0/16)
├── Public Subnets (shared)
│   ├── Bastion Host (AL2023, t3.micro)
│   └── ALB (prod only) + WAF
├── Private Subnets - Dev (10.2.10.0/24, 10.2.11.0/24)
│   └── 1 standalone EC2 (t3.medium)
├── Private Subnets - Staging (10.2.20.0/24, 10.2.21.0/24)
│   └── 1 standalone EC2 (t3.medium)
├── Private Subnets - Prod (10.2.30.0/24, 10.2.31.0/24)
│   └── ASG 2-6 EC2s (t3.xlarge) behind ALB + WAF
├── ElastiCache Valkey (cache.m5.xlarge, encrypted)
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
| dost-pagasa-monitoring-prod | monitoring.yaml | Prod only | CloudWatch alarms + SNS |

## Deploy Order

```
1. network
2. security-dev, security-staging, security-prod
3. certificates
4. cache (requires security-prod)
5. storage
6. compute-dev, compute-staging, compute-prod
7. monitoring-prod (requires compute-prod)
```

## How to Deploy

1. Go to **GitHub → Actions → Deploy Infrastructure**
2. Click **Run workflow**
3. Select the stack from the dropdown
4. Click **Run workflow**

Each stack deploys independently. Only changed resources get updated.

## Compute Stack Details

### Migration History: AL2023 → Amazon Linux 2

The compute stack was originally built on Amazon Linux 2023 (AL2023). It was migrated to Amazon Linux 2 (AL2) due to PAGASA's requirement for PHP 7.4 with FPM, which their existing applications depend on.

#### Original Setup (AL2023)

| Component | Value |
|-----------|-------|
| AMI | `al2023-ami-kernel-default-x86_64` |
| Package manager | `dnf` |
| PHP install method | Remi repo for EL9 + `dnf module enable php:remi-7.4` |
| Health check path | `/health` (directory with `index.php` inside) |
| CloudWatch agent | `fetch-config` without a config file |
| PHP packages | `php-zip`, `php-pecl-redis` via Remi |

#### Why It Failed

1. **AL2023 does not support `dnf module`** — modularity was dropped. The Remi EL9 RPMs are not compatible with AL2023 since it's not RHEL 9. PHP 7.4 installation failed silently during boot.
2. **`set -euxo pipefail` in UserData** — the script exited on the first error, so Apache and PHP-FPM never installed. But CloudFormation reported success because it doesn't monitor UserData execution.
3. **Health check used a directory path** — `/health` pointed to a directory (`/var/www/html/health/index.php`). Apache returned a 301 redirect to `/health/`, which the ALB treated as unhealthy (expected 200).
4. **CloudWatch agent had no config file** — `fetch-config` without `-c` flag failed on AL2, killing the rest of the script due to `set -e`.
5. **Prod ASG kept cycling** — unhealthy instances were terminated and replaced in a loop, causing slow deploys that never completed.
6. **Dev/staging silently broken** — no ALB health check to surface the failure, so stacks appeared successful but PHP was never installed.

#### Current Setup (Amazon Linux 2)

| Component | Value |
|-----------|-------|
| AMI | `amzn2-ami-hvm-x86_64-gp2` |
| Package manager | `yum` |
| PHP install method | `amazon-linux-extras install php7.4 -y` |
| Health check path | `/health.html` (plain HTML file, no PHP dependency) |
| CloudWatch agent | Explicit JSON config with `-c file:` flag |
| PHP packages | `php-pecl-zip`, `php-pecl-redis` from extras repo |

#### Why Amazon Linux 2

PAGASA's web applications require PHP 7.4 with FPM. AL2023 only ships PHP 8.1+ natively and does not support the Remi repository or `dnf module` commands needed for PHP 7.4. Amazon Linux 2 provides PHP 7.4 through `amazon-linux-extras`, which is the cleanest and most reliable path.

> **Note:** Amazon Linux 2 standard support ended June 30, 2025. A future migration to AL2023 with PHP 8.x should be planned once PAGASA's applications are updated.

### OS and Software

- AMI: Amazon Linux 2 (`amzn2-ami-hvm-x86_64-gp2` via SSM parameter)
- Web server: Apache httpd 2.4.x
- PHP: 7.4.33 (via `amazon-linux-extras install php7.4`)
- PHP-FPM: running as `apache` user
- CloudWatch Agent: custom config with CPU, memory, disk metrics + log shipping

### PHP Extensions Installed

`php-cli`, `php-fpm`, `php-common`, `php-pdo`, `php-mysqlnd`, `php-json`, `php-opcache`, `php-mbstring`, `php-xml`, `php-gd`, `php-pecl-zip`, `php-curl`, `php-bcmath`, `php-pecl-redis`

### ALB Health Check (prod only)

- Path: `/health.html`
- File on disk: `/var/www/html/health.html` (plain text "OK")
- Protocol: HTTP, Port: 80
- Healthy threshold: 3, Unhealthy threshold: 3, Interval: 30s
- Expected response: HTTP 200

### Important Notes

- **PHP install must use `amazon-linux-extras install php7.4 -y`** (not `enable` + `yum install`). Using `enable` causes a multilib conflict with the pre-installed `php-common-5.4` on AL2.
- **Dev/staging are standalone EC2 instances.** Updating UserData in CloudFormation does NOT replace the instance. You must delete and recreate the stack, or set up manually via SSH.
- **Prod uses ASG with rolling updates.** LaunchTemplate changes trigger automatic instance replacement via the `UpdatePolicy`.
- **CloudFormation `deploy` command does not pass SSM parameter overrides.** The `AmiId` parameter is resolved from the template default. Do not add `AmiId` to parameter files.
- **CloudWatch agent config cannot contain `${aws:InstanceId}` in CloudFormation templates.** The `validate-template` API rejects it. Use a placeholder + `sed` replacement at runtime instead.
- **Amazon Linux 2 EOL: June 30, 2025 (standard support ended).** Plan migration to AL2023 + PHP 8.x.

## SSH Access

```
On-premises → Bastion Host (public subnet) → EC2 instances (private subnets)
               dost-pagasa-bastion-key.pem     dost-pagasa-{env}-key.pem
```

Key pairs are stored on the bastion at `/tmp/dost-pagasa-{env}-key.pem`.

### Manual Instance Verification

```bash
echo "Host: $(hostname)"
echo "OS: $(cat /etc/system-release)"
echo "Apache: $(systemctl is-active httpd)"
echo "PHP-FPM: $(systemctl is-active php-fpm)"
echo "PHP Version: $(php -v | head -1)"
echo "Health Check: HTTP $(curl -s -o /dev/null -w '%{http_code}' http://localhost/health.html)"
```

### Manual PHP Setup (if UserData didn't run)

```bash
sudo amazon-linux-extras install php7.4 -y
sudo yum install -y httpd php-opcache php-mbstring php-xml php-gd php-pecl-zip php-curl php-bcmath php-pecl-redis
sudo sed -i 's/^user = .*/user = apache/' /etc/php-fpm.d/www.conf
sudo sed -i 's/^group = .*/group = apache/' /etc/php-fpm.d/www.conf
sudo systemctl enable httpd php-fpm
sudo systemctl start httpd php-fpm
echo "OK" | sudo tee /var/www/html/health.html
```

## Tags

All resources are tagged with:

| Tag | Value |
|-----|-------|
| Organization | DOST-PAGASA |
| Locality | Quezon City |
| Province | Metro Manila |
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
- WAF logging to CloudWatch Logs (365-day retention)
