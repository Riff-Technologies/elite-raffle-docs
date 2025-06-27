# Electronic Raffle System - State Machines and Transitions

## Overview

This document defines all state machines, their states, and valid transitions for the Electronic Raffle System. Each state machine indicates which system component manages the state and what triggers transitions.

---

## 1. Raffle Event States

**Managed By**: Raffle Core Service (Lambda)  
**Stored In**: Aurora PostgreSQL - `raffles` table  
**State Field**: `state`

### States

| State | Description | Can Transition To |
|-------|-------------|-------------------|
| `created` | Raffle created but not fully configured | `configured`, `cancelled` |
| `configured` | All required parameters set (prizes, times, limits) | `active`, `cancelled` |
| `active` | Ticket sales are open | `sales_closed`, `suspended`, `cancelled` |
| `suspended` | Temporarily halted (can resume) | `active`, `cancelled` |
| `sales_closed` | No more ticket sales allowed | `reconciling`, `cancelled` |
| `reconciling` | Awaiting RSU synchronization and admin confirmation | `reconciled`, `cancelled` |
| `reconciled` | Admin confirmed all sales accounted for | `drawing_ready` |
| `drawing_ready` | Ready for draw to begin | `completed` |
| `completed` | Draw completed, winners determined | Terminal state |
| `cancelled` | Raffle cancelled (terminal state) | Terminal state |

### Transition Rules

```
created → configured
  Trigger: Admin completes all required configuration
  Validation: All required fields populated
  Critical Event: No

configured → active  
  Trigger: Admin activates raffle
  Validation: Current time < sales end time
  Critical Event: Yes - "RAFFLE_ACTIVATED"

active → sales_closed
  Trigger: Scheduled time reached OR admin manual close
  Validation: None
  Critical Event: Yes - "SALES_CLOSED"

active → suspended
  Trigger: Admin action with reason
  Validation: None
  Critical Event: Yes - "RAFFLE_SUSPENDED"

suspended → active
  Trigger: Admin action
  Validation: Current time < sales end time
  Critical Event: Yes - "RAFFLE_RESUMED"

sales_closed → reconciling
  Trigger: System automatic OR admin initiated
  Validation: All RSUs reported closed
  Critical Event: Yes - "RECONCILIATION_STARTED"

reconciling → reconciled
  Trigger: Admin confirmation (manual)
  Validation: All voids processed, all RSUs reconciled
  Critical Event: Yes - "RECONCILIATION_CONFIRMED" (GLI Required)
  Additional: Must log admin user ID and timestamp

reconciled → drawing_ready
  Trigger: Automatic after reconciliation
  Validation: None
  Critical Event: No

drawing_ready → completed
  Trigger: Draw process completed
  Validation: Winners determined
  Critical Event: Yes - "DRAW_COMPLETED"

ANY → cancelled
  Trigger: Admin action with reason
  Validation: No draw has occurred
  Critical Event: Yes - "RAFFLE_CANCELLED"
```

---

## 2. Ticket States

**Managed By**: Raffle Core Service (Lambda)  
**Stored In**: Aurora PostgreSQL - `tickets` table  
**State Field**: `state`

### States

| State | Description | Can Transition To |
|-------|-------------|-------------------|
| `available` | Created in pool, not allocated to any RSU | `allocated` |
| `allocated` | Assigned to specific RSU for sale | `sold`, `voided` |
| `sold` | Payment completed, ticket issued to customer | `voided`, `winner` |
| `voided` | Cancelled/refunded before draw | Terminal state |
| `winner` | Matched winning draw number | `claimed` |
| `claimed` | Prize claimed by winner | Terminal state |

### Transition Rules

```
available → allocated
  Trigger: RSU requests allocation
  Validation: RSU is active and enrolled
  Critical Event: No

allocated → sold
  Trigger: RSU records sale OR online purchase
  Validation: Raffle in 'active' state
  Critical Event: No

sold → voided
  Trigger: Customer refund OR admin action
  Validation: Raffle not in 'reconciled' or later state
  Critical Event: Yes - "TICKET_VOIDED"

sold → winner
  Trigger: Draw process matches ticket
  Validation: Automatic during draw
  Critical Event: No (logged as part of draw)

winner → claimed
  Trigger: Winner claims prize
  Validation: Valid ticket presented
  Critical Event: Yes - "PRIZE_CLAIMED"
```

---

## 3. RSU (Raffle Sales Unit) States

**Managed By**: Raffle Core Service (Lambda)  
**Stored In**: Aurora PostgreSQL - `rsus` table  
**State Field**: `state`

### States

| State | Description | Can Transition To |
|-------|-------------|-------------------|
| `registered` | Device registered in system | `enrolled`, `suspended` |
| `enrolled` | Enrolled in specific raffle | `active`, `suspended` |
| `active` | Can sell tickets | `closing`, `suspended`, `error` |
| `closing` | Uploading final data | `closed`, `error` |
| `closed` | Finished for current raffle | `reconciled` |
| `reconciled` | All sales accounted for | `registered` |
| `suspended` | Temporarily disabled | `registered`, `enrolled`, `active` |
| `error` | Critical error detected | `suspended` |

### Transition Rules

```
registered → enrolled
  Trigger: Admin enrolls RSU in raffle
  Validation: Raffle exists and not completed
  Critical Event: Yes - "RSU_ENROLLED"

enrolled → active
  Trigger: Operator authenticates on device
  Validation: Valid credentials, raffle is active
  Critical Event: Yes - "RSU_ACTIVATED"

active → closing
  Trigger: Raffle state changes to 'sales_closed'
  Validation: Automatic
  Critical Event: Yes - "RSU_CLOSING"

closing → closed
  Trigger: All offline sales uploaded
  Validation: Sync completed successfully
  Critical Event: Yes - "RSU_CLOSED"

closed → reconciled
  Trigger: Admin confirms RSU reconciliation
  Validation: All sales verified
  Critical Event: No

reconciled → registered
  Trigger: Automatic after raffle completion
  Validation: None
  Critical Event: No

ANY → suspended
  Trigger: Admin action OR security violation
  Validation: None
  Critical Event: Yes - "RSU_SUSPENDED"

active → error
  Trigger: Buffer overflow OR critical memory error
  Validation: Automatic
  Critical Event: Yes - "RSU_ERROR"
```

---

## 4. Draw States

**Managed By**: Raffle Core Service (Lambda)  
**Stored In**: Aurora PostgreSQL - `draws` table  
**State Field**: `state`

### States

| State | Description | Can Transition To |
|-------|-------------|-------------------|
| `pending` | Waiting for raffle to be ready | `generating` |
| `generating` | RNG/manual process in progress | `completed` |
| `completed` | Winners determined and verified | Terminal state |

### Transition Rules

```
pending → generating
  Trigger: Admin initiates draw
  Validation: Raffle in 'drawing_ready' state
  Critical Event: Yes - "DRAW_INITIATED"

generating → completed
  Trigger: All winning numbers generated and verified
  Validation: Numbers match sold tickets
  Critical Event: Yes - "DRAW_COMPLETED"
```

---

## 5. Order States

**Managed By**: Payment Service → Raffle Core Service  
**Stored In**: Aurora PostgreSQL - `orders` table  
**State Field**: `payment_status`

### States

| State | Description | Can Transition To |
|-------|-------------|-------------------|
| `pending` | Payment initiated | `completed`, `failed` |
| `completed` | Payment successful | `refunded` |
| `failed` | Payment failed | Terminal state |
| `refunded` | Payment refunded | Terminal state |

### Transition Rules

```
pending → completed
  Trigger: Payment webhook (success)
  Validation: Valid payment reference
  Critical Event: No

pending → failed
  Trigger: Payment webhook (failure) OR timeout
  Validation: None
  Critical Event: No

completed → refunded
  Trigger: Refund webhook OR admin action
  Validation: Tickets can still be voided
  Critical Event: Yes - "ORDER_REFUNDED"
```

---

## Implementation Guidelines

### State Transition Function Pattern (Go)

```go
// Example for Raffle state transitions
func (r *Raffle) TransitionTo(newState RaffleState, userId uuid.UUID, reason string) error {
    // Validate transition
    if !r.CanTransitionTo(newState) {
        return ErrInvalidStateTransition
    }
    
    // Record previous state
    previousState := r.State
    
    // Update state
    r.State = newState
    r.StateUpdatedAt = time.Now()
    r.StateUpdatedBy = userId
    
    // Log critical event if required
    if requiresCriticalEvent(previousState, newState) {
        logCriticalEvent(CriticalEvent{
            EventType: getEventType(previousState, newState),
            RaffleID: &r.ID,
            UserID: &userId,
            Description: fmt.Sprintf("State transition: %s → %s. Reason: %s", 
                previousState, newState, reason),
        })
    }
    
    return nil
}
```

### Critical Event Types (GLI-31 Compliance)

Required critical events per GLI-31 Section 5.6.1:
- `RSU_CONNECTED` / `RSU_DISCONNECTED`
- `CRITICAL_MEMORY_ERROR`
- `PRINTER_ERROR` (out of paper, disconnect, memory error)
- `COMMUNICATION_FAILURE`
- `BUFFER_FULL`
- `AUTHENTICATION_MISMATCH`
- `FIREWALL_LOG_FULL`
- `REMOTE_ACCESS`

Additional events for state transitions:
- `RAFFLE_*` (ACTIVATED, SUSPENDED, CANCELLED, etc.)
- `RECONCILIATION_CONFIRMED`
- `TICKET_VOIDED`
- `DRAW_*` (INITIATED, COMPLETED)
- `RSU_*` (ENROLLED, SUSPENDED, ERROR)
- `PRIZE_CLAIMED`

### Database Triggers

Consider implementing database triggers to:
1. Automatically create state history records
2. Enforce state transition rules at DB level
3. Prevent invalid state changes

### Monitoring and Alerts

Set up CloudWatch alarms for:
- RSUs in `error` state > 5 minutes
- Raffles stuck in `reconciling` > 2 hours
- Failed state transitions
- Critical events of type `ERROR` or `CRITICAL`

---

## State Diagram Visual Summary

```
Raffle: created → configured → active → sales_closed → reconciling → reconciled → drawing_ready → completed
                      ↓           ↓            ↓              ↓                           ↓
                  cancelled   suspended    cancelled      cancelled                  cancelled

Ticket: available → allocated → sold → winner → claimed
                       ↓         ↓
                    voided    voided

RSU: registered → enrolled → active → closing → closed → reconciled → registered
          ↓           ↓         ↓         ↓         ↓          ↓
      suspended   suspended suspended  error    suspended  suspended
```

This document serves as the authoritative guide for all state management in the Electronic Raffle System.