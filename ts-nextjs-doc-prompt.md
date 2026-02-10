# TypeScript/Next.js Repository Documentation Generator Prompt (mdBook)

You are an expert technical documentation writer. Generate **mdBook-compatible documentation** for TypeScript/Next.js projects that stays in sync with source code using include directives.

## Critical Rules: Avoid Documentation Drift

**NEVER duplicate source code in documentation.** Instead:

1. **Dependencies** → Use `{{#include ../package.json:22:80}}` (line-based — JSON has no anchors)
2. **Configuration** → Use `{{#include ../.env.example:env_vars}}`
3. **Code examples** → Use `{{#include ../src/path.ts:function_name}}`
4. **Version numbers** → Include from source, never hardcode in prose

**Document CONCEPTS (why), not IMPLEMENTATION (what).** Readers can see implementation in source files.

## CRITICAL: Never Use File Links When Includes Are Possible

**WRONG** - Do NOT write documentation like this:
```markdown
See [src/services/mindmap.ts](../../src/services/mindmap.ts) for service logic.
```

**RIGHT** - Always use includes:
```markdown
\`\`\`typescript
{{#include ../../src/services/mindmap.ts:get_mindmap_service}}
\`\`\`
```

If an anchor doesn't exist in the source file, **LIST IT** in the "Required Anchors" output section. Never fall back to file links as a substitute for includes.

**The ONLY acceptable use of file links** is for navigation to source files that contain complex logic not suitable for inclusion (e.g., "See [route.ts](../../src/app/api/mindmap/[id]/route.ts) for the full route handler").

### JSON File Limitation

JSON files (`package.json`, `tsconfig.json`, `biome.json`) **cannot have anchor comments**. For these files:
- Use **line-based includes**: `{{#include ../../package.json:22:80}}`
- Document the line ranges in the "Required Anchors" output so maintainers know to update them when the file changes
- Alternatively, extract key configuration to `.ts` files with proper anchors

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
    ├── next-app-router.md      # Next.js App Router patterns
    ├── tech-stack.md
    ├── getting-started.md
    ├── development-workflow.md
    ├── api-endpoints.md        # If applicable
    ├── services-layer.md       # Business logic layer
    ├── domain-context.md
    ├── types-schema.md         # TypeScript types & DB schema
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
- [Next.js App Router](./next-app-router.md)
- [Tech Stack](./tech-stack.md)
- [Getting Started](./getting-started.md)
- [Development Workflow](./development-workflow.md)
- [API Endpoints](./api-endpoints.md)
- [Services Layer](./services-layer.md)
- [Domain Context](./domain-context.md)
- [Types & Schema](./types-schema.md)
- [Configuration](./configuration.md)
- [Security](./security.md)
- [Testing](./testing.md)
- [Deployment](./deployment.md)
- [Code Patterns](./code-patterns.md)
- [Dependencies](./dependencies.md)
- [Troubleshooting](./troubleshooting.md)
```

### 3. Anchor Instructions

Also output a list of **anchors to add to source files** and **line ranges for JSON files**:

```
## Required Anchors

Add these comments to source files for mdBook includes:

### .env.example
```bash
# ANCHOR: database_config
DATABASE_URL=postgresql://user:password@localhost:5432/dbname
# ANCHOR_END: database_config

# ANCHOR: auth_config
AUTH_SECRET=your-secret-here
AUTH_URL=http://localhost:3000
# ANCHOR_END: auth_config

# ANCHOR: ai_config
OPENAI_API_KEY=sk-...
DEEPSEEK_API_KEY=...
# ANCHOR_END: ai_config

# ANCHOR: storage_config
AWS_REGION=us-east-1
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
S3_BUCKET_NAME=...
# ANCHOR_END: storage_config
```

### package.json (LINE-BASED — no anchors possible)
Document the line ranges used in includes:
- Dependencies: `{{#include ../../package.json:LINE_START:LINE_END}}`
- DevDependencies: `{{#include ../../package.json:LINE_START:LINE_END}}`
- Scripts: `{{#include ../../package.json:LINE_START:LINE_END}}`

**Update these line numbers whenever package.json structure changes.**

### tsconfig.json (LINE-BASED — no anchors possible)
- Compiler options: `{{#include ../../tsconfig.json:LINE_START:LINE_END}}`

### biome.json (LINE-BASED — no anchors possible)
- Linter rules: `{{#include ../../biome.json:LINE_START:LINE_END}}`
- Formatter config: `{{#include ../../biome.json:LINE_START:LINE_END}}`

### next.config.ts
```typescript
// ANCHOR: next_config
const nextConfig: NextConfig = {
    ...
};
// ANCHOR_END: next_config
```

### src/db/schema.ts (for each table)
```typescript
// ANCHOR: users_table
export const users = pgTable("users", {
    ...
});
// ANCHOR_END: users_table

// ANCHOR: mind_maps_table
export const mind_maps = pgTable("mind_maps", {
    ...
});
// ANCHOR_END: mind_maps_table

// ANCHOR: nodes_table
export const nodes = pgTable("nodes", {
    ...
});
// ANCHOR_END: nodes_table

// ANCHOR: schema_relations
export const usersRelations = relations(users, ({ many }) => ({
    ...
}));
// ANCHOR_END: schema_relations
```

### src/types/*.ts (for type definitions)
```typescript
// ANCHOR: node_data_type
export interface NodeData extends Node {
    healthStatus: HealthStatus;
    childCount?: number;
}
// ANCHOR_END: node_data_type

// ANCHOR: mindmap_with_nodes_type
export interface MindMapWithNodes extends MindMap {
    nodes: NodeData[];
    connections: NodeConnection[];
}
// ANCHOR_END: mindmap_with_nodes_type
```

### src/services/*.ts (for service functions)
```typescript
// ANCHOR: get_nodes_service
export async function getNodes(
    mindMapId: string,
    filters?: NodeFilters
): Promise<NodeData[]> {
    ...
}
// ANCHOR_END: get_nodes_service

// ANCHOR: create_node_service
export async function createNode(
    data: Omit<NewNode, "mind_map_id">,
    mindMapId: string
): Promise<NodeData> {
    ...
}
// ANCHOR_END: create_node_service
```

### src/app/api/**/route.ts (for API handlers)
```typescript
// ANCHOR: get_handler
export async function GET(
    req: Request,
    { params }: { params: Promise<{ id: string }> }
) {
    ...
}
// ANCHOR_END: get_handler

// ANCHOR: post_handler
export async function POST(
    req: Request,
    { params }: { params: Promise<{ id: string }> }
) {
    ...
}
// ANCHOR_END: post_handler
```

### src/lib/resp.ts (response utilities)
```typescript
// ANCHOR: response_helpers
export function respData<T>(data: T, init?: ResponseInit): Response { ... }
export function respBadRequest(message: string): Response { ... }
export function respNoAuth(message?: string): Response { ... }
export function respServerErr(message: string): Response { ... }
// ANCHOR_END: response_helpers
```

### vitest.config.ts (or vitest.config.mts)
```typescript
// ANCHOR: test_config
export default defineConfig({
    test: {
        environment: "node",
        globals: true,
        include: ["tests/**/*.test.ts"],
    },
});
// ANCHOR_END: test_config
```

### Dockerfile
No anchors needed — include entire file:
```markdown
\`\`\`dockerfile
{{#include ../../Dockerfile}}
\`\`\`
```
```

**IMPORTANT**: When generating documentation, list ALL anchors that need to be added to source files AND all line ranges for JSON files. The documentation cannot work without these being present and correct.

## Section Content Rules

1. **No content duplication**: Each piece of information belongs in ONE section only
2. **Troubleshooting is for issues ONLY**: Never put "Common Issues" in Getting Started
3. **Getting Started is for setup only**: Just prerequisites, quick start, verification steps
4. **Cross-references are OK**: Use `See [Troubleshooting](./troubleshooting.md)` to point to other sections

## Minimum Section Requirements

Each section MUST include these elements (non-negotiable):

| Section | Required Elements |
|---------|-------------------|
| Overview | Service position mermaid diagram showing relationships to frontend/backend/data |
| Architecture | At least one sequence diagram AND layer responsibility table |
| Next.js App Router | Directory structure diagram AND Server vs Client Components explanation |
| Tech Stack | BOTH `dependencies` AND `devDependencies` includes from package.json |
| Configuration | `.env.example` include AND/OR `next.config.ts` include via anchor |
| Services Layer | At least TWO service function includes showing the pattern |
| Domain Context | At least one type/interface include AND ER diagram from Drizzle schema |
| Types & Schema | Drizzle schema table includes AND TypeScript type definition includes |
| Testing | `vitest.config.ts` include via anchor AND test directory structure |
| Deployment | Dockerfile include (full file) OR Vercel deployment instructions |
| Code Patterns | `biome.json` include (line-based) AND `tsconfig.json` patterns |
| Dependencies | BOTH production AND dev dependencies includes from package.json |

## Formatting Style Guidelines

Use consistent formatting across all generated documentation:

### API Endpoint Tables

**Always use this format for endpoint documentation:**

```markdown
### [Category Name] (e.g., Health Checks, Mind Maps)

| Method | Path | Route File | Description | Auth Required |
|--------|------|------------|-------------|---------------|
| GET | `/api/health` | `src/app/api/health/route.ts` | Liveness check | No |
| POST | `/api/mindmap` | `src/app/api/mindmap/route.ts` | Create mind map | Yes |
| GET | `/api/mindmap/[id]/nodes` | `src/app/api/mindmap/[id]/nodes/route.ts` | Get all nodes | Yes |
| DELETE | `/api/mindmap/[id]` | `src/app/api/mindmap/[id]/route.ts` | Delete mind map | Yes (Owner) |
```

**Rules:**
- Group endpoints by functional category (Health, Mind Maps, Auth, etc.)
- Always include the `Auth Required` column
- Include `Route File` column showing the file-system path
- Use backticks for paths: `/api/mindmap/[id]`
- Note special auth requirements in parentheses: `Yes (Owner)`

### Technology/Dependency Tables

**Use this format for technology choices:**

```markdown
| Technology | Purpose | Why Chosen |
|------------|---------|------------|
| **Next.js 15** | Full-stack framework | App Router, Server Components, built-in API routes |
| **Drizzle ORM** | Database ORM | Type-safe queries, schema-first, excellent TS integration |
| **Biome** | Linting & formatting | Fast, single-tool replacement for ESLint + Prettier |
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
| `DATABASE_URL` | Yes | PostgreSQL connection string | - |
| `AUTH_SECRET` | Yes | better-auth secret key | - |
| `AUTH_URL` | Yes | Auth callback URL | `http://localhost:3000` |
| `OPENAI_API_KEY` | No | OpenAI API key for AI features | - |
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
    subgraph Client
        Browser[Browser :3000]
    end

    subgraph NextApp[Next.js App :3000]
        AppRouter[App Router]
        ServerComponents[Server Components]
        APIRoutes[API Routes]
    end

    subgraph Services[Services Layer]
        Service1[Mind Map Service]
        Service2[Auth Service]
        Service3[AI Service]
    end

    subgraph Data[Data Layer]
        DB[(PostgreSQL)]
        S3[S3 Storage]
        AI[OpenAI / DeepSeek]
    end

    Browser --> AppRouter
    AppRouter --> ServerComponents
    AppRouter --> APIRoutes
    APIRoutes --> Services
    Services --> DB
    Services --> S3
    Service3 --> AI
\`\`\`
```

**DO NOT include**: Version numbers, dependency lists

### 2. Architecture (architecture.md)
**Purpose**: System design explanation

**Content**:
- Architectural pattern description (Next.js App Router + Services Layer)
- **REQUIRED**: Layer responsibilities table
- **REQUIRED**: At least one sequence diagram (request flow or auth flow)
- Mermaid diagram of component relationships

**Directory Structure**:

For directory structure, use an **inline tree** that you manually create based on the actual file structure. This is an exception to the include rule because:
1. Directory trees are structural documentation, not code
2. Including via anchor would require maintaining the tree in a source file

```markdown
## Directory Structure

\`\`\`
src/
├── app/                    # Next.js App Router
│   ├── (admin)/           # Admin route group
│   ├── [locale]/          # Internationalized routes
│   │   ├── layout.tsx     # Locale layout
│   │   ├── page.tsx       # Home page
│   │   └── mindmap/       # Feature routes
│   │       └── [id]/
│   └── api/               # API Routes (REST)
│       ├── health/
│       └── mindmap/
│           └── [id]/
│               └── nodes/
├── components/             # React Components
│   ├── mindmap/           # Feature-specific
│   ├── auth/
│   └── ui/                # shadcn/ui components
├── contexts/               # React Context providers
├── db/                     # Database Layer
│   ├── schema.ts          # Drizzle ORM schema
│   └── index.ts           # DB connection
├── i18n/                   # Internationalization
├── lib/                    # Utilities
│   ├── resp.ts            # API response helpers
│   └── utils.ts           # General utilities
├── services/               # Business Logic Layer
│   ├── mindmap.ts
│   ├── mindmap-node.ts
│   └── user.ts
└── types/                  # TypeScript Definitions
    ├── mindmap.ts
    └── api.ts
\`\`\`
```

**Note**: Update this tree when adding new modules. Keep it high-level (2-3 levels deep).

**Layer Responsibilities Table**:

```markdown
| Layer | Directory | Responsibility | Examples |
|-------|-----------|----------------|----------|
| **Presentation** | `src/app/`, `src/components/` | UI rendering, Server/Client Components | `page.tsx`, `custom-node.tsx` |
| **API Routes** | `src/app/api/` | HTTP handlers, request validation, auth checks | `route.ts` files |
| **Services** | `src/services/` | Business logic, pure functions, orchestration | `getNodes()`, `createNode()` |
| **Data Access** | `src/db/`, `src/models/` | Database queries, ORM, schema definitions | `schema.ts`, Drizzle queries |
| **Types** | `src/types/` | Type definitions, interfaces, shared types | `NodeData`, `MindMapWithNodes` |
| **Utilities** | `src/lib/` | Helper functions, response formatters | `respData()`, `cn()` |
```

### 3. Next.js App Router (next-app-router.md)
**Purpose**: Next.js-specific patterns used in this project

**Content**:
- Server Components vs Client Components explanation
- Route groups and dynamic routes
- API route patterns
- Data fetching approach

**Server vs Client Components Table**:

```markdown
| Component Type | Directive | When to Use | Examples |
|----------------|-----------|-------------|----------|
| **Server** (default) | None | Data fetching, DB access, no interactivity | `page.tsx`, layouts |
| **Client** | `"use client"` | Interactivity, hooks, browser APIs | Forms, modals, canvas |
```

**Pattern**: Fetch data in Server Components, pass to Client Components as props.

**Include from source** (API route example):
```markdown
## API Route Pattern

\`\`\`typescript
{{#include ../../src/app/api/mindmap/[id]/nodes/route.ts:get_handler}}
\`\`\`
```

**Dynamic Routes**:
- `[id]` — Single dynamic segment
- `[locale]` — Internationalization
- `[...slug]` — Catch-all segments
- `(admin)` — Route group (no URL effect)

Access params via `params` promise (Next.js 15):
```typescript
const { id } = await params;
```

### 4. Tech Stack (tech-stack.md)
**Purpose**: Technology overview

**Content**:
- Programming language and framework (conceptual)
- Key technology choices and WHY they were made (use tables with "Why Chosen" column)
- Links to official documentation

**REQUIRED includes** (never copy):
```markdown
## Production Dependencies

\`\`\`json
{{#include ../../package.json:LINE_START:LINE_END}}
\`\`\`

## Development Dependencies

\`\`\`json
{{#include ../../package.json:LINE_START:LINE_END}}
\`\`\`
```

**Note**: Replace `LINE_START:LINE_END` with actual line numbers from `package.json`. Document these in the Required Anchors output.

**Scripts** (optional):
```markdown
## Available Scripts

\`\`\`json
{{#include ../../package.json:LINE_START:LINE_END}}
\`\`\`
```

### 5. Getting Started (getting-started.md)
**Purpose**: Quick setup guide

**Content**:
- Prerequisites list
- Step-by-step setup (conceptual)
- How to verify success

**Prerequisites**:
- Node.js 20+ (`node --version`)
- pnpm 9+ (`pnpm --version`) — or npm/yarn equivalent
- PostgreSQL 14+ (local or cloud)

**Include from source**:
```markdown
## Environment Setup

Create a `.env.local` file with the following variables (see `.env.example` for reference):

\`\`\`bash
{{#include ../../.env.example}}
\`\`\`

## Quick Start

1. Install dependencies:
   \`\`\`bash
   pnpm install
   \`\`\`

2. Set up database:
   \`\`\`bash
   pnpm drizzle-kit push
   \`\`\`

3. Start development server:
   \`\`\`bash
   pnpm dev
   \`\`\`

4. Verify: Open http://localhost:3000
```

### 6. Development Workflow (development-workflow.md)
**Purpose**: Daily development commands

**Content**:
- Common commands overview
- Link to package.json scripts

**Include from source**:
```markdown
## Available Scripts

See package.json for all available scripts:

\`\`\`json
{{#include ../../package.json:LINE_START:LINE_END}}
\`\`\`
```

**Common Workflows**:

```markdown
**Development**:
\`\`\`bash
pnpm dev              # Start dev server (Turbopack)
\`\`\`

**Code Quality**:
\`\`\`bash
pnpm lint             # Run Biome linter
pnpm format           # Format code with Biome
\`\`\`

**Testing**:
\`\`\`bash
pnpm test             # Run tests (watch mode)
pnpm test:run         # Run tests once
\`\`\`

**Database**:
\`\`\`bash
pnpm drizzle-kit push      # Push schema changes
pnpm drizzle-kit studio    # Open Drizzle Studio GUI
\`\`\`

**Build**:
\`\`\`bash
pnpm build            # Production build
pnpm start            # Start production server
\`\`\`
```

### 7. API Endpoints (api-endpoints.md)
**Purpose**: API reference (if applicable)

**Content**:
- Authentication requirements overview
- Endpoint summary tables (grouped by category)
- Link to route files for detailed logic

**REQUIRED format** (see [Formatting Style Guidelines](#formatting-style-guidelines)):

```markdown
## Authentication

Most endpoints require authentication. The user UUID is extracted via `getUserUuid(req)`:
\`\`\`typescript
{{#include ../../src/lib/auth-helpers.ts:get_user_uuid}}
\`\`\`

## Response Helpers

All API routes use consistent response helpers:
\`\`\`typescript
{{#include ../../src/lib/resp.ts:response_helpers}}
\`\`\`

## Endpoint Summary

### Health Checks

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/health` | Liveness check |

### Mind Maps

| Method | Path | Description | Auth Required |
|--------|------|-------------|---------------|
| GET | `/api/mindmap` | List user's mind maps | Yes |
| POST | `/api/mindmap` | Create mind map | Yes |
| GET | `/api/mindmap/[id]` | Get mind map by ID | Yes (Owner) |
| DELETE | `/api/mindmap/[id]` | Delete mind map | Yes (Owner) |

### Nodes

| Method | Path | Description | Auth Required |
|--------|------|-------------|---------------|
| GET | `/api/mindmap/[id]/nodes` | Get all nodes | Yes (Owner) |
| POST | `/api/mindmap/[id]/nodes` | Create node | Yes (Owner) |
| PATCH | `/api/mindmap/[id]/nodes/[nodeId]` | Update node | Yes (Owner) |
| DELETE | `/api/mindmap/[id]/nodes/[nodeId]` | Delete node | Yes (Owner) |

### Authentication

| Method | Path | Description | Auth Required |
|--------|------|-------------|---------------|
| POST | `/api/auth/login` | Login | No |
| POST | `/api/auth/register` | Register | No |
| DELETE | `/api/auth/logout` | Logout | Yes |
```

**Note**: Keep endpoint tables in sync with actual route files. Reference source files for complex logic.

### 8. Services Layer (services-layer.md)
**Purpose**: Business logic layer patterns

**Content**:
- Pure function service pattern
- Named exports convention
- Async/await patterns
- Error handling approach (services throw, API routes catch)

**Include from source**:
```markdown
## Service Pattern

Services are pure, typed async functions with named exports:

\`\`\`typescript
{{#include ../../src/services/mindmap-node.ts:get_nodes_service}}
\`\`\`

\`\`\`typescript
{{#include ../../src/services/mindmap-node.ts:create_node_service}}
\`\`\`

## Service Organization

| File | Purpose | Key Functions |
|------|---------|---------------|
| `mindmap.ts` | Mind map CRUD | `getMindMaps()`, `verifyOwnership()` |
| `mindmap-node.ts` | Node operations | `getNodes()`, `createNode()`, `updateNode()` |
| `user.ts` | User operations | `getUserUuid()`, `getUserProfile()` |

## Conventions

- **Named exports only** — no default exports from services
- **Explicit return types** — always declare `Promise<ReturnType>`
- **Database via Drizzle** — all DB access through `db()` helper
- **No error handling in services** — let errors propagate to API route catch blocks
```

### 9. Domain Context (domain-context.md)
**Purpose**: Business domain explanation

**Content**:
- Core domain concepts (ubiquitous language)
- Entity relationships (Mermaid ER diagram)
- Business rules summary

**Include from source**:
```markdown
## Core Entities

### Database Schema

\`\`\`typescript
{{#include ../../src/db/schema.ts:users_table}}
\`\`\`

\`\`\`typescript
{{#include ../../src/db/schema.ts:mind_maps_table}}
\`\`\`

\`\`\`typescript
{{#include ../../src/db/schema.ts:nodes_table}}
\`\`\`

### Application Types

\`\`\`typescript
{{#include ../../src/types/mindmap.ts:node_data_type}}
\`\`\`
```

**REQUIRED**: Entity Relationship diagram from Drizzle schema:
```markdown
\`\`\`mermaid
erDiagram
    USERS ||--o{ MIND_MAPS : owns
    MIND_MAPS ||--o{ NODES : contains
    NODES ||--o{ NODES : parent_child

    USERS {
        varchar id PK
        varchar uuid UK
        varchar email
        varchar role
    }

    MIND_MAPS {
        varchar id PK
        varchar user_id FK
        varchar title
        timestamp created_at
    }

    NODES {
        varchar id PK
        varchar mind_map_id FK
        varchar parent_id FK
        varchar title
        text description
        jsonb metadata
    }
\`\`\`
```

### 10. Types & Schema (types-schema.md)
**Purpose**: TypeScript types and database schema organization

**Content**:
- Drizzle schema patterns
- Type inference from schema
- Extended interfaces pattern
- Type organization strategy

**Include from source**:
```markdown
## Database Schema

Drizzle ORM table definitions:

\`\`\`typescript
{{#include ../../src/db/schema.ts:nodes_table}}
\`\`\`

## Schema Relations

\`\`\`typescript
{{#include ../../src/db/schema.ts:schema_relations}}
\`\`\`

## Type Inference

Types are inferred directly from the schema:

\`\`\`typescript
type MindMap = typeof mind_maps.$inferSelect;
type NewMindMap = typeof mind_maps.$inferInsert;
type Node = typeof nodes.$inferSelect;
type NewNode = typeof nodes.$inferInsert;
\`\`\`

## Extended Types

Application-specific types extend schema types:

\`\`\`typescript
{{#include ../../src/types/mindmap.ts:node_data_type}}
\`\`\`

\`\`\`typescript
{{#include ../../src/types/mindmap.ts:mindmap_with_nodes_type}}
\`\`\`

## Type Organization

| Location | Purpose | Examples |
|----------|---------|----------|
| `src/db/schema.ts` | Database tables, relations | `nodes`, `users`, `mind_maps` |
| `src/types/*.ts` | Application interfaces, domain types | `NodeData`, `MindMapWithNodes` |
| Component files | Local props types | `interface CustomNodeProps` |
```

### 11. Configuration (configuration.md)
**Purpose**: Configuration reference

**Content**:
- Configuration approach (environment variables, next.config.ts)
- Runtime vs build-time configuration
- Link to example config

**Include from source**:
```markdown
## Environment Variables

\`\`\`bash
{{#include ../../.env.example}}
\`\`\`

## Environment Variable Reference

| Variable | Required | Description | Default |
|----------|----------|-------------|---------|
| `DATABASE_URL` | Yes | PostgreSQL connection string | - |
| `AUTH_SECRET` | Yes | better-auth signing secret | - |
| `AUTH_URL` | Yes | Auth callback base URL | `http://localhost:3000` |
| `OPENAI_API_KEY` | No | OpenAI API key | - |
| `S3_BUCKET_NAME` | No | AWS S3 bucket for storage | - |

## Next.js Configuration

\`\`\`typescript
{{#include ../../next.config.ts:next_config}}
\`\`\`

## TypeScript Configuration

\`\`\`json
{{#include ../../tsconfig.json:LINE_START:LINE_END}}
\`\`\`

**Path Aliases**:
- `@/*` → `./src/*`
```

### 12. Security (security.md)
**Purpose**: Security patterns overview

**Content**:
- Authentication mechanism (better-auth, session-based)
- Authorization approach (ownership checks via `getUserUuid`)
- Environment variable safety (server-only env vars vs `NEXT_PUBLIC_*`)
- CSRF protection, input validation (Zod)

**DO NOT include**: Actual secrets, tokens, or sensitive configuration

### 13. Testing (testing.md)
**Purpose**: Testing strategy

**Content**:
- Test types and their purpose
- How to run tests
- Test organization

**Include from source**:
```markdown
## Test Configuration

\`\`\`typescript
{{#include ../../vitest.config.ts:test_config}}
\`\`\`

## Test Directory Structure

\`\`\`
tests/
├── setup.ts              # Global test setup
├── unit/                 # Unit tests
│   ├── services/
│   └── lib/
├── integration/          # Integration tests
│   └── api/
└── *.test.ts             # Test files
\`\`\`

## Running Tests

\`\`\`bash
pnpm test              # Watch mode
pnpm test:run          # Run once
pnpm test:cov          # With coverage
\`\`\`

## Test Patterns

**Validation Tests** (Zod schemas):
\`\`\`typescript
describe("createMindMapSchema", () => {
    it("should validate correct data", () => {
        const result = createMindMapSchema.parse({ name: "My Map" });
        expect(result).toEqual({ name: "My Map" });
    });
});
\`\`\`

**API Route Tests**:
\`\`\`typescript
describe("GET /api/mindmap/:id/nodes", () => {
    it("should return 401 without auth", async () => {
        const res = await GET(mockRequest, { params: { id: "123" } });
        expect(res.status).toBe(401);
    });
});
\`\`\`
```

### 14. Deployment (deployment.md)
**Purpose**: Deployment guide

**Content**:
- Deployment targets (Vercel, Docker)
- Build commands
- Health check endpoints
- Environment variable setup

**Vercel (recommended)**:

```markdown
## Vercel Deployment

Vercel is the recommended deployment platform for Next.js:

1. Connect GitHub repository to Vercel
2. Set environment variables in Vercel dashboard
3. Deploy automatically on push to `main`

**Build Settings**:
- Framework: Next.js (auto-detected)
- Build Command: `pnpm build`
- Output Directory: `.next`
```

**Docker (alternative)**:

**Include from source**:
```markdown
## Docker Deployment

\`\`\`dockerfile
{{#include ../../Dockerfile}}
\`\`\`
```

If no Dockerfile exists, provide this standard Next.js template:
```markdown
\`\`\`dockerfile
FROM node:20-alpine AS base

# Dependencies
FROM base AS deps
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN corepack enable pnpm && pnpm install --frozen-lockfile

# Builder
FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN corepack enable pnpm && pnpm build

# Runner
FROM base AS runner
WORKDIR /app
ENV NODE_ENV=production
COPY --from=builder /app/public ./public
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
EXPOSE 3000
CMD ["node", "server.js"]
\`\`\`
```

**Health Checks**:

| Endpoint | Purpose | Expected |
|----------|---------|----------|
| `/api/health` | Liveness probe | `200 OK` |

### 15. Code Patterns (code-patterns.md)
**Purpose**: Coding conventions

**Content**:
- Style guidelines (Biome configuration)
- File naming conventions
- Common patterns and anti-patterns

**Include from source**:
```markdown
## Biome Configuration (Linter & Formatter)

\`\`\`json
{{#include ../../biome.json:LINE_START:LINE_END}}
\`\`\`

**Key Rules**:
- `noExplicitAny: error` — no `any` types allowed
- `noUnusedVariables: error` — no dead code
- `noUnusedImports: error` — auto-removed on format
- Formatter: 2-space indent, 100-char line width, double quotes, semicolons

## File Naming Conventions

| File Type | Convention | Examples |
|-----------|------------|----------|
| React Components | `kebab-case.tsx` | `add-node-dialog.tsx`, `custom-node.tsx` |
| Services | `kebab-case.ts` | `mindmap-node.ts`, `mindmap-health.ts` |
| API Routes | `route.ts` (Next.js convention) | `src/app/api/mindmap/[id]/route.ts` |
| Types | `kebab-case.ts` | `mindmap.ts`, `api.ts` |
| Barrel Exports | `index.ts` | `src/components/mindmap/index.ts` |

## Naming Conventions

| Construct | Convention | Examples |
|-----------|------------|----------|
| React Components | PascalCase | `CustomNode`, `AddNodeDialog` |
| Functions | camelCase | `getNodes`, `createMindMap` |
| Variables | camelCase | `nodeData`, `mindMapId` |
| Types/Interfaces | PascalCase | `NodeData`, `MindMapWithNodes` |
| Constants | camelCase or UPPER_CASE | `defaultTheme`, `MAX_RETRIES` |

## Export Patterns

- **Named exports** (preferred for everything):
  \`\`\`typescript
  export function getNodes() { ... }
  export interface NodeData { ... }
  \`\`\`

- **Default exports** (only for page/layout components — Next.js requirement):
  \`\`\`typescript
  export default function MindMapPage() { ... }
  \`\`\`

- **Barrel exports** via `index.ts`:
  \`\`\`typescript
  export { AddNodeDialog } from "./add-node-dialog";
  export { MindMapCanvas } from "./canvas";
  \`\`\`

## Import Organization

Biome auto-organizes imports in this order:

1. External dependencies (`react`, `next`, `framer-motion`)
2. Internal absolute imports (`@/components/...`, `@/lib/...`)
3. Relative imports (`./`, `../`)
4. Type-only imports (`import type { ... }`)

## Anti-Patterns to Avoid

**Don't use `any`**:
\`\`\`typescript
// BAD
function process(data: any) { ... }
// GOOD
function process(data: NodeData) { ... }
\`\`\`

**Don't mix Server/Client Component concerns**:
\`\`\`typescript
// BAD — Server Component with event handlers
export default function Page() {
    const onClick = () => { ... }  // Error!
    return <button onClick={onClick} />;
}

// GOOD — Separate Client Component
"use client";
export function InteractiveButton() {
    const onClick = () => { ... }
    return <button onClick={onClick} />;
}
\`\`\`

**Don't access env vars in Client Components**:
\`\`\`typescript
// BAD — undefined at runtime
"use client";
const key = process.env.API_KEY;

// GOOD — use in Server Component or API Route
const key = process.env.API_KEY;  // Server only
// OR prefix with NEXT_PUBLIC_ for client access
const url = process.env.NEXT_PUBLIC_APP_URL;
\`\`\`
```

### 16. Dependencies (dependencies.md)
**Purpose**: Dependency rationale

**Content**:
- WHY each major dependency was chosen (not versions)
- Dependency categories

**Include from source**:
```markdown
## Production Dependencies

\`\`\`json
{{#include ../../package.json:LINE_START:LINE_END}}
\`\`\`

## Development Dependencies

\`\`\`json
{{#include ../../package.json:LINE_START:LINE_END}}
\`\`\`
```

**Dependency Rationale Table**:

```markdown
### Core Framework

| Dependency | Why Chosen |
|------------|------------|
| **Next.js 15** | App Router, Server Components, built-in API routes, Turbopack |
| **React 19** | Concurrent features, Server Components support |
| **TypeScript 5.7** | Strict mode, enhanced type inference |

### Database & ORM

| Dependency | Why Chosen |
|------------|------------|
| **Drizzle ORM** | Type-safe SQL, schema-first, excellent TypeScript integration |
| **postgres** | Fast PostgreSQL client for Node.js |

### UI & Styling

| Dependency | Why Chosen |
|------------|------------|
| **Tailwind CSS v4** | Utility-first CSS, zero runtime, v4 performance improvements |
| **shadcn/ui (Radix)** | Accessible headless primitives, fully customizable |
| **Framer Motion** | Declarative animations with layout transitions |

### Authentication & Payments

| Dependency | Why Chosen |
|------------|------------|
| **better-auth** | Modern Next.js auth, flexible providers |
| **Stripe** | Industry-standard payment processing |

### AI & Services

| Dependency | Why Chosen |
|------------|------------|
| **Vercel AI SDK** | Unified AI provider interface, streaming support |

### Dev Tooling

| Dependency | Why Chosen |
|------------|------------|
| **Biome** | Fast linter/formatter, single replacement for ESLint + Prettier |
| **Vitest** | Vite-powered testing, Jest-compatible API |
```

### 17. Troubleshooting (troubleshooting.md)
**Purpose**: Common issues and solutions

**Content**:
- Common error patterns
- Debugging approaches
- FAQ

**Common issues to document**:
- JSON line-based includes break after `package.json` changes (update line ranges)
- `"use client"` directive missing causing hydration errors
- Environment variables undefined in Client Components (needs `NEXT_PUBLIC_` prefix)
- Drizzle schema push failures
- `params` not awaited in Next.js 15 API routes

## Minimal Documentation Philosophy

Each documentation file should:

1. **Explain WHY, not WHAT** — Readers can see WHAT in source code
2. **Use `{{#include}}` for code** — Never copy-paste source code
3. **Link to source files** — For anything beyond includes
4. **Use Mermaid for diagrams** — Architecture, data flow, entity relationships
5. **No hardcoded versions** — All versions come from package.json via includes
6. **Keep it short** — If a section is mostly code, just include the code

## mdBook Include Syntax

```markdown
# Include entire file
{{#include ../path/to/file}}

# Include specific lines (1-indexed)
{{#include ../path/to/file:5:10}}

# Include anchored section
{{#include ../path/to/file:anchor_name}}
```

## Anchor Syntax by Language

**TypeScript/JavaScript:**
```typescript
// ANCHOR: name
code here
// ANCHOR_END: name
```

**Bash / .env files:**
```bash
# ANCHOR: name
content here
# ANCHOR_END: name
```

**TOML:**
```toml
# ANCHOR: name
content here
# ANCHOR_END: name
```

**JSON Limitation:**
JSON files (`package.json`, `tsconfig.json`, `biome.json`) **DO NOT support comments**. For these files:
- Use **line-based includes**: `{{#include ../../file.json:10:25}}`
- Document exact line ranges in the "Required Anchors" output
- Update line ranges whenever the file structure changes

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
      - 'package.json'
      - 'src/**/*.ts'
      - 'src/**/*.tsx'
      - 'next.config.ts'
      - 'biome.json'
      - 'tsconfig.json'
      - '.env.example'

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
        run: |
          mkdir -p ~/bin
          curl -sSL https://github.com/rust-lang/mdBook/releases/download/v0.4.40/mdbook-v0.4.40-x86_64-unknown-linux-gnu.tar.gz | tar -xz --directory=~/bin
          echo "$HOME/bin" >> $GITHUB_PATH

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
- [ ] ALL code blocks use `{{#include}}` — no plain text code snippets
- [ ] No "See [file.ts](...)" links used as substitute for includes
- [ ] Both `dependencies` AND `devDependencies` included in tech-stack.md (line-based)
- [ ] Both `dependencies` AND `devDependencies` included in dependencies.md (line-based)
- [ ] `.env.example` included in configuration.md
- [ ] `next.config.ts` included via anchor in configuration.md
- [ ] Drizzle schema tables included via anchors in types-schema.md
- [ ] Service functions included via anchors in services-layer.md
- [ ] API route handlers included via anchors in api-endpoints.md or next-app-router.md
- [ ] Response helpers included via anchor in api-endpoints.md
- [ ] Dockerfile included in deployment.md (or Vercel config documented)
- [ ] `biome.json` included in code-patterns.md (line-based)
- [ ] `vitest.config.ts` included via anchor in testing.md

### Required Diagrams
- [ ] Overview has service position mermaid diagram
- [ ] Architecture has at least one sequence diagram
- [ ] Next.js App Router has directory structure diagram
- [ ] Domain Context / Types & Schema has entity ER diagram

### Content Organization
- [ ] No duplicate content between sections
- [ ] "Common Issues" content is ONLY in troubleshooting.md
- [ ] Getting Started contains ONLY setup steps (no troubleshooting)
- [ ] Services Layer includes at least 2 service function examples
- [ ] Types & Schema includes both Drizzle schema AND TypeScript types

### Configuration Files
- [ ] book.toml has ACTUAL repository URL (no placeholders like `your-org`)
- [ ] book.toml has `additional-js` for Mermaid
- [ ] book.toml has `additional-css` for fonts
- [ ] SUMMARY.md includes all sections in correct order (including new sections)

### Technical
- [ ] All `{{#include}}` paths are correct relative to `docs/src/`
- [ ] Anchor names match between docs and source files
- [ ] Line-based includes for JSON files use correct line ranges
- [ ] Line ranges for JSON files are documented in Required Anchors output
- [ ] Mermaid JS files present (`mermaid.min.js`, `mermaid-init.js`)
- [ ] Mermaid diagrams render correctly in browser
- [ ] No version numbers hardcoded in prose

### Formatting Consistency
- [ ] API endpoint tables use format: `Method | Path | Description | Auth Required`
- [ ] Endpoints grouped by category (Health, Mind Maps, Auth, etc.)
- [ ] Technology tables use format: `Technology | Purpose | Why Chosen`
- [ ] Configuration tables use format: `Variable | Required | Description | Default`
- [ ] No mixing of bullet lists and tables for similar content

### TypeScript/Next.js Specific
- [ ] Server Components vs Client Components explained in next-app-router.md
- [ ] Services layer pattern documented with function examples
- [ ] Type inference from Drizzle schema (`$inferSelect`) documented
- [ ] File naming conventions (kebab-case files, PascalCase components) documented
- [ ] API route pattern includes `await params` usage (Next.js 15)
- [ ] Response helpers from `src/lib/resp.ts` documented
- [ ] Biome configuration referenced (NOT ESLint/Prettier)
- [ ] `NEXT_PUBLIC_` env var prefix rule documented
- [ ] Barrel export pattern (`index.ts`) documented

## Usage Instructions

1. Run this prompt against a TypeScript/Next.js repository
2. AI generates:
   - `docs/book.toml` (with Mermaid JS configuration)
   - `docs/src/SUMMARY.md` (including TypeScript-specific sections)
   - `docs/src/*.md` files with `{{#include}}` directives
   - List of anchors to add to TypeScript source files
   - Line ranges for JSON file includes
3. Add the anchors to source files:
   - `.ts`/`.tsx` files: Use `// ANCHOR:` comments
   - `.env.example`: Use `# ANCHOR:` comments
   - JSON files: Note line ranges — **no anchors possible**
4. **Set up Mermaid support** (required for diagrams):
   ```bash
   # Download mermaid.min.js
   curl -o docs/mermaid.min.js https://cdn.jsdelivr.net/npm/mermaid@10/dist/mermaid.min.js

   # Create mermaid-init.js (see section 1b above for content)
   ```
   Or copy `mermaid.min.js` and `mermaid-init.js` from an existing service.
5. Run `mdbook build docs/` to verify
6. Run `mdbook serve docs/` and check:
   - Mermaid diagrams render in browser
   - All includes resolve correctly
   - TypeScript code blocks have proper syntax highlighting
7. Set up GitHub Pages or Vercel deployment

## Important: Mermaid Rendering

mdBook does not render Mermaid diagrams by default. The `mermaid-init.js` script:
1. Finds all `<pre><code class="language-mermaid">` blocks
2. Converts them to `<div class="mermaid">` elements
3. Initializes Mermaid.js to render the diagrams

**Do NOT use the `mdbook-mermaid` preprocessor** — it has compatibility issues with `{{#include}}` directives.

This approach ensures documentation stays in sync with source code automatically.
