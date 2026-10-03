# AWS CodeArtifact - DOP-C02 Exam Notes

**What it is:** A fully managed artifact repository for language-native package managers (npm, Maven, Gradle, pip/twine, NuGet, and others). The AWS-native answer to self-hosted Nexus/Artifactory, sitting between your developers and CodeBuild/CodePipeline.

**Exam-testable facts:**
- **Two-level model:** a **Domain** is the top-level container — it owns a single KMS key, stores each asset *once* no matter how many repositories reference it (dedupe), and is where org-wide policy lives. **Repositories** are the per-format endpoints clients publish to and resolve from; repositories are polyglot (one repo can hold multiple package types) and cannot be moved between domains.
- **Upstream repositories:** a repository can chain to other CodeArtifact repositories as upstreams, so one package-manager endpoint transparently resolves packages living in several repositories.
- **External connections:** one repository gets a maximum of *one* external connection to a public registry (npmjs, Maven Central, PyPI, NuGet). Public packages fetched through it are cached and retained inside your domain.
- **Package origin controls:** govern whether a given package name may be *published directly* or only *ingested from upstream* — the defense against dependency-confusion attacks.
- **Auth:** short-lived tokens from `get-authorization-token` (12-hour default), wired into npm/pip/mvn/dotnet client config. Resource policies grant cross-account repository access.
- **Events:** package publish/update/delete events flow to EventBridge, so a new artifact version can kick off a CodePipeline execution or a Lambda.

**Common traps:**
- One external connection per *repository*, not per domain.
- Repositories can't move between domains; assets are stored once per *domain*.
- Origin controls are the "prevent dependency confusion" answer.

**Sources:** https://awscli.amazonaws.com/v2/documentation/api/2.1.29/reference/codeartifact/index.html

---
*Added during the 2026-10-03 review. All prose is original; facts verified against the AWS documentation links in Sources above.*
