# Features

## ECS cluster and Service Connect

`template.yaml` creates an Amazon ECS cluster and sets its Service Connect
default namespace from the `ServiceConnectNamespace` parameter. The scaffolder
defaults that value to `<system-name>.local`.

This template provisions the cluster boundary and shared integration values.
It does **not** create ECS services, task definitions, container images,
compute capacity, or service-level security groups. Application components
must supply those resources and configure their cluster integration.

## Optional Application Load Balancer

When **Create Load Balancer?** is enabled, the template creates:

- An internet-facing Application Load Balancer in the configured VPC subnets.
- A security group allowing inbound TCP 443 from `0.0.0.0/0`.
- A DNS-validated ACM certificate for the configured hostname.
- An HTTPS listener on port 443 and a Route 53 alias A record.
- An SSM parameter containing the listener ARN for consuming services.

The listener's current default action returns a fixed JSON health response. It
has no target group or service routing rules, so a successful listener response
does not demonstrate that an ECS service is healthy. Services that use the
shared ALB need their own target group and listener rules.

**The load-balancer-disabled option is not deployable as currently generated.**
The security group is conditional, but `ClusterAlbSecurityGroupIdSsmParam`
references it unconditionally. Selecting No therefore leaves a dangling
CloudFormation reference. Correct that template condition before using the
option.

## Shared values for consuming services

The cluster writes values to Parameter Store under
`/<domain>/<system>/<component>/<branch>/`:

| Parameter suffix | Value | When available |
| --- | --- | --- |
| `EcsClusterName` | Name of the created ECS cluster. | Always |
| `ClusterAlbSecurityGroupId` | ID of the ALB security group. | Only when the ALB is enabled; currently the parameter declaration is unconditional, causing the disabled-option issue above. |
| `EcsAlbListenerArn` | ARN of the HTTPS listener. | Only when the ALB is enabled |
| `ServiceConnectNamespace` | Service Connect namespace associated with the cluster. | Always |

The ECS service template in this repository reads the cluster name and Service
Connect namespace by these SSM paths. When configured for ALB routing, it also
reads the listener ARN and ALB security group ID. Keep the stack path and
parameter suffixes compatible when changing the cluster contract.

See [Configuration](03-configuration.md) for parameter sources and
[Architecture](04-architecture.md) for the resource relationships.
