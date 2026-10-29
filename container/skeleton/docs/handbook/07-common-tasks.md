# Common changes and tasks

## Set up a development environment

Install the AWS SAM CLI. From the **generated repository root**, run the same
infrastructure checks as CI:

```sh
sam validate --lint
sam build --parallel --template template.yaml
```

For a Node.js development application, also install Node.js 20 and enable
Corepack so the pinned Yarn version is used, then run the CI application steps:

```sh
cd src
corepack enable
yarn install
yarn run compile
yarn run test:unit
yarn run lint
docker build -t ${{ values.component_name }}:local .
```

The image-based variant has no application source; test the image in its own
repository.

## Develop the Node.js starter

The generated starter compiles nothing and the image runs nothing until you add
code. Before the first useful deployment:

1. Add TypeScript source under `src/src/` (for example `src/src/index.ts`)
   that starts an HTTP server on the container port and answers `GET /` with
   a 2xx status for the ALB health check.
2. Add Jest tests and configuration for TypeScript (the starter includes
   `ts-jest`), and an `eslint.config.ts`. CI runs `compile`, `test:unit`, and
   `lint` before building the image; `jest` fails when it finds no tests.
3. Commit `src/yarn.lock` for repeatable installs.
4. Complete `src/Dockerfile`: install dependencies, copy the compiled `dist/`
   output, set `NODE_ENV=production`, run as the `node` user, and add a `CMD`
   such as `["node", "dist/index.js"]`. Consider a multi-stage build so dev
   dependencies are not shipped.
5. Build and run the image locally, then push a feature branch to confirm the
   CI build before merging to `main`.

## Update the container image

**Image-based service:**

1. Set `ContainerImage` and `ContainerTag` in `cfn-parameters.json`. The task
   definition uses `<ContainerImage>:<ContainerTag>`.
2. Use an immutable version tag. With a mutable tag such as `latest`,
   pushing a new image does not change the template, so nothing deploys, and
   tasks replaced later silently pick up whatever the tag points to.
3. Confirm the image is Linux `amd64`, listens on `ContainerPort`, and that the
   deployment account can pull it.
4. Merge to `main` and verify the new task definition revision and image digest.

**Development application:** merge source changes to `main`. The workflow builds
and pushes the image tagged with the commit SHA and deploys that tag. To roll
back, revert the commit on `main`.

## Configure environment variables, secrets, and health checks

Uncomment and adapt the relevant blocks on the application container in
`template.yaml`:

- `Environment` for non-sensitive configuration, typically from new template
  parameters supplied in `cfn-parameters.json`.
- `Secrets` for sensitive values, with `ValueFrom` set to a Secrets Manager
  secret ARN or an SSM parameter ARN. ECS reads them at task start using the
  **execution role**, so grant `secretsmanager:GetSecretValue` or
  `ssm:GetParameters` (and `kms:Decrypt` for customer-managed keys) on those
  specific ARNs to `ExecutionIamRole`.
- `HealthCheck` with a command that exists in the image. Alpine-based images
  include `wget` but not `curl`.

Do not put literal credentials in `template.yaml` or `cfn-parameters.json`.
Changes to these values only reach running tasks through a new deployment.

## Change CPU, memory, and desired count

Update `ContainerCpu`, `ContainerMemory`, or `ContainerDesiredCount` in
`cfn-parameters.json`. Use a valid Fargate CPU/memory combination and size from
measured demand, remembering that the Datadog sidecar uses 256 CPU / 512 MB of
the task. Keep at least two tasks for availability; with
`MinimumHealthyPercent: 75`, a single-task service must start the replacement
before stopping the old task.

## Configure a port or load-balancer route

Change `ContainerPort`, `AlbRulePath`, and `AlbRulePriority` in
`cfn-parameters.json` together with the application's listening port and
routes. Coordinate priorities and paths with every other service on the
cluster listener: priorities must be unique, and a broad pattern such as `*`
at a low priority value captures traffic intended for others. Changing the port
replaces the target group.

To make the **load-balancer-disabled** option deployable, move the
`ContainerPort` and `VpcId` parameters (and their `cfn-parameters.json`
entries) outside the load-balancer conditional blocks, and require
**Container port** in the form regardless of the load-balancer choice, in the
template source repository. Apply the same change to an already-generated
`template.yaml` that lacks them.

## Call or expose a service through Service Connect

- **To call another service**, use
  `http://<other-component>.<system-name>.local:<port>`. Both services must be
  in the same cluster namespace.
- **To accept calls**, add an `AWS::EC2::SecurityGroupIngress` on
  `ServiceSecurityGroup` for the container port from each caller's security
  group, or from a shared security group your platform defines. The starter
  only admits the ALB.

## Grant application AWS permissions

Add a policy to `TaskIamRole` in `template.yaml` with only the actions and
resource ARNs the application needs. AWS SDKs pick up these credentials
automatically inside the task. Do not add application permissions to
`ExecutionIamRole`; it is for image pulls, logs, and `Secrets` retrieval.

## Enable ECS Exec

Set `EnableContainerExec` to `"true"` in `cfn-parameters.json` and deploy.
Existing tasks must be replaced (a deployment does this) before they accept
sessions. Operators need `ecs:ExecuteCommand` permission and the Session
Manager plugin:

```sh
aws ecs execute-command --cluster <cluster-name> --task <task-id> \
  --container ${{ values.component_name }} --interactive --command "/bin/sh"
```

Follow your organization's approval rules for interactive access to production
and return the value to `"false"` when finished.

## Add or update Datadog

The Datadog option is chosen in Backstage. To change it later, add or remove
the `datadog-agent` container, its two parameters, and their
`cfn-parameters.json` entries together. Ensure `DATADOG_AGENT_API_KEY` and
`DATADOG_SITE` are available to the deploy job.

To keep the API key out of the task definition, store it in Secrets Manager,
replace the `DD_API_KEY` `Environment` entry with a `Secrets` entry referencing
that secret, and grant `ExecutionIamRole` read access to it. Consider pinning
the Agent image to a specific version instead of `latest`.

## Validate and deploy changes

Run `sam validate --lint` and `sam build --parallel --template template.yaml`,
plus the Node.js checks for a development application. Review the
CloudFormation change set for IAM changes, security-group rules, replaced
target groups or listener rules, and the image reference.

Commit related infrastructure, application, and handbook updates together and
open a PR under your review policy. Feature-branch pushes build but do not
deploy; merging to `main` deploys to `production`. After deployment, inspect
the ECS service events, task health, image digest, and logs as described in
[CI/CD pipeline](05-ci-cd.md#reviewing-a-deployment) and
[Observability](06-observability.md), and obtain human review.
