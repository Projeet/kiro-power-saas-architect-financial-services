# Financial PII and Cardholder Data Handling

## Financial Data Classification

Financial data spans multiple sensitivity tiers governed by different regulations. Understanding which data type triggers which regulation is the first architectural decision.

| Data Type | Examples | Governing Regulation | Can Be Stored? | Storage Requirement |
|---|---|---|---|---|
| Primary Account Number (PAN) | 16-digit card number | PCI-DSS Req. 3 | Yes, with strong cryptography | AES-256 via KMS or tokenized |
| Sensitive Authentication Data (SAD) | CVV/CVC, PIN, full magnetic stripe, EMV chip data | PCI-DSS Req. 3 | **NEVER after authorization** | Must be purged immediately |
| Account Numbers | Bank account, routing number | GLBA, state laws | Yes, encrypted | KMS encryption, access control |
| Social Security Numbers (SSN) | Full or last-4 | GLBA, FCRA, state laws | Yes, encrypted | Per-tenant KMS CMK, Macie scanning |
| Date of Birth | Full DOB | GLBA, CCPA | Yes, with access control | Encrypted, limited access |
| Credit Scores / Reports | FICO, VantageScore, bureau tradelines | FCRA | Yes, for permissible purpose | Encrypted, permissible purpose logged |
| Transaction Data | Payment amounts, merchant, timestamp, location | GLBA, PCI-DSS (if card) | Yes | Encrypted at rest, immutable for AML |
| Income / Employment | Salary, employer, income verification | GLBA | Yes | Encrypted, access-controlled |
| Tax Information | W-2, 1099, AGI, tax returns | IRS regulations, state laws | Yes | Strongly encrypted, strict access |
| Investment Holdings | Portfolio positions, trade history | GLBA, SEC regulations | Yes | Encrypted, SOX-relevant if public co. |
| Insurance Policy Data | Coverage, claims, premiums | State insurance regulations | Yes | Encrypted, state-specific retention |

### What People Forget Is Financial PII

- **Tokenized data with a vault reference** — the token itself is not sensitive, but the vault that maps token → PAN is in PCI scope and must be treated as CHD
- **Log entries with account numbers** — if your Lambda writes `Processing account 4111-xxxx-xxxx-xxxx` to CloudWatch, those logs are in PCI scope
- **AI-generated outputs** — credit memos, loan summaries, fraud narratives generated from financial PII are themselves financial PII; see `genai-and-financial-data.md`
- **Derived financial data** — a credit risk score derived from a consumer report is subject to FCRA even if the underlying bureau data is not stored
- **Analytics aggregates that can be reverse-engineered** — if a cohort is small enough to identify individuals, aggregated financial data may still be PII

---

## PCI-DSS CDE Scoping — Minimizing Your Attack Surface

The Cardholder Data Environment (CDE) is the system components, people, and processes that store, process, or transmit cardholder data or sensitive authentication data, and all system components that are connected to or could impact the security of the CDE.

### Scope Minimization Strategies

**Strategy 1: Hosted Payment Fields (Most Effective)**
Embed a payment field iframe hosted by your payment processor (Stripe Elements, Braintree Drop-in UI, Spreedly iFrame). The PAN is entered directly into the processor's domain and never touches your servers. Your entire application is out of PCI scope for card capture.

**Strategy 2: Network Segmentation**
If PAN does traverse your systems (e.g., you're building a payment gateway or acquirer platform):
- Deploy CDE components in a dedicated VPC or dedicated subnets with strict security groups
- No connectivity between CDE subnets and non-CDE subnets except explicitly required flows
- VPC Flow Logs on CDE VPCs stored in a separate S3 bucket with Object Lock
- WAF in front of all CDE-accessible endpoints

**Strategy 3: Tokenize at Ingress**
Accept PAN at the edge (API Gateway), immediately tokenize via Amazon Payment Cryptography or a third-party token vault, and never forward PAN downstream. All downstream services see only the token — they are out of PCI scope.

```
Client → API Gateway (TLS 1.3) → Lambda Tokenization Function
                                           ↓
                              Amazon Payment Cryptography (PAN → Token)
                                           ↓
                              DynamoDB (stores Token, not PAN) ← OUT of CDE scope
```

**Strategy 4: P2PE (Point-to-Point Encryption)**
For physical card acceptance (if applicable), use a PCI-validated P2PE solution. The PAN is encrypted at the hardware terminal and decrypted only at the payment processor's HSM — your systems never see the PAN.

### Multi-Tenant CDE Architecture

In a multi-tenant payment SaaS:
- The tokenization service and payment processing microservice ARE in scope — they handle PAN
- The CDE must be network-isolated from the rest of the application plane
- Per-tenant tokens: token namespace is tenant-scoped so tokens from Tenant A cannot be used to look up Tenant B's PANs
- The token vault (Amazon Payment Cryptography key configuration) must enforce tenant scoping at the key level


---

## Tokenization Patterns on AWS

### Amazon Payment Cryptography

Amazon Payment Cryptography is AWS's managed payment HSM service, PCI-certified at the HSM level. It handles key management for payment cryptography operations and supports tokenization, PIN translation, and card verification.

**Use Amazon Payment Cryptography for:**
- PAN tokenization (format-preserving or non-format-preserving)
- PIN block translation between payment networks
- CVV/CVC generation and verification
- 3DS authentication cryptograms
- Key injection for HSM-to-HSM key exchange

**How tokenization works with Amazon Payment Cryptography:**
1. PAN arrives at your tokenization Lambda (in the CDE subnet)
2. Lambda calls Amazon Payment Cryptography `GenerateCardValidationData` or a token generation operation
3. Amazon Payment Cryptography returns a token — the PAN-to-token mapping is managed by the service
4. Store the token in your application database (out of PCI scope)
5. To use the PAN for a payment: pass the token to Amazon Payment Cryptography, which exchanges it for the PAN when communicating with the payment network

**CDK example — Amazon Payment Cryptography tokenization Lambda in CDE subnet:**
```typescript
const tokenizationFn = new lambda.Function(this, 'TokenizationFn', {
  runtime: lambda.Runtime.NODEJS_20_X,
  handler: 'index.handler',
  code: lambda.Code.fromAsset('src/tokenization'),
  vpc: cdeVpc,
  vpcSubnets: { subnetType: ec2.SubnetType.PRIVATE_ISOLATED }, // CDE subnet — no internet access
  environment: {
    PAYMENT_CRYPTO_KEY_ARN: paymentCryptoKeyArn,
    // No PAN ever written to environment variables
  },
  // No PHI/PAN in Lambda logs — structured logging with token IDs only
});

// CDE security group — restrict inbound to API Gateway private endpoint only
tokenizationFn.connections.allowFrom(apiGwEndpoint, ec2.Port.tcp(443));
```

### Third-Party Token Vaults

If not using Amazon Payment Cryptography, validated third-party token vaults (Spreedly, VGS, Very Good Security) can also de-scope PAN from your application. These services proxy payment data — your application sends PAN to their vault and receives a token, and the vault handles PCI compliance for storage.

**When to choose third-party vault over Amazon Payment Cryptography:**
- You need multi-network token portability (Spreedly supports 200+ payment gateways)
- You need vaulted data beyond PAN (ACH account numbers, CVVs for recurring payments)
- You prefer a SaaS model with the vendor's AOC rather than managing your own PCI assessment

---

## PAN Masking and Format-Preserving Encryption

### PAN Masking

When PAN must be displayed (e.g., in a customer portal showing saved cards), display only the last 4 digits: `•••• •••• •••• 4242`. Never display more than the first 6 and last 4 digits — PCI-DSS Requirement 3.3.1.

**Implementation:** Store the masked representation (last 4 + card brand) alongside the token. Display the masked form. Never construct the full PAN for display.

### Format-Preserving Encryption (FPE) / Format-Preserving Tokenization

FPE produces a token that looks like a PAN (16 digits, passes Luhn check) but is cryptographically derived from the real PAN. Useful when downstream systems validate PAN format.

Amazon Payment Cryptography supports FPE via the FF3-1 (NIST SP 800-38G) algorithm.

**When to use FPE vs. random tokenization:**
- FPE: legacy systems that validate PAN format, systems that can't be updated to accept opaque tokens
- Random tokenization: new systems, preferred when format compatibility is not required (stronger security — no structural relationship between PAN and token)

---

## KMS Key Strategy for Financial Data

### Per-Tenant CMK Architecture

For financial services SaaS, use per-tenant KMS Customer Managed Keys (CMKs) for any data store containing financial PII (account numbers, SSNs, income data) or transaction data. This is the finance default, mirroring the healthcare power's per-tenant CMK approach for PHI.

**Why per-tenant keys:**
- Cryptographic isolation: even with an application bug, Tenant A's CMK cannot decrypt Tenant B's data
- Regulatory isolation: when a tenant churns, schedule key deletion — their data becomes permanently inaccessible (cryptographic erasure)
- Audit trail: CloudTrail KMS Decrypt events show which tenant's data was accessed and by whom
- Compliance story: "each tenant's financial data is encrypted with a key only they (and authorized services) can use"

**Key hierarchy for financial SaaS:**

```
AWS KMS
├── Tenant A CMK (alias: alias/tenant-a-financial-pii)
│   └── Used for: DynamoDB tables, S3 buckets, RDS instances holding Tenant A's financial data
├── Tenant B CMK (alias: alias/tenant-b-financial-pii)
│   └── Used for: Tenant B's financial data stores
├── CDE CMK (alias: alias/cde-payment-data)
│   └── Used for: any residual CHD encrypted storage in the CDE (PCI-scoped key)
└── Audit Log CMK (alias: alias/audit-logs)
    └── Used for: CloudTrail, application audit logs — NOT a tenant key — operational key
```

**CDK example — per-tenant KMS CMK for financial PII:**
```typescript
const tenantFinancialKey = new kms.Key(this, `TenantKey-${tenantId}`, {
  alias: `alias/tenant-${tenantId}-financial-pii`,
  description: `Financial PII encryption key for tenant ${tenantId}`,
  enableKeyRotation: true, // Automatic annual rotation — PCI Req. 3.7.1
  removalPolicy: RemovalPolicy.RETAIN, // Never auto-delete financial data keys
  keySpec: kms.KeySpec.SYMMETRIC_DEFAULT,
  keyUsage: kms.KeyUsage.ENCRYPT_DECRYPT,
});

// Restrict key usage to the tenant's application role only
tenantFinancialKey.addToResourcePolicy(new iam.PolicyStatement({
  sid: 'AllowTenantApplicationRole',
  effect: iam.Effect.ALLOW,
  principals: [new iam.ArnPrincipal(tenantAppRoleArn)],
  actions: ['kms:Decrypt', 'kms:Encrypt', 'kms:GenerateDataKey', 'kms:DescribeKey'],
  resources: ['*'],
  conditions: {
    StringEquals: { 'kms:CallerAccount': Stack.of(this).account },
  },
}));

// Explicit deny for all other principals — belt and suspenders
tenantFinancialKey.addToResourcePolicy(new iam.PolicyStatement({
  sid: 'DenyOtherPrincipals',
  effect: iam.Effect.DENY,
  principals: [new iam.AnyPrincipal()],
  actions: ['kms:Decrypt', 'kms:Encrypt', 'kms:GenerateDataKey'],
  resources: ['*'],
  conditions: {
    ArnNotEquals: { 'aws:PrincipalArn': tenantAppRoleArn },
    StringNotEquals: { 'aws:PrincipalAccount': Stack.of(this).account },
  },
}));
```

### Key Rotation Policy

PCI-DSS Requirement 3.7 mandates cryptographic key management procedures including key rotation. AWS KMS automatic key rotation rotates the key material annually while preserving the key ID — data encrypted with old material can still be decrypted. For CDE keys, annual rotation satisfies PCI Requirement 3.7.1.

### Cryptographic Erasure for Tenant Offboarding

When a tenant churns:
1. Export any data the tenant is entitled to take with them (data portability)
2. Block access (disable IAM role, revoke Cognito credentials)
3. Start data retention countdown (per your contractual and regulatory retention obligations)
4. After retention period: schedule KMS key deletion (7-30 day waiting period is mandatory)
5. After key deletion: all data encrypted with that key is permanently inaccessible — cryptographic erasure complete
6. Document the erasure event in your audit trail

**Important:** Do NOT schedule key deletion before your retention obligation expires. GLBA requires 5 years for certain records. Some state laws and SOX require longer. Gate the key deletion on the retention policy, not the offboarding date.

---

## Encryption at Rest — Per Service

| Service | How to Enable | Finance Key Strategy | Notes |
|---|---|---|---|
| S3 | SSE-KMS, per-bucket | Per-tenant CMK | Block all public access. Enable versioning for financial records. Object Lock for audit-required records. |
| DynamoDB | Table-level KMS encryption | Per-tenant CMK (silo/bridge) or AWS-managed (pool with IAM isolation) | Enable PITR. For transaction data: append-only, never overwrite. |
| RDS/Aurora | Instance-level encryption at creation | Per-tenant CMK (silo/bridge) | Cannot add encryption after creation — must snapshot/restore. Force SSL. |
| OpenSearch | Domain-level + node-to-node encryption | Per-domain CMK | Used for transaction search, fraud alert queues, AML case management. |
| ElastiCache (Redis) | Cluster-level encryption at rest + in-transit | Per-cluster CMK | Never cache raw financial PII in Memcached — no encryption support. Redis only. |
| EBS | Volume-level encryption | CMK or AWS-managed | Set account default to encrypt all new volumes. |
| Kinesis Data Streams | Stream-level encryption | CMK for streams carrying financial PII or transaction data | Per-tenant streams if transaction data is tenant-specific. |
| SQS | Queue-level encryption | CMK for queues carrying payment events or financial PII | SSE-SQS uses AWS-managed key — use SSE-KMS for financial data queues. |
| Secrets Manager | Secret-level encryption | AWS-managed or CMK | Use for all database credentials, API keys, payment processor secrets. |

## Encryption in Transit

- TLS 1.2+ everywhere. TLS 1.3 preferred for new systems. No exceptions.
- ALB: HTTPS only, redirect HTTP → HTTPS, enforce ELBSecurityPolicy-TLS13-1-2-2021-06 or equivalent
- API Gateway: minimum TLS 1.2 — enforce via custom domain settings
- Database connections: `rds.force_ssl = 1` for MySQL/MariaDB, `require_secure_transport = ON` for Aurora MySQL, `rds.force_ssl` for PostgreSQL
- Inter-service: use VPC endpoints for all AWS service calls — keeps traffic off the public internet
- Open banking (FDX/PSD2): mTLS required for TPP authentication — see `api-gateway-and-networking.md`

---

## Macie for Financial PII Detection

Amazon Macie discovers and classifies sensitive data in S3 using managed data identifiers. For financial services SaaS, configure Macie to scan for:

**Relevant Macie managed data identifiers:**
- `CREDIT_CARD_NUMBER` — PAN detection in S3 objects
- `US_SOCIAL_SECURITY_NUMBER` — SSN detection
- `US_BANK_ACCOUNT_NUMBER` — ACH account numbers
- `US_BANK_ROUTING_NUMBER` — ABA routing numbers
- `US_DRIVERS_LICENSE` — for KYC document storage buckets
- `US_PASSPORT_NUMBER` — for KYC documents
- `US_TAX_IDENTIFICATION_NUMBER` — EIN/TIN detection

**Use Macie for:**
1. **Continuous monitoring:** Alert when PAN or SSN appears in an S3 bucket that should only contain non-sensitive data (e.g., log buckets, analytics buckets)
2. **Compliance evidence:** Generate periodic Macie findings reports for PCI-DSS and GLBA auditors as evidence of data discovery controls
3. **Data inventory:** Map where financial PII lives across your S3 estate — required for GLBA data inventory element

**EventBridge + SNS pattern for Macie alerting:**
```
Macie Finding (HIGH severity, CREDIT_CARD_NUMBER in log bucket)
    → EventBridge rule
    → SNS topic → Security team PagerDuty
    → Lambda → create Security Hub finding
    → Lambda → trigger automated remediation (block public access, quarantine bucket)
```

---

## De-identification for Analytics and ML Pipelines

Financial data used for analytics, ML model training, or development environments must be de-identified to remove the regulatory obligations.

### Techniques

**Tokenization (Reversible)**
Replace financial PII with a token that maps back to the original via a controlled vault. Use for: analytics pipelines that might need to re-link to a customer record, pre-production environments that need realistic-looking account numbers.

**Pseudonymization (Reversible with Key)**
Replace customer identifiers with pseudonyms using a keyed hash. Pseudonymized data is still personal data under GDPR/CCPA — it only reduces the regulatory burden, it does not eliminate it.

**Data Masking / Redaction (Irreversible)**
Replace sensitive fields with masked values (e.g., last 4 digits of SSN: `•••-••-6789`). Use for: customer-facing displays, read-only analytics where you don't need the full value, QA environments.

**Synthetic Data Generation (Irreversible)**
Generate statistically representative financial data that contains no real customer records. Use for: ML model training, load testing, demo environments. AWS offers SageMaker Data Wrangler for synthetic data generation.

### GLBA/CCPA Retention vs. Right to Erasure Tension

GLBA requires retention of certain records for 5 years. CCPA/GDPR allows consumers to request deletion. For financial SaaS this creates a tension:

- **Resolution:** GLBA/regulatory retention obligations generally supersede CCPA deletion requests for data required by law to be retained. Document this in your Privacy Policy and consumer-facing disclosures.
- **Architecture:** Implement tiered storage — retain the legally-required minimum data set in a compliance archive (S3 Object Lock, Compliance Mode, 5-year minimum). Delete non-required data in response to CCPA requests. The compliance archive is not subject to CCPA deletion.
- **Be explicit:** When you deny a deletion request due to legal retention, tell the consumer which law requires retention and for how long.

---

## Common Mistakes

1. **SAD in logs or databases.** CVV, full magnetic stripe data, and EMV chip data must NEVER be stored after authorization. This is the most common PCI-DSS Requirement 3 violation. Scan all data stores and log pipelines for CVV patterns — a 3-4 digit number adjacent to a PAN pattern in a log is a finding.

2. **Shared KMS keys across tenants.** Per-tenant CMKs are the finance default. Shared keys mean one tenant's key compromise affects all tenants. More importantly, shared keys weaken your compliance story for SOX and PCI auditors.

3. **PAN in CloudWatch Logs.** Lambda functions that log request/response bodies can accidentally log PAN. Use structured logging with token IDs only. Scan existing log groups with Macie or Comprehend.

4. **Not rotating encryption keys.** PCI-DSS Requirement 3.7.1 requires periodic key rotation. AWS KMS automatic rotation handles this — but you must enable it explicitly; it is not on by default.

5. **Treating tokenized data as out of scope without verification.** A token vault in your own infrastructure (not Amazon Payment Cryptography) is still in PCI scope. Only tokens managed by a PCI-certified vault where you have NO access to the underlying PAN mapping are truly out of scope.

6. **Forgetting that AI model outputs are financial PII.** A credit score, adverse action narrative, or fraud risk explanation generated from consumer financial data is itself subject to FCRA/GLBA/CCPA. Apply the same access controls and retention policies to AI-generated outputs as to the source data.

7. **No Macie on log buckets and analytics buckets.** PAN frequently ends up in unexpected places via debug logging or ETL pipeline bugs. Macie on ALL S3 buckets — not just the ones you think hold sensitive data — is the control that catches this.

---

## Discovery Questions for This Domain

**Data classification:**
- What types of financial data does your system handle? (PAN, ACH account numbers, SSNs, credit reports, transaction data, investment holdings?)
- Do you store cardholder data (PAN) anywhere in your system, or does all card data go directly to a payment processor?
- Do you store Sensitive Authentication Data (CVV, track data) anywhere? (If yes: this must stop — SAD cannot be stored post-authorization)

**Encryption:**
- Is encryption at rest enabled on all data stores?
- Are you using per-tenant KMS CMKs or shared keys for financial PII?
- Is key rotation enabled on all CMKs?
- Is TLS enforced on all database connections (not just default, but enforced at the DB level)?

**Tokenization:**
- How do you handle PAN? Hosted fields, iframe, direct capture?
- What tokenization service do you use? Amazon Payment Cryptography, a third-party vault, or your own implementation?
- Is your token vault in-scope for PCI, and what is its PCI certification status?

**Macie and discovery:**
- Have you run Macie across your full S3 estate to discover where financial PII lives?
- Do you have alerts for PAN or SSN appearing in unexpected S3 buckets (log buckets, analytics buckets)?

**Data lifecycle:**
- What is your retention policy for financial PII, transaction data, and credit bureau data?
- Have you addressed the GLBA/CCPA tension — how do you handle consumer deletion requests for data you're required by law to retain?
- Do you use cryptographic erasure (KMS key deletion) for tenant offboarding? If so, is it gated by your retention obligations?

---

## References

- [PCI-DSS v4.0 — Requirement 3 (Protect Stored Account Data)](https://www.pcisecuritystandards.org/document_library/)
- [Amazon Payment Cryptography Developer Guide](https://docs.aws.amazon.com/payment-cryptography/latest/userguide/what-is-payment-cryptography.html)
- [AWS KMS Key Management Best Practices](https://docs.aws.amazon.com/kms/latest/developerguide/best-practices.html)
- [Amazon Macie — Financial Data Identifiers](https://docs.aws.amazon.com/macie/latest/user/managed-data-identifiers.html)
- [FTC Safeguards Rule — Data Inventory and Classification](https://www.ftc.gov/business-guidance/resources/ftc-safeguards-rule-what-your-business-needs-know)
- [GLBA — Privacy of Consumer Financial Information](https://www.ftc.gov/business-guidance/resources/how-comply-privacy-consumer-financial-information-rule-gramm-leach-bliley-act)
- [CFPB — Protecting Consumer Financial Data](https://www.consumerfinance.gov/data-research/consumer-complaints/)
- [AWS Compliance — PCI DSS](https://aws.amazon.com/compliance/pci-dss-level-1-faqs/)
- [Building Tokenization Solutions on AWS](https://aws.amazon.com/solutions/financial-services/payment-tokenization/)
