# Electronic Raffle System Architecture Document

## GLI-31 Compliant Design

### Document Purpose

This architecture document outlines the high-level technology choices and system design for an electronic raffle system that will be certified to the GLI-31 standard. It serves as a reference for developers to understand the overall system architecture, technology decisions, and compliance considerations.

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
- Manages complete raffle lifecycle and state transitions
- Handles ticket generation, allocation, and sales
- Processes draws and determines winners
- Manages RSU device enrollment and synchronization
- Implements reconciliation workflows
- Critical file component per GLI requirements
1. **RNG Service**
- Provides cryptographically secure random numbers
- Uses Go’s crypto/rand library (GLI approved)
- Stateless design for simplicity
- Critical file component per GLI requirements
1. **Payment Service**
- Integrates with payment providers (initially Stripe)
- Processes payment webhooks
- Maintains payment reference data only
- Designed for provider portability
1. **Reporting Service**
- Generates all GLI-31 required reports
- Provides audit trails and compliance reports
- Supports jurisdiction-specific requirements

#### 3.2 Data Storage

**Primary Database**: Aurora PostgreSQL

- All transactional raffle data
- Pre-generated ticket inventory
- Complex state management
- High availability with automated backups

**Critical Event Storage**: DynamoDB

- Audit logs and critical events
- Short-term retention with TTL
- Automatic archival to S3

**Archive Storage**: S3 with Glacier

- Long-term audit log retention
- Cross-region replication for compliance
- Lifecycle policies for cost optimization

**Configuration Storage**: SSM Parameter Store

- System configuration
- Expected checksums for verification
- Secure credential management

#### 3.3 Frontend Applications

1. **Public Website**
- Customer ticket purchases
- Raffle information and results
- System verification status display
1. **Administrator Website**
- Complete raffle management
- Reconciliation workflows
- System monitoring and reporting
- User and device management
1. **RSU Android Application**
- Offline-capable ticket sales
- Real-time synchronization
- Push notifications for operators
- Countdown timer for coordinated closing
- Checksum display for verification

#### 3.4 API and Security Layer

**API Gateway**

- Single entry point for all clients
- Authentication and authorization
- Request validation and rate limiting

**Web Application Firewall (WAF)**

- Application-level protection per GLI-31
- Comprehensive audit logging
- DDoS protection

**Authentication**: AWS Cognito

- Multiple user pools (public, admin, operator)
- MFA for administrative access
- Device binding for RSUs

#### 3.5 System Verification and Monitoring

**Verification Lambda**

- Scheduled checksum validation
- Prevents operations on verification failure
- Publishes status for regulatory review

**Monitoring**: CloudWatch

- System metrics and logging
- Business event tracking
- Alerting and notifications

**Time Synchronization**

- AWS Time Sync Service
- Critical for coordinated operations
- Sub-second accuracy

### 4. Network Architecture

- **VPC**: Isolated network for internal services
- **Multi-AZ Deployment**: High availability
- **API Gateway**: Only public-facing endpoint
- **Internet Gateway**: Outbound connections for integrations

### 5. Critical File Components

Per GLI requirements, these components are designated as critical files:

1. RNG algorithm and implementation
1. Control program (self) verification
1. Critical memory handling and validation
1. Generation of ticket validation numbers
1. Raffle Event Logic (ticket allocation, refunds, winner determination)

### 6. Disaster Recovery Strategy

**Recovery Objectives**:

- RTO: 8 hours
- RPO: 4 hours

**Key Strategies**:

- Automated backups for all data stores
- Infrastructure as Code for rapid recovery
- Multi-region capabilities (future phase)
- Regular DR testing procedures

### 7. RSU Offline Capabilities

- Local SQLite storage for offline sales
- Automatic synchronization when online
- Coordinated countdown timers
- Push notifications between devices
- Automatic deactivation on buffer limits

### 8. Technology Standards

**Primary Language**: Go (for all critical components)

- Exceptions: Frontend (JavaScript/TypeScript), Android (Kotlin/Java/TypeScript), IaC (Terraform)

**Infrastructure**:

- Terraform and TerraGrunt for IaC
- Separate repositories for infrastructure and services
- Environment separation (dev/staging/prod)

### 9. Deployment and Operations

**CI/CD Pipeline**:

- Automated testing and security scanning
- Checksum generation for verification
- Blue/green deployments

**Monitoring**:

- 24/7 system monitoring
- Automated alerting
- Regular security patching

### 10. Compliance Controls

**Key GLI-31 Implementations**:

- TLS 1.2+ for all communications
- Comprehensive audit logging
- Manual reconciliation confirmation
- MFA for administrative access
- Regular access reviews
- Data integrity validations

### 11. Key Architectural Decisions

|Decision                  |Rationale                                         |
|--------------------------|--------------------------------------------------|
|AWS as primary cloud      |Comprehensive services, compliance capabilities   |
|Serverless architecture   |Scalability, reduced operational overhead         |
|Go for critical components|Performance, type safety, easy checksum validation|
|Pre-generated ticket model|Ensures ticket uniqueness and allocation control  |
|Separate IaC repositories |Clear separation of concerns                      |
|Aurora PostgreSQL         |Managed service with strong consistency           |

### 12. Future Considerations

- Multi-region deployment for global operations
- Enhanced dynamic pricing capabilities
- Additional payment provider integrations
- Extended offline capabilities for RSUs