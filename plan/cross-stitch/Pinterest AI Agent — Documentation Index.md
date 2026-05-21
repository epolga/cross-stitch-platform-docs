# Pinterest AI Agent — Documentation Index

(Migrated from cross-stitch/plan/)

## Purpose

This document serves as the central navigation/index for the Pinterest AI Agent project documentation.

As the project grows, documentation is being split into specialized thematic documents to improve:

* maintainability
* navigation
* editing safety
* scalability

---

# Current Documents

Modularization is complete. The following thematic documents now exist in `plan/`:

1. Pinterest AI Agent — Milestones and Roadmap
2. Pinterest AI Agent — API Integrations
3. Pinterest AI Agent — AI Reasoning
4. Pinterest AI Agent — WPF Uploader Integration
5. Pinterest AI Agent — Memory and Trend Analysis
6. Pinterest AI Agent — Design-Level Intelligence
7. Pinterest AI Agent — AWS Deployment
8. Pinterest AI Agent — Practical Setup Notes

## Retired

### Pinterest AI Agent — VS Code Technical Implementation Plan

The original 4,500-line master planning document, retired after its content was redistributed across the thematic docs above. Git history preserves the original full content.

Topic-to-doc mapping (where each original section went):

| Original topic | New home |
|---|---|
| High-level architecture, AWS components, IAM, deployment | AWS Deployment |
| DynamoDB schema, env vars, AI memory architecture, insights table, embeddings | Memory and Trend Analysis |
| Milestones 1-7, Phase 2 actions, recommended sequence | Milestones and Roadmap |
| AI agent role, scoring logic, AI memory principles, confidence scoring, learning loop, reasoning philosophy | AI Reasoning |
| Google OAuth, Pinterest API, Anthropic API, OAuth scopes, token strategy | API Integrations |
| WPF uploader, future uploader API contract, recommendation categories | WPF Uploader Integration |
| Design-level intelligence, pin ↔ design relationship, creative loop | Design-Level Intelligence |
| External platform applications (Meta, Reddit, etc.) | Milestones and Roadmap (Milestone 0) |
| Tools to install, original repo layout, "what not to build first", business warning, first AI prompt | Practical Setup Notes |
