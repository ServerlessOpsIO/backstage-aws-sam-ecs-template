# CI/CD pipeline

## Workflow triggers

| Workflow | Automatic trigger | Manual trigger |
| --- | --- | --- |
| `.github/workflows/main.yml` (`Main`) | Push to `main` | `workflow_dispatch` |
| `.github/workflows/branch.yml` (`Branches`) | Push to any branch except `main` | `workflow_dispatch` |

There is no `pull_request` trigger. A feature-branch push starts its workflow
without needing a pull request. Both workflows run on `ubuntu-latest` with
`contents: read` and `id-token: write`; OIDC is used by the shared AWS
credential action rather than committing long-lived AWS credentials.

## Infrastructure build

Both workflows run the infrastructure build steps:

1. `ServerlessOpsIO/gha-setup-workspace@v2` sets up the workspace.
2. `ServerlessOpsIO/gha-assume-aws-credentials@v1` obtains build-account
   credentials using `AWS_CICD_ACCOUNT_ID`.
3. `aws-actions/setup-sam@v3` installs the SAM CLI.
4. `sam validate --lint` validates the infrastructure template.
5. `sam build --parallel --template template.yaml` builds the SAM artifact.
6. `ServerlessOpsIO/gha-store-artifacts@v2` stores the SAM output for deploy.

## Container image build

When the generated workflow is configured for a development-container
application, it also runs `build_container`. For the Node.js variant, that job
sets up Node.js from `src/package.json`, compiles TypeScript, runs unit tests,
and runs lint before building the image. It publishes tags including `latest`
and branch/commit-derived tags to ECR, and applies the repository policy from
`.github/workflows/ecr-policy.json`.

The standard ECS Service Container flow expects an existing image; it does not
contain application source to build by default. Confirm the generated
workflow's conditions and `cfn-parameters.json` image values to determine
whether the repository builds an image or consumes an external one.

The workflow does not add a pull-request trigger. Review the image-building
action and ECR policy in the generated repo before changing image publishing,
tagging, or access controls.

## Deploy job

Only the `main` workflow has a `deploy` job. It depends on the infrastructure
build and, when present, the container image build. It targets the GitHub
environment `production`, retrieves the SAM build artifact, assumes
credentials for the build and deployment accounts, then invokes
`ServerlessOpsIO/gha-deploy-aws-sam@v1` with serialized workflow environment,
variables, and secrets.

The non-main workflow validates and builds infrastructure and may build a
container image, but has **no deploy job**. A feature branch therefore does
not deploy an ECS service from this workflow. Manual dispatch can still target
a branch, so check the selected workflow and ref carefully.

Shared actions implement artifact handling, credential assumption, parameter
and tag expression resolution, and deployment behavior; their internals are
external to this repository. Verify expected IAM roles, region, artifact
storage, stack name, and parameter handling against the shared actions'
configuration.

## Branches and stack identity

`CodeBranch` is derived from the workflow ref and used in the task-definition
family and Docker labels. It does not automatically make deployments safe or
isolated. `samconfig.toml` uses a component-wide stack name, and the
`main` workflow deploys to the shared GitHub environment `production`.

Before adding branch deployments, define distinct stack names, image tags,
accounts/environments, and a cleanup procedure. Verify that each branch cannot
overwrite the production service or unintentionally register a competing ALB
listener rule. There is no branch deployment or branch-stack cleanup workflow
in the current branch pipeline.

## Reviewing a deployment

Inspect GitHub Actions build/deploy logs, the CloudFormation stack status and
events, ECS service events, task health, and the actual image tag running in
the cluster. If load balanced, smoke-test the configured ALB path and health
check. See [Observability](06-observability.md) to follow container logs and
service metrics.
