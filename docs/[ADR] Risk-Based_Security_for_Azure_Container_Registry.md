# ADR-003: Risk-Based Security for Azure Container Registry

## Context, Problem statement and methodology

To be efficient and deployed at scale, agents as well as their related workloads rely heavily on containerization and OCI artifacts; specifically container images, stored and managed on the Azure Container Registry (ACR) thanks to its range of features. And assuming ACR is secure by default is the first issue.

As the shared responsibility model of Microsoft leaves quite a substantial attack surface unaddressed, and compromising the ACR has impacts on the entire chain of operations

So protecting it means keeping these artifacts safe throughout the different phases of their lifecycle, which becomes a crucial part of securing our agents.

How can we secure ACR to protect its entrusted artifacts throughout their lifecycle, without loosing on the flexibility its features offer?

The adopted **methodology** of producing this architecture is based on a **risk management** methodology:

* identifying the operational procedures (what ACR does?) and assets involved in those procedures (ACR does it with what/to what?) as well as classifying them based on their security properties,
* then the authentication, authorization and availability risks related to those aspects (what could go wrong ?)
* and following the SWIFT (Structured What-If Technique) to uncover more potential risks,
* and finally addressing the identified risks with appropriate security controls in the architecture, which follows a zero-trust approach. \
Note; the risks related to Microsoft’s part of the shared responsibility model were omitted.

## Decision Drivers
* We need to follow zero trust architecture with granular, least privelege access control. \
* No ACR operation may depend on a secret, password, or the admin account, every authentication must be identity-based and short-lived. \
* Artifact protection needs to be applied throughout all phases of its lifecycle. \
* Recovery from destructive events is independent of geo-replication.




## Options Considered

### Network exposure

1. Private Link + dedicated data endpoints + public access disabled: \
Private link allows for direct private tunnels between ACR and internal Azure private VNets and services and between ACR and private on-prem networks. \
Dedicated data endpoints allow scoped firewall rules on `*.data.azurecr.io`. \
Eliminiates the public data-plane attack surface.

**Chosen** because it is the option that shrinks the attack surface the most.

2. Public access with IP allowlist \
ACR firewall allows explicit CIDRs and blocks the rest; public endpoint remains enabled. \
This is simpler to configure

**Rejected** because it leaves the registry publicly resolvable and expands the attack surface. \
Reconsider if a service cannot be placed in a VNet.

3. Trusted-services-only exception \
Public endpoint enabled but limited to Azure trusted services.

**Deferred** because it is an exception path, not a primary control. Reconsider if a required Azure service cannot reach the private endpoint and no managed-identity workaround exists. |

### Authentication

1. Microsoft Entra ID with managed identities / workload identity federation (OIDC) \
Azure-hosted workloads use managed identity; GitHub Actions and Azure DevOps use federated credentials. \ 
No stored secrets; short-lived tokens; auditable principal identity; satisfies Driver 1 and 2.

**Chosen** because it eliminates long-lived credentials on both Azure-native and external CI/CD paths, reducing the great risk that comes with them, and allows for more control.

2. Service principal with client secret \
App registration + secret stored in pipeline or Key Vault. \
Universally supported and simple to set up.

**Rejected** because secrets are long-lived, commonly leak into logs and production code; violates Driver 2. \
Reconsider only for a legacy integration that cannot use federated approach, but with a fixed rotation time and explicit owner for control.

3. ACR admin account \
Built-in registry-wide username/password. \
Immediate bootstrap access.

**Rejected** because it is a shared credential leaves no accountability ; violates Driver 2. It is disabled by default in this decision.

### Authorization

1. ABAC repository permissions with Container Registry Repository Reader / Writer / Contributor \
Role assignments scoped to specific repositories via ABAC conditions. \
Least-privilege at repository granularity; separates pull, push, and delete...

**Chosen** because it has the most granular access control and satisfies Driver 1

2. Legacy RBAC at registry scope \
Registry-wide data-plane roles. Simpler role model; familiar to existing pipelines.

**Rejected** because, for example, any 'push' role holder can push to every repository, reducing granularity thus violating Driver 1.



### Artifact integrity
1. Notation signing + Ratify/Azure Policy admission verification \
Images signed at build with Notation; signatures verified at deployment via Ratify and Azure Policy deny effect. \
supports Key Vault or Trusted Signing certificate management;

**Chosen** because signing alone does not prevent an unsigned image from being deployed; verification at admission is the only falsifiable enforcement of Driver 3.

2. Docker Content Trust (DCT) \
Legacy Notary v1 signing. \
Existing tooling in some pipelines.

**Rejected** because DCT cannot be enabled on new registries after May 31, 2026, and will be removed March 31, 2028; it is not a viable forward-looking architecture.

3. Tag-based immutability only \
Reliance on tags plus repository locks. \
Low tooling overhead.

**Rejected** because tags are mutable references and do not prove authenticity; it cannot satisfy Driver 3 without signature verification.

### Encryption at rest

1. Microsoft-managed keys (default) \
Platform-managed encryption at rest. \
No key-lifecycle burden; available on all SKUs.

**Chosen** as the default because it reduces operational dependencies and is sufficient unless a regulatory or contractual key-control requirement exists; it does not weaken Drivers.

2. Customer-managed keys (CMK) \
Key Vault key encrypts registry content; Premium SKU only; must be configured at registry creation. \
Greater key-control; supports HSM-backed keys.

**Deferred** because CMK adds key-availability, rotation, and recovery dependencies and is only configurable at creation on Premium SKU. \
Reconsider if an external driver is added (regulation, customer or organizational  policy..).
