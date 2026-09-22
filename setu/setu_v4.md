You are a Principal Software Architect and Lead Full-Stack Engineer. Your goal is to design an architecture, data strategy, and execution roadmap for "Project Sanjaya"—a highly scalable Enterprise Remote Config & Feature Flagging Server. 

This system operates similarly to the core targeting engines of LaunchDarkly or PostHog, built for deep B2B multi-tenant organizational hierarchies.

CRITICAL CONSTRAINTS:
1. DO NOT WRITE ANY CODE IN THIS RESPONSE.
2. Focus strictly on system design, database architecture, security controls, and a step-by-step task breakdown.
3. Keep the output token-efficient, dense, and actionable.

# 1. Tech Stack & Environment
- Frontend: Next.js (App Router), Tailwind CSS, shadcn/ui, React Hook Form, TanStack Query, Zustand
- Backend: Node.js, Express.js, Zod (validation)
- Database: PostgreSQL (leveraging recursive CTEs for hierarchical context resolution)
- Cache (Tier 1): Redis (for high-performance read-heavy configuration fetching)
- Workspace: NPM/Yarn Workspaces or Turborepo (shared `types` package)
- DevOps: Docker Compose (containers for frontend, backend, postgres, redis)

# 2. Core Architectural Requirements
1. N-Level Hierarchical Contexts: The root boundary is the **Application**. Applications contain **Projects**, which contain infinite N-level **Sub-Projects**.
2. Context-Aware Targeting (The Cascade): The engine walks up the N-level tree, inheriting and merging flags from the parent Project and root Application. Node-level configs override inherited ones.
3. Advanced Rules Engine: Beyond static booleans, configurations must support dynamic runtime evaluation. The JSONB payload must support Attribute Targeting (e.g., "if user.email ends with @acme.com"), Percentage Rollouts (e.g., "serve true to 10% of users via deterministic hashing"), and Time-based Targeting.
4. Tier 1 Redis Caching: To prevent Postgres degradation, flattened configuration payloads must be cached in Redis. When a config is updated via the UI, the backend must surgically invalidate only the affected branch's cache.
5. Environment Segregation: Configurations are strictly isolated by Environment (e.g., Dev, Staging, Prod). 
6. The Kill Switch (Graceful Sunsetting): Hard deletions are forbidden. EVERY context node and EVERY feature flag MUST have an `isActive` boolean (the kill-switch).

# 3. Database Schema Strategy
Part A: Hierarchical Registry Tables (The Context Tree)
- `Applications` (id, name, is_active, created_at, updated_at)
- `Environments` (id, name, is_active, created_at)
- `Projects` (id, application_id, parent_project_id, name, is_active, created_at, updated_at)

Part B: Remote Config Tables (Polymorphic EAV Pattern)
- `Application_Configs` & `Project_Configs` 
- Columns: `Node_id` (Reference), `Environment_id`, `Key` (String), `Value` (JSONB - Supports static values OR Rules Engine schemas), `is_active` (Boolean), Audit Columns.

# 4. API Payload Representation (Cascading & Rules)
Example of a cached, flattened payload stored in Redis and returned to a microservice:
{
  "metadata": {
    "application": "AgencyPortal",
    "environment": "production",
    "target_project": "Sub-Project AA",
    "cached_at": "2026-09-21T19:57:42Z"
  },
  "flattened_active_output": {
    "enforce_mfa": true, // Static evaluation
    "useNewAuthFlow": { // Rules Engine evaluation
      "rules": [
        { "if": "user.email", "operator": "endsWith", "target": "@agency.gov", "serve": true },
        { "rolloutPercentage": 20, "serve": true }
      ],
      "default": false
    }
  }
}

# 5. Required Deliverable Format
Provide a detailed architecture document covering the following sections:

## Section 1: Architectural Strategy & System Boundaries
- Context Resolution Strategy: Postgres Recursive CTE implementation for infinite nesting.
- Redis Caching Strategy: Define the cache key nomenclature, TTL strategy, and surgical cache invalidation pattern when a parent node updates.
- Advanced Rules Engine: Architecture for parsing the JSONB rules (targeting, hashing for rollouts) in the backend before returning final states, or passing rules to downstream SDKs.

## Section 2: Database Design & Data Integrity
- Formalize the PostgreSQL schema, explicitly mapping the `is_active` kill switches.
- Define compound unique constraints and indexing strategies for fast resolution.

## Section 3: API Architecture & Type Contracts
- RESTful API design specs for:
  - Context Tree Traversal (fetching the N-level tree for the UI selector).
  - The Evaluation Endpoint (Cache-first check -> Postgres Fallback -> Rule Evaluation).
  - The Upsert Endpoint (Writing EAV rows and invalidating Redis).

## Section 4: Frontend UI/UX Engineering
- N-Level Context Explorer UI: Strategy for rendering the recursive TanStack Query tree.
- Dynamic Flag & Rules Dashboard: Architecture for rendering nested JSON / Rules Engine builders (e.g., adding "If/Then" rule rows) into shadcn/ui form fields with `isActive` kill switches.

## Section 5: Docker & Local Developer Experience (DX)
- Topology for `docker-compose.yml` (Next.js, Express, Postgres, Redis, and DB seed scripts).

## Section 6: Step-by-Step Implementation Roadmap
Break the development into bite-sized, sequential micro-tasks (Init Monorepo -> Postgres Schema -> Redis Integration -> Rules Engine -> Express APIs -> UI Tree -> UI Rule Builder) for isolated coding sessions.

## Section 7: Future Upgrades (Out of Scope for Phase 1, but must not be blocked by architecture)
Briefly acknowledge how the architecture leaves room for:
- Tier 2 Edge Delivery (CDN / Cloudflare Workers KV).
- Real-Time Delivery via Server-Sent Events (SSE).
- Client-Side Resilience (Local Evaluation SDKs with memory caching).
- Enterprise Governance (Maker/Checker workflows, Drafts, Diff Views).
- Observability (Stale flag telemetry).