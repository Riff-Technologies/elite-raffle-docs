# Purpose

Describe the critical file verification process that will be used.

## Build and Deployment Process

### Build Phase

- Compile Go code into Linux binary (`bootstrap`)
- Package binary into ZIP file

### Deploy Phase

- Upload ZIP to versioned S3 path (e.g., `s3://bucket/lambdas/rng/v1.2.3/function.zip`)
- Update Lambda function to use new S3 object
- Wait for Lambda update to complete
- Call `GetFunctionConfiguration` to retrieve Lambda's `CodeSha256`
- Store in S3 bucket (e.g. `ers-verification-974328656764/prod/current.json`):
  - Function name
  - Lambda's CodeSha256 (for verification)
  - Timestamp

Example:
```json
{
  "version": "1.2.3",
  "generated_at": "2024-01-15T10:00:00Z",
  "generated_by": "deployment-pipeline",
  "environment": "prod",
  "checksums": {
    "rng-service": {
      "lambda_name": "elite-raffle-rng-prod",
      "code_sha256": "abc123...",
      "last_modified": "2024-01-15T09:55:00Z"
    },
    "core-service": {
      "lambda_name": "elite-raffle-core-prod",
      "code_sha256": "def456...",
      "last_modified": "2024-01-15T09:56:00Z"
    },
    "validation-generator": {
      "lambda_name": "elite-raffle-validation-prod",
      "code_sha256": "ghi789...",
      "last_modified": "2024-01-15T09:57:00Z"
    }
  },
  "aggregate_checksum": "sha256:xyz789..."  // Hash of all individual checksums
}
```

## Verification Process

### Verification Process (scheduled AND exposed via an API Gateway endpoint for on-demand triggering)

- For each critical Lambda function:
  - Retrieve expected `CodeSha256` from S3 bucket
  - Call `GetFunctionConfiguration` for the Lambda function
  - Compare actual `CodeSha256` with expected value
  - Log result to CloudWatch Logs
  - Return a response with clear VALID/INVALID values to indicate success/failure of verification

### Verification Output

- Generate verification report with:
  - Timestamp of verification
  - Overall status (VALID/INVALID)
  - Individual function results
  - Aggregate checksum of all CodeSha256 values
- Upload report to public S3 bucket
- If any verification fails:
  - Log critical event
  - Send alert notifications
  - Prevent new raffle creation
