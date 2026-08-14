<div align="center">

# Anomaly Detector

**Parallel weather anomaly detection across 30 Atlanta-metro cities using AWS Lambda**

[![AWS SAM](https://img.shields.io/badge/AWS-SAM-FF9900?style=flat&logo=amazon-aws)](https://docs.aws.amazon.com/serverless-application-model/)
[![Python](https://img.shields.io/badge/Python-3.11-3776AB?style=flat&logo=python)](https://python.org)
[![React](https://img.shields.io/badge/React-19-61DAFB?style=flat&logo=react)](https://react.dev)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

[Overview](#overview) · [Architecture](#architecture) · [Prerequisites](#prerequisites) · [Quick Start](#quick-start) · [Dashboard](#dashboard) · [Usage](#usage) · [Troubleshooting](#troubleshooting)

</div>

---

## Overview

Anomaly Detector is a serverless cloud system that benchmarks parallel versus sequential execution by running three specialized AWS Lambda workers simultaneously against a weather data workload. Each morning at 8 AM Eastern, the system processes 30 Atlanta-area cities across three data sources and computes temperature anomalies against a 5-year historical baseline.

Demo: [Anomaly Detector](http://anomaly-detector-dashboard.s3-website.us-east-2.amazonaws.com/)

The project measures and compares three execution models:

| Model | Description | Expected time |
|-------|-------------|---------------|
| **Parallel** | 3 workers fire simultaneously | ~3–5 seconds dispatch |
| **Sequential** | Same 3 workers, one at a time | ~75–90 seconds total |
| **Single worker** | 1 Lambda handles all 30 cities | ~90–110 seconds total |

Results are visualized in a live React dashboard hosted on S3 with benchmark analysis, Amdahl's Law validation, and a weather anomaly report.

---

## Architecture

```
EventBridge Schedules
│
├── 8:00 AM ──► AnomalyDetectorCoordinator
│                   ├──► WorkerForecast  (cities  1–10 · Open-Meteo current)
│                   ├──► WorkerArchive   (cities 11–20 · Open-Meteo 5yr avg)
│                   └──► WorkerNWS       (cities 21–30 · NWS observations)
│
├── 8:15 AM ──► AnomalyDetectorSequentialRunner
│                   └──► Same 3 workers, one at a time (benchmark)
│
├── 8:30 AM ──► AnomalyDetectorAggregator
│                   └──► Reads DynamoDB · fetches weather · writes S3 JSON
│
└── 9:00 AM ──► AnomalyDetectorWorkerSingle
                    └──► 1 Lambda · all 30 cities · all 3 sources (benchmark)

Storage
├── DynamoDB: Anomaly_Detector_Results  (weather records per city per day)
├── DynamoDB: AD_Time_Results           (benchmark timing per run)
└── S3: anomaly-detector-dashboard      (JSON results + React dashboard)
```

---

## Prerequisites

Ensure the following are installed before continuing.

| Tool | Minimum version | Install |
|------|----------------|---------|
| Python | 3.11 | [python.org](https://python.org/downloads) |
| Node.js | 18.x | [nodejs.org](https://nodejs.org) |
| AWS CLI | v2 | [Installation guide](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html) |
| AWS SAM CLI | Latest | [Installation guide](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html) |
| Git | Any | [git-scm.com](https://git-scm.com) |

You also need an **AWS account** with an IAM user that has the following policies attached:

```
AmazonDynamoDBFullAccess
AWSLambda_FullAccess
AmazonS3FullAccess
AWSCloudFormationFullAccess
AmazonEventBridgeFullAccess
IAMFullAccess
```

> **Cost:** This project runs entirely within the [AWS Free Tier](https://aws.amazon.com/free/). Estimated monthly cost is **$0.00**.

---

## Quick Start

### 1. Clone and configure AWS

```bash
git clone https://github.com/your-username/anomaly-detector.git
cd anomaly-detector

# Configure AWS CLI with your IAM credentials
aws configure
# Region: us-east-1
# Output format: json
```

### 2. Create AWS resources

```bash
# DynamoDB — main results table
aws dynamodb create-table \
  --table-name Anomaly_Detector_Results \
  --attribute-definitions \
    AttributeName=city,AttributeType=S \
    AttributeName=date,AttributeType=S \
  --key-schema \
    AttributeName=city,KeyType=HASH \
    AttributeName=date,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1

# DynamoDB — benchmark timing table
aws dynamodb create-table \
  --table-name AD_Time_Results \
  --attribute-definitions \
    AttributeName=runID,AttributeType=S \
    AttributeName=src,AttributeType=S \
  --key-schema \
    AttributeName=runID,KeyType=HASH \
    AttributeName=src,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1

# S3 bucket
aws s3 mb s3://anomaly-detector-dashboard --region us-east-1

# Disable public access block
aws s3api put-public-access-block \
  --bucket anomaly-detector-dashboard \
  --region us-east-1 \
  --public-access-block-configuration \
  "BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false"

# Apply bucket policy
aws s3api put-bucket-policy \
  --bucket anomaly-detector-dashboard \
  --region us-east-1 \
  --policy file://aws-config/bucket-policy.json

# Apply CORS configuration
aws s3api put-bucket-cors \
  --bucket anomaly-detector-dashboard \
  --region us-east-1 \
  --cors-configuration file://aws-config/cors.json
```

### 3. Deploy Lambda functions

```bash
sam build 
sam deploy
```

On first deploy SAM will prompt for configuration. Use these values:

```
Stack Name: Anomaly-Detector
AWS Region: us-east-1
Confirm changes before deploy:  y
Allow SAM CLI IAM role creation: y
Save arguments to samconfig.toml: y
```

### 4. Verify deployment

```bash
# Confirm all 7 functions deployed
aws lambda list-functions \
  --region us-east-1 \
  --query 'Functions[*].FunctionName' \
  --output table

# Confirm EventBridge rules created
aws events list-rules --region us-east-1 --output table
```

### 5. Run your first benchmark

```bash
# Trigger parallel run
aws lambda invoke \
  --function-name AnomalyDetectorCoordinator \
  --region us-east-1 \
  --payload file://aws-config/empty.json \
  parallel_response.json && cat parallel_response.json

# Wait 60 seconds, then trigger sequential run
aws lambda invoke \
  --function-name AnomalyDetectorSequentialRunner \
  --region us-east-1 \
  --payload file://aws-config/empty.json \
  --cli-read-timeout 600 \
  sequential_response.json && cat sequential_response.json

# Trigger aggregator to generate anomaly results
aws lambda invoke \
  --function-name AnomalyDetectorAggregator \
  --region us-east-1 \
  --payload file://aws-config/empty.json \
  --cli-read-timeout 600 \
  agg_response.json
```

---

## Dashboard

### Build and deploy

```bash
cd dashboard
npm install
npm run build

aws s3 sync dist/ s3://anomaly-detector-dashboard \
  --acl public-read \
  --region us-east-1
```

### Access

```
http://anomaly-detector-dashboard.s3-website.us-east-1.amazonaws.com
```

The dashboard has five tabs:

| Tab | Description |
|-----|-------------|
| **Benchmark** | Parallel vs sequential speedup, Amdahl's Law analysis |
| **Workers** | Per-worker assignments, timing breakdown, DynamoDB records |
| **Single Worker** | Three-way comparison: parallel vs sequential vs single |
| **Anomalies** | Cities deviating >10°F from 5-year historical average |
| **Archive** | Browse any past date's results |

---

## Project Structure

```
anomaly-detector/
│
├── coordinator/            # Fires all 3 workers simultaneously
│   └── app.py
├── worker_forecast/        # Worker A: Open-Meteo current (cities 1–10)
│   └── app.py
├── worker_archive/         # Worker B: Open-Meteo 5yr avg (cities 11–20)
│   └── app.py
├── worker_nws/             # Worker C: NWS observations (cities 21–30)
│   └── app.py
├── sequential_sys/         # Sequential benchmark runner
│   └── app.py
├── worker_single/          # Single-worker benchmark
│   └── app.py
├── aggregator/             # Computes anomalies, writes dashboard JSON
│   └── app.py
│
├── dashboard/
│   ├── src/
│   │   ├── App.jsx         # React dashboard — 5 tabs, dark mode, mobile responsive
│   │   └── App.css
│   └── package.json
│
├── aws-config/
│   ├── bucket-policy.json  # S3 public read policy
│   ├── cors.json           # S3 CORS rules for browser access
│   └── empty.json          # Empty Lambda payload {}
│
├── template.yaml           # SAM infrastructure definition
├── samconfig.toml          # SAM deployment defaults
└── README.md
```

---

## Usage

### Manual benchmark runs

Run these any time outside the automatic schedule. Run all three on the same calendar day so the dashboard can compare them side by side.

```bash
# 1. Parallel run (~3–5 seconds)
aws lambda invoke \
  --function-name AnomalyDetectorCoordinator \
  --region us-east-1 \
  --payload file://aws-config/empty.json \
  parallel_response.json

# 2. Sequential benchmark (~75–90 seconds)
aws lambda invoke \
  --function-name AnomalyDetectorSequentialRunner \
  --region us-east-1 \
  --payload file://aws-config/empty.json \
  --cli-read-timeout 600 \
  sequential_response.json

# 3. Anomaly aggregation (~5–8 minutes)
aws lambda invoke \
  --function-name AnomalyDetectorAggregator \
  --region us-east-1 \
  --payload file://aws-config/empty.json \
  --cli-read-timeout 600 \
  agg_response.json

# 4. Single-worker benchmark (~90–110 seconds)
aws lambda invoke \
  --function-name AnomalyDetectorWorkerSingle \
  --region us-east-1 \
  --payload file://aws-config/empty.json \
  --cli-read-timeout 600 \
  single_response.json
```

### View logs

```bash
# Stream logs for any function
aws logs tail /aws/lambda/AnomalyDetectorCoordinator --region us-east-1
aws logs tail /aws/lambda/AnomalyDetectorSequentialRunner --region us-east-1
aws logs tail /aws/lambda/AnomalyDetectorAggregator --region us-east-1
aws logs tail /aws/lambda/AnomalyDetectorWorkerForecast --region us-east-1
aws logs tail /aws/lambda/AnomalyDetectorWorkerArchive --region us-east-1
aws logs tail /aws/lambda/AnomalyDetectorWorkerNWS --region us-east-1
aws logs tail /aws/lambda/AnomalyDetectorWorkerSingle --region us-east-1
```

### Redeploy after code changes

```bash
# Lambda functions
sam build
sam deploy

# Dashboard
cd dashboard && npm run build
aws s3 sync dist/ s3://anomaly-detector-dashboard --acl public-read --region us-east-1
```

### Verify S3 output files

```bash
aws s3 ls s3://anomaly-detector-dashboard/ --recursive --region us-east-1
```

Expected output after a full run:

```
results/YYYY-MM-DD.json # anomaly results — read by Anomalies tab
results/YYYY-MM-DD_sources.json # per-source records — read by Workers tab
timing/YYYY-MM-DD_parallel.json # parallel timing — read by Benchmark tab
timing/YYYY-MM-DD_sequential.json # sequential timing — read by Benchmark tab
timing/YYYY-MM-DD_single.json # single worker timing — read by Single Worker tab
```

---

## Schedule

All times Eastern. EventBridge uses UTC — the cron expressions are set accordingly.

| Time (ET) | UTC | Function | Purpose |
|-----------|-----|----------|---------|
| 8:00 AM | 12:00 | Coordinator | Daily parallel run |
| 8:15 AM | 12:15 | Sequential Runner | Daily sequential benchmark |
| 8:30 AM | 12:30 | Aggregator | Generate anomaly results + sources files |
| 9:00 AM | 13:00 | Single Worker | Daily single-worker benchmark |

---

## Troubleshooting

### Benchmark tab shows no data

Both `timing/{date}_parallel.json` and `timing/{date}_sequential.json` must exist in S3.

```bash
# Check what timing files exist
aws s3 ls s3://anomaly-detector-dashboard/timing/ --region us-east-1

# If parallel file is missing, trigger the coordinator
aws lambda invoke --function-name AnomalyDetectorCoordinator \
  --region us-east-1 --payload file://aws-config/empty.json response.json

# If sequential file is missing, trigger the sequential runner
aws lambda invoke --function-name AnomalyDetectorSequentialRunner \
  --region us-east-1 --payload file://aws-config/empty.json \
  --cli-read-timeout 600 response.json
```

### Anomalies tab shows no data

```bash
# Trigger the aggregator manually
aws lambda invoke --function-name AnomalyDetectorAggregator \
  --region us-east-1 --payload file://aws-config/empty.json \
  --cli-read-timeout 600 agg_response.json
```

### AccessDenied on Lambda invoke or S3 write

Add the missing policy to the function's `Policies` block in `template.yaml` and redeploy.

```yaml
# Common missing policies:
- S3CrudPolicy:
    BucketName: anomaly-detector-dashboard
- DynamoDBCrudPolicy:
    TableName: Anomaly_Detector_Results
- LambdaInvokePolicy:
    FunctionName: AnomalyDetectorWorkerForecast
```

```bash
sam build
sam deploy
```

### ROLLBACK_COMPLETE on sam deploy

The stack failed a previous deploy and is locked. Delete it and redeploy.

```bash
aws cloudformation delete-stack \
  --stack-name Anomaly-Detector \
  --region us-east-1

# Wait ~2 minutes then:
sam build
sam deploy
```

> Deleting the stack does **not** delete DynamoDB tables or the S3 bucket. All data is preserved.

### Sequential runner times out in the terminal

The Lambda runs fine — the CLI just gives up waiting after 60 seconds. Add `--cli-read-timeout 600`.

```bash
aws lambda invoke \
  --function-name AnomalyDetectorSequentialRunner \
  --region us-east-1 \
  --payload file://aws-config/empty.json \
  --cli-read-timeout 600 \    # <-- this is required
  sequential_response.json
```

---

## Teardown

```bash
# Disable scheduled runs without deleting anything
aws events list-rules --region us-east-1   # find your rule names
aws events disable-rule --name "RULE_NAME" --region us-east-1

# Full teardown — remove all AWS resources
aws cloudformation delete-stack --stack-name Anomaly-Detector --region us-east-1
aws s3 rm s3://anomaly-detector-dashboard --recursive --region us-east-1
aws s3 rb s3://anomaly-detector-dashboard --region us-east-1
aws dynamodb delete-table --table-name Anomaly_Detector_Results --region us-east-1
aws dynamodb delete-table --table-name AD_Time_Results --region us-east-1
```

---

## Data Sources

All three APIs are **free with no API key required**.

| API | Endpoint | Data |
|-----|----------|------|
| Open-Meteo Forecast | `api.open-meteo.com/v1/forecast` | Current temp, rain, wind |
| Open-Meteo Archive | `archive-api.open-meteo.com/v1/archive` | 5-year historical averages |
| National Weather Service | `api.weather.gov` | Official observations and alerts |
| Overpass (city discovery) | `overpass-api.de/api/interpreter` | City coordinates within 50km of Atlanta |

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Serverless compute | AWS Lambda (Python 3.11) |
| Database | Amazon DynamoDB |
| Object storage | Amazon S3 |
| Scheduler | Amazon EventBridge |
| Deployment | AWS SAM CLI + CloudFormation |
| Frontend | React 19 + Vite |
| HTTP client | Axios |
| Font | Geist (Vercel) |

---

## License

This project is licensed under the MIT License. See [LICENSE](LICENSE) for details.
