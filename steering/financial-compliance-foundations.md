# Financial Compliance Foundations

## Why This File Exists

Every financial services SaaS architecture decision is constrained by regulation. PCI-DSS is the floor for card data, GLBA is the floor for consumer financial data, SOX governs audit trails for public companies, and BSA/AML obligations apply to anyone touching money movement. This file covers the regulatory landscape that shapes every other architectural decision in this power. Load this file early in any financial services SaaS conversation.

---

## PCI-DSS v4.0 — Cardholder Data Environment (CDE) Architecture

PCI-DSS (Payment Card Industry Data Security Standard) v4.0 applies to any entity that stores, processes, or transmits cardholder data (CHD) or sensitive authentication data (SAD). For SaaS platforms, this means your platform is in scope if any tenant processes card payments through your infrastructure.

### The 12 Requirements Mapped to AWS

**Requirement 1 — Install and Maintain Network Security Controls**
Segment the CDE from all other systems at the network level. In AWS this means:
- Dedicated VPC (or subnet tier) for CDE workloads — no CDE resources in shared subnets
- Security groups: inbound to CDE limited to explicitly required ports/sources
- NACLs: deny-all baseline with allowlist exceptions
- VPC Flow Logs enabled for all CDE VPCs — stored in S3 with Object Lock
- No direct internet access to CDE components — traffic enters via WAF + API Gateway

**Requirement 2 — Apply Secure Configurations**
- No default credentials on any CDE resource
- AWS Config Rules: detect non-compliant configurations continuously
- Systems Manager Parameter Store (or Secrets Manager) for all credentials — no hardcoded secrets
- IMDSv2 enforced on all EC2 instances in the CDE

**Requirement 3 — Protect Stored Account Data**
- **PAN must never be stored in plaintext.** Tokenize at ingress using Amazon Payment Cryptography or a validated third-party token vault before any storage.
- If PAN must be stored: render unreadable via AES-256 encryption with AWS KMS CMK
- SAD (CVV, full track data, PIN blocks) must NEVER be stored after authorization — not in logs, not in databases, not in backups
- KMS key policy: restrict PAN decryption to explicitly authorized roles only

**Requirement 4 — Protect Cardholder Data with Strong Cryptography During Transmission**
- TLS 1.2+ on all connections involving CHD — TLS 1.3 preferred
- No CHD over unencrypted channels (HTTP, FTP, telnet) — enforce at ALB listener and API Gateway
- Certificate management via ACM — automated rotation, no self-signed certs in CDE
- PrivateLink for service-to-service communication involving CHD within AWS

**Requirement 5 — Protect All Systems from Malicious Software**
- GuardDuty enabled for all CDE accounts — covers malware, compromised credentials, anomalous behavior
- Inspector for vulnerability scanning of EC2, Lambda, and container images in the CDE
- ECR image scanning for container workloads

**Requirement 6 — Develop and Maintain Secure Systems and Software**
- WAF in front of all public-facing CDE applications — OWASP Top 10 ruleset + rate limiting
- Dependency scanning in CI/CD pipeline (Inspector, Dependabot, or equivalent)
- Security code review for CDE application code changes

**Requirement 7 — Restrict Access to System Components by Business Need**
- IAM least-privilege: roles scoped to specific CDE resources, no wildcard resource ARNs for CHD stores
- IAM Access Analyzer to detect overly permissive policies continuously
- No persistent credentials for humans — use IAM Identity Center with time-limited sessions

**Requirement 8 — Identify Users and Authenticate Access**
- MFA required for all interactive access to CDE — no exceptions
- No shared accounts in the CDE — each user/service has a unique identity
- Service-to-service: IAM roles only — no long-lived access keys in the CDE
- Password policy: Cognito or IAM Identity Center enforced complexity and rotation

**Requirement 9 — Restrict Physical Access to Cardholder Data**
- Handled by AWS shared responsibility — AWS manages physical data center security
- Your responsibility: prevent unauthorized logical access, document AWS's role in your QSA assessment

**Requirement 10 — Log and Monitor All Access to System Components and Cardholder Data**
- CloudTrail with data events enabled for all S3 buckets, DynamoDB tables, and KMS keys in the CDE
- CloudWatch Logs retention: 12 months online, 12 months archived minimum (PCI requires 12 months total with 3 months immediately available)
- S3 Object Lock (Compliance Mode) for CloudTrail logs — prevent log deletion or modification
- Automated alerting on: failed logins, privilege escalation, access outside business hours, large data exports

**Requirement 11 — Test Security of Systems and Networks Regularly**
- Quarterly vulnerability scans (external + internal) — can use Inspector + third-party for external
- Annual penetration test of CDE — must be performed by qualified internal or external resource
- Change-detection mechanisms: AWS Config, CloudTrail Insights, Security Hub

**Requirement 12 — Support Information Security with Organizational Policies**
- Documented Information Security Policy covering PCI scope
- Annual risk assessment
- Vendor/third-party management policy (OCC Third-Party Risk guidance applies here too)
- Incident response plan with PCI breach notification procedures

### CDE Scoping in SaaS — The Critical Decision

In a multi-tenant SaaS platform, CDE scoping is the most important PCI architecture decision. Every service that is in scope increases your assessment footprint, cost, and operational burden.

**Minimize CDE scope by:**
1. Tokenize PAN at the earliest possible point — ideally before it reaches your application layer
2. Use hosted payment fields or a payment page iframe (Stripe.js, Braintree Drop-in) so PAN never traverses your servers
3. Route CHD through a dedicated payment microservice with its own VPC subnet — segment everything else out of scope
4. Use Amazon Payment Cryptography for any HSM operations — it is PCI-certified and shifts HSM management to AWS

**CDE in multi-tenant SaaS:**
- Each tenant's payment processing may go through shared payment infrastructure — this is fine
- The shared payment infrastructure IS the CDE; the rest of your platform can be out of scope
- Per-tenant tokenization: tokens are tenant-scoped so Token A for Tenant A cannot be used to look up Tenant B's PAN
- Isolate CDE compute from non-CDE compute at the subnet/security group level, not just the application level


---

## GLBA Safeguards Rule — 9 Elements Mapped to AWS

The Gramm-Leach-Bliley Act (GLBA) Safeguards Rule (16 CFR Part 314, revised 2023) applies to financial institutions under FTC jurisdiction — which includes most fintech companies, lenders, mortgage brokers, auto dealers, and non-bank financial services providers. It requires a comprehensive information security program with 9 specific elements.

**Element 1 — Qualified Individual**
Designate a qualified individual responsible for overseeing and implementing the information security program. Architecture implication: that individual needs visibility into all AWS security controls — Security Hub, GuardDuty, Config, CloudTrail.

**Element 2 — Risk Assessment**
Conduct a periodic risk assessment covering: criteria for evaluating risks, weighted risk analysis, how safeguards address identified risks. Architecture implication: AWS Well-Architected Review results and Security Hub findings serve as evidence.

**Element 3 — Safeguards (9 specific controls)**
- Access controls: IAM least privilege, MFA for console access, Cognito for customer access
- Data inventory: maintain an inventory of customer financial information and where it resides (S3, DynamoDB, RDS) — Macie can discover and classify this
- Encryption: customer financial information encrypted at rest (KMS) and in transit (TLS 1.2+)
- Secure development practices: secure SDLC, dependency scanning, SAST/DAST in CI/CD
- Multi-factor authentication: required for any system holding customer financial information
- Disposal of customer information: documented data retention and disposal policy, KMS key deletion as cryptographic erasure
- Change management: documented procedures for changes to information systems
- Monitoring and testing: continuous monitoring (Security Hub, GuardDuty) + annual penetration test
- Security awareness training: documented training program for employees with access to customer data

**Element 4 — Vendor Oversight**
Oversee service providers with access to customer financial information. Architecture implication: maintain a vendor inventory (DPA/data sharing agreements), include security requirements in contracts, periodically assess vendor security posture. AWS qualifies as a vendor — retain AWS SOC 2 reports and PCI certificates as evidence.

**Element 5 — Incident Response Plan**
Written incident response plan covering: goals, internal processes, roles and responsibilities, external communications, remediation steps. Architecture implication: EventBridge + SNS for automated alerting, Step Functions for incident response orchestration, pre-defined runbooks in Systems Manager.

**Element 6 — Periodic Reassessment**
Evaluate and adjust the security program based on results of testing and monitoring, material changes to operations, and circumstances that may have a material impact. Architecture implication: treat Security Hub findings and Well-Architected Review results as triggers for reassessment.

**Element 7 — Annual Report to the Board**
The qualified individual must report to the board (or equivalent) annually on the overall status of the information security program. Architecture implication: Security Hub compliance summary and Config rule compliance dashboards are useful evidence artifacts.

**GLBA in SaaS context:** If your SaaS platform processes or stores customer financial information (NPI — Nonpublic Personal Information) on behalf of a GLBA-covered tenant, your platform is a service provider under GLBA. Your tenants need contractual assurance that you protect their customers' NPI appropriately. GLBA does not require you to have your own GLBA compliance program per se, but your tenants' compliance programs will require them to assess yours.

---

## SOX Section 404 — IT General Controls (ITGC) Mapped to AWS

The Sarbanes-Oxley Act Section 404 requires public companies (and their SaaS vendors that support financial reporting systems) to document and test IT General Controls (ITGCs). If your SaaS platform is used in a public company's financial reporting process, your SOX controls will be assessed as part of their audit.

### The Four ITGC Domains

**1. Access Control (AC)**
Controls that prevent unauthorized access to financial systems and data.

| Control | AWS Implementation |
|---|---|
| Unique user IDs, no shared accounts | IAM Identity Center with per-user identities, Cognito per-user accounts |
| MFA for privileged access | IAM Identity Center MFA enforcement, Cognito MFA required for admin roles |
| Least privilege access | IAM permission boundaries, role-based access, quarterly access reviews |
| Terminated user access removal | Automated offboarding via SBT lifecycle events, IAM Identity Center suspension |
| Privileged access monitoring | CloudTrail + CloudWatch Alarms on AssumeRole, console sign-in events |

**2. Change Management (CM)**
Controls that ensure changes to systems are authorized, tested, and documented before implementation.

| Control | AWS Implementation |
|---|---|
| Formal change request and approval | CodePipeline with manual approval gates, Change Manager in Systems Manager |
| Test before production | Separate dev/staging/production accounts, automated test suite in CI/CD |
| Code review | Pull request approvals enforced in CodeCommit/GitHub |
| Emergency change process | Break-glass deployment runbook with CloudTrail evidence |
| No developer access to production | IAM policies: developers have no console access to production account |

**3. Availability and Recovery (AR)**
Controls that ensure systems are available and recoverable per defined RTO/RPO.

| Control | AWS Implementation |
|---|---|
| Backup of financial data | AWS Backup with cross-region copy, DynamoDB PITR, RDS automated snapshots |
| Documented RPO/RTO | Well-Architected Reliability pillar review, documented targets per tier |
| DR testing | Quarterly DR drills, AWS Resilience Hub for resilience scoring |
| Monitoring and alerting | CloudWatch alarms for availability metrics, PagerDuty/OpsGenie integration |

**4. Audit Logging (AL)**
Controls that ensure all access and changes to financial data are logged, retained, and protected.

| Control | AWS Implementation |
|---|---|
| Comprehensive audit trail | CloudTrail (management + data events) for all production accounts |
| Log integrity protection | S3 Object Lock (Compliance Mode) for CloudTrail logs |
| Log retention | Minimum 7 years for SOX (SEC 17a-4 requires 6 years, many SOX programs require 7) |
| Log review | CloudWatch Log Insights queries, Security Hub, SIEM integration |

### SOX in SaaS Context

If your SaaS platform supports a public company's financial close, general ledger, revenue recognition, or financial reporting workflow, your platform's ITGCs will be examined during the customer's SOX audit. Common requests from enterprise fintech buyers:
- SOC 1 Type II report (covers controls relevant to user entity ITGC assessments)
- Penetration test results
- Evidence of quarterly access reviews
- Change management policy documentation
- Business continuity and DR plan

Budget 6-12 months and engage a Big Four or mid-tier audit firm for SOC 1 Type II. The controls above are the foundation.

---

## BSA/AML — Transaction Monitoring Architecture

The Bank Secrecy Act (BSA) requires financial institutions to implement an AML (Anti-Money Laundering) compliance program with four pillars: internal policies, a designated BSA Officer, ongoing employee training, and independent testing. Architecture specifically enables the technical pillars: SAR filing, CTR reporting, and OFAC sanctions screening.

### SAR (Suspicious Activity Report) Filing Architecture

SARs must be filed with FinCEN within 30 calendar days of detecting a suspicious transaction (60 days if no suspect identified).

```
Transaction Events → Kinesis Data Streams → Lambda (rule engine) → SageMaker (ML scoring)
                                                    ↓
                                           DynamoDB (alert queue, per-tenant)
                                                    ↓
                                           Step Functions (case management workflow)
                                                    ↓
                              Human Review Queue → FinCEN BSA E-Filing API
```

Key architectural requirements:
- Transaction events must be immutable once written — use S3 with Object Lock or DynamoDB append-only
- Alert queue must be per-tenant — one tenant's SAR workflow must not be visible to another tenant
- Case management audit trail: who reviewed, what decision was made, when, and why — see `audit-logging-and-access.md`
- SAR filings are confidential — cannot disclose to the subject of the SAR ("tipping off" prohibition)

### CTR (Currency Transaction Report) Architecture

CTRs must be filed for cash transactions exceeding $10,000 (single or aggregated over 24 hours per customer).

- Aggregate transaction amounts by customer per calendar day
- Detect $10K threshold crossings in near-real-time — use Kinesis with Lambda aggregation
- CTR filing must occur within 15 calendar days of the transaction
- Exemption management: record customers exempted from CTR filing (Phase I/II exemptions) in DynamoDB with audit trail

### OFAC Sanctions Screening

All US financial institutions must screen counterparties against OFAC SDN (Specially Designated Nationals) list and other sanctions lists.

- Screen at onboarding (KYC/KYB) AND at transaction time for payment counterparties
- OFAC list updates occur frequently — implement webhook or scheduled sync (OFAC provides SDN file downloads)
- Real-time screening: Lambda synchronous call to screening service before payment authorization
- Batch screening: nightly scan of all customer records against updated OFAC list
- Match resolution workflow: fuzzy matches require human review — Step Functions case workflow
- False positive rate management: document screening logic and threshold for QA evidence

---

## FCRA — Credit Data Handling

The Fair Credit Reporting Act governs how consumer credit information is collected, used, and shared. If your SaaS platform pulls credit bureau data or produces credit-related outputs, FCRA applies.

**Permissible Purpose:** Credit bureau data can only be pulled for permissible purposes (credit transaction, employment, insurance, account review, etc.). Your platform must enforce that tenants only pull bureau data for permissible purposes — log every bureau pull with the stated purpose.

**Adverse Action Notices:** If a consumer is denied credit, insurance, or employment based on a consumer report, a formal adverse action notice is required within 30 days. If your SaaS platform drives credit decisions, you must support adverse action notice generation. See `lending-and-credit.md` for the full ECOA/FCRA adverse action architecture.

**Data Accuracy:** Furnishers of data to credit bureaus (if you report payment history) must follow Metro 2 format and maintain data accuracy. Disputes must be investigated within 30 days.

**Retention and Disposal:** Retain records of credit pulls and their stated permissible purpose. Dispose of consumer reports securely — cryptographic erasure via KMS key deletion is acceptable.

---

## FFIEC Guidance — Cloud Risk Management

The Federal Financial Institutions Examination Council (FFIEC) IT Handbook provides examination guidance for how bank examiners assess technology risk at regulated financial institutions. If your SaaS platform serves banks, credit unions, or other FFIEC-supervised entities, examiners may review your platform as part of their technology assessment.

**Key FFIEC expectations for cloud SaaS vendors:**
- **Vendor due diligence:** Your bank tenants must conduct vendor due diligence on your platform — provide SOC 2 Type II, penetration test results, and architectural documentation
- **Data location and sovereignty:** Document where customer data is stored and processed geographically — FFIEC examiners ask this
- **Business continuity:** Documented BCP/DRP with tested RTOs — FFIEC examiners expect bank tenants to know your RTO/RPO
- **Subcontractor oversight:** If you use sub-processors (AWS, third-party APIs), document the chain and your oversight process
- **Concentration risk:** FFIEC examiners are increasingly focused on cloud concentration risk — document your multi-region or multi-cloud resilience story

---

## OCC Third-Party Risk Guidance

The OCC Bulletin 2023-17 (and the interagency guidance it aligns with) establishes expectations for how banks manage third-party relationships including cloud SaaS vendors. If your tenants include OCC-supervised banks, they will apply this guidance to their relationship with you.

**What this means for your platform:**
- Banks will conduct due diligence before contracting and ongoing monitoring during the relationship
- You should be prepared to provide: SOC 2 Type II reports, penetration test summaries, incident response policy, BCP/DRP documentation, subprocessor list, data residency documentation
- Contracts must include: right to audit provisions, incident notification SLAs (typically 24-72 hours), data return/destruction on termination
- Critical activities (core banking functions, payment processing) face more scrutiny than non-critical ones

---

## State Laws — NY DFS Part 500 and California CCPA/CPRA

### New York — DFS Part 500 (Cybersecurity Regulation)

NY DFS Part 500 applies to entities licensed by the NY Department of Financial Services (banks, insurance companies, mortgage servicers, money transmitters). The 2023 amendments significantly strengthened requirements.

**Key architecture requirements:**
- MFA required for all privileged access to information systems (no exceptions for DFS-regulated entities)
- Encryption of nonpublic information (NPI) in transit and at rest
- Audit trails: retain for minimum 5 years and protect from unauthorized alteration or deletion
- Penetration testing: annual for Class A companies, every 3 years otherwise
- Vulnerability scanning: quarterly minimum
- CISO designation and annual cybersecurity report to DFS (Form 500 filing)
- 72-hour breach notification to DFS Superintendent

**AWS mapping:** Same controls as GLBA Safeguards, but with DFS-specific filing requirements and stricter MFA mandate.

### California — CCPA/CPRA for Financial Data

Financial data held by businesses subject to CCPA/CPRA (California Consumer Privacy Act / Privacy Rights Act) gives California consumers rights over their financial information beyond GLBA:
- Right to know what personal information is collected and how it is used
- Right to delete (with exceptions for legal retention obligations)
- Right to opt out of sale or sharing of personal information
- Right to correct inaccurate personal information

**GLBA/CCPA tension:** CCPA has a partial exemption for data already subject to GLBA — but only for the specific data elements covered by GLBA's notice and opt-out provisions. Data not covered by GLBA (e.g., browsing data, device data, inferred data) remains subject to CCPA.

**Architecture implications:**
- Build configurable consent management — different rules for GLBA-covered vs. non-covered data
- Right-to-delete workflow: customer deletion requests must propagate across all storage services (DynamoDB, S3, RDS, OpenSearch, backups)
- Data inventory is mandatory — Macie for S3, manual tagging for structured databases

---

## Shared Responsibility Model for Finance

AWS secures the infrastructure (physical security, hypervisor, global network). You secure everything else:

| Your Responsibility | AWS Responsibility |
|---|---|
| IAM configuration (least privilege, tenant isolation, MFA enforcement) | Physical security of data centers |
| Encryption configuration (KMS keys enabled, TLS enforced, key rotation) | Hypervisor security and network isolation |
| Network configuration (security groups, NACLs, VPC endpoints, WAF rules) | Hardware maintenance and disposal |
| Application security (input validation, injection prevention, auth/authz) | Global network infrastructure |
| Logging and monitoring (CloudTrail data events enabled, log retention set) | AWS service availability and patching |
| Financial data handling (PAN tokenization, PII classification, access controls) | Compliance certifications for AWS infrastructure (PCI Level 1 Service Provider, SOC 1/2, ISO 27001) |
| PCI-DSS compliance of your application (requirements 1-12 from the application layer up) | PCI-DSS compliance of AWS infrastructure (covered in AWS PCI Responsibility Summary) |

**AWS being PCI-certified does not make YOUR application PCI-compliant.** AWS's PCI compliance covers the infrastructure. You must implement the application-level controls, get your own QSA assessment, and maintain your own Attestation of Compliance (AOC) or SAQ.

---

## Common Mistakes

1. **Assuming AWS PCI certification means your app is PCI-compliant.** AWS is a PCI Level 1 Service Provider. That means the infrastructure is certified. Your application — the code, the IAM policies, the encryption configuration, the network segmentation — still requires its own QSA assessment and AOC.

2. **Storing SAD (CVV, full track data) anywhere.** Sensitive Authentication Data must NEVER be stored after authorization. Not in logs, not in DynamoDB, not in S3. Not even encrypted. This is a hard PCI requirement with no exceptions.

3. **Under-scoping the CDE.** Every service that can touch CHD or SAD is in scope. If your logging service receives logs with PAN in them, the logging service is in scope. Solve this by tokenizing at ingress and never letting PAN reach services that don't need it.

4. **Not implementing OFAC screening at transaction time.** Screening only at onboarding is insufficient. New designations occur daily. Screen at transaction initiation for payment counterparties.

5. **Treating BSA/AML as a checkbox.** FinCEN examiners look for evidence that your transaction monitoring program is tuned and effective, not just present. Log tuning decisions, false positive rates, and threshold changes as evidence of an active program.

6. **Ignoring the SOC 1 requirement.** Enterprise fintech buyers and their auditors need a SOC 1 Type II report to complete their own SOX ITGC testing. If you don't have one, you'll lose enterprise deals. Budget for it early.

7. **GLBA applies beyond banks.** Many fintech companies — lending platforms, payment processors, tax software — are "financial institutions" under GLBA's broad definition. If you collect NPI from consumers in connection with financial services, GLBA likely applies.

8. **Not documenting the permissible purpose for every credit bureau pull.** FCRA enforcement includes examination of permissible purpose logs. If you can't demonstrate every pull had a valid permissible purpose, you have an FCRA problem.

---

## Discovery Questions for This Domain

**Regulatory scope:**
- What regulations apply to your platform? (PCI-DSS? GLBA? SOX? BSA/AML?)
- Are any of your tenants FFIEC-supervised institutions (banks, credit unions)? Or OCC-chartered banks?
- Are any of your tenants licensed in New York (DFS Part 500 obligations)?
- Do you serve EU/UK customers? (Triggers GDPR, PSD2, MiFID II depending on activity)
- Do you store or process cardholder data, or do you rely on a payment processor/tokenization service to keep PAN out of your environment?

**Compliance program maturity:**
- Do you have a designated BSA Officer (if AML obligations apply)?
- Have you completed a PCI-DSS QSA assessment, or are you targeting SAQ compliance? What SAQ type?
- Do you have a SOC 2 Type II report? SOC 1 Type II?
- Have you completed a GLBA risk assessment?
- Do you have a documented incident response plan with regulatory notification SLAs?

**Third-party and vendor risk:**
- Who is your payment processor, and what is their PCI compliance scope?
- Do you use third-party AML/fraud screening services? What is their data handling posture?
- What credit bureaus do you integrate with, and are your data sharing agreements current?

**Data inventory:**
- Do you know where all customer financial information (NPI) is stored across your AWS environment?
- Have you used Macie to scan S3 for NPI/financial PII?
- Do you have a documented data retention and disposal policy?

---

## Content Freshness

| Item | Current Status | Verify At |
|---|---|---|
| PCI-DSS v4.0 | Current standard (v4.0 published March 2022; v3.2.1 retired March 2024) | https://www.pcisecuritystandards.org/ |
| GLBA Safeguards Rule | 2023 amendments effective June 9, 2023 | https://www.ftc.gov/business-guidance/resources/ftc-safeguards-rule-what-your-business-needs-know |
| CFPB Section 1033 (Open Banking) | Final rule published October 2024; compliance begins April 2026 for large institutions | https://www.consumerfinance.gov/personal-financial-data-rights/ |
| SR 26-2 (Model Risk) | 2026 revision supersedes SR 11-7 for Federal Reserve-supervised entities | https://www.federalreserve.gov/supervisionreg/srletters/SR2602.htm |
| NY DFS Part 500 | 2023 amendments effective November 2023 | https://www.dfs.ny.gov/industry_guidance/cybersecurity |
| OCC Third-Party Risk | Bulletin 2023-17 (interagency guidance) | https://www.occ.gov/news-issuances/bulletins/2023/bulletin-2023-17.html |
| OFAC SDN List | Updated frequently (sometimes multiple times per week) | https://ofac.treasury.gov/specially-designated-nationals-and-blocked-persons-list-sdn-human-readable-lists |
| AWS PCI DSS Responsibility Summary | Check for version aligned with current PCI-DSS v4.0 | https://aws.amazon.com/compliance/pci-dss-level-1-faqs/ |

**Agent rule:** When citing compliance deadlines or regulatory guidance, add: "Verify current status at [link] as this may have changed since this power was last updated."

---

## References

- [AWS Financial Services Industry Lens](https://docs.aws.amazon.com/wellarchitected/latest/financial-services-industry-lens/financial-services-industry-lens.html)
- [AWS FSI Compliance Center (US)](https://aws.amazon.com/financial-services/security-compliance/compliance-center/us/)
- [PCI-DSS v4.0 Standard](https://www.pcisecuritystandards.org/document_library/)
- [FTC Safeguards Rule — GLBA](https://www.ftc.gov/business-guidance/resources/ftc-safeguards-rule-what-your-business-needs-know)
- [SOX Section 404 — IT Controls](https://www.sec.gov/divisions/finance/verifyadequacy.htm)
- [FinCEN BSA/AML Statutes and Regulations](https://www.fincen.gov/resources/statutes-regulations/bank-secrecy-act)
- [OFAC Sanctions Programs](https://ofac.treasury.gov/)
- [FFIEC IT Handbook](https://ithandbook.ffiec.gov/)
- [OCC Third-Party Risk Guidance — Bulletin 2023-17](https://www.occ.gov/news-issuances/bulletins/2023/bulletin-2023-17.html)
- [NY DFS Part 500 Cybersecurity Regulation](https://www.dfs.ny.gov/industry_guidance/cybersecurity)
- [CFPB Personal Financial Data Rights (Section 1033)](https://www.consumerfinance.gov/personal-financial-data-rights/)
- [FCRA — Fair Credit Reporting Act](https://www.ftc.gov/legal-library/browse/statutes/fair-credit-reporting-act)
- [Federal Reserve SR 26-2 — Model Risk Management](https://www.federalreserve.gov/supervisionreg/srletters/SR2602.htm)
