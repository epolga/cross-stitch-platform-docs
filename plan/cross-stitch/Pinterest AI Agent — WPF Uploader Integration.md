# Pinterest AI Agent — WPF Uploader Integration

(Migrated from cross-stitch/plan/)

## Purpose

This document centralizes architecture and future plans related to integration between the Pinterest AI Agent and the existing WPF Pinterest Uploader application.

---

# Current Situation

The existing WPF uploader already supports:
* Pinterest OAuth
* token refresh logic
* pin publishing
* board management
* token persistence

The uploader currently acts primarily as:
manual publishing application

---

# Strategic Direction

The uploader should evolve into:
publishing interface for the AI agent

---

# Recommended Architecture

Recommended long-term flow:
WPF Uploader
↓
Requests recommendations from AI Agent backend
↓
Receives:
  title suggestions
  description suggestions
  board suggestions
  keyword suggestions
  UTM suggestions
↓
User reviews/approves
↓
Uploader publishes to Pinterest

---

# Important Operational Philosophy

Initially:
AI suggests
Human approves
Uploader publishes

The uploader should remain:
human-supervised operational interface
rather than fully autonomous publisher.

---

# Example Future API Contract

## Request
{
  "designId": "245",
  "theme": "horse",
  "pageUrl": "https://cross-stitch.com/Horse-9-245-Free-Design.aspx"
}

## Response
{
  "title": "Horse Cross Stitch Pattern PDF",
  "description": "A printable horse cross-stitch chart.",
  "recommendedBoard": "Animals",
  "utmContent": "horse_245_pin_01"
}

---

# Planned Future Features
* AI title suggestions
* AI description suggestions
* AI keyword generation
* AI board recommendations
* seasonal recommendations
* campaign recommendations
* creative suggestions

---

# Long-Term Vision

The uploader becomes:
operational frontend for the intelligent marketing system
while the AI/backend handles:
* reasoning
* analytics
* optimization
* recommendations
* historical memory
