## 1. **System Overview**

**Key Principles:**
- Modular, serverless-first architecture for easy scaling, reliability, and cost control
- Immutable, verifiable records and audit trails for GLI-31
- Strong authentication and secure networking
- Automated compliance monitoring and reporting

---

## 2. **System Components & Services**

| Component                | AWS Services                                                                                     | Purpose/Notes                                                                                                      |
|--------------------------|--------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| **Auth & User Mgmt**     | Cognito, IAM, VPC Endpoints, VPN                                                                 | Secure user/device auth (users, admins, RSUs), least-privilege access                                              |
| **API Layer**            | API Gateway (+ WAF), VPC Link, Private APIs                                                      | Secure, scalable API exposure; firewall for OWASP Top 10 coverage                                                  |
| **Serverless Compute**   | Lambda (Go), Step Functions (for orchestrating multi-step compliance checks), EventBridge         | Core business logic, RNG, reporting, verification, anomaly detection, scheduling                                   |
| **Database**             | Aurora PostgreSQL (Multi-AZ, encrypted), DynamoDB (immutable event log, append-only)             | Strong consistency, relational integrity, immutable event logging                                                  |
| **Config & Secrets**     | SSM Parameter Store (encrypted), Secrets Manager                                                 | Store config, self-verification checksums, secrets                                                                 |
| **Audit, Logs, Trails**  | CloudWatch Logs, CloudWatch Alarms, DynamoDB (append-only), S3 Glacier (archive), AWS Config     | Real-time logs, alarms, immutable event/audit trail, long-term archive, compliance snapshots                       |
| **Monitoring & Alerting**| CloudWatch, SNS, Security Hub, GuardDuty, CloudTrail                                             | Monitoring, compliance, SIEM, alerting, anomaly detection                                                          |
| **Website & Portal**     | Amplify (CDK/CodePipeline deploys), S3 (static assets), CloudFront, ACM                          | Static hosting, admin/public portals, HTTPS, global scale                                                          |
| **Mobile/Edge**          | Android RSU app (offline mode, package updates, local self-verification, periodic sync)           | Secure, certifiable, offline-first, syncs via VPN or HTTPS when available                                          |
| **Networking**           | VPC, Subnets, Security Groups, VPC Endpoints, Site-to-Site VPN, Transit Gateway (if scaling VPNs)| Secure, segmented, private communication; RSUs connect via VPN; all AWS resources in private subnets                |
| **Infrastructure as Code**| Terraform, Terragrunt, AWS CodeBuild/CodePipeline                                              | Repeatable, auditable, easy deployments                                                                            |
| **Encryption**           | KMS (CMK, HSM-backed), S3 encryption, Aurora/DynamoDB encryption                                | Data at rest/in transit, FIPS 140-2 L3 via CloudHSM if highest compliance needed                                   |

---

## 3. **Data Flow**

### **A. Raffle/Lottery Lifecycle**

1. **User/RSU Authenticates** (Cognito, VPN for RSUs)
2. **API Request** (API Gateway + WAF → Lambda)
3. **Core Logic** (Lambda → Aurora for state, DynamoDB for immutable event log)
4. **RNG** (Lambda, seeded from HSM/KMS-generated entropy)
5. **Event Logging** (DynamoDB append-only, CloudWatch logs, S3 archival)
6. **Verification** (Scheduled Lambda runs compliance checks, hashes logs, posts self-verification hash to public site)
7. **Incident/Anomaly Detection** (CloudWatch Alarms, SNS notifications, lockout on failure)
8. **Admin/Regulator Access** (Amplify portal, view audit/compliance reports and verification hashes)
9. **RSU Sync** (Android app syncs via VPN or HTTPS to central API, submits status, receives updates)
10. **Archive** (Periodic export of logs/events to S3 Glacier, with signed hash for immutability)

---

## 4. **Networking Architecture**

### **A. VPC Design**
- **Single VPC** (or multiple for dev/prod isolation)
  - **Public Subnets**:
    - ALB/NLB for API Gateway VPC endpoints (if using private API Gateway)
    - Bastion hosts (if needed, strongly locked down)
  - **Private Subnets**:
    - Lambda (placed in VPC, no public internet)
    - Aurora PostgreSQL (no public IP)
    - DynamoDB via VPC endpoint
    - SSM, KMS, S3 endpoints for secure private access
  - **VPN Gateway**:
    - For RSU devices to connect securely (Site-to-Site or Client VPN)
    - Optionally use Transit Gateway if scaling VPN connections

- **Security Groups**:
  - Minimum necessary ports (e.g., 443, 3306 for Aurora, only from Lambda SG)
  - Principle of least privilege

- **NACLs**:
  - Extra layer for subnet-level controls

- **WAF**:
  - Protect API Gateway from common web attacks

---

## 5. **Additional AWS Services/Enhancements**

- **AWS CloudTrail**:
  - Full API/audit logging for all AWS actions—critical for GLI-31
- **AWS Security Hub & GuardDuty**:
  - Centralized threat detection, compliance checks
- **AWS Certificate Manager (ACM)**:
  - Automated SSL/TLS for endpoints
- **AWS Key Management Service (KMS)/CloudHSM**:
  - HSM-backed keys for RNG, FIPS 140-2 L3 compliance
- **AWS Step Functions**:
  - For orchestrating multi-step compliance/self-verification tasks
- **AWS Glue**:
  - If you want to enable analytics/reporting over event logs
- **AWS S3 Object Lock**:
  - For WORM (write-once-read-many) archival of logs (meets immutability)
- **Amazon EventBridge**:
  - For decoupled event-driven architecture, scheduled jobs, compliance triggers

---

## 6. **Self-Verification & Immutability**

- **Self-Verification Lambda**:
  - Runs scheduled checks on all service logs, state, code hashes
  - Posts a signed, immutable checksum to the Amplify public site
  - Failed checks trigger SNS alerts, system lockout, and admin notifications

- **Immutable Audit Trail**:
  - All events written to DynamoDB (append-only, no updates)
  - Periodic export to S3 with Object Lock (WORM) and signed hash
  - All logs and audit data are cryptographically signed and available for regulator verification

- **Android App Self-Verification**:
  - App verifies its own critical assets (checksums, signed)
  - Reports status on check-in; failure disables operations until resolved

---

## 7. **Deployment & Maintainability**

- **Terraform + Terragrunt**:
  - All infrastructure as code for repeatable, auditable deployments
- **CI/CD Pipeline with CodePipeline/CodeBuild**:
  - Automated code and infra deployment, integration with Amplify for website/portal
- **Monitoring/Dashboard**:
  - CloudWatch Dashboards for system, compliance, and security status

---

## 8. **Cost Optimization**

- **Serverless** (Lambdas, DynamoDB, S3):
  - Scales to zero, only pay for usage
- **Aurora Serverless v2** (if workload is bursty)
- **Right-size instance types, use reserved capacity for DB if predictable**
- **S3 Glacier for archival**

---

## 9. **Summary Diagram (Textual)**

```
RSU (Android App) --VPN--> API Gateway (WAF, Auth) --Lambda (Go)--> Aurora Postgres
                                            |                     |
                                         DynamoDB (Event Log)     |
                                            |                     |
                                         S3 (Archive) <-----------|
                                            |
                                      CloudWatch/SNS/Config
                                            |
                                     Admin/Regulator Portal (Amplify)
```

---

## 10. **Key Integration Points**

- **API Gateway–Lambda**: All business logic, API exposure
- **Lambda–Aurora/DynamoDB**: Transaction/state management
- **Lambda–KMS/CloudHSM**: RNG, cryptographic ops
- **Lambda–CloudWatch/SNS**: Logging, monitoring, alerting
- **Amplify–S3**: Host signed verification results, reports
- **VPN–VPC**: Secure RSU integration

---

## 11. **Final Recommendations**

- Favor DynamoDB (append-only) for audit trails (with periodic S3 WORM backup) over CloudWatch alone, for true immutability
- Use S3 Object Lock for regulatory archive
- Use Step Functions for orchestrating complex compliance checks/self-verification
- Use Security Hub, GuardDuty, and Config for continuous compliance monitoring
- Use CloudTrail for all AWS API activity logging
- Use CloudHSM for RNG if GLI-31 scrutiny/fips 140-2 L3 is strict
- Use Aurora Serverless for cost savings if DB load is not constant

---
