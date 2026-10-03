# CloudWatch Deep Dive — MetricsInsights

**What it is:** CloudWatch Metrics Insights is a SQL-like query engine for your metrics. Instead of building one dashboard widget per resource, you write a single query that runs across your whole fleet — "show me the 10 ECS services with the highest CPU right now."

**Exam-testable facts:**
- **SQL dialect:** `SELECT`, `FROM` (namespace + metric name, or `SCHEMA` for dimension filtering), `WHERE`, `GROUP BY`, `ORDER BY`, `LIMIT`. Example shape: `SELECT AVG(CPUUtilization) FROM SCHEMA("AWS/ECS", ClusterName, ServiceName) GROUP BY ClusterName, ServiceName ORDER BY AVG() DESC LIMIT 10`.
- **`SCHEMA()` function:** the key to fleet-wide queries — it selects metrics by their dimensions instead of naming each resource.
- **Alarms on queries:** a Metrics Insights query can back a CloudWatch alarm, so one alarm covers a dynamic fleet (e.g. "alarm if *any* service's error rate exceeds 5%") — no per-service alarms to maintain.
- **Use case signal:** the exam reaches for Metrics Insights when the scenario involves *many* resources and per-resource dashboards/alarms don't scale.
- **Limits:** queries time out after a bounded execution window; results are paginated. It queries *metrics*, not logs (that's Logs Insights).

**Common traps:**
- Metrics Insights (metrics, SQL-like) vs Logs Insights (logs, query language) vs Contributor Insights (top-N contributors via rules). Three different tools — the exam names all three.
- A Metrics Insights *alarm* evaluates the query on a schedule; it does not stream.
- `SCHEMA()` filters by dimensions — you can't query metrics you haven't published dimensions for.

**Sources:** https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/query_with_cloudwatch-metrics-insights.html

*Added during the 2026-10-03 review. All prose is original; facts verified against the AWS documentation link above.*
