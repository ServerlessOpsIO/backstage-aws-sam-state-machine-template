{%- set is_express = values.state_machine_type == 'EXPRESS' -%}
{%- set name = values.state_machine_name -%}
# Common changes and tasks

## Set up a development environment

Install the AWS SAM CLI and the AWS CLI. The
[AWS Toolkit for VS Code](https://docs.aws.amazon.com/toolkit-for-vscode/latest/userguide/welcome.html)
is optional and adds Workflow Studio for editing the definition visually.
From the **generated repository root**, run:

```sh
sam validate --lint
sam build
```

These are the same checks the pipeline runs. `sam validate` and `sam build`
need the deployment tooling and relevant AWS configuration. Neither runs the
state machine; see [Test changes](#test-changes).

## Replace the starter workflow

The starter `ProcessInput` and `Done` states do nothing. Before adding states:

1. Decide the workflow's input shape and what a successful output looks like.
{%- if values.is_api %}
   {% if is_express %}The output is the API response body, so treat it as part of the API contract.{% else %}Callers only receive the execution ARN, so decide how they learn the outcome.{% endif %}
{%- endif %}
2. List the steps and which AWS service each one calls. Prefer calling services
   directly from the workflow over adding Lambda functions.
3. Decide which failures each step should retry, which should end the
   execution, and which should be handled by another state.
4. Make each step safe to run more than once for the same input{% if not values.is_api and (is_express or values.trigger_type == 'eventbridge') %}:
   {% if values.trigger_type == 'eventbridge' %}EventBridge delivers events at least once{% if is_express %}, and asynchronous Express executions run at least once{% endif %}{% else %}asynchronous Express executions run at least once{% endif %}{% endif %}.

Then edit `statemachine/${{ name }}.asl.yaml`. Keep `StartAt` pointing at the
first state and make every path end in a `Succeed` or `Fail` state or a state
with `End: true`. Update the README's state machine section and this handbook
when the workflow's purpose changes.

## Call an AWS service

Task states call AWS services in two ways:

- **Optimized integrations**, such as `arn:aws:states:::dynamodb:putItem`,
  `arn:aws:states:::sns:publish`, `arn:aws:states:::sqs:sendMessage`, and
  `arn:aws:states:::lambda:invoke`, have Step Functions-specific behavior.
- **AWS SDK integrations**, `arn:aws:states:::aws-sdk:<service>:<apiAction>`,
  call almost any AWS API action, for example
  `arn:aws:states:::aws-sdk:s3:getObject`.

See [the integration list](https://docs.aws.amazon.com/step-functions/latest/dg/integrate-services.html).
{%- if is_express %} Express workflows support the request-response pattern
only; `.sync` and `.waitForTaskToken` resources are rejected.
{%- endif %}

For example, to write to a DynamoDB table this stack defines as `OrdersTable`:

1. Pass the table's name into the definition with a substitution in
   `template.yaml`:

   ```yaml
   DefinitionSubstitutions:
     # ...existing entries...
     OrdersTableName: !Ref OrdersTable
   ```

2. Grant the state machine permission in the same resource.
{%- if values.log_level or values.enable_tracing %} Add to the existing
   `Policies` list:
{%- else %} Uncomment
   `Policies:` and replace the commented example:
{%- endif %}

   ```yaml
   Policies:
     - DynamoDBWritePolicy:
         TableName: !Ref OrdersTable
   ```

3. Add a Task state to the definition and point the previous state's `Next` at
   it:

   ```yaml
   {% raw %}SaveOrder:
     Type: Task
     Resource: arn:aws:states:::dynamodb:putItem
     Arguments:
       TableName: ${OrdersTableName}
       Item:
         pk:
           S: '{% "order#" & $states.input.orderId %}'
         sk:
           S: '{% "order#" & $states.input.orderId %}'
     Output: '{% $states.input %}'
     Next: Done{% endraw %}
   ```

Use [SAM policy templates](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-policy-templates.html)
such as `SNSPublishMessagePolicy`, `SQSSendMessagePolicy`,
`EventBridgePutEventsPolicy`, and `LambdaInvokePolicy` where they fit; write an
IAM statement scoped to specific resources where they do not. AWS SDK
integrations need the IAM action for the API they call.

## Handle errors and retries

Add `Retry` for transient failures and `Catch` to route other failures to a
state that records or compensates for them:

```yaml
{% raw %}SaveOrder:
  # ...Type, Resource, Arguments as above...
  Retry:
    - ErrorEquals: [States.TaskFailed]
      IntervalSeconds: 2
      MaxAttempts: 3
      BackoffRate: 2
  Catch:
    - ErrorEquals: [States.ALL]
      Output: '{% $states.errorOutput %}'
      Next: SaveFailed
  Next: Done

SaveFailed:
  Type: Fail
  Error: SaveFailed
  Cause: Could not save the order{% endraw %}
```

Do not retry errors that cannot succeed on a retry, such as validation or
conditional-check failures. A `Fail` state ends the execution as failed, which
appears in the `ExecutionsFailed` metric{% if values.log_level %} and the execution log{% endif %}{% if is_express and values.is_api %} and returns `500` from the API{% endif %}.

## Add a Lambda function

Use a function when no service integration or JSONata expression can do the
work. This project has no application code, dependency management, or tests;
adding a function means adding them.

1. Add the code, for example under `src/handlers/<FunctionName>/`, and an
   `AWS::Serverless::Function` to `template.yaml`:

   ```yaml
   ProcessOrderFunction:
     Type: AWS::Serverless::Function
     Properties:
       CodeUri: src/handlers/ProcessOrder/
       Handler: function.handler
       Runtime: python3.13
       Timeout: 10
   ```

2. Pass its ARN into the definition and grant the state machine permission to
   invoke it:

   ```yaml
   DefinitionSubstitutions:
     # ...existing entries...
     ProcessOrderFunctionArn: !GetAtt ProcessOrderFunction.Arn
   Policies:
     - LambdaInvokePolicy:
         FunctionName: !Ref ProcessOrderFunction
   ```

3. Call it from a Task state:

   ```yaml
   {% raw %}ProcessOrder:
     Type: Task
     Resource: arn:aws:states:::lambda:invoke
     Arguments:
       FunctionName: ${ProcessOrderFunctionArn}
       Payload: '{% $states.input %}'
     Output: '{% $states.result.Payload %}'
     Next: Done{% endraw %}
   ```

4. Add the language toolchain, unit tests, and a test step to both workflows
   before the SAM validation step. `sam build` packages functions declared in
   the template, but the pipeline does not test them. The Python serverless
   templates in Backstage show one way to lay out functions and tests.

Give each function its own execution role permissions; the state machine's
`Policies` do not apply to functions.
{%- if values.is_api %}

## Change the endpoint

The endpoint is defined in `openapi.yaml` under
`${{ values.url_path_prefix }}/${{ values.resource_name }}`.

- **Path or method.** Change the path key or the operation key
  (`${{ values.http_method | lower }}`). Update the `ApiEndpoint` output in
  `template.yaml`, `README.md`, and this handbook. Methods with a body need a
  `requestBody`; methods without one should not declare it. Keep the
  integration's own `httpMethod: POST`.
- **Scope.** Change the `security` entry, then configure the scope on the
  Cognito resource server and app client.
- **Input from the request.** The request template builds the execution input.
  To pass a query string parameter instead of, or as well as, the body, declare
  the parameter on the operation with `in: query` and `required: true` so the
  validator enforces it, then build the input in the template. For a simple
  value:

  ```yaml
  "application/json":
    Fn::Sub: |
      {
        "input": "{\"orderId\": \"$util.escapeJavaScript($input.params('orderId'))\"}",
        "stateMachineArn": "${${{ name }}StateMachine}"
      }
  ```

  Values that may contain quotes or backslashes need more careful escaping.
  Request templates are written in Velocity (VTL); see
  [API Gateway mapping templates](https://docs.aws.amazon.com/apigateway/latest/developerguide/models-mappings.html).
- **Response.** Edit the integration's `responses` templates and the operation's
  `responses` declarations together.
{%- if not is_express %}
- **Execution status.** To let callers check an execution, add a `GET`
  operation, for example `${{ values.url_path_prefix }}/${{ values.resource_name }}/{executionId}`,
  with an `aws` integration calling `states:action/DescribeExecution`. Build
  the execution ARN from the path parameter and the state machine's name in
  the request template, add `states:DescribeExecution` on this state machine's
  executions to `RestApiIamRole`, map `status`, `output`, `error`, and `cause` in
  the response template, and require a read scope. Anyone with that scope can
  read any execution's output, so add ownership checks if outputs are private.
{%- endif %}

To add an endpoint that starts a different workflow, add another state machine
resource, its own role or a statement in `RestApiIamRole`, and a new operation
copied from the existing one. After a safely isolated deployment, call each
endpoint with valid and invalid tokens and verify the documented status and
body.
{%- else %}

## Change the trigger

Triggers are entries under the state machine's `Events` property in
`template.yaml`.
{%- if values.trigger_type == 'schedule' %} This project has a `Schedule` event.
{%- elif values.trigger_type == 'eventbridge' %} This project has an
`EventBridgeRule` event.
{%- else %} This project has none.
{%- endif %} SAM creates the
schedule or rule and the IAM role that starts executions; do not add those
separately.

A schedule, with an optional time zone and fixed input:

```yaml
Events:
  Schedule:
    Type: ScheduleV2
    Properties:
      ScheduleExpression: cron(0 6 * * ? *)
      ScheduleExpressionTimezone: America/New_York
      Input: '{"source": "schedule"}'
```

An EventBridge rule matching several detail types and a field in `detail`:

```yaml
Events:
  EventBridgeRule:
    Type: EventBridgeRule
    Properties:
      Pattern:
        source:
          - com.example.orders
        detail-type:
          - OrderCreated
          - OrderUpdated
        detail:
          status:
            - PAID
```

Both event types also accept `RetryPolicy` and `DeadLetterConfig`, so failed
attempts to start an execution are retried and then kept rather than lost.
Use `EventBusName` on a rule to match a custom bus instead of the default one.
See the `Events` property of
[`AWS::Serverless::StateMachine`](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-resource-statemachine.html)
for other sources, including API and CloudWatch Events.

To remove the trigger, delete `Events`. Update the README's Triggers section and
[Features](01-features.md#trigger) when the trigger changes, and review
[branch deployments](05-ci-cd.md#feature-branch-builds-and-deployments): every
deployed branch stack gets its own copy of the trigger.
{%- endif %}

## Change the workflow type

Changing `Type` between `STANDARD` and `EXPRESS` **replaces** the state machine:
CloudFormation creates a new one with a new ARN and name, then deletes the old
one. Before changing it:

- Check the definition only uses features the new type supports: Express
  workflows run for at most five minutes and do not support `.sync` or
  `.waitForTaskToken` integrations.
- Check for running executions of the old state machine.
- Make sure consumers read the ARN from the SSM parameter or stack output
  rather than a copied value.
- When moving to Express, turn on logging; see the next section.
{%- if values.is_api %}
- Update `openapi.yaml`: the integration `uri` action
  (`StartSyncExecution` for Express, `StartExecution` for Standard), the
  `default` response template and status, and the success response and schema.
  Update the action allowed by `RestApiIamRole` to match. Generating a
  throwaway project with the other type is the quickest way to see every
  difference.
{%- endif %}

## Change logging or tracing
{%- if values.log_level %}

To change the level, edit `Logging.Level` on the state machine. To change
retention, set `StateMachineLogRetentionInDays` in `cfn-parameters.json`. To
keep payloads out of the logs, set `IncludeExecutionData: false`. To turn logging
off, remove the `Logging` block, the log group, the retention parameter, and the
log-delivery policy statement together.
{%- else %}

To turn logging on, add all of the following to `template.yaml`:

- A `StateMachineLogRetentionInDays` parameter of type `Number`, for example
  with `Default: 30`.
- A log group resource:

  ```yaml
  ${{ name }}StateMachineLogGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: !Sub /aws/vendedlogs/states/${AWS::StackName}-${{ name }}
      RetentionInDays: !Ref StateMachineLogRetentionInDays
  ```

- A `Logging` block on the state machine:

  ```yaml
  Logging:
    Level: ERROR
    IncludeExecutionData: true
    Destinations:
      - CloudWatchLogsLogGroup:
          LogGroupArn: !GetAtt ${{ name }}StateMachineLogGroup.Arn
  ```

- A policy statement allowing `logs:CreateLogDelivery`, `logs:GetLogDelivery`,
  `logs:UpdateLogDelivery`, `logs:DeleteLogDelivery`, `logs:ListLogDeliveries`,
  `logs:PutResourcePolicy`, `logs:DescribeResourcePolicies`, and
  `logs:DescribeLogGroups` on `Resource: '*'`. These actions do not support
  resource-level permissions.
{%- endif %}
{%- if values.enable_tracing %}

To turn tracing off, remove `Tracing` and the `AWSXrayWriteOnlyAccess` policy
together.
{%- else %}

To turn tracing on, add `Tracing: {Enabled: true}` to the state machine and the
`AWSXrayWriteOnlyAccess` policy to its `Policies`.
{%- endif %}

## Test changes

Unit tests are not part of this project, so test the workflow against AWS in an
isolated deployment:

- **One state.** The
  [TestState API](https://docs.aws.amazon.com/step-functions/latest/dg/test-state-isolation.html)
  runs a single state's definition with a given input and role, and reports
  its output, error, and data transformations:

  ```sh
  aws stepfunctions test-state \
    --definition file://state.json \
    --role-arn <state-machine-role-arn> \
    --input '{"orderId": "test-1"}' \
    --inspection-level DEBUG
  ```

  `state.json` holds one state as JSON, with substitutions such as
  `${OrdersTableName}` replaced by real values and `"QueryLanguage": "JSONata"`
  set on the state. Task states call the real service, so use disposable
  resources.
- **The whole workflow.** Start an execution on a deployed branch stack with
  `aws stepfunctions start-execution`{% if is_express %} or `start-sync-execution`{% endif %},
  or `sam remote invoke ${{ name }}StateMachine --stack-name <stack-name> --event '<json>'`.
  Exercise success, each `Catch` path, and inputs the workflow should reject.
{%- if values.is_api %}
- **The API.** Call the endpoint with valid and invalid tokens, bodies, and
  content types, and check the documented status and body.
{%- endif %}

Use synthetic data. Never use real customer records or credentials as test
input.

## Validate and deploy changes

Run `sam validate --lint` and `sam build`. Review the definition, IAM changes,
{% if values.is_api %}API contract, {% endif %}and any added code, then commit the
related definition, template, {% if values.is_api %}OpenAPI, {% endif %}and
handbook updates and open a PR under your review policy.
Remember that **pushing a feature branch starts deployment**: complete the
[branch-isolation setup](05-ci-cd.md) before pushing.

After deployment, run a test execution with disposable data, inspect execution
history, logs, and metrics, and obtain human review. Linting cannot prove that
IAM permissions, service integrations, or the workflow's logic work.
