# Task: C# Pinterest Auto-Pinner for DynamoDB designs (Standard access)

## Goal
Implement a C# (.NET 8) worker that:
1) Pulls designs from DynamoDB ordered by DesignID DESC.
2) Selects up to 1000 latest designs where PinterestID is missing.
3) Creates Pinterest Pins for them (organic pins via API).
4) Stores the returned pin id back into DynamoDB as PinterestID.
5) Runs continuously or as a scheduled job (every 5–10 minutes), safely and idempotently.
6) Emails the operator on problems (with deduplication/cooldown).

We have Pinterest Developer app with **Standard access active**.

---

## Non-negotiable rules (must implement)

### R1) Idempotency / no duplicates
- Never create a second pin for a design that already has PinterestID.
- Prevent race conditions: if two runs pick the same design, only one may post.
- Implement a “claim/lock” update in DynamoDB:
  - Set `PinterestStatus = "POSTING"` with a conditional update only if PinterestID is missing and status is not POSTING/POSTED.
  - If conditional update fails, skip that design (this is normal and not an error).

### R2) Rate limiting + backoff
- Post conservatively to avoid spam signals.
- Default: **max 1 pin per 5 minutes** (configurable), plus a daily cap (configurable).
- On HTTP 429 / transient errors: exponential backoff with jitter.
- Do not retry indefinitely; mark FAILED after N attempts.

### R3) Content quality / anti-spam constraints
- Do NOT post 1000 quickly.
- Ensure variation in title/description templates.
- Do not post multiple pins with identical title+description+destination link in a short window.
- Enforce:
  - Min interval between pins (e.g., 300 seconds).
  - Optional similarity guard (spread topics; avoid posting many near-identical pins back-to-back).

### R4) Observability
- Log every step (design selected, locked, posted, updated).
- Persist state/errors to DynamoDB fields:
  - `PinterestStatus`: NEW | POSTING | POSTED | FAILED
  - `PinterestLastError`, `PinterestAttemptCount`, `PinterestLastAttemptAt`
- Print a summary at end of each run: counts posted/skipped/failed.

### R5) Configuration via environment variables
No secrets in code.
- AWS_REGION
- DDB_TABLE_NAME
- PINTEREST_ACCESS_TOKEN (or reference to secret)
- POST_INTERVAL_SECONDS (default 300)
- DAILY_CAP (default 200)
- MAX_BATCH_PER_RUN (default 1–5)
- BASE_URL for destination link (cross-stitch.com)
- ENVIRONMENT_NAME (dev/stage/prod) for logging + email subjects

---

## DynamoDB requirements / assumptions
Claude Code is aware of the existing DynamoDB structure and indexes. Implement the query for “latest designs by DesignID DESC” using the existing keys/indexes.

Worker behavior:
- Query pages until it collected up to N items with missing PinterestID.
- Stop once it has enough or reaches a safety page limit.

---

## Pinterest API requirements
- Use Pinterest v5 endpoint to create a Pin.
- Each pin needs:
  - board_id
  - title
  - description
  - link (destination to design page; include UTMs if desired)
  - media_source (image URL)
- Parse response and store returned pin id as PinterestID.

---

## Board selection (NO new mapping logic)
Do NOT implement any board mapping or configuration.
Use the **existing mechanism already used in the project** to decide which board to post to (Claude Code already knows/has access to it).  
The worker should simply call into that existing board-selection logic/service and receive the correct `board_id` for each design.

---

## Scheduling / runtime modes
Implement two run modes:
1) `--once` : process up to MAX_BATCH_PER_RUN and exit.
2) `--daemon` : loop forever, sleep POST_INTERVAL_SECONDS between attempts.

This allows:
- Linux cron calling `--once` every 5–10 min
- or a long-running service/worker calling `--daemon`

---

## Posting algorithm (high level)
1) Load config and create AWS DynamoDB client + HTTP client.
2) Fetch candidates (latest designs, missing PinterestID).
3) For each candidate (up to MAX_BATCH_PER_RUN):
   A) Try to acquire lock:
      - Conditional update: set PinterestStatus=POSTING, attemptCount++, lastAttemptAt=now
   B) Build pin payload:
      - board_id: obtained via existing mechanism (see Board selection section)
      - title/description from templates (see below)
      - link: design page + UTMs (utm_source=Pinterest&utm_medium=Organic&utm_campaign=AutoPins)
      - media_source: use ImageUrl
   C) POST to Pinterest create pin API.
   D) On success:
      - Update DynamoDB: PinterestID=<id>, PinterestStatus=POSTED, clear last error.
   E) On failure:
      - Update DynamoDB: PinterestStatus=FAILED, PinterestLastError=<message>, keep attempt count.
      - Trigger email notifier (see Email section).

4) Print summary and exit or sleep.

---

## Title/description templating rules
Implement 3–5 title templates and rotate, e.g.:
- “{Name} Cross Stitch Pattern (PDF)”
- “Free Cross Stitch Pattern: {Name}”
- “Cute {Name} Cross Stitch Chart – Printable PDF”

Descriptions:
- Must include:
  - “cross stitch pattern”
  - “printable PDF”
  - optional size/colors if available
- Add 5–10 hashtags max, rotate sets to avoid identical blocks.

If Design has tags/category, use them; otherwise infer from title.

---

# Add-on requirement: Email alerts on problems (must implement)

## Goal
If the auto-pinner encounters problems, it must notify the operator by email.

## Rules for emailing (avoid spam)

### E1) Notify only when it matters
Send an email when any of these happen:
- Pinterest API request fails with a **non-transient** error (4xx except 429), or transient errors exceed retry limit.
- Any DynamoDB read/write fails (network/permission/throughput), except the normal “conditional lock failed” skip case.
- The job hits **daily cap** earlier than expected due to excessive failures/retries.
- The job sees **N consecutive failures** (default N=5).
- The job cannot load critical config (missing token/table/etc.).

### E2) Deduplicate alerts
- Keep an in-memory throttle + persistent “last alert” record to avoid emailing the same error repeatedly.
- Do not send the same alert more than once per **ALERT_COOLDOWN_MINUTES** (default 60).
- If errors continue, send a single follow-up “still failing” email at most once per cooldown window.

### E3) Include actionable details
Email subject includes environment + severity:
- `[AutoPinner][PROD][ERROR] Pinterest post failed (429 retries exceeded)`

Body must include:
- Timestamp (UTC)
- DesignID (if applicable)
- Operation (CreatePin / UpdateDDB / QueryDDB / Config)
- HTTP status + response snippet (safe, no secrets)
- Attempt count and next action (retry/marked FAILED)
- Correlation id / run id

### E4) Optional daily summary (recommended)
Send one daily email summary (configurable) with:
- posted count, skipped count, failed count
- top failure reasons
- remaining backlog count (items with no PinterestID)

## Implementation guidance (C#)

### Approach
Implement an `IEmailNotifier` with two implementations:
1) **AWS SES** (preferred): send via SES API (AWS SDK) or SMTP.
2) **SMTP** (fallback).

### Required configuration (env vars)
- ALERT_EMAIL_TO
- ALERT_EMAIL_FROM
- ALERT_COOLDOWN_MINUTES (default 60)
- ALERT_CONSECUTIVE_FAILURE_THRESHOLD (default 5)
- ALERT_DAILY_SUMMARY_ENABLED (true/false)
- ALERT_DAILY_SUMMARY_HOUR_UTC (default 07)

If using SES SMTP:
- SES_SMTP_HOST, SES_SMTP_PORT, SES_SMTP_USER, SES_SMTP_PASS

If using SES API:
- AWS_REGION + IAM permissions for `ses:SendEmail` (or `ses:SendRawEmail`)

### Where to store persistent “last alert”
Use DynamoDB (same table or a small separate item) with a key like:
- PK = `SYS#ALERTS`, SK = `AUTOPINNER`

Fields:
- LastAlertAtUtc
- LastAlertFingerprint (hash of error class + status + endpoint)

### When to call notifier
- In posting loop: on failure after retries exhausted, call notifier.
- On startup: if config invalid, call notifier once and exit.
- On consecutive failure threshold: call notifier immediately (still dedupe by fingerprint).

### Security
- Never email access tokens or full request payloads containing secrets.
- Truncate API response bodies (e.g., first 500 chars).

---

## Project structure
Create a new repo folder:
- /src/AutoPinner
  - Program.cs
  - Config.cs
  - DynamoDbDesignRepository.cs
  - PinterestClient.cs
  - PinComposer.cs (templates + URL builder)
  - EmailNotifier/
    - IEmailNotifier.cs
    - SesEmailNotifier.cs
    - SmtpEmailNotifier.cs
    - AlertDeduplicator.cs
  - Models/Design.cs, Models/PinterestCreatePinRequest.cs, Models/PinterestCreatePinResponse.cs
  - Utils/RetryPolicy.cs, Utils/RateLimiter.cs
- /src/AutoPinner.Tests (optional)
- README.md with setup + env vars + run commands

---

## Acceptance criteria
- Running `dotnet run -- --once` posts up to MAX_BATCH_PER_RUN new pins and updates DynamoDB PinterestID.
- Re-running does NOT duplicate pins for already posted designs.
- On 429/transient errors it retries with backoff and eventually marks FAILED.
- Logs show clear steps and a final summary.
- Configurable caps (interval, daily cap, batch size).
- On serious errors, an email is sent (and deduped with cooldown).
- Daily summary email works if enabled.

---

## Deliverables
1) Implement the code.
2) Provide README with:
   - required env vars
   - how to run locally
   - how to schedule (cron example + Windows Task Scheduler example)
3) Safe default configuration:
   - 1 pin per 5 min
   - daily cap 200
   - batch=1
4) Proceed to implement now.