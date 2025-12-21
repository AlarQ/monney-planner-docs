# Product Requirements Document (PRD) Construction Rules

## Document Purpose
This document defines the standard structure, format, and content requirements for all Product Requirements Documents (PRDs) in the Money Planner project. These rules ensure consistency, completeness, and clarity across all feature documentation.

**Target Audience**: Solo developers who need structured documentation for complex features without team collaboration overhead.

## Document Structure

### 1. Document Header (Required)

Every PRD must begin with:

---
# PRD: [Feature Name]

## Document Information
- **Version**: [X.Y]
- **Date**: [Month Year]
- **Status**: [Draft | Under Implementation | Implemented]

---

**Rules**:
- Version follows semantic versioning (1.0, 1.1, 2.0)
- Date format: "1 January 2025" (day + month name + year)
- Status workflow: Draft → Under Implementation → Implemented
- No approval gates - you own the decision

### 2. Problem Statement (Required)

**Purpose**: Clearly define the problem being solved

**Content**:
- Current state description
- Pain points and limitations
- Impact on users or business
- Why this problem needs solving now

**Format Example**:

---
## Problem Statement

Currently, [current state]. This creates several issues:
- [Pain point 1]
- [Pain point 2]
- [Pain point 3]

Without [feature], users cannot [capability], leading to [negative impact].

---

**Rules**:
- Must be specific and measurable
- Should reference actual user feedback or data when available
- Must explain why current solution is insufficient
- Should avoid vague statements like "improve user experience"

### 3. Current System Analysis (Required)

**Purpose**: Document existing architecture and identify gaps

**Content**:
- Existing system architecture
- Current implementation details
- Database schema (if applicable)
- API endpoints (if applicable)
- Integration points
- Known limitations and gaps

**Format Example**:

---
## Current System Analysis

### Existing Architecture

**Service Name** (`service-name`):
- **Current State**: [Description]
- **Endpoints**: [List of endpoints]
- **Database Schema**: [Relevant tables/fields]
- **Missing**: [What's not implemented]
- **Limitations**: [Current constraints]

---

**Rules**:
- Must be accurate (verify against actual codebase)
- Should reference actual file paths or code locations when relevant
- Must identify all integration points
- Should clearly distinguish between "exists" and "missing"
- Use ✅ for implemented features, ❌ for missing features, ⚠️ for partial/incomplete

### 4. Functional Requirements (Required)

**Purpose**: Define what the system must do with clear acceptance criteria

**Content**:
- Core functionality requirements
- User experience requirements
- Business rules
- Edge cases and error handling
- Acceptance criteria for each requirement (embedded)

**Format Example**:

---
## Functional Requirements

### FR-1: [Requirement Name]

**Requirement**: [What must be done]

**Details**:
- [Detail 1]
- [Detail 2]
- [Detail 3]

**Acceptance Criteria**:
- [ ] [Testable criterion 1]
- [ ] [Testable criterion 2]
- [ ] [Testable criterion 3]

---

**Rules**:
- Number requirements sequentially (FR-1, FR-2, etc.)
- Each requirement must have clear acceptance criteria inline
- Must be testable and measurable
- Should include edge cases and error scenarios
- Group related requirements logically
- Use subsections for complex requirements (e.g., FR-1.1, FR-1.2)
- Use checkboxes for acceptance criteria to track completion

### 5. Technical Requirements (Required)

**Purpose**: Define how the system must be built

**Content**:
- Backend requirements (database, APIs, services)
- Frontend requirements (components, UI/UX)
- Integration requirements (APIs, events, services)
- Infrastructure requirements (deployment, scaling)
- Performance requirements
- Testing requirements (unit, integration, E2E)

**Format Example**:

---
## Technical Requirements

### TR-1: [Technical Requirement Name]

**Requirement**: [What must be implemented]

**Details**:
- [Implementation detail 1]
- [Implementation detail 2]

**Database Schema**:
```sql
[SQL schema if applicable]
```

**API Endpoints**:
```http
[API specification]
```

**Testing Requirements**:
- **Unit Tests**: [What must be unit tested]
- **Integration Tests**: [What must be integration tested]
- **E2E Tests**: [What must be E2E tested]

---

**Rules**:
- Number requirements sequentially (TR-1, TR-2, etc.)
- Must include actual code examples or specifications
- Database schemas must include indexes and constraints
- API endpoints must include request/response formats
- Should reference existing patterns in codebase
- Must specify error handling and validation
- Include performance targets when applicable
- **MANDATORY**: Must specify testing requirements for each technical requirement:
  - **Unit tests**: MANDATORY for all domain logic, business rules, and data transformations
  - **Integration tests**: MANDATORY for all API endpoints, database operations, and service integrations
  - **E2E tests**: MANDATORY for all user-facing features, complete workflows, and critical user paths
  - **Performance tests**: Required for high-traffic endpoints, batch operations, and data processing
  - **Security tests**: Required for authentication, authorization, and data validation

### 6. Implementation Plan (Required)

**Purpose**: Define how to build the feature

**Content**:
- Phased approach
- Task breakdown per phase
- Dependencies between phases
- Deliverables per phase
- Testing requirements per phase

**Format Example**:

---
## Implementation Plan

### Phase 1: [Phase Name]

**Goal**: [What this phase achieves]

**Scope**: [Which services/components this phase covers]

**Tasks**:
1. [Task 1]
2. [Task 2]
3. [Task 3]

**Testing Requirements**:
- **Unit Tests**: [What must be unit tested]
- **Integration Tests**: [What must be integration tested]
- **E2E Tests**: [What must be E2E tested]

**Deliverables**:
- [Deliverable 1]
- [Deliverable 2]

**Dependencies**:
- [Dependency 1]

---

**Rules**:
- **MANDATORY**: Features spanning multiple services MUST be split into phases
- **MANDATORY**: Features with complex dependencies MUST be split into phases
- **MANDATORY**: Features with complex integration points MUST be split into phases
- Break into logical phases (typically 3-6 phases)
- Each phase should deliver working, testable functionality
- Must identify dependencies between phases
- **MANDATORY**: Each phase must specify testing requirements:
  - **Unit Tests**: MANDATORY for all phases with domain logic, business rules, or data transformations
  - **Integration Tests**: MANDATORY for all phases with API endpoints, database operations, or service integrations
  - **E2E Tests**: MANDATORY for phases that deliver user-facing functionality or complete workflows
  - **Performance Tests**: Required for phases with high-traffic endpoints or batch operations
  - **Security Tests**: Required for phases with authentication, authorization, or data validation
- Must specify deliverables
- Mark completed phases with ✅
- Mark in-progress phases with 🔄
- Mark pending phases with ⏳

**Phase Requirements for Large Features**:
- **Feature Complexity Assessment**: Before writing implementation plan, assess:
  - Number of services involved
  - Number of dependencies
  - Complexity of integration points
  - Infrastructure requirements
- **If any of the following apply, phases are MANDATORY**:
  - Involves 3+ services
  - Requires 3+ major dependencies
  - Has complex integration points (Kafka, multiple databases, external APIs)
  - Requires infrastructure changes
  - Has multiple independent deliverables that can be tested separately
- **Each phase must**:
  - Be independently testable
  - Deliver working functionality (not just partial implementation)
  - Have clear acceptance criteria
  - Be deployable without breaking existing functionality
  - Focus on a single service or logical component when possible
  - Include appropriate test coverage:
    - Backend phases: Unit tests + Integration tests (MANDATORY)
    - Frontend phases: Unit tests + E2E tests (MANDATORY)
    - API phases: Integration tests + E2E tests (MANDATORY)
    - Database phases: Integration tests (MANDATORY)
    - Complete features: All test types (MANDATORY)
- **Phase 0 (Prerequisites)**: If feature requires missing infrastructure or dependencies, create Phase 0:
  - Example: "Phase 0: Kafka Consumer Implementation - PREREQUISITE"
  - Must be completed before Phase 1 can begin
  - Should be clearly marked as blocking dependency

## Content Quality Rules

### Accuracy Requirements

1. **Codebase Verification**: All references to existing code must be verified against actual codebase
   - Use grep/codebase_search to verify claims
   - Check file paths and line numbers
   - Verify API endpoints exist
   - Confirm database schemas match reality

2. **Status Indicators**: Use consistent status indicators
   - ✅ Complete/Implemented
   - 🔄 In Progress
   - ⏳ Pending/Not Started
   - ❌ Missing/Not Implemented
   - ⚠️ Partial/Incomplete/Needs Attention

3. **Dependency Accuracy**: All dependencies must be verified
   - Check if services/features actually exist
   - Verify API endpoints are available
   - Confirm database schemas match
   - Validate integration points

### Completeness Requirements

1. **All Sections**: Every required section must be present
   - Missing sections indicate incomplete analysis
   - Use "N/A" only when truly not applicable
   - Document why sections are skipped

2. **Edge Cases**: Must document edge cases
   - Error scenarios
   - Boundary conditions
   - Failure modes
   - Race conditions

3. **Integration Points**: Must document all integration points
   - Service-to-service communication
   - Database interactions
   - External API calls
   - Event publishing/consuming

### During PRD Writing

1. **Assumption Validation**: Challenge all assumptions
   - "Does this actually exist?"
   - "Is this the correct approach?"
   - "Are there better alternatives?"
   - "What are the risks?"

2. **Gap Identification**: Identify missing pieces
   - What's not implemented?
   - What needs to be built first?
   - What are the blockers?
   - What are the prerequisites?

### After PRD Writing

**Review Checklist**: Verify completeness
- [ ] All required sections present
- [ ] All code references verified
- [ ] All dependencies identified
- [ ] All acceptance criteria defined
- [ ] Implementation plan is realistic

## Common Pitfalls to Avoid

### 1. Assumption Errors

**Problem**: Assuming features/services exist without verification

**Example**:
```markdown
❌ Bad: "Budget service processes Kafka events" (without checking)
✅ Good: "Budget service processes Kafka events (verified in budget-service/src/domain/transaction_events.rs)"
```

**Solution**: Always verify against codebase before documenting

### 2. Missing Prerequisites

**Problem**: Not identifying what must be built first

**Example**:
```markdown
❌ Bad: "Settlement feature uses real-time spending data"
✅ Good: "Settlement feature requires Kafka consumer implementation (Phase 0) before Phase 1 can begin"
```

**Solution**: Explicitly list all prerequisites and dependencies

### 3. Vague Requirements

**Problem**: Requirements that can't be tested or measured

**Example**:
```markdown
❌ Bad: "System should be fast"
✅ Good: "API response time < 200ms for 95% of requests"
```

**Solution**: Make all requirements specific and measurable

### 4. Incomplete Integration Documentation

**Problem**: Not documenting how services interact

**Example**:
```markdown
❌ Bad: "Frontend calls backend API"
✅ Good: "Frontend calls POST /v1/users/{user_id}/transactions/categorize via BFF service, which forwards to transaction-service with JWT authentication"
```

**Solution**: Document complete integration flow with all steps

### 5. Missing Error Handling

**Problem**: Not documenting error scenarios

**Example**:
```markdown
❌ Bad: "API returns error on failure"
✅ Good: "API returns 400 Bad Request with error code 'VALIDATION_ERROR' when category name is invalid, 403 Forbidden when user doesn't own budget, 500 Internal Server Error with error details when database operation fails"
```

**Solution**: Document all error scenarios with specific error codes and messages

### 6. Not Splitting Large Features into Phases

**Problem**: Attempting to implement large features as single monolithic phases

**Example**:
```markdown
❌ Bad: "Phase 1: Implement Budget Settlement Feature (all services, all components)"
✅ Good: "Phase 0: Kafka Consumer Implementation - PREREQUISITE
         Phase 1: Backend Infrastructure (database, domain models, repositories)
         Phase 2: API Endpoints (REST API, BFF integration)
         Phase 3: Insights Generation (business logic, algorithms)
         Phase 4: Frontend Components (UI, state management)"
```

**Solution**:
- Assess feature complexity before writing implementation plan
- Split features spanning multiple services into phases
- Split features with complex dependencies into phases
- Each phase must deliver independently testable functionality
- Create Phase 0 for prerequisites if needed
- Ensure each phase is deployable without breaking existing functionality
- Focus each phase on a single service or logical component when possible

## Status Tracking

### Implementation Status Section

Every PRD should include an implementation status section that tracks progress:

---
## Implementation Status

### Overall Completion: [X]%

### ✅ Completed
- [Feature/Component 1]
- [Feature/Component 2]

### 🔄 In Progress
- [Feature/Component 1] - [% complete]

### ⏳ Pending
- [Feature/Component 1]
- [Feature/Component 2]

---

**Rules**:
- Update status as implementation progresses
- Include completion percentages
- Mark completed items with ✅
- Mark in-progress items with 🔄
- Mark pending items with ⏳
- Include last updated date

## Review Process

### PRD Self-Review Checklist

Before marking PRD as "Under Implementation", verify:

- [ ] All required sections present
- [ ] All code references verified against codebase
- [ ] All dependencies identified and verified
- [ ] All acceptance criteria defined and testable
- [ ] **Multi-service features split into phases** (MANDATORY)
- [ ] **Features with complex dependencies split into phases** (MANDATORY)
- [ ] **Each phase delivers independently testable functionality** (MANDATORY)
- [ ] **Each phase specifies testing requirements** (MANDATORY)
- [ ] **Technical requirements include testing requirements** (MANDATORY)
- [ ] **Unit tests specified for domain logic/business rules** (MANDATORY)
- [ ] **Integration tests specified for API endpoints/database operations** (MANDATORY)
- [ ] **E2E tests specified for user-facing features** (MANDATORY)
- [ ] Implementation plan is realistic
- [ ] API specifications match existing patterns
- [ ] Database schemas include indexes and constraints
- [ ] Security considerations addressed
- [ ] Testing strategy comprehensive
- [ ] No assumptions left unverified
- [ ] All prerequisites clearly identified
- [ ] Phase 0 (prerequisites) created if needed

## Document Maintenance

### Archive Rules

- Move completed PRDs to `implemented/` directory
- Keep version history
- Reference in system documentation
- Use for future feature planning

### Future Enhancement Tracking

**Do not include "Future Enhancements" section in PRDs**. Instead:
- Track future enhancements in your issue tracker (GitHub Issues, Jira, etc.)
- Keep PRDs focused on current implementation scope
- Reference the implemented feature in future PRDs when building enhancements

---

**Document Version**: 2.0
**Last Updated**: January 2025
**Target**: Solo Developer Workflows
