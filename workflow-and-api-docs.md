# Luqya AI — Workflow & API Documentation

**Project:** لُقيا AI — University AI-Powered Lost & Found System
**Stack:** PHP 8 (procedural) · MySQL/MariaDB (PDO) · Vanilla JS · Google Gemini REST API
**Language & direction:** the user interface and the entire system are designed and developed **fully in Arabic (right-to-left, RTL)** to seamlessly support the Saudi university environment; the document root is served as `<html lang="ar" dir="rtl">`.
**Base URL:** `https://<host>/luqya-ai/` (all endpoints relative; no version prefix)
**Document status:** Living document — extracted from the live codebase
**Last verified against code:** `database/schema.sql`, `config/*.php`, `api/*.php`

---

## Table of Contents

1. [User Journey & Workflow](#1-user-journey--workflow)
   - [1.1 Actors](#11-actors)
   - [1.2 Journey A — Student reports a LOST item](#12-journey-a--student-reports-a-lost-item)
   - [1.3 Journey B — Finder reports a FOUND item](#13-journey-b--finder-reports-a-found-item)
   - [1.4 Journey C — Automatic matching](#14-journey-c--automatic-matching)
   - [1.5 Journey D — Security verification & handover](#15-journey-d--security-verification--handover)
   - [1.6 Journey E — Student profile & report lifecycle](#16-journey-e--student-profile--report-lifecycle)
   - [1.7 Journey F — Password recovery](#17-journey-f--password-recovery)
   - [1.7.1 Quick demo login (evaluator convenience)](#171-quick-demo-login-evaluator-convenience)
   - [1.8 Report state machine](#18-report-state-machine)
   - [1.9 Workflow rules summary](#19-workflow-rules-summary)
2. [API Documentation](#2-api-documentation)
   - [2.1 General conventions](#21-general-conventions)
   - [2.2 Rate limits](#22-rate-limits)
   - [2.3 Error codes](#23-error-codes)
   - [2.4 Authentication endpoints](#24-authentication-endpoints-api-authphp)
   - [2.5 Reports endpoints](#25-reports-endpoints-api-reportsphp)
   - [2.6 Security dashboard endpoints](#26-security-dashboard-endpoints-api-securityphp)
   - [2.7 Notifications endpoints](#27-notifications-endpoints-api-notificationsphp)
   - [2.8 AI endpoints](#28-ai-endpoints)
   - [2.9 Master endpoint index](#29-master-endpoint-index)
3. [Gemini AI Integration](#3-gemini-ai-integration)
   - [3.1 Where it is used](#31-where-it-is-used)
   - [3.2 Centralised wrapper functions](#32-centralised-wrapper-functions-configgeminiphp)
   - [3.3 Configuration & model fallback](#33-configuration--model-fallback)
   - [3.4 Prompt construction approach](#34-prompt-construction-approach)
   - [3.5 Response handling](#35-response-handling)
   - [3.6 Graceful degradation](#36-graceful-degradation)
   - [3.7 Security & cost controls](#37-security--cost-controls)
4. [Appendix — Glossary](#4-appendix--glossary)
5. [Appendix — Known gaps](#5-appendix--known-gaps)

---

# 1. User Journey & Workflow

> **Note:** all Arabic UI strings are omitted below for readability. Steps marked 🔒
> require an authenticated session; 🛡️ require the `security` role.

## 1.1 Actors

| Actor | Role value | Capabilities |
|---|---|---|
| **Visitor** | *(none)* | Browse public home page, metrics, report grid, search, FAQ |
| **Student** | `student` | Register, log in, create LOST + FOUND reports, view own reports and lifecycle, view high-confidence potential matches, close own report |
| **Security Officer** | `security` | All student actions except creating LOST reports; plus verify codes, confirm/reject matches, record handover, CRUD found items |
| **Administrator** | `admin` | Declared in the schema; **no dedicated UI or API branch exists yet** |

> Officers cannot self-register. Their accounts are provisioned with the login code
> `SEC-USR-<id>`, stored only as a bcrypt hash in `users.security_code_hash`.

---

## 1.2 Journey A — Student reports a LOST item

| # | Step | Actor | System behaviour | API call |
|---|---|---|---|---|
| 1 | Opens the home page and clicks **"Report a lost item"** | Visitor | Modal opens; no login required to *view*, but submitting requires login | — |
| 2 | Submits university ID + name + password | Visitor | Validates ID format (`S` + 9 digits), hashes password with bcrypt, forces role = `student`, derives institutional email, creates session | `auth.php?action=register` |
| 3 | Redirected home; clicks **"I lost something"** | Student 🔒 | Modal opens | — |
| 4 | Optionally **scans/uploads a photo** | Student 🔒 | `POST` image → real MIME detection via `finfo` → allow-list `jpg/png/webp` → ≤ 8 MB → random filename → saved to `uploads/`. Gemini analyses the image and returns category, title, tags, description | `vision_ai.php` |
| 5 | Fills title, category, location, event date, description | Student 🔒 | Client-side hints; all re-validated server-side | — |
| 6 | Submits the form | Student 🔒 | `POST` with CSRF token; validates lengths against `config/schema.php` limits; defaults `date_event` to today if not `YYYY-MM-DD`; inserts row with `status='active'`, `pickup_code = NULL` | `reports.php?action=create` |
| 7 | **AI enhancement runs inline** | System | `enhanceDescriptionWithGemini()` rewrites the description. If Gemini fails, the original text is kept and the flow continues | *(internal)* |
| 8 | **Matching engine runs inline** | System | Scores the new report against all `active` opposite-type reports | *(internal)* |
| 9 | Success message + optional match result | Student 🔒 | Returns `report_id` and `match_info`; if matched, also the shared `pickup_code` | — |

**Key point:** the student sees a pickup code *immediately only if a match was found*.
A LOST report starts with no code of its own.

---

## 1.3 Journey B — Finder reports a FOUND item

| # | Step | Actor | System behaviour | API call |
|---|---|---|---|---|
| 1 | Clicks **"I found something"** | Student 🔒 / Officer 🛡️ | Modal opens | — |
| 2 | Optionally **scans the owner's ID card** | Finder 🔒 | `mode = 'id_card'`. Gemini extracts `student_id`, owner name, college → stored in `reports.ocr_data`. Pre-fills the report fields | `vision_ai.php` |
| 3 | Or optionally scans a **general item photo** | Finder 🔒 | `mode = 'general_item'` → suggested category, title, tags, description | `vision_ai.php` |
| 4 | Completes and submits the form | Finder 🔒 | Same validation as Journey A | `reports.php?action=create` |
| 5 | **A unique pickup code is generated** | System | `generateRetrievalCode()` produces `RET-XXXXXX`, checked for uniqueness against **both** `reports` and `matches` before insert | *(internal)* |
| 6 | Matching engine runs | System | Same as Journey A, step 8 | *(internal)* |

**Key difference:** the pickup code is generated **only for `type = 'found'`**, so a found
item can be verified at the security desk immediately, even before any match exists.

---

## 1.4 Journey C — Automatic matching

Triggered automatically after every successful report insert. Implementation:
`config/matching.php` → `runAutomaticMatching()` (exposed to the API as the one-line
alias `runAIMatchingEngine()` in `api/ocr_sim.php`, which is not cosmetic — it routes into
the hybrid engine described in §3.1.1).

| # | Step | System behaviour |
|---|---|---|
| 1 | Load candidates | All reports of the **opposite type** with `status = 'active'` |
| 2 | Score each pair | Weighted deterministic scoring — see table below |
| 3 | Filter | Keep candidates with score ≥ `MATCH_SUGGEST_THRESHOLD` (45); cap at 12 |
| 4 | Exclude | Skip pairs that already have a `matches` row, and pairs an officer has rejected |
| 5 | Semantic re-rank | `semanticRerankCandidates()` blends the Gemini opinion into the top pairs — one batched call, ≤ `MATCH_AI_MAX_PAIRS` pairs, skipped entirely if no key. Shortlist floor is `MATCH_AI_MIN_LOCAL` (45) so the blend can promote an under-rated pair |
| 6 | Decide | If top **blended** score ≥ `MATCH_AUTO_THRESHOLD` (75) → **create the match**; otherwise → record nothing and let the officer see it as a *suggestion* |
| 7 | On match: write `matches` | `status = 'pending'`, `confidence_score = <blended score>`, `pickup_code = <shared code>` |
| 8 | On match: update both reports | LOST → `matched`; FOUND → `pending_pickup`; **both receive the same `pickup_code`** |
| 9 | Notify | `match_found` notification to the LOST report owner, stating the confidence % and the pickup code |
| 10 | Audit | `activity_logs` row with `action_type = 'match'` |
| 11 | Return | `{matched, confidence, pickup_code, local_score, ai_score, ai_verified, ai}` — `ai_verified: false` means the number shown is purely local |

### Scoring weights (`computeMatchScore`)

| Signal | Weight |
|---|---|
| Category match | 40 |
| Location match | 25 |
| Central text overlap — base | 40 |
| Central text overlap — high-value item | 55 |
| Description overlap | 10 |
| Distinctive features bonus — base | 15 |
| Distinctive features bonus — high-value | 25 |
| OCR student-ID exact match | 30 |

> "High-value" small items (headphones, chargers, cables, keys) receive the **higher**
> weights because they are easily confused with each other.

### Thresholds

| Constant | Value | Meaning |
|---|---|---|
| `MATCH_AUTO_THRESHOLD` | **75** | Minimum score to create a match automatically |
| `MATCH_SUGGEST_THRESHOLD` | **45** | Minimum score to show as a suggestion to an officer |
| `MATCH_STUDENT_THRESHOLD` | **75** | Minimum score to show as a suggestion to the student |

> Low-confidence suggestions are **hidden from the student** and left for officer review.
> The matching engine is a **local deterministic algorithm** — it makes no AI call.

---

## 1.5 Journey D — Security verification & handover 🛡️

### D.1 Work queues

`security_dashboard.php` renders two queues from `buildPendingReviewItems()`:

| Queue | Contents |
|---|---|
| **Pending pickup** | FOUND reports with status `pending_pickup` or `matched` |
| **Suggestions** | Pairs scoring 45–74 that still need a human decision |

### D.2 Verify a code

| # | Step | System behaviour |
|---|---|---|
| 1 | Officer opens the **Verify** panel | — |
| 2 | Enters a code, scans an ID, or types a university ID | Sent as `code` |
| 3 | Single lookup matches **any** of | `reports.pickup_code` **OR** `users.university_id` **OR** `reports.ocr_data LIKE %code%` |
| 4 | Expand to the full pair | Follows `matches` by `pickup_code`, else by shared `pickup_code` on the opposite-type report |
| 5 | Return | Both reports **plus** the `match` row, in one response |
| 6 | No match | `404` with a clear message |

### D.3 Confirm a match (suggestion → resolution)

| # | Step | System behaviour |
|---|---|---|
| 1 | Officer clicks **Confirm** | `confirm_match` |
| 2 | Idempotent write | If a `matches` row already exists → `UPDATE`; else → `INSERT`. Re-running is safe and keeps the same pickup code |
| 3 | Resolve | `UPDATE reports SET status='resolved', pickup_code=?, resolved_at=NOW() WHERE id IN (?, ?)` — both sides at once |
| 4 | Notify | `resolved` notification to both owners |
| 5 | Audit | `activity_type = 'match_confirmed'` |

### D.4 Reject a match

| # | Step | System behaviour |
|---|---|---|
| 1 | Officer clicks **Reject** | `reject_match` |
| 2 | Preconditions | Refuses with `409` if the match is already `pending` or `verified`; `404` if the two reports are not `lost` + `found` |
| 3 | Persist the refusal | `INSERT INTO matches (..., confidence_score, pickup_code, status) VALUES (?, ?, 0, NULL, 'rejected')` |
| 4 | Effect | The pair is **permanently excluded** from future matching |

> ⚠️ `pickup_code` must be `NULL`, not `''`. A `UNIQUE` index permits many `NULL`s but
> only one empty string — using `''` would make the second rejection fail.

### D.5 Physical handover (all-or-nothing)

Executed inside a single database transaction:

```
BEGIN
  1. Resolve the pair   → status='resolved', resolved_at=NOW()   (both reports)
  2. Verify the match   → status='verified'                      (matches)
  3. RE-QUERY both reports; if either is NOT 'resolved' → ROLLBACK
  4. Insert 'resolved' notification for each owner
  5. Write activity_logs row  action_type='handover'
COMMIT   (any failure → ROLLBACK, nothing is persisted)
```

> Step 3 is the safety net: it guarantees a **partial handover can never be committed**.

---

## 1.6 Journey E — Student profile & report lifecycle 🔒

`profile.php` shows per-report statistics and a **4-step tracker**:

| Step | Arabic label | Driven by `reports.status` |
|---|---|---|
| 1 submitted | تم الإبلاغ | `active` |
| 2 smart match | تمت المطابقة | `matched` |
| 3 ready for pickup | بانتظار الاستلام | `pending_pickup` |
| 4 received | تم الاستلام | `resolved` |

The student may **close** any report that is not yet resolved. Closing cascades across the
matched pair via `collectMatchedReportIds()`, so both sides close together and no
half-closed pair is left behind.

The profile also hosts an **"الإعدادات الأمنية" (security settings)** card. The card
only shows its form after clicking the "تغيير الرمز الأمني" toggle button, so the page
stays clean; the form requires the current password and calls
`auth.php?action=set_security_code` to set or replace the recovery code. See §1.7.

---

## 1.7 Journey F — Password recovery

| # | Step | System behaviour |
|---|---|---|
| 1 | Student submits university ID **and** registered email (+ security code when the account has one) | `forgot_password` |
| 2 | **Strict identity verification** | Account resolved by university ID; the email must match the registered address exactly (case-insensitive). The security code is checked with `password_verify()` against `security_code_hash` and is required **only when the account has one** — legacy accounts created before this feature (`security_code_hash` empty) reset with ID + email alone |
| 3 | **No account enumeration** | One generic message for every outcome; a dummy bcrypt check on the no-account / no-code paths keeps response timing flat, so the endpoint reveals nothing |
| 4 | Token issued | Random token; **only `SHA-256(token)`** stored in `password_resets.token_hash` (`CHAR(64)`), with `expires_at` and a `used_at` marker |
| 5 | Delivery | Server log only, and **only** when `LUQYA_EXPOSE_RESET_TOKEN=1` (default `0`) |
| 6 | Student submits token + new password | `reset_password`, min 6 characters |
| 7 | Consume | Token marked used; password re-hashed; user logged in |

#### There is no mailer in this build

The recovery flow is deliberately **complete without a live email server**. The project
contains no SMTP configuration, no mailer dependency and no outbound mail code, so step 4
above is the only delivery mechanism that exists today.

| | Development | Production |
|---|---|---|
| Flag | `LUQYA_EXPOSE_RESET_TOKEN=1` | must stay `0` |
| Where the link goes | `error_log()`, i.e. `C:\xampp\apache\logs\error.log` | a real mailer must be added |
| Reviewer steps | open the log, grep `Luqya AI: [DEV ONLY]`, copy the `token=` value, open `forgot_password.php?step=reset&token=…` | — |

```
[Fri Oct 02 01:36:06 2026] [php:notice] Luqya AI: [DEV ONLY] password reset link for user #2
  - forgot_password.php?step=reset&token=c8c0ab1662ef5dab3103a525dd598dfb513ec9e05d0ac39f196f71c3a6c5d037
```

`forgot_password.php` renders an on-page notice (gated by the same flag) telling the
reviewer exactly this, so no email account is needed to test the feature.

> **Log path caveat.** `php.ini` sets `error_log = C:\xampp\php\logs\php_error_log`, but
> that directory does not exist in this XAMPP install, so PHP falls back to writing into
> Apache's log. The Apache path above is the verified destination. Creating
> `C:\xampp\php\logs\` or correcting `php.ini` would remove the ambiguity.

Security properties that hold in both modes: the raw token is never stored (only its
SHA-256 hash), it expires after **1800 s**, it is single-use, and requesting a new token
deletes any earlier unused token for that user. The response is a single generic message
in every case, so the endpoint cannot enumerate accounts.

Because the seeded demo students were created before the security-code feature, they are
legacy accounts and reset with university ID + email only; officers (`SEC-USR-…`, hashed
code) and any student who sets a recovery code must supply it. A logged-in user creates
or changes the code through `auth.php?action=set_security_code` (profile → security
settings), and registration accepts an optional security code stored as a bcrypt hash.

---

## 1.7.1 Quick demo login (evaluator convenience)

`login.php` can render a **"دخول تجريبي سريع"** panel so a reviewer can exercise the
system instantly instead of registering an account or reverse-engineering seed data.

| Aspect | Behaviour |
|---|---|
| Gate | `demoLoginAccounts()` in `config/db.php`; empty array when `LUQYA_DEMO_LOGIN=0` |
| Default | enabled (flag unset, or `=1`) — set `0` in production to hide it |
| Mechanism | one button per role; click copies the values into `university_id` and `password`, then the user presses the normal **تسجيل الدخول** |
| Officer demo | `SEC-USR-1` / `security123` → `security_dashboard.php` |
| Student demo | `S202403400` / `password123` → `index.php` |
| Verification | both confirmed with `password_verify()` against the seeded `users` table, and end-to-end over HTTP |

**This is a pre-fill, not a bypass.** No new endpoint and no alternate code path was
added: the click handler only writes to the two DOM inputs, and submission still goes
through `POST api/auth.php?action=login` with `requireCsrf()`,
`enforceRateLimit('login', 10, 300)` and `password_verify()` executing exactly as they
would for a hand-typed login. The flag controls **only whether the panel is rendered**.

`S202447940` is intentionally not offered as a demo: its password digest differs from
the shared student password, so no one-click credential exists for it.

---

## 1.8 Report state machine

```
                  (created)
                       │
                       ▼
                   ┌────────┐
      ┌────────────│ active │◄────────────┐
      │            └────────┘             │
      │                 │                 │
      │  automatic match│ (score ≥ 75)    │ manual status update
      │                 ▼                 │
      │            ┌──────────┐           │
      │            │ matched  │           │
      │            └──────────┘           │
      │                 │                 │
      │   confirm match │ OR handover     │
      │                 ▼                 ▼
      │          ┌───────────────┐   ┌───────────┐
      │          │ pending_pickup│   │ cancelled │
      │          └───────────────┘   └───────────┘
      │                 │
      │   officer handover / confirm match
      │                 ▼
      │           ┌──────────┐
      └──────────►│ resolved │  (terminal — resolved_at stamped)
                  └──────────┘
```

| From → To | Trigger |
|---|---|
| *new* → `active` | Report insert |
| `active` → `matched` / `pending_pickup` | Automatic match (LOST side / FOUND side) |
| `matched`\|`pending_pickup` → `resolved` | Officer handover or confirm match |
| any → `cancelled` | Manual status update |
| any → `resolved` | Owner closes own report |

**Invariant:** only `active` reports are offered to the matching engine. A report that
leaves `active` is invisible to future matching.

### `matches` state machine (separate, smaller)

| Status | Meaning |
|---|---|
| `pending` | Auto-created or suggested — awaiting an officer decision |
| `verified` | Officer confirmed; both reports closed |
| `rejected` | Officer refused; pair permanently excluded, `pickup_code` is `NULL` |

---

## 1.9 Workflow rules summary

| # | Rule |
|---|---|
| 1 | A pickup code is generated **only** for `type = 'found'` |
| 2 | A matched pair shares **one** pickup code, copied to both reports |
| 3 | The matching engine **never** resolves a report — a human always must |
| 4 | A `(lost, found)` pair can exist only once (`uk_matches_pair`) |
| 5 | A non-null pickup code can identify only one match (`uk_matches_pickup`) |
| 6 | Handover is atomic; partial persistence is impossible |
| 7 | Rejecting stores `pickup_code = NULL` and `confidence_score = 0` |
| 8 | Every AI call degrades gracefully — the platform works with no API key |
| 9 | Reports leave the matching pool as soon as they are not `active` |

---

# 2. API Documentation

## 2.1 General conventions

| Aspect | Rule |
|---|---|
| **Transport** | HTTP, `application/x-www-form-urlencoded` (except AI endpoints, which accept JSON) |
| **Action routing** | `api/<file>.php?action=<name>` |
| **Authentication** | PHP native session cookie. Helper `isLoggedIn()` / `getCurrentUser()` |
| **Authorization** | Helper `isSecurityOfficer()`; enforced per action |
| **CSRF** | `requireCsrf()` on **every** state-changing action |
| **Success envelope** | `{ "success": true, "message": "<human text>", ... }` |
| **Error envelope** | `{ "success": false, "message": "<human text>" }` |
| **SQL** | 100% PDO prepared statements; no value concatenation |
| **Output** | `escapeOutput()` on render, `sanitizeInput()` on input |
| **Language** | **Fully Arabic (RTL).** The platform interface and the entire system are designed and developed in Arabic — rendered right-to-left — to seamlessly support the Saudi university environment; messages are served as Arabic UTF-8 |

> **Response surface is allowlist-only — verified 2026-09-28.** A full audit of every
> `jsonResponse()` call across the API found **no** `internal_*` keys, no debug keys, and no
> private data. Specifically confirmed clean:
>
> - **No secrets in responses.** `password_hash`, `security_code_hash` and
>   `security_code` are read only inside `password_verify()` / migration code. The
>   password-reset token is written to `error_log()`, never to a response body.
> - **No exception detail leaks.** Every `catch` block logs `$e->getMessage()` via
>   `error_log()` and returns a generic Arabic message.
> - **No raw row dumps.** `getCurrentUser()` returns a fixed 9-field allowlist, not the
>   session or the DB row. `ai_search.php` selects an explicit column list. The remaining
>   `SELECT *` sites target `reports` / `notifications`, which hold no secrets.
> - **No debug artefacts.** No `print_r`, `var_dump`, `var_export`, stack traces, SQL text
>   or filesystem paths in any response.
>
> `gemini_error` in the officer-only `ai_analyze_matches` response is *deliberate*: it is
> gated by `isSecurityOfficer()` + CSRF, it never contains the API key (which travels in a
> header), and the dashboard needs it to explain why AI analysis failed.

---

## 2.2 Rate limits

| Endpoint | Limit | Window |
|---|---|---|
| `auth.php?action=register` | 5 | 3600 s |
| `auth.php?action=login` | 10 | 300 s |
| `auth.php?action=forgot_password` | 5 | 300 s |
| `auth.php?action=reset_password` | 5 | 900 s |
| `auth.php?action=set_security_code` | 5 | 900 s |
| `ai_chat.php` | 30 | 300 s |
| `ai_search.php` | 20 | 300 s |

Enforced per client IP via `enforceRateLimit($bucket, $limit, $window)`.

---

## 2.3 Error codes

| Code | Meaning | Typical cause |
|---|---|---|
| `200` | OK | — |
| `400` | Bad request | Validation failed, field too long, bad university ID, missing required field |
| `401` | Unauthorised | Not logged in |
| `403` | Forbidden | Wrong role (e.g. officer creating a LOST report), or IDOR attempt |
| `404` | Not found | Unknown report / no match for the code |
| `405` | Method not allowed | `GET` sent to a POST-only AI endpoint |
| `409` | Conflict | Rejecting a match that is already `pending` or `verified` |
| `429` | Too many requests | Rate limit exceeded (also surfaced by the Gemini layer) |
| `500` | Server error | DB unavailable, AI provider failure, or any uncaught exception |

---

## 2.4 Authentication endpoints (`api/auth.php`)

### 2.4.1 `POST ?action=register`

Rate limit 5 / 3600 s. **No auth required.**

**Request**

| Field | Type | Required | Rules |
|---|---|---|---|
| `university_id` | string | ✔ | `S` + exactly 9 digits; unique |
| `full_name` | string | ✔ | ≤ 100 chars, XSS-sanitised |
| `phone` | string | — | ≤ 20 chars; digits, `+`, spaces, dashes |
| `college` | string | — | ≤ 100 chars |
| `major` | string | — | ≤ 100 chars |
| `password` | string | ✔ | ≥ 6 chars; stored as bcrypt |
| `csrf_token` | string | ✔ | — |

**Response `200`**
```json
{ "success": true, "message": "تم إنشاء الحساب بنجاح!", "redirect": "index.php" }
```

**Response `400`**
```json
{ "success": false, "message": "رقم الطالب غير صحيح. يجب أن يبدأ بحرف S متبوعاً بـ 9 أرقام." }
```

> Role is **forced** to `student` server-side. The institutional email is derived
> automatically (`S202403400` → `s202403400@uoh.edu.sa`); it cannot be supplied by the client.

---

### 2.4.2 `POST ?action=login`

Rate limit 10 / 300 s. **No auth required.**

**Request**

| Field | Type | Required | Rules |
|---|---|---|---|
| `university_id` | string | ✔ | University ID **or** institutional email |
| `password` | string | ✔ | Verified against the bcrypt hash |
| `csrf_token` | string | ✔ | — |

**Response `200`**
```json
{ "success": true, "message": "تم تسجيل الدخول بنجاح!", "redirect": "security_dashboard.php" }
```

> `redirect` is role-aware via `homePathForRole($user)` — officers land on the dashboard.
> The session ID is regenerated on success.

> **Demo shortcut.** When `LUQYA_DEMO_LOGIN` is on, `login.php` pre-fills this request's
> two fields from a button, but the call is still this endpoint with this same CSRF and
> rate-limit enforcement — there is no `action=demo_login` and no bypass. See §1.7.1.

---

### 2.4.3 `POST ?action=forgot_password`

Rate limit 5 / 300 s. **No auth required.**

| Field | Type | Required | Rules |
|---|---|---|---|
| `university_id` | string | ✔ | University ID (`S…`) or officer login code (`SEC-USR-…`); upper-cased |
| `email` | string | ✔ | Registered university email; matched lower-cased against the stored address |
| `security_code` | string | — | Required **only** when the account has one (`security_code_hash` set); ignored for legacy accounts |
| `csrf_token` | string | ✔ | — |

**Strict identity verification.** The account is resolved by university ID, then the
supplied email must match the registered address exactly. If the account has a security
code it is verified with `password_verify()` against `security_code_hash`; legacy
accounts created before this feature (empty hash) pass on university ID + email alone,
so they can never be locked out. A dummy bcrypt check runs on the no-account and
no-code paths to keep response timing flat.

**Response `200`** — identical whether or not the account exists (prevents enumeration):
```json
{ "success": true, "message": "تم إرسال تعليمات إعادة تعيين كلمة المرور إلى بريدك الجامعي." }
```

---

### 2.4.4 `POST ?action=reset_password`

Rate limit 5 / 900 s. **No auth required.**

| Field | Type | Required | Rules |
|---|---|---|---|
| `token` | string | ✔ | Raw token; matched against the stored SHA-256 |
| `password` | string | ✔ | ≥ 6 chars |
| `csrf_token` | string | ✔ | — |

**Response `200`**
```json
{ "success": true, "message": "تم تحديث كلمة المرور بنجاح.", "redirect": "index.php" }
```

---

### 2.4.5 `POST ?action=set_security_code` 🔒

Rate limit 5 / 900 s. **Requires login.**

| Field | Type | Required | Rules |
|---|---|---|---|
| `current_password` | string | ✔ | Verified against the account's bcrypt hash before any change |
| `security_code` | string | ✔ | 6–64 chars; stored only as a bcrypt hash |
| `security_code_confirm` | string | ✔ | Must equal `security_code` |
| `csrf_token` | string | ✔ | — |

Creates or replaces the user's recovery security code used by `forgot_password`. The
code is never returned or logged.

**Response `200`**
```json
{ "success": true, "message": "تم حفظ الرمز الأمني بنجاح." }
```

---

## 2.5 Reports endpoints (`api/reports.php`)

### 2.5.1 `POST ?action=create` 🔒

| Field | Type | Required | Max | Rules |
|---|---|---|---|---|
| `type` | string | ✔ | — | `lost` \| `found`; defaults to `lost` |
| `title` | string | ✔ | 200 | — |
| `category` | string | ✔ | 50 | — |
| `description` | string | ✔ | 4000 | AI-enhanced server-side |
| `location` | string | ✔ | 250 | — |
| `date_event` | string | ✔ | — | `YYYY-MM-DD`; defaults to today |
| `image_url` | string | — | 255 | Normalised + allow-listed |
| `ai_tags` | string | — | 500 | — |
| `ocr_data` | string | — | — | JSON from ID-card scan |
| `distinctive_features` | string | — | 2000 | — |
| `csrf_token` | string | ✔ | — | — |

**Response `200`**
```json
{
  "success": true,
  "message": "تم إرسال البلاغ بنجاح!",
  "report_id": 29,
  "pickup_code": "RET-482913",
  "match_info": {
    "matched": true,
    "pickup_code": "RET-482913",
    "confidence": 96,
    "lost_report_id": 29,
    "found_report_id": 25
  }
}
```

**Errors**

| Code | Cause |
|---|---|
| `401` | Not logged in |
| `403` | Security officer attempting `type=lost` |
| `400` | Missing `title`/`description`/`location`, or a field over its max length |

> `pickup_code` is `null` for a LOST report unless `match_info.matched` is `true`.

---

### 2.5.2 `POST ?action=close` 🔒

| Field | Type | Required |
|---|---|---|
| `report_id` | int | ✔ |
| `csrf_token` | string | ✔ |

**Response `200`** — `updated_report_ids` includes the whole matched pair:
```json
{
  "success": true,
  "message": "تم إغلاق البلاغ.",
  "updated_report_ids": [26, 25]
}
```

**Errors:** `401` not logged in · `404` not the owner · `403` IDOR attempt

---

### 2.5.3 `GET ?action=get&id=<int>` 🔒

Returns the report joined with the owner's `full_name`, `university_id`, `email`.

```json
{ "success": true, "report": { "id": 29, "user_id": 2, "type": "lost", "...": "..." } }
```

**Errors:** `401` not logged in · `403` not the owner and not an officer · `404` unknown id

---

### 2.5.4 `GET ?action=my_status` 🔒

```json
{
  "success": true,
  "reports": [
    { "id": 29, "status": "matched",  "pickup_code": "RET-482913", "resolved_at": null },
    { "id": 26, "status": "resolved", "pickup_code": "RET-372629", "resolved_at": "2026-09-18 01:40:30" }
  ]
}
```

---

### 2.5.5 `GET ?action=statuses`

Returns the permitted status values (used to populate filters).

---

### 2.5.6 `GET ?action=potential_matches` 🔒

High-confidence candidates for the caller's own LOST reports, threshold
`MATCH_STUDENT_THRESHOLD` (75), capped at 8.

---

## 2.6 Security dashboard endpoints (`api/security.php`)

> All write actions 🛡️ require the `security` role **and** a CSRF token.
> All return `403` otherwise.

### 2.6.1 `GET ?action=pending_count` 🛡️

```json
{ "success": true, "count": 5 }
```

### 2.6.2 `POST ?action=verify` 🛡️

| Field | Type | Required | Rules |
|---|---|---|---|
| `code` | string | ✔ | Pickup code, university ID, or a value inside `ocr_data`; upper-cased |
| `csrf_token` | string | ✔ | — |

**Response `200`**
```json
{
  "success": true,
  "code": "RET-849201",
  "reports":  [ { "id": 1,  "type": "lost",  "...": "..." } ],
  "match":    { "id": 1, "lost_report_id": 1, "found_report_id": 2,
                "confidence_score": 96, "status": "verified", "...": "..." }
}
```

**Response `404`**
```json
{ "success": false, "message": "لم يتم العثور على أي بلاغ مطابق للكود: RET-000000" }
```

---

### 2.6.3 `POST ?action=handover` 🛡️

| Field | Type | Required | Rules |
|---|---|---|---|
| `pickup_code` | string | one of | Upper-cased |
| `report_id` | int | one of | Match id, lost id, or found id |
| `csrf_token` | string | ✔ | — |

> At least one of `pickup_code` / `report_id` must be present, else `400`.

**Response `200`**
```json
{
  "success": true,
  "message": "تم تسليم العنصر بنجاح (2 بلاغ).",
  "updated_report_ids": [1, 2],
  "reports": [ { "id": 1, "status": "resolved", "...": "..." } ]
}
```

**Response `500`** — transaction rolled back, nothing persisted:
```json
{ "success": false, "message": "فشل التسليم: <reason>" }
```

---

### 2.6.4 `POST ?action=add_found` 🛡️

Same field set as `reports.php?action=create` (see [2.5.1](#251-post-actioncreate-)), but
`type` is implicitly `found`, defaults are officer-friendly, and a pickup code is always
generated.

---

### 2.6.5 `POST ?action=update_found` 🛡️

Adds `report_id` and an **optional** `status`.

> ⚠️ If `status` is **omitted**, the report's current status is **preserved** — it is not
> reset to `active`. An invalid status is rejected with `400`.

---

### 2.6.6 `POST ?action=delete_found` 🛡️

| Field | Type | Required |
|---|---|---|
| `report_id` | int | ✔ |
| `csrf_token` | string | ✔ |

---

### 2.6.7 `GET ?action=list_pending_matches` 🛡️

```json
{
  "success": true,
  "items": [
    { "kind": "pending_pickup", "lost": { }, "found": { }, "confidence": 99 },
    { "kind": "suggestion",     "lost": { }, "found": { }, "confidence": 62 }
  ]
}
```

> `kind` is either `pending_pickup` or `suggestion`.

---

### 2.6.8 `POST ?action=ai_analyze_matches` 🛡️

Runs Gemini over the queued pairs (max 6 per call). Results are cached in
`$_SESSION['ai_match_cache']` keyed `"<lostId>-<foundId>"`.

| Field | Type | Required |
|---|---|---|
| `lost_id` | int | ✔ |
| `found_id` | int | ✔ |
| `csrf_token` | string | ✔ |

```json
{
  "success": true,
  "lost_id": 26,
  "found_id": 25,
  "confidence": 98,
  "reason": "كلمة حمراء على العلبة، نفس الماركة، نفس اليوم",
  "ai_powered": true
}
```

If Gemini is unavailable the local deterministic score is returned instead, with
`"ai_powered": false` — the officer still gets a usable number.

---

### 2.6.9 `POST ?action=confirm_match` 🛡️

| Field | Type | Required |
|---|---|---|
| `lost_id` | int | ✔ |
| `found_id` | int | ✔ |
| `csrf_token` | string | ✔ |

Resolves both reports to `resolved`, assigns the shared pickup code, notifies both owners,
and writes `match_confirmed` to the audit log. **Idempotent** — `UPDATE` if a match row
exists, `INSERT` otherwise.

---

### 2.6.10 `POST ?action=reject_match` 🛡️

| Field | Type | Required |
|---|---|---|
| `lost_id` | int | ✔ |
| `found_id` | int | ✔ |
| `csrf_token` | string | ✔ |

**Response `200`**
```json
{ "success": true, "message": "تم رفض المطابقة." }
```

**Errors:** `409` match already `pending`/`verified` · `404` reports are not a `lost`+`found` pair

---

## 2.7 Notifications endpoints (`api/notifications.php`)

### 2.7.1 `GET` (default action) 🔒

```json
{
  "unread_count": 3,
  "notifications": [
    { "id": 11, "user_id": 2, "type": "match_found", "title": "…", "message": "…",
      "is_read": 0, "created_at": "2026-09-17 02:18:48" }
  ]
}
```

> Returns the **latest 10** notifications plus the total unread count.
> For a **visitor who is not logged in** it returns `{"unread_count": 0, "notifications": []}`
> with `200` — deliberately **not** an error, so the public header never breaks.
> Note this response has no `success` key.

### 2.7.2 `POST ?action=mark_read` 🔒

Marks **all** of the user's notifications as read.

```json
{ "success": true, "unread_count": 0 }
```

---

## 2.8 AI endpoints

All three are **POST-only** (`GET` → `405`), require a logged-in session, and accept either
form-encoded POST or a JSON body.

### 2.8.1 `POST api/vision_ai.php`

| Field | Type | Required | Notes |
|---|---|---|---|
| `mode` | string | — | `general_item` (default) \| `id_card` |
| `report_type` | string | — | `lost` \| `found`; defaults to `lost` |
| `image` | file | one of | ≤ 8 MB; `jpg`/`png`/`webp` |
| `image_base64` | string | one of | Alternative to upload; MIME re-detected |

**Response `200`**
```json
{
  "success": true,
  "ai_powered": true,
  "data": {
    "mode": "id_card",
    "report_type": "lost",
    "category": "البطاقة الجامعية / الصراف",
    "title": "بطاقة جامعية",
    "tags": ["بطاقة جامعية", "جامعة حائل", "QR"],
    "description": "بطاقة جامعية بيضاء بزرقاء وذهبي…",
    "student_id": "S202402200",
    "image_path": "uploads/item_1789599745_a1b2c3d4.jpg"
  }
}
```

**Response `500` — no API key configured** (image already saved, so the UI can still show it):
```json
{
  "success": false,
  "ai_powered": false,
  "error_type": "missing_api_key",
  "message": "لم يتم ضبط مفتاح GEMINI_API_KEY في النظام أو ملف .env",
  "image_path": "uploads/item_1789599745_a1b2c3d4.jpg"
}
```

> Note the ordering: the image is **uploaded and validated before** the AI call, so a
> Gemini outage never loses the user's photo.

### 2.8.2 `POST api/ai_search.php`

Rate limit 20 / 300 s.

| Field | Type | Required | Max |
|---|---|---|---|
| `query` | string | ✔ | 500 chars |
| `csrf_token` | string | ✔ | — |

**Response `200`**
```json
{
  "success": true,
  "query": "محفظة جلدية سوداء",
  "ids": [1, 2],
  "reason": "المحافظتان تتطابقان في اللون والماركة والموقع.",
  "reports": [ { "id": 1, "type": "lost", "title": "محفظة جلدية سوداء" } ],
  "count": 2
}
```

**Response `200` — no results** (still a success, not an error):
```json
{ "success": true, "ids": [], "reason": "لم يتم العثور على نتائج مطابقة.", "reports": [], "count": 0 }
```

> Candidate pool: the **50** most recent reports with status in
> `('active','matched','pending_pickup')`.

### 2.8.3 `POST api/ai_chat.php`

Rate limit 30 / 300 s.

| Field | Type | Required | Max |
|---|---|---|---|
| `message` | string | ✔ | 1000 chars |
| `history` | array | — | Prior turns for context |
| `csrf_token` | string | ✔ | — |

**Response `200`**
```json
{
  "success": true,
  "reply": "يبدو أنك فقدت سماعة رأس زرقاء، لدينا بلاغ مطابق بنسبة 99%.",
  "reports_count": 12,
  "matched_reports_count": 1,
  "matched_report_ids": [26],
  "source": "gemini-grounded-database"
}
```

**`source` values** — always tells you which path answered:

| Value | Meaning |
|---|---|
| `gemini-grounded-database` | Gemini answered, grounded on real DB rows |
| `luqya-assistant` | Deterministic canned reply (greeting, general help) |
| `luqya-assistant-guidance` | Deterministic "how to report a found item" guidance |
| `luqya-database-lookup` | Answered from a direct DB lookup by report ID |
| `luqya-database-search` | Answered from a local keyword search — **Gemini failed** |

**Response `200` — Gemini unavailable** (note: still `200`, with an explicit flag):
```json
{
  "success": true,
  "reply": "…",
  "reports_count": 12,
  "matched_reports_count": 0,
  "matched_report_ids": [],
  "source": "luqya-database-search",
  "gemini_unavailable": true,
  "gemini_error": "لم يتم ضبط مفتاح GEMINI_API_KEY في النظام أو ملف .env"
}
```

---

## 2.9 Master endpoint index

| # | Method | Endpoint | 🔒 | CSRF | Rate limit |
|---|---|---|---|---|---|
| 1 | POST | `auth.php?action=register` | — | ✔ | 5 / 3600 s |
| 2 | POST | `auth.php?action=login` | — | ✔ | 10 / 300 s |
| 3 | POST | `auth.php?action=forgot_password` | — | ✔ | 5 / 300 s |
| 4 | POST | `auth.php?action=reset_password` | — | ✔ | 5 / 900 s |
| 5 | POST | `auth.php?action=set_security_code` | ✔ | ✔ | 5 / 900 s |
| 6 | POST | `auth.php?action=logout` | ✔ | ✔ | — |
| 7 | POST | `reports.php?action=create` | ✔ | ✔ | — |
| 8 | POST | `reports.php?action=close` | ✔ | ✔ | — |
| 9 | GET | `reports.php?action=get` | ✔ | — | — |
| 10 | GET | `reports.php?action=my_status` | ✔ | — | — |
| 11 | GET | `reports.php?action=statuses` | — | — | — |
| 12 | GET | `reports.php?action=potential_matches` | ✔ | — | — |
| 13 | GET | `security.php?action=pending_count` | 🛡️ | — | — |
| 14 | POST | `security.php?action=verify` | 🛡️ | ✔ | — |
| 15 | POST | `security.php?action=handover` | 🛡️ | ✔ | — |
| 16 | POST | `security.php?action=add_found` | 🛡️ | ✔ | — |
| 17 | POST | `security.php?action=update_found` | 🛡️ | ✔ | — |
| 18 | POST | `security.php?action=delete_found` | 🛡️ | ✔ | — |
| 19 | GET | `security.php?action=list_pending_matches` | 🛡️ | — | — |
| 20 | POST | `security.php?action=ai_analyze_matches` | 🛡️ | ✔ | — |
| 21 | POST | `security.php?action=confirm_match` | 🛡️ | ✔ | — |
| 22 | POST | `security.php?action=reject_match` | 🛡️ | ✔ | — |
| 23 | GET | `notifications.php` | ✔ | — | — |
| 24 | POST | `notifications.php?action=mark_read` | ✔ | ✔ | — |
| 25 | POST | `vision_ai.php` | ✔ | — | — |
| 26 | POST | `ai_search.php` | ✔ | ✔ | 20 / 300 s |
| 27 | POST | `ai_chat.php` | ✔ | ✔ | 30 / 300 s |

---

# 3. Gemini AI Integration

## 3.1 Where it is used

| Call site | Wrapper | Purpose | Degrades to |
|---|---|---|---|
| `api/vision_ai.php` | `callGeminiVisionAPI()` | Image analysis: tagging, title, category, description; ID-card OCR | Saves the image, returns `ai_powered: false` + `error_type` |
| `api/reports.php:93` | `enhanceDescriptionWithGemini()` | Rewrites a submitted description into a richer one | Original description kept |
| `api/ai_search.php:55` | `geminiSmartSearch()` | Natural-language search over report candidates | Empty result set + `reason` |
| `api/ai_chat.php` | `callGeminiAPI()` | Conversational assistant, grounded on the user's reports | Local keyword search, `source: luqya-database-search` |
| `api/security.php` | `geminiBatchMatchAnalysis()` | On-demand re-analysis of candidate pairs for the officer | Local deterministic score, `ai_powered: false` |
| `config/matching.php` | `geminiSemanticVerdict()` / `geminiSemanticScore()` / `geminiBatchMatchAnalysis()` | **Automatic semantic re-ranking inside the matching engine** | Local score kept verbatim, `ai: null` |

### 3.1.1 The matching engine is AI-powered (hybrid, not AI-only)

`config/matching.php` runs a **two-stage** engine. The deterministic weighted scorer
(`computeMatchScore()`) always runs first and is never skipped; the Gemini layer re-ranks
its shortlist on top.

```
findMatchingCandidates()            ← local weighted score, always
        │
        ▼
semanticRerankCandidates()          ← the AI stage
        │  • skips entirely if no GEMINI_API_KEY or AI_MATCHING_ENABLED=0
        │  • only candidates scoring >= MATCH_AI_MIN_LOCAL (45) are paid for
        │  • at most MATCH_AI_MAX_PAIRS (5) pairs, in ONE API call
        │  • 1 pair  → geminiSemanticVerdict() + geminiSemanticScore()  (one call, memoised)
        │  • N pairs → geminiBatchMatchAnalysis()                      (batched)
        │
        ▼
blend = round(local × (1 − W) + ai × W),  W = MATCH_AI_BLEND_WEIGHT (0.40)
        │
        ▼
re-sorted; compared against MATCH_AUTO_THRESHOLD (75)
```

Consequences of the blend, which are the intended safety properties:

| Scenario | local | ai | blended | Effect |
|---|---|---|---|---|
| Agreement | 90 | 88 | 89 | Match stands |
| **AI veto** — same words, different item | 80 | 20 | 56 | Falls below 75, so no auto-match |
| **AI rescue** — reworded description | 55 | 98 | 72 | Can climb, but not past 75 alone |
| AI alone cannot create a match | 45 | 100 | 67 | Still below 75 |

The AI can therefore **lower** a match below the auto threshold, and can **rescue** an
under-rated pair, but it can never on its own authorise a match: the local scorer remains
a hard prerequisite.

Active call sites of the AI stage:

- `runAutomaticMatching()` — on report creation. When the layer is active it shortlists from
  `MATCH_AI_MIN_LOCAL` (45) instead of `MATCH_AUTO_THRESHOLD` (75), so the blend can promote
  a pair the local scorer underrated; the final decision still requires blended ≥ 75.
- `getPotentialMatchesForUser()` — student dashboard.
- `buildPendingReviewItems()` — officer review queue; the officer's own `ai_analyze_matches`
  result (session cache) takes precedence over the engine's automatic verdict.

Each re-ranked item exposes `local_score`, `score` (the blended value used for thresholds),
and `ai` = `{score, reason, flags, signals, local_score, blended_score, disagreement,
strong_disagreement}`.

### 3.1.2 Cost and latency control

- `MATCH_AI_MAX_CALLS_PER_REQUEST` (2) is a **hard per-request ceiling**, enforced by
  `claimSemanticAiBudget()`. It is per-request, not per-call-site, so a page that scores many
  reports cannot fan out into many API calls.
- `MATCH_AI_MIN_LOCAL` (45) is a spend floor: a pair the local scorer barely likes is not
  worth a paid model call.
- `geminiSemanticVerdict()` memoises per pair for the life of the request, so calling
  `geminiSemanticScore()` right after it is free — it reads the memo, it does not re-call.
  The pair is also ordered by report `type` internally, so the memo hit is order-independent.
- Both the same-pair success **and the same-pair failure** are memoised, which prevents a
  retry storm while Gemini is down.
- Timeouts: `GEMINI_TIMEOUT` 25 s for text, `GEMINI_VISION_TIMEOUT` 60 s for images.

### 3.1.3 Graceful degradation

`isSemanticMatchingEnabled()` is `false` when `GEMINI_API_KEY` is unset or
`AI_MATCHING_ENABLED=0`. In that case `semanticRerankCandidates()` returns the candidate list
untouched (scores unchanged, `ai: null`, `local_score` backfilled) and the platform runs as a
pure deterministic engine. No report submission, match, verification or handover path fails
because of a missing or broken Gemini key. `runAutomaticMatching()` additionally returns
`ai_verified`, `ai_score` and the full `ai` payload so the caller can tell whether the number
it is showing was AI-blended or purely local.

---

## 3.2 Centralised wrapper functions (`config/gemini.php`)

| Function | Returns | Notes |
|---|---|---|
| `getGeminiEndpoint($model)` | `string` | Builds the REST URL for a model |
| `isGeminiAvailable()` | `bool` | Cheap key-presence check, no network call |
| `callGeminiAPI($prompt, $maxTokens = 512)` | `?string` | **Core text call.** `null` on any failure |
| `callGeminiVisionAPI($b64, $mime, $prompt, $genCfg = [])` | `array` | **Core vision call.** Shared by `vision_ai.php` and `analyzeImageWithGeminiVision()`; returns `{success, data, http_code, model}` or `{success, error_type, message, http_code}` |
| `analyzeImageWithGeminiVision($imgB64, $mime, $mode, $type)` | `?array` | Canonical vision analysis; builds the description and tags in PHP from confirmed facts only |
| `enhanceDescriptionWithGemini($title, $category, $desc)` | `string` | Always returns a usable string |
| `geminiSemanticVerdict($a, $b)` | `array` | Single-pair semantic review. `{available, score, reason, mismatches, signals, error, cached}`. Memoised per pair per request |
| `geminiSemanticScore($r1, $r2)` | `int` | `0..100` int accessor over the same memo; order-independent |
| `geminiReportLine($report)` | `string` | One-line serialisation for prompt grounding (injection-safe) |
| `geminiBatchMatchAnalysis($pairs, $maxPairs = 6)` | `array` | Analyses several pairs per call (batching) |
| `geminiMatchAnalysisOfPair($lost, $found)` | `array` | Single-pair convenience wrapper |
| `geminiSmartSearch($query, $reports)` | `array` | `{ ids: int[], reason: string }` |
| `getGeminiLastError()` | `array` | Last failure: `curl_error`, `http_code`, `api_message` |
| `setGeminiLastError(...)` | `array` | Records a transport/API failure |
| `setGeminiLocalError($msg)` | `array` | Records a **pre-flight** failure (missing key, empty image) without misreporting it as a cURL error |
| `decodeGeminiJson($text)` | `?array` | Tolerant JSON extraction: raw, fenced, prose-wrapped |
| `extractGeminiText($res, $code, $curlErr)` | `array` | `{ok, text, http_code, error}`; surfaces safety-block reasons |
| `normalizeGeminiStudentId($v)` | `string` | Canonicalises OCR student IDs to `S\d{9}`: converts Arabic-Indic digits, strips separators, uppercases, rejects junk |
| `sanitizePromptField($v, $max)` | `string` | Flattens newlines (blocks fake prompt sections) and truncates |

---

## 3.3 Configuration & model fallback

| Variable | Default | Purpose |
|---|---|---|
| `GEMINI_API_KEY` | *(unset)* | API key. Read from `.env` or the process environment. **Never committed** |
| `GEMINI_MODEL` | `gemini-3-flash-preview` | Primary model. A value of `gemini-1.5-flash` is silently upgraded to the default |
| `LUQYA_EXPOSE_RESET_TOKEN` | `0` | Dev-only; logs the raw password-reset token |

**Fallback chain**, tried in order:

```
gemini-3-flash-preview → gemini-3.6-flash → gemini-flash-latest
                        → gemini-3.5-flash → gemini-3.1-flash-lite
```

| Behaviour | Value |
|---|---|
| Fallback triggers on | HTTP **404**, **429**, **503** only |
| Other HTTP errors | Returned to the caller immediately, no retry |
| Text timeout | `GEMINI_TIMEOUT` = **25 s** |
| Vision timeout | `GEMINI_VISION_TIMEOUT` = **60 s** (image inference legitimately needs longer) |
| Failure record | `error_log()` + `getGeminiLastError()` |
| Pre-flight failures | `setGeminiLocalError()` — `http_code: 0`, no fake cURL error, so the UI can say "key not configured" instead of "connection failed" |

### AI matching tuning constants (`config/matching.php`)

| Constant | Default | Purpose |
|---|---|---|
| `MATCH_AI_BLEND_WEIGHT` | `0.40` | How much the model's opinion counts inside the blend |
| `MATCH_AI_MAX_PAIRS` | `5` | Candidates analysed in a single call |
| `MATCH_AI_MIN_LOCAL` | `45` | Spend floor — below this local score AI is not called |
| `MATCH_AI_MAX_CALLS_PER_REQUEST` | `2` | Hard per-request Gemini call ceiling |
| `MATCH_AI_VETO_DELTA` | `25` | Local↔AI gap flagged as `strong_disagreement` for the officer |

`AI_MATCHING_ENABLED=0` in the environment disables the whole semantic layer without a code
change. See `.env.example`.

> Batching (`geminiBatchMatchAnalysis`, max 6 pairs per call) is a deliberate cost
> control: one round trip instead of six.

---

## 3.4 Prompt construction approach

All prompts follow the same four-part pattern: **role → task → grounded context → output format**.

### 3.4.1 Vision (`api/vision_ai.php`)

```
You are a campus lost-and-found assistant.            ← role
Analyse the attached image.                           ← task
Mode: {general_item | id_card}                        ← task parameter
Report type: {lost | found}

Return ONLY a JSON object with exactly these keys:   ← output format
{ "title", "category", "tags": [...], "description", "student_id" }
```

The image is sent as base64 in `inlineData`:

```json
"parts": [
  { "text": "<prompt>" },
  { "inlineData": { "mimeType": "image/jpeg", "data": "<base64>" } }
]
```

> Output is **defensively extracted**: the code locates the JSON object inside the reply
> with a regex and `json_decode`s it, so surrounding prose or Markdown fences do not break
> parsing.

### 3.4.2 Smart search (`geminiSmartSearch`)

```
Which of the following reports match this query: "<user query>"   ← task
<pickup_code> | <type> | <title> | <category> | <location> | <description>
…
Return ONLY JSON: { "ids": [1,2], "reason": "<one sentence>" }
```

> Records are **pre-serialised** by `geminiReportLine()` (ID, type, title, category,
> location, description truncated to 80 chars) so the prompt stays small. The model can
> only choose from IDs that were actually supplied — hallucinated IDs are not filtered
> client-side, so validate before use.

### 3.4.3 Chat assistant (`api/ai_chat.php`)

```
You are the Luqya AI assistant. Answer ONLY from the reports below.
If the answer is not present, say so plainly.
<the user's own reports, serialised>
<optional: the referenced report, serialised in full>
<conversation history>
Question: "<user message>"
```

> **Grounding is the whole point.** The model receives real DB rows, and the response
> carries `matched_report_ids` so the UI can link straight to the referenced report.

### 3.4.4 Batch match analysis (`geminiBatchMatchAnalysis`)

```
For each pair below, decide whether it is the same item.
Return ONLY a JSON array, one object per pair, in the same order:
[ { "pair": 1, "confidence": 0-100, "reason": "<one sentence>" } ]
```

> Confidence is clamped by `normalizeConfidenceScore()` and further bounded by the
> database `CHECK (confidence_score BETWEEN 0 AND 100)`.

---

## 3.5 Response handling

| Step | Action |
|---|---|
| 1 | `curl_exec` with `CURLOPT_TIMEOUT` set |
| 2 | Non-2xx → `setGeminiLastError()`; if the code is 404/429/503 **and** models remain, advance down the fallback chain |
| 3 | cURL transport error → recorded, `null` returned |
| 4 | Reply is wrapped in Markdown fences → locate the JSON with a regex, then `json_decode` |
| 5 | Missing or malformed keys → safe defaults, never a fatal error |
| 6 | Every field is passed through `sanitizeInput()` before it reaches the page |
| 7 | 60-character placeholders such as "غير واضح" are treated as "not recognised" and replaced |

### Standard failure shape

```json
{
  "curl_error": "",
  "curl_errno": 0,
  "http_code": 429,
  "response": "<raw body>",
  "api_message": "…",
  "message": "…"
}
```

Retrieved with `getGeminiLastError()`.

---

## 3.6 Graceful degradation

**Requirement:** the platform is fully functional with **no** AI provider at all.

```
GEMINI_API_KEY missing
        │
        ▼
callGeminiAPI() → sets error → error_log() → returns null
        │
        ├─► enhanceDescription  → original description preserved
        ├─► vision_ai           → 200/500 with ai_powered:false, image still saved
        ├─► ai_search           → empty ids[] + explanatory reason
        ├─► ai_chat             → local keyword search, HTTP 200,
        │                         source: luqya-database-search,
        │                         gemini_unavailable: true
        └─► security analysis   → local deterministic score, ai_powered: false
```

| Layer | Without Gemini |
|---|---|
| Reporting, matching, verification, handover, notifications | ✅ **Fully working** — none of it depends on AI |
| Image tagging / OCR | ❌ Manual entry |
| Description enhancement | ❌ Original kept |
| Natural-language search | ❌ Empty result + reason |
| Assistant | ✅ Deterministic canned replies |
| Match analysis | ✅ Local weighted score |

---

## 3.7 Security & cost controls

| Control | Implementation |
|---|---|
| Key isolation | Read from the environment only; never hard-coded or committed |
| Output sanitisation | `sanitizeInput()` on every field before render |
| Path allow-list | `normalizeImagePath()` rejects remote URLs, traversal, non-image extensions, and double extensions like `x.php.jpg` |
| Upload validation | Real MIME via `finfo` (not the client header); extension **derived from the detected type**; random filename |
| Size limit | 8 MB (`isValidImageUpload`, and enforced again in `vision_ai.php`) |
| Allowed formats | `image/jpeg`, `image/png`, `image/webp` |
| Rate limiting | Per-IP limits on every AI endpoint (see [2.2](#22-rate-limits)) |
| Batching | Max 6 pairs per Gemini round trip |
| Token caps | `maxTokens` bounded per call: 512 default, 4096 for the assistant |
| Prompt injection | User text is inserted as **data inside a structured prompt**, and the model is constrained to a fixed JSON output shape |
| Logging | Only error metadata is logged — never the key, never the raw token (unless the dev flag is explicitly on) |

### 3.7.1 Response-surface audit (run this after touching any endpoint)

`audit_api_surface.php` is a dependency-free static auditor that parses every
`jsonResponse()` call in `api/`, `config/`, `views/` and the root pages, and fails the
build on `internal_*` keys, debug artefacts, credentials, superglobal dumps, or stray
`print_r`/`var_dump`/`var_export`.

```bash
php audit_api_surface.php   # exit 0 = clean, 1 = breach (prints file, reason, matched key)
```

Current state — 2026-09-28: **24 files, 125 `jsonResponse()` calls, 0 violations.**

Limits of the check, stated plainly: it matches *key names*, not values, so
`'user' => $row` is reported as a warning rather than a breach (it is only a breach if
`$row` is a raw DB row — `getCurrentUser()` is an allowlist and is safe). It does not
inspect rendered HTML or inline JavaScript, and it only follows `jsonResponse()`. It
backs up the invariant; it does not replace review.

---

# 4. Appendix — Glossary

| Term | Meaning |
|---|---|
| **Pickup code** | `RET-XXXXXX`, the code a student shows at the security desk. Generated for FOUND reports and shared by both sides of a match |
| **Match** | A row linking exactly one LOST report to one FOUND report |
| **Confidence** | 0–100 score for a candidate pair, bounded by a DB `CHECK` constraint |
| **Suggestion** | A pair scoring 45–74, held for officer review — never auto-applied |
| **Handover** | The officer's act of physically releasing an item; runs in one transaction |
| **`general_item` mode** | Vision mode for a general object photo |
| **`id_card` mode** | Vision mode for a university ID or ATM card → OCR into `ocr_data` |
| **`ocr_data`** | JSON blob holding OCR output, e.g. `{"student_id":"S202402200","name":"…"}` |
| **Graceful degradation** | Returning a usable result without the optional dependency, instead of an error |

---

# 5. Appendix — Known gaps

| # | Gap | Impact |
|---|---|---|
| 1 | `README.md` still describes the AI as "simulated" | Misleading — Gemini is genuinely wired in |
| 2 | *(resolved 2026-10-02)* `README.md` quoted demo credentials that no longer matched the seed data | Corrected and `password_verify()`-verified; `login.php` now has a gated quick demo panel (§1.7.1) |
| 3 | `README.md` says 6-digit codes and an "85%" threshold | Actual: `RET-XXXXXX` and **75** |
| 4 | `geminiMatchAnalysisOfPair()` is still unreferenced | Harmless thin wrapper; `geminiSemanticVerdict()` supersedes it |
| 5 | The AI stage has no automated regression test in-repo | The blending and fallback behaviour is verified manually (48 checks) but not guarded by CI |
| 6 | `users.security_code` is vestigial — always `NULL`; the code lives in `security_code_hash` | Two identifiers documented where one exists |
| 7 | The `admin` role exists in the ENUM with no UI or API branch | Dead enum value |
| 8 | Notifications are in-app only — no email or SMS | Do not promise email delivery |
| 9 | `uoh.edu.sa` is hard-coded in `generateUniversityEmail()` | Blocks reuse at another university |
| 10 | `sql_mode` lacks `STRICT_TRANS_TABLES` | Over-length values are silently truncated; app-level validation is the only guard |
| 11 | *(resolved)* The `internal_*` response keys that survived earlier refactors are **gone**. A 2026-09-28 audit of every `jsonResponse()` call confirmed no `internal_*` keys, no debug keys, and no private data on the response surface. See §2.1. | — |
| 12 | *(new 2026-10-02)* **Password recovery has no mailer.** `forgot_password` cannot send email, so the flow is only completable via the `LUQYA_EXPOSE_RESET_TOKEN=1` development path. | Do not write "an email is sent". Production needs a mailer or out-of-band link issuance. See §1.7 |
| 13 | *(new 2026-10-02)* `php.ini` points `error_log` at `C:\xampp\php\logs\`, which does not exist, so PHP log output lands in `C:\xampp\apache\logs\error.log` | The dev reset-link instructions would point at a non-existent file. Documented; ideally fix the path |
| 14 | *(new 2026-10-02)* MariaDB 10.4.32 in this XAMPP install **crashed** (`mysqld.dmp` written) on the prepared `DELETE FROM password_resets …` in `auth.php`'s `forgot_password` handler, leaving all three of that table's indexes corrupt. `OPTIMIZE TABLE password_resets` rebuilt them. | A server bug, not an app bug — but password recovery is the trigger, and a repeat crash would re-corrupt the table. Consider upgrading MariaDB |
| 15 | *(new 2026-10-08)* Recovery security codes are **two-tier by design**: the seeded demo students are legacy accounts (`security_code_hash` empty) and reset with university ID + email alone, while officers and any user who sets a code must supply it. This is intentional (nobody is locked out), but any SRS must state the rule instead of a universal-code requirement. | Documented, not a defect — keeps legacy accounts recoverable |

### Resolved since the first draft

- `geminiSemanticScore()` and `analyzeImageWithGeminiVision()` are no longer dead code.
  The former is on the live single-pair path in `semanticRerankCandidates()`; the latter is
  the canonical vision analyser.
- `vision_ai.php` no longer carries its own copy of the vision cURL caller. Transport,
  fallback chain, timeout and JSON extraction are unified in `callGeminiVisionAPI()`
  (~150 duplicated lines removed).
- `runAIMatchingEngine()` is no longer cosmetic — the matching engine genuinely calls Gemini.
- The matching engine now publishes `local_score` alongside the blended `score`, so a purely
  local decision is always distinguishable from an AI-blended one (`ai_verified`).

---

*End of document — keep in sync with the codebase.*
