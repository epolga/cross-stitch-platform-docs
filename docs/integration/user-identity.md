# Contract: User Identity (cid, USR# prefix, UnsubscribeToken, verification, password storage)

## 1. Contract Name

**User identity — cid, USR# prefix, UnsubscribeToken, verification, password storage.**

A naming-convention + schema contract that pins the shared identity surface for end-users across the cross-stitch web app (writer of canonical values on registration) and the Uploader WPF tool (back-fill / migration writer + email-tracking consumer). This contract focuses on identity invariants and flows; per-attribute DynamoDB type detail lives in `dynamodb-schema.md` and is not duplicated here.

## 2. Purpose

Capture the user-identity invariants that `cross-stitch/` (Next.js web app) and `Uploader/` (WPF tool) must agree on so that:

- A given person resolves to exactly one DynamoDB row across both apps.
- Email tracking links emitted by Uploader (`?cid=…&eid=…`) can be resolved by cross-stitch on click-through (`updateLastEmailEntryByCid` at `cross-stitch/src/lib/data-access.ts:766`, `updateLastEmailEntryInUsersTable` at `cross-stitch/src/lib/users.ts:539`).
- The single `UnsubscribeToken` minted by cross-stitch on user create (`cross-stitch/src/lib/users.ts:302`) is the same token Uploader embeds in its newsletter blasts (`Uploader/Uploader/MainWindow.xaml.cs:2306` / `:2316`).
- The verification flag set by either the cross-stitch email-verify flow (`cross-stitch/src/lib/users.ts:511-530`) or the Uploader admin migration (`Uploader/Uploader/MainWindow.xaml.cs:3072 MarkUsersVerifiedAsync`) means the same thing on both sides.
- Password storage semantics are documented honestly: today **the cleartext password is stored** under attribute `Password` (`cross-stitch/src/lib/users.ts:325`) and read back as `OpenPwd` in the primary table (`cross-stitch/src/lib/data-access.ts:684`). See §10.

## 3. Scope

In scope:
- DynamoDB table `CrossStitchUsers` (secondary users table) — PK `ID = USR#<email>` (`cross-stitch/src/lib/users.ts:230`, `Uploader/Uploader/App.config:7` declares the table name).
- DynamoDB table `CrossStitchItems` (primary single-table) — rows with `ID` starting `USR#` and `EntityType = 'USER'` (cross-stitch reads them in `updateLastEmailEntryByCid` at `cross-stitch/src/lib/data-access.ts:782-792`; Uploader migrates them in `InitializeItemsUserCidFieldsAsync` at `Uploader/Uploader/MainWindow.xaml.cs:1538`).
- Attributes: `ID`, `Email`, `FirstName`, `Password` / `OpenPwd`, `cid`, `UnsubscribeToken`, `Unsubscribed`, `VerificationToken`, `VerificationTokenExpiresAt`, `Verified`, `VerifiedAt`, `CreatedAt`, `ReceiveUpdates`, `LastEmailEntry`, `LastSeenAt`, `LastSeenCount`, `SubscriptionId`, `SubscriptionActive`, `SubscriptionStartedAt`, `SubscriptionStatusUpdatedAt`, `TrialStartedAt`, `TrialEndsAt`, `TrialDownloadLimit`, `TrialDownloadsUsed`, `TrialDownloadedDesignIds`.
- The `PasswordResetTokens` table (PK `Token`) used by `cross-stitch/src/lib/password-reset.ts:14-78`.
- The four maintenance flows in Uploader that act as identity-attribute migrations: `InitializeUserUnsubscribe_Click` / `InitializeUserUnsubscribeFieldsAsync` (`Uploader/Uploader/MainWindow.xaml.cs:467, 1275`), `InitializeUserCid_Click` / `InitializeUserCidFieldsAsync` (`:502, 1419`), `InitializeItemsUserCid_Click` / `InitializeItemsUserCidFieldsAsync` (`:3734, 1538`), `MarkUsersVerified_Click` / `MarkUsersVerifiedAsync` (`:3796, 3072`), `RemoveSuppressedUsers_Click` / `RemoveSuppressedUsersAsync` (`:3772, 2983`).

Out of scope (covered by other contracts or files):
- DynamoDB attribute types and table indexes — see `dynamodb-schema.md`.
- Pinterest user-of-app credentials and token store — different surface entirely.
- HTTP cookies, NextAuth sessions, CSRF — neither repo currently uses session middleware in scope of this contract.
- Unsubscribe URL shape / unsubscribe-page handler — covered indirectly here but the email-flow specifics belong in a future `email-and-unsubscribe.md`.

## 4. Data Formats

### 4.1 Primary key — `USR#<email>`

- The canonical ID is `` `USR#${normalizedEmail}` `` where `normalizedEmail = email.trim().toLowerCase()` (`cross-stitch/src/lib/users.ts:125-127, 230, 314`; same form in `cross-stitch/src/lib/password-reset.ts:33`).
- Uploader uses the same string-concatenation form for direct lookups: `` string userId = $"USR#{email}"; `` (`Uploader/Uploader/MainWindow.xaml.cs:2995` in `RemoveSuppressedUsersAsync`).
- Email is treated case-insensitively. cross-stitch lower-cases on write and on the secondary-table fallback compares `dbEmail = item.Email?.S?.trim().toLowerCase()` (`cross-stitch/src/lib/data-access.ts:78`); Uploader compares emails with `StringComparison.OrdinalIgnoreCase` (`Uploader/Uploader/MainWindow.xaml.cs:2288, 2326`).
- **Invariant:** PK = `USR#` followed by the trimmed, lower-cased email. Mixed-case writes would silently create duplicates because both `saveUserToDynamoDB` and the back-fill paths assume lower-case.

### 4.2 `cid` — user correlation GUID (used in email-tracking URLs)

- Format: lowercase GUID. Two flavors observed:
  - cross-stitch generates `cid: randomUUID()` on user create (`cross-stitch/src/lib/users.ts:303`), which yields the dashed lowercase form `xxxxxxxx-xxxx-…`.
  - Uploader back-fill generates `Guid.NewGuid().ToString("N")` for legacy rows (`Uploader/Uploader/MainWindow.xaml.cs:1489`), which yields **32 hex chars without dashes**. Both shapes coexist in production.
- Attribute name: lowercase `cid` (note: not PascalCase — both writers spell it `cid`).
- Read path: cross-stitch scans by `cid` to update `LastEmailEntry` after a tracked click (`cross-stitch/src/lib/data-access.ts:766-812`) and also looks up a verified user by cid (`cross-stitch/src/lib/users.ts:387-441`).
- Write path A — cross-stitch on registration: `cross-stitch/src/lib/users.ts:303, 330`.
- Write path B — Uploader back-fill on `CrossStitchUsers` only when missing: `Uploader/Uploader/MainWindow.xaml.cs:1468-1494`.
- Write path C — Uploader back-fill on `CrossStitchItems` for legacy USR# rows: `Uploader/Uploader/MainWindow.xaml.cs:1538`.
- Embedding in URLs: appended as the `cid` query parameter by `AppendTrackingParameters` (`Uploader/Uploader/MainWindow.xaml.cs:2352-2368`). Both pattern and site URLs are decorated (e.g. `:2511, 2618, 2619`).

### 4.3 `UnsubscribeToken` — one-per-user random token

- Format observed varies by writer (a sync hazard, see §9):
  - cross-stitch on user create: `randomUUID()` (`cross-stitch/src/lib/users.ts:302, 328`) → dashed lowercase UUID.
  - Uploader back-fill: `GenerateRandomToken()` (`Uploader/Uploader/MainWindow.xaml.cs:1329, 2333`) → 32 random bytes, URL-safe base64 with padding stripped (`ToBase64Url` at `:2344-2350`).
- Attribute name: `UnsubscribeToken` (string).
- Companion flag: `Unsubscribed` (BOOL) defaults to `false` (`cross-stitch/src/lib/users.ts:329`, Uploader back-fill at `:1334-1341`).
- Write path A — cross-stitch on user create, always: `cross-stitch/src/lib/users.ts:302`.
- Write path B — Uploader back-fill, **only when the row has no token** (`Uploader/Uploader/MainWindow.xaml.cs:1305-1332`). Crucially, this is guarded by `hasToken`, so Uploader cannot clobber an existing canonical token unless future code drops the guard. The risk is documented in §9.
- Lookup path: cross-stitch scans `CrossStitchUsers` (and optionally `CrossStitchItems` filtered by `EntityType = 'USER'`) by `UnsubscribeToken` in `unsubscribeUserByToken` (`cross-stitch/src/lib/users.ts:1230-1276, 1282-1336`).
- URL shape: `${SiteBaseUrl}/unsubscribe?token=<url-encoded-UnsubscribeToken>` built by Uploader at `BuildUnsubscribeUrl` (`Uploader/Uploader/MainWindow.xaml.cs:2306-2314`) and `BuildUnsubscribeUrlFromStoredToken` (`:2316-2322`). The override base URL is `UnsubscribeBaseUrl` from `Uploader/Uploader/App.config:18` (`https://cross-stitch.com/unsubscribe`).
- **`UnsubscribeSecret` in `App.private.config.example:8` is dead config:** it appears in the example file but is not consumed anywhere in `Uploader/Uploader/MainWindow.xaml.cs` (no matches). The original architecture summary claim of an HMAC envelope is unsupported by current code — the URL embeds the raw token verbatim.

### 4.4 `Verified` flag + verification tokens

- Attributes on user row: `Verified` (BOOL, default `false`), `VerifiedAt` (string ISO, default `""`), `VerificationToken` (string UUID), `VerificationTokenExpiresAt` (string ISO, default `now + 48h`). All initialized in `saveUserToDynamoDB` (`cross-stitch/src/lib/users.ts:331-335`).
- Default expiry window: 48 hours (`cross-stitch/src/lib/users.ts:307`; also enforced at the API entry point `cross-stitch/src/app/api/register-only/route.ts:43`).
- Verify flow: cross-stitch `verifyUserByToken` scans for the matching `VerificationToken`, checks `VerificationTokenExpiresAt`, then sets `Verified = true`, `VerifiedAt = now`, and **removes** the two token attributes (`cross-stitch/src/lib/users.ts:513-530`).
- Bulk migration path: Uploader `MarkUsersVerifiedAsync` (`Uploader/Uploader/MainWindow.xaml.cs:3072+`) scans rows lacking `Verified=true` AND `VerifiedAt`, and writes them en masse using `CreatedAt` as the `VerifiedAt` value. This is a one-shot migration; **Uploader does not perform identity verification**, it just retro-actively marks legacy rows.
- Verified-by-cid read: `cross-stitch/src/lib/users.ts:387-441 getVerifiedUserByCid` returns the user only if `Verified === true` OR `VerifiedAt` is non-empty.

### 4.5 Password storage — **PLAINTEXT** (security debt, see §10)

- On registration, password is written as-is to the `Password` string attribute:
  ```ts
  Password: { S: password }, // Storing plain password for migration purposes
  ```
  (`cross-stitch/src/lib/users.ts:325`, exact code comment preserved verbatim).
- The primary `CrossStitchItems` table stores the same plaintext value under the attribute name `OpenPwd`. Login reads it directly: `const storedPassword = Items[0].OpenPwd?.S;` and `const isMatch = storedPassword === password;` (`cross-stitch/src/lib/data-access.ts:684, 691`).
- The fallback secondary-table check also matches plaintext: `FilterExpression: '#password = :pwd'` (`cross-stitch/src/lib/data-access.ts:66-68`).
- `password-reset.ts updateUserPassword` writes the new password as-is into the same `Password` attribute (`cross-stitch/src/lib/password-reset.ts:144-166`) — no hashing introduced on reset.
- The header docstring of `saveUserToDynamoDB` mentions `passwordHash` / `passwordSalt` (`cross-stitch/src/lib/users.ts:288-289`), but **the actual implementation does not hash**. The comment is stale.

### 4.6 `PasswordResetTokens` table

- Table name: `process.env.DDB_RESET_TOKENS_TABLE || 'PasswordResetTokens'` (`cross-stitch/src/lib/password-reset.ts:16`).
- PK: `Token` (string).
- Attributes: `Token` (PK), `Email` (lowercased), `CreatedAt` (ISO), `ExpiresAtEpoch` (numeric Unix seconds).
- Token format: `crypto.randomBytes(32).toString('hex')` → 64 hex chars (`cross-stitch/src/lib/password-reset.ts:58`).
- TTL: `PASSWORD_RESET_TTL_SECONDS` env, default **7200 s (2 hours)** (`cross-stitch/src/lib/password-reset.ts:20-21`).
- Single-use: token is deleted on consumption (whether expired, malformed, or successfully redeemed) (`cross-stitch/src/lib/password-reset.ts:106-135`).
- Uploader has no read or write path for `PasswordResetTokens`.

### 4.7 Subscription state on the user row (transitions only — see existing PayPal docs for full event flow)

- Attributes: `SubscriptionId` (string), `SubscriptionActive` (BOOL), `SubscriptionStartedAt` (ISO), `SubscriptionStatusUpdatedAt` (ISO, written on PayPal-driven flips).
- Activation/deactivation by subscription id: `setSubscriptionActiveBySubscriptionId` (`cross-stitch/src/lib/users.ts:978-1010`) is the only mutation path PayPal hits.
- PayPal webhook → user-row transitions (`cross-stitch/src/app/api/paypal-webhook/route.js:152-211`):
  - `BILLING.SUBSCRIPTION.ACTIVATED` / `BILLING.SUBSCRIPTION.RE-ACTIVATED` → `SubscriptionActive = true`.
  - `BILLING.SUBSCRIPTION.CANCELLED` / `BILLING.SUBSCRIPTION.SUSPENDED` / `BILLING.SUBSCRIPTION.EXPIRED` → `SubscriptionActive = false`.
  - All other event types: no user-row change.
- Uploader does **not** mutate subscription attributes (no writers in `MainWindow.xaml.cs`).

### 4.8 Trial state (cross-stitch only)

- Attributes initialized on `startTrial` path: `TrialStartedAt`, `TrialEndsAt` (default 30 days later), `TrialDownloadLimit` (env-configurable, default 10), `TrialDownloadsUsed`, `TrialDownloadedDesignIds` (string set) — `cross-stitch/src/lib/users.ts:72-73, 317-342, 906-928`.
- Uploader does not touch trial attributes.

## 5. API Endpoints / Interfaces

There is no direct HTTP API between Uploader and cross-stitch (consistent with `platform-architecture-summary.md:249`). The contract surfaces are:

### 5.1 DynamoDB tables

- `CrossStitchUsers` — secondary users table; both repos write here. Configured in `Uploader/Uploader/App.config:7` and resolved by `process.env.DDB_USERS_TABLE` (default `'CrossStitchUsers'`) in `cross-stitch/src/lib/users.ts:16` and `cross-stitch/src/lib/password-reset.ts:14`.
- `CrossStitchItems` — primary single-table; holds USR# rows alongside DESIGN/ALBUM. cross-stitch reads via `process.env.DYNAMODB_TABLE_NAME` (`cross-stitch/src/lib/data-access.ts:658, 767`); Uploader resolves the same name via `ConfigurationManager.AppSettings["DynamoTableName"]` defaulting to `"CrossStitchItems"` (`Uploader/Uploader/MainWindow.xaml.cs:1540, 2985`).
- `PasswordResetTokens` — cross-stitch only.

### 5.2 cross-stitch HTTP routes that mutate identity attributes

- `POST /api/register-only` (`cross-stitch/src/app/api/register-only/route.ts:29-95`) — calls `saveUserToDynamoDB`, sends verification email via SES (`cross-stitch/src/lib/email-service.ts:27-52`). Response: 200 `{ok, message}` / 400 missing fields / 409 `EmailExists` / 500 server error.
- The `verify-token` route consumes `VerificationToken` (delegates to `verifyUserByToken` at `cross-stitch/src/lib/users.ts:446-533`).
- `POST /api/paypal-webhook` (`cross-stitch/src/app/api/paypal-webhook/route.js`) — drives subscription transitions.
- `GET /unsubscribe?token=…` — consumes `UnsubscribeToken` via `unsubscribeUserByToken` (`cross-stitch/src/lib/users.ts:1282-1336`).

### 5.3 Uploader maintenance buttons (DynamoDB read-write)

All of these are WPF event handlers that act as **migration scripts** (see §6):

| Button | Handler | Implementation |
|---|---|---|
| Initialize User Unsubscribe | `InitializeUserUnsubscribe_Click` (`MainWindow.xaml.cs:467`) | `InitializeUserUnsubscribeFieldsAsync` (`:1275`) — adds `UnsubscribeToken` + `Unsubscribed=false` to `CrossStitchUsers` rows missing them |
| Initialize User cid | `InitializeUserCid_Click` (`:502`) | `InitializeUserCidFieldsAsync` (`:1419`) — adds 32-hex-no-dash `cid` to `CrossStitchUsers` rows missing it |
| Initialize Items User cid | `InitializeItemsUserCid_Click` (`:3734`) | `InitializeItemsUserCidFieldsAsync` (`:1538`) — same back-fill against `CrossStitchItems` rows with `ID` prefix `USR#` |
| Mark Users Verified | `MarkUsersVerified_Click` (`:3796`) | `MarkUsersVerifiedAsync` (`:3072`) — sets `Verified=true`, `VerifiedAt=CreatedAt` on legacy rows |
| Remove Suppressed Users | `RemoveSuppressedUsers_Click` (`:3772`) | `RemoveSuppressedUsersAsync` (`:2983`) — deletes USR# rows from `CrossStitchItems` for emails listed in a suppression file |
| Initialize User Subscriptions | `InitializeUserSubscriptions_Click` (`:485`) | `InitializeUserSubscriptionFieldsAsync` — back-fill subscription attributes (out of scope for this doc; see §4.7) |

### 5.4 Uploader email-blast read interface

When sending newsletters, Uploader **reads** `UnsubscribeToken` from the row by email and refuses to send if missing (`Uploader/Uploader/MainWindow.xaml.cs:2291-2303`), then builds the URL via `BuildUnsubscribeUrlFromStoredToken` (`:2316-2322, 2508, 2621`).

## 6. Versioning

**Unversioned (implicit).** Neither attribute names nor the `USR#` prefix carry a version number, and DynamoDB writes have no `schemaVersion` attribute on user rows.

Uploader's four `Initialize*` buttons are effectively schema-migration shims (back-fill `cid`, `UnsubscribeToken`/`Unsubscribed`, `Verified`/`VerifiedAt`) that exist precisely because the schema evolved without versioning. They are idempotent — each guards on attribute presence (`Uploader/Uploader/MainWindow.xaml.cs:1305-1315, 1468-1476, 3112-3119`) — and can be re-run safely.

**Recommendation:** when adding any new identity attribute, (a) add it to `saveUserToDynamoDB` initialization with a sane default so new rows have it, and (b) add a matching idempotent back-fill button to Uploader for legacy rows. Document the addition here; do not silently introduce attributes.

## 7. Ownership & Contacts

- **Maintainer:** Olga (`epolga`, olga.epstein@gmail.com).
- **Code owner of canonical writes (cid, UnsubscribeToken, Verified, Password, subscription transitions, trial state):** `cross-stitch/` — all initial writes happen in `src/lib/users.ts` (`saveUserToDynamoDB` at line 292) and PayPal-driven transitions in `src/lib/users.ts` + `src/app/api/paypal-webhook/route.js`.
- **Code owner of migration / back-fill writes:** `Uploader/` — `Uploader/Uploader/MainWindow.xaml.cs` handlers listed in §5.3.
- **Code owner of `PasswordResetTokens`:** `cross-stitch/` exclusively (`src/lib/password-reset.ts`).

## 8. Dependencies

- **AWS DynamoDB** — tables `CrossStitchUsers`, `CrossStitchItems`, `PasswordResetTokens`. Region `us-east-1` (`cross-stitch/src/lib/users.ts:15`).
- **AWS SES** — verification email and password-reset email (`cross-stitch/src/lib/email-service.ts:15`); Uploader-side newsletter blasts (`EmailHelper.cs`).
- **Node `crypto.randomUUID` / `crypto.randomBytes`** — token + cid generation in cross-stitch (`cross-stitch/src/lib/users.ts:13`, `cross-stitch/src/lib/password-reset.ts:10`).
- **.NET `System.Security.Cryptography.RandomNumberGenerator`** — Uploader token generation (`Uploader/Uploader/MainWindow.xaml.cs:2336`); `System.Guid.NewGuid` for cid back-fill (`:1489`).
- **Related contracts (do not duplicate):**
  - `dynamodb-schema.md` — full attribute typing / table layout / GSIs.
  - `s3-paths.md`, `pdf-structure.md`, `pinterest-metadata.md`, `album-id.md`, `design-id.md` — referenced from the existing contracts inventory; none touch user identity.
  - A future `email-and-unsubscribe.md` should consume this contract for the `UnsubscribeToken` definition.

## 9. Error Handling

Observed (code-level) failure modes, not theoretical:

1. **Duplicate-email registration.** `saveUserToDynamoDB` uses `ConditionExpression: 'attribute_not_exists(ID)'` (`cross-stitch/src/lib/users.ts:358`). On collision, the SDK throws `ConditionalCheckFailedException`, which `cross-stitch/src/lib/users.ts:368-373` translates into a typed `EmailExistsError`; `api/register-only/route.ts:80-84` returns HTTP 409. **Caveat:** uniqueness depends on the email already being trimmed/lower-cased (§4.1).

2. **`UnsubscribeToken` back-fill race / two-writer hazard.** Both cross-stitch (`users.ts:302`) and Uploader (`MainWindow.xaml.cs:1325-1332`) can write the attribute. Uploader's writer guards on `hasToken` so it will not overwrite an existing token, but the two token shapes (UUID vs URL-safe base64 of 32 random bytes) are **not interchangeable for re-discovery debugging**. Surface: if a user registers in cross-stitch *while* Uploader is mid-scan, both could attempt a write; the Uploader path's `attribute_not_exists` guard is **not** present in the `UpdateItemRequest` (`MainWindow.xaml.cs:1351-1359`) — last write wins. Risk is small (Uploader is interactive, single-user) but unsealed.

3. **Missing attribute on read.** cross-stitch defensively reads each attribute with optional chaining; e.g. `item.Email?.S?.trim().toLowerCase()` (`cross-stitch/src/lib/data-access.ts:78`), `item.SubscriptionId?.S?.trim() || null` (`cross-stitch/src/lib/users.ts:168`). Missing attributes degrade to `null`/`undefined`/`false`/`Set()`. No throw.

4. **`UnsubscribeToken` missing at send time.** Uploader throws `InvalidOperationException("Unsubscribe token for {email} was not found in the database.")` from both `BuildUnsubscribeUrlFromStoredToken` (`Uploader/Uploader/MainWindow.xaml.cs:2319`) and the per-row lookup loop (`:2297, 2303`). Sending is aborted for that recipient. Mitigation is to run `InitializeUserUnsubscribe_Click` first.

5. **Verification token expired.** `verifyUserByToken` logs `Verification token expired for user <id>` (`cross-stitch/src/lib/users.ts:507`) and returns `null`. The caller (verify route) is responsible for surfacing this to the user; the row is **not** updated.

6. **Password reset token consume failures.** Missing record, missing email/expires fields, expired, or NaN expiry — all silently delete the token and return `null` (`cross-stitch/src/lib/password-reset.ts:97-127`). No distinction in the return type between expired and not-found.

7. **PayPal webhook signature failures.** Skipped/bypassed when `PAYPAL_WEBHOOK_SKIP_SIGNATURE_VERIFICATION === 'true'` (`cross-stitch/src/app/api/paypal-webhook/route.js:13-14, 137-142`) — admin is notified but the event is accepted. This is a deliberate dev/testing knob with production blast radius (subscription state can be flipped without verification).

8. **`LastEmailEntry` / `LastSeenAt` updates silently no-op** when the table env-var is missing or the user can't be found (`cross-stitch/src/lib/data-access.ts:768-799`, `cross-stitch/src/lib/users.ts:577-580, 666-668`).

9. **`RemoveSuppressedUsersAsync` deletes from `CrossStitchItems` only**, not from `CrossStitchUsers` (`Uploader/Uploader/MainWindow.xaml.cs:2985-3038`). After suppression a user can still log in via the secondary table fallback (`cross-stitch/src/lib/data-access.ts:670-680`). Surface as an invariant violation if "suppressed" is supposed to mean "removed everywhere."

## 10. Security & Compliance

**The following are open security-debt items, surfaced as part of authoring this contract.**

### 10.1 Plaintext password storage (P1)

- The `Password` attribute on `CrossStitchUsers` rows holds the user's password in **cleartext**. The exact code is:
  ```ts
  Password: { S: password }, // Storing plain password for migration purposes
  ```
  at `cross-stitch/src/lib/users.ts:325`. The "for migration purposes" comment has been present long enough to coexist with a stale docstring (lines 288-289 of the same file) that promises `passwordHash` + `passwordSalt` columns that do not exist.
- The primary single-table mirror uses the attribute name `OpenPwd`. Login compares it with `===` (string equality, no constant-time compare): `cross-stitch/src/lib/data-access.ts:684, 691`.
- `updateUserPassword` (`cross-stitch/src/lib/password-reset.ts:144-166`) overwrites the same plaintext attribute on reset — no hash is introduced even at password-rotation time.
- **Impact:** any read access to the `CrossStitchUsers` or `CrossStitchItems` tables (IAM principal with DynamoDB read, AWS account compromise, accidental export) exposes all customer passwords directly. Many users reuse passwords; the blast radius is not bounded to cross-stitch.com.
- **Recommended remediation (out of scope here, captured for follow-up):**
  1. Add `PasswordHash` + `PasswordSalt` attributes using a memory-hard KDF (Argon2id or scrypt). Write both on new registrations and on `updateUserPassword`.
  2. Make `verifyUserWithProfile` prefer the hashed pair; only fall back to plaintext for un-migrated rows, and lazy-migrate on successful login.
  3. Once all live rows have hashes, drop the `Password` / `OpenPwd` attributes.

### 10.2 Credentials logged to stdout (P1)

- `verifyUserWithProfile` logs the user's password verbatim:
  ```ts
  console.log('verifyUser called with:', { email, password });
  // ...
  const storedPassword = Items[0].OpenPwd?.S;
  console.log('Stored password:', storedPassword);
  ```
  (`cross-stitch/src/lib/data-access.ts:653, 684-685`).
- Stdout from cross-stitch is captured by Elastic Beanstalk → CloudWatch Logs, so every successful and failed login leaks the plaintext password into a logging system whose access controls are looser than DynamoDB's. Search and rotation are non-trivial.
- The PayPal webhook route also dumps the full webhook body (which can include subscriber email + custom_id) via `console.log('Received PayPal webhook:', body)` at `cross-stitch/src/app/api/paypal-webhook/route.js:28`. Lower severity but still PII in logs.
- **Recommended remediation:** remove the `password` field from every `console.log` in `data-access.ts` (and any test fixtures), then scrub historical CloudWatch streams.

### 10.3 PayPal webhook signature can be bypassed via env flag (P2)

- `PAYPAL_WEBHOOK_SKIP_SIGNATURE_VERIFICATION=true` accepts and acts on the webhook with only an admin notification (`cross-stitch/src/app/api/paypal-webhook/route.js:13-14, 137-142`). If this flag is ever set in prod, any actor who can reach `/api/paypal-webhook` can flip `SubscriptionActive` on any subscription id they know. Treat this flag as a development-only feature and verify it is unset in EB env vars before each deploy.

### 10.4 `UnsubscribeSecret` is referenced but unused (P3)

- `Uploader/Uploader/App.private.config.example:8` lists `UnsubscribeSecret`, suggesting an HMAC-envelope design. The current code embeds the raw `UnsubscribeToken` in the URL with no HMAC wrapper (`Uploader/Uploader/MainWindow.xaml.cs:2306-2314`). Either (a) implement HMAC verification on `/unsubscribe` and Uploader-side signing, or (b) delete the dead config key to avoid confusion.

### 10.5 PII inventory on the user row

- The row holds: lower-cased email (PK), first name, plaintext password, multiple correlation tokens (cid, UnsubscribeToken, VerificationToken), PayPal subscription id, behavioral counters (`LastSeenAt`, `LastSeenCount`, `LastEmailEntry`, `TrialDownloadedDesignIds`). Under GDPR Article 17 (right to erasure) you must delete from **both** `CrossStitchUsers` and any `USR#`-prefixed row in `CrossStitchItems`. `RemoveSuppressedUsersAsync` only handles the latter; the former needs a paired path before the suppression flow is considered complete.

## 11. Testing & Validation

A single `USR#<email>` row should satisfy the following projection. Substitute a real email and run with AWS credentials granting `dynamodb:GetItem`:

```bash
aws dynamodb get-item \
  --table-name CrossStitchUsers \
  --key '{"ID":{"S":"USR#test@example.com"}}' \
  --projection-expression "ID, Email, FirstName, cid, UnsubscribeToken, Unsubscribed, Verified, VerifiedAt, VerificationToken, VerificationTokenExpiresAt, CreatedAt, ReceiveUpdates, SubscriptionId, SubscriptionActive"
```

Expected on a row created by the cross-stitch register-only flow (`cross-stitch/src/app/api/register-only/route.ts:46-53`):

- `ID.S == "USR#" + lowercase(Email.S)`.
- `cid.S` parses as a UUID (dashed or 32-hex — accept both shapes per §4.2).
- `UnsubscribeToken.S` is present and non-empty; `Unsubscribed.BOOL == false`.
- Before verify-click: `Verified.BOOL == false`, `VerifiedAt.S == ""`, `VerificationToken.S` non-empty, `VerificationTokenExpiresAt.S` within `CreatedAt + ~48h`.
- After verify-click: `Verified.BOOL == true`, `VerifiedAt.S` ≈ now, `VerificationToken` and `VerificationTokenExpiresAt` attributes **absent** (removed by the `REMOVE` clause in `cross-stitch/src/lib/users.ts:518`).
- `Password.S` should be PRESENT (plaintext today — see §10) — flag any row where it is missing.

For Uploader-back-filled rows, additionally check that `cid` is 32 lowercase hex chars without dashes (Guid `"N"` format) and that `UnsubscribeToken` decodes as URL-safe base64.

Cross-stitch unit test seed: a Vitest case in `cross-stitch/` should assert `userId.startsWith('USR#')` and that `randomUUID` is called exactly twice (cid + UnsubscribeToken) plus once for VerificationToken when not supplied (`cross-stitch/src/lib/users.ts:302-304`).

## 12. References

### Source code (cross-stitch)

- `cross-stitch/src/lib/users.ts:230` — `USR#` PK construction.
- `cross-stitch/src/lib/users.ts:292-377` — `saveUserToDynamoDB`, the canonical user-create write.
- `cross-stitch/src/lib/users.ts:325` — **the plaintext-password line** (security debt §10.1).
- `cross-stitch/src/lib/users.ts:387-441` — `getVerifiedUserByCid`.
- `cross-stitch/src/lib/users.ts:446-533` — `verifyUserByToken`.
- `cross-stitch/src/lib/users.ts:539-593` — `updateLastEmailEntryInUsersTable` (cid → LastEmailEntry).
- `cross-stitch/src/lib/users.ts:1230-1336` — unsubscribe-token resolution.
- `cross-stitch/src/lib/password-reset.ts` — full reset-token surface.
- `cross-stitch/src/lib/data-access.ts:48-93` — `verifyUserInSecondaryTable` (plaintext password compare).
- `cross-stitch/src/lib/data-access.ts:648-712` — `verifyUserWithProfile` / `verifyUser` (plaintext password logging at line 653).
- `cross-stitch/src/lib/data-access.ts:766-812` — `updateLastEmailEntryByCid` (primary table).
- `cross-stitch/src/lib/email-service.ts` — SES wrapper used for verification email.
- `cross-stitch/src/app/api/register-only/route.ts` — registration HTTP entry point.
- `cross-stitch/src/app/api/paypal-webhook/route.js:148-211` — subscription-state mutations on user.

### Source code (Uploader)

- `Uploader/Uploader/App.config:7` — `UsersTableName=CrossStitchUsers`.
- `Uploader/Uploader/App.config:18` — `UnsubscribeBaseUrl=https://cross-stitch.com/unsubscribe`.
- `Uploader/Uploader/App.private.config.example:8` — `UnsubscribeSecret` (unused; §10.4).
- `Uploader/Uploader/MainWindow.xaml.cs:467, 1275` — `InitializeUserUnsubscribe*`.
- `Uploader/Uploader/MainWindow.xaml.cs:502, 1419` — `InitializeUserCid*` (back-fill `cid` on `CrossStitchUsers`).
- `Uploader/Uploader/MainWindow.xaml.cs:1538` — `InitializeItemsUserCidFieldsAsync` (back-fill `cid` on `CrossStitchItems`).
- `Uploader/Uploader/MainWindow.xaml.cs:2306-2322` — `BuildUnsubscribeUrl` / `BuildUnsubscribeUrlFromStoredToken`.
- `Uploader/Uploader/MainWindow.xaml.cs:2333-2350` — `GenerateRandomToken` + `ToBase64Url`.
- `Uploader/Uploader/MainWindow.xaml.cs:2352-2368` — `AppendTrackingParameters` (`cid` + `eid` query params).
- `Uploader/Uploader/MainWindow.xaml.cs:2983-3070` — `RemoveSuppressedUsersAsync` (only touches `CrossStitchItems`).
- `Uploader/Uploader/MainWindow.xaml.cs:3072+` — `MarkUsersVerifiedAsync`.

### Related docs (do not duplicate)

- `cross-stitch-platform-docs/docs/integration/dynamodb-schema.md` — per-attribute DDB typing.
- `cross-stitch-platform-docs/docs/cross-stitch/platform-architecture-summary.md` — high-level facts incl. `user` section that seeded this contract.
- `cross-stitch-platform-docs/plan/integration/CONTRACT-TEMPLATE.md` — template this doc follows.
- Existing contracts inventory: `album-id`, `design-id`, `dynamodb-schema`, `s3-paths`, `pdf-structure`, `pinterest-metadata`, `README`.
- Suggested follow-up: `email-and-unsubscribe.md` (UnsubscribeToken flow end-to-end) and `paypal-subscription.md` (subscription state machine) — both should `import` the identity attributes defined here.
