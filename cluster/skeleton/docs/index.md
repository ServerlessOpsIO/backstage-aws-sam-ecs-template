# Documentation

${{ values.domain_title }} / ${{ values.system_title }} / ${{ values.component_name }}

${{ values.component_description }}

This component is the shared ECS cluster for the services of the
**${{ values.system_title }}** system. It provides the Service Connect
namespace `${{ values.system_name }}.local`
{%- if values.create_load_balancer %}, a shared HTTPS load balancer at
`${{ values.hostname }}`,{% endif %} and the SSM parameters that service
components read to attach to it. It contains no application code.

## Where to start

- **Running a service in this cluster:** read
  [Connect an ECS service](handbook/07-common-tasks.md#connect-an-ecs-service).
- **Deploying the cluster for the first time:** complete the
  [deployment prerequisites](handbook/03-configuration.md#deployment-prerequisites).
- **Investigating a problem:** follow the
  [investigation checklist](handbook/06-observability.md#investigation-checklist).
