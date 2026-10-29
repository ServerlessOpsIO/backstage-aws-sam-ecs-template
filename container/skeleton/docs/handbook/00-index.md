# Project handbook

This handbook describes the ECS service generated for **${{ values.component_name }}**.
Keep it current as the service's container, infrastructure, and workflows change.

- [Features](01-features.md): Fargate service, image source, load balancing, Service Connect, ECS Exec, and Datadog.
- [Creating a component](02-creating-a-component.md): Backstage form pages and generated outputs for both service templates.
- [Configuration](03-configuration.md): deployment parameters, tags, secrets, and prerequisites.
- [Architecture](04-architecture.md): AWS resources, request flows, task and repository anatomy.
- [CI/CD pipeline](05-ci-cd.md): infrastructure and image builds, deployment, and feature branches.
- [Observability](06-observability.md): logs, health, metrics, and automatic Datadog integration.
- [Common tasks](07-common-tasks.md): update the image, configure the container, grant permissions, and validate changes.

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

The pages can also be read directly in GitHub. Paths shown in backticks are
relative to the generated repository root. The template source repository
keeps their inputs under `container/skeleton/`, `container/pipeline/`, and
`container/nodejs_app/`.

## Which variant generated this repository?

Two Backstage templates generate this kind of repository, and parts of this
handbook apply to only one:

| Variant | Backstage template | How to recognize it |
| --- | --- | --- |
| **Image-based service** | AWS ECS Service Container | No `src/` directory; `ContainerImage` and `ContainerTag` in `cfn-parameters.json` are literal values. The workflows have no `build_container` job. |
| **Development application** | Containerized Development Application | `ContainerRepo` and `ContainerOrg` appear in `cfn-parameters.json`; the workflows have a `build_container` job; with Node.js selected, `src/` holds the starter application. |

## Content review

Before treating this starter handbook as the service's operational reference,
review its image source, network exposure, IAM permissions, deployment
assumptions, and runbooks. Template-generated documentation does not
constitute human approval.
