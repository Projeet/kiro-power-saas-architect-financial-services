# Tenancy Models & Architecture Foundations

## Core Concept: SaaS Is a Business Model, Not Just an Architecture

Per the AWS SaaS Architecture Fundamentals whitepaper: SaaS and multi-tenancy are not the same thing. SaaS is a business and delivery model. Multi-tenancy describes how resources are shared within that model. A fully siloed SaaS where every tenant has dedicated resources is still SaaS — as long as tenants are managed, operated, and deployed collectively through a unified experience.

## The Three Tenancy Models

### Pool Model (Shared Resources)
All tenants share the same compute, storage, and infrastructure. Tenant separation is logical (e.g., tenant ID in database rows, IAM policies scoping access).

**Strengths:** Lowest cost per tenant, simplest deployment, fastest onboarding, best operational efficiency.
**Weaknesses:** Noisy neighbor risk, isolation complexity, compliance challenges (harder to prove data separation to PCI QSAs and SOX auditors), right-to-erasure is harder, blast radius affects all tenants.
**Best for:** High tenant count (hundreds to thousands), SMB customers, cost-sensitive markets.

### Silo Model (Dedicated Resources)
Each tenant gets their own dedicated infrastructure — separate compute, storage, and potentially separate AWS accounts.

**Strengths:** Strongest isolation, no noisy neighbor, simplest compliance story for QSAs/auditors, per-tenant customization possible, clearest cost attribution.
**Weaknesses:** Highest cost per tenant, operational overhead scales with tenant count, slower onboarding, doesn't scale well beyond ~50-100 tenants without heavy automation.
**Best for:** Chartered banks, broker-dealers, large enterprise tenants, regulated entities with contractual isolation requirements.

### Bridge Model (Hybrid)
Some resources are shared, others are dedicated. Most common: shared compute, separate storage per tenant.

**Strengths:** Balances cost and isolation, flexible, good compliance middle ground.
**Weaknesses:** More complex than either pure model, onboarding more complex than pool.
**Best for:** Mid-market fintech, situations where financial data isolation is required but compute isolation isn't.

## Critical Insight: Tenancy Varies Per Service

In a real financial services SaaS system, each service can have a different tenancy model:
- **Payment processing service**: Bridge (shared compute, isolated tokenization per tenant)
- **Transaction ledger service**: Bridge or Silo (financial data requires strong isolation)
- **Analytics/reporting service**: Pool (aggregated data, lower sensitivity)
- **Compliance/AML service**: Bridge (shared detection engine, per-tenant alert queues and case files)
- **Admin/configuration service**: Pool (low sensitivity, operational data)

**When advising on tenancy, evaluate each service independently.**

## Control Plane vs. Application Plane

### Control Plane (Always Shared)
Manages and operates your SaaS environment across ALL tenants regardless of their tenancy model:
- Tenant management: CRUD, configuration, regulatory tier
- Identity and authentication: KYC/KYB-gated onboarding, MFA, tenant-user binding
- Billing and metering: usage tracking, interchange revenue share, Marketplace integration
- Tiering and throttling: tier-based API limits, transaction volume caps
- Metrics and analytics: tenant activity dashboards, operational metrics
- Deployment management: rolling updates to all tenants

### Application Plane (Tenancy Model Varies)
The actual SaaS application — payments, lending, trading, etc. This is where tenancy model decisions apply.

## Tiering Strategies for Financial Services

| Tier | Tenancy Model | Isolation | Throttling | Typical Tenant |
|------|--------------|-----------|------------|----------------|
| Starter | Pool | Logical (IAM + app-level) | Aggressive limits | Early-stage fintechs, small credit unions |
| Professional | Bridge | Logical + separate storage, per-tenant KMS | Moderate limits | Mid-size lenders, community banks |
| Enterprise | Silo | Infrastructure-level, dedicated account possible | Custom / generous | Chartered banks, broker-dealers, large insurers |


## Decision Framework: Choosing a Tenancy Model

### 1. Regulatory Requirements
- **PCI-DSS CDE isolation?** → The CDE itself must be network-segmented regardless of tenancy model. But per-tenant CDE isolation (separate CDE per tenant) is silo; shared CDE with per-tenant tokenization is bridge.
- **SOX audit requirements?** → Enterprise customers undergoing SOX audits prefer silo — their auditors want to see dedicated infrastructure.
- **BSA/AML per-tenant obligations?** → SAR filings are per-institution. Alert queues and case management MUST be tenant-isolated even in pool compute.
- **OCC/FFIEC examination expectations?** → Bank tenants' examiners will scrutinize your isolation story. Bridge or silo for any data they regulate.

### 2. Tenant Type — The Finance Distinction
| Tenant Type | Typical Isolation Requirement | Why |
|---|---|---|
| Chartered bank | Silo or strong bridge | Examined by OCC/FDIC; examiners expect demonstrable isolation |
| Broker-dealer (FINRA registered) | Silo | SEC/FINRA rules on record separation; competing firms cannot share infrastructure |
| Credit union | Bridge | NCUA examination expectations, but smaller budgets than banks |
| Fintech (non-bank) | Pool or bridge | Fewer regulatory isolation mandates; partner bank may impose requirements |
| Insurance company | Bridge | State insurance regulations vary; data sensitivity moderate |
| Money services business (MSB) | Bridge | FinCEN oversight; transaction data must be isolated |

### 3. Noisy Neighbor Risk
- Payment processing has burst patterns (Black Friday, payroll days) → consider silo for high-volume merchants
- Lending has batch patterns (end-of-month processing, bureau pulls) → bridge with burst capacity
- Trading has microsecond requirements → silo is often mandatory for competing firms

### 4. Tenant Count and Growth
- 10-50 tenants (large banks, enterprises): silo manageable
- 100-1000 tenants (community banks, mid-market fintechs): bridge required
- 1000+ tenants (SMB merchants, small lenders): pool required for economics

### 5. Revenue Per Tenant
- $500K+/year: silo justified
- $5K-$50K/year: bridge sweet spot
- <$5K/year: pool or you can't make the economics work

## Default Recommendations for Financial Services

**Banking & Neobanking SaaS (BaaS platforms, core banking):**
Bridge as the default for ALL services handling account data, transaction data, or financial PII. Pool only for non-sensitive services (feature flags, rate tables, public reference data). Silo for chartered bank tenants where the contract or examiner requires it. The finance default is bridge — not pool — because PCI-DSS CDE scoping, GLBA data protection, and bank examiner expectations all require demonstrable data isolation.

**Lending & Credit SaaS (loan origination, credit decisioning):**
Bridge for loan data services (application data, credit bureau responses, decisioning outputs). Pool for reference data (rate tables, product catalogs). Bridge for credit decisioning models (per-tenant model configuration, shared inference infrastructure). Silo for large lender tenants with contractual requirements or those subject to direct CFPB examination.

**Capital Markets & Trading SaaS (brokerage, OMS, market data):**
Silo for competing trading firms — this is non-negotiable. Competing broker-dealers sharing pool infrastructure creates unacceptable regulatory and competitive risk. Bridge for market data distribution (shared feed, per-tenant entitlements). Pool for reference data (instruments, exchanges, calendars). Silo for order execution and position management.

**Insurance & Wealth Management SaaS:**
Bridge for policy and portfolio data (per-tenant storage, shared compute). Pool for product catalog and rate engines (shared configuration). Silo for large carriers with state-specific regulatory requirements. Bridge for robo-advisory (shared model infrastructure, per-tenant portfolio data with per-tenant KMS keys).

### Bank Tenant vs. Fintech Tenant

In financial services SaaS, your tenants fall into distinct regulatory tiers:
- **Chartered institutions** (banks, credit unions, broker-dealers): have direct regulatory oversight, their examiners will review your architecture
- **Licensed fintechs** (money transmitters, lending licensees): state-regulated, less intensive examination
- **Partner-bank-dependent fintechs**: operate under a bank's charter, the bank's compliance team reviews your architecture on their behalf
- **Non-regulated tenants** (B2B fintech tools without direct consumer finance): minimal regulatory isolation requirements

This distinction affects tenancy decisions the same way covered entity vs. business associate does in healthcare — the regulated tenant demands stronger isolation.

## Common Tension Patterns

**"We need PCI compliance but serve 10,000 merchants"**
Pool the compute, but isolate the CDE. The CDE (tokenization service, payment processing microservice) can be shared infrastructure — it's still PCI-compliant as long as it's properly segmented from non-CDE services. Each merchant's tokens are tenant-scoped so one merchant's token cannot resolve to another merchant's PAN. You don't need 10,000 separate CDE environments — you need one properly segmented, assessed CDE that serves all merchants with per-tenant tokenization.

**"A large bank wants silo but we serve 500 small fintechs in pool"**
Tiering solves this. Pool/bridge for the 95% that are small fintechs. Silo (or dedicated account) for the bank. The control plane manages both. Design your data access layer to abstract storage location. The bank pays a premium that justifies the operational cost.

**"We handle payments but want pool for cost efficiency"**
Bridge. Pool the compute (Lambda, ECS), silo the financial data stores (per-tenant DynamoDB tables for transactions, per-tenant RDS schemas for ledger, per-tenant KMS keys). You get most of pool's cost benefits while maintaining data isolation that satisfies PCI QSAs and bank examiners.

**"We serve competing broker-dealers — they can't share anything"**
Silo with separate AWS accounts. No exceptions. Competing trading firms cannot share compute, storage, or even metadata visibility. Their regulators (FINRA, SEC) expect complete separation. The operational cost of account-per-tenant is justified by the regulatory and competitive requirements.

**"We're a BaaS platform — our fintech tenants serve end consumers"**
Your tenants' regulatory obligations flow through to you. If a fintech tenant has BSA/AML obligations (they do, via their partner bank), your platform must support per-tenant AML transaction monitoring isolation. Bridge minimum for all financial data; the partner bank's compliance team will audit your isolation.

## Common Mistakes

1. **Picking one model for everything** — Evaluate per service. Your payment processing service and your analytics service have different needs.

2. **Over-siloing early** — Silo is expensive. Don't silo because "it feels safer." Silo when a regulation, examiner, or contract requires it.

3. **Under-isolating in pool** — Pool without IAM-enforced isolation is a cross-tenant data breach waiting to happen. Application-level filtering is necessary but insufficient.

4. **Defaulting to pool for financial data in fintech SaaS** — In generic SaaS, pool is the default. In financial services SaaS, bridge is the default for any service handling financial PII, transaction data, or cardholder data.

5. **Not distinguishing bank tenants from fintech tenants** — A chartered bank has different isolation expectations than an early-stage fintech. Your tenancy model should account for this via tiering.

6. **Ignoring the CDE scope implications** — If your payment service is pool and a PCI QSA puts the entire shared infrastructure in scope, your PCI assessment cost explodes. Segment the CDE regardless of tenancy model.

7. **Not planning for competing tenants** — If two competing institutions are both your tenants, even metadata visibility (tenant names, transaction volumes) can be a competitive concern. Design for zero cross-tenant visibility from day one.

## Discovery Questions for This Domain

**If you don't know the services yet:**
- What are the main services in your system? (Even rough: "payments API, loan engine, admin dashboard")
- Which services handle financial PII, transaction data, or cardholder data?

**If you know services but not the model:**
- For each service: does it store tenant-specific financial data, or is it stateless/reference?
- Are there services where one tenant's burst could affect others? (Payment spikes, batch loan processing)
- Do any tenants have contractual or regulatory requirements for dedicated infrastructure?

**If choosing between models:**
- How many tenants in year 1? Year 3?
- What's your average revenue per tenant?
- How big is your platform/ops team?
- Are any tenants chartered banks, broker-dealers, or entities with direct regulatory examination?

**If designing tiers:**
- What differentiates tiers from the customer's perspective? (Isolation? Volume limits? Support?)
- Do enterprise tenants explicitly require "dedicated" infrastructure, or is that your assumption?
- Can tenants upgrade tiers without downtime or data migration?

## References

- [SaaS Architecture Fundamentals — Re-defining Multi-tenancy](https://docs.aws.amazon.com/whitepapers/latest/saas-architecture-fundamentals/re-defining-multi-tenancy.html)
- [SaaS Architecture Fundamentals — Data Partitioning](https://docs.aws.amazon.com/whitepapers/latest/saas-architecture-fundamentals/data-partitioning.html)
- [SaaS Lens — General Design Principles](https://docs.aws.amazon.com/wellarchitected/latest/saas-lens/general-design-principles.html)
- [AWS Financial Services Industry Lens](https://docs.aws.amazon.com/wellarchitected/latest/financial-services-industry-lens/financial-services-industry-lens.html)
- [Tenant Portability: Move Tenants Across Tiers](https://aws.amazon.com/blogs/architecture/tenant-portability-move-tenants-across-tiers-in-a-saas-application/)
- [Let's Architect! Building Multi-Tenant SaaS Systems](https://aws.amazon.com/blogs/architecture/lets-architect-building-multi-tenant-saas-systems/)
