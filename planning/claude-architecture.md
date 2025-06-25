# Electronic Raffle System Architecture Document
## GLI-31 Compliant Design

### Document Purpose
This architecture document outlines the technology choices and system design for an electronic raffle system that will be certified to the GLI-31 standard. It serves as a reference for developers to understand the overall system architecture, technology decisions, and compliance considerations.

### 1. Executive Summary

The Electronic Raffle System is built on AWS infrastructure using a serverless architecture pattern with Lambda functions primarily written in Go. The system is designed to meet GLI-31 compliance requirements while maintaining scalability, security, and operational efficiency. The system targets an 8-hour Recovery Time Objective (RTO) and 4-hour Recovery Point Objective (RPO).

### 2. Cloud Infrastructure

**Primary Cloud Provider**: Amazon Web Services (AWS)
- Rationale: Comprehensive service ecosystem, strong security compliance, global availability
- Third-party services will be evaluated on a case-by-case basis for specific functionality

### 3. System Architecture Overview

#### 3.1 Core Services (Lambda Functions - Go Runtime)

All critical services are implemented as Lambda functions using Go runtime for:
- Static compilation enabling easy checksum validation
- Cross-region deployment consistency
- Performance efficiency
- Strong typing for critical financial calculations

**Components:**

1. **Raffle Core Service**
   - Manages raffle lifecycle (creation, configuration, closing)
   - Ticket allocation and validation
   - Draw number generation and winner determination
   - RSU device management and authorization
   - Tracks ticket allocation per RSU in PostgreSQL
   - Critical file component per GLI requirements

2. **RNG Service**
   - Uses Go's crypto/rand library (GLI approved)
   - No state persistence required - crypto/rand manages entropy internally
   - Provides cryptographically secure random numbers for draws
   - Implements proper scaling algorithms for draw ranges
   - Critical file component per GLI requirements

3. **Payment Service**
   - Handles validating payment intents from Stripe, processing webhooks, and communicating with Raffle Core Service
   - Handles payment reconciliation
   - Normalizes payment data before Raffle Core Service interaction
   - Stores payment reference IDs only (per GLI guidance)
   - Designed for provider portability

4. **Reporting Service**
   - Generates all GLI-31 required reports (section 2.8)
   - Provides audit trails and exception reports
   - Supports jurisdictional reporting requirements

#### 3.2 Data Storage

**Primary Database**: Aurora PostgreSQL (within VPC)
- Stores all raffle transactional data
- Tracks RSU ticket allocations
- Automated backups every 5 minutes (point-in-time recovery) ** if feasible for cost
- Multi-AZ deployment for high availability
- Aurora Global Database for disaster recovery (cross-region) ** if feasible for cost

**Critical Event Storage**: DynamoDB
- Stores all critical events and audit logs generated from Raffle Core Service
- Configurable TTL for jurisdictional requirements (default 3 years)
- DynamoDB Streams trigger archival process
- Global Tables for multi-region availability (future requirement)

**Archive Storage**: S3 with Glacier storage class
- Long-term storage of expired critical events
- Cross-region replication enabled
- Lifecycle policies for automatic transitions
- Compliance with data retention requirements

**Configuration Storage**: SSM Parameter Store
- Expected checksums for verification
- System configuration parameters
- Secure credential storage
- Automatic backup via AWS Config

#### 3.3 Frontend Applications

1. **Public Website**
   - Built with Amplify/CloudFront CDN
   - Supports ticket purchases
   - Displays raffle information and results
   - Shows system verification status for regulators
   - Displays current checksum values for transparency

2. **Administrator Website**
   - Built with Amplify/CloudFront CDN
   - Full raffle management capabilities
   - Reconciliation confirmation interface
   - System monitoring and reporting
   - User and RSU management

3. **RSU Android Application**
   - Expo framework for rapid development
   - PAX OS integration for payment processing
   - Android Package (critical file) for backend communication
   - Local SQLite for offline ticket storage
   - Automatic sync when connection restored
   - Displays package checksum for verification

#### 3.4 API and Security Layer

**API Gateway** (Internet-facing)
- Single entry point for all system access
- JWT authorizer for Cognito tokens
- Public routes for specific operations
- Request/response validation
- API key management for RSU devices

**Web Application Firewall (WAF)**
- Application-level firewall per GLI-31 requirements
- Audit logs stored in S3 bucket with:
  - CloudWatch Logs integration for real-time monitoring
  - S3 lifecycle policies (never expire audit logs)
  - CloudWatch alarms for S3 write failures
  - SNS notifications if log delivery fails
- DDoS protection via AWS Shield
- Custom rule sets for raffle-specific security

**Authentication**: AWS Cognito
- User pools for public users (optional accounts)
- Admin user pool with MFA required
- RSU operator authentication with device binding
- Integration with IAM for service permissions

#### 3.5 System Verification and Monitoring

**Verification Lambda** (Scheduled)
- Runs every hour (configurable)
- Validates Lambda function checksums in S3
- Compares against expected values in Parameter Store
- Publishes verification reports to public S3 bucket
- Sends SNS notifications on mismatch detection
- Prevents new raffle creation on verification failure

**Monitoring**: CloudWatch
- System-wide logging and metrics
- Custom metrics for business operations
- Alarm configuration for critical events
- Log retention per compliance requirements
- Integration with AWS X-Ray for distributed tracing

**Archive Lambda**
- Subscribes to DynamoDB Streams
- Archives deleted critical events to S3 Glacier
- Maintains audit trail continuity
- Implements checksum validation on archives

**Time Synchronization**
- AWS Time Sync Service (chrony) for all compute resources
- Automatic NTP synchronization
- Sub-millisecond accuracy
- No additional configuration required

### 4. Network Architecture

- **VPC Configuration**:
  - Isolated network for internal services
  - Multi-AZ deployment across 2 availability zones (can be expanded in the future)
  - Private subnets for databases and Lambda functions

- **Internet Gateway**: Enables outbound connections for:
  - Third-party payment provider APIs
  - External service integrations

- **API Gateway**: Only public-facing endpoint

### 5. Critical File Management

Per GLI requirements, the following components are designated as critical files:

1. **RNG Package**
   - Implements crypto/rand wrapper
   - Scaling algorithms for draw ranges

2. **Verification Package**
   - Self-verification routines
   - Checksum calculation and comparison

3. **Database Package**
   - All write operations to raffle data
   - Transaction management

4. **Validation Package**
   - UUID generation for ticket validation numbers
   - Validation number verification

5. **Raffle Logic Package**
   - Ticket allocation and draws
   - Winner determination
   - Refund processing

### 6. Disaster Recovery and Backup Strategy

**Recovery Objectives**:
- RTO (Recovery Time Objective): 8 hours
- RPO (Recovery Point Objective): 4 hours

**Backup Strategy**:

Ideal state, but might not be feasible for costs, so different strategies can be considered.

1. **Aurora PostgreSQL**:
   - Continuous backup to S3 (5-minute recovery points)
   - Daily automated snapshots retained for 35 days
   - Aurora Global Database with read replica in secondary region
   - Automated failover capability

2. **DynamoDB**:
   - Point-in-time recovery enabled (35-day window)
   - Global Tables for multi-region replication
   - On-demand backups before major changes

3. **S3 Data**:
   - Cross-region replication for all critical buckets
   - Versioning enabled
   - MFA delete protection for production buckets

4. **Infrastructure**:
   - All infrastructure defined in Terraform
   - Code repositories in Github
   - Automated deployment pipelines for rapid recovery
   - Runbooks for disaster recovery procedures

**Recovery Procedures**:
- Automated health checks trigger failover processes
- SNS notifications to on-call team
- Step-by-step runbooks in AWS Systems Manager
- Regular DR drills (quarterly)

### 7. RSU Offline Capabilities

**Local Storage**:
- SQLite database for ticket data
- Stores allocated ticket ranges
- Maintains sale records during offline periods
- Encryption at rest using Android Keystore

**Synchronization**:
- Automatic sync when connection restored
- Conflict resolution favors local sales
- Batch upload of offline transactions
- Critical event logging for offline periods

**Limitations**:
- Cannot receive new ticket allocations while offline
- Cannot process refunds offline
- Must sync before draw can occur

### 8. Technology Standards

**Primary Language**: Go
- Used for all critical file components
- Version: Latest stable (currently 1.21+)
- Exceptions allowed for:
  - Frontend applications (JavaScript/TypeScript)
  - Android application (Kotlin/Java/TypeScript)
  - Infrastructure as Code (Terraform)

**Code Organization**:
- Critical packages isolated in separate modules
- Clear API boundaries between packages
- Comprehensive unit test coverage (>80% for critical files)
- Integration tests for all API endpoints

### 9. Deployment and Operations

**Infrastructure as Code**:
- Terraform and TerraGrunt
- All infrastructure version controlled
- Environment separation (dev/prod)

**CI/CD Pipeline**:
- Github Actions
- Automated testing including:
  - Unit tests for all critical files
  - Integration tests
  - Security scanning (static and dynamic)
  - GLI compliance checks
- Checksum generation and storage
- Blue/green deployments for zero downtime

**Operational Procedures**:
- 24/7 monitoring via CloudWatch
- On-call rotation for critical issues
- Automated alerting for system anomalies
- Monthly security patching windows

### 10. Compliance Controls

**GLI-31 Specific Implementations**:

1. **Communication Security**:
   - All API communications use TLS 1.2+
   - Certificate pinning for RSU applications
   - API key rotation every 90 days

2. **Audit Logging**:
   - Comprehensive CloudTrail logging
   - Application-level audit logs in CloudWatch
   - Immutable log storage in S3
   - Real-time log analysis with CloudWatch Insights

3. **Access Controls**:
   - MFA required for all administrative access
   - Principle of least privilege via IAM
   - Regular access reviews (quarterly)
   - Session timeout after 30 minutes of inactivity

4. **Data Integrity**:
   - Database transactions for all critical operations
   - Checksum validation for all data transfers
   - Reconciliation processes before draws
   - Daily integrity checks on stored data

### 11. Decision Log

| Decision | Rationale |
|----------|-----------|
| AWS as primary cloud | Comprehensive services, compliance capabilities |
| Go's crypto/rand for RNG | GLI approved, no external dependencies |
| Aurora PostgreSQL | Managed service, automatic backups, global database |
| Serverless architecture | Scalability, reduced operational overhead, faster recovery |
| AWS WAF for firewall | Meets GLI-31 requirements, unlimited log storage |
| Time Sync Service | Automatic synchronization, meets GLI requirements |
