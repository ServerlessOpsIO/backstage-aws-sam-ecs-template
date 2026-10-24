# Project handbook

This handbook describes the ECS service generated for **${{ values.component_name }}**.
Keep it current as the service's container, infrastructure, and workflows change.

- [Features](01-features.md): Fargate service behavior, load balancing, Service Connect, and Datadog.
- [Creating a component](02-creating-a-component.md): Backstage form pages and generated outputs.
- [Configuration](03-configuration.md): deployment parameters, tags, and prerequisites.
- [Architecture](04-architecture.md): AWS resources, task roles, networking, and repository layout.
- [CI/CD pipeline](05-ci-cd.md): image and infrastructure builds, deployment, and feature branches.
- [Observability](06-observability.md): CloudWatch logs, metrics, and optional Datadog integration.
- [Common tasks](07-common-tasks.md): update an image, configure a container, and validate changes.

## Reading in Backstage

The generated `catalog-info.yaml` currently does not declare a
`backstage.io/techdocs-ref` annotation, and the skeleton does not include a
repository-root `mkdocs.yml`. These handbook files can be read directly in
GitHub. To publish them in TechDocs, add the required TechDocs metadata and site
configuration and ensure the Backstage platform has configured a builder and
publisher. The application workflows do not build or publish documentation.

Application source paths shown in backticks are relative to the generated
repository root. The ECS template's source files live in this repository under
`container/skeleton/` and `container/pipeline/`.

## Content review

Before treating this starter handbook as the service's operational reference,
review its image source, network exposure, IAM permissions, deployment
assumptions, and runbooks. Template-generated documentation does not
constitute human approval.
