# Project Organization & Structure

## Repository Structure

This repository contains planning and documentation for the Elite Raffle System. The actual implementation will be split across multiple repositories following the architecture decisions.

### Current Structure

```
├── .kiro/                    # Kiro AI assistant configuration
│   └── steering/            # AI guidance documents
├── agent-rules/             # AI agent instructions and conventions
│   ├── AGENTS.md           # Core agent instructions
│   └── archive/            # Historical agent rules
├── planning/               # All project planning documentation
│   ├── claude/            # Claude AI project context files
│   ├── datastore/         # Database schema and design
│   ├── endpoints/         # API specifications
│   └── archive/           # Historical planning documents
└── prompts/               # AI prompt templates
```

## Documentation Organization

### Active vs Archive Pattern

- **Active Documentation**: Current and relevant documents are in root folders
- **Archive Management**: Outdated documents moved to `archive/` subfolders
- **Version Control**: Document evolution tracked through git history
- **Context Preservation**: Historical decisions preserved for reference

### Planning Documentation

- **`planning/claude/`**: Current Claude AI project context
- **`planning/endpoints/`**: API specifications and endpoint definitions
- **`planning/datastore/`**: Database schemas and data models
- **`planning/archive/`**: Historical architecture iterations and alternatives

## Future Repository Structure

The implementation will be organized across separate repositories:

### Service Repositories

Each Lambda service will have its own repository:

- `raffle-core-service` - Core raffle lifecycle management
- `rng-service` - Random number generation (critical file)
- `payment-service` - Payment processing integration
- `reporting-service` - GLI-31 compliance reporting

### Infrastructure Repositories

- `infrastructure-terraform` - Terraform/Terragrunt IaC
- `infrastructure-shared` - Shared infrastructure components

### Frontend Repositories

- `public-website` - Customer-facing ticket purchase site
- `admin-website` - Administrative management interface
- `rsu-android-app` - Retail Sales Unit mobile application

## File Naming Conventions

- Use kebab-case for directories and files
- Prefix version numbers for iterative documents (e.g., `v1`, `v2`)
- Use descriptive names that indicate purpose and scope
- Archive files maintain original names for traceability

## Documentation Standards

- All planning documents in Markdown format
- API specifications in OpenAPI 3.0 YAML
- Database schemas in SQL with comprehensive comments
- Architecture diagrams in Mermaid format when present
