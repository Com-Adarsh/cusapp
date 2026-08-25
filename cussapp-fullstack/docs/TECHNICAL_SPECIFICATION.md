# CUSSApp Technical Specification

**Product:** CUSSApp - The Digital Campus for Every CUSAT Student  
**Status:** Build-ready MVP specification  
**Target:** Responsive web application, with a future mobile client using the same API

## 1. Product Scope

CUSSApp is a two-sided campus platform:

```text
Student web/mobile client <-> CUSSApp API <-> PostgreSQL
                                      |
                         Admin dashboard and notification workers
                                      |
                 Verified CUSAT/Union sources for Ask CUSAT
```

The MVP (Phase 1) includes:

- Home feed: announcements, today's events, deadlines, scholarships and notification inbox
- Events: browse, detail, registration and attendance QR foundation
- Announcements: official, targeted and source-linked notices
- Opportunities: scholarships first, with a reusable opportunity model
- Campus: searchable map, locations, services and emergency contacts
- Grievances: submission, status tracking, attachments and staff workflow
- Notifications: in-app notifications and optional email/push delivery
- Admin dashboard: content, users, grievances, notifications and source management

Phase 2 adds polls, study circles, digital ID, event QR attendance, profile achievements and memories. Phase 3 adds Ask CUSAT and recommendations. Phase 4 adds transport, food/mess, academic integrations and the student portfolio.

## 2. Architectural Decisions

### 2.1 Recommended stack

- **Frontend:** React + TypeScript, responsive PWA, TanStack Query for server state
- **API:** Node.js 20+, Express, TypeScript, Zod request validation
- **Database:** PostgreSQL 16 with migrations; PostGIS when map radius/search is needed
- **Files:** S3-compatible object storage for documents, grievance attachments and certificates
- **Authentication:** CUSAT SSO/OIDC when available; email/password only for development and approved external users
- **Jobs:** Redis-backed queue for notifications, document processing and scheduled publishing
- **Observability:** structured JSON logs, request IDs, metrics and error tracking
- **Deployment:** separate web, API, worker and PostgreSQL services behind HTTPS

The current Express/vanilla-JavaScript prototype can remain the local reference implementation. JSON persistence must not be used for production or concurrent writes.

### 2.2 Service boundaries

Start as a modular monolith. Keep these modules separated in code and database ownership:

1. Identity and access
2. Home/feed and notifications
3. Events and registrations
4. Opportunities
5. Campus directory and map
6. Grievances
7. Community
8. Admin/content management
9. Verified knowledge and Ask CUSAT
10. Audit and analytics

Extract workers and search only when load requires it. A microservice split is not an MVP requirement.

## 3. Roles and Authorization

Use deny-by-default RBAC. A user may have more than one role through `user_roles`.

| Role | Access |
| --- | --- |
| `student` | Own profile, published content, event registration, own grievances, polls and notifications |
| `faculty` | Published content and approved faculty workflows; no student-private data by default |
| `union_editor` | Draft and submit union announcements/events/opportunities; moderate assigned community content |
| `department_editor` | Manage content scoped to an assigned department |
| `grievance_officer` | View and update assigned grievance queues; internal notes and status transitions |
| `content_reviewer` | Review, approve, reject and schedule official content |
| `knowledge_manager` | Manage verified sources, document chunks and Ask CUSAT publishing status |
| `admin` | Full operational access, user/role management and configuration |
| `super_admin` | Break-glass access, security settings and role assignment |

Every admin mutation records actor, target, old value, new value, IP and timestamp in `audit_logs`. Scope checks must happen server-side, not only in the dashboard.

## 4. Database Schema

All tables use UUID primary keys, UTC timestamps, and soft deletion where content history matters. Add `created_at`, `updated_at`, and `deleted_at` to mutable domain tables as appropriate.

### Identity

**`users`**

`id`, `email` unique, `name`, `avatar_url`, `student_id` unique nullable, `department_id`, `programme_id`, `year_of_study`, `status` (`active|suspended|pending`), `last_login_at`, timestamps.

**`auth_identities`**

`id`, `user_id`, `provider` (`cusat_oidc|email`), `provider_subject` unique, `password_hash` nullable, `verified_at`, timestamps.

**`roles`**, **`user_roles`**

Role catalogue and user-to-role membership. `user_roles` contains optional `scope_type` and `scope_id` for department/organization-scoped permissions.

**`notification_preferences`**

`user_id`, `in_app_enabled`, `email_enabled`, `push_enabled`, category JSON, quiet hours, timestamps.

### Campus reference data

**`departments`**, **`programmes`**

Names, codes, descriptions and active status. Programmes belong to departments.

**`campus_locations`**

`id`, `name`, `type`, `description`, `building_code`, `latitude`, `longitude`, `map_data`, `phone`, `hours_json`, `accessibility_json`, `status`.

**`emergency_contacts`**

`id`, `label`, `phone`, `location_id`, `priority`, `active`.

### Content

**`announcements`**

`id`, `title`, `summary`, `body`, `category`, `audience_json`, `source_url`, `source_label`, `status` (`draft|in_review|scheduled|published|archived`), `published_at`, `expires_at`, `created_by`, `reviewed_by`.

**`events`**

`id`, `title`, `description`, `start_at`, `end_at`, `location_id`, `organizer_name`, `organizer_contact`, `capacity`, `registration_required`, `registration_deadline`, `audience_json`, `status`, `source_url`, `created_by`, `published_at`.

**`event_registrations`**

Unique `(event_id, user_id)`, plus `status`, `registered_at`, `checked_in_at`, `check_in_token_hash`, and optional cancellation reason.

**`opportunities`**

`id`, `type` (`scholarship|internship|fellowship|competition|job|career_event`), `title`, `description`, `provider`, `eligibility_json`, `department_ids`, `programme_ids`, `application_url`, `deadline_at`, `status`, `source_url`, `created_by`, `published_at`.

**`polls`**, **`poll_options`**, **`poll_votes`**

Poll lifecycle, audience and closing time. Enforce unique `(poll_id, user_id)` for one vote per poll unless an explicit anonymous mode is configured.

### Student activity and support

**`grievances`**

`id`, `ticket_number` unique, `reporter_id`, `category`, `title`, `description`, `location_id`, `priority`, `status`, `assigned_to`, `public_updates`, `created_at`, `resolved_at`.

**`grievance_events`**

Append-only status history: `grievance_id`, `from_status`, `to_status`, `note`, `is_internal`, `actor_id`, timestamp.

**`attachments`**

Polymorphic reference to grievance/content/profile item, object-storage key, original filename, MIME type, byte size, checksum, uploader and malware-scan status.

Phase 2 profile tables: `student_profiles`, `certificates`, `achievements`, `volunteer_activities`, `union_roles`, `memories`, `study_groups`, `study_group_members`.

### Notifications and audit

**`notifications`**

`id`, `user_id`, `type`, `title`, `body`, `data_json`, `read_at`, `dedupe_key`, timestamps.

**`notification_deliveries`**

`notification_id`, `channel`, `status`, provider message ID, attempts, `sent_at`, `failure_reason`.

**`audit_logs`**

`id`, `actor_id`, action, resource type/id, before JSON, after JSON, request ID, IP, user agent, timestamp.

### Ask CUSAT knowledge layer

**`knowledge_sources`**

`id`, `title`, `source_type` (`official_url|uploaded_document|admin_entry`), `uri`, `authority`, `owner`, `version`, `valid_from`, `valid_until`, `review_status`, `reviewed_by`, checksum, timestamps.

**`knowledge_chunks`**

`id`, `source_id`, chunk text, metadata JSON, embedding/vector reference, content hash, active status.

**`ask_questions`**

`id`, `user_id` nullable, question, normalized question, answer status, answer text, confidence, refusal reason, request ID, timestamps.

**`ask_citations`**

`question_id`, `source_id`, `chunk_id`, quoted excerpt, relevance score.

An answer is publishable only when it has active citations from approved sources. Otherwise the API must return an explicit `unverified`/`no_verified_answer` result.

## 5. API Contract

All endpoints are under `/api/v1`. JSON errors use `{ "error": { "code": "...", "message": "...", "requestId": "..." } }`. List endpoints return `{ "data": [], "page": 1, "pageSize": 20, "total": 0 }`.

### Authentication

| Method | Endpoint | Purpose |
| --- | --- | --- |
| `GET` | `/auth/oidc/start` | Begin CUSAT SSO |
| `GET` | `/auth/oidc/callback` | Complete SSO and issue session |
| `POST` | `/auth/dev/register` | Development-only account creation |
| `POST` | `/auth/refresh` | Rotate refresh token |
| `POST` | `/auth/logout` | Revoke current session |
| `GET` | `/me` | Current user, roles and preferences |
| `PATCH` | `/me` | Update allowed profile fields |
| `PATCH` | `/me/notification-preferences` | Update preferences |

Use short-lived access tokens (15 minutes) and rotating, hashed refresh tokens in secure, HttpOnly cookies. Do not store production refresh tokens in local storage.

### Student API

| Method | Endpoint | Purpose |
| --- | --- | --- |
| `GET` | `/home` | Personalized feed and quick-action counts |
| `GET` | `/announcements` | Published notices with audience filtering |
| `GET` | `/events` | Filter by date, type, location and page |
| `GET` | `/events/:id` | Event detail and registration state |
| `POST` | `/events/:id/registrations` | Register current student |
| `DELETE` | `/events/:id/registrations` | Cancel registration |
| `GET` | `/opportunities` | Filter by type, department, programme and deadline |
| `GET` | `/campus/locations` | Searchable campus directory/map data |
| `GET` | `/campus/emergency-contacts` | Current emergency contacts |
| `POST` | `/grievances` | Submit a grievance with optional attachment IDs |
| `GET` | `/grievances` | Current user's tickets |
| `GET` | `/grievances/:id` | Ticket details and public history |
| `GET` | `/notifications` | Current user's inbox |
| `POST` | `/notifications/:id/read` | Mark notification read |
| `GET` | `/polls` | Active polls |
| `POST` | `/polls/:id/votes` | Cast one vote |
| `POST` | `/ask` | Ask against approved knowledge only |

### Admin API

Admin routes are protected by permission middleware and resource scope checks.

`/admin/announcements`, `/admin/events`, `/admin/opportunities`, `/admin/polls`, `/admin/campus/locations`, `/admin/grievances`, `/admin/users`, `/admin/notifications`, `/admin/knowledge-sources`, and `/admin/analytics` support CRUD, moderation, assignment, publishing and export operations. Mutating routes accept an `Idempotency-Key` for retriable actions such as notification sends and event registration.

Admin content follows: `draft -> in_review -> approved/scheduled -> published -> archived`. Only reviewers can approve content authored by another account. Published source URLs and effective dates are required for official notices, opportunities and knowledge sources.

## 6. Authentication and Security

- Prefer CUSAT-managed OIDC with verified university identity and immutable provider subject.
- Hash development passwords with Argon2id or bcrypt cost 12+; never log credentials or tokens.
- Validate and normalize all input with shared schemas; use parameterized queries.
- Apply rate limits per IP and account: auth, Ask CUSAT, grievance creation and voting need stricter limits.
- Enforce CSRF protection for cookie-authenticated mutations and strict CORS for known clients.
- Require HTTPS, secure cookies, CSP, HSTS and frame-ancestor restrictions in production.
- Scan uploads, cap size/type, store outside the web root, and serve through signed URLs.
- Encrypt sensitive data at rest where required by institutional policy.
- Redact student identifiers from logs and analytics; define retention and deletion policies with CUSAT.
- Back up PostgreSQL daily, test restoration monthly, and maintain migration scripts.

## 7. Notification System

1. An approved event, announcement, grievance update or targeted admin message creates a notification intent.
2. The API writes the intent and an outbox record in one database transaction.
3. A worker resolves the audience from department, programme, year, role or explicit user IDs.
4. Preferences, quiet hours, deduplication and opt-outs are applied.
5. Channel adapters send in-app immediately and email/push asynchronously.
6. Delivery status, retry count and provider response are recorded.

Notifications must be idempotent by `dedupe_key`, retry with exponential backoff, and dead-letter after a bounded number of attempts. High-severity emergency notifications may bypass quiet hours but must be explicitly authorized and audited.

## 8. Ask CUSAT Architecture

Ask CUSAT is retrieval-augmented generation with a strict evidence gate, not an open-ended chatbot.

### Ingestion

1. A `knowledge_manager` adds an official URL, approved document or admin-authored answer.
2. The worker fetches or extracts text, records checksum/version and removes unsupported content.
3. Text is chunked with source metadata and indexed for keyword plus vector retrieval.
4. A reviewer verifies authority, effective dates and access scope.
5. Only `approved` and non-expired sources become retrievable.

### Answering

1. Authenticate the request when personalization or private data is needed.
2. Moderate and normalize the question; apply rate limits.
3. Retrieve top candidate chunks from active approved sources.
4. Apply an evidence threshold and detect conflicting or expired sources.
5. Generate a concise answer constrained to retrieved evidence.
6. Return citations with source title, URL/label, excerpt and source date.
7. If evidence is missing, stale or conflicting, return `NO_VERIFIED_ANSWER` and recommend the relevant official contact or page.

Response shape:

```json
{
  "status": "answered",
  "answer": "...",
  "citations": [{ "sourceId": "...", "title": "...", "url": "...", "excerpt": "..." }],
  "confidence": "high",
  "asOf": "2026-08-25T00:00:00Z"
}
```

For no evidence:

```json
{
  "status": "no_verified_answer",
  "answer": "I could not find verified CUSAT information for this question.",
  "citations": [],
  "nextStep": { "label": "Contact the relevant office", "url": "..." }
}
```

Never invent deadlines, office names, policies, phone numbers, eligibility rules or citations. Log retrieval metadata and user feedback for evaluation, while excluding sensitive question content from product analytics unless consented.

## 9. Screen-by-Screen UI Flow

### Student app

1. **Session bootstrap:** splash/loading -> SSO sign-in -> consent/profile completion -> Home.
2. **Home:** greeting, notification badge, Ask CUSAT entry, Today @ CUSAT, urgent announcements, upcoming events, deadlines, scholarships and quick actions: Report Issue, Events, Map, Ask CUSAT.
3. **Announcements:** filter chips -> notice detail -> source link/share/bookmark -> read notification state.
4. **Events:** calendar/list toggle -> filters -> event detail -> register -> confirmation with calendar action -> QR pass when enabled.
5. **Opportunities:** tabs by type -> filters for department, programme, eligibility and deadline -> detail -> official application link -> save/remind.
6. **Campus:** search/map/list toggle -> location detail with directions, hours, contact and accessibility -> emergency contacts.
7. **Grievance:** category -> title/description/location/attachment -> review and submit -> ticket number -> timeline and updates.
8. **Notifications:** inbox grouped by date -> open deep link -> mark read -> preferences.
9. **Community (Phase 2):** polls, study circles, events, achievements, memories and organizations, each with report/moderation affordances.
10. **Profile:** digital ID, department/programme, registrations, certificates, achievements, volunteering, union roles and notification settings.
11. **Ask CUSAT (Phase 3):** question -> loading/evidence state -> answer and source cards, or clear no-verified-answer state -> source/contact follow-up.

Every screen needs loading, empty, error, offline and permission-aware states. Destructive actions require confirmation. Keyboard navigation, readable contrast, focus states and screen-reader labels are required.

### Admin dashboard

1. **Sign-in and role-aware home:** pending reviews, open grievances, delivery failures and content metrics.
2. **Content queue:** draft/review/approved/scheduled/published views, preview, source validation and audit history.
3. **Event manager:** create event, capacity/registration settings, attendee export, QR check-in and cancellation notices.
4. **Audience composer:** select filters, preview recipient count, schedule notification, inspect delivery results.
5. **Grievance desk:** queue filters, assignment, internal notes, status transition, public response and SLA indicators.
6. **Campus manager:** location CRUD, map coordinates, hours, services and emergency contacts.
7. **Knowledge manager:** source upload/URL, extraction status, review/version/expiry and test question with citations.
8. **Moderation:** reports, content actions, appeals and immutable audit trail.
9. **Analytics:** adoption, active users, event registrations, opportunity clicks, grievance resolution time, notification delivery and Ask CUSAT grounded-answer rate. Do not expose raw private student data by default.

## 10. Non-Functional Requirements and Acceptance Criteria

- p95 read API latency below 500 ms at 100 concurrent users, excluding file and AI processing.
- 99.5% monthly availability for the student API during MVP operations.
- All authenticated endpoints return 401/403 correctly; students cannot access another student's private data.
- A published notice, event or opportunity always shows its source and publication/expiry metadata.
- Grievance status history is append-only and visible to the reporter only through public updates.
- Duplicate event registration, poll vote and notification send are rejected or safely idempotent.
- Ask CUSAT never returns an answer without approved citations; missing evidence produces `no_verified_answer`.
- Automated tests cover auth, RBAC, content publication, event registration, grievance ownership, notification deduplication and Ask CUSAT evidence gating.
- CI runs formatting, type checking, unit tests, API integration tests, dependency audit and migration checks.

## 11. Delivery Plan

### Sprint 0: foundation

TypeScript migration plan, PostgreSQL schema/migrations, environment configuration, OIDC stub, RBAC middleware, API error format, CI and seed data.

### Phase 1: launch

Home feed, announcements, events, scholarships/opportunities, campus locations, grievances, in-app notifications and the first admin workflows.

### Phase 2: engagement

Polls, study groups, digital ID, QR attendance, profile activity, certificates, achievements and memories with moderation.

### Phase 3: intelligence

Approved-source ingestion, hybrid retrieval, Ask CUSAT evidence gate, citations, feedback and recommendation opt-in.

### Phase 4: ecosystem

Transport and food data, academic integrations, portfolio exports and institution-wide operational analytics.

Before production launch, obtain CUSAT approval for identity integration, data retention, emergency messaging, official source ownership, grievance privacy and AI disclosure language.