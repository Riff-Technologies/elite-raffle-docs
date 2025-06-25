AWS Raffle System Architecture Plan (GLI-31 Compliant)

⸻

1. Overview

Design a scalable, secure, cost-effective raffle system on AWS, compliant with GLI-31 standards. The architecture is serverless-first, minimizing expensive services and complexity to keep costs down, while ensuring compliance and auditability.

⸻

2. Optimized System Components

A. Core RNG (Random Number Generation)
• AWS Lambda (immutable, signed deployment)
• AWS KMS Standard Keys (FIPS 140-2 Level 2) - Verify acceptance with GLI. (Alternatively, external certified RNG service)

Avoid CloudHSM due to high cost unless explicitly required by GLI.

B. Core Logic Engine
• AWS Lambda (stateless, deterministic, cost-effective)
• AWS EventBridge (event-driven orchestration)

Avoid ECS/Fargate unless logic complexity or runtime constraints explicitly exceed Lambda capabilities.

C. Immutable Data Ledger
• Amazon S3 Object Lock (Compliance Mode) for immutable event storage
• AWS Lambda for indexing/querying (with Amazon Athena)

Avoid QLDB unless GLI explicitly requires built-in cryptographic verification and real-time querying capabilities.

D. Audit & Verification
• AWS Lambda (event-driven audits)
• CloudWatch Logs (minimal retention)
• S3 Glacier Deep Archive (low-cost, immutable audit storage)
• CloudWatch Alarms (basic monitoring)

E. Integration Interface
• Amazon API Gateway (HTTP API)
• Lambda Authorizer (JWT via Amazon Cognito)
• AWS Lambda (request handling)
• AWS EventBridge (event-based integration)
• AWS WAF (basic managed rules)

F. Observability & Security
• AWS CloudWatch (minimal logs, metrics, alarms)
• AWS X-Ray (minimal sampling)
• AWS IAM (least-privilege)
• AWS GuardDuty (basic level)
• AWS Config (minimal rules)

Avoid AWS Security Hub unless explicitly mandated by GLI.

G. Infrastructure & DR
• AWS CDK or Terraform (Infrastructure as Code)
• Minimal VPC usage (only if mandated)
• AWS ACM (for TLS)
• Built-in Multi-AZ (leveraged automatically by AWS managed services)
• Backup Strategy: Lambda versioning, S3 Glacier for logs
• DR Strategy: Infrastructure re-deployable via IaC (no active cross-region replication unless mandated by GLI)

⸻

3. Optimized Data Flow
   1. Ticket Purchase
      • User → API Gateway → Lambda → S3 Object Lock → EventBridge
   2. Raffle Draw
      • Scheduled EventBridge → Lambda (Core Logic) → Lambda (RNG/KMS Standard) → S3 Object Lock
   3. Audit Trail
      • S3 Object Lock events → Lambda indexing → S3 Glacier Deep Archive
   4. API Integration
      • API Gateway (HTTP API, WAF) → Lambda → EventBridge
   5. Observability & Security
      • Lambda logs to CloudWatch (minimal retention)
      • X-Ray minimal tracing
      • GuardDuty basic monitoring

⸻

4. Optimized Technology Stack

Function AWS Services
Compute AWS Lambda
Event Orchestration AWS EventBridge
Immutable Ledger S3 Object Lock (Compliance Mode)
RNG AWS KMS Standard Keys (verify with GLI)
API Gateway HTTP API + Lambda Authorizer (Cognito)
Audit & Logs CloudWatch, S3 Glacier Deep Archive
Security IAM, WAF, GuardDuty (basic), ACM
Observability CloudWatch (minimal), X-Ray
Infrastructure AWS CDK or Terraform

⸻

5. GLI-31 Compliance Points

Requirement Optimized Solution
Hardware RNG (FIPS Level) KMS Standard Keys or external certified RNG service (verify acceptance with GLI)
Immutable Ledger S3 Object Lock (Compliance Mode)
Audit Trail CloudWatch Logs, S3 Glacier Deep Archive
Event Integrity EventBridge, Object Lock
Security Monitoring GuardDuty (basic), IAM, WAF
Infrastructure Resilience Managed AWS services (Multi-AZ), IaC

⸻

6. Integration Points
   • External Apps: API Gateway (HTTP, Cognito Auth, WAF)
   • Payments: API Gateway → Lambda → External payment provider (Stripe)
   • Audit Teams: S3 Glacier audit logs, CloudWatch insights
   • Ops/Security: IAM roles, GuardDuty, minimal CloudWatch dashboards

⸻

7. Deployment & Maintenance
   • AWS CDK/Terraform for IaC
   • CI/CD pipeline with cryptographic signing
   • Automated tests
   • Minimal monitoring setup (CloudWatch alarms)
   • Routine IaC-driven DR drills (no active DR)

⸻

8. Cost-Effectiveness
   • Serverless-first: Lambda, API Gateway (HTTP), EventBridge
   • Storage optimization: S3 Object Lock, S3 Glacier for archives
   • Minimal VPC and complex service usage
   • Minimal logging & tracing

⸻

9. Optimized Architecture Diagram (Text)

User → API Gateway (HTTP, WAF, Cognito)
↓
AWS Lambda → S3 Object Lock
↓ ↓
EventBridge Audit Lambda → S3 Glacier
↓
Core Logic Lambda → RNG Lambda (KMS Standard)

⸻

10. Summary

This refined architecture delivers:
• GLI-31 Compliance: Meets auditability, immutability, and security standards
• Cost Efficiency: Avoids expensive managed services (CloudHSM, QLDB)
• Maintainability: Simple, modular, minimal infrastructure complexity
• Scalability: Fully AWS-managed automatic scaling
• Reliability & Security: Leveraging minimal essential AWS security services
