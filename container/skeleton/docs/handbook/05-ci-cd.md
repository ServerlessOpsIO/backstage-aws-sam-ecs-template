# CI/CD pipeline

## Workflow triggers

| Workflow | Automatic trigger | Manual trigger |
| --- | --- | --- |
| `.github/workflows/main.yml` (`Main`) | Push to `main` | `workflow_dispatch` |
| `.github/workflows/branch.yml` (`Branches`) | Push to any branch except `main` | `workflow_dispatch` |

There is no `pull_request` trigger. A push to a feature branch starts its
workflow without needing a PR. Manual dispatch is also supported; select the
intended workflow and ref carefully, because dispatching `Main` from any ref
**deploys** that ref.

All jobs run on `ubuntu-latest` and grant `contents: read` and
`id-token: write`. The latter allows OIDC-based AWS role assumption rather than
committing long-lived AWS credentials.

## Jobs by workflow and variant

| Job | `main.yml` | `branch.yml` |
| --- | --- | --- |
| `build_infra` | Always | Always |
| `build_container` | Development variant | Development variant |
| `deploy` | Always, after the build jobs | Never |

## Infrastructure build job (`build_infra`)

1. `ServerlessOpsIO/gha-setup-workspace@v2` sets up the checked-out workspace.
2. `ServerlessOpsIO/gha-assume-aws-credentials@v1` obtains credentials using
   `AWS_CICD_ACCOUNT_ID`.
3. `aws-actions/setup-sam@v3` installs SAM.
4. `sam validate --lint` validates the template with CloudFormation linting.
5. `sam build --parallel --template template.yaml` builds the SAM artifact.
6. `ServerlessOpsIO/gha-store-artifacts@v2`, with `use_aws_sam: true`, stores
   the build output for the deploy job.

## Container image build job (`build_container`)

Development variant only. The job runs in parallel with `build_infra`:

1. Sets up the workspace and assumes credentials in the CI/CD account.
2. Node.js only: `ServerlessOpsIO/gha-setup-node@v1` selects Node.js from
   `src/package.json`, then `yarn run compile`, `yarn run test:unit`, and
   `yarn run lint` run in `src/`. A failure stops the image build and the
   deployment.
3. `kciter/aws-ecr-action@v4` builds `src/Dockerfile` with `src/` as the
   context, creates the ECR repository `<owner>/<repository>` in the CI/CD
   account if needed, pushes the tags below, and applies
   `.github/workflows/ecr-policy.json` as the repository policy.

| Workflow | Image tags pushed |
| --- | --- |
| `main.yml` | `latest`, branch slug, short SHA, full SHA |
| `branch.yml` | branch slug, short SHA, full SHA |

The deployment always uses the **full commit SHA** tag, so `latest` and the
branch tags are for people and tooling, not for ECS. Branch builds publish
images to the same repository but never move `latest`.

The image-based variant has no `build_container` job. It deploys whatever
`ContainerImage:ContainerTag` names; publishing that image is the
responsibility of the image's own pipeline.

## Deploy job

`deploy` exists only in `main.yml`. It depends on `build_infra` and, in the
development variant, `build_container`, and only proceeds after they succeed.
It uses the GitHub environment named **`production`**.

1. The workspace action runs with `checkout_artifact: true` to retrieve the
   build artifact.
2. The credential action receives both `AWS_CICD_ACCOUNT_ID` and
   `DEPLOYMENT_ACCOUNT_ID` for build/deployment account access.
3. `ServerlessOpsIO/gha-deploy-aws-sam@v1` deploys with the target account ID
   and serialized workflow environment, variables, and secrets.

CloudFormation waits for the ECS deployment to finish. If new tasks cannot
start or stay healthy, the deployment circuit breaker rolls the service back
and the stack update fails; see
[Architecture](04-architecture.md#deployment-rollout-sequence).

The shared actions implement artifact handling, credential assumptions, and
parameter/tag expression resolution. Their internals are external to this
repository. Confirm the expected IAM roles, region, artifact storage, and
stack-name behavior against the shared actions' configuration.

## Feature branches and stack identity

Pushing a feature branch validates the template and, in the development
variant, tests the application and publishes a branch-tagged image. **It does
not deploy.** There is no preview environment out of the box.

`CodeBranch` only changes the task-definition family and a Docker label.
`samconfig.toml` has a component-wide stack name, and the service name,
listener-rule priority, and path are fixed. Before adding branch deployments,
establish and verify a platform policy:

1. Give each branch a distinct stack name and ECS service name.
2. Give each branch a distinct listener-rule priority and path; two stacks
   cannot share a priority on the cluster listener.
3. Deploy branch images by SHA, never by `latest`.
4. Use an appropriate account/environment and approval rules that prevent
   unintended production changes.
5. Define teardown for branch stacks. There is no branch-deletion cleanup
   workflow, and ECR images accumulate without a lifecycle policy.

## Reviewing a deployment

Inspect the GitHub Actions logs, then check the CloudFormation stack status,
the ECS service's **Deployments** and **Events**, running task count, and the
image digest of the running tasks. With the load balancer, check target health
and request the service path on the cluster hostname. Use
[Observability](06-observability.md) to follow logs and metrics.
