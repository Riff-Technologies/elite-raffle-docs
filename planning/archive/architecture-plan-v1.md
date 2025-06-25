# GLI-31 Economic Raffle System Architecture

## Executive Summary

This architecture prioritizes GLI-31 compliance using the most cost-effective AWS services available. We adopt a serverless-first approach with S3 Object Lock for immutable storage and KMS Standard Keys for RNG (pending GLI verification), dramatically reducing costs while maintaining full compliance and auditability.

## Core Architecture Decisions

### 1. Data Storage Strategy (Ultra Cost-Optimized)

- **GLI Core Data**: Amazon S3 with Object Lock (Compliance Mode)
  - Immutable by design with write-once-read-many (WORM)
  - 95% cost savings vs Aurora or QLDB
  - Compliance mode prevents deletion even by root user
  - Athena for querying when needed
- **Business Data**: DynamoDB On-Demand
  - Pay-per-request pricing (zero baseline cost)
  - Handles user sessions, transactions, settings
  - Aurora Serverless v2 only if complex relational queries required

### 2. Compute Strategy (Pure Serverless)

- **All Components**: AWS Lambda only
  - No ECS/Fargate (eliminate container overhead)
  - 512MB memory default (optimize based on profiling)
  - Provisioned concurrency only for RNG service if needed

### 3. RNG Strategy (Pending GLI Verification)

- **Primary**: AWS KMS Standard Keys (FIPS 140-2 Level 2)
  - $1/month per key vs $400+/month for CloudHSM
  - Must verify GLI acceptance before implementation
- **Fallback**: External certified RNG service API
- **Alternative**: CloudHSM only if GLI explicitly requires Level 3

## Detailed Component Specifications

### GLI-31 Certified Core System

#### 1. Random Number Generator (RNG)

- **Service**: AWS Lambda + KMS Standard Keys
- **Configuration**:
  ```
  Runtime: Python 3.11
  Memory: 256 MB (reduced from 512 MB)
  Timeout: 10 seconds
  Reserved Concurrency: None (on-demand)
  ```
- **KMS Integration**:
  - KMS Standard Keys for FIPS 140-2 Level 2
  - GenerateRandom API for cryptographic entropy
  - **Critical**: Verify GLI acceptance before implementation
- **Cost Impact**: ~$5/month (vs $400/month for CloudHSM)

#### 2. Immutable Ledger (S3 Object Lock)

- **Purpose**: Store all GLI-compliance critical data
- **Configuration**:
  - Bucket with Object Lock in Compliance Mode
  - Retention period: 7 years (adjustable based on jurisdiction)
  - Default encryption with S3-managed keys
- **Data Structure**:
  ```
  /raffles/{raffle-id}/
    manifest.json
    tickets/{batch-id}.json
    draws/{draw-id}.json
    results/{draw-id}.json
  ```
- **Query Strategy**:
  - Lambda functions for write operations
  - Athena for complex queries (on-demand)
  - DynamoDB for hot data indexes
- **Cost Impact**: ~$5-20/month (vs $50-100 for QLDB)

#### 3. Draw & Ticket Management

- **Service**: AWS Lambda functions
- **Data Flow**:
  ```
  EventBridge → Lambda → KMS (RNG) → S3 Object Lock
                      ↓
                 DynamoDB (indexes)
  ```
- **Ticket Generation**:
  - Batch generation with sequential numbering
  - Store in S3 with Object Lock
  - Index in DynamoDB for fast lookups

#### 4. Audit System

- **Primary**: Direct to S3 with Object Lock
- **Lifecycle**:
  - S3 Standard: 0-90 days
  - S3 Glacier Instant: 90 days - 1 year
  - S3 Glacier Deep Archive: 1+ years
- **CloudWatch Logs**: 7-day retention (minimal)
- **Cost Impact**: ~$2-10/month

### Business Logic Layer

#### 1. User Management

- **Service**: Amazon Cognito User Pools
- **Configuration**:
  - Pay-per-MAU pricing
  - Social login optional (adds cost)
  - MFA optional

#### 2. Payment Processing

- **Service**: Lambda + Stripe/Payment Provider
- **Queue**: EventBridge (instead of SQS)
- **Storage**: DynamoDB for transaction records

#### 3. Business API

- **Service**: API Gateway HTTP API (cheaper than REST API)
- **Database**: DynamoDB (preferred) or Aurora Serverless v2
- **Caching**: DynamoDB with TTL

### Client Applications

#### 1. Admin Portal

- **Stack**: React 18 (simplified - no TypeScript requirement)
- **Hosting**: S3 + CloudFront
- **Auth**: Cognito SDK

#### 2. POS Mobile App (Minimal Viable Product)

- **Platform**: Progressive Web App (PWA)
  - Eliminates app store requirements
  - Works offline with service workers
  - Single codebase
- **Offline Strategy**:
  - IndexedDB for local storage
  - Pre-reserved ticket ranges
  - Background sync when online

#### 3. Public Website

- **Stack**: Static HTML/CSS/JS
- **Hosting**: S3 + CloudFront
- **Data**: Cached results from S3

### Security & Operations

#### 1. Infrastructure as Code

- **Tool**: Terraform (simpler than CDK for basic infrastructure)
- **Benefits**:
  - Wider community support
  - Simpler syntax
  - Multi-cloud ready

#### 2. Security Stack (Minimal)

- **WAF**: Basic managed rules only
- **Monitoring**: GuardDuty (basic tier)
- **Secrets**: Parameter Store (free tier)
- **Certificates**: ACM (free)

#### 3. Monitoring (Bare Minimum)

- **Metrics**: CloudWatch basic metrics only
- **Logs**: 7-day retention
- **Alerts**: Email only (no PagerDuty)
- **Tracing**: X-Ray with 1% sampling

## Cost Optimization Summary

### Monthly Cost Comparison

| Component          | Original Plan  | Economic Plan | Savings    |
| ------------------ | -------------- | ------------- | ---------- |
| GLI Database       | $300-500       | $5-20         | ~$450      |
| Business Database  | $300-500       | $20-50        | ~$400      |
| Compute (GLI Core) | $200-400       | $10-30        | ~$350      |
| RNG Service        | $600-800       | $5            | ~$750      |
| Audit Storage      | $50-100        | $2-10         | ~$80       |
| Monitoring         | $50-100        | $5-10         | ~$80       |
| **Total**          | **$1500-2400** | **$47-125**   | **~$2110** |

### Critical Cost Optimizations

1. **S3 Object Lock instead of QLDB** (95% cost reduction)
2. **KMS Standard instead of CloudHSM** (99% cost reduction)\*
3. **DynamoDB On-Demand** for business data (80% cost reduction)
4. **HTTP API instead of REST API** (70% cost reduction)
5. **Minimal monitoring and retention** (90% cost reduction)

\*Pending GLI verification of KMS acceptability

## Implementation Phases

### Phase 1: GLI Verification & Foundation (Weeks 1-2)

- **Critical**: Verify KMS Standard Keys acceptance with GLI
- AWS account setup
- Basic VPC (if required)
- IAM roles and policies
- Terraform project setup

### Phase 2: Core Storage & RNG (Weeks 3-6)

- S3 bucket with Object Lock configuration
- RNG Lambda with KMS integration
- Basic EventBridge setup
- Proof of concept for GLI review

### Phase 3: Core Raffle Logic (Weeks 7-10)

- Draw management Lambda
- Ticket generation Lambda
- Winner determination Lambda
- S3 storage patterns implementation

### Phase 4: APIs & Integration (Weeks 11-14)

- API Gateway HTTP API setup
- Lambda authorizers
- Basic WAF rules
- EventBridge integrations

### Phase 5: Business Logic (Weeks 15-18)

- Cognito setup (basic)
- Payment integration
- DynamoDB tables
- Business APIs

### Phase 6: Minimal Viable Product (Weeks 19-22)

- Admin portal (basic React app)
- PWA for POS (online-first)
- Static public website
- Basic monitoring

### Phase 7: Compliance & Launch (Weeks 23-26)

- GLI certification submission
- Security review
- Load testing (minimal)
- Documentation

## GLI-31 Compliance Strategy

### Pre-Implementation Requirements

1. **RNG Verification** (Week 1)

   - Submit KMS Standard Keys documentation to GLI
   - Get written confirmation of acceptability
   - Have CloudHSM fallback plan ready

2. **Object Lock Verification** (Week 1)

   - Confirm S3 Object Lock meets immutability requirements
   - Document retention policies
   - Prepare compliance mode justification

3. **Audit Trail Verification** (Week 2)
   - Document S3 + CloudWatch Logs approach
   - Confirm minimal retention meets requirements
   - Prepare lifecycle policy documentation

### Compliance Documentation

- **RNG Certification**: KMS FIPS 140-2 Level 2 documentation
- **Immutability Proof**: S3 Object Lock compliance mode specs
- **Audit Trail**: Complete data flow with timestamps
- **Security**: IAM policies and access controls
- **Testing**: Statistical randomness test results

## Risk Mitigation

### High-Priority Risks

1. **KMS Rejection by GLI**

   - **Mitigation**: Early verification in Week 1
   - **Fallback**: CloudHSM deployment plan ready
   - **Cost Impact**: +$400/month if required

2. **S3 Object Lock Limitations**

   - **Mitigation**: Prototype query patterns early
   - **Fallback**: DynamoDB for hot data, S3 for cold
   - **Cost Impact**: +$20-50/month

3. **Performance Concerns**
   - **Mitigation**: Implement caching layer early
   - **Fallback**: Increase Lambda memory/concurrency
   - **Cost Impact**: +$10-30/month

### Medium-Priority Risks

1. **Offline POS Requirements**

   - **Mitigation**: Start with online-only
   - **Solution**: PWA with service workers
   - **Timeline Impact**: +2 weeks if complex

2. **Complex Reporting Needs**
   - **Mitigation**: Use Athena for ad-hoc queries
   - **Solution**: Pre-aggregate in DynamoDB
   - **Cost Impact**: +$5-20/month

## Operational Considerations

### Monitoring Strategy (Minimal)

- **CloudWatch Alarms**:
  - Lambda errors > 1%
  - API Gateway 4xx/5xx > 5%
  - RNG service failures
- **Logs**: 7-day retention with critical events to S3
- **Dashboards**: Single operational dashboard

### Backup Strategy

- **S3 Object Lock**: Inherently immutable
- **DynamoDB**: Point-in-time recovery enabled
- **Lambda**: Versioning enabled
- **Infrastructure**: Terraform state in S3

### Disaster Recovery

- **RTO**: 4 hours (redeploy via Terraform)
- **RPO**: 0 for S3 data, 5 minutes for DynamoDB
- **Strategy**: Full redeploy from IaC
- **Testing**: Quarterly DR drills

## Conclusion

This economic architecture delivers:

- **95% cost reduction** vs enterprise solutions
- **Full GLI-31 compliance** (pending KMS verification)
- **Minimal operational overhead**
- **Quick time to market** (26 weeks)
- **Scalability** through serverless services

The architecture prioritizes cost savings while maintaining compliance. The critical path is GLI verification of KMS Standard Keys for RNG - this must be confirmed before proceeding with implementation.
