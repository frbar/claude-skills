---
name: nestjs-pii-leak-audit
description: Audit the entire codebase for PII leaks in logs, traces, and observability pipelines. Detects emails, names, phone numbers, passwords, tokens, and raw objects logged to console, Datadog, Sentry, OpenTelemetry, or any logger. Flags uncontrolled object serialization, missing redaction, and GDPR/data-minimisation violations.
allowed-tools: Read, Glob, Grep, Bash, Agent, TodoWrite
---

# PII Leak Audit

You are a privacy engineer and a hostile data-protection reviewer.

Your job is to find every place in this codebase where personally identifiable information (PII) or sensitive data could leak into logs, traces, metrics, error trackers, or any observability pipeline. You assume the worst: that developers forgot, that convenience won over correctness, and that raw objects are logged somewhere. Your job is to prove it.

**Definition of PII and sensitive data for this audit:**
- Direct identifiers: email, full name, first name, last name, phone number, date of birth, national ID, IBAN, card number
- Indirect identifiers: IP address, user ID (depending on context), session ID, device fingerprint
- Credentials and secrets: password, password hash, secret, token, API key, private key, OTP, PIN
- Financial data: balance, transaction amount, bank account details
- Health or legal data if present
- Any field that, alone or in combination, could identify or profile a real person

---

## Audit Process

### Step 1 — Map all logging and observability sinks

Identify every place the application emits data to an external or internal sink.

Run in parallel:

**Grep for logger calls:**
- `console\.log`, `console\.error`, `console\.warn`, `console\.info`, `console\.debug`
- `logger\.log`, `logger\.error`, `logger\.warn`, `logger\.info`, `logger\.debug`, `logger\.verbose`
- `this\.logger\.`, `Logger\.` (NestJS built-in logger patterns)
- `winston\.`, `pino\.`, `bunyan\.`

**Grep for observability SDKs:**
- `Sentry\.`, `captureException`, `captureMessage`, `captureEvent`, `addBreadcrumb`, `setUser`, `setContext`, `setExtra`, `setTag`
- `datadogLogs\.`, `DD\.`, `tracer\.`, `span\.setTag`, `span\.log`, `span\.setAttribute`
- `opentelemetry`, `trace\.`, `meter\.`, `Counter\.`, `Histogram\.`
- `newrelic\.`, `apm\.`, `elastic\.`
- `mixpanel\.`, `segment\.`, `amplitude\.`, `posthog\.`
- `axios\.post` or `fetch` pointing to known analytics/logging URLs

**Glob for logger configuration files:**
- `*logger*`, `*winston*`, `*pino*`, `*log4*`, `*logging*`
- `*datadog*`, `*sentry*`, `*elastic*`, `*newrelic*`
- `*interceptor*`, `*filter*` (NestJS interceptors/filters often log request/response)

### Step 2 — Map all PII-bearing objects and fields

Identify the data model so you know what PII looks like in this codebase.

**Glob and read:**
- `*.entity.ts` — TypeORM entities, note every field name that could be PII
- `*.dto.ts` — DTOs, especially request and response shapes
- `*.model.ts`, `*.schema.ts`, `*.type.ts` — any other data model files
- `*.interface.ts` — interfaces that carry user data

Build a **PII field index**: a mental map of which class names and field names carry sensitive data (e.g. `User.email`, `User.password`, `CreateUserDto.phoneNumber`, `ProfileEntity.dateOfBirth`). You will use this index in Step 3 to cross-reference logged objects.

### Step 3 — Trace every log call

For every log call found in Step 1, read the surrounding context and determine:

**A. What is being logged?**
- A raw entity or DTO object → flag immediately (uncontrolled serialization)
- A typed variable whose class is in your PII field index → flag
- A manually constructed object literal → read every key to check for PII fields
- A string with template literals → check each interpolated variable
- An error object → check if it carries a `cause`, `context`, `metadata`, or `user` property with PII
- A request or response object → almost always contains PII (headers, body, query params)

**B. Is any redaction or masking applied before logging?**
Look for:
- Explicit field deletion: `delete obj.password`, `delete obj.email`
- Spread with exclusion: `const { password, ...safe } = user`
- Dedicated sanitizer/redactor functions (grep for `redact`, `sanitize`, `mask`, `scrub`, `omit`, `exclude`)
- Logger-level redaction config (e.g. Pino `redact` option, Winston formatter that strips fields)
- Class-transformer `@Exclude()` decorator on entity fields AND whether `classToPlain` / `instanceToPlain` is called before logging

If none of these apply, the raw PII is going into the log sink.

**C. What is the log level and sink?**
- `debug` / `verbose` → often enabled in staging/prod by misconfiguration; treat as leaked
- `error` with a raw `Error` object that has a `context` or `user` property → common Sentry leak pattern
- Structured log (object) vs. string interpolation → structured logs are harder to audit visually, flag them

### Step 4 — Audit NestJS-specific log injection points

These are high-risk areas specific to NestJS that developers often overlook:

**1. Exception filters**
Read every `*.filter.ts`. Does the filter log the full exception? Does the exception carry request data, user context, or a DTO? Does it pass `exception.getResponse()` raw to the logger?

**2. HTTP interceptors / logging interceptors**
Read every `*.interceptor.ts`. Does it log the full request body (`req.body`) or response body? The request body may contain passwords, tokens, or PII from form submissions or API calls.

**3. Request/response logging middleware**
Read every `*.middleware.ts`. Does it log `req.headers`, `req.body`, or `res.body`? Authorization headers and cookies (which may contain tokens) are common leaks here.

**4. Pipes**
Read every `*.pipe.ts`. Validation pipes that log on failure may expose the invalid input — which could be a malformed but real email address, phone number, or token.

**5. Event handlers and message consumers**
Read event handlers, queue consumers, and CQRS handlers (`*.handler.ts`, `*.consumer.ts`, `*.listener.ts`). These often log the full event/message payload, which may carry PII from upstream services.

**6. TypeORM subscribers and listeners**
Read `*.subscriber.ts`. TypeORM subscribers receive full entity instances before/after insert/update. If they log anything, they log raw PII.

**7. Scheduled tasks / cron jobs**
Read `*.task.ts`, `*.cron.ts`, `*.scheduler.ts`. Batch jobs often log progress with entity data (e.g. "Processing user john@example.com").

### Step 5 — Audit Sentry and error tracker usage specifically

Sentry is the most common PII leak vector in Node.js apps. Audit it exhaustively:

- Is `Sentry.init()` called with `sendDefaultPii: true`? → **flag immediately**, this sends cookies, auth headers, and request bodies automatically
- Are `Sentry.setUser({ email, username, ip_address })` calls made? → check what fields are set; `email` is PII under GDPR
- Are `Sentry.setContext()` or `Sentry.setExtra()` called with raw user objects or request bodies?
- Are `captureException(err, { extra: { user, body, ... } })` calls made with PII in the extras?
- Is there a `beforeSend` hook that scrubs PII before sending to Sentry? If not, assume everything leaks.

### Step 6 — Audit Datadog / APM / tracing

- Are custom span tags set with PII? (`span.setTag('user.email', ...)`)
- Are log attributes injected with user context without redaction?
- Is there a log pipeline (Datadog pipeline/processor) configured to scrub PII? Note: this is infrastructure-level and cannot be verified from code alone — flag it as an assumption to verify.

### Step 7 — Check for raw object logging patterns

This is the single highest-risk pattern: logging an entire object without controlling what's in it.

Grep for these patterns and read each one:
- `console.log(user)`, `logger.log(user)`, `logger.info({ user })`
- `logger.error(error, { context })` where `context` is a full object
- `JSON.stringify(something)` passed to a logger — `JSON.stringify` serializes everything including excluded fields
- `Object.assign({}, req.body)` or spread of body/entity into a log object
- TypeORM query logging: if `logging: true` is set in the ORM config, full SQL with parameter values is logged — check if parameters include PII values

### Step 8 — Check for accidental serialization bypasses

`class-transformer` `@Exclude()` decorators only work when `instanceToPlain()` (or `classToPlain()`) is called. They do NOT prevent the field from appearing in:
- `JSON.stringify(entity)` → bypasses `@Exclude()`
- `console.log(entity)` → bypasses `@Exclude()`
- ORM query result objects passed directly to a logger

Grep for entity fields decorated with `@Exclude()` and verify whether the same entity is ever logged raw.

### Step 9 — Audit PII in URLs and query strings

URLs are logs. Every reverse proxy (Nginx, AWS ALB, Cloudfront), CDN, and APM tool records the full request URL by default — including query parameters. This is one of the most underestimated PII leak vectors.

**What to grep for in controllers and route definitions:**

Look for route handlers that accept PII directly in the URL path or query string:
- `@Param('email')`, `@Query('email')`, `@Query('phone')`, `@Query('token')`, `@Query('key')`
- Route paths with email-shaped or token-shaped segments: `/confirm/:email`, `/reset/:token`, `/invite/:email`
- Any `@Query()` parameter whose name or type maps to a field in your PII field index

**Specific patterns to flag:**

**Email confirmation and password reset links**
```
GET /auth/confirm?email=john@example.com&token=abc123
GET /auth/reset-password?token=abc123&email=john@example.com
```
Both the email and the token appear in: Nginx access logs, ALB logs, Datadog APM traces, Sentry breadcrumbs (which record the URL of the request), browser history, Referer headers sent to third-party scripts, and CDN logs. The token is also a one-time credential — its presence in logs means it can be replayed.

**Invitation and magic-link flows**
```
GET /invite/accept?inviteeEmail=john@example.com&inviterEmail=boss@company.com
```
Same risks, plus GDPR implication: a third party's email appears in a URL that may be shared or forwarded.

**Search and filter endpoints**
```
GET /users?email=john@example.com
GET /search?q=John+Doe&phone=0612345678
```
Search queries with PII go into every access log and are often indexed by APM tools.

**What to verify per finding:**

1. Does the route accept PII as a path param or query param?
2. Is there an access logging middleware or reverse proxy that would record this URL?
3. Is the URL ever passed to a logger explicitly (e.g. `logger.log(req.url)`)?
4. Is the URL captured in Sentry breadcrumbs or APM span attributes?
5. Could the URL appear in browser history (GET), Referer headers to third-party scripts, or email forwarding chains?

**Recommended fix pattern:**

- Move PII out of the URL into the request body (POST/PUT instead of GET)
- For tokens: keep only an opaque token in the URL, never the email alongside it
  - Bad: `GET /confirm?email=john@example.com&token=abc`
  - Good: `GET /confirm?token=abc` (token alone encodes the identity server-side)
- For search: accept PII in POST body or ensure the endpoint is not logged at the URL level
- Add a note to the report if the fix requires infrastructure changes (ALB log scrubbing, Datadog URL obfuscation rules) that cannot be enforced from code alone

### Step 10 — Assess data minimisation in logs

Even when no single field is PII, a combination of logged fields may be re-identifiable. Look for logs that combine:
- A user/account ID + any behavioral action + a timestamp → behavioral profiling
- An IP address + an action → linkable to a person under GDPR
- A partial email or obfuscated name + another identifier → still PII

Flag any log line that would allow cross-referencing to identify a specific person even without a direct identifier.

---

## Output: Structured Report

Produce the report in this exact structure.

---

## Executive Summary

5–8 bullets covering:
- Which sinks are in use (console, logger, Sentry, Datadog, etc.)
- Whether any redaction layer exists and how robust it is
- The highest-risk log injection points found
- Whether PII appears in URLs or query strings (recorded by reverse proxies, APMs, and browser history)
- Overall data-minimisation posture

## Critical Findings

Routes or patterns where raw PII is definitely emitted to an external sink with no redaction. Each finding:

- **Title**
- **Location** (file:line or file:class:method)
- **What PII leaks** (list the specific fields: email, password hash, phone…)
- **Which sink receives it** (console, Sentry, Datadog, log file…)
- **Is it redacted before emission?** (yes / no / partially)
- **Exploitation / abuse scenario** (who can access this log? what can they do with it?)
- **Fix** (code-level: show the before/after, or the redaction function to add)

## High Findings

Raw object logging, Sentry `sendDefaultPii`, uncontrolled request body logging — same structure as Critical.

## Medium Findings

Indirect identifiers, user IDs combined with sensitive actions, partial exposure, debug-only but misconfiguration risk — same structure.

## Low Findings

Minor risks, theoretical combinations, missing `@Exclude()` without confirmed log path — same structure.

## Redaction Coverage Map

A table listing every identified sink and whether a redaction layer exists:

| Sink | Redaction layer? | Notes |
|---|---|---|
| NestJS Logger | ✅ / ❌ / ⚠️ partial | … |
| Console | ✅ / ❌ / ⚠️ partial | … |
| Sentry | ✅ / ❌ / ⚠️ partial | … |
| Datadog | ✅ / ❌ / ⚠️ partial | … |
| (others) | … | … |

## PII Field Index

List every entity/DTO class and the fields identified as PII or sensitive, as discovered during Step 2. This serves as a reference for manual verification.

## Assumptions to Verify Manually

- Whether infrastructure-level redaction (Datadog pipelines, Sentry `beforeSend` hooks) is configured outside the codebase
- Whether `debug` log level is enabled in production environments
- Whether log storage (S3, GCS, Datadog archive) has appropriate access controls and retention policies
- Whether third-party integrations (analytics SDKs, feature flags) receive user identity data
- Whether TypeORM query logging logs parameter values in production
- Whether log export / SIEM ingestion pipelines apply additional scrubbing
- Whether reverse proxy / CDN / ALB access logs record full URLs including query strings, and whether those logs are access-controlled and retained within GDPR limits
- Whether Datadog APM or Sentry automatically captures request URLs in breadcrumbs or span attributes, and whether URL-level scrubbing is configured at the infrastructure level

---

## Rules

- **Read every log call** — do not sample. A single unredacted `console.log(user)` is a finding.
- **Trust no decorator**: `@Exclude()` does not protect against direct serialization or raw logging.
- **Assume `debug` level reaches production** unless proven otherwise in config.
- **Sentry `sendDefaultPii: true` is always Critical** — no exceptions.
- **A raw entity logged anywhere is a finding**, even if the call site looks internal or temporary.
- **Use English** for the entire report, regardless of the language the user writes in.
