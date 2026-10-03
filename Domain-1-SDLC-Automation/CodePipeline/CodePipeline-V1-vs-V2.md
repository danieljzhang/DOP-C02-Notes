# CodePipeline V1 vs V2 - DOP-C02 Exam Notes

**What it is:** CodePipeline has two pipeline *types*. V1 is the original model; V2 adds release-safety and trigger features and bills differently. The type is decided implicitly — putting any V2-only parameter (e.g., a trigger on Git tags) into the pipeline JSON makes it a V2 pipeline, with V2 pricing.

**Exam-testable facts:**
- **Pricing:** V1 = flat monthly fee per *active* pipeline (active = at least one execution that month). V2 = per *action execution minute* — only actions that actually run are billed.
- **V2-only features:** QUEUED and PARALLEL execution modes (V1 supports only SUPERSEDED); pipeline-level variables; triggers with filters on Git tags, pull requests, branches, and file paths; stage conditions; automatic retry of failed stages; rollback for pipeline stages; source revision overrides; the Commands action; entry conditions with a Skip result.
- **Execution modes** (how a pipeline handles overlapping executions): SUPERSEDED (default — a new execution replaces the in-flight one), QUEUED (executions line up and run in order), PARALLEL (executions run concurrently).
- **No in-place upgrade:** a V1 pipeline cannot be converted to V2. Create a new V2 pipeline, validate it, retire the V1.
- **Trigger side effect:** once a trigger configuration is specified, the default change detection for repository/branch commits is disabled.

**Common traps:**
- "Which execution modes require V2?" → QUEUED and PARALLEL.
- Any pipeline filtering on tags/branches/paths is V2 by definition — and billed per action minute.
- Cost angle: V1's flat fee wins for pipelines that run constantly; V2 wins for rarely-run pipelines.

**Sources:** https://docs.aws.amazon.com/codepipeline/latest/userguide/pipeline-types-planning.html · https://docs.aws.amazon.com/codepipeline/latest/APIReference/API_PipelineDeclaration.html

---
*Added during the 2026-10-03 review. All prose is original; facts verified against the AWS documentation links in Sources above.*
