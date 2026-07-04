# Data Partitioning Strategies

## The Core Decision

Data partitioning is where your tenancy model meets reality. The model you chose (silo, pool, bridge) must be implemented differently for each storage service. In financial services SaaS, the key constraint is: financial transaction data and PII must be demonstrably isolated per tenant for PCI-DSS, GLBA, and bank examiner expectations.

**Key principle:** Evaluate data partitioning per service, not globally. Your transaction ledger strategy might be bridge while your analytics strategy is pool.

---

## DynamoDB for Financial Transaction Data

### Pool Model: Tenant ID in Partition Key

All tenants share one table. Tenant ID is the leading partition key element.

**Partition key design for transactions:**
- PK: `TENANT#abc123`, SK: `TXN#2024-01-15T14:30:00Z#pay_xyz789`
- PK: `TENANT#abc123`, SK: `ACCT#checking-001#BAL`
- PK: `TENANT#abc123`, SK: `ENTRY#led_001#2024-01-15T14:30:00Z`

**Isolation:** IAM session policy with `dynamodb:LeadingKeys` condition.

**Hot partition risk for financial workloads:**
- A high-volume merchant tenant can generate thousands of transactions/second under one partition key prefix
- **Mitigation:** Write sharding — append a random suffix to the partition key: `TENANT#abc123#SHARD#03`
- Query requires scatter-gather across shards, but write throughput scales linearly
- Alternative: use on-demand mode + adaptive capacity (DynamoDB automatically redistributes capacity to hot keys)

**When to use pool DynamoDB:** High tenant count (1000+), uniform transaction volumes, cost-sensitive.

### Silo Model: Table Per Tenant

Each tenant gets their own DynamoDB table: `tenant-abc123-transactions`, `tenant-abc123-ledger`.

**When to use silo DynamoDB:** Low tenant count, enterprise banks, tenants with very different throughput needs, tenants requiring dedicated backup/restore.

### Bridge Model: Shared Transaction Table + Per-Tenant Ledger Tables

Shared table for high-frequency transaction events (pool). Dedicated tables for ledger (per-tenant, immutable, strongly consistent). This balances cost with the regulatory requirement for ledger isolation.

---

## RDS/Aurora for Ledger Data

### Pool Model: Shared Schema with Row-Level Security

All tenants share one Aurora PostgreSQL cluster, same schema, same tables. RLS enforces tenant isolation.

```sql
-- Enable RLS on ledger_entries table
ALTER TABLE ledger_entries ENABLE ROW LEVEL SECURITY;

-- Policy: users can only see their tenant's entries
CREATE POLICY tenant_isolation ON ledger_entries
  USING (tenant_id = current_setting('app.current_tenant')::text);

-- Set tenant context at connection start
SET app.current_tenant = 'tenant-abc123';
```

**Pros:** Cost-efficient, single database to manage, SQL reporting across all data.
**Cons:** RLS overhead, shared connection pool, noisy neighbor on shared instance.
**When to use:** High tenant count, cost-sensitive lending platforms, fintechs with moderate data volumes.

### Bridge Model: Schema Per Tenant

Each tenant gets their own PostgreSQL schema on a shared Aurora instance.

```sql
CREATE SCHEMA tenant_abc123;
CREATE TABLE tenant_abc123.ledger_entries (...);
CREATE TABLE tenant_abc123.accounts (...);
```

**Pros:** Clear data separation, easy per-tenant backup (pg_dump per schema), straightforward right-to-erasure (DROP SCHEMA).
**Cons:** Connection routing complexity, instance limits on schemas.
**When to use:** Mid-market (tens to low hundreds of tenants), ledger workloads, community banks.

### Silo Model: Instance Per Tenant

Each tenant gets their own Aurora cluster. Complete infrastructure isolation.

**When to use:** Enterprise banks, broker-dealers, tenants with dedicated compliance/audit requirements.

### Aurora Serverless v2 for Variable Financial Workloads
Financial workloads are bursty — end-of-month processing, payroll days, tax season. Aurora Serverless v2 auto-scales ACUs (Aurora Capacity Units) based on demand:
- Minimum: 0.5 ACU during quiet periods
- Maximum: configurable per tenant tier
- Scales in seconds — handles payment spikes without pre-provisioning

---

## S3 for Financial Documents

### What Goes in S3
- Loan documents (applications, disclosures, signed agreements)
- KYC/KYB documents (uploaded IDs, formation documents, beneficial ownership records)
- Account statements (generated PDFs)
- NACHA files (ACH batch files — inbound and outbound)
- Settlement files (processor reconciliation files)
- Audit logs (CloudTrail, application audit events)
- AML case files (evidence, SAR supporting documentation)

### Per-Tenant Bucket (Silo — Recommended for Financial Documents)
```
s3://tenant-abc123-documents/
├── kyc/
├── loan-files/
├── statements/
├── ach-files/
├── settlements/
└── compliance/
```

**Why per-tenant buckets for finance:** Financial documents have long retention requirements (GLBA 5yr, SEC 17a-4 6yr), per-tenant access logging is required for SOX, and right-to-erasure after retention is straightforward (delete the bucket).

### Shared Bucket with Prefix (Pool — Acceptable for Non-Sensitive)
```
s3://platform-shared/
├── tenant-abc123/reference-data/
├── tenant-abc123/public-configs/
└── tenant-def456/reference-data/
```

Only for non-sensitive, non-regulated data (product catalogs, rate tables, public reference data).

---

## Timestream / DynamoDB for Time-Series Transaction Data

Financial analytics often requires time-series queries (transaction volumes over time, daily balances, trend analysis).

### DynamoDB with Time-Based Sort Keys
```
PK: TENANT#abc123#ACCT#checking-001
SK: TS#2024-01-15T14:30:00Z#TXN#pay_xyz789
```
- Query by time range: `SK BETWEEN 'TS#2024-01-01' AND 'TS#2024-01-31'`
- Efficient for per-account transaction history

### Amazon Timestream (for Metrics/Analytics)
- Store per-tenant financial metrics: transaction volume, average transaction value, error rates
- Time-series queries for dashboards and anomaly detection (AML signals)
- Automatic tiering: recent data in memory, historical data in magnetic storage
- Multi-tenant: use `tenant_id` as a dimension, query with tenant filter

---

## Redshift for Financial Analytics

### Multi-Tenant Data Warehouse Patterns

**Shared Cluster with Schema Per Tenant:**
Each tenant gets their own Redshift schema. Cross-tenant analytics (for your platform) queries across schemas. Tenant-facing analytics queries only their schema.

**Redshift Serverless with Data Sharing:**
- Serverless workgroup per tenant tier (shared for pool, dedicated for enterprise)
- Data sharing between producer (operational data) and consumer (analytics)
- Per-tenant access policies via Redshift row-level security

### What to Put in Redshift
- Aggregated transaction analytics (not raw PII — de-identified or aggregated)
- Reconciliation reports and discrepancy tracking
- Revenue/interchange analytics per tenant
- AML pattern detection on historical transaction data
- Portfolio analytics (lending: loan performance, delinquency rates)

---

## Financial Data Backup and Recovery

### RPO/RTO Requirements by Workload

| Workload | RPO | RTO | Backup Method |
|---|---|---|---|
| Payment processing (real-time) | 0 (no data loss) | < 5 min | Multi-AZ Aurora, DynamoDB global tables |
| Transaction ledger | < 1 min | < 15 min | Aurora continuous backup + PITR |
| Loan/account data | < 15 min | < 1 hour | DynamoDB PITR + S3 versioning |
| Financial documents (S3) | 0 (S3 durability) | < 1 hour | Cross-region replication |
| Audit logs | 0 (S3 durability) | < 1 hour | S3 Object Lock + cross-region replication |
| Analytics (Redshift) | < 1 hour | < 4 hours | Automated snapshots + cross-region copy |

### RTP/FedNow Availability
Real-time payment rails require 24/7/365 availability. For systems processing RTP or FedNow:
- Multi-AZ is the minimum — Multi-Region active-active is recommended
- Zero-downtime deployments mandatory (blue/green, canary)
- Failover must be automatic and tested quarterly

---

## GLBA Retention vs. GDPR/CCPA Right to Erasure

### The Tension
- **GLBA** requires retention of certain financial records for 5 years
- **SEC Rule 17a-4** requires broker-dealers to retain records for 6 years (some categories: 3 years)
- **SOX** requires 7-year retention for audit evidence
- **CCPA/GDPR** allows consumers to request deletion of personal data

### Resolution Architecture

```
Consumer Deletion Request
    ↓
Check: Is this data subject to regulatory retention?
    ↓
┌───────────────────────────────┐     ┌───────────────────────────────┐
│  NOT under retention           │     │  UNDER retention (GLBA/SOX)    │
│  → Delete from all stores     │     │  → Move to compliance archive  │
│  → Delete from backups        │     │  → Remove from active systems  │
│  → Confirm deletion to user   │     │  → Inform user: "retained per  │
│                                │     │    federal law until [date]"   │
└───────────────────────────────┘     └───────────────────────────────┘
                                              ↓ (after retention expires)
                                       Cryptographic erasure (KMS key deletion)
```

**Implementation:**
- Tag every data record with a `retention_category` and `retention_expires_at`
- Deletion request workflow checks retention category before acting
- Compliance archive: S3 bucket with Object Lock (Compliance Mode) set to retention period
- After retention: schedule KMS key deletion → cryptographic erasure

---

## Common Mistakes

1. **Mutable transaction records.** Financial transactions in DynamoDB or RDS should be append-only. Never update or delete a transaction record — create reversals.

2. **No write sharding for high-volume tenants.** A single DynamoDB partition key prefix for a high-volume merchant will throttle. Use write sharding or table-per-tenant for hot tenants.

3. **Shared RDS connections without limits.** In pool model, one tenant's burst can exhaust database connections for everyone. Use RDS Proxy + per-tenant connection limits.

4. **Financial documents in shared S3 bucket.** Loan files, KYC documents, and compliance records should be in per-tenant buckets for clean access logging, retention management, and erasure.

5. **Not planning for pool → silo migration.** When a fintech tenant grows into a bank charter, they'll demand dedicated infrastructure. Design your data access layer so tenant data can be migrated without application code changes.

6. **Ignoring the retention vs. erasure tension.** A consumer deletion request cannot override GLBA/SOX retention. Document this in your privacy policy and implement the tiered approach above.

---

## Discovery Questions for This Domain

**Storage services:**
- What storage services are you using? (DynamoDB, Aurora/RDS, S3, ElastiCache, OpenSearch, Redshift?)
- For each: what data does it hold and how sensitive is it?
- Do you have an existing schema, or is this greenfield?

**Partitioning:**
- For each storage service: does financial data need per-tenant isolation, or is pool acceptable?
- Do you need per-tenant backup and restore?
- Do you have right-to-erasure requirements (CCPA/GDPR)? How do you handle the GLBA retention conflict?
- What's the expected transaction volume per tenant? (Hot partition risk assessment)

**Access patterns:**
- Do you need cross-tenant analytics? (Platform-level reporting, AML pattern detection)
- What's the read/write ratio for transaction data?
- Are there batch operations? (End-of-day reconciliation, NACHA file processing, bulk statement generation)

**Scale:**
- How many tenants per database instance?
- Do you expect significant variance in data volume across tenants?
- Do you need to support tenant data migration between models (pool → silo for tier upgrades)?

---

## References

- [Multi-Tenant SaaS Storage Strategies Whitepaper](https://docs.aws.amazon.com/whitepapers/latest/multi-tenant-saas-storage-strategies/multi-tenant-saas-storage-strategies.html)
- [Partitioning Pooled Multi-Tenant SaaS Data with DynamoDB](https://aws.amazon.com/blogs/apn/partitioning-pooled-multi-tenant-saas-data-with-amazon-dynamodb/)
- [Multi-Tenant Data Isolation with PostgreSQL Row Level Security](https://aws.amazon.com/blogs/database/multi-tenant-data-isolation-with-postgresql-row-level-security/)
- [Scale Your Relational Database for SaaS — Sharding and Routing](https://aws.amazon.com/blogs/database/scale-your-relational-database-for-saas-part-2-sharding-and-routing/)
- [Managed Database Backup and Recovery in Multi-Tenant SaaS](https://aws.amazon.com/blogs/database/managed-database-backup-and-recovery-in-a-multi-tenant-saas-application/)
- [AWS Financial Services Industry Lens](https://docs.aws.amazon.com/wellarchitected/latest/financial-services-industry-lens/financial-services-industry-lens.html)
