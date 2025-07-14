### Core Decisions Made:

1. **Dynamic Pricing Architecture**
- Ticket packages can be updated in real-time by admins during active events
- Optional scheduled pricing with `activeFrom`/`activeTo` timestamps
- No advance reservation of ticket blocks for specific packages
- Future: Automatic pricing adjustments based on metrics/data points
2. **Ticket Package Structure**
- Packages define: name, ticket count, price, active status, display order
- Open-ended perks system that allows ticket packages to include perks (type + description)
- Some perks may have limited inventory that needs tracking
- Packages configured separately from raffle creation (dedicated endpoints needed)
3. **Ticketing**
- Tickets will be created "just-in-time", meaning a ticket record will be created at the time of a sale.
  - This will limit the records the database has to store, and will make allocation of tickets simpler.
- A raffle event can define "sequential" or "random" ticket sales.
- For sequential:
  - Ticket ranges will be allocated when the raffle is created:
    - A range will be available for RSUs, and a separate range will be available for online orders.
- For random:
  - A "seed" range will be allocated to each RSU.
  - Each seed will use a permutation function that maps each seed to a unique random ticket number across the entire raffle range.
  - This will be done with format-preserving encryption / permutation.
- An RSU will request a ticket range allocation, which will allocate a block of tickets to the RSU.
- The entire order must be voided, not individual tickets, if a void is required.
  - The value associated with that voided order is subtracted from the jackpot.
4. **Jackpot Seeding**
- Seed amount = guaranteed minimum jackpot
- Jackpot grows when event revenue exceeds seed amount
- Revenue calculation method should be configurable per organization/event (e.g. net revenue, gross revenue)
5. **RSU**
- RSUs cache ticket package pricing data
- RSUs will calculate their own unique "validation" numbers (alphanumeric characters) as ticket orders are created.
  - The validation number generation is a critical file, and must be shared on the RSU and the core system.
  - The validation numbers must not be predictable, and must be unique across a raffle event.
  - They should be relatively short to enable manually entry in the admin website, but they must be unique.
- Package updates fetched during regular RSU check-ins with system
- RSUs report specific package sold + tickets consumed (eliminates offline pricing conflicts)
- An RSU can be associated with an organization and/or a venue
- When our app is started, it will allow the user to select from a list of raffles related to the organization and/or venue its associated with to enable enrollment in the raffle event.
- If the RSU has no organization nor venue association, it can be considered an "admin" RSU and will fetch a list of all raffle events.
- An RSU can manage a "PIN" for a user, in lieu of the full password, to make logging in quick during an event.
  - The full password will be required initially, and then the user can set a PIN for the event to quickly access the event.
- RSUs will be deployed by the payment facilitator.
6. **Sales Tracking**
- Pricing changes must be tracked for reconciliation/refunds
7. **Raffle State**
- Raffle event state should remain in `created` status until it is `activated`, and an `isConfigured` property should be calculated upon getting raffle details.
- Raffle events need to automatically start at the start date that was configured, or by manually starting the event.
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
- Lambda functions should be single-responsibility, and have a 1:1 relationship with API Gateway endpoints, e.g. POST user endpoint will hit a specific Lambda function for creating a user.
10. **Critical Files**
- Critical files should be as simple as possible, and should generally be executable without any dependencies.
  - This will enable GLI to test the functionality of the critical files, e.g. RNG, without the overhead over AWS dependencies, logging, etc.
  - Those dependencies should be injected so that the modules can be independent of them for the purposes of certification.
11. **SQL**
- PostgreSQL Database will be used to store the raffle data and associated data.
- The database should not maintain business logic nor stored procedures CRUD.
- Business logic should be maintained in the Lambda services, and a separate Go module should be created to write data to the database, and another separate module should be created to read data from the database.
12. **Users**
- Users will be created in AWS Cognito.
- Users will also have a record in the raffle database, where the user data can be maintained, such as name and permissions.
- Cognito will use custom attributes for: role (admin, operator, customer), organization membership (which organizations the user can access), MFA settings
- The database will store user permissions (e.g. can void a ticket, can refund, etc.), organization-specific roles (admin, operator), permission history.
13. **Onboarding of other clients**
- We want the ability to license our raffle system to other clients to use our software to run their own raffles.
- Need to be able to quickly onboard other clients to use our software.
- PaySafe (or another platform) can act as a payment facilitator which would pay the other clients directly to their bank accounts.
- The system needs to be architected in such a way that enables this type of relationship with clients.
