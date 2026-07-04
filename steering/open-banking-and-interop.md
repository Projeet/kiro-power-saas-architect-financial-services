# Open Banking & Interoperability

## Why This File Exists

Open banking is to financial services what FHIR is to healthcare — the regulatory and technical framework for data interoperability between institutions. If your SaaS platform holds consumer financial data, you will be required to share it via standardized APIs under consumer direction. This file covers the standards, consent models, and architecture patterns for open banking on AWS in a multi-tenant SaaS context.

---

## FDX API (US/Canada Open Banking Standard)

The Financial Data Exchange (FDX) is the US industry standard for consumer-permissioned financial data sharing. Over 62 million consumer accounts are already connected via FDX.

### FDX Data Categories
| Category | What's Included | API Endpoint Pattern |
|---|---|---|
| Account Details | Account type, status, nickname, institution info | `/accounts/{accountId}` |
| Account Balances | Current, available, ledger balances | `/accounts/{accountId}/balances` |
| Transactions | Transaction history, amounts, merchants, categories | `/accounts/{accountId}/transactions` |
| Statements | PDF statements, statement periods | `/accounts/{accountId}/statements` |
| Contact Info | Name, address, phone, email | `/accounts/{accountId}/contact` |
| Payment Networks | ACH routing, card info (masked) | `/accounts/{accountId}/payment-networks` |
| Tax Forms | 1099s, W-2s | `/tax-forms` |
| Rewards | Points, miles, cashback | `/accounts/{accountId}/rewards` |

### FDX Consent Model
- **Consumer-directed:** The consumer (account holder) explicitly grants permission
- **Scope-limited:** Consent is for specific data categories, not blanket access
- **Time-limited:** Consent has an expiration (default 90 days, renewable up to 1 year)
- **Revocable:** Consumer can revoke at any time — takes effect immediately
- **Minimization:** Only data needed for the stated purpose should be shared

### FDX Authorization (OAuth 2.0 + OpenID Connect)
```
Consumer → Data Recipient (TPP) → "Connect my bank account"
    ↓
Consumer redirected to Data Provider (your platform) consent screen
    ↓
Consumer authenticates with their FI credentials (SCA applies)
    ↓
Consumer reviews and approves data sharing scope
    ↓
Authorization code returned to Data Recipient
    ↓
Data Recipient exchanges code for access + refresh tokens
    ↓
Data Recipient calls FDX APIs with access token (scoped to consent)
```

---

## CFPB Section 1033 — Personal Financial Data Rights

The Consumer Financial Protection Bureau's Personal Financial Data Rights Rule (Section 1033 of Dodd-Frank) mandates that covered data providers make consumer financial data available to consumers and their authorized third parties via standardized interfaces.

### Who Must Comply
- Banks and credit unions
- Card issuers
- Payment facilitators and digital wallets
- Any entity that controls or possesses consumer financial data covered by the rule

### Compliance Deadlines (Phased)
| Tier | Entity Size | Compliance Deadline |
|---|---|---|
| Tier 1 | Largest depository institutions (assets > $250B) and non-depository | April 1, 2026 |
| Tier 2 | Depository institutions $10B–$250B assets | April 1, 2027 |
| Tier 3 | Depository institutions $3B–$10B assets | April 1, 2028 |
| Tier 4 | Depository institutions $1.5B–$3B assets | April 1, 2029 |
| Tier 5 | Depository institutions $850M–$1.5B assets | April 1, 2030 |

**Verify current deadlines at:** https://www.consumerfinance.gov/personal-financial-data-rights/

### What Must Be Available
- Transaction information (at least 24 months of history)
- Account balance
- Information needed to initiate payments from the account
- Terms and conditions
- Upcoming bill information
- Basic account verification information

### Screen-Scraping Prohibition
Section 1033 is explicitly designed to end screen-scraping. Once a data provider offers a compliant developer interface (API), authorized third parties must use the API — not screen-scraping credentials.

### Architecture Implication for SaaS
If your platform is the data provider (you hold consumer financial data for your FI tenants), you must expose Section 1033-compliant APIs on behalf of each tenant. This is a multi-tenant API — each tenant's consumers see only their own data, and each authorized TPP accesses only the data they've been consented to see.

---

## PSD2 / Open Banking (EU/UK)

The Payment Services Directive 2 (PSD2) mandates open banking in the EU/UK. It requires banks to provide APIs to licensed Third-Party Providers (TPPs).

### TPP Types
| Type | Abbreviation | What They Do | Regulatory Status |
|---|---|---|---|
| Account Information Service Provider | AISP | Read account data (balances, transactions) | Licensed by NCA |
| Payment Initiation Service Provider | PISP | Initiate payments from the consumer's account | Licensed by NCA |
| Card-Based Payment Instrument Issuer | CBPII | Confirm funds availability | Licensed by NCA |

### Key Requirements
- **Strong Customer Authentication (SCA):** Required when consumer accesses account or initiates payment (see `identity-and-onboarding.md`)
- **eIDAS Certificates:** TPPs must identify themselves with qualified certificates (QWAC for transport, QSeal for signing)
- **Dynamic Linking:** For payment initiation, SCA must be dynamically linked to the amount and payee
- **90-Day Re-authentication:** Consumer must re-authenticate with SCA every 90 days to maintain TPP access
- **Dedicated Interface (API):** Banks must provide a dedicated interface for TPP access — fallback to customer-facing interface only if dedicated interface underperforms

### mTLS for PSD2 TPP Communication
PSD2 requires mutual TLS (mTLS) between TPPs and account-holding institutions:
- TPP presents its eIDAS QWAC certificate in the TLS handshake
- Your API Gateway validates: certificate is a valid QWAC, issued by a trusted CA, TPP is registered with appropriate NCA, certificate is not revoked
- See `api-gateway-and-networking.md` for mTLS implementation on API Gateway


---

## ISO 20022 — Financial Message Standard

ISO 20022 is the universal financial messaging standard replacing legacy formats (SWIFT MT, NACHA fixed-width, various proprietary formats). It uses XML/JSON and is required for RTP, FedNow, SWIFT cross-border payments (CBPR+), and TARGET2 (EU).

### Message Categories
| Category | Prefix | Purpose | Example |
|---|---|---|---|
| Payments Initiation | pain | Customer-to-bank payment instructions | pain.001 (Credit Transfer Initiation) |
| Payments Clearing & Settlement | pacs | Bank-to-bank payment messages | pacs.008 (FI to FI Customer Credit Transfer) |
| Cash Management | camt | Account reporting and cash management | camt.053 (Bank to Customer Statement) |
| Securities | sese/setr | Securities trading and settlement | sese.023 (Securities Settlement) |

### SWIFT MT → ISO 20022 Migration
SWIFT is migrating cross-border payments from MT (Message Type) format to ISO 20022 MX format:
- **November 2025:** Full migration deadline for cross-border payments and reporting (CBPR+)
- **Coexistence period:** Translation services bridge MT and MX during migration
- If your SaaS processes SWIFT messages, you must support ISO 20022 MX format

### Architecture for ISO 20022 Processing
```
Inbound ISO 20022 Message (XML/JSON)
    → API Gateway (validate schema against XSD)
    → Lambda (parse, extract payment details, tenant routing)
    → DynamoDB (message store — immutable, per-tenant partition)
    → EventBridge (route to appropriate payment processor based on message type)
    → Outbound ISO 20022 response generation
```

**Multi-tenant:** Each tenant (financial institution) may have different message profiles, routing rules, and correspondent banking relationships. Store per-tenant configuration in DynamoDB.

---

## Multi-Tenant Open Banking Architecture

### One Platform Serving Multiple FI Tenants

Your platform IS the open banking API for your tenants (financial institutions). Each tenant has their own consumer base, their own authorized TPPs, and their own consent policies.

```
TPP (Data Recipient) → API Gateway → Lambda Authorizer
                                          ↓
                              Validate: OAuth token + consent + TPP registration
                                          ↓
                              Route to correct tenant's data
                                          ↓
                              DynamoDB / RDS (tenant-scoped query)
                                          ↓
                              Return FDX-compliant response
```

### Consent Management Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  Consent Store (DynamoDB)                                    │
│  PK: TENANT#{tenant_id}#CONSUMER#{consumer_id}              │
│  SK: TPP#{tpp_id}#CONSENT#{consent_id}                      │
│                                                              │
│  Fields:                                                     │
│  - scopes: ["fdx:transactions:read", "fdx:balances:read"]  │
│  - granted_at: ISO 8601 timestamp                           │
│  - expires_at: ISO 8601 timestamp (max 1 year from grant)   │
│  - status: active | revoked | expired                       │
│  - revoked_at: timestamp (if revoked)                       │
│  - purpose: "Personal financial management"                  │
└─────────────────────────────────────────────────────────────┘
```

**Consent enforcement at request time:**
```typescript
async function validateConsent(tenantId: string, consumerId: string, tppId: string, requestedScope: string): Promise<boolean> {
  const consent = await dynamodb.query({
    TableName: 'ConsentStore',
    KeyConditionExpression: 'pk = :pk AND begins_with(sk, :sk)',
    ExpressionAttributeValues: {
      ':pk': `TENANT#${tenantId}#CONSUMER#${consumerId}`,
      ':sk': `TPP#${tppId}#CONSENT#`,
    },
  });
  
  const activeConsent = consent.Items?.find(c => 
    c.status === 'active' && 
    new Date(c.expires_at) > new Date() &&
    c.scopes.includes(requestedScope)
  );
  
  if (!activeConsent) {
    // Log consent denial for audit
    await auditLog({ event: 'CONSENT_DENIED', tenantId, consumerId, tppId, requestedScope });
    return false;
  }
  
  return true;
}
```

### Per-Tenant TPP Management
Each tenant may authorize different TPPs:
- Tenant A (Bank A) allows TPPs X, Y, Z
- Tenant B (Bank B) allows TPPs X and W (not Y or Z)
- TPP registration status checked on every request
- Per-tenant rate limiting per TPP (API Gateway usage plans scoped by tenant + TPP)

---

## API Gateway Patterns for Open Banking

### Rate Limiting Per TPP Per Tenant
```
API Gateway → Usage Plan per (tenant_id + tpp_id)
    → 100 requests/second for premium TPPs
    → 10 requests/second for standard TPPs
    → Burst: 2x sustained rate
```

### Consent Validation Middleware (Lambda Authorizer)
The custom Lambda authorizer on every open banking endpoint:
1. Validates OAuth access token (signature, expiry, audience)
2. Extracts tenant_id, consumer_id, tpp_id from token claims
3. Queries ConsentStore to verify active consent for requested scope
4. If consent invalid: return 403 with standardized FDX error response
5. If consent valid: return IAM policy allowing access + context (tenant, consumer, scopes)

### Versioning
Open banking APIs must be versioned — FDX releases major versions (v5, v6) with breaking changes. Support multiple versions simultaneously:
- `/fdx/v5/accounts` and `/fdx/v6/accounts`
- API Gateway stage variables or path-based routing to different Lambda versions

---

## Common Mistakes

1. **Not checking consent on every request.** Consent can be revoked at any time. Caching consent status without real-time validation means you'll serve data after revocation — a Section 1033 / PSD2 violation.

2. **Screen-scraping as a fallback.** Section 1033 explicitly prohibits screen-scraping once a compliant API exists. Don't build both — invest in the API.

3. **Ignoring Section 1033 deadlines.** If your tenants are Tier 1 institutions, April 2026 is the deadline. If your platform is their data provider, your platform must be ready.

4. **Shared rate limits across TPPs.** TPP A hammering your API should not affect TPP B's access. Rate limit per TPP, per tenant.

5. **No consent audit trail.** Every consent grant, denial, revocation, and expiration must be logged. Regulators will ask for evidence of consumer consent management.

6. **Not supporting consent revocation in real-time.** Consumer revokes consent at 2:00 PM. If the TPP can still pull data at 2:01 PM because your system checks consent hourly, that's a violation.

7. **Forgetting ISO 20022 migration.** If your platform handles SWIFT messages, the MT → MX migration deadline is November 2025. Supporting only MT format means you'll break when correspondents switch.

---

## Discovery Questions for This Domain

**Open banking scope:**
- Do you need to expose consumer financial data APIs? (FDX, PSD2, Section 1033?)
- Are your tenants subject to Section 1033 compliance deadlines? What tier?
- Do you operate in EU/UK? (Triggers PSD2 requirements)
- Are you the data provider (hold the data) or data recipient (consume the data)?

**Consent:**
- How do you manage consumer consent today? (UI, storage, enforcement, revocation)
- What's the consent duration? (FDX default 90 days, PSD2 requires 90-day re-auth)
- Is consent checked on every API call or only at token issuance?
- Can consumers view and revoke consent through your platform?

**TPP management:**
- How do you onboard and authenticate TPPs? (OAuth client registration, eIDAS certificates?)
- Do you rate-limit per TPP? Per tenant?
- How do you handle TPP registration revocation by a regulator?

**Messaging standards:**
- Do you process ISO 20022 messages? Which categories? (pain, pacs, camt?)
- Are you migrating from SWIFT MT to MX format?
- Do you support RTP or FedNow? (Both use ISO 20022)

---

## References

- [Financial Data Exchange (FDX)](https://financialdataexchange.org/)
- [CFPB Section 1033 — Personal Financial Data Rights](https://www.consumerfinance.gov/personal-financial-data-rights/)
- [ISO 20022 Standard](https://www.iso20022.org/iso-20022-standard)
- [EBA PSD2 — Regulatory Technical Standards on SCA](https://www.eba.europa.eu/regulation-and-policy/payment-services-and-electronic-money/regulatory-technical-standards-on-strong-customer-authentication-and-common-and-secure-communication)
- [SWIFT — ISO 20022 Migration](https://www.swift.com/standards/iso-20022-migration)
- [FDX API Technical Specification](https://financialdataexchange.org/FDX/API)
- [AWS API Gateway mTLS](https://docs.aws.amazon.com/apigateway/latest/developerguide/rest-api-mutual-tls.html)
