# Security and Reliability — Training Management System

> This document describes the security controls and reliability patterns implemented in the training management system. All statements are based on direct code inspection.
>
> **Important:** This case study does not claim the system is production-hardened for all threat models or compliance frameworks. The controls described reduce attack surface and reflect standard Laravel security practices. Additional production hardening may still be required depending on the deployment environment.

---

## Authentication

### Web (Session-Based)
Standard Laravel session authentication is used for all staff and administrative users accessing the Filament panel. The session driver is configured as `database` (visible in `.env.example`), which stores session data server-side rather than in client-side cookies.

Session lifetime is configured generously (suitable for operational users who remain logged in for extended periods). HTTPS enforcement is assumed at the infrastructure level and is not enforced at the application layer in the reviewed configuration.

The `User` model implements Filament's `FilamentUser` contract. The `canAccessPanel()` method restricts panel access to users with `active = true` — deactivated users cannot log in even if their credentials are valid.

### API (Sanctum Token-Based)
The REST API (`/api/v1`) is protected by Laravel Sanctum. API clients authenticate via a login endpoint that returns a token. All subsequent API requests must include this token in the `Authorization: Bearer` header. The `auth:sanctum` middleware enforces this on the protected route group.

The Sanctum personal access token table was introduced via a dedicated migration in 2026, confirming the feature is fully integrated.

---

## Role-Based Access Control (RBAC)

The system implements **ten distinct user roles** as integer constants defined on the `User` model:

| Constant | Value | Label (translated) |
|----------|-------|--------------------|
| `ROLE_ADMIN` | 1 | System Administrator |
| `ROLE_DEPARTMENT` | 2 | Department Head |
| `ROLE_SECTION` | 3 | Section Head |
| `ROLE_MOH` | 4 | Ministry of Health |
| `ROLE_COLLEGE` | 5 | College Supervisor |
| `ROLE_HOA` | 6 | Administrative Manager |
| `ROLE_HOM` | 7 | Medical Manager |
| `ROLE_GTM` | 8 | General Training Manager |
| `ROLE_MONITOR` | 9 | Monitor (read-only) |
| `ROLE_ASSISTANT_TRAINING_MANAGER` | 10 | Assistant Training Manager |

Role helpers (`isAdmin()`, `isMinistry()`, `isSectionHead()`, etc.) are defined on the `User` model and used consistently throughout policies and scopes.

### Organizational Binding
Several roles are scoped to a specific organizational unit:
- `ROLE_SECTION` → bound to a single `Section`
- `ROLE_DEPARTMENT` → bound to a single `Department`
- `ROLE_HOA` → bound to an `Administrative` area
- `ROLE_HOM` → bound to an `Administrative` area (medical head)
- `ROLE_COLLEGE` → bound to a `College`
- `ROLE_MOH` → optionally bound to a `Department` (linked MOH department)
- `ROLE_ASSISTANT_TRAINING_MANAGER` → bound to one or more `Department` records via a many-to-many relationship

A user not linked to an active organizational entity will receive empty query results due to the `whereRaw('1 = 0')` fallback in `Application::scopeForUser()`.

---

## Authorization Policies

Eight Laravel policy classes are registered via `AuthServiceProvider` and applied throughout the application:

| Policy | Protects |
|--------|---------|
| `ApplicationPolicy` | Application CRUD, absorption paper download, cancellation |
| `SectionPolicy` | Section creation, editing, enabling |
| `TraineePolicy` | Trainee record viewing and editing |
| `AdministrativePolicy` | Administrative area management |
| `CollegePolicy` | College management |
| `DepartmentPolicy` | Department management |
| `InstitutionPolicy` | Institution management |
| `UserPolicy` | User management |

The `ApplicationPolicy` is the most complex, implementing:
- `viewAny` — open to all authenticated users (list-level visibility is controlled by query scope)
- `view` — role and organizational binding checks for each role
- `create` — restricted to Admin, College Supervisor (with institution/college flags enabled), and Ministry users
- `update` — restricted to Admin, GTM, ATM (scoped), College Supervisor (limited statuses), Ministry (limited statuses)
- `delete` — Admin only
- `downloadAbsorptionPaper` — Ministry only, for PRACTICE type, in specific statuses
- `cancel` — Admin, GTM, ATM (scoped), College Supervisor, Ministry; only on STARTED_TRAINING

### Database-Level Scoping
`Application::scopeForUser($user)` is applied at the ORM query level. This prevents any possibility of a UI bug exposing unauthorized records — even if a controller accidentally omitted an authorization check, the query scope ensures the database only returns permitted records.

The scope handles all ten roles with explicit conditions for each, falling back to `whereRaw('1 = 0')` for any role not explicitly handled.

---

## Rate Limiting

- **Public trainee routes** (`/welcome`, `/welcome/form`) are protected by `throttle:60,1` middleware — 60 requests per minute per IP
- **Authenticated routes** share the same rate limiter configuration
- **WhatsApp queue job** applies `RateLimited('whatsapp')` middleware at the job level, controlling the rate of outbound API calls independently of the web request layer

---

## CSRF Protection

- All web routes use Laravel's built-in CSRF middleware
- API routes (`/api/*`) use Sanctum token authentication and do not require CSRF tokens
- Webhook routes (`/telegram/webhook`, `/whatsapp/webhook`) use custom CSRF exemption via `VerifyCsrfForApi` middleware — these are stateless endpoints receiving calls from external systems
- The `VerifyCsrfToken` middleware excludes webhook paths explicitly

---

## Secure File Storage

- Uploaded application letter files are stored on the **private (local) disk** via Spatie Media Library, not in the publicly accessible `public` storage
- Download is gated by the authenticated route `GET /applications/{application}/download-trainee-files`, which requires login
- Absorption paper PDFs are generated per-request and saved temporarily to the public storage for streaming, but filenames are derived from application IDs rather than user-supplied input

---

## Input Validation

- Livewire form components validate fields before submission using Laravel validation rules
- The API layer uses dedicated request validation for the MOH sync endpoint
- The `SubmitApplicationAction` performs secondary validation (section capacity check) after initial form validation

---

## Webhook Security

- The **Telegram webhook** accepts any POST to `/telegram/webhook`. Authorization is enforced at the application level: the `TelegramSubscriber` record must be active, and a separate `/auth <code>` command must be used to activate a subscriber. All commands are blocked for non-active subscribers.
- The **WhatsApp webhook** uses a standard GET verification flow (`verify`) that the WhatsApp platform calls before activating the webhook.
- Webhook routes are excluded from CSRF verification, which is standard practice for external webhooks. The Telegram webhook does not implement additional IP allowlisting in the reviewed code.

---

## Output Safety

- Filament and Livewire handle output escaping through their respective template systems
- Blade templates use `{{ }}` escaping by default
- User-supplied content is not rendered as raw HTML except where explicitly intended with `{!! !!}` (standard Laravel practice)

---

## Operational Security Controls

### User Impersonation
- Only users with `ROLE_ADMIN` can impersonate other users (`canImpersonate()` returns true only for admin)
- Admin users cannot be impersonated (`canBeImpersonated()` returns false for admin)
- Impersonation is provided by the `stechstudio/filament-impersonate` package

### User Activation
- Users must have `active = true` to access the Filament panel
- Deactivated users are excluded from all notification queries via the `scopeActive()` scope
- Deactivated users are excluded from organizational head selections

### Maintenance Mode
- A custom `CheckMaintenanceMode` middleware reads `TrainingSettings::is_maintenance_mode` on each request
- When enabled, users whose role is in `maintenance_roles` are redirected to the maintenance page
- The middleware always allows access to the maintenance page itself, logout, Livewire requests, and Filament assets to prevent redirect loops

---

## Reliability Patterns

### Database Transactions
- `SubmitApplicationAction::execute()` wraps trainee creation, application creation, and file upload reference in a single `DB::transaction()` — if any step fails, no partial records are created
- `SyncMohApplicationAction::execute()` wraps the entire MOH sync (trainee upsert + application upsert) in a `DB::transaction()`

### Soft Deletes
- The `Application` and `User` models use `SoftDeletes` — deleted records are marked with `deleted_at` rather than permanently removed, preserving audit history
- Telegram monitors application deletions and restorations via model events

### Queue-Backed External Calls
- All outbound HTTP calls to WhatsApp and Telegram are dispatched asynchronously via queued jobs, preventing failed external API calls from blocking or crashing web requests
- The `SendWhatsAppMessageJob` retries up to 3 times with a 60-second release delay

### Graceful Failure in Model Events
- All Telegram and WhatsApp calls in `Application::booted()` model event hooks are wrapped in individual try/catch blocks
- Failures are logged but do not interrupt the application lifecycle event

### Backup Retention
- Daily automated SQL backups are retained for 10 days and then automatically purged
- Backup filenames follow a consistent format allowing date-based retention logic without external tools

### Stale Data Management
- A daily command marks applications with no updates for 7+ days as `STATUS_UNKNOWN`, preventing stale applications from indefinitely blocking trainee re-application eligibility

### Export Cleanup
- The custom `DownloadExportController` overrides Filament's export download route to delete the export file from storage immediately after the download completes, preventing accumulation of exported data files

---

## Public Disclosure Boundaries

This public case study intentionally omits the following categories of information:

| Category | Reason for Omission |
|----------|-------------------|
| Source code | Proprietary; belongs to the institution |
| Database schema details | Contains internal naming that could expose organizational structure |
| Environment variables and credentials | Security-sensitive |
| Telegram chat IDs and access codes | Operational security |
| WhatsApp phone number IDs and access tokens | API credentials |
| Webhook URLs | Could enable targeted attacks |
| Internal IP addresses and hostnames | Infrastructure confidentiality |
| Real trainee names, national IDs, phone numbers | Personal data protection |
| Real application records | Operational and personal data |
| Uploaded documents | Confidential institutional files |
| Internal user accounts and roles | Operational security |
| Real error logs and stack traces | Could expose implementation details |
| Deployment scripts and configurations | Infrastructure-sensitive |

All screenshots included in this case study use sanitized dummy data. Real personal information is never shown in public materials derived from this system.

The source code remains the intellectual property of the institution. This case study presents only architectural patterns, workflow logic, and technology choices at a level appropriate for public professional visibility.