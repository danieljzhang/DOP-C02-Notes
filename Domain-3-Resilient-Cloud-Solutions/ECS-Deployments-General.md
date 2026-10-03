# ECS Deployments - DOP-C02 Exam Notes

**What it is:** How new container revisions reach production on ECS — the resilient-deployment core for container workloads.

**Exam-testable facts:**
- **Deployment controllers:** `ECS` (default), `CODE_DEPLOY` (blue/green powered by CodeDeploy), `EXTERNAL` (bring your own controller, e.g., a third-party progressive-delivery tool).
- **ECS-controller strategies:** `ROLLING` (default; governed by `minimumHealthyPercent`/`maximumPercent`), `BLUE_GREEN`, `LINEAR` (equal-percentage traffic steps), `CANARY` (small slice first, then the rest at once).
- **Deployment circuit breaker:** watches a rolling deployment and *automatically rolls back* on repeated task-launch failures — the "self-healing deploy" answer. Can also roll back on CloudWatch alarms.
- **CODE_DEPLOY blue/green:** CodeDeploy stands up a *replacement task set*, validates it against a **test listener**, then shifts production traffic (all-at-once, canary, or linear); AppSpec file plus Lambda lifecycle hooks (`BeforeAllowTraffic`, `AfterAllowTraffic`).
- **Scaling knobs are different things:** *Service Auto Scaling* changes the **task count** (target tracking on CPU/memory/ALB request count); for the EC2 launch type, **capacity providers** with managed scaling change the **instance count**.
- **Fargate note:** with `CODE_DEPLOY` or `EXTERNAL` controllers on Fargate, `minimumHealthyPercent` is ignored.

**Common traps:**
- "ECS deployment failed and rolled back by itself" → circuit breaker.
- Blue/green *with test-listener validation* → CODE_DEPLOY controller, not the ECS rolling strategy.
- Don't confuse task-count scaling (service) with instance-count scaling (capacity provider).

**Sources:** http://docs.aws.amazon.com/AmazonECS/latest/developerguide/service_definition_parameters.html · https://docs.aws.amazon.com/AmazonECS/latest/APIReference/API_DeploymentConfiguration.html · https://docs.aws.amazon.com/whitepapers/latest/introduction-devops-aws/deployment-strategies-matrix.html

---
*Added during the 2026-10-03 review. All prose is original; facts verified against the AWS documentation links in Sources above.*
