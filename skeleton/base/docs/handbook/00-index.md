# Project handbook

This handbook describes the starter state machine generated for
**${{ values.component_name }}**. Keep it current as the workflow,
infrastructure, and processes change.

{%- if values.is_api %}

- [Features](01-features.md): state machine, API definition, DNS, Cognito, logging, and outputs.
{%- else %}

- [Features](01-features.md): state machine, trigger, logging, and outputs.
{%- endif %}
- [Creating a component](02-creating-a-component.md): Backstage form pages and generated outputs.
- [Configuration](03-configuration.md): deployment parameters, tags, and prerequisites.
- [Architecture](04-architecture.md): AWS resources, state machine anatomy, and repository layout.
- [CI/CD pipeline](05-ci-cd.md): build, deployment, and feature branches.
- [Observability](06-observability.md): execution history, logs, metrics, and automatic Datadog integration.
{%- if values.is_api %}
- [Common tasks](07-common-tasks.md): add states, service calls, functions, and endpoints, and test changes.
{%- else %}
- [Common tasks](07-common-tasks.md): add states, service calls, functions, and triggers, and test changes.
{%- endif %}

## Reading in Backstage

The component's `catalog-info.yaml` contains `backstage.io/techdocs-ref: dir:.`.
TechDocs uses the repository-root `mkdocs.yml` and the Markdown files in
`docs/handbook/`.
Open the component in the catalog and select **Docs**.

The platform must enable the TechDocs frontend and backend and configure either
on-demand documentation generation or an external builder and publisher. The
builder needs MkDocs and `mkdocs-techdocs-core` for the `techdocs-core` plugin.
The generated application workflows do not build or publish documentation.
With the Backstage TechDocs CLI and its Docker prerequisites installed, preview
from the generated repository root with `techdocs-cli serve`.

The pages can also be read directly in GitHub. File paths shown in backticks
are relative to the generated repository root.

## Content review

Before treating this starter handbook as your service's operational reference,
review its workflow, {% if values.is_api %}API contract, {% endif %}IAM
permissions, deployment assumptions, and runbooks. Template-generated
documentation does not constitute human approval.
