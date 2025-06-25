# GLI proposal for architecture

## Compliance Questions

- Will AWS KMS FIPS 140-2 Level 2 be sufficient for the RNG?
- Does the game system need to explicitly support the same number of tickets, number of winners, number of prizes?
  - Or can it be dynamic (X number of tickets, Y number of winners, replacement true/false)?
  - Can this be handled by the business logic? Or is it a "critical file" that can't be updated?
- Does the game system (or critical files) include payment integration? e.g. Stripe, Payfacto
- How do we handle "closing" raffles that had a manual draw?
  - e.g. in MN they barrel print and pull a winning ticket
  - we need to add that "winning ticket" to the database
- For a serverless system, what is included in the checksum?
  - Does it include the infrastructure code?
  - How would you deploy it for lab testing?
- Can we control immutable logs via application code?
  - Do we need to use truly immutable logs?
- Would it a good idea to have documented tests trying to break code?
  - e.g. try to update an audit log
- self Validation
  - how does this work with a distributed system?
- what is remote access?
- do we want to offer a second chance ticket?

## Architecture questions

- Does it make sense / is it acceptable to maintain 2 DBs: core raffle, business logic
- Would it be feasible to have a single Lambda function for all of the methods?
  - Might be easier to deploy, validate, etc.
  - But might have implications for scalability
- How do we handle RSUs?
  - It would be preferable to store this in the business logic DB
- How do we handle allocating tickets to an RSU?
- Can we have a countdown timer for closing the raffle?
  - also need to be able to "fix" or "reconcile" any sold tickets after the deadline are impossible to generate

## Rafflebox issues

- Selling tickets on RSU after the raffle is closed
  - makes it impossible to draw
- Voiding a CC transaction does not automatically refund (requires the last 4 digits)

## Critical Files

Here is the list of files that we consider to be critical for an Electronic Raffle System:
- The RNG algorithm and implementation
- The files responsible for “Control program (self) verification”
- Anything related to “Critical memory handling and validation”
- Source code responsible for “Generation of ticket validation numbers.”
- Source code related to Raffle Event Logic (i.e. ticket allocation, refunds, winner determination, etc.).

## RNG

- Use AWS KMS GenerateRandom API
  - FIPS 140-2 Level 2 validated HSMs
- Lambda function to generate the random bytes
  - AES-256-CTR-DRBG deterministic random bit generator (DRBT)
  - 256-bit entropy
- Secure API Gateway
  - Encrypted endpoints with mutual authentication
  - AWS signature version 4 (sigv4)
  - IAM auth with least privilege
  - rate limiting
- Fresh KMS entropy generation on each Lambda cold start
- In-memory DRBG state (destroyed on function termination)
- No persistent storage of intermediate RNG states
- Cryptographically secure scaling algorithms for raffle number ranges
- Serverless Seeding Strategy
  - Initial Seed: AWS KMS GenerateRandom + Lambda execution context entropy
  - Seed Composition: KMS entropy || Lambda request ID || execution timestamp || memory allocation patterns
  - Seed Mixing: SHA-256 hash of combined entropy sources
  - Lambda Context: Unique execution environment provides additional entropy

## Monitoring

- Cloudwatch for monitoring, alerting, statistical validation



## Gaming system endpoints

### System management
- GET    /health                              # System health check
- GET    /verification                        # Component hashes (5 critical files)
- GET    /system/clock                        # System time (Section 5.3)
- POST   /system/clock/sync                   # Synchronize clocks across components
- GET    /system/configuration                # System-wide settings
- PUT    /system/configuration                # Update system settings (supervised)

### Operator management
- GET    /operators                           # List all operators (paged, filtered)
- GET    /operators/{id}                      # Get operator details
- POST   /operators                           # Create operator (sync from Cognito)
- PUT    /operators/{id}                      # Update operator
- DELETE /operators/{id}                      # Deactivate operator
- POST   /operators/{id}/assign-rsu           # Assign operator to RSU
- POST   /operators/{id}/unassign-rsu         # Remove operator from RSU
- GET    /operators/{id}/activity             # Operator activity log
- POST   /operators/authenticate              # Record login (after Cognito auth)
- POST   /operators/{id}/logout               # Record logout
- GET    /operators/{id}/permissions          # Get operator permissions
- PUT    /operators/{id}/permissions          # Update operator permissions

### Raffle Management
- POST   /raffles                             # Create raffle
- GET    /raffles                             # List raffles (paged, filtered, sorted)
- GET    /raffles/{id}                        # Get raffle header
- GET    /raffles/{id}/details                # Get full raffle details
- PUT    /raffles/{id}/configuration          # Update config (before sales only)
- PUT    /raffles/{id}/close-sales            # Close ticket sales
- GET    /raffles/{id}/status                 # Get raffle status
- DELETE /raffles/{id}                        # Cancel raffle (before sales only)

### Draw management
- POST   /raffles/{id}/draw                   # Execute/record drawing
- GET    /raffles/{id}/draw-results           # Get drawing results
- POST   /raffles/{id}/draw-results/verify    # Verify draw results official
- GET    /raffles/{id}/winners                # List all winners
- POST   /raffles/{id}/rng-draw               # Execute electronic RNG draw

### Ticket management
- GET    /raffles/{id}/tickets                # List tickets for raffle
- GET    /tickets/{id}                        # Get ticket details
- POST   /tickets/sell                        # Record ticket sale
- POST   /tickets/batch-sell                  # Batch ticket sales
- POST   /tickets/{id}/void                   # Void ticket
- POST   /tickets/batch-void                  # Batch void tickets
- GET    /raffles/{id}/tickets/voided         # List voided tickets
- POST   /tickets/validate                    # Validate by validation number
- GET    /tickets/search                      # Search tickets (multiple criteria)

### Winner management
- POST   /winners/verify                      # Verify winning ticket
- POST   /winners/{id}/claim                  # Record prize claim
- GET    /winners/{id}/status                 # Get claim status
- PUT    /winners/{id}/documentation          # Update winner documentation

### RSU management
- GET    /rsus                                # List RSUs (paged, filtered, sorted)
- GET    /rsus/{id}                           # Get RSU details
- POST   /rsus/enroll                         # Enroll new RSU
- POST   /rsus/{id}/unenroll                 # Unenroll RSU
- PUT    /rsus/{id}/enable                   # Enable RSU for sales
- PUT    /rsus/{id}/disable                  # Disable RSU
- POST   /rsus/{id}/validate                 # RSU validation/heartbeat
- GET    /rsus/{id}/validation-history       # Validation attempt history
- PUT    /rsus/{id}/configuration            # Update RSU settings

### RSU Ticket management
- POST   /rsus/{id}/allocate-tickets         # Allocate ticket batch to RSU
- GET    /rsus/{id}/tickets                  # Get RSU's allocated tickets
- POST   /rsus/{id}/reconcile                # Reconcile RSU sales
- GET    /rsus/{id}/reconciliation-status    # Get reconciliation status
- POST   /rsus/{id}/return-tickets           # Return unsold allocated tickets

### Reporting
- GET    /reports/raffle-drawing/{id}        # Raffle drawing report
- GET    /reports/exceptions                 # Exception report
- GET    /reports/bearer-tickets/{id}        # Bearer tickets report
- GET    /reports/sales-by-rsu               # Sales by RSU report
- GET    /reports/voided-tickets             # Voided draw numbers report
- GET    /reports/rsu-events                 # RSU event log
- GET    /reports/rsu-corruption             # RSU corruption log
- POST   /reports/custom                     # Generate custom report
- GET    /reports/financial-summary/{id}     # Financial summary for raffle

### Event & Audit
- GET    /events                             # List significant events
- GET    /events/search                      # Search events (date, type, component)
- GET    /events/{id}                        # Get event details
- GET    /audit/changes                      # Data modification audit trail
- GET    /audit/access                       # Access log (including remote)
- GET    /audit/authentication               # Authentication attempts log

### RNG Management
- GET    /rng/status                         # RNG health and statistics
- GET    /rng/verification                   # Verify RNG integrity
- POST   /rng/test                           # Run RNG statistical tests
- GET    /rng/entropy-status                 # Check entropy pool status
- POST   /rng/reseed                         # Force RNG reseed (supervised)

### Security
- GET    /firewall/status                    # Firewall status
- GET    /firewall/logs                      # Firewall audit logs
- GET    /remote-access/sessions             # Active remote sessions
- GET    /remote-access/history              # Remote access history

### Backup & Recovery
- POST   /backup/create                      # Create system backup
- GET    /backup/list                        # List available backups
- POST   /backup/restore                     # Restore from backup (supervised)
- GET    /backup/verify/{id}                 # Verify backup integrity

## RSU (raffle sales unit)

- RSU must track the user who is assigned to it
- Must check in and authenticate with the system at customizable intervals (heartbeat / keepalive)
- System tracks last validation timestamp for each RSU
- Disablement RSU if failed validation attempts exceed customizable threshold
- Remote disablement support
  - RSU must either validate before each sale or respond to push commands (push notifications)
  - Tickets allocated to the device have to be handled somehow
    - Void them, or reallocate?
- Corruption log
  - Data in the system doesn't match what is in the RSU
  - Need a process to reconcile the RSU and the allocated tickets
- Develop a state machine for RSU states (active > warning > disabled > corrupted)
- Develop a strategy for reconciling corrupted RSUs
