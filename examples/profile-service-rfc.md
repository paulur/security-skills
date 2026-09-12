# RFC: User Profile Service (Notification Preferences & Sharing)

| | |
|---|---|
| **Status** | Draft |
| **Author** | Engineering |
| **Date** | 2026-08-27 |
| **Reviewers** | Backend, Frontend, Platform, Security |

## 1. Summary

Introduce a **Profile Service**: the system of record for a user's account/profile information, replacing the narrower "Preferences service" originally scoped for this work. This RFC specifies three capabilities — **Account Profile** (the user's core PII: display name, email, phone number), **Notification Preferences** (which notifications a user receives, on which channels, per category), and **Notification Sharing** (letting a user share a notification they received with another user on the platform) — and establishes Profile Service as the owner of all three. Because Account Profile is PII, it is treated as sensitive data throughout this RFC and carries controls beyond those applied to preferences/sharing data (see §5.1–§5.2). This RFC covers the public/internal API surface, the database schema, the rollout plan, and a security-control review of every edge in the system (client-facing, service-to-service, and asynchronous/event-driven).

## 2. Motivation

- Users currently receive all notifications with no granular control, driving unsubscribe/spam complaints.
- Support tickets show ~12% of contacts relate to unwanted notifications.
- Upcoming marketing and billing-alert features require a preferences system to exist first.
- Notification preferences, sharing, and future account-facing capabilities (e.g. contact settings, communication consent) all belong to the same "how does this user want to be treated" domain — hosting them on a single Profile Service avoids splitting closely related user-account data across multiple narrow services that would otherwise duplicate auth, audit, and rate-limiting infrastructure.

### Goals
- Let users view and update notification preferences from web settings.
- Support per-category, per-channel opt-in/opt-out.
- Provide a backend API other services can query before sending a notification.
- Sensible default preferences for new users.
- Let a user share a notification they received with another user on the platform, with the recipient able to view what was shared and by whom.
- Ensure every edge in the system — client-facing, internal service-to-service, and event-driven — has an explicit, deterministic security control; no edge should be a silent gap in the design.
- Let Profile Service own the user's core PII profile fields (display name, email, phone number) as the system of record, with security controls appropriate to PII rather than the lighter controls used for non-sensitive preference data.

### Non-goals
- Building the notification-sending/delivery pipeline itself (assumed to exist or be built separately).
- Mobile app UI (API will support it, but mobile client work is out of scope here).
- Frequency/digest scheduling (future RFC).
- Sharing notifications outside the platform (email forward, external link, social share) — this RFC only covers user-to-user sharing within the product.
- Group/broadcast sharing (share with a team, channel, or all followers) — single-recipient sharing only for now.
- Authentication credentials (passwords, MFA secrets, recovery codes) — these remain owned by the existing auth/identity service; Profile Service stores PII profile fields but never credentials.
- Binary asset storage (avatar images) — Profile Service may store a reference/URL to an avatar asset if needed, but does not itself become a file/blob store.

## 3. Proposed Design

### 3.1 High-level architecture

```
Web Client
   │  REST/JSON over HTTPS
   ▼
Profile Service
   │
   ▼
Profile DB (Postgres)

Web Client (view/edit account profile — PII)
   │  REST/JSON over HTTPS
   ▼
Profile Service — Account profile endpoints
   │
   ▼
Profile DB (Postgres: user_profile, profile_audit_log)

Profile Service
   │  publish preference.updated
   ▼
Event Bus
   │  subscribe
   ▼
Notification Sender (invalidates its local cache)

Other internal services (Notification Sender, Billing, Marketing)
   │  gRPC/internal REST
   ▼
Profile Service (read-only "can-send" check — never returns PII)

Web Client (share action)
   │  REST/JSON over HTTPS
   ▼
Profile Service — Sharing endpoints
   │
   ├──▶ Profile DB (Postgres: notifications, notification_shares)
   │
   └──▶ Notification Sender (triggers a new notification to the recipient: "X shared a notification with you")
```

Profile Service is the single source of truth for the data in this RFC's scope, including PII. Other services call it synchronously before sending, or subscribe to a change event (see 3.6) for caching — neither path ever returns PII to a caller other than the profile's own owner. Sharing reuses the same service and datastore — it does not introduce a new service. Every edge shown above has its own subsection in §5 (§5.1–§5.7), broken into the same seven security dimensions; the event-driven edges (§5.3, §5.4) and the PII-carrying edges (§5.1, §5.2) get that same treatment rather than being left as implicit exceptions.

### 3.2 Data model

**Table: `user_profile`** — contains PII; see §5.1 and §5.2 for controls
| Column | Type | Notes |
|---|---|---|
| id | uuid, PK | |
| user_id | uuid, FK → users.id, unique | one profile row per user |
| display_name | text | PII |
| email | text | PII |
| phone_number | text, nullable | PII |
| email_verified_at | timestamptz, nullable | |
| phone_verified_at | timestamptz, nullable | |
| updated_at | timestamptz | |

**Table: `profile_audit_log`**
| Column | Type | Notes |
|---|---|---|
| id | uuid, PK | |
| user_id | uuid | |
| field | text | which field changed (`display_name`, `email`, `phone_number`) |
| changed_by | text | `user` or `system` |
| changed_at | timestamptz | |

Deliberately does NOT store old/new PII values in plaintext (unlike `preference_audit_log`, which safely stores booleans) — see §5.2 for why, and for what this log is expected to support instead (that a change happened, when, and by whom, without itself becoming a second copy of the PII).

**Table: `notification_categories`**
| Column | Type | Notes |
|---|---|---|
| id | uuid, PK | |
| key | text, unique | e.g. `billing_alerts`, `product_updates` |
| display_name | text | |
| default_enabled | boolean | default opt-in/out for new users |
| is_required | boolean | e.g. security alerts can't be disabled |
| is_shareable | boolean | whether notifications in this category may be shared with another user (see 3.7); defaults `false` for new categories |

**Table: `user_notification_preferences`**
| Column | Type | Notes |
|---|---|---|
| id | uuid, PK | |
| user_id | uuid, FK → users.id | indexed |
| category_id | uuid, FK → notification_categories.id | indexed |
| channel | enum(`email`,`sms`,`push`,`in_app`) | |
| enabled | boolean | |
| updated_at | timestamptz | |

Unique constraint: `(user_id, category_id, channel)`

**Table: `preference_audit_log`**
| Column | Type | Notes |
|---|---|---|
| id | uuid, PK | |
| user_id | uuid | |
| category_id | uuid | |
| channel | enum | |
| old_value | boolean | |
| new_value | boolean | |
| changed_by | text | `user` or `system` |
| changed_at | timestamptz | |

This table is not optional (see §5.1–§5.2) — it is required for both compliance and abuse investigation.

**Table: `notifications`** (sent notification instances — the objects a user can share)
| Column | Type | Notes |
|---|---|---|
| id | uuid, PK | |
| user_id | uuid, FK → users.id | recipient/owner; indexed |
| category_id | uuid, FK → notification_categories.id | |
| channel | enum(`email`,`sms`,`push`,`in_app`) | channel it was delivered on |
| title | text | |
| body | text | |
| created_at | timestamptz | |

This table is written by the Notification Sender (or equivalent delivery service) at send time, not by this RFC's write path — it's included here because Sharing (3.7) reads and references it. If a `notifications` instance table already exists elsewhere in the system, this RFC reuses it rather than duplicating it; the schema above documents the minimum fields Sharing depends on.

**Table: `notification_shares`**
| Column | Type | Notes |
|---|---|---|
| id | uuid, PK | |
| notification_id | uuid, FK → notifications.id | the notification being shared; indexed |
| shared_by_user_id | uuid, FK → users.id | indexed |
| shared_with_user_id | uuid, FK → users.id | indexed |
| message | text, nullable | optional note from sharer, max 280 chars |
| created_at | timestamptz | |

Unique constraint: `(notification_id, shared_with_user_id)` — a given notification can only be shared with the same user once (re-sharing returns the existing share rather than creating a duplicate).

**Table: `share_audit_log`**
| Column | Type | Notes |
|---|---|---|
| id | uuid, PK | |
| share_id | uuid | |
| notification_id | uuid | |
| shared_by_user_id | uuid | |
| shared_with_user_id | uuid | |
| action | enum(`created`,`deleted`) | |
| actor_user_id | uuid | who performed the action (sharer or recipient, for deletes) |
| occurred_at | timestamptz | |

Parallel to `preference_audit_log`, required by §5.1 so share creation/deletion is independently reconstructable without inferring it from the live `notification_shares` table (which loses history on delete).

Indexes: `(user_id)` on `user_notification_preferences` for fast lookup on settings page load; `(category_id, channel)` for admin/reporting queries; `(shared_with_user_id, created_at)` on `notification_shares` for the recipient's "shared with me" list.

### 3.3 Account profile API (PII — used by web client)

Base path: `/api/v1/profile/account`

**GET `/api/v1/profile/account`**
Returns the caller's own profile.
```json
{
  "display_name": "Jordan Lee",
  "email": "jordan@example.com",
  "email_verified": true,
  "phone_number": "+1555…",
  "phone_verified": false
}
```
Auth: session cookie / bearer token, scoped to `self`. No endpoint or role may fetch another user's profile through this path — see §5.1.

**PATCH `/api/v1/profile/account`**
Body:
```json
{ "display_name": "Jordan A. Lee" }
```
- `display_name` updates take effect immediately.
- An `email` or `phone_number` change does NOT take effect immediately — it MUST go through the platform's existing verification flow (confirmation link / OTP) before the new value replaces the old one, and the corresponding `*_verified_at` MUST be cleared until re-verified. This RFC assumes that verification flow already exists elsewhere and does not redesign it.
- 400 on malformed email/phone format; 409 if the target email/phone is already verified on another account.
- Every field change is recorded in `profile_audit_log` (§3.2) — field name and who changed it, not the value.

There is no internal `check`-style endpoint for account profile — unlike notification preferences, no other service needs a synchronous "may I use this field" gate. Any internal service that legitimately needs a user's PII (e.g. a support tool) must go through its own explicitly scoped, audited access path — this RFC does not grant broad internal read access to `user_profile` by default (§5.1–§5.2).

### 3.4 Public API (used by web client, notification preferences)

Base path: `/api/v1/profile/notification-preferences`

**GET `/api/v1/profile/notification-preferences`**
Returns all categories with the current user's settings, merged with defaults.
```json
{
  "categories": [
    {
      "key": "billing_alerts",
      "display_name": "Billing Alerts",
      "is_required": true,
      "channels": { "email": true, "sms": false, "push": true, "in_app": true }
    },
    {
      "key": "product_updates",
      "display_name": "Product Updates",
      "is_required": false,
      "channels": { "email": true, "sms": false, "push": false, "in_app": true }
    }
  ]
}
```
Auth: session cookie / bearer token, scoped to `self`.

**PATCH `/api/v1/profile/notification-preferences`**
Body:
```json
{
  "updates": [
    { "category": "product_updates", "channel": "email", "enabled": false }
  ]
}
```
- Validates category exists and `is_required` is false before allowing disable.
- Returns updated full preference set (same shape as GET).
- 400 on unknown category/channel; 422 if attempting to disable a required category.
- Rate-limited per §5.8.

**POST `/api/v1/profile/notification-preferences/reset`**
Resets caller's preferences to system defaults.

### 3.5 Internal API (used by other services)

**GET `/internal/v1/profile/notification-preferences/check`**
Query params: `user_id`, `category`, `channel`
Returns `{ "can_send": true }` — used synchronously by the notification-sender before dispatch.

- Should be low-latency (<20ms p99); backed by a cache (see 3.6).
- Required-category checks always return `can_send: true`.
- Caller identity is both authenticated and authorized (see §5.5/§5.6) — authentication alone is not sufficient.
- This endpoint returns only a boolean; it never returns PII, regardless of caller (§5.5/§5.6).

### 3.6 Caching & change propagation

- Profile Service publishes a `preference.updated` event (user_id, category, channel) to the internal event bus on every write.
- The Notification Sender maintains a local read-through cache (e.g., Redis, 5 min TTL) keyed by `user_id:category:channel`, invalidated on the event.
- Avoids a synchronous DB round-trip on every notification send at scale.
- Event publication and consumption are both covered by deterministic controls in §5.3/§5.4 — this was a gap in earlier drafts of this RFC. Per §5.3, this event never carries PII.

### 3.7 Defaults & new-user provisioning

- On user creation, no rows are written to `user_notification_preferences`; absence of a row means "use `notification_categories.default_enabled`."
- This keeps the table small and makes changing a default retroactively easy (no backfill needed) — only explicit overrides are stored.
- A `user_profile` row IS written at user-creation time (unlike preferences), seeded from whatever the signup flow collected (e.g. email); it is not left absent-by-default, since account profile is expected to always exist once a user exists.

### 3.8 Notification sharing

Base path: `/api/v1/profile/notifications`

**POST `/api/v1/profile/notifications/{notification_id}/share`**
Body:
```json
{
  "target_user_id": "b3f1...",
  "message": "thought you'd want to see this"
}
```
- Caller must be the `user_id` (recipient) on the `notifications` row identified by `notification_id` — sharing something you didn't receive is not allowed.
- The notification's category must have `is_shareable: true`; otherwise reject.
- `target_user_id` must be an existing, active user and must not equal the caller's own id (no self-share).
- On success, creates (or returns the existing) `notification_shares` row and triggers a new notification to `target_user_id` on their own preferred channels (subject to their own preferences, per 3.4/3.6 — sharing does not bypass the recipient's opt-outs).
- Returns the created/existing share object.
- 429 if the caller exceeds the share rate limit (see 5).

**GET `/api/v1/profile/notifications/shared-with-me`**
Returns notifications shared with the caller, most recent first, paginated.
```json
{
  "shares": [
    {
      "share_id": "9c2a...",
      "notification": { "id": "...", "title": "...", "body": "...", "category": "product_updates" },
      "shared_by": { "user_id": "...", "display_name": "..." },
      "message": "thought you'd want to see this",
      "shared_at": "2026-08-20T14:03:00Z"
    }
  ],
  "next_cursor": "..."
}
```
Auth: session cookie / bearer token, scoped to `self` (same as 3.4). Note that `shared_by.display_name` here IS a PII field surfaced across a user boundary (the recipient sees the sharer's name) — this is the one deliberate exception to "PII stays self-scoped," and it's covered explicitly in §5.1.

**DELETE `/api/v1/profile/notifications/shares/{share_id}`**
Lets either party (sharer or recipient) remove a share from their own view. Deletes the `notification_shares` row (writing a `deleted` row to `share_audit_log` first, per §5.1); does not delete the underlying `notifications` row.

## 4. API Error Handling

| Code | Meaning |
|---|---|
| 400 | Malformed request / unknown category or channel / malformed email or phone format |
| 401 | Unauthenticated |
| 403 | Attempting to modify another user's preferences or profile; attempting to share a notification you don't own, share to yourself, or share a non-shareable category; internal caller authenticated but not authorized for the requested category/service scope |
| 404 | Notification, share, or target user not found |
| 409 | Target email/phone on a profile update is already verified on another account |
| 422 | Attempting to disable a required category |
| 429 | Rate limit exceeded (preferences, sharing, account profile, or internal check, per §5.8) |
| 500 | Internal error |

## 5. Security & Privacy Considerations

Each control below is stated as a deterministic requirement (MUST/MUST NOT) that any implementation has to satisfy, not as a recommended technique. Where more than one mechanism could satisfy a requirement (e.g. which session-token format, which encryption-at-rest provider, which message-queue technology), this RFC intentionally does not pick one — that choice is an implementation detail for the owning team, constrained only by the invariant stated here.

§5.1–§5.7 cover every endpoint pair (edge) in the architecture (§3.1) individually, each broken into the same seven dimensions: sensitive data, authentication method, authorization, input validation, data protection in storage, data protection in transit, and logging. Where the RFC specifies nothing for a given dimension on a given edge, that's stated explicitly as "Not specified by the RFC" rather than left blank or invented — a real gap should be visible, not silently absent. Rate limiting and denial-of-service controls don't map one-to-one onto individual edges (several of them are cross-cutting, e.g. a per-user limit that spans multiple endpoints), so they're kept as their own subsections, §5.8 and §5.9, rather than forced into each edge's breakdown.

### 5.1 Web user → Profile Service

- **Sensitive data**: Auth token (session cookie / bearer token) on every call; PII (display name, email, phone number) additionally on account-profile calls (§3.3). One deliberate cross-user PII exposure exists in this design: `shared_by.display_name` on `GET .../shared-with-me` (§3.8) lets a recipient see the sharer's display name — scoped to display name only (never email or phone), and MUST NOT be extended to other PII fields without a fresh review.
- **Authentication method**: Session cookie or bearer token. Session tokens MUST follow the platform's existing expiry and rotation policy; this RFC does not define a new session mechanism.
- **Authorization**: The acting user MUST always be derived from the authenticated session — no endpoint accepts a client-supplied `user_id`. Every write (`PATCH` preferences, `POST`/`DELETE` share, `PATCH` account) MUST re-check ownership against the database at request time. Account-profile endpoints (`GET`/`PATCH /api/v1/profile/account`) MUST only ever operate on the caller's own row; there is no internal-service equivalent (§3.3). Changing `email` or `phone_number` MUST NOT take effect until the platform's existing verification flow confirms the new value, and the corresponding `*_verified_at` MUST be cleared until re-verified — this prevents account-takeover-via-profile-edit. Sharing endpoints MUST reject self-share, and a notification MUST NOT be shareable unless its category's `is_shareable` flag is explicitly `true` (fail closed, default `false`); a share response MUST expose only the shared notification's own fields, never the original recipient's other data.
- **Input validation**: `category`, `channel`, `notification_id`, `share_id` MUST be validated against known enum values / existence before any write (`400` on invalid, never reaching the database as a raw query). The share `message` MUST be capped at 280 characters and escaped before storage/render; notification `title`/`body` MUST be escaped the same way when rendered via sharing (both are stored-XSS surfaces). Email/phone format MUST be validated on account-profile writes.
- **Data protection (storage)**: N/A at this edge — see §5.2 for how data written here is protected once it reaches Profile DB.
- **Data protection (in-transit)**: All client-facing traffic MUST be transport-encrypted (TLS); a plaintext request MUST be rejected before it reaches application code.
- **Logging**: Preference changes → `preference_audit_log`; share create/delete → `share_audit_log`; account-profile field changes → `profile_audit_log` (field name, who, when — MUST NOT include the PII value itself, plaintext or otherwise). No log at this edge may contain a raw session token, bearer token, or other credential. A user's PII deletion/export request (GDPR/CCPA-style) MUST cover `user_profile` explicitly, not just preferences/sharing data.

### 5.2 Profile Service → Profile DB

- **Sensitive data**: PII (display name, email, phone number) on `user_profile` reads/writes. Every field in `user_profile` MUST be classified and documented as PII; a future field added to this table MUST go through the same classification before being added, not be assumed non-sensitive by default.
- **Authentication method**: Only Profile Service's own service identity has database credentials to these tables. This design MUST NOT introduce any new long-lived secret — any service credential it relies on MUST be issued and rotated through the platform's existing credential-management system, not managed ad hoc by this service.
- **Authorization**: No other service is granted direct database access to these tables, regardless of network reachability (least privilege) — an internal service that needs data here MUST go through Profile Service's API, not the database directly.
- **Input validation**: All database access MUST use parameterized queries or an ORM; string-concatenated SQL MUST NOT be used anywhere in this feature.
- **Data protection (storage)**: Data at rest (preferences, notifications, shares, `user_profile`, and all three audit logs) MUST be encrypted using the platform's standard at-rest encryption. `profile_audit_log` specifically MUST NOT store old/new PII values in plaintext — only that a named field changed.
- **Data protection (in-transit)**: Internal database traffic MUST be transport-encrypted, per the general rule that "internal network" is not an exemption from encryption.
- **Logging**: Same three audit tables as §5.1; no credential or raw PII value may appear in any application-level log at this edge. Audit records MUST be retained per the platform's existing data-retention policy; this RFC does not define a new retention schedule.

### 5.3 Profile Service → Event bus (publish `preference.updated`)

- **Sensitive data**: None — the event payload is limited by design to `user_id`, `category`, `channel`; PII and credentials MUST NOT be added to it without a corresponding update to this security review.
- **Authentication method**: Authenticated producer identity required; an unauthenticated or unidentified producer MUST NOT be able to publish.
- **Authorization**: Producer identity MUST be scoped to publishing this event only.
- **Input validation**: Event schema changes MUST be backward compatible or explicitly versioned — a producer MUST NOT change the shape of `preference.updated` in a way that silently breaks an existing consumer's parsing.
- **Data protection (storage)**: N/A — this is an in-flight message, not data at rest in this service's own stores.
- **Data protection (in-transit)**: MUST be transport-encrypted; "internal" is not by itself a justification for skipping encryption.
- **Logging**: Not specified by the RFC.

### 5.4 Event bus → Notification sender (subscribe)

- **Sensitive data**: None stated.
- **Authentication method**: Authenticated consumer identity required; the event bus MUST NOT allow anonymous or unauthenticated subscription to this topic.
- **Authorization**: Consumer identity MUST be scoped to this topic.
- **Input validation**: Consumers MUST treat delivery as at-least-once and MUST make handling of the event idempotent (keyed on `user_id:category:channel`, per §3.6), so redelivery cannot corrupt the local cache or trigger duplicate side effects. A message the consumer cannot process (malformed, unknown schema version) MUST be routed to a dead-letter queue rather than dropped silently or retried indefinitely.
- **Data protection (storage)**: N/A.
- **Data protection (in-transit)**: MUST be transport-encrypted.
- **Logging**: Not specified by the RFC.

### 5.5 Notification sender → Profile Service (`check`)

- **Sensitive data**: None — this endpoint is boolean-only by design and MUST NOT return PII under any circumstance, regardless of caller identity or requested fields.
- **Authentication method**: mTLS or service-mesh identity (or an equivalent) — reachable only over authenticated service-to-service channels, never from the public internet.
- **Authorization**: Authentication alone is NOT sufficient — the endpoint MUST also authorize the caller's service identity against the category/data it is requesting, so a compromised or misconfigured internal service cannot query arbitrary users' preferences just because it holds a valid service credential.
- **Input validation**: `user_id`, `category`, `channel` MUST be validated the same way as on the client-facing edge (§5.1).
- **Data protection (storage)**: N/A.
- **Data protection (in-transit)**: Restricted to authenticated service-to-service channels; not reachable from the public internet.
- **Logging**: Not specified by the RFC.

### 5.6 Other internal services → Profile Service (`check`)

Same endpoint as §5.5, different caller class — the requirements are identical:

- **Sensitive data**: None — same boolean-only constraint; this RFC grants no internal service broad read access to `user_profile`.
- **Authentication method**: mTLS or service-mesh identity.
- **Authorization**: Authenticated AND authorized per-service/per-category scope, same as §5.5.
- **Input validation**: Same as §5.5.
- **Data protection (storage)**: N/A.
- **Data protection (in-transit)**: Same as §5.5.
- **Logging**: Not specified by the RFC.

### 5.7 Profile Service → Notification sender (trigger share alert)

- **Sensitive data**: None stated.
- **Authentication method**: Restricted to authenticated service-to-service channels.
- **Authorization**: MUST only fire once the `is_shareable` fail-closed gate (§5.1) has passed for the notification being shared.
- **Input validation**: Notification content triggering this call was already validated/escaped at the point it was created (§5.1); no additional validation is specified at this edge.
- **Data protection (storage)**: N/A.
- **Data protection (in-transit)**: Same internal-traffic-encrypted rule as other service-to-service edges.
- **Logging**: Not specified by the RFC.

### 5.8 Rate limiting & abuse prevention (cross-cutting)

- The `PATCH` preferences endpoint MUST enforce a per-user rate limit low enough to block scripted preference-flipping abuse; the exact algorithm and threshold are an implementation decision (an illustrative starting point is on the order of tens of requests per minute).
- The `POST .../share` endpoint MUST enforce its own, independent per-user rate limit, set more conservatively than the preferences limit, since this endpoint delivers content to other users and is a higher-value abuse target than a self-scoped edit.
- The internal `check` endpoint MUST enforce a per-caller-service rate limit or circuit breaker, distinct from the client-facing limits, so a misbehaving or looping internal caller cannot degrade the endpoint for every other consumer.
- Repeated share attempts for the same `(notification_id, shared_with_user_id)` pair MUST NOT create duplicate share records or duplicate outbound notifications — this MUST be enforced as a database-level uniqueness constraint (per §3.8), not only as client-side or application-layer deduplication, so it holds regardless of retries or concurrent requests.
- A recipient MUST have a self-serve way to remove an unwanted share (`DELETE .../shares/{share_id}`) without contacting support. Whether a recipient can also block a sharer from future shares is an open question (§8) and is not required for launch.

### 5.9 Denial-of-service / resource exhaustion (cross-cutting)

- A failure of the preference-check cache MUST NOT cause the internal `check` endpoint to hang or time out the caller indefinitely; it MUST fail toward a defined, explicit behavior (either allow or deny) that is tested before launch — which of the two is the right default is a product decision, not specified here.
- `GET .../shared-with-me` MUST be paginated (cursor-based, per §3.8) so that no single request can force an unbounded database scan regardless of how large a user's share history grows.
- The event bus consumer(s) in §5.4 MUST apply backpressure or buffering rather than unbounded in-memory queuing if event volume exceeds processing capacity, so a burst of preference updates cannot exhaust the Notification Sender's memory.

## 6. Rollout Plan

1. Ship schema + internal API behind a feature flag; backfill `notification_categories` table.
2. Integrate Notification Sender to call `check` before every send (default to "allow" if flag off).
3. Ship web settings UI calling the public API.
4. Enable flag for internal employees, then 5% → 50% → 100% of users.
5. Monitor: preference-check latency, cache hit rate, error rates, support ticket volume.
6. Ship sharing schema (`notifications` reuse, `notification_shares`, `share_audit_log`) behind a separate flag; mark an initial small set of categories `is_shareable: true` (e.g. `product_updates`) — leave sensitive categories non-shareable until product/legal signs off case by case.
7. Ship share/unshare UI and the "shared with me" view, gated on the same flag; roll out 5% → 50% → 100% independently of the preferences rollout.
8. Ship `user_profile` and account-profile endpoints behind their own flag, gated on privacy/legal sign-off given this is the service's first PII surface; migrate existing `users`-table fields (display name, email, phone) into `user_profile` via a backfill job rather than a big-bang cutover.
9. Before either flag reaches 100%, verify the §5.3/§5.4 event-bus controls (producer/consumer auth, dead-letter handling, idempotent consumption) and the §5.1/§5.2 PII controls (verification-gated email/phone changes, no-plaintext-PII audit log, internal `check` never leaking PII) in a staging environment.
10. Monitor: share volume per user (abuse signal), share rate-limit hit rate, internal `check` rate-limit hit rate, dead-letter queue depth, profile-change verification completion rate, report/block rate if that lands alongside.

## 7. Alternatives Considered

- **Store preferences as a JSON blob on the user row** — simpler, but harder to query/report on (e.g., "how many users opted out of SMS marketing") and no per-field audit trail. Rejected.
- **Push-based only (no internal check API)** — sender services subscribe to full preference dumps. Rejected due to staleness risk and higher integration complexity for consumers.

## 8. Open Questions

- Should required (non-disable-able) categories be enforced server-side only, or also reflected as disabled toggles in the UI?
- Do we need per-channel default overrides at the organization/tenant level (for B2B customers), or is this strictly per-user for now?
- SMS channel requires phone verification — should the API reject `sms: true` if no verified phone exists, or silently no-op?
- Should a recipient be able to block a specific user from sharing with them again, independent of the general per-category share setting? Would need a new `share_blocks` table if so.
- Should sharing require the target user to be an existing connection/follower (if such a concept exists elsewhere in the product), or can any user share with any other user by id/handle?
- Does a shared notification need its own moderation/reporting flow (report abusive `message` text), or does this piggyback on an existing platform-wide reporting system?
- Should `is_shareable` be settable per-notification-instance by the sender (e.g. a marketing campaign wants shares even though its category defaults to non-shareable), or is category-level the only granularity we support at launch?
- Which specific authorization model should back the internal `check` endpoint's per-service scoping (§5.5/§5.6) — static allowlist, per-service policy, or a central policy engine? This RFC establishes the requirement but leaves the mechanism to a follow-up design.
- Does the platform's existing email/phone verification flow (assumed by §3.3/§5.1) support being triggered from Profile Service directly, or does it need an integration point built for this?
- Should `display_name` have its own separate visibility/format rules (e.g. profanity filtering, uniqueness) beyond what's specified here, given it's now shown across a user boundary via sharing (§3.8)?
- Should field-level access within `user_profile` be more granular than "the whole row, self-scoped only" — e.g. could a future internal service legitimately need `display_name` without `email`/`phone_number`? This RFC deliberately grants no internal access at all (§5.1–§5.2) pending that design.
