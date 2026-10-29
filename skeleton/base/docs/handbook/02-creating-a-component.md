# Creating a component via Backstage

{%- if values.is_api %}

Choose **Create** in Backstage and select **AWS Step Functions State Machine API**.
{%- else %}

Choose **Create** in Backstage and select **AWS Step Functions State Machine**.
{%- endif %}
The platform needs GitHub publishing, catalog registration, the template's
custom form fields/actions, and catalog entities for the choices below.

The repository's template source offers two templates. **AWS Step Functions
State Machine** creates a state machine started manually, on a schedule, or by
an EventBridge rule. **AWS Step Functions State Machine API** creates a state
machine started by an API Gateway endpoint. This project was created with the
{% if values.is_api %}API{% else %}event-triggered{% endif %} template.

## Page 1: Component Information

Paths below are relative to the **newly generated repository root**, not the
template source repository. Catalog selections generally contribute the entity's
name rather than its display title; the exceptions are noted below.

| Input | Purpose | Where to see it after creation |
| --- | --- | --- |
| Component name | Unique component and repository name, lowercased by the template. | GitHub repository name; Component{% if values.is_api %} and API{% endif %} `metadata.name`{% if values.is_api %}, `spec.providesApis`,{% endif %} and repository references in `catalog-info.yaml`; `stack_name` in `samconfig.toml`; the first line of `Description` in `template.yaml`; `Comment` in the state machine definition;{% if values.is_api %} `info.title` in `openapi.yaml`;{% endif %} headings in `README.md` and `docs/handbook/00-index.md`; `site_name` in `mkdocs.yml`. `Component` in `cfn-parameters.json` and `org:component` in `cfn-tags.json` resolve from the repository-name workflow variable rather than a literal form value. |
| Description | Describes the repository, component, and SAM application. | GitHub repository description; Component `metadata.description` in `catalog-info.yaml`; top-level `Description` in `template.yaml`; introductory text in `README.md`. |
| Owner | Owning group selected from your groups. | Component{% if values.is_api %} and API{% endif %} `spec.owner` in `catalog-info.yaml` use `group:<selected-group-name>`; GitHub repository Settings → Custom properties → `owner` contains the selected group's `metadata.title`, not its entity name. |
| Domain | Catalog domain, also used for deployment metadata, tagging, and SSM parameter names. | `Domain` in `cfn-parameters.json`; `org:domain` in `cfn-tags.json`; GitHub Settings → Custom properties → `domain`; the first segment of the stack's SSM parameter names. There is no direct domain relationship in the generated `catalog-info.yaml`. |
| System | Catalog system filtered by the chosen domain. | Component{% if values.is_api %} and API{% endif %} `spec.system` in `catalog-info.yaml`; `System` in `cfn-parameters.json`; `org:system` in `cfn-tags.json`; the second segment of the stack's SSM parameter names; GitHub Settings → Custom properties → `system`. The `README.md` heading uses the system's `metadata.title`. |
| Environment | Catalog environment filtered by the chosen system; narrows the cloud-account picker. | Scaffolder selection only: the chosen environment is not written to a generated file. `.github/workflows/main.yml` and `.github/workflows/branch.yml` independently set `jobs.deploy.environment: production`. |
| Account ID | Catalog cloud-account resource related to the environment; its `metadata.title` must contain the AWS account ID. | GitHub Settings → Secrets and variables → Actions → repository secret `DEPLOYMENT_ACCOUNT_ID`; both workflow files reference it in deployment steps. The account ID is not written into application files. GitHub shows the secret's name but does not reveal its saved value; confirm the source catalog entity's title before creation. |

Hidden defaults are lifecycle `production`, namespace `default`, and GitHub
organization `ServerlessOpsIO`. Namespace is collected but is not currently
passed into generated catalog metadata.

Inspect lifecycle in {% if values.is_api %}both entities'{% else %}the Component's{% endif %}
`spec.lifecycle` in `catalog-info.yaml`. The GitHub organization is visible in
the repository URL{% if values.is_api %}, `metadata.annotations.github.com/project-slug`, and the API definition's `$text`
URL{% else %} and `metadata.annotations.github.com/project-slug`{% endif %} in
`catalog-info.yaml`. The hidden namespace value is not written as
`metadata.namespace`; it remains a scaffolder input only.

Choosing an Environment does not configure a matching GitHub environment:
the generated workflows both explicitly use `production`.

## Page 2: State Machine Configuration

| Input | Purpose | Where to see it after creation |
| --- | --- | --- |
| State Machine Name | PascalCase name used as the CloudFormation logical ID prefix and definition filename, for example `OrderProcessing`. The form requires it to start with an uppercase letter and contain only letters and digits. | Logical IDs `<Name>StateMachine`,{% if values.log_level %} `<Name>StateMachineLogGroup`,{% endif %} `<Name>StateMachineArnSsmParam`, and `<Name>StateMachineNameSsmParam` in `template.yaml`; output names and SSM parameter name suffixes `<Name>StateMachineArn` and `<Name>StateMachineName`; `DefinitionUri` and the file `statemachine/<Name>.asl.yaml`;{% if values.log_level %} the log group name suffix;{% endif %}{% if values.is_api %} the operation summary and the request template's state machine reference in `openapi.yaml`;{% endif %} `README.md`. It is **not** the deployed state machine's physical name, which CloudFormation generates. |
| State Machine Type | `STANDARD` (default) for durable, long-running workflows or `EXPRESS` for high-volume workflows of up to five minutes. | `Type` on the state machine in `template.yaml`; `README.md`.{% if values.is_api %} In `openapi.yaml`, it selects the integration action (`StartSyncExecution` for Express, `StartExecution` for Standard), the success response, and its schema; in `template.yaml`, it selects the `RestApiIamRole` action.{% endif %} See [Features](01-features.md) for the differences. |
| Log Level | Execution history events sent to CloudWatch Logs: All, Error (default), Fatal, or Off. | `Logging.Level` on the state machine in `template.yaml`. Off omits the `Logging` configuration, the log group, the `StateMachineLogRetentionInDays` parameter, and the log-delivery policy statement entirely. |
| Enable X-Ray Tracing | Sends execution traces to AWS X-Ray; defaults to off. | `Tracing.Enabled` and the `AWSXrayWriteOnlyAccess` policy on the state machine in `template.yaml`. When off, neither is present. |

This project was created with `${{ values.state_machine_name }}`,
`${{ values.state_machine_type }}`, log level
{% if values.log_level %}`${{ values.log_level }}`{% else %}Off{% endif %},
and tracing {% if values.enable_tracing %}on{% else %}off{% endif %}.
The definition file is
`statemachine/${{ values.state_machine_name }}.asl.yaml`.
{%- if values.is_api %}

## Page 3: API Configuration

| Input | Purpose | Where to see it after creation |
| --- | --- | --- |
| Hostname | Fully qualified API DNS name for TLS, the custom domain, and DNS record. | Literal `Hostname` in `cfn-parameters.json`; the endpoint URL in `README.md`. In `template.yaml`, `SiteCertificate.Properties.DomainName`, `RestApi.Properties.Domain.DomainName`, and the `ApiEndpoint` output reference this parameter; the API's `Domain.Route53` configures its DNS record. In `openapi.yaml`, `servers` and security scopes use `Fn::Sub` expressions referencing `${Hostname}`, resolved during deployment rather than replaced with a literal hostname during scaffolding. |
| Path Prefix | Prefix before the resource name, such as `/v1`; defaults to `/v1`. Include the leading slash and avoid a trailing slash; it may be empty. | The endpoint's key under `paths` in `openapi.yaml`; the `ApiEndpoint` output in `template.yaml`; the endpoint URL in `README.md`. It does not change `/health` or become a separate stack parameter. |
| Resource Name | Path segment after the prefix, such as `orders`. Letters, digits, and `.`, `_`, `~`, `-` only. | The endpoint's key under `paths` and its security scope in `openapi.yaml`; the `ApiEndpoint` output; the endpoint URL and scope in `README.md`. |
| HTTP Method | Client-facing method that starts the state machine: `POST` (default), `PUT`, `PATCH`, `GET`, or `DELETE`. | The operation key under the endpoint's path in `openapi.yaml`. `POST`, `PUT`, and `PATCH` declare a required JSON request body that becomes the execution input; `GET` and `DELETE` start executions with `{}`. `GET` requires the `.read` scope; the other methods require `.write`. |
| API Description | Describes the API separately from the component. | `info.description` in `openapi.yaml` and the API entity's `metadata.description` in `catalog-info.yaml`, not the Component entity's description. |

For example, prefix `/v1`, resource `orders`, and method `POST` produce
`POST https://<hostname>/v1/orders` with the scope
`https://<hostname>/orders.write`.

This project's endpoint is
`${{ values.http_method }} https://${{ values.hostname }}${{ values.url_path_prefix }}/${{ values.resource_name }}`.
These values are copied into multiple files at creation time, not maintained by
a single live configuration source. If you change the hostname, path, or method
later, update `cfn-parameters.json`, `openapi.yaml`, the `ApiEndpoint` output,
`README.md`, and this handbook together, and review DNS/TLS and Cognito scopes
in [Configuration](03-configuration.md) and [Features](01-features.md).
{%- else %}

## Page 3: State Machine Trigger

| Input | Purpose | Where to see it after creation |
| --- | --- | --- |
| Trigger Type | What starts executions: None / Manual (default), Schedule, or EventBridge Rule. | The `Events` property of the state machine in `template.yaml`, which is absent for None / Manual; the Triggers section of `README.md`. |
| Schedule Expression | Shown for Schedule. A `cron(...)`, `rate(...)`, or `at(...)` EventBridge Scheduler expression, for example `rate(1 hour)`. The form does not validate it; an invalid expression fails at deployment. | `Events.Schedule.Properties.ScheduleExpression` in `template.yaml`; `README.md`. |
| Event Source | Shown for EventBridge Rule. Value of the event `source` field to match on the default event bus, for example `com.example.orders`. | `Events.EventBridgeRule.Properties.Pattern.source` in `template.yaml`; `README.md`. |
| Event Detail Type | Shown for EventBridge Rule; optional. Value of the event `detail-type` field to match, for example `OrderCreated`. | `Events.EventBridgeRule.Properties.Pattern.detail-type` in `template.yaml` when supplied; `README.md`. |

Each field holds a single value. To match several sources or detail types, or
other event fields, edit the pattern in `template.yaml`; see
[Common tasks](07-common-tasks.md#change-the-trigger).
{%- endif %}

## Page 4: Copilot Cloud Agent

**Launch Copilot agent?** defaults to false. Leaving it disabled generates the
service without launching an agent.

If enabled, choose whether to initialize Spec Kit:

- With Spec Kit disabled, supply a non-empty **Agent prompt** describing the
  work to perform. After publication and catalog registration, the
  `github:copilot:agent:launch` action launches the agent.
- With Spec Kit enabled, supply a **Spec Kit constitution** describing the
  project and its principles. The scaffolder initializes Spec Kit before
  publication, then the `github:copilot:speckit:constitution` action creates
  the constitution task, which opens a pull request.
{%- if not values.is_api %}

The default constitution text describes a state machine started through an
API Gateway endpoint. For this template, edit it to describe the trigger you
chose before submitting.
{%- endif %}

Both paths request GitHub credentials under `USER_GITHUB_TOKEN` with the
additional `read:user` scope. They require platform support and appropriate
GitHub access. Do not put secrets in the prompt or constitution.

## Review and create

Review the form summary, especially repository name, account,
{% if values.is_api %}hostname, path, and method{% else %}state machine name, and trigger{% endif %},
before submitting. The scaffolder then:

1. Fetches the selected domain, system, owner, and account catalog entities.
2. Copies shared files, the state machine definition, and this handbook, then
   the {% if values.is_api %}API infrastructure and OpenAPI document{% else %}event-trigger infrastructure{% endif %},
   and GitHub Actions workflows.
3. Optionally initializes Spec Kit in the workspace.
4. Publishes a **public** repository with `main` as the default branch and sets
   `DEPLOYMENT_ACCOUNT_ID`.
5. Registers the generated Component{% if values.is_api %} and OpenAPI API entity{% endif %} in Backstage.
6. Optionally launches Copilot.

The results link to the repository, `catalog-info.yaml`, catalog component,
and optional agent task. Default branch protection is disabled and the required
approving review count is zero in the publisher configuration; apply your
review policy before development.

## After creation

- Confirm catalog ownership and relationships.
- Complete [deployment configuration](03-configuration.md) before assuming the
  initial push can deploy successfully.
- Review [branch deployment isolation](05-ci-cd.md) before pushing additional branches.
- Open the component's **Docs** tab, or read `docs/handbook/00-index.md` in the repository.
- Replace the placeholder workflow using [Common tasks](07-common-tasks.md), and
  request human review of the generated handbook.
