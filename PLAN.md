# Kiro Power for Financial Services on AWS — Build Plan

**Power Name:** `saas-architect-for-finance-aws`
**Author:** TBD
**Target Completion:** 4 Weeks
**Reference:** Modeled after the SaaS Architect for Healthcare on AWS power structure

---

## Overview

This plan outlines the 4-week roadmap to build a Kiro Power that turns Kiro into an AWS
financial services SaaS architecture advisor. The power will cover multi-tenant
fintech/banking SaaS patterns, financial data protection (PCI-DSS, GLBA, SOX), open
banking interoperability (FDX, ISO 20022, PSD2), fraud & AML patterns, and AI/ML model
risk — grounded in AWS documentation, the Well-Architected Financial Services Industry
Lens, and official regulatory references.

---

## Deliverables Summary

| Artifact | Count |
|---|---|
| `POWER.md` — main power definition file | 1 |
| Core SaaS steering files (tenancy, isolation, data, identity, observability, etc.) | 7 |
| Finance-specific compliance & domain steering files | 14 |
| **Total steering files** | **21** |

---

## Finance Domain Segments

The power will cover four financial services segments, analogous to the healthcare power's
four segments:

| Segment | Description |
|---|---|
| **Banking & Neobanking** | Core banking SaaS, deposits, payments, cards, lending |
| **Lending & Credit** | Loan origination, servicing, collections, BNPL, credit decisioning |
| **Capital Markets & Trading** | Brokerage platforms, trading, market data, post-trade |
| **Insurance & Wealth Management** | InsurTech SaaS, robo-advisory, portfolio management |

---

## Key Regulatory References

All steering files will be grounded in these official public sources:

| Regulation | Authority | Link |
|---|---|---|
| GLBA Safeguards Rule | FTC | https://www.ftc.gov/business-guidance/resources/ftc-safeguards-rule-what-your-business-needs-know |
| GLBA Privacy Rule | FTC | https://www.ftc.gov/business-guidance/resources/how-comply-privacy-consumer-financial-information-rule-gramm-leach-bliley-act |
| FCRA | FTC | https://www.ftc.gov/legal-library/browse/statutes/fair-credit-reporting-act |
| CFPB Section 1033 (Open Banking) | CFPB | https://www.consumerfinance.gov/personal-financial-data-rights/ |
| Sarbanes-Oxley Act (SOX) | govinfo.gov | https://www.govinfo.gov/content/pkg/BILLS-107hr3763enr/html/BILLS-107hr3763enr.htm |
| SEC Rule 17a-4 (Record Retention) | SEC | https://sec.gov/investment/amendments-electronic-recordkeeping-requirements-broker-dealers |
| Dodd-Frank Act | CFTC | https://www.cftc.gov/LawRegulation/DoddFrankAct/index.htm |
| FINRA Rules | FINRA | https://www.finra.org/rules-guidance/rulebooks/finra-rules/ |
| Bank Secrecy Act / AML | FinCEN | https://fincen.gov/resources/statutes-regulations/bank-secrecy-act |
| SR 26-2 (Model Risk — 2026 revision) | Federal Reserve | https://www.federalreserve.gov/supervisionreg/srletters/SR2602.htm |
| OCC Third-Party Risk Guidance | OCC | https://www.occ.gov/news-issuances/bulletins/2023/bulletin-2023-17.html |
| FFIEC IT Handbook | FFIEC | https://ithandbook.ffiec.gov/ |
| PSD2 Directive | EBA / EUR-Lex | https://www.eba.europa.eu/regulation-and-policy/single-rulebook/interactive-single-rulebook/14575 |
| MiFID II / MiFIR | ESMA | https://www.esma.europa.eu/policy-rules/mifid-ii-and-mifir/ |
| ISO 20022 | iso20022.org | https://www.iso20022.org/iso-20022-standard |
| FDX API Standard | FDX | https://financialdataexchange.org/ |
| AWS FSI Well-Architected Lens | AWS | https://docs.aws.amazon.com/wellarchitected/latest/financial-services-industry-lens/financial-services-industry-lens.html |
| AWS FSI Compliance Center (US) | AWS | https://aws.amazon.com/financial-services/security-compliance/compliance-center/us/ |

---

## Week 1: Foundation & Compliance Core

**Goal:** Define the power identity and build the two most critical steering files —
compliance foundations and financial data handling. These are the bedrock that every other
steering file references.

**Days 1–2: Power identity & POWER.md skeleton**

- [ ] Create `POWER.md` with:
  - Front-matter: name, displayName, description, keywords, author
  - Overview paragraph (what the power does and does not do)
  - Onboarding section with example journeys for each of the 4 segments
  - Agent behavior guidelines: discovery flow (Step 1 segment, Step 2 compliance, Step 3 tech)
  - Steering file routing table (stub — complete as files are written)
  - Response style rules (opinionated, propose next step, short first turn)
  - Artifact generation strategy
  - Key AWS references table
  - Quick start agent reference table

**Days 3–4: `financial-compliance-foundations.md`**

Finance equivalent of `healthcare-compliance-foundations.md`. The most important file —
every architecture decision is constrained by regulation.

Sections to write:
- PCI-DSS v4.0 — 12 requirements mapped to AWS services (CDE scoping, network
  segmentation, encryption, access control, logging, testing)
- GLBA Safeguards Rule — 9 elements of an information security program mapped to AWS
- SOX Section 404 — IT general controls (ITGC) mapped to AWS (access control, change
  management, availability, audit logging)
- BSA/AML — SAR/CTR filing architecture, transaction monitoring pipeline on AWS
- FCRA — Credit data handling, adverse action, data accuracy requirements
- FFIEC guidance — Cloud risk management, IT examination expectations
- OCC Third-Party Risk — SaaS vendor risk management requirements
- State laws — NY DFS Part 500, CA CCPA/CPRA for financial data
- Shared responsibility model for finance (what AWS covers, what you cover)
- Common mistakes
- Discovery questions for this domain
- Content freshness table with verify-at links

**Day 5: `pii-financial-data-handling.md`**

Finance equivalent of `phi-data-handling.md`. Covers how to protect financial PII, PAN
data, and sensitive financial records.

Sections to write:
- Financial data classification (PAN, CVV, SSN, account numbers, transaction data,
  credit scores — and which regulations govern each type)
- PCI-DSS CDE scoping — what counts as cardholder data environment, how to minimize scope
- Tokenization patterns on AWS (Amazon Payment Cryptography, third-party vaults)
- PAN masking and format-preserving encryption
- KMS key strategy for financial data (per-tenant CMKs, key hierarchy)
- Macie for financial PII detection in S3
- De-identification for analytics and ML pipelines (k-anonymity, differential privacy)
- Data residency requirements (state laws, EU/GDPR Article 9 for financial data)
- AWS services for financial data protection and their compliance posture
- Discovery questions
- References

**Week 1 Exit Criteria:**
- [ ] `POWER.md` skeleton complete (routing table stubs OK)
- [ ] `financial-compliance-foundations.md` complete
- [ ] `pii-financial-data-handling.md` complete

---

## Week 2: Core SaaS + Payments & Open Banking

**Goal:** Build the core SaaS steering files adapted for finance, plus the two most
impactful domain-specific files — payments/ledger patterns and open banking interoperability.
These are the files that differentiate a generic SaaS power from a finance SaaS power.

**Days 1–2: Core SaaS steering files (finance-adapted)**

These files are structurally similar to the healthcare power but with finance-specific
defaults, examples, and compliance references replacing healthcare ones.

- [ ] `tenancy-models.md` — Silo/pool/bridge for fintech SaaS. Finance-specific defaults:
  - Bridge as default for any service handling financial PII or transaction data
  - Silo for regulated entities (banks, broker-dealers) with strict data residency
  - Covered entity equivalents: bank tenant vs. fintech-as-a-service tenant vs. enterprise
  - Finance tension patterns: "We need PCI compliance but serve 10,000 merchants"

- [ ] `tenant-isolation.md` — IAM-based isolation for financial data. Finance additions:
  - CDE isolation: cardholder data environment must be network-segmented even in SaaS
  - IAM dynamic policies for per-tenant transaction data in DynamoDB
  - PCI-DSS Requirement 7 (access control) and Requirement 8 (authentication) mapped to AWS isolation
  - Testing isolation for PCI scope reduction

- [ ] `identity-and-onboarding.md` — Finance identity personas and onboarding. Finance-specific:
  - KYC/KYB (Know Your Customer / Know Your Business) onboarding flow
  - Strong Customer Authentication (SCA) per PSD2 and FFIEC guidance
  - OAuth 2.0 / OIDC for open banking (FDX, PSD2 TPP auth)
  - MFA requirements (FFIEC guidance, PCI-DSS Requirement 8)
  - Cognito patterns for bank tenant SSO (SAML from enterprise identity providers)
  - JWT claims for finance: tenant_id, account_type, regulatory_tier, kyc_status

**Day 3: `payments-and-ledger.md`**

One of the most finance-specific files. No healthcare equivalent.

Sections to write:
- Payment processing architecture on AWS (API Gateway → Lambda → payment processor)
- Double-entry ledger patterns (event sourcing, CQRS, idempotency keys)
- Payment rails: ACH/NACHA, card networks (Visa/Mastercard), RTP (real-time payments),
  wire transfers (Fedwire/CHIPS)
- Tokenization: replacing PAN with tokens, Amazon Payment Cryptography service
- Idempotency: exactly-once payment processing, deduplication patterns with DynamoDB
- Settlement and reconciliation architecture
- Multi-tenant payment isolation: one tenant's payment failure must not affect others
- PCI-DSS scope minimization for SaaS payment processing
- AWS Payment Cryptography — key management for payment HSM workloads
- Chargeback and dispute management patterns
- Discovery questions
- References

**Day 4: `open-banking-and-interop.md`**

Finance equivalent of `fhir-and-interop.md`. Covers the standards and patterns for
financial data interoperability.

Sections to write:
- FDX API (US/Canada open banking standard) — data categories, consent model, OAuth scopes
- CFPB Section 1033 — what data must be available, compliance deadlines (April 2026 start)
- PSD2 / Open Banking (EU/UK) — AIS, PIS, SCA requirements, TPP authentication
- ISO 20022 — payment message formats (pacs, pain, camt), migration from SWIFT MT
- SWIFT — correspondent banking, gpi for cross-border payments
- Screen-scraping vs. API: why Section 1033 and PSD2 are ending screen-scraping
- Multi-tenant open banking architecture: one platform serving multiple FI tenants
- Consent management for financial data (FDX consent model, PSD2 consent flows)
- API Gateway patterns for open banking (rate limiting per TPP, consent validation)
- Discovery questions
- References

**Day 5: `data-partitioning.md`**

Finance-adapted version of the healthcare data partitioning file.

Sections to write:
- DynamoDB for financial transaction data (partition key design, hot partition risk,
  write sharding for high-volume merchants)
- RDS/Aurora for ledger data (per-tenant schema, RLS, Aurora Serverless for variable load)
- S3 for financial documents (loan files, statements, audit records — per-tenant buckets)
- Timestream / DynamoDB for time-series transaction data
- Redshift for financial analytics (multi-tenant data warehouse patterns)
- Financial data backup and recovery: RPO/RTO requirements for payment systems
- Right-to-erasure for financial data (CCPA/GDPR vs. GLBA retention requirements —
  there is a tension: GLBA requires 5-year retention while GDPR allows erasure)
- Key strategy: per-tenant CMKs for CDE data, per-tenant CMKs for PII
- Discovery questions
- References

**Week 2 Exit Criteria:**
- [ ] `tenancy-models.md` complete (finance-adapted)
- [ ] `tenant-isolation.md` complete (finance-adapted)
- [ ] `identity-and-onboarding.md` complete (finance-adapted)
- [ ] `payments-and-ledger.md` complete
- [ ] `open-banking-and-interop.md` complete
- [ ] `data-partitioning.md` complete (finance-adapted)

---

## Week 3: Domain-Specific Finance Steering Files

**Goal:** Write the four domain-specific steering files that cover the specialized
workloads unique to financial services — fraud/AML, trading/market data, lending/credit,
and GenAI with financial data. Also complete the remaining infrastructure files.

**Day 1: `fraud-detection-and-aml.md`**

One of the most technically rich files. No healthcare equivalent.

Sections to write:
- AML program architecture: transaction monitoring pipeline (Kinesis → Lambda →
  detection models → SQS → review queue)
- SAR (Suspicious Activity Report) filing workflow on AWS
- CTR (Currency Transaction Report) — automated $10K+ cash transaction detection
- OFAC sanctions screening architecture (real-time API integration, batch screening)
- Real-time fraud detection: Amazon Fraud Detector, SageMaker for custom models,
  event-driven scoring pipeline
- Multi-tenant fraud patterns: shared models vs. per-tenant models; model isolation
  to prevent data leakage between competitors
- FinCEN BSA/AML requirements mapped to AWS architecture
- Model risk for fraud and AML models (SR 26-2 applicability)
- Case management and investigation workflow patterns
- Discovery questions
- References

**Day 2: `trading-and-market-data.md`**

Capital markets segment specific file.

Sections to write:
- Low-latency trading architecture on AWS (EC2 with enhanced networking, placement
  groups, EFA for HPC)
- Market data ingestion: real-time feeds (WebSocket, Kinesis, MSK/Kafka)
- FIX protocol on AWS: order routing, execution reports, drop copy
- Order management system (OMS) multi-tenant patterns
- Pre-trade risk controls (position limits, credit checks) — latency requirements
- Post-trade: trade confirmation, settlement, reconciliation with DTC/DTCC
- Market data storage: time-series data at scale (Timestream, MemoryDB, DynamoDB)
- MiFID II compliance: trade reporting, best execution, record retention
- SEC Reg SCI: system compliance and integrity for trading platforms
- Multi-tenant isolation for competing trading firms (silo required — competitors
  cannot share infrastructure)
- Discovery questions
- References

**Day 3: `lending-and-credit.md`**

Lending segment specific file.

Sections to write:
- Loan origination system (LOS) architecture: application intake → underwriting →
  decisioning → closing → servicing
- Credit decisioning pipeline: bureau data pull (Equifax, Experian, TransUnion via API)
  → scorecard → decision → adverse action
- FCRA compliance in decisioning: adverse action notices, permissible purpose,
  accuracy requirements
- ECOA / Reg B: fair lending, disparate impact analysis, model explainability
- TILA / RESPA: truth-in-lending disclosures, mortgage servicing requirements
- Multi-tenant LOS: one platform, multiple lender tenants — loan data is highly
  sensitive and must be isolated
- BNPL (Buy Now Pay Later) architecture patterns
- Loan servicing: payment processing, delinquency management, escrow administration
- Collections architecture: FDCPA compliance considerations
- SR 26-2 model risk for credit scoring models
- Discovery questions
- References

**Day 4: `genai-and-financial-data.md`**

Finance equivalent of `genai-and-phi.md`. Covers AI/ML with sensitive financial data.

Sections to write:
- GenAI use cases in finance: fraud narrative generation, credit memo drafting,
  document extraction (loan applications, financial statements), robo-advisory,
  regulatory reporting assistance
- AWS Bedrock for financial services: GLBA/PCI scope considerations, data handling,
  no training on customer data
- SR 26-2 model risk for GenAI: validation requirements for LLM-based decisioning,
  human oversight requirements
- Dual-zone architecture: financial data zone (persists PII/financial data) vs.
  AI zone (inference only, no persistent financial data storage)
- RAG (Retrieval-Augmented Generation) for internal financial knowledge bases:
  per-tenant embeddings, tenant-scoped vector search (OpenSearch Serverless)
- Prompt injection risks in financial applications
- Explainability requirements: ECOA/Reg B adverse action for AI-driven credit decisions
- Model monitoring: drift detection, fairness monitoring, backtesting for financial models
- Discovery questions
- References

**Day 5: Remaining infrastructure files**

- [ ] `audit-logging-and-access.md` — SOX ITGC audit trails, SEC 17a-4 WORM storage,
  immutable financial records, CloudTrail + S3 Object Lock configuration, break-the-glass
  for financial data, retention periods by regulation (GLBA 5yr, SEC 17a-4 6yr/3yr by type)

- [ ] `api-gateway-and-networking.md` — PCI network segmentation in SaaS, CDE
  isolation with VPC/PrivateLink, API security for open banking (mTLS, rate limiting per
  TPP/tenant), WAF rules for financial APIs, SWIFT connectivity patterns

**Week 3 Exit Criteria:**
- [ ] `fraud-detection-and-aml.md` complete
- [ ] `trading-and-market-data.md` complete
- [ ] `lending-and-credit.md` complete
- [ ] `genai-and-financial-data.md` complete
- [ ] `audit-logging-and-access.md` complete
- [ ] `api-gateway-and-networking.md` complete

---

## Week 4: Artifacts, Remaining Files & Power Finalization

**Goal:** Write the artifact templates, complete the remaining steering files, finalize POWER.md with the complete routing table, and do an end-to-end review.

**Day 1: `artifacts-finance.md`**

Finance equivalents of `artifacts-healthcare.md`. Defines templates for finance-specific artifacts.

- PCI Scoping Matrix (which services are in CDE, which are out of scope)
- SOX ITGC Control Matrix (access control, change management, availability per AWS service)
- AML Risk Assessment template
- Financial Data Flow Map (PAN/PII ingress → processing → storage → egress with encryption annotations)
- Open Banking Consent Flow diagram
- Adverse Action Notice design (for AI-driven credit decisions under ECOA/FCRA)
- BAA equivalent: Data Processing Agreements (DPA) inventory for financial SaaS

**Day 2: `artifacts-saas.md` + `artifacts-lld.md`**

These are largely reusable from the healthcare power with finance-specific tweaks:

- [ ] `artifacts-saas.md` — Update HLD template: replace healthcare segment/regulatory fields with finance equivalents (PCI tier, SOX scope, AML obligation). All other artifact templates (Tenant Isolation Matrix, ADR, Onboarding Flow, Data Partitioning Map, Tiering Matrix, Cost Attribution) carry over with minor edits.
- [ ] `artifacts-lld.md` — Reuse as-is from healthcare power. No domain-specific changes needed.

**Day 3: Remaining steering files**

- [ ] `observability-and-operations.md` — Tenant-aware logging for financial SaaS, per-tenant transaction metrics, noisy neighbor detection, SLA monitoring for payment systems (99.99%+ availability), CloudWatch dashboards for financial KPIs, anomaly detection for fraud signals

- [ ] `resilience-and-deployment.md` — Financial system availability requirements, multi-AZ/multi-region for payment-critical services, RTO/RPO targets by workload type, cell-based architecture for large payment platforms, CI/CD for regulated financial software, change management (SOX change control integration)

- [ ] `cost-attribution-and-billing.md` — Per-tenant cost visibility for fintech SaaS, interchange and revenue share metering, AWS Marketplace integration for financial software, cost allocation by CDE vs. non-CDE workloads

- [ ] `saas-lens-review.md` — Well-Architected SaaS Lens + Financial Services Industry Lens combined review. Finance-specific review questions across all five pillars, findings templates with PCI/SOX/GLBA risk annotations

**Day 4: POWER.md finalization**

- [ ] Complete the steering file routing table with all 21 files
- [ ] Write all example journeys (one per segment: neobank, lending, trading, insurtech)
- [ ] Write the Quick Start agent reference table
- [ ] Write the Key AWS References table (FSI Lens, SaaS Lens, compliance center links)
- [ ] Add the GRC (Governance, Risk, Compliance) trigger conditions — equivalent to healthcare's GxP triggers. Finance triggers: "Is your platform subject to SOX?" / "Do you process card payments?" / "Are you a registered broker-dealer?"
- [ ] Final review: verify all steering file names match the routing table, all example journeys reference real files, no broken cross-references

**Day 5: End-to-end review & packaging**

- [ ] Read through all 21 steering files as a connected set — check for gaps, contradictions, missing cross-references
- [ ] Verify every regulatory claim has a source link
- [ ] Check all AWS service names are current (no deprecated services)
- [ ] Add content freshness table to `financial-compliance-foundations.md` (compliance deadlines, verify-at links)
- [ ] Create `README.md` at repo root: installation instructions, what's in the power, how to use it
- [ ] Final folder structure review

**Week 4 Exit Criteria:**
- [ ] `artifacts-finance.md` complete
- [ ] `artifacts-saas.md` complete (finance-adapted)
- [ ] `artifacts-lld.md` complete (reused)
- [ ] `observability-and-operations.md` complete
- [ ] `resilience-and-deployment.md` complete
- [ ] `cost-attribution-and-billing.md` complete
- [ ] `saas-lens-review.md` complete
- [ ] `POWER.md` fully complete with routing table and all journeys
- [ ] End-to-end review done
- [ ] `README.md` written

---

## Complete File List

### Root
| File | Status |
|---|---|
| `POWER.md` | Week 1 (skeleton) + Week 4 (finalize) |
| `README.md` | Week 4 |

### steering/
| File | Week | Finance Equivalent Of |
|---|---|---|
| `financial-compliance-foundations.md` | 1 | `healthcare-compliance-foundations.md` |
| `pii-financial-data-handling.md` | 1 | `phi-data-handling.md` |
| `tenancy-models.md` | 2 | `tenancy-models.md` (adapted) |
| `tenant-isolation.md` | 2 | `tenant-isolation.md` (adapted) |
| `identity-and-onboarding.md` | 2 | `identity-and-onboarding.md` (adapted) |
| `payments-and-ledger.md` | 2 | *(new — no healthcare equivalent)* |
| `open-banking-and-interop.md` | 2 | `fhir-and-interop.md` |
| `data-partitioning.md` | 2 | `data-partitioning.md` (adapted) |
| `fraud-detection-and-aml.md` | 3 | *(new — no healthcare equivalent)* |
| `trading-and-market-data.md` | 3 | *(new — no healthcare equivalent)* |
| `lending-and-credit.md` | 3 | `payer-saas-patterns.md` (analog) |
| `genai-and-financial-data.md` | 3 | `genai-and-phi.md` |
| `audit-logging-and-access.md` | 3 | `audit-logging-and-access.md` (adapted) |
| `api-gateway-and-networking.md` | 3 | `api-gateway-and-networking.md` (adapted) |
| `artifacts-finance.md` | 4 | `artifacts-healthcare.md` |
| `artifacts-saas.md` | 4 | `artifacts-saas.md` (adapted) |
| `artifacts-lld.md` | 4 | `artifacts-lld.md` (reused) |
| `observability-and-operations.md` | 4 | `observability-and-operations.md` (adapted) |
| `resilience-and-deployment.md` | 4 | `resilience-and-deployment.md` (adapted) |
| `cost-attribution-and-billing.md` | 4 | `cost-attribution-and-billing.md` (adapted) |
| `saas-lens-review.md` | 4 | `saas-lens-review.md` (adapted) |

---

## Dependencies & Sequencing Notes

- `financial-compliance-foundations.md` must be written first — all other files reference it for regulatory context.
- `pii-financial-data-handling.md` must be ready before `payments-and-ledger.md` (PAN handling) and `fraud-detection-and-aml.md` (PII in transaction monitoring).
- `tenancy-models.md` must be ready before `artifacts-saas.md` (the HLD template references tenancy decisions).
- `artifacts-saas.md` must be ready before `artifacts-finance.md` (finance artifacts inherit the HLD template and readiness rules).
- `POWER.md` routing table can only be finalized in Week 4 once all 21 files exist and their exact filenames are confirmed.
