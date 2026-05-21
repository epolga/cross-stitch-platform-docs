# Pinterest AI Agent — Design-Level Intelligence

(Migrated from cross-stitch/plan/)

## Purpose

This document covers the per-design analytics layer: connecting Pinterest pins to specific designs in DynamoDB, measuring performance at the design level, and the future creative feedback loop that uses those signals to drive new design creation.

It exists separately from Memory and Trend Analysis because design-level intelligence is a distinct architectural concern — it learns *which designs* work, not just whether spend produced revenue.

The milestone framing for this work lives in Milestones and Roadmap — **Milestone 6b — Design-Level Intelligence V1** (completed).

---

# The Design ↔ Pin Relationship

The website structure is:

Albums → Designs → Pinterest pins

Each album contains designs. Each pin is created from a specific design image. DynamoDB already stores the design → pin id mapping, so the system can reliably connect:

design image
  ↔ Pinterest pin
  ↔ Pinterest performance
  ↔ GA4 traffic
  ↔ AdSense revenue estimate

This relationship is strategically important because the agent can learn not only which pins work, but which underlying designs work.

---

# Why this matters

Without this relationship, the agent only sees:

Pin A performed well.

With it, the agent can reason:

Design 245 performed well as a Pinterest creative.
Horse designs in Album X produce stronger traffic.
Warm animal designs generate better monetized visitors.

This turns the system from campaign reporting into design-level creative intelligence.

---

# Version 1 — Shipped

Version 1 of design-level intelligence has shipped and is wired into the daily cron.

## What was built

scripts/export-design-pin-map.ts    -> reports/design-pin-map.json
scripts/build-design-performance.ts -> reports/design-performance.json
scripts/test-ai-design-analysis.ts  -> reports/design-insights.{md,json}
                                       reports/ai-analysis/*-design-analysis.{md,json}

## What it captures per design

designId
albumId
albumCaption (used as temporary theme/category)
pinId
designCaption
designUrl
impressions
clicks
outboundClicks
ctr
saves

Window: trailing 30 days, anchored on yesterday.

## What the AI produces

`test-ai-design-analysis.ts` asks Claude (Sonnet — see API Integrations) to identify:
* strongest design themes/styles (by impressions and engagement)
* underperforming albums (with explicit statistical-significance caveats)
* design directions to create more of
* caveats the data cannot resolve (seasonality, pin age, audience drift)

Each run appends to `reports/ai-recommendations-history.json` with `analysisType: "design"`.

---

# Categorization strategy

## Current (Version 1)

Album caption is used as the design's theme/category:

Album Caption -> helpful but imperfect categorization

It is intentionally treated as a starting point, not absolute truth. Designs that fit poorly in their album will skew the analysis.

## Planned improvements (Version 2+)

* manual or AI-assisted tagging
* subject extraction (e.g. "kitten", "horse", "bouquet")
* style extraction (e.g. "warm pastel", "flat vector", "detailed")
* main color extraction
* complexity scoring
* beginner-friendly detection
* seasonal / religious / animal / floral tags
* per-design metadata in DynamoDB beyond album caption

Strategic reason: better categorization will produce better AI insights. A "Panda" inside a generic "Animals" album may be invisible compared to the same "Panda" inside an album with stronger keyword discoverability — see the saved memory entry on the Free-vs-Animals signal.

---

# Future data model

Future per-design records should eventually carry:

designId
albumId
pinId
designTitle
designTheme
imageStyle
boardId
pageUrl
Pinterest metrics
GA4 metrics (per page)
AdSense estimate (per page)

This enables analysis at multiple levels: pin / design / album / theme / style / board.

DynamoDB persistence for these enriched records is tracked under Memory and Trend Analysis → Milestone 8 — DynamoDB Persistence Layer, using the schema names `DesignPinMap`, `DesignPerformance`, `AIDesignInsight`.

---

# Future creative intelligence loop

Design-level intelligence is the foundation for a future closed-loop creative system:

Pinterest statistics
  ↓
Identify which pins/designs work better
  ↓
Extract visual and thematic patterns
  ↓
Generate creative briefs/prompts
  ↓
Create new image candidates
  ↓
User approves and converts selected images into cross-stitch designs
  ↓
Uploader publishes new pins
  ↓
New data returns into the system

Human approval stays in the loop. The agent suggests; Olga approves.

## Stage 1 — Learn which pins work (in progress)

The agent collects and compares pin-level metrics:

pin title
board
theme
image style
colors
subject
impressions
outbound clicks
CTR
saves
spend (where applicable)
GA4 sessions
AdSense value

Version 1 covers a subset of these (impressions, clicks, outboundClicks, ctr, saves, plus the album-level theme proxy). Richer metadata (board, image style, colors, subject) requires either additional Pinterest API calls or per-design tagging — both still planned.

## Stage 2 — Store visual findings

The system stores not only numbers, but AI-generated creative findings:

{
  "finding": "Warm-color animal pins with one large central subject perform better than detailed multi-object designs.",
  "confidence": 0.71,
  "evidence": ["pin_123", "pin_245", "pin_301"],
  "recommendedAction": "Create more simple warm animal variants."
}

The current `reports/design-insights.json` is a Version 1 of this idea, structured for the album/design level rather than per-finding.

## Stage 3 — Generate creative briefs/prompts

Before generating images directly, the agent first produces creative briefs the user can review:

Create a flat vector horse head, warm colors, white background, 4–5 colors,
simple poster style, suitable for cross-stitch conversion.

This keeps the creative process controllable.

## Stage 4 — Generate image candidates

Future workflow:

Agent suggests prompts
  ↓
Image model generates candidates
  ↓
User reviews
  ↓
Selected images become cross-stitch designs
  ↓
Uploader publishes pins

Human approval remains part of this workflow.

---

# Strategic significance

Design-level intelligence converts the system from campaign-level analysis (spend → revenue) into creative-pattern learning (which designs → which performance). Over time this creates a full creative feedback loop:

performance data
  ↓
creative pattern discovery
  ↓
new image ideas
  ↓
new pins
  ↓
new performance data

The long-term value is helping the business produce more designs similar to proven winners, without relying purely on guesswork — while keeping a human in the approval path.
