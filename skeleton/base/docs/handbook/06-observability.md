{%- set is_express = values.state_machine_type == 'EXPRESS' -%}
# Observability

## Execution history

{%- if is_express %}

This is an **Express** workflow. Step Functions does not keep execution history
for Express workflows: `DescribeExecution` and `GetExecutionHistory` do not
work for Express executions, and the console can only show them by reading the
execution log group. That log group, described below, is the only record of
what each execution did.
{%- if not values.log_level %}

Logging is **off** in this project, so executions leave no record beyond
aggregate metrics{% if values.enable_tracing %} and X-Ray traces{% endif %}. Turn logging on before relying
on this state machine in production; see
[Common tasks](07-common-tasks.md#change-logging-or-tracing).
{%- endif %}
{%- else %}

This is a **Standard** workflow. Step Functions keeps each execution's full
event history for 90 days after it closes. In the Step Functions console,
open the state machine to list executions, then open one to see its graph,
each state's input and output, and the error and cause of any failure. The
same data is available with `aws stepfunctions describe-execution` and
`aws stepfunctions get-execution-history`.

Execution history is available whether or not logging is on. Logging copies
selected events to CloudWatch Logs, where they can be searched, retained longer,
and shipped to Datadog.
{%- endif %}

## State machine logging
{%- if values.log_level %}

`${{ values.state_machine_name }}StateMachineLogGroup` is named
`/aws/vendedlogs/states/<stack-name>-${{ values.state_machine_name }}`. The
`/aws/vendedlogs/` prefix avoids CloudWatch Logs resource policy size limits.
Retention comes from the `StateMachineLogRetentionInDays` parameter, which
defaults to **30** days.

The log level is `${{ values.log_level }}`:

| Level | Events logged |
| --- | --- |
| `ALL` | Every history event: execution and state starts and ends, and each service call. |
| `ERROR` | Error events, including failed tasks and states, and execution failures, aborts, and timeouts. |
| `FATAL` | Only execution failures, aborts, and timeouts. |

{%- if values.log_level != 'ALL' %}

Successful executions{% if values.log_level == 'FATAL' %}, and executions that recovered from a failed state,{% endif %}
produce no log entries at this level.
{%- if is_express %} For an Express workflow,
that means there is no record of them at all; use `ALL` while developing.
{%- endif %}
{%- endif %}

`IncludeExecutionData` is **true**, so logged events include execution and
state input and output. That data can contain personal information or
secrets passed through the workflow. Review what the workflow carries
**before production use**, and set `IncludeExecutionData: false` in
`template.yaml` if logs must not contain payloads. Never pass tokens or
credentials through execution input.

Each log entry is JSON with fields such as `execution_arn`, `type` (the
history event type), `id`, `previous_event_id`, and `details`. Search by
execution ARN to follow one execution.
{%- else %}

Logging is off: the template defines no `Logging` configuration, log group,
or log-delivery permissions. To turn it on, see
[Common tasks](07-common-tasks.md#change-logging-or-tracing).
{%- endif %}

## Tracing
{%- if values.enable_tracing %}

X-Ray tracing is on. Each execution produces a trace with a segment per state
and per service call, visible in the X-Ray and CloudWatch trace views. The
state machine role has `AWSXrayWriteOnlyAccess` for this.
{%- if values.is_api %} API Gateway
tracing is not enabled on `RestApi`, so traces start at Step Functions rather
than at the API request.
{%- endif %} Lambda functions you add join the trace only if their own
tracing is enabled.
{%- else %}

X-Ray tracing is off. To turn it on, see
[Common tasks](07-common-tasks.md#change-logging-or-tracing).
{%- endif %}
{%- if values.is_api %}

## API Gateway logging

`RestApiLogGroup` creates `/aws/api-gateway/<stack-name>` with **seven-day**
retention. Access logs include extended request ID, source IP, caller/user,
request time, HTTP method/resource path/protocol, status, response length, and
request ID.

Method settings enable INFO execution logging for all paths and methods,
enable detailed metrics, and set `DataTraceEnabled: false` so full request and
response payload tracing is not enabled by the template. Execution logs are
separate from the configured access log group.

API Gateway needs the account/region-level CloudWatch logging role configured
for REST APIs. This template does not define `AWS::ApiGateway::Account` or that
logging role; `RestApiIamRole` only starts executions and is **not** a
CloudWatch logging role. Confirm the platform has configured API logging.

Use access logs for route/status-level investigation and execution logs or
history for workflow behavior.
{%- if is_express %} A `500` from the start endpoint usually means
the execution failed; its error name and cause are in the response body.
{%- else %} The `202` response's `executionArn` links an API request to its
execution; record it in client logs if you need to correlate them.
{%- endif %}
{%- endif %}

## Metrics

Step Functions publishes execution metrics in the `AWS/States` namespace with a
`StateMachineArn` dimension: `ExecutionsStarted`, `ExecutionsSucceeded`,
`ExecutionsFailed`, `ExecutionsAborted`, `ExecutionsTimedOut`,
`ExecutionThrottled`, and `ExecutionTime`. Service integrations and Lambda
calls publish their own failure, timeout, and duration metrics.
{%- if values.is_api %}
API Gateway metrics include Count, Latency, IntegrationLatency, 4XXError, and
5XXError; the template enables detailed method metrics.
{%- endif %}

Filter by the deployed state machine ARN{% if values.is_api %} and API stage{% endif %}, especially when
investigating branch deployments. Watch failed, timed-out, and throttled
executions alongside duration, not just start counts.
{%- if values.trigger_type %} A trigger does not
report failed executions anywhere else, so `ExecutionsFailed` is the main
signal that scheduled or event-driven work is not completing.
{%- endif %}

The starter does not configure alarms or dashboards. Add these intentionally
based on your service's reliability targets.

## Availability in Datadog

The Datadog AWS integration and log shipping are **configured automatically in
all AWS accounts**. Developers do not need to enable the integration or configure
log forwarding for generated services. Use Datadog to inspect the service's
Step Functions{% if values.is_api %} and API Gateway{% endif %} metrics and shipped logs.

Baseline tagging is also automatic: `cfn-tags.json` supplies `org:domain`,
`org:system`, and `org:component`, and `ServerlessOpsIO/gha-deploy-aws-sam` adds
additional deployment tags. Use the existing tags to locate and distinguish
resources; extra tagging is only needed for service-specific requirements.

After deployment, verify that a test execution's{% if values.log_level %} log entries and{% endif %}
metrics appear in Datadog.

In Datadog Logs Explorer, locate the deployed state machine or log-group
identifier and narrow the time range, then search the execution ARN from a
failure. Use Metrics Explorer to inspect the imported AWS metrics and
account/region/state machine dimensions. Exact field names and tags depend on
the integration's pipelines. If data is missing, first confirm it exists in
CloudWatch and check the Datadog time range and resource/tag filters. Treat
persistent missing data as a problem with the existing collection or shipping
configuration, not a requirement to set up a new integration.

## Execution investigation checklist

{%- if values.is_api %}

1. Note the timestamp, HTTP method, status, and any returned request ID or
   execution ARN.
2. Find the API Gateway access entry and determine whether the failure happened
   during authorization/validation, when starting the execution, or in the
   execution itself.
{%- else %}

1. Note the timestamp and, if known, the execution ARN or the triggering
   {% if values.trigger_type == 'eventbridge' %}event's ID{% elif values.trigger_type == 'schedule' %}scheduled time{% else %}caller{% endif %}.
2. Confirm an execution started. If none did, check the
   {% if values.trigger_type == 'eventbridge' %}rule's `FailedInvocations` and `TriggeredRules` metrics and the event pattern{% elif values.trigger_type == 'schedule' %}schedule's state and its invocation metrics{% else %}caller's error and IAM permissions{% endif %}.
{%- endif %}
3. Find the execution in {% if is_express %}the execution logs{% else %}the console or execution history{% endif %} and identify
   the first failed state, its error, and its cause, without copying sensitive
   payloads into tickets.
4. Check the failed state's service: its permissions in the state machine role,
   its own logs, and its limits.
5. Compare failure, timeout, and throttling metrics with recent deployments.
6. Check Datadog ingestion separately if CloudWatch data is present but missing
   from Datadog.
