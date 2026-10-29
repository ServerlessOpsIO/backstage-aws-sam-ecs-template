# CI/CD pipeline

## Workflow triggers

| Workflow | Automatic trigger | Manual trigger |
| --- | --- | --- |
| `.github/workflows/main.yml` (`Main`) | Push to `main` | `workflow_dispatch` |

There is no `pull_request` trigger and no non-main branch workflow. A push to a
feature branch runs nothing. Manual dispatch can select any ref and still
**deploys**; select the intended ref carefully.

Both jobs run on `ubuntu-latest` and grant `contents: read` and
`id-token: write`. The latter allows OIDC-based AWS role assumption rather than
committing long-lived AWS credentials.

## Build job

The build job executes these steps in order:

1. `ServerlessOpsIO/gha-setup-workspace@v2` sets up the checked-out workspace.
2. `ServerlessOpsIO/gha-assume-aws-credentials@v1` obtains credentials using
   `AWS_CICD_ACCOUNT_ID`.
3. `aws-actions/setup-sam@v3` installs SAM.
4. `sam validate --lint` validates the template with CloudFormation linting.
5. `sam build --parallel --template template.yaml` builds the SAM artifact.
6. `ServerlessOpsIO/gha-store-artifacts@v2`, with `use_aws_sam: true`, stores
   the build output for the deploy job.

There are no unit tests because the repository contains no application code.
`sam validate --lint` does not catch account-specific problems such as missing
SSM parameters, IAM permissions, or certificate validation. TechDocs generation
is not a workflow step.

## Deploy job

`deploy` depends on `build` and only proceeds after it succeeds. It uses the
GitHub environment named **`production`**.

1. The workspace action runs with `checkout_artifact: true` to retrieve the
   build artifact.
2. The credential action receives both `AWS_CICD_ACCOUNT_ID` and
   `DEPLOYMENT_ACCOUNT_ID` for build/deployment account access.
3. `ServerlessOpsIO/gha-deploy-aws-sam@v1` deploys with the target account ID
   and serialized workflow environment, variables, and secrets.

The shared actions implement artifact handling, credential assumptions, and
parameter/tag expression resolution. Their internals are external to this
repository. Confirm the expected IAM roles, region, artifact storage, and
stack-name behavior against the shared actions' configuration.

See [Configuration](03-configuration.md) for parameter files, tags, secrets,
and AWS prerequisites. GitHub environment protection rules can require
deployment approval or restrict which refs deploy.

## Branch and manual deployment isolation

`CodeBranch` becomes an SSM path segment, but `samconfig.toml` sets one
component-wide stack name and the hostname is fixed. A manual dispatch from
another ref therefore **updates the same stack**, unless the shared deployment
action branch-qualifies the stack name, and replaces its SSM
parameters with ones under a different branch path. Because the stack owns
those parameters, the `main` path is deleted in that update, and every service
that reads `/<domain>/<system>/<component>/main/...` fails its next deployment.

Do not dispatch from a non-main ref unless you have established a platform
policy for branch-specific stack names, hostnames, accounts or environments,
and teardown. There is no branch-deletion cleanup workflow.

## Reviewing a deployment

Inspect the GitHub Actions build and deploy logs, then check the CloudFormation
stack status, events, and resources. With the ALB, the first deployment waits
for ACM DNS validation; a long-running `SiteCertificate` usually means the
hosted zone is wrong or not authoritative. Confirm the SSM parameter values,
then request the hostname: the fixed `{ "healthy": true }` response confirms
DNS, TLS, and the listener only. See [Observability](06-observability.md).
