# Creating an ECS service via Backstage

Choose **Create** in Backstage and select either template:

- **AWS ECS Service Container** runs an existing container image.
- **Containerized Development Application** builds an image from source in the
  generated repository.

The platform needs GitHub publishing, catalog registration, the template's
custom form fields, and catalog entities for the choices below, including an
ECS cluster component created with the **AWS ECS Cluster** template.

## Page 1: ECS Service Info

Both templates share this page. Paths below are relative to the **newly
generated repository root**, not the template source repository. Catalog
selections generally contribute the entity's name rather than its display
title; Account ID uses the exception noted below.

| Input | Purpose | Where to see it after creation |
| --- | --- | --- |
| ECS Service Name | Base name for the component and repository. The template lowercases it and appends `-app`. | GitHub repository name; Component `metadata.name` and `github.com/project-slug` in `catalog-info.yaml`; `stack_name` in `samconfig.toml`; headings in `README.md` and this handbook; `site_name` in `mkdocs.yml`; the lowercased base name in `template.yaml`'s `Description`. The ECS service name, container name, and Service Connect alias use `Component` in `cfn-parameters.json`, which resolves from the repository-name workflow variable. |
| Description | Describes the repository, component, and service. | GitHub repository description; Component `metadata.description` in `catalog-info.yaml`; `template.yaml`'s `Description`; `README.md`. In the development variant, also `description` in `src/package.json`. |
| Owner | Owning group selected from your groups. | Component `spec.owner` in `catalog-info.yaml`; GitHub Settings → Custom properties → `owner`. |
| Domain | Catalog domain, also used for deployment metadata, tagging, and the cluster SSM paths. | `Domain` and the first segment of each `Cluster*` SSM path in `cfn-parameters.json`; `org:domain` in `cfn-tags.json`; GitHub Settings → Custom properties → `domain`. |
| System | Catalog system filtered by the chosen domain; also filters the Cluster picker. | Component `spec.system` in `catalog-info.yaml`; `System` and the second segment of each `Cluster*` SSM path in `cfn-parameters.json`; `org:system` in `cfn-tags.json`; GitHub Settings → Custom properties → `system`. |
| Environment | Catalog environment filtered by the chosen system; narrows the cloud-account picker. | Scaffolder selection only: not written to a generated file. The `main` workflow's deploy job uses the GitHub environment `production`. |
| Account ID | Catalog cloud-account resource related to the environment; its `metadata.title` must contain the AWS account ID. | `spec.dependsOn` in `catalog-info.yaml`; repository secret `DEPLOYMENT_ACCOUNT_ID` holds the title. GitHub does not reveal saved secret values; confirm the entity's title before creation. |
| Cluster | A `container-cluster` component in the selected system. | `spec.dependsOn` in `catalog-info.yaml`; the third segment of each `Cluster*` SSM path in `cfn-parameters.json`. |

Hidden defaults are namespace `default` (written to `metadata.namespace`),
lifecycle `production`, and GitHub organization `ServerlessOpsIO`. The
component's `spec.type` is `container-service`.

The cluster SSM paths are built from **this service's** domain and system, the
cluster's entity name, and a fixed `main` branch segment:
`/<domain>/<system>/<cluster>/main/<parameter>`. The cluster must therefore be
deployed from `main` with the same domain and system, in the same account and
region as this service.

Choosing an Environment does not configure a matching GitHub environment.

## Page 2: Container Configuration (image-based service)

| Input | Purpose | Where to see it after creation |
| --- | --- | --- |
| Container Image | Image repository to run, for example an ECR repository URI, without a tag. | `ContainerImage` in `cfn-parameters.json`; the application container's `Image` in `template.yaml` as `<image>:<tag>`. |
| Container Tag | Image tag; defaults to `latest`. Prefer an immutable version tag. | `ContainerTag` in `cfn-parameters.json`. |
| Container CPU shares | Task CPU units; defaults to `1024` (one vCPU). | `ContainerCpu` in `cfn-parameters.json`; `TaskDefinition.Properties.Cpu`. |
| Container Memory | Task memory in MB; defaults to `2048`. | `ContainerMemory` in `cfn-parameters.json`; `TaskDefinition.Properties.Memory`. |
| Container Count | Desired number of tasks; defaults to `3`, one per Availability Zone. | `ContainerDesiredCount` in `cfn-parameters.json`; `Service.Properties.DesiredCount`. |

## Page 2: ECS Service Configuration (development application)

| Input | Purpose | Where to see it after creation |
| --- | --- | --- |
| Application Language | `nodejs` (default) or `python`. | With `nodejs`, the Node.js starter in `src/` and the Node.js setup, compile, test, and lint steps in both workflows. **`python` has no starter yet**: no `src/` is generated, so the image build fails until you add `src/Dockerfile`. |
| Container CPU shares, Container Memory, Container Count | As above. | As above. |

The development variant has no image or tag inputs. `cfn-parameters.json`
resolves them at deployment from the workflow (`AWS_ECR_REPO` secret,
repository owner and name, and commit SHA).

Task CPU and memory must form a valid Fargate combination, for example 1024
CPU with 2048–8192 MB. With Datadog enabled, the sidecar takes 256 CPU units
and 512 MB of the task's allocation.

## Page 3: Infrastructure Configuration

| Input | Purpose | Where to see it after creation |
| --- | --- | --- |
| Enable Datadog integration? | Defaults to `true`. Adds the Datadog Agent sidecar. | `DatadogApiKey` and `DatadogSite` in `cfn-parameters.json` and `template.yaml`; the `datadog-agent` container in the task definition. |
| Use cluster load balancer? | Defaults to `true`. Attaches the service to the cluster's ALB. **Do not select No**; see [Features](01-features.md#load-balancer-disabled-option). | `ClusterAlbListenerArn`, `ClusterAlbSecurityGroupId`, `VpcId`, `AlbRulePath`, and `AlbRulePriority` in `cfn-parameters.json`; `AlbTargetGroup`, `AlbListenerRule`, `ServiceSecurityGroupAlbIngress`, and the service's `LoadBalancers` in `template.yaml`. |
| Container port | Port the application listens on. Required with the load balancer. | `ContainerPort` in `cfn-parameters.json`; port mapping, Service Connect alias, target group, and ingress rule in `template.yaml`. |
| Load balancer path | Listener path pattern; defaults to `*` (all paths). For example, `/app/*`. | `AlbRulePath` in `cfn-parameters.json`. |
| Load balancer rule priority | Defaults to `1`. Lower values are evaluated first; must be unique on the cluster listener. | `AlbRulePriority` in `cfn-parameters.json`. |

The defaults `*` and priority `1` claim **all** traffic on the shared listener
ahead of every other service, and a second service with priority `1` fails to
deploy. Choose a specific path and an unused priority when the cluster hosts
more than one service.

These values are copied into generated files at creation time. If you change
the port, path, or priority later, update `cfn-parameters.json` and the
application together.

## Review and create

Review the form summary, especially the service name, system, cluster,
account, image, port, path, and priority, before submitting. The scaffolder
then:

1. Fetches the selected owner, domain, system, account, and cluster entities.
2. Renders the service infrastructure, handbook, and GitHub Actions workflows,
   plus the Node.js starter for a Node.js development application.
3. Publishes a **public** repository in `ServerlessOpsIO` with `main` as the
   default branch, sets `DEPLOYMENT_ACCOUNT_ID`, and sets the `system`,
   `domain`, and `owner` custom properties.
4. Registers the generated Component in Backstage.

Default branch protection is disabled and the required approving review count
is zero in the publisher configuration; apply your review policy before
development.

## After creation

- Confirm catalog ownership and relationships, including the cluster
  dependency.
- Complete the [deployment prerequisites](03-configuration.md#deployment-prerequisites).
  The initial commit to `main` starts a deployment immediately, and without the
  required secrets or a pullable image it fails.
- Review [branch behavior](05-ci-cd.md#feature-branches-and-stack-identity)
  before pushing additional branches.
- Open the component's **Docs** tab, or read `docs/handbook/00-index.md` in the
  repository, and request human review of the generated handbook.
