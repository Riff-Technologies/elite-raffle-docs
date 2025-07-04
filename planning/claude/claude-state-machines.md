# Electronic Raffle System - State Machines and Transitions

## Overview

This document defines all state machines, their states, and valid transitions for the Electronic Raffle System. Each state machine indicates which system component manages the state and what triggers transitions. All state transition logic is implemented in the service layer, not through database triggers.

## Prerequisites

- All user operations require a valid `user_reference` record that maps to the authenticated Cognito user
- The service layer automatically creates/updates `user_reference` records when users first access the system

-----

## 1. Raffle Event States

**Managed By**: Raffle Core Service (Lambda)  
**Stored In**: Aurora PostgreSQL - `raffles` table  
**State Field**: `state`

### States

|State          |Description                                        |Can Transition To                       |
|---------------|---------------------------------------------------|----------------------------------------|
|`created`      |Raffle created but not fully configured            |`configured`, `cancelled`               |
|`configured`   |All required parameters set (prizes, times, limits)|`active`, `cancelled`                   |
|`active`       |Ticket sales are open                              |`sales_closed`, `suspended`, `cancelled`|
|`suspended`    |Temporarily halted (can resume)                    |`active`, `cancelled`                   |
|`sales_closed` |No more ticket sales allowed                       |`reconciling`, `cancelled`              |
|`reconciling`  |Awaiting RSU synchronization and admin confirmation|`reconciled`, `cancelled`               |
|`reconciled`   |Admin confirmed all sales accounted for            |`drawing_ready`                         |
|`drawing_ready`|Ready for draw to begin                            |`completed`                             |
|`completed`    |Draw completed, winners determined                 |Terminal state                          |
|`cancelled`    |Raffle cancelled (terminal state)                  |Terminal state                          |

### Transition Rules

```
created → configured
  Trigger: Admin completes all required configuration
  Validation: 
    - All required fields populated
    - Organization and venue are active
    - Valid ticket packages assigned
  Additional Actions:
    - Generate all tickets (state: 'available')
  Critical Event: No

configured → active  
  Trigger: Admin activates raffle OR scheduled time reached
  Validation: 
    - Current time < sales end time
    - Organization and venue still active
  Critical Event: Yes - "RAFFLE_ACTIVATED"

active → sales_closed
  Trigger: Scheduled time reached OR admin manual close
  Validation: None
  Additional Actions:
    - Send close notification to all RSUs
    - Start countdown timer on RSUs
  Critical Event: Yes - "SALES_CLOSED"

active → suspended
  Trigger: Admin action with reason
  Validation: None
  Required Data:
    - suspension_reason (text)
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
  Validation: 
    - All voids processed
    - All RSUs reconciled
    - All offline sales synced
  Required Data:
    - reconciliation_confirmed_by (user ID)
    - reconciliation_confirmed_at (timestamp)
  Critical Event: Yes - "RECONCILIATION_CONFIRMED" (GLI Required)

reconciled → drawing_ready
  Trigger: Automatic after reconciliation
  Validation: None
  Critical Event: No

drawing_ready → completed
  Trigger: Draw process completed
  Validation: Winners determined for all prizes
  Additional Actions:
    - Create draw_result records
    - Update winning tickets to 'winner' state
    - Calculate actual values for percentage prizes
  Critical Event: Yes - "DRAW_COMPLETED"

ANY → cancelled
  Trigger: Admin action with reason
  Validation: No draw has occurred
  Required Data:
    - cancellation_reason (text)
  Critical Event: Yes - "RAFFLE_CANCELLED"
```

-----

## 2. Ticket States

**Managed By**: Raffle Core Service (Lambda)  
**Stored In**: Aurora PostgreSQL - `tickets` table  
**State Field**: `state`

### States

|State      |Description                                 |Can Transition To  |
|-----------|--------------------------------------------|-------------------|
|`available`|Created in pool, not allocated to any RSU   |`allocated`, `sold`|
|`allocated`|Assigned to specific RSU for sale           |`sold`, `voided`   |
|`sold`     |Payment completed, ticket issued to customer|`voided`, `winner` |
|`voided`   |Cancelled/refunded before draw              |Terminal state     |
|`winner`   |Matched winning draw number                 |`claimed`          |
|`claimed`  |Prize claimed by winner                     |Terminal state     |

### Transition Rules

```
available → allocated
  Trigger: RSU requests allocation
  Validation: 
    - RSU is active and enrolled
    - Ticket within allocation range
  Additional Actions:
    - Update allocated_to_rsu_id
    - Update allocation_id
    - Set allocated_at timestamp
  Critical Event: No

available → sold
  Trigger: Online purchase completed
  Validation: 
    - Raffle in 'active' state
    - Order payment completed
  Required Data:
    - order_id
    - sold_at (timestamp)
    - sold_by (user ID if admin sale)
  Critical Event: No

allocated → sold
  Trigger: RSU records sale
  Validation: 
    - Raffle in 'active' state
    - RSU in 'active' state
    - Ticket within RSU's allocation
  Required Data:
    - order_id
    - sold_at (timestamp)
    - sold_by (RSU operator ID)
  Additional Actions:
    - Increment rsu_allocation.tickets_sold counter
  Critical Event: No

sold → voided
  Trigger: Customer refund OR admin action
  Validation: 
    - Raffle not in 'reconciled' or later state
  Required Data:
    - voided_at (timestamp)
    - voided_by (user ID)
    - voided_reason (text)
  Additional Actions:
    - If RSU sale: Increment rsu_allocation.tickets_voided counter
    - Check if all tickets in order are voided
    - If yes, trigger order refund process
  Critical Event: Yes - "TICKET_VOIDED"

allocated → voided
  Trigger: Admin voids allocated ticket
  Validation: 
    - Ticket not sold
  Required Data:
    - voided_at (timestamp)
    - voided_by (user ID)
    - voided_reason (text)
  Additional Actions:
    - Increment rsu_allocation.tickets_voided counter
  Critical Event: Yes - "TICKET_VOIDED"

sold → winner
  Trigger: Draw process matches ticket
  Validation: Automatic during draw
  Additional Actions:
    - Create draw_result record
  Critical Event: No (logged as part of draw)

winner → claimed
  Trigger: Winner claims prize
  Validation: 
    - Valid ticket presented
    - Validation number matches
  Required Data:
    - claimed_at (timestamp)
    - claimed_by (user ID)
  Additional Actions:
    - Update draw_result.claim_status
    - Update draw_result.claimed_at
    - Update draw_result.claimed_by
  Critical Event: Yes - "PRIZE_CLAIMED"
```

-----

## 3. RSU (Raffle Sales Unit) States

**Managed By**: Raffle Core Service (Lambda)  
**Stored In**: Aurora PostgreSQL - `rsus` table  
**State Field**: `state`

### States

|State       |Description                |Can Transition To                 |
|------------|---------------------------|----------------------------------|
|`registered`|Device registered in system|`enrolled`, `suspended`           |
|`enrolled`  |Enrolled in specific raffle|`active`, `suspended`             |
|`active`    |Can sell tickets           |`closing`, `suspended`, `error`   |
|`closing`   |Uploading final data       |`closed`, `error`                 |
|`closed`    |Finished for current raffle|`reconciled`                      |
|`reconciled`|All sales accounted for    |`registered`                      |
|`suspended` |Temporarily disabled       |`registered`, `enrolled`, `active`|
|`error`     |Critical error detected    |`suspended`                       |

### Transition Rules

```
registered → enrolled
  Trigger: Admin enrolls RSU in raffle
  Validation: 
    - Raffle exists and not completed
    - Organization owns RSU
  Additional Actions:
    - Create rsu_allocation record
    - Allocate ticket number range
    - Update tickets to 'allocated' state
  Critical Event: Yes - "RSU_ENROLLED"

enrolled → active
  Trigger: Operator authenticates on device
  Validation: 
    - Valid credentials
    - Raffle is active
    - RSU has allocation
  Critical Event: Yes - "RSU_ACTIVATED"

active → closing
  Trigger: Raffle state changes to 'sales_closed'
  Validation: Automatic
  Additional Actions:
    - Start sync of any offline sales
    - Display countdown on device
  Critical Event: Yes - "RSU_CLOSING"

closing → closed
  Trigger: All offline sales uploaded
  Validation: 
    - Sync completed successfully
    - All tickets accounted for
  Additional Actions:
    - Final allocation count verification
  Critical Event: Yes - "RSU_CLOSED"

closed → reconciled
  Trigger: Admin confirms RSU reconciliation
  Validation: 
    - All sales verified
    - Allocation counts match
  Critical Event: No

reconciled → registered
  Trigger: Automatic after raffle completion
  Validation: None
  Additional Actions:
    - Clear allocation data
    - Reset device for next raffle
  Critical Event: No

ANY → suspended
  Trigger: Admin action OR security violation
  Validation: None
  Additional Actions:
    - Log suspension reason
    - Prevent all sales
  Critical Event: Yes - "RSU_SUSPENDED"

active → error
  Trigger: Buffer overflow OR critical memory error
  Validation: Automatic
  Additional Actions:
    - Halt all operations
    - Alert administrators
  Critical Event: Yes - "RSU_ERROR"
```

-----

## 4. Draw States

**Managed By**: Raffle Core Service (Lambda)  
**Stored In**: Aurora PostgreSQL - `draws` table  
**State Field**: `state`

### States

|State       |Description                    |Can Transition To|
|------------|-------------------------------|-----------------|
|`pending`   |Waiting for raffle to be ready |`generating`     |
|`generating`|RNG/manual process in progress |`completed`      |
|`completed` |Winners determined and verified|Terminal state   |

### Transition Rules

```
pending → generating
  Trigger: Admin initiates draw
  Validation: 
    - Raffle in 'drawing_ready' state
    - All prizes configured
  Required Data:
    - conducted_by (user ID)
  Additional Actions:
    - Initialize RNG with seed
    - Lock raffle from modifications
  Critical Event: Yes - "DRAW_INITIATED"

generating → completed
  Trigger: All winning numbers generated and verified
  Validation: 
    - Numbers match sold tickets
    - One winner per prize position
  Required Data:
    - draw_time (timestamp)
    - rng_seed (for audit)
    - verification_checksum
  Additional Actions:
    - Create draw_result record for each winner
    - Update winning tickets to 'winner' state
    - Calculate actual prize values for percentage prizes
    - Update raffle to 'completed' state
  Critical Event: Yes - "DRAW_COMPLETED"
```

-----

## 5. Order States

**Managed By**: Payment Service → Raffle Core Service  
**Stored In**: Aurora PostgreSQL - `orders` table  
**State Field**: `payment_status`

### States

|State      |Description       |Can Transition To    |
|-----------|------------------|---------------------|
|`pending`  |Payment initiated |`completed`, `failed`|
|`completed`|Payment successful|`refunded`           |
|`failed`   |Payment failed    |Terminal state       |
|`refunded` |Payment refunded  |Terminal state       |

### Transition Rules

```
pending → completed
  Trigger: Payment webhook (success)
  Validation: Valid payment reference
  Additional Actions:
    - Generate unique validation number
    - Update all tickets in order to 'sold' state
    - Send confirmation to customer
  Critical Event: No

pending → failed
  Trigger: Payment webhook (failure) OR timeout
  Validation: None
  Additional Actions:
    - Release any reserved tickets
    - Log failure reason
  Critical Event: No

completed → refunded
  Trigger: Refund webhook OR admin action
  Validation: 
    - Tickets can still be voided
    - Raffle not in 'reconciled' or later state
  Additional Actions:
    - Update all tickets in order to 'voided' state
    - Process payment provider refund
  Critical Event: Yes - "ORDER_REFUNDED"
```

-----

## Required Data Capture

### Raffle State Transitions

- `ANY → cancelled`: cancellation_reason (required)
- `active → suspended`: suspension_reason (required)
- `reconciling → reconciled`: reconciliation_confirmed_by, reconciliation_confirmed_at

### Ticket State Transitions

- `ANY → sold`: order_id, sold_at, sold_by (optional for online)
- `sold → voided`: voided_reason, voided_by, voided_at
- `winner → claimed`: claimed_at, claimed_by (updates draw_result)

### Order State Transitions

- `pending → completed`: validation_number (generated)
- `completed → refunded`: refund_reason

### Draw State Transitions

- `pending → generating`: conducted_by
- `generating → completed`: draw_time, rng_seed, verification_checksum

-----

## State Transition Side Effects

### Order Completion

- Generates unique validation number
- Updates all associated tickets to ‘sold’ state
- Triggers customer notification
- Updates RSU allocation counters if RSU sale

### Ticket Void

- Updates rsu_allocation counters if RSU sale
- May trigger order refund if all tickets voided
- Logs critical event

### Draw Completion

- Creates draw_result records for all winners
- Updates winning tickets to ‘winner’ state
- Calculates actual values for percentage prizes
- Transitions raffle to ‘completed’ state

### RSU Allocation

- Creates allocation record with number range
- Updates tickets in range to ‘allocated’ state
- Sets allocation counters to zero

-----

## Implementation Guidelines

### State Transition Function Pattern (Go)

```go
// Example for Raffle state transitions
func (r *Raffle) TransitionTo(newState RaffleState, userId uuid.UUID, reason string) error {
    // Validate transition
    if !r.CanTransitionTo(newState) {
        return ErrInvalidStateTransition
    }
    
    // Check additional validations
    if err := r.ValidateTransition(newState); err != nil {
        return err
    }
    
    // Record previous state
    previousState := r.State
    
    // Update state
    r.State = newState
    r.StateUpdatedAt = time.Now()
    r.StateUpdatedBy = userId
    
    // Capture required data
    switch newState {
    case RaffleStateCancelled:
        if reason == "" {
            return ErrCancellationReasonRequired
        }
        r.CancellationReason = &reason
    case RaffleStateReconciled:
        r.ReconciliationConfirmedAt = &time.Now()
        r.ReconciliationConfirmedBy = &userId
    }
    
    // Execute side effects
    if err := r.ExecuteSideEffects(previousState, newState); err != nil {
        return err
    }
    
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
- `ORDER_REFUNDED`

### Service Layer Responsibilities

The service layer handles:

1. State transition validation
1. Required data capture
1. Side effect execution
1. Counter updates (e.g., rsu_allocation)
1. Critical event logging
1. Cross-entity state coordination

### Monitoring and Alerts

Set up CloudWatch alarms for:

- RSUs in `error` state > 5 minutes
- Raffles stuck in `reconciling` > 2 hours
- Failed state transitions
- Critical events of type `ERROR` or `CRITICAL`
- Orders in `pending` state > 30 minutes

-----

## State Diagram Visual Summary

```
Raffle: created → configured → active → sales_closed → reconciling → reconciled → drawing_ready → completed
                      ↓           ↓            ↓              ↓                           ↓
                  cancelled   suspended    cancelled      cancelled                  cancelled

Ticket: available → allocated → sold → winner → claimed
            ↓          ↓         ↓
           sold     voided    voided

RSU: registered → enrolled → active → closing → closed → reconciled → registered
          ↓           ↓         ↓         ↓         ↓          ↓
      suspended   suspended suspended  error    suspended  suspended

Order: pending → completed → refunded
          ↓
       failed

Draw: pending → generating → completed
```

This document serves as the authoritative guide for all state management in the Electronic Raffle System.