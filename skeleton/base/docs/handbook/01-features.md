{%- set is_sync = values.state_machine_type == 'EXPRESS' -%}
{%- set has_body = values.http_method in ['POST', 'PUT', 'PATCH'] -%}
{%- set scope = 'read' if values.http_method == 'GET' else 'write' -%}
# Features

## Step Functions state machine

`template.yaml` declares `${{ values.state_machine_name }}StateMachine`, an
`AWS::Serverless::StateMachine` with `Type: ${{ values.state_machine_type }}`.
Its `DefinitionUri` points to
`statemachine/${{ values.state_machine_name }}.asl.yaml`, so editing that file
changes the deployed workflow. CloudFormation generates the deployed state
machine's physical name; use the stack outputs or SSM parameters described
below rather than guessing it.

The workflow type is fixed when the state machine is created. The two types
behave differently:

| | Standard | Express |
| --- | --- | --- |
| Maximum duration | One year | Five minutes |
| Execution semantics | Exactly once | At least once when started asynchronously; at most once when started synchronously |
| Execution history | Kept by Step Functions for 90 days after the execution closes; visible in the console and API | Not kept by Step Functions; available only through CloudWatch Logs |
| `.sync` (run a job) and `.waitForTaskToken` (callback) integration patterns | Supported | Not supported |
| Billing | Per state transition | Per request, duration, and memory |

This project uses **${{ values.state_machine_type }}**. Changing the type
replaces the state machine; see
[Common tasks](07-common-tasks.md#change-the-workflow-type).

## Amazon States Language definition

The definition is written in [Amazon States Language (ASL)](https://docs.aws.amazon.com/step-functions/latest/dg/concepts-amazon-states-language.html)
as YAML. It sets `QueryLanguage: JSONata`, so states transform data with
[JSONata](https://docs.aws.amazon.com/step-functions/latest/dg/transforming-data.html)
expressions rather than JSONPath fields such as `InputPath` and `ResultPath`.

The starter definition has two states:

| State | Type | Behavior |
| --- | --- | --- |
| `ProcessInput` | `Pass` | Placeholder; passes its input through unchanged. |
| `Done` | `Succeed` | Ends the execution successfully. |

The starter does no work. Replace these states with your workflow before
treating the deployment as a real service.

`DefinitionSubstitutions` in `template.yaml` replaces `${Domain}`, `${System}`,
`${Component}`, and `${CodeBranch}` in the definition with the stack parameter
values at deployment. Add entries there to pass resource names and ARNs into
the definition instead of hard-coding them. See
[Architecture](04-architecture.md#anatomy-of-the-state-machine-definition).

You can edit the definition visually with
[Workflow Studio in the AWS Toolkit for VS Code](https://docs.aws.amazon.com/step-functions/latest/dg/workflow-studio-local.html).

## Starter IAM permissions

SAM creates the state machine's execution role from the `Policies` list on the
state machine resource. The starter grants only what the selected options need:

{%- if values.log_level %}

- CloudWatch Logs delivery actions, so Step Functions can write execution logs.
  These actions do not support resource-level permissions, so the statement uses
  `Resource: '*'`.
{%- endif %}
{%- if values.enable_tracing %}

- `AWSXrayWriteOnlyAccess`, so executions can send X-Ray trace data.
{%- endif %}
{%- if not values.log_level and not values.enable_tracing %}

- No permissions. `Policies` is commented out with a `LambdaInvokePolicy` example.
{%- endif %}

The role grants no access to other AWS services. Each state that calls a
service needs a matching policy; see [Common tasks](07-common-tasks.md#call-an-aws-service).
{%- if values.is_api %}

## OpenAPI and API Gateway

`openapi.yaml` defines the API contract and its API Gateway integrations.
`template.yaml` includes it through `AWS::Include` in `RestApi.DefinitionBody`,
so editing it changes both the published API definition and deployed routes.
The Backstage API entity also references this file.

| Method and path | Implementation |
| --- | --- |
| `GET /health` | API Gateway mock integration; no state machine |
| `${{ values.http_method }} ${{ values.url_path_prefix }}/${{ values.resource_name }}` | Step Functions `${{ 'StartSyncExecution' if is_sync else 'StartExecution' }}` AWS service integration; no Lambda |

API Gateway calls Step Functions directly with the `RestApiIamRole` role, which
only allows `${{ 'states:StartSyncExecution' if is_sync else 'states:StartExecution' }}`
on this state machine. The integration method is `POST` even when the
client-facing method is not, because that is how API Gateway calls the
Step Functions API.
{%- if has_body %}

The JSON request body becomes the execution input. Step Functions limits
execution input to 256 KiB.
{%- else %}

Executions start with an empty object (`{}`) as their input. Query string
parameters are not passed to the state machine unless you change the request
template; see [Common tasks](07-common-tasks.md#change-the-endpoint).
{%- endif %}
The integration sets `passthroughBehavior: never` and only maps
`application/json`, so requests with another `Content-Type` are rejected with
`415 Unsupported Media Type`.
{%- if is_sync %}

Because the state machine is an Express workflow, API Gateway runs it
synchronously:

| Outcome | Response |
| --- | --- |
| Execution succeeds | `200` with the execution output as the body |
| Execution fails or times out | `500` with the error `name` and `cause` as `message` |
| Step Functions rejects the request (`4xx`) | `400` with the error type and message |
| Step Functions fails (`5xx`) | `500` with a generic message |

API Gateway waits at most 29 seconds for an integration by default, so
executions behind this endpoint must finish within that time even though
Express workflows can run for five minutes.
{%- else %}

Because the state machine is a Standard workflow, API Gateway starts it
asynchronously:

| Outcome | Response |
| --- | --- |
| Execution starts | `202` with `executionArn` and `startDate` (epoch seconds) |
| Step Functions rejects the request (`4xx`) | `400` with the error type and message |
| Step Functions fails (`5xx`) | `500` with a generic message |

The response does not include the execution's result. The API has no status
endpoint: callers that need the outcome must call `DescribeExecution` with
their own IAM permissions, or you can add an endpoint for it; see
[Common tasks](07-common-tasks.md#change-the-endpoint).
{%- endif %}

The `all` request validator enables body and parameter validation.
{%- if has_body %} The body
must be a JSON object; its fields are not constrained by the starter schema.
{%- endif %} Response declarations describe the contract; they do not
validate execution output. The file includes CloudFormation `Ref`, `Fn::Sub`,
and `Fn::GetAtt` expressions, so it is a deployment-aware definition rather
than a fully resolved standalone OpenAPI file.

## DNS and TLS

The stack creates a DNS-validated ACM certificate, a regional API Gateway
custom domain, and a Route 53 record for the configured hostname. It obtains
the hosted zone ID through an existing SSM parameter. The hosted zone and SSM
parameter are prerequisites, not resources created by this service.
See [Configuration](03-configuration.md).

## Cognito authentication and authorization

The API uses an existing Cognito User Pool through `CognitoUserPoolArn`; it does
not create a pool, resource server, app client, or users.
The `serverlessOpsCognitoPool` OpenAPI security scheme reads the `Authorization`
header. Both starter operations, including `/health`, have a scope requirement:

| Operation | Required scope |
| --- | --- |
| Health | `https://<hostname>/health.read` |
| Start ${{ values.state_machine_name }} | `https://<hostname>/${{ values.resource_name }}.${{ scope }}` |

The hostname here is the deployed `Hostname` parameter. Configure matching
resource-server scopes and allow them on the Cognito app client. Obtain an
**access token** containing the required scope using the client's configured
OAuth flow, then send it in the `Authorization` header. Machine clients may use
client credentials if enabled; browser clients should use authorization code
with PKCE and must not embed a client secret.

API Gateway verifies the token and operation scope. This does not provide
per-request ownership or tenant authorization: the execution input does not
include the caller's identity, so add those checks deliberately if needed. Do
not commit tokens or app-client secrets.
{%- else %}

## Trigger
{%- if values.trigger_type == 'schedule' %}

The state machine's `Events` property declares a `ScheduleV2` event, so SAM
creates an EventBridge Scheduler schedule with the expression
`${{ values.schedule_expression }}` and an IAM role that lets the scheduler
start executions.

Scheduler evaluates `cron` and `at` expressions in UTC unless the event sets
`ScheduleExpressionTimezone`. The event does not set `Input`, so executions do
not receive details about the schedule; add `Input` to the event's properties
if the workflow needs a fixed input.
{%- elif values.trigger_type == 'eventbridge' %}

The state machine's `Events` property declares an `EventBridgeRule` event, so
SAM creates a rule on the account's **default** event bus and an IAM role that
lets EventBridge start executions. The rule matches events with:

| Field | Value |
| --- | --- |
| `source` | `${{ values.event_pattern_source }}` |
{%- if values.event_pattern_detail_type %}
| `detail-type` | `${{ values.event_pattern_detail_type }}` |
{%- endif %}

Each matching event becomes the execution input, including its envelope fields
such as `source`, `detail-type`, `time`, and `detail`. EventBridge delivers
events at least once, so make the workflow safe to run more than once for the
same event.
{%- else %}

The state machine has no trigger. Start executions from the AWS console, the
AWS CLI, or another service using the state machine ARN. To add a trigger, see
[Common tasks](07-common-tasks.md#change-the-trigger).
{%- endif %}
{%- endif %}

## Logging and tracing
{%- if values.log_level %}

Executions send `${{ values.log_level }}` level history events to the
`${{ values.state_machine_name }}StateMachineLogGroup` CloudWatch Logs log group,
including execution input and output data.
{%- else %}

Execution logging is off: the template defines no `Logging` configuration or
log group.
{%- if is_sync %} Because this is an
Express workflow, executions leave no history anywhere. Turn logging on before
relying on this state machine in production.
{%- endif %}
{%- endif %}
{%- if values.enable_tracing %}
X-Ray tracing is on.
{%- else %}
X-Ray tracing is off.
{%- endif %}
See [Observability](06-observability.md).

## Stack outputs and SSM parameters

The stack publishes the state machine's identifiers so other services and
operators can find them without knowing the generated physical name:

| Value | Stack output | SSM parameter |
| --- | --- | --- |
| State machine ARN | `${{ values.state_machine_name }}StateMachineArn` | `/<Domain>/<System>/<Component>/<CodeBranch>/${{ values.state_machine_name }}StateMachineArn` |
| State machine name | `${{ values.state_machine_name }}StateMachineName` | `/<Domain>/<System>/<Component>/<CodeBranch>/${{ values.state_machine_name }}StateMachineName` |
{%- if values.is_api %}
| Endpoint URL | `ApiEndpoint` | None |
{%- endif %}

For the `main` branch, the ARN parameter is
`/${{ values.domain_name }}/${{ values.system_name }}/<Component>/main/${{ values.state_machine_name }}StateMachineArn`,
where `<Component>` is the repository-name slug the deployment workflow supplies.
Consumers that read the SSM parameter pick up a new ARN after the state machine
is replaced; consumers that copied the ARN do not.

## AWS SAM CLI for validation and builds

This project contains no application code or package manager configuration.
The [AWS SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html)
is the only local tool the pipeline relies on: `sam validate --lint` checks the
template with CloudFormation linting, and `sam build` prepares the deployment
artifact. Run both from the generated repository root before pushing. Add a
language toolchain only if you add Lambda functions; see
[Common tasks](07-common-tasks.md#add-a-lambda-function).

## CI/CD and feature branches

GitHub Actions supplies separate main and non-main branch workflows. Both
validate and build with SAM, and deploy after a successful build.
Branch deployments need deliberate stack{% if values.is_api %} and hostname{% endif %}
isolation; both workflows currently select the `production` GitHub environment.
See [CI/CD pipeline](05-ci-cd.md) before pushing a feature branch.
