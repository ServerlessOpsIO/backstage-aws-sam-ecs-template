# Project handbook

This handbook describes the AWS ECS cluster generated for
**${{ values.component_name }}**. Keep it current as the cluster infrastructure,
deployment assumptions, and workflows change.

- [Features](01-features.md): ECS cluster capabilities and exported integration points.
- [Creating a component](02-creating-a-component.md): Backstage form inputs and generated outputs.
- [Configuration](03-configuration.md): CloudFormation parameters, tags, and deployment prerequisites.
- [Architecture](04-architecture.md): AWS resources, relationships, and generated repository layout.
- [CI/CD pipeline](05-ci-cd.md): workflow triggers, build/deploy steps, and branch caveats.
- [Observability](06-observability.md): available AWS metrics and logging boundaries.
- [Common tasks](07-common-tasks.md): deploy, integrate a service, and change cluster settings.

## Reading this handbook

The pages can be read directly in GitHub from `docs/handbook/`. The generated
`catalog-info.yaml` does not currently set `backstage.io/techdocs-ref`, and the
repository does not include an `mkdocs.yml`; the handbook is therefore not
automatically configured for TechDocs. To publish it in Backstage, configure
the repository's TechDocs site and catalog annotation using the platform's
standard documentation setup.

Paths shown in backticks are relative to the generated repository root unless
otherwise noted. The cluster template is rendered from this repository's
`cluster/skeleton/` and its workflow from `cluster/pipeline/`.

## Content review

Review this starter handbook against your AWS account, network and DNS
conventions, repository policies, and operational requirements before relying
on it as a production runbook. Template-generated documentation does not
constitute human approval.
