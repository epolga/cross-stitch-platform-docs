# Focus

## Current goal

Finish Milestone 5 soak window, then build Milestone 8 daily summary email.

## Active work

### Milestone 5 — DynamoDB soak window
Soak restarted from 2026-05-27 after two pipeline failures (5/25 and 5/26)
caused by duplicate DESIGN_PIN_MAP DDB keys. Root cause fixed in AutoPinner
(`GetLatestUnpinnedAsync` now skips DesignIDs already pinned on another row).

Soak runs through 2026-06-02. Check `daily-run.log` each morning and tick
SOAK-WINDOW.md. If all 7 days pass green, proceed to the read cutover
(switch historyBuilder from local JSON to DDB) and strip the JSON writes.

### Milestone 8 — Daily summary email (next after soak)
SES is already wired. Build the daily summary email: yesterday's KPIs +
latest AI trend recommendation, sent at the end of every cron run.
Estimated ~1 day of work.

## Pending / lower priority

- Milestone 7: migrate cron agent from Windows Task Scheduler to AWS Lambda + EventBridge (~1 day)
- Milestone 8 remainder: AI recommendation change alerts, Telegram bot for phone notifications

## Out of scope (do not touch)

- Uploader WPF app (Milestone 10 — planned, not started)
- Meta / Reddit / TikTok expansion (Milestone 11 — future)
- Controlled automation (Milestone 13 — long-term)

## Done when

- [ ] SOAK-WINDOW.md days 1–7 all ✓ (target 2026-06-02)
- [ ] Read cutover: historyBuilder reads from DDB, not local JSON
- [ ] JSON writes stripped from daily pipeline scripts
- [ ] Daily summary email sent and verified end-to-end via SES
