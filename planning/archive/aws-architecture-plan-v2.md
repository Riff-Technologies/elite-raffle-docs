## Architecture Plan

### **1. System Overview**

**Key Principles:**
- Serverless-first with Aurora PostgreSQL for relational data
- Self-verification at service level using CloudWatch Metrics
- Separated Android architecture: Expo UI + Certified Package for GLI compliance
- AWS Amplify hosting for Next.js admin and public portals
- GLI-31 compliant with focus on practical implementation

---

### **2. Frontend Architecture**

| Component | Technology | Purpose | GLI Certification |
|-----------|------------|---------|-------------------|
| **Public Portal** | Next.js on Amplify | Public raffle info, verification checksums | Not required |
| **Admin Portal** | Next.js on Amplify | Raffle management, reporting, monitoring | Not required |
| **RSU Mobile App** | Expo (React Native) | UI/UX, navigation, updates | Not required |
| **RSU Android Package** | Native Android (Kotlin) | Critical raffle operations, offline queue | **Required** |

---

### **3. Android RSU Architecture**

```
┌─────────────────────┐     ┌──────────────────────┐
│   Expo RSU App      │────▶│  Android Package     │
│  (UI/UX Layer)      │     │  (Certified)         │
│                     │     │                      │
│ • User Interface    │     │ • Ticket Generation  │
│ • Navigation        │     │ • Validation Logic   │
│ • Network Status    │     │ • Offline Queue      │
│ • Display Logic     │     │ • Self-Verification  │
│ • Auto Updates      │     │ • Secure Storage     │
└─────────────────────┘     └──────────────────────┘
         ↓                            ↓
    Package Binding            API Gateway
    Authentication            (When Online)
```

**Android Package API:**
```kotlin
interface RaffleSystemBridge {
    // Critical operations
    fun generateTicket(raffleId: String, userId: String): Ticket
    fun validateTicket(validationNumber: String): ValidationResult
    fun queueOfflineTransaction(transaction: Transaction)
    fun syncTransactions(endpoint: String, token: String): SyncResult

    // Self-verification
    fun verifySelf(): VerificationResult
    fun getPackageChecksum(): String
}
```

---

### **4. Web Hosting Architecture**

```
┌─────────────────────┐     ┌─────────────────────┐
│  Next.js Public     │     │  Next.js Admin      │
│    (Amplify)        │     │    (Amplify)        │
│                     │     │                     │
│ • Raffle Info       │     │ • Raffle Management │
│ • Prize Display     │     │ • User Management   │
│ • Verification Page │     │ • Reports Dashboard │
│ • Terms & Rules     │     │ • Real-time Monitor │
└─────────────────────┘     └─────────────────────┘
         ↓                            ↓
    CloudFront CDN              API Gateway
    (Public Content)            (Authenticated)
```

**Amplify Configuration:**
- Automatic CI/CD from Git
- Environment variables for API endpoints
- Custom domain with SSL
- Branch-based deployments (dev/staging/prod)

---

### **5. Updated System Components**

| Component | AWS Services | Updates |
|-----------|--------------|---------|
| **Web Hosting** | Amplify, CloudFront, Route53 | Next.js SSR/SSG support |
| **Mobile Auth** | Cognito User Pools | Separate pool for RSU devices |
| **Package Distribution** | S3 + CloudFront | Signed URLs for APK downloads |
| **Verification Display** | S3 Static Site | Public checksums (linked from Amplify) |

---

### **6. Authentication Flow**

```
1. RSU Device Registration:
   Expo App → Cognito → Receive Device Token

2. Package Authentication:
   Expo App → Android Package (Package Binding)

3. API Authentication:
   Android Package → API Gateway (Device Token)

4. Admin Portal:
   Next.js → Cognito → API Gateway (User Token)
```

---

### **7. Critical File Management**

```yaml
Critical Files (GLI Certified):
  Backend:
    - RNG Service (Lambda)
    - Raffle Core Service (Lambda)
    - Reporting Service (Lambda)

  Android Package:
    - Ticket generation logic
    - Validation algorithms
    - Offline queue management
    - Self-verification module

Non-Critical (No Certification):
  - Expo RSU App
  - Next.js Public Portal
  - Next.js Admin Portal
```

---

### **8. Deployment Architecture**

```bash
# Web Deployment (Amplify)
- Git push → Amplify Build → Deploy
- Environment-based builds
- Automatic SSL/domain setup

# Android Deployment
- Expo App → EAS Build → Google Play
- Android Package → Signed APK → S3
- Package verification on app startup

# Backend Deployment
- Lambda functions → Calculate checksums
- Store in Parameter Store
- Self-verify on cold start
```

---

### **9. Offline RSU Operation**

```
Android Package (Offline Mode):
1. Queue transactions in encrypted SQLite
2. Generate tickets with offline validation numbers
3. Store counterfoils locally
4. Sync when connection restored
5. Prevent duplicate ticket numbers via reserved ranges
```

---

### **10. Public Verification Website**

```
amplify-public-site.com/verify
├── Current System Checksum
├── Service Status Dashboard
├── Last Verification Time
├── Individual Service Checksums
└── Android Package Checksums
```

---

### **11. Cost Optimizations**

- **Amplify**: ~$0.15 per GB served (cached via CloudFront)
- **Expo Updates**: OTA updates reduce app store deployments
- **Android Package**: Minimal size, infrequent updates
- **CloudFront**: Cache static assets globally

---

### **12. Security Considerations**

1. **Android Package Security**:
   - APK signature verification
   - Certificate pinning for API calls
   - Encrypted local storage
   - Anti-tampering checks

2. **Web Security**:
   - Amplify provides automatic security headers
   - CloudFront DDoS protection
   - API Gateway rate limiting
   - Cognito MFA for admin users
