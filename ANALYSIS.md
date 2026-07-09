# Analysis of the Broken Deployment Pipeline

The current workflow in `.github/workflows/deployment.yml` is unsafe because it combines checkout, install, build, deploy, and notification into one path and lets deployment happen before validation.

## Missing validation stages

- No dedicated source validation stage beyond checkout.
- No build artifact stage that can be passed to later jobs.
- No lint gate before tests or deployment.
- No unit or integration test gate before deployment.
- No security stage for dependency audit or secret scanning.
- No staging verification stage before production.
- No approval gate for production.
- No rollback stage or rollback trigger after verification failure.

## Incorrect execution order

- The workflow deploys first and validates later, which allows unsafe code to reach production before tests run.
- Lint and test jobs are placed after deploy instead of before it.
- The smoke test step does not block the pipeline because it uses `|| echo`, so failure is ignored.

## Missing safety gates

- The trigger matches all branches, so every branch can reach the deployment path.
- Production is not protected by a separate environment gate with required reviewers.
- There is no `needs` chain to force sequential execution.
- There is no artifact handoff between build and downstream jobs.

## Why failures are hard to isolate

- The broken workflow groups multiple responsibilities into a single deploy job.
- There are no isolated job boundaries for source, build, test, security, deploy, and verify.
- A failing stage does not clearly tell you which validation was responsible because the pipeline keeps going.

## Rollback gaps

- No rollback script is called when verification fails.
- No last-known-good release tag is preserved for recovery.
- No verification step confirms the deployment is healthy before the pipeline is considered successful.

## Target design

- Source -> Build -> Test -> Security -> Deploy Staging -> Deploy Production -> Verify -> Rollback on verify failure.
- Use artifacts from build in the downstream deploy jobs.
- Keep production behind an environment approval gate.
- Make smoke tests fail hard and stop the pipeline.
- Log commit SHA, timestamps, and job status in each stage.
