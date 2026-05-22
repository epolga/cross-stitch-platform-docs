# Integration Planning

This folder contains all planning documents for integration strategies, contracts, and workflows between:
- The cross-stitch web application
- The WPF uploader application
- External services: AWS (S3, DynamoDB, etc.), Pinterest API, PayPal

## Integration Points (Summary)
- **Uploader ↔ Web App**: Upload API, metadata sync, status updates
- **Uploader ↔ AWS**: S3 uploads, authentication, DynamoDB (if used)
- **Uploader ↔ Pinterest**: Board/pin creation, metadata automation
- **Web App ↔ AWS**: S3 access, DynamoDB, authentication
- **Web App ↔ Pinterest**: Board/pin sync, analytics
- **Web App ↔ PayPal**: Subscription and payment processing

## What is Documented Here
- Data contracts and API endpoints between cross-stitch and uploader
- S3 path conventions and file formats
- Shared metadata and synchronization points
- Integration risks and mitigation strategies
- Versioning and change management for shared contracts
- Mermaid diagrams and architecture summaries

## Shared Secrets
- Pinterest token file location: `Uploader/secrets/pinterest_tokens.json` (used by both projects)

## Planned Contracts (not yet implemented)
- [business-history-schema.md](./business-history-schema.md) — `CrossStitchBusinessHistory` DynamoDB table planned for Milestone 5 (Pinterest AI Agent historical memory: daily business metrics, AI analyses, design-level analytics, anomaly events). Companion S3 bucket `cross-stitch-ai-reports` for AI markdown bodies.

## References
- See ARCHITECTURE-SUMMARY.md for a textual summary
- See INTEGRATION-MAP.md for a visual map
- See CONTRACT-TEMPLATE.md for the contract structure
