
## Executive Summary

Engineering teams scaling their delivery pipelines face three recurring
challenges: inconsistent build environments that cause hard-to-reproduce
failures, fragile state management between pipeline stages, and CI
infrastructure that demands ongoing maintenance overhead.

This document demonstrates how Buildkite addresses all three through a
working pipeline implementation — and maps those technical capabilities
to the business outcomes that matter to engineering leadership.

---

## Business Context

As organisations grow their engineering function, CI/CD infrastructure
becomes a bottleneck rather than an enabler. The symptoms are familiar:

- Builds that pass locally but fail in CI due to environment differences
- Pipeline failures caused by state not transferring reliably between stages
- Engineering time spent maintaining build servers rather than shipping product
- Security and compliance gaps in how build infrastructure is accessed
  and audited
- Sensitive source code and build artifacts leaving the organisation's
  own environment to run on third-party compute

These are not tooling problems. They are business risk problems — they
slow release velocity, introduce compliance exposure, and erode
engineering confidence in the delivery pipeline.

---

## Why Buildkite

The CI/CD market is crowded. The decision to evaluate Buildkite over
alternatives such as GitHub Actions, Jenkins, CircleCI, or GitLab CI
warrants a clear rationale. Three factors differentiate Buildkite
at an architectural level — not a feature level.

**1. The hybrid model is a genuine architectural advantage**

Most CI/CD platforms make a binary choice: fully managed SaaS (GitHub
Actions, CircleCI) where compute runs on the vendor's infrastructure,
or fully self-hosted (Jenkins) where the organisation owns everything
including the control plane.

Buildkite takes a third position. The control plane — scheduling,
orchestration, the web interface, build history — is managed by
Buildkite. The compute — the agents that actually run builds — lives
inside the organisation's own environment. AWS, GCP, Azure, on-premise,
or any combination.

The business implication is significant: source code, build artifacts,
environment variables, and secrets never leave the organisation's
infrastructure. The vendor never touches the data. This is not a
compliance checkbox — it is a structural guarantee.

For organisations in regulated industries (financial services,
healthcare, defence) or those with enterprise customers who audit
their supply chain, this architecture removes an entire category of
risk that fully managed platforms cannot address regardless of their
certifications.

**2. Security is a first-class design decision, not a feature addition**

GitHub Actions and CircleCI were designed for developer convenience
first and hardened over time. Buildkite was designed with the
assumption that build infrastructure is security-critical infrastructure.

The practical differences are visible at every layer:

- Agent communication is outbound-only from the agent to Buildkite.
  No inbound ports need to be opened on agent infrastructure. The
  attack surface is structurally smaller.
- Clustered queues enforce environment isolation at the scheduler
  level — a development build cannot be routed to a production agent
  by misconfiguration.
- Pipeline signing allows organisations to cryptographically verify
  that pipeline steps have not been tampered with between definition
  and execution — directly addressing supply chain attack vectors.
- Secrets never pass through Buildkite's infrastructure. They are
  retrieved by the agent from the organisation's own secrets store
  (AWS Secrets Manager, HashiCorp Vault) at runtime.

**3. Scale without the Jenkins tax**

Jenkins remains the default for organisations that need self-hosted
compute, but it carries a significant operational burden: plugin
management, agent maintenance, Groovy DSL complexity, and a security
model that requires continuous attention.

Buildkite provides self-hosted compute without the Jenkins tax. The
agent is a single lightweight binary. Pipelines are defined in clean
YAML. Plugins are versioned and maintained by Buildkite and the
community. The control plane scales automatically — organisations
running tens of thousands of builds per day do not manage that
infrastructure themselves.

The commercial outcome: engineering teams get self-hosted security
posture at SaaS operational overhead. That combination does not exist
elsewhere in the market at Buildkite's maturity level.

---

## Solution Overview

A three-step pipeline was implemented and validated end-to-end,
demonstrating Buildkite's core capabilities in a working context.

Forked from: https://github.com/mkmrgn/example  
Solution repo: https://github.com/Vaibhavarora08/Buildkite-task/blob/main/.buildkite/SOLUTION.md

The pipeline was connected via GitHub webhook — reflecting how a
customer would experience Buildkite from day one, with builds
triggered automatically on every code push.

**Pipeline flow:**

| Step | What happens | Business capability demonstrated |
|---|---|---|
| Build | Go binary compiled inside an isolated Docker container | Consistent, reproducible builds regardless of agent configuration |
| Review | Pipeline pauses for human input before proceeding | Approval gates for change control and release management workflows |
| Execute | Binary retrieved and run with validated input | Reliable artifact promotion between pipeline stages |

**Outcome:** `Hello, Vaibhav Arora!` printed in the final job log,
confirming end-to-end pipeline execution. Screenshot enclosed.

---

## How Buildkite Capabilities Deliver Business Value

**Isolated build environments**  
Using the Buildkite Docker plugin, the build runs inside a
`golang:1.18.0` container — no language runtime required on the host
agent. Every build runs in an identical environment regardless of which
agent picks it up.

Business impact: eliminates the class of failures caused by environment
drift between agents — a leading cause of unreliable pipelines in
Jenkins-based setups. Reduces debugging time and increases developer
confidence in build results.

**Clean state management between steps**  
Buildkite provides two native mechanisms for passing state between
pipeline steps: an artifact store for files and binaries, and a
metadata store for lightweight key-value data. Each serves a distinct
purpose and neither requires external infrastructure.

Business impact: teams do not need to provision and maintain external
storage to share build outputs between stages. Reduces architectural
complexity and eliminates a common source of pipeline fragility.

**Human approval gates**  
The block step pauses pipeline execution and requires explicit human
sign-off before proceeding — with structured form input captured
directly in the Buildkite interface.

Business impact: maps directly to existing change management and release
approval processes. Compliance-sensitive organisations can enforce
four-eyes principles on deployments without building custom tooling.

**Agent isolation by environment**  
Buildkite's clustered queue model ensures builds run only on agents
explicitly configured for that environment. A development build cannot
be picked up by a production agent.

Business impact: a foundational control for organisations with
environment separation requirements — relevant to SOC2, ISO 27001,
and internal change management policies.

---

## Infrastructure and Security Posture

The agent infrastructure was configured to reflect the security
standards appropriate for a production Buildkite deployment.

**No SSH access — zero open inbound ports**  
Agent access is managed entirely through AWS Systems Manager Session
Manager. Port 22 is closed. There are no SSH key pairs to manage,
rotate, or audit. Every session is logged through AWS CloudTrail,
providing a complete audit trail of who accessed what and when.

**Instance Metadata Service hardened**  
IMDSv2 enforced with a hop limit of 1. This protects against
server-side request forgery attacks — the same vulnerability class
behind several high-profile cloud breaches — and prevents containers
running on the instance from accessing the metadata endpoint.

**Right-sized for purpose**  
t3.micro on-demand instance on Amazon Linux 2023. Kept stopped when
not in use to avoid idle cost. No over-engineering for a validation
deployment — the right infrastructure decision at each stage of maturity.

---

## Observations Worth Noting

Three behaviours surfaced during implementation that are relevant to
customer onboarding conversations.

**Pipeline variable interpolation**  
Buildkite evaluates environment variables at pipeline parse time rather
than shell execution time. Teams migrating from GitHub Actions or Jenkins
will encounter this difference early. Clear onboarding support at this
point prevents unnecessary friction during initial adoption.

**Docker plugin agent availability**  
The Buildkite agent binary must be explicitly mounted into Docker
containers for artifact operations to function. Customers adopting
Docker-based builds will encounter this in their first pipeline.
A guided setup path or pre-configured base image would reduce
time-to-first-success for this segment.

**Block step timeout behaviour**  
Native timeout controls do not apply to block steps — a pipeline
waiting for human approval will remain in that state indefinitely
until actioned or cancelled. For customers with automated release
pipelines and SLA commitments, this requires a workaround via the
Buildkite REST API. Worth surfacing early in customer conversations
to set expectations and demonstrate the available mitigation path.

---

## Path to Production

The implementation above represents a validation deployment. The
table below outlines how each component evolves as a customer moves
toward production scale.

| Area | Validation deployment | Production deployment |
|---|---|---|
| Agent infrastructure | Single persistent EC2, stopped when idle | Ephemeral agents on Spot instances — no idle cost, automatic retry on termination |
| Artifact storage | Buildkite managed store | Customer-owned S3 bucket with KMS encryption — satisfies data residency and encryption key control requirements |
| Secrets management | Buildkite metadata | AWS SSM Parameter Store — IAM-controlled access, encryption at rest, full audit trail |
| Network architecture | Default public subnet | Private subnet with NAT Gateway and VPC endpoints — build traffic contained within customer VPC |
| Pipeline triggers | Manual and webhook | Webhook-driven with branch filtering, PR builds, and environment promotion gates |
| Agent access | SSM Session Manager | SSM Session Manager with session logging to S3 and CloudWatch |

The pipeline definition itself requires no changes as infrastructure
matures. Buildkite's separation of pipeline logic from execution
infrastructure means organisations can adopt incrementally and harden
their environment without disrupting existing workflows — a significant
advantage during enterprise onboarding and a key differentiator from
platforms where infrastructure assumptions are embedded in the pipeline
configuration itself.