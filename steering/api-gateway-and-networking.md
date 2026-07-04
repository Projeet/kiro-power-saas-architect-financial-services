# API Gateway & Networking

## Why This File Exists

Financial services SaaS has networking requirements beyond generic SaaS: PCI-DSS network segmentation for the CDE, mTLS for open banking TPP authentication, SWIFT connectivity patterns, PrivateLink for bank-to-platform connectivity, and WAF rules specific to financial APIs. This file covers the network architecture patterns.

---

## PCI Network Segmentation in SaaS

### CDE Isolation Architecture

PCI-DSS Requirement 1 mandates network security controls that restrict traffic between trusted and untrusted networks, and between the CDE and everything else.

```
┌─────────────────────────────────────────────────────────────────┐
│  VPC — Financial Services SaaS Platform                          │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  Public Subnets (AZ-a, AZ-b)                                │ │
│  │  - ALB / API Gateway (WAF attached)                         │ │
│  │  - CloudFront distribution                                  │ │
│  └─────────────────────────┬──────────────────────────────────┘ │
│                            │ HTTPS only                          │
│  ┌─────────────────────────┴──────────────────────────────────┐ │
│  │  Application Subnets (Private) — Non-CDE                    │ │
│  │  - Lambda / ECS (application logic)                         │ │
│  │  - Lending, analytics, admin services                       │ │
│  │  - SG: allow from ALB, deny to CDE subnets                 │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  CDE Subnets (Private Isolated) — IN SCOPE                  │ │
│  │  - Tokenization Lambda                                      │ │
│  │  - Payment processing functions                             │ │
│  │  - Amazon Payment Cryptography endpoint                     │ │
│  │  - SG: allow ONLY from API Gateway endpoint + payment proc  │ │
│  │  - NO NAT Gateway, NO internet access                       │ │
│  │  - VPC Flow Logs → S3 (Object Lock)                         │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  Data Subnets (Private Isolated)                             │ │
│  │  - RDS/Aurora, DynamoDB endpoints, ElastiCache              │ │
│  │  - SG: allow from application + CDE subnets only            │ │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### Security Group Rules for CDE

```
CDE Security Group — Inbound:
  - Port 443 from API Gateway VPC Endpoint SG (payment API paths only)
  - Port 443 from Payment Processor PrivateLink SG
  - DENY all other inbound

CDE Security Group — Outbound:
  - Port 443 to Amazon Payment Cryptography VPC Endpoint
  - Port 443 to Payment Processor PrivateLink endpoint
  - Port 443 to KMS VPC Endpoint
  - DENY all other outbound (no internet, no other subnets)
```

---

## mTLS for Open Banking (PSD2 / FDX)

### Why mTLS
PSD2 requires mutual TLS between Third-Party Providers (TPPs) and account-holding institutions. The TPP must present an eIDAS certificate during the TLS handshake, and your platform must validate it.

### API Gateway mTLS Configuration

```typescript
// CDK — Custom domain with mTLS for open banking API
const openBankingDomain = new apigateway.DomainName(this, 'OpenBankingDomain', {
  domainName: 'openbanking.platform.com',
  certificate: acmCertificate,
  mutualTlsAuthentication: {
    truststoreUri: `s3://${trustStoreBucket.bucketName}/truststore.pem`,
    truststoreVersion: 'v1', // Version the truststore for rotation
  },
});
```

**Truststore management:**
- Store trusted CA certificates (eIDAS-issuing CAs for PSD2, or FDX-recognized CAs) in S3
- Update truststore when new CAs are added or old CAs are revoked
- API Gateway validates the client certificate against the truststore on every request
- Lambda authorizer can extract TPP identity from the certificate (subject DN, serial number) for additional validation

### Certificate Validation Chain
```
TPP presents client certificate in TLS handshake
    → API Gateway validates: cert signed by trusted CA (in truststore)
    → API Gateway validates: cert not expired
    → API Gateway validates: cert not revoked (OCSP/CRL — configure in truststore)
    → Lambda authorizer: extract TPP ID from cert, verify TPP registration status
    → Lambda authorizer: check consent + rate limits for this TPP
    → Request proceeds to backend (or 403 if any check fails)
```

---

## PrivateLink for Enterprise Bank Connectivity

### When to Use PrivateLink
When enterprise bank tenants need to connect their internal systems to your SaaS platform without traversing the public internet:
- Bank's core banking system → Your platform's payment APIs
- Bank's AML system → Your platform's transaction monitoring APIs
- Bank's data warehouse → Your platform's reporting APIs

### Architecture
```
Bank's VPC (their AWS account)       Your Platform VPC
┌─────────────────────┐             ┌─────────────────────┐
│  Bank Application    │             │  NLB (internal)      │
│       ↓              │             │       ↓              │
│  VPC Endpoint        │─PrivateLink─│  Endpoint Service    │
│  (Interface)         │             │       ↓              │
└─────────────────────┘             │  Your API (ECS/Lambda)│
                                     └─────────────────────┘
```

**Multi-tenant PrivateLink:**
- Each enterprise bank tenant that requires PrivateLink gets a separate VPC Endpoint Service (or a shared one with endpoint-level access policies)
- The bank's VPC endpoint connects to YOUR NLB → your API backend
- Traffic never leaves the AWS network — no internet traversal
- Bank's compliance team can verify: "traffic stays on AWS backbone" for their FFIEC assessment

---

## WAF Rules for Financial APIs

### Standard Financial API Protection
```typescript
const waf = new wafv2.CfnWebACL(this, 'FinancialApiWaf', {
  scope: 'REGIONAL',
  defaultAction: { allow: {} },
  rules: [
    // Rate limiting per IP
    { name: 'RateLimit', priority: 1,
      statement: { rateBasedStatement: { limit: 2000, aggregateKeyType: 'IP' } },
      action: { block: {} } },
    // AWS Managed Rules — Core
    { name: 'AWSManagedRulesCore', priority: 2,
      statement: { managedRuleGroupStatement: { vendorName: 'AWS', name: 'AWSManagedRulesCommonRuleSet' } },
      overrideAction: { none: {} } },
    // AWS Managed Rules — Known Bad Inputs
    { name: 'AWSManagedRulesKnownBadInputs', priority: 3,
      statement: { managedRuleGroupStatement: { vendorName: 'AWS', name: 'AWSManagedRulesKnownBadInputsRuleSet' } },
      overrideAction: { none: {} } },
    // AWS Managed Rules — SQL Injection
    { name: 'AWSManagedRulesSQLi', priority: 4,
      statement: { managedRuleGroupStatement: { vendorName: 'AWS', name: 'AWSManagedRulesSQLiRuleSet' } },
      overrideAction: { none: {} } },
    // Geo-blocking (if required by OFAC sanctions)
    { name: 'GeoBlock', priority: 5,
      statement: { geoMatchStatement: { countryCodes: ['CU', 'IR', 'KP', 'SY', 'RU'] } },
      action: { block: {} } },
  ],
});
```

### Financial-Specific WAF Considerations
- **Geo-blocking for OFAC:** Block requests from comprehensively sanctioned countries (Cuba, Iran, North Korea, Syria — verify current OFAC list)
- **Rate limiting per TPP:** For open banking APIs, rate limit per TPP (use custom header or certificate fingerprint as key)
- **Bot protection:** Financial APIs are targets for credential stuffing, account enumeration, and card testing attacks
- **PCI Requirement 6.4:** WAF is required in front of all public-facing CDE applications

---

## SWIFT Connectivity Patterns

### SWIFT Alliance Lite2 on AWS
For institutions that need SWIFT connectivity from their AWS environment:
- SWIFT Alliance Lite2: cloud-based SWIFT access (approved by SWIFT for AWS deployment)
- Deploy in a dedicated VPC subnet with strict network controls
- SWIFT CSP (Customer Security Programme) controls apply to the hosting environment
- Multi-tenant: each tenant that needs SWIFT connectivity typically has their own SWIFT BIC and Alliance Lite2 instance (silo)

### Alternative: SWIFT Service Bureau
If your SaaS platform serves many institutions needing SWIFT:
- Partner with a SWIFT Service Bureau
- Your platform sends payment instructions to the Service Bureau via API
- Service Bureau handles SWIFT message formatting and delivery
- Reduces your direct SWIFT infrastructure footprint

---

## Common Mistakes

1. **CDE subnets with NAT Gateway.** If CDE components can reach the internet, your PCI scope expands massively. CDE subnets should have NO internet access — only VPC endpoints for AWS services and PrivateLink for payment processors.

2. **No WAF on public-facing financial APIs.** PCI Requirement 6.4 requires a WAF (or equivalent) in front of public-facing web applications in the CDE. Financial APIs are high-value targets — WAF is mandatory.

3. **mTLS truststore not versioned.** When you update the truststore (add/remove CA certificates), version it in S3. If a bad truststore is deployed, you need to roll back quickly.

4. **PrivateLink without per-tenant access policies.** A shared VPC Endpoint Service without endpoint-level resource policies means any connected VPC can reach any tenant's API. Add endpoint policies that scope access to the connecting tenant.

5. **OFAC countries not geo-blocked at WAF.** If your platform processes payments, requests from comprehensively sanctioned jurisdictions should be blocked at the edge before they consume compute resources.

---

## Discovery Questions for This Domain

**Network architecture:**
- Do you have CDE network segmentation today? (Separate subnets, strict SGs, no internet access?)
- How do enterprise bank tenants connect to your platform? (Public internet, VPN, PrivateLink?)
- Do you have a WAF in front of all public-facing APIs?

**Open banking:**
- Do you need mTLS for TPP authentication? (PSD2 requirement)
- How many TPPs connect to your platform? (Determines rate limiting strategy)
- Do you validate TPP certificates against eIDAS CAs?

**Connectivity:**
- Do any tenants require SWIFT connectivity? (Direct Alliance Lite2 or via Service Bureau?)
- Do you need Direct Connect for high-throughput data exchange with enterprise tenants?
- Are there geographic restrictions on where traffic can originate? (OFAC, data residency)

---

## References

- [PCI-DSS Requirement 1 — Network Security Controls](https://www.pcisecuritystandards.org/document_library/)
- [AWS API Gateway — Mutual TLS](https://docs.aws.amazon.com/apigateway/latest/developerguide/rest-api-mutual-tls.html)
- [AWS PrivateLink](https://docs.aws.amazon.com/vpc/latest/privatelink/what-is-privatelink.html)
- [AWS WAF](https://docs.aws.amazon.com/waf/latest/developerguide/what-is-aws-waf.html)
- [SWIFT on AWS](https://aws.amazon.com/financial-services/swift/)
- [AWS Financial Services — Network Security](https://aws.amazon.com/financial-services/security-compliance/)
