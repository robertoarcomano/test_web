# MATRIX 2.0

Project management platform for Fondazione Matrice.
Full-stack TypeScript monorepo: **Next.js 15** (frontend) + **NestJS** (backend).

The core paradigm is the shift from hard-coded project cards to a dynamic, **configuration-driven** model orchestrated by the **Module Engine**: new grant requirements are met by composing `ModuleDefinition` objects — no code releases required.

---

## Structure

```
matrix/
├── apps/
│   ├── web/          # Next.js 15 + React 19 + Tailwind CSS 4
│   └── api/          # NestJS + Fastify + TypeORM + PostgreSQL
├── packages/
│   └── shared/       # Shared types (Module Engine, Auth, API)
└── docker-compose.yml
```

### API modules

```
apps/api/src/
├── config/               # Typed env config (Joi validation, registerAs namespaces)
├── common/
│   ├── guards/           # JwtAuthGuard, RolesGuard
│   └── decorators/       # @CurrentUser(), @Roles()
└── modules/
    ├── auth/             # Login, register, refresh, logout — JWT access + refresh token
    ├── users/            # UserEntity, UsersService, GET /users (admin/fondazione)
    ├── organizations/    # OrganizationEntity, UserOrganizationEntity, membership management
    ├── projects/         # ProjectEntity + ProjectTemplateEntity; auto-creates ModuleInstances on POST
    └── module-engine/    # ModuleDefinition, ModuleInstance, validation engine, audit log,
                          #   file upload (disk storage via @fastify/multipart, served at /uploads/)
```

### Frontend pages

```
apps/web/src/
├── app/
│   ├── (app)/                    # Authenticated route group — AppLayout with sidebar
│   │   ├── layout.tsx            # Shell: fixed sidebar + scrollable main
│   │   ├── organizations/
│   │   │   ├── page.tsx          # Organization list
│   │   │   ├── new/page.tsx      # Create organization form
│   │   │   └── [id]/page.tsx     # Organization detail + member management
│   │   └── projects/
│   │       ├── page.tsx          # Project list (table with status and template)
│   │       ├── new/page.tsx      # Create project form
│   │       ├── [id]/page.tsx     # Project detail + ModuleInstance list
│   │       └── [id]/modules/
│   │           └── [moduleInstanceId]/page.tsx  # Module renderer dispatcher → type-specific renderer
│   ├── login/page.tsx            # Public login page
│   └── page.tsx                  # Redirect → /projects
├── components/
│   ├── ui/                       # Atomic components (PageHeader, Breadcrumb, DataTable,
│   │                             #   FormField, LoadingState, ErrorState, EmptyState,
│   │                             #   OrgTypeBadge)
│   ├── form/                     # FormRenderer
│   ├── budget/                   # BudgetRenderer
│   ├── matrix/                   # MatrixRenderer
│   ├── tree/                     # TreeRenderer
│   ├── planning/                 # PlanningRenderer
│   ├── financial/                # FinancialRenderer
│   ├── documents/                # DocumentsRenderer
│   ├── indicators/               # IndicatorsRenderer
│   ├── sidebar.tsx               # Lateral navigation with logout
│   └── status-badge.tsx          # Colored badge for project/instance states
├── hooks/                        # Data-fetching hooks (useProjects, useOrganizations,
│                                 #   useProjectDetail, useOrgDetail)
├── lib/
│   ├── api-client.ts             # Fetch wrapper with automatic JWT refresh
│   ├── auth-context.tsx          # AuthProvider (login, logout, refresh)
│   └── token-sync.tsx            # Syncs token into api-client
├── types/                        # Local TS interfaces (Organization, ProjectRow, UserOption…)
└── utils/                        # Pure functions (formatDate, formatDateLong, toSlug)
```

---

## Prerequisites

- Node.js ≥ 22
- Docker + Docker Compose (for the local database)

---

## Quick start

### 1. Install dependencies

```bash
npm install
```

### 2. Start the PostgreSQL database

```bash
docker compose up -d
```

### 3. Configure the API environment

```bash
cp apps/api/.env.example apps/api/.env
# Generate secrets:
#   openssl rand -base64 48  → JWT_SECRET
#   openssl rand -base64 48  → JWT_REFRESH_SECRET
```

### 4. Start everything in development mode

```bash
npm run dev
```

- **Frontend**: http://localhost:3000
- **API**: http://localhost:3001/api
- **Swagger**: http://localhost:3001/api/docs
- **Adminer** (DB GUI): http://localhost:8080
- **Mailpit** (local email inbox): http://localhost:8025

---

## Environment Files

The project intentionally keeps development and deployment environment files separate:

| File | Purpose | Committed |
|---|---|---|
| `apps/api/.env` | Local API development values used by NestJS when running `npm run dev -w apps/api`. | no |
| `apps/api/.env.example` | Template for local API development. | yes |
| `.env.prod` | Production deploy values, used only for manual Docker Compose deploys or as a local source for GitHub environment secrets. | no |
| `.env.staging` | Staging deploy values, used only as a local source for GitHub staging secrets. | no |
| `.env.prod.example` | Template for production deploy values. | yes |

Do not use production secrets in `apps/api/.env`; that file is for local development only.

GitHub Actions does not read `.env.prod` or `.env.staging` from the repository. During deploy it creates a temporary `.env.prod` on the self-hosted runner from the selected GitHub environment secrets and variables. Branch-specific values such as exposed ports and Docker volume names are defined directly in `.github/workflows/deploy.yml`.

---

## Deploy with GitHub Actions

The repository includes `.github/workflows/deploy.yml`, configured for a Docker Compose deployment on a **self-hosted GitHub Actions runner**.

### Deployment flow

On every push to `main` or `staging`, GitHub Actions:

1. Installs dependencies on `ubuntu-latest`.
2. Runs typecheck, tests and production build.
3. Builds the Docker images on the self-hosted runner.
4. Stops the application containers while keeping PostgreSQL available.
5. Starts PostgreSQL and waits for it to be healthy.
6. Runs TypeORM migrations with the API image.
7. Starts or updates the remaining services with `docker-compose.prod.yml`.

Branch mapping:

| Branch | GitHub environment | Compose project | Public port |
|---|---|---|---|
| `main` | `production` | `matrix2-main` | `10004` |
| `staging` | `staging` | `matrix2-staging` | `20004` |

### Server requirements

Install these on the target server:

- Docker Engine + Docker Compose plugin
- A self-hosted GitHub Actions runner registered on this repository
- Network access from the exposed app port to your reverse proxy or firewall

The runner user must be allowed to run Docker commands.

### GitHub configuration

Create two GitHub environments: `production` and `staging`.

For each environment, configure these **secrets**:

```text
DB_USER
DB_PASSWORD
DB_NAME
WEB_URL
JWT_SECRET
JWT_REFRESH_SECRET
```

Optional **secrets** for SMTP:

```text
SMTP_HOST
SMTP_USER
SMTP_PASSWORD
```

Optional **environment variables**:

```text
JWT_EXPIRES_IN=1h
SMTP_PORT=25
SMTP_SECURE=false
SMTP_FROM=MATRIX 2.0 <noreply@matrix.it>
```

`DB_SYNCHRONIZE` is forced to `false` by the deployment workflow for both `production` and `staging`. Schema changes must be deployed through TypeORM migrations.

### Manual server deploy

For a one-off deploy without GitHub Actions:

```bash
cp .env.prod.example .env.prod
docker compose --env-file .env.prod -f docker-compose.prod.yml build
docker compose --env-file .env.prod -f docker-compose.prod.yml stop api web nginx
docker compose --env-file .env.prod -f docker-compose.prod.yml up -d --wait db
docker compose --env-file .env.prod -f docker-compose.prod.yml run --rm api npm run migration:run
docker compose --env-file .env.prod -f docker-compose.prod.yml up -d --remove-orphans
```

### Schema migrations

Migrations live in `apps/api/src/migrations` and are executed through the compiled API bundle. Build the API before running migration commands locally or in Docker:

```bash
npm run build -w apps/api
npm run migration:show -w apps/api
npm run migration:run -w apps/api
```

For a new schema change:

1. Update the TypeORM entities.
2. Build the API so `dist/apps/api/src/data-source.js` reflects the new metadata.
3. Generate a migration, for example:

```bash
npm run migration:generate -w apps/api -- src/migrations/AddProjectBudget
```

4. Review the generated SQL, run it against a local or staging database, and commit the migration with the entity changes.

The initial migration is tolerant of staging databases that were already created with the temporary `DB_SYNCHRONIZE=true` workaround: if the full initial table set already exists, it marks the migration as applied without recreating tables. A partial schema still fails loudly and should be repaired or reset before deploying.

### Default seed accounts

On first boot (or after `docker compose down -v`) the API seeds these accounts automatically:

| Email | Password | Role |
|---|---|---|
| `admin@matrice.it` | `Admin1234!` | admin |
| `fondazione@matrice.it` | `Fondazione1234!` | fondazione |
| `capofila@matrice.it` | `Capofila1234!` | capofila |
| `partner@matrice.it` | `Partner1234!` | partner |
| `revisore@matrice.it` | `Revisore1234!` | revisore |

---

## Tech stack

Legend: ✅ implemented · 🚧 in progress · ☐ planned

### Backend

| Component | Technology | Status |
|---|---|---|
| Core framework | NestJS 11 + Fastify | ✅ |
| ORM | TypeORM 0.3 | ✅ |
| Database | PostgreSQL 17 (JSONB for module data) | ✅ |
| Auth | JWT access token (1 h) + HttpOnly refresh token (7 d) | ✅ |
| Validation | class-validator + class-transformer | ✅ |
| API docs | Swagger / OpenAPI (`/api/docs`) | ✅ |
| Config | Joi-validated typed env, `registerAs` namespaces | ✅ |
| 2FA | — | ☐ |
| WebSocket rooms (Yjs provider) | y-websocket | ✅ |
| Vector search | pgvector extension | ☐ |
| External integrations | Accounting APIs, digital signature | ☐ |

### Frontend

| Component | Technology | Status |
|---|---|---|
| Core framework | Next.js 15 (App Router, Server Components) | ✅ |
| UI runtime | React 19 | ✅ |
| Styling | Tailwind CSS 4 (utility-first, design tokens in `globals.css`) | ✅ |
| Component library | MUI X Community — Data Grid v8 (Budget, Financial transactions) | ✅ |
| Form engine | Custom JSON-schema renderer (FormRenderer) | ✅ |
| Real-time sync | Yjs (CRDT) + y-websocket + awareness cursors | ✅ |
| Drag-and-drop | @dnd-kit/core (matrix quadrants drag between cells) | ✅ |
| Tree view | he-tree-react (hierarchical objectives → actions → activities, native drag-to-reorder) | ✅ |
| Gantt chart | @svar-ui/react-gantt — Willow theme (Planning module) | ✅ |
| Rich-text editor | Tiptap (with Yjs binding for co-editing) | ✅ |

### Shared

| Component | Technology | Status |
|---|---|---|
| Shared types | `@matrix2/shared` (importable in both apps) | ✅ |
| Schema validation | Zod (runtime validation of ModuleDefinition schemas) | ☐ |

---

## Auth & permissions

- **Access token**: JWT, 1 h, returned in response body
- **Refresh token**: JWT, 7 d, HttpOnly cookie on `/api/auth`
- **Platform roles** (on `UserEntity`): `admin | fondazione | capofila | partner | revisore | readonly`
- **Org membership** (`UserOrganizationEntity`): per-user per-organization role
- `RolesGuard` + `@Roles()` for platform-level control; `OrganizationsService.userHasRole()` for org-level checks
- `PermissionMatrix` on each `ModuleDefinition` controls read/write/validate/comment at runtime
- Project visibility derives from organization membership and module permissions.

### Roles

| Role | Responsibility |
|---|---|
| `admin` | Platform administrator for technical and operational management. |
| `fondazione` | Fondazione Matrice staff: manages grants, projects and reviews. |
| `revisore` | External reviewer appointed by the Foundation for review work. |
| `capofila` | Lead organization responsible for formal project submission. |
| `partner` | Partner organization that contributes data and documents. |
| `readonly` | Observer/auditor with read-only access. |

### Permission matrix

| Action | admin | fondazione | revisore | capofila | partner | readonly |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| Create/archive projects | yes | yes | no | yes | no | no |
| See all projects | yes | yes | yes | no | no | no |
| See own organization projects | yes | yes | yes | yes | yes | yes |
| Fill and save module drafts | yes | yes | no | yes | yes | no |
| Submit modules for review | yes | yes | no | yes | no | no |
| Approve/reject/request changes | yes | yes | yes | no | no | no |
| See module data | yes | yes | yes | yes | yes | yes |
| Create/edit/publish definitions | yes | yes | no | no | no | no |
| See definitions | yes | yes | no | no | no | no |
| Create/edit organizations | yes | yes | no | no | no | no |
| Manage users | yes | no | no | no | no | no |
| See user list | yes | yes | no | no | no | no |
| Upload document files | yes | yes | no | yes | yes | no |
| Download/view document files | yes | yes | yes | yes | yes | yes |
| Write comments and mentions | yes | yes | yes | yes | yes | no |
| Receive in-app/email notifications | yes | yes | yes | yes | yes | no |

Important rules:

- `partner` can edit drafts but cannot submit modules. Formal submission is owned by `capofila`.
- `capofila`, `partner` and `readonly` see only projects whose `leadOrganizationId` belongs to one of their organizations.
- `revisore` sees all projects because the role is modeled as a global review function.
- Form Builder is visible only to `admin` and `fondazione`.
- `PermissionMatrix` adds per-module `canRead`, `canWrite`, `canValidate` and `canComment` checks on top of platform/org roles.
- `approvalChain` on `ModuleDefinition` can require multi-step review, such as `revisore` before `fondazione`; `admin` can bypass the chain.

---

## Module Engine architecture

The entity hierarchy is defined in `packages/shared/src/module-engine.types.ts`:

```
ModuleType → ModuleDefinition → ProjectTemplate → Project → ModuleInstance
```

| Entity | Role |
|---|---|
| `ModuleType` | Platform primitive (e.g. Form, Matrix, Budget). Defines the UI renderer and computational capabilities. |
| `ModuleDefinition` | Specific configuration of a ModuleType (e.g. "SWOT Analysis", "Project Budget"). Holds sections, fields, and validation rules as JSONB (AST). |
| `ProjectTemplate` | Ordered composition of ModuleDefinitions associated with a grant program. Defines ordering and enabled roles per phase. |
| `Project` | Concrete instance of a template created by a lead organization (capofila). Has a unique identity and defined timeline. |
| `ModuleInstance` | Materialization of data (`ModuleData`) within a single project, with audit log and validation state. |

`ModuleDefinition` objects hold the layout (sections + fields) serialized as JSONB.
Compiled data (`ModuleInstance.data`) is validated at runtime by `ModuleEngineService` against the corresponding definition — no schema migrations needed when grant requirements change.

Each field type in a definition can be: primitive (`text`, `number`, `date`), derived (`formula`), homogeneous list (`list`), or a reference to system entities (`ref: organization | user`).
Conditional visibility and cross-field validators are declared directly in the schema.

### ModuleInstance lifecycle

Module instances move through a small review state machine:

```
empty → draft → submitted → approved
                          ↘ rejected
                          ↘ needs_revision → draft → submitted
```

The workflow is shared by every module type. Transitions are triggered by submit/review actions in the frontend or by authorized reviewers. Each transition writes an `AuditEntry` with action, delta, author and timestamp.

| Status | Set by | Editability |
|---|---|---|
| `empty` | System, when a project creates module instances | Writable |
| `draft` | First data save, or resumed after requested changes | Writable |
| `submitted` | `capofila`, via submit | Read-only |
| `approved` | `fondazione`, `revisore` or `admin` | Read-only |
| `rejected` | `fondazione`, `revisore` or `admin` | Read-only |
| `needs_revision` | `fondazione`, `revisore` or `admin` | Writable |

Role timeline:

- `capofila` opens the module, edits draft data, handles requested changes, and submits when the content is ready.
- `partner` can fill and save drafts but cannot submit; formal submission to the Foundation stays with the lead organization.
- `fondazione`, `revisore` and `admin` see submitted modules in review mode and can approve, reject or request changes with an optional note.

Workflow endpoints:

| Method | Path | Effect |
|---|---|---|
| `PATCH` | `/module-engine/instances/:id/data` | Saves data and moves the instance to `draft`. |
| `POST` | `/module-engine/instances/:id/submit` | Submits the instance for review. |
| `POST` | `/module-engine/instances/:id/review` | Sets `approved`, `rejected` or `needs_revision`. |

`POST /review` body:

```json
{ "status": "approved | rejected | needs_revision", "note": "optional" }
```

Notifications use type `module_review` and are sent in-app and by email:

| Transition | Recipients | Method |
|---|---|---|
| To `submitted` | Active `fondazione`, `revisore` and `admin`, excluding the actor | `notifyModuleSubmitted()` |
| To `approved`, `rejected` or `needs_revision` | The submitter from the audit log, with active `capofila` and `partner` as fallback | `notifyModuleReview()` |

Audit actions are `update`, `submit`, `approve`, `reject` and `needs_revision`.

### Field types

`ModuleDefinition.layout.sections[].fields[]` supports:

| Type | Description |
|---|---|
| `text`, `textarea`, `rich_text` | String input with optional min/max length. |
| `number` | Numeric input with min/max/step/unit. |
| `date` | ISO date with optional min/max date. |
| `select`, `multiselect`, `radio` | Enum values from configured options. |
| `checkbox` | Boolean value. |
| `formula` | Derived value referencing other field IDs. |
| `list` | Repeatable group of sub-fields. |
| `ref` | Reference to a system entity: user, organization, project or module instance. |
| `file` | File attachment field. |

Conditional visibility and cross-field validators must be evaluated client-side for immediate feedback and server-side for data integrity.

### ModuleDefinition versioning

- ✅ Published definitions are classified as cosmetic / compatible / breaking.
- ✅ Compatible changes can update in place; breaking changes deprecate the previous definition version.
- ✅ `schemaVersion` is stored on instances when they are created.
- ☐ Runtime migration of existing instances after breaking schema changes is still planned.

### File uploads

The Documents renderer uploads files through:

```http
POST /module-engine/instances/:id/documents/:reqId/upload
```

Uploads use `multipart/form-data`. Files are saved to `UPLOADS_DIR`, which defaults to `<cwd>/uploads`. In development, Next.js proxies `/uploads/*` to the API. In production, point `UPLOADS_DIR` to a persistent volume and serve it directly with Nginx or equivalent at `/uploads/`.

---

## Module catalog

Nine primitive module types form the platform core. Each maps one or more legacy MATRIX 1.x sections to a configurable component.

| Module type | Legacy coverage | Key capabilities | Status |
|---|---|---|---|
| **Form** | Narrative sections, typed fields (Formulario B, Marketing Plan) | Repeatable groups, cross-field validators, conditional visibility | ✅ entity seeded |
| **Matrix** | Configurable quadrant grids (SWOT, BMC, Stakeholder Matrix) | Quantitative axes, risk color zones, NxM layout | ✅ entity seeded · ✅ renderer |
| **Indicators** | KPI monitoring over time with traffic-light thresholds | Baseline, target, periodicity, automatic trends | ✅ entity seeded · ✅ renderer |
| **Tree** | Multi-level hierarchies (Objectives → Actions → Activities) | Typed nodes, task generation, WBS integration | ✅ entity seeded · ✅ renderer |
| **Planning** | Schedule and Gantt visualization | Milestones, finish-to-start dependencies, monthly view | ✅ entity seeded · ✅ renderer |
| **Budget** | Financial management with variance calculations | Macro-items, implementing organizations, percentage constraints | ✅ entity seeded · ✅ renderer |
| **Documents** | File archive with upload checklist and notes | Versioning, taxonomic tags, PDF dossier generation | ✅ entity seeded · ✅ renderer |
| **Collaborative editor** | Real-time multi-user writing (Final Report) | Co-editing, smart blocks, inline comments on selection | ✅ entity seeded · ✅ renderer |
| **Financial transactions** | Actual cash-flow tracking and reconciliation | CSV/API import, automatic categorization | ✅ entity seeded · ✅ renderer |

---

## Collaboration & workflow features

| Feature | Description | Status |
|---|---|---|
| Comment threads | Contextual threads with inline `@mention` autocomplete on modules, document requirements, Form fields, KPI indicators, Tree tasks and projects; email notification sent to mentioned users | ✅ |
| In-app notifications | Mention notifications with unread counter and read/read-all actions in the sidebar | ✅ MVP |
| Real-time co-editing | Yjs CRDT with cursor awareness for the Collaborative Editor module | ✅ |
| Tree task tracking | Tree activity nodes → assignable work units with status, assignee and due date | ✅ |
| Operational Task Manager + collaborative diary | Cross-project task workspace, calendar/list views, diary entries, files/photos, milestone links | ☐ |
| Module approval workflow | Unified review flow for all modules: submit → approve / reject / needs_revision; audit log, in-app notifications, and transactional email (Mailpit locally, internal SMTP in production) | ✅ |
| Auto-generated Final Report | Assembled from data already entered across modules | ☐ |
| Immutable audit log | Each Module Engine write generates an `AuditEntry` (delta + author + timestamp) | ✅ |

### Tree task states

Leaf nodes in the Tree module have an operational task state independent from `ModuleInstance.status`; it is not an approval workflow.

```
todo → in_progress → done → todo
```

| Status | Label |
|---|---|
| `todo` | To do |
| `in_progress` | In progress |
| `done` | Done |

Any user with write access to the Tree module can cycle the task state, set `assignedTo`, set `dueDate`, and comment through the standard thread panel. The renderer stores task state in the node data inside `ModuleInstance.data` and shows progress counters for completed/total tasks.

---

## Admin tools

| Feature | Description | Status |
|---|---|---|
| ModuleDefinition admin | Admin interface for creating/editing `ModuleDefinition` objects (`/definitions`) | ✅ MVP |
| Visual drag-and-drop Form Builder | Palette, canvas, properties panel and live preview for schema-driven module configuration | ✅ |
| Form Builder — version manager | Cosmetic / compatible / breaking classification on publish; auto version bump; deprecation of previous | ✅ |
| JSON mode, import/export and changelog | Controlled schema editing, reusable JSON definitions and version changelog | ☐ |

---

## Planning

Current planning lives in GitHub issues and is organized around a controlled go-live by **19 November 2026**.

### Completed foundation

- Authentication, users, organizations, role guards and organization membership.
- Project and template foundations, with automatic ModuleInstance creation from templates.
- Nine seeded ModuleTypes and renderers for the main module families.
- File upload for the Documents module.
- Yjs real-time co-editing for the Collaborative Editor.
- Contextual comments, mentions and in-app notifications MVP.
- Unified module review workflow with audit log and transactional email.
- ModuleDefinition version classification and publish lifecycle.

### Go-live controlled scope

- Visual drag-and-drop Form Builder and safer definition editing.
- JSON mode, import/export and changelog for ModuleDefinitions.
- Runtime handling for instances whose schema version changed.
- Final Report and PDF dossier generation.
- 2FA and go-live security hardening.
- Workspace, project dashboard and timeline.
- Operational Task Manager and collaborative diary.
- Read-only access to essential MATRIX 1.x historical data.
- Backup/restore, object storage and async jobs for long-running tasks.

### Optional before go-live

- Smart blocks in the collaborative editor.
- Digital signature integration.
- Full-text search and broader historical indexing.
- Excel/Word/JSON exports beyond the minimum needed by the pilot.

### Post go-live

- Deep accounting API integrations.
- Integration Gateway for external systems.
- Full legacy migration and irreversible shutdown of MATRIX 1.x.


```mermaid
erDiagram
        text description
        varchar icon
        boolean supports_realtime
        boolean supports_comments
    }

    notifications {
        uuid id PK
        uuid userId FK
        uuid actorId FK
        varchar type
        varchar targetType
        varchar targetId
        uuid threadId
        uuid commentId
        varchar title
        text body
        timestamp readAt
        timestamp createdAt
    }

    organizations {
        uuid id PK
        varchar name
        varchar slug
        text description
        timestamp createdAt
        timestamp updatedAt
    }

    project_templates {
        uuid id PK
        varchar name
        varchar slug
        text description
        jsonb structure
        timestamp createdAt
        timestamp updatedAt
    }

    projects {
        uuid id PK
        varchar name
        varchar slug
        text description
        uuid template_id FK
        uuid lead_organization_id FK
        varchar status
        timestamp createdAt
        timestamp updatedAt
    }

    user_organizations {
        uuid id PK
        uuid user_id FK
        uuid organization_id FK
        varchar role
        timestamp createdAt
    }

    users {
        uuid id PK
        varchar email
        varchar password_hash
        varchar first_name
        varchar last_name
        varchar role
        boolean is_active
        timestamp createdAt
        timestamp updatedAt
    }

    users ||--o{ comment_threads : creates
    users ||--o{ comments : writes
    users ||--o{ notifications : receives
    users ||--o{ notifications : triggers

    comment_threads ||--o{ comments : contains

    organizations ||--o{ projects : owns
    project_templates ||--o{ projects : templates

    users ||--o{ user_organizations : member
    organizations ||--o{ user_organizations : includes

    module_definitions ||--o{ module_instances : instantiates

```
