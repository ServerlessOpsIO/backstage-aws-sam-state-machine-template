# CI/CD pipeline

## Workflow triggers

| Workflow | Automatic trigger | Manual trigger |
| --- | --- | --- |
| `.github/workflows/main.yml` (`Main`) | Push to `main` | `workflow_dispatch` |
| `.github/workflows/branch.yml` (`Branches`) | Push to any branch except `main` | `workflow_dispatch` |

There is no `pull_request` trigger. A push to a feature branch starts its
workflow without needing a PR. Manual dispatch is also supported; select the
intended workflow and ref carefully.

Both workflows run on `ubuntu-latest` and grant jobs `contents: read` and
`id-token: write`. The latter allows OIDC-based AWS role assumption rather
than committing long-lived AWS credentials.

## Build job

The build job executes these steps in order:

1. `ServerlessOpsIO/gha-setup-workspace@v2` sets up the checked-out workspace.
2. `ServerlessOpsIO/gha-assume-aws-credentials@v1` obtains credentials using
   `AWS_CICD_ACCOUNT_ID`.
3. `aws-actions/setup-sam@v3` installs SAM.
4. `sam validate --lint` validates infrastructure with CloudFormation linting.
5. `sam build` builds the deployable application.
6. `ServerlessOpsIO/gha-store-artifacts@v2`, with `use_aws_sam: true`, stores
   the build output for the deploy job.

There is no test step: the project has no application code, and the workflows
do not execute the state machine. Linting checks the template's structure, not
whether the workflow does the right thing. Execution tests, TechDocs
generation, and any language toolchain for Lambda functions you add are not
steps in these workflows. Add them explicitly if your project requires them.

## Deploy job

`deploy` depends on `build` and only proceeds after it succeeds. It uses the
GitHub environment named **`production`** in both workflows.

1. The workspace action runs with `checkout_artifact: true` to retrieve the
   build artifact.
2. The credential action receives both `AWS_CICD_ACCOUNT_ID` and
   `DEPLOYMENT_ACCOUNT_ID` for build/deployment account access.
3. `ServerlessOpsIO/gha-deploy-aws-sam@v1` deploys with the target account ID
   and serialized workflow environment, variables, and secrets.

The shared actions implement artifact handling, credential assumptions, and
parameter/tag expression resolution. Their internals are external to this
repository. Confirm the expected IAM roles, region, artifact storage, and
stack-name behavior against the shared actions' configuration.

See [Configuration](03-configuration.md) for parameter files, tags, secrets,
{% if values.is_api %}DNS/Cognito prerequisites, {% endif %}and SAM defaults. GitHub environment protection
rules can require deployment approval or restrict which branches deploy.

Deploying a new definition does not stop executions that are already running:
{% if values.state_machine_type == 'STANDARD' %}running Standard executions continue
with the definition they started with, which matters for long-running
workflows.{% else %}running Express executions finish with the definition they
started with.{% endif %}

## Feature branch builds and deployments

Branch support means a non-main push executes **both build and deploy**, not
just CI checks. `CodeBranch` is resolved from `$env.GITHUB_REF_SLUG_CS` and
appears in the SSM parameter names{% if values.is_api %} and names the API Gateway stage{% endif %}.
That alone does not isolate an entire deployment:

- `samconfig.toml` has a component-wide default stack name, so a branch can
  update the same stack as `main` and replace its state machine definition.
{%- if values.is_api %}
- `cfn-parameters.json` has one fixed hostname from scaffolding.
{%- endif %}
{%- if values.trigger_type == 'schedule' %}
- Each separately deployed branch stack has its own schedule, so the workflow
  runs once per branch at every scheduled time.
{%- elif values.trigger_type == 'eventbridge' %}
- Each separately deployed branch stack has its own rule on the same default
  event bus, so **every branch's state machine runs for every matching event**,
  including production events in a shared account.
{%- endif %}
- Both workflows select the same GitHub environment, `production`.

Before enabling branch deployments, establish and verify a platform policy:

1. Give each deployed branch a distinct stack name{% if values.is_api %} and hostname{% endif %} through the
   shared deployment action's supported configuration.
{%- if values.is_api %}
2. Ensure the branch hostname belongs to the configured hosted zone and that
   matching Cognito resource-server scopes and app-client permissions exist.
   Scope identifiers depend on the hostname.
{%- elif values.trigger_type %}
2. Decide whether branch stacks should be triggered at all, and disable or
   narrow the {% if values.trigger_type == 'schedule' %}schedule{% else %}rule's event pattern{% endif %}
   for branches if not. Branch executions call the same downstream services as
   `main` unless the definition substitutes branch-specific resources.
{%- else %}
2. Make sure branch executions use branch-specific downstream resources, or
   are only started deliberately, so tests do not change shared data.
{%- endif %}
3. Use an appropriate account/environment and OIDC trust policy for branch
   workloads, with approval rules that prevent unintended production changes.
4. Confirm the branch slug, final stack name, state machine ARN{% if values.is_api %}, and custom domain{% endif %}
   in the deployment output before starting test executions.
5. Define teardown for branch stacks and any external resources when a branch
   is no longer needed. There is no branch-deletion cleanup workflow.

Without this policy, deployments may target the same stack{% if values.is_api %} or collide on custom
domains and DNS records{% endif %}. Do not assume a branch push creates a safe
preview environment out of the box.

## Reviewing a deployment

Inspect the GitHub Actions build and deploy logs, then check the deployed
CloudFormation stack's status, resources, and outputs.
{%- if values.is_api %} Smoke-test `/health`
with an access token containing its scope, then call the start endpoint with
disposable input and confirm the
{% if values.state_machine_type == 'EXPRESS' %}`200` response contains the expected output{% else %}`202` response's execution reaches `SUCCEEDED`{% endif %}.
Health uses a mock integration and does not establish that the state machine
works.
{%- else %} Start an execution with disposable input, using the ARN from the
stack outputs, and confirm it reaches `SUCCEEDED`:

```sh
aws stepfunctions start-execution \
  --state-machine-arn <state-machine-arn> \
  --input '{"hello": "world"}'
```
{%- if values.trigger_type == 'schedule' %}

Confirm the schedule exists and is enabled in EventBridge Scheduler, then check
for an execution after its next scheduled time.
{%- elif values.trigger_type == 'eventbridge' %}

To exercise the rule, put a test event with source
`${{ values.event_pattern_source }}`{% if values.event_pattern_detail_type %} and detail type `${{ values.event_pattern_detail_type }}`{% endif %}
on the default bus with `aws events put-events`, and confirm an execution
starts. Do this only where downstream effects are safe.
{%- endif %}
{%- endif %}
Use [Observability](06-observability.md) to inspect execution history, errors,
and metrics.
