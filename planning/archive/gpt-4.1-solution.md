**AWS Raffle System Architecture Plan (GLI-31 Compliant)**

---

## 1. Overview

Design a scalable, secure, cost-effective raffle system on AWS, compliant with GLI-31 standards (notably secure, auditable RNG and immutable logging). The system is serverless-first where possible, with options for containerization for core logic if required for maintainability or performance.

---

## 2. System Components

### A. Core RNG (Random Number Generation)

- **AWS Lambda** (stateless, event-driven execution)
- **AWS Key Management Service (KMS) Custom Key Store** backed by **AWS CloudHSM**
  - Use CloudHSM for FIPS 140-2 Level 3 hardware-backed entropy.
  - RNG logic runs in Lambda, which calls CloudHSM for true random numbers.
  - Lambda code is immutable, versioned, and signed.

Alternatives:
| Solution | FIPS Level | Cost | Complexity | GLI-31 Suitability |
|----------|------------|------|------------|-------------------|
| KMS + CloudHSM (current) | FIPS 140-2 Level 3 ✅ | High 💰💰💰 | High ⚠️⚠️⚠️ | Fully compliant ✅ |
| AWS KMS standard (no HSM) | FIPS 140-2 Level 2 | Low 💰 | Low ✅ | Possibly acceptable ⚠️ (Verify) |
| External Certified RNG Service | FIPS 140-2 Level 3 ✅ | Moderate 💰💰 | Moderate ⚠️⚠️ | Likely acceptable ✅ (Verify) |
| AWS Nitro Enclaves | FIPS 140-2 Level 2+ | Moderate 💰💰 | Moderate-High ⚠️⚠️ | Possibly acceptable ⚠️ (Verify) |

### B. Core Logic Engine

- **AWS Lambda** (preferred for simplicity/cost) or **AWS ECS with Fargate** (if stateful/complex logic)
- **AWS EventBridge** for inter-component orchestration and decoupling.
- All logic code is version-controlled, with cryptographic signing and deployment via CI/CD, ensuring deterministic, auditable outcomes.

Alternatives:
| Solution | Compliance Strength | Cost 💰 | Complexity 🛠️ | When to Use |
| ----------------------- | -------------------------------- | ------------------ | ------------------ | -------------------------------- |
| Lambda (current pref.) | Excellent ✅ | Low 💰 | Low ✅ | Default choice (recommended) |
| ECS/Fargate | Excellent ✅ | Moderate-high 💰💰 | Moderate-high ⚠️⚠️ | Logic exceeds Lambda limits only |
| Step Functions + Lambda | Excellent (State Tracking) ✅ | Moderate 💰💰 | Moderate ⚠️ | Complex stateful workflows |
| App Runner | Good (container immutability) ✅ | Moderate 💰💰 | Moderate-low ⚠️ | Simple container, low infra mgmt |
| EC2 | Good (with effort) ⚠️ | High 💰💰💰 | High ⚠️⚠️⚠️ | Not recommended |

### C. Immutable Data Ledger

- **Amazon QLDB (Quantum Ledger Database)**
  - All draw events, RNG outputs, ticket sales, and winner selections are recorded immutably.
  - QLDB provides cryptographic verification (hash-chained journal).
  - Supports audit queries and verification for GLI-31.

Alternatives:
| Solution | Immutability / Compliance | Cost 💰 | Complexity 🛠️ | Auditability 📝 | GLI-31 Suitability |
|----------|--------------------------|---------|---------------|-----------------|-------------------|
| QLDB (current) | Excellent ✅ | Moderate 💰💰 | Moderate-low ✅ | Built-in cryptographic verification ✅ | Fully compliant ✅ |
| DynamoDB Append-only (manual) | Good (requires manual proof) ⚠️ | Low 💰 | Moderate-high ⚠️⚠️ | Requires additional Lambda logic ⚠️ | Possibly compliant ⚠️ (verify) |
| S3 Object Lock (Compliance Mode) | Excellent ✅ | Very low 💰 | Low ✅ | Requires external indexing ⚠️ | Likely compliant ✅ (verify) |
| Managed Blockchain (Fabric) | Excellent ✅ | High 💰💰💰 | High ⚠️⚠️⚠️ | Excellent built-in verification ✅ | Fully compliant (but costly) ⚠️ |

### D. Audit & Verification

- **AWS Lambda** for periodic and event-driven audits.
- **CloudWatch Logs** for operational auditing.
- **QLDB Streams** to push immutable events to audit processors or data lake.
- **S3 Glacier Deep Archive** for long-term, immutable cold storage of logs.
- **CloudWatch Alarms/Events** for anomaly detection.

Recommendations:
| Service | Recommendation | Reasoning |
|---------|---------------|-----------|
| AWS Lambda | ✅ (Keep) | Cost-effective, fully managed audit logic execution |
| CloudWatch Logs | ✅ (Keep, minimal logging enabled) | Essential logging for operational audits, minimal cost |
| QLDB Streams → S3 | ✅ (Use S3 Glacier or Glacier Deep Archive) | Cheapest long-term, immutable cold storage |
| CloudWatch Alarms | ✅ (Keep basic alarms) | Essential anomaly detection, low cost |

### E. Integration Interface

- **Amazon API Gateway** (REST/HTTP API)
- **Lambda Authorizer** for token-based authentication (JWT/OAuth2/cert).
- **Lambda** for request processing.
- **EventBridge** for event-based integrations (e.g., external notification, payment, etc.).
- **WAF (Web Application Firewall)** for protection against common web exploits.

Recommendations:
| Service | Recommendation | Reasoning |
|---------|---------------|-----------|
| API Gateway | ✅ (HTTP API preferred for lower cost) | Cost-effective compared to REST APIs |
| Lambda Authorizer | ✅ (JWT via Cognito recommended) | Integrated, low-cost authentication, highly secure |
| AWS Lambda | ✅ (Keep, lightweight request handling) | Lowest possible execution cost, auto-scaling |
| EventBridge | ✅ (Keep) | Efficient decoupling, extremely low cost |
| AWS WAF | ✅ (Basic rules only) | Minimal cost, protects against common web threats |

### F. Observability & Security

- **AWS CloudWatch** (metrics, logs, alarms)
- **AWS X-Ray** (distributed tracing)
- **AWS IAM** (least-privilege roles)
- **AWS WAF**
- **AWS GuardDuty** (threat detection)
- **AWS Security Hub** (centralized security posture)
- **AWS Config** (continuous compliance monitoring)

Recommendations:
| Service | Recommendation | Reasoning |
|---------|---------------|-----------|
| AWS CloudWatch | ✅ (Logs, Metrics, Alarms with minimal retention) | Essential observability, low cost if minimal retention |
| AWS X-Ray | ✅ (Keep basic sampling enabled only) | Essential minimal tracing, avoid full sampling |
| AWS IAM | ✅ (Least privilege, minimal roles) | Essential security, no added cost |
| AWS WAF | ✅ (Minimal ruleset, managed rules only) | Security compliance at lowest possible cost |
| AWS GuardDuty | ⚠️ (Enable Basic) | Essential minimal threat detection |
| AWS Security Hub | ❌ (Optional; skip unless explicitly required) | Higher cost, consider enabling only if GLI mandates |
| AWS Config | ⚠️ (Minimal rule set for compliance) | Only minimal compliance rules, reduce cost significantly |

### G. Infrastructure & DR

- **AWS CDK or Terraform** for infrastructure as code (IaC) and easy deployments.
- **VPC** with private subnets for all compute/storage.
- **ACM (AWS Certificate Manager)** for TLS.
- **Multi-AZ** deployment for all stateful services (e.g., QLDB, ECS if used).
- **Backup Strategy**: QLDB point-in-time recovery, Lambda code versioning, and S3 backups.
- **DR**: Cross-region replication for QLDB streams and S3; infrastructure redeployable via IaC.

Recommendations:
| Service | Recommendation | Reasoning |
|---------|---------------|-----------|
| AWS CDK or Terraform | ✅ (CDK preferred if using AWS extensively) | Essential for consistent, auditable deployments |
| VPC | ⚠️ (Minimal private subnet for compliance-critical services only, like CloudHSM if used) | Avoid VPC unless explicitly required for compliance |
| ACM (TLS Certificates) | ✅ (Essential for API endpoints only) | Zero cost, essential security |
| Multi-AZ deployments | ✅ (Only if automatically provided by AWS managed services) | Avoid manual multi-AZ unless explicitly required |
| Backup Strategy | ✅ (Point-in-time QLDB recovery built-in) | Essential, minimal-cost QLDB native backups |
| Lambda Code Versioning | ✅ (Built-in, zero extra cost) | Essential, no additional cost |
| S3 Cross-region backup | ⚠️ (Use only if explicitly mandated by GLI; minimal cross-region backups) | Avoid unless explicitly required (significant cost) |
| Cross-region DR | ⚠️ (Infrastructure IaC only, no hot standby unless GLI explicitly requires) | Deployable IaC DR, no active hot DR to save cost |

---

## 3. Data Flow

1. **Ticket Purchase**
   - API Gateway → Lambda (auth & validation) → QLDB (record purchase) → EventBridge (emit purchase event)
2. **Raffle Draw**
   - Scheduled EventBridge event → Lambda (core logic) → Lambda (Core RNG) → CloudHSM-generated random number
   - Lambda (core logic) records draw and winner in QLDB (with hash of RNG output).
   - EventBridge event triggers notifications, audit Lambda, etc.
3. **Audit Trail**
   - QLDB streams all events to S3 (data lake) for offline audit.
   - CloudWatch logs every operation; alarms for anomalies.
   - S3 Glacier Deep Archive for long-term retention, immutability.
4. **API Integration**
   - API Gateway endpoint with WAF and Lambda Authorizer.
   - Downstream Lambda functions interact with QLDB and EventBridge.
5. **Observability & Security**
   - All Lambda/ECS logs to CloudWatch.
   - X-Ray traces for performance and anomaly detection.
   - GuardDuty monitors VPC; Security Hub aggregates findings.

---

## 4. Technology Stack

- **Compute**: AWS Lambda (primary), ECS Fargate (optional for advanced logic)
- **Event Bus**: EventBridge
- **Ledger**: Amazon QLDB
- **RNG**: CloudHSM via Lambda (integrated with KMS Custom Key Store)
- **API**: API Gateway with Lambda Authorizer
- **Audit**: CloudWatch, QLDB Streams, S3/Glacier
- **Security**: IAM, WAF, GuardDuty, Security Hub, CloudHSM, ACM
- **IaC**: AWS CDK or Terraform
- **Monitoring**: CloudWatch, X-Ray, Config

---

## 5. GLI-31 Compliance Points

| Requirement                  | Solution Element                                         |
| ---------------------------- | -------------------------------------------------------- |
| Hardware RNG (FIPS 140-2 L3) | CloudHSM + KMS Custom Key Store + Lambda RNG function    |
| Immutable Ledger             | QLDB, S3 Glacier, versioned Lambda code                  |
| Audit Trail                  | QLDB Streams, CloudWatch Logs, S3 Glacier, Lambda audits |
| Event Integrity              | EventBridge, QLDB, cryptographic hash chains             |
| Security Monitoring          | GuardDuty, Security Hub, IAM, WAF, VPC isolation         |
| Infrastructure Resilience    | IaC, Multi-AZ, DR, QLDB recovery, S3 cross-region backup |

---

## 6. Integration Points

- **External applications**: API Gateway (authenticated, WAF-protected endpoints)
- **Payment providers**: EventBridge or direct API integrations (never direct to QLDB)
- **Audit teams**: Read-only access to QLDB, S3 audit logs, CloudWatch Logs Insights
- **Operations/Security**: Security Hub dashboard, IAM roles, CloudWatch dashboards

---

## 7. Deployment & Maintenance

- **IaC (CDK/Terraform)** enables repeatable, versioned deployments.
- **CI/CD pipeline** for Lambda/ECS code, with cryptographic signing for compliance.
- **Automated tests** and canary deploys.
- **Monitoring and alerting** for all critical paths.
- **Routine DR drills** and backup validation.
- **Centralized secrets management** via Secrets Manager or Parameter Store.

---

## 8. Cost-Effectiveness

- **Serverless-first** approach minimizes idle costs and scales automatically.
- **QLDB** is pay-per-use, only stores what is required.
- **S3 Glacier** for long-term, low-cost archive storage.
- **ECS/Fargate** only if needed for complex logic.

---

## 9. Diagram (Text Description)

```
[User] -> [API Gateway + WAF + Lambda Authorizer] -> [Lambda Functions]
     |                                                   |
     |                                                   v
     |---------------------------------------------> [EventBridge]
     |                                                   |
     |                             +---------------------+-----------------+
     |                             |                                       |
     v                             v                                       v
[QLDB (Immutable Ledger)]   [Lambda (Core RNG) + CloudHSM]        [Audit Lambdas/CloudWatch]
     |                             |                                       |
     |                             v                                       v
     +---------------------> [QLDB Streams] -> [S3/Glacier] <--------------+
```

---

## 10. Summary

This architecture is:

- **GLI-31 compliant**: FIPS 140-2 L3 RNG, immutable ledgers, auditable, secure.
- **Cost-effective**: Serverless, managed services, only pay for usage.
- **Maintainable**: IaC, versioned logic, modular.
- **Scalable**: AWS-managed scaling throughout.
- **Secure**: Hardware-backed keys, WAF, VPC, IAM, monitoring.
- **Reliable**: Multi-AZ, DR, backup, immutable logs.
- **Easy to deploy**: CDK/Terraform, CI/CD, blue/green.

---

**Next steps:**

- Finalize detailed security controls and access patterns.
- Create a deployment pipeline (CDK/Terraform) with all compliance checks.
- Prepare runbooks and audit documentation for GLI-31 certification.
