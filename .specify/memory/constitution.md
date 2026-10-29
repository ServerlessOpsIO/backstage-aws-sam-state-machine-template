<!--
Sync Impact Report
- Version change: unratified scaffold → 1.0.0
- Modified principles: replaced the five generic scaffold placeholders with six
  repository-specific principles; no previously ratified principle titles existed.
- Added sections: Development workflow and quality gates.
- Removed sections: empty generic Section 2 and Section 3 placeholders, consolidated
  into the project-specific development workflow section.
- Omitted rules: workflow-template selection, branch/pull-request, and inline
  shell behavior, because this repository contains project templates only.
- Follow-up TODOs: none.
-->

# backstage-aws-sam-state-machine-template Constitution

## Core Principles

### I. Every option combination renders correctly

- Every combination of template options MUST render a complete, valid project.
  The repository contains `template.yaml` for an event-triggered state machine
  (manual, scheduled, or EventBridge rule) and `template-api.yaml` for a state
  machine started through API Gateway. Optional features include schedule or
  EventBridge triggers, logging settings, X-Ray tracing, Copilot agent launch,
  and Spec Kit initialization.
- Content inside a `{% if %}` block MUST NOT be referenced from outside that
  block. Every variable used by a skeleton, including in file and directory
  names, MUST be passed in the `fetch:template` `values` for every template
  that renders it.
- Rendered JSON and YAML MUST be valid for every option combination, including
  commas and indentation around conditional blocks. GitHub Actions expressions
  written as `${{ }}` in skeleton files MUST be wrapped in
  `{% raw %}...{% endraw %}` so template rendering preserves them.
- Generated projects MUST pass the validation and build steps in the generated
  pipeline without hand edits. The `pipeline/` directory supplies
  `.github/workflows/main.yml` and `branch.yml`; both run `sam validate --lint`
  and `sam build`.
- Changes to a skeleton or option MUST be verified by rendering every affected
  combination, not only the default path.

### II. The catalog drives the inputs

- Ownership and placement MUST come from catalog pickers with
  `allowArbitraryValues: false`, in this order: owner (`MyGroupsPicker`),
  Domain, System, then Environment and cloud account where deployment context
  is needed. Free-text fields MUST NOT replace catalog entity values.
- Domain, System, Environment, and cloud-account pickers MUST filter against
  the preceding catalog selection. The current templates filter Systems by
  Domain, Environments by System, and cloud accounts by Environment.
- Values from `catalog:fetch` MUST be read from the entity they belong to.
  Changing the order of `entityRefs` or adding to it MUST update every place
  that reads results by index.
- Implicit catalog contracts MUST be documented next to their use. In
  particular, the cloud-account entity's `metadata.title` is used as the AWS
  account ID passed to the generated repository as `DEPLOYMENT_ACCOUNT_ID`.

### III. Published repositories and entities are predictable

- Both project templates create a repository and a Backstage `Component` with
  `spec.type: service`. `template.yaml` creates the event-triggered state
  machine project. `template-api.yaml` also creates an `API` entity with
  `spec.type: openapi`, exposed by the Component through `providesApis`.
- The GitHub repository name and Component entity name MUST be the same value:
  the user-provided component name lowercased, with no added suffix. The API
  entity uses that same name.
- The GitHub organization MUST come from a hidden `github_org` parameter
  defaulting to `ServerlessOpsIO`; it MUST NOT be hard-coded in steps or
  skeletons.
- Every generated catalog entity MUST set its applicable `owner`, `system`,
  and `lifecycle`. Each generated `catalog-info.yaml` MUST set the
  `github.com/project-slug` annotation, a `spec.type` describing the entity,
  and `dependsOn` or `providesApis` relations for catalog entities the project
  relies on or exposes.
- Project templates MUST publish with `main` as the default branch, set
  repository custom properties for domain, system, and owner from catalog
  selections, and register `/catalog-info.yaml` at the repository root.
  Current publishing settings are public visibility, no default-branch
  protection, and zero required approving reviews; all templates MUST use the
  same settings unless a template explains its difference.
- Commits MUST be authored as the user running the template, using their
  catalog profile name and email.

### IV. Steps are ordered, optional, and fail safely

- Project-template steps MUST follow this order: fetch catalog entities,
  render skeletons, render the pipeline, add optional starters, publish,
  register, then run optional steps that act on the published repository or
  external services.
- Optional steps MUST be off unless selected, guarded with `if:`, and their
  inputs MUST be shown and required only when the option is on. The current
  Spec Kit workspace initializer is an optional starter run before publishing;
  Copilot launch and Spec Kit constitution actions run after registration.
- Steps that call external services or act on the published repository MUST
  run after registration so a failure leaves a registered project rather than
  a half-created one.
- Comments MUST explain non-obvious behavior next to the relevant step,
  including explicit value passing, external-resource matching, and catalog
  filter dependencies.

### V. Credentials and secrets stay out of templates

- Secrets and account-specific values MUST reach generated repositories only
  through scaffolder secrets and the `secrets` input of `publish:github`.
  Secrets MUST NOT be written into skeleton files, template defaults, or this
  repository. Skeletons MUST refer to secrets by name for the generated
  project's pipeline to resolve at run time.
- Actions taken on the user's behalf, including Copilot or Spec Kit actions,
  MUST use the user's GitHub credentials requested with
  `SoRequestUserCredentials` into `USER_GITHUB_TOKEN`, with only the scopes
  needed.
- Tokens MUST NOT be written to files, git remotes, logs, or pull-request text.

### VI. The form is honest and the templates are consistent

- Parameter titles and descriptions MUST match the field's behavior, including
  hidden fields. Defaults MUST be sensible values users can accept as-is.
- Fields that apply only to some choices MUST be shown and required through
  JSON Schema `dependencies`, and only when those choices are selected. An
  enum option MUST NOT be offered unless the template renders a working
  project for it.
- Step IDs and names MUST describe what the step does.
- Templates in this repository MUST share the same first parameter page,
  metadata conventions (owner, tags, and description format), and
  `publish:github` settings. Changes to a shared field MUST be applied to every
  template that has it unless the change explains the difference.
- Every template's `output.links` MUST link to what it created: the repository,
  catalog entity, and any agent task or external resource created by an
  optional step.

## Development workflow and quality gates

- Before merge, the affected templates MUST be run through the Backstage
  template editor dry run. Every affected option combination MUST be rendered
  and checked against Principle I. Changes to publishing, registration,
  or optional external steps MUST be exercised by a real run against a test
  component or sandbox. The pull request MUST list what was run.
- Changes to template inputs, generated files, required catalog entities,
  required secrets, required Backstage actions, or other prerequisites MUST
  update `README.md` in the same change. Update the generated project README
  when it is affected.
- A template requiring a new or changed scaffolder action MUST name the action
  and the Backstage module that provides it in `README.md`.
- Generated, operating-system, and editor files, including `.DS_Store`,
  `__pycache__`, and `node_modules`, MUST NOT be committed.
- Work MUST use `feature/` or `bug/` branches that reference an issue number
  where one exists. Commit messages MUST use conventional-commit style such as
  `feat:`, `fix:`, or `refactor:`. Changes MUST merge through pull requests.

## Governance

- This constitution supersedes conflicting conventions in this repository.
  Every spec, plan, and pull request MUST check compliance with it.
- Amendments MUST be made in the same change that introduces the new rule and
  MUST include a Sync Impact Report.
- Versioning follows semantic versioning: MAJOR for incompatible principle
  removals or redefinitions, MINOR for added principles or sections or
  materially expanded guidance, and PATCH for clarifications.

**Version**: 1.0.0 | **Ratified**: 2026-10-29 | **Last Amended**: 2026-10-29
