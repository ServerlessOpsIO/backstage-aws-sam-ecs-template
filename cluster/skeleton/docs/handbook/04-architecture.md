# Architecture

## Infrastructure

### Logical architecture

The cluster is shared platform infrastructure. It separates the public entry
point, workload placement, and service discovery, and it publishes an
integration contract for the services that run in it:

- The ECS cluster is the placement boundary for services. It owns no compute
  capacity; consuming services run on Fargate.
- The cluster's Service Connect namespace gives services in the system a
  private discovery domain.
- An optional shared, internet-facing ALB terminates TLS for one hostname.
  Each consuming service attaches its own listener rule and target group.
- SSM parameters publish the identifiers consumers need, so service stacks never
  hard-code generated names or ARNs.

#### Logical architecture diagram

```mermaid
flowchart LR
    Client["Internet client"]
    Entry["Shared HTTPS entry point (optional)"]
    Default["Default fixed response"]
    Cluster["ECS cluster"]
    Namespace["Service Connect namespace"]
    Contract["SSM integration contract"]
    Services["Consuming ECS services"]

    Client --> Entry
    Entry -->|"No service rule matches"| Default
    Entry -->|"Service path rule"| Services
    Services -->|"Run in"| Cluster
    Services -->|"Discover each other through"| Namespace
    Contract -.->|"Read at deploy time"| Services
    Cluster -.->|"Publishes"| Contract
```

#### Request routing call sequence

The listener evaluates rules added by consuming services in ascending priority
order and falls back to the cluster's default action.

```mermaid
sequenceDiagram
    participant Client
    participant DNS as Route 53
    participant ALB as Shared ALB HTTPS listener
    participant Rule as Service listener rule
    participant Target as Service target group

    Client->>DNS: Resolve hostname
    DNS-->>Client: ALB addresses (alias record)
    Client->>ALB: HTTPS request on 443
    Note over ALB: TLS terminated with the stack's ACM certificate
    alt Path matches a service rule
        ALB->>Rule: Matching rule with the lowest priority value
        Rule->>Target: Forward to a healthy task IP
        Target-->>Client: Service response
    else No service rule matches
        ALB-->>Client: 200 fixed response with healthy true
    end
```

The fixed response is the listener's default action. It does not indicate that
any service is running or healthy.

### Physical architecture

```mermaid
flowchart TD
    Zone["Route 53 hosted zone (existing)"]
    Record["Route 53 alias A record"]
    Certificate["ACM certificate"]
    Subnets["Public subnets (existing)"]
    SG["ALB security group: TCP 443 from 0.0.0.0/0"]
    ALB["Internet-facing ALB"]
    Listener["HTTPS listener :443"]
    Cluster["ECS cluster"]
    CloudMap["Cloud Map namespace: system-name.local"]
    SSM["SSM parameters"]
    Services["Consuming service stacks (external)"]

    Zone --- Record
    Record -->|"Alias target"| ALB
    Certificate -->|"TLS certificate"| Listener
    Zone -.->|"DNS validation records"| Certificate
    Subnets --- ALB
    SG --- ALB
    ALB --> Listener
    Cluster -->|"Service Connect default"| CloudMap
    Services -->|"Listener rules and target groups"| Listener
    Services -->|"Fargate services"| Cluster
    SSM -.->|"Resolved at deploy time"| Services
```

`template.yaml` owns the cluster, SSM parameters, and the optional ALB, its
security group, listener, certificate, and DNS record. The VPC, subnets, hosted
zone, the `/org/...` SSM parameters, and CI roles are external dependencies.
The ALB, listener, certificate, DNS record, and their security group are
omitted when the ALB is disabled.

### CloudFormation template

`template.yaml` is the infrastructure-as-code entry point. Its
`Transform: AWS::Serverless-2016-10-31` enables AWS SAM processing; the
resources themselves are standard CloudFormation ECS, EC2, Elastic Load
Balancing, ACM, Route 53, and SSM types. Backstage renders the template's Nunjucks conditional
blocks at creation time, so the generated file contains only the resources
for the selected option rather than CloudFormation `Conditions`.

| Section/resource | Purpose |
| --- | --- |
| `Parameters` | Domain/system/component metadata, branch slug, VPC, Service Connect namespace, and, with the ALB, subnets, hostname, and hosted zone. |
| `EcsCluster` | The cluster and its Service Connect default namespace. |
| `EcsAlbSecurityGroup`, `EcsAlb`, `EcsAlbListener` | Shared ALB, its public ingress, and the HTTPS listener with a fixed-response default action. |
| `SiteCertificate`, `SiteDnsRecord` | DNS-validated certificate and alias record for the hostname. |
| `EcsClusterNameSsmParam`, `ServiceConnectNamespaceSsmParam` | Always-published integration values. |
| `EcsAlbListenerArnSsmParam` | Listener ARN; rendered with the ALB. |
| `ClusterAlbSecurityGroupIdSsmParam` | ALB security-group ID; currently rendered unconditionally. |

#### Template resource relationships

Arrows run from the referencing resource to its target; they show
infrastructure references, not request flow.

```mermaid
flowchart TD
    VpcId["VpcId parameter"]
    Subnets["VpcSubnets parameter"]
    ZoneId["DnsZoneId parameter"]
    Hostname["Hostname parameter"]
    NamespaceParam["ServiceConnectNamespace parameter"]
    Path["Domain / System / Component / CodeBranch parameters"]
    Cluster["EcsCluster"]
    SG["EcsAlbSecurityGroup"]
    ALB["EcsAlb"]
    Listener["EcsAlbListener"]
    Cert["SiteCertificate"]
    Record["SiteDnsRecord"]
    ClusterSsm["EcsClusterNameSsmParam"]
    NamespaceSsm["ServiceConnectNamespaceSsmParam"]
    ListenerSsm["EcsAlbListenerArnSsmParam"]
    SgSsm["ClusterAlbSecurityGroupIdSsmParam"]

    Cluster -->|"ServiceConnectDefaults"| NamespaceParam
    SG -->|"VpcId"| VpcId
    ALB -->|"Subnets"| Subnets
    ALB -->|"SecurityGroups"| SG
    Listener -->|"LoadBalancerArn"| ALB
    Listener -->|"CertificateArn"| Cert
    Cert -->|"DomainName"| Hostname
    Cert -->|"Validation HostedZoneId"| ZoneId
    Record -->|"Name"| Hostname
    Record -->|"HostedZoneId"| ZoneId
    Record -->|"AliasTarget"| ALB
    ClusterSsm -->|"Value"| Cluster
    NamespaceSsm -->|"Value"| NamespaceParam
    ListenerSsm -->|"Value"| Listener
    SgSsm -->|"Value"| SG
    ClusterSsm -.->|"Name path"| Path
    NamespaceSsm -.->|"Name path"| Path
    ListenerSsm -.->|"Name path"| Path
    SgSsm -.->|"Name path"| Path
```

`SgSsm → SG` is the reference that breaks the ALB-disabled rendering.
`cfn-parameters.json` and `cfn-tags.json` provide deployment-action inputs,
while `samconfig.toml` supplies SAM defaults. Validate changes with
`sam validate --lint` and `sam build` before deployment.

## Repository file layout

```text
catalog-info.yaml          Backstage Component entity
mkdocs.yml                 TechDocs site configuration
docs/handbook/             Handbook pages
README.md                  Project entry point
template.yaml              SAM/CloudFormation cluster infrastructure
cfn-parameters.json        Deployment-action parameter expressions
cfn-tags.json              Deployment-action tag expressions
samconfig.toml             SAM build, validation, and deploy defaults
.github/workflows/main.yml Main-branch build and deployment workflow
```

These are generated-project paths. The template source repository keeps their
inputs under `cluster/skeleton/` and `cluster/pipeline/` before the scaffolder
combines them. The cluster repository has no application source code or tests;
services and their images live in separate consuming components.
