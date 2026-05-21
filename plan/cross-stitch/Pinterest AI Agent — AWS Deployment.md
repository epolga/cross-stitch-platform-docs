# Pinterest AI Agent — AWS Deployment

(Migrated from cross-stitch/plan/)

## Purpose

This document centralizes planned AWS deployment architecture and operational automation strategy for the Pinterest AI Agent.

---

# Current State

Current development and operations environment:

Local VS Code development
Windows Task Scheduler running daily-run.bat once per day
Local JSON report storage under automation/pinterest-agent/reports/
.env-based credentials (one .env per consumer)
DynamoDB read access from the agent via a scoped IAM user (read-only)

The AWS components below are the planned production target. They are not yet in use — the agent currently runs entirely from the developer machine.

---

# Planned AWS Architecture

Recommended future architecture:

EventBridge
↓
Lambda execution
↓
API integrations
↓
AI analysis
↓
DynamoDB/S3 persistence
↓
SES reporting

---

# Planned AWS Components

## Lambda
Planned responsibilities:
* scheduled execution
* metrics retrieval
* AI analysis
* report generation
* orchestration

## EventBridge
Planned responsibilities:
* daily scheduling
* automation timing
* orchestration triggering

## Secrets Manager
Planned responsibilities:
* API credential storage
* OAuth credential storage

Important understanding:
Secrets Manager stores credentials. It does NOT refresh OAuth tokens automatically.

---

## DynamoDB
Planned responsibilities:
* historical business memory
* trend persistence
* recommendation history
* longitudinal analysis

---

## SES
Planned responsibilities:
* report delivery
* alerts
* anomaly notifications
* operational summaries

---

# Planned Production Flow

EventBridge triggers Lambda
↓
Lambda retrieves credentials
↓
Metrics collected from APIs
↓
AI reasoning executed
↓
Results persisted
↓
Email/report generated

---

# Operational Philosophy

The initial production system should prioritize:
* reliability
* observability
* safety
* controlled automation
before:
* aggressive autonomous actions

---

# Long-Term Future

Possible future capabilities:
* semi-autonomous optimization
* controlled campaign adjustments
* automated experiments
* operational anomaly handling

All future automation should include:
* rollback logic
* confidence thresholds
* human oversight
