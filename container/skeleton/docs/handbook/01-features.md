# Features

## ECS Fargate service

`template.yaml` defines an `AWS::ECS::TaskDefinition` and an `AWS::ECS::Service`
that runs it with the Fargate launch type, `awsvpc` networking, Linux, and the
`X86_64` architecture. The service name and the application container name are
the component name (the repository name).

| Setting | Starter value |
| --- | --- |
| Task CPU / memory | `ContainerCpu` / `ContainerMemory` from Backstage (defaults 1024 / 2048 MB) |
| Desired tasks | `ContainerDesiredCount` from Backstage (default 3) |
| Deployment limits | `MaximumPercent: 200`, `MinimumHealthyPercent: 75` |
| Circuit breaker | Enabled with automatic rollback to the last completed deployment |
| Task-definition family | `<component>-<branch>` |
| Tags | ECS-managed tags enabled; tags propagate from the task definition to tasks |

There is no autoscaling policy; the service keeps the desired count.

## Container image source

The image source depends on the [variant](00-index.md#which-variant-generated-this-repository):

- **Image-based service**: runs the image repository and tag entered in
  Backstage, as `<ContainerImage>:<ContainerTag>`. The repository contains no
  application source; build and publish the image elsewhere.
- **Development application**: the workflows build `src/Dockerfile` and push it
  to ECR in the CI/CD account as `<owner>/<repository>`. Deployment runs
  `<AWS_ECR_REPO>/<owner>/<repository>:<commit SHA>`, so every `main` deployment
  runs the image built from that commit. With Node.js selected, `src/` holds a
  minimal Node.js/TypeScript starter.

The task execution role can pull from ECR. The ECR repository policy in
`.github/workflows/ecr-policy.json` allows image pulls from principals in the
organization's AWS Organization.

## Load balancing

When **Use cluster load balancer?** is enabled, the stack attaches the service to
the cluster's shared, internet-facing ALB:

- An IP target group on the container port, with an HTTP health check on `/`
  every 5 seconds (4-second timeout, 2 healthy / 2 unhealthy thresholds).
- A listener rule forwarding the configured path pattern to that target group
  at the configured priority.
- An ingress rule allowing the container port only from the ALB security group.
- A 60-second health-check grace period for new tasks.

The ALB is reached through the cluster's hostname over HTTPS; the service has no
hostname or certificate of its own. Path patterns are matched as-is, so the
application must serve requests under its path prefix (for example `/app/*`);
the ALB does not strip it.

## Service Connect

The service always enables ECS Service Connect in the cluster's namespace,
registering the container's named port and a client alias
`<component>.<namespace>` on the container port. Other services in the cluster
can call it privately at, for example,
`http://${{ values.component_name }}.<system-name>.local:<port>`.
Service Connect adds a proxy container to each task, managed by ECS. It does
not expose an internet-facing endpoint.

## Load-balancer-disabled option

**Disabling the load balancer does not produce a deployable template.** The
container port, `VpcId`, and the port mapping are only rendered with the load
balancer, but the service security group always references `VpcId`, and Service
Connect always references `ContainerPort` and the port name. Keep the load
balancer enabled, or fix the template as described in
[Common tasks](07-common-tasks.md#configure-a-port-or-load-balancer-route).

## Container configuration

The application container gets Datadog unified-tagging Docker labels and
`awslogs` logging. The task definition includes commented examples for
`Environment`, `Secrets`, and a container `HealthCheck`; none are enabled by
default. See [Common tasks](07-common-tasks.md#configure-environment-variables-secrets-and-health-checks).

## IAM roles

The **task execution role** is used by ECS to start tasks: it can pull ECR images
and write CloudWatch Logs. The **task role** is assumed by application code and
has no policies by default. Grant the application's AWS permissions to the task
role only. See [Architecture](04-architecture.md#iam-roles).

## ECS Exec

`EnableContainerExec` defaults to `"false"` and `cfn-parameters.json` deploys it
as `"false"`. Setting it to `"true"` enables execute-command on the service and
attaches an SSM Messages policy to the task role. Operators also need ECS and
SSM permissions and the Session Manager plugin, and tasks need a network path to
the SSM Messages endpoints.

## Datadog Agent sidecar

When **Enable Datadog integration?** is selected, the task definition adds a
`datadog-agent` container (`public.ecr.aws/datadog/agent:latest`, 256 CPU units,
512 MB) with `ECS_FARGATE=true`, `DD_LOG_LEVEL=ERROR`, and an `agent health`
check. It receives the Datadog API key and site from the
`DATADOG_AGENT_API_KEY` and `DATADOG_SITE` GitHub secrets. The sidecar shares
the task's CPU and memory with the application container.

The sidecar collects live container metrics from the task. Log shipping and the
AWS integration are already automatic; see
[Observability](06-observability.md#availability-in-datadog). The task does not
configure APM tracing or DogStatsD for the application.

The API key is passed as a plain container environment variable, so anyone who
can describe the task definition can read it. See
[Common tasks](07-common-tasks.md#add-or-update-datadog) to move it to `Secrets`.

## CI/CD and feature branches

GitHub Actions supplies `main.yml`, which builds and deploys, and `branch.yml`,
which builds without deploying. See [CI/CD pipeline](05-ci-cd.md).
