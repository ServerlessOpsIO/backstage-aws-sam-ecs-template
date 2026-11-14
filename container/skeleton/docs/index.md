# Documentation

${{ values.domain_title }} / ${{ values.system_title }} / ${{ values.component_name }}

${{ values.component_description }}

This component is an ECS service that runs on Fargate in the
**${{ values.cluster_component_name }}** cluster of the
**${{ values.system_title }}** system. The repository holds the service's
infrastructure, deployment workflows, and, for a development application, its
source code and container image build.

## Where to start

- **Deploying the service for the first time:** complete the
  [deployment prerequisites](handbook/03-configuration.md#deployment-prerequisites).
- **Changing the service:** find the change in
  [Common tasks](handbook/07-common-tasks.md), then
  [validate and deploy](handbook/07-common-tasks.md#validate-and-deploy-changes).
- **Investigating a problem:** follow the
  [investigation checklist](handbook/06-observability.md#investigation-checklist).
