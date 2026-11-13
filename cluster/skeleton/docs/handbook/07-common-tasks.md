# Common changes and tasks

## Set up a development environment

Install the AWS SAM CLI. From the **generated repository root**, run:

```sh
sam validate --lint
sam build --parallel --template template.yaml
```

These are the checks the build job runs. Both run locally without AWS
credentials, so neither shows that the account's SSM parameters, IAM
permissions, or DNS are ready.

{% if not values.create_load_balancer %}
## Make the template deployable

The cluster was created without a load balancer, and the generated
`template.yaml` fails `sam validate --lint` until you remove the resource that
publishes the load balancer's security group:

1. Delete the whole `ClusterAlbSecurityGroupIdSsmParam` resource from
   `Resources` in `template.yaml`.
2. Optionally delete the `VpcId` parameter from `template.yaml` and the `VpcId`
   entry from `cfn-parameters.json`. No remaining resource references it.
3. Run the checks above, then merge to `main`.

Report the problem to the owners of the **AWS ECS Cluster** template so newly
created clusters do not need this change.

{% endif %}
## Connect an ECS service

Create the service with the **AWS ECS Service Container** or **Containerized
Development Application** template. Deploy this cluster from `main` first.

1. Select the same domain and system as this cluster,
   **${{ values.system_title }}**. The **Cluster** picker only offers clusters
   in the selected system, and the service builds its SSM paths from its own
   domain and system.
2. Select **${{ values.component_name }}** as the cluster.
{%- if values.create_load_balancer %}
3. To receive public traffic, keep **Use cluster load balancer?** enabled and
   choose the rule's path and priority deliberately; the form defaults to path
   `*` and priority `1`.
4. Make the container answer `GET /` on its container port with a success
   status. The service's target group health check uses that fixed path, even
   when the rule path is a prefix such as `/app/*`.
{%- else %}
3. Set **Use cluster load balancer?** to No. This cluster publishes no listener
   or load balancer security group, so a service that expects them fails to
   deploy.
{%- endif %}

The service appears under this component's dependents in the catalog. Other
services call it through Service Connect at
`<service-component-name>.${{ values.system_name }}.local:<container-port>`.

{% if values.create_load_balancer %}
### Choose listener rule paths and priorities

All services share one listener at `https://${{ values.hostname }}`, and the
rules are not coordinated by any stack:

- Each priority can be used once. A second service deployed with a priority
  that is already taken fails to deploy.
- Rules are evaluated from the lowest priority number, and the first match
  wins. A `*` rule matches every request, so any rule with a higher number
  never receives traffic.
- Give specific prefixes such as `/orders/*` low numbers, and reserve a high
  number for at most one catch-all service.
- The path is forwarded unchanged. A service routed at `/orders/*` receives
  requests with the `/orders` prefix.

List the rules currently on the listener before choosing, and keep a record of
assigned paths and priorities for this cluster. The listener ARN is in the
`EcsAlbListenerArn` SSM parameter:

```sh
aws elbv2 describe-rules --listener-arn "$(aws ssm get-parameter \
  --name /${{ values.domain_name }}/${{ values.system_name }}/${{ values.component_name }}/main/EcsAlbListenerArn \
  --query Parameter.Value --output text)"
```

A service changes its own path or priority through `AlbRulePath` and
`AlbRulePriority` in its `cfn-parameters.json`.

{% endif %}
## Change the Service Connect namespace

1. Change `ServiceConnectNamespace` in `cfn-parameters.json`, currently
   `${{ values.system_name }}.local`.
2. Deploy the cluster, then confirm the cluster's default namespace and the
   `ServiceConnectNamespace` SSM parameter.
3. Redeploy every service in the cluster so it registers in the new namespace,
   and update the addresses its callers use.

Services resolve the namespace only when they deploy. Until each one is
redeployed it stays registered under the old name, and services in different
namespaces cannot call each other. Plan the order of redeployment. The old
Cloud Map namespace is not deleted by this stack.

{% if values.create_load_balancer %}
## Change the hostname

1. Confirm the new name belongs to the hosted zone named by `/org/dns/ZoneId`.
2. Change `Hostname` in `cfn-parameters.json`, currently
   `${{ values.hostname }}`.
3. Deploy. CloudFormation issues a new certificate, switches the listener to
   it, and replaces the DNS record.

The old hostname stops resolving once the record is replaced. Every service on
the load balancer moves to the new hostname together; update clients and any
configuration that names the old one.

## Restrict or extend load balancer access

Edit `EcsAlbSecurityGroup` in `template.yaml` to narrow the source range, or
add resources to `template.yaml` for an HTTP listener that redirects to HTTPS,
a specific `SslPolicy`, AWS WAF, or deletion protection. Keep the security
group itself in place: service stacks reference its ID to allow traffic to
their tasks, and replacing it breaks that rule until each service redeploys.

To remove the load balancer entirely, first redeploy every service without
**Use cluster load balancer?**. The listener cannot be deleted while service
rules are attached to it.

{% endif %}
## Change SSM parameter paths

The parameter names in `template.yaml` are built from `Domain`, `System`,
`Component`, and `CodeBranch`; the ECS service templates generate matching
names into each service's `cfn-parameters.json`. To change a path segment or a
parameter name:

1. Publish the new names **in addition to** the existing ones by adding
   `AWS::SSM::Parameter` resources, and deploy the cluster.
2. Update each consuming service's `cfn-parameters.json` to the new names and
   redeploy it.
3. Remove the old parameters from `template.yaml` once no service reads them.
4. Update the service template source if new services should generate the new
   names.

Renaming in a single step deletes the names services still read. Running
services continue, but none can deploy until both sides agree again.

## Add monitoring

The stack creates no alarms or dashboards{% if values.create_load_balancer %} and no load balancer access logs{% endif %}.
Add them to `template.yaml`:

- Set `ClusterSettings` on `EcsCluster` to enable Container Insights when you
  need task-level metrics.
{%- if values.create_load_balancer %}
- Add `LoadBalancerAttributes` on `EcsAlb` to deliver access logs to an S3
  bucket whose policy allows Elastic Load Balancing to write to it.
- Alarm on `HTTPCode_ELB_5XX_Count` for the load balancer. Alarms on a
  service's `UnHealthyHostCount` and `HTTPCode_Target_5XX_Count` belong in that
  service's stack, because it owns the target group.
{%- endif %}

Datadog collection is automatic; see
[Observability](06-observability.md#availability-in-datadog).

## Validate and deploy changes

Run `sam validate --lint` and `sam build --parallel --template template.yaml`.
No workflow runs on feature branches, so these local checks are the only
validation before `main`.

Review the change for anything that replaces the cluster{% if values.create_load_balancer %}, load balancer, listener, security group,{% endif %}
or an SSM parameter, because each replacement affects every consuming service.
Commit the related template, parameter, and handbook updates together and open
a PR under your review policy. Remember that **merging to `main` deploys**, and
do not dispatch the workflow from another ref; see
[CI/CD pipeline](05-ci-cd.md#deploying-from-other-refs).

After deployment, verify the SSM parameter values and redeploy one consuming
service to confirm the contract still resolves.
