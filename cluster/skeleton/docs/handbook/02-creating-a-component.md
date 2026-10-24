# Creating an ECS cluster via Backstage

Choose **Create** in Backstage and select **AWS ECS Cluster**. The platform
needs GitHub publishing, the custom entity-picker fields, catalog entities for
the selections below, and the AWS deployment workflow and account configuration.

## Page 1: ECS Cluster Info

Paths below are relative to the **newly generated repository root**, not this
template source repository. Catalog selections generally contribute the
entity's name rather than its display title; Account ID uses the exception
noted below.

| Input | Purpose | Where to see it after creation |
| --- | --- | --- |
| Namespace | Backstage namespace for the component; hidden and defaults to `default`. | `metadata.namespace` in `catalog-info.yaml`. |
| Cluster name | Base name for the component and GitHub repository. The template lowercases it and appends `-ecs-cluster`. | Repository name; Component `metadata.name` in `catalog-info.yaml`; `stack_name` in `samconfig.toml`; `Component` in `cfn-parameters.json` is derived from the repository name. |
| Description | Describes the component and cluster. | GitHub repository description; Component `metadata.description` in `catalog-info.yaml`; top-level `Description` in `template.yaml`. |
| Owner | Owning group selected from your groups. | Component `spec.owner` in `catalog-info.yaml`; GitHub Settings → Custom properties → `owner`. |
| Domain | Catalog domain for the component and deployment metadata. | `org:domain` in `cfn-tags.json`; `Domain` in `cfn-parameters.json`; GitHub Settings → Custom properties → `domain`. |
| System | Catalog system filtered by the selected domain. | Component `spec.system` in `catalog-info.yaml`; `org:system` in `cfn-tags.json`; `System` in `cfn-parameters.json`; `ServiceConnectNamespace` in `cfn-parameters.json` is `<system-name>.local`; GitHub Settings → Custom properties → `system`. |
| Environment | Catalog environment filtered by the selected system; narrows the cloud-account picker. | Scaffolder selection only. The generated workflow deploys to the GitHub Actions `production` environment, regardless of the selected catalog environment. |
| Account ID | Cloud-account resource related to the environment; its `metadata.title` supplies the AWS account ID. | GitHub Settings → Secrets and variables → Actions → repository secret `DEPLOYMENT_ACCOUNT_ID`. The ID is not written to application files. GitHub shows the secret's name, not its saved value; confirm the source catalog entity's title before creation. |

The hidden namespace defaults to `default`. The GitHub organization is
`ServerlessOpsIO`, and the repository is public with `main` as its default
branch. The selected account is used for deployment; it does not change the
generated template's standard network parameter paths.

## Page 2: ECS cluster configuration

| Input | Purpose | Where to see it after creation |
| --- | --- | --- |
| Create Load Balancer? | Adds an internet-facing Application Load Balancer, HTTPS listener, certificate, and DNS record when enabled. Defaults to `true`. | Conditional resources in `template.yaml`; `VpcSubnets`, `Hostname`, and `DnsZoneId` in `cfn-parameters.json`; `EcsAlbListenerArn` SSM parameter is created only when enabled. |
| Hostname | Fully qualified DNS name for the load balancer when it is enabled, for example `app.sandbox.serverlessops.io`. | `Hostname` in `cfn-parameters.json`; used for the ACM certificate domain and Route 53 alias record in `template.yaml`. |

With the load balancer enabled, the template reads public subnets from
`/org/networking/VpcPublicSubnets` and the hosted zone ID from
`/org/dns/ZoneId`. It creates a DNS-validated ACM certificate, a public ALB,
an HTTPS listener on port 443, and an alias A record for the hostname. The
listener currently returns a fixed `{"healthy": true}` response; it does not
route to an ECS service. The ALB security group accepts TCP 443 from any IPv4
address.

**Do not select No for the load balancer with the current template.** In that
mode the ALB security group resource is omitted, but the template still
declares an SSM parameter that references that security group. This leaves a
dangling resource reference and prevents a valid deployment. The No option
must be corrected in the template before it can be used.

The ECS cluster itself uses Service Connect namespace `<system-name>.local`.
The cluster template does not create ECS services, tasks, or compute capacity;
those are expected to be supplied by applications that use the cluster.
The generated `.github/workflows/main.yml` validates and builds the SAM
template, then deploys it on pushes to `main` or a manual workflow dispatch.
The deployment job uses the GitHub Actions `production` environment.

## Review and create

Review the component name, catalog selections, account, and hostname before
submitting. The scaffolder then:

1. Fetches the selected owner, domain, system, and cloud-account catalog entities.
2. Renders the cluster infrastructure and deployment workflow.
3. Publishes a **public** repository in `ServerlessOpsIO`, with `main` as the
   default branch, and sets the `DEPLOYMENT_ACCOUNT_ID` repository secret from
   the selected cloud-account entity's title.
4. Registers the generated Component in Backstage.

Default branch protection is disabled and the required approving review count
is zero in the publisher configuration. Apply your repository review policy
before development.

## After creation

- Confirm the component's owner, system, domain, and account relationships in
  the catalog.
- Ensure the deployment account has the VPC and (if enabled) public subnet and
  Route 53 hosted-zone SSM parameters expected by `cfn-parameters.json`.
- Ensure the GitHub Actions deployment prerequisites are configured, including
  the AWS CI/CD account secret and access to the `production` environment.
- Check the generated SSM parameters under
  `/<domain>/<system>/<component>/<branch>/`: `EcsClusterName`,
  `ClusterAlbSecurityGroupId`, and `ServiceConnectNamespace`; with the load
  balancer enabled, `EcsAlbListenerArn` is also created.
- Remember that selecting a catalog Environment does not choose a matching
  GitHub Actions environment; deployment uses `production`.
