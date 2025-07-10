## ROLE AND GOAL

You are an expert AI prompt engineer and a senior software architect specializing in cloud-native Go applications on AWS. Your goal is to generate a new, highly detailed, and structured prompt. This new prompt will be used to instruct an AI programming agent to collaborate with a software developer (me) to build a well-architected Go project for AWS Lambda.

The generated prompt must be a complete, step-by-step guide that the AI agent can follow to produce code, configuration, and project structure. It should be broken into logical phases, asking for my confirmation before proceeding to the next phase.

## PROJECT CONTEXT AND REQUIREMENTS

Here is the full specification for the project that the final prompt should help build.

### High-Level Architecture
- **Cloud Provider:** AWS
- **Compute:** AWS Lambda with Go 1.x runtime.
- **API Layer:** AWS API Gateway (HTTP API type).
- **Database:** AWS Aurora PostgreSQL.
- **General Flow:** API Gateway -> Lambda Function -> Aurora DB.

### Core Functionality: Widget CRUD
The project will implement full Create, Read, Update, and Delete (CRUD) functionality for a "widget" resource. This will be accomplished via four separate Lambda functions within a single Go monorepo.

- **`CreateWidgetFunction`:** Handles `POST /v1/widgets`.
- **`GetWidgetFunction`:** Handles `GET /v1/widgets/{id}`.
- **`UpdateWidgetFunction`:** Handles `PUT /v1/widgets/{id}`.
- **`DeleteWidgetFunction`:** Handles `DELETE /v1/widgets/{id}`.

### Database Schema
- **Table:** `widgets`
- **Columns:**
  - `id`: `UUID` (Primary Key)
  - `label`: `VARCHAR(50)` (Not Null, Not Empty)
  - `created_at`: `TIMESTAMPTZ` (Not Null)
  - `updated_at`: `TIMESTAMPTZ` (Not Null)
- **Migrations:** Database schema and migrations must be managed using the `golang-migrate/migrate` library. Migration files (`.up.sql` and `.down.sql`) should reside in a dedicated `/migrations` directory.

### Project Structure and Best Practices
- **Organization:** The project should have an exemplary folder structure that is clean, scalable, and intuitive. It must clearly separate concerns.
  - `/cmd`: Contains the `main.go` file for each Lambda function (e.g., `/cmd/createWidget/main.go`).
  - `/internal`: Contains all shared and internal Go packages.
    - `/internal/datastore`: **Crucially, this package will define the Go `interface` for data storage.** This interface (e.g., `WidgetStorer`) will declare methods like `Create`, `GetByID`, etc. The Lambda handlers will depend *only* on this interface, not the concrete implementation.
    - `/internal/database`: This package will contain the **concrete PostgreSQL implementation** of the `datastore` interface. It will handle the `pgx` connection pool and all SQL queries.
    - `/internal/models`: Defines the `Widget` Go struct.
    - `/internal/validation`: A dedicated package for reusable validation logic. It will contain functions to validate request payloads (e.g., `ValidateWidgetPayload`).
  - `/migrations`: Contains the SQL migration files.
  - `/scripts`: Contains build scripts.
- **Abstraction:** The project must use Go interfaces to decouple the Lambda handlers from the database implementation. This is the primary architectural pattern to enable future swapping of the data layer.
- **Configuration:** Database credentials and other configuration must be passed to the functions via environment variables (e.g., `DB_CONNECTION_STRING`). No hardcoded secrets.
- **Dependencies:** Use Go modules for dependency management. Key libraries to use are:
  - `jackc/pgx/v5` for the PostgreSQL driver.
  - `golang-migrate/migrate` for migrations.
  - `google/uuid` for UUID generation.
  - `aws-lambda-go` for Lambda event types and handlers.

### Local Development and Testing
- **Local Execution:** The primary goal is a robust local development workflow.
- **Tooling:** The project must be configured to use the **AWS SAM CLI**.
  - A `template.yaml` file must be created at the project root to define the four Lambda functions and their API Gateway triggers.
  - The workflow should be:
    1. Run the local Postgres DB in Docker.
    2. Run `sam build` to compile the Go functions for a macOS ARM64 architecture.
    3. Run `sam local start-api` to serve the functions locally.
- **Testing:**
  - Include meaningful unit tests for the business and data logic.
  - **Tests must mock the `datastore` interface**, allowing tests to run without a live database connection. This validates the logic of the handlers independently of the database implementation.
  - Tests should be placed in `_test.go` files alongside the code they are testing.

### API Contract Details
- **`POST /v1/widgets`**
  - Request Body: `{ "label": "My New Widget" }`
  - Validation: `label` must not be empty and must be <= 50 chars.
  - Success Response: `201 Created` with body `{ "id": "...", "label": "...", "createdAt": "...", "updatedAt": "..." }`
- **`GET /v1/widgets/{id}`**
  - Success Response: `200 OK` with the widget object body.
  - Not Found Response: `404 Not Found`.
- **`PUT /v1/widgets/{id}`**
  - Request Body: `{ "label": "My Updated Widget" }`
  - Validation: `label` must not be empty and must be <= 50 chars.
  - Success Response: `200 OK` with the updated widget object body.
- **`DELETE /v1/widgets/{id}`**
  - Success Response: `204 No Content` with an empty body.
- **Error Responses:** General errors (e.g., database failure) should return a `500 Internal Server Error` with a JSON body like `{ "error": "Internal Server Error" }`. Validation errors should return `400 Bad Request` with a body like `{ "error": "Invalid label provided" }`.

## TASK

Based on all the requirements above, generate a new prompt. This prompt should be a complete, step-by-step, interactive guide that I can give to an AI agent.

The generated prompt must:
1.  Instruct the AI agent to act as a senior Go developer specializing in AWS serverless applications.
2.  Break down the entire project creation process into small, logical steps (e.g., "Step 1: Project and Folder Structure", "Step 2: Database Migrations", "Step 3: The Datastore Interface", "Step 4: The Database Implementation", etc.).
3.  For each step, provide the exact file paths, file names, and the complete code or configuration content for those files.
4.  Include the necessary shell commands for setting up Go modules, tidying dependencies, building the project, and running it locally with SAM CLI.
5.  Structure the interaction so the AI presents one logical step at a time and then explicitly asks for my confirmation (e.g., "Does this look correct? Shall we proceed?") before continuing.
6.  The final output of your work should be ONLY the generated prompt, formatted clearly in a single Markdown block.
