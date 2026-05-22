# DynamoDB schema — CrossStitchBusinessHistory

## 1. Contract Name

**DynamoDB schema — `CrossStitchBusinessHistory`** (single-table store for the Pinterest AI Agent's historical memory: daily business metrics, AI analyses, design-level analytics, and anomaly events).

This table is the persistence layer planned in Milestone 5 of [Pinterest AI Agent — Milestones and Roadmap.md](../cross-stitch/Pinterest%20AI%20Agent%20—%20Milestones%20and%20Roadmap.md#L128-L186). It replaces the local JSON layer currently in `d:/ann/Git/cross-stitch/automation/pinterest-agent/reports/`.

## 2. Purpose

Persist every report family the Pinterest AI Agent currently writes to disk so that:

1. The agent has authoritative longitudinal memory across machines (today the JSON layer is local to the developer's workstation).
2. Multi-day AI trend analysis and design-level intelligence read from a single queryable source rather than reading and parsing per-day JSON files.
3. Anomaly events have a place to live for later notification (Milestone 8).
4. The path to Lambda/EventBridge (Milestone 7) is unblocked — Lambda has no persistent local disk to write JSON to.

- **Producer (writer):** `pinterest-agent` daily run scripts under `d:/ann/Git/cross-stitch/automation/pinterest-agent/scripts/`. Each script that writes `reports/*.json` today will also write a DDB row.
- **Consumer (reader):** `historyBuilder.ts`, trend-analysis scripts, design-analysis scripts. Future readers: anomaly detector, SES report builder (Milestone 8), Lambda runner (Milestone 7).

## 3. Scope

In scope:

- One DynamoDB table: `CrossStitchBusinessHistory` in `us-east-1`, account `358174257684` (same as `CrossStitchItems`, see [dynamodb-schema.md §4.1](../../docs/integration/dynamodb-schema.md#41-crossstitchitems--key-schema-and-indexes)).
- One companion S3 bucket: `cross-stitch-ai-reports` for AI analysis markdown bodies (DDB items hold an `S3Key` pointer instead of the body).
- Five entity families (§4.2 – §4.6).

Out of scope:

- `CrossStitchItems`, `CrossStitchUsers`, `PasswordResetTokens`, `SubscriptionEvents` — covered by [dynamodb-schema.md](../../docs/integration/dynamodb-schema.md).
- Raw Google API debug reports (`reports/YYYY-MM-DD-google-report.json` from [daily-google-report.ts:54](../../../cross-stitch/automation/pinterest-agent/scripts/daily-google-report.ts#L54)). The data they hold is already captured in `DAILY_BUSINESS` via [daily-business-report.ts:58](../../../cross-stitch/automation/pinterest-agent/scripts/daily-business-report.ts#L58). Kept on disk for debugging only; not persisted to DDB.
- The intermediate `reports/business-history.json` aggregate from [build-business-history.ts:44](../../../cross-stitch/automation/pinterest-agent/scripts/build-business-history.ts#L44) — becomes a *derived* artifact, recomputed on read from `DAILY_BUSINESS` rows.
- SES delivery (Milestone 8) and Lambda execution (Milestone 7).

## 4. Data Formats

### 4.1 Table key schema and indexes

| Element            | Attribute      | Type | Notes |
|--------------------|----------------|------|-------|
| Table              | `CrossStitchBusinessHistory` |  | Same region/account as `CrossStitchItems` |
| Partition key (PK) | `EntityType`   | S    | Discriminator. Values: `DAILY_BUSINESS`, `AI_ANALYSIS`, `DESIGN_PIN_MAP`, `DESIGN_PERFORMANCE`, `ANOMALY_EVENT` |
| Sort key (SK)      | `SortKey`      | S    | Composition varies by entity (see §4.2 – §4.6) |
| GSIs               | (none in v1)   |      | Add only when a real query demands one |
| Billing            | `PAY_PER_REQUEST` | | Matches `CrossStitchItems` |
| DeletionProtection | `true`         |      | Differs from `CrossStitchItems` — historical memory should be hard to drop accidentally |
| PITR               | enabled        |      | Backstop for backfill errors during the dual-write window |

SK string ordering matches numeric/chronological ordering for every entity family below — dates are ISO `YYYY-MM-DD`, timestamps are ISO 8601 UTC, and numeric IDs are zero-padded.

### 4.2 `EntityType = "DAILY_BUSINESS"` — one row per day

SK: `YYYY-MM-DD` (the report's `date` field, e.g. `2026-05-21`).

One row per calendar day. Replaces `reports/YYYY-MM-DD-business-report.json` written by [daily-business-report.ts:58](../../../cross-stitch/automation/pinterest-agent/scripts/daily-business-report.ts#L58).

Source shape: `BusinessReport` interface at [types.ts:19-28](../../../cross-stitch/automation/pinterest-agent/src/services/types.ts#L19-L28). Flattened into top-level attributes (no nested maps) so anomaly detection and trend reads stay cheap:

| Attribute | Type | Required | Source attribute | Notes |
|---|---|---|---|---|
| `EntityType` | S | yes | constant | `"DAILY_BUSINESS"` |
| `SortKey` | S | yes | `BusinessReport.date` | `YYYY-MM-DD` |
| `spend` | N | yes | `pinterestAds.spend` | rounded by `toDay()` in [historyBuilder.ts:21](../../../cross-stitch/automation/pinterest-agent/src/services/historyBuilder.ts#L21) |
| `impressions` | N | yes | `pinterestAds.impressions` | |
| `clicks` | N | yes | `pinterestAds.clicks` | |
| `ctr` | N | yes | `pinterestAds.ctr` | unrounded decimal (e.g. `0.005326`) |
| `cpc` | N | yes | `pinterestAds.cpc` | |
| `outboundClicks` | N | yes | `pinterestAds.outboundClicks` | |
| `ga4Sessions` | N | yes | `ga4PinterestSessions.total` | |
| `ga4PaidSessions` | N | yes | `ga4PinterestSessions.paidSocial` | |
| `ga4OrganicSessions` | N | yes | `ga4PinterestSessions.organic` | |
| `ga4ReferralSessions` | N | yes | `ga4PinterestSessions.referral` | |
| `adsenseRevenue` | N | yes | `adsense.estimatedEarnings` | |
| `revenuePerHundredSessions` | N | optional | `derived.revenuePerHundredPinterestSessions` | nullable in source; omit attribute when source is `null` |
| `profit` | N | yes | `derived.roughProfitEstimate` | |
| `writtenAt` | S | yes | (new) | ISO timestamp of the DDB write |

### 4.3 `EntityType = "AI_ANALYSIS"` — one row per AI run

SK: `{generatedAt}#{analysisType}` where `generatedAt` is ISO 8601 UTC and `analysisType ∈ {"trend", "design"}`.

Examples: `2026-05-21T02:34:00Z#trend`, `2026-05-21T02:34:05Z#design`.

Replaces three coupled files written by [test-ai-trend-analysis.ts:97-137](../../../cross-stitch/automation/pinterest-agent/scripts/test-ai-trend-analysis.ts#L97-L137) and [test-ai-design-analysis.ts:178-224](../../../cross-stitch/automation/pinterest-agent/scripts/test-ai-design-analysis.ts#L178-L224):

- `reports/ai-analysis/{date}-{trend|design}-analysis.md` (markdown body)
- `reports/ai-analysis/{date}-{trend|design}-analysis.json` (full Claude response)
- `reports/ai-recommendations-history.json` (append-only history of structured recs)

One DDB row per AI run holds everything except the markdown body (which goes to S3 — see §10).

| Attribute | Type | Required | Notes |
|---|---|---|---|
| `EntityType` | S | yes | `"AI_ANALYSIS"` |
| `SortKey` | S | yes | `{generatedAt}#{analysisType}` |
| `analysisType` | S | yes | `"trend"` or `"design"` |
| `forDate` | S | yes | The report date the analysis is *about*, not when it ran. Same as `date` in current JSON. |
| `recommendedAction` | S | optional | trend only — `"hold_budget"`, etc. From `ai-recommendations-history.json[].recommendedAction`. |
| `confidence` | N | optional | `0.0`-`1.0`. Source: `*-confidence.json` from [test-ai-trend-analysis.ts:106](../../../cross-stitch/automation/pinterest-agent/scripts/test-ai-trend-analysis.ts#L106) or `recommendation.confidence` from design analysis [test-ai-design-analysis.ts:203](../../../cross-stitch/automation/pinterest-agent/scripts/test-ai-design-analysis.ts#L203). |
| `reasoning` | S | yes | Short structured-output rationale (≤2KB). |
| `topAlbums` | L\<S\> | optional (design) | from `recommendation.topAlbums` |
| `underperformingAlbums` | L\<S\> | optional (design) | from `recommendation.underperformingAlbums` |
| `designDirectionsToCreate` | L\<S\> | optional (design) | from `recommendation.designDirectionsToCreate` |
| `sourceWindow` | M | optional | `{ label, startDate, endDate }` |
| `sourceHistoryRange` | M | optional (trend) | `{ first, last }` |
| `totalDaysAnalyzed` | N | optional (trend) | |
| `totalDesignsAnalyzed` | N | optional (design) | |
| `markdownS3Key` | S | yes | S3 object key for the full markdown body. See §10. |
| `writtenAt` | S | yes | (new) ISO timestamp of the DDB write. |

The markdown body itself does **not** live in DDB. DDB items are capped at 400KB and current trend-analysis markdowns are 8–25KB plus a 14KB design-insights markdown observed in `reports/ai-analysis/2026-05-21-trend-analysis.json`. They would fit, but S3 keeps the schema small, scans cheap, and avoids future surprise when analyses grow.

### 4.4 `EntityType = "DESIGN_PIN_MAP"` — one row per design

SK: `{designId:D5}` (e.g. `05386`).

Snapshot — overwritten on every export run. Replaces `reports/design-pin-map.json` written by [export-design-pin-map.ts:139](../../../cross-stitch/automation/pinterest-agent/scripts/export-design-pin-map.ts#L139).

Source: array of `{designId, albumId, albumCaption, pinId, designCaption, designUrl}` observed in `reports/design-pin-map.json`. The data originates from `CrossStitchItems` via the same DDB-read pattern documented in [dynamodb-schema.md §4.2](../../docs/integration/dynamodb-schema.md#42-entitytype--design--pattern-row); this table is a denormalized, AI-friendly projection.

| Attribute | Type | Required | Notes |
|---|---|---|---|
| `EntityType` | S | yes | `"DESIGN_PIN_MAP"` |
| `SortKey` | S | yes | `{designId:D5}` zero-padded |
| `designId` | N | yes | raw integer for reader convenience |
| `albumId` | N | yes | |
| `albumCaption` | S | yes | |
| `pinId` | S | yes | Pinterest pin ID, e.g. `"257127459971116086"`. Stored as `S` because IDs are 18-digit strings; see PinID drift note in [dynamodb-schema.md §4.4](../../docs/integration/dynamodb-schema.md#44-pinid-drift). |
| `designCaption` | S | yes | |
| `designUrl` | S | yes | Site-relative URL, e.g. `/Kitten-15-202-Free-Design.aspx` |
| `writtenAt` | S | yes | (new) ISO timestamp |

D5 padding accommodates 99,999 designs. Current max in `CrossStitchItems` is in the ~5,500 range; 18× headroom.

### 4.5 `EntityType = "DESIGN_PERFORMANCE"` — one row per (date, design)

SK: `{date}#{designId:D5}` (e.g. `2026-05-21#05386`).

Replaces `reports/design-performance.json` written by [build-design-performance.ts:105](../../../cross-stitch/automation/pinterest-agent/scripts/build-design-performance.ts#L105). The current file is one flat snapshot containing the latest window. By keying rows per-date we preserve history without overwriting; queries can fetch any past date's snapshot.

Source attributes (per `designs[]` entry in `design-performance.json`):

| Attribute | Type | Required | Notes |
|---|---|---|---|
| `EntityType` | S | yes | `"DESIGN_PERFORMANCE"` |
| `SortKey` | S | yes | `{snapshotDate}#{designId:D5}` |
| `snapshotDate` | S | yes | the `window.endDate` of the source file |
| `windowLabel` | S | yes | `"30d"` |
| `windowStartDate` | S | yes | |
| `windowEndDate` | S | yes | |
| `designId` | N | yes | |
| `albumId` | N | yes | |
| `albumCaption` | S | yes | |
| `pinId` | S | yes | |
| `designCaption` | S | yes | |
| `designUrl` | S | yes | |
| `impressions` | N | yes | |
| `clicks` | N | yes | |
| `outboundClicks` | N | yes | |
| `ctr` | N | yes | |
| `saves` | N | yes | |
| `writtenAt` | S | yes | (new) ISO timestamp |

A "this run had errors" row count is recorded out-of-band; only successful per-pin entries become DDB rows. Errors stay in the run log.

### 4.6 `EntityType = "ANOMALY_EVENT"` — one row per detected anomaly

SK: `{detectedAt}#{metric}` (e.g. `2026-05-21T02:35:00Z#ctr`).

New — no existing source file. Written by `src/services/anomalyDetector.ts` (to be added in implementation step 11 of [Milestones and Roadmap.md §Milestone 5](../cross-stitch/Pinterest%20AI%20Agent%20—%20Milestones%20and%20Roadmap.md#L128-L186)).

| Attribute | Type | Required | Notes |
|---|---|---|---|
| `EntityType` | S | yes | `"ANOMALY_EVENT"` |
| `SortKey` | S | yes | `{detectedAt}#{metric}` |
| `detectedAt` | S | yes | ISO timestamp |
| `forDate` | S | yes | the `DAILY_BUSINESS` date the anomaly fires on |
| `metric` | S | yes | one of: `ctr`, `revenuePerHundredSessions`, `profit`, `ga4Sessions` |
| `observedValue` | N | yes | the value that tripped the detector |
| `expectedMean` | N | yes | trailing-7-day mean |
| `expectedSigma` | N | yes | trailing-7-day stddev |
| `deviationSigmas` | N | yes | `(observed - mean) / sigma` |
| `direction` | S | yes | `"above"` or `"below"` |
| `notified` | BOOL | yes | `false` until Milestone 8 SES picks it up |

Detector default threshold: `\|deviationSigmas\| > 2`. Tunable per metric.

### 4.7 Example items

`DAILY_BUSINESS`:

```json
{
  "EntityType": "DAILY_BUSINESS",
  "SortKey": "2026-05-21",
  "spend": 12.05,
  "impressions": 22530,
  "clicks": 120,
  "ctr": 0.00533,
  "cpc": 0.10,
  "outboundClicks": 121,
  "ga4Sessions": 150,
  "ga4PaidSessions": 90,
  "ga4OrganicSessions": 55,
  "ga4ReferralSessions": 5,
  "adsenseRevenue": 14,
  "revenuePerHundredSessions": 9.33,
  "profit": 1.95,
  "writtenAt": "2026-05-22T02:02:54Z"
}
```

`AI_ANALYSIS` (design):

```json
{
  "EntityType": "AI_ANALYSIS",
  "SortKey": "2026-05-22T02:02:54Z#design",
  "analysisType": "design",
  "forDate": "2026-05-21",
  "topAlbums": ["Horses", "Cats", "Free"],
  "underperformingAlbums": ["Animals"],
  "designDirectionsToCreate": ["Additional horse portrait designs…", "…"],
  "confidence": 0.52,
  "reasoning": "Top-line signals are clear…",
  "sourceWindow": { "label": "30d", "startDate": "2026-04-22", "endDate": "2026-05-21" },
  "totalDesignsAnalyzed": 60,
  "markdownS3Key": "analysis/2026-05-21/2026-05-22T02-02-54Z-design.md",
  "writtenAt": "2026-05-22T02:02:55Z"
}
```

## 5. API Endpoints / Interfaces

Not a network API. Access is through the AWS SDK from inside `pinterest-agent`.

Two new TypeScript modules wrap the SDK so call sites stay readable:

- `src/services/historyStore.ts` — DDB wrapper. Method names listed in [Milestones and Roadmap.md §Milestone 5 step 3](../cross-stitch/Pinterest%20AI%20Agent%20—%20Milestones%20and%20Roadmap.md#L128-L186).
- `src/services/aiArtifactStore.ts` — S3 wrapper for the markdown bodies.

Required env vars (read by `readPlatformConfig.ts`-style loader):

| Var | Default | Purpose |
|---|---|---|
| `AWS_REGION` | `us-east-1` | matches existing AWS calls |
| `HISTORY_TABLE_NAME` | `CrossStitchBusinessHistory` | overridable for test/staging tables |
| `AI_ARTIFACT_BUCKET` | `cross-stitch-ai-reports` | overridable for test |
| `HISTORY_SOURCE` | `json` (during dual-write), then `ddb` | feature flag for step 8 read cutover |

### 5.1 Credentials

Two distinct AWS principals access this layer:

- **Daily-run agent scripts** (`daily`, `history`, `ai:trend`, `pinmap`, `perf`, `ai:design`) authenticate as the `pinterest-agent` IAM user. Static credentials live in `automation/pinterest-agent/.env` and are loaded via `dotenv/config`. The user inherits `CrossStitch-DynamoDB-ReadOnly`, `CrossStitch-BusinessHistory-Write`, and `CrossStitch-AIReports-Write` through the `CrossStitch-Agents` group.
- **Init / admin scripts** (`init`, future `backfill`, `verify-parity`) authenticate as the `claude-dev` IAM user via the **named profile `claude-dev`** in `~/.aws/credentials`. Invoke with `AWS_PROFILE=claude-dev npm run init`. The user inherits `CrossStitch-DynamoDB-FullAccess` and `CrossStitch-BusinessHistory-Admin` through the `CrossStitch-Developers` group.

The init script actively clears `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` from its process env when `AWS_PROFILE` is set, so a System- or User-scope static-credential env var doesn't conflict with the profile and trigger the SDK's "Multiple credential sources detected" warning.

## 6. Versioning

- **Schema version:** `v1`. Stored implicitly — the `EntityType` discriminator + attribute set is the contract; no `schemaVersion` attribute in v1.
- **Strategy:** additive only during v1. Adding new attributes to existing entity families requires no migration. Adding new entity families requires only a new `EntityType` value and a new section in this doc.
- **Breaking changes** (rename, type change, key reshape) require: bump `schemaVersion` attribute on affected rows, write migration script, update this contract.
- **Change history:**
  - 2026-05-22 — v1 initial draft for Milestone 5 (this document).

## 7. Ownership & Contacts

- **Owner:** Olga (`olga.epstein@gmail.com`).
- **Producer code:** `d:/ann/Git/cross-stitch/automation/pinterest-agent/`.
- **Consumer code:** same repo today; future Lambda runtime (Milestone 7).
- **Schema doc owner:** this file. Update here when entity shapes change in code.

## 8. Dependencies

- `CrossStitchItems` for the source-of-truth design ↔ pin mapping that feeds `DESIGN_PIN_MAP` rows (see [dynamodb-schema.md §4.2](../../docs/integration/dynamodb-schema.md#42-entitytype--design--pattern-row)).
- Pinterest Ads API + Pin Analytics API — feeds `DAILY_BUSINESS` and `DESIGN_PERFORMANCE`.
- GA4 Reporting API + AdSense Management API — feeds `DAILY_BUSINESS`.
- Anthropic API — feeds `AI_ANALYSIS`.
- S3 bucket `cross-stitch-ai-reports` for markdown bodies referenced by `AI_ANALYSIS.markdownS3Key`.
- AWS SDK v3 (`@aws-sdk/client-dynamodb`, `@aws-sdk/lib-dynamodb`, `@aws-sdk/client-s3`) — already in [pinterest-agent/package.json](../../../cross-stitch/automation/pinterest-agent/package.json) for `@aws-sdk/client-dynamodb`; add `lib-dynamodb` (document client) and `client-s3`.

## 9. Error Handling

Failure model for the producer (daily run):

- **DDB write fails for a single entity:** log, continue with the rest of the daily run, fail the overall run at the end if any write failed. JSON fallback still writes during the dual-write window, so no data loss.
- **S3 upload fails for an AI markdown body:** abort the AI write for that run (don't write a DDB row with a stale or empty `markdownS3Key`). Retry on next run.
- **DDB throttling:** SDK default retries (3, exponential). `PAY_PER_REQUEST` billing means we will not hit provisioned-throughput limits in normal use; bursts during backfill may need batched writes.
- **Conditional write conflicts:** `DAILY_BUSINESS` uses `PutItem` with no condition — re-runs for the same date overwrite. This is intentional: the daily run is idempotent and the latest run is canonical.
- **Schema drift detected by `verify-history-parity.ts`:** the parity check fails the daily run; we investigate before continuing. Better to halt than silently diverge.

Reader side (history queries):

- **Missing day:** return gap silently. Trend math already tolerates gaps (no daily row for `2026-05-17` in the existing local JSON sample — `historyBuilder.ts` just sorts and works with what's present).
- **Query throttling on large backfill reads:** use the SDK paginator pattern; never `Scan`.

## 10. Security & Compliance

- **Data sensitivity:** business metrics (spend, revenue, sessions) — internal only, no end-user PII. The Pinterest pin IDs and design URLs are public. AI reasoning bodies are internal strategy.
- **Encryption at rest:** DynamoDB AWS-managed encryption (default). S3 bucket: `SSE-S3` (AWS-managed) is sufficient; no need for KMS at this stage.
- **Access control (IAM):** new policy attached to the agent's IAM principal:
  - `dynamodb:PutItem`, `GetItem`, `Query`, `BatchWriteItem` on `arn:aws:dynamodb:us-east-1:358174257684:table/CrossStitchBusinessHistory`
  - `s3:PutObject`, `GetObject` on `arn:aws:s3:::cross-stitch-ai-reports/analysis/*`
  - No public access on the S3 bucket — block-public-access on, no bucket policy granting read.
- **Deletion protection:** enabled on the table (differs from `CrossStitchItems`, which has it off per [dynamodb-schema.md §10](../../docs/integration/dynamodb-schema.md#10-security--compliance)). Historical memory loss is harder to recover from than catalog loss because there is no second source.
- **PITR:** enabled — backstop for the backfill window.
- **Compliance:** no GDPR/PII concerns; no end-user records.

## 11. Testing & Validation

- **`scripts/verify-history-parity.ts`** runs daily during the soak window and fails the daily run if DDB-reconstructed JSON doesn't match the on-disk JSON. Specific checks:
  - `DAILY_BUSINESS` for every date in `reports/` exists in DDB with byte-identical rounded values.
  - `AI_ANALYSIS` count for each day matches the count of `*-analysis.json` files plus the `ai-recommendations-history.json` entries for that day.
  - `DESIGN_PIN_MAP` row count matches `design-pin-map.json` length.
  - `DESIGN_PERFORMANCE` row count for the latest snapshot date matches `design-performance.json.designs.length`.
  - `AI_ANALYSIS.markdownS3Key` resolves to an S3 object whose body equals the local `.md` file.
- **Manual smoke test before cutover:** run `scripts/build-business-history.ts` once against `HISTORY_SOURCE=ddb` and diff its `business-history.json` output against the previous day's local-JSON run.
- **Backfill correctness:** `backfill-history.ts` is idempotent (deterministic SKs, `PutItem`). Re-running over the same source files must produce the same row set; verified by row-count assertion at the end of the script.

## 12. References

- [Pinterest AI Agent — Milestones and Roadmap.md §Milestone 5](../cross-stitch/Pinterest%20AI%20Agent%20—%20Milestones%20and%20Roadmap.md#L128-L186) — the implementation plan this contract supports.
- [Pinterest AI Agent — Memory and Trend Analysis.md](../cross-stitch/Pinterest%20AI%20Agent%20—%20Memory%20and%20Trend%20Analysis.md) — strategic framing for historical memory.
- [Pinterest AI Agent — AWS Deployment.md §DynamoDB](../cross-stitch/Pinterest%20AI%20Agent%20—%20AWS%20Deployment.md#L69-L75) — high-level AWS plan.
- [Pinterest AI Agent — Design-Level Intelligence.md](../cross-stitch/Pinterest%20AI%20Agent%20—%20Design-Level%20Intelligence.md#L131) — original mention of `DesignPinMap`, `DesignPerformance`, `AIDesignInsight` names (now realized as `EntityType` values here).
- [dynamodb-schema.md](../../docs/integration/dynamodb-schema.md) — the existing `CrossStitchItems` / `CrossStitchUsers` schema this table sits alongside; this contract follows the same single-table-design conventions.
- [pinterest-agent/src/services/types.ts](../../../cross-stitch/automation/pinterest-agent/src/services/types.ts) — TypeScript types for the source `BusinessReport` / `DayMetrics` shapes.
- Writer code paths cited per attribute table above.
