# Project progress

This document tracks what we've completed so far, but does not track what remains to be done.

### Infrastructure

- VPC created.
- A test Lambda function created for a "hello world" response.
- A test API Gateway created in the AWS console to test the Lambda function.
- Github Actions pipeline is configured to deploy infrastructure on push to main.

### Services

- RNG function created with Lambda bootstrap, but not deployed to AWS yet.
- Data models completed.
- CRUD demonstration project created for Go Lambda function development.
- Jupyter notebook project created that demonstrates how RSUs can be assigned a range of "seeds" that map to ticket numbers using FPE algorithm.
- API Spec created for MVP phase.

### Database

- First pass at SQL database tables for the raffle core system created locally, but not deployed.
  - Scripts created for table creation.
