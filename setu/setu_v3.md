You are a Principal Software Architect and Lead Full-Stack Engineer. Your goal is to design an architecture, data strategy, and execution roadmap for a highly scalable Enterprise Remote Config & Feature Flagging Server. 

This system operates similarly to the core targeting engines of LaunchDarkly or PostHog, but is purpose-built for deep B2B multi-tenant organizational hierarchies rather than flat user lists.

CRITICAL CONSTRAINTS:
1. DO NOT WRITE ANY CODE IN THIS RESPONSE.
2. Focus strictly on system design, database architecture, security controls, and a step-by-step task breakdown.
3. Keep the output token-efficient, dense, and actionable.

# 1. Tech Stack & Environment
- Frontend: Next.js (App Router), Tailwind CSS, shadcn/ui, React Hook Form, TanStack Query, Zustand
- Backend: Node.js, Express.js, Zod (for validation)
- Database: PostgreSQL (leveraging recursive CTEs for hierarchical context resolution)
- Workspace: NPM/Yarn Workspaces or Turborepo (shared `types` package across UI and API)
- DevOps: Docker Compose (containers for frontend, backend, postgres)

# 2. Core Architectural Requirements
1. N-Level Hierarchical Contexts: The root boundary is the **Application** (e.g., AgencyPortal). Applications contain **Projects** (e.g., BillingModule), which can contain infinite N-level **Sub-Projects**.
2. Context-Aware Targeting & Resolution (The Cascade): A downstream microservice requesting its configuration will provide its context (e.g., "Sub-Project AA"). The engine must walk up the tree, inheriting and merging flags from the parent Project and root Application. Node-level configs override inherited ones.
3. Environment Segregation: Configurations and feature flags must be strictly isolated by Environment (e.g., Development, Staging, Production). 
4. The Kill Switch (Graceful Sunsetting): Hard deletions are forbidden. EVERY context node (Application, Project) and EVERY individual feature flag MUST have an `isActive` boolean. This acts as an instant kill-switch to sunset features or disable entire tenant sub-trees.
5. Data Transformation (Flatten & Rehydrate): The UI manages configurations via nested JSON forms, but the backend must flatten this into strict Entity-Attribute-Value (EAV) records to store in the relational database.

# 3. Database Schema Strategy
The database must handle infinite recursion and polymorphic payloads efficiently.

Part A: Hierarchical Registry Tables (The Context Tree)
- `Applications` (id, name, is_active, created_at, updated_at) - The Root boundaries.
- `Environments` (id, name, is_active, created_at) - Context identifiers.
- `Projects` (id, application_id, parent_project_id, name, is_active, created_at, updated_at) - The self-referencing N-level tree.

Part B: Remote Config Tables (Polymorphic EAV Pattern)
Table 1: `Application_Configs` (Root-level global policies & flags)
- `Application_id` (Reference)
- `Environment_id` (Reference)
- `Key` (String - The feature flag or config key)
- `Value` (JSONB - Supports boolean, string, number, or complex JSON objects)
- `is_active` (Boolean - The kill switch for this specific flag)
- Audit Columns (`Created_by`, `Created_date`, `Updated_by`, `Updated_date`)

Table 2: `Project_Configs` (Node-level flag overrides at any depth)
- `Project_id`, `Environment_id`, `Key`, `Value` (JSONB), `is_active`, Audit Columns.

# 4. API Payload Representation (Cascading Resolution)
When a client evaluates flags for "Sub-Project AA", the API executes a recursive tree-walk and returns the merged state.

Example Resolution Payload:
{
  "metadata": {
    "application": "AgencyPortal",
    "environment": "production",
    "target_project": { "name": "Sub-Project AA", "isActive": true },
    "project_path": [
      { "name": "Project A", "isActive": true },
      { "name": "Sub-Project AA", "isActive": true }
    ]
  },
  "resolved_config_tree": {
    "inherited_from_application": {
      "enforce_mfa": { "value": true, "isActive": true }
    },
    "inherited_from_project_a": {
      "default_region": { "value": "us-east-1", "isActive": false } // Kill switch activated
    },
    "node_config": {
      "version": { "value": "1.4.2", "isActive": true },
      "feature_flags": { 
        "value": { "enableAuditLogging": true, "useNewAuthFlow": false }, 
        "isActive": true 
      }
    }
  },
  "flattened_active_output": {
    // The final payload consumed by downstream apps (filters out isActive: false)
    "enforce_mfa": true,
    "version": "1.4.2",
    "feature_flags": {
      "enableAuditLogging": true,
      "useNewAuthFlow": false
    }
  }
}

# 5. Required Deliverable Format
Provide a detailed architecture document covering the following sections:

## Section 1: Architectural Strategy & System Boundaries
- Monorepo Strategy for sharing Zod validation schemas between Next.js and Express.
- Context Resolution Strategy: Deep dive into the Postgres implementation for infinite nesting. Compare Adjacency List (Recursive CTEs) vs. Materialized Paths (`ltree`). Recommend the best approach for read-heavy flag evaluation.
- Flatten & Rehydrate Pattern: Architecture for translating flat database EAV rows into nested JSON payloads, and vice versa.

## Section 2: Database Design & Data Integrity
- Formalize the exact PostgreSQL schema, explicitly mapping the `is_active` kill switches.
- Define compound unique constraints (e.g., Node + Env + Key), indexing strategies for fast resolution, and audit-logging approaches.

## Section 3: API Architecture & Type Contracts
- RESTful API design specs for:
  - Context Tree Traversal (fetching the N-level project tree for the UI selector).
  - The Evaluation Endpoint (resolving and merging flags for a specific target node).
  - The Upsert Endpoint (accepting an array of `{key, value, isActive}` objects to flatten into the database).

## Section 4: Frontend UI/UX Engineering
- N-Level Context Explorer UI: Strategy for using TanStack Query to fetch and render a recursive, folder-like tree, applying visual badges to "sunset" (inactive) nodes.
- Dynamic Flag Dashboard: Architectural design for rendering nested JSON `Values` into shadcn/ui form fields, ensuring a prominent "Sunset / Active" toggle is bound to every flag.

## Section 5: Docker & Local Developer Experience (DX)
- Topology for `docker-compose.yml` (Next.js, Express, Postgres, and DB seed scripts).

## Section 6: Step-by-Step Implementation Roadmap
Break the development into bite-sized, sequential micro-tasks (e.g., Task 1: Init Monorepo, Task 2: Postgres Recursive Schema, Task 3: Express Traversal Service, Task 4: UI Context Selector, etc.) designed to be coded individually in isolated chat sessions.