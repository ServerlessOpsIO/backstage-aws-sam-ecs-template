# Observability

## ECS cluster

The cluster stack creates no services, tasks, or log groups. Task and service
logs are configured by each consuming service's task definition; the ECS
service template writes them to `/aws/ecs/<cluster-name>/<service-stack-name>`.
Investigate application behavior in the consuming component.

ECS publishes service-level `CPUUtilization` and `MemoryUtilization` metrics
for services in the cluster. The template does not set `ClusterSettings`, so
CloudWatch Container Insights (task-level and cluster-level metrics) follows
the account's default; enable it explicitly if you need those metrics and
accept their cost.

## Application Load Balancer

When enabled, the ALB publishes standard CloudWatch metrics such as
`RequestCount`, `TargetResponseTime`, `HTTPCode_ELB_4XX_Count`,
`HTTPCode_ELB_5XX_Count`, and per-target-group `HealthyHostCount` and
`UnHealthyHostCount`. Target-group metrics only exist once a consuming service
attaches a target group. Requests answered by the default fixed response count
toward ALB metrics without reaching any service.

The template does not configure ALB access logs, alarms, or dashboards. ACM and
Route 53 provide TLS and name resolution; they do not provide request tracing.

## Availability in Datadog

The Datadog AWS integration and log shipping are **configured automatically in
all AWS accounts**. Developers do not need to enable the integration or
configure log forwarding for generated components. Use Datadog to inspect the
ECS and ALB metrics imported from AWS.

Baseline tagging is also automatic: `cfn-tags.json` supplies `org:domain`,
`org:system`, and `org:component`, and `ServerlessOpsIO/gha-deploy-aws-sam`
adds additional deployment tags. Use the existing tags to locate and
distinguish resources; extra tagging is only needed for service-specific
requirements.

In Metrics Explorer, filter by the load balancer, target group, or cluster
name, or by the `org:*` tags. Exact field names and tags depend on the
integration's configuration. If data is missing, first confirm it exists in
CloudWatch and check the Datadog time range and filters. Treat persistent
missing data as a problem with the existing collection configuration, not a
requirement to set up a new integration.

## SSM integration values

The stack publishes identifiers to
`/<domain>/<system>/<component>/<branch>/<parameter-name>`. Most "service will
not deploy" problems that involve the cluster are a missing or mismatched path:
wrong account or region, a cluster deployed from a branch other than `main`, a
domain or system that differs between cluster and service, or the ALB values
missing because the ALB was disabled. See
[Features](01-features.md#ssm-integration-contract).

## Investigation checklist

1. Record the time, workflow run, selected ref, target account and region, and
   stack name.
2. Check the build/deploy logs and CloudFormation stack events for validation,
   permission, or resource-creation failures.
3. Verify that the expected SSM parameters exist and resolve to this stack's
   cluster, namespace, and ALB resources.
4. For ALB issues, check certificate status, Route 53 resolution, listener
   rules and their priorities, and target-group health. A fixed response means
   no rule matched the request.
5. For service issues, continue in the consuming component: its ECS service
   events, stopped-task reasons, and container logs.
6. Check Datadog separately if CloudWatch data is present but missing from
   Datadog.

Do not copy secrets or sensitive request data into tickets or workflow logs.
