# Common changes and tasks

## Set up a development environment

Install the AWS SAM CLI. From the **generated repository root**, run the same
checks as the build job:

```sh
sam validate --lint
sam build --parallel --template template.yaml
```

A successful build does not prove that account-specific SSM parameters, IAM
permissions, certificate validation, or DNS are ready.

## Deploy the cluster

The workflow deploys on pushes to `main` and on manual dispatch. Before the
first deployment, complete the [deployment prerequisites](03-configuration.md#deployment-prerequisites).
Merge changes to `main` through your review policy rather than dispatching from
another ref; see [CI/CD pipeline](05-ci-cd.md#branch-and-manual-deployment-isolation).

## Connect an ECS service

Create the service with the **AWS ECS Service Container** or **Containerized
Development Application** template and select this cluster. The service reads
the cluster's SSM parameters by path; it does not reference the stack directly.

1. Create the service in the **same domain and system** as this cluster. The
   service template builds the SSM paths from its own domain and system.
2. Deploy this cluster from `main` first. The service reads the `main` path.
3. To route public traffic, the cluster needs the ALB. Give the service a path
   pattern and a listener-rule priority that no other service on this listener
   already uses, and make sure its health check path returns success.
4. Call other services privately through Service Connect at
   `<service-component>.<system-name>.local:<port>`.

The service stack owns its target group, listener rule, security group, and
task networking. Track rule priorities and paths for all services on the shared
listener; overlapping patterns are resolved by priority alone.

## Change the Service Connect namespace

1. Change `ServiceConnectNamespace` in `cfn-parameters.json`.
2. Deploy the cluster and confirm the cluster's default namespace and the
   exported `ServiceConnectNamespace` SSM parameter.
3. Redeploy every dependent service so it registers in the new namespace,
   and update any client aliases that callers use.

Services keep their existing registration until redeployed, and callers using
old aliases break once providers move. Plan the order of redeployment.

## Change load balancer or DNS configuration

For a **hostname** change, update `Hostname` in `cfn-parameters.json`. The
certificate and DNS record are replaced; the new name must be in the
configured hosted zone. All services on the ALB move to the new hostname.

To make the **no-ALB option** deployable, render
`ClusterAlbSecurityGroupIdSsmParam` only when the ALB is enabled by moving it
inside the template's existing ALB conditional block in `template.yaml` in the
template source repository. An already-generated cluster without an ALB needs
the same change in its own `template.yaml`. Services for such a cluster must
not select **Use cluster load balancer?**.

To **remove** an ALB from an existing cluster, first remove every consuming
service's listener rule, or the listener cannot be deleted.

## Change SSM parameter paths

The domain, system, component, and branch values form the path prefix in
`template.yaml`; the ECS service template reads specific paths and suffixes.
If changing either:

1. Update the cluster's SSM parameter names.
2. Update every consuming service's `cfn-parameters.json` and the service
   template source.
3. Roll out so services are never deployed against missing parameters. SSM
   parameters are resolved at deploy time, so running services are unaffected
   until they next deploy.
4. Verify parameter values in the target account and region.

## Add monitoring

This stack creates no alarms, dashboards, ALB access logs, or Container
Insights setting. Add them to `template.yaml` according to the workloads'
reliability targets; for example, alarm on `HTTPCode_ELB_5XX_Count` and on
`UnHealthyHostCount` for each target group. Datadog collection is automatic;
see [Observability](06-observability.md#availability-in-datadog).

## Validate and deploy changes

Run `sam validate --lint` and `sam build --parallel --template template.yaml`.
Review the CloudFormation change set for replacements of the cluster, ALB,
listener, certificate, or SSM parameters, because each affects consuming
services. Commit template and handbook updates together and open a PR under
your review policy. After deployment, verify the SSM values and redeploy one
consuming service to confirm the contract still resolves.
