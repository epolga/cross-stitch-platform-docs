# AWS Integration Planning

This folder contains planning documents for integration with AWS services, including:
- S3 bucket and path conventions for uploads and storage (used by both web app and uploader)
- DynamoDB schema and usage (if used for metadata or other storage)
- Authentication and access control for both apps
- Deployment and resource management
- Monitoring and error handling

## Key Integration Points
- Uploader ↔ AWS: S3 uploads, authentication, DynamoDB
- Web App ↔ AWS: S3 access, DynamoDB, authentication

## References
- See plan/integration/ARCHITECTURE-SUMMARY.md and INTEGRATION-MAP.md for overall context
