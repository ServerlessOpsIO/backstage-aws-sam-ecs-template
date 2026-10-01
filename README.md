# Backstage AWS SAM ECS Templates

This repository contains Backstage scaffolder templates for creating AWS ECS-based application infrastructure with AWS SAM and GitHub Actions. It is designed to help platform teams generate consistent ECS projects from Backstage, publish them to GitHub, and register them back into the catalog with a repeatable deployment pattern.

The templates in this repo cover the core building blocks needed to stand up a production-style ECS environment:

- ECS cluster creation with optional Application Load Balancer support
- ECS service and container deployment on Fargate
- development-focused container app scaffolding for Node.js-based services

## What this repository creates

When used from Backstage, each template generates a repository skeleton that includes:

- an AWS SAM project for ECS infrastructure
- an ECS cluster definition with optional ALB and Route53 configuration
- an ECS service and task definition for container deployment
- security groups, IAM roles, and networking configuration for Fargate workloads
- optional Datadog integration for task sidecars
- GitHub Actions workflows for validation and deployment
- Backstage catalog metadata ready for registration

## Included templates

### 1. ECS Cluster template

The cluster template creates a new repository for an ECS cluster and can optionally provision:

- an Application Load Balancer
- HTTPS listener and certificate management
- Route53 hostname creation
- ECS cluster resources and Service Connect namespace configuration
- SSM parameters for runtime integration

This is useful when a team wants to create the shared cluster foundation before deploying individual application containers.

### 2. ECS Container template

The container template creates an application repository for a containerized service deployed into an existing ECS cluster. It includes:

- a Fargate task definition
- ECS service configuration with Service Connect
- optional ALB integration for public traffic routing
- configurable CPU, memory, port, and desired count settings
- optional Datadog sidecar configuration
- AWS IAM role and security group scaffolding

### 3. Development container template

The development template extends the container flow for application development work and supports:

- Node.js application scaffolding
- optional load balancer routing and Datadog integration
- deployment into a selected ECS cluster from Backstage
- generated GitHub Actions pipeline and repository setup

## Repository layout

- `template.yaml` — the Backstage scaffolder template definition containing the ECS cluster, service, and dev application templates
- `cluster/` — project skeleton and pipeline templates for cluster creation
- `cluster/skeleton/` — generated ECS cluster SAM files and metadata
- `cluster/pipeline/` — GitHub Actions pipeline files for the cluster project
- `container/` — project skeleton and pipeline templates for service/container deployment
- `container/skeleton/` — generated ECS service SAM files and metadata
- `container/pipeline/` — GitHub Actions pipeline files for the container project
- `container/nodejs_app/` — Node.js starter application template used by development app generation

## Included technology

These generated projects are built around standard AWS container and deployment tooling:

- AWS SAM for infrastructure-as-code and deployment
- Amazon ECS on Fargate
- Application Load Balancer for public traffic routing
- AWS Systems Manager Parameter Store for environment integration
- Amazon Route53 and Certificate Manager for DNS and TLS when enabled
- Datadog agent sidecar support for observability
- GitHub Actions for CI/CD automation

## Typical use in Backstage

The scaffolder prompts for component metadata such as:

- component name and description
- owning group
- domain and system
- deployment environment
- target AWS account
- ECS cluster reference for application services
- container image, tag, CPU, memory, and desired count settings
- whether to enable Datadog and/or a load balancer

It then:

1. fetches the related Backstage catalog entities
2. copies the appropriate project skeleton
3. adds the ECS infrastructure or service resources
4. creates GitHub Actions pipeline files
5. publishes the repository to GitHub
6. registers the generated component back in the catalog

## Generated project structure

Each generated project is intentionally opinionated and ready to be customized for a real service. Typical generated content includes:

- AWS SAM `template.yaml` files for ECS resources
- catalog metadata in `catalog-info.yaml`
- deployment configuration in `samconfig.toml`
- environment parameter and tag files for AWS deployment
- repository-level `.gitignore` and project README
- CI/CD pipelines to build and deploy the stack

## Requirements

Before using these templates in a Backstage installation, make sure the following are available:

- Backstage with the Scaffolder plugin enabled
- GitHub publisher integration configured for repository creation
- Catalog entities for domain, system, owner, environment, and cloud account
- AWS account and VPC/subnet configuration for the target deployment environment
- Route53 hosted zone and certificate configuration if load balancer hostname support is required
- AWS SAM and deployment tooling available in the generated project pipeline

## Next steps

After generating a cluster or service from this template:

- customize the generated SAM resources to match your container architecture
- update the container image and runtime settings for your real application
- adjust load balancer routing, port mappings, and security settings for the environment
- configure Datadog credentials and deployment parameters for your AWS account
- extend the generated Node.js app or replace it with your own service implementation

This repository is a strong starting point for teams standardizing ECS-based application delivery through Backstage with repeatable AWS deployment patterns.
