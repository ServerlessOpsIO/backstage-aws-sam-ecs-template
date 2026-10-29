# Observability

## Container and Service Connect logs

The application container uses the `awslogs` driver. It creates or writes to
`/aws/ecs/<cluster-name>/<stack-name>` in the deployment region, with
`app` as the stream prefix. The Service Connect configuration writes to the
same group with the `sidecar` prefix. When Datadog is enabled, the Datadog
Agent container also uses this log configuration.

The template does not set CloudWatch log-group retention. Set an explicit
retention policy appropriate to the service and data classification. The
application image must write useful output to stdout/stderr for it to appear
in the `app` stream.

Avoid logging credentials, tokens, or sensitive request payloads. Scrub
application logs before enabling verbose/debug output, and use the smallest
amount of operational data necessary for troubleshooting.

## ECS service and task health

Use ECS service events and task state to diagnose deployments, placement,
image-pull, and health-check problems. The deployment circuit breaker is
enabled and configured to roll back a failed deployment. If using the ALB,
the target group performs an HTTP health check on `/` at the container port
with short interval/timeout settings. Ensure the application serves that path
successfully before routing traffic.

There is no application-container health check in the task definition by
default. Add one if the image needs a process-level health signal. The
Datadog Agent sidecar has its own `agent health` check when enabled.

## Metrics

ECS and Fargate expose service/task metrics such as CPU and memory utilization.
The ALB and target group expose request count, latency, and healthy/unhealthy
target metrics when load balancing is enabled. CloudWatch Logs Insights can
help inspect container log streams. The starter does not define alarms,
dashboards, autoscaling policies, or application-specific metrics; add these
to the service based on its reliability and capacity requirements.

## Datadog integration

The Backstage form can enable a Datadog Agent sidecar. The generated task
definition passes `DD_API_KEY`, `DD_SITE`, and `ECS_FARGATE=true` and configures
a sidecar health check. It does not explicitly configure log collection,
application tracing, or custom metrics. Confirm the Agent configuration and
account-level Datadog integration before assuming those signals are available.

Supply the API key and site through GitHub secrets used by the deployment. Do
not commit keys. Container labels provide `com.datadoghq.tags.domain`,
`system`, `component`, and `code_branch` metadata to support resource
identification.

## Investigation checklist

1. Record the deployment time, stack name, ECS cluster/service, task revision,
   and image tag.
2. Inspect CloudFormation events and ECS service events for rollout, placement,
   networking, and image-pull failures.
3. Check task stopped reason and the application/sidecar CloudWatch log streams
   without copying secrets or sensitive payloads into tickets.
4. For load-balanced traffic, compare ALB target health and request/latency
   metrics with the container health endpoint.
5. Check Datadog separately; first establish whether expected logs and metrics
   exist in CloudWatch/AWS before treating missing Datadog data as an
   application failure.
