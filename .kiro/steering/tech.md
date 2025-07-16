# Technology Stack & Build System

## Primary Technology Stack

### Backend Services

- **Language**: Go (primary for all critical components)
- **Runtime**: AWS Lambda functions with Go runtime
- **Database**: Aurora PostgreSQL for transactional data
- **Cache/Events**: DynamoDB for critical event logging with TTL
- **Storage**: S3 with Glacier for long-term audit log retention
- **Configuration**: SSM Parameter Store for system configuration

### Frontend Applications

- **Public Website**: JavaScript/TypeScript for customer ticket purchases
- **Admin Website**: JavaScript/TypeScript for raffle management and reconciliation
- **RSU Android App**: Kotlin/Java/TypeScript for offline-capable ticket sales

### Infrastructure

- **Cloud Provider**: Amazon Web Services (AWS)
- **Infrastructure as Code**: Terraform and Terragrunt
- **API Gateway**: Single entry point with authentication and rate limiting
- **Authentication**: AWS Cognito with multiple user pools
- **Monitoring**: CloudWatch for metrics, logging, and alerting

## Architecture Patterns

### Serverless Design

- Lambda functions are single-responsibility with 1:1 API endpoint mapping
- Stateless design for scalability and GLI compliance
- Event-driven architecture with critical file separation

### Critical Files Approach

- Critical components (RNG, ticket validation) are dependency-free for GLI certification
- Interfaces used for all external dependencies vs concrete implementations
- Plugin architecture considered for runtime functionality loading
- Configuration-driven behavior preferred over code-based control

### Data Architecture

- Just-in-time ticket creation (not pre-generated)
- Pre-allocated ticket ranges for different sales channels
- Immutable audit logging with comprehensive event tracking
- Separate repositories for infrastructure and services

## Module Naming Convention

Go modules follow the pattern: `github.com/riff-technologies/{module_name}`

## Database Strategy

- PostgreSQL for business logic and CRUD operations
- No stored procedures - all business logic in Lambda services
- Separate Go modules for critical vs non-critical data operations
- Currency values stored in lowest denomination (cents)
