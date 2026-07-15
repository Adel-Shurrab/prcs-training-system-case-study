# Training Management System — Public Case Study

> **Production Training Management System** built with Laravel 12, Filament 4, Livewire 3, and MySQL to manage trainee applications, multi-role review workflows, external integrations, document generation, and lifecycle automation for a healthcare organization.

![PHP 8.3](https://img.shields.io/badge/PHP-8.3-blue?logo=php)
![Laravel 12](https://img.shields.io/badge/Laravel-12-red?logo=laravel)
![Filament 4](https://img.shields.io/badge/Filament-4-orange)
![Livewire 3](https://img.shields.io/badge/Livewire-3-purple)
![MySQL](https://img.shields.io/badge/MySQL-8-blue?logo=mysql)

---

> **📌 Case Study Notice**
>
> This repository contains a public technical case study. The production source code, database, environment configuration, and operational data remain private and are not published here.
> Screenshots in `assets/` use sanitized or anonymized data only.

---

## Overview

This is a production backend system managing the full lifecycle of training placements — from initial public application through administrative review, external entity confirmation, active training, and completion — for a healthcare institution operating multiple departments, sections, and affiliated colleges.

The system replaced a paper-based and spreadsheet-driven process with a structured, role-aware, automated platform. It handles two distinct training types: university-sponsored practicum training (requiring university confirmation) and licensing-track clinical training (requiring Ministry of Health confirmation).

---

## Operational Problem

Before this system, training coordinators managed applications manually across disconnected channels. There was no single source of truth for application status, capacity was tracked in spreadsheets prone to errors, external partners (universities and the Ministry of Health) communicated through informal channels, and there was no automation for lifecycle transitions or end-of-training notifications.

The system addresses these problems by:

- Providing a public-facing digital application form for trainees
- Enforcing a structured, status-driven review lifecycle
- Enforcing section capacity limits automatically
- Integrating with external institutional partners (Ministry of Health) via REST API
- Automating status transitions, notifications, and scheduled lifecycle checks
- Giving each role a scoped, read-only or read-write view appropriate to their position

---

## My Role and Contributions

I contributed to the development and maintenance of this system as a Laravel backend developer. My work included:

- **Public application flow** — Implementing the Livewire-based public trainee form with eligibility checks, duplicate detection, date-of-birth verification, dynamic field dependencies, section capacity validation, and transactional record creation via a dedicated action class
- **Application status service** — Building a centralized service to handle eligibility checking, cache status lookups, manage re-application policy, and generate accurate user-facing messages based on current system settings
- **Ministry of Health API integration** — Implementing the Sanctum-authenticated REST endpoint and sync action that accepts Ministry-originated applications, normalizes phone numbers, resolves sections by department linkage, calculates end dates, and upserts application records transactionally
- **Filament administrative resources** — Contributing to Filament resources for applications, trainees, departments, sections, colleges, and users, including role-scoped list filtering
- **Role-based access control** — Implementing the multi-role policy system covering 10 distinct user roles with scoped visibility and action restrictions
- **Telegram monitoring integration** — Implementing the Telegram monitor service, webhook controller, and long-polling command for real-time operational monitoring, authenticated bot commands, application lookup, and error log forwarding
- **WhatsApp notification integration** — Implementing the notification service and queued job to deliver status change messages to trainees via the WhatsApp Cloud API, with queue-backed rate-limited delivery
- **Scheduled lifecycle commands** — Implementing automated daily processes including waiting-list promotion, training end detection, end-date reminders, and stale application marking
- **PHP-native database backup** — Implementing backup export, backup command, and cleanup command to produce SQL dumps without requiring `mysqldump` on restricted hosting environments, with automatic retention cleanup
- **Runtime settings system** — Contributing to the settings class allowing administrators to toggle form availability, WhatsApp and Telegram features, re-application policies, maintenance mode, and AI-assisted search without code deployments
- **Document generation** — Implementing PDF absorption paper generation using Laravel mPDF with Arabic language and RTL support, policy-gated by role and application status
- **Notification system** — Implementing the notification classes and routing logic that distributes Filament notifications to the correct recipients at each status transition
- **Maintenance mode middleware** — Implementing database-driven, role-scoped maintenance mode toggling configurable from the admin panel
- **Export functionality** — Contributing to application export via Filament, Excel import template download, and the custom export download controller that auto-deletes files after download

---

## Core Capabilities

### Public Trainee Interface
- Public landing page with form availability toggle (runtime-controlled)
- Eligibility check before form rendering: national ID lookup, date-of-birth verification, and duplicate detection
- Dynamic form with cascading dropdowns (administrative → department → section) driven by Livewire reactive state
- Two training types: university practicum and Ministry-track clinical training
- Section capacity enforcement at submission time
- Optional application letter file upload (stored on private disk via Spatie Media Library)
- Existing application prefill for editable statuses (New, Initial Approval)

### Administrative Panel (Filament)
- Filament 4 administrative interface for all staff roles
- Role-scoped application and trainee lists — each user role sees only the records and statuses relevant to their position
- Application creation and status management for Admin, General Training Manager, College Supervisor, and Ministry users
- Application import via Excel with a downloadable template
- Filament Export for application lists with auto-cleanup of exported files after download
- Dashboard widgets for capacity overview, recent applications, trainees finishing soon, and per-department statistics
- User management with role assignment, activation, and user impersonation (admin-only)
- Runtime settings page covering all operational toggles
- Internal mailbox for inter-role messaging

### Application Lifecycle
Ten defined statuses with controlled transitions:

| Status | Description |
|--------|-------------|
| New | Submitted, pending GTM/Admin review |
| Initial Approval | Approved by GTM/Admin, awaiting external confirmation |
| Confirmation | Confirmed by university or Ministry |
| Waiting List | Confirmed but section is at capacity |
| Started Training | Actively training |
| Ended Training | Training period completed |
| Rejected | Rejected by reviewers |
| Dropped | Trainee withdrew |
| Unknown | Stale application marked by automated process |
| Cancelled | Admin-cancelled during active training |

### External Integrations
- **Ministry of Health REST API** — Sanctum-authenticated endpoint accepting application submissions from the Ministry's system, resolving the correct department and section, and upserting trainees and applications
- **Telegram monitoring bot** — Webhook-based and long-polling bot for real-time application activity alerts, error log forwarding, daily summary reports, and `/find` lookup by application ID
- **WhatsApp Cloud API** — Queue-backed notification delivery for trainees at key status transitions (initial approval, training start, training end, and upcoming end reminders)

### Automation and Scheduling
Six scheduled commands run on the Laravel scheduler:

| Command | Schedule | Purpose |
|---------|----------|---------|
| `app:start-waiting-applications` | Daily | Promote waiting-list applications to Started Training when capacity allows |
| `app:end-training` | Daily at 00:01 | End training for applications past their end date |
| `app:check-training-end-dates` | Daily at 09:00 | Send WhatsApp reminders X days before training ends |
| `app:database-backup` | Daily at 10:00 | Create PHP-native SQL dump backup |
| `app:clean-old-backups` | Every 10 days | Delete backup files older than 10 days |
| `telegram:daily-report` | Configurable | Send daily operational summary to Telegram |

---

## Main Users and Roles

The system implements ten distinct user roles, each with scoped visibility and a defined set of permitted actions:

| Role | Description |
|------|-------------|
| System Administrator | Full access; user management; impersonation |
| General Training Manager | Full application management across all sections |
| Assistant Training Manager | Application management for assigned departments only |
| Head of Administrative | Scoped to active applications in their administrative area |
| Head of Section | Scoped to active applications in their section |
| Head of Department | Scoped to active applications in their department |
| Medical Manager | Scoped to medical-department applications in their administrative area |
| Ministry of Health | Scoped to practice-type applications; API integration |
| College Supervisor | Scoped to university-type applications from their institution |
| Monitor | Read-only access across all applications |

---

## Key Workflows

### 1. Public Trainee Application
1. Trainee visits the public landing page
2. System checks if the form is enabled (runtime setting)
3. Trainee selects training type, enters national ID and date of birth
4. Application status service checks eligibility: duplicate detection, DOB verification, re-application policy
5. If eligible, trainee completes the full form (administrative → department → section cascade, educational details, training hours)
6. Submit action validates section capacity, then creates trainee and application records within a database transaction
7. Application created with status **New**; Telegram notification fired; GTM/Admin notified via Filament notification

### 2. Administrative Review
1. GTM or Admin reviews New applications
2. Application may be moved to **Initial Approval**, triggering WhatsApp notification to trainee
3. If university type: College Supervisor receives notification and can confirm
4. If practice type: Ministry of Health receives notification; Ministry user can confirm via admin panel or via the REST API endpoint
5. On confirmation, status moves to **Confirmation**
6. GTM/Admin moves application to **Waiting List** or **Started Training** depending on section capacity

### 3. Waiting List Promotion (Automated)
1. `app:start-waiting-applications` runs daily
2. For each waiting-list application whose start date has arrived:
   - If section has capacity: status updated to **Started Training**
   - If section is full: start date and end date recalculated based on earliest finishing application

### 4. Training End (Automated)
1. `app:end-training` runs daily at 00:01
2. Applications in **Started Training** with end date in the past are updated to **Ended Training**
3. WhatsApp notification dispatched to trainee if enabled in settings

### 5. Ministry of Health API Synchronization
1. Ministry system authenticates via `POST /api/v1/auth/login` (Sanctum token)
2. Ministry system submits trainee data to `POST /api/v1/moh/applications`
3. Action validates section ownership, normalizes phone number, upserts trainee and application at **Waiting List** status with calculated end date

### 6. Document Generation (PDF)
1. Authorized Ministry user requests absorption paper for a confirmed practice application
2. Policy verifies role and application status
3. Controller generates RTL Arabic PDF via Laravel mPDF, saves to storage, and streams or downloads

### 7. Database Backup
1. `app:database-backup` invokes the PHP-native backup export
2. SQL dump is generated (no `mysqldump` binary required) and stored on the private disk
3. `app:clean-old-backups` deletes files older than 10 days on its schedule

---

## Architecture Overview

```
Public Web (Livewire)  →  Application Layer (Actions / Services)  →  Eloquent ORM  →  MySQL
                                        ↕
Admin Panel (Filament)  →  Policies / Authorization  →  Eloquent ORM
                                        ↕
REST API (Sanctum)  →  Actions  →  Eloquent ORM
                                        ↕
Queue Workers  ←  Jobs  ←  WhatsApp Cloud API
                                        ↕
Telegram Webhook / Long-Poll  ←  TelegramMonitorService
                                        ↕
Scheduler  →  Console Commands  →  Models / Notifications
```

See [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) for full diagrams.

---

## Technology Stack

| Layer | Technology | Version |
|-------|-----------|---------|
| Language | PHP | 8.3 |
| Framework | Laravel | 12 |
| Admin UI | Filament | 4 |
| Interactive forms | Livewire | 3 (via Filament) |
| Frontend build | Vite + TailwindCSS | 4 |
| Database | MySQL | 8+ (compatible) |
| API authentication | Laravel Sanctum | * |
| Background jobs | Laravel Queue (database driver) | — |
| PDF generation | Laravel mPDF | — |
| Spreadsheet | PhpSpreadsheet | 5.3 |
| File storage | Spatie Media Library | — |
| Runtime settings | Spatie Laravel Settings | — |
| External messaging | WhatsApp Cloud API (Meta) + Telegram Bot API | — |
| AI search | Laravel AI (installed; feature-flagged) | 0.6 |
| Impersonation | Filament Impersonate | — |

---

## Integrations

### Ministry of Health REST API
A versioned REST API (`/api/v1`) protected by Laravel Sanctum. The Ministry's system authenticates with credentials and receives an API token. Subsequent calls submit trainee and training placement data that is synchronized into the local system. The API validates department ownership, enforces section activity, normalizes phone numbers, and returns application and trainee identifiers.

### Telegram Bot
Two integration modes are supported: webhook (HTTP push from Telegram) and long-polling (via Artisan command). The bot authenticates operators via a configurable access code, then provides commands for application lookup, live summary, daily report triggering, and per-subscriber notification preference toggles. Error logs from the Laravel logging stack are forwarded to Telegram via a custom log handler.

### WhatsApp Cloud API (Meta)
Outbound notifications only. Messages are dispatched via a queued job with rate-limiting middleware. The service sends notifications on: initial approval, training start, training end (on status change), and upcoming end reminders (via scheduled command). All notification types are independently toggle-able from the runtime settings page.

---

## Security and Access Control

- **Authentication** — Standard Laravel session-based authentication for web users; Sanctum token authentication for API clients
- **Role-Based Access Control** — Ten user roles implemented as integer constants on the User model, with helper methods used throughout policies and scopes
- **Policies** — Eight Laravel policy classes enforce per-action authorization
- **Query scopes** — The `Application::scopeForUser()` scope restricts database queries at the ORM layer to records the requesting user is permitted to see
- **Rate limiting** — Public routes (trainee form) are rate-limited; WhatsApp delivery uses Laravel's rate-limited queue middleware
- **CSRF protection** — All web routes use Laravel's CSRF middleware; API routes use Sanctum; webhook routes have custom CSRF handling
- **Secure file access** — Application letter uploads are stored on the private disk; download routes are protected by authentication and policy checks
- **Telegram bot security** — Bot subscribers must authenticate with an access code before receiving any system data

See [`docs/SECURITY_AND_RELIABILITY.md`](docs/SECURITY_AND_RELIABILITY.md) for full details.

---

## Reliability and Operational Safeguards

- **Database transactions** — Application submission and MOH API sync use database transactions to ensure atomicity
- **Soft deletes** — Applications and users are soft-deleted to preserve audit history
- **Queue-backed external calls** — WhatsApp messages are dispatched to a queue worker; jobs retry up to 3 times with a 60-second release delay
- **Graceful Telegram failure** — All Telegram calls in model events are wrapped in try/catch; failures are logged but do not interrupt the application lifecycle
- **Automated daily backups** — PHP-native SQL dump runs daily; old backups are purged automatically after 10 days
- **Stale application management** — A daily command marks applications with no activity for 7+ days as Unknown
- **Runtime maintenance mode** — Administrators can enable maintenance mode with a custom message, selectively restricting specific roles without a code deployment
- **Export file cleanup** — Custom export download controller deletes export files immediately after download

---

## Technical Challenges and Solutions

### 1. PHP-native SQL Export
**Challenge:** The production hosting environment did not have `mysqldump` available.
**Solution:** Implemented a PHP class that queries each table and generates valid SQL `INSERT` statements — producing a `.sql` dump without any binary dependency.

### 2. Capacity-Aware Waiting List Automation
**Challenge:** When a waiting-list application's start date arrives but the target section is full, the system must reschedule rather than silently skip.
**Solution:** The daily command checks capacity. If full, it queries the earliest finishing application in that section, calculates a new start date and end date based on training hours and schedule, and updates the application without changing its status.

### 3. Multi-Role Notification Routing
**Challenge:** Different status transitions must notify different combinations of roles, with some including external partners and others not.
**Solution:** Notification routing is centralized in model event hooks, with private helper methods that build recipient collections dynamically based on role, organizational assignment, and training type.

### 4. Ministry API Integration
**Challenge:** Applications from the Ministry bypass section capacity checks by institutional agreement, but must still validate section ownership and trainee uniqueness.
**Solution:** The sync action explicitly bypasses the capacity check (with a documented comment), while still validating that the section belongs to the Ministry user's linked department and that the section is active. Existing applications are upserted rather than duplicated.

### 5. Arabic RTL PDF Generation
**Challenge:** The absorption paper required proper Arabic text rendering in generated PDFs, including right-to-left layout.
**Solution:** Laravel mPDF was selected over DomPDF for its Arabic/RTL support. The PDF controller configures Arabic shaping and RTL options to ensure correct character rendering.

---

## Engineering Practices

- Strict types declared in key service and action files
- Single-responsibility actions encapsulating business operations, keeping controllers thin
- Service layer separating external integrations and domain logic from HTTP and UI layers
- Authorization delegated to policy classes registered via the service provider
- Database-level scoping enforcing role-based data visibility at the ORM layer
- Soft deletes on core models supporting audit history without data loss
- Runtime configuration through the settings system, modifiable from the admin panel without code deployments
- Queue-backed external API calls keeping web request latency low

---

## Screenshots

Screenshots will be added following the checklist in [`docs/SCREENSHOT_CHECKLIST.md`](docs/SCREENSHOT_CHECKLIST.md).

All screenshots in `assets/` use sanitized dummy data. National IDs, phone numbers, real names, and personal information are obscured before publication.

---

## Repository Structure (Case Study)

```
public-case-study/
├── README.md                        ← This file
├── docs/
│   ├── ARCHITECTURE.md              ← System architecture with diagrams
│   ├── WORKFLOWS.md                 ← Verified workflow documentation
│   ├── SECURITY_AND_RELIABILITY.md  ← Security controls and reliability patterns
│   ├── TECHNICAL_DECISIONS.md       ← Engineering decision records
│   └── SCREENSHOT_CHECKLIST.md      ← Screenshot guidance for public publication
└── assets/
    └── .gitkeep                     ← Placeholder (screenshots added manually)
```

---

## Lessons and Outcomes

Working on this production system provided hands-on experience with:

- Designing multi-role permission systems in Laravel where each role needs genuinely different views of the same dataset
- The practical tradeoffs of Filament 4 for administrative UI: fast to build, highly flexible through hooks and custom pages, but requiring careful policy integration
- Integrating external messaging APIs (WhatsApp Cloud, Telegram) reliably — particularly the importance of queue-backed delivery, graceful failure handling, and per-feature toggles
- Building automation that accounts for real-world edge cases (e.g., what happens when a section is full when a waiting-list date arrives)
- The value of runtime-configurable settings over hard-coded rules for a system where operational requirements change frequently
- PHP-native solutions for infrastructure constraints (no `mysqldump` binary, limited server access)
- Arabic language and RTL considerations in both the user interface and document generation

---

## Contact

**Adel Shurrab**
Laravel Backend Developer

- GitHub: [ADD_GITHUB_PROFILE_URL_HERE]
- LinkedIn: [ADD_LINKEDIN_PROFILE_URL_HERE]
- Email: [ADD_CONTACT_EMAIL_HERE]

---

*This repository contains a public technical case study. The production source code and operational data remain private.*