# Tenant Isolation Strategies

## Why Isolation Is Non-Negotiable

Tenant isolation is the most critical security concern in multi-tenant financial services SaaS. One tenant accessing another tenant's financial data — transaction records, account balances, cardholder data, credit reports — is not just a security incident, it's a regulatory violation that can result in enforcement actions, loss of banking relationships, and business termination. The AWS SaaS Tenant Isolation Strategies whitepaper is clear: isolation must be enforced at the infrastructure level, not just the application level.

**The finance isolation bar:** Even if application code has a bug, the underlying infrastructure must prevent cross-tenant access to financial data, payment records, and cardholder information.

## Isolation Models by Tenancy Type

### Silo Isolation (Dedicated Resources)
Each tenant's resources are in separate infrastructure.

**Implementation patterns:**
- **Account-per-tenant**: Each tenant gets their own AWS account via Organizations. SCPs provide guardrails. Strongest isolation — required for competing broker-dealers and large chartered banks.
- **VPC-per-tenant**: Tenants share an account but have separate VPCs. Network isolation via security groups and NACLs.
- **Resource-per-tenant**: Separate DynamoDB tables, separate RDS instances, separate S3 buckets per tenant.

### Pool Isolation (Shared Resources)
All tenants share the same resources. Isolation enforced at runtime via IAM policies, application logic, and database-level controls.

**Runtime IAM Policy Generation (Dynamic Policies):**
1. Request arrives with JWT containing tenant ID
2. Application extracts tenant context from JWT
3. Application calls STS AssumeRole with a session policy scoped to the current tenant
4. Scoped credentials only allow access to the current tenant's data
5. All subsequent AWS SDK calls use scoped credentials

**DynamoDB pool isolation example — session policy:**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["dynamodb:GetItem", "dynamodb:PutItem", "dynamodb:Query"],
    "Resource": "arn:aws:dynamodb:*:*:table/Transactions",
    "Condition": {
      "ForAllValues:StringEquals": {
        "dynamodb:LeadingKeys": ["TENANT#${tenantId}"]
      }
    }
  }]
}
```

### Bridge Isolation (Mixed)
Shared compute with pool-style IAM isolation, plus dedicated storage with silo-style resource boundaries.

## CDE Isolation — PCI-DSS Network Segmentation in SaaS

PCI-DSS Requirement 1 mandates network security controls that segment the Cardholder Data Environment (CDE) from all other systems. In multi-tenant SaaS, this means:

### CDE Network Architecture

```
┌─────────────────────────────────────────────────────────┐
│  VPC — Application Plane                                 │
│  ┌──────────────────────┐   ┌──────────────────────────┐│
│  │  Non-CDE Subnets      │   │  CDE Subnets (Isolated)  ││
│  │  - Analytics          │   │  - Tokenization Lambda    ││
│  │  - Admin dashboard    │   │  - Payment Processing     ││
│  │  - Reporting          │   │  - Amazon Payment Crypto  ││
│  │  - Lending logic      │   │  - Card data in-memory    ││
│  │                       │   │                           ││
│  │  SG: no access to CDE│   │  SG: only from API GW     ││
│  └──────────────────────┘   └──────────────────────────┘│
└─────────────────────────────────────────────────────────┘
```

**Key rules:**
- CDE subnets have NO outbound internet access (no NAT Gateway, no public IPs)
- CDE subnets communicate with payment processors via PrivateLink or dedicated VPC endpoints
- Only the API Gateway (or ALB) can send traffic into CDE subnets — and only specific payment-related paths
- Non-CDE services CANNOT reach CDE subnets — security groups deny all traffic from non-CDE source
- VPC Flow Logs enabled on CDE subnets — stored in S3 with Object Lock for PCI Requirement 10

### Per-Tenant CDE vs. Shared CDE

**Shared CDE (recommended for most payment SaaS):**
One CDE serving all tenants. Tenant isolation within the CDE is achieved via per-tenant tokenization (each tenant's tokens map to their PANs only) and IAM session policies. PCI QSA assesses one CDE.

**Per-tenant CDE (rare, expensive):**
Each tenant gets their own CDE subnets and infrastructure. Only justified when a tenant is itself a PCI-assessed entity (e.g., a bank running their own card program) and their QSA requires dedicated infrastructure.


## IAM-Based Isolation Patterns for Financial Data

### Dynamic Policies with STS (Pool)

**Code example — tenant-scoped credentials for financial transaction access:**
```typescript
import { STSClient, AssumeRoleCommand } from '@aws-sdk/client-sts';

const sts = new STSClient({});

export async function getTenantScopedCredentials(tenantId: string, baseRoleArn: string) {
  const sessionPolicy = JSON.stringify({
    Version: '2012-10-17',
    Statement: [{
      Effect: 'Allow',
      Action: ['dynamodb:GetItem', 'dynamodb:PutItem', 'dynamodb:Query'],
      Resource: 'arn:aws:dynamodb:*:*:table/TransactionLedger',
      Condition: {
        'ForAllValues:StringEquals': {
          'dynamodb:LeadingKeys': [`TENANT#${tenantId}`]
        }
      }
    }, {
      Effect: 'Allow',
      Action: ['kms:Decrypt', 'kms:GenerateDataKey'],
      Resource: 'arn:aws:kms:*:*:key/*',
      Condition: {
        StringEquals: { 'kms:RequestAlias': `alias/tenant-${tenantId}-financial-pii` }
      }
    }]
  });

  const { Credentials } = await sts.send(new AssumeRoleCommand({
    RoleArn: baseRoleArn,
    RoleSessionName: `tenant-${tenantId}-${Date.now()}`,
    Policy: sessionPolicy,
    DurationSeconds: 900, // 15 min — short-lived for financial data access
  }));

  return Credentials;
}
```
*Note: Scopes both DynamoDB access (LeadingKeys) and KMS key usage to the tenant. Even with a bug, IAM + KMS prevent cross-tenant financial data access.*

### ABAC (Attribute-Based Access Control)

Use resource tags and session tags for isolation without per-tenant policies:
1. Tag resources with `TenantId` tag
2. Pass `TenantId` as session tag when assuming role
3. IAM policy uses `aws:PrincipalTag/TenantId` to match resource tags

```json
{
  "Effect": "Allow",
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::financial-documents/*",
  "Condition": {
    "StringEquals": {
      "s3:ExistingObjectTag/TenantId": "${aws:PrincipalTag/TenantId}"
    }
  }
}
```

### Amazon Verified Permissions for Application-Level Authorization

For fine-grained authorization beyond IAM (which controls AWS resource access):

| Concern | Use IAM | Use Verified Permissions |
|---|---|---|
| AWS resource access (DynamoDB, S3) | ✅ | ❌ |
| Application permissions (can user X approve loan Y?) | ❌ | ✅ |
| Tenant-scoped AWS API calls | ✅ | ❌ |
| Feature access per tier (premium analytics, bulk export) | ❌ | ✅ |
| Role-based access within a tenant (admin, analyst, auditor) | ❌ | ✅ |
| Transaction approval limits (can this user approve > $100K?) | ❌ | ✅ |

## PCI-DSS Requirements 7 & 8 — Mapped to Isolation

**Requirement 7 — Restrict Access by Business Need:**
- No human has standing access to CDE production data — access is just-in-time, time-limited, and audited
- Service roles follow least privilege — a reporting service has NO access to CDE resources
- IAM Access Analyzer continuously validates that no policy grants unintended CDE access

**Requirement 8 — Identify Users and Authenticate Access:**
- Every identity accessing CDE systems is unique (no shared accounts, no shared IAM roles between tenants)
- MFA required for all interactive CDE access
- Service-to-service: IAM roles only (no access keys in CDE)
- Password/credential requirements enforced via Cognito or IAM Identity Center

## Testing Isolation

### What to Test
1. **Cross-tenant transaction access:** Auth as Tenant A, attempt to query Tenant B's transactions. Must fail.
2. **Cross-tenant KMS key usage:** Auth as Tenant A, attempt to use Tenant B's CMK. Must fail.
3. **CDE boundary enforcement:** From non-CDE subnet, attempt to reach CDE resources. Must be blocked at network level.
4. **Token namespace isolation:** Use Tenant A's payment token in a detokenize call authenticated as Tenant B. Must fail.
5. **IAM policy enforcement:** Bypass application code, call AWS APIs directly with scoped credentials. Verify cross-tenant access blocked.
6. **AML alert isolation:** Auth as Tenant A, attempt to view Tenant B's SAR alerts or case files. Must fail.

### How to Test
- Automated integration tests on every deployment
- Cross-tenant access attempts in CI/CD pipeline
- Separate test tenants with known data
- Annual penetration test focused on cross-tenant financial data access

### The Compliance Test
Can you demonstrate to a PCI QSA, bank examiner, or SOX auditor that Tenant A cannot access Tenant B's financial data — even if a developer introduces a bug? If the answer relies solely on application code correctness, you fail.

## Common Mistakes

1. **Relying only on application-level checks** — `WHERE tenant_id = X` is not isolation. It's a bug away from exposing another tenant's financial data. Back it with IAM.

2. **Not testing isolation** — If you don't have automated cross-tenant access tests, you don't know if isolation works.

3. **Forgetting the CDE boundary** — You isolated DynamoDB per-tenant but the tokenization service is reachable from non-CDE subnets. CDE segmentation is a network control, not just a data control.

4. **No per-tenant KMS keys for financial PII** — Shared encryption keys weaken the isolation story for PCI QSAs and bank examiners.

5. **Not isolating AML case data** — SAR filings are confidential. One tenant viewing another tenant's SAR alerts is a BSA violation.

6. **Competing tenants sharing metadata visibility** — If competing broker-dealers can see each other's transaction volumes or tenant names, that's a competitive intelligence leak even without data access.

## Discovery Questions for This Domain

**Current state:**
- How is tenant isolation enforced today? (Application-level only? IAM policies? Network isolation?)
- Do you have automated tests that verify one tenant can't access another's financial data?
- Is your CDE network-segmented from non-CDE workloads?

**Requirements:**
- Do you have PCI-DSS scope? Is CDE isolation documented for your QSA?
- Do any tenants' examiners (OCC, FDIC, NCUA, FINRA) review your isolation architecture?
- Do you serve competing institutions that must not share any infrastructure visibility?

**Implementation:**
- Are you using IAM session policies or ABAC for runtime isolation?
- How is tenant context propagated through your stack? (JWT? Headers? Message attributes?)
- For async workflows (SQS, EventBridge, batch jobs) — does tenant context survive the async boundary?

## References

- [SaaS Tenant Isolation Strategies Whitepaper](https://docs.aws.amazon.com/whitepapers/latest/saas-tenant-isolation-strategies/saas-tenant-isolation-strategies.html)
- [Isolating SaaS Tenants with Dynamically Generated IAM Policies](https://aws.amazon.com/blogs/apn/isolating-saas-tenants-with-dynamically-generated-iam-policies/)
- [How to Implement SaaS Tenant Isolation with ABAC and AWS IAM](https://aws.amazon.com/blogs/security/how-to-implement-saas-tenant-isolation-with-abac-and-aws-iam/)
- [SaaS Access Control Using Amazon Verified Permissions](https://aws.amazon.com/blogs/security/saas-access-control-using-amazon-verified-permissions-with-a-per-tenant-policy-store/)
- [Multi-Tenant Authorization and API Access Control](https://docs.aws.amazon.com/prescriptive-guidance/latest/saas-multitenant-api-access-authorization/introduction.html)
- [AWS Financial Services Industry Lens — Security Pillar](https://docs.aws.amazon.com/wellarchitected/latest/financial-services-industry-lens/security-pillar.html)
