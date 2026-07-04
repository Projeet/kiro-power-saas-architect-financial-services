# SaaS Architecture Artifacts

## Overview

As a SaaS architect, produce concrete deliverables — not just advice. When the conversation reaches a decision point, offer to generate the relevant artifact as a markdown file in the workspace.

Default save location: `docs/saas-architecture/` in the workspace root.

This file covers **core SaaS artifacts** that apply to any multi-tenant SaaS on AWS. For finance-specific artifacts (PCI Scoping Matrix, SOX ITGC Matrix, AML Risk Assessment, Financial Data Flow Map, Adverse Action design), load `artifacts-finance.md`.

---

## Artifact Relationships

- **HLD ↔ Everything:** The HLD is the top-level synthesis. It references detail artifacts, never duplicates them.
- **Tenant Isolation Matrix ↔ Data Partitioning Map:** Isolation Matrix has high-level storage; Data Partitioning Map goes deeper.
- **Tenant Isolation Matrix ↔ Tiering Matrix:** Isolation Matrix shows service-level tenancy per tier; Tiering Matrix covers business dimensions.
- **Cost Attribution ↔ Tiering Matrix:** Metering dimensions should align with billing dimensions.

**General rule:** When generating an artifact and a related one exists, read it first and reference it.

---

## Artifact 1: High-Level Design (HLD)

The top-level synthesis artifact. Answers "what does the whole system look like?"

### Template

```markdown
# High-Level Design

**Product:** {product name}
**Date:** {date}
**Segment:** {banking & neobanking | lending & credit | capital markets & trading | insurance & wealth management}
**Regulatory Scope:** {PCI-DSS, GLBA, SOX, BSA/AML, FCRA, FINRA, SEC — as applicable}

## 1. Executive Summary
{4-6 sentences: what it does, who uses it, key architecture choices, compliance posture.}

## 2. System Context
### Users and Personas
| Persona | Authentication | Primary Use Cases |
|---|---|---|
| {Institution Admin} | {SAML SSO + MFA} | {Configuration, user management} |
| {Operations User} | {Cognito + MFA} | {Payment processing, loan operations} |
| {End Consumer} | {Cognito + SCA} | {Account access, transactions} |

### External Systems
| System | Protocol | Direction |
|---|---|---|
| {Card Processor} | {REST/ISO 8583} | {Bidirectional} |
| {Credit Bureau} | {REST} | {Outbound pull} |
| {SWIFT Network} | {ISO 20022} | {Bidirectional} |

## 3. Logical Architecture
| Component | Responsibility | Tenancy Model |
|---|---|---|
| Control Plane | Tenant mgmt, identity, billing | Shared |
| {Payment Service} | {description} | {Bridge} |
| {Ledger Service} | {description} | {Bridge/Silo} |

## 4. Physical Architecture on AWS
{Mermaid diagram — deployed system with tenancy annotations.}

## 5. Security and Compliance Posture
- **Tenant isolation:** {philosophy + link to Isolation Matrix}
- **PCI-DSS:** {CDE scope + link to PCI Scoping Matrix}
- **Encryption:** {per-tenant CMK strategy}
- **Audit:** {CloudTrail + application audit + retention}

## 6. Key Architectural Decisions
| # | Decision | Status | Link |
|---|---|---|---|
| ADR-001 | {Tenancy model} | Accepted | [ADR-001](./adr/ADR-001.md) |

## 7. Open Questions and Risks
| # | Question/Risk | Impact | Owner |
|---|---|---|---|
```

### Readiness Requires
- Segment identified
- Regulatory scope clear
- Personas identified
- Tenancy model decided per major component
- AWS services selected
- Account structure decided

---

## Artifact 2: Tenant Isolation Matrix

Maps every service to its tenancy model, isolation mechanism, and data strategy.

### Template

```markdown
# Tenant Isolation Matrix

**Product:** {product name}
**Date:** {date}

## Service-Level Tenancy

| Service | Tenancy | Compute | Storage | Isolation Mechanism | Rationale |
|---|---|---|---|---|---|
| {Payment Service} | Bridge | Lambda (shared) | DynamoDB (per-tenant PK) | IAM LeadingKeys + per-tenant CMK | PCI CDE scope minimization |
| {Ledger} | Silo | ECS (per-tenant) | Aurora (per-tenant schema) | Schema isolation + dedicated CMK | SOX audit requirement |

## Tier Mapping
| Tier | Pool Services | Bridge Services | Silo Services |
|---|---|---|---|
| Starter | {list} | {list} | — |
| Enterprise | {list} | {list} | {list} |
```

---

## Artifact 3: Architecture Decision Record (ADR)

### Template

```markdown
# ADR-{number}: {Title}

**Date:** {date}
**Status:** Proposed | Accepted | Superseded

## Context
{What situation? What constraints?}

## Decision
{What did we decide?}

## Rationale
{Why this over alternatives?}

## Alternatives Considered
### Option A: {name}
- Pros / Cons / Why rejected

## Consequences
### Positive
### Negative (accepted trade-offs)
### Risks
```

---

## Artifact 4: SaaS Lens Review Report

### Template

```markdown
# SaaS Lens + FSI Lens Architecture Review

**Product:** {product name}
**Date:** {date}

## Executive Summary
{Maturity level, top risks, strengths.}

## Findings

| # | Pillar | Finding | Risk Level | Effort | Priority |
|---|---|---|---|---|---|
| 1 | Security | {finding} | Critical | Medium | P1 |

## Detailed Findings
### Finding {n}: {title}
**Current State:** {observed}
**Risk:** {what could go wrong}
**Recommendation:** {what to do}

## Roadmap
### Phase 1: Critical (Weeks 1-4)
### Phase 2: High (Weeks 5-12)
### Phase 3: Improvements (Weeks 13+)
```

---

## Artifact 5: Onboarding Flow

### Template

```markdown
# Tenant Onboarding Flow

**Product:** {product name}

## Steps

| Step | Action | Service | Failure Handling |
|---|---|---|---|
| 1 | KYC/KYB verification | KYC provider | Block until verified |
| 2 | OFAC screening | Screening service | Block if match |
| 3 | Create tenant record | Control Plane | Rollback |
| 4 | Provision identity | Cognito | Rollback: delete pool/user |
| 5 | Provision storage | DynamoDB/RDS/S3 | Rollback: delete resources |
| 6 | Provision KMS key | KMS | Rollback: schedule deletion |
| 7 | Configure billing | Billing service | Rollback |
| 8 | Activate | Control Plane | — |

## Per-Tier Variations
| Step | Starter (Pool) | Enterprise (Silo) |
|---|---|---|
| Provision storage | Shared (partition) | Dedicated account/VPC |
| Time | ~30s after KYC | ~10 min |
```

---

## Artifact 6: Data Partitioning Map

### Template

```markdown
# Data Partitioning Map

**Product:** {product name}

## Storage by Service

### {Service Name}
| Aspect | Decision |
|---|---|
| Storage service | {DynamoDB / Aurora / S3} |
| Partitioning model | {Pool / Bridge / Silo} |
| Key design | {PK: TENANT#id, SK: TXN#timestamp} |
| Isolation | {IAM LeadingKeys / RLS / Separate schema} |
| Encryption | {Per-tenant CMK / Shared CMK} |
| Backup | {PITR / Snapshots / Cross-region} |
| Right-to-erasure | {Cryptographic erasure / Schema drop} |
```

---

## Artifact 7: Tiering Matrix

### Template

```markdown
# Tiering Matrix

**Product:** {product name}

| Dimension | Starter | Professional | Enterprise |
|---|---|---|---|
| Price | ${X}/mo | ${Y}/mo | Custom |
| Tenancy | Pool | Bridge | Silo |
| Transaction volume | {limit} | {limit} | Unlimited |
| API rate limit | {X} req/s | {Y} req/s | Custom |
| Support | Email | Priority | Dedicated |
| Compliance | Shared SOC 2 | + PCI AOC | + Dedicated QSA |
| SLA | 99.9% | 99.95% | 99.99% |
```

---

## Artifact 8: Cost Attribution Strategy

### Template

```markdown
# Cost Attribution Strategy

**Product:** {product name}

## Metering Dimensions
| Dimension | How Measured | Billing Impact |
|---|---|---|
| API calls | API Gateway usage plan metrics | Per-call pricing |
| Transaction volume | DynamoDB write events counted | Per-transaction fee |
| Storage | S3/DynamoDB per-tenant size | Included up to limit |
| Compute | Lambda duration per tenant | Included in tier |

## Cost Allocation
| AWS Service | Attribution Method |
|---|---|
| Lambda | Per-invocation tenant tag |
| DynamoDB | Per-table (silo) or item-count (pool) |
| S3 | Per-prefix or per-bucket (silo) |
| KMS | Per-key (per-tenant CMK = per-tenant cost) |
```

---

## Artifact 9: Low-Level Design (LLD)

See `artifacts-lld.md` for the full LLD template. LLDs are per-component detail documents covering: internal structure, API contracts, data model, algorithms, sequence diagrams, tenant isolation implementation. Always generate the HLD before any LLD.

---

## Never Generate with Gaps

If information is missing, ask for it. Never use "TBD" placeholders. Every field must contain real, specific content.

## Always Propose Next Step

After generating any artifact, suggest the next logical artifact or action.
