### Core Decisions Made:

1. **Dynamic Pricing Architecture**

- Ticket packages can be updated in real-time by admins during active events (to be confirmed with GLI that this is acceptable)
  - This might also be based on jurisdiction
- Optional scheduled pricing with `activeFrom`/`activeTo` timestamps
- No advance reservation of ticket blocks for specific packages
- Future: Automatic pricing adjustments based on metrics/data points (if GLI confirms it can be dynamic during an event vs pre-planned changes)

2. **Ticket Package Structure**

- Packages define: name, ticket count, price, active status, display order
- Open-ended perks system that allows ticket packages to include perks (type + description)
- Some perks may have limited inventory that needs tracking of inventory
- Packages configured separately from raffle creation (dedicated endpoints needed)
  - Raffles should be able to select the ticket packages available for the event

3. **Ticketing**

- Tickets will be created "just-in-time", meaning a ticket record will be created at the time of a sale.
  - This will limit the records the database has to store, and will make allocation of tickets simpler.
- A raffle event can define "sequential" or "random" ticket sales.
- Ticket ranges will be allocated when the raffle is created:
  - A range will be available for RSUs, and a separate range will be available for online orders, and potentially other ranges will be allocated, like admin "reserve" range.
- For sequential:
  - The RSU will get a range of tickets allocated to it, from the total pool of tickets available for RSUs.
- For random:
  - A "seed" range will be allocated to each RSU.
  - Each seed will use a permutation function that maps each seed to a unique random ticket number across the entire raffle range.
  - This will be done with format-preserving encryption / permutation.
- An RSU will request a ticket range allocation, which will allocate a block of tickets to the RSU.
- For voids, the entire "order" of tickets will be voided, not individual tickets.
  - The value associated with that voided order is subtracted from the jackpot.
    - This includes any calculations required for jackpots and jackpot displays.

4. **Jackpots**

- Jackpots for displays should be calculated in realtime, based on the rules for the raffle and the regulations.
  - For example, a voided ticket should not directly subtract from the jackpot, but its state of "voided" should cause it to be excluded from the jackpot calculation.
- Seed amount = the amount of money the raffle actually starts with
  - Every sale adds to the jackpot
- Minimum guaranteed jackpot = the amount of money that is guaranteed
  - Jackpot grows when event revenue exceeds seed amount
  - This depends on jurisdictional requirements as well (gross vs net jackpots)
- Revenue calculation method should be configurable per organization/event (e.g. net revenue, gross revenue)
  - Jurisdictions might require that the gross jackpot is displayed to the public, and other jurisdictions might require that the net jackpot is displayed (total gross minus any fees to operate the raffle)

5. **RSU**

- RSUs cache ticket package pricing data
- RSUs will calculate their own unique "validation" numbers (alphanumeric characters) as ticket orders are created.
  - The validation number generation is a critical file, and must be shared on the RSU and the core system.
  - The validation numbers must not be predictable, and must be unique across a raffle event.
  - They should be relatively short to enable manually entry in the admin website, but they must be unique.
- Package updates fetched during regular RSU check-ins with system, if manual ticket package updates are allowed after a raffle event has started.
  - If not allowed, then the RSUs will receive the ticket packages along with the other raffle event information when they are enrolled in the event.
- RSUs report specific package sold + tickets consumed (eliminates offline pricing conflicts)
- An RSU can be associated with an organization and/or a venue
  - An RSU will be configured with certain merchant data for the built-in payment provider, so will need to be configured by organization or venue.
  - This is related to bank account info for the venue or organization.
- When our app is started, it will allow the user to select from a list of raffles related to the organization and/or venue its associated with to enable enrollment in the raffle event.
- If the RSU has no organization nor venue association, it can be considered an "admin" RSU and will fetch a list of all raffle events.
- An RSU can manage a "PIN" for a user, in lieu of the full password, to make logging in quick during an event.
  - The full password will be required initially, and then the user can set a PIN for the event to quickly access the event.
  - The PIN will be cached on the RSU itself, rather than on the server side.
- RSU devices will be deployed by the payment facilitator, e.g. Paysafe.
  - They will be configured with payment information, such as bank account, merchant settings, etc.
- RSU app should have a feature to view recent sales or transactions, see the statuses, and quickly send a message to the admin to void a ticket.

6. **Sales Tracking**

- Pricing changes must be tracked for reconciliation/refunds
  - A log table should exist to track the history of ticket package pricing changes.

7. **Raffle State**

- Raffle event state should remain in `created` status until it is `activated`, and an `isConfigured` property should be calculated upon getting raffle details.
- Raffle events need to automatically start at the start date that was configured, or by manually starting the event.
- Raffle events should end automatically at the end date that was configured, or by manually ending the raffle before the end date.

8. **Infrastructure as code**

- Infrastructure will be created with Terraform and Terragrunt.
- The IaC code will live in separate repos and will not be part of the service projects.

9. **Lambda Services**

- AWS Lambda functions will be used for the application services.
- A project repo will be created for each service, which contain all the functions that can be compiled and deployed.
- The project should contain one or more Go files that can be compiled and uploaded to S3.
- The compilation and upload will be part of the deployment pipeline, and will not be part of the service project.
- Env vars will be injected by the deployment pipeline or in Terraform.
- The naming of Go modules should follow the convention `github.com/riff-technologies/{module_name}`
- Lambda functions should be single-responsibility, and have a 1:1 relationship with API Gateway endpoints, e.g. POST /v1/user endpoint will hit a specific Lambda function for creating a user.

10. **Critical Files**

- Critical files should be as simple as possible, and should generally be executable without any dependencies.
  - This will enable GLI to test the functionality of the critical files, e.g. RNG, without the overhead over AWS dependencies, logging, etc.
  - Those dependencies should be injected so that the modules can be independent of them for the purposes of certification.
  - Interfaces should be used for all external dependencies, as opposed to concrete implementations.
- Critical files should emit events rather than doing things like writing directly to a database.
  - This will enable us flexibility for what is executed for each event that is triggered.
- Configuration-driven behavior should be preferred, e.g. SSM Parameter Store, rather than controlling behavior via the code.
- A plugin architecture should be considered so that new functionality can be loaded at runtime without modifying critical files.

11. **SQL**

- PostgreSQL Database will be used to store the raffle data and associated data.
- The database should not maintain business logic nor stored procedures CRUD.
- Business logic should be maintained in the Lambda services, and a separate Go module should be created to write data to the database, and another separate module should be created to read data from the database.
  - The module to write data to the database can be split into 2 modules: 1 for critical data (certified), and 1 for non-critical data (non-certified).

12. **Users**

- Users will be created in AWS Cognito.
- Users will also have a record in the raffle database, where the user data can be maintained, such as name and permissions.
- Cognito will use custom attributes for: role (admin, operator, customer), organization membership (which organizations the user can access), MFA settings
- The database will store user permissions (e.g. can void a ticket, can refund, etc.), organization-specific roles (admin, operator), permission history.

13. **Onboarding of other clients**

- We want the ability to license our raffle system to other clients to use our software to run their own raffles.
- Need to be able to quickly onboard other clients to use our software.
- Paysafe (or another platform) can act as a payment facilitator which would pay the other clients directly to their bank accounts.
- The system needs to be architected in such a way that enables this type of relationship with clients.

14. **System Verification**

- A verification Lambda function will pull the expected checksums of the critical files from SSM Parameter Store and compare those the value of each Lambda function's `GetFunctionConfiguration.CodeSha256` values.
- The verification Lambda will be scheduled to execute daily, and will be executable on-demand via an API Gateway endpoint.
- The results will be logged to CloudWatch logs, and will be uploaded to an S3 bucket for display on the public website.

### TBD:

- What payload does the RSU send to the server for ticket sales?
  - Should it be an array of orders? Or should it be a request for each order?
  - It should include order numbers, tickets sold, and ticket packages & pricing.
