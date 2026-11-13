# CI/CD pipeline

## Workflow triggers

| Workflow | Automatic trigger | Manual trigger |
| --- | --- | --- |
| `.github/workflows/main.yml` (`Main`) | Push to `main` | `workflow_dispatch` |

There is no `pull_request` trigger and no branch workflow. A push to a feature
branch starts nothing, so a change is first validated by the pipeline when it
reaches `main`; run the local checks in
[Common tasks](07-common-tasks.md#validate-and-deploy-changes) before merging.
Manual dispatch accepts any ref and **deploys** it; select the ref carefully.

The workflow runs on `ubuntu-latest` and grants jobs `contents: read` and
`id-token: write`. The latter allows OIDC-based AWS role assumption rather
than committing long-lived AWS credentials.

## Build job

The build job executes these steps in order:

1. `ServerlessOpsIO/gha-setup-workspace@v2` sets up the checked-out workspace.
2. `ServerlessOpsIO/gha-assume-aws-credentials@v1` obtains credentials using
   `AWS_CICD_ACCOUNT_ID`.
3. `aws-actions/setup-sam@v3` installs SAM.
4. `sam validate --lint` validates infrastructure with CloudFormation linting.
5. `sam build --parallel --template template.yaml` prepares the template for
   deployment. There is no code to compile or package.
6. `ServerlessOpsIO/gha-store-artifacts@v2`, with `use_aws_sam: true`, stores
   the build output for the deploy job.

There are no tests, and TechDocs generation is not a step in this workflow.
Validation checks the template's structure only. It does not show that the
`/org/...` SSM parameters exist, that the deployment role has the needed
permissions, or that the hosted zone can validate a certificate.

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
AWS prerequisites, and SAM defaults. GitHub environment protection rules can
require deployment approval or restrict which refs deploy.

## Deploying from other refs

`CodeBranch` is resolved from `$env.GITHUB_REF_SLUG_CS` and is part of every
SSM parameter name the stack publishes. That does not make a second ref a
separate deployment:

- `samconfig.toml` has one component-wide default stack name.
{%- if values.create_load_balancer %}
- `cfn-parameters.json` has one fixed hostname from scaffolding.
{%- endif %}
- The workflow always selects the GitHub environment `production`.

Unless the shared deployment action qualifies the stack name by ref, a manual
dispatch from another ref updates **the same stack**. CloudFormation replaces
each SSM parameter with one under the new ref's path and deletes the one under
`.../main/`. Running services are unaffected, but every service that reads the
`main` path fails its next deployment until the cluster is deployed from `main`
again.

Restrict the `production` environment to `main`, or establish and verify a
platform policy before deploying other refs:

1. Give each deployed ref a distinct stack name{% if values.create_load_balancer %} and hostname{% endif %}
   through the shared deployment action's supported configuration.
2. Use an appropriate account/environment and OIDC trust policy, with approval
   rules that prevent unintended production changes.
3. Decide how services select a cluster that is not published under `main`;
   the ECS service templates only generate `main` paths.
4. Define teardown for the extra stacks. There is no branch-deletion cleanup
   workflow.

## Reviewing a deployment

Inspect the GitHub Actions build and deploy logs, then check the deployed
CloudFormation stack's status, events, and resources. Confirm that the SSM
parameters listed in [Features](01-features.md#ssm-integration-contract) exist
in the target account and region and hold this stack's values.

{% if values.create_load_balancer %}
The first deployment waits for ACM to validate the certificate through DNS. A
`SiteCertificate` that stays in progress usually means `/org/dns/ZoneId` names
a zone that is not authoritative for `${{ values.hostname }}`.

Then request `https://${{ values.hostname }}/`. Before any service attaches, the
fixed `{ "healthy": true }` response confirms DNS, the certificate, and the
listener. It never establishes that a service is healthy.
{% else %}
There is no endpoint to smoke-test. The stack is working when the parameters
resolve and a service created against this cluster deploys.
{% endif %}
Use [Observability](06-observability.md) to investigate the cluster and its
services after deployment.
