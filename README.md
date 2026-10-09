# AWS SAM Step Functions State Machine Backstage Templates

This repository contains Backstage scaffolder templates for creating a new AWS Serverless Application Model (SAM) project that defines an AWS Step Functions state machine.

They give teams a working state machine they can deploy right away, then extend with their own workflow steps. Backstage generates the repository, publishes it to GitHub, and registers it in the catalog.

## Templates

| Template file | Template | How the state machine is started |
| --- | --- | --- |
| `template.yaml` | AWS Step Functions State Machine | Manually, on a schedule, or by an EventBridge rule |
| `template-api.yaml` | AWS Step Functions State Machine API | By an API Gateway endpoint |

## What the templates create

Both templates generate a new repository with:

- An AWS SAM `template.yaml` that defines an `AWS::Serverless::StateMachine`
- A starter Amazon States Language (ASL) definition in `statemachine/`
- Optional CloudWatch Logs logging and AWS X-Ray tracing
- SSM parameters and stack outputs for the state machine ARN and name
- Backstage catalog metadata in `catalog-info.yaml`
- Standard project files such as `.gitignore`, `samconfig.toml`, and deployment parameter/tag config
- A starter `README.md` for the generated project
- GitHub Actions workflows for build and deployment via AWS SAM

`template.yaml` adds an optional EventBridge Scheduler schedule or EventBridge rule trigger.

`template-api.yaml` adds an API Gateway REST API that starts the state machine directly, without a Lambda function:

- One endpoint at `<path prefix>/<resource name>`, using the HTTP method you choose (`POST` by default)
- For Express workflows, the endpoint runs the state machine synchronously and returns its output
- For Standard workflows, the endpoint starts the state machine asynchronously and returns the execution ARN
- A `/health` endpoint, a custom domain with an ACM certificate, and Cognito authorization, set up the same way as the Python CRUD API template
- An `openapi.yaml` document, which the SAM template uses to create the API and which is registered in the catalog as an API entity

## Repository layout

- `template.yaml`: the scaffolder template for an event-triggered state machine
- `template-api.yaml`: the scaffolder template for a state machine started through API Gateway
- `skeleton/base/`: files copied into every generated project
- `skeleton/event/`: SAM template, README, and parameters for `template.yaml`
- `skeleton/api/`: SAM template, OpenAPI document, README, and parameters for `template-api.yaml`
- `pipeline/`: GitHub Actions pipeline templates used by the generated project

## Typical use in Backstage

Both templates prompt for:

- component name and description
- owning group
- domain and system
- deployment environment
- target AWS account
- state machine name and type (Standard or Express)
- log level and whether to enable X-Ray tracing
- optionally, whether to launch a GitHub Copilot cloud agent, and either a prompt for it or, with Spec Kit, a description of the project's principles for its constitution

`template.yaml` also asks for the trigger type: none, schedule, or EventBridge rule.

`template-api.yaml` also asks for the hostname, path prefix, resource name, HTTP method, and API description.

The templates then:

1. fetch related catalog entities,
2. copy the project skeleton,
3. create the deployment pipeline files,
4. optionally initialize [Spec Kit](https://github.com/github/spec-kit) in the project,
5. publish the repository to GitHub,
6. register the generated component in the Backstage catalog, and
7. optionally launch a Copilot cloud agent on the new repository. With Spec Kit, the agent creates the project's Spec Kit constitution; otherwise, it works on your prompt.

## Prerequisites for the API template

The API template expects the same shared infrastructure as the Python CRUD API template:

- The Route 53 hosted zone ID in the `/org/dns/ZoneId` SSM parameter
- A Cognito user pool, whose ARN is available to the deployment workflow as the `COGNITO_USER_POOL_ARN` secret
- An API Gateway account-level CloudWatch Logs role, so the API can write access logs

## Next steps

After generating a service from these templates:

- replace the placeholder states in the ASL definition with your workflow
- add IAM policies to `template.yaml` for any AWS services the state machine calls
- add Lambda functions or other resources the workflow needs, and pass their ARNs in with `DefinitionSubstitutions`
- update the generated project README and metadata to match the service purpose
