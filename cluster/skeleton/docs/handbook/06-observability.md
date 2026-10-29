# Observability

## ECS cluster

This template creates an ECS cluster but does not create ECS services, tasks,
or container log groups. There is no application logging configuration in the
cluster stack. Task and service logs depend on each consuming service's task
definition and log driver; inspect that service's configuration and deployed
CloudWatch log group.

ECS and CloudWatch publish cluster/service metrics according to the resources
and monitoring configuration in use. This stack does not configure Container
Insights, alarms, dashboards, or X-Ray tracing. Add these intentionally based
on the service's reliability and cost requirements.

## Application Load Balancer

When enabled, the stack creates an ALB and HTTPS listener. The ALB publishes
standard CloudWatch metrics such as request count, target response time, and
HTTP load balancer/target error counts. This template does not create ALB
access-log delivery or target groups. Until a consuming service registers
targets and adds listener rules, target-level metrics and application traffic
are not available; the listener's default action is a fixed response.

The ACM certificate and Route 53 record support TLS and name resolution; they
do not provide request tracing or application health monitoring.

## SSM integration values

The stack exports resource identifiers to Parameter Store using paths of the
form `/<domain>/<system>/<component>/<branch>/<parameter-name>`. When
troubleshooting a service integration, verify that the consuming service
references the expected account, region, branch path, and parameter values.
See [Features](01-features.md) for the exported parameter list.

## Operational investigation checklist

1. Record the deployment time, workflow run, selected ref, target account, and
   CloudFormation stack name.
2. Check the workflow build/deploy logs and CloudFormation stack events for
   validation, permission, or resource-creation failures.
3. Verify that the expected SSM parameters exist and resolve to the cluster,
   namespace, and (if enabled) ALB resources.
4. For ALB issues, inspect listener rules, certificate state, Route 53
   resolution, and available CloudWatch metrics. A fixed listener response
   does not prove application health.
5. For service issues, continue in the consuming component: inspect its ECS
   service events, task state, task-definition logging, and application
   metrics.

Do not copy secrets or sensitive request data into tickets or workflow logs.
