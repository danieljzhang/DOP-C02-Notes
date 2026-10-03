# Amazon CloudWatch Synthetics - DOP-C02 Exam Notes

## 1. Overview

**What it is:** Proactive monitoring using *canaries* — configurable scripts that run on a schedule and simulate user interactions with your endpoints. Synthetics finds problems before real users do.

**What problem it solves:** Reactive monitoring (alarms on real traffic metrics) only fires after users are already affected. Synthetics runs synthetic transactions continuously so you detect outages, broken APIs, and slow pages even during low-traffic periods.

---

## 2. Canaries

A **canary** is a script (Node.js or Python) that runs on a schedule in a managed Lambda-like environment. It makes HTTP requests, checks responses, and reports pass/fail to CloudWatch.

### Canary Blueprints (pre-built templates)

| Blueprint | What it does |
|---|---|
| **Heartbeat monitor** | Simple HTTP GET — checks endpoint is reachable and returns 2xx |
| **API canary** | Tests REST API endpoints (GET, POST, etc.) with request/response validation |
| **Broken link checker** | Crawls a page and checks all links return 2xx |
| **Visual monitoring** | Takes screenshots and compares to a baseline (detects UI regressions) |
| **Canary recorder** | Records browser interactions in the console and generates the canary script |
| **GUI workflow builder** | Step-by-step UI interactions (login, form fill, checkout) |

---

## 3. How Canaries Work

```
Schedule (rate/cron)
    ↓
Canary script runs (Node.js / Python)
    ↓
Makes HTTP requests / browser interactions
    ↓
Reports metrics to CloudWatch:
  - SuccessPercent
  - Duration
    ↓
Stores artifacts in S3:
  - Screenshots
  - HAR files (HTTP Archive)
  - Logs
    ↓
CloudWatch Alarm on SuccessPercent < 100%
    ↓
SNS → PagerDuty / Slack / Incident Manager
```

---

## 4. Key Configuration

```bash
# Create a heartbeat canary
aws synthetics create-canary \
  --name my-api-heartbeat \
  --code '{"Handler":"pageLoadBlueprint.handler","S3Bucket":"my-canary-bucket","S3Key":"heartbeat.zip"}' \
  --artifact-s3-location s3://my-canary-artifacts/my-api-heartbeat/ \
  --execution-role-arn arn:aws:iam::123456789012:role/CloudWatchSyntheticsRole \
  --schedule '{"Expression":"rate(5 minutes)"}' \
  --run-config '{"TimeoutInSeconds":60,"MemoryInMB":960}' \
  --runtime-version syn-nodejs-puppeteer-6.2
```

### Runtime versions
- `syn-nodejs-puppeteer-X.X` — Node.js with Puppeteer (headless Chrome) for UI canaries
- `syn-python-selenium-X.X` — Python with Selenium for UI canaries
- `syn-nodejs-2.X` — Node.js for API/heartbeat canaries (no browser)

---

## 5. Canary Groups

Group related canaries together for aggregate monitoring. A group alarm fires if *any* canary in the group fails — useful for "is my entire checkout flow healthy?" composite view.

---

## 6. VPC Support

Canaries can run inside a VPC to test private endpoints (internal APIs, databases via proxy, etc.). The canary Lambda needs:
- VPC configuration (subnet + security group)
- NAT Gateway or VPC endpoint for CloudWatch/S3 access (to report results)

---

## 7. IAM — Execution Role

The canary execution role needs:
```json
{
  "Effect": "Allow",
  "Action": [
    "s3:PutObject",
    "s3:GetBucketLocation",
    "logs:CreateLogGroup",
    "logs:CreateLogStream",
    "logs:PutLogEvents",
    "cloudwatch:PutMetricData",
    "xray:PutTraceSegments"
  ],
  "Resource": "*"
}
```

---

## 8. Metrics and Alarms

CloudWatch Synthetics publishes to the `CloudWatchSynthetics` namespace:

| Metric | Description |
|---|---|
| `SuccessPercent` | % of canary runs that passed in the period |
| `Duration` | Time taken for the canary run (ms) |
| `Failed` | Count of failed runs |

Standard alarm pattern:
```bash
aws cloudwatch put-metric-alarm \
  --alarm-name canary-api-health \
  --namespace CloudWatchSynthetics \
  --metric-name SuccessPercent \
  --dimensions Name=CanaryName,Value=my-api-heartbeat \
  --statistic Average \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 100 \
  --comparison-operator LessThanThreshold \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:ops-alerts
```

---

## 9. Synthetics vs Other Monitoring Tools

| Tool | What it monitors | When it fires |
|---|---|---|
| **CloudWatch Synthetics** | Synthetic (simulated) transactions | Before users are affected |
| **CloudWatch Alarms** | Real traffic metrics (latency, errors) | After users are affected |
| **CloudWatch RUM** | Real user browser sessions | After users are affected |
| **X-Ray** | Distributed traces of real requests | After users are affected |

**Exam angle:** "Detect API outage even when there is no real traffic" → Synthetics. "Monitor real user experience" → RUM.

---

## 10. Common Exam Scenarios

### Scenario 1: Detect API outage during off-peak hours
**Solution:** Heartbeat canary running every 5 minutes. CloudWatch alarm on `SuccessPercent < 100%` triggers SNS. Fires even at 3 AM when real traffic is near zero.

### Scenario 2: Validate deployment didn't break the login flow
**Solution:** GUI workflow canary that simulates login → dashboard → logout. Run it as a post-deployment validation step in CodePipeline. Fail the pipeline if the canary fails.

### Scenario 3: Monitor internal API in a private VPC
**Solution:** Canary with VPC configuration pointing to the private subnet. NAT Gateway allows the canary to reach CloudWatch and S3 for result reporting.

### Scenario 4: Proactive SLA monitoring
**Solution:** API canary measuring `Duration`. Alarm when p95 duration exceeds SLA threshold — catches slowdowns before they breach SLA with real users.

---

## 11. Exam Tips

- Synthetics = **proactive** (synthetic traffic). RUM = **reactive** (real user traffic). CloudWatch Alarms = **reactive** (real metrics). Know which is which.
- Canaries run on a **schedule** — they are not triggered by real traffic.
- Artifacts (screenshots, HAR files) go to **S3** — useful for debugging failures.
- VPC canaries need a **NAT Gateway or VPC endpoints** to report results to CloudWatch/S3.
- "No traffic at night but still need to know if the API is down" → Synthetics canary.
- Canary blueprints are the fast answer — you don't need to write scripts from scratch.

---

*Added during the 2026-10 Amazon Q review. Facts cross-referenced against AWS documentation.*
