# SaaS Lens + FSI Lens Review

## Overview

This file guides a combined Well-Architected review using the SaaS Lens AND the Financial Services Industry (FSI) Lens. The SaaS Lens covers multi-tenancy best practices; the FSI Lens covers financial services compliance and operational resilience. Together they provide a comprehensive architecture assessment for financial services SaaS.

---

## Review Process

### Step 1: Establish Context
Before reviewing, gather:
- Product description and segment (banking, lending, trading, insurance)
- Tenant count and tier breakdown
- Regulatory scope (PCI-DSS, GLBA, SOX, BSA/AML, FINRA, SEC)
- AWS services in use
- Current pain points or motivating concern

### Step 2: Walk Through Pillars
Ask questions from each pillar below. Not all questions apply to every engagement — select based on the product's segment and regulatory scope.

### Step 3: Document Findings
Use the SaaS Lens Review Report template in `artifacts-saas.md` (Artifact 4) to document findings with risk level, effort, and priority.

### Step 4: Produce Roadmap
Group findings into phases (Critical → High → Improvement) with timelines.

---

## Pillar 1: Operational Excellence

### SaaS Lens Questions
- How are new tenants onboarded? (Automated or manual? How long does it take?)
- How is tenant context propagated through the stack?
- Do you have per-tenant observability (metrics, logs, dashboards)?
- How do you detect and respond to noisy neighbors?
- Is your deployment pipeline fully automated with rollback capability?

### FSI Lens Additions
- Is your change control process SOX-compliant? (Documented approval, segregation of duties)
- Do you generate SOX audit evidence automatically from your CI/CD pipeline?
- Can you produce per-tenant SLA reports for bank examiners?
- Do you test disaster recovery regularly? (Documented results)
- Is there a documented incident response plan with regulatory notification SLAs?

### Common Findings (Finance)
- Manual tenant onboarding with no KYC/KYB gate → automate with compliance verification
- No per-tenant metrics → implement tenant-aware CloudWatch with EMF
- Deployment without approval gate → add SOX-compliant approval for production changes
- DR not tested → quarterly DR drills with documented outcomes

---

## Pillar 2: Security

### SaaS Lens Questions
- How is tenant isolation enforced? (IAM, network, both?)
- Is isolation tested continuously?
- How is tenant context in JWTs validated?
- Can you prevent cross-tenant access even with application bugs?

### FSI Lens Additions
- Is the CDE network-segmented from non-CDE workloads? (PCI Requirement 1)
- Are per-tenant KMS CMKs used for financial PII?
- Is MFA enforced for all interactive access to financial systems? (FFIEC, PCI Req 8)
- Are there no shared accounts in production? (PCI, SOX)
- Is OFAC screening implemented at onboarding AND transaction time?
- Is there a break-the-glass procedure with dual-control approval?
- Are audit logs immutable (S3 Object Lock Compliance Mode)?

### Common Findings (Finance)
- Application-level isolation only (no IAM backing) → add dynamic session policies
- CDE accessible from non-CDE subnets → network segmentation with strict SGs
- Shared KMS key across tenants → per-tenant CMKs for financial PII
- No OFAC screening at transaction time → real-time screening pipeline
- Logs deletable by admins → S3 Object Lock Compliance Mode

---

## Pillar 3: Reliability

### SaaS Lens Questions
- How do you handle tenant-specific failures without affecting other tenants?
- Is there per-tenant circuit breaking or bulkhead isolation?
- What's your blast radius if a deployment fails?

### FSI Lens Additions
- What's your availability target for payment-critical services? (99.99% for RTP/FedNow)
- Is your architecture multi-AZ at minimum for all financial workloads?
- For real-time payments: is it multi-region?
- Are payment APIs idempotent? (Exactly-once processing)
- What's your tested RTO/RPO for each data tier?
- Can you handle ACH returns and chargebacks that arrive days after settlement?

### Common Findings (Finance)
- Single-AZ database for payment workloads → Multi-AZ Aurora with automatic failover
- No idempotency on payment APIs → implement idempotency key pattern
- Shared payment queue for all tenants → per-tenant queues to isolate blast radius
- RTO/RPO not defined or tested → document targets and test quarterly

---

## Pillar 4: Performance Efficiency

### SaaS Lens Questions
- How do you handle per-tenant performance variance?
- Is there per-tenant throttling at the API layer?
- How do you scale per service based on tenant demand?

### FSI Lens Additions
- Is your payment authorization latency < 2 seconds end-to-end?
- For trading workloads: is pre-trade risk checking sub-millisecond?
- Are database connections managed to prevent per-tenant exhaustion? (RDS Proxy)
- Is market data distribution optimized for latency? (MemoryDB, placement groups)

### Common Findings (Finance)
- No per-tenant rate limiting → API Gateway usage plans per tenant
- Database connections shared without limits → RDS Proxy + per-tenant connection caps
- Payment latency spikes during batch processing → separate batch from real-time path

---

## Pillar 5: Cost Optimization

### SaaS Lens Questions
- Can you attribute AWS costs to individual tenants?
- Are all resources tagged with tenant context?
- Is there a tiering strategy that aligns price with resource consumption?

### FSI Lens Additions
- Can you separate CDE costs from non-CDE costs? (PCI scope cost visibility)
- Are per-tenant third-party costs tracked? (Bureau pulls, processor fees, KYC)
- Do you know your per-tenant gross margin?
- Are you using reserved capacity or savings plans for predictable financial workloads?
- Is there cost alerting for runaway tenant consumption?

### Common Findings (Finance)
- No resource tagging → enforce tags via SCP from day one
- CDE cost unknown → tag PCI_Scope on all resources
- Third-party costs not attributed → track per-tenant bureau, processor, and KYC costs
- Over-provisioned silo resources → right-size or move to serverless where appropriate

---

## FSI Lens Specific Review Areas

### Regulatory Compliance Posture
- Is there a documented compliance matrix (PCI, GLBA, SOX, BSA/AML)?
- Are compliance certifications current (SOC 2, PCI AOC)?
- Is there a designated BSA Officer (if AML obligations apply)?
- Is there a GLBA-compliant information security program?

### Third-Party Risk
- Is there a vendor/processor inventory with DPA status?
- Are critical third parties assessed annually?
- Do contracts include right-to-audit and incident notification SLAs?

### Data Governance
- Is there a data inventory (where financial PII lives across all services)?
- Is there a documented retention and disposal policy?
- Is the GLBA/CCPA retention vs. erasure tension addressed?

---

## Findings Severity Scale

| Level | Definition | Timeline |
|---|---|---|
| Critical | Active compliance violation, data exposure risk, or regulatory deadline imminent | Fix within 2 weeks |
| High | Gap that would be flagged in next audit/assessment, or reliability risk for financial workloads | Fix within 1 month |
| Medium | Best practice gap with moderate risk, or operational inefficiency | Fix within 1 quarter |
| Low | Improvement opportunity, optimization, or future-proofing | Plan for next quarter |

---

## References

- [AWS Well-Architected SaaS Lens](https://docs.aws.amazon.com/wellarchitected/latest/saas-lens/saas-lens.html)
- [AWS Financial Services Industry Lens](https://docs.aws.amazon.com/wellarchitected/latest/financial-services-industry-lens/financial-services-industry-lens.html)
- [AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html)
- [AWS Well-Architected Tool](https://docs.aws.amazon.com/wellarchitected/latest/userguide/intro.html)
