# Technical Decisions — Training Management System

> This document records important engineering decisions observed in the production repository. Each decision is described with context, rationale, trade-offs, and observed limitations. Only decisions directly verifiable from the code are included.

---

## Decision 1: Laravel as the Core Framework

**Context:** The system needed to manage a multi-entity domain (institutions, colleges, departments, sections, trainees, applications) with complex relationships, role-based access, scheduling, and external integrations — all while being maintainable by a small team.

**Decision:** Laravel 12 (PHP 8.3) was chosen as the primary framework.

**Why it was suitable:**
- Laravel's Eloquent ORM provides expressive model relationships matching the domain's hierarchical structure
- The scheduler, queue system, and Artisan command framework cover all automation requirements natively
- Laravel Sanctum provides API token authentication without additional infrastructure
- The Laravel ecosystem (Sanctum, Spatie packages, Filament) dramatically reduces boilerplate

**Trade-offs:**
- Monolithic architecture means the web app, API, and background jobs share the same codebase and deployment unit
- Performance scaling requires horizontal scaling of the PHP process pool and queue workers rather than microservice-level scaling

**Limitations observed:**
- No HTTP client abstraction layer; external API calls (Telegram, WhatsApp) use `Http::` directly in service classes, making mocking in tests more involved

---

## Decision 2: Filament 4 for the Administrative Panel

**Context:** The system requires a rich, role-aware administrative UI covering approximately 11 Filament resources, 4 pages, 10 dashboard widgets, and custom actions. Building this from scratch would have been very costly.

**Decision:** Filament 4 was adopted for the entire administrative interface.

**Why it was suitable:**
- Filament provides a complete admin panel framework with CRUD resources, actions, widgets, tables, and forms — with minimal boilerplate
- Filament integrates natively with Laravel policies for authorization
- Filament's export and import functionality (Filament Excel/Export) was used directly, with customizations layered on top
- Filament's plugin ecosystem provided settings page integration (`spatie-laravel-settings-plugin`) and user impersonation (`stechstudio/filament-impersonate`)

**Trade-offs:**
- Filament's opinionated structure means UI customizations sometimes require overriding internal defaults (e.g., the custom `DownloadExportController` overrides Filament's default export download to add auto-delete behavior)
- Filament's version lock (v4) means package compatibility must be monitored closely

**Limitations observed:**
- The `ManageTrainingSettings` page is very large (~49KB), suggesting it may benefit from being split into multiple settings pages or tabs in the future
- Dashboard widgets are defined without lazy loading; a high number of widgets on a low-powered server could increase page load time

---

## Decision 3: Livewire for the Public Trainee Form

**Context:** The public application form requires reactive behavior: cascading dropdowns (administrative area → department → section), real-time eligibility feedback, and conditional field visibility based on training type selection.

**Decision:** Livewire 3 (used via Filament, also as a standalone component for the public form) was chosen over a JavaScript SPA or vanilla JS.

**Why it was suitable:**
- Livewire allows reactive, stateful forms with server-side validation without writing a separate frontend API
- The backend eligibility logic, capacity checks, and form data loading can be called directly from the Livewire component without an intermediate API layer
- Livewire's `wire:model` binding handles cascading dropdown state naturally

**Trade-offs:**
- Each Livewire reactive action (dropdown change, eligibility check) generates an HTTP round-trip; on slow connections this is slower than a purely client-side SPA
- The `TraineeForm` component is large (~29KB), containing substantial logic; it could be refactored using Livewire concerns or child components

**Limitations observed:**
- Public routes are rate-limited at `throttle:60,1`; high-volume submission periods (e.g., the start of a new academic year) may need rate limit tuning

---

## Decision 4: Eloquent Query Scopes for Data-Level RBAC

**Context:** Ten user roles with different organizational bindings need to see different subsets of the same `Application` table. Enforcing this purely at the controller/UI level risks data leakage if a control point is missed.

**Decision:** `Application::scopeForUser($user)` was implemented as an Eloquent query scope that enforces visibility rules at the database query level.

**Why it was suitable:**
- Any query using `->forUser($user)` is guaranteed to only return permitted records, regardless of which part of the application initiated the query
- The scope handles all ten roles explicitly; unknown or unlinked users receive `whereRaw('1 = 0')` — an empty result set rather than full access
- This is a defense-in-depth measure supplementing policy-level checks

**Trade-offs:**
- The scope is complex (~120 lines); changes to organizational structure require updating the scope along with policies
- The scope is applied manually by callers (Filament resource `modifyQueryUsing` or controller); it is not a global scope, so callers must remember to apply it

**Limitations observed:**
- Not applied as a global scope; relies on consistent use across all list endpoints

---

## Decision 5: Laravel Sanctum for API Authentication

**Context:** The Ministry of Health's external system requires machine-to-machine API authentication. A simple token-based scheme was needed without the complexity of OAuth 2.0.

**Decision:** Laravel Sanctum was chosen for API token authentication.

**Why it was suitable:**
- Sanctum provides per-user API tokens with minimal configuration
- Works within the existing Laravel user model and middleware stack
- No additional OAuth server infrastructure is required

**Trade-offs:**
- Sanctum tokens do not have built-in expiry in the used configuration; token rotation must be managed at the application level
- No scopes or fine-grained permissions on tokens — all tokens grant the same level of API access for the authenticated user's role

**Limitations observed:**
- Token management (revocation, listing) is not exposed via the admin panel in the reviewed code; this is a potential operational gap

---

## Decision 6: PHP-Native Database Backup (No mysqldump)

**Context:** The production hosting environment did not provide `mysqldump` as an accessible binary. The Spatie Laravel Backup package was installed but would require `mysqldump`. A pure-PHP alternative was needed.

**Decision:** A custom `BackupExport` class was implemented that generates SQL `INSERT` statements by querying each table via PDO/Eloquent and writing them to a `.sql` file.

**Why it was suitable:**
- No binary dependencies; works on any shared hosting environment with PHP and MySQL access
- The output is a valid `.sql` file that can be imported with standard MySQL tools
- Integrated with the Laravel scheduler and admin download route without any infrastructure changes

**Trade-offs:**
- PHP-based export is slower than `mysqldump` for large datasets; it also loads all rows into memory per table
- Does not support streaming compression (gzip) natively; the output file size grows with the dataset
- Does not replicate all `mysqldump` features (views, stored procedures, triggers)

**Limitations observed:**
- For very large tables, the PHP-based exporter may exhaust memory limits; a chunked approach would be advisable if datasets grow substantially
- The `spatie/laravel-backup` package remains installed but is not the active backup mechanism; this may cause confusion

---

## Decision 7: Queue-Backed WhatsApp Notifications with Rate Limiting

**Context:** WhatsApp Cloud API has rate limits per phone number ID. Status change events in the system can trigger notifications for multiple trainees simultaneously (e.g., when a bulk action changes many applications). Synchronous API calls would block web requests and could exceed rate limits.

**Decision:** All WhatsApp messages are dispatched via a queued `SendWhatsAppMessageJob` with a `RateLimited('whatsapp')` middleware applied at the job level.

**Why it was suitable:**
- Web requests return immediately; WhatsApp delivery happens asynchronously
- Rate limiting is enforced at the queue middleware level, preventing API throttling errors
- Jobs retry up to 3 times with a 60-second delay, handling transient API failures automatically

**Trade-offs:**
- WhatsApp delivery is not real-time; trainees receive messages after queue processing
- Queue worker must be running; if the queue worker dies, messages are delayed until it restarts
- Failed jobs (after 3 retries) are moved to the failed jobs table; no automatic alerting on failed WhatsApp deliveries is implemented in the reviewed code

**Limitations observed:**
- The `whatsapp` rate limiter configuration is referenced but not seen explicitly defined in the reviewed files; it would need to be configured in `AppServiceProvider` or `RouteServiceProvider`

---

## Decision 8: Runtime Settings via Spatie Laravel Settings

**Context:** The system's operational requirements change frequently — the form may be closed during off-hours, notification features may be toggled per event type, and maintenance windows need to be scheduled. Hard-coding these as environment variables would require redeployments.

**Decision:** `spatie/laravel-settings` was adopted, backed by the `settings` database table. The `TrainingSettings` class defines ~30 typed settings properties.

**Why it was suitable:**
- Settings are editable from the Filament admin panel at runtime without redeployment
- Typed properties provide IDE support and prevent misconfiguration
- Settings can be cached (Spatie supports caching) to avoid per-request database reads

**Trade-offs:**
- Settings are stored in the database; if the database is unavailable, settings cannot be read
- Adding a new setting requires a migration to add the default value to the settings table
- The `TrainingSettings` class now has ~30 properties; future growth may warrant splitting into multiple settings groups

**Current limitations observed:**
- Some settings (e.g., `enable_change_password`, `ai_search_enabled`) appear to be recent additions with dedicated migrations; the migration file shows this is an evolving API

---

## Decision 9: Telegram for Operational Monitoring

**Context:** Administrators needed a low-friction way to monitor system events (new applications, status changes, errors) without checking the admin panel constantly. Email was considered too slow; SMS was not configured.

**Decision:** A Telegram bot was implemented with two operating modes (webhook and long-polling), a subscriber authentication system using a configurable access code, and a custom `TelegramLoggerHandler` that forwards Laravel error logs to Telegram.

**Why it was suitable:**
- Telegram is free, has a well-documented Bot API, and supports rich HTML-formatted messages
- The bot can be operated without dedicated server infrastructure (webhook mode) or with a simple Artisan process (long-polling)
- Per-subscriber preference toggles (activities, errors) reduce notification noise

**Trade-offs:**
- Telegram is a consumer messaging platform, not enterprise monitoring infrastructure; it lacks SLA guarantees
- The custom access code authentication is simpler than OAuth but must be kept confidential; if leaked, unauthorized subscribers can access system information
- The bot sends some personal data (trainee names, application IDs) to Telegram subscribers when `/find` is used; this has privacy implications that must be understood in the deployment context

**Limitations observed:**
- Telegram webhook does not validate the `X-Telegram-Bot-Api-Secret-Token` header (optional Telegram security feature); adding this would provide additional webhook authenticity verification

---

## Decision 10: Separate Training Types with Type-Specific Workflows

**Context:** Two fundamentally different training categories exist: university practicum (requires university supervisor confirmation) and Ministry-track clinical training (requires Ministry of Health confirmation). These have different approval chains, different external partners, and different document requirements.

**Decision:** Training type is encoded as an integer constant (`UNIVERSITY = 1`, `PRACTICE = 2`) on the `Application` model. All workflows, notifications, policies, and query scopes branch on this value.

**Why it was suitable:**
- A single unified application model with type branching is simpler than two separate models and two separate tables
- Status lifecycle is identical for both types; only the external confirmation step and notification recipients differ
- Ministry API integration only accepts PRACTICE applications, which is enforced by the `scopeForUser()` Ministry branch

**Trade-offs:**
- Branching logic on `training_type` appears in multiple places (model, policies, scopes, notifications, controllers); adding a third training type in the future would require updating all these locations
- The runtime settings page has separate toggles for each training type (`enable_training_type_practice`, `enable_training_type_university`), which correctly supports enabling/disabling each independently

**Possible future improvements:**
- Introduce an enum (PHP 8.1 native enum) for training type to replace the integer constants and benefit from type safety and IDE autocompletion — the commented-out enum imports in some files suggest this was planned