# Configuration

## Stack parameters: `cfn-parameters.json`

This file supplies the deployment action with values for the `Parameters` in
`template.yaml`. The file is named **`cfn-parameters.json`**, without a leading
underscore.

| Parameter | Starter value/source | Meaning |
| --- | --- | --- |
| `Domain` | Selected catalog domain name | First segment of the SSM integration path; deployment metadata. |
| `System` | Selected catalog system name | Second segment of the SSM path; deployment metadata. |
| `Component` | `$env.GITHUB_REPOSITORY_NAME_PART_SLUG_CS` | Repository-derived component identifier provided through the workflow environment; third SSM path segment. |
| `CodeBranch` | `$env.GITHUB_REF_SLUG_CS` | Branch slug; fourth SSM path segment. |
| `VpcId` | `/org/networking/VpcId` | **SSM parameter name**, resolved through `AWS::SSM::Parameter::Value<String>`. Used by the ALB security group. |
| `ServiceConnectNamespace` | `<system-name>.local` | Cluster's Service Connect default namespace. |
| `VpcSubnets` | `/org/networking/VpcPublicSubnets` | SSM parameter name resolving a comma-delimited list of public subnets. ALB only. |
| `Hostname` | Hostname from Backstage | Certificate domain and DNS record name. ALB only. |
| `DnsZoneId` | `/org/dns/ZoneId` | SSM parameter name resolving the Route 53 hosted zone ID. ALB only. |

The `$env` expressions are intentionally unquoted in the starter file. It is
deployment-action input, **not strict JSON** for direct consumption by
`json.load`, `jq`, or the SAM CLI. The same applies to expressions in
`cfn-tags.json`. The workflow passes `env_json`, `vars_json`, and
`secrets_json` to `ServerlessOpsIO/gha-deploy-aws-sam@v1` for resolution.
Consult that action's configuration when changing expression syntax or adding
environment-specific values.

For a manual deployment, supply resolved CloudFormation parameter values
through SAM's parameter overrides; do not pass this file as though it were the
AWS CLI's parameter-list format.

## Stack tags: `cfn-tags.json`

This file is also named without a leading underscore. It defines:

| Tag | Value |
| --- | --- |
| `org:domain` | Selected domain name |
| `org:system` | Selected system name |
| `org:component` | `$env.GITHUB_REPOSITORY_NAME_PART_SLUG_CS` |

Keep these aligned with catalog metadata and your organization's inventory and
cost-allocation conventions. CloudFormation propagates stack tags to resources
that support them. These metadata tags are not runtime configuration or access
controls.

## Deployment prerequisites

- A target AWS account and region with permissions to create the ECS cluster,
  SSM parameters, and, with the ALB, EC2 security groups, the load balancer and
  listener, ACM certificate, and Route 53 record.
- SSM parameters in that account and region: `/org/networking/VpcId` always,
  and with the ALB, `/org/networking/VpcPublicSubnets` and `/org/dns/ZoneId`.
- Public subnets in at least two Availability Zones, as required for an
  Application Load Balancer.
- A Route 53 hosted zone authoritative for the hostname. DNS validation must be
  able to create certificate records; the stack waits for validation to finish.
- GitHub Actions secrets `AWS_CICD_ACCOUNT_ID` and `DEPLOYMENT_ACCOUNT_ID`. The
  scaffolder only creates `DEPLOYMENT_ACCOUNT_ID`; configure the other through
  your organization's repository or environment settings.
- AWS OIDC trust and build/deployment roles expected by the shared
  `gha-assume-aws-credentials` action. Account IDs alone do not grant
  deployment access. Verify role names, trust restrictions, region, and
  artifact storage against the shared action's requirements.
- GitHub environment `production` settings appropriate for the deploy job,
  including any required approvals and branch restrictions.

The ALB accepts public IPv4 traffic on 443. Review this exposure against your
organization's network policy. The template does not configure ALB access
logs, AWS WAF, or deletion protection.

## SAM and stack settings

`samconfig.toml` sets the default stack name to the component name, enables
cached parallel builds and linting, disables changeset confirmation, and
requests `CAPABILITY_NAMED_IAM` and `CAPABILITY_AUTO_EXPAND`. It does not
define separate branch environments. Verify the final stack name used by the
shared deployment action before deploying.

`CodeBranch` appears in the SSM path, but the stack name is not
branch-qualified. See [CI/CD pipeline](05-ci-cd.md) before dispatching
deployments from non-main refs.
