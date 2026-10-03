# Terraform Deep Dive — Workspaces

**What it is:** Workspaces let you manage multiple instances of the *same* Terraform configuration — typically one per environment (dev/stage/prod) — each with its own separate state file, without duplicating code.

**Exam-testable facts:**
- **One config, many states:** `terraform workspace new prod` creates a workspace; `terraform workspace select dev` switches. Each workspace keeps an independent state, so `prod` and `dev` never clobber each other's resources.
- **State separation:** with local state, each workspace gets its own `terraform.tfstate.d/<name>/` file; with remote backends (S3), each workspace gets its own state object/key prefix.
- **`${terraform.workspace}` interpolation:** reference the current workspace name inside configuration — e.g. `name = "app-${terraform.workspace}"` or `count = terraform.workspace == "prod" ? 3 : 1`. This is how one codebase behaves differently per environment.
- **tfvars per workspace:** pair workspaces with per-environment variable files (`terraform apply -var-file=prod.tfvars`) — workspaces separate *state*, tfvars separate *inputs*. The exam likes asking which does which.
- **Not for strong isolation:** workspaces share the same backend credentials and provider configuration. For real blast-radius separation (separate AWS accounts per environment), use separate configurations/backends, not workspaces.

**Common traps:**
- Workspaces ≠ separate AWS accounts. Same backend, same credentials — a `terraform destroy` in the wrong workspace still hurts.
- Switching workspaces does not change variable values — that's the tfvars file's job.
- The `default` workspace always exists and cannot be deleted.

**Sources:** https://developer.hashicorp.com/terraform/language/state/workspaces

*Added during the 2026-10-03 review. All prose is original; facts verified against the documentation link above.*
