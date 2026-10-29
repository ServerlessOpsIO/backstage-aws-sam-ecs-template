# Creating an ECS cluster via Backstage

Choose **Create** in Backstage and select **AWS ECS Cluster**. The platform
needs GitHub publishing, catalog registration, the template's custom form
fields, and catalog entities for the choices below.

## Page 1: ECS Cluster Info

Paths below are relative to the **newly generated repository root**, not the
template source repository. Catalog selections generally contribute the
entity's name rather than its display title; Account ID uses the exception
noted below.

| Input | Purpose | Where to see it after creation |
| --- | --- | --- |
| Cluster name | Base name for the component and repository. The template lowercases it and appends `-ecs-cluster`. | GitHub repository name; Component `metadata.name` and `github.com/project-slug` in `catalog-info.yaml`; `stack_name` in `samconfig.toml`; headings in `README.md`; `site_name` in `mkdocs.yml`; the lowercased base name in the top-level `Description` of `template.yaml`. `Component` in `cfn-parameters.json` and `org:component` in `cfn-tags.json` resolve from the repository-name workflow variable rather than a literal form value. |
| Description | Describes the repository, component, and cluster. | GitHub repository description; Component `metadata.description` in `catalog-info.yaml`; top-level `Description` in `template.yaml`; `README.md`. |
| Owner | Owning group selected from your groups. | Component `spec.owner` in `catalog-info.yaml` (full entity reference); GitHub repository Settings → Custom properties → `owner` contains the group's entity name. |
| Domain | Catalog domain, also used for deployment metadata and tagging. | `Domain` in `cfn-parameters.json`; `org:domain` in `cfn-tags.json`; the domain title in `README.md`; GitHub Settings → Custom properties → `domain`. There is no direct domain relationship in `catalog-info.yaml`. |
| System | Catalog system filtered by the chosen domain. | Component `spec.system` in `catalog-info.yaml`; `System` in `cfn-parameters.json`; `org:system` in `cfn-tags.json`; `ServiceConnectNamespace` in `cfn-parameters.json` (`<system-name>.local`); the system title in `README.md` and `template.yaml`'s `Description`; GitHub Settings → Custom properties → `system`. |
| Environment | Catalog environment filtered by the chosen system; narrows the cloud-account picker. | Scaffolder selection only: not written to a generated file. `.github/workflows/main.yml` independently sets `jobs.deploy.environment: production`. |
| Account ID | Catalog cloud-account resource related to the environment; its `metadata.title` must contain the AWS account ID. | `spec.dependsOn` in `catalog-info.yaml` references the account entity. GitHub Settings → Secrets and variables → Actions → repository secret `DEPLOYMENT_ACCOUNT_ID` holds the title. GitHub shows the secret's name but not its value; confirm the source entity's title before creation. |

Hidden defaults are namespace `default`, lifecycle `production`, and GitHub
organization `ServerlessOpsIO`. Unlike some other templates, the namespace is
written to `metadata.namespace` in `catalog-info.yaml`. Lifecycle appears in
`spec.lifecycle`; the organization appears in the repository URL and
`github.com/project-slug`. The component's `spec.type` is `container-cluster`;
the ECS service templates use that type to offer this cluster in their
**Cluster** picker.

Choosing an Environment does not configure a matching GitHub environment: the
generated workflow always uses `production`.

## Page 2: ECS cluster Configuration

| Input | Purpose | Where to see it after creation |
| --- | --- | --- |
| Create Load Balancer? | Adds a shared internet-facing ALB, HTTPS listener, certificate, and DNS record. Defaults to `true`. **Do not select No** with the current template; see [Features](01-features.md#load-balancer-disabled-option). | Conditional resources and parameters in `template.yaml`; `VpcSubnets`, `Hostname`, and `DnsZoneId` in `cfn-parameters.json`; the `EcsAlbListenerArn` SSM parameter. |
| Hostname | Fully qualified DNS name for the ALB, for example `app.sandbox.serverlessops.io`. Shown only when the ALB is enabled. | Literal `Hostname` in `cfn-parameters.json`. In `template.yaml`, `SiteCertificate.Properties.DomainName` and `SiteDnsRecord.Properties.Name` reference the parameter. |

The hostname must belong to the hosted zone whose ID is stored in
`/org/dns/ZoneId` in the deployment account and region. Every service that
uses the shared ALB is reached through this one hostname and is distinguished
by its listener-rule path.

These values are copied into generated files at creation time, not maintained
by a single live configuration source. If you change the hostname or ALB option
later, update `cfn-parameters.json` and `template.yaml` together and review
[Configuration](03-configuration.md).

## Review and create

Review the form summary, especially the cluster name, system, account, and
hostname, before submitting. The scaffolder then:

1. Fetches the selected owner, domain, system, and cloud-account catalog
   entities.
2. Renders the cluster infrastructure, handbook, and deployment workflow.
3. Publishes a **public** repository in `ServerlessOpsIO` with `main` as the
   default branch, sets `DEPLOYMENT_ACCOUNT_ID`, and sets the `system`,
   `domain`, and `owner` custom properties.
4. Registers the generated Component in Backstage.

The results link to the repository and the catalog component. Default branch
protection is disabled and the required approving review count is zero in the
publisher configuration; apply your review policy before development.

## After creation

- Confirm catalog ownership and relationships.
- Complete the [deployment prerequisites](03-configuration.md#deployment-prerequisites).
  The initial commit to `main` starts a deployment immediately.
- After a successful deployment, check the SSM parameters under
  `/<domain>/<system>/<component>/main/`; see
  [Features](01-features.md#ssm-integration-contract).
- Open the component's **Docs** tab, or read `docs/handbook/00-index.md` in the
  repository, and request human review of the generated handbook.
- Create services with the **AWS ECS Service Container** or **Containerized
  Development Application** template, selecting this cluster.
