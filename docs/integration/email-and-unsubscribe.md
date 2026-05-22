# Email and unsubscribe — SES newsletter, transactional, and unsubscribe URL contract

## 1. Contract Name

**Email and unsubscribe (SES sender, newsletter blasts, transactional, unsubscribe URL, suppression).**

This contract defines the cross-repo agreement between the WPF Uploader (newsletter blast sender) and the Next.js cross-stitch web app (transactional sender + unsubscribe receiver) for delivering email via AWS SES, formatting newsletter content, embedding the unsubscribe URL, and cleaning up DynamoDB after SES suppresses an address.

## 2. Purpose

Two senders share one mailing list and one SES tenant:

- **Uploader** sends newsletter blasts (`BtnSendEmails_Click` at `Uploader/Uploader/MainWindow.xaml.cs:194` and `BtnSendTextEmails_Click` at `Uploader/Uploader/MainWindow.xaml.cs:241`) and per-upload admin notifications (`SendNotificationMailToAdminAsync` at `Uploader/Uploader/MainWindow.xaml.cs:1228`).
- **cross-stitch** sends transactional mail (`cross-stitch/src/lib/email-service.ts:27` `sendEmail`) and password resets (`cross-stitch/src/lib/password-reset-email.ts:18` `sendPasswordResetEmail`), and owns the unsubscribe landing page (`cross-stitch/src/app/unsubscribe/page.tsx:32`).

The shared coupling points are: (a) the `UnsubscribeToken` attribute on the user record (written by cross-stitch on registration, embedded in mail by Uploader, looked up by cross-stitch on click), (b) the `https://cross-stitch.com/unsubscribe?token=…` URL, (c) the `Unsubscribed` boolean and (d) the SES suppression list, which Uploader uses to delete suppressed records from DynamoDB.

## 3. Scope

**In scope.**
- AWS SES sender identities (`SenderEmail`, `AdminEmail`, optional `SesConfigurationSetName`) used by Uploader.
- Cross-stitch SES sender defaults (`AWS_SES_FROM_EMAIL`, `ADMIN_EMAIL`, `SES_FROM_EMAIL`) (`cross-stitch/src/lib/email-service.ts:11-13`, `cross-stitch/src/lib/password-reset-email.ts:7-11`).
- Newsletter HTML and text template files and their `[Section]` / placeholder format (`cross-stitch-platform-docs/docs/uploader/HtmlEmailTemplate.txt:1-36`, `cross-stitch-platform-docs/docs/uploader/TextEmailTemplate.txt:1-35`).
- Unsubscribe URL construction (`Uploader/Uploader/MainWindow.xaml.cs:2306`) and one-click `List-Unsubscribe` headers (`Uploader/Uploader/MainWindow.xaml.cs:2435-2443`).
- Unsubscribe URL resolution on the receiver side (`cross-stitch/src/app/unsubscribe/page.tsx:32-84` and `cross-stitch/src/lib/users.ts:1282-1336`).
- DDB suppression cleanup driven by an operator-maintained file (`Uploader/Uploader/MainWindow.xaml.cs:83`, `RemoveSuppressedUsersAsync` at `Uploader/Uploader/MainWindow.xaml.cs:2983`).

**Out of scope.**
- SES domain/identity verification, DKIM/SPF/DMARC and the AWS-side suppression-list configuration itself (operator-managed in the AWS console; not in either repo).
- DynamoDB user schema (deferred to `dynamodb-schema.md`).
- User-identity attribute semantics for `UnsubscribeToken` and `cid` (deferred to user-identity / `dynamodb-schema.md`).
- Tracking-parameter (`cid`, `eid`) encoding for in-email links (referenced but not specified here; see `AppendTrackingParameters` at `Uploader/Uploader/MainWindow.xaml.cs:2418-2433`).

## 4. Data Formats

### 4.1 SES sender configuration

| Setting | Source | Default / observed | Cited |
| --- | --- | --- | --- |
| `SenderEmail` | Uploader `App.private.config` | operator-supplied (`your-sender@example.com` in example) | `Uploader/Uploader/App.private.config.example:3` |
| `AdminEmail` | Uploader `App.private.config` | operator-supplied (`your-admin@example.com` in example) | `Uploader/Uploader/App.private.config.example:4` |
| `SesConfigurationSetName` | Uploader `App.private.config` | operator-supplied; `null` allowed | `Uploader/Uploader/App.private.config.example:9`, `Uploader/Uploader/MainWindow.xaml.cs:52` |
| `UnsubscribeBaseUrl` | Uploader `App.config` | `https://cross-stitch.com/unsubscribe` | `Uploader/Uploader/App.config:18` |
| `UsersTableName` | Uploader `App.config` | `CrossStitchUsers` | `Uploader/Uploader/App.config:7` |
| `UserEmailAttribute` | Uploader `App.config` | `Email` | `Uploader/Uploader/App.config:8` |
| `HtmlEmailTemplatePath` | Uploader `App.config` | `D:\ann\Git\cross-stitch-platform-docs\plan\uploader\HtmlEmailTemplate.txt` (absolute Windows path, operator-specific) | `Uploader/Uploader/App.config:25` |
| `TextEmailTemplatePath` | Uploader `App.config` | `D:\ann\Git\cross-stitch-platform-docs\plan\uploader\TextEmailTemplate.txt` | `Uploader/Uploader/App.config:26` |
| `AWS_SES_FROM_EMAIL` | cross-stitch env | falls back to `buildSiteEmail('ann')` | `cross-stitch/src/lib/email-service.ts:12` |
| `ADMIN_EMAIL` | cross-stitch env | falls back to `olga.epstein@gmail.com` (hardcoded) | `cross-stitch/src/lib/email-service.ts:13` |
| `SES_FROM_EMAIL` | cross-stitch env | falls back to `buildSiteEmail('no-reply')` | `cross-stitch/src/lib/password-reset-email.ts:10-11` |
| `AWS_REGION` | both | `us-east-1` (default) | `cross-stitch/src/lib/email-service.ts:11`, `cross-stitch/src/lib/password-reset-email.ts:7` |

### 4.2 Newsletter template format

Both `HtmlEmailTemplate.txt` and `TextEmailTemplate.txt` are operator-edited plain-text files split into named `[Section]` blocks. The parser requires the following sections per template kind:

- **HTML** (`Uploader/Uploader/MainWindow.xaml.cs:86-87`): `Subject`, `Greeting`, `BeforeImage`, `ImageWithLink`, `AfterImage`, `Unsubscribe`, `Closing`, `Signature`.
- **Text** (`Uploader/Uploader/MainWindow.xaml.cs:88-89`): `Subject`, `Greeting`, `BeforeBody`, `AfterBody`, `Unsubscribe`, `Closing`, `Signature`.

Verbatim placeholders observed in the live templates (`cross-stitch-platform-docs/docs/uploader/HtmlEmailTemplate.txt:1-36`):

| Placeholder | Substituted by | Citation |
| --- | --- | --- |
| `[FName]` (in `[Greeting]`) | `recipient.FirstName` (or `"admin"` for previews) | `cross-stitch-platform-docs/docs/uploader/HtmlEmailTemplate.txt:5`, `Uploader/Uploader/MainWindow.xaml.cs:2105`, `Uploader/Uploader/MainWindow.xaml.cs:2625` |
| `<pattern_url>` (in `[ImageWithLink]`, `[AfterBody]`) | `BuildPatternUrl(latestDesign)` + `AppendTrackingParameters` (cid, eid=yyMMdd) | `cross-stitch-platform-docs/docs/uploader/HtmlEmailTemplate.txt:14-15`, `Uploader/Uploader/MainWindow.xaml.cs:2618` |
| `<image_url>` (in `[ImageWithLink]`) | `_linkHelper.BuildImageUrl(designId, albumId)` | `cross-stitch-platform-docs/docs/uploader/HtmlEmailTemplate.txt:14`, `Uploader/Uploader/MainWindow.xaml.cs:2089` |
| `<unsubscribe_url>` (in `[Unsubscribe]`) | `BuildUnsubscribeUrlFromStoredToken(email, token)` | `cross-stitch-platform-docs/docs/uploader/HtmlEmailTemplate.txt:29`, `cross-stitch-platform-docs/docs/uploader/TextEmailTemplate.txt:28`, `Uploader/Uploader/MainWindow.xaml.cs:2508`, `Uploader/Uploader/MainWindow.xaml.cs:2621` |
| `[Subject]` block | Email `Subject` header (verbatim, first line under the marker) | `cross-stitch-platform-docs/docs/uploader/HtmlEmailTemplate.txt:1-2` |

Example `[Subject]` and `[Greeting]` (running-horse running campaign, verbatim from `cross-stitch-platform-docs/docs/uploader/HtmlEmailTemplate.txt:1-5`):

```
[Subject]
Running horse cross-stitch pattern – new freebie!

[Greeting]
Hi [FName],
```

### 4.3 Unsubscribe URL format

`https://cross-stitch.com/unsubscribe?token={Uri.EscapeDataString(token)}` (`Uploader/Uploader/MainWindow.xaml.cs:2313`). The base URL is read from `UnsubscribeBaseUrl` in `App.config` and the trailing `/` is trimmed (`Uploader/Uploader/MainWindow.xaml.cs:2308-2311`). If the config key is missing, the code falls back to `{SiteBaseUrl}/unsubscribe`.

`token` is the per-user `UnsubscribeToken` DynamoDB attribute. cross-stitch generates it with `randomUUID()` at user creation (`cross-stitch/src/lib/users.ts:302`, `cross-stitch/src/lib/users.ts:328`); Uploader back-fills it for users missing the attribute (`InitializeUserUnsubscribeFieldsAsync` at `Uploader/Uploader/MainWindow.xaml.cs:1275`). The token is not signed: there is no HMAC computed at write or at click time (see §10).

### 4.4 List-Unsubscribe headers (RFC 8058 one-click)

For every newsletter recipient Uploader attaches:

```
List-Unsubscribe: <mailto:{sender}>, <{unsubscribeUrl}>
List-Unsubscribe-Post: List-Unsubscribe=One-Click
```

(`Uploader/Uploader/MainWindow.xaml.cs:2438-2442`). Because the message now carries custom headers, `EmailHelper.SendEmailAsync` switches from `SendEmailAsync` to `SendRawEmailAsync` and emits a hand-built `multipart/alternative` MIME body (`Uploader/Uploader/Helpers/EmailHelper.cs:28-42`, `Uploader/Uploader/Helpers/EmailHelper.cs:74-130`).

### 4.5 Admin-preview unsubscribe token

When the operator sends a preview to themselves (`SendAdminUserStyleEmailAsync` at `Uploader/Uploader/MainWindow.xaml.cs:2078`), the constant string `"preview-admin-unsubscribe-token"` is embedded (`Uploader/Uploader/MainWindow.xaml.cs:82`, `Uploader/Uploader/MainWindow.xaml.cs:2101`). This is a sentinel: it will not resolve in `CrossStitchUsers`, so clicking it on a preview message returns "Link not recognized" (`cross-stitch/src/app/unsubscribe/page.tsx:73-77`).

### 4.6 Suppression input file

The suppression cleanup reads a hardcoded operator file:

```
D:\ann\Git\cross-stitch\list-suppressed.txt
```

(`Uploader/Uploader/MainWindow.xaml.cs:83`). The operator exports the SES account-level suppression list to this file out-of-band; the contract is "one email address per line" by virtue of `ReadSuppressedEmails(SuppressedListPath)` (`Uploader/Uploader/MainWindow.xaml.cs:3778`). For every email, Uploader deletes every `(ID=USR#<email>, NPage=<each NPage>)` row from `CrossStitchItems` (`Uploader/Uploader/MainWindow.xaml.cs:3026-3037`). It does **not** flip `Unsubscribed` and does **not** touch `CrossStitchUsers`.

## 5. API Endpoints / Interfaces

This is a multi-protocol contract with **no HTTP API between repos** (consistent with `cross-stitch-platform-docs/docs/cross-stitch/platform-architecture-summary.md:249`). The interface surfaces are:

### 5.1 AWS SES — Uploader (sender)

- `EmailHelper.SendEmailAsync(sesClient, sender, recipients, subject, textBody, htmlBody?, headers?, configurationSetName?)` (`Uploader/Uploader/Helpers/EmailHelper.cs:17`). If `headers` is non-empty (i.e., a newsletter with `List-Unsubscribe`), it routes to `SendRawEmailAsync` (`Uploader/Uploader/Helpers/EmailHelper.cs:74`); otherwise it calls `AmazonSimpleEmailServiceClient.SendEmailAsync` (`Uploader/Uploader/Helpers/EmailHelper.cs:71`).
- Call sites:
  - `SendNotificationMailToAdminAsync` (`Uploader/Uploader/MainWindow.xaml.cs:1252`) — single-recipient admin notification, no headers, subject `"Upload Successful"` (`Uploader/Uploader/MainWindow.xaml.cs:1256`).
  - `SendAdminUserStyleEmailAsync` (`Uploader/Uploader/MainWindow.xaml.cs:2120`) — admin self-preview of a newsletter, with `List-Unsubscribe` headers and the preview token.
  - `SendTextEmailsWithProgressAsync` (`Uploader/Uploader/MainWindow.xaml.cs:2514`) — text-only newsletter blast.
  - `SendEmailsWithProgressAsync` (`Uploader/Uploader/MainWindow.xaml.cs:2632`) — HTML newsletter blast.

### 5.2 AWS SES — cross-stitch (sender)

- `sendEmail({ to, subject, body, html?, from? })` (`cross-stitch/src/lib/email-service.ts:27`) — single-recipient SES `SendEmailCommand`. Defaults: `Source = FROM_EMAIL`, charset `UTF-8`, HTML when `html === true`.
- `sendEmailToAdmin(subject, body, html?)` (`cross-stitch/src/lib/email-service.ts:55`) — fixed-recipient wrapper sending to `ADMIN_EMAIL`.
- `sendPasswordResetEmail({ to, token })` (`cross-stitch/src/lib/password-reset-email.ts:18`) — builds a `/reset-password/{token}` link via `buildCanonicalUrl`, sends both text and HTML parts.

### 5.3 Unsubscribe receiver (cross-stitch)

- **Route:** `GET https://cross-stitch.com/unsubscribe?token=<UnsubscribeToken>` (Next.js App Router page, `cross-stitch/src/app/unsubscribe/page.tsx:32`).
- **Behaviour:** server component, `force-dynamic` (`cross-stitch/src/app/unsubscribe/page.tsx:6`), one-click (no GET-to-POST confirmation step).
- **Lookup:** `unsubscribeUserByToken(token)` (`cross-stitch/src/lib/users.ts:1282`) — does a `Scan` of `CrossStitchUsers` filtered by `UnsubscribeToken = :token` (`cross-stitch/src/lib/users.ts:1230-1276`); if `DYNAMODB_TABLE_NAME` env is set and differs, also scans the primary table filtered by `EntityType = USER` (`cross-stitch/src/lib/users.ts:1294-1296`).
- **Mutation:** if found and `Unsubscribed != true`, `SET Unsubscribed = :true` (`cross-stitch/src/lib/users.ts:1311-1324`). Returns `{ status: 'updated' | 'already-unsubscribed' | 'not-found', email? }`.
- **Side effect:** on `'updated'`, the page sends an admin notification via `sendEmailToAdmin('User unsubscribed', …, false)` (`cross-stitch/src/app/unsubscribe/page.tsx:57-62`); failure to notify is logged and swallowed (`cross-stitch/src/app/unsubscribe/page.tsx:63-68`).
- **Response:** HTML page with one of four titles: `"Processing unsubscribe request"`, `"You are unsubscribed"`, `"Already unsubscribed"`, `"Link not recognized"`, `"Unable to complete request"` (`cross-stitch/src/app/unsubscribe/page.tsx:38-83`). The page is `noindex, nofollow` (`cross-stitch/src/app/unsubscribe/page.tsx:11`).

### 5.4 DynamoDB — shared attributes

These are read/written by both repos; see [`dynamodb-schema.md`](./dynamodb-schema.md) for the full schema. Only the email-relevant subset:

| Attribute | Writer | Reader (for email/unsubscribe) | Citation |
| --- | --- | --- | --- |
| `Email` (S, lowercased) | cross-stitch on create; Uploader back-fill | Uploader to address recipients | `cross-stitch/src/lib/users.ts:295`, `cross-stitch/src/lib/users.ts:323`, `Uploader/Uploader/MainWindow.xaml.cs:2278` |
| `FirstName` (S) | cross-stitch on create | Uploader for `[FName]` substitution | `cross-stitch/src/lib/users.ts:324`, `Uploader/Uploader/MainWindow.xaml.cs:2056` |
| `UnsubscribeToken` (S, UUID) | cross-stitch on create (`randomUUID()`); Uploader back-fill | Uploader embeds in URL; cross-stitch matches on click | `cross-stitch/src/lib/users.ts:302`, `cross-stitch/src/lib/users.ts:328`, `Uploader/Uploader/MainWindow.xaml.cs:2049-2054`, `cross-stitch/src/lib/users.ts:1239-1244` |
| `Unsubscribed` (BOOL) | cross-stitch on create (false); cross-stitch on click | Uploader filter (`onlySubscribed`) skips when true | `cross-stitch/src/lib/users.ts:329`, `cross-stitch/src/lib/users.ts:1315-1322`, `Uploader/Uploader/MainWindow.xaml.cs:2041-2046` |
| `LastEmailDate` / `LastEmailEntry` | Uploader on send (best-effort) | none | `Uploader/Uploader/MainWindow.xaml.cs:2526-2528`, `cross-stitch/src/lib/users.ts:539-593` |
| `cid` (S, UUID) | cross-stitch on create | Uploader for tracking params | `cross-stitch/src/lib/users.ts:303`, `cross-stitch/src/lib/users.ts:330`, `Uploader/Uploader/MainWindow.xaml.cs:2056`, `Uploader/Uploader/MainWindow.xaml.cs:2510` |

### 5.5 Suppression file

- **Path (hardcoded):** `D:\ann\Git\cross-stitch\list-suppressed.txt` (`Uploader/Uploader/MainWindow.xaml.cs:83`).
- **Format:** one email per line, parsed by `ReadSuppressedEmails(SuppressedListPath)` (`Uploader/Uploader/MainWindow.xaml.cs:3778`).
- **Trigger:** `RemoveSuppressedUsers_Click` (`Uploader/Uploader/MainWindow.xaml.cs:3772`).
- **Effect:** for each address, delete every `(ID=USR#<email>, NPage=*)` row from `CrossStitchItems` (`Uploader/Uploader/MainWindow.xaml.cs:3026-3037`). The CrossStitchUsers row is **not** updated, and `Unsubscribed` is **not** flipped (this is a hard delete, not a soft unsubscribe).

## 6. Versioning

**Unversioned (implicit).** No version field exists anywhere in the templates, the URL, the headers, or the DDB attributes. The two file template formats and the URL shape are coordinated only by the fact that both repos live in one developer's tree.

**Observed drift risk.** `HtmlEmailTemplate.txt` / `TextEmailTemplate.txt` exist in three places: `cross-stitch-platform-docs/docs/uploader/`, `cross-stitch-platform-docs/docs/uploader/templates/`, and `cross-stitch-platform-docs/plan/uploader/`. Uploader's `App.config` points at the `plan/uploader/` copies via absolute Windows paths (`Uploader/Uploader/App.config:25-26`), while the docs hub treats `docs/uploader/` as the source of truth (see `cross-stitch-platform-docs/docs/integration/README.md:5`). Three on-disk copies with no marker for which is canonical.

**Recommendation.** Pin one canonical location (`docs/uploader/`), make Uploader read from there (or read via `PLATFORM_CONFIG_PATH`), and delete the duplicates. Add a `# template-version: YYYY-MM-DD` header to the template files so a sender can refuse an out-of-date file. If the URL ever needs to evolve, embed a version under the path (`/unsubscribe/v2?…`) rather than mutating the existing one.

## 7. Ownership & Contacts

- **Maintainer:** Olga (`epolga`, `olga.epstein@gmail.com`).
- **Code owners by surface:**
  - SES sender plumbing, newsletter templates, suppression cleanup → **Uploader** (`Uploader/Uploader/Helpers/EmailHelper.cs`, `Uploader/Uploader/MainWindow.xaml.cs`).
  - Transactional email + password reset + unsubscribe receiver → **cross-stitch** (`cross-stitch/src/lib/email-service.ts`, `cross-stitch/src/lib/password-reset-email.ts`, `cross-stitch/src/app/unsubscribe/page.tsx`, `cross-stitch/src/lib/users.ts`).
  - `UnsubscribeToken` value: writer-of-record is **cross-stitch** (`cross-stitch/src/lib/users.ts:302`); Uploader is back-fill only and must not overwrite an existing token (cross-reference user-identity contract; flagged at `cross-stitch-platform-docs/docs/cross-stitch/platform-architecture-summary.md:248`).
  - Admin notification recipient is whatever `AdminEmail` resolves to in operator-local `App.private.config` (`Uploader/Uploader/App.private.config.example:4`); cross-stitch hardcodes `olga.epstein@gmail.com` as the default (`cross-stitch/src/lib/email-service.ts:13`).

## 8. Dependencies

### 8.1 External services
- **AWS SES** in `us-east-1` (sender, suppression list owner). Charged per recipient; subject to send-quota and per-second send-rate limits configured on the AWS account.
- **AWS DynamoDB** tables `CrossStitchUsers` and `CrossStitchItems` (for `Unsubscribed` mutation and suppression-driven deletes). See [`dynamodb-schema.md`](./dynamodb-schema.md).

### 8.2 SDKs / libraries
- Uploader: `AWSSDK.SimpleEmail` (`Amazon.SimpleEmail.AmazonSimpleEmailServiceClient`, `Amazon.SimpleEmail.Model.SendEmailRequest`, `SendRawEmailRequest`) — `Uploader/Uploader/Helpers/EmailHelper.cs:7-8`.
- cross-stitch: `@aws-sdk/client-ses` v3 (`SESClient`, `SendEmailCommand`) — `cross-stitch/src/lib/email-service.ts:4-8`, `cross-stitch/src/lib/password-reset-email.ts:1-4`.

### 8.3 Configuration / env
- Uploader: `App.config` (committed) + `App.private.config` (operator-local, gitignored). Required keys: `SenderEmail`, `AdminEmail`, optional `SesConfigurationSetName`, optional `UnsubscribeSecret` (declared in example but unused — see §10).
- cross-stitch: env vars `AWS_REGION`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SES_FROM_EMAIL`, `SES_FROM_EMAIL`, `ADMIN_EMAIL`, `DDB_USERS_TABLE`, `DYNAMODB_TABLE_NAME` (`cross-stitch/src/lib/email-service.ts:11-13`, `cross-stitch/src/lib/users.ts:15-17`).

### 8.4 Sibling contracts
- [`dynamodb-schema.md`](./dynamodb-schema.md) — owns the `UnsubscribeToken`, `Unsubscribed`, `Email`, `FirstName`, `cid`, `LastEmailDate`, `LastEmailEntry` attribute definitions.
- [`s3-paths.md`](./s3-paths.md) — owns `<image_url>` resolution rules.
- [`pinterest-metadata.md`](./pinterest-metadata.md) — shares the link/tracking helpers in `PatternLinkHelper.cs`.

## 9. Error Handling

### 9.1 Uploader-side

- **Template file missing.** `LoadHtmlEmailTemplate` (`Uploader/Uploader/MainWindow.xaml.cs:2685-2688`) and `LoadTextEmailTemplate` (`Uploader/Uploader/MainWindow.xaml.cs:2691-2694`) take `(configKey, defaultPath)` pairs and fall back to bundled `Templates\HtmlEmailTemplate.txt` / `Templates\TextEmailTemplate.txt` (`Uploader/Uploader/MainWindow.xaml.cs:80-81`) if the configured absolute path is unreadable. Section validation is enforced against the required-section lists (`Uploader/Uploader/MainWindow.xaml.cs:86-89`).
- **Recipient without unsubscribe token.** Skipped, logged as `Skipping {n} recipient(s) without unsubscribe token.` (`Uploader/Uploader/MainWindow.xaml.cs:2479-2491`, `Uploader/Uploader/MainWindow.xaml.cs:2588-2600`). This is the load-bearing reason Uploader runs `InitializeUserUnsubscribeFieldsAsync` back-fill (`Uploader/Uploader/MainWindow.xaml.cs:1275`).
- **`LastEmailDate` update failure.** Caught, logged per-recipient, swallowed; the send is not retried (`Uploader/Uploader/MainWindow.xaml.cs:2524-2535`, `Uploader/Uploader/MainWindow.xaml.cs:2644-2656`).
- **SES `SendEmail` / `SendRawEmail` failure.** Not caught inside the per-recipient loop, so the whole blast aborts and the top-level button handler logs `Failed to send notification emails: {ex.Message}` and shows a `MessageBox` (`Uploader/Uploader/MainWindow.xaml.cs:218-227`).
- **`SenderEmail` or `AdminEmail` missing.** `SendNotificationMailToAdminAsync` and `SendAdminUserStyleEmailAsync` silently `return` if either is null/empty (`Uploader/Uploader/MainWindow.xaml.cs:1233-1234`, `Uploader/Uploader/MainWindow.xaml.cs:2083-2084`). No error is surfaced.
- **Concurrent send guard.** `_isSendingEmails` / `_isSendingTextEmails` flags prevent double-clicking; second click logs `[Email] Send already in progress.` (`Uploader/Uploader/MainWindow.xaml.cs:196-200`, `Uploader/Uploader/MainWindow.xaml.cs:243-247`).
- **Suppression-list missing email lookup.** When `RemoveSuppressedUsersAsync` finds no rows for `USR#<email>` it increments `missingCount` and continues; per-email exceptions increment `errors` (`Uploader/Uploader/MainWindow.xaml.cs:3011-3016`, `Uploader/Uploader/MainWindow.xaml.cs:3040-3047`).

### 9.2 cross-stitch-side

- **Empty / missing token.** Returns `{ status: 'not-found' }` without touching DDB (`cross-stitch/src/lib/users.ts:1285-1288`).
- **Token not in any table.** Returns `'not-found'`; page renders `"Link not recognized"` (`cross-stitch/src/lib/users.ts:1335`, `cross-stitch/src/app/unsubscribe/page.tsx:73-77`).
- **Token already used.** Detected via `Unsubscribed?.BOOL === true`, returns `'already-unsubscribed'`; page renders `"Already unsubscribed"` (`cross-stitch/src/lib/users.ts:1309`, `cross-stitch/src/app/unsubscribe/page.tsx:69-72`).
- **Update failure.** Exception bubbles to the page, which logs and renders `"Unable to complete request"` (`cross-stitch/src/app/unsubscribe/page.tsx:78-83`).
- **Admin-notify failure on success.** Logged, swallowed; the unsubscribe still succeeded (`cross-stitch/src/app/unsubscribe/page.tsx:63-68`).
- **`DDB_USERS_TABLE` env unset.** Many helpers `console.warn` and short-circuit (`cross-stitch/src/lib/users.ts:391-394`, `cross-stitch/src/lib/users.ts:450-453`, `cross-stitch/src/lib/users.ts:542-545`). Note `unsubscribeUserByToken` itself reads `USERS_TABLE_NAME` (which defaults to `CrossStitchUsers` at `cross-stitch/src/lib/users.ts:16`), so it does not silently no-op.

### 9.3 SES quota / throttling / suppression
- **Per-send-rate throttling.** Not handled in code; Uploader does no rate-limit shaping between recipients, only per-50-recipient progress logging (`Uploader/Uploader/MainWindow.xaml.cs:2539-2552`, `Uploader/Uploader/MainWindow.xaml.cs:2660-…`).
- **SES suppression-list bounce.** SES rejects sends to suppressed addresses, which surfaces as a `SendEmail` exception, which (as above) aborts the blast. The operator's recovery flow is: export the suppression list to `list-suppressed.txt`, run `RemoveSuppressedUsers_Click`, re-send.

## 10. Security & Compliance

### 10.1 Secrets

- **AWS credentials.** Resolved by the default AWS SDK credential chain (instance profile / `~/.aws/credentials` on the operator workstation for Uploader; env vars in the Elastic Beanstalk env for cross-stitch — `cross-stitch/src/lib/email-service.ts:17-23`). Neither repo stores them in `App.config` or in source.
- **`SesConfigurationSetName`.** Operator-local in `App.private.config` (`Uploader/Uploader/App.private.config.example:9`). Not a secret, but operator-specific.
- **`UnsubscribeSecret`.** Declared as a key in `App.private.config.example:8` but **not referenced anywhere in the C# source** (grep confirms: only hit is the example file itself). It is an orphan config key — likely a vestige of a planned HMAC scheme that was never implemented. **This means there is no HMAC validation of the unsubscribe token on either side**; cross-stitch verifies the token by a `Scan` for an exact `UnsubscribeToken = :token` match (`cross-stitch/src/lib/users.ts:1239-1244`). Token security therefore depends only on the per-user UUID being unguessable.
- **`PinterestClientSecret`, `PinterestAccessToken`.** Out of scope of this contract; see `pinterest-metadata.md`.

### 10.2 PII in email

- `Email`, `FirstName`, and the user's `cid` (UUID) appear in clear inside outbound mail and in tracking parameters on outbound links (`Uploader/Uploader/MainWindow.xaml.cs:2510-2511`). Both repos store `Email` lowercased in DDB (`cross-stitch/src/lib/users.ts:295`, `cross-stitch/src/lib/users.ts:323`).
- The unsubscribe URL contains only `UnsubscribeToken` (UUID); the email address is not in the link.

### 10.3 One-click unsubscribe

- `List-Unsubscribe-Post: List-Unsubscribe=One-Click` is set (`Uploader/Uploader/MainWindow.xaml.cs:2441`), which under RFC 8058 invites the user agent to send a `POST`. The receiver page is implemented as `GET` only (default Next.js page export at `cross-stitch/src/app/unsubscribe/page.tsx:32-84`); a Gmail-issued one-click `POST` would 405 unless an MTA proxies it.
- No confirmation step. A single GET with a valid token is sufficient to set `Unsubscribed = true`.

### 10.4 Auth on the unsubscribe receiver

- None beyond knowing the token. There is no rate limit, no CAPTCHA, no signature verification (`cross-stitch/src/lib/users.ts:1282-1336`). The threat model assumes the token is sufficiently random (UUIDv4 from `randomUUID()` at `cross-stitch/src/lib/users.ts:302`).

### 10.5 Admin-email exposure
- `AdminEmail` (Uploader) and the hardcoded `olga.epstein@gmail.com` default in `cross-stitch/src/lib/email-service.ts:13` are operator-personal email addresses. Both flow into outbound SES `Destination` lists.

## 11. Testing & Validation

### 11.1 Smoke tests (manual)

1. **Admin notification.** Run an upload via `BtnUpload_Click` and confirm that `SendNotificationMailToAdminAsync` (`Uploader/Uploader/MainWindow.xaml.cs:1228`) delivers a `"Upload Successful"` mail to `AdminEmail`. Status bar should log `Sent notification email to admin.` (`Uploader/Uploader/MainWindow.xaml.cs:1263`).
2. **Admin preview newsletter.** Use the operator's "send to admin" button which calls `SendAdminUserStyleEmailAsync` (`Uploader/Uploader/MainWindow.xaml.cs:2078`). Inspect the received message: verify `[FName] = admin`, that `<unsubscribe_url>` points to `https://cross-stitch.com/unsubscribe?token=preview-admin-unsubscribe-token`, and that clicking it lands on the `"Link not recognized"` page (`cross-stitch/src/app/unsubscribe/page.tsx:73-77`).
3. **Real unsubscribe round-trip.** Pick one row in `CrossStitchUsers`, copy its `UnsubscribeToken`, hit `GET https://cross-stitch.com/unsubscribe?token=<value>`. Confirm `"You are unsubscribed"`, then re-query DDB and confirm `Unsubscribed=true`. Hit the URL again and expect `"Already unsubscribed"` (`cross-stitch/src/app/unsubscribe/page.tsx:69-72`).

### 11.2 Re-verification greps

| Claim | Re-check |
| --- | --- |
| URL shape `https://cross-stitch.com/unsubscribe?token=…` | `rg -n "UnsubscribeBaseUrl" Uploader/Uploader/App.config Uploader/Uploader/MainWindow.xaml.cs` |
| One HMAC-related symbol anywhere | `rg -n "Hmac\|HMAC\|UnsubscribeSecret" Uploader/Uploader cross-stitch/src` — should hit only `App.private.config.example:8` |
| Template placeholders | `rg -n "\[FName\]\|<pattern_url>\|<image_url>\|<unsubscribe_url>" cross-stitch-platform-docs/docs/uploader/` |
| Suppression file path | `rg -n "list-suppressed.txt\|SuppressedListPath" Uploader/Uploader/MainWindow.xaml.cs` |
| Token writer | `rg -n "UnsubscribeToken" cross-stitch/src/lib/users.ts Uploader/Uploader/MainWindow.xaml.cs` |

### 11.3 SDK-level sanity check

```bash
aws ses list-suppressed-destinations --region us-east-1
aws dynamodb get-item --table-name CrossStitchUsers \
    --key '{"ID":{"S":"USR#<email>"}}' \
    --projection-expression "UnsubscribeToken,Unsubscribed,LastEmailDate"
```

If a recipient appears in the SES suppression list but has `Unsubscribed != true` in DDB, the suppression-cleanup flow has fallen behind.

### 11.4 Negative cases worth keeping in mind

- Token containing `+` or `&` — covered by `Uri.EscapeDataString` (`Uploader/Uploader/MainWindow.xaml.cs:2313`); cross-stitch reads `searchParams.token` which Next.js URL-decodes for you (`cross-stitch/src/app/unsubscribe/page.tsx:20-30`).
- Token from a stale email after a back-fill rewrite — currently impossible because the back-fill only sets `UnsubscribeToken` when missing (`Uploader/Uploader/MainWindow.xaml.cs:1271-1274` doc comment), but the invariant is not enforced by code and is flagged as a drift risk in the platform summary (`cross-stitch-platform-docs/docs/cross-stitch/platform-architecture-summary.md:248`).

## 12. References

### 12.1 Code (Uploader)
- `Uploader/Uploader/Helpers/EmailHelper.cs` — `SendEmailAsync` / `SendRawEmailAsync` plumbing.
- `Uploader/Uploader/MainWindow.xaml.cs` — `BtnSendEmails_Click:194`, `BtnSendTextEmails_Click:241`, `SendNotificationMailToAdminAsync:1228`, `SendAdminUserStyleEmailAsync:2078`, `BuildUnsubscribeUrl:2306`, `BuildUnsubscribeUrlFromStoredToken:2316`, `BuildUnsubscribeHeaders:2435`, `SendTextEmailsWithProgressAsync:2459`, `SendEmailsWithProgressAsync:2564`, `LoadHtmlEmailTemplate:2685`, `LoadTextEmailTemplate:2691`, `RemoveSuppressedUsersAsync:2983`, `RemoveSuppressedUsers_Click:3772`, `AdminPreviewUnsubscribeToken:82`, `SuppressedListPath:83`.
- `Uploader/Uploader/App.config` — `SiteBaseUrl:6`, `UsersTableName:7`, `UserEmailAttribute:8`, `UnsubscribeBaseUrl:18`, `HtmlEmailTemplatePath:25`, `TextEmailTemplatePath:26`.
- `Uploader/Uploader/App.private.config.example` — `SenderEmail:3`, `AdminEmail:4`, `UnsubscribeSecret:8` (unused), `SesConfigurationSetName:9`.

### 12.2 Code (cross-stitch)
- `cross-stitch/src/lib/email-service.ts` — `sendEmail`, `sendEmailToAdmin`.
- `cross-stitch/src/lib/password-reset-email.ts` — `sendPasswordResetEmail`.
- `cross-stitch/src/app/unsubscribe/page.tsx` — receiver route.
- `cross-stitch/src/lib/users.ts` — `saveUserToDynamoDB:292` (token mint), `unsubscribeUserByToken:1282`, `findUserByUnsubscribeToken:1230`.

### 12.3 Templates
- `cross-stitch-platform-docs/docs/uploader/HtmlEmailTemplate.txt`
- `cross-stitch-platform-docs/docs/uploader/TextEmailTemplate.txt`
- `cross-stitch-platform-docs/docs/uploader/templates/HtmlEmailTemplate.txt` (duplicate)
- `cross-stitch-platform-docs/docs/uploader/templates/TextEmailTemplate.txt` (duplicate)
- `cross-stitch-platform-docs/plan/uploader/HtmlEmailTemplate.txt` (the one Uploader currently reads from)
- `cross-stitch-platform-docs/plan/uploader/TextEmailTemplate.txt` (same)

### 12.4 Sibling contracts (this folder)
- [`dynamodb-schema.md`](./dynamodb-schema.md) — owns `UnsubscribeToken`, `Unsubscribed`, `Email`, `FirstName`, `cid`, `LastEmailDate`, `LastEmailEntry`.
- [`s3-paths.md`](./s3-paths.md) — `<image_url>` resolution.
- [`pinterest-metadata.md`](./pinterest-metadata.md) — `cid`/`eid` link-tracking origin.
- [`album-id.md`](./album-id.md), [`design-id.md`](./design-id.md) — the IDs encoded into `<pattern_url>` and `<image_url>`.

### 12.5 Architecture / process
- `cross-stitch-platform-docs/docs/cross-stitch/platform-architecture-summary.md:181` — unsubscribe URL contract row.
- `cross-stitch-platform-docs/docs/cross-stitch/platform-architecture-summary.md:203` — SES sender split.
- `cross-stitch-platform-docs/docs/cross-stitch/platform-architecture-summary.md:248` — `UnsubscribeToken` two-writer drift risk.
- `cross-stitch-platform-docs/CLAUDE.md` — do-not-invent rule and `docs/` vs `plan/` split.
- `cross-stitch-platform-docs/plan/integration/CONTRACT-TEMPLATE.md` — the 12-section template followed here.
