# Observability & Operations

## Why This File Exists

Multi-tenant financial services SaaS requires tenant-aware observability: per-tenant metrics, payment SLA monitoring (99.99%+ for real-time rails), noisy neighbor detection, and operational dashboards that satisfy both platform operators and bank examiners reviewing your monitoring capabilities.

---

## Tenant-Aware Logging

### The Golden Rule
Every log entry must include `tenant_id`. No exceptions. Without it, you cannot: attribute errors to a tenant, detect noisy neighbors, generate per-tenant SLA reports, or respond to a bank examiner asking "show me the logs for our institution."

### What to Log (and What NOT to)

**Always include:**
- `tenant_id` — which tenant this request belongs to
- `correlation_id` — traces the request across services
- `user_id` — who initiated (internal ID, not name)
- `action` — what operation was performed
- `status` — success/failure
- `latency_ms` — how long it took
- `timestamp` — UTC ISO 8601

**NEVER include:**
- PAN (card numbers) — use token only
- Full SSN — use last-4 or reference ID
- Account numbers — use masked or internal reference
- Credit bureau report content
- Passwords, secrets, API keys

### Structured Logging Format (EMF)

```typescript
// CloudWatch Embedded Metric Format — finance SaaS pattern
const metrics = {
  _aws: {
    Timestamp: Date.now(),
    CloudWatchMetrics: [{
      Namespace: 'FinanceSaaS/Payments',
      Dimensions: [['TenantId', 'PaymentRail']],
      Metrics: [
        { Name: 'PaymentLatency', Unit: 'Milliseconds' },
        { Name: 'PaymentSuccess', Unit: 'Count' },
      ],
    }],
  },
  TenantId: tenantId,
  PaymentRail: 'ACH',
  PaymentLatency: 245,
  PaymentSuccess: 1,
  correlation_id: correlationId,
  // NO PII fields here
};
console.log(JSON.stringify(metrics));
```

---

## Per-Tenant Metrics

### Key Financial SaaS Metrics

| Metric | Dimensions | Alert Threshold | Why |
|---|---|---|---|
| Payment success rate | tenant_id, rail | < 99.5% over 5 min | Payment failures impact revenue |
| Payment latency (p99) | tenant_id, rail | > 2s (card), > 5s (ACH) | SLA breach risk |
| Transaction volume | tenant_id | > 200% of baseline | Fraud indicator or legitimate spike |
| API error rate (5xx) | tenant_id | > 1% over 5 min | Service degradation |
| API latency (p95) | tenant_id | > 1s | Noisy neighbor or capacity issue |
| KMS decrypt calls | tenant_id | > 150% baseline | Unusual data access pattern |
| Failed auth attempts | tenant_id | > 10 in 5 min | Brute force attempt |
| Cross-tenant access denied | tenant_id | Any occurrence | Isolation breach attempt |

### CloudWatch Dimensions Strategy

Use dimensions sparingly — CloudWatch charges per unique metric series:
- **Always:** `TenantId` (core dimension for per-tenant visibility)
- **When useful:** `PaymentRail`, `Environment`, `ServiceName`
- **Avoid:** High-cardinality dimensions like `UserId` or `TransactionId` as dimensions (use log filters instead)

---

## Payment SLA Monitoring

### Availability Requirements by Rail

| Payment Rail | Target Availability | Max Acceptable Downtime/Month |
|---|---|---|
| RTP / FedNow | 99.99% | 4.3 minutes |
| Card authorization | 99.95% | 21.6 minutes |
| ACH batch | 99.9% | 43.8 minutes |
| Open banking APIs | 99.9% (PSD2 mandate) | 43.8 minutes |
| Admin/reporting | 99.5% | 3.6 hours |

### SLA Dashboard Architecture

```
CloudWatch Metrics (per-tenant, per-rail)
    → CloudWatch Composite Alarms (SLA breach threshold)
    → SNS → PagerDuty/OpsGenie (immediate alerting)
    → CloudWatch Dashboard (real-time visibility)
    → Monthly SLA Report (generated via Lambda + S3)
```

### Per-Tenant SLA Reporting
Enterprise bank tenants will contractually require monthly SLA reports. Automate this:
- Lambda on monthly schedule queries CloudWatch Metrics API for the tenant
- Calculates availability percentage, incident count, mean time to resolution
- Generates PDF report, stores in per-tenant S3 bucket
- Optionally emails to tenant admin

---

## Noisy Neighbor Detection

### What Is a Noisy Neighbor in Finance SaaS
One tenant consuming disproportionate resources, degrading performance for others:
- A merchant with a flash sale floods the payment API (10x normal volume)
- A lender runs end-of-month batch processing, consuming all Aurora connections
- An AML re-screening batch job monopolizes compute

### Detection Architecture

```
Per-tenant metrics (CloudWatch)
    → CloudWatch Anomaly Detection (ML-based baseline per tenant)
    → Alarm: tenant exceeds 3σ from their own baseline
    → SNS notification to platform ops
    → Optional: automatic throttling via API Gateway usage plan adjustment
```

### Mitigation Strategies
| Strategy | Implementation | When to Use |
|---|---|---|
| API Gateway usage plans | Per-tenant rate limits | Always — baseline protection |
| Lambda concurrency limits | Reserved concurrency per tenant (silo) or shared with throttle | High-value tenants |
| DynamoDB on-demand | Auto-scales without affecting other tenants | Pool model |
| Aurora connection limits | Per-tenant max connections via application middleware | Pool/bridge RDS |
| SQS per-tenant queues | Isolate processing backlog per tenant | Payment/AML processing |
| Circuit breaker per tenant | If Tenant A's processor errors, only Tenant A's circuit opens | Payment processing |

---

## Operational Dashboards

### Platform Operator Dashboard
- Total transaction volume (all tenants)
- Payment success rate (aggregate + per-rail)
- Top 5 tenants by volume (for capacity planning)
- Error rates by service
- Active alerts and incidents
- Noisy neighbor indicators

### Tenant-Facing Dashboard (White-Label)
- Their transaction volume and success rate
- Their API usage vs. tier limits
- Their SLA performance (current month)
- Active incidents affecting them
- No visibility into other tenants' data

---

## Common Mistakes

1. **No `tenant_id` in logs.** Without it, debugging a tenant-specific issue requires grepping through all logs. Add it as a required field in your logging middleware.

2. **PII in CloudWatch Logs.** PAN, SSN, or account numbers in logs put those log groups in PCI/GLBA scope. Structured logging with reference IDs only.

3. **Shared rate limits across tenants.** If all tenants share one API Gateway usage plan, one tenant's spike affects everyone. Per-tenant usage plans are the baseline.

4. **No per-tenant SLA tracking.** Enterprise bank tenants will ask "what was our availability last month?" If you can't answer from metrics, you'll lose their trust.

5. **Alerting on aggregate only.** A 1% error rate across all tenants might hide a 50% error rate for one small tenant. Alert per-tenant, not just aggregate.

---

## Discovery Questions for This Domain

- Do you have per-tenant metrics today, or only aggregate?
- What are your contractual SLA commitments per tier?
- How do you detect and mitigate noisy neighbors?
- Do enterprise tenants require monthly SLA reports?
- Are your logs structured with tenant context in every entry?

---

## References

- [SaaS Lens — Operational Excellence](https://docs.aws.amazon.com/wellarchitected/latest/saas-lens/operational-excellence-pillar.html)
- [CloudWatch Embedded Metric Format](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Embedded_Metric_Format.html)
- [CloudWatch Anomaly Detection](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Anomaly_Detection.html)
- [AWS Well-Architected — Noisy Neighbor](https://docs.aws.amazon.com/wellarchitected/latest/saas-lens/noisy-neighbor.html)
