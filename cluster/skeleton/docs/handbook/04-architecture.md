# Architecture

## Logical architecture

The cluster stack provides shared ECS and networking integration points:

- The ECS cluster is configured with Service Connect defaults.
- A conditional internet-facing ALB provides an HTTPS listener and a DNS
  endpoint, but the starter listener returns a fixed response rather than
  forwarding to an application.
- SSM parameters publish the cluster name, Service Connect namespace, and
  (when enabled) ALB listener and security group identifiers.
- Consuming application stacks create services, tasks, capacity, target groups,
  and routing rules as needed.

#### Logical architecture diagram

```mermaid
flowchart LR
    Scaffold["Backstage scaffolder"]
    Workflow["GitHub Actions: SAM build and deploy"]
    Stack["CloudFormation cluster stack"]
    Cluster["ECS cluster"]
    Namespace["Service Connect namespace"]
    ALB["Optional public ALB + HTTPS listener"]
    DNS["Route 53 alias + ACM certificate"]
    Params["SSM integration parameters"]
    Service["Consuming ECS service stack"]

    Scaffold --> Workflow --> Stack
    Stack --> Cluster
    Cluster --> Namespace
    Stack -. conditional .-> ALB
    ALB --> DNS
    Stack --> Params
    Params --> Service
    Service --> Cluster
    Service -. when using shared ALB .-> ALB
```

The ALB listener has a fixed-response default action and no target group or
application route in this template. A service stack must configure its own
target registration and routing rules before the ALB can serve application
traffic.

## Physical architecture

```mermaid
flowchart TD
    VPC["Existing VPC"]
    Subnets["Existing public subnets (ALB only)"]
    Zone["Existing Route 53 hosted zone (ALB only)"]
    ECS["ECS cluster"]
    SC["Service Connect default namespace"]
    SG["ALB security group: inbound TCP 443 from 0.0.0.0/0"]
    ALB["Internet-facing Application Load Balancer"]
    Listener["HTTPS listener :443; fixed JSON response"]
    Cert["ACM DNS-validated certificate"]
    Record["Route 53 alias A record"]
    SSM["SSM parameters: cluster, namespace, ALB identifiers"]
    Consumer["Consuming service stack (external)"]

    VPC --> ECS
    ECS --> SC
    VPC --> SG
    Subnets --> ALB
    SG --> ALB
    ALB --> Listener
    Cert --> Listener
    Zone --> Cert
    Zone --> Record
    ALB --> Record
    ECS --> SSM
    Listener --> SSM
    SSM --> Consumer
    Consumer --> ECS
    Consumer -. optional target/rule integration .-> Listener
```

The VPC, subnet list, and hosted zone are external prerequisites. When the load
balancer is disabled, the unconditional ALB security-group SSM parameter
currently references the conditionally omitted security group; that template
path must be fixed before use.

## CloudFormation template

`template.yaml` is the infrastructure-as-code entry point. The
`Transform: AWS::Serverless-2016-10-31` enables AWS SAM and CloudFormation
expands the template during deployment.

| Section/resource | Purpose |
| --- | --- |
| `Parameters` | Supplies organizational identifiers, branch slug, VPC, Service Connect namespace, and conditional ALB/DNS values. |
| `EcsAlbSecurityGroup` | Allows inbound TCP 443 from any IPv4 address; created only when the ALB option is enabled. |
| `SiteCertificate`, `EcsAlb`, `EcsAlbListener`, `SiteDnsRecord` | Conditional ACM, ALB, listener, and DNS resources. |
| `EcsCluster` | Creates the cluster and its Service Connect defaults. |
| `EcsClusterNameSsmParam`, `ServiceConnectNamespaceSsmParam` | Publish shared cluster integration values. |
| `ClusterAlbSecurityGroupIdSsmParam`, `EcsAlbListenerArnSsmParam` | Publish ALB integration values; the listener parameter is conditional, but the security-group parameter is currently unconditional. |

#### Template resource relationships

```mermaid
flowchart TD
    VpcId["VpcId parameter"]
    Subnets["VpcSubnets parameter"]
    ZoneId["DnsZoneId parameter"]
    Hostname["Hostname parameter"]
    ECS["EcsCluster"]
    Namespace["ServiceConnectNamespace parameter"]
    SG["EcsAlbSecurityGroup"]
    ALB["EcsAlb"]
    Listener["EcsAlbListener"]
    Cert["SiteCertificate"]
    Record["SiteDnsRecord"]
    ClusterParam["EcsClusterNameSsmParam"]
    SGParam["ClusterAlbSecurityGroupIdSsmParam"]
    ListenerParam["EcsAlbListenerArnSsmParam"]
    NamespaceParam["ServiceConnectNamespaceSsmParam"]

    ECS --> Namespace
    ECS --> ClusterParam
    ECS --> NamespaceParam
    VpcId --> SG
    SG --> ALB
    Subnets --> ALB
    ALB --> Listener
    Listener --> ListenerParam
    SG --> SGParam
    Hostname --> Cert
    ZoneId --> Cert
    Cert --> Listener
    Hostname --> Record
    ZoneId --> Record
    ALB --> Record
```

All ALB resources are conditional except `ClusterAlbSecurityGroupIdSsmParam`;
that inconsistency explains why the no-ALB rendering is invalid.

## Generated repository layout

```text
catalog-info.yaml              Backstage Component entity
README.md                       Generated component summary
template.yaml                   SAM/CloudFormation cluster infrastructure
cfn-parameters.json             Deployment-action parameter expressions
cfn-tags.json                   Deployment-action tag expressions
samconfig.toml                  SAM stack and build defaults
docs/handbook/                  This handbook
.github/workflows/main.yml      Main-branch build and deployment workflow
```

The cluster repository has no application source code or tests. ECS services
and their application artifacts are managed by separate consuming components.
