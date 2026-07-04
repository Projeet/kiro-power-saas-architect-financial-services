# SaaS Architect for Finance on AWS — Kiro Power

A Kiro Power that turns Kiro into an AWS financial services SaaS architecture advisor.

## What It Does

Provides opinionated, regulation-grounded architectural guidance for teams building multi-tenant financial services SaaS on AWS. Covers tenancy models, financial data protection, compliance posture (PCI-DSS, GLBA, SOX, BSA/AML), payments & ledger architecture, open banking interoperability, fraud detection, and Well-Architected reviews — producing concrete architecture artifacts.

## Segments Covered

| Segment | Key Topics |
|---|---|
| Banking & Neobanking | Core banking SaaS, deposits, payments, cards, BaaS platforms |
| Lending & Credit | Loan origination, BNPL, credit decisioning, FCRA/ECOA compliance |
| Capital Markets & Trading | Brokerage, OMS, market data, FIX protocol, MiFID II |
| Insurance & Wealth Management | InsurTech, robo-advisory, portfolio management |

## Structure

```
├── POWER.md                              # Power definition (identity, routing, behavior)
├── steering/
│   ├── financial-compliance-foundations.md  # PCI-DSS, GLBA, SOX, BSA/AML, FCRA, FFIEC
│   ├── pii-financial-data-handling.md      # PAN, tokenization, KMS, Macie, de-identification
│   ├── tenancy-models.md                   # Silo/pool/bridge for finance
│   ├── tenant-isolation.md                 # IAM isolation, CDE segmentation
│   ├── identity-and-onboarding.md          # KYC/KYB, SCA, OAuth for open banking
│   ├── payments-and-ledger.md              # Payment rails, double-entry, idempotency
│   ├── open-banking-and-interop.md         # FDX, Section 1033, PSD2, ISO 20022
│   ├── data-partitioning.md                # DynamoDB, Aurora, S3 for financial data
│   ├── fraud-detection-and-aml.md          # AML pipeline, SAR/CTR, OFAC, Fraud Detector
│   ├── trading-and-market-data.md          # Low-latency, FIX, OMS, MiFID II, SEC 17a-4
│   ├── lending-and-credit.md               # LOS, credit decisioning, ECOA, adverse action
│   ├── genai-and-financial-data.md         # Bedrock, model risk (SR 11-7), RAG, explainability
│   ├── audit-logging-and-access.md         # SOX ITGC, SEC 17a-4 WORM, break-the-glass
│   ├── api-gateway-and-networking.md       # PCI segmentation, mTLS, PrivateLink, WAF
│   ├── artifacts-finance.md                # PCI Scoping Matrix, SOX Matrix, AML Risk, DPA
│   ├── artifacts-saas.md                   # HLD, Isolation Matrix, ADR, Onboarding, Tiering
│   ├── artifacts-lld.md                    # Low-Level Design template
│   ├── observability-and-operations.md     # Per-tenant metrics, payment SLA, noisy neighbor
│   ├── resilience-and-deployment.md        # Multi-region, SOX CI/CD, DR testing
│   ├── cost-attribution-and-billing.md     # Per-tenant cost, Marketplace, CDE cost tracking
│   └── saas-lens-review.md                 # SaaS Lens + FSI Lens review framework
└── README.md
```

## Installation

Install this power in Kiro via the Powers panel. The `POWER.md` file defines the power identity and the `steering/` folder contains the knowledge base.

## What This Power Does NOT Do

- Provide legal, regulatory, or compliance advice
- Replace a PCI-DSS QSA, BSA Officer, or compliance team
- Advise on banking charter or licensing strategy
- Generate production-ready code without human review

Always verify compliance decisions with qualified professionals.

## Key AWS References

- [AWS Financial Services Industry Lens](https://docs.aws.amazon.com/wellarchitected/latest/financial-services-industry-lens/)
- [AWS FSI Compliance Center](https://aws.amazon.com/financial-services/security-compliance/compliance-center/us/)
- [SaaS Architecture Fundamentals](https://docs.aws.amazon.com/whitepapers/latest/saas-architecture-fundamentals/)
- [SaaS Tenant Isolation Strategies](https://docs.aws.amazon.com/whitepapers/latest/saas-tenant-isolation-strategies/)
- [Amazon Payment Cryptography](https://docs.aws.amazon.com/payment-cryptography/)
