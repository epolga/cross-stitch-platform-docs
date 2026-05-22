## 1. Contract Name

PDF structure — kit downloads for cross-stitch patterns

## 2. Purpose

Cross-stitch pattern "kit" PDFs are the downloadable artifact end users receive from `cross-stitch.com` for each design. They are produced upstream of the platform by an external chart program (which exports pattern PDFs), reshaped by the Uploader's adjacent `Converter.exe` tool, then uploaded by the WPF Uploader to S3 under a deterministic key pattern. The Next.js `cross-stitch` app reads them back through CloudFront and serves them to logged-in / paid users via `DownloadPdfLink`.

The Uploader is the **producer / owner** of the S3 PDF objects; the cross-stitch web app is a **read-only consumer**. There is no shared library; the path templates are duplicated as string literals on each side.

## 3. Scope

In scope:

- The S3 object keys used to publish per-design kit PDFs and the rules that decide what gets written to which key.
- The set of "format variants" (`1`, `3`, `5`) that the Uploader treats as mandatory.
- The CloudFront-fronted read path consumed by `DownloadPdfLink.tsx`.
- The role of `iTextSharp` in the Uploader (parsing/inspection only, **not** generation).

Out of scope:

- The internal layout of the PDF (cover page, chart grid, legend). The Uploader does not author the PDF body; it accepts the file as input from the user's batch folder and from `Converter.exe`. See §4.
- The user-facing UI/UX of `DownloadPdfLink` (auth gating, paid checks) — only the URL contract is part of this document.
- The `.scc` chart file and `.jpg` preview that share the same DesignID — those are sibling contracts.

## 4. Data Formats

### 4.1 Naming and S3 key layout

Two key shapes are written for **every** design:

| Role               | S3 key                                                          | Source code |
|--------------------|------------------------------------------------------------------|-------------|
| "Main" kit         | `pdfs/{AlbumID}/Stitch{DesignID}_Kit.pdf`                        | `Uploader/MainWindow.xaml.cs:957` |
| Per-variant kits   | `pdfs/{AlbumID}/{DesignID}/Stitch{DesignID}_{N}_Kit.pdf`         | `Uploader/MainWindow.xaml.cs:958-961` |

Where `N ∈ {"1", "3", "5"}` — declared as `RequiredPdfVariants` at `Uploader/MainWindow.xaml.cs:85`:

```csharp
private static readonly string[] RequiredPdfVariants = { "1", "3", "5" };
```

The same three values are enforced when auditing for missing PDFs at `Uploader/MainWindow.xaml.cs:3645-3658` (`BuildExpectedPdfKeys`):

```csharp
foreach (string variant in RequiredPdfVariants)
{
    keys.Add($"pdfs/{albumPart}/{designPart}/Stitch{designPart}_{variant}_Kit.pdf");
}
keys.Add($"pdfs/{albumPart}/Stitch{designPart}_Kit.pdf");
```

The "main" kit at `pdfs/{AlbumID}/Stitch{DesignID}_Kit.pdf` is **not a fourth, distinct PDF** — it is the same bytes as the variant-`1` PDF, uploaded a second time under a flat-folder key for legacy / default consumption. Evidence at `Uploader/MainWindow.xaml.cs:967-968`:

```csharp
await UploadPdfFileAsync(convertedPdf1Path, mainKey).ConfigureAwait(false);
await UploadPdfFileAsync(convertedPdf1Path, key1).ConfigureAwait(false);
```

The variants `1` / `3` / `5` correspond to three different format renderings of the same chart (commonly: stitches-per-inch / page-layout densities). The Uploader treats them as opaque inputs — it expects three files named exactly `1.pdf`, `3.pdf`, `5.pdf` in the batch folder (`Uploader/MainWindow.xaml.cs:948-955`):

```csharp
string pdf1Path = Path.Combine(_batchFolderPath, "1.pdf");
string pdf3Path = Path.Combine(_batchFolderPath, "3.pdf");
string pdf5Path = Path.Combine(_batchFolderPath, "5.pdf");

if (!File.Exists(pdf1Path) || !File.Exists(pdf3Path) || !File.Exists(pdf5Path))
{
    throw new Exception("Required PDFs (1.pdf, 3.pdf, 5.pdf) not found.");
}
```

S3 bucket: `cross-stitch-designs` (configurable via `S3BucketName`, fallback hard-coded at `Uploader/MainWindow.xaml.cs:46-47` and `Uploader/App.config:4`). MIME type written is `application/pdf` (`Uploader/MainWindow.xaml.cs:1025`).

### 4.2 Public read URL (consumer side)

CloudFront distribution `d2o1uvvg91z7o4.cloudfront.net` is the read fronting:

- Default "main" link in DynamoDB-derived `Design.PdfUrl` (`cross-stitch/src/lib/data-access.ts:367`):

  ```ts
  `https://d2o1uvvg91z7o4.cloudfront.net/pdfs/${albumId.N}/Stitch${designId.N}_Kit.pdf`
  ```

- Per-format download in `DownloadPdfLink.tsx:73-74`:

  ```ts
  const chosenNumber = formatNumber || '1';
  return `${PDF_BASE}/${design.AlbumID}/${design.DesignID}/Stitch${design.DesignID}_${chosenNumber}_Kit.pdf`;
  ```

  with `PDF_BASE = 'https://d2o1uvvg91z7o4.cloudfront.net/pdfs'` (`cross-stitch/src/app/components/DownloadPdfLink.tsx:38`).

Default `formatNumber` when caller omits one is `'1'` — matching the variant that also lives at the legacy "main" key.

### 4.3 Generator / source

PDFs are **not generated** by the Uploader. The pipeline is:

1. The user produces three input PDFs (`1.pdf`, `3.pdf`, `5.pdf`) with an external chart program. These are dropped into the batch folder.
2. `Converter.exe` (a separate .NET 9 console app referenced as a hard-coded path at `Uploader/MainWindow.xaml.cs:84` — `D:\ann\Git\Converter\bin\Release\net9.0\Converter.exe`) rewrites each PDF into a `<name>.converted.pdf` (`Uploader/MainWindow.xaml.cs:973-1016`, `ConvertPdfForUploadAsync`).
3. The converted PDFs are uploaded to S3 with the keys above.

The Uploader uses `iTextSharp` **5.5.13.4** (declared at `Uploader/Uploader.csproj:15`) for **two read-only purposes only**:

- Parsing the pattern PDF as text in `PatternInfo.ParsePdf` to extract `Title`, `Notes`, `Width`, `Height`, `NColors` (`Uploader/PatternInfo.cs:111-253`). This reads the PDF as raw text and searches for fixed text-stream markers (e.g. `"(D.M.C.)"` color rows at `Uploader/PatternInfo.cs:233`, `"Design Size: "` at `Uploader/PatternInfo.cs:267`). Notably it does **not** use iTextSharp objects for this — just `File.ReadAllText`.
- Extracting embedded preview images from a PDF in `ExtractImages` at `Uploader/MainWindow.xaml.cs:3480-3510`, which **does** use `iTextSharp.text.pdf.PdfReader`, `PRStream`, `PdfImageObject` (imported at `Uploader/MainWindow.xaml.cs:26-27`).

**The PDF interior structure (cover page, chart layout, color legend, page grid) is determined upstream by the chart program and `Converter.exe`. This contract does not enforce internal layout — only the file naming, the S3 key shape, and the existence of variants 1/3/5.** If/when an internal layout becomes part of the contract, that should be documented in a separate "PDF interior" contract and validated with PDF inspection tooling rather than asserted here.

### 4.4 Pattern fields adjacent to the PDF (DynamoDB)

The fields stored in DynamoDB and returned in the `Design` payload that describe what is *inside* the PDF (extracted by the Uploader from the input PDF, **not** read from the published S3 PDF) are (`Uploader/PatternInfo.cs:14-104`, `cross-stitch/src/lib/data-access.ts:145-156`):

- `Width`, `Height` — stitches (parsed from `"Design Size: W x H stitches"`, `Uploader/PatternInfo.cs:259-288`).
- `NColors` — count of `(D.M.C.)` occurrences (`Uploader/PatternInfo.cs:228-253`).
- `Notes`, `Title`, `Description` (e.g. `"100 x 120 stitches 25 colors"`, `Uploader/PatternInfo.cs:128`).

These attributes are *adjacent* to the PDF contract — they describe the chart and are surfaced to UI listings — but they are not part of the PDF binary itself, so they are out of scope for this contract beyond noting their provenance.

## 5. API Endpoints / Interfaces

This contract has **no HTTP API**. The interface is file-IO over S3 / CloudFront:

| Side       | Operation | Path / surface |
|------------|-----------|----------------|
| Producer (Uploader) | Write | `s3://cross-stitch-designs/pdfs/{AlbumID}/Stitch{DesignID}_Kit.pdf` and `…/{DesignID}/Stitch{DesignID}_{1\|3\|5}_Kit.pdf` via AWS SDK `TransferUtility.UploadAsync` (`Uploader/MainWindow.xaml.cs:1018-1029`) |
| Producer (Uploader) | List / audit | `ListObjectsV2` over prefix `pdfs/` (`Uploader/MainWindow.xaml.cs:3590-3617`, `LoadAllPdfKeysAsync`) |
| Consumer (cross-stitch) | Read URL synthesis | `getPDFUrl` in `cross-stitch/src/lib/data-access.ts:365-369` |
| Consumer (cross-stitch) | Read URL synthesis (per-variant) | `DownloadPdfLink.tsx:71-75` |
| End user | Download | `GET https://d2o1uvvg91z7o4.cloudfront.net/pdfs/…` (anonymous; gated by client-side mode in `DownloadPdfLink.tsx`) |

There is **no signed-URL** mechanism. CloudFront serves the objects publicly; access control to the download is purely a client-side check (`mode === 'free' | 'register' | 'paid'`) in `DownloadPdfLink.tsx:367-424`. Once a user has any URL, the underlying PDF is reachable without auth.

## 6. Versioning

**Unversioned (implicit).** There is no PDF-version marker in:

- The filename (no `v1` / `v2` / date suffix).
- The S3 key.
- The PDF metadata (the Uploader doesn't write any — it just streams `Converter.exe`'s output).
- The DynamoDB row.

Re-uploading a design overwrites the previous PDF in place (S3 keys are deterministic, no etag check). The CloudFront cache TTL is the only "version" — stale clients may see old PDFs until the edge expires.

Recommendation (1-2 lines): If the PDF interior shape is ever standardized, embed a generator-version field in the PDF's `/Producer` or `/Keywords` metadata via `Converter.exe`, and surface it as a DynamoDB attribute (e.g. `PdfFormatVersion`). For breaking layout changes, prefer adding a new variant number (`7`, `9`) rather than mutating the existing `1`/`3`/`5` semantics.

## 7. Ownership & Contacts

- **Maintainer:** Olga (`epolga`, olga.epstein@gmail.com).
- **Code owner of the producer (writes the S3 PDF):** `Uploader` repo — primarily `Uploader/MainWindow.xaml.cs` (`UploadPdfToS3Async`, `BuildExpectedPdfKeys`) and the out-of-tree `Converter.exe` at `D:\ann\Git\Converter\`.
- **Code owner of the consumer (reads via CloudFront):** `cross-stitch` repo — `src/lib/data-access.ts` and `src/app/components/DownloadPdfLink.tsx`.
- **Docs hub:** `d:/ann/Git/cross-stitch-platform-docs`.

## 8. Dependencies

- `AlbumID` (integer; DynamoDB `AlbumID` numeric attribute) — segment 1 of the S3 key.
- `DesignID` (integer; DynamoDB `DesignID` numeric attribute) — segment 2 and the substring after `Stitch`.
- S3 bucket `cross-stitch-designs` (`Uploader/App.config:4`) — see the S3-bucket contract.
- CloudFront distribution `d2o1uvvg91z7o4.cloudfront.net` (`cross-stitch/src/lib/data-access.ts:367`, `cross-stitch/src/app/components/DownloadPdfLink.tsx:38`) — read fronting.
- `iTextSharp` **5.5.13.4** NuGet (`Uploader/Uploader.csproj:15`) — used only for inspection of input PDFs, not for writing the uploaded artifact.
- `Converter.exe` — out-of-tree .NET 9 binary at hard-coded path `D:\ann\Git\Converter\bin\Release\net9.0\Converter.exe` (`Uploader/MainWindow.xaml.cs:84`). The actual PDF body shape is determined by this binary.
- The three source PDFs `1.pdf`, `3.pdf`, `5.pdf` in the user's batch folder (`Uploader/MainWindow.xaml.cs:948-955`) — externally produced by a chart program.
- AWS SDK `AWSSDK.S3 3.7.400.1` (`Uploader/Uploader.csproj:13`) — `TransferUtility` for upload.

## 9. Error Handling

Observed failure modes from code:

- **Missing source PDFs:** if any of `1.pdf`, `3.pdf`, `5.pdf` is missing from the batch folder, `UploadPdfToS3Async` throws `new Exception("Required PDFs (1.pdf, 3.pdf, 5.pdf) not found.")` *before* any upload (`Uploader/MainWindow.xaml.cs:952-955`). This is a fail-closed pre-check — good.
- **`Converter.exe` failure:** `ConvertPdfForUploadAsync` throws if the converter exe is missing (`Uploader/MainWindow.xaml.cs:978-979`), if it returns a non-zero exit code (`Uploader/MainWindow.xaml.cs:1005-1010`), or if the expected output file is not produced (`Uploader/MainWindow.xaml.cs:1012-1013`). Conversion runs **serially** for variants 1, 3, 5 (`Uploader/MainWindow.xaml.cs:963-965`); a mid-pipeline failure leaves zero S3 writes for this design.
- **S3 upload failure mid-set:** uploads run **serially** (`Uploader/MainWindow.xaml.cs:967-970`):
  ```csharp
  await UploadPdfFileAsync(convertedPdf1Path, mainKey).ConfigureAwait(false);
  await UploadPdfFileAsync(convertedPdf1Path, key1).ConfigureAwait(false);
  await UploadPdfFileAsync(convertedPdf3Path, key3).ConfigureAwait(false);
  await UploadPdfFileAsync(convertedPdf5Path, key5).ConfigureAwait(false);
  ```
  If a later call throws, earlier successful uploads are **not rolled back**. This silently leaves an orphan `pdfs/{Album}/Stitch{Design}_Kit.pdf` and possibly `_1_Kit.pdf` without `_3_` / `_5_`. Downstream consumers will get a 404 from `DownloadPdfLink.tsx` for the missing variants (`design.PdfUrl ?? fallbackPdfUrl ?? null`, line 88 — null path renders `"PDF is not available for this design."` at line 354-356).
- **Orphan detection:** `FindDesignsWithMissingPdfs` (`Uploader/MainWindow.xaml.cs:3620-3658`) lists S3 keys under `pdfs/` and checks every DynamoDB design row has all four expected keys (main + 1/3/5). This is the only existing audit. It writes a CSV at `WriteMissingPdfReportAsync` (`Uploader/MainWindow.xaml.cs:3660-3668`). It does not currently auto-heal.
- **Consumer side:** `DownloadPdfLink.tsx` performs no HEAD-check on the URL — a 404 manifests as a broken browser download. The "missing" path is hinted only via a separate `MissingDesignPdfs.txt` flag passed in as `isMissing` (line 78-88), not via S3 reality.
- **PDF parse failure in PatternInfo:** parse errors in `ParsePdf` surface as a `MessageBox.Show` to the operator (`Uploader/PatternInfo.cs:132`, similar at 164, 220, 250, 284) — UI-only, no logging, no telemetry.

## 10. Security & Compliance

- **Public PDFs.** The PDFs are anonymously fetchable over CloudFront; the `cross-stitch.com` paid/registration gate is enforced only in the React component (`cross-stitch/src/app/components/DownloadPdfLink.tsx:367-424`). Anyone with the URL can download. Treat the PDF content accordingly.
- **No user-specific metadata.** Verified by code inspection: `UploadPdfToS3Async` (`Uploader/MainWindow.xaml.cs:946-971`) uploads the converter's output verbatim; the converter is invoked with only the input path as an argument (`Uploader/MainWindow.xaml.cs:993`). There is no per-recipient customization, no watermarking with `userEmail`, and no other PII branch in the upload path. `userEmail` does appear on the consumer side, but only for the paid-access HTTP check (`DownloadPdfLink.tsx:277`) — it is not appended to the PDF URL or to the file.
- **No signed URLs / no expiring links.** Once published, the URL is permanent and shareable.
- **Hard-coded converter path leak.** `D:\ann\Git\Converter\bin\Release\net9.0\Converter.exe` (`Uploader/MainWindow.xaml.cs:84`) embeds an absolute developer-machine path. Not a secret, but a portability/dev-environment coupling. Same applies to `SuppressedListPath = @"D:\ann\Git\cross-stitch\list-suppressed.txt"` on the adjacent line (`Uploader/MainWindow.xaml.cs:83`).
- **No PDF/A or signature.** The PDFs are not digitally signed and there is no provenance / tamper-evidence step.

## 11. Testing & Validation

Concrete checks that can run today:

- **Naming check (regex against both repos):** the canonical pattern is `pdfs/\d+/(?:Stitch\d+_Kit\.pdf|\d+/Stitch\d+_[135]_Kit\.pdf)`. Grep both sides for stray variants:
  ```
  rg -n 'Stitch\$\{?[A-Za-z]+\}?_(?:[A-Za-z]+|\d+)_Kit\.pdf' Uploader cross-stitch
  rg -n "_Kit\.pdf" Uploader cross-stitch
  ```
  Any hit outside the files listed in §12 indicates a new untracked consumer of the contract.
- **End-to-end download:** pick any design row in `CrossStitchItems`, build `https://d2o1uvvg91z7o4.cloudfront.net/pdfs/{AlbumID}/Stitch{DesignID}_Kit.pdf`, fetch it, open in any PDF reader, confirm it renders as a chart kit. Repeat with `_1_`, `_3_`, `_5_` suffixes under `…/pdfs/{AlbumID}/{DesignID}/`.
- **Orphan audit (already implemented):** run the WPF Uploader's "Find missing PDFs" code path (`FindDesignsWithMissingPdfs`, `Uploader/MainWindow.xaml.cs:3620-3658`) to produce a CSV of designs that lack one or more of the four required keys.
- **Variant-set drift:** assert `RequiredPdfVariants` in `MainWindow.xaml.cs:85` matches the consumer's default-variant logic in `DownloadPdfLink.tsx:73` (`formatNumber || '1'` — the `'1'` here must be present in `RequiredPdfVariants`).
- **Bucket / CloudFront mapping:** confirm `cross-stitch-designs` is the origin of `d2o1uvvg91z7o4.cloudfront.net` in the AWS console — the two repos do not share a constant for either name.
- **PDF interior:** **manual** — there is no automated assertion that the PDF body is a valid chart kit. If a `Converter.exe` regression silently produces an empty PDF, only a human opening the file will catch it. Recommended: a smoke test in `Converter.exe` (or a wrapping check in the Uploader) that re-opens the converted PDF via `iTextSharp.PdfReader` and asserts `NumberOfPages > 0` before upload.

## 12. References

- Producer code:
  - `d:/ann/Git/Uploader/Uploader/MainWindow.xaml.cs` (`UploadPdfToS3Async` at 946-971, `ConvertPdfForUploadAsync` at 973-1016, `UploadPdfFileAsync` at 1018-1029, `BuildExpectedPdfKeys` at 3645-3658, `FindDesignsWithMissingPdfs` at 3620-3643, `LoadAllPdfKeysAsync` at 3590-3618, `RequiredPdfVariants` at 85, `ConverterExePath` at 84, `_bucketName` at 46-47, `iTextSharp` imports at 26-27, `ExtractImages` at 3480-3510)
  - `d:/ann/Git/Uploader/Uploader/PatternInfo.cs` (PDF parsing into adjacent DB fields)
  - `d:/ann/Git/Uploader/Uploader/Helpers/PatternLinkHelper.cs` (sibling URL helpers — photos / site links, not PDFs)
  - `d:/ann/Git/Uploader/Uploader/Uploader.csproj` (line 15 — `iTextSharp 5.5.13.4`)
  - `d:/ann/Git/Uploader/Uploader/App.config` (line 4 — `S3BucketName`)
- Consumer code:
  - `d:/ann/Git/cross-stitch/src/lib/data-access.ts` (line 156, 365-369 — `getPDFUrl`)
  - `d:/ann/Git/cross-stitch/src/app/components/DownloadPdfLink.tsx` (line 38 — `PDF_BASE`; lines 71-75 — `fallbackPdfUrl`)
- Architecture summary: `d:/ann/Git/cross-stitch-platform-docs/`
- Related contracts (sibling docs):
  - S3 photo path `photos/{AlbumID}/{DesignID}/<file>.jpg`
  - S3 bucket `cross-stitch-designs`
  - DynamoDB table `CrossStitchItems` (rows that supply `AlbumID` / `DesignID`)
- Architecture-facts seed: `tasks/01KS7AND3KZ4WGYEEGH7SXFJ0H/task.json` → `prompt.context.architectureFacts.pdf`
