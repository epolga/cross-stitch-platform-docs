# Contract: Pinterest metadata — pin/board metadata, OAuth scopes, token store schema

> CONTRACT-TEMPLATE.md (12-section) — combines four closely-related Pinterest contracts that share OAuth credentials and a single token file: (a) pin caption/title/description templates, (b) board name/description conventions, (c) AlbumID→BoardID mapping CSV, (d) on-disk OAuth token JSON schema.

---

## 1. Contract Name

**Pinterest metadata** — pin/board metadata, OAuth scopes, token store schema.

Aliases: `pinterest-metadata`, `pinterest-pin-contract`, `pinterest-board-contract`, `pinterest-token-store`, `pinterest-oauth-scope`.

This contract bundles four implicit sub-contracts that travel together:

| Sub-contract | Producer | Consumer |
|---|---|---|
| Pin title/description/alt_text/link/media_source payload | Uploader `PinterestHelper` | Pinterest REST API v5 |
| Board name/description payload (create + SEO rename) | Uploader `PinterestBoardCreator`, `PinterestBoardRenamer` | Pinterest REST API v5 |
| `AlbumBoards.csv` (AlbumID 4-digit → BoardID) | Uploader `PinterestBoardCreator` | Uploader `PinterestHelper`, `PinterestBoardRenamer` |
| `pinterest_tokens.json` (OAuth tokens at rest) | Uploader `PinterestOAuthClient` (writer) | cross-stitch `pinterest-agent` `readPinterestAccessToken` (reader), Uploader `PinterestOAuthClient` (reader) |

---

## 2. Purpose

Document the exact strings, JSON shapes, file paths and OAuth scopes the Uploader uses when talking to Pinterest, and what cross-stitch's `pinterest-agent` expects to read back. The Uploader is the **sole writer** for Pinterest pins, boards, and the OAuth token file. The cross-stitch monorepo (specifically the `pinterest-agent` automation) is a **read-only consumer** of the token file and Pinterest's API.

This contract exists because:

- Both repos parse the same `pinterest_tokens.json` (writer in C# with Newtonsoft attributes — `d:/ann/Git/Uploader/Uploader/Helpers/PinterestTokenInfo.cs:17-34`; reader in TypeScript — `d:/ann/Git/cross-stitch/automation/pinterest-agent/src/services/readPinterestToken.ts:4-10`) with no shared schema file.
- The OAuth scope set (`pins:read pins:write boards:read boards:write ads:read`) is duplicated across `App.config` (`d:/ann/Git/Uploader/Uploader/App.config:14`), the planning doc (`d:/ann/Git/cross-stitch-platform-docs/plan/cross-stitch/Pinterest AI Agent — API Integrations.md:66-70`) and is implicitly validated by `test-pinterest-api.ts` (`d:/ann/Git/cross-stitch/automation/pinterest-agent/scripts/test-pinterest-api.ts:26`).
- Pin/board metadata templates are written in code only; downstream consumers (e.g. Pinterest's own SEO/search) and the `pinterest-agent` need to know what naming conventions to expect.

---

## 3. Scope

**In scope**

- Pin payload fields produced by `PinterestHelper.UploadPinForPatternAsync` (`d:/ann/Git/Uploader/Uploader/Helpers/PinterestHelper.cs:63-149`): `board_id`, `link`, `title`, `description`, `alt_text`, `media_source.{source_type,url}`.
- Title / description / alt_text string templates and length limits (100 / 500 / 500 chars).
- Theme keyword → hashtag table (`d:/ann/Git/Uploader/Uploader/Helpers/PinterestHelper.cs:283-362`).
- Board create payload `{name, description}` (`d:/ann/Git/Uploader/Uploader/Helpers/PinterestBoardCreator.cs:177-181`).
- SEO board rename payload `{name, description}` and the `< 50` char board-name rule (`d:/ann/Git/Uploader/Uploader/Helpers/PinterestBoardRenamer.cs:149-178`).
- `AlbumBoards.csv` (header `AlbumID,AlbumCaption,BoardID`, AlbumID 4-digit zero-padded) (`d:/ann/Git/Uploader/Uploader/AlbumBoards.csv:1-2`).
- OAuth scope set and token-endpoint URL (`d:/ann/Git/Uploader/Uploader/App.config:9-16`, `d:/ann/Git/Uploader/Uploader/Helpers/PinterestTokenInfo.cs:139`).
- Token JSON schema on disk (writer fields: `access_token`, `refresh_token`, `scope`, `token_type`, `expires_at_utc`; reader optional vs. required asymmetry).

**Out of scope**

- The Pinterest pin-ID DynamoDB attribute (`PinID`) and the six historical variant names the cross-stitch repo probes — covered by a separate "Pin-ID DynamoDB attribute" contract (`d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:1080`, `d:/ann/Git/cross-stitch/automation/pinterest-agent/scripts/export-design-pin-map.ts`).
- The S3 image URL passed as `media_source.url` (covered by "S3 photo path" contract, see `d:/ann/Git/Uploader/Uploader/Helpers/PatternLinkHelper.cs:73-76`).
- The design-page URL passed as `link` (covered by "Design page URL" contract, see `d:/ann/Git/Uploader/Uploader/Helpers/PatternLinkHelper.cs:62-71`).
- DynamoDB schema for albums (`EntityType=ALBUM`, ID prefix `ALB#`) — covered separately (`d:/ann/Git/Uploader/Uploader/Helpers/PinterestBoardCreator.cs:122,141-142`).

---

## 4. Data Formats

### 4.1 Pin payload (POST /v5/pins)

Built in `d:/ann/Git/Uploader/Uploader/Helpers/PinterestHelper.cs:111-123`:

```csharp
var payload = new
{
    board_id = boardId,             // resolved via AlbumBoards.csv (§4.4)
    link = patternUrl,              // see PatternLinkHelper.BuildPatternUrl
    title,                          // §4.2
    description,                    // §4.3
    alt_text = altText,             // §4.5
    media_source = new
    {
        source_type = "image_url",  // PinterestHelper.cs:120
        url = imageUrl              // CloudFront/S3 photo URL
    }
};
```

Serialized via `JsonConvert.SerializeObject(payload)` → `application/json` (`d:/ann/Git/Uploader/Uploader/Helpers/PinterestHelper.cs:125-126`).

Response is parsed into the private DTO `PinResponse { string id }` (`d:/ann/Git/Uploader/Uploader/Helpers/PinterestHelper.cs:594-597`) — this `id` is what later gets written to DynamoDB as `PinID`.

### 4.2 Pin title template

From `BuildPinTitle` (`d:/ann/Git/Uploader/Uploader/Helpers/PinterestHelper.cs:399-420`):

```
{Title or "Cross stitch pattern"} – {ToSentenceCase(theme.HumanName)}, printable PDF pattern
```

- Hyphen separator is the **en-dash** `–` (U+2013), not `-`.
- Hard-trimmed to `maxLength = 100` chars (`PinterestHelper.cs:413-417`).
- Example (synthetic, matching code comment at line 402): `"Cute Cats – Cat cross stitch pattern, printable PDF pattern"`.

### 4.3 Pin description template

From `BuildPinDescription` (`d:/ann/Git/Uploader/Uploader/Helpers/PinterestHelper.cs:472-561`):

Concatenation order (each step conditional):

1. `{pattern.Title.Trim()} – ` (only if Title non-empty) — `PinterestHelper.cs:481-486`.
2. `{theme.HumanName}. ` — `PinterestHelper.cs:488-489`.
3. If `Width>0 && Height>0 && NColors>0`:
   `Detailed counted cross stitch chart ({Width} × {Height} stitches, {NColors} colours). ` (note: `×` is U+00D7, invariant culture) — `PinterestHelper.cs:491-499`.
   Else: `Beautiful counted cross stitch design. ` — `PinterestHelper.cs:501-503`.
4. `{pattern.Description.Trim()} ` (if non-empty) — `PinterestHelper.cs:505-509`.
5. `{pattern.Notes.Trim()} ` (if non-empty) — `PinterestHelper.cs:511-515`.
6. `From album {AlbumId:D4}. Download printable PDF and see more details at {patternUrl}. ` — `PinterestHelper.cs:517-521` (AlbumId is **zero-padded to 4 digits** inside the description text).
7. `Perfect for embroidery lovers and cross stitch fans. ` — `PinterestHelper.cs:523`.
8. Newline + hashtags, joined with single space (`PinterestHelper.cs:545-549`):
   - Generic (always): `#crossstitch #crossstitchpattern #embroidery #needlework #crossstitchkit` — `PinterestHelper.cs:526-533`.
   - Plus theme-specific (see §4.6).
   - Deduplicated `OrdinalIgnoreCase`, original order preserved — `PinterestHelper.cs:539-543`.
9. Hard-trimmed to `maxLength = 500` chars — `PinterestHelper.cs:553-558`.

### 4.4 Alt text template

From `BuildAltText` (`d:/ann/Git/Uploader/Uploader/Helpers/PinterestHelper.cs:422-470`):

Joined by `", "` :

```
Counted cross stitch pattern, {Title or theme.HumanName}[, {Width} by {Height} stitches][, {NColors} colours]
```

- Optional size and colour parts only added when values > 0 (`PinterestHelper.cs:434-453`).
- Hard-trimmed to 500 chars (`PinterestHelper.cs:463-467`).
- Example (from code comment line 426): `"Counted cross stitch pattern with two cute cats, 150 by 200 stitches, 40 colours."`

### 4.5 AlbumBoards.csv schema

Path: defaults to `AlbumBoards.csv` (relative to working dir), overridable via `appSettings["PinterestBoardsCsvPath"]` (`d:/ann/Git/Uploader/Uploader/Helpers/PinterestHelper.cs:47-49`, `d:/ann/Git/Uploader/Uploader/Helpers/PinterestBoardCreator.cs:55`, `d:/ann/Git/Uploader/Uploader/Helpers/PinterestBoardRenamer.cs:25`).

Header (case-sensitive line 1) (`d:/ann/Git/Uploader/Uploader/Helpers/PinterestBoardCreator.cs:84`):

```
AlbumID,AlbumCaption,BoardID
```

Row format (`d:/ann/Git/Uploader/Uploader/Helpers/PinterestBoardCreator.cs:204-208`):

```
{AlbumID-4digit},"{Caption-with-"-doubled}",{BoardID}
```

| Column | Type | Source | Notes |
|---|---|---|---|
| AlbumID | string, **4-digit zero-padded** (`D4`) | DynamoDB ID prefix `ALB#` stripped, see `PinterestBoardCreator.cs:141-145` | Reader produces same key via `albumId.ToString("D4", CultureInfo.InvariantCulture)` — `PinterestHelper.cs:161` |
| AlbumCaption | string, **always wrapped in `"…"`**, `"` → `""` | DynamoDB `Caption` attr (`PinterestBoardCreator.cs:146`) | May contain commas; parsers in `PinterestHelper.cs:227-253` and `PinterestBoardRenamer.cs:87-116` handle this |
| BoardID | numeric string (Pinterest board ID, e.g. `257127528664615685`) | Pinterest `POST /v5/boards` response `id` (`PinterestBoardCreator.cs:193-198`) | Stored unquoted |

Example rows (`d:/ann/Git/Uploader/Uploader/AlbumBoards.csv:2-9`):

```
0104,"Cushion Covers",257127528664615685
0037,"Animals",257127528664615687
0082,"Western Theme",257127528664615690
```

CSV is written with UTF-8 (`PinterestBoardCreator.cs:103`), read with UTF-8 (`PinterestHelper.cs:201`, `PinterestBoardRenamer.cs:43`). First row is skipped as header (`PinterestHelper.cs:207`, `PinterestBoardRenamer.cs:58`). Missing-CSV behaviour: lookup returns empty map → fallback to `appSettings["PinterestBoardId"]` (`PinterestHelper.cs:175-182`).

### 4.6 Theme detection table

Themes table (`d:/ann/Git/Uploader/Uploader/Helpers/PinterestHelper.cs:283-362`). `DetectTheme` scores each theme by the count of its `Keywords` found in `{Title} {Description} {Notes}.ToLowerInvariant()` and picks the highest scorer; ties resolved by table order (first wins); zero matches → `DefaultTheme` (`PinterestHelper.cs:368-397`).

| Code | HumanName | Keywords | Hashtags |
|---|---|---|---|
| `general` (default) | `cross stitch pattern` | (none) | `#crossstitch #crossstitchpattern #embroidery #needlework` |
| `cats` | `cat cross stitch pattern` | cat, kitten, kitty | `#cat #cats #catlover #kitty` |
| `dogs` | `dog cross stitch pattern` | dog, puppy, pup | `#dog #dogs #doglover #puppy` |
| `birds` | `bird cross stitch pattern` | bird, sparrow, owl, eagle, parrot | `#birds #birdart` |
| `flowers` | `floral cross stitch pattern` | flower, rose, tulip, poppy, bouquet, floral | `#flowers #floral` |
| `nature` | `nature cross stitch pattern` | forest, tree, mountain, lake, river, landscape, nature | `#landscape #nature` |
| `seaside` | `seaside cross stitch pattern` | sea, ocean, beach, coast, harbor, harbour | `#seaside #ocean #beach` |
| `city` | `city cross stitch pattern` | city, street, house, houses, architecture, building | `#cityscape #architecture` |
| `people` | `people cross stitch pattern` | girl, boy, woman, man, people, portrait | `#portrait #people` |
| `fantasy` | `fantasy cross stitch pattern` | fairy, dragon, unicorn, wizard, magic, fantasy | `#fantasy #fairytales` |
| `christmas` | `Christmas cross stitch pattern` | christmas, xmas, santa, snowman, reindeer, christmas tree | `#christmas #christmasdecor #winter` |
| `easter` | `Easter cross stitch pattern` | easter, egg, eggs, bunny, rabbit | `#easter #spring` |

### 4.7 Board create payload

From `PinterestBoardCreator.CreateBoardForAlbumAsync` (`d:/ann/Git/Uploader/Uploader/Helpers/PinterestBoardCreator.cs:165-198`):

```jsonc
{
  "name":        "{Caption or \"Album {AlbumId}\"}",                     // line 171-173
  "description": "Cross-stitch patterns from album {AlbumId}: {Caption}" // line 175
}
```

- `name` falls back to literal `Album {AlbumId}` if `Caption` is empty.
- No length trim at create time; later corrected by `PinterestBoardRenamer` (§4.8).

### 4.8 SEO board-rename payload

From `PinterestBoardRenamer.BuildSeoBoardName` (`d:/ann/Git/Uploader/Uploader/Helpers/PinterestBoardRenamer.cs:149-178`):

Name template:

```
{Caption.Trim() or "Cross Stitch Free"} Cross Stitch Free
```

Idempotency rule: if `Caption` already contains the substring `"Cross Stitch Free"` (case-insensitive, `OrdinalIgnoreCase`), the suffix is **not** re-appended — `PinterestBoardRenamer.cs:160-167`.

Length rule: **MUST be < 50 chars** (Pinterest requirement); enforced as `maxLength = 48` with safety margin, then word-boundary trim via `TrimToMaxLength` (`PinterestBoardRenamer.cs:169-178, 183-194`). Hard cut only if no space found.

Description template (`PinterestBoardRenamer.cs:199-220`):

```
Cross stitch patterns from album {Caption or "beautiful counted cross stitch designs"}.
Discover printable cross stitch charts and downloadable PDF patterns at Cross-Stitch.com .
Perfect for both beginners and experienced stitchers who love detailed embroidery designs.
```

(Single line; rendered above with line breaks for readability.) Trimmed to 500 chars (`PinterestBoardRenamer.cs:213-217`).

Note: the description string contains the **literal** `"Cross-Stitch.com ."` — two characters separated from the period by a single space — exactly as in source line 209.

PATCH endpoint: `https://api.pinterest.com/v5/boards/{boardId}` with same `{name, description}` shape (`PinterestBoardRenamer.cs:232-247`).

### 4.9 OAuth scope set

Configured at `d:/ann/Git/Uploader/Uploader/App.config:14`:

```xml
<add key="PinterestScope" value="pins:read pins:write boards:read boards:write ads:read"/>
```

Note: **space-separated** in `App.config`. `BtnPinterestReAuth_Click` rewrites them to **comma-separated** before forming the auth URL (`d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:294-302`):

```csharp
var scopeParam = scope.Replace(' ', ',');
var authUrl = $"https://www.pinterest.com/oauth/?client_id={clientId}…&scope={Uri.EscapeDataString(scopeParam)}";
```

`test-pinterest-api.ts` indirectly asserts `pins:read`, `boards:read`, `ads:read` by exercising `/pins`, `/boards`, `/ad_accounts` (`d:/ann/Git/cross-stitch/automation/pinterest-agent/scripts/test-pinterest-api.ts:4-26`). The planning doc currently lists only `ads:read boards:read boards:write pins:read pins:write` (`d:/ann/Git/cross-stitch-platform-docs/plan/cross-stitch/Pinterest AI Agent — API Integrations.md:66-70`) — same effective set, different order.

### 4.10 Token JSON file shape

**Path** (both sides resolve identically): `<workspaceRoot>/Uploader/secrets/pinterest_tokens.json`, where `<workspaceRoot>` is the parent of `cross-stitch-platform-docs/`. Resolved via `platform-config.json`:

- Config key: `pinterestTokenPath` (`d:/ann/Git/cross-stitch-platform-docs/platform-config.json:2`).
- C# resolver: `PlatformConfig.ResolvePinterestTokenPath` (`d:/ann/Git/Uploader/Uploader/Helpers/PlatformConfig.cs:18-48`).
- TypeScript resolver: `resolvePinterestTokenPath` (`d:/ann/Git/cross-stitch/automation/pinterest-agent/src/services/readPlatformConfig.ts:64-78`).

**Writer schema** (Uploader serializes ALL fields via Newtonsoft, `Formatting.Indented`, `d:/ann/Git/Uploader/Uploader/Helpers/PinterestTokenInfo.cs:117-123`):

```jsonc
{
  "access_token":  "string",         // PinterestTokenInfo.cs:17-18  required
  "refresh_token": "string",         // PinterestTokenInfo.cs:20-21  required (writer always emits)
  "scope":         "string",         // PinterestTokenInfo.cs:23-24  e.g. "pins:read,pins:write,boards:read,boards:write,ads:read"
  "token_type":    "bearer",         // PinterestTokenInfo.cs:26-27  default literal "bearer"
  "expires_at_utc":"2026-05-22T12:34:56Z" // PinterestTokenInfo.cs:29-33  ISO-8601 UTC, computed from /oauth/token's expires_in (PinterestTokenInfo.cs:46-58)
}
```

**Reader schema** (cross-stitch `pinterest-agent`):

```ts
interface PinterestTokenFile {
  access_token: string;      // REQUIRED — readPinterestToken.ts:5, validated at line 28-30
  refresh_token?: string;    // OPTIONAL — readPinterestToken.ts:6
  scope?: string;            // OPTIONAL — readPinterestToken.ts:7
  token_type?: string;       // OPTIONAL — readPinterestToken.ts:8
  expires_at_utc?: string;   // OPTIONAL — readPinterestToken.ts:9
}
```

**Asymmetry**: the writer always serializes every field; the reader only requires `access_token` and silently ignores the rest. Consequence: a hand-edited token file containing only `{"access_token":"…"}` is acceptable to the reader but useless to the Uploader, which will fail in `RefreshAccessTokenAsync` with `"No refresh token available."` (`d:/ann/Git/Uploader/Uploader/Helpers/PinterestTokenInfo.cs:199-200`).

The standalone smoke script `d:/ann/Git/Uploader/scripts/test-pinterest-ads.ts:30-37` also reads only `data.access_token`.

---

## 5. API Endpoints / Interfaces

### 5.1 Pinterest REST API v5 (HTTP, called by Uploader)

Base URL constant: `https://api.pinterest.com/v5` (`d:/ann/Git/Uploader/Uploader/Helpers/PinterestHelper.cs:32`). Also duplicated in `App.config:11-12` and used directly by `PinterestBoardCreator.cs:186` and `PinterestBoardRenamer.cs:232`.

| Verb | Path | Used by | Purpose |
|---|---|---|---|
| `POST` | `/v5/pins` | `PinterestHelper.UploadPinForPatternAsync` (`d:/ann/Git/Uploader/Uploader/Helpers/PinterestHelper.cs:128`) | Create a Pin (returns `{id}`) |
| `POST` | `/v5/boards` | `PinterestBoardCreator.CreateBoardForAlbumAsync` (`d:/ann/Git/Uploader/Uploader/Helpers/PinterestBoardCreator.cs:186`) | Create a board |
| `PATCH` | `/v5/boards/{boardId}` | `PinterestBoardRenamer.RenameBoardAsync` (`d:/ann/Git/Uploader/Uploader/Helpers/PinterestBoardRenamer.cs:232,243`) | SEO rename a board |
| `POST` | `/v5/oauth/token` | `PinterestOAuthClient.ExchangeAuthorizationCodeAsync` / `RefreshAccessTokenAsync` (`d:/ann/Git/Uploader/Uploader/Helpers/PinterestTokenInfo.cs:139,167,205`) | Exchange code or refresh token |

OAuth authorization URL (browser leg, built in C# rather than constant) (`d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:299-302`):

```
https://www.pinterest.com/oauth/?client_id={PinterestClientId}&redirect_uri={url-escaped redirect}&response_type=code&scope={url-escaped comma-separated scopes}
```

Loopback redirect URI: `http://localhost:8080/callback` (`d:/ann/Git/Uploader/Uploader/App.config:13`).

### 5.2 Pinterest REST API v5 (HTTP, called by cross-stitch pinterest-agent — read-only)

Wrapper: `pinterestGet<T>(path)` in `d:/ann/Git/cross-stitch/automation/pinterest-agent/src/services/pinterestClient.ts:11-47`. Constants:

- Base: `https://api.pinterest.com/v5` (`pinterestClient.ts:4`).
- Max retries on `429`: 6 with exponential backoff capped at 30s, honours `Retry-After` header (`pinterestClient.ts:5,29-41`).

Endpoints exercised (read-only) (`d:/ann/Git/cross-stitch/automation/pinterest-agent/scripts/test-pinterest-api.ts`):

- `GET /ad_accounts` (line 5)
- `GET /boards` (line 11)
- `GET /pins` (line 19)

### 5.3 File-based interface: `pinterest_tokens.json`

The only direct cross-repo interface. Producer/consumer flow:

```
+---------------+ writes +-----------------------------+ reads  +-----------------------------+
| Uploader      | -----> | Uploader/secrets/           | -----> | pinterest-agent             |
| PinterestOAuthClient   |   pinterest_tokens.json     |        | readPinterestAccessToken    |
| (write+read)  |        | (resolved via               |        | (read-only)                 |
+---------------+        |  platform-config.json)      |        +-----------------------------+
                         +-----------------------------+
```

### 5.4 File-based interface: `AlbumBoards.csv`

Producer = `PinterestBoardCreator` (writes UTF-8). Consumers = `PinterestHelper` (lookup AlbumID → BoardID, lazy-cached `Dictionary<string,string>` keyed by `D4` AlbumID, `d:/ann/Git/Uploader/Uploader/Helpers/PinterestHelper.cs:39,159-183`) and `PinterestBoardRenamer` (iterates rows to PATCH boards). No cross-repo consumer.

### 5.5 File-based interface: `platform-config.json`

Producer = `cross-stitch-platform-docs` repo. Consumer = `PlatformConfig.cs` (C#) and `readPlatformConfig.ts` (TS). Sole key today: `pinterestTokenPath` (string, relative to workspace root or absolute). Resolution rule is symmetrical in both languages: walk up from the running assembly/file looking for sibling `cross-stitch-platform-docs/platform-config.json`, or honour `PLATFORM_CONFIG_PATH` env override. C#: `PlatformConfig.cs:50-74`. TS: `readPlatformConfig.ts:17-43`.

---

## 6. Versioning

**Unversioned (implicit) across all four sub-contracts.**

| Surface | Version state |
|---|---|
| Pinterest REST | `v5` hard-coded in URL constants (writer: `PinterestHelper.cs:32`, `App.config:11-12`; reader: `pinterestClient.ts:4`). Bumping Pinterest API would require coordinated edits in 4 files. |
| Pin/board/csv/token JSON schemas | No `schemaVersion` field anywhere. |
| Theme keyword table | Code-embedded (`PinterestHelper.cs:283-362`), no versioning. |
| OAuth scope string | Free-text in `App.config:14`; no migration path. |

**Recommendation**: add `"schemaVersion": "1"` to `pinterest_tokens.json` (writer emits, reader tolerates absence as `1`) and to `platform-config.json`. For pin/board metadata, accept this as a server-driven contract — Pinterest's `/v5` policy is the version.

---

## 7. Ownership & Contacts

- **Code owner**: the writer — Uploader repo `d:/ann/Git/Uploader/`. The Uploader is the sole producer of pins, boards, the CSV, and the token file.
- **Maintainer**: Olga (epolga, olga.epstein@gmail.com).
- **Read-side stakeholder**: cross-stitch `pinterest-agent` (`d:/ann/Git/cross-stitch/automation/pinterest-agent/`).
- **Docs**: `d:/ann/Git/cross-stitch-platform-docs/plan/cross-stitch/Pinterest AI Agent — API Integrations.md` (only the scope list is mirrored there today).

Any change to scope set, token JSON shape, or `AlbumBoards.csv` columns requires an update to both repos plus the planning doc.

---

## 8. Dependencies

Upstream (this contract reads from):

- **AlbumID** contract — 4-digit zero-padded numeric (`PinterestHelper.cs:161`, `AlbumBoards.csv:2`). Also flows into description text via `AppendFormat("From album {0:D4}…", albumId)` (`PinterestHelper.cs:517-521`).
- **DesignID** contract — required to build pin `link` URL (`PinterestHelper.cs:75-76, 84`).
- **Design page URL** contract — pin `link` target (`PatternLinkHelper.cs:62-71`).
- **S3 photo path** contract — pin `media_source.url` (`PatternLinkHelper.cs:73-76`, default photo `4.jpg`).
- **DynamoDB album records** — `EntityType = "ALBUM"`, ID prefix `ALB#`, attribute `Caption` (read by `PinterestBoardCreator.LoadAlbumsAsync`, `d:/ann/Git/Uploader/Uploader/Helpers/PinterestBoardCreator.cs:116-159`).
- **`platform-config.json`** — `pinterestTokenPath` key (`platform-config.json:2`).

Downstream (this contract writes to / is consumed by):

- **DynamoDB `PinID` attribute** — the `id` returned from `POST /v5/pins` is later written to DynamoDB by `MainWindow.xaml.cs:1080` (`{ "PinID", new AttributeValue { S = PatternInfo.PinId } }`). Cross-stitch's `export-design-pin-map.ts` reads this back under six historical names — see separate Pin-ID contract.
- **`AlbumBoards.csv`** — consumed by `PinterestHelper` and `PinterestBoardRenamer` within Uploader.

External services:

- Pinterest REST API v5 (HTTPS).
- Pinterest OAuth (browser + loopback HTTP listener on `localhost:8080`).

No Anthropic / LLM dependency exists in the Uploader's pin/board flow — captions are deterministically generated from `PatternInfo` fields (the planning doc mentions caption-AI ambitions, but there is no code path invoking an LLM in the current Pinterest helpers).

App.config settings consumed:

- `PinterestClientId` (line 9), `PinterestClientSecret` (in `App.private.config`), `PinterestRedirectUri` (line 13), `PinterestScope` (line 14), `PinterestBoardId` (line 15 — fallback board), `PinterestLinkUrl` (line 16 — alternate site base), `PinterestBoardsCsvPath` (not set in repo, defaults to `AlbumBoards.csv`), `PinterestAuthUrl`/`PinterestTokenUrl`/`PinterestPinsUrl` (lines 10-12, declared but most code uses hard-coded constants — drift risk).

---

## 9. Error Handling

Observed in code:

- **Pin create failure** — `PinterestHelper.UploadPinForPatternAsync` throws `Exception` on non-2xx with status + body (`d:/ann/Git/Uploader/Uploader/Helpers/PinterestHelper.cs:136-140`); throws again if response JSON has no `id` (`PinterestHelper.cs:143-146`). No retry, no partial-state cleanup (a created-but-unreported pin would leak).
- **Board create failure** — `PinterestBoardCreator.CreateBoardForAlbumAsync` throws on non-2xx (`PinterestBoardCreator.cs:190-191`) and on missing `id` (`PinterestBoardCreator.cs:194-195`). The CSV write only happens **after** the foreach (`PinterestBoardCreator.cs:103`), so a mid-loop throw leaves boards created on Pinterest without a CSV row → orphan.
- **Board rename failure** — `PinterestBoardRenamer.RenameBoardAsync` throws on non-2xx (`PinterestBoardRenamer.cs:251-254`). One failure aborts the loop; preceding renames are not rolled back.
- **Missing board mapping** — `GetBoardIdForAlbumAsync` returns CSV value, else `appSettings["PinterestBoardId"]`, else `InvalidOperationException("Board for album {id} not found in '{csv}', and no PinterestBoardId fallback is configured.")` (`PinterestHelper.cs:175-182`).
- **CSV missing entirely** — `LoadBoardsMappingAsync` returns empty map silently (`PinterestHelper.cs:195-199`), forcing fallback path.
- **CSV malformed line** — `TryParseAlbumBoardsCsvLine` returns `false`; the line is skipped without warning (`PinterestHelper.cs:213-214`). `PinterestBoardRenamer.cs:67-68` is louder — `progress?.Report("Skipping invalid CSV line …")`.
- **Token file missing or unreadable** — `PinterestTokenStore.LoadAsync` returns `null` (swallows all exceptions) (`PinterestTokenInfo.cs:99-115`). `GetValidAccessTokenAsync` then calls `RefreshAccessTokenAsync` which throws `"No refresh token available."` because there is nothing to refresh (`PinterestTokenInfo.cs:198-200`).
- **Token expired** — `IsExpired` returns true within a 60-second safety margin (`PinterestTokenInfo.cs:39-41`); `GetValidAccessTokenAsync` auto-refreshes (`PinterestTokenInfo.cs:241-249`).
- **Refresh failure** — surfaces as `Exception("Pinterest OAuth token refresh failed: {status} - {body}")` (`PinterestTokenInfo.cs:222-223`). Pinterest's refresh response with empty `refresh_token` is handled by carrying over the old one (`PinterestTokenInfo.cs:228-230`).
- **OAuth code exchange failure** — throws `"Pinterest OAuth token exchange failed: …"` (`PinterestTokenInfo.cs:181-182`).
- **Reader 401/403** — `test-pinterest-ads.ts:108-121` prints actionable hints (re-authorize for 401, missing scope for 403); `pinterestClient.ts:43` throws raw error otherwise.
- **Reader 429 rate limit** — `pinterestClient.ts:29-41` retries up to 6 times with exponential backoff, capped at 30s, honouring `Retry-After`.
- **Reader missing access_token** — `readPinterestToken.ts:28-30` throws `"Pinterest token file at … has no access_token"`.

Silent / no-warning cases (worth flagging in monitoring):

- CSV missing → fallback board for **all** albums (`PinterestHelper.cs:195-199`).
- Token file corrupted → caught and treated as missing (`PinterestTokenInfo.cs:110-114`).

---

## 10. Security & Compliance

- **OAuth client secret on disk**. `PinterestClientSecret` lives in `App.private.config` (gitignored sibling of `App.config`), read at `d:/ann/Git/Uploader/Uploader/Helpers/PinterestTokenInfo.cs:144`. The example template `d:/ann/Git/Uploader/Uploader/App.private.config.example:5` ships in the repo. Risk: leaking `App.private.config` leaks the secret.
- **Refresh + access token on disk**. `pinterest_tokens.json` at `Uploader/secrets/pinterest_tokens.json` is plain JSON with `Formatting.Indented` (`PinterestTokenInfo.cs:121`). No encryption, no ACL hardening, no DPAPI. Both repos read the same file. Risk: anyone with filesystem access (or accidental git-commit of `secrets/`) gets a refresh token that survives until manually revoked.
- **Loopback OAuth redirect**. `http://localhost:8080/callback` (`App.config:13`) — fine for a desktop app, but the `HttpListener` is started **before** the browser opens (`MainWindow.xaml.cs:313-318`), and the listener prefix is derived from the redirect URI's port (`MainWindow.xaml.cs:310`). If another local process binds the port first, `listener.Start()` will throw.
- **Authorization code in URL query**. Code is read from `ctx.Request.QueryString["code"]` (`MainWindow.xaml.cs:334`) — standard, but the URL appears in browser history.
- **HTTPS everywhere on Pinterest side** — all v5 endpoints are HTTPS.
- **Compliance scope**: tokens grant `ads:read` which can expose advertising spend data — flag for any future multi-user deployment.

Required mitigation if the token file is ever moved out of the operator's home dir:

- Encrypt at rest (DPAPI on Windows is the obvious fit, since the Uploader is WPF/Windows-only).
- Add `secrets/` to `.gitignore` at the repo root (confirm — see `d:/ann/Git/Uploader/.gitignore`).

---

## 11. Testing & Validation

Existing smoke tests:

- `d:/ann/Git/cross-stitch/automation/pinterest-agent/scripts/test-pinterest-api.ts` — verifies `ads:read`, `boards:read`, `pins:read` by listing items (lines 4-26). Run: `npx tsx scripts/test-pinterest-api.ts` from `automation/pinterest-agent/`.
- `d:/ann/Git/Uploader/scripts/test-pinterest-ads.ts` — same `GET /ad_accounts` probe but written against the Uploader's token-store path layout. **Duplicate** of `d:/ann/Git/cross-stitch/automation/pinterest-agent/scripts/test-pinterest-ads.ts` per architectureFacts. Run: `npx tsx scripts/test-pinterest-ads.ts` from repo root.
- `d:/ann/Git/cross-stitch/automation/pinterest-agent/scripts/test-pinterest-ad-report.ts` (exists, not inspected here).

Recommended additional checks:

- **Cross-repo scope drift**: `grep -RE "pins:(read|write)|boards:(read|write)|ads:read"` in both repos and confirm the same five tokens appear in `App.config:14`, `test-pinterest-api.ts`, and the planning markdown.
- **Token-shape drift**: `grep -nE "access_token|refresh_token|expires_at_utc|token_type|\"scope\""` in `PinterestTokenInfo.cs` and `readPinterestToken.ts` to confirm field-for-field parity.
- **CSV header drift**: `grep -n "AlbumID,AlbumCaption,BoardID"` should match in `PinterestBoardCreator.cs:84`, the actual `AlbumBoards.csv:1`, and any parser comment headers.
- **API base URL drift**: `grep -nE "api\.pinterest\.com/v5"` in both repos — currently 5 distinct files, easy to miss when bumping major version.
- **Board-name length**: assert every row in `AlbumBoards.csv` produces a `BuildSeoBoardName` output `< 50` chars before invoking `PinterestBoardRenamer`.
- **OAuth round-trip**: run Uploader's "Pinterest re-auth" button (`BtnPinterestReAuth_Click`, `MainWindow.xaml.cs:288-`) end-to-end on a fresh machine to confirm `pinterest_tokens.json` is created and immediately readable by `readPinterestAccessToken`.

---

## 12. References

Source code (writer side — Uploader):

- `d:/ann/Git/Uploader/Uploader/Helpers/PinterestHelper.cs` (pin payload, title/description/alt-text templates, theme table, CSV reader, fallback board)
- `d:/ann/Git/Uploader/Uploader/Helpers/PinterestBoardCreator.cs` (board POST, CSV writer)
- `d:/ann/Git/Uploader/Uploader/Helpers/PinterestBoardRenamer.cs` (board PATCH, SEO name rule)
- `d:/ann/Git/Uploader/Uploader/Helpers/PinterestTokenInfo.cs` (token DTOs, OAuth client, refresh logic)
- `d:/ann/Git/Uploader/Uploader/Helpers/PlatformConfig.cs` (token-path resolution)
- `d:/ann/Git/Uploader/Uploader/Helpers/PatternLinkHelper.cs` (pin `link` and `media_source.url`)
- `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:288-` (`BtnPinterestReAuth_Click` OAuth flow)
- `d:/ann/Git/Uploader/Uploader/App.config` (PinterestClientId, PinterestScope, redirect URI, fallback board ID, link URL)
- `d:/ann/Git/Uploader/Uploader/App.private.config.example` (secret placeholder)
- `d:/ann/Git/Uploader/Uploader/AlbumBoards.csv` (mapping data)
- `d:/ann/Git/Uploader/scripts/test-pinterest-ads.ts` (Ads API smoke test)

Source code (reader side — cross-stitch pinterest-agent):

- `d:/ann/Git/cross-stitch/automation/pinterest-agent/src/services/readPinterestToken.ts` (token reader)
- `d:/ann/Git/cross-stitch/automation/pinterest-agent/src/services/readPlatformConfig.ts` (path resolver)
- `d:/ann/Git/cross-stitch/automation/pinterest-agent/src/services/pinterestClient.ts` (v5 GET wrapper, 429 backoff)
- `d:/ann/Git/cross-stitch/automation/pinterest-agent/scripts/test-pinterest-api.ts` (scope verification)

Configuration & docs:

- `d:/ann/Git/cross-stitch-platform-docs/platform-config.json` (shared `pinterestTokenPath`)
- `d:/ann/Git/cross-stitch-platform-docs/plan/cross-stitch/Pinterest AI Agent — API Integrations.md` (scope list, OAuth architecture context)

Related contracts (split out separately):

- "AlbumID 4-digit zero-padded" (`PinterestHelper.cs:161`, `AlbumBoards.csv`)
- "Design page URL `/{Caption}-{AlbumID}-{NPage-1}-Free-Design.aspx`" (`PatternLinkHelper.cs:62-71`)
- "S3 photo path `photos/{AlbumID}/{DesignID}/<file>.jpg`" (`PatternLinkHelper.cs:73-76`)
- "Pinterest pin-ID DynamoDB attribute drift" (`MainWindow.xaml.cs:1080`, `export-design-pin-map.ts`)
- "Pinterest token file path" (`PlatformConfig.cs`, `readPlatformConfig.ts`)
