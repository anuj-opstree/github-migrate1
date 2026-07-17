# AWS AI Agent Cost Optimizer — Project Documentation

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Architecture](#2-architecture)
3. [Full Workflow](#3-full-workflow)
4. [Agent Pipeline — Step by Step](#4-agent-pipeline--step-by-step)
5. [Data Flow](#5-data-flow)
6. [File Structure](#6-file-structure)
7. [AI Integration](#7-ai-integration)
8. [Cost Trends Engine](#8-cost-trends-engine)
9. [Report Output](#9-report-output)
10. [Configuration](#10-configuration)
11. [Setup & Usage](#11-setup--usage)
12. [AWS Permissions Required](#12-aws-permissions-required)

---

## 1. Project Overview

The AWS AI Agent Cost Optimizer is a multi-agent system that:

- Scans your **real AWS account** dynamically (no fixed service list)
- Fetches **actual billing data** from Cost Explorer (daily, weekly, MTD)
- Collects **CloudWatch utilization metrics** per resource
- Sends all data to an **LLM (AI)** for intelligent analysis
- Generates **prioritized cost optimization recommendations** with USD savings estimates
- Explains **cost trends** — why costs changed, what was added/removed
- Outputs a **TXT + PDF report** saved to `output/`

**Supported AI Providers:**

| Provider       | Model                                      |
|----------------|--------------------------------------------|
| Groq           | llama-3.3-70b-versatile                    |
| OpenAI         | gpt-4o                                     |
| Anthropic      | claude-3-5-sonnet-20241022                 |
| AWS Bedrock    | anthropic.claude-3-5-sonnet-20241022-v2:0  |
| AWS Bedrock Nova | amazon.nova-pro-v1:0                     |

---

## 2. Architecture

```
+------------------------------------------------------------------+
|                        main.py (CLI Entry)                       |
|   --demo | --region | --provider | --model                       |
+---------------------------+--------------------------------------+
                            |
                            v
+------------------------------------------------------------------+
|                    OrchestratorAgent                             |
|         Runs all agents sequentially, handles errors             |
+------------------------------------------------------------------+
        |         |         |         |         |         |
        v         v         v         v         v         v
  [Agent 1]  [Agent 2]  [Agent 3]  [Agent 4]  [Agent 5]  [Agent 6]  [Agent 7]
  Discovery  Reconstruct  Billing  Utilization  Recommend  Savings   Report
        |         |         |         |         |         |         |
        +----+----+---------+---------+----+----+---------+---------+
             |                             |
             v                             v
      PipelineState                   AI (LLM)
   (shared data object)         LangChain ChatModel
```

---

## 3. Full Workflow

```
START
  |
  v
[1] CLI parses args (--demo / --region / --provider / --model)
  |
  v
[2] PipelineState initialized (region, run_id, account_id, demo_mode)
  |
  v
[3] OrchestratorAgent starts pipeline
  |
  +---------------------------------------------------------------+
  |                                                               |
  v                                                               |
[Agent 1 — DiscoveryAgent]                                        |
  |  - Calls STS → gets Account ID                               |
  |  - Resource Groups Tagging API → discovers ALL resources      |
  |  - Cost Explorer (MONTHLY) → spend by service                |
  |  - Cost Explorer (DAILY)   → daily costs last 30 days        |
  |  - Cost Explorer (MTD)     → month-to-date total             |
  |  - CloudWatch → CPU, Network, IOPS per EC2/RDS/Lambda/ECS    |
  |  - EC2/RDS/Lambda/ECS/EBS/EIP/S3/NAT/ELB/CloudFront metadata|
  |  → Stores in state.raw_inventory                             |
  |                                                               |
  v                                                               |
[Agent 2 — ReconstructionAgent]                                   |
  |  - Builds inventory snapshot (count per service)             |
  |  - Loads last_snapshot.json from output/                     |
  |  - Detects ADDED / REMOVED resources vs last scan            |
  |  - Saves new snapshot to output/last_snapshot.json           |
  |  → Stores in state.inventory_snapshot, state.inventory_changes|
  |                                                               |
  v                                                               |
[Agent 3 — BillingAgent]                                          |
  |  - Parses cost_explorer.by_service → cost_by_service         |
  |  - Computes total_monthly_spend                              |
  |  - Builds daily cost list (last 30 days)                     |
  |  - Groups daily into weekly buckets                          |
  |  - Computes Day-over-Day change                              |
  |  - Computes Week-over-Week change                            |
  |  - Finds top services last 7 days                            |
  |  → Stores in state.cost_by_service, state.cost_trends        |
  |                                                               |
  v                                                               |
[Agent 4 — UtilizationAgent]                                      |
  |  - Merges service_metadata + cloudwatch_metrics per resource |
  |  - EC2: actual instances only (from metadata, not tags)      |
  |  - RDS: actual DB instances only (filters out subnet groups) |
  |  - Lambda, ECS, DynamoDB, ElastiCache, SQS structured        |
  |  - EBS, EIP, S3, NAT, ELB, CloudFront passed through         |
  |  → Stores in state.utilization_data                          |
  |                                                               |
  v                                                               |
[Agent 5 — RecommendationAgent]  <--- LLM CALL HERE              |
  |  - Builds JSON payload:                                       |
  |      account_id, region, spend, cost_trends,                 |
  |      inventory, utilization, metadata, all_resources         |
  |  - LangChain chain: ChatPromptTemplate | LLM | JsonOutputParser|
  |  - AI returns structured AnalysisResult:                     |
  |      executive_summary, key_findings,                        |
  |      cost_trend_analysis, recommendations[]                  |
  |  - Each recommendation: priority, service, resource_id,      |
  |      issue, action, estimated_monthly_saving, effort, category|
  |  - Fallback: raw call_ai() if chain fails                    |
  |  → Stores in state.recommendations, state.ai_*              |
  |                                                               |
  v                                                               |
[Agent 6 — SavingsAgent]                                          |
  |  - Sums estimated_monthly_saving from all recommendations    |
  |  - Groups savings by priority / service / category           |
  |  - Computes RI opportunity (EC2 spend × 40%)                 |
  |  - Computes Savings Plan opportunity (total spend × 30%)     |
  |  - Uses max(AI total, computed total) for best estimate      |
  |  → Stores in state.savings_summary                           |
  |                                                               |
  v                                                               |
[Agent 7 — ReportAgent]                                           |
  |  - Generates TXT report (printed to terminal + saved)        |
  |  - Generates PDF report (styled, via reportlab)              |
  |  - Sections: Summary, Cost Trends, Spend, Inventory,         |
  |              Changes, Savings, Recommendations, Details       |
  |  - Safe path validation (prevents path traversal)            |
  |  → Saves to output/aws_cost_report_<run_id>.txt/.pdf         |
  |                                                               |
  +---------------------------------------------------------------+
  |
  v
END — Reports saved to output/
```

---

## 4. Agent Pipeline — Step by Step

### Agent 1 — DiscoveryAgent

**Purpose:** Scan the entire AWS account dynamically.

**What it does:**
- Calls `sts.get_caller_identity()` to get the real Account ID
- Uses `resourcegroupstaggingapi.get_resources()` (paginated) to discover every tagged/untagged resource across all services
- Calls Cost Explorer 4 times:
  - Monthly spend by service (last 30 days)
  - Daily spend totals (last 30 days, DAILY granularity)
  - Daily spend by service (last 30 days)
  - MTD total (1st of month to today)
- Collects CloudWatch metrics for: EC2, RDS, Lambda, ECS, DynamoDB, ElastiCache, SQS
- Fetches service-specific metadata via 11 focused helpers:

| Helper          | AWS API Called                        |
|-----------------|---------------------------------------|
| `_meta_ec2`     | ec2.describe_instances                |
| `_meta_rds`     | rds.describe_db_instances             |
| `_meta_lambda`  | lambda.list_functions (paginated)     |
| `_meta_ecs`     | ecs.list_clusters + list_services     |
| `_meta_dynamodb`| dynamodb.list_tables + describe_table |
| `_meta_ebs`     | ec2.describe_volumes (paginated)      |
| `_meta_eip`     | ec2.describe_addresses                |
| `_meta_s3`      | s3.list_buckets                       |
| `_meta_nat`     | ec2.describe_nat_gateways             |
| `_meta_elb`     | elbv2.describe_load_balancers         |
| `_meta_cloudfront` | cloudfront.list_distributions      |

**Output:** `state.raw_inventory` — complete raw AWS data dict

---

### Agent 2 — ReconstructionAgent

**Purpose:** Build inventory snapshot and detect infrastructure changes.

**What it does:**
- Counts resources per service from Resource Groups Tagging API results
- Also counts from metadata (EC2, RDS, Lambda, S3, EBS, EIP, etc.)
- Loads `output/last_snapshot.json` from previous run
- Compares current vs previous counts → detects ADDED / REMOVED
- Saves new snapshot for next run comparison

**Output:** `state.inventory_snapshot` (counts), `state.inventory_changes` (diff list)

---

### Agent 3 — BillingAgent

**Purpose:** Parse all cost data and build trend analysis.

**What it does:**
- Parses monthly spend by service → sorted by cost descending
- Builds daily cost list from Cost Explorer DAILY data
- Groups daily into 7-day weekly buckets
- Computes Day-over-Day (DoD) change in USD
- Computes Week-over-Week (WoW) change in USD
- Finds top services by spend in last 7 days

**Output:** `state.cost_by_service`, `state.total_monthly_spend`, `state.cost_trends`

```
cost_trends = {
    yesterday_date, yesterday_cost,
    mtd_total, mtd_start,
    daily_costs      [ {date, cost}, ... ],
    weekly_buckets   [ {start, end, total, days}, ... ],
    dod_change       (float, USD),
    wow_change       (float, USD),
    top_services_last7days  { service: cost }
}
```

---

### Agent 4 — UtilizationAgent

**Purpose:** Merge metadata + CloudWatch metrics into a clean utilization view.

**Key fixes applied:**
- EC2: uses `service_metadata["ec2"]` as source of truth — only real EC2 instances, not VPC subnets/tags
- RDS: filters out subnet groups — only actual DB instances with a known `class`
- All services: raw data passed to AI, no hardcoded thresholds

**Output:** `state.utilization_data` — structured per-service dict

---

### Agent 5 — RecommendationAgent

**Purpose:** Send all AWS data to LLM and get structured recommendations.

**LangChain Chain:**
```
ChatPromptTemplate
    |
    v
LLM (ChatGroq / ChatOpenAI / ChatAnthropic / ChatBedrock)
    |
    v
JsonOutputParser (pydantic_object=AnalysisResult)
    |
    v
AnalysisResult {
    executive_summary,
    total_estimated_monthly_savings,
    key_findings,
    cost_trend_analysis,
    recommendations: [
        {
            priority,        # HIGH / MEDIUM / LOW
            service,         # EC2, RDS, EBS, S3, ...
            resource_id,     # exact AWS resource ID
            resource_name,
            issue,           # what is wrong
            action,          # exact action to take
            estimated_monthly_saving,  # USD
            effort,          # LOW / MEDIUM / HIGH
            category         # Rightsizing | Idle Resource | ...
        }
    ]
}
```

**AI Prompt instructs the LLM to:**
- Cover every service present in the data
- Use exact AWS pricing for savings estimates (gp3=$0.08/GB, EIP=$3.65/mo, etc.)
- Explain daily/weekly/monthly cost trends
- Identify what changed in the environment
- Never invent resources — only analyze what's in the data

**Fallback:** If LangChain chain fails (e.g. token limit), falls back to raw `call_ai()` with JSON parsing.

---

### Agent 6 — SavingsAgent

**Purpose:** Aggregate all savings numbers into a summary.

**Calculations:**
```
total_savings     = sum(rec.estimated_monthly_saving for all recs)
best_total        = max(total_savings, ai_total_savings)
ri_opportunity    = ec2_monthly_spend × 40%
sp_opportunity    = total_monthly_spend × 30%
grand_total/month = best_total + ri_opportunity
grand_total/year  = grand_total/month × 12
```

**Output:** `state.savings_summary` with breakdowns by priority, service, category

---

### Agent 7 — ReportAgent

**Purpose:** Generate TXT and PDF reports.

**TXT Report Sections:**
1. Header (Run ID, Region, Account ID, Timestamp, AI Model)
2. AI Executive Summary
3. Key Findings
4. Cost & Savings Summary
5. **Cost Trends** (Daily, MTD, DoD, WoW, Weekly Breakdown, Daily Table, AI Trend Analysis)
6. Current Spend by Service
7. Resource Inventory
8. Infrastructure Changes
9. Savings by Service / Category
10. AI Recommendations (grouped by HIGH / MEDIUM / LOW)
11. EC2 Instance Details
12. RDS Instance Details
13. Unattached EBS Volumes
14. Idle Elastic IPs

**PDF Report:** Same sections with styled tables, AWS orange/dark color scheme via reportlab.

**Security:** `_safe_path()` validates all output paths with `os.path.realpath` boundary check to prevent path traversal.

---

## 5. Data Flow

```
AWS Account
    |
    |-- STS ──────────────────────────────────> account_id
    |-- Resource Groups Tagging API ──────────> all_resources[]
    |-- Cost Explorer (MONTHLY) ──────────────> by_service{}
    |-- Cost Explorer (DAILY) ────────────────> daily_costs[]
    |-- Cost Explorer (DAILY by service) ─────> daily_by_service{}
    |-- Cost Explorer (MTD) ──────────────────> mtd_total
    |-- CloudWatch ───────────────────────────> cloudwatch_metrics{}
    |-- EC2/RDS/Lambda/ECS/DDB/EBS/EIP/S3 ───> service_metadata{}
    |
    v
PipelineState.raw_inventory
    |
    +---> ReconstructionAgent ──> inventory_snapshot, inventory_changes
    |
    +---> BillingAgent ─────────> cost_by_service, total_monthly_spend, cost_trends
    |
    +---> UtilizationAgent ─────> utilization_data
    |
    +---> RecommendationAgent ──> [JSON payload to LLM]
                |
                v
           LLM Response
                |
                +──> recommendations[]
                +──> ai_executive_summary
                +──> ai_key_findings
                +──> ai_trend_analysis
                +──> ai_total_savings
                |
                v
           SavingsAgent ─────────> savings_summary
                |
                v
           ReportAgent ──────────> output/*.txt, output/*.pdf
```

---

## 6. File Structure

```
Agnet-cost-optmizer/
|
|-- main.py                        # CLI entry point
|-- config.py                      # AI provider settings, lookback days, RI/SP %
|-- requirements.txt               # Python dependencies
|-- .env                           # AWS credentials + AI API keys
|
|-- core/
|   |-- state.py                   # PipelineState dataclass (shared across all agents)
|   |-- base_agent.py              # Abstract BaseAgent with run() + execute()
|   |-- ai_client.py               # LangChain unified LLM client (get_llm, call_ai)
|   `-- demo_data.py               # Rich 15+ service demo data for --demo mode
|
|-- agents/
|   |-- orchestrator_agent.py      # Runs pipeline sequentially, handles errors
|   |-- discovery_agent.py         # AWS scan: Tagging API + Cost Explorer + CloudWatch
|   |-- reconstruction_agent.py    # Inventory snapshot + change detection
|   |-- billing_agent.py           # Cost parsing + daily/weekly/MTD trends
|   |-- utilization_agent.py       # Merge metadata + metrics per service
|   |-- recommendation_agent.py    # LangChain chain → AI recommendations
|   |-- savings_agent.py           # Aggregate savings numbers
|   `-- report_agent.py            # Generate TXT + PDF reports
|
`-- output/
    |-- aws_cost_report_<run_id>.txt
    |-- aws_cost_report_<run_id>.pdf
    `-- last_snapshot.json          # Saved after each run for change detection
```

---

## 7. AI Integration

### LangChain Chain (recommendation_agent.py)

```python
chain = ChatPromptTemplate | LLM | JsonOutputParser(pydantic_object=AnalysisResult)
result = chain.invoke({"format_instructions": ..., "aws_data": ...})
```

### Supported Providers (core/ai_client.py)

```python
"openai"       → ChatOpenAI(model="gpt-4o")
"anthropic"    → ChatAnthropic(model="claude-3-5-sonnet-20241022")
"bedrock"      → ChatBedrock(model_id="anthropic.claude-3-5-sonnet-20241022-v2:0")
"bedrock-nova" → ChatBedrock(model_id="amazon.nova-pro-v1:0")
"groq"         → ChatGroq(model="llama-3.3-70b-versatile")
```

### Pydantic Output Schema

```python
class Recommendation(BaseModel):
    priority:                  str    # HIGH | MEDIUM | LOW
    service:                   str    # EC2, RDS, EBS, S3, ...
    resource_id:               str    # exact AWS resource ID
    resource_name:             str
    issue:                     str
    action:                    str
    estimated_monthly_saving:  float  # USD
    effort:                    str    # LOW | MEDIUM | HIGH
    category:                  str    # Rightsizing | Idle Resource | ...

class AnalysisResult(BaseModel):
    executive_summary:               str
    total_estimated_monthly_savings: float
    key_findings:                    List[str]
    cost_trend_analysis:             str
    recommendations:                 List[Recommendation]
```

---

## 8. Cost Trends Engine

### Data Collection (DiscoveryAgent)

```
Cost Explorer API calls:
  1. MONTHLY granularity, grouped by SERVICE  → by_service{}
  2. DAILY granularity, total per day         → daily_costs[]
  3. DAILY granularity, grouped by SERVICE    → daily_by_service{}
  4. MTD (month start to today), MONTHLY      → mtd_total
```

### Trend Computation (BillingAgent)

```
Daily costs list:
  [ {date: "2026-07-01", cost: 12.50}, {date: "2026-07-02", cost: 14.20}, ... ]

Weekly buckets (7-day groups):
  Week 1: 2026-06-17 to 2026-06-23  → $87.40
  Week 2: 2026-06-24 to 2026-06-30  → $91.20
  Week 3: 2026-07-01 to 2026-07-07  → $95.60
  ...

Day-over-Day:  daily[-1].cost - daily[-2].cost
Week-over-Week: weekly[-1].total - weekly[-2].total
```

### AI Trend Analysis

The LLM receives the trend data and explains:
- Is the daily pattern stable, increasing, or decreasing?
- What drove week-over-week changes?
- Which services caused cost spikes?
- What changed in the environment (new resources, removed resources)?
- Why costs are at their current level?

---

## 9. Report Output

### TXT Report Structure

```
========================================================================
   AWS AI AGENT COST OPTIMIZER - REPORT
   Run ID    : 20260717_145930
   Region    : us-east-1
   Account   : 258048833558
   Generated : 2026-07-17 09:30 UTC
   AI Model  : groq/llama-3.3-70b-versatile
========================================================================

AI EXECUTIVE SUMMARY
------------------------------------------------------------------------
  [2-3 sentence AI-generated summary]

KEY FINDINGS
------------------------------------------------------------------------
  1. [finding]
  2. [finding]

COST & SAVINGS SUMMARY
------------------------------------------------------------------------
  Current Monthly AWS Spend       : $X.XX
  AI-Identified Savings/month     : $X.XX
  GRAND TOTAL Opportunity/year    : $X.XX
  HIGH Priority Issues            : N
  ...

COST TRENDS
------------------------------------------------------------------------
  Yesterday (YYYY-MM-DD)          : $X.XXXX
  MTD (YYYY-MM-01 to today)       : $X.XXXX
  Day-over-Day Change             : +$X.XXXX
  Week-over-Week Change           : +$X.XXXX

  WEEKLY BREAKDOWN
    Week 1 (start to end)         : $X.XXXX
    ...

  DAILY COST (Last 14 Days)
    YYYY-MM-DD                    : $X.XXXX
    ...

  AI COST TREND ANALYSIS
    [AI explanation of trends]

CURRENT SPEND BY SERVICE (Last 30 Days)
RESOURCE INVENTORY
INFRASTRUCTURE CHANGES (vs last scan)
SAVINGS OPPORTUNITY BY SERVICE
SAVINGS OPPORTUNITY BY CATEGORY
AI-GENERATED RECOMMENDATIONS & ACTION ITEMS
  [HIGH] → [MEDIUM] → [LOW]
EC2 INSTANCE DETAILS
RDS INSTANCE DETAILS
UNATTACHED EBS VOLUMES
IDLE ELASTIC IPs
```

### PDF Report

- AWS-branded color scheme (orange `#FF9900`, dark `#232F3E`)
- Styled tables with alternating row colors
- Priority color coding: RED=HIGH, AMBER=MEDIUM, GREEN=LOW
- Sections: Executive Summary, Key Findings, Cost Trends (weekly + daily tables), Savings Summary, Spend by Service, Recommendations table

---

## 10. Configuration

### config.py

```python
AI_PROVIDER    = "groq"                      # openai | anthropic | bedrock | bedrock-nova | groq
AI_MODEL       = "llama-3.3-70b-versatile"   # overridden by --provider or --model
AI_MAX_TOKENS  = 4096
AI_TEMPERATURE = 0.1

PROVIDER_MODEL_MAP = {
    "openai":        "gpt-4o",
    "anthropic":     "claude-3-5-sonnet-20241022",
    "bedrock":       "anthropic.claude-3-5-sonnet-20241022-v2:0",
    "bedrock-nova":  "amazon.nova-pro-v1:0",
    "groq":          "llama-3.3-70b-versatile",
}

COST_LOOKBACK_DAYS = 30    # Cost Explorer lookback
CW_LOOKBACK_DAYS   = 14    # CloudWatch metrics lookback
RI_SAVINGS_PCT     = 0.40  # Reserved Instance savings estimate (40%)
SP_SAVINGS_PCT     = 0.30  # Savings Plan savings estimate (30%)
```

### .env

```
AWS_ACCESS_KEY_ID=<your-key>
AWS_SECRET_ACCESS_KEY=<your-secret>
AWS_DEFAULT_REGION=us-east-1

GROQ_API_KEY=<your-groq-key>
OPENAI_API_KEY=<your-openai-key>
ANTHROPIC_API_KEY=<your-anthropic-key>
```

---

## 11. Setup & Usage

### Install

```bash
pip install -r requirements.txt
```

### Run — Demo Mode (no AWS credentials needed)

```bash
python main.py --demo
python main.py --demo --provider groq
python main.py --demo --provider openai
python main.py --demo --provider anthropic
```

### Run — Live AWS Scan

```bash
python main.py --region us-east-1 --provider groq
python main.py --region eu-west-1 --provider openai
python main.py --region us-east-1 --provider bedrock
python main.py --region us-east-1 --provider groq --model llama-3.1-8b-instant
```

### CLI Arguments

| Argument     | Description                                              |
|--------------|----------------------------------------------------------|
| `--demo`     | Use demo data, no AWS credentials needed                 |
| `--region`   | AWS region to scan (default: us-east-1)                  |
| `--provider` | AI provider: openai / anthropic / bedrock / bedrock-nova / groq |
| `--model`    | Override default model for the selected provider         |

---

## 12. AWS Permissions Required

For live scan, the IAM user/role needs:

```
sts:GetCallerIdentity
ce:GetCostAndUsage
tag:GetResources
cloudwatch:GetMetricStatistics
ec2:DescribeInstances
ec2:DescribeVolumes
ec2:DescribeAddresses
ec2:DescribeNatGateways
rds:DescribeDBInstances
lambda:ListFunctions
ecs:ListClusters
ecs:ListServices
dynamodb:ListTables
dynamodb:DescribeTable
elasticloadbalancing:DescribeLoadBalancers
cloudfront:ListDistributions
s3:ListBuckets
```

---

*Generated for: AWS AI Agent Cost Optimizer*
*Project path: c:\Users\Aryan\Downloads\Agnet-cost-optmizer*
