# Features

## ECS cluster

`template.yaml` creates one `AWS::ECS::Cluster`, the shared placement boundary
for the services of the **${{ values.system_title }}** system. The template
does not set a cluster name, so CloudFormation generates one from the stack
name and logical ID. Services find it through the
[SSM integration contract](#ssm-integration-contract), not by a fixed name.

The stack does **not** create ECS services, task definitions, container images,
capacity providers, container instances, or autoscaling. Services created from
the ECS service templates run on Fargate, which needs no cluster-managed
capacity, and each service stack owns its own task definition, security group,
and log group.

## Service Connect namespace

The cluster's Service Connect default namespace is
**`${{ values.system_name }}.local`**, supplied by the `ServiceConnectNamespace`
parameter. ECS uses the AWS Cloud Map namespace of that name, creating it if
it does not exist. The name derives from the system, so another cluster created
in the same system uses the same namespace name by default.

A service created from the ECS service templates registers under its component
name, and other services in the namespace call it at:

```text
<service-component-name>.${{ values.system_name }}.local:<container-port>
```

Service Connect provides private service-to-service discovery. It is not a
public DNS name and does not route internet traffic.

{% if values.create_load_balancer %}
## Shared Application Load Balancer

The cluster was created with **Create Load Balancer?** enabled, so the stack
owns one internet-facing Application Load Balancer shared by every service in
the cluster:

| Resource | Behavior |
| --- | --- |
| `EcsAlbSecurityGroup` | Allows inbound TCP 443 from `0.0.0.0/0`. There is no port 80 ingress. |
| `EcsAlb` | Internet-facing ALB in the subnets resolved from `/org/networking/VpcPublicSubnets`. |
| `EcsAlbListener` | HTTPS listener on 443 using `SiteCertificate` and the default ELB TLS policy. There is no HTTP listener or HTTP-to-HTTPS redirect. |

The listener's default action returns a fixed `200` response with the body
`{ "healthy": true }`. The cluster stack creates no target groups or rules.
Each service that uses the load balancer adds its own target group and a
path-based listener rule with a priority that is unique on this listener.

Any request that matches no service rule receives the fixed response, so a
successful response from `https://${{ values.hostname }}` does **not** show
that a service is running or healthy.

See [Common tasks](07-common-tasks.md#connect-an-ecs-service) for choosing
rule paths and priorities.

## DNS and TLS

The stack creates a DNS-validated ACM certificate and a Route 53 alias `A`
record for **`${{ values.hostname }}`**, pointing at the load balancer. Every
service on the load balancer is reached through this one hostname and is
distinguished by its listener-rule path.

The stack obtains the hosted zone ID through the existing SSM parameter
`/org/dns/ZoneId`. The hosted zone and SSM parameter are prerequisites, not
resources created by this stack. See [Configuration](03-configuration.md).
{% else %}
## No shared load balancer

The cluster was created with **Create Load Balancer?** set to No. The stack
creates no load balancer, listener, certificate, or DNS record, and services in
this cluster must be created with **Use cluster load balancer?** set to No.
They are reachable only through Service Connect.

**The generated `template.yaml` does not pass validation as created.** It
declares `ClusterAlbSecurityGroupIdSsmParam`, whose value references the
`EcsAlbSecurityGroup` resource that is only generated with the load balancer.
`sam validate --lint` fails in the build job, so nothing deploys until you
remove that resource. See
[Common tasks](07-common-tasks.md#make-the-template-deployable).
{% endif %}

## SSM integration contract

After a deployment from `main`, the stack publishes these String parameters.
Service stacks read them at deploy time, so they never hard-code generated
names or ARNs.

| SSM parameter | Value | Read by the ECS service templates |
| --- | --- | --- |
| `/${{ values.domain_name }}/${{ values.system_name }}/${{ values.component_name }}/main/EcsClusterName` | Generated ECS cluster name | Always |
| `/${{ values.domain_name }}/${{ values.system_name }}/${{ values.component_name }}/main/ServiceConnectNamespace` | `${{ values.system_name }}.local` | Always |
{%- if values.create_load_balancer %}
| `/${{ values.domain_name }}/${{ values.system_name }}/${{ values.component_name }}/main/EcsAlbListenerArn` | HTTPS listener ARN | When the service uses the cluster load balancer |
| `/${{ values.domain_name }}/${{ values.system_name }}/${{ values.component_name }}/main/ClusterAlbSecurityGroupId` | Load balancer security group ID | When the service uses the cluster load balancer |
{%- endif %}

The path is `/<Domain>/<System>/<Component>/<CodeBranch>/<name>`, built from
the stack parameters described in [Configuration](03-configuration.md). The ECS
service templates build the same paths from the service's own domain and
system, this cluster's catalog entity name, and a fixed `main` segment. A
service can therefore only consume a cluster that was deployed from `main` in
the same domain and system.

Treat the paths and parameter names as an interface. Changing them is a
coordinated change with every consuming service; see
[Common tasks](07-common-tasks.md#change-ssm-parameter-paths).

A service's generated `cfn-parameters.json` also names a
`.../main/ContainerSecurityGroupId` path. This cluster does not publish that
parameter, and the service's `template.yaml` declares no stack parameter that
reads it.

## CI/CD

GitHub Actions supplies one workflow, `main.yml`. It validates and builds the
SAM template, then deploys it, on every push to `main` and on manual dispatch.
There is no branch workflow, and a deployment from any ref other than `main`
moves the SSM parameters that services depend on.
See [CI/CD pipeline](05-ci-cd.md) before dispatching the workflow manually.
