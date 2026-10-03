# CloudWatch — General Notes (DOP-C02)

**What it is:** The observability backbone of AWS — metrics, alarms, logs, and dashboards. If you learn one service deeply for Domain 4, make it this one.

**Exam-testable facts:**
- **Metrics:** organized by *namespace* (`AWS/EC2`) with up to 30 *dimensions* per metric. Statistics: Sum, Average, Minimum, Maximum, and percentiles (p50–p99). Standard resolution 60s; high-resolution down to 1s. Retained 15 months.
- **Alarms:** three states — OK, ALARM, INSUFFICIENT_DATA. Metric alarms fire on thresholds; wire SNS actions to *state transitions*, not just ALARM.
- **Composite alarms:** boolean AND/OR over *child alarm states* (not metrics). Pattern: suppress SNS on noisy child alarms, put the single page on the composite — one alert per incident instead of an alert storm.
- **Anomaly detection:** ML builds a baseline from past data (hourly/daily/weekly patterns). `ANOMALY_DETECTION_BAND(m1, N)` draws a band N standard deviations wide; alarm when the metric goes above, below, or either side of the band. Needs roughly two weeks of history for a good model.
- **Metrics Insights:** SQL-like queries across your metric fleet ("top 10 ECS services by CPU") — for fleets where per-resource widgets don't scale.
- **Metric math:** combine metrics inside alarms/dashboards (e.g., error *rate* = errors ÷ requests) without precomputing.
- **Cross-account observability:** a *monitoring account* linked to *source accounts* (via Organizations) can view metrics and create alarms/anomaly detectors on source-account data — no account switching.
- **Logs:** log groups need explicit *retention* (default: never expire = unbounded cost); **Logs Insights** query language; **metric filters** turn log patterns into metrics/alarms while **subscription filters** stream log events to Lambda/Kinesis/Firehose; **Embedded Metric Format (EMF)** emits custom metrics straight from structured logs.
- **EC2 action alarms:** `arn:aws:automate:<region>:ec2:stop|terminate|reboot|recover` — remediate without writing a Lambda.

**Common traps:**
- Composite alarms combine alarm *states*; metric math combines *metrics*. Different tools.
- Anomaly band width is in standard deviations — wider band, fewer alarms.
- INSUFFICIENT_DATA is a state you can (and should) alarm on for dead-man's-switch patterns.

**Sources:** https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Create_Anomaly_Detection_Alarm.html · https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Create_Composite_Alarm.html

---
*Added during the 2026-10-03 review. All prose is original; facts verified against the AWS documentation links in Sources above.*
