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
3. **Inventory Management**
- RSUs allocated sequential ticket blocks
- Packages consume appropriate quantity from RSU’s allocated tickets
- No pre-allocation or reservation of tickets for specific packages
4. **Jackpot Seeding**
- Seed amount = guaranteed minimum jackpot
- Jackpot grows when event revenue exceeds seed amount
- Revenue calculation method should be configurable per organization/event (e.g. net revenue, gross revenue)
5. **RSU Synchronization**
- RSUs cache ticket package pricing data
- Package updates fetched during regular RSU check-ins with system
- RSUs report specific package sold + tickets consumed (eliminates offline pricing conflicts)
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
