# CI/CD pipeline

## Workflow triggers

| Workflow | Automatic trigger | Manual trigger |
| --- | --- | --- |
| `.github/workflows/main.yml` (`Main`) | Push to `main` | `workflow_dispatch` |

There is no pull-request trigger and no non-main branch workflow in this
cluster skeleton. A push to a feature branch does not run this workflow
automatically. A manual dispatch can select a ref; review the selected ref
carefully because the deployment job still targets the configured stack.

Both jobs run on `ubuntu-latest` and grant `contents: read` and `id-token:
write`. The latter permits OIDC-based AWS role assumption rather than storing
long-lived AWS credentials in the repository.

## Build job

The build job runs these steps:

1. `ServerlessOpsIO/gha-setup-workspace@v2` prepares the workspace.
2. `ServerlessOpsIO/gha-assume-aws-credentials@v1` obtains build-account
   credentials using `AWS_CICD_ACCOUNT_ID`.
3. `aws-actions/setup-sam@v3` installs AWS SAM.
4. `sam validate --lint` validates the SAM/CloudFormation template.
5. `sam build --parallel --template template.yaml` builds the SAM artifact.
6. `ServerlessOpsIO/gha-store-artifacts@v2` stores the build output for deploy.

The cluster workflow does not run application unit tests, because this
repository contains no application code or tests. It also does not generate
TechDocs.

## Deploy job

The deploy job depends on a successful build and uses the GitHub Actions
environment named `production`.

1. The workspace action retrieves the build artifact with
   `checkout_artifact: true`.
2. The credential action receives `AWS_CICD_ACCOUNT_ID` and
   `DEPLOYMENT_ACCOUNT_ID`.
3. `ServerlessOpsIO/gha-deploy-aws-sam@v1` deploys the SAM template to the
   deployment account and receives the workflow environment, variables, and
   secrets for parameter and tag expression resolution.

The shared actions implement artifact handling, credential assumptions, and
deployment input resolution. Their internals are external to this repository.
Verify the required IAM roles, region, artifact storage, and final stack-name
behavior against the action configuration. GitHub environment protection rules
can require approval or restrict deployment access.

See [Configuration](03-configuration.md) for repository secrets and AWS
prerequisites.

## Branch and manual deployment isolation

`CodeBranch` becomes an SSM path segment, but `samconfig.toml` sets one
component-wide stack name. The workflow's deployment environment is always
`production`. A manual dispatch from another ref can therefore update the same
stack while publishing values under a different branch path. Do not treat
manual dispatch as an isolated preview deployment.

Before enabling non-main deployments, define and verify a platform policy for
branch-specific stack naming, account/environment selection, and cleanup.
Confirm the final stack name and all SSM paths in the deployment output. No
branch-deletion cleanup workflow is included.

## Reviewing a deployment

Review the GitHub Actions build and deploy logs, then inspect the deployed
CloudFormation stack status, resources, and SSM parameter values. With the ALB
enabled, verify certificate validation and DNS before relying on its endpoint.
The listener's fixed response only confirms that the ALB listener responds; it
does not verify a consuming service. See [Observability](06-observability.md).
