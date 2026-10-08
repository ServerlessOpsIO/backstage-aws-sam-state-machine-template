{%- set is_sync = values.state_machine_type == 'EXPRESS' -%}
# ${{ values.system_title }} / ${{ values.component_name }}

${{ values.component_description }}

The full API can be found in the [OpenAPI document](./openapi.yaml).


## Architecture
This is an AWS Step Functions state machine that is started through an API. It is built on top of the following AWS services:
* API Gateway
* Step Functions
* Cognito (See _Authentication and Authorization_ for more)

API Gateway starts the state machine directly, without a Lambda function in between.


## New Project Getting Started
This repository was generated from a template intended to get a new AWS Step Functions state machine with an API up and running quickly. It uses the [AWS Serverless Application Model (SAM)](https://aws.amazon.com/serverless/sam/) to define the infrastructure as code.

### Project layout

- [`template.yaml`](template.yaml): AWS SAM template that defines the state machine, the API, and their supporting resources.
- [`openapi.yaml`](openapi.yaml): OpenAPI document that defines the API. The SAM template also uses it to create the API Gateway resources.
- [`statemachine/${{ values.state_machine_name }}.asl.yaml`](statemachine/${{ values.state_machine_name }}.asl.yaml): the state machine definition, written in [Amazon States Language (ASL)](https://docs.aws.amazon.com/step-functions/latest/dg/concepts-amazon-states-language.html).
- [`cfn-parameters.json`](cfn-parameters.json): parameter values used when the stack is deployed.
- [`cfn-tags.json`](cfn-tags.json): tags applied to the deployed stack.
- [`.github/workflows/`](.github/workflows/): GitHub Actions workflows that validate, build, and deploy the stack.

### API

`${{ values.http_method }} https://${{ values.hostname }}${{ values.url_path_prefix }}/${{ values.resource_name }}` starts an execution of the **${{ values.state_machine_name }}** state machine.
{%- if values.http_method in ['POST', 'PUT', 'PATCH'] %}

The JSON request body is passed to the state machine as its input.
{%- else %}

The state machine starts with an empty object (`{}`) as its input. Query string parameters are not passed to the state machine. To pass them, change the request template in the endpoint's `x-amazon-apigateway-integration` section of [`openapi.yaml`](openapi.yaml).
{%- endif %}
{%- if is_sync %}

The state machine is an `EXPRESS` workflow, so API Gateway runs it synchronously:

- If the execution succeeds, the API returns `200` with the state machine output as the response body.
- If the execution fails or times out, the API returns `500` with the error name and cause.

API Gateway waits at most 29 seconds for a response, so executions behind the API must finish within that time even though Express workflows can run for up to 5 minutes.
{%- else %}

The state machine is a `STANDARD` workflow, so API Gateway starts it asynchronously and returns `202` with the execution ARN and start time. Use the execution ARN to check the execution's status, for example with `aws stepfunctions describe-execution`.
{%- endif %}

#### Authentication and Authorization
This service is configured to use a pre-existing Cognito User Pool. Clients should obtain a JWT from the Cognito token endpoint using the client's clientId and clientSecret. Each endpoint's scope requirements are defined in the [OpenAPI document](./openapi.yaml). The state machine endpoint requires the `https://${{ values.hostname }}/${{ values.resource_name }}.${{ 'read' if values.http_method == 'GET' else 'write' }}` scope.

### State machine

The **${{ values.state_machine_name }}** state machine starts with a single placeholder `Pass` state followed by a `Succeed` state. Replace these states with the steps of your workflow.

The definition uses the JSONata query language. The `Domain`, `System`, `Component`, and `CodeBranch` stack parameters are passed into the definition as substitutions, so you can reference them as `${Domain}`, `${System}`, and so on. Add more entries to `DefinitionSubstitutions` in `template.yaml` to pass in resource ARNs, such as Lambda functions or SNS topics.

You can edit the definition visually with [Workflow Studio in the AWS Toolkit for VS Code](https://docs.aws.amazon.com/step-functions/latest/dg/workflow-studio-local.html).

### Permissions

The state machine runs with an IAM role that AWS SAM creates. When you add states that call other AWS services, grant the matching permissions under `Policies` in `template.yaml`. Use [AWS SAM policy templates](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-policy-templates.html) where possible.

API Gateway uses the `RestApiIamRole` role to start executions. It only allows starting this state machine.

### Local validation

Validate the template before you push:

```sh
sam validate --lint
```
