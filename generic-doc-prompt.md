# Generic Repository Documentation Generator Prompt (mdBook)

You are an expert technical documentation writer. Generate **mdBook-compatible documentation** that stays in sync with source code using include directives.

## Critical Rules: Avoid Documentation Drift

**NEVER duplicate source code in documentation.** Instead:

1. **Dependencies** → Use `{{#include ../Cargo.toml:dependencies}}`
2. **Configuration** → Use `{{#include ../run.sh:env_vars}}`
3. **Code examples** → Use `{{#include ../src/path.rs:function_name}}`
4. **Version numbers** → Include from source, never hardcode in prose

**Document CONCEPTS (why), not IMPLEMENTATION (what).** Readers can see implementation in source files.

## CRITICAL: Never Use File Links When Includes Are Possible

**WRONG** - Do NOT write documentation like this:
```markdown
See [src/config.rs](../../src/config.rs) for configuration.
```

**RIGHT** - Always use includes:
```markdown
\`\`\`rust
{{#include ../../src/config.rs:config_struct}}
\`\`\`
```

If an anchor doesn't exist in the source file, **LIST IT** in the "Required Anchors" output section. Never fall back to file links as a substitute for includes.

**The ONLY acceptable use of file links** is for navigation to source files that contain complex logic not suitable for inclusion (e.g., "See [routes.rs](../../src/api/routes.rs) for the full routing logic").

## Output Structure

Generate an mdBook documentation structure:

```
docs/
├── book.toml           # mdBook configuration
├── mermaid.min.js      # Mermaid JS library (download separately)
├── mermaid-init.js     # Mermaid initialization script
└── src/
    ├── SUMMARY.md      # Table of contents
    ├── overview.md
    ├── architecture.md
    ├── tech-stack.md
    ├── getting-started.md
    ├── development-workflow.md
    ├── api-endpoints.md        # If applicable
    ├── domain-context.md
    ├── configuration.md
    ├── security.md
    ├── testing.md
    ├── deployment.md
    ├── code-patterns.md
    ├── dependencies.md
    └── troubleshooting.md
```

## Required Outputs

For each repository, generate:

### 1. book.toml
```toml
[book]
title = "[Service Name] Documentation"
authors = ["Money Planner Team"]
language = "en"

[build]
build-dir = "book"

[output.html]
default-theme = "rust"
# IMPORTANT: Replace with actual repository URL - do NOT leave as placeholder
git-repository-url = "https://github.com/[ACTUAL-ORG]/[ACTUAL-REPO]"
edit-url-template = "https://github.com/[ACTUAL-ORG]/[ACTUAL-REPO]/edit/main/docs/src/{path}"
additional-js = ["mermaid.min.js", "mermaid-init.js"]
additional-css = ["theme/fonts.css"]
```

**CRITICAL**: Replace `[ACTUAL-ORG]` and `[ACTUAL-REPO]` with real values. Never leave placeholders in generated docs.

### 1b. Mermaid Support Files

After generating docs, run:
```bash
# Download mermaid.min.js (or copy from another service)
curl -o docs/mermaid.min.js https://cdn.jsdelivr.net/npm/mermaid@10/dist/mermaid.min.js

# Create mermaid-init.js
cat > docs/mermaid-init.js << 'EOF'
(() => {
    const darkThemes = ['ayu', 'navy', 'coal'];
    const classList = document.getElementsByTagName('html')[0].classList;
    let lastThemeWasLight = true;
    for (const cssClass of classList) {
        if (darkThemes.includes(cssClass)) {
            lastThemeWasLight = false;
            break;
        }
    }
    const theme = lastThemeWasLight ? 'default' : 'dark';
    const mermaidBlocks = document.querySelectorAll('pre code.language-mermaid');
    mermaidBlocks.forEach((block) => {
        const pre = block.parentElement;
        const div = document.createElement('div');
        div.className = 'mermaid';
        div.textContent = block.textContent;
        pre.parentElement.replaceChild(div, pre);
    });
    mermaid.initialize({ startOnLoad: true, theme });
})();
EOF
```

### 2. SUMMARY.md
```markdown
# Summary

- [Overview](./overview.md)
- [Architecture](./architecture.md)
- [Tech Stack](./tech-stack.md)
- [Getting Started](./getting-started.md)
- [Development Workflow](./development-workflow.md)
- [API Endpoints](./api-endpoints.md)
- [Domain Context](./domain-context.md)
- [Configuration](./configuration.md)
- [Security](./security.md)
- [Testing](./testing.md)
- [Deployment](./deployment.md)
- [Code Patterns](./code-patterns.md)
- [Dependencies](./dependencies.md)
- [Troubleshooting](./troubleshooting.md)
```

### 3. Anchor Instructions

Also output a list of **anchors to add to source files**:

```
## Required Anchors

Add these comments to source files for mdBook includes:

### Cargo.toml
\`\`\`toml
# ANCHOR: dependencies
[dependencies]
...
# ANCHOR_END: dependencies

# ANCHOR: dev_dependencies
[dev-dependencies]
...
# ANCHOR_END: dev_dependencies

# ANCHOR: features (if applicable)
[features]
...
# ANCHOR_END: features
\`\`\`

### run.sh
\`\`\`bash
# ANCHOR: env_vars
export SERVICE__HOST=...
...
# ANCHOR_END: env_vars
\`\`\`

### src/config.rs
\`\`\`rust
// ANCHOR: config_struct
pub struct AppConfig {
    ...
}
// ANCHOR_END: config_struct

// ANCHOR: app_state (if applicable)
pub struct AppState {
    ...
}
// ANCHOR_END: app_state
\`\`\`

### src/domain/*/entities.rs (for each entity)
\`\`\`rust
// ANCHOR: user_struct (or appropriate entity name)
pub struct User {
    ...
}
// ANCHOR_END: user_struct
\`\`\`

### tests/common.rs or tests/common/mod.rs
\`\`\`rust
// ANCHOR: test_setup
pub async fn setup_test_app() -> ... {
    ...
}
// ANCHOR_END: test_setup
\`\`\`

### rustfmt.toml
\`\`\`toml
# ANCHOR: formatting_config
edition = "2021"
...
# ANCHOR_END: formatting_config
\`\`\`
```

**IMPORTANT**: When generating documentation, list ALL anchors that need to be added to source files. The documentation cannot work without these anchors being present.

## Section Content Rules

1. **No content duplication**: Each piece of information belongs in ONE section only
2. **Troubleshooting is for issues ONLY**: Never put "Common Issues" in Getting Started
3. **Getting Started is for setup only**: Just prerequisites, quick start, verification steps
4. **Cross-references are OK**: Use `See [Troubleshooting](./troubleshooting.md)` to point to other sections

## Minimum Section Requirements

Each section MUST include these elements (non-negotiable):

| Section | Required Elements |
|---------|-------------------|
| Overview | Service position mermaid diagram showing relationships to other services |
| Architecture | At least one sequence diagram AND layer responsibility table |
| Tech Stack | BOTH `dependencies` AND `dev_dependencies` includes |
| Configuration | Config struct include via anchor (`config_struct`) |
| Domain Context | At least one entity struct include AND ER diagram |
| Testing | Test setup include AND test directory structure |
| Deployment | Dockerfile include (full file) |
| Code Patterns | rustfmt.toml include |
| Dependencies | BOTH production AND dev dependencies includes |

## Formatting Style Guidelines

Use consistent formatting across all generated documentation:

### API Endpoint Tables

**Always use this format for endpoint documentation:**

```markdown
### [Category Name] (e.g., Health Checks, User Management)

| Method | Path | Description | Auth Required |
|--------|------|-------------|---------------|
| GET | `/health` | Liveness check | No |
| POST | `/v1/users/` | Create new user | No |
| GET | `/v1/users/{id}` | Get user by ID | Yes |
| DELETE | `/v1/users/{id}` | Delete user | Yes (Admin) |
```

**Rules:**
- Group endpoints by functional category (Health Checks, User Management, Authentication, etc.)
- Always include the `Auth Required` column
- Use backticks for paths: `/v1/users/{id}`
- Note special auth requirements in parentheses: `Yes (Admin)`

### Technology/Dependency Tables

**Use this format for technology choices:**

```markdown
| Technology | Purpose | Why Chosen |
|------------|---------|------------|
| **Axum** | Web framework | Modern, type-safe, Tower ecosystem |
| **SQLx** | Database driver | Compile-time query checking |
```

**Rules:**
- Bold the technology name
- Keep "Purpose" short (3-5 words)
- "Why Chosen" explains the rationale

### Configuration Tables

**Use this format for environment variables:**

```markdown
| Variable | Required | Description | Default |
|----------|----------|-------------|---------|
| `SERVICE__HOST` | Yes | Server bind address | `0.0.0.0` |
| `SERVICE__PORT` | Yes | Server port | `8080` |
| `SERVICE__JWT_SECRET` | Yes | JWT signing secret | - |
```

**Rules:**
- Use backticks for variable names
- Default of `-` means no default (required)
- Group related variables together

### Lists vs Tables

- **Use tables** for structured data with 3+ attributes per item
- **Use bullet lists** for simple enumerations (prerequisites, features, etc.)
- **Never mix formats** for similar content within the same document

## Documentation Sections

### 1. Overview (overview.md)
**Purpose**: High-level summary

**Content**:
- Service name and purpose (1-2 sentences)
- Key responsibilities in the system
- Related services table with ports
- **REQUIRED**: Service position mermaid diagram

**Example service position diagram**:
```markdown
\`\`\`mermaid
flowchart TB
    subgraph Frontend
        FE[React App :3000]
    end

    subgraph Gateway[BFF Service :8082]
        GW[API Gateway]
    end

    subgraph ThisService[This Service :PORT]
        Component1[Component 1]
        Component2[Component 2]
    end

    subgraph OtherServices[Other Services]
        OS1[Other Service 1]
        OS2[Other Service 2]
    end

    FE --> GW
    GW --> ThisService
    GW --> OtherServices
\`\`\`
```

**DO NOT include**: Version numbers, dependency lists

### 2. Architecture (architecture.md)
**Purpose**: System design explanation

**Content**:
- Architectural pattern description (DDD, Clean Architecture, etc.)
- **REQUIRED**: Layer responsibilities table
- **REQUIRED**: At least one sequence diagram (data flow or authentication flow)
- Mermaid diagram of component relationships

**Directory Structure**:

For directory structure, use an **inline tree** that you manually create based on the actual file structure. This is an exception to the include rule because:
1. Directory trees are structural documentation, not code
2. Including via anchor would require maintaining the tree in a source file

```markdown
## Directory Structure

\`\`\`
src/
├── api/                    # HTTP Layer
│   ├── mod.rs             # Router setup
│   └── handlers/          # Request handlers
├── domain/                 # Domain Layer
│   ├── entities/          # Domain entities
│   └── interfaces/        # Repository traits
├── repositories/           # Infrastructure Layer
├── config.rs              # Configuration
└── main.rs                # Entry point
\`\`\`
```

**Note**: Update this tree when adding new modules. Keep it high-level (2-3 levels deep).

### 3. Tech Stack (tech-stack.md)
**Purpose**: Technology overview

**Content**:
- Programming language and framework (conceptual)
- Key technology choices and WHY they were made (use tables with "Why Chosen" column)
- Links to official documentation

**REQUIRED includes** (never copy):
```markdown
## Production Dependencies

\`\`\`toml
{{#include ../../Cargo.toml:dependencies}}
\`\`\`

## Development Dependencies

\`\`\`toml
{{#include ../../Cargo.toml:dev_dependencies}}
\`\`\`
```

**Feature Flags** (if applicable):

If the Cargo.toml defines feature flags, include them:

```markdown
## Feature Flags

\`\`\`toml
{{#include ../../Cargo.toml:features}}
\`\`\`
```

Add to Required Anchors if features exist in Cargo.toml.

### 4. Getting Started (getting-started.md)
**Purpose**: Quick setup guide

**Content**:
- Prerequisites list
- Step-by-step setup (conceptual)
- How to verify success

**Include from source**:
```markdown
## Environment Setup

Set the following environment variables (see run.sh for current values):

\`\`\`bash
{{#include ../../run.sh:env_vars}}
\`\`\`
```

### 5. Development Workflow (development-workflow.md)
**Purpose**: Daily development commands

**Content**:
- Common commands overview
- Link to Makefile or scripts

**Include from source**:
```markdown
## Available Commands

See the Makefile for all available targets:

\`\`\`makefile
{{#include ../../Makefile:targets}}
\`\`\`
```

### 6. API Endpoints (api-endpoints.md)
**Purpose**: API reference (if applicable)

**Content**:
- Interactive documentation links (Swagger UI, OpenAPI spec)
- Authentication requirements overview
- Endpoint summary tables (grouped by category)

**REQUIRED format** (see [Formatting Style Guidelines](#formatting-style-guidelines)):

```markdown
## Interactive Documentation

- **Swagger UI**: http://localhost:PORT/swagger-ui
- **OpenAPI Spec**: http://localhost:PORT/api-doc/openapi.json

## Authentication

Most endpoints require a JWT token in the Authorization header:
\`\`\`
Authorization: Bearer <jwt_token>
\`\`\`

## Endpoint Summary

### Health Checks

| Method | Path | Description |
|--------|------|-------------|
| GET | `/health` | Liveness check |
| GET | `/ready` | Readiness check (DB connectivity) |

### User Management

| Method | Path | Description | Auth Required |
|--------|------|-------------|---------------|
| POST | `/v1/users/` | Create new user | No |
| GET | `/v1/users/{id}` | Get user by ID | Yes |
| DELETE | `/v1/users/{id}` | Delete user | Yes |

### Authentication

| Method | Path | Description | Auth Required |
|--------|------|-------------|---------------|
| POST | `/v1/auth/login` | Login and get JWT | No |
| GET | `/v1/auth/session` | Validate session | Yes |
| DELETE | `/v1/auth/logout` | Logout user | Yes |
```

**Note**: Keep endpoint tables in sync with actual routes. Reference source files for complex logic.

### 7. Domain Context (domain-context.md)
**Purpose**: Business domain explanation

**Content**:
- Core domain concepts (ubiquitous language)
- Entity relationships (Mermaid diagram)
- Business rules summary

**Include from source**:
```markdown
## Core Entities

\`\`\`rust
{{#include ../../src/domain/entities/user.rs:user_struct}}
\`\`\`
```

### 8. Configuration (configuration.md)
**Purpose**: Configuration reference

**Content**:
- Configuration approach (env vars, files, etc.)
- Link to example config

**Include from source**:
```markdown
## Environment Variables

\`\`\`bash
{{#include ../../run.sh:env_vars}}
\`\`\`

## Configuration Schema

\`\`\`rust
{{#include ../../src/config.rs:config_struct}}
\`\`\`
```

### 9. Security (security.md)
**Purpose**: Security patterns overview

**Content**:
- Authentication mechanism (conceptual)
- Authorization approach
- Security best practices followed

**DO NOT include**: Actual secrets, tokens, or sensitive configuration

### 10. Testing (testing.md)
**Purpose**: Testing strategy

**Content**:
- Test types and their purpose
- How to run tests
- Test organization

**Include from source**:
```markdown
## Test Configuration

\`\`\`rust
{{#include ../../tests/common/mod.rs:test_setup}}
\`\`\`
```

### 11. Deployment (deployment.md)
**Purpose**: Deployment guide

**Content**:
- Deployment targets
- Build commands
- Health check endpoints

**Include from source**:
```markdown
## Dockerfile

\`\`\`dockerfile
{{#include ../../Dockerfile}}
\`\`\`
```

### 12. Code Patterns (code-patterns.md)
**Purpose**: Coding conventions

**Content**:
- Style guidelines (link to rustfmt.toml, .eslintrc)
- Common patterns used
- Anti-patterns to avoid

**Include from source**:
```markdown
## Formatting Configuration

\`\`\`toml
{{#include ../../rustfmt.toml}}
\`\`\`
```

### 13. Dependencies (dependencies.md)
**Purpose**: Dependency rationale

**Content**:
- WHY each major dependency was chosen (not versions)
- Dependency categories

**Include from source**:
```markdown
## Production Dependencies

\`\`\`toml
{{#include ../../Cargo.toml:dependencies}}
\`\`\`

## Development Dependencies

\`\`\`toml
{{#include ../../Cargo.toml:dev_dependencies}}
\`\`\`
```

### 14. Troubleshooting (troubleshooting.md)
**Purpose**: Common issues and solutions

**Content**:
- Common error patterns
- Debugging approaches
- FAQ

## Minimal Documentation Philosophy

Each documentation file should:

1. **Explain WHY, not WHAT** - Readers can see WHAT in source code
2. **Use `{{#include}}` for code** - Never copy-paste source code
3. **Link to source files** - For anything beyond includes
4. **Use Mermaid for diagrams** - Architecture, data flow, entity relationships
5. **No hardcoded versions** - All versions come from Cargo.toml via includes
6. **Keep it short** - If a section is mostly code, just include the code

## mdBook Include Syntax

```markdown
# Include entire file
{{#include ../path/to/file}}

# Include specific lines (1-indexed)
{{#include ../path/to/file:5:10}}

# Include anchored section
{{#include ../path/to/file:anchor_name}}

# Include with hidden lines (for rust)
{{#include ../path/to/file:anchor_name}}
```

## Anchor Syntax by Language

**Rust:**
```rust
// ANCHOR: name
code here
// ANCHOR_END: name
```

**TOML:**
```toml
# ANCHOR: name
content here
# ANCHOR_END: name
```

**Bash:**
```bash
# ANCHOR: name
content here
# ANCHOR_END: name
```

**TypeScript/JavaScript:**
```typescript
// ANCHOR: name
code here
// ANCHOR_END: name
```

## GitHub Pages Deployment

Include this GitHub Actions workflow in your output:

```yaml
# .github/workflows/docs.yml
name: Deploy Documentation

on:
  push:
    branches: [main]
    paths:
      - 'docs/**'
      - 'Cargo.toml'
      - 'run.sh'

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pages: write
      id-token: write
    steps:
      - uses: actions/checkout@v4

      - name: Install mdBook
        run: cargo install mdbook

      - name: Build docs
        run: mdbook build docs/

      - name: Setup Pages
        uses: actions/configure-pages@v4

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: docs/book

      - name: Deploy to GitHub Pages
        uses: actions/deploy-pages@v4
```

## Quality Checklist

Before finalizing documentation, verify ALL items:

### Include Directive Compliance
- [ ] ALL code blocks use `{{#include}}` - no plain text code snippets
- [ ] No "See [file.rs](...)" links used as substitute for includes
- [ ] Both `dependencies` AND `dev_dependencies` are included in tech-stack.md
- [ ] Both `dependencies` AND `dev_dependencies` are included in dependencies.md
- [ ] Config struct is included via anchor in configuration.md
- [ ] Dockerfile is included in deployment.md
- [ ] rustfmt.toml is included in code-patterns.md
- [ ] Test setup is included via anchor in testing.md

### Required Diagrams
- [ ] Overview has service position mermaid diagram
- [ ] Architecture has at least one sequence diagram
- [ ] Domain Context has entity ER diagram (if applicable)

### Content Organization
- [ ] No duplicate content between sections
- [ ] "Common Issues" content is ONLY in troubleshooting.md
- [ ] Getting Started contains ONLY setup steps (no troubleshooting)

### Configuration Files
- [ ] book.toml has ACTUAL repository URL (no placeholders like `your-org`)
- [ ] book.toml has `additional-js` for Mermaid
- [ ] book.toml has `additional-css` for fonts
- [ ] SUMMARY.md includes all sections in correct order

### Technical
- [ ] All `{{#include}}` paths are correct relative to docs/src/
- [ ] Anchor names match between docs and source files
- [ ] Mermaid JS files present (`mermaid.min.js`, `mermaid-init.js`)
- [ ] Mermaid diagrams render correctly in browser
- [ ] No version numbers hardcoded in prose

### Formatting Consistency
- [ ] API endpoint tables use standard format: `Method | Path | Description | Auth Required`
- [ ] Endpoints grouped by category (Health Checks, User Management, Authentication, etc.)
- [ ] Technology tables use format: `Technology | Purpose | Why Chosen`
- [ ] Configuration tables use format: `Variable | Required | Description | Default`
- [ ] No mixing of bullet lists and tables for similar content

## Usage Instructions

1. Run this prompt against a repository
2. AI generates:
   - `docs/book.toml` (with Mermaid JS configuration)
   - `docs/src/SUMMARY.md`
   - `docs/src/*.md` files with `{{#include}}` directives
   - List of anchors to add to source files
3. Add the anchors to source files
4. **Set up Mermaid support** (required for diagrams):
   ```bash
   # Download mermaid.min.js
   curl -o docs/mermaid.min.js https://cdn.jsdelivr.net/npm/mermaid@10/dist/mermaid.min.js

   # Create mermaid-init.js (see section 1b above for content)
   ```
   Or copy `mermaid.min.js` and `mermaid-init.js` from an existing service.
5. Run `mdbook build docs/` to verify
6. Run `mdbook serve docs/` and check Mermaid diagrams render in browser
7. Set up GitHub Pages deployment

## Important: Mermaid Rendering

mdBook does not render Mermaid diagrams by default. The `mermaid-init.js` script:
1. Finds all `<pre><code class="language-mermaid">` blocks
2. Converts them to `<div class="mermaid">` elements
3. Initializes Mermaid.js to render the diagrams

**Do NOT use the `mdbook-mermaid` preprocessor** - it has compatibility issues with `{{#include}}` directives.

This approach ensures documentation stays in sync with source code automatically.
