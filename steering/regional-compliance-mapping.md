# Regional Compliance Mapping — US, UK & EU

## Why This File Exists

Financial services SaaS platforms often serve customers in multiple jurisdictions. The US, UK, and EU have distinct regulatory bodies, compliance frameworks, and data residency expectations. This file provides a quick-reference mapping so the agent can tailor architectural guidance based on the customer's target region — loading the right frameworks, referencing the right regulators, and flagging region-specific requirements.

**Load this file when:** The user indicates their platform operates in the UK or EU, serves UK/EU-regulated financial institutions, or needs to support US + UK + EU markets simultaneously.

---

## Region Decision Matrix

| Question | US | UK | EU |
|----------|----|----|-----|
| Who regulates banks? | OCC, Federal Reserve, FDIC | FCA (conduct), PRA (prudential) | ECB (SSM for eurozone), national competent authorities (BaFin, ACPR, DNB, etc.) |
| Data protection law | GLBA, CCPA/CPRA (state-level) | UK GDPR + Data Protection Act 2018 | EU GDPR (Regulation 2016/679) |
| Payment card security | PCI-DSS v4.0 (global) | PCI-DSS v4.0 (global) | PCI-DSS v4.0 (global) |
| Open banking standard | FDX, CFPB Section 1033 | UK Open Banking (OBIE), PSD2 UK retained | PSD2 (Directive 2015/2366), PSD3 (proposed) |
| Payment services regulation | State money transmitter licenses | FCA PSR 2017 | PSD2 + national transpositions, EMD2 |
| Anti-money laundering | BSA/AML, FinCEN, OFAC | MLR 2017, NCA, HM Treasury sanctions | AMLD6, EU AML Authority (AMLA), national FIUs, EU sanctions |
| Credit reporting | FCRA, ECOA/Reg B | Consumer Credit Act 1974, FCA CONC | Consumer Credit Directive (CCD2 proposed), national laws |
| Operational resilience | FFIEC, OCC guidance | FCA/PRA PS21/3, SS1/21 | DORA (Regulation 2022/2554) — mandatory Jan 2025 |
| AI/ML model governance | SR 11-7 / SR 26-2 | PRA SS1/23 | EU AI Act (Regulation 2024/1689), EBA guidelines |
| Cloud/outsourcing | OCC Third-Party Risk, FFIEC | FCA/PRA SS2/21 | EBA Outsourcing Guidelines, DORA Chapter V (ICT third-party risk) |
| Digital operational resilience | No direct equivalent | Influenced by DORA (for EU operations) | DORA (mandatory, applies to all EU financial entities) |

---

## US Compliance Framework — Summary

### Regulatory Bodies

| Regulator | Scope | Key Expectations for SaaS Vendors |
|-----------|-------|----------------------------------|
| **OCC** | National banks, federal savings associations | Third-party risk management (OCC Bulletin 2023-17), examiner access |
| **Federal Reserve** | State member banks, BHCs | SR 26-2 (model risk), operational resilience |
| **FDIC** | State non-member banks | Third-party risk guidance, IT examination |
| **CFPB** | Consumer financial protection | Section 1033 (open banking), fair lending |
| **SEC** | Securities, broker-dealers | Rule 17a-4 (record retention), Reg SCI |
| **FINRA** | Broker-dealer conduct | Communications supervision, trade records |
| **FinCEN** | AML/CFT | BSA compliance, SAR/CTR filing |
| **State regulators** | Money transmitters, state-chartered banks | 50-state licensing patchwork |

### Key US Frameworks (Detailed in `financial-compliance-foundations.md`)

| Framework | Trigger | What It Requires |
|-----------|---------|-----------------|
| PCI-DSS v4.0 | Card data processing | CDE isolation, tokenization, annual QSA |
| GLBA Safeguards Rule | Consumer financial data | 9-element security program |
| SOX Section 404 | Public company financial systems | ITGC controls, audit trail |
| BSA/AML | Money movement, financial institutions | Transaction monitoring, SAR/CTR, OFAC screening |
| FCRA | Consumer credit data | Permissible purpose, adverse action notices |
| ECOA/Reg B | Credit decisions | Fair lending, explainability, disparate impact testing |
| CCPA/CPRA | California consumers | Right to delete (tension with GLBA retention) |

### US Data Residency

- **No federal data localization mandate** for financial data
- Some state laws impose restrictions (NY DFS Part 500 requires notification)
- GLBA does not mandate US-only storage
- PCI-DSS does not mandate geography — only security controls
- SEC Rule 17a-4 requires records be accessible to US regulators on demand

---

## UK Compliance Framework — Summary

### Regulatory Bodies

| Regulator | Scope | Key Expectations for SaaS Vendors |
|-----------|-------|----------------------------------|
| **FCA** (Financial Conduct Authority) | Conduct regulation for all FS firms | Outsourcing notifications (SS2/21), operational resilience (PS21/3) |
| **PRA** (Prudential Regulation Authority) | Prudential regulation for banks, insurers, major investment firms | Capital adequacy, operational resilience, model risk (SS1/23) |
| **ICO** (Information Commissioner's Office) | Data protection (UK GDPR) | DPIAs, breach notification (72 hours), data subject rights |
| **PSR** (Payment Systems Regulator) | Payment systems oversight | Access, competition, innovation in payments |
| **Bank of England** | Financial stability, RTGS | Systemic risk, CHAPS, settlement systems |
| **HM Treasury** | Sanctions, financial crime policy | UK sanctions list (distinct from OFAC) |
| **NCA** (National Crime Agency) | Suspicious Activity Reports (UK SARs) | UK SAR filing (different from US FinCEN SAR) |

### Key UK Frameworks

| Framework | Trigger | What It Requires |
|-----------|---------|-----------------|
| **UK GDPR + DPA 2018** | Processing personal data of UK residents | Lawful basis, DPIAs, 72-hour breach notification, data subject rights, international transfer safeguards |
| **PSD2 (UK retained)** | Payment services | Strong Customer Authentication (SCA), TPP access, dedicated interfaces |
| **FCA Operational Resilience (PS21/3)** | All FCA/PRA-regulated firms | Identify Important Business Services (IBS), set impact tolerances, test within tolerance by March 2025 |
| **FCA/PRA Outsourcing (SS2/21)** | Material outsourcing to cloud/SaaS | Pre-notification to FCA/PRA, exit strategies, sub-outsourcing controls, audit/access rights |
| **MLR 2017** (Money Laundering Regulations) | AML/CFT obligations | Customer due diligence, UK SAR reporting, sanctions screening (HM Treasury list) |
| **Consumer Credit Act 1974** | Consumer lending | Licensing (FCA), credit agreements, default notices, unfair relationships |
| **Consumer Duty (FCA)** | All retail financial products | Good outcomes for consumers, fair value, clear communications |
| **PRA SS1/23** (Model Risk Management) | AI/ML models in regulated firms | Model inventory, validation, monitoring (UK equivalent of SR 26-2) |
| **SMCR** (Senior Managers & Certification Regime) | Individual accountability | Senior managers personally liable for failures in their area |
| **MiFID II (UK onshored)** | Investment services | Best execution, transaction reporting, record retention |
| **DORA** (if EU operations) | Digital operational resilience | ICT risk management, incident reporting, third-party oversight |

### UK Data Residency

- **UK GDPR** restricts international transfers of personal data outside the UK
- Requires "adequate" jurisdiction (US is NOT automatically adequate) or appropriate safeguards:
  - UK International Data Transfer Agreement (IDTA)
  - UK Addendum to EU SCCs
  - Binding Corporate Rules (BCRs)
- **AWS UK Regions (eu-west-2 London)** — use for UK data residency requirements
- Financial regulators (FCA/PRA) expect notification and risk assessment for offshore data processing
- PRA SS2/21 requires that cloud providers allow regulatory access to data and audit rights

---

## EU Compliance Framework — Summary

### Regulatory Bodies

| Regulator | Scope | Key Expectations for SaaS Vendors |
|-----------|-------|----------------------------------|
| **ECB / SSM** (Single Supervisory Mechanism) | Direct supervision of significant eurozone banks | ICT risk management, outsourcing oversight, operational resilience |
| **EBA** (European Banking Authority) | Banking regulation and guidelines across EU | Outsourcing Guidelines, ICT risk guidelines, model risk |
| **ESMA** (European Securities & Markets Authority) | Securities, markets, trading venues | MiFID II/MiFIR, CSDR (settlement), EMIR (derivatives) |
| **EIOPA** (European Insurance & Occupational Pensions Authority) | Insurance, pensions | Solvency II, IORP II |
| **National Competent Authorities (NCAs)** | Country-level supervision (BaFin, ACPR, DNB, CBI, etc.) | Licensing, local supervision, DORA enforcement |
| **National Data Protection Authorities** | EU GDPR enforcement per country | DPIAs, breach notification, international transfers |
| **EU AMLA** (AML Authority — from 2025) | EU-wide AML/CFT supervision | Direct supervision of high-risk obliged entities |
| **National FIUs** (Financial Intelligence Units) | Suspicious transaction reports per country | Each EU country has its own FIU for SAR equivalent |

### Key EU Frameworks

| Framework | Trigger | What It Requires |
|-----------|---------|-----------------|
| **DORA** (Digital Operational Resilience Act) | All EU financial entities + their critical ICT third-party providers | ICT risk management, incident reporting (4-hour initial notification), digital operational resilience testing, third-party risk management (Chapter V registers) |
| **EU GDPR** (Regulation 2016/679) | Processing personal data of EU residents | Lawful basis, DPIAs, 72-hour breach notification, data subject rights (access, portability, erasure), international transfer safeguards (SCCs, adequacy) |
| **PSD2** (Payment Services Directive 2) | Payment services in the EU | Strong Customer Authentication (SCA), TPP access (AISP/PISP), dedicated interfaces, eIDAS certificates, 90-day re-authentication |
| **MiFID II / MiFIR** | Investment services and trading | Best execution, transaction reporting to ARM, position limits, record retention (5 years), algorithmic trading controls |
| **AMLD6** (6th Anti-Money Laundering Directive) | AML/CFT obligations for financial institutions | Customer due diligence, beneficial ownership, suspicious transaction reporting to national FIU, EU sanctions screening |
| **EU AI Act** (Regulation 2024/1689) | AI systems in financial services | Risk classification (credit scoring = high-risk), conformity assessments, transparency, human oversight, bias testing |
| **EBA Outsourcing Guidelines** | Material outsourcing by EU credit institutions | Notification to NCA, outsourcing register, exit strategies, sub-outsourcing controls, audit rights |
| **eIDAS Regulation** | Electronic identification and trust services | Qualified certificates (QWAC, QSeal) for TPP identification under PSD2 |
| **CSDR** (Central Securities Depositories Regulation) | Securities settlement | Settlement discipline, T+1 migration (planned), CSD requirements |
| **EMIR** (European Market Infrastructure Regulation) | Derivatives trading | Central clearing, trade reporting, risk mitigation for non-cleared derivatives |
| **Consumer Credit Directive (CCD2 — proposed)** | Consumer lending in the EU | Creditworthiness assessment, pre-contractual information, right of withdrawal |

### DORA — Critical for SaaS Providers (Effective January 2025)

DORA is the most impactful EU regulation for financial services SaaS vendors. If your platform is designated as a **Critical ICT Third-Party Provider (CTPP)**, you may be subject to direct EU oversight.

**Key DORA Requirements for SaaS:**

| DORA Chapter | Requirement | Architecture Impact |
|---|---|---|
| Chapter II — ICT Risk Management | Financial entities must maintain ICT risk management frameworks | Your platform must support tenants' ICT risk assessments |
| Chapter III — Incident Reporting | Initial notification within 4 hours, intermediate within 72 hours, final within 1 month | Automated incident detection + notification pipeline required |
| Chapter IV — Resilience Testing | Threat-Led Penetration Testing (TLPT) for significant entities | Your platform must accommodate TLPT exercises |
| Chapter V — Third-Party Risk | Register of all ICT third-party arrangements, concentration risk assessment | Provide DORA-compliant contractual clauses, support audit rights |
| Chapter V — CTPP Oversight | Critical ICT providers subject to EU Lead Overseer | May require direct engagement with ESA oversight framework |

**DORA Contractual Requirements (for your tenant agreements):**

| Clause | Requirement |
|--------|-------------|
| Service descriptions | Clear, detailed description of ICT services provided |
| Data locations | Specify where data is processed and stored (and any sub-processing) |
| Exit strategies | Documented transition plan if relationship terminates |
| Audit and access | Tenants (and their regulators) must have audit access rights |
| Incident notification | Obligation to report incidents affecting tenants within agreed timelines |
| Sub-outsourcing | Notify tenants of material sub-outsourcing; tenants can object |
| Performance targets | Quantitative availability and performance SLAs |

### EU Anti-Money Laundering

| Aspect | EU |
|--------|-----|
| Directive | AMLD6 (6th Anti-Money Laundering Directive) |
| Central authority | EU AMLA (from 2025) + national FIUs |
| Filing | Suspicious Transaction Report (STR) to national FIU |
| Sanctions | EU Restrictive Measures (Council Regulations) + OFSI for countries |
| Beneficial ownership | Central BO registers per member state (public access restricted post-CJEU ruling) |
| Customer due diligence | Risk-based CDD, enhanced due diligence for high-risk third countries |
| PEPs | Politically Exposed Persons — enhanced monitoring required |

### EU Data Residency

- **EU GDPR** restricts transfers of personal data outside the EU/EEA
- Adequacy decisions: UK (adequate), US (EU-US Data Privacy Framework — adequate since July 2023, verify current status)
- Without adequacy: Standard Contractual Clauses (SCCs) + Transfer Impact Assessment (TIA)
- **AWS EU Regions:** eu-west-1 (Ireland), eu-central-1 (Frankfurt), eu-south-1 (Milan), eu-north-1 (Stockholm), eu-west-3 (Paris), eu-central-2 (Zurich)
- DORA requires knowing exactly where data is processed — document all AWS regions used
- Financial regulators (ECB/NCAs) expect data to remain accessible within EU for supervisory purposes
- Some member states have additional data localization preferences (Germany, France)

### EU vs UK — Key Differences (Post-Brexit)

| Aspect | EU | UK |
|--------|----|----|
| GDPR version | EU GDPR (Regulation 2016/679) | UK GDPR (retained EU law, diverging over time) |
| Data transfer to US | EU-US Data Privacy Framework | UK Extension to DPF |
| Open banking | PSD2 (original directive) | PSD2 UK retained (diverging, FCA future framework) |
| Operational resilience | DORA (prescriptive, ICT-focused) | PS21/3 (IBS + impact tolerances, broader scope) |
| AML supervision | AMLA (new EU-level authority) | NCA (national) |
| AI regulation | EU AI Act (binding regulation) | No equivalent law (pro-innovation approach) |
| eIDAS | eIDAS 2.0 (updated) | UK Trust Services (post-Brexit, no eIDAS) |

---

## Side-by-Side Comparison — Key Differences

### Anti-Money Laundering

| Aspect | US | UK | EU |
|--------|----|----|-----|
| Filing authority | FinCEN | NCA (National Crime Agency) | National FIUs (per member state) + EU AMLA |
| Report name | SAR (Suspicious Activity Report) | UK SAR (different format) | STR (Suspicious Transaction Report) — varies by country |
| Filing system | BSA E-Filing | NCA SAR Online | National FIU portals (goAML in many countries) |
| Threshold reporting | CTR > $10,000 | No equivalent of CTR | Varies by member state (e.g., France €1,000 for e-money) |
| Sanctions list | OFAC SDN List | HM Treasury Consolidated List + OFSI | EU Consolidated Sanctions List (Council Regulations) |
| Tipping-off prohibition | Yes (post-filing) | Yes (stricter — pre-filing under POCA s333A) | Yes (AMLD6 Article 39) |
| Consent regime | No | Yes — NCA "appropriate consent" | Varies by member state |

### Open Banking

| Aspect | US | UK | EU |
|--------|----|----|-----|
| Standard | FDX API | UK Open Banking (OBIE) | PSD2 Berlin Group / STET / Polish API |
| Mandate | CFPB Section 1033 (phased 2026-2030) | PSD2 UK (live since 2018) | PSD2 (mandatory since Sep 2019) |
| API standard | FDX (REST, OAuth 2.0) | OBIE Read/Write API (FAPI) | No single EU-wide standard — varies by market |
| TPP registration | Not required (Section 1033) | FCA registration (AISP, PISP, CBPII) | NCA registration + passporting across EU |
| Authentication | OAuth 2.0 with consent | OAuth 2.0 + FAPI | OAuth 2.0 + eIDAS certificates (QWAC/QSeal) |
| Consent duration | 90 days default (FDX) | 90-day re-auth (PSD2) | 90-day re-auth (PSD2 RTS) |
| Directory | No central directory | Open Banking Directory | No EU-wide directory (national registers) |
| Future evolution | Section 1033 expansion | FCA Future Framework | PSD3 + PSR (proposed — single regulation) |

### Operational Resilience

| Aspect | US | UK | EU |
|--------|----|----|-----|
| Framework | FFIEC IT Handbook | FCA/PRA PS21/3 | DORA (Regulation 2022/2554) |
| Key concept | Business continuity planning | Important Business Services (IBS) + Impact Tolerances | ICT risk management + resilience testing |
| Incident reporting | No mandatory timeline (varies) | FCA notification | 4 hours (initial), 72 hours (intermediate), 1 month (final) |
| Testing | DR testing, pen testing | Scenario testing within impact tolerances | TLPT (Threat-Led Penetration Testing) for significant entities |
| Third-party focus | OCC Bulletin 2023-17 | SS2/21 | DORA Chapter V — register of ICT arrangements, CTPP oversight |
| Timeline | No specific deadline | Within tolerance since March 2025 | Mandatory since January 17, 2025 |

### Credit & Lending

| Aspect | US | UK | EU |
|--------|----|----|-----|
| Primary law | FCRA, ECOA/Reg B, TILA/Reg Z | Consumer Credit Act 1974, FCA CONC | Consumer Credit Directive (CCD), CCD2 (proposed) |
| Adverse action | Specific reason codes (CFPB) | Default notices + reasons | Pre-contractual information + rejection reasons (varies) |
| Fair lending | Disparate impact testing, HMDA | FCA Consumer Duty | EBA Guidelines on creditworthiness, non-discrimination |
| Bureaus | Equifax, Experian, TransUnion (US) | Equifax, Experian, TransUnion (UK) | Varies by country (SCHUFA in DE, Banque de France, etc.) |
| AI explainability | ECOA requires specific reasons | PRA SS1/23 + ICO guidance | EU AI Act (credit scoring = high-risk AI, requires conformity assessment) |

---

## Multi-Region Architecture Patterns

### Serving US, UK, and EU Customers

```
┌────────────────────────────────────────────────────────────────────────────────────┐
│  Control Plane (Shared — hosted in primary region)                                  │
│  - Tenant management, billing, deployment orchestration                            │
│  - No financial PII in control plane                                               │
└───────────┬──────────────────────────────┬──────────────────────────┬──────────────┘
            │                              │                          │
┌───────────▼───────────────┐ ┌────────────▼────────────┐ ┌──────────▼──────────────┐
│  US Application Plane     │ │  UK Application Plane    │ │  EU Application Plane    │
│  Region: us-east-1        │ │  Region: eu-west-2       │ │  Region: eu-central-1    │
│                           │ │  (London)                │ │  (Frankfurt) or          │
│  - US tenant data         │ │                          │ │  eu-west-1 (Ireland)     │
│  - GLBA/SOX/PCI controls  │ │  - UK tenant data        │ │                          │
│  - FinCEN SAR filing      │ │  - UK GDPR/FCA/PRA       │ │  - EU tenant data        │
│  - OFAC screening         │ │  - NCA SAR filing        │ │  - EU GDPR/DORA controls │
│  - FDX open banking APIs  │ │  - HMT sanctions         │ │  - National FIU STR      │
│                           │ │  - OBIE open banking     │ │  - EU sanctions          │
│                           │ │                          │ │  - PSD2 open banking     │
└───────────────────────────┘ └──────────────────────────┘ └──────────────────────────┘
```

### Key Architecture Decisions for Multi-Region

| Decision | US | UK | EU |
|----------|----|----|-----|
| Data residency | No federal mandate | eu-west-2 (London) | EU/EEA region required (eu-central-1, eu-west-1, etc.) |
| Sanctions screening | OFAC SDN List | HM Treasury/OFSI | EU Consolidated Sanctions List |
| SAR/STR filing | FinCEN (BSA E-Filing) | NCA (SAR Online) | National FIU per member state |
| Open banking | FDX APIs | OBIE APIs | PSD2 APIs (Berlin Group/STET/national) |
| Encryption keys | US-region KMS keys | eu-west-2 KMS keys | EU-region KMS keys (per DORA: document location) |
| Consent model | Section 1033 | PSD2 UK / OBIE | PSD2 EU + eIDAS |
| Regulatory access | US regulators | FCA/PRA | ECB/NCA per country |
| Incident reporting | No mandatory timeline | FCA notification | DORA: 4 hours initial |
| AI governance | SR 26-2 (guidance) | PRA SS1/23 (guidance) | EU AI Act (binding law) |

---

## AWS Region Considerations

| Requirement | US | UK | EU |
|-------------|----|----|-----|
| Primary region | us-east-1 or us-west-2 | eu-west-2 (London) | eu-central-1 (Frankfurt) or eu-west-1 (Ireland) |
| Data residency | No federal mandate | UK GDPR requires safeguards for transfers outside UK | EU GDPR requires data stays in EU/EEA or adequate country |
| PCI-DSS eligible | All regions | All regions | All regions |
| AWS Artifact reports | US SOC 2, PCI DSS | UK Cyber Essentials, ISO 27001, SOC 2 | C5 (Germany), ISO 27001, SOC 2, ENS (Spain) |
| Amazon Payment Cryptography | Available | Available (eu-west-2) | Available (eu-central-1, eu-west-1) |
| AWS Config conformance packs | GLBA, PCI-DSS, SOX | UK GDPR, PCI-DSS | EU GDPR, PCI-DSS, DORA (check availability) |
| Key compliance certifications | FedRAMP, SOC 1/2/3 | Cyber Essentials Plus, G-Cloud | C5 (BSI), TISAX, HDS (France), AgID (Italy) |

---

## Discovery Questions — Region-Specific

**Ask early in any engagement:**
1. Which regions do your customers operate in? (US only, UK only, EU only, or multi-region?)
2. Are your tenants regulated by UK (FCA/PRA) or EU (ECB/NCAs) financial regulators?
3. Do you need to support UK Open Banking (OBIE), EU PSD2, US open banking (FDX/Section 1033), or multiple?
4. Where will financial data be stored? (US regions, UK London, EU Frankfurt/Ireland, or multiple?)
5. Do you have data residency requirements under UK GDPR or EU GDPR?

**If UK is confirmed, follow up with:**
- Have you notified FCA/PRA of your material outsourcing to cloud?
- Have you identified your Important Business Services (IBS) per PS21/3?
- Do you need UK SAR filing capability (NCA)?
- Are any tenants subject to SMCR individual accountability?

**If EU is confirmed, follow up with:**
- Are you classified (or likely to be classified) as a Critical ICT Third-Party Provider under DORA?
- Which EU member states do your tenants operate in? (Determines which NCAs and FIUs are relevant)
- Do you need to support DORA incident reporting timelines (4-hour initial notification)?
- Are your AI/ML models in scope for EU AI Act (credit scoring, fraud detection = high-risk)?
- Do you need eIDAS certificates for PSD2 TPP authentication?
- Have you completed DORA Chapter V third-party register entries for your tenants?

---

## References

### US References (see also `financial-compliance-foundations.md`)
- [AWS FSI Compliance Center (US)](https://aws.amazon.com/financial-services/security-compliance/compliance-center/us/)
- [OCC Third-Party Risk Management (Bulletin 2023-17)](https://www.occ.gov/news-issuances/bulletins/2023/bulletin-2023-17.html)
- [CFPB Section 1033](https://www.consumerfinance.gov/personal-financial-data-rights/)
- [FinCEN BSA/AML](https://fincen.gov/resources/statutes-and-regulations/bank-secrecy-act)

### UK References
- [FCA Operational Resilience (PS21/3)](https://www.fca.org.uk/publications/policy-statements/ps21-3-building-operational-resilience)
- [PRA Outsourcing and Third-Party Risk (SS2/21)](https://www.bankofengland.co.uk/prudential-regulation/publication/2021/march/outsourcing-and-third-party-risk-management-ss)
- [PRA Model Risk Management (SS1/23)](https://www.bankofengland.co.uk/prudential-regulation/publication/2023/may/model-risk-management-principles-for-banks-ss)
- [FCA Consumer Duty](https://www.fca.org.uk/firms/consumer-duty)
- [UK Open Banking Implementation Entity (OBIE)](https://www.openbanking.org.uk/)
- [ICO — UK GDPR](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/)
- [HM Treasury — Financial Sanctions](https://www.gov.uk/government/organisations/office-of-financial-sanctions-implementation)
- [NCA — Suspicious Activity Reports](https://www.nationalcrimeagency.gov.uk/what-we-do/crime-threats/money-laundering-and-illicit-finance/suspicious-activity-reports)
- [FCA Register (TPP/AISP/PISP lookup)](https://register.fca.org.uk/)

### EU References
- [DORA Regulation (2022/2554)](https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX:32022R2554)
- [EU GDPR — Full Text](https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX:32016R0679)
- [EU AI Act (2024/1689)](https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX:32024R1689)
- [PSD2 Directive (2015/2366)](https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX:32015L2366)
- [EBA Outsourcing Guidelines (EBA/GL/2019/02)](https://www.eba.europa.eu/regulation-and-policy/internal-governance/guidelines-on-outsourcing-arrangements)
- [AMLD6 (Directive 2024/1640)](https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX:32024L1640)
- [MiFID II / MiFIR (ESMA)](https://www.esma.europa.eu/policy-rules/mifid-ii-and-mifir/)
- [EDPB — International Data Transfers](https://www.edpb.europa.eu/our-work-tools/general-guidance/international-transfers_en)
- [EU Sanctions Map](https://www.sanctionsmap.eu/)
- [ECB — SSM Supervisory Priorities](https://www.bankingsupervision.europa.eu/priorities/html/index.en.html)
- [AWS EU Region — Compliance](https://aws.amazon.com/compliance/eu-data-protection/)
- [AWS C5 Attestation (Germany)](https://aws.amazon.com/compliance/bsi-c5/)
