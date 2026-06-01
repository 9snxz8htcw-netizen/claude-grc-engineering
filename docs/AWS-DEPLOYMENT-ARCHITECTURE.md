# AWS Deployment Architecture

## Overview

This dashboard runs on AWS using managed services:

- **RDS PostgreSQL** — data layer
- **ECS Fargate** — React frontend
- **Lambda + EventBridge** — integrations
- **Secrets Manager** — credentials
- **CloudWatch** — monitoring

## Architecture Diagram

```
  Users / Okta SAML
         │
         ▼
  ┌─────────────┐
  │     ALB     │  (public subnet)
  └──────┬──────┘
         │
─────────┼──────────────── VPC ────────────────────────────────
         │                  private subnet
         ▼
  ┌──────────────────┐      ┌──────────────────────────┐
  │   ECS Fargate    │      │      RDS PostgreSQL       │
  │  (React SPA)     │─────▶│  Multi-AZ, db.t4g.large  │
  │  0.5–1 vCPU      │      │  Automated backups 7 days │
  │  1–2 GB RAM      │      └──────────────────────────┘
  └──────────────────┘
                                      ▲
  ┌───────────────────────────────────┼──────────────────┐
  │  Lambda Functions                 │                  │
  │  ├─ jira-adapter   (every 4h) ────┘                  │
  │  ├─ qualys-adapter (daily)                           │
  │  └─ splunk-adapter (daily)                           │
  └───────────────────────────────────────────────────────┘
         ▲
         │  schedules
  ┌──────┴──────┐     ┌──────────────────┐
  │ EventBridge │     │ Secrets Manager  │
  │  (triggers) │     │ Jira / Qualys /  │
  └─────────────┘     │ Splunk / RDS pw  │
                      └──────────────────┘

  All services → CloudWatch (logs, metrics, alarms)
```

## Services Breakdown

### 1. RDS PostgreSQL

| Setting | Dev | Prod |
| --- | --- | --- |
| Engine | PostgreSQL 14+ | PostgreSQL 14+ |
| Instance class | db.t4g.small | db.t4g.large |
| Multi-AZ | No | Yes |
| Automated backups | 7 days | 7 days |
| Credentials | Secrets Manager | Secrets Manager |

Row-level security is enabled so each tenant can only read their own data. The database is placed in a private subnet with no public endpoint; only ECS tasks and Lambda functions access it via security group rules.

### 2. ECS Fargate

| Setting | Dev | Prod |
| --- | --- | --- |
| Container | React SPA (nginx) | React SPA (nginx) |
| CPU | 0.25 vCPU | 0.5–1 vCPU |
| Memory | 512 MB | 1–2 GB |
| Auto-scaling | — | CloudWatch CPU/request metrics |

The task definition pulls the image from ECR. Fargate tasks run in private subnets and are only reachable through the Application Load Balancer.

### 3. Lambda Functions

| Function | Trigger | Schedule |
| --- | --- | --- |
| `jira-adapter` | EventBridge rule | Every 4 hours |
| `qualys-adapter` | EventBridge rule | Daily at 02:00 UTC |
| `splunk-adapter` | EventBridge rule | Daily at 02:30 UTC |

Each function reads its API key from Secrets Manager at invocation time via an IAM role — no environment-variable secrets. Results are written directly to RDS.

### 4. Secrets Manager

| Secret | Used by |
| --- | --- |
| `jira-api-key` | `jira-adapter` Lambda |
| `qualys-api-key` | `qualys-adapter` Lambda |
| `splunk-api-key` | `splunk-adapter` Lambda |
| `rds-password` | ECS tasks, Lambda functions |

Access is granted exclusively through IAM roles attached to each compute resource. No secret values are passed as environment variables or stored in task definitions.

### 5. CloudWatch

| Resource | What is logged / measured |
| --- | --- |
| ECS tasks | Application stdout/stderr → `/ecs/dashboard` log group |
| Lambda functions | Invocation logs, duration, errors → `/aws/lambda/<name>` |
| RDS | Enhanced Monitoring, Performance Insights |
| ALB | Access logs (5-minute intervals) |
| Alarms | Lambda error rate >1%, RDS CPU >80%, ECS task restarts |

SNS topics route alarm notifications to the on-call email/Slack channel.

## Security

| Control | Implementation |
| --- | --- |
| Network segmentation | RDS and ECS in private subnets; only ALB is public-facing |
| IAM least privilege | Each Lambda and ECS task has a dedicated role scoped to only the resources it needs |
| Credential storage | All API keys and passwords in Secrets Manager; never in code or environment variables |
| Database access control | Row-level security in PostgreSQL isolates tenant data |
| User authentication | Okta SAML 2.0 via ALB authentication rules |
| Encryption in transit | TLS 1.2+ enforced on ALB, RDS, and Secrets Manager endpoints |
| Encryption at rest | RDS storage encrypted with AWS-managed KMS key; S3 log buckets use SSE-S3 |

## Cost Estimation

Estimates based on us-east-1 on-demand pricing. Production assumes moderate load (~50 active users, adapters running on schedule).

| Service | Dev / month | Prod / month |
| --- | --- | --- |
| RDS db.t4g.small (single-AZ) | $25 | — |
| RDS db.t4g.large (Multi-AZ) | — | $200 |
| ECS Fargate (0.25 vCPU / 512 MB) | $10 | — |
| ECS Fargate (0.5 vCPU / 1 GB, auto-scale) | — | $40–80 |
| Lambda (3 functions, ~200 invocations/day) | $1 | $3 |
| Application Load Balancer | $20 | $20 |
| Secrets Manager (4 secrets) | $2 | $2 |
| CloudWatch (logs + metrics + alarms) | $5 | $15 |
| ECR (image storage) | $1 | $2 |
| Data transfer | $2 | $5 |
| **Total** | **~$66 / month** | **~$287–327 / month** |

> Savings tip: Reserved Instances on RDS (1-year, no upfront) cut the database cost by ~30% in production (~$140/month instead of $200).

## Deployment Steps

1. **Provision RDS** with Terraform (`terraform apply` in `infra/rds/`)
   - Creates the DB instance, subnet group, and security group
   - Stores the generated password in Secrets Manager automatically

2. **Create ECS task definition**
   - Reference the ECR image URI
   - Attach the ECS task IAM role
   - Point log configuration at the CloudWatch log group

3. **Deploy Lambda functions**
   - Package each adapter: `zip -r function.zip .`
   - Upload to Lambda or deploy via `aws lambda update-function-code`
   - Set the execution role and Secrets Manager resource policy

4. **Configure EventBridge rules**
   - Create a rule per adapter with the appropriate cron expression
   - Set the Lambda function as the target

5. **Set up Okta SAML**
   - Create an Okta app integration (SAML 2.0)
   - Configure the ALB OIDC/SAML authentication rule with the Okta metadata URL
   - Test login flow end-to-end

6. **Verify security groups and RLS**
   - Confirm ECS security group allows outbound to RDS on port 5432 only
   - Run `EXPLAIN (ANALYZE) SELECT ...` with a test tenant to confirm row-level security is active
   - Run `aws rds describe-db-instances` to verify Multi-AZ and encryption flags

## Monitoring & Alerts

| Alert | Trigger | Notification |
| --- | --- | --- |
| Lambda error spike | Error rate > 1% over 5 minutes | SNS → email / Slack |
| Lambda timeout | Duration > 80% of configured timeout | SNS → email / Slack |
| RDS CPU high | CPU utilisation > 80% for 10 minutes | SNS → PagerDuty |
| RDS storage low | Free storage < 10% | SNS → email |
| ECS task restarts | Task stopped unexpectedly | SNS → Slack |
| ALB 5xx rate | 5xx responses > 5% of requests | SNS → email / Slack |

CloudWatch Dashboards provide a single pane for CPU, memory, Lambda invocation counts, RDS connections, and ALB request rates. RDS Performance Insights is enabled for query-level diagnostics in production.
