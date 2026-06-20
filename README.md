# AWS Cost Governance

> Enterprise-grade FinOps framework for AWS cost attribution, chargeback, and visibility across multi-account organizations using CloudTrail, Athena, and Cost Explorer.

![AWS](https://img.shields.io/badge/AWS-CloudTrail%20%7C%20Athena%20%7C%20Cost%20Explorer-orange?logo=amazon-aws)
![FinOps](https://img.shields.io/badge/FinOps-Chargeback%20%7C%20Showback-blue)
![Terraform](https://img.shields.io/badge/IaC-Terraform-7B42BC?logo=terraform)
![License](https://img.shields.io/badge/License-MIT-green)

---

## Overview

This repository provides a production-ready cost governance framework for AWS enterprise environments. It addresses a common challenge in multi-account organizations: understanding **who is spending what, and why** — with enough granularity for accurate chargeback and actionable FinOps decisions.

The framework integrates:
- **CloudTrail** for API-level event data and resource attribution
- **Amazon Athena** for cost query execution at scale across S3-backed data lakes
- **AWS Cost Explorer** for dimension-level cost breakdowns and forecasting
- **Chargeback frameworks** for per-team, per-project, and per-environment cost allocation

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    AWS Organizations (Management Account)        │
│                                                                  │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────┐  │
│  │  Dev Account │    │ Prod Account │    │ Shared Services  │  │
│  └──────┬───────┘    └──────┬───────┘    └──────┬───────────┘  │
│         │                   │                   │               │
│         └───────────────────┴───────────────────┘               │
│                             │                                    │
│                    ┌────────▼────────┐                           │
│                    │   CloudTrail    │  (Org-wide trail)         │
│                    │  (S3 Central)   │                           │
│                    └────────┬────────┘                           │
│                             │                                    │
│           ┌─────────────────┼──────────────────┐                │
│           │                 │                  │                 │
│    ┌──────▼──────┐  ┌───────▼──────┐  ┌───────▼──────┐         │
│    │   AWS Glue  │  │ Cost & Usage │  │ Cost Explorer │         │
│    │  (Catalog)  │  │  Report (S3) │  │     API       │         │
│    └──────┬──────┘  └───────┬──────┘  └───────┬──────┘         │
│           │                 │                  │                 │
│           └─────────────────┼──────────────────┘                │
│                             │                                    │
│                    ┌────────▼────────┐                           │
│                    │  Amazon Athena  │  (Query Engine)           │
│                    └────────┬────────┘                           │
│                             │                                    │
│                ┌────────────▼────────────┐                       │
│                │  Chargeback Dashboard   │                       │
│                │  (QuickSight / Grafana) │                       │
│                └─────────────────────────┘                       │
└─────────────────────────────────────────────────────────────────┘
```

---

## Features

### Cost Attribution
- Tag-based cost allocation per team, project, environment, and cost center
- Resource-level spend breakdown using CUR (Cost and Usage Report) + Athena
- Untagged resource detection with automated alerting

### Chargeback Framework
- Shared services cost split (networking, logging, security tooling)
- Per-team monthly chargeback reports exported to S3
- Amortized vs. blended rate breakdowns for reserved capacity allocation

### Athena Query Templates
- Top spenders by service, account, region, and tag
- Day-over-day anomaly detection queries
- Reserved Instance / Savings Plan coverage and utilization
- Idle and underutilized resource queries (EC2, RDS, NAT Gateways)

### Multi-Account Visibility
- Consolidated spend view across AWS Organizations
- Per-account budget threshold alerts via SNS
- Account-level cost trend reports with month-over-month deltas

---

## Repository Structure

```
aws-cost-governance/
├── athena/
│   ├── queries/
│   │   ├── top_services_by_spend.sql
│   │   ├── untagged_resources.sql
│   │   ├── ri_coverage_report.sql
│   │   ├── daily_cost_anomaly.sql
│   │   └── per_team_chargeback.sql
│   └── views/
│       ├── cost_by_account.sql
│       └── cost_by_tag_dimension.sql
├── chargeback/
│   ├── shared_services_split.py
│   ├── monthly_report_generator.py
│   └── templates/
│       └── chargeback_template.xlsx
├── cloudtrail/
│   ├── org_trail_setup.tf
│   └── s3_bucket_policy.json
├── cur/
│   ├── cur_setup.tf
│   └── glue_catalog.tf
├── dashboards/
│   └── quicksight/
│       └── cost_governance_dashboard.json
├── lambda/
│   ├── cost_anomaly_notifier/
│   └── untagged_resource_reporter/
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
└── docs/
    ├── tagging-strategy.md
    └── chargeback-methodology.md
```

---

## Prerequisites

| Requirement | Version |
|---|---|
| Terraform | >= 1.5 |
| AWS CLI | >= 2.x |
| Python | >= 3.11 |
| AWS Organizations | Enabled |
| Cost and Usage Report | Enabled (hourly, S3) |

### IAM Permissions Required

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ce:GetCostAndUsage",
        "ce:GetReservationCoverage",
        "ce:GetSavingsPlansCoverage",
        "athena:StartQueryExecution",
        "athena:GetQueryResults",
        "glue:GetTable",
        "s3:GetObject",
        "s3:PutObject",
        "cloudtrail:DescribeTrails",
        "organizations:ListAccounts"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## Getting Started

### 1. Enable Cost and Usage Report

```hcl
# terraform/cur_setup.tf
resource "aws_cur_report_definition" "main" {
  report_name                = "enterprise-cur"
  time_unit                  = "HOURLY"
  format                     = "Parquet"
  compression                = "Parquet"
  additional_schema_elements = ["RESOURCES"]
  s3_bucket                  = aws_s3_bucket.cur.id
  s3_region                  = "us-east-1"
  report_versioning          = "OVERWRITE_REPORT"
}
```

### 2. Configure Athena + Glue Catalog

```bash
cd terraform/
terraform init
terraform apply -target=module.glue_catalog
terraform apply -target=module.athena_workgroup
```

### 3. Run Chargeback Queries

```bash
# Query top 10 services by cost this month
aws athena start-query-execution \
  --query-string file://athena/queries/top_services_by_spend.sql \
  --work-group cost-governance \
  --query-execution-context Database=cur_database
```

### 4. Generate Monthly Chargeback Reports

```bash
pip install -r requirements.txt
python chargeback/monthly_report_generator.py \
  --month 2024-12 \
  --output s3://your-bucket/chargeback/
```

---

## Sample Athena Queries

### Top Services by Spend (Current Month)

```sql
SELECT
  line_item_product_code AS service,
  SUM(line_item_unblended_cost) AS total_cost,
  line_item_usage_account_id AS account_id
FROM cur_database.cur_table
WHERE
  year = '2024' AND month = '12'
  AND line_item_line_item_type NOT IN ('Tax', 'Credit')
GROUP BY 1, 3
ORDER BY total_cost DESC
LIMIT 20;
```

### Untagged Resources

```sql
SELECT
  line_item_resource_id,
  line_item_product_code,
  SUM(line_item_unblended_cost) AS cost
FROM cur_database.cur_table
WHERE
  resource_tags_user_team IS NULL
  AND line_item_line_item_type = 'Usage'
  AND year = '2024' AND month = '12'
GROUP BY 1, 2
HAVING cost > 10
ORDER BY cost DESC;
```

---

## Tagging Strategy

All resources must carry these mandatory tags for cost attribution:

| Tag Key | Example Value | Purpose |
|---|---|---|
| `team` | `platform-engineering` | Team chargeback |
| `project` | `eks-migration` | Project-level attribution |
| `environment` | `prod` / `staging` | Env split |
| `cost-center` | `CC-1042` | Finance chargeback |
| `owner` | `team-owner@example.com` | Accountability |

---

## Cost Anomaly Detection

Budget alerts are configured via AWS Budgets + SNS:

```hcl
resource "aws_budgets_budget" "per_account" {
  for_each    = var.account_ids
  name        = "budget-${each.key}"
  budget_type = "COST"
  limit_amount = var.monthly_budget_limit
  limit_unit   = "USD"
  time_unit    = "MONTHLY"

  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 80
    threshold_type             = "PERCENTAGE"
    notification_type          = "ACTUAL"
    subscriber_sns_topic_arns  = [aws_sns_topic.cost_alerts.arn]
  }
}
```

---

## Contributing

1. Fork the repository
2. Create a feature branch: `git checkout -b feat/your-feature`
3. Follow the tagging strategy and SQL style guidelines in `docs/`
4. Submit a pull request with context on the cost query or framework change

---

## License

MIT — see [LICENSE](LICENSE) for details.
