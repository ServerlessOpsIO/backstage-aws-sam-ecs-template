# Common changes and tasks

## Validate and build the cluster template

Install the AWS SAM CLI and use the generated repository root:

```sh
sam validate --lint
sam build --parallel --template template.yaml
```

These are the same SAM validation and build operations used by the main
workflow. A successful build does not prove that account-specific SSM
parameters, IAM permissions, certificate validation, or DNS are ready.

## Deploy the cluster

The generated workflow deploys on pushes to `main` and manual dispatches. Before
deploying, confirm the prerequisites in [Configuration](03-configuration.md),
especially the target account's VPC and DNS SSM parameters and GitHub Actions
OIDC settings. Review the selected workflow ref and final stack name; the stack
name is not branch-qualified by default.

## Connect an ECS service

The ECS service template in this repository consumes the cluster contract
through SSM parameters. Configure it with the cluster's domain, system,
component, and branch path, and verify that it reads the expected
`EcsClusterName` and `ServiceConnectNamespace`.

For ALB-backed routing, the cluster must have been created with the ALB enabled
and the service must also be configured to consume `EcsAlbListenerArn` and
`ClusterAlbSecurityGroupId`. The service stack is responsible for its target
group, listener rules, task networking, and application health checks. The
cluster listener's default fixed response is not service routing.

## Change the Service Connect namespace

1. Change the `ServiceConnectNamespace` value in the generated
   `cfn-parameters.json` or update its source in the scaffolder.
2. Deploy the cluster stack and confirm the cluster's Service Connect default
   namespace and exported `ServiceConnectNamespace` SSM parameter.
3. Update dependent service stacks to use the same namespace and redeploy them
   in a coordinated order.

Namespace changes can affect service discovery for existing consumers. Plan
the transition and validate dependent services rather than changing only the
cluster parameter.

## Change load balancer or DNS configuration

The hostname and ALB setting are selected in Backstage and rendered into the
generated parameter file and CloudFormation template. For a hostname change,
review ACM DNS validation, the hosted zone, DNS record replacement, and
consuming-service listener rules.

Do not select the no-load-balancer option until the template is corrected:
`EcsAlbSecurityGroup` is conditional but
`ClusterAlbSecurityGroupIdSsmParam` currently references it unconditionally.
If correcting that behavior, condition the security-group SSM parameter on the
same option and ensure downstream templates do not require ALB parameters when
disabled.

## Change SSM parameter paths

The domain, system, component, and branch values form the path prefix in
`template.yaml`. The ECS service template reads specific paths for cluster and
ALB values. If changing the prefix or suffixes:

1. Update the cluster output parameter names.
2. Update every consuming template's corresponding SSM input path.
3. Coordinate rollout so services are not deployed against missing parameters.
4. Verify parameter values in the target AWS account and region.

## Add monitoring

Choose monitoring appropriate to the workloads and configure it in the
relevant infrastructure. This cluster stack does not create alarms, dashboards,
ALB access logs, or Container Insights. Container log groups and log retention
are normally controlled by each consuming service's task definition. Verify
both CloudWatch collection and any organization-standard downstream integration
after rollout.
