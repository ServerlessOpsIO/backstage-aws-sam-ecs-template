# Observability

## Container logging

All containers in the task use the `awslogs` driver and write to one log group,
`/aws/ecs/<cluster-name>/<stack-name>`, in the deployment region:

| Stream prefix | Source |
| --- | --- |
| `app/<component>/<task-id>` | Application container stdout and stderr |
| `sidecar/...` | Service Connect proxy, and the Datadog Agent when enabled |

The driver creates the group on first use (`awslogs-create-group: true`), so it
is not a CloudFormation resource: it has **no retention policy** and is not
deleted with the stack. Set retention explicitly and clean it up when the
service is retired.

The application must write useful output to stdout/stderr; files written inside
the container are not collected. Prefer structured (JSON) log lines that include
a request or correlation ID.

Never log tokens, credentials, or sensitive request payloads. Sanitize output
before enabling verbose or debug logging.

## ECS service and task health

| Signal | Where | Meaning |
| --- | --- | --- |
| Service events | ECS console or `aws ecs describe-services` | Placement, deployment, target registration, and circuit-breaker messages. |
| Stopped reason | Stopped tasks | Image-pull failures, essential-container exits, and failed health checks. |
| Target health | Target group | Result of the ALB's HTTP `/` check every 5 seconds; 2 consecutive failures mark a target unhealthy. |
| Container health | Task details | Only when you add a `HealthCheck`; the Datadog Agent has its own `agent health` check. |

The 60-second health-check grace period lets slow-starting tasks come up before
failing ALB checks count. If the application needs longer, raise
`HealthCheckGracePeriodSeconds` rather than loosening the checks.

## Metrics

ECS publishes service `CPUUtilization` and `MemoryUtilization`. With the load
balancer, the ALB publishes per-target-group `RequestCount`,
`TargetResponseTime`, `HTTPCode_Target_5XX_Count`, `HealthyHostCount`, and
`UnHealthyHostCount`. Task-level metrics require Container Insights, which the
cluster leaves at the account default.

Filter by the deployed service name, cluster, and target group. Watch error
rates, unhealthy targets, and memory alongside request counts.

The starter does not emit custom application metrics, configure alarms,
dashboards, or autoscaling, or enable tracing. Add these intentionally based
on the service's reliability targets.

## Availability in Datadog

The Datadog AWS integration and log shipping are **configured automatically in
all AWS accounts**. Developers do not need to enable the integration or
configure log forwarding for generated services. Use Datadog to inspect the
service's AWS metrics and the shipped CloudWatch container logs.

Baseline tagging is also automatic: `cfn-tags.json` supplies `org:domain`,
`org:system`, and `org:component`, `ServerlessOpsIO/gha-deploy-aws-sam` adds
additional deployment tags, and the application container's Docker labels set
Datadog `domain`, `system`, `component`, and `code_branch` tags. Use the
existing tags to locate and distinguish resources; extra tagging is only needed
for service-specific requirements.

When the Datadog sidecar is enabled, it adds live container and process-level
metrics from inside the task. It does not configure APM tracing or DogStatsD;
add those intentionally, including the application-side library.

After deployment, verify that a sanitized test log and recent ECS metrics appear
in Datadog. If data is missing, first confirm it exists in CloudWatch and check
the Datadog time range and filters. Treat persistent missing data as a problem
with the existing collection or shipping configuration, not a requirement to
set up a new integration.

## Investigation checklist

1. Note the timestamp, request path, status, and any request ID.
2. Determine where the request stopped: a 503 from the ALB means no healthy
   targets; the cluster's fixed response means the path matched no rule; an
   application status code means the request reached a task.
3. Check the ECS service events and stopped-task reasons for crashes, failed
   health checks, image-pull errors, or a circuit-breaker rollback.
4. Read the `app` log streams for the affected tasks, without copying sensitive
   payloads into tickets.
5. Compare error, latency, CPU, and memory metrics with recent deployments and
   the running image tag.
6. Check Datadog ingestion separately if CloudWatch data is present but missing
   from Datadog.
