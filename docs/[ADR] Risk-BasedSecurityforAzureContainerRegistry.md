# ADR-003: Risk-Based Security for Azure Container Registry

## Context, Problem statement and methodology
To be efficient and deployed at scale, agents as well as their related workloads rely heavily on containerization and OCI artifacts; specifically container images, stored and managed on the Azure Container Registry (ACR) thanks to its range of features. And assuming ACR is secure by default is the first issue.

As the shared responsibility model of Microsoft leaves quite a substantial attack surface unaddressed, and compromising the ACR has impacts on the entire chain of operations

So protecting it means keeping these artifacts safe throughout the different phases of their lifecycle, which becomes a crucial part of securing our agents.

How can we secure ACR to protect its entrusted artifacts throughout their lifecycle, without loosing on the flexibility its features offer?

The adopted methodology of producing this architecture is based on a risk management methodology:

* identifying the operational procedures (what ACR does?) and assets involved in those procedures (ACR does it with what/to what?) as well as classifying them based on their security properties,
* then the risks related to those aspects (what could go wrong ?)
* and following the SWIFT (Structured What-If Technique) to uncover more potential risks,
* and finally addressing the identified risks with appropriate security controls in the architecture, which follows a zero-trust approach.
  Note; the risks related to Microsoft’s part of the shared responsibility model were omitted.

