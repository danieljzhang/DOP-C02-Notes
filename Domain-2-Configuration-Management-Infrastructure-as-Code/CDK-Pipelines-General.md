# CDK Pipelines - DOP-C02 Exam Notes

> CDK-General.md covers CDK fundamentals. This file focuses specifically on **CDK Pipelines** — the self-mutating pipeline construct — which is a distinct exam topic.

## 1. Overview

**What it is:** A high-level construct (`aws-cdk-lib/pipelines`) that builds a CodePipeline pipeline from your CDK app definition. The defining feature: the pipeline **updates itself** before deploying your application — if you change the pipeline structure in code, the next run applies those changes automatically.

**What problem it solves:** Manually keeping a CodePipeline in sync with your CDK app is error-prone. CDK Pipelines makes the pipeline itself part of the CDK app — infrastructure and pipeline are versioned together.

---

## 2. Self-Mutation — The Key Concept

```
Git push to main branch
    ↓
CodePipeline triggers (Source stage)
    ↓
Synth stage: cdk synth → produces cloud assembly
    ↓
UpdatePipeline stage: deploys the pipeline stack itself
    ↑ if the pipeline changed, it restarts here with the new definition
    ↓
Application stages: deploy your stacks (dev → staging → prod)
```

The `UpdatePipeline` stage runs `cdk deploy` on the pipeline stack before touching your application. This means:
- Add a new stage in code → next run adds it to the pipeline automatically
- Change a build step → next run uses the new step
- No manual console changes needed

---

## 3. Minimal Pipeline Example

```typescript
import { Stack, StackProps } from 'aws-cdk-lib';
import { CodePipeline, CodePipelineSource, ShellStep } from 'aws-cdk-lib/pipelines';
import { Construct } from 'constructs';

export class PipelineStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    const pipeline = new CodePipeline(this, 'Pipeline', {
      synth: new ShellStep('Synth', {
        input: CodePipelineSource.gitHub('my-org/my-repo', 'main', {
          authentication: SecretValue.secretsManager('github-token')
        }),
        commands: ['npm ci', 'npm run build', 'npx cdk synth']
      })
    });

    // Add application stage
    pipeline.addStage(new MyAppStage(this, 'Prod', {
      env: { account: '123456789012', region: 'us-east-1' }
    }));
  }
}
```

---

## 4. Stages and Wave Deployments

A **Stage** is a CDK construct containing one or more stacks deployed together. Stages are added to the pipeline in order.

```typescript
// Sequential stages (dev → staging → prod)
pipeline.addStage(new MyAppStage(this, 'Dev', { env: devEnv }));
pipeline.addStage(new MyAppStage(this, 'Staging', { env: stagingEnv }));
pipeline.addStage(new MyAppStage(this, 'Prod', { env: prodEnv }));

// Wave: deploy to multiple regions in parallel
const wave = pipeline.addWave('MultiRegion');
wave.addStage(new MyAppStage(this, 'UsEast1', { env: { region: 'us-east-1' } }));
wave.addStage(new MyAppStage(this, 'EuWest1', { env: { region: 'eu-west-1' } }));
// Both regions deploy simultaneously; pipeline waits for both before continuing
```

---

## 5. Pre/Post Steps — Validation Gates

Add validation steps before or after a stage:

```typescript
pipeline.addStage(new MyAppStage(this, 'Prod', { env: prodEnv }), {
  pre: [
    new ShellStep('RunUnitTests', {
      commands: ['npm test']
    }),
    new ManualApprovalStep('ApproveProduction')  // human gate
  ],
  post: [
    new ShellStep('SmokeTest', {
      commands: ['curl -f https://api.example.com/health']
    })
  ]
});
```

**Exam angle:** Pre-steps run before the stage deploys; post-steps run after. A failed pre-step blocks deployment. A failed post-step can trigger rollback.

---

## 6. Cross-Account Deployments

CDK Pipelines handles cross-account deployments natively. Each stage can target a different account:

```typescript
pipeline.addStage(new MyAppStage(this, 'Prod', {
  env: { account: '999999999999', region: 'us-east-1' }
}));
```

Requirements:
- **Bootstrap** each target account/region with `cdk bootstrap --trust <pipeline-account-id>`
- The bootstrap creates a `cdk-hnb659fds-cfn-exec-role` and `cdk-hnb659fds-deploy-role` in the target account
- The pipeline assumes these roles via cross-account trust

```bash
# Bootstrap target account (run once per account/region)
cdk bootstrap aws://999999999999/us-east-1 \
  --trust 111111111111 \
  --cloudformation-execution-policies arn:aws:iam::aws:policy/AdministratorAccess
```

---

## 7. Source Options

```typescript
// GitHub (via CodeStar connection — recommended, no stored token)
CodePipelineSource.connection('my-org/my-repo', 'main', {
  connectionArn: 'arn:aws:codestar-connections:us-east-1:123456789012:connection/xxx'
})

// CodeCommit
CodePipelineSource.codeCommit(repo, 'main')

// S3
CodePipelineSource.s3(bucket, 'path/to/source.zip')

// ECR image
CodePipelineSource.ecr(repository, { imageTag: 'latest' })
```

---

## 8. Docker in the Synth Step

If your CDK app uses Docker assets (Lambda container images, custom resources), the synth step needs Docker:

```typescript
const pipeline = new CodePipeline(this, 'Pipeline', {
  dockerEnabledForSynth: true,  // enables Docker in the synth CodeBuild project
  synth: new ShellStep('Synth', {
    commands: ['npm ci', 'npx cdk synth']
  })
});
```

---

## 9. CDK Pipelines vs Raw CodePipeline

| | CDK Pipelines | Raw CodePipeline (in CDK) |
|---|---|---|
| **Self-mutation** | ✅ Built-in | ❌ Manual |
| **Cross-account bootstrap** | ✅ Handled | Manual role setup |
| **Abstraction level** | High (opinionated) | Low (full control) |
| **Flexibility** | Less (follows CDK Pipelines model) | Full |
| **Exam scenario** | "IaC-native CI/CD, self-updating pipeline" | "Custom pipeline with specific action types" |

---

## 10. Common Exam Scenarios

### Scenario 1: Pipeline that updates itself when pipeline code changes
**Solution:** CDK Pipelines — the `UpdatePipeline` stage is automatic. No manual intervention needed when you add stages or change build steps.

### Scenario 2: Deploy to dev, then staging, then prod with manual approval
**Solution:** CDK Pipelines with three stages. Add `ManualApprovalStep` as a pre-step on the prod stage.

### Scenario 3: Deploy to 3 regions simultaneously
**Solution:** CDK Pipelines wave — `pipeline.addWave()` with one stage per region. All three deploy in parallel; pipeline waits for all to succeed.

### Scenario 4: Cross-account deployment from a central pipeline account
**Solution:** CDK Pipelines with target stages pointing to different account/region envs. Bootstrap each target account with `--trust <pipeline-account>`.

---

## 11. Exam Tips

- **Self-mutation** is the defining CDK Pipelines feature — the pipeline deploys itself before your app.
- `cdk bootstrap --trust` is required for cross-account deployments — forgetting this is a common trap.
- Waves = **parallel** stage execution; sequential `addStage` calls = **serial**.
- `ManualApprovalStep` is a pre/post step, not a separate stage.
- CDK Pipelines uses CodePipeline under the hood — CloudWatch, CloudTrail, and CodeBuild all apply.
- The synth step produces a **cloud assembly** — a directory of CloudFormation templates and assets.

---

*Added during the 2026-10 Amazon Q review. Facts cross-referenced against AWS documentation.*
