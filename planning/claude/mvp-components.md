# MVP Components and Features for GLI-31 submission and Certification

## Roles
- super admin (does everything)
- admin (GLI would typically get this role)
- regulator (if we want to generate a regulator-specific account for checksum verification)
- operator

## Administrator Website

### Raffle Management
- **Create/Configure Raffle**
  - Set sales start/end times and draw time
  - Configure maximum tickets
  - Define ticket allocation ranges (RSU pool/online pool/reserve pool) *not visible to GLI during certification*
  - Configure single 50/50 prize, or prize raffle
    - Configure other types of raffle events (multiple winners, different game types)
  - Prize configuration
  - Select venue and organization (org/venue/jurisdiction data will be added directly to the DB)
  - Set ticket package pricing

- **Raffle State Control**
  - Activate raffle (created/draft → active)
  - Suspend/resume with reason tracking (active -> suspended, suspended -> active)
  - Manual sales close option (active -> closed)
  - Cancel raffle with mandatory reason (active -> cancelled - if this is allowed)

- **Reconciliation Interface** (Critical for GLI)
  - RSU synchronization status dashboard (shows RSU, RSU sync status)
  - **Manual confirmation button for reconciliation complete** (logs user + timestamp)
  - Notes field for reconciliation *not required for certification, but nice-to-have*

- **Draw Management**
  - Initiate 50/50 draw
  - Display winning number

### RSU Management
- **Device Registry**
  - View all registered RSUs for the organization
  - Monitor device states (enrolled, ticket statistics, last sync datetime)
  - View which raffles RSUs are enrolled in
  - Real-time status indicators (what are the possible statuses? just enrolled or not?)
  - Last sync timestamps
  - Sales counters per RSU
  - Error state alerts

### Data Management
*Does not need UI for certification*
- Organization setup
- Venue configuration
- Jurisdiction settings

### Critical Event Reporting (no UI required)
- **Event Log Interface** *requires adhoc support for reporting during certification and for regulators*
  - Date/time range search
  - Event type filtering
  - Export functionality

## RSU Application

### Architecture Note
- Android native shell with JavaScript injection capability
- REST API integration (no SDKs) for payment processing, unless the SDK provides significant benefits
- OTA update mechanism for non-critical features
- APK gets the checksum for critical file

### Authentication
- **Operator Login**
  - Secure login screen with username & password/PIN
  - Operator ID display

### Raffle Operations
- **Raffle Enrollment** (Initiated from RSU)
  - List available raffles (by organization/venue)
  - Enroll button
  - Request ticket allocation / seed allocation
  - Display assigned range

- **Sales Interface**
  - Ticket package selection / price point selection
  - Quantity selector
  - Price calculation
  - Payment processing interface for cards (3rd party)  *we could use cash-only for certification, but adding a payment provider cannot change the checksum*
  - Sale completion
  - Ticket printing / emailing
  - customer contact info for contact if winner

- **Ticket Management**
  - Void ticket/order function
  - Reprint with "REPRINT" marking
  - Transaction history *not required, but nice-to-have*

### Status Displays
- **Device Information**
  - app version
  - Current state indicator (offline / online) *not required*
  - Offline ticket counter *not required*
  - Buffer capacity warning *not required*
  - Sync status *not required*
  - **Checksum display in-app**

### Sales Closing
- **Countdown Feature** (countdown could come after initial phase)
  - *requirement: display a "sales closed" status*
  - Visual countdown timer for sales closed *optional*
  - Upload progress bar
  - Upload confirmation display

### Error Handling
- **Error Displays**
  - Printer status (paper/connection) *disconnect status needs to be displayed*
  - Network failure alerts *disconnect status needs to be displayed*
  - Buffer warnings *not required for certification*
  - Critical error states *could be tracked on admin website, not on RSU*

## Public Website (Required for Certification)

### Public Display Features
- **Raffle Information**
  - Current raffle state
  - Sales status (jackpot)
  - Draw time countdown/timestamp *required to show when sales close*
  - Winner display (post-draw)

### System Verification Page for regulators
- **Compliance Display**
  - System checksums
  - Last verification timestamp
  - Verification status (valid/invalid)
  - Critical file checksums

### Kiosk Mode *not required for certification, but want to have this*
- Simplified view for in-venue displays
- Auto-refresh capability
- Large, readable formatting

## Key Simplifications for MVP/Certification

1. **50/50 Prize, and multi-prize game types** - Eliminates complex prize management UI *need to have all game types before GLI submission*
2. **No Customer Data on RSU** - Faster transactions, simpler UI
3. **RSU-Initiated Enrollment** - Reduces admin complexity *could enable this in the admin website*
4. **REST APIs Only** - Avoids SDK certification issues
5. **Reporting in admin website** - defined in GLI-31 section 2.8

## Critical GLI UI Elements

**Must-Have Features:**
- Manual reconciliation confirmation (with logging)
- Checksum displays (public website)
- State transition controls with reasons
- Error state visibility
- Countdown timer for coordinated closing

**Architecture Decisions Supporting Compliance:**
- JavaScript injection allows updates without affecting certified Android shell
- REST API approach avoids SDK dependencies
- Public website handles verification display requirements
