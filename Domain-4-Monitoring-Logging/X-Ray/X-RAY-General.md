# X-Ray — General Notes (DOP-C02)

**What it is:** Distributed tracing — follow one request across microservices and see exactly where time went and what failed. Complements CloudWatch metrics (what) and logs (details) with the *where*.

**Exam-testable facts:**
- **Trace** = one end-to-end request. **Segment** = one service's work within that trace. **Subsegment** = a downstream call inside a segment (DB query, HTTP call, AWS SDK call).
- **Annotations vs metadata** — the classic exam question: **annotations are indexed** key-value pairs (searchable, filterable); **metadata is not indexed** (debugging context only, max 64 KB per segment).
- **Sampling rules:** a *reservoir* (fixed number of traces per second, always kept) plus a *rate* (percentage of the rest). Rules have priorities; tune them to control cost at scale.
- **Service map:** auto-generated topology of services and edges with latency/error/fault stats per node — the first place to look for "which service is slow."
- **Trace header** `X-Amzn-Trace-Id` propagates sampling decisions across service boundaries.
- **Collection:** instrument with the X-Ray SDK; the **daemon** (UDP) or ADOT collector forwards segments. **Lambda needs no daemon** — enable active tracing and grant `AWSXRayWriteOnlyAccess`.
- **X-Ray Insights:** ML-driven anomaly detection over trace data.

**Common traps:**
- "Make this trace field searchable" → annotation, not metadata.
- EC2/ECS/EKS need the daemon running; Lambda does not.
- Sampling happens at the *start* of the request — you can't retroactively trace an untraced one.

**Sources:** https://docs.aws.amazon.com/prescriptive-guidance/latest/implementing-logging-monitoring-cloudwatch/application-tracing-xray.html

---
*Added during the 2026-10-03 review. All prose is original; facts verified against the AWS documentation links in Sources above.*
