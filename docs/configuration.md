# Configuration Guide

## Prerequisites

Before configuring the agent, ensure the following infrastructure is set up in the relevant AWS accounts.

### Athena Setup for CUR (Payer or Centralized Account)

The account hosting CUR data needs:

1. CUR exported to S3 in Parquet format (recommended for cost efficiency)
   - Set up via AWS Billing Console → Cost & Usage Reports → Create Report
   - Enable "Amazon Athena" integration when creating the report
   - See [CUR setup guide](https://docs.aws.amazon.com/cur/latest/userguide/what-is-cur.html)

2. Athena workgroup with an S3 output location for query results
   ```
   Workgroup:  primary (or custom)
   Output:     s3://YOUR-ACCOUNT-athena-results/cur-queries/
   ```

3. Glue database and table created by the CUR integration (auto-created if you enable Athena integration during CUR setup)

### Athena Setup for VPC Flow Logs (Member Accounts)

Each member account with VPC Flow Log analysis needs:

1. VPC Flow Logs delivered to S3 in Parquet format
   - Configure via VPC Console → Flow Logs → Create
   - Destination: S3 bucket
   - File format: Parquet (recommended)
   - See [VPC Flow Logs to S3 guide](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs-s3.html)

2. Athena workgroup with an S3 output location for query results
   ```
   Workgroup:  primary (or custom)
   Output:     s3://YOUR-ACCOUNT-athena-results/vpc-queries/
   ```

3. Glue database and table pointing to the VPC Flow Log S3 data
   - Create via Athena Console or Glue Crawler
   - Partition by year/month/day for query performance

### S3 Buckets Summary

| Bucket | Purpose | Account |
|--------|---------|---------|
| CUR data bucket | Stores CUR export files | Payer or centralized |
| CUR Athena results bucket | Stores Athena query output for CUR | Same as CUR data |
| VPC Flow Log data bucket | Stores VPC Flow Log files | Each member account |
| VPC Flow Log Athena results bucket | Stores Athena query output for VPC | Each member account |

> These buckets can be combined (e.g., one results bucket per account), but the IAM roles must have access to all relevant buckets. See [IAM Permissions](iam-permissions.md) for details.

---

## Overview

The agent is configured via `agent/config.yaml`. Copy from the example template:

```bash
cp agent/config.yaml.example agent/config.yaml
```

## Account Configuration

Athena settings (CUR and VPC Flow Logs) are configured per-account inside the `accounts` section.

- Exactly one payer account is required (must have `role_arn`)
- Member accounts may omit `role_arn` when the agent runs in the same account
- Athena config can appear on any account type

### Account Fields

| Field | Description | Required |
|-------|-------------|----------|
| `account_id` | AWS account ID (12 digits) | Yes |
| `account_type` | `"payer"` or `"member"` | Yes |
| `role_arn` | IAM role ARN to assume | Payer: Yes, Member: No |
| `external_id` | External ID for confused deputy prevention | No |
| `region` | AWS region (defaults to `aws.region`) | No |
| `athena.cur.database` | CUR Athena database name | No |
| `athena.cur.table` | CUR Athena table name | No |
| `athena.vpc_flowlogs.database` | VPC Flow Logs database | No |
| `athena.vpc_flowlogs.table` | VPC Flow Logs table | No |

## Deployment Scenarios

### Scenario A: Centralized Logging Account

Agent runs in the same account as CUR/VPC data. No STS calls needed for Athena queries.

```yaml
accounts:
  # Payer — billing APIs only
  - account_id: "111111111111"
    role_arn: "arn:aws:iam::111111111111:role/CostAnalyzerAgentPayerRole"
    account_type: payer

  # Local account — agent runs here, no role_arn needed
  - account_id: "999999999999"
    account_type: member
    athena:
      cur:
        database: cur_db
        table: cur_table
      vpc_flowlogs:
        database: vpc_db
        table: flow_logs
```

### Scenario B: Audit Account (Distributed Data)

Agent runs in a separate account. CUR in payer, VPC logs in member accounts. All need `role_arn`.

```yaml
accounts:
  - account_id: "111111111111"
    role_arn: "arn:aws:iam::111111111111:role/CostAnalyzerAgentPayerRole"
    account_type: payer
    athena:
      cur:
        database: cur_db
        table: cur_table

  - account_id: "222222222222"
    role_arn: "arn:aws:iam::222222222222:role/CostAnalyzerAgentMemberRole"
    account_type: member
    athena:
      vpc_flowlogs:
        database: vpc_db
        table: flow_logs
```

## Model Configuration

```yaml
agent:
  model:
    model_id: global.anthropic.claude-sonnet-4-5-20250929-v1:0
    temperature: 0.1
    max_tokens: 8192
    cache_tools: true   # Tool definition caching
    cache_ttl: "5m"     # "5m" for interactive, "1h" for batch
```

## Prompt Caching

Enabled by default. Caches the system prompt and tool definitions to reduce costs by ~90%.

| TTL | Best for |
|-----|----------|
| `"5m"` | Interactive CLI sessions (default) |
| `"1h"` | Batch processing, infrequent access |

Cost example (100 queries): $1.50 without caching → $0.17 with caching.

## Customizing Prompts

Edit `shared/prompts.yaml` to add or modify prompts:

```yaml
custom:
  name: "My Team's Queries"
  icon: "🎯"
  prompts:
    - title: "Production Costs"
      prompt: "Show me costs for resources tagged with env=prod"
```

## MCP Configuration

The AWS Knowledge MCP server is enabled by default for documentation search:

```yaml
mcp:
  enabled: true
  servers:
    aws_knowledge:
      type: http
      url: https://knowledge-mcp.global.api.aws
      enabled: true
```
