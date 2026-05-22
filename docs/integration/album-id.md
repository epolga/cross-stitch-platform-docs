# Contract: AlbumID

> Status: implicit cross-repo contract — no shared definition file, no version field. Identifier is generated in the WPF Uploader (Olga/epolga) and consumed by `cross-stitch` (Next.js site) and the `pinterest-agent` automation. Encoded in three distinct shapes: raw integer, 4-digit zero-padded string (`D4`), and as an `ALB#{D4}` DynamoDB partition key.

---

## 1. Contract Name

**AlbumID — cross-repo identifier for design albums.**

Also referred to in code as `AlbumId` (C# property casing) and `AlbumID` (DynamoDB attribute / TypeScript type field). The same semantic identifier travels under all three names.

Canonical type seeds:
- `int AlbumId` on `PatternInfo` and helpers — `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:90` (`private int _albumId;`) and `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:1698` (`public int AlbumId { get; }`).
- `albumId: number` on the TypeScript side — `d:/ann/Git/cross-stitch/automation/pinterest-agent/scripts/export-design-pin-map.ts:23` (`albumId: number;` on `DesignPinRecord`).

---

## 2. Purpose

`AlbumID` groups one-to-many cross-stitch designs into a themed album (for example "Animals", "Christmas", "Cushion Covers" — see the `AlbumCaption` column at `d:/ann/Git/Uploader/Uploader/AlbumBoards.csv:2-4`).

It is the join key that lets four otherwise-decoupled systems talk to each other:

1. **WPF Uploader** writes it. Source of truth at `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:598-624` (`LoadAlbumIdFromTxt`) — the integer is read from the filename of the single `.txt` file in the batch folder.
2. **DynamoDB `CrossStitchItems`** stores it as the `AlbumID` numeric attribute on every `EntityType=DESIGN` and `EntityType=ALBUM` item, and additionally encoded inside the partition key `ID = "ALB#{albumId:D4}"` (`d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:107`, `:1067-1069`).
3. **S3 object keys** use it as a path segment under both `photos/{AlbumID}/{DesignID}/…` and `pdfs/{AlbumID}/…` (`d:/ann/Git/Uploader/Uploader/Helpers/PatternLinkHelper.cs:75`, `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:957-961`).
4. **Public website URLs** embed it as the second-to-last hyphen-delimited segment of the legacy `.aspx` slug — written in C# at `d:/ann/Git/Uploader/Uploader/Helpers/PatternLinkHelper.cs:70`, mirrored in TS at `d:/ann/Git/cross-stitch/src/lib/url-helper.ts:58`, triplicated again in `d:/ann/Git/cross-stitch/automation/pinterest-agent/scripts/export-design-pin-map.ts:55-58`.
5. **Uploader-local `AlbumBoards.csv`** maps each `AlbumID` (4-digit string) to a Pinterest BoardID for pin destination routing (`d:/ann/Git/Uploader/Uploader/Helpers/PinterestHelper.cs:159-183`).

---

## 3. Scope

**In scope:**
- A non-zero positive integer identifying a design album.
- Allowed shapes the contract recognises:
  - Raw integer (`int`/`number`) — primary form across most code paths.
  - 4-digit zero-padded string (`albumId.ToString("D4")`) — used only inside the Uploader as the CSV lookup key and as the suffix of the DDB partition key `ALB#{D4}`. Examples: `0007`, `0014`, `0037`, `0104`, `0128` (`d:/ann/Git/Uploader/Uploader/AlbumBoards.csv:1-30`).
  - URL slug segment — embedded as a plain decimal between the caption slug and `NPage-1` (`d:/ann/Git/Uploader/Uploader/Helpers/PatternLinkHelper.cs:70`).
- Travels via: DDB attributes, S3 key prefixes, CSV rows, URL slugs, the Uploader's `*.txt` filename convention.

**Out of scope (not an AlbumID):**
- `DesignID` — sibling integer with different semantics (one design belongs to one album); see `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:872`.
- The string `"ALB#0037"` itself — that is the DDB partition key *containing* an AlbumID, not the AlbumID (`d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:107`, `:1123-1126` strips the `ALB#` prefix to recover the bare value).
- `BoardID` (Pinterest) — an opaque numeric string controlled by Pinterest; mapped *from* AlbumID, not equal to it (`d:/ann/Git/Uploader/Uploader/AlbumBoards.csv:1`).
- The album's `Caption` (e.g. `"Animals"`) — human-readable label resolved separately via `getAlbumIdByCaption` (`d:/ann/Git/cross-stitch/src/app/[slug]/page.tsx:70`).

---

## 4. Data Formats

### 4.1 Raw integer (canonical wire form)

This is the form that appears in DynamoDB, S3 paths, URL slugs, and most code:

- C# property: `public int AlbumId { get; set; }` on `PatternInfo` (set at `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:575`: `patternInfo.AlbumId = LoadAlbumIdFromTxt();`).
- DynamoDB write — numeric attribute:
  ```csharp
  { "AlbumID", new AttributeValue { N = _albumId.ToString() } },
  ```
  `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:1069`.
- DynamoDB read in TS (note the `.N` numeric branch):
  ```ts
  const albumId = readNumber(item.AlbumID);
  ```
  `d:/ann/Git/cross-stitch/automation/pinterest-agent/scripts/export-design-pin-map.ts:87`, `:106` (where `readNumber` calls `parseInt(v.N, 10)` at `:36-40`).
- URL slug:
  ```ts
  return `/${formattedCaption}-${Design.AlbumID}-${Design.NPage-1}-Free-Design.aspx`;
  ```
  `d:/ann/Git/cross-stitch/src/lib/url-helper.ts:58` (no padding — interpolated as the bare integer).
- S3 photo path uses the raw integer too:
  ```csharp
  return $"{_imageBaseUrl}/{_photoPrefix}/{albumId}/{designId}/{photoFileName}";
  ```
  `d:/ann/Git/Uploader/Uploader/Helpers/PatternLinkHelper.cs:75`.
- S3 PDF path likewise:
  ```csharp
  string mainKey = $"pdfs/{_albumId}/Stitch{designId}_Kit.pdf";
  ```
  `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:957`.

### 4.2 4-digit zero-padded string (Uploader-internal only)

```csharp
string albumKey = albumId.ToString("D4", CultureInfo.InvariantCulture);
```
`d:/ann/Git/Uploader/Uploader/Helpers/PinterestHelper.cs:161` — used as the dictionary key when looking up `AlbumBoards.csv` rows.

The CSV itself stores AlbumID pre-padded:
```
AlbumID,AlbumCaption,BoardID
0104,"Cushion Covers",257127528664615685
0096,"Taiwan",257127528664615686
0037,"Animals",257127528664615687
```
`d:/ann/Git/Uploader/Uploader/AlbumBoards.csv:1-4`.

Same `D4` form is the suffix of the DDB partition key:
```csharp
AlbumPartitionKey = $"ALB#{albumId:D4}";
```
`d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:107`.

And it appears in the Pinterest pin description text (presentation-only):
```csharp
sb.AppendFormat(CultureInfo.InvariantCulture,
    "From album {0:D4}. Download printable PDF and see more details at {1}. ",
    albumId, patternUrl);
```
`d:/ann/Git/Uploader/Uploader/Helpers/PinterestHelper.cs:517-521`.

### 4.3 Regex / parser used by cross-stitch

`cross-stitch` recovers AlbumID from the public URL slug by splitting on `-` and `parseInt` on the 4th-from-last segment:
```ts
const albumId = parseInt(parts[parts.length - 4], 10);
```
`d:/ann/Git/cross-stitch/src/app/[slug]/page.tsx:19` (and `isNaN(albumId)` check at `:21`).

A compatible validation regex (not asserted in code today) would be `^[A-Za-z0-9-]+-(\d+)-\d+-Free-Design\.aspx$` against the slug, capture group 1 = AlbumID.

---

## 5. API Endpoints / Interfaces

**No HTTP API.** AlbumID is never the path/body of an HTTP route owned by either repo. It is exchanged exclusively through five file-/DB-based interfaces:

| # | Interface | Direction | Encoding | Producer cite | Consumer cite |
|---|---|---|---|---|---|
| 1 | DynamoDB `CrossStitchItems`, attribute `AlbumID` (`N`) | Uploader → cross-stitch | raw integer | `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:1069` | `d:/ann/Git/cross-stitch/automation/pinterest-agent/scripts/export-design-pin-map.ts:87,106` |
| 2 | DynamoDB `CrossStitchItems`, partition key `ID = "ALB#{albumId:D4}"` | Uploader → cross-stitch | `ALB#` + D4 string | `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:107`, `:1067` | parsed back at `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:1123-1126` (`id.Substring(4)`); cross-stitch reads via `getAlbumIdByCaption` (`d:/ann/Git/cross-stitch/src/app/[slug]/page.tsx:70`) |
| 3 | S3 key `photos/{AlbumID}/{DesignID}/<file>.jpg` | Uploader → cross-stitch (via CloudFront) | raw integer | `d:/ann/Git/Uploader/Uploader/Helpers/PatternLinkHelper.cs:75` | (read indirectly by cross-stitch image URL builders; see architecture summary) |
| 4 | S3 keys `pdfs/{AlbumID}/Stitch{DesignID}_Kit.pdf` and `pdfs/{AlbumID}/{DesignID}/Stitch{DesignID}_{N}_Kit.pdf` | Uploader → cross-stitch | raw integer | `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:957-961`, `:3653,3656` | cross-stitch site PDF link builder |
| 5 | Public URL slug `/{Caption}-{AlbumID}-{NPage-1}-Free-Design.aspx` | Uploader (writer) → cross-stitch (parser) | raw integer | `d:/ann/Git/Uploader/Uploader/Helpers/PatternLinkHelper.cs:70`; mirrored TS `d:/ann/Git/cross-stitch/src/lib/url-helper.ts:58`; re-mirrored at `d:/ann/Git/cross-stitch/automation/pinterest-agent/scripts/export-design-pin-map.ts:55-58` | `d:/ann/Git/cross-stitch/src/app/[slug]/page.tsx:13-25` |
| 6 | Uploader-local CSV `AlbumBoards.csv`, column 1 = AlbumID | Uploader → Uploader | 4-digit padded string | written by `PinterestBoardCreator` (per `d:/ann/Git/Uploader/Uploader/Helpers/PinterestHelper.cs:45-46` comment) | `d:/ann/Git/Uploader/Uploader/Helpers/PinterestHelper.cs:159-220` |
| 7 | Uploader batch folder convention: `*.txt` filename = AlbumID | human/operator → Uploader | raw integer in filename | n/a (operator-supplied) | `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:598-624` |

---

## 6. Versioning

**Unversioned (implicit).** The identifier has no version field; the same `AlbumID=37` (`"Animals"` per `d:/ann/Git/Uploader/Uploader/AlbumBoards.csv:4`) has been carried unchanged across DDB, S3, URLs, and CSVs since the schema was established.

**Recommendation:** treat the **raw integer** as the canonical form and the **4-digit zero-padded string (`D4`)** as a presentation/sort-only artefact. The `D4` choice is undocumented and brittle:

- `D4` will silently overflow once any album crosses ID `9999`. The current max observed in the live CSV is `0128` (`d:/ann/Git/Uploader/Uploader/AlbumBoards.csv:5`), so there is headroom, but a future `D5` migration would have to be coordinated across `MainWindow.xaml.cs:107`, `PinterestHelper.cs:161`, and the CSV file itself.
- Any new consumer of the partition key should parse `ID.Substring(4)` and then `int.Parse` — never string-compare `ALB#0037` to a synthetic `ALB#37`.

A pragmatic next step: introduce a `SCHEMA_VERSION` attribute on the album item in `CrossStitchItems` so future format migrations (e.g. wider zero-padding, GUID overlay) are detectable.

---

## 7. Ownership & Contacts

- **Maintainer:** Olga (epolga) — olga.epstein@gmail.com.
- **Code owner (writer):** the WPF Uploader. Allocation happens at `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:598-624` (`LoadAlbumIdFromTxt`); persistence at `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:1067-1069`. The Uploader is therefore the authoritative source of any new AlbumID values.
- **Code consumers (readers):**
  - `cross-stitch` Next.js app — `d:/ann/Git/cross-stitch/src/app/[slug]/page.tsx`, `d:/ann/Git/cross-stitch/src/lib/url-helper.ts`.
  - `cross-stitch/automation/pinterest-agent` — `d:/ann/Git/cross-stitch/automation/pinterest-agent/scripts/export-design-pin-map.ts`.

---

## 8. Dependencies

AlbumID is coupled to (and never meaningful without) these neighbours:

1. **DesignID** — pair `(AlbumID, DesignID)` uniquely names a design and is required jointly by every S3 photo/PDF key (`d:/ann/Git/Uploader/Uploader/Helpers/PatternLinkHelper.cs:75`; `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:957-961`) and by the email/pin link builders (`d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:1237`).
2. **`NPage`** — needed in addition to AlbumID to look up a specific design by URL via `getDesignIdByAlbumAndPage(parsed.albumId, parsed.nPage)` (`d:/ann/Git/cross-stitch/src/app/[slug]/page.tsx:43`).
3. **S3 bucket + CloudFront path conventions** — the bucket name defaults to `cross-stitch-designs` (`d:/ann/Git/Uploader/Uploader/Helpers/PatternLinkHelper.cs:34`); the prefix `photos` is also configurable (`PatternLinkHelper.cs:40-41`).
4. **DDB `CrossStitchItems`** schema — single-table `(ID, NPage)` with `EntityType` discriminator (`d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:1065-1073`).
5. **Pinterest BoardID mapping** — entirely dependent on the local CSV (`d:/ann/Git/Uploader/Uploader/Helpers/PinterestHelper.cs:159-220`); if the CSV is missing the call falls back to the configured `PinterestBoardId` app-setting (`PinterestHelper.cs:175-178`).
6. **Caption slugification** — site URLs depend on `Caption.replace(/\s+/g, '-')` (`d:/ann/Git/cross-stitch/src/lib/url-helper.ts:57`) being consistent with C# `Title.Replace(' ', '-')` (`d:/ann/Git/Uploader/Uploader/Helpers/PatternLinkHelper.cs:66`). Both flatten only space characters — punctuation drift between them would break round-trip parsing.

---

## 9. Error Handling

Observed failure modes from real code paths (not theoretical):

- **Missing/extra `.txt` in batch folder** — `LoadAlbumIdFromTxt` shows a `MessageBox` and returns `0`; `CreatePatternInfoAsync` then throws (`d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:604-608`, `:576-579`):
  ```csharp
  MessageBox.Show("Exactly one .txt file expected for AlbumID.", ...);
  return 0;
  ...
  if (patternInfo.AlbumId == 0) throw new Exception("Failed to load AlbumID from .txt file.");
  ```
- **Non-numeric `.txt` filename** — same path, second guard at `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:611-617`: `"Invalid AlbumID in .txt file."` and `return 0`.
- **AlbumID not present in `AlbumBoards.csv`** — falls back to configured `PinterestBoardId`; if that is also missing, throws (`d:/ann/Git/Uploader/Uploader/Helpers/PinterestHelper.cs:175-182`):
  ```csharp
  throw new InvalidOperationException(
      $"Board for album {albumId} not found in '{_boardsCsvPath}', " +
      "and no PinterestBoardId fallback is configured.");
  ```
- **`AlbumBoards.csv` missing on disk** — silently returns an empty map (`d:/ann/Git/Uploader/Uploader/Helpers/PinterestHelper.cs:195-199`), which then defers to the default-board fallback above.
- **Malformed URL slug at the cross-stitch parser** — `parseSlugForDesign` returns `null` if either `albumId` or `nPage` is `NaN` (`d:/ann/Git/cross-stitch/src/app/[slug]/page.tsx:21-23`), which causes `GetDesignPageFromSlug` to call `notFound()` (`:50-52`). Net effect: 404 (no silent partial render).
- **DDB lookup returns no item** — `getDesignIdFromSlug` returns `null` and the page calls `notFound()` (`d:/ann/Git/cross-stitch/src/app/[slug]/page.tsx:48-52`). Likewise `GetAlbumDesignsPageFromSlug` 404s when `getAlbumIdByCaption` returns null (`:73-75`).
- **Album-list scan fails** — caught and logged into `txtStatus`; the function returns an empty list and downstream "recent albums" features silently degrade (`d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:1139-1146`).
- **Partial-write risk** — the upload flow writes S3 PDFs, then DDB, then Pinterest (`MainWindow.xaml.cs:957-970`, `:1065-1079`). A crash between steps leaves orphan S3 objects under `pdfs/{AlbumID}/…` and no DDB row; nothing reconciles them automatically (the orphan-detector at `:3645-3658` is a manual report, not a cleanup).

---

## 10. Security & Compliance

- **Public identifier.** AlbumID is embedded in publicly indexable URLs (e.g. `https://cross-stitch.com/Animals-37-…-Free-Design.aspx` per `d:/ann/Git/cross-stitch/src/lib/url-helper.ts:58`) and in publicly fetchable S3/CloudFront paths (`photos/{AlbumID}/{DesignID}/4.jpg` — `d:/ann/Git/Uploader/Uploader/Helpers/PatternLinkHelper.cs:75`). There is no expectation of secrecy.
- **No PII.** AlbumID is a small, monotonically-issued integer; it does not encode user identity, location, or any personal data.
- **No authorization gating in code paths reviewed** — neither the DDB read nor the URL parser checks any auth state for AlbumID-based lookups (`d:/ann/Git/cross-stitch/src/app/[slug]/page.tsx:37-79`).
- **No secrets on disk** related to this contract. (Pinterest tokens are a separate concern — see the Pinterest token contract.)

---

## 11. Testing & Validation

Concrete checks Olga can run today:

1. **Cross-repo presence grep.** Confirm both shapes are still in use:
   ```
   rg -nE '\.AlbumI[Dd]\b' d:/ann/Git/Uploader d:/ann/Git/cross-stitch
   rg -nE 'AlbumI[Dd]"\s*[,)]?\s*$|AlbumI[Dd]\s*:\s*number' d:/ann/Git/cross-stitch
   rg -nE 'ToString\("D4"' d:/ann/Git/Uploader
   ```
   Expect hits in `PinterestHelper.cs`, `MainWindow.xaml.cs`, `PatternLinkHelper.cs`, `url-helper.ts`, `[slug]/page.tsx`, `export-design-pin-map.ts`.
2. **Slug round-trip.** Build a URL with C# `PatternLinkHelper.BuildPatternUrl` (`d:/ann/Git/Uploader/Uploader/Helpers/PatternLinkHelper.cs:62-71`), then feed the slug into `parseSlugForDesign` (`d:/ann/Git/cross-stitch/src/app/[slug]/page.tsx:13-25`) — the recovered `albumId` must equal the original integer.
3. **D4 / DDB partition key consistency.** Assert that `_albumId.ToString("D4")` (`MainWindow.xaml.cs:1149`) and `AlbumPartitionKey.Substring(4)` (`:1126`) round-trip to the same integer.
4. **CSV smoke test.** Every line of `AlbumBoards.csv` must match `^\d{4},"[^"]+",\d+$` — `rg -nE '^\d{4},"[^"]+",\d+$' d:/ann/Git/Uploader/Uploader/AlbumBoards.csv` should return all rows except the header.
5. **DDB attribute presence.** Run the existing `export-design-pin-map.ts` against the live table and confirm `unknownAlbum === 0` (`d:/ann/Git/cross-stitch/automation/pinterest-agent/scripts/export-design-pin-map.ts:99,117,132`) — non-zero means an ALBUM row is missing for some DESIGN row's `AlbumID`.
6. **S3-vs-DDB drift.** Run the existing missing-PDF report (`d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:3625-3658`) — any non-empty `DesignToAlbumMap.csv` output flags AlbumID-keyed orphans.

---

## 12. References

**Authoritative writers (Uploader):**
- `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:85-108` — `_albumId`, `AlbumPartitionKey`, `SetAlbumInfo`.
- `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:570-624` — `CreatePatternInfoAsync`, `LoadAlbumIdFromTxt` (single source-of-truth for new AlbumIDs).
- `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:1065-1079` — DDB write of `AlbumID` numeric attribute.
- `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:957-961` — S3 PDF key construction.
- `d:/ann/Git/Uploader/Uploader/Helpers/PatternLinkHelper.cs:62-95` — URL and photo-URL builders (`BuildPatternUrl`, `BuildImageUrl`, `BuildAlbumUrl`).
- `d:/ann/Git/Uploader/Uploader/Helpers/PinterestHelper.cs:155-221` — D4 padding for CSV key lookup.
- `d:/ann/Git/Uploader/Uploader/AlbumBoards.csv` — the only on-disk artefact that uses the 4-digit padded form.

**Readers (cross-stitch):**
- `d:/ann/Git/cross-stitch/src/lib/url-helper.ts:53-76` — `CreateDesignUrl`, `CreateAlbumUrl` (mirror of the C# builders).
- `d:/ann/Git/cross-stitch/src/app/[slug]/page.tsx:13-79` — slug parser and DDB lookup.
- `d:/ann/Git/cross-stitch/automation/pinterest-agent/scripts/export-design-pin-map.ts:23,55-58,87,106,121,125` — TypeScript consumer that reads AlbumID from DDB and rebuilds the slug.

**Architecture / docs:**
- Architecture summary (this run's input — `d:/ann/Git/Uploader/.a5c/runs/01KS7AGP936JGQ989X3GHEQV1T/tasks/01KS7ANCGJJWQFMP444WVB2YEK/task.json`, see `prompt.context.architectureFacts.albumId`).
- `d:/ann/Git/cross-stitch-platform-docs/plan/cross-stitch/Pinterest AI Agent — WPF Uploader Integration.md` (referenced in the architecture-facts evidence list for the slug contract).
- `d:/ann/Git/Uploader/CLAUDE.md` and `d:/ann/Git/cross-stitch/CLAUDE.md` — repo-level developer notes for the writer and reader respectively.
