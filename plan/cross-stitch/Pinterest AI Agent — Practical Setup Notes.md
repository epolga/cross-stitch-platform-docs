# Pinterest AI Agent — Practical Setup Notes

(Migrated from cross-stitch/plan/)

## Purpose

Salvaged practical guidance from the original `VS Code Technical Implementation Plan` master doc. Most of that file's content was either implemented and now lives in the code, or already covered by the other thematic docs. The sections below are the parts that still carry value:

* what to install if setting up the agent on a fresh machine
* the originally-suggested repo layout (for reference; the actual layout lives in `automation/pinterest-agent/`)
* strategic guardrails that are timeless (don't-do-X advice, business warnings, the first useful AI prompt)

---

# Tools to install

## Required
For local development on a new machine:
1. **Node.js 20 LTS or newer**
2. **Git**
3. **VS Code**
4. **AWS CLI v2**
5. **TypeScript** (project provides via devDependencies)
6. **Postman** or **Thunder Client** VS Code extension for API testing

The agent itself currently runs via `tsx` against TypeScript files; no Lambda/SAM tooling is required until the AWS migration milestone (see AWS Deployment).

## Recommended VS Code extensions
* ESLint
* Prettier
* AWS Toolkit
* GitHub Pull Requests and Issues
* Thunder Client
* DotENV
* GitLens

Optional:
* Claude Code (or other coding assistant) for AI-assisted edits

---

# Original suggested repository structure

The original plan proposed a standalone repo (`pinterest-ai-agent/`). The actual code lives inside the cross-stitch website monorepo at `automation/pinterest-agent/`, and the structure has evolved. Kept here for historical reference:

pinterest-ai-agent/
  package.json
  tsconfig.json
  .env.example
  README.md
  src/
    index.ts
    config.ts
    types.ts
    services/
      pinterestClient.ts
      ga4Client.ts
      adsenseClient.ts
      dynamoClient.ts
      sesClient.ts
      aiClient.ts
    jobs/
      dailyReportJob.ts
    logic/
      attribution.ts
      scoring.ts
      recommendations.ts
      emailTemplate.ts
    scripts/
      testPinterest.ts
      testGa4.ts
      testAdsense.ts
      testEmail.ts
  infra/
    template.yaml

The current actual layout under `automation/pinterest-agent/` differs:
* `src/services/` for the integration clients
* `scripts/` for both production scripts (`daily`, `history`, `ai:trend`, `pinmap`, `perf`, `ai:design`) and one-off tests
* `reports/` for JSON outputs (gitignored)
* No `infra/` yet — that arrives with the AWS migration milestone

---

# Strategic guardrails

## What NOT to build first
Do not start with:
* automatic budget scaling
* automatic campaign pausing
* a complex dashboard
* too many AI prompts at once
* multiple AI models in parallel
* real-time monitoring

Start with one daily report. Add complexity only when the simple version is working and the next problem is concrete.

## Business warning: don't optimize for clicks alone
The right metric for this business is closer to:
AdSense revenue per 100 Pinterest sessions
or:
AdSense revenue per promoted Pin / ad spend

A pin with cheap clicks may still be a loss if visitors don't stay, view pages, or generate ad revenue. Click count is a vanity number; revenue per visitor is the real signal.

## First useful AI prompt template
A starting prompt for any new AI analysis loop:

You are analyzing Pinterest ads for a cross-stitch pattern website.
The business earns mainly from AdSense on the website.
Be conservative.
Do not recommend increasing ad spend unless engagement and estimated revenue support it.
Do not recommend automatic actions.
Classify each ad as KEEP, WATCH, CREATE_VARIATION, PAUSE_CANDIDATE, or NEEDS_MORE_DATA.
Explain the reasoning briefly.

The principle this prompt encodes — be conservative, demand evidence, never recommend automation — is the AI Reasoning doc's philosophy compressed into operational instructions. New prompts in this project should inherit the same stance.
