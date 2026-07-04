# Audit Logging & Access Controls

## Why This Is Non-Negotiable

Financial services SaaS operates under multiple audit trail mandates simultaneously:
- **PCI-DSS Requirement 10:** Log and monitor all access to system components and cardholder data
- **SOX ITGC:** Comprehensive audit trail for financial systems supporting reporting
- **SEC Rule 17a-4:** Records must be stored in non-rewritable, non-erasable (WORM) format for broker-dealers
- **GLBA Safeguards Rule:** Monitor and test effectiveness of safeguards
- **BSA/AML:** Document all alert dispositions, SAR filings, and investigation decisions

If you can't prove who accessed what financial data, when, and why — you can't prove compliance.

---

## What Must Be Logged

### Infrastructure-Level Events (CloudTrail)
- All API calls to AWS services handling financial data (management events)
- Data-level access: S3 objects, DynamoDB items, KMS key usage (data events)
- Console sign-in events, IAM policy changes, role assumptions
- Security group changes, VPC configuration changes
- CDE-specific: all network traffic logs (VPC Flow Logs) for CDE subnets

### Application-Level Events (Your Responsibility)
- Financial data access: who accessed which account/transaction, when, from where
- Payment operations: who initiated, approved, or canceled a payment
- Credit decisions: model input, output, reasons, who reviewed (for ECOA)
- AML actions: alert creation, disposition, SAR filing decision, who decided
- Configuration changes: who changed risk thresholds, rate tables, product rules
- Failed access: authentication failures, authorization denials, cross-tenant attempts

### What NOT to Log
- PAN in log entries — log the token, not the card number. If PAN appears in logs, those logs enter PCI scope.
- Full SSN — log last-4 or a reference ID only
- Full credit bureau reports — log the pull event (who, when, permissible purpose), not the report content

---

## SOX ITGC Audit Trail — Immutable Financial Records

### S3 Object Lock for WORM Compliance

SOX and SEC 17a-4 require that financial records cannot be modified or deleted during the retention period. S3 Object Lock in Compliance Mode satisfies this:

```typescript
const auditBucket = new s3.Bucket(this, 'FinancialAuditLogs', {
  bucketName: `${props.appName}-audit-logs`,
  encryption: s3.BucketEncryption.KMS,
  encryptionKey: auditKmsKey, // Dedicated key — not a tenant key
  versioned: true, // Required for Object Lock
  objectLockEnabled: true,
  objectLockDefaultRetention: s3.ObjectLockRetention.compliance(Duration.days(2555)), // 7 years (SOX)
  blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
  removalPolicy: RemovalPolicy.RETAIN,
});

// Deny deletion — even by root account in Compliance Mode
auditBucket.addToResourcePolicy(new iam.PolicyStatement({
  sid: 'DenyDeleteActions',
  effect: iam.Effect.DENY,
  principals: [new iam.AnyPrincipal()],
  actions: ['s3:DeleteObject', 's3:DeleteObjectVersion'],
  resources: [auditBucket.arnForObjects('*')],
}));
```

### Retention Periods by Regulation

| Regulation | Minimum Retention | Immediately Accessible |
|---|---|---|
| PCI-DSS Req. 10 | 12 months | 3 months |
| SOX | 7 years | Full period (for SOX evidence) |
| SEC 17a-4 (broker-dealers) | 6 years (some: 3 years) | 2 years |
| GLBA | 5 years | Reasonable access |
| BSA/AML (SAR) | 5 years from filing | Full period |
| FINRA (communications) | 3-6 years depending on type | Full period |

**Rule:** Set your Object Lock retention to the LONGEST applicable requirement. If you serve broker-dealer tenants (SEC 17a-4: 6 years) AND are SOX-relevant (7 years): use 7 years.

---

## SEC 17a-4 WORM Storage

Broker-dealers must retain books and records in a non-rewritable, non-erasable format (WORM). AWS has published guidance that S3 Object Lock in Compliance Mode and AWS CloudTrail Lake meet SEC 17a-4(f) requirements when properly configured.

**What must be WORM-stored for broker-dealers:**
- Order tickets, execution reports, confirmations
- Account statements
- Customer communications related to trading
- Compliance records (trade surveillance alerts, decisions)
- All FIX messages (drop copy) — see `trading-and-market-data.md`

**Configuration:**
- S3 Object Lock: Compliance Mode (cannot be overridden even by root)
- Retention period: 6 years from record creation (some categories: 3 years)
- Designated third party (D3P): SEC 17a-4(f)(3) requires a designated third party who can access records for regulatory examination. Document AWS + your own access procedures.

---

## Application-Level Audit Event Schema

```json
{
  "timestamp": "2025-01-15T14:15:00.123Z",
  "event_id": "550e8400-e29b-41d4-a716-446655440000",
  "event_type": "PAYMENT_INITIATED",
  "tenant_id": "tenant-abc123",
  "user_id": "user-ops-456",
  "user_role": "payment_operator",
  "resource_type": "payment",
  "resource_id": "pay_xyz789",
  "action": "create",
  "outcome": "success",
  "source_ip": "10.0.1.50",
  "user_agent": "treasury-app/2.1.0",
  "amount": 50000.00,
  "currency": "USD",
  "counterparty_masked": "****6789",
  "approval_required": true,
  "approved_by": "user-manager-789",
  "session_id": "sess-abc",
  "correlation_id": "req-def-123"
}
```

**Key fields for finance:**
- `amount` and `currency`: financial transaction amount (for materiality thresholds)
- `counterparty_masked`: masked identifier (never full account number in logs)
- `approval_required` / `approved_by`: dual-control evidence for high-value operations
- `correlation_id`: traces the request across microservices

---

## Break-the-Glass for Financial Data

### When It Applies
Emergency access to production financial data when normal access controls are insufficient:
- System outage requiring direct database access for recovery
- Regulatory examination requiring immediate data access
- Fraud investigation requiring cross-normal-boundary access

### Architecture Pattern
1. User requests break-the-glass access through the application (documented request)
2. Requires two-person approval (dual control — SOX requirement)
3. System grants time-limited elevated access (maximum 4 hours)
4. ALL actions during elevated session logged with `event_type: BREAK_THE_GLASS`
5. Automatic notification to: CISO, compliance team, tenant admin (if tenant data accessed)
6. Session expires automatically
7. Post-incident review mandatory within 24 hours

---

## Common Mistakes

1. **Not enabling CloudTrail data events for CDE services.** Management events don't show who accessed which DynamoDB item or S3 object. Data events are required for PCI Requirement 10 in the CDE.

2. **Audit logs stored without WORM protection.** If logs can be deleted or modified, they're worthless for SOX, SEC 17a-4, and PCI compliance. Object Lock Compliance Mode is the standard.

3. **PAN in audit log entries.** If your payment Lambda logs the full card number, those logs enter PCI scope. Log the token or last-4 only.

4. **No dual-control evidence.** SOX expects that high-value or high-risk operations (large payments, configuration changes) require two-person approval. Your audit trail must show who initiated AND who approved.

5. **Insufficient retention.** SOX is 7 years, SEC 17a-4 is 6 years, PCI is 12 months. Set retention to the longest applicable period and make it non-negotiable.

6. **No break-the-glass procedure.** If emergency access requires a developer to modify IAM policies ad-hoc, you have no audit trail of the access decision. Formalize it.

---

## Discovery Questions for This Domain

**Current state:**
- Are CloudTrail data events enabled for all financial data services (S3, DynamoDB, KMS)?
- Do you have application-level audit logging for financial operations?
- Are audit logs immutable (S3 Object Lock / WORM)?
- What is your current log retention period?

**Compliance requirements:**
- Are you subject to SEC 17a-4? (Broker-dealer tenants)
- Do you need SOX ITGC evidence? (Public company or serving public company tenants)
- What's your PCI DSS assessment scope? (Requirement 10 obligations for CDE)

**Access control:**
- Do you have dual-control (two-person approval) for high-value operations?
- Is there a documented break-the-glass procedure?
- How often do you review who has access to financial data? (Quarterly for SOX/PCI)

---

## References

- [SEC Rule 17a-4 — Recordkeeping Requirements](https://www.sec.gov/investment/amendments-electronic-recordkeeping-requirements-broker-dealers)
- [AWS CloudTrail — Data Events](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/logging-data-events-with-cloudtrail.html)
- [S3 Object Lock](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html)
- [PCI-DSS Requirement 10 — Logging and Monitoring](https://www.pcisecuritystandards.org/document_library/)
- [AWS Financial Services — Compliance](https://aws.amazon.com/financial-services/security-compliance/)
- [SOX IT General Controls](https://www.sec.gov/spotlight/soxcomp.htm)
