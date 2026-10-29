# Project handbook

This handbook describes the AWS ECS cluster generated for
**${{ values.component_name }}**. Keep it current as the cluster infrastructure,
its integration contract, and workflows change.

- [Features](01-features.md): ECS cluster, Service Connect, optional shared ALB, DNS/TLS, and the SSM integration contract.
- [Creating a component](02-creating-a-component.md): Backstage form pages and generated outputs.
- [Configuration](03-configuration.md): deployment parameters, tags, and prerequisites.
- [Architecture](04-architecture.md): AWS resources, request routing, template relationships, and repository layout.
- [CI/CD pipeline](05-ci-cd.md): build, deployment, and branch caveats.
- [Observability](06-observability.md): metrics, logging boundaries, and automatic Datadog integration.
- [Common tasks](07-common-tasks.md): validate, deploy, connect services, and change cluster settings.

## Reading in Backstage

The component's `catalog-info.yaml` contains `backstage.io/techdocs-ref: dir:.`.
TechDocs uses the repository-root `mkdocs.yml` and the Markdown files in
`docs/handbook/`.
Open the component in the catalog and select **Docs**.

The platform must enable the TechDocs frontend and backend and configure either
on-demand documentation generation or an external builder and publisher. The
builder needs MkDocs and `mkdocs-techdocs-core` for the `techdocs-core` plugin.
The generated cluster workflow does not build or publish documentation.
With the Backstage TechDocs CLI and its Docker prerequisites installed, preview
from the generated repository root with `techdocs-cli serve`.

The pages can also be read directly in GitHub. Paths shown in backticks are
relative to the generated repository root. The template source repository
keeps their inputs under `cluster/skeleton/` and `cluster/pipeline/`.

## Content review

Before treating this starter handbook as the cluster's operational reference,
review its network exposure, DNS and TLS configuration, deployment assumptions,
integration contract, and runbooks against your AWS account and organization
conventions. Template-generated documentation does not constitute human approval.
