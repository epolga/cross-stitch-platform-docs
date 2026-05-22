# Cross-Stitch Platform — Integration Contracts

This folder holds **operational/reference** integration contracts for the Cross-Stitch platform: the implicit rules that bridge the WPF Uploader, the Next.js web app, AWS infrastructure, and Pinterest. Each document follows the 12-section [CONTRACT-TEMPLATE.md](../../plan/integration/CONTRACT-TEMPLATE.md) format.

Per [CLAUDE.md](../../CLAUDE.md), these are placed under `docs/` (what exists) rather than `plan/` (what we intend). They describe behavior already in code; they do not propose new contracts.

## Contracts

### Identifiers, schemas, and paths
| Contract | What it covers | Density |
|---|---|---|
| [album-id.md](./album-id.md) | AlbumID format (raw integer, `D4` zero-padded Uploader-internal form, `ALB#{D4}` DDB partition shape), interfaces (DDB / S3 / URL / CSV), upgrade path when ID exceeds 9999 | 12 sections, 62 citations |
| [design-id.md](./design-id.md) | DesignID as the cross-repo design identifier; `(ID, NPage)` compound PK; `DesignsByID-index` GSI; URL slug using `NPage-1` (not DesignID); padded `D5` form for SCC chart keys | 12 sections, 68 citations |
| [dynamodb-schema.md](./dynamodb-schema.md) | `CrossStitchItems` + `CrossStitchUsers` schemas (every observed attribute with type, writer, reader). Includes `PasswordResetTokens` and `SubscriptionEvents`. Flags PinID drift across 6 historical names, plaintext password storage, and `DeletionProtectionEnabled: false`. | 12 sections, 266 citations |
| [s3-paths.md](./s3-paths.md) | Bucket names (`cross-stitch-designs`, `cross-stitch-sitemap-cache`); CloudFront origin; photo path `photos/{AlbumID}/{DesignID}/4.jpg` (the `4` is a constant, not a range); PDF paths; `charts/{DesignID:D5}_*` write-only family | 12 sections, 75 citations |
| [pdf-structure.md](./pdf-structure.md) | Variant set `{1, 3, 5}` declared at `MainWindow.xaml.cs:85`; "main" PDF is bytes-identical to variant 1; PDFs come from an out-of-tree `Converter.exe`, not iTextSharp; partial-failure orphan risk; CloudFront-public, no PII | 12 sections, 39 citations |
| [pinterest-metadata.md](./pinterest-metadata.md) | Pin title/description templates (verbatim from `PinterestHelper.cs`), board-naming + SEO-rename rules, `AlbumBoards.csv` format, OAuth scope `pins:read pins:write boards:read boards:write ads:read`, token JSON writer/reader asymmetry | 12 sections, 118 citations |

### Flows, deployment, and identity
| Contract | What it covers | Density |
|---|---|---|
| [aws-deployment.md](./aws-deployment.md) | EB app `cross-stitch-com`, env `cross-stitch-com-env-clone`, `.ebextensions/*` option configs, EB-config-vs-snapshot drift (TLS policies, EC2 key pair), manual `eb deploy` with no CI, Uploader's Restart-EB button at `MainWindow.xaml.cs:367`, `cloudwatch_logs` log-group name predates the env rename | 12 sections, 114 citations |
| [upload-flow.md](./upload-flow.md) | F1 design publish sequence: chart→PDF→photo→**Pinterest pin (step 3)**→DDB insert→EB restart. Non-idempotent (no in-progress guard, no `ConditionExpression`); Pinterest pin can succeed while DDB insert fails (orphan pin pointing at 404); audit `FindDesignsWithMissingPdfs` catches DDB-without-S3 but not the common S3-without-DDB orphan | 12 sections, 76 citations |
| [email-and-unsubscribe.md](./email-and-unsubscribe.md) | SES newsletter blasts + transactional flows; unsubscribe URL `https://cross-stitch.com/unsubscribe?token=…` is raw token (no HMAC; `UnsubscribeSecret` is dead config); `RemoveSuppressedUsersAsync` is asymmetric (deletes CrossStitchItems USR# rows but leaves CrossStitchUsers); `List-Unsubscribe-Post: One-Click` header but the receiver is GET-only | 12 sections, 171 citations |
| [user-identity.md](./user-identity.md) | PK `USR#<email>`; two coexisting `cid` formats (dashed UUID from cross-stitch vs 32-hex-no-dash from Uploader); two coexisting `UnsubscribeToken` formats; plaintext `Password`/`OpenPwd` storage (with the literal "Storing plain password for migration purposes" code comment); `verifyUser` logs email+password to stdout; PayPal signature can be bypassed via env var | 12 sections, 95 citations |
| [url-conventions.md](./url-conventions.md) | Design URL `/{Caption}-{AlbumID}-{NPage-1}-Free-Design.aspx` is **triplicated** across 3 builders with subtly different whitespace handling (`Replace(' ', '-')` vs `replace(/\s+/g, '-')`); album URL `/Free-{slug}-Charts.aspx`; site base URL fallback chain in `PatternLinkHelper.cs:46-60` | 12 sections, 98 citations |

See also: [../cross-stitch/architecture-diagram.md](../cross-stitch/architecture-diagram.md) — Mermaid C4-style component view + 4 sequence diagrams that visualize how these contracts connect.

## Why these exist

All seven items on CLAUDE.md's *do-not-invent* list now have documented contracts (DynamoDB schemas, AlbumID/DesignID formats, S3 path structures, PDF generation conventions, uploader↔website contracts, Pinterest metadata conventions, AWS deployment assumptions). This folder captures what's actually in the code, with every claim cited by file:line so the contracts can be re-verified by re-grep at any time.

## How to use

- **Before adding a new attribute / path / convention**: read the relevant contract first to avoid drift.
- **When the live system changes**: update the corresponding contract in the same PR. The §11 "Testing & Validation" section in each contract lists the grep/SDK check to re-confirm.
- **When you find a discrepancy** between what's in a contract and what's in the live system: the live system wins. File an issue and update the contract.

## Provenance

Generated by two babysitter runs on 2026-05-22:
- First 6 contracts (album-id, design-id, dynamodb-schema, s3-paths, pdf-structure, pinterest-metadata): run `01KS7AGP936JGQ989X3GHEQV1T`.
- Latter 5 contracts (aws-deployment, upload-flow, email-and-unsubscribe, user-identity, url-conventions) + the architecture-diagram doc: run `01KS7C014YMACBR1ABF9ABT2TH`.

Each draft was authored by an agent reading the cited source files, then independently verified by a second agent reading the same files. 24/24 sampled claims passed verification across both runs; 0 CLAUDE.md `do-not-invent` violations after one refinement loop in run 2 (which removed an invented HMAC claim from the architecture diagram and refreshed stale line numbers).

## Related

- [../../CLAUDE.md](../../CLAUDE.md) — repo rules of the road and the `do-not-invent` list.
- [../cross-stitch/platform-architecture-summary.md](../cross-stitch/platform-architecture-summary.md) — system-level description of where each contract fits.
- [../../plan/integration/CONTRACT-TEMPLATE.md](../../plan/integration/CONTRACT-TEMPLATE.md) — the 12-section template every contract here follows.
- [../../plan/integration/ARCHITECTURE-SUMMARY.md](../../plan/integration/ARCHITECTURE-SUMMARY.md) — older integration-focused summary; complements but does not supersede this folder.
