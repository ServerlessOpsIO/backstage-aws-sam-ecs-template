# Common changes and tasks

## Validate infrastructure changes

Install AWS SAM CLI and use the same template validation/build steps as CI.
From the generated repository root:

```sh
sam validate --lint
sam build --parallel --template template.yaml
```

For the Node.js development-container variant, install the Node.js version
specified by `src/package.json` and the configured Yarn package manager, then
run:

```sh
cd src
yarn install
yarn run compile
yarn run test:unit
yarn run lint
```

The standard image-based ECS service has no application source or unit test
suite in its skeleton. Test the application image in its own repository and
verify that the target ECS account can pull it before deployment.

## Update the container image

1. Set `ContainerImage` and `ContainerTag` in `cfn-parameters.json` to the
   intended repository and tag. The task definition joins them as
   `<image>:<tag>`.
2. Confirm the image architecture is compatible with Linux X86_64 Fargate.
3. Confirm the ECR repository policy and target account allow the task
   execution role to pull the image.
4. Use an immutable version or commit tag for repeatable releases; avoid
   relying on a mutable `latest` tag for production.
5. Deploy through the reviewed workflow and verify the new task revision and
   image digest in ECS.

If the repository builds its own development image, update the source and
Dockerfile, then inspect the workflow's published tags and ECR policy. Ensure
the deployed `ContainerTag` matches a tag that the workflow actually pushes.

## Configure environment variables, secrets, and health checks

The main container definition in `template.yaml` contains commented examples
for `Environment`, `Secrets`, and `HealthCheck`. Uncomment and adapt the
relevant section:

- Use `Environment` for non-sensitive configuration.
- Use `Secrets` with a Secrets Manager or Systems Manager parameter ARN for
  sensitive values; grant the task execution role the required read access.
- Define a process-level health check using a command available in the image.

Do not place literal credentials in `template.yaml` or commit them to
`cfn-parameters.json`. Confirm that each value is available in the task's
account/region and that the principal retrieving it has least-privilege access.

## Change CPU, memory, and desired count

Update `ContainerCpu`, `ContainerMemory`, or `ContainerDesiredCount` in
`cfn-parameters.json` to change task sizing or replica count. Use a
CPU/memory combination supported by Fargate and size capacity for the
application's measured demand. Review deployment settings (`MaximumPercent`
and `MinimumHealthyPercent`) when the desired count is small; a healthy
replacement may require temporary additional capacity.

## Configure a port or load-balancer route

The form supplies the container port, ALB path pattern, and listener priority
when using the cluster load balancer. Change the corresponding
`ContainerPort`, `AlbRulePath`, and `AlbRulePriority` deployment values
together with the application listener. Coordinate priorities with other
services on the shared ALB listener and ensure the health-check endpoint
responds at `/`.

The current template always references `VpcId` and `ContainerPort` for
security-group and Service Connect configuration, but conditionally declares
those parameters only when load balancing is enabled. Do not disable the
load-balancer option until the template supplies these values unconditionally
or otherwise removes the references.

## Grant application AWS permissions

Add required application permissions to `TaskIamRole` in `template.yaml`.
The task execution role is for ECS initialization (image pulls and log
delivery); do not use it to grant runtime application access. Scope actions
and resources to the minimum necessary. If adding secret retrieval, decide
whether the secret is read by ECS at startup or by application code and grant
access to the corresponding role.

## Enable ECS Exec

Set `EnableContainerExec` to `"true"` in `cfn-parameters.json` to enable the
service setting and conditional SSM Messages task-role policy. Verify operator
IAM permissions, task networking to the SSM Messages endpoints, and
organizational approval requirements before using ECS Exec in production.
Return it to `"false"` when interactive access is not needed.

## Add or update Datadog

Toggle the Backstage Datadog option when scaffolding. When enabled, ensure
`DATADOG_AGENT_API_KEY` and `DATADOG_SITE` are available to the deploy job and
that the site value matches the Datadog organization. The generated sidecar is
not configured for log collection or tracing by default; make and review any
Agent configuration changes explicitly.

## Validate and deploy changes

Run `sam validate --lint` and `sam build --parallel --template template.yaml`
before deployment. Review the generated CloudFormation change set, including
IAM, public IP assignment, security-group rules, desired count, and image
reference. Confirm `AWS_CICD_ACCOUNT_ID`, `DEPLOYMENT_ACCOUNT_ID`, and any
Datadog secrets are configured in GitHub.

Only `main.yml` currently deploys. The branch workflow builds but does not
deploy; do not assume feature branches are isolated deployment environments.
After deployment, inspect the ECS service events, task health, image revision,
and logs as described in [CI/CD pipeline](05-ci-cd.md) and
[Observability](06-observability.md).
