# Features

## ECS Fargate service

`template.yaml` defines an `AWS::ECS::TaskDefinition` and an
`AWS::ECS::Service` that runs the task on Fargate with `awsvpc` networking,
Linux, and the X86_64 architecture. The service uses the selected cluster,
desired task count, CPU, memory, image, and image tag supplied at deployment.
The deployment circuit breaker is enabled with rollback; the service allows
up to 200% maximum and 75% minimum healthy tasks during deployments.

This skeleton does not contain an application image or application source.
For the standard ECS Service Container template, provide an image available to
the target account. The separate development-container template can build an
image from its application starter when that workflow variant is used.

## Networking and load balancing

The service is attached to the VPC subnets supplied by platform SSM parameters,
gets a service security group, and currently sets `AssignPublicIp` to
`ENABLED`. When load balancing is enabled, the stack creates a target group
and listener rule on the cluster's existing Application Load Balancer. The
security group permits traffic to the container port from the cluster ALB
security group. The target group's health check is HTTP on `/`.

The template currently has a conditional-input mismatch: VPC ID and container
port parameters are declared only when load balancing is enabled, but the
service security group and Service Connect configuration always reference
them. Disabling load balancing therefore leaves required references without
parameters. Keep load balancing enabled unless the generated template has been
corrected to supply those values independently.

## Service Connect

The service always enables ECS Service Connect using the namespace associated
with the selected cluster. It registers the service using the component name
and creates a client alias of `<component>.<namespace>` on the container port.
This supports private service-to-service discovery in the cluster's configured
Service Connect namespace; it does not by itself expose an internet-facing
endpoint.

## Container configuration

The task definition contains the application container and can include
container port mappings, environment variables, secrets, and a container
health check. The skeleton shows examples for environment variables, Secrets
Manager/SSM secrets, and Docker health checks as commented configuration; they
are not enabled by default. Add only the configuration the application
requires and keep secret values out of source control.

The task execution role can retrieve ECR images and write CloudWatch logs. The
application task role has no application permissions by default. Add
least-privilege permissions to the task role when the application calls AWS
services.

## ECS Exec

`EnableContainerExec` is a CloudFormation parameter defaulting to `false`, and
`cfn-parameters.json` explicitly deploys it as `false`. When enabled, the
template grants the task role the SSM Messages channel actions needed for ECS
Exec. Enabling the task/service setting is not enough on its own: the operator
also needs appropriate ECS/SSM permissions, and network connectivity to the
ECS Exec messaging endpoints must be available.

## Datadog sidecar

When Datadog integration is selected, the task definition adds a Datadog Agent
sidecar with the supplied API key and site, Fargate mode enabled, and an
`agent health` check. The key and site are deployment inputs and should be
provided through repository/environment secrets, not committed. Confirm the
platform's Datadog configuration and data collection requirements before
assuming the sidecar collects application logs or traces; the task definition
does not configure those features explicitly.

See [Architecture](04-architecture.md) for resource relationships,
[Configuration](03-configuration.md) for inputs and prerequisites, and
[Common tasks](07-common-tasks.md) for customization.
