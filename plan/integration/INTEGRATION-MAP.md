# Cross-Stitch Platform Integration Map

## Overview
This document visually maps the key integration points and data flows between the major components of the Cross-Stitch platform.

---

## Mermaid Diagram

```mermaid
graph TD
    Uploader[WPF Uploader App]
    WebApp[Cross-Stitch Web App]
    AWS[(AWS: S3, DynamoDB, etc.)]
    Pinterest[(Pinterest API)]
    PayPal[(PayPal)]

    Uploader -- Uploads, Metadata, Status --> WebApp
    Uploader -- S3 Uploads, Auth --> AWS
    Uploader -- Board/Pin Automation --> Pinterest
    WebApp -- S3 Access, DB, Auth --> AWS
    WebApp -- Board/Pin Sync, Analytics --> Pinterest
    WebApp -- Subscription, Payments --> PayPal

    AWS -- Storage, Data --> WebApp
    AWS -- Storage, Data --> Uploader
    Pinterest -- Board/Pin Data --> WebApp
    Pinterest -- Board/Pin Data --> Uploader
    PayPal -- Payment Events --> WebApp
```

---

## Integration Points
- Uploader ↔ Web App: Upload API, metadata sync, status updates
- Uploader ↔ AWS: S3 uploads, authentication, DynamoDB (if used)
- Uploader ↔ Pinterest: Board/pin creation, metadata automation
- Web App ↔ AWS: S3 access, DynamoDB, authentication
- Web App ↔ Pinterest: Board/pin sync, analytics
- Web App ↔ PayPal: Subscription and payment processing

## References
- See ARCHITECTURE-SUMMARY.md for a textual summary
- See CONTRACT-TEMPLATE.md for contract structure
- See individual contract files for details
