# Configuration

## Stack parameters: `cfn-parameters.json`

This file supplies the deployment action with values for the `Parameters` in
`template.yaml`. The file is named **`cfn-parameters.json`**, without a leading
underscore. Which entries exist depends on the options selected in Backstage.

| Parameter | Starter value/source | Meaning |
| --- | --- | --- |
| `Domain`, `System` | Selected catalog domain and system names | Deployment metadata and Datadog Docker labels. |
| `Component` | `$env.GITHUB_REPOSITORY_NAME_PART_SLUG_CS` | Service name, container name, Service Connect name, and Docker label. |
| `CodeBranch` | `$env.GITHUB_REF_SLUG_CS` | Task-definition family suffix and Docker label. |
| `ClusterName` | `/<domain>/<system>/<cluster>/main/EcsClusterName` | **SSM parameter name** resolving the cluster name. |
| `ClusterServiceConnectNamespace` | `/<domain>/<system>/<cluster>/main/ServiceConnectNamespace` | SSM parameter name resolving the Service Connect namespace. |
| `ClusterSecurityGroupId` | `/<domain>/<system>/<cluster>/main/ContainerSecurityGroupId` | **Stale entry**: `template.yaml` declares no such parameter and the cluster publishes no such SSM value. See the note below. |
| `ClusterAlbListenerArn` | `/<domain>/<system>/<cluster>/main/EcsAlbListenerArn` | Load balancer only. Cluster HTTPS listener. |
| `ClusterAlbSecurityGroupId` | `/<domain>/<system>/<cluster>/main/ClusterAlbSecurityGroupId` | Load balancer only. Source of allowed ingress to tasks. |
| `ContainerImage`, `ContainerTag` | Image-based: Backstage values. Development: `$env.GITHUB_REPOSITORY_NAME_PART_SLUG` and `$env.GITHUB_SHA`. | Image repository and tag. |
| `ContainerRepo`, `ContainerOrg` | Development only: `$secrets.AWS_ECR_REPO` and `$env.GITHUB_REPOSITORY_OWNER_PART_SLUG` | ECR registry and repository namespace prefixed to the image. |
| `ContainerCpu`, `ContainerMemory`, `ContainerDesiredCount` | Backstage values | Task size and desired tasks. |
| `EnableContainerExec` | `"false"` | ECS Exec toggle. |
| `DatadogApiKey`, `DatadogSite` | Datadog only: `$secrets.DATADOG_AGENT_API_KEY`, `$secrets.DATADOG_SITE` | Agent credentials and site. |
| `ContainerPort` | Backstage value, when a port was entered | Port mapping, Service Connect, target group, and ingress. |
| `AlbRulePriority`, `AlbRulePath` | Load balancer only: Backstage values | Listener rule priority and path pattern. |
| `VpcId` | Load balancer only: `/org/networking/VpcId` | SSM parameter name resolving the VPC for the security group and target group. |
| `VpcSubnets` | `/org/networking/VpcPublicSubnets` | SSM parameter name resolving the task subnets. |

`ClusterName`, `ClusterServiceConnectNamespace`, `ClusterAlb*`, `VpcId`, and
`VpcSubnets` are SSM parameter names, resolved by CloudFormation through
`AWS::SSM::Parameter::Value<...>` types at every deployment. A missing parameter
fails the deployment before any resource changes.

`ClusterSecurityGroupId` is passed to the deployment action but has no matching
template parameter. Whether that is ignored or rejected depends on how the
shared deployment action builds parameter overrides; if deployments fail on an
unknown parameter, remove the entry.

The `$env` and `$secrets` expressions are intentionally unquoted in the starter
file. It is deployment-action input, **not strict JSON** for direct consumption
by `json.load`, `jq`, or the SAM CLI. The same applies to expressions in
`cfn-tags.json`. The workflow passes `env_json`, `vars_json`, and
`secrets_json` to `ServerlessOpsIO/gha-deploy-aws-sam@v1` for resolution.
Consult that action's configuration when changing expression syntax or adding
environment-specific values.

For a manual deployment, supply resolved CloudFormation parameter values
through SAM's parameter overrides. Do not replace `$secrets` references with
committed credentials.

## Stack tags: `cfn-tags.json`

This file is also named without a leading underscore. It defines:

| Tag | Value |
| --- | --- |
| `org:domain` | Selected domain name |
| `org:system` | Selected system name |
| `org:component` | `$env.GITHUB_REPOSITORY_NAME_PART_SLUG_CS` |

Keep these aligned with catalog metadata and your organization's inventory and
cost-allocation conventions. CloudFormation propagates stack tags to resources
that support them, and the service propagates task-definition tags to tasks.
These metadata tags are not runtime configuration or access controls.

## Deployment prerequisites

- A target AWS account and region with the IAM permissions required by the
  generated ECS, EC2, IAM, and Elastic Load Balancing resources.
- The selected cluster, deployed from `main` in the same account and region,
  with its SSM parameters under `/<domain>/<system>/<cluster>/main/`. With the
  load balancer, the cluster must have been created with its ALB.
- SSM parameters `/org/networking/VpcPublicSubnets` and, with the load
  balancer, `/org/networking/VpcId`.
- An image the task execution role can pull. For an image-based service in
  another account's ECR, that repository's policy must allow the deployment
  account. Images from other registries may need repository credentials, which
  the template does not configure.
- GitHub Actions secrets: `AWS_CICD_ACCOUNT_ID` and `DEPLOYMENT_ACCOUNT_ID`
  always; `AWS_ECR_REPO` for the development variant; `DATADOG_AGENT_API_KEY`
  and `DATADOG_SITE` with Datadog. The scaffolder only creates
  `DEPLOYMENT_ACCOUNT_ID`; configure the others through your organization's
  repository or environment settings.
- AWS OIDC trust and build/deployment roles expected by the shared
  `gha-assume-aws-credentials` action. Account IDs alone do not grant
  deployment access.
- GitHub environment `production` settings appropriate for the deploy job,
  including any required approvals and branch restrictions.

Tasks run in the public subnets with `AssignPublicIp: ENABLED`, which they need
to reach ECR and CloudWatch without NAT or VPC endpoints. The security group
only admits the ALB on the container port, but review public IP assignment
against your network policy.

## SAM and runtime settings

`samconfig.toml` sets the default stack name to the component name, enables
cached parallel builds and linting, disables changeset confirmation, and
requests `CAPABILITY_NAMED_IAM` and `CAPABILITY_AUTO_EXPAND`. It does not
define separate branch environments. Verify the final stack name used by the
shared deployment action.

The image defines its own runtime, command, and listening port; the task
definition sets none of these. Make sure the image listens on `ContainerPort`
and serves a success response at `/` when using the load balancer.
