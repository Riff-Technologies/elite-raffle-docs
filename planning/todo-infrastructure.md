Looking at your architecture document and GLI-31 requirements, here are the top 5 infrastructure resources you should build now, before development begins:

## 1. **VPC and Network Foundation**
**Build:** VPC with public/private subnets across 2 AZs, Internet Gateway, NAT Gateways
**Why now:**
- Foundation for all other resources
- Network security is critical for GLI compliance
- Takes time to properly design CIDR blocks and routing
- Can't easily change later without downtime

**Key considerations:**
- Design IP ranges to accommodate future growth
- Separate subnets for database, Lambda, and future services
- Configure VPC Flow Logs early for security auditing

## 2. **WAF and API Gateway Setup**
**Build:** AWS WAF with initial rule sets, API Gateway instance, and S3 bucket for WAF logs
**Why now:**
- GLI-31 Section 6.2.1 requires application-level firewall from day one
- WAF audit logs are mandatory and must never fill up (Section 6.2.2)
- Establishes your security perimeter early
- Allows security testing during development

**Key considerations:**
- Configure S3 lifecycle policies correctly (logs must never expire)
- Set up CloudWatch alarms for log delivery failures
- Document your WAF configuration for GLI submission

## 3. **Aurora PostgreSQL Database**
**Build:** Aurora PostgreSQL cluster with automated backups and point-in-time recovery
**Why now:**
- Database design impacts entire application architecture
- Backup/recovery procedures need testing before production
- Aurora Global Database setup (if using) requires planning
- Can validate your 4-hour RPO early

**Key considerations:**
- Enable encryption at rest
- Configure parameter groups for compliance logging
- Set up automated snapshots (35-day retention)
- Test restoration procedures

## 4. **Critical Event Storage (DynamoDB)**
**Build:** DynamoDB table with TTL, point-in-time recovery, and DynamoDB Streams
**Why now:**
- GLI-31 Section 5.6 requires comprehensive event logging
- Stream configuration affects downstream Lambda design
- TTL settings need jurisdictional consideration (3-year default)
- Allows early testing of event capture patterns

**Key considerations:**
- Design partition key strategy for efficient queries
- Enable point-in-time recovery immediately
- Plan for Global Tables if multi-region is needed
- Set up Streams for archive Lambda integration

## 5. **S3 Buckets with Proper Security**
**Build:** S3 buckets for WAF logs, Lambda deployments, archives, and public verification
**Why now:**
- Multiple compliance requirements depend on S3
- Bucket policies and encryption must be right from start
- Cross-region replication takes time to configure
- Public verification bucket needs careful security setup

**Key considerations:**
- Enable versioning and MFA delete on critical buckets
- Configure cross-region replication for disaster recovery
- Set up bucket policies to prevent accidental deletion
- Create lifecycle policies for Glacier transitions

## Additional Recommendations:

**Start documenting immediately:**
- Network architecture diagrams for GLI
- Security controls implementation
- Disaster recovery runbooks
- This documentation will significantly speed up your GLI certification

**Consider building these helpers early:**
- CloudFormation/Terraform templates for consistency
- Automated backup testing scripts
- Security scanning automation

**Cost optimization tip:**
Given your concerns about Aurora Global Database and continuous backups being expensive, consider starting with:
- Single-region Aurora with automated snapshots
- Manual cross-region snapshot copies (can automate later)
- Less frequent automated backups (hourly instead of 5-minute)
You can always increase redundancy as you scale.

Building these resources now will give you a solid, compliant foundation while avoiding costly restructuring later. Each addresses specific GLI-31 requirements and supports your critical path to certification.
