# Features

## ECS cluster

`template.yaml` creates one `AWS::ECS::Cluster`. The template does not set a
cluster name, so CloudFormation generates one from the stack name and logical
ID; consuming services discover it through SSM rather than by a fixed name.

The cluster stack provisions the shared boundary and integration values only.
It does **not** create ECS services, task definitions, container images,
capacity providers, EC2 container instances, autoscaling, or service-level
security groups. The ECS service template runs tasks with the Fargate launch
type, which needs no cluster-managed capacity.

The template does not set `ClusterSettings`, so CloudWatch Container Insights
follows the account's default setting. See [Observability](06-observability.md).

## Service Connect namespace

The cluster's `ServiceConnectDefaults.Namespace` is set from the
`ServiceConnectNamespace` parameter. The scaffolder renders
`<system-name>.local`, so every cluster in the same system uses the same
namespace name by default. ECS uses or creates the AWS Cloud Map namespace
of that name.

Services in the cluster register with Service Connect and call each other by
client alias, for example `<service-component>.<system-name>.local:<port>`.
Service Connect provides private service-to-service discovery; it is not a
public DNS name and does not route internet traffic.

## Optional shared Application Load Balancer

When **Create Load Balancer?** is enabled, the template creates:

| Resource | Behavior |
| --- | --- |
| `EcsAlbSecurityGroup` | Allows inbound TCP 443 from `0.0.0.0/0`. No port 80 ingress or HTTP-to-HTTPS redirect. |
| `EcsAlb` | Internet-facing Application Load Balancer in the subnets resolved from `/org/networking/VpcPublicSubnets`. |
| `EcsAlbListener` | HTTPS listener on 443 with the stack's certificate and the default ELB TLS policy. |
| `SiteCertificate` | DNS-validated ACM certificate for the configured hostname. |
| `SiteDnsRecord` | Route 53 alias `A` record pointing the hostname at the ALB. |

The listener's default action returns a fixed `200` response with body
`{ "healthy": true }`. It has no target group. Services that use the shared ALB
add their own target group and a path-based listener rule with a unique
priority. Any request that matches no service rule receives the fixed response,
so a successful response from the hostname does **not** show that a service is
healthy.

## DNS and TLS

With the ALB enabled, the stack obtains the hosted zone ID through the existing
SSM parameter `/org/dns/ZoneId`, validates the certificate with DNS records in
that zone, and creates the alias record. The hosted zone and SSM parameter are
prerequisites, not resources created by this stack. See
[Configuration](03-configuration.md).

## Load-balancer-disabled option

**Selecting No for Create Load Balancer? does not produce a deployable
template.** `EcsAlbSecurityGroup` is only rendered when the ALB is enabled, but
`ClusterAlbSecurityGroupIdSsmParam` is rendered unconditionally and references
it. CloudFormation rejects the unresolved reference. Fix the template, as
described in [Common tasks](07-common-tasks.md#change-load-balancer-or-dns-configuration),
before using that option.

## SSM integration contract

The cluster publishes these String parameters under
`/<domain>/<system>/<component>/<branch>/`:

| Parameter suffix | Value | Rendered when | Read by the ECS service template |
| --- | --- | --- | --- |
| `EcsClusterName` | Generated ECS cluster name. | Always | Always, as `ClusterName` |
| `ServiceConnectNamespace` | Service Connect namespace name. | Always | Always, as `ClusterServiceConnectNamespace` |
| `EcsAlbListenerArn` | HTTPS listener ARN. | ALB enabled | When the service uses the load balancer, as `ClusterAlbListenerArn` |
| `ClusterAlbSecurityGroupId` | ALB security-group ID. | Always (see the disabled-option issue above) | When the service uses the load balancer, as `ClusterAlbSecurityGroupId` |

The ECS service template builds these paths from **its own** selected domain
and system, the selected cluster's entity name, and a fixed `main` branch
segment. A service can therefore consume a cluster only when the cluster was
deployed from `main` with the same domain and system. Keep path segments and
suffixes stable; changing them is a coordinated change with every consumer.

See [Configuration](03-configuration.md) for parameter sources and
[Architecture](04-architecture.md) for resource relationships.

## CI/CD

GitHub Actions supplies one workflow, `main.yml`, which validates and builds the
SAM template and deploys it on pushes to `main` or manual dispatch. There is no
branch workflow. See [CI/CD pipeline](05-ci-cd.md).
