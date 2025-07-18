## RSU Functional Notes

### Authentication & Security

- RSU must have secure login for operators before allowing any ticket sales
  - This will be provided by AWS Cognito
  - Should support PIN for password for operators
- Device must be enrolled in a specific raffle before becoming active
- RSU should display its unique identifier and current checksum value for verification
  - This should be displayed on the login screen, and also in another screen after a user has logged in
- Must authenticate with backend using the logged-in user's Cognito JWT

### Ticket Sales Operations

- For raffle events that are configured for consecutive ticket numbers, each RSU will receive a range of tickets it can sell
- For raffle events that are configured for random ticket numbers, each RSU will receive a range of "seeds" that it can use to calculate the ticket numbers (using format-preserving encryption)
- The allocation range will prevent duplicate ticket numbers across RSUs
- The ticket receipt will be printed from the RSU and will include the required information per GLI-31
- Voided tickets must be flagged and their draw numbers cannot be resold
- Reprinted tickets must clearly show "REPRINT" indication

### Offline Capabilities

- RSU can continue selling tickets when offline by accepting cash transactions (stored in local SQLite database)
- Must maintain critical memory of all tickets sold including draw numbers and validation numbers
- RSU automatically deactivates if local buffer reaches capacity
- Upon reconnection, all offline sales must sync to backend before any new operations (this communication should be as lightweight as possible)

### State Management

- RSU states: unregistered → registered → enrolled → sales closing → sales closed → reconciled
- RSU should show current connection status clearly to operator
- A countdown timer will be triggered for the "sales closing" event at which point further sales will be prevented
- RSU must display confirmation when all sales have been uploaded successfully
- RSU should display the ticket/seed range capacity remaining

### Error Handling

- Must detect and display printer errors: low battery, out of paper, disconnected
- Critical memory corruption must halt RSU operations (determine how to verify this in a simple manner)
- Communication failures must be logged but not affect stored ticket data
- Buffer overflow must trigger automatic RSU deactivation

### Reconciliation Support

- RSU must be capable of displaying that all sales have been uploaded
- After upload confirmation, RSU can be reset/closed for current raffle
- No further sales allowed for current raffle after sales have closed for the raffle event
- The admin dashboard must have a way to view the status of the RSU reconciliation state

### Additional Features

- RSU app should support push notifications for various features, including messaging between operators during an event, and receiving the "sales closing" event
- When a raffle is being closed, there should be a countdown initiated of a configurable time. This should send a notification with the ending timestamp to all RSU units. The RSUs should display a countdown and automatically prevent sales when the ending timestamp is reached, even if the RSU is offline
- Time synchronization with backend is critical for coordinated countdowns
- RSU should validate it's running expected code version before allowing sales
