# Weekly Status Report: Customer Portal

**Sprint:** Sprint 24 (2026-10-19 – 2026-10-30)
**Report date:** 2026-10-23
**Prepared by:** Program Manager

> Illustrative example with fictional data.

---

## Overall Status: AMBER

> Delivery is on pace (90% of expected), but one open blocker puts the Release 3.2 date at risk.

## Executive Summary

Sprint 24 is at the halfway point with 19 of 42 committed points complete (45%, against 50% expected). The team is pacing close to its 40-point average velocity. The main concern is a delay in the third-party payment sandbox, which blocks testing of the checkout work needed for Release 3.2 on 2026-11-13. We need leadership to help escalate the vendor issue by 2026-10-27 to keep that date.

## Key Metrics

| Metric | This Sprint | Previous / Average |
|---|---|---|
| Committed points | 42 | 40 |
| Completed points | 19 (45%) | 40 |
| Expected completion at this point | 50% | – |
| Velocity | 19 (in progress) | 40 (avg of last 3 sprints) |
| Scope change (points added / removed) | +3 / −3 | – |
| Carryover from previous sprint | 2 items | 3 items |
| Bugs opened / closed this week | 7 / 9 | – |

## Completed This Week

- **Checkout Redesign:** Customers can now save and reuse multiple delivery addresses.
- **Reporting API:** Monthly usage data is available to partners through the new export endpoint.
- **Platform Upgrade:** The staging environment now runs on the upgraded database version.

## In Progress / Planned Next Week

- **Checkout Redesign:** Payment flow changes are built and waiting on sandbox testing (see blockers).
- **Reporting API:** Access controls and rate limiting are in development, with completion expected next week.
- **Platform Upgrade:** Production migration rehearsal is planned. It may slip if staging issues recur.

## Blockers and Risks

| # | Description | Impact | Owner (role/team) | Mitigation / Next step | Severity |
|---|---|---|---|---|---|
| 1 | Payment vendor sandbox is unavailable. | Checkout payment testing cannot start, which puts Release 3.2 (2026-11-13) at risk. | Vendor management / Backend team | Escalate to vendor account contact on 2026-10-24; prepare a mock-based fallback test plan. | High |
| 2 | Production migration rehearsal depends on a single environment slot. | A scheduling conflict could delay the platform upgrade by about one week. | DevOps team | Reserve the slot for 2026-10-29. | Low |

## Asks of Leadership

- Escalate the payment sandbox outage with the vendor's executive contact by 2026-10-27.

## Milestones and Releases

| Milestone / Release | Target date | % complete | Status |
|---|---|---|---|
| Release 3.1 (hotfix) | 2026-10-16 | 100% | Done |
| Release 3.2 | 2026-11-13 | 55% | At risk |
| Release 3.3 | 2026-12-11 | 10% | On track |

## Epic Progress

| Epic | Points done / total | % complete |
|---|---|---|
| Checkout Redesign | 9 / 18 | 50% |
| Reporting API | 7 / 14 | 50% |
| Platform Upgrade | 3 / 10 | 30% |

---

_Data source: Jira (Customer Portal board), as of 2026-10-23 09:00 UTC._
