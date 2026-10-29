# Configuration

## Stack parameters: `cfn-parameters.json`

This file provides deployment-action values for the CloudFormation parameters
declared in `template.yaml`.

| Parameter | Starter value/source | Meaning |
| --- | --- | --- |
| `Domain` | Selected catalog domain name | First component of the cluster's SSM parameter path and deployment metadata. |
| `System` | Selected catalog system name | Second component of the SSM path and deployment metadata. |
| `Component` | `$env.GITHUB_REPOSITORY_NAME_PART_SLUG_CS` | Repository-derived component identifier supplied by the workflow environment. |
| `CodeBranch` | `$env.GITHUB_REF_SLUG_CS` | Branch slug included in the SSM parameter path. |
| `VpcId` | `/org/networking/VpcId` | SSM parameter name resolved as the VPC ID. |
| `ServiceConnectNamespace` | `<system-name>.local` | Cluster's Service Connect default namespace. |
| `VpcSubnets` | `/org/networking/VpcPublicSubnets` | SSM parameter name resolving the comma-delimited public subnets; only included when the ALB is enabled. |
| `Hostname` | Hostname entered in Backstage | ACM certificate domain and Route 53 record name; only included when the ALB is enabled. |
| `DnsZoneId` | `/org/dns/ZoneId` | SSM parameter name resolving the Route 53 hosted-zone ID; only included when the ALB is enabled. |

The `$env` expression is intentionally unquoted in the starter file. This is
deployment-action input, not strict JSON for direct use with `json.load`,
`jq`, or the SAM CLI. The same applies to expressions in `cfn-tags.json`. The
workflow passes serialized environment, variables, and secrets to
`ServerlessOpsIO/gha-deploy-aws-sam@v1` for resolution. Consult that action's
configuration before changing expression syntax.

`VpcId`, `VpcSubnets`, and `DnsZoneId` are SSM parameter names, not literal
resource IDs. CloudFormation resolves the values using the declared
`AWS::SSM::Parameter::Value<...>` parameter types. The target account and
deployment region must contain these parameters.

## Stack tags: `cfn-tags.json`

The template supplies these organizational tags:

| Tag | Value |
| --- | --- |
| `org:domain` | Selected domain name |
| `org:system` | Selected system name |
| `org:component` | `$env.GITHUB_REPOSITORY_NAME_PART_SLUG_CS` |

Keep these aligned with catalog metadata and inventory/cost-allocation
conventions. They are metadata, not runtime configuration or access controls.

## Deployment prerequisites

- A target AWS account and region with permissions to create the ECS cluster,
  SSM parameters, and (when enabled) EC2 security groups, an ALB, ACM
  certificate, and Route 53 record.
- AWS SSM parameters `/org/networking/VpcId` and, with the ALB enabled,
  `/org/networking/VpcPublicSubnets` and `/org/dns/ZoneId`.
- A VPC with public subnets suitable for an internet-facing ALB. The Route 53
  hosted zone must be authoritative for the selected hostname, and certificate
  DNS validation must be able to create validation records.
- GitHub Actions secret `AWS_CICD_ACCOUNT_ID` and the repository secret
  `DEPLOYMENT_ACCOUNT_ID`. The scaffolder explicitly creates the latter from
  the selected account catalog entity's title.
- AWS OIDC trust and build/deployment roles expected by
  `gha-assume-aws-credentials`. Account IDs alone do not grant AWS access.
- The GitHub Actions environment named `production`, used by the deployment
  job, with suitable protection rules and access.

The ALB security group accepts public IPv4 traffic on 443. Review this exposure
against your organization's network policy. The template does not configure
ALB access-log delivery.

## SAM and stack settings

`samconfig.toml` sets `stack_name` to the generated component name and enables
SAM build caching and parallelism. The deploy action uses this SAM template and
configuration; confirm final stack-name and region behavior against the shared
deployment action before deploying.

`CodeBranch` appears in the SSM path, but the stack name is not branch-qualified
by the generated SAM configuration. See [CI/CD pipeline](05-ci-cd.md) before
dispatching deployments from non-main refs.

## Load balancer option

The Backstage form defaults **Create Load Balancer?** to true. With it enabled,
the template expects the subnet, hostname, and DNS-zone values above. Do not
disable it until `ClusterAlbSecurityGroupIdSsmParam` in `template.yaml` is
conditioned consistently with the `EcsAlbSecurityGroup` resource; otherwise
CloudFormation sees a reference to a resource that was not created.
