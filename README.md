# AWS SAM Step Functions State Machine Backstage Template

This repository is a Backstage scaffolder template for creating a new AWS Serverless Application Model (SAM) project that defines an AWS Step Functions state machine.

It gives teams a working state machine they can deploy right away, then extend with their own workflow steps. Backstage generates the repository, publishes it to GitHub, and registers it in the catalog.

## What this template creates

When used from Backstage, it generates a new repository skeleton with:

- An AWS SAM `template.yaml` that defines an `AWS::Serverless::StateMachine`
- A starter Amazon States Language (ASL) definition in `statemachine/`
- Optional CloudWatch Logs logging and AWS X-Ray tracing
- An optional EventBridge Scheduler schedule or EventBridge rule trigger
- SSM parameters and stack outputs for the state machine ARN and name
- Backstage catalog metadata in `catalog-info.yaml`
- Standard project files such as `.gitignore`, `samconfig.toml`, and deployment parameter/tag config
- A starter `README.md` for the generated project
- GitHub Actions workflows for build and deployment via AWS SAM

## Repository layout

- `template.yaml`: the Backstage scaffolder template definition
- `skeleton/`: files copied into the generated project
- `pipeline/`: GitHub Actions pipeline templates used by the generated project

## Typical use in Backstage

The template prompts for:

- component name and description
- owning group
- domain and system
- deployment environment
- target AWS account
- state machine name and type (Standard or Express)
- log level and whether to enable X-Ray tracing
- trigger type: none, schedule, or EventBridge rule
- optionally, a prompt for a GitHub Copilot cloud agent

It then:

1. fetches related catalog entities,
2. copies the project skeleton,
3. creates the deployment pipeline files,
4. publishes the repository to GitHub,
5. registers the generated component in the Backstage catalog, and
6. optionally launches a Copilot cloud agent on the new repository.

## Next steps

After generating a service from this template:

- replace the placeholder states in the ASL definition with your workflow
- add IAM policies to `template.yaml` for any AWS services the state machine calls
- add Lambda functions or other resources the workflow needs, and pass their ARNs in with `DefinitionSubstitutions`
- update the generated project README and metadata to match the service purpose
