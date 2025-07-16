# Elite Raffle System - Product Overview

Elite Raffle is a comprehensive electronic raffle system designed for GLI-31 certification compliance. The system provides secure ticket sales, transparent drawing processes, and complete audit trails for modern raffle operations.

## Core Features

- **Multi-channel Sales**: Online web platform and RSU (Raffle Sales Unit) devices
- **Real-time Jackpot Tracking**: Dynamic jackpot calculations with configurable revenue methods
- **Comprehensive Audit Trail**: Full GLI-31 compliant logging and reporting
- **Offline Capability**: RSU devices can operate offline to accept cash transactions with automatic synchronization
- **Prize Management**: Support for both fixed-value and percentage-based prizes
- **Organization Multi-tenancy**: Support for multiple organizations and venues

## Key Business Rules

- Tickets are created "just-in-time" at sale time (not pre-generated)
- Ticket ranges are allocated to different sales channels (RSU, online, reserve)
- Dynamic pricing is supported with time-based package activation
- All monetary values are stored in lowest denomination (cents for USD)
- Reconciliation must be manually confirmed before draws
- Complete void/refund capabilities with audit trails
- Support for user roles with fine-grained permissions (void, raffle event creation, etc.)

## Compliance Focus

The system is designed specifically for GLI-31 certification with emphasis on:

- Cryptographically secure random number generation
- Immutable audit logs and critical event tracking
- System verification and checksum validation
- Comprehensive reporting for regulatory requirements
- Multi-factor authentication for administrative functions
