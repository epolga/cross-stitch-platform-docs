# Contract: S3 Paths and Bucket Conventions

## 1. Contract Name

**S3 paths and bucket conventions** — the layout of the `cross-stitch-designs` S3 bucket: photo keys (`photos/{AlbumID}/{DesignID}/<file>.jpg`), PDF keys (`pdfs/{AlbumID}/Stitch{DesignID}_Kit.pdf` and `pdfs/{AlbumID}/{DesignID}/Stitch{DesignID}_{N}_Kit.pdf`), chart keys (`charts/{DesignID:D5}_{Title}.scc`), and the CloudFront distribution `d2o1uvvg91z7o4.cloudfront.net` through which the cross-stitch web app consumes them.

## 2. Purpose

The Uploader (a Windows WPF app) is the **sole writer** to S3 for design assets — photos, PDFs, and the proprietary `.scc` chart file. The cross-stitch Next.js web app is a **read-only consumer** that builds image and PDF URLs by templating strings against the CloudFront hostname; it never names the S3 bucket directly for this content. This contract codifies the exact key shapes so the writer and reader cannot drift out of sync silently. Today the templates are duplicated as bare string literals on both sides — no shared constant, no schema check — and that is the risk this contract documents.

## 3. Scope

In scope:
- Bucket name `cross-stitch-designs` (the design-assets bucket).
- Three S3 key families: `photos/`, `pdfs/`, `charts/`.
- CloudFront distribution `d2o1uvvg91z7o4.cloudfront.net` (origin = the bucket above; only `/photos/**` is whitelisted in Next.js image config).
- The image/PDF URL builders on both repos.

Out of scope:
- The unrelated `cross-stitch-sitemap-cache` bucket (a separate, cross-stitch-only artifact bucket; see §4).
- DynamoDB schema (see DynamoDB-keys contract).
- Article/marketing images served directly from `cross-stitch-designs.s3.us-east-1.amazonaws.com` under `/images/articles/...` (one-off legacy `<img>` tags, not part of the design pipeline).
- The `/images/default.jpg` open-graph fallback on CloudFront (referenced but not produced by Uploader).

## 4. Data Formats

### 4.1 Buckets

| Bucket | Purpose | Who writes | Who reads |
|---|---|---|---|
| `cross-stitch-designs` | Design photos, PDFs, charts | Uploader (only) | Browser via CloudFront; cross-stitch SSR for sitemap (never named directly in code path) |
| `cross-stitch-sitemap-cache` | Sitemap cache (separate concern) | cross-stitch (Elastic Beanstalk env) | cross-stitch only |

- `cross-stitch-designs` is hardcoded as a fallback in C#:
  - `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:46-47` — `private readonly string _bucketName = ConfigurationManager.AppSettings["S3BucketName"] ?? "cross-stitch-designs";`
  - `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:59-60` — `new S3Helper(RegionEndpoint.USEast1, "cross-stitch-designs");` (note: literal string, *not* read from config — drift risk)
  - `d:/ann/Git/Uploader/Uploader/Helpers/PatternLinkHelper.cs:32-37` — same fallback used to build a direct-S3 image URL when `S3PublicBaseUrl` is unset
  - `d:/ann/Git/Uploader/Uploader/App.config:4` — `<add key="S3BucketName" value="cross-stitch-designs"/>`
- `cross-stitch-sitemap-cache` is referenced only in `d:/ann/Git/cross-stitch/saved_configs/eb-configuration-2025-12-12.cfg.yml:18` (`S3_BUCKET_NAME: cross-stitch-sitemap-cache`) and is unrelated to the design-asset contract.

### 4.2 CloudFront distribution

- Hostname: `d2o1uvvg91z7o4.cloudfront.net`.
- Next.js whitelists **only `/photos/**`** as a remote image pattern: `d:/ann/Git/cross-stitch/next.config.js:11-14`:
  ```
  { protocol: 'https', hostname: 'd2o1uvvg91z7o4.cloudfront.net', pathname: '/photos/**' }
  ```
- PDFs are loaded as plain `window.open(...)`, not via `next/image`, so they live under the same host but a different prefix (`/pdfs/...`), see `d:/ann/Git/cross-stitch/src/app/components/DownloadPdfLink.tsx:38` (`const PDF_BASE = 'https://d2o1uvvg91z7o4.cloudfront.net/pdfs';`).

### 4.3 Photo key — `photos/{AlbumID}/{DesignID}/{fileName}.jpg`

Writer (Uploader):
- Prefix constant: `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:73` — `private const string PhotoPrefix = "photos";`
- Key builder: `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:2229-2232`:
  ```
  private string GetPhotoKey(int designId, string fileName)
      => $"{PhotoPrefix}/{_albumId}/{designId}/{fileName}";
  ```
- Two files are written per design (NOT a 1-4 range — only two specific filenames):
  - `4.jpg` — the catalog/listing photo, hardcoded filename: `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:124` — `_imageFilePath = Path.Combine(_batchFolderPath, "4.jpg");`
  - `4_pinterest.jpg` — the 1000x1500 watermarked Pinterest variant: `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:74` — `private const string PinterestPhotoFileName = "4_pinterest.jpg";` (uploaded only if it exists on disk: `MainWindow.xaml.cs:1031-1039` `UploadImageToS3Async` / `UploadPhotoFileAsync`).
- App.config makes the prefix theoretically configurable but it is hardcoded to `"photos"` in the writer and never read from `S3PhotoPrefix` in `MainWindow`:
  - `d:/ann/Git/Uploader/Uploader/App.config:5` — `<add key="S3PhotoPrefix" value="photos"/>` (this key *is* honored only by `PatternLinkHelper.cs:39-41`, not by the actual upload).

Reader (cross-stitch):
- The catalog image URL is built as a literal template against AlbumID/DesignID with the filename `4.jpg`:
  - `d:/ann/Git/cross-stitch/src/lib/data-access.ts:153-155`:
    ```
    ImageUrl: item.ImageUrl?.S || (item.AlbumID?.N && item.DesignID?.N
      ? `https://d2o1uvvg91z7o4.cloudfront.net/photos/${item.AlbumID.N}/${item.DesignID.N}/4.jpg`
      : null),
    ```
  - `d:/ann/Git/cross-stitch/src/app/components/RegisterForm.tsx:160` — `` `https://d2o1uvvg91z7o4.cloudfront.net/photos/${albumId}/${designId}/4.jpg` ``
- The Uploader-side equivalent (used for emails and Pinterest pin links, not for S3 upload) is `d:/ann/Git/Uploader/Uploader/Helpers/PatternLinkHelper.cs:73-76`:
  ```
  public string BuildImageUrl(int designId, int albumId, string photoFileName = "4.jpg")
      => $"{_imageBaseUrl}/{_photoPrefix}/{albumId}/{designId}/{photoFileName}";
  ```
  Default `photoFileName` is `"4.jpg"`. This is the third copy of the template.

> **Filename-N is not a range.** Uploader writes exactly `4.jpg` and `4_pinterest.jpg`. The numeral `4` is a magic constant from a legacy multi-photo era; today only the `4`-prefixed files are produced. The reader hardcodes `/4.jpg` everywhere.

### 4.4 PDF keys — two parallel layouts

Writer (Uploader) emits **four** PDF objects per design: one "main" alias plus three per-format variants.

- `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:946-971` — `UploadPdfToS3Async`:
  ```
  string mainKey     = $"pdfs/{_albumId}/Stitch{designId}_Kit.pdf";
  string designFolder = $"pdfs/{_albumId}/{designId}";
  string key1 = $"{designFolder}/Stitch{designId}_1_Kit.pdf";
  string key3 = $"{designFolder}/Stitch{designId}_3_Kit.pdf";
  string key5 = $"{designFolder}/Stitch{designId}_5_Kit.pdf";
  ```
- The variant set is **`{"1", "3", "5"}` — not a 1..N range**: `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:85` — `private static readonly string[] RequiredPdfVariants = { "1", "3", "5" };` (also used by the audit at `MainWindow.xaml.cs:3645-3658` `BuildExpectedPdfKeys`).
  - `1` / `3` / `5` correspond to the input files `1.pdf`, `3.pdf`, `5.pdf` read from the batch folder (`MainWindow.xaml.cs:948-955`, `MainWindow.xaml.cs:132-143`). These represent kit/stitch format codes; meaning is encoded by Converter.exe and is not documented in the Uploader source.
  - **The `_1_Kit.pdf` is uploaded twice** — once as `Stitch{designId}_Kit.pdf` (the main alias) and once as `Stitch{designId}_1_Kit.pdf` (`MainWindow.xaml.cs:967-968` both use `convertedPdf1Path`). So format "1" is the canonical/default.

Reader (cross-stitch) builds both shapes:

- Legacy / "main" PDF URL — used when DynamoDB has no per-format hint:
  - `d:/ann/Git/cross-stitch/src/lib/data-access.ts:365-369`:
    ```
    function getPDFUrl(albumId, designId) {
      return (albumId?.N && designId?.N)
        ? `https://d2o1uvvg91z7o4.cloudfront.net/pdfs/${albumId.N}/Stitch${designId.N}_Kit.pdf`
        : null;
    }
    ```
- Per-format URL — used by the download button:
  - `d:/ann/Git/cross-stitch/src/app/components/DownloadPdfLink.tsx:38` — `const PDF_BASE = 'https://d2o1uvvg91z7o4.cloudfront.net/pdfs';`
  - `d:/ann/Git/cross-stitch/src/app/components/DownloadPdfLink.tsx:71-75`:
    ```
    const chosenNumber = formatNumber || '1';
    return `${PDF_BASE}/${design.AlbumID}/${design.DesignID}/Stitch${design.DesignID}_${chosenNumber}_Kit.pdf`;
    ```
  - The component defaults `formatNumber` to `'1'`. The caller is expected to pass `'1'`, `'3'`, or `'5'`; nothing in the reader enforces this set against the writer's `RequiredPdfVariants`.

Together: `N ∈ {1, 3, 5}` (writer); reader passes whatever the UI hands it, defaulting to `'1'`.

### 4.5 Chart key — `charts/{DesignID:D5}_{Title}.scc`

- `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:930-944` — `UploadChartToS3Async`:
  ```
  string paddedDesignId = designId.ToString("D5");
  string key = $"charts/{paddedDesignId}_{PatternInfo?.Title}.scc";
  ```
  - DesignID is zero-padded to 5 digits (unique to this key family — photos and PDFs use unpadded IDs).
  - The title is interpolated **raw**, including spaces and Unicode — no slugification. ContentType is `"text/scc"` (non-standard).
- No reader observed in either repo. Charts appear to be write-only telemetry/source-of-truth for re-rendering, not consumed by the web app.

### 4.6 Summary table

| Key shape | Writer source | Reader source | N values |
|---|---|---|---|
| `photos/{AlbumID}/{DesignID}/4.jpg` | `MainWindow.xaml.cs:73,124,2231` | `data-access.ts:154`, `RegisterForm.tsx:160`, `PatternLinkHelper.cs:75` | filename literal `4.jpg` |
| `photos/{AlbumID}/{DesignID}/4_pinterest.jpg` | `MainWindow.xaml.cs:74,125,1037` | (not read by web) | filename literal `4_pinterest.jpg` |
| `pdfs/{AlbumID}/Stitch{DesignID}_Kit.pdf` | `MainWindow.xaml.cs:957,3656` | `data-access.ts:367` | (no N) |
| `pdfs/{AlbumID}/{DesignID}/Stitch{DesignID}_{N}_Kit.pdf` | `MainWindow.xaml.cs:959-961,3653` | `DownloadPdfLink.tsx:74` | writer: `{1, 3, 5}`; reader: `'1'` default |
| `charts/{DesignID:D5}_{Title}.scc` | `MainWindow.xaml.cs:933` | (none) | n/a |

## 5. API Endpoints / Interfaces

This contract has **no HTTP API surface**. It is two file-based / object-storage interfaces:

### 5.1 Write side: AWS S3 `PutObject`

From the Uploader, all writes go through `Amazon.S3.Transfer.TransferUtility.UploadAsync` against `_bucketName`:
- Photo upload: `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:1041-1055` (`UploadPhotoFileAsync`, ContentType `image/jpeg`)
- PDF upload: `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:1018-1029` (`UploadPdfFileAsync`, ContentType `application/pdf`)
- Chart upload: `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:935-943` (TransferUtility, ContentType `text/scc`)
- An alternate `S3Helper.UploadPdfAsync` path also exists at `d:/ann/Git/Uploader/Uploader/Helpers/S3Helper.cs:26-47` (uses `AmazonS3Client.PutObjectAsync` directly) — currently only used by `_s3Helper` (constructed at `MainWindow.xaml.cs:59-60` with a hardcoded bucket name, distinct from `_bucketName` even though both resolve to `"cross-stitch-designs"`).

Bucket-listing is also performed by the writer for audits:
- `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:3590-3617` — `LoadAllPdfKeysAsync` (`ListObjectsV2`, Prefix=`"pdfs/"`).
- `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:3679-3709` — `CreateDesignToAlbumMapAsync` (`ListObjectsV2`, Prefix=`"photos/"`).

### 5.2 Read side: CloudFront HTTPS GET

From the browser, every read is an unauthenticated `GET https://d2o1uvvg91z7o4.cloudfront.net/<key>`. The cross-stitch app itself **never calls S3 directly** for this content — it only string-templates the public CDN URL.

This is a **non-obvious fact worth highlighting**: searching `cross-stitch` for `cross-stitch-designs` returns only one hit referencing the bucket by name (`d:/ann/Git/cross-stitch/src/app/Embroidery_History.aspx/page.tsx:32` — a hardcoded marketing-image URL, not the design pipeline), plus a handful of `s3.us-east-1.amazonaws.com` article-image URLs in `WhyCrossStitch/page.tsx`. The design/PDF pipeline goes entirely through CloudFront. Consequently: the cross-stitch app has **no IAM credentials, no bucket policy dependency, and no SDK call** against this bucket — the only contract surface is the URL shape.

### 5.3 Interface points (file-based)

- App.config `S3BucketName`, `S3PhotoPrefix`, `S3PublicBaseUrl`: `d:/ann/Git/Uploader/Uploader/App.config:4-5` (only the first two are present today).
- Next.js `images.remotePatterns` for the `/photos/**` allow-list: `d:/ann/Git/cross-stitch/next.config.js:9-15`.

## 6. Versioning

**Unversioned (implicit).** No `v1/`, no `latest/`, no S3 object-version pinning observed. Keys are overwritten in place on re-upload; consumers always fetch the live key.

- The bucket itself is the production path; **no staging or preview prefix is observed** in either repo. Test uploads go to the same `cross-stitch-designs` namespace.
- Recommended: introduce either (a) a shared TypeScript/C# constants module exposing `PHOTO_BASE`, `PDF_BASE`, and a `formatPhotoKey(albumId, designId, file)` helper that both repos import, or (b) a versioned prefix (`photos/v1/...`, `pdfs/v1/...`) the next time the layout changes — without a version segment, any layout drift becomes a coordinated, multi-repo deploy.

## 7. Ownership & Contacts

- **Maintainer:** Olga (epolga / olga.epstein@gmail.com).
- **Code owner:** the Uploader repo, because it is the writer — every key shape originates from C# string interpolations in `Uploader/MainWindow.xaml.cs`. The cross-stitch repo is downstream.
- **Bucket owner (AWS):** `cross-stitch-designs` is in `us-east-1` (RegionEndpoint.USEast1, `MainWindow.xaml.cs:60`); ownership inside AWS is the same single-maintainer account.

## 8. Dependencies

- **AlbumID** (numeric, unpadded in S3 keys) — see DynamoDB-keys / AlbumID contract. Note the asymmetry: AlbumID is zero-padded to 4 digits as a DynamoDB partition (`ALB#{albumId:D4}`, `MainWindow.xaml.cs:107`) but **unpadded** in S3 keys (`MainWindow.xaml.cs:957`).
- **DesignID** (numeric, unpadded in photo/PDF keys; zero-padded to 5 digits only in `charts/` keys, `MainWindow.xaml.cs:932-933`).
- **NPage** — not embedded in S3 keys directly, but DynamoDB `NPage` ↔ DesignID via `getDesignIdByAlbumAndPage` (`data-access.ts:840-850`).
- **PDF format variants `{1, 3, 5}`** — `MainWindow.xaml.cs:85`. Meaning of each variant is encoded inside `Converter.exe` (`MainWindow.xaml.cs:84`), not in either web repo.
- **CloudFront cache (TTLs unknown)** — the distribution caches `/photos/**` and `/pdfs/**`. No TTL or invalidation policy is checked into either repo; cache-busting after overwrites is not coded anywhere. When the Uploader overwrites `4.jpg`, the CloudFront edge can serve the old object until its TTL elapses. This is a real-world failure mode worth noting.
- **AWS region:** `us-east-1` (`MainWindow.xaml.cs:54-60`).

## 9. Error Handling

Observed in code, not theoretical:

- **No retries on write.** `TransferUtility.UploadAsync` is called once per file (`MainWindow.xaml.cs:943, 1028, 1054`). `S3Helper.UploadPdfAsync` (`Helpers/S3Helper.cs:28-46`) wraps in `try/catch` and **swallows the exception** with `Console.WriteLine($"Error uploading ...")` — the caller cannot tell that the upload failed.
- **Silent overwrites.** Every write is an unconditional `PutObject` with no `If-None-Match`. Re-running an upload silently overwrites prior content (intentional for re-renders, but irreversible without S3 versioning, which is unconfigured).
- **S3 auto-creates the prefix.** AlbumID/DesignID "directories" are not pre-created — S3 has no directories, only keys, so `pdfs/9999/9999/Stitch9999_1_Kit.pdf` works without setup. No code path checks for or creates a prefix.
- **Partial-failure / orphan mode (FLAGGED).** The upload sequence is: chart (`MainWindow.xaml.cs:930`) → PDFs (`MainWindow.xaml.cs:946`) → photos (`MainWindow.xaml.cs:1031`) → DynamoDB insert (`MainWindow.xaml.cs:1057+`) → Pinterest pin. If the run fails between steps — for example, photo upload throws after PDFs have already been put — the PDFs and chart **remain in the bucket forever as orphans** (no DynamoDB row to point to them). The reverse is also true: a successful photo upload + failed DynamoDB insert leaves a "ghost" `photos/{albumId}/{designId}/4.jpg` that is never referenced. There is no transaction, no cleanup pass, and no `Try*` rollback path. The audit helper `LoadAllPdfKeysAsync` + `FindDesignsWithMissingPdfs` (`MainWindow.xaml.cs:3590-3658`) detects the *opposite* case (DDB row with no PDF) but does not detect orphans.
- **`S3Helper` uses a parallel hardcoded bucket name** (`MainWindow.xaml.cs:60` literal `"cross-stitch-designs"`) instead of `_bucketName` (which respects `App.config`). If `S3BucketName` is ever changed in `App.config`, the two writers will diverge.
- **Reader has graceful-degrade fallbacks.** `data-access.ts:153-155` falls back from DynamoDB `ImageUrl?.S` to a templated CloudFront URL; `DownloadPdfLink.tsx:77-88` falls back between `design.PdfUrl`, the per-format URL, and a `MissingDesignPdfs.txt` allow-list. So a missing DynamoDB attribute does not break the page; a missing S3 object yields a 403/404 from CloudFront and the browser shows a broken image / failed download.

## 10. Security & Compliance

- **Public-read bucket.** All design photos and PDFs are served via CloudFront with no signing — they are public-by-design and the contract assumes that.
- **No secrets in keys.** Key shape is `{numeric AlbumID}/{numeric DesignID}/...`; no email, token, or PII is embedded. Filenames are deterministic.
- **Chart filename interpolates raw user-supplied title** (`MainWindow.xaml.cs:933` — `$"charts/{paddedDesignId}_{PatternInfo?.Title}.scc"`). Title is uploader-controlled (extracted from the PDF), not visitor-controlled, but if a title contains slashes or control characters, the resulting S3 key has the same characters — this is a minor hygiene smell, not a security issue.
- **No IAM credentials in cross-stitch for this bucket.** The web app reads only via the public CDN; bucket policy can stay minimal (`s3:GetObject` to CloudFront OAI/OAC).
- **Write credentials live on the developer workstation.** The Uploader uses the default AWS credential chain (`new AmazonS3Client()` with no key — `MainWindow.xaml.cs:50`); credential lifecycle is out of scope for this contract but is the single point of compromise for the write path.

## 11. Testing & Validation

Concrete probes that should be runnable from any shell with AWS CLI installed:

- **List photos for a known design** (spot-check the writer wrote what the reader expects):
  ```
  aws s3 ls s3://cross-stitch-designs/photos/<AlbumID>/<DesignID>/
  ```
  Expected output: `4.jpg` and `4_pinterest.jpg` (the latter only if the Pinterest variant was generated).

- **List PDFs for a known design** (verify the four-object pattern):
  ```
  aws s3 ls s3://cross-stitch-designs/pdfs/<AlbumID>/ | grep "Stitch<DesignID>_Kit.pdf"
  aws s3 ls s3://cross-stitch-designs/pdfs/<AlbumID>/<DesignID>/
  ```
  Expected: one `Stitch<DesignID>_Kit.pdf` at the album level, plus `Stitch<DesignID>_1_Kit.pdf`, `_3_Kit.pdf`, `_5_Kit.pdf` in the design subfolder. If any of the latter three are missing, the existing audit at `MainWindow.xaml.cs:3645-3658` (`BuildExpectedPdfKeys`) flags it.

- **CloudFront round-trip** (confirms the reader's URL templates resolve):
  ```
  curl -I https://d2o1uvvg91z7o4.cloudfront.net/photos/<AlbumID>/<DesignID>/4.jpg
  curl -I https://d2o1uvvg91z7o4.cloudfront.net/pdfs/<AlbumID>/Stitch<DesignID>_Kit.pdf
  curl -I https://d2o1uvvg91z7o4.cloudfront.net/pdfs/<AlbumID>/<DesignID>/Stitch<DesignID>_1_Kit.pdf
  ```
  All three should return `HTTP/2 200` with the appropriate `content-type`.

- **Cross-repo grep for drift** (catches the moment writer and reader diverge):
  ```
  rg -n "photos/\\\$\\{.*\\}/\\\$\\{.*\\}/4\\.jpg" d:/ann/Git/cross-stitch d:/ann/Git/Uploader
  rg -n "Stitch\\\$\\{.*\\}_(Kit|\\d_Kit)\\.pdf"   d:/ann/Git/cross-stitch d:/ann/Git/Uploader
  rg -n "d2o1uvvg91z7o4\\.cloudfront\\.net"        d:/ann/Git/cross-stitch d:/ann/Git/Uploader
  ```
  Today (May 2026): exactly three call sites for the photo template (`PatternLinkHelper.cs:75`, `data-access.ts:154`, `RegisterForm.tsx:160`); two for the PDF main template (`MainWindow.xaml.cs:957`, `data-access.ts:367`); two for the per-format PDF template (`MainWindow.xaml.cs:959-961`, `DownloadPdfLink.tsx:74`). If any of those counts grows without a corresponding shared-constant refactor, drift risk is increasing.

- **Suggested probe script.** A small Node/TS script (e.g. dropped at `d:/ann/Git/cross-stitch/scripts/probe-s3-contract.ts`) iterating a sample of recent DynamoDB DESIGN rows and `HEAD`-ing the expected CloudFront URLs would mechanise the spot-check above. None exists today.

## 12. References

- **Writer (Uploader, C#):**
  - `d:/ann/Git/Uploader/Uploader/App.config:4-5` — `S3BucketName`, `S3PhotoPrefix`.
  - `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:46-47` — `_bucketName` field.
  - `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:59-60` — `S3Helper` ctor with hardcoded bucket.
  - `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:73-74` — `PhotoPrefix`, `PinterestPhotoFileName`.
  - `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:85` — `RequiredPdfVariants = { "1", "3", "5" }`.
  - `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:124-125` — `4.jpg` / `4_pinterest.jpg` local path init.
  - `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:930-944` — `UploadChartToS3Async`.
  - `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:946-971` — `UploadPdfToS3Async` (four-object pattern).
  - `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:1018-1029` — `UploadPdfFileAsync`.
  - `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:1031-1055` — `UploadImageToS3Async` + `UploadPhotoFileAsync`.
  - `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:2229-2232` — `GetPhotoKey`.
  - `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:3590-3617` — `LoadAllPdfKeysAsync`.
  - `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:3645-3658` — `BuildExpectedPdfKeys` (audit).
  - `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs:3679-3709` — `CreateDesignToAlbumMapAsync`.
  - `d:/ann/Git/Uploader/Uploader/Helpers/S3Helper.cs:17-47` — alternate write path with swallowed errors.
  - `d:/ann/Git/Uploader/Uploader/Helpers/PatternLinkHelper.cs:19-42` — image-base resolution and S3 fallback URL.
  - `d:/ann/Git/Uploader/Uploader/Helpers/PatternLinkHelper.cs:73-76` — `BuildImageUrl` (third copy of photo template).
- **Reader (cross-stitch, TS/Next.js):**
  - `d:/ann/Git/cross-stitch/next.config.js:9-15` — `/photos/**` remote pattern allow-list.
  - `d:/ann/Git/cross-stitch/src/lib/data-access.ts:153-155` — photo URL template.
  - `d:/ann/Git/cross-stitch/src/lib/data-access.ts:365-369` — `getPDFUrl` legacy template.
  - `d:/ann/Git/cross-stitch/src/app/components/DownloadPdfLink.tsx:38` — `PDF_BASE`.
  - `d:/ann/Git/cross-stitch/src/app/components/DownloadPdfLink.tsx:71-75` — per-format PDF URL.
  - `d:/ann/Git/cross-stitch/src/app/components/RegisterForm.tsx:160` — duplicate photo URL template.
  - `d:/ann/Git/cross-stitch/saved_configs/eb-configuration-2025-12-12.cfg.yml:18` — separate `cross-stitch-sitemap-cache` bucket (out of scope).
- **Architecture & docs:**
  - `d:/ann/Git/Uploader/CLAUDE.md` — do-not-invent policy that gates this document.
  - `d:/ann/Git/Uploader/.a5c/runs/01KS7AGP936JGQ989X3GHEQV1T/tasks/01KS7ANCZ7731FH25MQVVP7PB5/task.json` — `architectureFacts.s3` seed.
