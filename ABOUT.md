# New Kiro Power — SaaS Architect for Finance on AWS

A co-architect for teams building multi-tenant financial services SaaS on AWS. It has already digested the AWS SaaS whitepapers, the Well-Architected SaaS Lens, the Financial Services Industry Lens, PCI-DSS v4.0, GLBA Safeguards Rule, SOX ITGC controls, BSA/AML requirements, open banking standards (FDX, ISO 20022, PSD2), and the SaaS Builder Toolkit — so you don't have to.

## Why it matters

Financial services SaaS architecture conversations usually stall on the same things — how do we minimize PCI CDE scope in a multi-tenant platform, how do we isolate financial data per tenant for bank examiners, which payment rail architecture fits our use case, how do we meet BSA/AML obligations without building a compliance team of 20, does our credit model trigger SR 11-7 model risk governance. Reading the source material end-to-end is a multi-month effort across dozens of regulatory and technical documents, and getting it wrong shows up as rework, failed QSA assessments, lost enterprise bank deals, or enforcement actions.

## What it helps with

- **Tenancy model choice** — silo, pool, bridge — for financial services workloads, with finance-specific defaults (bridge for any service handling financial PII, silo for competing broker-dealers)
- **Financial data isolation** — per-tenant KMS, IAM session scoping, CDE network segmentation, tenant context propagation
- **PCI-DSS CDE scoping** — minimize what's in scope via tokenization at ingress, hosted payment fields, Amazon Payment Cryptography
- **Payments & ledger architecture** — ACH, RTP, FedNow, card networks, SWIFT, double-entry ledger patterns, idempotency, settlement and reconciliation
- **Open banking** — FDX API, CFPB Section 1033 compliance deadlines, PSD2, ISO 20022 migration, consent management
- **Fraud detection & AML** — real-time transaction monitoring pipelines, SAR/CTR filing workflows, OFAC sanctions screening, Amazon Fraud Detector multi-tenant patterns
- **Capital markets** — low-latency trading on AWS, FIX protocol, pre-trade risk controls, SEC 17a-4 record retention, MiFID II compliance
- **Lending & credit** — loan origination, credit decisioning with ECOA-compliant explainability, adverse action notice generation, SR 11-7 model risk governance
- **GenAI with financial data** — Bedrock for finance (GLBA/PCI scope), dual-zone architecture, per-tenant RAG, model risk management
- **SaaS Lens + FSI Lens reviews** — combined Well-Architected assessment with financial services compliance overlay
- **Artifact and code generation** grounded in the above

## Impact

- **Cuts financial services architecture discovery from weeks to days.** The power has pre-digested PCI-DSS v4.0, GLBA, SOX, BSA/AML, FCRA, FFIEC guidance, and the FSI Lens — no more reading 1000+ pages before your first architecture decision.
- **Catches PCI scope creep, GLBA gaps, and SOX ITGC weaknesses before they reach QSA assessment or bank examiner review.** Flags issues like SAD in logs, missing OFAC screening at transaction time, or shared KMS keys across tenants.
- **Produces design artifacts that compliance, security, and bank examiners can actually use** — PCI Scoping Matrix, SOX ITGC Control Matrix, Financial Data Flow Map, Tenant Isolation Matrix, AML Risk Assessment, Adverse Action Notice design, and more.
- **Flags regulatory triggers early** — SR 11-7 model risk for credit/fraud models, Section 1033 compliance deadlines, MiFID II transaction reporting — avoiding expensive re-architecture late in the program.
- **Generates IaC and application code that follows SaaS and financial services principles by default** — per-tenant KMS, IAM session scoping, CDE network segmentation, idempotent payment APIs, tenant-aware logging, PAN tokenization at ingress.
- **Supports four financial segments in one power** — banking & neobanking, lending & credit, capital markets & trading, and insurance & wealth management — with segment-specific steering files loaded on demand based on context.
