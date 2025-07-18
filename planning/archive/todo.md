## **Top 5 Priorities for GLI Meeting**

### **1. Create Architecture Diagram (Visual)**
GLI will want to see a clear system diagram. Create one showing:
- Three Lambda services (RNG, Raffle Core, Reporting)
- Android Package separation from Expo app
- Self-verification flow with CloudWatch Metrics
- Critical vs non-critical components (color code them)

**Tool suggestion**: draw.io or Lucidchart for quick professional diagrams

---

### **2. Document Critical File Identification**
Prepare a clear table showing:
```
Component         | Critical Files                    | Verification Method
-----------------|-----------------------------------|--------------------
RNG Service      | /src/rng/generator.go            | SHA-256 checksum
Raffle Core      | /src/raffle/tickets.go           | SHA-256 checksum
                 | /src/raffle/validation.go        |
Android Package  | RaffleSystemBridge.kt            | APK signature
                 | OfflineQueue.kt                  |
```

---

### **3. Prepare Self-Verification Demo/Pseudocode**
Show exactly how self-verification works:
```go
// Lambda cold start
func init() {
    expected := os.Getenv("DEPLOY_CHECKSUM")
    actual := calculateSelfChecksum()

    if expected != actual {
        cloudwatch.PutMetric("ChecksumValid", 0)
        panic("Checksum mismatch")
    }
    cloudwatch.PutMetric("ChecksumValid", 1)
}
```

---

### **4. Address GLI's Key Concerns**
Have clear answers for:
- **Q: "How do you prevent unauthorized code changes?"**
  A: Self-verification on every cold start + CloudWatch alarms

- **Q: "How do you handle offline RSUs?"**
  A: Reserved ticket ranges, encrypted queue, Android package handles all critical ops

- **Q: "What's your RNG approach?"**
  A: Go's crypto/rand, software-based, ~15 lines as you suggested

---

### **5. Prepare Questions for GLI**
Show you're thinking ahead:
1. "Is our Android package separation approach acceptable for certification?"
2. "Do you see any issues with using CloudWatch Metrics for self-verification?"
3. "Should we implement any specific GLI test harnesses now?"
4. "Any concerns with Aurora Serverless v2 for critical memory?"
5. "What documentation should we prepare in parallel with development?"


### **6. PCI Compliance**
PCI Compliance Steps:
- Complete SAQ A self-assessment questionnaire annually
- Use Stripe's PCI compliant libraries
- Never log or store card details
- Implement secure coding practices

**Bonus**: Print the GLI-31 standard sections you're addressing (2.3, 5.6, 5.8) to show you've mapped requirements to architecture.
