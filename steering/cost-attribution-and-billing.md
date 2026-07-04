# Cost Attribution & Billing

## Why This File Exists

Multi-tenant financial SaaS must track per-tenant costs for: pricing decisions (are we profitable per tenant?), customer billing (usage-based pricing), tier enforcement (is this tenant exceeding their plan?), and AWS Marketplace integration. Financial services SaaS also has unique billing dimensions — interchange revenue share, per-transaction pricing, and regulatory-driven cost allocation (CDE vs. non-CDE workloads).

---

## Per-Tenant Cost Visibility

### Cost Allocation Strategy

| Method | How It Works | Best For |
|---|---|---|
| Resource tagging | Tag every AWS resource with `TenantId` → Cost Explorer filtering | Silo model (dedicated resources) |
| CUR + custom allocation | Process Cost and Usage Report, allocate shared costs by usage metrics | Bridge/pool model |
| Application metering | Application-level counters (API calls, transactions) → billing system | Usage-based pricing |
| Per-tenant KMS costs | Each tenant's CMK has a direct monthly cost ($1/key/month + usage) | Attributable encryption cost |

### Tagging Strategy for Financial SaaS

```
Required tags on ALL resources:
- TenantId: {tenant identifier}
- Environment: production | staging | development
- Service: {service name}
- CostCenter: platform | tenant-{id}
- PCI_Scope: in-scope | out-of-scope | connected-to
- Regulatory: sox | pci | glba | none
```

**PCI_Scope tag is finance-specific:** Allows you to calculate the cost of your CDE separately from out-of-scope services. Useful for: justifying scope reduction investments, reporting PCI compliance costs to leadership, allocating security costs to the correct budget.

---

## Metering Dimensions for Financial SaaS

### Common Billing Dimensions

| Dimension | How Measured | Example Pricing |
|---|---|---|
| API calls | API Gateway usage plan metrics per tenant | $0.001 per call above tier limit |
| Transactions processed | DynamoDB write event count per tenant | $0.10 per payment transaction |
| Transaction volume ($) | Sum of transaction amounts per billing period | Basis points on volume (5 bps) |
| Active accounts | Count of active financial accounts per tenant | $2/account/month |
| Storage (documents) | S3 per-tenant bucket/prefix size | Included up to 100GB, then $0.023/GB |
| Credit bureau pulls | Count of bureau API calls per tenant | Pass-through + margin |
| OFAC screenings | Count of screening API calls per tenant | Included in transaction fee |
| Open banking API calls (from TPPs) | Per-tenant, per-TPP API call count | $0.005 per TPP API call |

### Interchange and Revenue Share

For payment platforms, revenue often includes interchange-like components:
- Platform takes a percentage of transaction volume (e.g., 25 bps)
- Revenue split between platform and tenant varies by tier
- Metering must capture: transaction count, total volume, network fees, net revenue per tenant

### Metering Architecture

```
Financial Event (payment, API call, bureau pull)
    → CloudWatch EMF metric (tenant_id dimension)
    → Kinesis Data Firehose → S3 (detailed usage records)
    → Lambda (hourly aggregation) → DynamoDB (billing summaries per tenant)
    → Billing service reads summaries → generates invoice
```

---

## AWS Marketplace Integration

### When to Use Marketplace
- Enterprise fintech buyers who want to pay for your SaaS through their AWS bill
- Simplifies procurement for bank tenants (AWS is already an approved vendor)
- Private offers for enterprise contracts with custom pricing
- Marketplace Metering API for usage-based billing

### Integration Pattern

```
Tenant signs up via AWS Marketplace
    → Marketplace sends SaaS redirect to your onboarding endpoint
    → Your control plane creates tenant, links AWS account ID
    → Usage metering: your platform calls Marketplace BatchMeterUsage API hourly
    → AWS bills the tenant's AWS account
    → AWS pays you (minus marketplace fee)
```

### Marketplace Metering for Financial SaaS
```typescript
import { MarketplaceMeteringClient, BatchMeterUsageCommand } from '@aws-sdk/client-marketplace-metering';

const metering = new MarketplaceMeteringClient({});

// Report hourly usage per tenant
await metering.send(new BatchMeterUsageCommand({
  ProductCode: 'prod-abc123',
  UsageRecords: [{
    CustomerIdentifier: tenantAwsAccountId,
    Dimension: 'TransactionsProcessed', // matches Marketplace listing dimension
    Quantity: transactionCount,
    Timestamp: new Date(),
  }],
}));
```

---

## CDE vs. Non-CDE Cost Allocation

Financial platforms should track the cost of PCI compliance separately:

| Cost Category | Example Resources | Attribution |
|---|---|---|
| CDE compute | Tokenization Lambda, payment processing ECS | Tag: PCI_Scope=in-scope |
| CDE networking | CDE subnets, VPC endpoints, Flow Logs | Tag: PCI_Scope=in-scope |
| PCI assessment | QSA fees, penetration testing | Operational budget |
| Non-CDE (everything else) | Lending logic, analytics, admin | Tag: PCI_Scope=out-of-scope |

**Why track this:** Justifies scope reduction investments. If CDE costs $50K/month and you can reduce scope by tokenizing earlier, the ROI is clear.

---

## Per-Tenant Profitability

### Cost Model

```
Tenant Revenue (monthly)
  - AWS infrastructure cost (attributed via tags + CUR)
  - Third-party costs (credit bureau, payment processor, KYC provider — per tenant)
  - Platform operational cost (allocated share: total ops cost / tenant count or by usage weight)
  = Tenant Gross Margin
```

### Unprofitable Tenant Detection
- Monthly automated report: revenue vs. attributed cost per tenant
- Alert if any tenant's gross margin drops below threshold (e.g., < 20%)
- Common causes: tenant on wrong tier (using enterprise resources at starter pricing), excessive API calls, high dispute/chargeback volume

---

## Common Mistakes

1. **No resource tagging from day one.** Retrofitting tags across hundreds of resources is painful. Enforce tagging via SCP or AWS Config rule from the start.

2. **Metering only infrastructure, not application events.** AWS costs (tagged resources) give you infrastructure cost. But your billing dimensions (transactions, API calls) come from application-level metering. You need both.

3. **No CDE cost visibility.** If you can't tell leadership "PCI compliance costs us $X/month," you can't justify scope reduction investments.

4. **Marketplace integration as afterthought.** Enterprise fintech buyers increasingly want to pay via their AWS bill. Build Marketplace metering into your billing system early.

5. **Not tracking third-party costs per tenant.** Credit bureau pulls, payment processor fees, and KYC verification costs vary per tenant. Include them in profitability analysis.

---

## Discovery Questions for This Domain

- How do you price your product today? (Per-seat, per-transaction, volume-based, flat fee?)
- Can you attribute AWS costs to individual tenants today?
- Are all resources tagged with tenant ID?
- Do you use AWS Marketplace, or plan to?
- What are your per-tenant third-party costs? (Bureau, processor, KYC)
- Do you know your per-tenant gross margin?

---

## References

- [SaaS Lens — Cost Optimization](https://docs.aws.amazon.com/wellarchitected/latest/saas-lens/cost-optimization-pillar.html)
- [AWS Cost Allocation Tags](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/cost-alloc-tags.html)
- [AWS Marketplace — SaaS Integration](https://docs.aws.amazon.com/marketplace/latest/userguide/saas-getting-started.html)
- [Metering for Usage in AWS Marketplace](https://docs.aws.amazon.com/marketplace/latest/userguide/metering-for-usage.html)
