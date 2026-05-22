# Cross-Stitch Platform Integration Architecture Summary

## Overview
This document summarizes the architecture and integration points between the major components of the Cross-Stitch platform:
- cross-stitch web application
- WPF uploader application
- External services (AWS, Pinterest, PayPal, etc.)

## Component Map
- **cross-stitch web app**: Provides the main user interface, APIs, and business logic. Integrates with AWS for storage and database, Pinterest for marketing, and PayPal for payments.
- **Uploader (WPF app)**: Handles batch uploads, metadata entry, and Pinterest automation. Integrates with AWS (S3), Pinterest API, and communicates with the web app via defined contracts.
- **External Services**: AWS (S3, DynamoDB, etc.), Pinterest API, PayPal.

## Integration Points
- **Uploader ↔ Web App**: Data contracts for uploads, metadata, and status updates.
- **Both Apps ↔ AWS**: S3 path conventions, DynamoDB schemas, authentication.
- **Both Apps ↔ Pinterest**: API usage, board/pin management, metadata mapping.
- **Web App ↔ PayPal**: Subscription and payment processing.

## Shared Contracts
- All integration contracts are versioned and documented in this folder using the CONTRACT-TEMPLATE.md.
- Contracts cover data formats, API endpoints, error handling, and security requirements.

## Shared Secrets and Credentials
- The Pinterest token file is stored at `Uploader/secrets/pinterest_tokens.json`.
- Both the WPF uploader and the web app reference this path via configuration (never hardcoded).
- This location is outside of source code, can be gitignored, and is the single source of truth for Pinterest API credentials.

## Change Management
- All changes to integration points or contracts must be documented and versioned here before implementation.

## Risks & Mitigation
- Integration blind spots: Mitigated by maintaining up-to-date, versioned contracts.
- Assumption drift: Mitigated by explicit documentation and review of all shared contracts.

## References
- See CONTRACT-TEMPLATE.md for the required structure of all integration contracts.
- See individual contract files for details on each integration point.
