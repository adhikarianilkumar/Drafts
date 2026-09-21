You are a Principal Enterprise Architect and Lead Full-Stack Engineer. Your goal is to design an architecture, data strategy, and execution roadmap for an enterprise-grade configuration management platform.

CRITICAL CONSTRAINTS:
1. DO NOT WRITE ANY CODE IN THIS RESPONSE.
2. Focus strictly on system design, database architecture, security controls, and step-by-step task breakdown.
3. Keep the output token-efficient, dense, and actionable.

# 1. Tech Stack & Environment
- Frontend: Next.js (App Router), Tailwind CSS, shadcn/ui, React Hook Form, TanStack Query (React Query), Zustand (for state)
- Backend: Node.js, Express.js, Zod (for validation)
- Database: PostgreSQL
- Workspace: NPM/Yarn Workspaces or Turborepo (to share a common `types` package)
- DevOps: Docker Compose (separate containers for frontend, backend, postgres)

# 2. Core Functional Requirements
1. Enterprise Global Configs (Single Source of Truth): A dedicated space for enterprise-wide settings (e.g., global security policies, SSO endpoints). All applications under an enterprise must inherit/access this data automatically.
2. Application-Level Configs: Non-technical users can dynamically create and manage configurations for specific applications via the UI.
3. Standardized Context Selection: Users must select Enterprise, Application, and Environment from predefined UI dropdowns (populated from registry tables) to ensure data consistency.
4. Search & Discovery: Users must be able to search configurations globally or within an application using the `Key`.
5. Data Transformation: The UI will represent data as nested forms/JSON, but the backend must flatten this into strict Key-Value relational database tables.

# 3. Database Schema Strategy
The system will use a highly scalable flattened Key-Value store pattern paired with Reference tables.

Part A: Registry Tables (For UI Dropdowns)
- `Enterprises` (id, name, created_at)
- `Applications` (id, enterprise_id, name, created_at)
- `Environments` (id, name, created_at)

Part B: Configuration Tables (EAV Pattern)
Table 1: `Enterprise_Global_Configs` (For agency-wide policies)
- `Enterprise_name` (Reference)
- `Environment` (Reference)
- `Key` (String - e.g., "global_auth_policy")
- `Value` (JSONB)
- Audit Columns (`Created_by`, `Created_date`, `Updated_by`, `Updated_date`)

Table 2: `Application_Configs` (For app-specific settings)
- `Enterprise_name` (Reference)
- `Application_name` (Reference)
- `Environment` (Reference)
- `Key` (String - e.g., "feature_flags")
- `Value` (JSONB)
- Audit Columns (`Created_by`, `Created_date`, `Updated_by`, `Updated_date`)

# 4. Frontend Payload Representation
While the database is flat, the API must return a structured JSON object to the UI and downstream consumers. The backend should ideally merge inherited global configs with app configs. Example payload:
{
  "Enterprise_name": "FederalAgencyX",
  "Application_name": "AgencyConfigService",
  "Environment": "production",
  "inherited_global_policies": {
    "enforce_mfa": true,
    "session_timeout_minutes": 30
  },
  "app_config": {
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
- Monorepo Strategy: How the workspace will be structured to share TypeScript interfaces between Next.js and Express.
- Inheritance & Cascade Strategy: How the backend will resolve and merge `Enterprise_Global_Configs` with `Application_Configs` when an application requests its configuration.
- Flatten & Rehydrate Pattern: Detail the architectural approach for the backend to accept nested JSON payloads, flatten them into individual DB records, and rehydrate them into JSON trees.

## Section 2: Database Design & Data Integrity
- Formalize the exact PostgreSQL schema for both Registry tables and both Configuration tables.
- Define compound unique constraints, indexing strategies (e.g., Enterprise + App + Env + Key), and full-text search strategies for the `Key` column.
- Audit Strategy: How to handle config versioning (e.g., an append-only history table).

## Section 3: API Architecture & Type Contracts
- RESTful API design specs for:
  - Fetching dropdown registry data.
  - CRUD operations for Global Enterprise Configs vs Application Configs.
  - The "Resolution Endpoint" (returning the merged Global + App payload for downstream microservices).
- Validation strategy: How Zod will be used on the Express backend.

## Section 4: Frontend UI/UX Engineering
- Context Selector UI: Strategy for using TanStack Query to fetch and cache registry data for the global Enterprise/Application/Environment dropdowns.
- Management UI: How the UI will distinguish between managing Global Enterprise configurations vs. Application-specific configurations.
- Recursive Dynamic Form Generator: Architectural design for translating the JSON structure into human-friendly shadcn/ui form fields utilizing React Hook Form.

## Section 5: Docker & Development Environment Setup
- Topology for `docker-compose.yml` (Frontend, Backend, Postgres, environment networks, and database initialization scripts).

## Section 6: Step-by-Step Implementation Roadmap (Token-Optimized)
Break the development into bite-sized, sequential micro-tasks (e.g., Task 1: Init Monorepo, Task 2: DB Schema & Registry, Task 3: Express Flatten/Merge Service, Task 4: UI Dropdowns & Search, etc.) designed to be coded individually in isolated chat sessions.