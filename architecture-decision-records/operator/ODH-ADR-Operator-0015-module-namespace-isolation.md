# Open Data Hub - Per-Module Network Isolation by Namespace

|                |                                  |
| -------------- |----------------------------------|
| Date           | 2026-09-08                       |
| Scope          | Open Data Hub Operator           |
| Status         | Approved                         |
| Authors        | [Luca Burgazzoli](@burgazzoli)   |
| Supersedes     | N/A                              |
| Superseded by: | N/A                              |
| Tickets        | [RHOAIENG-90465](https://redhat.atlassian.net/browse/RHOAIENG-90465) |
| Other docs:    | [Module Onboarding Architecture](ODH-ADR-Operator-0012-module-onboarding.md) |

## What

This ADR extends [ODH-ADR-Operator-0012](ODH-ADR-Operator-0012-module-onboarding.md) with a network-isolation requirement implemented through namespaces. 
It addresses network ingress only; ADR-0012 remains authoritative for the broader module-controller architecture and responsibility boundaries.

Each module operator will run in a dedicated system namespace. 
The module operator is always installed in that namespace, while the module team decides whether its operands run there or in one or more additional namespaces.

Every namespace used by a module is an ingress-isolation boundary. ODH creates a default-deny ingress policy for the module operator namespace, and the module ships explicit ingress policies for the traffic that its operator and operands require. 
This does not make the namespace a general-purpose resource, RBAC, or lifecycle isolation boundary.

Additional namespaces are reserved exclusively for resources belonging to the module or its operands. 
They are not user workload namespaces.

## Why

The shared `redhat-ods-applications` namespace makes it difficult to establish independent network boundaries between modules. In particular:

* Network policies are shared across unrelated modules.
* A policy change for one module can unintentionally affect another module.
* Module teams cannot independently define the ingress boundary of their operator and operands.
* Namespace-specific integrations and cross-module dependencies become tightly coupled to a legacy namespace name.

Per-module namespaces provide independent network boundaries and allow each module team to define the ingress required by its operator and operands. This follows the general OpenShift pattern of applying explicit network policies to isolated workloads.

## Goals

* Place each module operator in a dedicated system namespace for network-policy purposes.
* Give module teams ownership of module and operand ingress policy design.
* Establish a default-deny ingress boundary for every namespace used by a module.
* Require explicit ingress policies for module operators and operands.
* Keep module network-policy namespaces separate from namespaces containing unrelated user workloads.
* Ensure that each module operator has an individual system namespace.

## Non-Goals

* Defining implementation details such as registration/configuration schemas, namespace validation, or specific NetworkPolicy rules.
* Defining egress, metrics, observability, or other adjacent policy concerns.
* Defining workload/data migration, rollout sequencing, or project-management gates.
* Defining general RBAC, resource ownership, or namespace lifecycle semantics; operand namespaces must remain dedicated to module resources and operands.

## How

### Module operator namespace

The ODH Operator creates and initializes a dedicated system namespace for each module operator so that the operator's ingress policy can be isolated. ODH derives the module's default namespace from the module's preferred namespace in registration metadata, or uses `redhat-ai-${module-name}-system` when no preferred namespace is declared. A user may override this default through ODH operator configuration or environment variables. Namespace selection is not defined in the Platform or PlatformModule CRDs because ODH must resolve the namespace before configuring its cache and watch scope.

The ODH Operator enforces one module operator per resolved system namespace. If the user override or derived default namespace is already associated with another module, ODH detects the collision before bootstrap, reports the provisioning failure on the available parent object—the Platform CR or PlatformModule CR—and emits a Kubernetes event. ODH does not deploy the module, adopt the conflicting namespace, or modify its policies.

The ODH Operator establishes and continuously reconciles the default-deny ingress baseline, applies the module deployment package—including at least one module-provided `NetworkPolicy`—and then installs the module operator. ODH checks that at least one module `NetworkPolicy` was deployed as part of the module deployment; the checking mechanism is an implementation detail. During this bootstrap phase, ODH is responsible for applying the module's bootstrap policy resources so that the module can start. Once the module controller is successfully deployed and ready, ongoing reconciliation of module-specific ingress policies shifts to the module operator. The default-deny policy remains mandatory: if it is deleted or modified, the ODH Operator recreates or restores it. If the module deployment does not result in at least one module `NetworkPolicy`, ODH does not consider the module ready, reports the provisioning failure on the available parent object—the Platform CR or PlatformModule CR—and emits Kubernetes events. The ODH Operator does not interpret module-specific policy rules or manage the module's operands.

### Operand placement

The module operator is responsible for its operands and may deploy them:

* in the module operator's system namespace; or
* in one or more additional, module-owned operand namespaces.

For example, a KServe module operator may run in namespace `foo`, while its KServe controller and ODH-specific controller operands run in namespace `bar`. In this case, `bar` is a child namespace owned by the parent module, and it is reserved for those module operands.

When a module uses additional operand namespaces, the parent module operator owns the network-policy setup for those namespaces and continuously reconciles the same ingress-isolation model in each namespace. This includes the default-deny ingress policy and the module-owned allow policies required by the operands.

A module must not create, adopt, or impose its ingress-isolation policy on a namespace containing unrelated user workloads. Namespaces for module operands must be dedicated to, and logically owned by, the parent module.

### Network policy ownership

The module team owns ingress policies for:

* the module operator;
* all operands;
* operator-to-operand and operand-to-operand communication;
* required platform and dependency traffic; and
* other ingress required by the module's declared integrations.

The ODH Operator owns and continuously reconciles the namespace-level baseline for the module operator namespace. The module operator must not delete or replace this baseline; if it does, the ODH Operator restores it. ODH owns the initial application of module-shipped bootstrap policies only until the module controller is ready. Responsibility for ongoing reconciliation of those module-specific policies, plus all operand policies, then belongs to the module operator. These policies remain additive to the ODH baseline.

## Alternatives

### Keep all modules in `redhat-ods-applications`

This preserves the current deployment model but retains the shared network boundaries that motivate this decision.

### Make the ODH Operator manage all module and operand policies

This centralizes policy management but couples the ODH Operator to module-specific traffic flows and conflicts with the independent module-controller architecture.

### Require operands to use the operator namespace

This would simplify policy placement but would prevent modules from choosing an operand topology appropriate to their architecture. Operand placement remains a module responsibility; every module-used namespace still receives the same ingress-policy treatment.

## Security and Privacy Considerations

Default-deny ingress policies reduce unintended network access between module workloads. Explicit module-owned policies make required communication auditable and keep policy decisions with the team that understands the workload.

The policy model defined here intentionally does not restrict egress. Modules and platform security controls may define egress behavior separately.

## Risks

* Modules that omit required ingress policies may fail to start or become degraded under the default-deny baseline.
* Additional network-policy namespaces increase the number of policies that module teams must operate and test.

## Stakeholder Impacts

| Group                  | Key Contacts | Date       | Impacted? |
| ---------------------- | ------------ | ---------- | --------- |
| ODH Platform Team      |              | 2026/09/08 | YES       |
| Module Teams            |              | 2026/09/08 | YES       |
| Security/Productivity   |              | 2026/09/08 | YES       |

## References

* [RHOAIENG-90465: Network Policies for RHOAI Operands](https://redhat.atlassian.net/browse/RHOAIENG-90465)
* [ODH-ADR-Operator-0012: Module Onboarding Architecture](ODH-ADR-Operator-0012-module-onboarding.md)
* [Module Onboarding Guide](design/module-onboarding-guide.md)
* [OpenShift Network Policy documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/network_security/network-policy)
