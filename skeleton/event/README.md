# ${{ values.system_title }} / ${{ values.component_name }}

${{ values.component_description }}


The [project handbook](docs/handbook/00-index.md) covers features, project creation,
configuration, architecture, CI/CD, observability, and common development tasks.
It is also available in the component's **Docs** tab when Backstage TechDocs is enabled.


## New Project Getting Started
This repository was generated from a template intended to get a new AWS Step Functions state machine up and running quickly. It uses the [AWS Serverless Application Model (SAM)](https://aws.amazon.com/serverless/sam/) to define the infrastructure as code.

### Project layout

- [`template.yaml`](template.yaml): AWS SAM template that defines the state machine and its supporting resources.
- [`statemachine/${{ values.state_machine_name }}.asl.yaml`](statemachine/${{ values.state_machine_name }}.asl.yaml): the state machine definition, written in [Amazon States Language (ASL)](https://docs.aws.amazon.com/step-functions/latest/dg/concepts-amazon-states-language.html).
- [`cfn-parameters.json`](cfn-parameters.json): parameter values used when the stack is deployed.
- [`cfn-tags.json`](cfn-tags.json): tags applied to the deployed stack.
- [`.github/workflows/`](.github/workflows/): GitHub Actions workflows that validate, build, and deploy the stack.
- [`mkdocs.yml`](mkdocs.yml) and [`docs/handbook/`](docs/handbook/00-index.md): the project handbook, published through Backstage TechDocs.

### State machine

The **${{ values.state_machine_name }}** state machine is a `${{ values.state_machine_type }}` workflow. It starts with a single placeholder `Pass` state followed by a `Succeed` state. Replace these states with the steps of your workflow.

The definition uses the JSONata query language. The `Domain`, `System`, `Component`, and `CodeBranch` stack parameters are passed into the definition as substitutions, so you can reference them as `${Domain}`, `${System}`, and so on. Add more entries to `DefinitionSubstitutions` in `template.yaml` to pass in resource ARNs, such as Lambda functions or SNS topics.

You can edit the definition visually with [Workflow Studio in the AWS Toolkit for VS Code](https://docs.aws.amazon.com/step-functions/latest/dg/workflow-studio-local.html).

### Permissions

The state machine runs with an IAM role that AWS SAM creates. When you add states that call other AWS services, grant the matching permissions under `Policies` in `template.yaml`. Use [AWS SAM policy templates](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-policy-templates.html) where possible.

### Triggers
{%- if values.trigger_type == 'schedule' %}

The state machine runs on the schedule `${{ values.schedule_expression }}` using an EventBridge Scheduler schedule.
{%- elif values.trigger_type == 'eventbridge' %}

The state machine runs when an event with source `${{ values.event_pattern_source }}`{% if values.event_pattern_detail_type %} and detail type `${{ values.event_pattern_detail_type }}`{% endif %} is put on the default EventBridge event bus.
{%- else %}

The state machine has no trigger. Start executions from the AWS console, the AWS CLI, or another service. To add a trigger, see the `Events` property of [`AWS::Serverless::StateMachine`](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-resource-statemachine.html).
{%- endif %}

### Running the state machine

After the stack is deployed, the state machine ARN is available as a stack output and in SSM Parameter Store at:

```
/${{ values.domain_name }}/${{ values.system_name }}/${{ values.component_name }}/<branch>/${{ values.state_machine_name }}StateMachineArn
```

To start an execution with the AWS CLI:

```sh
aws stepfunctions start-execution \
  --state-machine-arn <state-machine-arn> \
  --input '{"hello": "world"}'
```

### Local validation

Validate the template before you push:

```sh
sam validate --lint
```
