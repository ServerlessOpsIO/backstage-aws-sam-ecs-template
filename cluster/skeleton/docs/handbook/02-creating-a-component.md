# Creating a component via Backstage

Choose **Create** in Backstage and select **AWS ECS Cluster**.
The platform needs GitHub publishing, catalog registration, the template's
custom form fields, and catalog entities for the choices below.

## Page 1: ECS Cluster Info

Paths below are relative to the **newly generated repository root**, not the
template source repository. Catalog selections generally contribute the entity's
name rather than its display title; Account ID uses the exception noted below.

| Input | Purpose | Where to see it after creation |
| --- | --- | --- |
| Cluster name | Base name of the cluster. The template lowercases it and appends `-ecs-cluster` to form the component and repository name; this cluster is `${{ values.component_name }}`. | GitHub repository name; Component `metadata.name` and repository reference in `catalog-info.yaml`; `stack_name` in `samconfig.toml`; headings in `README.md`, `docs/index.md`, and `docs/handbook/00-index.md`. The lowercased base name alone appears in the top-level `Description` of `template.yaml`. `Component` in `cfn-parameters.json` and `org:component` in `cfn-tags.json` resolve from the repository-name workflow variable rather than a literal form value. |
| Description | Describes the repository, component, and SAM application. | GitHub repository description; Component `metadata.description` in `catalog-info.yaml`; top-level `Description` in `template.yaml`; introductory text in `README.md`. |
| Owner | Owning group selected from your groups. | Component `spec.owner` in `catalog-info.yaml` holds the selected group's entity reference; GitHub repository Settings → Custom properties → `owner` contains the group's entity name. |
| Domain | Catalog domain, also used for deployment metadata, tagging, and the SSM path. | `Domain` in `cfn-parameters.json`; `org:domain` in `cfn-tags.json`; the domain title in `README.md`; GitHub Settings → Custom properties → `domain`. There is no direct domain relationship in the generated `catalog-info.yaml`. |
| System | Catalog system filtered by the chosen domain. It also names the Service Connect namespace and decides which services can select this cluster. | Component `spec.system` in `catalog-info.yaml`; `System` and `ServiceConnectNamespace` in `cfn-parameters.json`; `org:system` in `cfn-tags.json`; the system title in `README.md` and the `Description` of `template.yaml`; GitHub Settings → Custom properties → `system`. |
| Environment | Catalog environment filtered by the chosen system; narrows the cloud-account picker. | Scaffolder selection only: the chosen environment is not written to a generated file. `.github/workflows/main.yml` independently sets `jobs.deploy.environment: production`. |
| Account ID | Catalog cloud-account resource related to the environment; its `metadata.title` must contain the AWS account ID. | Component `spec.dependsOn` in `catalog-info.yaml` references the account entity. GitHub Settings → Secrets and variables → Actions → repository secret `DEPLOYMENT_ACCOUNT_ID` holds its title; the workflow references it in the deploy job. The account ID is not written into repository files. GitHub shows the secret's name but does not reveal its saved value; confirm the source catalog entity's title before creation. |

Hidden defaults are namespace `default`, lifecycle `production`, and GitHub
organization `ServerlessOpsIO`. The namespace is written to
`metadata.namespace` and lifecycle to `spec.lifecycle` in `catalog-info.yaml`.
The GitHub organization is visible in the repository URL and
`metadata.annotations.github.com/project-slug`.

The generated Component has `spec.type: container-cluster`. The ECS service
templates offer components of that type, in the service's selected system, in
their **Cluster** picker.

Choosing an Environment does not configure a matching GitHub environment:
the generated workflow explicitly uses `production`.

## Page 2: ECS cluster Configuration

| Input | Purpose | Where to see it after creation |
| --- | --- | --- |
| Create Load Balancer? | Whether the stack owns a shared internet-facing load balancer with an HTTPS listener, certificate, and DNS record. Defaults to Yes. | The `EcsAlb*`, `SiteCertificate`, and `SiteDnsRecord` resources and the `VpcSubnets`, `Hostname`, and `DnsZoneId` parameters in `template.yaml`; the matching entries in `cfn-parameters.json`. These are present or absent as a whole: the template is generated for the chosen option and has no CloudFormation `Conditions`. This handbook is generated for the chosen option too. |
| Hostname | Fully qualified DNS name of the load balancer, such as `app.sandbox.serverlessops.io`. Shown only when the load balancer is enabled. | Literal `Hostname` in `cfn-parameters.json`. In `template.yaml`, `SiteCertificate.Properties.DomainName` and `SiteDnsRecord.Properties.Name` reference this parameter. |

{% if values.create_load_balancer %}
This cluster was created with the load balancer **enabled** and hostname
`${{ values.hostname }}`. The form does not mark Hostname as required; if the
value shown here is empty, set `Hostname` in `cfn-parameters.json` before the
stack can deploy. The hostname must belong to the hosted zone described in
[Configuration](03-configuration.md).
{% else %}
This cluster was created with the load balancer **disabled**. That option
currently generates a `template.yaml` that fails validation; follow
[Common tasks](07-common-tasks.md#make-the-template-deployable) before
expecting a deployment.
{% endif %}

These values are copied into multiple files at creation time, not maintained by
a single live configuration source. Renaming the repository, or changing the
domain or system later, changes the SSM paths that services read; see
[Features](01-features.md#ssm-integration-contract).

## Review and create

Review the form summary, especially cluster name, system, account, and
hostname, before submitting. The scaffolder then:

1. Fetches the selected owner, domain, system, and account catalog entities.
2. Copies the cluster infrastructure and handbook, then the GitHub Actions
   workflow.
3. Publishes a **public** repository with `main` as the default branch, sets
   `DEPLOYMENT_ACCOUNT_ID`, and sets the `system`, `domain`, and `owner`
   custom properties.
4. Registers the generated Component in Backstage.

The template defines no result links; find the repository in the
`ServerlessOpsIO` organization and the component in the catalog by name.
Default branch protection is disabled and the required approving review count
is zero in the publisher configuration; apply your review policy before
development.

## After creation

- Confirm catalog ownership and relationships.
- Complete [deployment configuration](03-configuration.md) before assuming the
  initial push can deploy successfully. The initial commit to `main` starts the
  workflow immediately.
- Review [non-main deployments](05-ci-cd.md#deploying-from-other-refs) before
  dispatching the workflow manually.
- Open the component's **Docs** tab, or read `docs/handbook/00-index.md` in the repository.
- After the first successful deployment, create services with the
  **AWS ECS Service Container** or **Containerized Development Application**
  template as described in [Common tasks](07-common-tasks.md), and request
  human review of the generated handbook.
