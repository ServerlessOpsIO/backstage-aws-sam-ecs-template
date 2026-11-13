# Configuration

## Stack parameters: `cfn-parameters.json`

This file supplies the deployment action with values for the `Parameters` in
`template.yaml`. The file is named **`cfn-parameters.json`**, without a leading
underscore.

| Parameter | Starter value/source | Meaning |
| --- | --- | --- |
| `Domain` | `${{ values.domain_name }}` | Selected catalog domain name. First segment of the SSM integration path. |
| `System` | `${{ values.system_name }}` | Selected catalog system name. Second segment of the SSM integration path. |
| `Component` | `$env.GITHUB_REPOSITORY_NAME_PART_SLUG_CS` | Repository-derived component identifier provided through the workflow environment. Third segment of the SSM integration path. |
| `CodeBranch` | `$env.GITHUB_REF_SLUG_CS` | Slug of the deployed ref. Fourth segment of the SSM integration path; services expect `main`. |
| `ServiceConnectNamespace` | `${{ values.system_name }}.local` | The cluster's Service Connect default namespace. |
| `VpcId` | `/org/networking/VpcId` | **SSM parameter name**, not the literal VPC ID; CloudFormation resolves it using `AWS::SSM::Parameter::Value<String>`. {% if values.create_load_balancer %}Used by the load balancer security group.{% else %}No generated resource uses it without the load balancer, but the parameter must still resolve.{% endif %} |
{%- if values.create_load_balancer %}
| `VpcSubnets` | `/org/networking/VpcPublicSubnets` | **SSM parameter name** resolving to a comma-delimited list of public subnet IDs for the load balancer. |
| `Hostname` | `${{ values.hostname }}` | Hostname from Backstage. Certificate domain name and DNS record name. |
| `DnsZoneId` | `/org/dns/ZoneId` | **SSM parameter name**, not the literal hosted zone ID; CloudFormation resolves it using `AWS::SSM::Parameter::Value<String>`. |
{%- endif %}

The `$env` expressions are intentionally unquoted in the starter file. It is
deployment-action input, **not strict JSON** for direct consumption by
`json.load`, `jq`, or the SAM CLI. The same applies to expressions in
`cfn-tags.json`. The workflow passes `env_json`, `vars_json`, and `secrets_json`
to `ServerlessOpsIO/gha-deploy-aws-sam@v1` for resolution.
Consult that action's configuration when changing expression syntax or adding
environment-specific values.

For a manual deployment, supply resolved CloudFormation parameter values through
SAM's parameter overrides; do not pass this file as though it were the AWS CLI's
parameter-list format. Use `Component=${{ values.component_name }}` and
`CodeBranch=main` so the SSM paths match what services read.

`Component` comes from the repository name, while services build the path from
this cluster's catalog entity name. The two match as created. Keep them
matching if you rename the repository or the catalog entity.

## Stack tags: `cfn-tags.json`

This file is also named without a leading underscore. It defines:

| Tag | Value |
| --- | --- |
| `org:domain` | `${{ values.domain_name }}` |
| `org:system` | `${{ values.system_name }}` |
| `org:component` | `$env.GITHUB_REPOSITORY_NAME_PART_SLUG_CS` |

Keep these aligned with catalog metadata and your organization's inventory and
cost-allocation conventions. Add tags through the existing deployment action;
CloudFormation propagates stack tags to resources that support them.
These metadata tags are not runtime configuration or access controls.

## Deployment prerequisites

{% if values.create_load_balancer -%}
- A target AWS account and deployment region with IAM permissions to create
  the ECS cluster, SSM parameters, security group, load balancer and listener,
  ACM certificate, and Route 53 record.
- A VPC with its ID stored in the deployment account/region's SSM parameter
  `/org/networking/VpcId`, and public subnets in at least two Availability
  Zones with their IDs stored as a comma-delimited list in
  `/org/networking/VpcPublicSubnets` (or your chosen parameter names).
- A Route 53 hosted zone authoritative for `${{ values.hostname }}`, with its
  ID stored in the SSM parameter `/org/dns/ZoneId` (or your chosen parameter
  name). DNS validation must be able to create certificate records; the stack
  waits until the certificate is issued.
{%- else -%}
- A target AWS account and deployment region with IAM permissions to create
  the ECS cluster and SSM parameters.
- A VPC with its ID stored in the deployment account/region's SSM parameter
  `/org/networking/VpcId` (or your chosen parameter name).
- The template change described in
  [Common tasks](07-common-tasks.md#make-the-template-deployable).
{%- endif %}
- GitHub Actions secrets `AWS_CICD_ACCOUNT_ID` and `DEPLOYMENT_ACCOUNT_ID`,
  available to the relevant jobs. The scaffolder only explicitly creates
  `DEPLOYMENT_ACCOUNT_ID`; configure the other through your organization's
  repository or environment settings.
- AWS OIDC trust and build/deployment roles expected by the shared
  `gha-assume-aws-credentials` action. Account IDs alone do not grant deployment
  access. Verify role names, trust restrictions, region, and artifact storage
  against the shared action's requirements.
- GitHub environment `production` settings appropriate for the deploy job,
  including any required approvals and branch restrictions.

Services created from the ECS service templates have their own prerequisites,
including `/org/networking/VpcPublicSubnets`; see each service's handbook.

{% if values.create_load_balancer %}
## Network exposure

The load balancer is internet-facing and accepts TCP 443 from any IPv4 address.
The template does not configure AWS WAF, load balancer access logs, deletion
protection, or a specific TLS security policy. Review this exposure against
your organization's network policy before routing services through it.

Service tasks are not reachable through this security group directly. Each
service stack that uses the load balancer adds an ingress rule to its own
security group allowing the load balancer's security group on the container
port.

{% endif %}
## SAM settings

`samconfig.toml` sets the default stack name to the component name,
`${{ values.component_name }}`, enables linting for `sam validate`, disables
changeset confirmation, and requests `CAPABILITY_NAMED_IAM` and
`CAPABILITY_AUTO_EXPAND`. It does not define separate branch environments.
Verify the final stack name used by the shared deployment action before
deploying.

The stack name is not branch-qualified, although `CodeBranch` is part of every
SSM parameter name. See [CI/CD pipeline](05-ci-cd.md#deploying-from-other-refs)
before deploying from any ref other than `main`.
