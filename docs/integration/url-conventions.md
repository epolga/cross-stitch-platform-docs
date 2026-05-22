# Contract: URL Conventions (design, album, share, tracking)

## 1. Contract Name

`url-conventions` — canonical URL patterns for design pages, album pages, share/tracking URLs, and the slug normalization rules used across the cross-stitch platform.

This contract is classified as **naming-convention** (one of five contract types catalogued in the platform contracts inventory) and is **triplicated-observed**: the same URL pattern is implemented in three separate code locations that must change together (see §12).

## 2. Purpose

Define the canonical, machine-buildable and machine-parseable URL shapes for the public site `https://cross-stitch.com`, so that:

- The Uploader (C#/WPF) and the cross-stitch web app (Next.js) generate URLs that resolve to the same pages.
- Newsletter emails and Pinterest pin descriptions written by the Uploader can be parsed and routed by the cross-stitch App Router without coordination beyond this document.
- Tracking parameters (`cid`, `eid`, UTM) attached by the Uploader are recognised — but not required — by the cross-stitch slug handler.
- The historical `.aspx` extension is preserved as a stable, SEO-bearing route segment even though no ASP.NET runtime is involved.

## 3. Scope

**In scope**

- The canonical design URL shape `/{Caption}-{AlbumID}-{NPage-1}-Free-Design.aspx`.
- The canonical album URL shape `/Free-{CaptionSlug}-Charts.aspx`.
- The alternate, internal design route `/designs/[designId]`.
- The albums-index route `/XStitch-Charts.aspx`.
- The unsubscribe URL `https://cross-stitch.com/unsubscribe?token=<UnsubscribeToken>`.
- Slug normalization (whitespace-to-dash, album-side letter/digit filtering + Title-Case).
- Site base URL resolution and the legacy host redirect.
- Email/Pinterest tracking query parameters (`cid`, `eid`, `utm_*`).

**Out of scope (covered by other contracts)**

- DynamoDB attribute names and types for `Caption`, `AlbumID`, `NPage`, `DesignID` (see `design-id`, `album-id`, `dynamodb-schema`).
- S3 image and PDF object keys (see `s3-paths`, `pdf-structure`).
- Pinterest pin payload shape (see `pinterest-metadata`).
- Unsubscribe-token generation algorithm; this contract only documents the URL shape that carries it.

## 4. Data Formats

### 4.1 Canonical design URL — triplicated

Shape: `/{Caption-with-spaces-as-dashes}-{AlbumID}-{NPage-1}-Free-Design.aspx`

Example: `/Lion-37-114-Free-Design.aspx` (where `Caption="Lion"`, `AlbumID=37`, `NPage=115`).

**Implementation A — Uploader (C#), authoritative for outbound emails and Pinterest pins:**

```csharp
// Uploader/Helpers/PatternLinkHelper.cs:62-71
public string BuildPatternUrl(PatternInfo patternInfo)
{
    if (patternInfo == null) throw new ArgumentNullException(nameof(patternInfo));

    string caption = (patternInfo.Title ?? "Cross-stitch-pattern").Replace(' ', '-');
    int.TryParse(patternInfo.NPage, out int nPage);
    string baseUrl = _siteBaseUrl;

    return $"{baseUrl}/{caption}-{patternInfo.AlbumId}-{nPage-1}-Free-Design.aspx";
}
```

**Implementation B — cross-stitch web app (TypeScript), authoritative for on-site links, sitemap, and canonical metadata:**

```ts
// cross-stitch/src/lib/url-helper.ts:54-59
export function CreateDesignUrl(
 Design: Design
): string {
    const formattedCaption = Design.Caption.replace(/\s+/g, '-');
  return `/${formattedCaption}-${Design.AlbumID}-${Design.NPage-1}-Free-Design.aspx`;
}
```

**Implementation C — pinterest-agent script (TypeScript), reads DynamoDB directly:**

```ts
// cross-stitch/automation/pinterest-agent/scripts/export-design-pin-map.ts:54-58
// Mirrors CreateDesignUrl in src/lib/url-helper.ts
function buildDesignUrl(caption: string, albumId: number, nPage: number): string {
  const slug = caption.replace(/\s+/g, "-");
  return `/${slug}-${albumId}-${nPage - 1}-Free-Design.aspx`;
}
```

Observable differences between the three:

| Aspect | C# `BuildPatternUrl` | TS `CreateDesignUrl` | TS `buildDesignUrl` |
|---|---|---|---|
| Whitespace replacement | `Replace(' ', '-')` (single ASCII space only) | `.replace(/\s+/g, '-')` (all whitespace runs) | `.replace(/\s+/g, "-")` (all whitespace runs) |
| Title fallback when null | `"Cross-stitch-pattern"` | none (would throw on null Caption) | none |
| NPage parse | `int.TryParse` → 0 on failure, then `nPage-1 = -1` | trusts caller (number) | trusts caller (number) |
| Returns | absolute URL (`{baseUrl}/...`) | path only | path only |
| Caption case | preserved verbatim | preserved verbatim | preserved verbatim |

The C# variant prepends the site base URL; both TS variants return path-only and rely on `buildCanonicalUrl` (`cross-stitch/src/lib/url-helper.ts:31-37`) at call sites to attach `https://cross-stitch.com`.

### 4.2 Design URL parser (reader side)

The cross-stitch App Router matches the design path with a split-from-the-right parser:

```ts
// cross-stitch/src/app/[slug]/page.tsx:13-25
function parseSlugForDesign(slug: string): { caption: string; albumId: number; nPage: number } | null {
  const parts = slug.split('-');
  if (parts.length < 4 || parts[parts.length - 2].toLowerCase() !== 'free' || parts[parts.length - 1].toLowerCase() !== 'design.aspx') {
    return null;
  }
  const nPage = parseInt(parts[parts.length - 3], 10);
  const albumId = parseInt(parts[parts.length - 4], 10);
  const caption = parts.slice(0, parts.length - 4).join('-');
  if (isNaN(albumId) || isNaN(nPage)) {
    return null;
  }
  return { caption, albumId, nPage };
}
```

Routing rule: the slug check is case-insensitive on the trailing `-free-design.aspx` (`cross-stitch/src/app/[slug]/page.tsx:119, 183`), so URLs like `/Lion-37-114-Free-Design.aspx` and `/lion-37-114-free-design.aspx` both render.

Note the parser does **not** reconstruct the Caption from DynamoDB; it resolves `(albumId, nPage) → designId` via `getDesignIdByAlbumAndPage` (`cross-stitch/src/app/[slug]/page.tsx:37-45`) and then renders `DesignPage` with the numeric `designId`. The Caption portion of the slug is therefore decorative for SEO and is not validated.

### 4.3 Canonical album URL

Shape: `/Free-{CaptionSlug}-Charts.aspx`

Example: `/Free-Nature-Charts.aspx` (where `Caption="nature"` or `"Nature"`).

**Implementation A — Uploader (C#), with optional template override:**

```csharp
// Uploader/Helpers/PatternLinkHelper.cs:78-95
public string BuildAlbumUrl(string albumId, string? caption = null)
{
    if (string.IsNullOrWhiteSpace(albumId))
        throw new ArgumentException("AlbumId must be provided.", nameof(albumId));

    string template = ConfigurationManager.AppSettings["AlbumUrlTemplate"] ?? string.Empty;
    string baseUrl = _siteBaseUrl;
    string slug = BuildAlbumCaptionSlug(caption, albumId);

    if (!string.IsNullOrWhiteSpace(template))
    {
        return template
            .Replace("{AlbumId}", albumId)
            .Replace("{CaptionSlug}", slug);
    }

    return $"{baseUrl}/Free-{slug}-Charts.aspx";
}
```

`AlbumUrlTemplate` is **not present** in `Uploader/Uploader/App.config` (App.config:1-28 lists no `AlbumUrlTemplate` key), so in current production builds the Uploader always uses the literal `/Free-{slug}-Charts.aspx` form. The setting is a future-facing override hook that must be added to `App.config` (or `App.private.config`) to take effect.

**Implementation B — cross-stitch (TypeScript):**

```ts
// cross-stitch/src/lib/url-helper.ts:71-76
export function CreateAlbumUrl(
  Caption:string)
  : string{
  const formattedCaption = Caption?.replace(/\s+/g, '-');
  return `/Free-${formattedCaption}-Charts.aspx`;
}
```

The cross-stitch implementation has **no template override**. If the Uploader ever sets `AlbumUrlTemplate` to something other than `{baseUrl}/Free-{CaptionSlug}-Charts.aspx`, the cross-stitch route parser (§4.4) and sitemap (`cross-stitch/src/app/sitemap.xml/route.ts:67`) will diverge.

Album-list-page link generation in the catalog uses the same shape inline (`cross-stitch/src/app/albums/page.tsx:56-58`):
```ts
const slug = album.Caption.replace(/\s+/g, '-');
return (
  <Link key={album.Caption} href={`/Free-${slug}-Charts.aspx`} ...
```

### 4.4 Album URL parser (reader side)

```ts
// cross-stitch/src/app/[slug]/page.tsx:27-34
async function getAlbumCaptionFromSlug(slug: string): Promise<string | null> {
 const parts = slug.split('-');
 if (parts.length < 3 || parts[0].toLowerCase() !== 'free' || parts[parts.length - 1].toLowerCase() !== 'charts.aspx') {
    return null;
  }
  const albumCaption = parts.slice(1, parts.length - 1).join(' ');
  return albumCaption;
}
```

Reverse normalization: the parser joins the inner parts with **spaces** (`.join(' ')`) before looking up the album by caption via `getAlbumIdByCaption` (`cross-stitch/src/app/[slug]/page.tsx:70`). This means the round-trip relies on:

1. Album `Caption` in DynamoDB containing **no characters that the Uploader's slug builder strips**, and
2. The case the DynamoDB caption-lookup uses being whichever case the Uploader/cross-stitch produced.

### 4.5 Slug rules

| Side | Function | Input → Output |
|---|---|---|
| Design (C#) | `BuildPatternUrl` | `"Running Horse"` → `"Running-Horse"` (single-space-to-dash only); preserves case; no special-char stripping |
| Design (TS) | `CreateDesignUrl` | `"Running\tHorse"` → `"Running-Horse"` (any whitespace run to single dash); preserves case; no special-char stripping |
| Design (TS, pinterest-agent) | `buildDesignUrl` | same as `CreateDesignUrl` |
| Album (C#) | `BuildAlbumCaptionSlug` | letter/digit-only token extractor, lowercases each token, Title-Cases first letter, joins with `-`; `"Sunny Day!"` → `"Sunny-Day"`; empty/punctuation-only → `"Album-{albumId}"` |
| Album (TS, builder) | `CreateAlbumUrl` | `.replace(/\s+/g, '-')`; **no** letter/digit filter, **no** case normalization |
| Album (TS, route) | `albums/page.tsx:56` | identical to `CreateAlbumUrl` |

Drift risk: an album titled `"Mother's Day"` is rendered by C# as `/Free-Mothers-Day-Charts.aspx` (apostrophe stripped) but by TS as `/Free-Mother's-Day-Charts.aspx` (apostrophe preserved, which then breaks the URL since `'` is reserved). Today the platform avoids this by convention only.

Album-side C# slug builder for reference:

```csharp
// Uploader/Helpers/PatternLinkHelper.cs:97-134
private static string BuildAlbumCaptionSlug(string? caption, string albumId)
{
    if (string.IsNullOrWhiteSpace(caption))
        return $"Album-{albumId}";

    var parts = new List<string>();
    var current = new StringBuilder();

    foreach (char c in caption)
    {
        if (char.IsLetterOrDigit(c))
        {
            current.Append(c);
        }
        else
        {
            if (current.Length > 0)
            {
                parts.Add(current.ToString());
                current.Clear();
            }
        }
    }

    if (current.Length > 0)
        parts.Add(current.ToString());

    if (parts.Count == 0)
        return $"Album-{albumId}";

    for (int i = 0; i < parts.Count; i++)
    {
        string word = parts[i].ToLowerInvariant();
        parts[i] = char.ToUpperInvariant(word[0]) + word.Substring(1);
    }

    return string.Join("-", parts);
}
```

### 4.6 Alternate canonical route `/designs/[designId]`

Shape: `/designs/{DesignID}` — uses the raw numeric `DesignID` and bypasses the slug entirely.

This is the route the `[slug]` parser actually delegates to after resolving `(albumId, nPage) → designId` (`cross-stitch/src/app/[slug]/page.tsx:47-57`). It is also referenced directly from on-site components (e.g., generated metadata at `cross-stitch/src/app/designs/[designId]/page.tsx:68-150`) and produces canonical URLs back through `CreateDesignUrl` (line 84 and line 192). The page fetches the design via the in-process API call `http://localhost:3000/api/designs/${designId}` (`cross-stitch/src/app/designs/[designId]/page.tsx:73, 160`) — a localhost URL hardcoded into a server component.

Behaviour: `/designs/123` does **not** redirect to the canonical slug form; both URLs render the same page but resolve canonical-link tags to the slug form via `buildCanonicalUrl(CreateDesignUrl(design))` (line 192). SEO-canonical-equivalence therefore depends on the canonical tag being honoured by crawlers.

### 4.7 Site base URL

Shape: `https://cross-stitch.com` (no trailing slash, no port, always HTTPS).

**Uploader resolution chain:**

```csharp
// Uploader/Helpers/PatternLinkHelper.cs:46-60
private static string GetSiteBaseUrlFromConfig()
{
    string siteBaseUrl =
        ConfigurationManager.AppSettings["SiteBaseUrl"] ??
        ConfigurationManager.AppSettings["PinterestLinkUrl"] ??
        string.Empty;

    if (string.IsNullOrWhiteSpace(siteBaseUrl))
    {
        throw new ConfigurationErrorsException(
            "SiteBaseUrl must be configured in appSettings.");
    }

    return siteBaseUrl.TrimEnd('/');
}
```

Configured values (`Uploader/Uploader/App.config:6, 16`):
- `SiteBaseUrl=https://cross-stitch.com`
- `PinterestLinkUrl=https://cross-stitch.com` (legacy fallback, still read first if `SiteBaseUrl` missing)

**cross-stitch resolution chain:**

```ts
// cross-stitch/src/lib/url-helper.ts:5-29
export function normalizeBaseUrl(input: string): string {
  let url = input.trim();
  // Force https
  url = url.replace(/^http:\/\//i, "https://");
  // Remove trailing slashes
  url = url.replace(/\/+$/, "");
  // Remove default HTTPS port
  url = url.replace(/:443$/i, "");
  // Safety net in production
  if (process.env.NODE_ENV === "production" &&
      url.includes("cross-stitch-pattern.net")) {
    url = "https://cross-stitch.com";
  }
  return url;
}

export function getSiteBaseUrl(): string {
  return normalizeBaseUrl(process.env.NEXT_PUBLIC_SITE_URL ?? 'https://cross-stitch.com');
}
```

Both sides land on the same default: `https://cross-stitch.com`.

### 4.8 Legacy-host redirect

`cross-stitch/src/middleware.ts:4-17` redirects `cross-stitch-pattern.net` and `www.cross-stitch-pattern.net` to `https://cross-stitch.com` with HTTP 308 (permanent, method-preserving). This is the only host normalisation; all other hosts (including localhost) pass through.

### 4.9 Tracking parameters

Three orthogonal tracking parameters are appended by the Uploader to every newsletter link:

```csharp
// Uploader/MainWindow.xaml.cs:2352-2389
private static string AppendTrackingParameters(string url, string? cid, string? eid)
{
    if (string.IsNullOrWhiteSpace(url))
        return url;

    var queryParts = new List<string>();
    if (!string.IsNullOrWhiteSpace(cid))
        queryParts.Add($"cid={Uri.EscapeDataString(cid)}");
    if (!string.IsNullOrWhiteSpace(eid))
        queryParts.Add($"eid={Uri.EscapeDataString(eid)}");

    string trackedUrl = queryParts.Count == 0
        ? url
        : AppendQueryParameters(url, queryParts);

    return AppendUtmParameters(trackedUrl);
}
```

| Param | Source | Meaning | Carries PII? |
|---|---|---|---|
| `cid` | DynamoDB user record (lowercase GUID per architecture facts) | user correlation id | **Yes — PII-adjacent**: identifies a specific user account |
| `eid` | per-email id (typically `yyMMdd` per `MainWindow.xaml.cs:2511`) or `"admin"` (`MainWindow.xaml.cs:2149`) | email send id | No |
| `utm_source` | hardcoded `"newsletter"` | analytics | No |
| `utm_medium` | hardcoded `"email"` | analytics | No |
| `utm_campaign` | `DateTime.UtcNow.ToString("yyyy-MM-dd")` (`MainWindow.xaml.cs:2383-2384`) | analytics | No |

The cross-stitch slug page reads `eid` and `cid` from `searchParams` and emits a server-side notification email **and** updates DynamoDB on each click (`cross-stitch/src/app/[slug]/page.tsx:158-181`).

### 4.10 Unsubscribe URL

Shape: `https://cross-stitch.com/unsubscribe?token=<UnsubscribeToken>`

Builder (`Uploader/MainWindow.xaml.cs:2306-2314`):

```csharp
private string BuildUnsubscribeUrl(string token)
{
    string configuredBaseUrl = ConfigurationManager.AppSettings["UnsubscribeBaseUrl"];
    string baseUrl = !string.IsNullOrWhiteSpace(configuredBaseUrl)
        ? configuredBaseUrl.TrimEnd('/')
        : $"{_linkHelper.SiteBaseUrl}/unsubscribe";

    return $"{baseUrl}?token={Uri.EscapeDataString(token)}";
}
```

Config: `Uploader/Uploader/App.config:18` → `UnsubscribeBaseUrl=https://cross-stitch.com/unsubscribe`. Token semantics belong to the `dynamodb-schema` / users contract.

### 4.11 Image URL (S3/CloudFront convenience)

For completeness, `PatternLinkHelper.BuildImageUrl` (`Uploader/Helpers/PatternLinkHelper.cs:73-76`) returns `{S3PublicBaseUrl}/{S3PhotoPrefix}/{albumId}/{designId}/{photoFileName}` with `S3PhotoPrefix=photos` (`App.config:5`). The shape itself is owned by the `s3-paths` contract; this contract only notes that `PatternLinkHelper` is the C#-side caller.

## 5. API Endpoints / Interfaces

There is **no HTTP API contract between Uploader and cross-stitch** for URL building (architectureFacts.architecture). The contract surface is:

### 5.1 Writer-side (Uploader, C#)

| Symbol | File:line | Purpose |
|---|---|---|
| `PatternLinkHelper.SiteBaseUrl` | `Uploader/Helpers/PatternLinkHelper.cs:44` | resolved base URL property |
| `PatternLinkHelper.BuildPatternUrl(PatternInfo)` | `Uploader/Helpers/PatternLinkHelper.cs:62-71` | absolute design URL |
| `PatternLinkHelper.BuildAlbumUrl(string, string?)` | `Uploader/Helpers/PatternLinkHelper.cs:78-95` | absolute album URL |
| `PatternLinkHelper.BuildImageUrl(int, int, string)` | `Uploader/Helpers/PatternLinkHelper.cs:73-76` | absolute S3/CloudFront image URL |
| `PatternLinkHelper.BuildAlbumCaptionSlug(string?, string)` | `Uploader/Helpers/PatternLinkHelper.cs:97-134` | private helper, album-only slug |
| `MainWindow.AppendTrackingParameters` | `Uploader/MainWindow.xaml.cs:2352` | adds `cid`, `eid`, `utm_*` |
| `MainWindow.BuildUnsubscribeUrl` | `Uploader/MainWindow.xaml.cs:2306` | builds unsubscribe URL from token |

### 5.2 Writer-side (pinterest-agent, TS) — duplicate

| Symbol | File:line | Purpose |
|---|---|---|
| `buildDesignUrl(caption, albumId, nPage)` | `cross-stitch/automation/pinterest-agent/scripts/export-design-pin-map.ts:55-58` | duplicate of `CreateDesignUrl`, used by the design→pin export script |

### 5.3 Reader-side (cross-stitch, TS)

| Symbol | File:line | Purpose |
|---|---|---|
| `normalizeBaseUrl(input)` | `cross-stitch/src/lib/url-helper.ts:5-24` | canonicalises any base URL |
| `getSiteBaseUrl()` | `cross-stitch/src/lib/url-helper.ts:27-29` | reads `NEXT_PUBLIC_SITE_URL` |
| `buildCanonicalUrl(path)` | `cross-stitch/src/lib/url-helper.ts:31-37` | path → absolute |
| `getSiteHostname()` | `cross-stitch/src/lib/url-helper.ts:39-46` | host extraction |
| `CreateDesignUrl(design)` | `cross-stitch/src/lib/url-helper.ts:54-59` | path-only design URL |
| `CreateImageUrl(caption, designId)` | `cross-stitch/src/lib/url-helper.ts:62-68` | path-only image URL |
| `CreateAlbumUrl(caption)` | `cross-stitch/src/lib/url-helper.ts:71-76` | path-only album URL |
| `middleware()` | `cross-stitch/src/middleware.ts:4-37` | legacy-host redirect, HSTS |
| `parseSlugForDesign(slug)` | `cross-stitch/src/app/[slug]/page.tsx:13-25` | parser for design URLs |
| `getAlbumCaptionFromSlug(slug)` | `cross-stitch/src/app/[slug]/page.tsx:27-34` | parser for album URLs |
| `SlugPage` | `cross-stitch/src/app/[slug]/page.tsx:151-192` | router for both shapes |
| `DesignPage` | `cross-stitch/src/app/designs/[designId]/page.tsx:152-312` | numeric-id design route |
| `AlbumsPage` | `cross-stitch/src/app/albums/page.tsx:20-75` | catalog page, generates album hrefs inline |

### 5.4 Config keys consumed

| Key | Repo | Default | File:line |
|---|---|---|---|
| `SiteBaseUrl` | Uploader | `https://cross-stitch.com` | `App.config:6` |
| `PinterestLinkUrl` | Uploader | `https://cross-stitch.com` (legacy fallback for `SiteBaseUrl`) | `App.config:16` |
| `AlbumUrlTemplate` | Uploader | unset (uses built-in `/Free-{slug}-Charts.aspx`) | not in `App.config` |
| `UnsubscribeBaseUrl` | Uploader | `https://cross-stitch.com/unsubscribe` | `App.config:18` |
| `S3PhotoPrefix` | Uploader | `photos` | `App.config:5` |
| `S3PublicBaseUrl` | Uploader | unset (computed from `S3BucketName`) | not in `App.config` |
| `S3BucketName` | Uploader | `cross-stitch-designs` | `App.config:4` |
| `NEXT_PUBLIC_SITE_URL` | cross-stitch | `https://cross-stitch.com` (hardcoded fallback) | `url-helper.ts:28` |

## 6. Versioning

**Unversioned (implicit).** No version field exists in any URL shape, builder, or parser. There is no documented migration story for changing the design or album URL pattern.

The `.aspx` extension is a vestige of a previous tech stack: cross-stitch.com formerly ran on ASP.NET. The current Next.js App Router preserves the suffix purely for SEO compatibility — incoming inbound links from old indexed pages, Pinterest pins, and email archives still resolve. The literal substring `Free-Design.aspx` / `Charts.aspx` is hardcoded in the parser (`cross-stitch/src/app/[slug]/page.tsx:15, 29, 119, 134, 183, 187`), so a rename would require coordinated changes in: all three URL builders (§4.1), both parsers (§4.2, §4.4), the sitemap generator (`cross-stitch/src/app/sitemap.xml/route.ts:67, 84`), the metadata canonical-tag generator, and any external content already indexed.

**Recommendation:** when a URL-shape change becomes necessary, add a new path prefix (e.g. `/v2/designs/...`) and serve the legacy `.aspx` path via permanent redirect. Do not mutate the existing pattern in place.

## 7. Ownership & Contacts

| Role | Owner |
|---|---|
| Maintainer | Olga (epolga) — `olga.epstein@gmail.com` |
| C# code owner | Uploader repo (`Uploader/Helpers/PatternLinkHelper.cs`, `Uploader/MainWindow.xaml.cs`) |
| TS code owner | cross-stitch repo (`src/lib/url-helper.ts`, `src/app/[slug]/page.tsx`, `src/middleware.ts`) |
| Automation code owner | cross-stitch repo (`automation/pinterest-agent/scripts/export-design-pin-map.ts`) |
| Docs owner | cross-stitch-platform-docs hub (this file) |

Per the docs-hub source-of-truth rule, this contract is authoritative; if any builder drifts from the patterns in §4, the builder is wrong.

## 8. Dependencies

| Depends on | Why | Contract |
|---|---|---|
| `DynamoDB.CrossStitchItems.Caption` | encoded into design and album slug | `dynamodb-schema` |
| `DynamoDB.CrossStitchItems.AlbumID` | embedded in design URL between caption and page | `album-id`, `dynamodb-schema` |
| `DynamoDB.CrossStitchItems.NPage` | `NPage-1` embedded in design URL | `design-id`, `dynamodb-schema` |
| `DynamoDB.CrossStitchItems.DesignID` | path segment of `/designs/[designId]` | `design-id` |
| `DynamoDB.CrossStitchUsers.cid` | tracking parameter value | (user-record contract, undocumented) |
| `DynamoDB.CrossStitchUsers.UnsubscribeToken` | query-string value of unsubscribe URL | (user-record contract) |
| Site base URL config | absolute URL prefix | this contract §4.7 |
| S3 / CloudFront image paths | composed into `CreateImageUrl` / `BuildImageUrl` | `s3-paths` |

This contract has **no downstream dependents that mutate it** — only readers.

## 9. Error Handling

Code-observed failure modes:

| Failure | Detected at | Behaviour |
|---|---|---|
| `PatternInfo` null on design build | `PatternLinkHelper.cs:64` | throws `ArgumentNullException` |
| `PatternInfo.Title` null on design build | `PatternLinkHelper.cs:66` | falls back to string `"Cross-stitch-pattern"` (silently produces a generic-looking URL) |
| `NPage` non-numeric | `PatternLinkHelper.cs:67` | `int.TryParse` returns 0; URL contains `-1-Free-Design.aspx`. **Silent corruption** — no log, no throw. |
| `AlbumId` blank on album build | `PatternLinkHelper.cs:80-81` | throws `ArgumentException` |
| `Caption` blank/punctuation-only on album build | `PatternLinkHelper.cs:99-100, 124-125` | returns `Album-{albumId}` |
| `SiteBaseUrl` missing in app settings | `PatternLinkHelper.cs:53-57` | throws `ConfigurationErrorsException` at constructor time |
| Slug fails to match design pattern (reader) | `[slug]/page.tsx:21-23` | `parseSlugForDesign` returns `null` → `notFound()` (Next 404) |
| Slug matches design pattern but `(albumId, nPage)` not in DynamoDB | `[slug]/page.tsx:50-52` | `notFound()` |
| Slug fails to match album pattern (reader) | `[slug]/page.tsx:30-33, 66-68` | returns `null` → `notFound()` |
| Album caption parsed but not found in DDB | `[slug]/page.tsx:73-75` | `notFound()` |
| Design fetch fails in `/designs/[designId]` | `designs/[designId]/page.tsx:75-82, 163-173` | renders error UI (does not 404) |
| Tracking-email send fails | `[slug]/page.tsx:178-180` | caught and logged, page renders normally |
| `MissingDesignPdfs.txt` read fails | `designs/[designId]/page.tsx:62-65` | caught, returns empty set, page renders normally |
| Legacy host received | `middleware.ts:4-17` | HTTP 308 redirect to `cross-stitch.com` |

Notable silent-fail: the `NPage`-not-parseable case produces a URL with negative page index without any log, and depending on the DynamoDB GSI shape, this URL may quietly 404 in production while looking syntactically valid in email previews.

## 10. Security & Compliance

- All URLs in scope are **public** by design (catalog pages indexed by search engines).
- Slugs are derived from album/design `Caption` values authored by Olga; **no user-supplied content enters a slug**, so URL-injection is out-of-scope.
- The `cid` tracking parameter (§4.9) **carries the user-correlation GUID** from `CrossStitchUsers.cid`. While the value is opaque, it is per-user and persistent, making it **PII-adjacent**: anyone who intercepts a newsletter link (forwarded email, screenshot, server-side log line) can correlate subsequent clicks to that account. The cross-stitch handler explicitly logs the `cid` and the IP address to an admin notification email (`cross-stitch/src/app/[slug]/page.tsx:165-170`).
- HSTS is enabled for all HTTPS responses with `max-age=31536000` (`cross-stitch/src/middleware.ts:33`).
- HTTPS is enforced one-way: `normalizeBaseUrl` rewrites `http://` to `https://` (`url-helper.ts:9`); there is no fallback for development without HTTPS aside from the `localhost` exception in `resolveBaseUrl` (`[slug]/page.tsx:84-86`).
- Unsubscribe token is URL-escaped via `Uri.EscapeDataString` (`MainWindow.xaml.cs:2313`); the token generation algorithm (HMAC, per architectureFacts.user) is documented in the users contract, not here.
- The cross-stitch design page makes a server-side fetch to `http://localhost:3000/api/designs/${designId}` (`cross-stitch/src/app/designs/[designId]/page.tsx:73, 160`) — this hardcoded `http://localhost:3000` is a code smell rather than a security issue in production (Elastic Beanstalk single-instance), but breaks any reverse-proxy or multi-instance deployment.

## 11. Testing & Validation

### 11.1 Round-trip test (recommended, not present)

```ts
// hypothetical Vitest spec in cross-stitch
import { CreateDesignUrl } from '@/lib/url-helper';
import { parseSlugForDesign } from '@/app/[slug]/page'; // would need to be exported

const d = { Caption: 'Lion', AlbumID: 37, NPage: 115, DesignID: 9999 };
const url = CreateDesignUrl(d); // '/Lion-37-114-Free-Design.aspx'
const parsed = parseSlugForDesign(url.slice(1)); // strip leading '/'
expect(parsed).toEqual({ caption: 'Lion', albumId: 37, nPage: 114 });
```

Equivalent in C# (xUnit) against `PatternLinkHelper.BuildPatternUrl`:

```csharp
var helper = new PatternLinkHelper(); // requires SiteBaseUrl configured
var url = helper.BuildPatternUrl(new PatternInfo {
    Title = "Lion", AlbumId = "37", NPage = "115"
});
Assert.EndsWith("/Lion-37-114-Free-Design.aspx", url);
```

### 11.2 Triplication drift detector (recommended)

A CI grep check to catch divergent edits to any one of the three implementations:

```bash
# Should print exactly the same trailing path template literal in all three places.
grep -nE "Free-Design\.aspx" \
  Uploader/Uploader/Helpers/PatternLinkHelper.cs \
  d:/ann/Git/cross-stitch/src/lib/url-helper.ts \
  d:/ann/Git/cross-stitch/automation/pinterest-agent/scripts/export-design-pin-map.ts
```

If any one file disagrees on the literal `{slug}-{id}-{n}-Free-Design.aspx` shape, the check should fail the PR.

### 11.3 Manual smoke tests

- Navigate to `https://cross-stitch.com/Lion-37-114-Free-Design.aspx` → renders design.
- Navigate to `https://cross-stitch.com/lion-37-114-free-design.aspx` → renders the same design (case-insensitive parser).
- Navigate to `https://cross-stitch.com/Free-Nature-Charts.aspx` → renders album.
- Navigate to `https://cross-stitch-pattern.net/Lion-37-114-Free-Design.aspx` → 308-redirected to `cross-stitch.com`.
- Send a preview email from the Uploader (`BtnSendEmails_Click` flow) and inspect the rendered HTML for: design URL, `cid=...&eid=...&utm_source=newsletter&utm_medium=email&utm_campaign=YYYY-MM-DD`, unsubscribe URL.

## 12. References

### 12.1 The triplication problem (CRITICAL)

The canonical design URL pattern `/{Caption}-{AlbumID}-{NPage-1}-Free-Design.aspx` is implemented **three times** in three different languages across two repos:

1. **`Uploader/Uploader/Helpers/PatternLinkHelper.cs:62-71`** — `BuildPatternUrl` (C#). Used by every newsletter email blast and every Pinterest pin description authored by the Uploader.
2. **`cross-stitch/src/lib/url-helper.ts:54-59`** — `CreateDesignUrl` (TypeScript). Used by on-site links, canonical metadata, sitemap, and like-API responses.
3. **`cross-stitch/automation/pinterest-agent/scripts/export-design-pin-map.ts:55-58`** — `buildDesignUrl` (TypeScript). The function body is preceded by the comment `// Mirrors CreateDesignUrl in src/lib/url-helper.ts` — an explicit acknowledgement of the duplication.

These three implementations must change together. The `platform-architecture-summary.md:190, 241` already flags this as a known issue.

**Recommendation, in priority order:**

1. **Short-term (lowest effort):** add a CI grep check (§11.2) to fail any PR that modifies the literal template in one file without modifying the other two. This is a 5-line GitHub Action.
2. **Medium-term:** extract the literal pattern to `platform-config.json` at the docs-hub root (the same file that already carries `pinterestTokenPath`, per architectureFacts.architecture), with both `PlatformConfig` loaders (C# Uploader + cross-stitch pinterest-agent TS) reading it. Note that the cross-stitch web app does not currently load `platform-config.json`, so this path requires extending the web-app loader.
3. **Long-term (highest effort, cleanest):** codegen a single typed URL-builder module from a JSON schema, emitted into both languages at build time. This eliminates the literal entirely from source code and makes the URL shape a versionable artifact.

### 12.2 Source files (verified read in full)

- `d:/ann/Git/Uploader/Uploader/Helpers/PatternLinkHelper.cs` (lines 1-136) — design + album + image builders, slug helper, base-URL resolution
- `d:/ann/Git/Uploader/Uploader/App.config` (lines 1-28) — `SiteBaseUrl`, `PinterestLinkUrl`, `UnsubscribeBaseUrl`, `S3PhotoPrefix`
- `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:2306-2389` — `BuildUnsubscribeUrl`, `BuildUnsubscribeUrlFromStoredToken`, `AppendTrackingParameters`, `AppendUtmParameters`
- `d:/ann/Git/cross-stitch/src/lib/url-helper.ts` (lines 1-77) — `normalizeBaseUrl`, `getSiteBaseUrl`, `buildCanonicalUrl`, `CreateDesignUrl`, `CreateImageUrl`, `CreateAlbumUrl`
- `d:/ann/Git/cross-stitch/src/middleware.ts` (lines 1-43) — legacy-host redirect, HSTS
- `d:/ann/Git/cross-stitch/src/app/[slug]/page.tsx` (lines 1-192) — `parseSlugForDesign`, `getAlbumCaptionFromSlug`, slug router
- `d:/ann/Git/cross-stitch/src/app/albums/page.tsx` (lines 1-75) — albums-catalog page, inline href construction
- `d:/ann/Git/cross-stitch/src/app/designs/[designId]/page.tsx` (lines 1-312) — numeric-id design route
- `d:/ann/Git/cross-stitch/automation/pinterest-agent/scripts/export-design-pin-map.ts` (lines 1-146) — DynamoDB design-to-pin export, contains third `buildDesignUrl`

### 12.3 Related contracts (do not duplicate)

- `album-id` — semantics and uniqueness of `AlbumID`
- `design-id` — semantics and uniqueness of `DesignID`, and the `NPage`-to-page-index relationship
- `dynamodb-schema` — attribute definitions for `Caption`, `AlbumID`, `NPage`, `DesignID`, `EntityType`
- `s3-paths` — image and PDF object key shapes
- `pdf-structure` — PDF file naming
- `pinterest-metadata` — what fields end up in a pin payload (the URL is one of them; this contract owns the URL shape, the pinterest contract owns the wrapping payload)

### 12.4 Architecture references

- `cross-stitch-platform-docs/platform-architecture-summary.md:34-36` — URL helpers, middleware, customer-facing routes
- `cross-stitch-platform-docs/platform-architecture-summary.md:190, 241` — known triplication
- `cross-stitch-platform-docs/platform-architecture-summary.md:191` — `AlbumUrlTemplate` override
- `cross-stitch-platform-docs/platform-architecture-summary.md:193` — base-URL resolution chains
- `cross-stitch-platform-docs/platform-architecture-summary.md:181` — unsubscribe URL contract
