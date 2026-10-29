<!--
Sync Impact Report
Version change: unratified placeholder scaffold -> 1.0.0
Modified principles: placeholder principles -> I. Every Option Combination Renders Correctly;
  II. The Catalog Drives the Inputs; III. Published Repositories and Entities Are Predictable;
  IV. Steps Are Ordered, Optional, and Fail Safely; V. Credentials and Secrets Stay Out of
  Templates; VI. The Form Is Honest and Templates Are Consistent.
Added sections: Scope and Applicability; Development Workflow and Quality Gates.
Removed sections: None.
Rules omitted and why: Workflow-template branch and pull-request rules are omitted because this
  repository contains only project templates. User-credential requirements for Copilot and Spec
  Kit actions are omitted because no such actions are present. External Datadog provisioning
  requirements are omitted because Datadog is rendered as project configuration, not provisioned
  by an external scaffolder step.
Follow-up TODOs: Existing templates hard-code the GitHub organization, default optional feature
  flags to true, offer Python although its starter is marked TODO, use different first parameter
  pages, and do not define output links. These implementation gaps are recorded for conformance
  follow-up; this update changes only the constitution.
-->

# backstage-aws-sam-ecs-template Constitution

## Core Principles

### I. Every Option Combination Renders Correctly

- Every supported combination of template options MUST render a complete, valid project.
  A form choice MUST NOT produce a broken file or an incomplete project.
- Content declared inside a `{% if %}` block MUST NOT be referenced outside that block.
- Every variable used by a skeleton, including in file and directory names, MUST be supplied
  through the `values` of every `fetch:template` step that renders it.
- Rendered JSON and YAML MUST be valid for every supported option combination, including
  conditional commas and indentation.
- GitHub Actions expressions `${{ ... }}` in skeleton files MUST be enclosed in
  `{% raw %}...{% endraw %}` so template rendering preserves them.
- A generated project MUST pass the validation, build, and test steps in its generated pipeline
  without hand edits. Changes to a skeleton or option MUST be checked by rendering every
  affected combination, not only the default.

These rules prevent a valid form selection from producing files that cannot be parsed or built.

### II. The Catalog Drives the Inputs

- Ownership and placement MUST come from catalog pickers with
  `allowArbitraryValues: false`: owner from `MyGroupsPicker`, then Domain, System, and the
  applicable Environment and cloud account; service templates MUST also select the existing
  cluster Component.
- Free-text fields MUST NOT substitute for values represented by catalog entities.
- Values obtained through `catalog:fetch` MUST be read from the entity they belong to. Changing
  `entityRefs` order or adding an entity MUST update every result read by index.
- Implicit catalog contracts MUST be documented next to their use. In this repository, the
  cloud-account Resource title is passed as `DEPLOYMENT_ACCOUNT_ID`; generated cluster Components
  have type `container-cluster`, and service Components depend on that cluster.

Catalog-backed choices keep generated ownership, placement, and dependencies aligned with the
Backstage catalog rather than accepting unvalidated references.

### III. Published Repositories and Entities Are Predictable

- Repository names and generated catalog entity names MUST match. Names MUST be derived from the
  user's name input using the template's fixed suffix rule and lowercased: `-ecs-cluster` for
  `aws-sam-ecs-cluster`, and `-app` for both container templates.
- The GitHub organization MUST come from a hidden `github_org` parameter defaulting to
  `ServerlessOpsIO`; steps and skeletons MUST NOT hard-code it.
- Each generated `catalog-info.yaml` MUST set `owner`, `system`, `lifecycle`, the
  `github.com/project-slug` annotation, and a `spec.type` matching the generated Component.
  It MUST include `dependsOn` or `providesApis` relations for catalog entities the project uses
  or exposes. Current types are `container-cluster` and `container-service`.
- Project templates MUST publish with `main` as the default branch, set repository custom
  properties for domain, system, and owner from catalog selections, and register the root
  `/catalog-info.yaml`.
- All templates MUST use the same repository visibility and branch-review settings.
  The current convention is public visibility, no default-branch protection, and zero required
  approving reviews; any intentional difference MUST be explained in the template.

Consistent repository and catalog metadata makes the generated resource discoverable and its
relationships explicit.

### IV. Steps Are Ordered, Optional, and Fail Safely

- Project-template steps MUST follow this order: fetch catalog entities; render project
  skeletons; render pipelines; add optional starters; publish; register; then run any optional
  steps acting on the published repository or external services.
- Optional features and starter steps MUST be disabled unless selected, guarded with `if:`, and
  expose and require their inputs only when enabled.
- Steps that call external services MUST run after catalog registration so a failure does not
  leave a published project unregistered.
- Any inline shell used by a template step MUST begin with `set -eu` and stop on unexpected
  state.
- Comments MUST explain non-obvious behavior next to the relevant step, including explicit value
  passing, external-resource matching, and catalog filters.

The ordering ensures generated files exist before publication and that external failures do not
prevent catalog registration.

### V. Credentials and Secrets Stay Out of Templates

- Secrets and account-specific values MUST reach generated repositories only through
  scaffolder secrets and the `secrets` input of `publish:github`.
- Secrets MUST NOT appear in skeleton files, template defaults, or commits to this repository.
  Generated pipeline files MUST refer to secrets by name for resolution at runtime.
- Tokens MUST NOT be written to files, Git remotes, logs, or pull-request text.
- Commits MUST be authored using the template runner's catalog profile name and email.

Keeping secret values out of rendered files and source control limits accidental disclosure.

### VI. The Form Is Honest and Templates Are Consistent

- Parameter titles and descriptions MUST accurately describe their fields, including hidden
  fields. Defaults MUST be values users can accept without unexpected behavior.
- Fields applicable only to certain choices MUST be shown and required only through JSON Schema
  `dependencies` or equivalent conditional schema.
- An enum option MUST NOT be offered unless the template renders a working project for it.
- Step IDs and names MUST describe the action they perform.
- All templates MUST share the same first parameter page, metadata conventions for owner, tags,
  and description format, and `publish:github` settings. Changes to shared fields MUST be
  mirrored across applicable templates unless the difference is explained.
- Every template's `output.links` MUST link to each created repository or pull request, catalog
  entity, and optional agent task or external resource.

Accurate forms and consistent metadata make supported outcomes clear and reduce template drift.

## Scope and Applicability

This constitution governs the Backstage scaffolder templates in `template.yaml` and the files
they render from `cluster/skeleton/`, `cluster/pipeline/`, `container/skeleton/`,
`container/pipeline/`, and `container/nodejs_app/`. The templates create Backstage Component
entities for ECS clusters (`container-cluster`) and ECS services (`container-service`), including
a development-container variant with an optional Node.js starter.

It governs template parameters, steps, rendering, publishing, catalog registration, and optional
scaffolder behavior. It does not govern the infrastructure or application design of generated
projects; each generated project's own constitution governs those concerns.

## Development Workflow and Quality Gates

- Before merge, run affected templates through the Backstage template editor dry run and render
  every affected option combination against Principle I.
- Changes to publishing, registration, or optional external steps MUST be exercised by a real
  run against a test component or sandbox. The pull request MUST list the checks and runs
  performed.
- Changes to template inputs, generated files, required catalog entities, required secrets,
  required Backstage actions, or other prerequisites MUST update the repository `README.md` in
  the same change; update a generated project README when it is affected.
- A new or changed scaffolder action MUST be identified in the README together with the Backstage
  module that provides it.
- Generated, operating-system, and editor files, including `.DS_Store`, `__pycache__`, and
  `node_modules`, MUST NOT be committed.
- Work MUST use `feature/` or `bug/` branches referencing an issue number when one exists.
  Commit messages MUST use conventional-commit style such as `feat:`, `fix:`, or `refactor:`.
  Changes MUST merge through pull requests.

These gates validate the rendered result and keep template documentation and prerequisites in
sync with the scaffolder behavior.

## Governance

This constitution supersedes conflicting conventions in this repository. Every specification,
plan, and pull request MUST check compliance with it. Amendments MUST be made in the same change
that introduces a new rule and MUST include a Sync Impact Report.

Versioning follows semantic versioning: MAJOR removes or incompatibly redefines a principle;
MINOR adds a principle or materially expands a section; PATCH clarifies wording or makes a
non-semantic refinement. Amendments and pull requests MUST describe their impact and verify
compliance with the resulting constitution.

**Version**: 1.0.0 | **Ratified**: 2026-10-29 | **Last Amended**: 2026-10-29
