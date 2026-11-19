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

## Decision

Adopt a risk-based, zero-trust ACR architecture that makes private networking, identity-based authentication, repository-scoped ABAC authorization, deployment-time signature verification, immutable digest references, and independent recovery the default.

**Network exposure**: deny by default! \
Every ACR instance MUST have `publicNetworkAccess` set to Disabled and MUST be reachable only through an Azure Private Link private endpoint in an approved VNet. \
Private endpoints automatically enable dedicated data endpoints (`myregistry.<region>.data.azurecr.io`); all pull and push paths MUST use the private endpoint, and public *.azurecr.io and *.blob.core.windows.net data paths MUST NOT be reachable from the internet.

**Authentication**: no long-lived credentials! \
Every ACR authentication path MUST use Microsoft Entra ID. Azure-hosted workloads MUST use managed identities; GitHub Actions and Azure DevOps MUST use workload identity federation (OIDC). Service principals with client secrets are prohibited for new integrations. \
The ACR admin account MUST be disabled.

**Authorization**: repository ABAC, exact roles! \
Registries MUST be created or migrated with roleAssignmentMode=AbacRepositoryPermissions. The following roles are the ONLY data-plane roles permitted, and they MUST be assigned at the narrowest repository scope possible using ABAC conditions:
* Container Registry Repository Reader for runtime pull consumers;
* Container Registry Repository Writer for CI/CD publishers;
* Container Registry Repository Contributor only for break-glass or repository administration, never for normal workloads. \
Legacy AcrPull, AcrPush, and AcrDelete roles MUST NOT be assigned to new identities; any existing assignment MUST be migrated before the ABAC cutover, not after.

**Data path**: who may call what! \
Pull consumers (AKS kubelets, Container Apps, deployment controllers) MUST have Container Registry Repository Reader scoped to the specific repository or repository prefix they consume, and MUST NOT have write or delete. \
Publishers (CI/CD identities) MUST have Container Registry Repository Writer scoped to the repositories they own, and MUST NOT have delete or registry control-plane rights. \
Deletion MUST be performed only by a separate, non-pipeline identity holding Container Registry Repository Contributor on the target repository, with approval. \

**Transport security**. All communication with registry MUST use TLS 1.2 or higher; plaintext HTTP registry access MUST be blocked by client configuration and network policy. \
Connections to Key Vault and signing services MUST use TLS and managed identity or federated credentials; no signing or key-access secret MAY traverse an unencrypted channel.

**Encryption at rest**. Microsoft-managed encryption at rest is the default and MUST remain enabled! \
The architecture SHOULD preserve the ability to adopt customer-managed keys (CMK) for relevant cases where it is required.

**Monitoring and audit**. Every registry MUST route ContainerRegistryLoginEvents and ContainerRegistryRepositoryEvents logs, plus Azure Activity Logs for control-plane changes, to a centralized Log Analytics workspace (Azure monitor's log analystics). \
The following events MUST generate a security alert with a named owner: 
* authentication failures from unexpected identities or networks
* push/delete activity from identities not in the approved publisher/deleter set
* public-network access attempts when publicNetworkAccess=Disabled
* changes to publicNetworkAccess, roleAssignmentMode, network rules, or diagnostic settings
* signature-verification denials at admission; and certificate or key expiration within 30 days.

Log retention MUST meet the organization’s incident-response requirement and MUST NOT rely on ACR’s default retention alone.


**Artifact identification and immutability**. Production deployments MUST reference images by digest (<repository>@sha256:<digest>), not by mutable tag. \
Production repositories MUST be locked against deletion and tag mutation as well as policy freezes, except through the approved deleter identity. \
Tags MAY be used for human readability but MUST NOT be the deployment reference. \
Content delete lock with `CanNotDelete` to prevent accidental delete
ACR soft delete policy MUST be turned on for where unintended accidental delete occurs.


**Vulnerability and provenance gating**. Microsoft Defender for Cloud container image assessment MUST be enabled on every registry. Build pipelines MUST produce an SBOM and provenance attestation bound to the image digest. Production promotion MUST be blocked when an image violates the organization’s critical-vulnerability threshold, unless a time-bound exception with a named owner is recorded. (Driver 3)

**Recovery**. redundant Container registries MUST be set up to act as a vault where an automated pipeline could backup Critical production images, their signatures, SBOMs, and provenance to an air-gapped registry  outside the main one's blast radius, along side restore tests that MUST be performed frequently for integrity and availability verification. \
Geo-replication MAY be used for regional availability but MUST NOT be treated as backup. \

## Consequences

### Gains
Addressed most of the security risks and satisified Drivers:
* no secret-based ACR path remains for new integrations, reducing credential-leak blast radius.
* a public endpoint scan from outside the approved VNet returns no reachable registry endpoint.
* a compromised registry push alone is insufficient to deploy a malicious image, because admission control rejects unsigned or untrusted-signer artifacts.
* a compromised pull identity cannot push, and a compromised push identity cannot delete production repositories.
* every mutation is tied to an Entra principal and a source network, and alerts are based on events that are out of pattern.
* a deleted or ransomware-affected registry does not eliminate the last trusted copy of production artifacts.

### trade-offs

* More configuration complexity: private DNS, VNet peering, access per identity...
* ABAC migration breaks existing identities:  AcrPush / AcrPull assignments stop working once roleAssignmentMode=AbacRepositoryPermissions, and so will the dependant Pipelines.
* Signing and admission policy add build-time and dependencies: certificate lifecycle, Ratify installation, and policy maintenance...
* Independent recovery exports add storage cost and a recurring restore-test obligation.


### Validation 

*
