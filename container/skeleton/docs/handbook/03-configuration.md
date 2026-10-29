# Configuration

## Stack parameters: `cfn-parameters.json`

This file supplies the deployment action with values for the CloudFormation
`Parameters` in `template.yaml`. It is a deployment-action input rather than a
standalone SAM parameter-override file.

| Parameter | Starter value/source | Meaning |
| --- | --- | --- |
| `Domain` | Selected catalog domain name | Application domain metadata and Docker tag. |
| `System` | Selected catalog system name | Application system metadata and Docker tag. |
| `Component` | `$env.GITHUB_REPOSITORY_NAME_PART_SLUG_CS` | Component/container/service identifier derived from the repository name. |
| `CodeBranch` | `$env.GITHUB_REF_SLUG_CS` | Branch slug used in task-definition family naming and Docker labels. |
| `ClusterName` | `/<domain>/<system>/<cluster>/main/EcsClusterName` | SSM parameter path resolving to the existing ECS cluster. |
| `ClusterServiceConnectNamespace` | `/<domain>/<system>/<cluster>/main/ServiceConnectNamespace` | SSM parameter path resolving the cluster's Service Connect namespace. |
| `ClusterAlbListenerArn` | `/<domain>/<system>/<cluster>/main/EcsAlbListenerArn` when load balancing is enabled | Existing cluster ALB listener. |
| `ClusterAlbSecurityGroupId` | `/<domain>/<system>/<cluster>/main/ClusterAlbSecurityGroupId` when load balancing is enabled | Existing ALB security group used as the service ingress source. |
| `ContainerImage` | Selected image name, or repository-derived value for a development image | Container image repository. |
| `ContainerTag` | Selected tag (default `latest`), or workflow commit tag for a development image | Image tag appended to the repository. |
| `ContainerCpu` | Backstage value (default `1024`) | Fargate task CPU units. |
| `ContainerMemory` | Backstage value (default `2048`) | Fargate task memory in MB. |
| `ContainerDesiredCount` | Backstage value (default `3`) | Number of desired service tasks. |
| `EnableContainerExec` | `false` | Whether to enable ECS Exec in the service. |
| `ContainerPort` | Selected port when required by the load-balancer form | Container port for port mapping and Service Connect. |
| `AlbRulePriority`, `AlbRulePath`, `VpcId` | Supplied when load balancing is enabled | Shared ALB route, listener priority, and VPC used by the target group/security group. |
| `VpcSubnets` | `/org/networking/VpcPublicSubnets` | SSM parameter resolving to subnet IDs. |
| `DatadogApiKey`, `DatadogSite` | Datadog deployment secrets when enabled | Agent credentials and Datadog site. |

Exact parameter presence depends on the selected template options. In
particular, the current skeleton still references `VpcId` and `ContainerPort`
when load balancing is disabled; see [Features](01-features.md).

The `$env` and `$secrets` expressions in this file are deployment-action
expressions and are intentionally not quoted as ordinary JSON values. The
GitHub workflow passes environment variables, repository/environment
variables, and secrets to `ServerlessOpsIO/gha-deploy-aws-sam@v1` for
resolution. Consult that action's configuration when changing expression
syntax or adding values. Do not commit credentials or replace expressions with
secret literals.

## Stack tags: `cfn-tags.json`

The deployment tags are:

| Tag | Value |
| --- | --- |
| `org:domain` | Selected domain name |
| `org:system` | Selected system name |
| `org:component` | Repository-derived component identifier |

Keep these aligned with catalog metadata and inventory/cost-allocation
conventions. These tags are metadata, not runtime configuration or access
controls.

## Deployment prerequisites

- An AWS target account and region with the IAM permissions required by the
  shared build/deploy actions and the generated CloudFormation resources.
- The selected ECS cluster, Service Connect namespace, and required SSM
  parameters. When using load balancing, the cluster listener and ALB security
  group must also exist.
- VPC and subnet values resolvable from the configured SSM parameter paths.
  The current template assigns public IPs to tasks; review this setting against
  account networking policy.
- A container image in a registry the task execution role can pull. The
  generated task execution role includes ECR read actions; image access and
  cross-account repository policies must still permit the target account.
- GitHub Actions secrets `AWS_CICD_ACCOUNT_ID` and `DEPLOYMENT_ACCOUNT_ID`.
  Scaffolding creates the latter from the selected cloud-account entity title;
  the former must be configured for the shared build action. With Datadog
  enabled, configure `DATADOG_AGENT_API_KEY` and `DATADOG_SITE` as well.
- AWS OIDC trust and roles expected by
  `ServerlessOpsIO/gha-assume-aws-credentials@v1`. Account IDs alone do not
  grant AWS access; check the shared action's requirements.
- GitHub environment `production` settings appropriate for deployments,
  including any approval and branch restrictions.

## SAM and runtime settings

`samconfig.toml` sets the stack name from the component name, enables cached
parallel builds and linting, and grants `CAPABILITY_NAMED_IAM` and
`CAPABILITY_AUTO_EXPAND` for deployment. It does not define separate branch
stack names. Verify the final stack name used by the shared deployment action.

Container CPU and memory must be a valid Fargate combination. The application
image defines its own runtime and process behavior; this infrastructure
template does not select an application language runtime or set a command.
Review the health-check and port requirements for the image before deployment.

`CodeBranch` changes the task-definition family and is included in Docker
labels. The stack name and other shared resources are not automatically
isolated by branch. See [CI/CD pipeline](05-ci-cd.md) before deploying
additional branches.
