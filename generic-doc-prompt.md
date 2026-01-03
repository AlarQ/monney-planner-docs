# Generic Repository Documentation Generator Prompt (mdBook)

You are an expert technical documentation writer. Generate **mdBook-compatible documentation** that stays in sync with source code using include directives.

## Critical Rules: Avoid Documentation Drift

**NEVER duplicate source code in documentation.** Instead:

1. **Dependencies** → Use `{{#include ../Cargo.toml:dependencies}}`
2. **Configuration** → Use `{{#include ../run.sh:env_vars}}`
3. **Code examples** → Use `{{#include ../src/path.rs:function_name}}`
4. **Version numbers** → Include from source, never hardcode in prose

**Document CONCEPTS (why), not IMPLEMENTATION (what).** Readers can see implementation in source files.

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
git-repository-url = "https://github.com/your-org/your-repo"
edit-url-template = "https://github.com/your-org/your-repo/edit/main/docs/src/{path}"
additional-js = ["mermaid.min.js", "mermaid-init.js"]
```

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
\`\`\`

### run.sh
\`\`\`bash
# ANCHOR: env_vars
export SERVICE__HOST=...
...
# ANCHOR_END: env_vars
\`\`\`

### src/main.rs (example)
\`\`\`rust
// ANCHOR: main_function
fn main() {
    ...
}
// ANCHOR_END: main_function
\`\`\`
```

## Documentation Sections

### 1. Overview (overview.md)
**Purpose**: High-level summary

**Content**:
- Service name and purpose (1-2 sentences)
- Key responsibilities in the system
- Link to architecture diagram

**DO NOT include**: Version numbers, dependency lists

### 2. Architecture (architecture.md)
**Purpose**: System design explanation

**Content**:
- Architectural pattern description (DDD, Clean Architecture, etc.)
- Mermaid diagram of component relationships
- Data flow explanation
- Layer responsibilities

**Include from source**:
```markdown
Directory structure overview:

\`\`\`
{{#include ../../src/lib.rs:module_structure}}
\`\`\`
```

### 3. Tech Stack (tech-stack.md)
**Purpose**: Technology overview

**Content**:
- Programming language and framework (conceptual)
- Key technology choices and WHY they were made
- Links to official documentation

**Include from source** (never copy):
```markdown
## Dependencies

Current production dependencies:

\`\`\`toml
{{#include ../../Cargo.toml:dependencies}}
\`\`\`
```

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
- Base URL pattern
- Authentication requirements
- Link to OpenAPI spec

**Note**: Reference `/swagger-ui` or `/api-docs/openapi.json` instead of duplicating endpoint definitions.

```markdown
## API Documentation

Interactive API documentation is available at runtime:
- Swagger UI: `http://localhost:PORT/swagger-ui`
- OpenAPI spec: `http://localhost:PORT/api-docs/openapi.json`

See [routes.rs](../src/api/routes.rs) for endpoint definitions.
```

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

Before finalizing documentation:

- [ ] No hardcoded code snippets (all use `{{#include}}`)
- [ ] No version numbers in prose
- [ ] All `{{#include}}` paths are correct relative to docs/src/
- [ ] Anchor names match between docs and source files
- [ ] Mermaid JS files present (`mermaid.min.js`, `mermaid-init.js`)
- [ ] Mermaid diagrams render correctly in browser
- [ ] Links to source files use correct paths
- [ ] SUMMARY.md includes all sections
- [ ] book.toml has correct repository URL and `additional-js` for Mermaid

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
