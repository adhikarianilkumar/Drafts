You are a Principal Software Architect and Lead Full-Stack Engineer. Your goal is to design an architecture, data strategy, and execution roadmap for a highly scalable, enterprise-grade configuration management platform.

CRITICAL CONSTRAINTS:
1. DO NOT WRITE ANY CODE IN THIS RESPONSE.
2. Focus strictly on system design, database architecture, security controls, and step-by-step task breakdown.
3. Keep the output token-efficient, dense, and actionable.

# 1. Tech Stack & Environment
- Frontend: Next.js (App Router), Tailwind CSS, shadcn/ui, React Hook Form, TanStack Query (React Query), Zustand (for state)
- Backend: Node.js, Express.js, Zod (for validation)
- Database: PostgreSQL (leveraging recursive CTEs or `ltree` for hierarchy)
- Workspace: NPM/Yarn Workspaces or Turborepo (to share a common `types` package)
- DevOps: Docker Compose (separate containers for frontend, backend, postgres)

# 2. Core Functional Requirements
1. N-Level Hierarchical Data: The root namespace is the **Application**. Applications contain **Projects**, which can contain **Sub-Projects**, down to infinite N-levels of depth.
2. N-Level Config Cascading (Inheritance): A Sub-Sub-Project must inherit configs from its parent Sub-Project, its grandparent Project, and the root Application. Lower-level configs override inherited ones.
3. Sunset/Soft-Delete Protocol: Data is never hard-deleted. EVERY level (Application, Project) and EVERY individual configuration key MUST have an `isActive` boolean switch so features or entire projects can be gracefully sunset/retired.
4. Tree-Based Context Selection: Users navigate and select their target context (Application -> Project path -> Environment) via a recursive tree-view UI.
5. Universal CRUD & Search: Users can create, read, update, sunset (`isActive: false`), and search (`Key`) for configurations at any node in the hierarchy.
6. Data Transformation: The UI handles data as nested JSON forms, but the backend flattens this into strict Key-Value relational database tables linked to the hierarchical tree.

# 3. Database Schema Strategy
The system must handle N-level hierarchies and EAV (Entity-Attribute-Value) storage elegantly.

Part A: Hierarchical Registry Tables (The Tree)
- `Applications` (id, name, is_active, created_at, updated_at) - The Root nodes.
- `Environments` (id, name, is_active, created_at) - Context identifiers (e.g., dev, staging, prod).
- `Projects` (id, application_id, parent_project_id, name, is_active, created_at, updated_at) - The self-referencing tree.

Part B: Configuration Tables (EAV Pattern)
Table 1: `Application_Configs` (Root-level global policies for the application)
- `Application_id` (Reference)
- `Environment_id` (Reference)
- `Key` (String)
- `Value` (JSONB - stores string, boolean, number, or object)
- `is_active` (Boolean - allows sunsetting a specific config)
- Audit Columns (`Created_by`, `Created_date`, `Updated_by`, `Updated_date`)

Table 2: `Project_Configs` (Node-level settings at any depth)
- `Project_id` (Reference)
- `Environment_id` (Reference)
- `Key` (String)
- `Value` (JSONB)
- `is_active` (Boolean)
- Audit Columns (`Created_by`, `Created_date`, `Updated_by`, `Updated_date`)

# 4. API Payload Representation (Cascading Resolution)
When a client requests the config for "Sub-Project AA", the API must walk up the tree, merging configurations and evaluating the `isActive` flags. 

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
      "default_region": { "value": "us-east-1", "isActive": false } // This feature is sunset
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
    // What downstream microservices consume (filters out isActive: false)
    "enforce_mfa": true,
    "version": "1.4.2",
    "feature_flags": {
      "enableAuditLogging": true,
      "useNewAuthFlow": false
    }
  }
}

# 5. Required Deliverable Format

Provide a detailed architecture document covering the following key sections:

## Section 1: Architectural Strategy & System Boundaries
- Monorepo Strategy: How the workspace will be structured.
- N-Level Hierarchy Strategy: Deep dive into the Postgres implementation for infinite nesting. Compare Adjacency List (`parent_id`) with Recursive CTEs vs. materialized paths (`ltree`). Recommend the absolute best approach for read-heavy cascading.
- Flatten & Rehydrate Pattern: Detail the backend architecture for flattening dynamic JSON forms into EAV records with `isActive` flags, and merging them back during hierarchical resolution.

## Section 2: Database Design & Data Integrity
- Formalize the exact PostgreSQL schema for the Registry (Tree) tables and Configuration tables, explicitly including the `is_active` flags.
- Define compound unique constraints, indexing strategies for fast hierarchical querying, and full-text search for the `Key` column.

## Section 3: API Architecture & Type Contracts
- RESTful API design specs for:
  - Fetching the N-level project tree (handling `isActive` UI visibility).
  - The "Resolution Endpoint" (traversing the tree to return the merged config).
  - The Upsert Endpoint (accepting an array of `{key, value, isActive}` objects from the frontend to flatten into the database).

## Section 4: Frontend UI/UX Engineering
- N-Level Tree Explorer UI: Strategy for using TanStack Query to fetch and cache hierarchical registry data (rendering an expandable folder-like tree, marking inactive nodes with a badge).
- Recursive Dynamic Form Generator: Architectural design for translating nested JSON values into shadcn/ui form fields. Ensure the UI includes a prominent "Sunset / Active" toggle switch next to every configuration key.

## Section 5: Docker & Development Environment Setup
- Topology for `docker-compose.yml` (Frontend, Backend, Postgres, and DB initialization scripts for tree seeding).

## Section 6: Step-by-Step Implementation Roadmap (Token-Optimized)
Break the development into bite-sized, sequential micro-tasks (e.g., Task 1: Init Monorepo, Task 2: Postgres Recursive Schema, Task 3: Express Traversal Service, Task 4: UI Tree Selector, Task 5: Dynamic Forms with isActive toggles, etc.) designed to be coded individually in isolated chat sessions.