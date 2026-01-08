# Money Planner Documentation

Comprehensive documentation for the Money Planner microservices architecture, covering system design, features, and AI-assisted development guidelines.

## Quick Navigation

- [Repository Structure](#repository-structure)
- [System Documentation](#system-documentation)
- [Feature Roadmap](#feature-roadmap)
- [AI/LLM Development](#aillm-development)
- [Service Documentation](#service-documentation)
- [Documentation Standards](#documentation-standards)

## Repository Structure

```
monney-planner-docs/
├── prompt-context/          # AI/LLM prompt engineering templates
├── system-overview/         # Architecture, MVP status, production readiness
│   ├── features/           # PRDs organized by status (implemented/in-progress/to-implement)
│   ├── transaction-service/ # Transaction service documentation
│   └── bff-service/        # BFF/API gateway documentation
└── README.md               # This file
```

## System Documentation

### Core Documents
- **[system-overview.md](system-overview/system-overview.md)** - Complete microservices architecture
- **[MVP-FEATURE-SUMMARY.md](system-overview/MVP-FEATURE-SUMMARY.md)** - MVP status and roadmap
- **[SasS-prod-readiness-checklist.md](system-overview/SasS-prod-readiness-checklist.md)** - Production transformation plan

### Current Status
- **MVP Progress**: 92% complete
- **Production Readiness**: 45/100 (target: 90/100 in 16 weeks)
- **Active Features**: 2 in-progress, 7 planned

## Feature Roadmap

### Implemented (2)
- [PRD-async-ai-categorization.md](system-overview/features/implemented/PRD-async-ai-categorization.md)
- [PRD-generic-categories.md](system-overview/features/implemented/PRD-generic-categories.md)

### In-Progress (2)
- [PRD-draft-budgets.md](system-overview/features/in-progress/PRD-draft-budgets.md) - 97% complete
- [PRD-budget-settlement.md](system-overview/features/in-progress/PRD-budget-settlement.md)

### To-Implement (7)
- [PRD-backoffice-analysis.md](system-overview/features/to-implement/PRD-backoffice-analysis.md)
- [PRD-backoffice-application.md](system-overview/features/to-implement/PRD-backoffice-application.md)
- [PRD-budget-settlement.md](system-overview/features/to-implement/PRD-budget-settlement.md)
- [PRD-opentelemetry.md](system-overview/features/to-implement/PRD-opentelemetry.md)
- [PRD-sign-with-google.md](system-overview/features/to-implement/PRD-sign-with-google.md)
- [PRD-stripe-integration.md](system-overview/features/to-implement/PRD-stripe-integration.md)
- [PRD-user-mailing.md](system-overview/features/to-implement/PRD-user-mailing.md)

## AI/LLM Development

### Prompt Templates (prompt-context/)
- **[prompt-engineer.md](prompt-context/prompt-engineer.md)** - Comprehensive prompt engineering methodology
- **[general-refactoring.md](prompt-context/general-refactoring.md)** - Code change proposal template
- **[healt-check-prompts.md](prompt-context/healt-check-prompts.md)** - System diagnostics prompts
- **[modular-refactoring-prompt.md](prompt-context/modular-refactoring-prompt.md)** - Modular refactoring guidelines
- **[refactoring-impl.md](prompt-context/refactoring-impl.md)** - Refactoring implementation details

## Service Documentation

### Frontend (MoneyPlannerFE)
- React 18 + TypeScript + Material-UI
- User interface and expense tracking

### Transaction Service
- [feature-budget.md](system-overview/transaction-service/feature-budget.md)
- [post-mvp-budget-features.md](system-overview/transaction-service/post-mvp-budget-features.md)
- [transaction-service-openapi.json](system-overview/transaction-service/transaction-service-openapi.json)

### BFF Service
- [bff-openapi.json](system-overview/bff-service/bff-openapi.json)

## Documentation Standards

All Product Requirements Documents (PRDs) follow the standardized structure defined in:
- **[PRD-rules.md](system-overview/features/PRD-rules.md)** - PRD structure, format rules, and writing guidelines

### PRD Structure
- Document Header (Version, Date, Status)
- Problem Statement (current state, pain points, impact)
- Current System Analysis (architecture, endpoints, schema)
- Functional Requirements (acceptance criteria)
- Implementation Details (when applicable)

## Architecture Highlights

**Microservices**: Frontend (React), BFF (Rust), User Service, Transaction Service, Budget Service
**Database**: PostgreSQL (single instance, multiple databases)
**Events**: Kafka for real-time updates
**Infrastructure**: Kubernetes with Helm charts
**Security**: Argon2 password hashing, JWT inter-service auth, session-based user auth
**AI**: OpenAI integration for transaction categorization

---

For full architecture details, see [system-overview.md](system-overview/system-overview.md).
