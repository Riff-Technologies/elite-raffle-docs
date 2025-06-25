| Component                | AWS Resources                                                   | GLI Compliance Points                  |
| ------------------------ | --------------------------------------------------------------- | -------------------------------------- |
| Core RNG                 | Lambda, KMS Custom Key Store, CloudHSM                          | Hardware-backed RNG (FIPS 140-2 L3)    |
| Core Logic Engine        | Lambda/ECS/Fargate, EventBridge                                 | Immutable logic, deterministic output  |
| Immutable Data Ledger    | QLDB                                                            | Immutable, cryptographic verification  |
| Audit & Verification     | Lambda, CloudWatch, S3 Glacier Deep Archive, QLDB Streams       | Immutable audit trails, anomaly alerts |
| Integration Interface    | API Gateway, Lambda Authorizers, EventBridge                    | Controlled boundary, secure validation |
| Observability & Security | CloudWatch, X-Ray, IAM, WAF, GuardDuty                          | Security monitoring, detailed auditing |
| Infrastructure & DR      | CDK/Terraform, Backup Strategy, VPC isolation, ACM Certificates | Infrastructure resilience and recovery |
