# Cross-Stitch Platform — Architecture Summary

## Overview

The Cross-Stitch platform spans four sibling repositories on disk, coordinated by a documentation hub.

- **`cross-stitch-platform-docs/`** — documentation-only hub; no source code, no build, no tests; central knowledge base for the other repos (`d:/ann/Git/cross-stitch-platform-docs`).
- **`cross-stitch/`** — the public-facing Next.js 15 / React 19 / TypeScript web app at `cross-stitch.com` (`d:/ann/Git/cross-stitch`, `d:/ann/Git/cross-stitch/package.json`).
- **`Uploader/`** — a Windows-only WPF desktop tool used by the operator to ingest designs, push S3/DynamoDB content, blast newsletters via SES, manage Pinterest boards/pins, and restart the EB environment (`d:/ann/Git/Uploader`, `d:/ann/Git/Uploader/Uploader/Uploader.csproj`).
- **`AutoPinner/`** — a .NET 8 console worker that pulls designs without a `PinID` from DynamoDB (newest first), creates Pinterest pins for them via the v5 API, and writes the returned pin id back. Runs as `--once` (cron) or `--daemon`. Spec: `cross-stitch-platform-docs/docs/tasks/TASK_AutoPinner.md`. Source: `d:/ann/Git/AutoPinner`.
- Cross-repo coupling happens through shared AWS resources (DynamoDB, S3, Elastic Beanstalk, SES) and a shared Pinterest token file pointed to by `platform-config.json` in the docs hub (`d:/ann/Git/cross-stitch-platform-docs/platform-config.json`).

### Repository roles

| Repo | Role | Build/run | Source of truth for |
|------|------|-----------|---------------------|
| `cross-stitch-platform-docs/` | Planning + reference hub | None — Markdown/text/PDF/Excel only (bootstrap rules in task context) | `platform-config.json`; integration plans under `plan/integration/*`; task specs under `docs/tasks/*` |
| `cross-stitch/` | Customer-facing site, AI Pinterest agent | `npm` / Next.js / Vitest / Playwright (`d:/ann/Git/cross-stitch/package.json`) | DynamoDB schema dump (`d:/ann/Git/cross-stitch/TableDescription.txt`); web routes |
| `Uploader/` | Operator publishing + maintenance UI | `.NET 8 WPF` (`Uploader/Uploader.csproj`, `targetFramework: net8.0-windows`) | Pinterest token writer; `AlbumBoards.csv` (`d:/ann/Git/Uploader/Uploader/AlbumBoards.csv`) |
| `AutoPinner/` | Backfill worker that pins existing designs missing a `PinID` | `.NET 8` console (`AutoPinner/src/AutoPinner/AutoPinner.csproj`, `TargetFramework: net8.0`) | None — read-only of DDB designs, write-only of `PinID`/`PinterestStatus`; mirrors `AlbumBoards.csv` from `Uploader/` |

## cross-stitch web app

### Stack

- Next.js 15.5.7, React 19, TypeScript 5, Tailwind CSS 4, Vitest 4, Playwright 1.58 (`d:/ann/Git/cross-stitch/package.json`).
- Strict TS with `@/* -> ./src/*` path alias (`d:/ann/Git/cross-stitch/tsconfig.json`).
- ESLint flat config extending `next/core-web-vitals` and `next/typescript` (`d:/ann/Git/cross-stitch/eslint.config.mjs`).
- Image `remotePatterns` pin CloudFront origin in `next.config.js` (`d:/ann/Git/cross-stitch/next.config.js`).

### Structure (App Router under `src/`)

- App Router routing — `routing: "app"` per inventory (`d:/ann/Git/cross-stitch/src/`).
- Top-level dirs: `.ebextensions`, `.elasticbeanstalk`, `.lighthouse`, `automation`, `docs`, `plan`, `public`, `saved_configs`, `src`, `tests` (inventory `csCodeMap.topLevelDirs`).
- Data-access layer at `src/lib/data-access.ts` (DynamoDB), `src/lib/users.ts`, `src/lib/password-reset.ts`, `src/lib/design-likes.ts`, `src/lib/subscription-events.ts` (`d:/ann/Git/cross-stitch/src/lib/`).
- URL helpers at `src/lib/url-helper.ts` produce the `/Free-{slug}-Charts.aspx` and `/{Caption}-{AlbumID}-{NPage-1}-Free-Design.aspx` patterns (`d:/ann/Git/cross-stitch/src/lib/url-helper.ts`).
- Site middleware at `src/middleware.ts` resolves base URL (`d:/ann/Git/cross-stitch/src/middleware.ts`).
- Customer-facing routes include `/[slug]`, `/albums`, `/designs/[designId]`, `/unsubscribe`, `/etsy-uploader`, `/api/register-only`, `/api/paypal-webhook`, `/sitemap.xml` (inventory `csCodeMap.integrations` evidence files).
- A separate `automation/pinterest-agent/` Node project lives inside the cross-stitch repo (TS scripts under `automation/pinterest-agent/src/services/` and `automation/pinterest-agent/scripts/`).

### Integrations status (from inventory)

| Integration | Status in cross-stitch | Primary evidence |
|-------------|------------------------|------------------|
| AWS DynamoDB | code-observed | `src/lib/data-access.ts`, `src/lib/users.ts` (`d:/ann/Git/cross-stitch/src/lib/data-access.ts`) |
| AWS SES | code-observed | `src/lib/email-service.ts`, `src/lib/password-reset-email.ts` (`d:/ann/Git/cross-stitch/src/lib/email-service.ts`) |
| AWS CloudFront | code-observed | `next.config.js` remotePatterns (`d:/ann/Git/cross-stitch/next.config.js`) |
| PayPal | code-observed | `src/app/api/paypal-webhook/route.js`, `paypal-monthly-plan-code.txt`, `paypal-yearly-plan-code.txt` (`d:/ann/Git/cross-stitch/src/app/api/paypal-webhook/route.js`) |
| AWS S3 | documented-only (no direct client) | `package.json` deps only (`d:/ann/Git/cross-stitch/package.json`) |
| Etsy | documented-only | `src/app/etsy-uploader/page.tsx`, `etsy_openapi_step_by_step_plan.pdf` (`d:/ann/Git/cross-stitch/src/app/etsy-uploader/page.tsx`) |
| Pinterest (consumer side) | documented-only | `src/app/components/PinterestSaveLink.tsx` (`d:/ann/Git/cross-stitch/src/app/components/PinterestSaveLink.tsx`) |

### Deploy

- **AWS Elastic Beanstalk** is the only deploy target — `.ebextensions/04_options.config`, `.elasticbeanstalk/config.yml` pointing at `us-east-1`, application `cross-stitch-com`, branch-default environment `cross-stitch-com-env-clone` (`d:/ann/Git/cross-stitch/.ebextensions/`, `d:/ann/Git/cross-stitch/.elasticbeanstalk/config.yml:1-3`). Deploy is invoked manually with `eb deploy`.
- Saved EB configs preserved under `saved_configs/` including `eb-configuration-2025-12-12.cfg.yml` (`d:/ann/Git/cross-stitch/saved_configs/eb-configuration-2025-12-12.cfg.yml`).

### CI

- No CI is configured. A `Jenkinsfile` previously existed in the repo root (Checkout → `npm install` → build → test) but was never wired to a Jenkins instance and was removed; its only commit was the original `Add Jenkinsfile`. Tests and builds are run locally via `npm run test` / `npm run build`.

### Adjacent (not in the deploy path)

- **Lighthouse** — `.lighthouse/` is a directory of saved performance trace JSON snapshots (e.g. `baby-giraffe-*.json`); there is no `.lighthouserc.js`, no scheduled job, and no reference to the directory from any deploy artifact (`d:/ann/Git/cross-stitch/.lighthouse/`). Ad-hoc performance-investigation output, not tooling.

## Uploader WPF app

### Target framework and shape

- **Target framework**: `net8.0-windows` (`d:/ann/Git/Uploader/Uploader/Uploader.csproj`).
- **Entry point**: `Uploader/MainWindow.xaml.cs` — single WPF window with operator buttons calling helpers (`d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs`).
- **Companion automation script**: `Uploader/scripts/test-pinterest-ads.ts` (TypeScript Pinterest Ads API smoke test, run outside the WPF process) (`d:/ann/Git/Uploader/scripts/test-pinterest-ads.ts`). A near-duplicate previously lived at `Uploader/automation/pinterest-agent/scripts/test-pinterest-ads.ts`; it was removed because nothing in the WPF app referenced it and the canonical pinterest-agent project lives in the cross-stitch repo (see Deployment → pinterest-agent automation).

### Helpers/* purposes (from `uploaderCodeMap.helpers`)

| File | Purpose |
|------|---------|
| `Uploader/Helpers/S3Helper.cs` | Upload/delete files (PDF, photos) to S3 via `AmazonS3Client` (`d:/ann/Git/Uploader/Uploader/Helpers/S3Helper.cs`) |
| `Uploader/Helpers/EmailHelper.cs` | Send via AWS SES `SendEmail` / `SendRawEmail` with optional config-set headers (`d:/ann/Git/Uploader/Uploader/Helpers/EmailHelper.cs`) |
| `Uploader/Helpers/PinterestTokenInfo.cs` | Pinterest OAuth model + JSON token store + OAuth client (auth-code exchange, refresh, `GetValidAccessToken`) (`d:/ann/Git/Uploader/Uploader/Helpers/PinterestTokenInfo.cs`) |
| `Uploader/Helpers/PinterestHelper.cs` | Create Pinterest pins via API v5 with theme detection, SEO text, hashtags; resolves board by AlbumID from CSV (`d:/ann/Git/Uploader/Uploader/Helpers/PinterestHelper.cs`) |
| `Uploader/Helpers/PinterestBoardCreator.cs` | Create boards from DynamoDB album records; writes `AlbumID,Caption,BoardID` CSV (`d:/ann/Git/Uploader/Uploader/Helpers/PinterestBoardCreator.cs`) |
| `Uploader/Helpers/PinterestBoardRenamer.cs` | PATCH boards on Pinterest to SEO-friendlier names from `AlbumBoards.csv` (`d:/ann/Git/Uploader/Uploader/Helpers/PinterestBoardRenamer.cs`) |
| `Uploader/Helpers/PatternLinkHelper.cs` | Build site URLs and S3 image URLs for emails + Pinterest flows (`d:/ann/Git/Uploader/Uploader/Helpers/PatternLinkHelper.cs`) |
| `Uploader/Helpers/PlatformConfig.cs` | Resolve the Pinterest token store path from `cross-stitch-platform-docs/platform-config.json` (with `PLATFORM_CONFIG_PATH` env override) (`d:/ann/Git/Uploader/Uploader/Helpers/PlatformConfig.cs`) |
| `Uploader/Helpers/ElasticBeanstalkHelper.cs` | Restart the configured EB application environment (`d:/ann/Git/Uploader/Uploader/Helpers/ElasticBeanstalkHelper.cs`) |
| `Uploader/Helpers/EC2Helper.cs` | Reboot EC2 instances by `Name` tag; check health (`d:/ann/Git/Uploader/Uploader/Helpers/EC2Helper.cs`) |
| `Uploader/Helpers/EtsyHelper.cs` | Stub for Etsy v3 digital-listing upload — placeholders, **not wired** (`d:/ann/Git/Uploader/Uploader/Helpers/EtsyHelper.cs`) |

### Integrations (Uploader-side)

- Pinterest (active): `PinterestTokenInfo.cs`, `PinterestHelper.cs`, `PinterestBoardCreator.cs`, `PinterestBoardRenamer.cs`, `App.config`, `App.private.config.example`.
- AWS S3, DynamoDB, SES, EC2, Elastic Beanstalk — all referenced from `Uploader.csproj` + `App.config` (`d:/ann/Git/Uploader/Uploader/Uploader.csproj`, `d:/ann/Git/Uploader/Uploader/App.config`).
- iTextSharp (PDF), log4net (logging), Newtonsoft.Json — listed in `Uploader.csproj` (`d:/ann/Git/Uploader/Uploader/Uploader.csproj`).
- Etsy: **stub-only, not active** (`d:/ann/Git/Uploader/Uploader/Helpers/EtsyHelper.cs`).

### Secrets layout

- `Uploader/App.private.config` — gitignored runtime secrets: `SenderEmail`, `AdminEmail`, `PinterestClientSecret`, `PinterestAccessToken`, `PinterestTokenStorePath`, `UnsubscribeSecret`, `SesConfigurationSetName` (`d:/ann/Git/Uploader/Uploader/App.private.config`, template at `d:/ann/Git/Uploader/Uploader/App.private.config.example`).
- `Uploader/secrets/pinterest_tokens.json` — Pinterest OAuth token store at repo root; path is resolved through `cross-stitch-platform-docs/platform-config.json -> pinterestTokenPath` (`d:/ann/Git/Uploader/secrets/pinterest_tokens.json`).
- *(Removed)* `Uploader/automation/pinterest-agent/` was deleted — it held only an `.env` with one ad-account ID and a near-duplicate of `Uploader/scripts/test-pinterest-ads.ts`. The real pinterest-agent project lives in the cross-stitch repo at `d:/ann/Git/cross-stitch/automation/pinterest-agent/`; the WPF app never referenced the Uploader-side copy.

### `MainWindow.xaml.cs` highlights (from inventory)

- `BtnUpload_Click` at line 160 → `RunFullUploadFlowAsync` at line 604: end-to-end design upload pipeline (chart → S3, PDF → S3, image → S3, then DynamoDB insert) (`d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:160`, `:604`).
- `BtnSendEmails_Click` line 194 / `BtnSendTextEmails_Click` line 241 → `SendNotificationEmailsAsync` / `SendTextOnlyEmailsAsync`; progress variants at lines 2534 and 2429 (`d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:194`, `:241`, `:2534`, `:2429`).
- `BtnPinterestReAuth_Click` line 288 — runs the Pinterest OAuth re-authorization flow via `PinterestOAuthClient` (`d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:288`).
- `BtnTestPinterest_Click` line 366, `BtnCreateBoards_Click` line 382, `BtnRenameBoards_Click` line 411 — Pinterest test/create/rename actions (`d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:366`, `:382`, `:411`).
- User-maintenance buttons: `InitializeUserUnsubscribe_Click`, `InitializeUserSubscriptions_Click`, `InitializeUserCid_Click`, `InitializeItemsUserCid_Click`, `MarkUsersVerified_Click`, `RemoveSuppressedUsers_Click` — DynamoDB user-record migrations (`d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs`).
- S3 helpers: `UploadChartToS3Async` line 900, `UploadPdfToS3Async` line 916, `UploadImageToS3Async` line 1001, `UploadPhotoFileAsync` line 1011 (`d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:900`, `:916`, `:1001`, `:1011`).
- `InsertItemIntoDynamoDbAsync` line 1027 — writes design metadata to DynamoDB after S3 upload (`d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:1027`).
- Unsubscribe URL builders: `BuildUnsubscribeUrl` line 2276, `BuildUnsubscribeUrlFromStoredToken` line 2286 — uses HMAC secret from `App.private.config` (`d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:2276`, `:2286`).
- Admin previews/notifications: `SendNotificationMailToAdminAsync` line 1198, `SendAdminUserStyleEmailAsync` line 2048 (`d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:1198`, `:2048`).
- Pinterest image handling: `GetAndShowImage` line 3285, `SavePinterestImage` line 3337 (`d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:3285`, `:3337`).

## cross-stitch-platform-docs hub

### `plan/` vs `docs/` discipline

- The split is load-bearing: `plan/` holds **planning / roadmap / milestone / operational-strategy** docs only; `docs/` holds **templates / helper notes / reference / automation / build-output documentation** (task `bootstrap.rules`).
- "Do not put non-planning files under plan/" — a previous restructure existed specifically to enforce this (task `bootstrap.rules`).
- Both directories use a parallel topic-based subfolder layout (`cross-stitch/`, `uploader/`, `automation/`, plus topical folders under `plan/` like `aws/`, `etsy/`, `paypal/`, `integration-*/`) (task `bootstrap.planSubfolders`, `bootstrap.docsSubfolders`).

### Source-of-truth rule

- "The documentation directories are considered the source of truth unless explicitly overridden by newer instructions" (task `bootstrap.sourceOfTruthStatement`).
- Concrete source-of-truth artifact: `platform-config.json` at the docs-hub root, holding `pinterestTokenPath = 'Uploader/secrets/pinterest_tokens.json'` consumed by both sibling repos (`d:/ann/Git/cross-stitch-platform-docs/platform-config.json`).

### What is actually populated vs placeholder

Plan-side inventory (`planInventory.totalFiles = 42`, but most subfolders are empty placeholders):

| `plan/` subfolder | Populated? | Notes |
|---|---|---|
| `cross-stitch/` | **populated** | Pinterest AI Agent series + migration/operational stubs (e.g. `Pinterest AI Agent — API Integrations.md`, `— AI Reasoning.md`, `— AWS Deployment.md`, `— Design-Level Intelligence.md`, `— Memory and Trend Analysis.md`, `— Milestones and Roadmap.md`, `— Practical Setup Notes.md`, `— WPF Uploader Integration.md`, `— Documentation Index.md`) (`d:/ann/Git/cross-stitch-platform-docs/plan/cross-stitch/`) |
| `uploader/` | partial | `README.md`, `Task.txt` (engagement-tracking proposal), `App.txt`, `TestTemplate.json`, plus redirect notices for moved email templates (`d:/ann/Git/cross-stitch-platform-docs/plan/uploader/`) |
| `integration/` | **populated** | `CONTRACT-TEMPLATE.md`, `INTEGRATION-MAP.md` (Mermaid diagram), `ARCHITECTURE-SUMMARY.md`, `README.md` (`d:/ann/Git/cross-stitch-platform-docs/plan/integration/`) |
| `integration-aws/` | proposal only | `README.md` scoping doc only (`d:/ann/Git/cross-stitch-platform-docs/plan/integration-aws/README.md`) |
| `integration-pinterest/` | proposal only | `README.md` scoping doc only (`d:/ann/Git/cross-stitch-platform-docs/plan/integration-pinterest/README.md`) |
| `integration-wpf/` | proposal only | `README.md` scoping doc only (`d:/ann/Git/cross-stitch-platform-docs/plan/integration-wpf/README.md`) |
| `ai`, `api`, `aws`, `backlink`, `code`, `design`, `docs`, `etsy`, `integration-etsy`, `memory`, `milestones`, `misc`, `paypal`, `pdf`, `roadmap`, `setup`, `subscription`, `test`, `trend`, `vscode`, `wpf` | **empty placeholders** | All `files: []` per inventory (task `planInventory.subfolders`) |

Docs-side inventory (`docsInventory.totalFiles = 12`):

| `docs/` subfolder | Populated? | Notes |
|---|---|---|
| `automation/` | placeholder only | `README.md` is a single line saying the folder is for automation scripts/docs (`d:/ann/Git/cross-stitch-platform-docs/docs/automation/README.md:1`) |
| `cross-stitch/` | placeholder only | `README.md` only — no schema fields or contracts (`d:/ann/Git/cross-stitch-platform-docs/docs/cross-stitch/README.md:1`) |
| `uploader/` | partial | Real email templates (`HtmlEmailTemplate.txt`, `TextEmailTemplate.txt` both at root and under `templates/`) for the "Running horse" freebie; README placeholders for `helpers/`, `images/`, `bin/`, `obj/` (`d:/ann/Git/cross-stitch-platform-docs/docs/uploader/`) |

## Cross-repo contracts

All 19 contracts from `crossRepoContracts.contracts`, grouped by type and tagged with status.

### File-path contracts

| Contract | Status | Primary evidence |
|---|---|---|
| Pinterest token file path (`Uploader/secrets/pinterest_tokens.json`, resolved via `platform-config.json`) | both-observed | `d:/ann/Git/Uploader/Uploader/Helpers/PlatformConfig.cs`, `d:/ann/Git/cross-stitch/automation/pinterest-agent/src/services/readPlatformConfig.ts`, `d:/ann/Git/cross-stitch-platform-docs/platform-config.json` |
| Email templates at `plan/uploader/HtmlEmailTemplate.txt` and `TextEmailTemplate.txt` (Uploader hardcodes absolute paths) | one-sided (Uploader → docs) | `d:/ann/Git/Uploader/Uploader/App.config`, `d:/ann/Git/cross-stitch-platform-docs/plan/uploader/HtmlEmailTemplate.txt`, `…/TextEmailTemplate.txt` |
| Pinterest Ads API smoke test script (two copies after cleanup, not a real contract) | both-observed (duplicated) | `d:/ann/Git/Uploader/scripts/test-pinterest-ads.ts`, `d:/ann/Git/cross-stitch/automation/pinterest-agent/scripts/test-pinterest-ads.ts` *(third copy at `Uploader/automation/pinterest-agent/scripts/test-pinterest-ads.ts` removed)* |

### Schema contracts

| Contract | Status | Primary evidence |
|---|---|---|
| Pinterest token file JSON schema (`access_token` required, `refresh_token?/scope?/token_type?/expires_at_utc?` optional on read) | both-observed (implicit; no shared schema file) | `d:/ann/Git/Uploader/Uploader/Helpers/PinterestTokenInfo.cs`, `d:/ann/Git/cross-stitch/automation/pinterest-agent/src/services/readPinterestToken.ts` |
| DynamoDB table **`CrossStitchItems`** — single-table (PK=ID, SK=NPage) holding `EntityType=DESIGN` / `ALBUM`, plus `USR#`-prefixed users; Uploader writes, cross-stitch reads | both-observed (schema lives only in `cross-stitch/TableDescription.txt`; **no docs-hub schema**) | `d:/ann/Git/cross-stitch/TableDescription.txt`, `d:/ann/Git/cross-stitch/src/lib/data-access.ts`, `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs`, `d:/ann/Git/Uploader/Uploader/Helpers/PinterestBoardCreator.cs` |
| DynamoDB table **`CrossStitchUsers`** (secondary users table; PK `USR#<email>`; shared attrs `Email`, `UnsubscribeToken`, `cid`) | both-observed | `d:/ann/Git/cross-stitch/src/lib/users.ts`, `d:/ann/Git/cross-stitch/src/lib/password-reset.ts`, `d:/ann/Git/Uploader/Uploader/App.config`, `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs` |
| Pinterest pin-ID DynamoDB attribute name drift — Uploader writes `PinID`; cross-stitch probes six historical variants (`PinterestPinId/PinterestPinID/PinterestID/PinterestId/PinID/PinId`) | both-observed (defensive read, not a true contract) | `d:/ann/Git/cross-stitch/automation/pinterest-agent/scripts/export-design-pin-map.ts`, `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs`, `d:/ann/Git/Uploader/Uploader/PatternInfo.cs` |
| DynamoDB attribute `cid` (lowercase user correlation GUID) — generated by cross-stitch on registration, back-filled by Uploader | both-observed | `d:/ann/Git/cross-stitch/src/lib/users.ts`, `d:/ann/Git/cross-stitch/src/lib/data-access.ts`, `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs` |

### Env-var contracts

| Contract | Status | Primary evidence |
|---|---|---|
| `PLATFORM_CONFIG_PATH` env override — consumed by both `PlatformConfig` loaders | both-observed (undocumented in docs hub) | `d:/ann/Git/Uploader/Uploader/Helpers/PlatformConfig.cs`, `d:/ann/Git/cross-stitch/automation/pinterest-agent/src/services/readPlatformConfig.ts` |

### API contracts

| Contract | Status | Primary evidence |
|---|---|---|
| `UnsubscribeToken` attribute + URL `https://cross-stitch.com/unsubscribe?token=` — cross-stitch generates `randomUUID` on user create; Uploader embeds the same token in outbound emails | both-observed | `d:/ann/Git/cross-stitch/src/lib/users.ts`, `d:/ann/Git/cross-stitch/src/app/unsubscribe/page.tsx`, `d:/ann/Git/Uploader/Uploader/App.config`, `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs` |

### Naming-convention contracts

| Contract | Status | Primary evidence |
|---|---|---|
| S3 bucket `cross-stitch-designs` — Uploader hardcodes it as fallback; cross-stitch consumes via CloudFront and never names this bucket | **one-sided** (Uploader-only) | `d:/ann/Git/Uploader/Uploader/App.config`, `d:/ann/Git/Uploader/Uploader/Helpers/PatternLinkHelper.cs`, `d:/ann/Git/Uploader/Uploader/Helpers/S3Helper.cs` |
| S3 photo path `photos/{AlbumID}/{DesignID}/<file>.jpg` (e.g. `4.jpg`) — Uploader writes, cross-stitch reads via `https://d2o1uvvg91z7o4.cloudfront.net/photos/{AlbumID}/{DesignID}/4.jpg` | both-observed (duplicated string templates, no shared constant) | `d:/ann/Git/Uploader/Uploader/Helpers/PatternLinkHelper.cs`, `d:/ann/Git/cross-stitch/src/lib/data-access.ts`, `d:/ann/Git/cross-stitch/src/app/components/RegisterForm.tsx`, `d:/ann/Git/cross-stitch/next.config.js` |
| S3 PDF path `pdfs/{AlbumID}/Stitch{DesignID}_Kit.pdf` and `pdfs/{AlbumID}/{DesignID}/Stitch{DesignID}_{N}_Kit.pdf` | both-observed (duplicated, magic naming) | `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs`, `d:/ann/Git/cross-stitch/src/lib/data-access.ts`, `d:/ann/Git/cross-stitch/src/app/components/DownloadPdfLink.tsx` |
| Design page URL `/{Caption}-{AlbumID}-{NPage-1}-Free-Design.aspx` | both-observed (**triplicated** — `PatternLinkHelper.BuildPatternUrl` in C#, `url-helper.ts CreateDesignUrl`, plus an inline `buildDesignUrl` in `export-design-pin-map.ts` whose comment says "Mirrors CreateDesignUrl") | `d:/ann/Git/Uploader/Uploader/Helpers/PatternLinkHelper.cs`, `d:/ann/Git/cross-stitch/src/lib/url-helper.ts`, `d:/ann/Git/cross-stitch/src/app/[slug]/page.tsx`, `d:/ann/Git/cross-stitch/automation/pinterest-agent/scripts/export-design-pin-map.ts`, `d:/ann/Git/cross-stitch-platform-docs/plan/cross-stitch/Pinterest AI Agent — WPF Uploader Integration.md` |
| Album page URL `/Free-{CaptionSlug}-Charts.aspx` — Uploader supports override via `AlbumUrlTemplate` app setting | both-observed | `d:/ann/Git/Uploader/Uploader/Helpers/PatternLinkHelper.cs`, `d:/ann/Git/cross-stitch/src/lib/url-helper.ts`, `d:/ann/Git/cross-stitch/src/app/albums/page.tsx` |
| AlbumID 4-digit zero-padded (`albumId.ToString("D4")`) — Uploader-internal CSV key only | **one-sided** (Uploader-only) | `d:/ann/Git/Uploader/Uploader/Helpers/PinterestHelper.cs`, `d:/ann/Git/Uploader/Uploader/AlbumBoards.csv` |
| Site base URL `https://cross-stitch.com` — Uploader `SiteBaseUrl`/`PinterestLinkUrl` vs cross-stitch `normalizeBaseUrl`/`getSiteBaseUrl` (default `NEXT_PUBLIC_SITE_URL`) | both-observed | `d:/ann/Git/Uploader/Uploader/App.config`, `d:/ann/Git/Uploader/Uploader/Templates/HtmlEmailTemplate.txt`, `d:/ann/Git/cross-stitch/src/lib/url-helper.ts`, `d:/ann/Git/cross-stitch/src/middleware.ts` |
| Elastic Beanstalk env name — `cross-stitch-com-env-clone` consistent across `.elasticbeanstalk/config.yml`, Uploader's `App.config:17`, `MainWindow.xaml.cs:57` default, and `ENVIRONMENT-SETUP.md:12` (drift resolved in commits `342e31e`, `a979ab7`, `140e389`) | both-observed (resolved) | `d:/ann/Git/cross-stitch/.elasticbeanstalk/config.yml:1-7`, `d:/ann/Git/Uploader/Uploader/App.config:17`, `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:57`, `d:/ann/Git/cross-stitch/ENVIRONMENT-SETUP.md:12` |
| Pinterest OAuth scope set `pins:read pins:write boards:read boards:write ads:read` | both-observed | `d:/ann/Git/Uploader/Uploader/App.config`, `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs`, `d:/ann/Git/cross-stitch/automation/pinterest-agent/scripts/test-pinterest-api.ts`, `d:/ann/Git/cross-stitch-platform-docs/plan/cross-stitch/Pinterest AI Agent — API Integrations.md` |

## External integrations

| Provider | Used by | Where in code | Where in docs |
|---|---|---|---|
| **AWS S3** | Uploader (writes), cross-stitch (reads via CDN — documented-only in package.json) | `d:/ann/Git/Uploader/Uploader/Helpers/S3Helper.cs`; cross-stitch package deps only (`d:/ann/Git/cross-stitch/package.json`) | `d:/ann/Git/cross-stitch-platform-docs/plan/integration-aws/README.md` (proposal) |
| **AWS DynamoDB** | Uploader (writes designs/users), cross-stitch (reads + writes users/likes/subs) | `d:/ann/Git/Uploader/Uploader/Helpers/PinterestBoardCreator.cs`, `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs`; `d:/ann/Git/cross-stitch/src/lib/data-access.ts`, `d:/ann/Git/cross-stitch/src/lib/users.ts`, `…/design-likes.ts`, `…/subscription-events.ts`, `…/password-reset.ts` | Schema in `d:/ann/Git/cross-stitch/TableDescription.txt` only; **no doc-hub schema** |
| **AWS SES** | Uploader (newsletter blasts), cross-stitch (transactional + password reset) | `d:/ann/Git/Uploader/Uploader/Helpers/EmailHelper.cs`; `d:/ann/Git/cross-stitch/src/lib/email-service.ts`, `d:/ann/Git/cross-stitch/src/lib/password-reset-email.ts` | `d:/ann/Git/cross-stitch-platform-docs/plan/cross-stitch/Pinterest AI Agent — AWS Deployment.md` (aspirational) |
| **AWS EC2** | Uploader only (reboot by Name tag) | `d:/ann/Git/Uploader/Uploader/Helpers/EC2Helper.cs` | Not in docs hub (not-observed) |
| **AWS Elastic Beanstalk** | Uploader ("Restart Elastic Beanstalk" button in More actions panel, `BtnRestartEb_Click`; also called automatically as step 5 of `RunFullUploadFlowAsync`); cross-stitch (deployment target — live env `cross-stitch-com-env-clone`) | `d:/ann/Git/Uploader/Uploader/MainWindow.xaml`, `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:367` (BtnRestartEb_Click), `d:/ann/Git/Uploader/Uploader/Helpers/ElasticBeanstalkHelper.cs`; `d:/ann/Git/cross-stitch/.ebextensions/`, `d:/ann/Git/cross-stitch/.elasticbeanstalk/config.yml` | `d:/ann/Git/cross-stitch/ENVIRONMENT-SETUP.md` |
| **AWS CloudFront** | cross-stitch (origin `d2o1uvvg91z7o4.cloudfront.net`) | `d:/ann/Git/cross-stitch/next.config.js` | not-observed in docs hub |
| **Etsy** | Uploader stub (not active); cross-stitch documented-only | `d:/ann/Git/Uploader/Uploader/Helpers/EtsyHelper.cs` (stub); `d:/ann/Git/cross-stitch/src/app/etsy-uploader/page.tsx` | `d:/ann/Git/cross-stitch/etsy_openapi_step_by_step_plan.pdf`; `plan/etsy/` empty, `plan/integration-etsy/` empty |
| **Pinterest (REST API v5 + OAuth)** | Uploader (active publisher), cross-stitch pinterest-agent (analytics, Ads API), cross-stitch site (PinterestSaveLink — documented-only) | `d:/ann/Git/Uploader/Uploader/Helpers/PinterestHelper.cs`, `…/PinterestBoardCreator.cs`, `…/PinterestBoardRenamer.cs`, `…/PinterestTokenInfo.cs`; `d:/ann/Git/cross-stitch/automation/pinterest-agent/src/services/pinterestClient.ts`, `…/scripts/test-pinterest-api.ts`, `…/scripts/test-pinterest-ads.ts`, `…/scripts/export-design-pin-map.ts`; `d:/ann/Git/cross-stitch/src/app/components/PinterestSaveLink.tsx` | `d:/ann/Git/cross-stitch-platform-docs/plan/cross-stitch/Pinterest AI Agent — API Integrations.md`, `…/— AI Reasoning.md`, `…/— Design-Level Intelligence.md`, `…/— WPF Uploader Integration.md`, `…/— Milestones and Roadmap.md`; `plan/integration-pinterest/README.md` (proposal) |
| **PayPal** | cross-stitch only | `d:/ann/Git/cross-stitch/src/app/api/paypal-webhook/route.js`, `…/components/AuthControl.tsx`, `…/components/RegisterForm.tsx`, `…/components/DownloadPdfLink.tsx`, `d:/ann/Git/cross-stitch/src/lib/subscription-events.ts`, plan-code files in repo root | `plan/paypal/` empty placeholder in docs hub; redirect stubs `plan/cross-stitch/paypal-monthly-plan-code.txt` and `…-yearly-plan-code.txt` |
| **AI / Anthropic API** | cross-stitch pinterest-agent (documented usage); no in-code reference captured in inventory beyond docs | not-observed in `csCodeMap` integrations; planning only | `d:/ann/Git/cross-stitch-platform-docs/plan/cross-stitch/Pinterest AI Agent — AI Reasoning.md`, `…/— API Integrations.md` |
| **Google OAuth / GA4 / AdSense** | cross-stitch pinterest-agent (documented) | not in code map (documented-only) | `d:/ann/Git/cross-stitch-platform-docs/plan/cross-stitch/Pinterest AI Agent — API Integrations.md` |

## Deployment

### cross-stitch web app

- **Deploy path**: production runs on **AWS Elastic Beanstalk**, `us-east-1`, application `cross-stitch-com`, branch-default environment **`cross-stitch-com-env-clone`** (`d:/ann/Git/cross-stitch/.elasticbeanstalk/config.yml:1-7`). The env name is consistent across `App.config:17`, `MainWindow.xaml.cs:57` default, and `ENVIRONMENT-SETUP.md:12`. Deploy is invoked manually with `eb deploy`; saved EB configs are preserved under `saved_configs/` (`d:/ann/Git/cross-stitch/saved_configs/eb-configuration-2025-12-12.cfg.yml`).
- **Restart control plane from Uploader**: the WPF tool has a dedicated "Restart Elastic Beanstalk" button under *More actions* (`BtnRestartEb_Click` at `MainWindow.xaml.cs:367`) that invokes `_elasticBeanstalkHelper.RestartEnvironmentAsync`. The same call also runs automatically as step 5 of `RunFullUploadFlowAsync` after every successful upload.
- EB option configs under `.ebextensions/` (e.g. `04_options.config`) carry DynamoDB table names (`d:/ann/Git/cross-stitch/.ebextensions/04_options.config`).
- **No CI**: tests and builds are run locally with `npm run test` / `npm run build`. A stale `Jenkinsfile` was previously in the repo root but was removed (never wired to a Jenkins instance).
- **Not in the deploy loop**:
  - `.lighthouse/` holds saved performance trace JSON snapshots from manual runs; no `.lighthouserc.js`, no scheduled job, not referenced by any deploy artifact (`d:/ann/Git/cross-stitch/.lighthouse/`).

### Uploader

- **Local-only WPF tool** — `net8.0-windows`, no remote deployment; the operator runs it on a Windows machine (`d:/ann/Git/Uploader/Uploader/Uploader.csproj`).
- Secrets live next to the binary (`App.private.config`, `secrets/pinterest_tokens.json`); no CI/CD pipeline observed in inventory (not-observed).

### pinterest-agent automation

- **Today**: runs locally via Windows Task Scheduler invoking `daily-run.bat`, reads `.env` for credentials (`d:/ann/Git/cross-stitch-platform-docs/plan/cross-stitch/Pinterest AI Agent — AWS Deployment.md`, `…/— Milestones and Roadmap.md`).
- **Planned migration**: EventBridge → Lambda → Pinterest/GA4/AdSense APIs → AI → DynamoDB/S3 → SES, with credentials in Secrets Manager. Status **aspirational** (`d:/ann/Git/cross-stitch-platform-docs/plan/cross-stitch/Pinterest AI Agent — AWS Deployment.md`).
- Per-design analytics V1 is **done** and wired into the daily cron (`d:/ann/Git/cross-stitch-platform-docs/plan/cross-stitch/Pinterest AI Agent — Design-Level Intelligence.md`).

## Open questions and undocumented assumptions

### `syncRisks` from the cross-repo-contracts inventory

- **Pinterest token JSON schema is implicit.** If Uploader (Newtonsoft model in `PinterestTokenInfo.cs`) renames or drops a field (e.g. `expires_at_utc`), cross-stitch's `readPinterestToken.ts` silently produces `undefined`; only `access_token` is hard-required. No validation, no shared schema file (`d:/ann/Git/Uploader/Uploader/Helpers/PinterestTokenInfo.cs`, `d:/ann/Git/cross-stitch/automation/pinterest-agent/src/services/readPinterestToken.ts`).
- **`cross-stitch/.gitignore` is stale** — line 63 still lists the legacy path `automation/pinterest-agent/pinterest_tokens.json` after commit `ea360ad` moved the token to `Uploader/secrets/pinterest_tokens.json`. A token accidentally committed at the new location would not be ignored by cross-stitch's repo (`d:/ann/Git/cross-stitch/.gitignore:63`).
- **Design URL pattern is triplicated** — `PatternLinkHelper.BuildPatternUrl` (C#), `url-helper.ts CreateDesignUrl` (TS), and `buildDesignUrl` in `export-design-pin-map.ts` (whose comment literally says "Mirrors CreateDesignUrl"). A change in one will silently break Pinterest pin.link and analytics joins (`d:/ann/Git/Uploader/Uploader/Helpers/PatternLinkHelper.cs`, `d:/ann/Git/cross-stitch/src/lib/url-helper.ts`, `d:/ann/Git/cross-stitch/automation/pinterest-agent/scripts/export-design-pin-map.ts`).
- **S3 path conventions are undocumented** — `photos/{AlbumID}/{DesignID}/4.jpg` and `pdfs/{AlbumID}/Stitch{DesignID}_Kit.pdf` live as string templates in both repos with no shared constant and no docs-hub file, despite the bootstrap "do-not-invent" list explicitly flagging S3 path structures (task `bootstrap.doNotInvent`; `d:/ann/Git/Uploader/Uploader/Helpers/PatternLinkHelper.cs`, `d:/ann/Git/cross-stitch/src/lib/data-access.ts`).
- **DynamoDB schema lives only in `cross-stitch/TableDescription.txt`** (a raw `describe-table` dump). The docs hub has no DynamoDB schema document despite the do-not-invent rule (`d:/ann/Git/cross-stitch/TableDescription.txt`; task `bootstrap.doNotInvent`).
- **Pin-ID attribute name drift** — Uploader writes `PinID`, cross-stitch defensively reads six variants. Indicates past schema churn without a written contract (`d:/ann/Git/cross-stitch/automation/pinterest-agent/scripts/export-design-pin-map.ts`).
- **EB env rename to `cross-stitch-com-env-clone` has propagated** to `App.config`, `MainWindow.xaml.cs` default, and `ENVIRONMENT-SETUP.md` (resolved in commits `342e31e`, `a979ab7`, `140e389`). No outstanding drift in EB naming.
- **Hardcoded absolute Windows paths in `App.config`** point to `D:\ann\Git\cross-stitch-platform-docs\plan\uploader\*.txt`. Any operator with a different checkout layout (or CI) silently falls back to bundled `Templates/*.txt` defaults (`d:/ann/Git/Uploader/Uploader/App.config`).
- **`AlbumBoards.csv` lives only inside Uploader.** cross-stitch and the docs hub have no copy, so the AlbumID→Pinterest BoardID mapping cannot be cross-validated (`d:/ann/Git/Uploader/Uploader/AlbumBoards.csv`).
- **`UnsubscribeToken` has two writers** — cross-stitch on user create, Uploader on back-fill — across both `CrossStitchItems` and `CrossStitchUsers`. No documented invariant; easy to clobber (`d:/ann/Git/cross-stitch/src/lib/users.ts`, `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs`).
- **No HTTP API contract Uploader ↔ cross-stitch.** All cross-repo coupling is via shared AWS resources and the shared Pinterest token file. The "Example Future API Contract" in `plan/cross-stitch/Pinterest AI Agent — WPF Uploader Integration.md` is aspirational only (`d:/ann/Git/cross-stitch-platform-docs/plan/cross-stitch/Pinterest AI Agent — WPF Uploader Integration.md`).

### `CLAUDE.md` "do-not-invent" items that ARE in code but NOT in docs

Items the bootstrap rules name as forbidden assumption areas (task `bootstrap.doNotInvent`) and their observed-in-code-but-not-documented status:

| Forbidden assumption area | Observed in code? | Documented in docs hub? |
|---|---|---|
| **DynamoDB schemas** | yes — single-table `CrossStitchItems` (PK=ID, SK=NPage; `EntityType=DESIGN/ALBUM/USR#…`) and `CrossStitchUsers` (`d:/ann/Git/cross-stitch/TableDescription.txt`, `d:/ann/Git/cross-stitch/src/lib/data-access.ts`, `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs`) | **no** — no schema doc in `cross-stitch-platform-docs/` |
| **AlbumID/DesignID formats** | partially — Uploader uses 4-digit zero-pad `D4` for the Pinterest board CSV key; cross-stitch uses raw numeric AlbumID (`d:/ann/Git/Uploader/Uploader/Helpers/PinterestHelper.cs`) | **no** — not documented as a contract |
| **S3 path structures** | yes — `photos/{AlbumID}/{DesignID}/4.jpg`, `pdfs/{AlbumID}/Stitch{DesignID}_Kit.pdf`, `pdfs/{AlbumID}/{DesignID}/Stitch{DesignID}_{N}_Kit.pdf` (`d:/ann/Git/Uploader/Uploader/Helpers/PatternLinkHelper.cs`, `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs`, `d:/ann/Git/cross-stitch/src/lib/data-access.ts`) | **no** — only listed as a do-not-invent area |
| **PDF generation conventions** | partially — `Stitch{DesignID}_Kit.pdf` and `Stitch{DesignID}_{1\|3\|5}_Kit.pdf` size variants in Uploader; iTextSharp is the engine (`d:/ann/Git/Uploader/Uploader/Uploader.csproj`, `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs`) | **no** — `plan/pdf/` is an empty placeholder |
| **Uploader ↔ website contracts** | yes — implicit via shared DynamoDB attributes (`cid`, `UnsubscribeToken`, `PinID`) and shared S3/URL paths | partially — `plan/integration/ARCHITECTURE-SUMMARY.md` exists; no per-attribute schema |
| **Pinterest metadata conventions** | yes — `AlbumBoards.csv` (AlbumID→BoardID), `PinID` attribute, scope set, "From album {0:D4}" description prefix (`d:/ann/Git/Uploader/Uploader/Helpers/PinterestHelper.cs`, `d:/ann/Git/Uploader/Uploader/AlbumBoards.csv`) | partially — Pinterest AI Agent series describes scopes and pin URLs by example; no metadata-field spec |
| **AWS deployment assumptions** | yes — bucket name `cross-stitch-designs`, sitemap bucket `cross-stitch-sitemap-cache`, CloudFront `d2o1uvvg91z7o4.cloudfront.net`, EB env `cross-stitch-com-env-clone` (consistent across all 4 sources), EB app `cross-stitch-com`, region `us-east-1` (`d:/ann/Git/Uploader/Uploader/App.config`, `d:/ann/Git/cross-stitch/.elasticbeanstalk/config.yml`, `d:/ann/Git/cross-stitch/next.config.js`) | partially — `plan/integration-aws/README.md` is a proposal-only README |

### Other open items

- The `plan/` tree contains 27 subfolders but 21 of them are **empty placeholders** (task `planInventory.subfolders`). The intent (per `plan/README.md`) is wider than the populated documentation.
- The docs-side `uploader/helpers/` folder is explicitly meant to hold copies of Uploader `.cs` files for reference but is currently empty aside from its README (`d:/ann/Git/cross-stitch-platform-docs/docs/uploader/helpers/README.md`).
- `docs/uploader/HtmlEmailTemplate.txt` is duplicated at `docs/uploader/templates/HtmlEmailTemplate.txt` (same for the text variant) — two copies of the "Running horse" freebie template with no canonical pointer (`d:/ann/Git/cross-stitch-platform-docs/docs/uploader/HtmlEmailTemplate.txt`, `d:/ann/Git/cross-stitch-platform-docs/docs/uploader/templates/HtmlEmailTemplate.txt`).
