# Creating an ECS service via Backstage

Choose **Create** in Backstage and select **AWS ECS Service Container**. The
template creates a public GitHub repository named `<service-name>-app` in
`ServerlessOpsIO`, then registers the generated Component in Backstage. It
requires GitHub publishing and catalog registration, the template's custom
picker fields, and the relevant catalog entities for the owner, domain, system,
environment, account, and ECS cluster.

Paths below are relative to the **newly generated repository root**, not the
template source repository.

## Page 1: ECS Service Info

| Input | Purpose | Where to see it after creation |
| --- | --- | --- |
| Namespace | Hidden default is `default`. | `metadata.namespace` in `catalog-info.yaml`. |
| ECS Service Name | Name used to create the service repository. | Lowercased and suffixed with `-app` for the GitHub repository, `metadata.name` in `catalog-info.yaml`, and the default SAM stack name in `samconfig.toml`. The deployed ECS service name and container name use the generated repository name. |
| Description | Short description of the component. | GitHub repository description, Component `metadata.description` in `catalog-info.yaml`, and the CloudFormation template description. |
| Owner | Owning group selected from your groups. | Component `spec.owner` in `catalog-info.yaml`; GitHub Settings → Custom properties → `owner`. |
| Domain | Catalog domain for the component and deployment tags. | `Domain` in `cfn-parameters.json`; `org:domain` in `cfn-tags.json`; GitHub Settings → Custom properties → `domain`. |
| System | Catalog system, filtered by the chosen domain. | `System` in `cfn-parameters.json`; `org:system` in `cfn-tags.json`; Component `spec.system` in `catalog-info.yaml`; GitHub Settings → Custom properties → `system`. |
| Environment | Environment associated with the system; it filters the account picker. | The chosen environment is not passed into the generated files. The `main` workflow's deploy job uses the GitHub environment `production`; the branch workflow has no deployment job. |
| Account ID | Cloud-account catalog resource related to the selected environment. | The selected entity's `metadata.title` is written to the GitHub repository secret `DEPLOYMENT_ACCOUNT_ID`. The value is not written to application files, and GitHub does not reveal a saved secret value. Confirm the catalog entity's title before creation. |
| Cluster | Existing ECS cluster in the selected system. | Component `spec.dependsOn` in `catalog-info.yaml`; cluster SSM parameter paths in `cfn-parameters.json`. Cluster values are selected from the catalog entity name. |

The Component's `spec.lifecycle` is `production`; its catalog namespace comes
from the hidden namespace default. The repository is created with `main` as
the default branch. The publisher configuration disables default-branch
protection and sets the required approving review count to zero; apply your
organization's review policy before development.

## Page 2: Container Configuration

| Input | Purpose | Where to see it after creation |
| --- | --- | --- |
| Container Image | Container image repository to run. Supply the image repository URI expected by ECS; the template appends the tag. | `ContainerImage` in `cfn-parameters.json`; the task definition's container `Image` in `template.yaml`. |
| Container Tag | Image tag; defaults to `latest`. | `ContainerTag` in `cfn-parameters.json`; appended to the image repository in the task definition. For repeatable deployments, prefer an immutable tag rather than `latest`. |
| Container CPU shares | CPU allocation units; defaults to `1024` (one vCPU). | `ContainerCpu` in `cfn-parameters.json` and `TaskDefinition.Properties.Cpu` in `template.yaml`. |
| Container Memory | Task memory in MB; defaults to `2048`. | `ContainerMemory` in `cfn-parameters.json` and `TaskDefinition.Properties.Memory` in `template.yaml`. Choose a CPU/memory combination supported by Fargate. |
| Container Count | Desired number of running tasks; defaults to `3`. The form notes that production services typically run one per Availability Zone (often three). | `ContainerDesiredCount` in `cfn-parameters.json` and `Service.Properties.DesiredCount` in `template.yaml`. |

The task runs on Linux Fargate with the X86_64 CPU architecture. The generated
task definition does not add application-specific environment variables,
secrets, or container health checks by default; configure these in
`template.yaml` if the application requires them.

## Page 3: Infrastructure Configuration

| Input | Purpose | Where to see it after creation |
| --- | --- | --- |
| Enable Datadog integration? | Defaults to enabled. Adds a Datadog Agent sidecar to the task. | Conditional `DatadogApiKey` and `DatadogSite` parameters and `datadog-agent` container in `template.yaml`; values are supplied by deployment secrets. |
| Use cluster load balancer? | Defaults to enabled. Adds a listener rule and target group using the selected cluster's load balancer. | Conditional load-balancer parameters and `AlbTargetGroup`, `AlbListenerRule`, and service `LoadBalancers` in `template.yaml`. |
| Container port | Port exposed by the application; required when load balancing is enabled. | `ContainerPort` in `cfn-parameters.json`; task port mapping, Service Connect alias, target group, and security-group ingress in `template.yaml`. |
| Load balancer path | Listener path pattern; defaults to `*`. For example, `/app/*` routes paths under `/app`. | `AlbRulePath` in `cfn-parameters.json` and the listener rule path condition in `template.yaml`. |
| Load balancer rule priority | Listener rule priority; defaults to `1`. Lower values are evaluated first. | `AlbRulePriority` in `cfn-parameters.json` and the listener rule in `template.yaml`. Coordinate priorities with other services on the shared listener. |

The service always enables ECS Service Connect and uses the cluster's
Service Connect namespace. It also always creates a service security group,
which references the VPC ID, and the Service Connect configuration references
the container port. In the current template, **turning off the load balancer
omits both the VPC ID and container-port parameters even though these
resources still reference them**. Keep the load balancer enabled unless the
template is changed to provide those values independently.

The service uses the cluster's public subnet parameter and sets
`AssignPublicIp` to `ENABLED`. Review the generated networking configuration
with your platform team if the service must run without public IP assignment.

## Deployment and generated files

The scaffolder copies the service skeleton and pipeline into the new
repository, publishes it as a **public** repository, stores
`DEPLOYMENT_ACCOUNT_ID`, and registers `catalog-info.yaml` in Backstage.

| File | What it configures |
| --- | --- |
| `template.yaml` | Fargate task definition, ECS service, IAM roles, security group, Service Connect, CloudWatch logs, and optional load-balancer resources. |
| `cfn-parameters.json` | Deployment parameters, including selected cluster SSM paths, image, task sizing, and optional integration/load-balancer settings. |
| `cfn-tags.json` | Domain, system, and component CloudFormation tags. |
| `samconfig.toml` | SAM stack name and build, validation, and deploy defaults. |
| `catalog-info.yaml` | Backstage Component metadata, ownership, system, cluster dependency, and account dependency. |
| `.github/workflows/main.yml` | Validates and builds the SAM template on pushes to `main`; builds the container image when the pipeline is configured to do so; deploys the SAM application. |
| `.github/workflows/branch.yml` | Validates and builds the SAM template on non-`main` branches; it does not include a deployment job. |

The workflows require `AWS_CICD_ACCOUNT_ID` and use the
`DEPLOYMENT_ACCOUNT_ID` secret created by scaffolding. With Datadog enabled,
the deployment also expects `DATADOG_AGENT_API_KEY` and `DATADOG_SITE` secrets.
The `main` workflow deploys to the GitHub environment `production`, even when
a different catalog Environment was selected in the form. The branch workflow
builds but does not deploy.

The container pipeline references the image URI and tag as separate values.
Verify that the deploy account and ECS task execution role can pull the image
before expecting the service to start successfully.

## Review and create

Before submitting, review the generated repository name, catalog owner and
relationships, AWS account, cluster, image URI and tag, task sizing, and
load-balancer route. The scaffolder then:

1. Fetches the selected owner, domain, system, account, and cluster catalog
   entities.
2. Renders the ECS service skeleton and GitHub Actions workflows.
3. Publishes the repository to `ServerlessOpsIO` with `main` as the default
   branch and saves `DEPLOYMENT_ACCOUNT_ID`.
4. Registers the generated Component in Backstage.

After creation, confirm the Component's catalog relationships, review the
repository's branch protection and deployment environment policies, and verify
the required GitHub secrets and AWS image-pull access before relying on the
first deployment.
