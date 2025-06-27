## RSU Functional Notes

### Authentication & Security
- RSU must have secure login for operators before allowing any ticket sales
- Device must be enrolled in a specific raffle before becoming active
- RSU should display its unique identifier and current checksum value for verification
- Must authenticate with backend using API keys that rotate every 90 days

### Ticket Sales Operations
- RSU must prevent duplicate ticket numbers within the same raffle event
- Each ticket sold must include the required data, which will be defined in a different document
- Voided tickets must be flagged and their draw numbers cannot be resold
- Reprinted tickets must clearly show "REPRINT" indication

### Offline Capabilities
- RSU can continue selling tickets when offline (stored in local SQLite database)
- Must maintain critical memory of all tickets sold including draw numbers and validation numbers
- RSU automatically deactivates if local buffer reaches capacity
- Upon reconnection, all offline sales must sync to backend before any new operations (this communication should be as lightweight as possible)

### State Management
- RSU states: registered → enrolled → active → offline → online → closing → closed → reconciled
- RSU should show current state clearly to operator
- When raffle moves to 'sales_closed', RSU automatically enters 'closing' state
- RSU must display confirmation when all sales have been uploaded successfully

### Error Handling
- Must detect and display printer errors: low battery, out of paper, disconnected
- Critical memory corruption must halt RSU operations (determine how to verify this in a simple manner)
- Communication failures must be logged but not affect stored ticket data
- Buffer overflow must trigger automatic RSU deactivation

### Reconciliation Support
- RSU must be capable of displaying that all sales have been uploaded
- After upload confirmation, RSU can be reset/closed for current raffle
- No further sales allowed for current raffle after closing

### Additional Features
- RSU app should support push notifications for various features, including messaging between operators during an event
- When a raffle is being closed, there should be a countdown initiated of a configurable time. This should send a notification with the ending timestamp to all RSU units. The RSUs should display a countdown and automatically prevent sales when the ending timestamp is reached, even if the RSU is offline
- Time synchronization with backend is critical for coordinated countdowns
- RSU should validate it's running expected code version before allowing sales
