# Elite Raffle

A comprehensive electronic raffle system designed for modern raffle operations, providing secure ticket sales, transparent drawing processes, and complete audit trails.

## 🗂️ Project Structure

### Documentation Organization

All project documentation is organized in the `./planning` folder following a structured approach:

- **Active Documentation**: Current and relevant planning documents are stored in `./planning/`
- **Claude Project Documentation**: Current context for Claude are stored in `./planning/claude`
- **Archive Management**: Outdated or superseded documents are moved to `./planning/archive/` rather than deleted, maintaining a complete project history
- **Version Control**: Documentation versions are preserved to track the evolution of requirements and architectural decisions

### Claude project files

In order to use Claude desktop with the project context, the project files for Claude are maintained in the `./planning/claude` folder. These project files can be shared with the team to ensure their local Claude desktop project has all the same context.

### Archived Documentation

The `./planning/archive/` folder contains historical documents including:
- Previous architecture iterations
- Initial planning documents
- Alternative solution proposals
- Sequence diagrams for various user flows
- Legacy endpoint specifications

## 🔧 Development Guidelines

### Cursor Rules Integration

This project includes a comprehensive `cursor-rules.md` file that can be reused across other projects. The file contains:

#### High-Level Development Rules:

1. **Task-Driven Development**: All code changes must be associated with explicitly defined and agreed-upon tasks
2. **Product Backlog Item (PBI) Management**: Structured approach to feature planning and tracking
3. **User Authority**: Clear designation that the user maintains ultimate responsibility and decision-making authority
4. **Controlled File Creation**: AI agents cannot create files outside defined structures without explicit approval
5. **External Package Documentation**: Mandatory research and documentation of third-party libraries before integration
6. **Task Granularity**: Features broken down into small, testable units of work
7. **DRY Principle**: Information maintained in single locations to prevent duplication and inconsistencies

#### Key Workflow Components:

- **PBI Lifecycle Management**: From creation through completion with defined status transitions
- **Task Status Synchronization**: Ensuring consistency across all documentation
- **Testing Strategy**: Comprehensive guidelines for test planning and implementation
- **Change Management**: Structured approach to handling scope changes and updates

The `cursor-rules.md` file serves as a complete AI coding agent policy that can be adapted for other software development projects requiring strict governance and accountability.
