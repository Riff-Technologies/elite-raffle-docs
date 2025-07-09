# MVP Components and Features for GLI-31 submission and Certification

## Administrator Website

### Raffle Management
- **Create/Configure Raffle**
  - Set sales start/end times and draw time
  - Configure maximum tickets
  - Define ticket allocation ranges (RSU pool/online pool/reserve pool)
  - Configure single 50/50 prize
  - Select venue and organization (org/venue/jurisdiction data will be added directly to the DB)
  - Set ticket package pricing

- **Raffle State Control**
  - Activate raffle (configured → active)
  - Suspend/resume with reason tracking
  - Manual sales close option
  - Cancel raffle with mandatory reason

- **Reconciliation Interface** (Critical for GLI)
  - RSU synchronization status dashboard
  - Discrepancy identification
  - **Manual confirmation button** (logs user + timestamp)
  - Notes field for reconciliation

- **Draw Management**
  - Initiate 50/50 draw
  - Display winning number
  - Show verification checksum

### RSU Management
- **Device Registry**
  - View all registered RSUs
  - Monitor device states
  - Suspend/reactivate devices
  - View which raffles RSUs are enrolled in

- **RSU Monitoring Dashboard**
  - Real-time status indicators
  - Last sync timestamps
  - Sales counters per RSU
  - Error state alerts

### Data Management (May not need UI for certification)
- Organization setup
- Venue configuration
- Jurisdiction settings

### Critical Event Viewing
- **Event Log Interface** *(Note: Clarifying with GLI if needed in UI or can be provided with adhoc reports)*
  - Date/time range search
  - Event type filtering
  - Export functionality

## RSU Application

### Architecture Note
- Android native shell with JavaScript injection capability
- REST API integration (no SDKs) for payment processing
- OTA update mechanism for non-critical features

### Authentication
- **Operator Login**
  - Secure login screen
  - Quick PIN setup
  - Operator ID display

### Raffle Operations
- **Raffle Enrollment** (Initiated from RSU)
  - List available raffles (by organization/venue)
  - Enroll button
  - Request ticket allocation
  - Display assigned range

- **Sales Interface**
  - Package selection
  - Quantity selector
  - Price calculation
  - Sale completion

- **Ticket Management**
  - Void ticket function
  - Reprint with "REPRINT" marking
  - Transaction history

### Status Displays
- **Device Information**
  - Current state indicator
  - Offline ticket counter
  - Buffer capacity warning
  - Sync status
  - **Checksum display** (for verification)

### Sales Closing
- **Countdown Feature**
  - Visual countdown timer
  - Auto-disable sales at zero
  - Upload progress bar
  - Upload confirmation display

### Error Handling
- **Error Displays**
  - Printer status (paper/connection)
  - Network failure alerts
  - Buffer warnings
  - Critical error states

## Public Website (Required for Certification)

### Public Display Features
- **Raffle Information**
  - Current raffle state
  - Sales status
  - Draw time countdown
  - Winner display (post-draw)

### System Verification Page
- **Compliance Display**
  - System checksums
  - Last verification timestamp
  - Verification status (valid/invalid)
  - Critical file checksums

### Kiosk Mode
- Simplified view for in-venue displays
- Auto-refresh capability
- Large, readable formatting

## Key Simplifications for MVP/Certification

1. **Single 50/50 Prize** - Eliminates complex prize management UI
2. **No Customer Data on RSU** - Faster transactions, simpler UI
3. **RSU-Initiated Enrollment** - Reduces admin complexity
4. **REST APIs Only** - Avoids SDK certification issues

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
