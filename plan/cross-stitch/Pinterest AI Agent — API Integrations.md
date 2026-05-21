# Pinterest AI Agent — API Integrations

(Migrated from cross-stitch/plan/)

## Purpose

This document extracts and centralizes all API-related architecture and implementation details from the original master planning document.

This includes:

* OAuth flows
* token lifecycle management
* API providers
* scopes
* authentication architecture
* refresh-token strategy
* service separation

---

# Current Integrated APIs

## Google APIs

### Current integrations

* Google OAuth
* GA4 Data API
* AdSense Management API

### Current status

Working

### Authentication model

OAuth Desktop Application.

### Important understanding

Google OAuth testing mode is acceptable for the current private/internal tool architecture.

### Current stored credentials

Client ID
Client Secret
Refresh Token

### Important operational understanding

Google OAuth app testing mode does NOT mean fake/test data.
Real production analytics data is still used.

---

## Pinterest APIs

### Current integrations

* Pinterest Ads API
* Pinterest OAuth
* Pinterest metrics reporting

### Current scopes

ads:read
boards:read
boards:write
pins:read
pins:write
