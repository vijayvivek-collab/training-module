# Weekly Status Report Generator: Technical Specification

## 1. Overview

A tool that pulls sprint data from Jira Cloud and produces a concise weekly status report for executive management, delivered as an HTML email. It is built for a Program Manager overseeing a 10-person software development team that works in Scrum sprints.

**Primary user:** Program Manager (reviews and sends).
**Report audience:** Executive management (reads for status, risk, and milestones).

## 2. Goals and Non-Goals

### Goals
- Replace manual weekly status compilation with a generated draft.
- Give executives a clear overall RAG (Red/Amber/Green) status at a glance.
- Keep the PM in control: generate a draft, allow edits and commentary, then send.

### Non-Goals
- No per-individual performance reporting (team and epic level only).
- No writing back to Jira (read-only integration).
- No real-time dashboard (weekly report only).
- No support for Jira Server/Data Center (Cloud only).

## 3. Requirements Summary (from interview)

| Area | Decision |
|---|---|
| Project type | Software development |
| Team size | 10 |
| Jira | Cloud, Scrum with sprints |
| Audience | Executive management |
| Delivery | Email (HTML) |
| Status | Overall RAG, rule-based with PM override |
| Granularity | Team and epic level; no individual names |
| Commentary | Generated draft, PM reviews/edits before send; LLM narrative optional |
| Trigger | On demand, plus optional weekly schedule (default Fri 09:00) |
| Auth | Jira API token via environment variables |
| Language | Python |

## 4. Functional Requirements

### FR-1: Jira data retrieval
- Connect to Jira Cloud REST API (v3) and Agile API using email and API token.
- Fetch for the configured board/project(s):
  - The active sprint: name, goal, start and end dates.
  - Issues in the sprint with status, status category, issue type, story points, epic link, priority, labels, flagged/blocked state, resolution date, and fix version.
  - Issues completed in the last 7 days.
  - Issues that entered the sprint after the start (scope change).
  - The previous N completed sprints (default 3) for velocity.
  - Fix versions (releases) with their release dates and completion percentage.
- Handle pagination, rate limiting (HTTP 429 with Retry-After), and transient errors with retry and backoff.

### FR-2: Metrics calculation
- Committed vs. completed story points (sprint to date).
- Percent complete and the expected percent, based on elapsed sprint days.
- Carryover: issues not done that carried over from the previous sprint.
- Scope change: points added or removed after sprint start.
- Velocity: average of the last N sprints, compared with the current sprint.
- Blockers: issues flagged, labeled "blocked", or with a configured blocked status.
- Bugs: opened vs. closed this week, and open critical/high bugs.
- Epic progress: points done / total per epic.
- Milestones: each tracked fix version with its date, percent complete, and an on-track or at-risk indication.

### FR-3: RAG status
Computed from configurable rules. The PM can override the result.

| Status | Default condition |
|---|---|
| Green | Completion is at least 80% of expected, and no open blockers |
| Amber | Completion is 60-80% of expected, or any open blocker, or a milestone at risk |
| Red | Completion is below 60% of expected, or a critical blocker, or a missed milestone |

- The worst applicable condition wins.
- The report states the reason(s) behind the RAG status in one line.
- Thresholds live in the config file.

### FR-4: Report content and structure
Ordered for executive skimming; the target is one screen to one page:
1. **Header:** project, sprint name and dates, report date.
2. **Overall RAG** with a one-line reason.
3. **Executive summary:** 3-5 sentences.
4. **Key metrics:** committed vs. completed, velocity trend, scope change, carryover.
5. **Completed this week:** top highlights grouped by epic.
6. **In progress / next week:** grouped by epic.
7. **Blockers and risks:** description, impact, and owner (role or team, not a person's name), plus the PM's asks of leadership.
8. **Milestones and releases:** a table with dates and status.
9. **Footer:** data timestamp and link to the Jira board.

Ticket keys are shown only as links in the appendix or tooltips, not in the main text.

### FR-5: Commentary and narrative
- Generate the draft summary by deterministic templates (default).
- Optional LLM-generated narrative, enabled by a flag. It must receive only the aggregated metrics and sanitized issue titles, with a configurable redaction. It is off by default pending the compliance answer (see Open Questions).
- The PM can add or edit manual commentary (risks, asks) in a Markdown file (`commentary.md`) or in the draft before sending. This commentary is merged into the report.

### FR-6: Delivery
- Render HTML email with inline CSS (compatible with Outlook and Gmail) and a plain-text fallback.
- Modes:
  - `preview`: write the HTML to a local file and open it.
  - `draft`: create a draft in the email system (default).
  - `send`: send immediately through SMTP or the provider API.
- Recipients, subject template, and sender live in config. Subject example: `[Weekly Status] <Project> | Sprint 24 | AMBER | 2026-10-30`.
- Every sent report is archived locally, with the HTML and a JSON snapshot of the metrics.

### FR-7: Execution
- CLI: `report generate [--sprint ID] [--mode preview|draft|send] [--llm]`.
- Optional scheduling by cron or GitHub Actions. The default is Friday 09:00 in a configurable timezone, in `draft` mode so the PM can review.
- Dry-run mode uses cached or fixture data, which allows testing without hitting Jira.

## 5. Non-Functional Requirements

- **Security:** Jira token and email credentials are read from environment variables or a `.env` file that is git-ignored. They are never logged. The Jira token needs read-only scope.
- **Privacy:** no individual names in output by default. The LLM, if enabled, receives only sanitized, aggregated data.
- **Reliability:** retry with backoff. A failed fetch exits with a clear error and never sends a partial report.
- **Performance:** a full report generates in under 60 seconds for one sprint of roughly 100 issues.
- **Maintainability:** modular, typed Python (3.11+), config-driven, with unit tests for metrics and RAG logic.
- **Observability:** structured logging and a `--verbose` flag.

## 6. Architecture

```
┌────────────┐   ┌────────────┐   ┌─────────────┐   ┌────────────┐   ┌────────────┐
│ Jira Client│──▶│  Metrics   │──▶│ RAG Engine  │──▶│  Renderer  │──▶│ Delivery   │
│ (REST/Agile)│  │ Calculator │   │ + Override  │   │ (Jinja2)   │   │ draft/send │
└────────────┘   └────────────┘   └─────────────┘   └────────────┘   └────────────┘
                                         ▲                 ▲
                                         │                 │
                                   config.yaml      commentary.md / LLM (optional)
```

### Proposed layout
```
hello-genai/
├── project_spec.md
├── config.example.yaml
├── .env.example
├── pyproject.toml
├── src/status_report/
│   ├── cli.py
│   ├── config.py
│   ├── jira_client.py
│   ├── models.py          # dataclasses/pydantic models
│   ├── metrics.py
│   ├── rag.py
│   ├── narrative.py       # templates + optional LLM
│   ├── render.py
│   ├── delivery.py
│   └── templates/report.html.j2
├── tests/
│   ├── fixtures/          # sample Jira JSON
│   ├── test_metrics.py
│   ├── test_rag.py
│   └── test_render.py
└── archive/               # generated reports (git-ignored)
```

### Technology choices
- Python 3.11+, `httpx` or `requests`, `pydantic`, `jinja2`, `premailer` (CSS inlining), `typer` (CLI), `pytest`.
- Jira REST API v3 and Jira Agile API (`/rest/agile/1.0`).

## 7. Configuration (example)

```yaml
jira:
  base_url: https://<your-org>.atlassian.net
  project_keys: [ABC]          # TBD
  board_id: 123                # TBD
  story_points_field: customfield_10016   # verify per instance
  blocked_labels: [blocked]
  blocked_statuses: ["Blocked"]
  velocity_sprints: 3
  tracked_fix_versions: []     # TBD

rag:
  green_min_pct_of_expected: 80
  red_below_pct_of_expected: 60
  blocker_forces_amber: true
  critical_blocker_forces_red: true

report:
  title: "Weekly Status Report"
  timezone: "UTC"
  show_ticket_keys: false
  show_individuals: false

delivery:
  mode: draft                  # preview | draft | send
  provider: smtp               # smtp | graph | gmail (TBD)
  from: pm@example.com
  to: [exec-list@example.com]
  subject: "[Weekly Status] {project} | {sprint} | {rag} | {date}"

llm:
  enabled: false
```

Environment variables: `JIRA_EMAIL`, `JIRA_API_TOKEN`, `SMTP_*` (or provider credentials), and `ANTHROPIC_API_KEY` if the LLM is enabled.

## 8. Data Model (core)

- `Sprint`: id, name, goal, start, end, state.
- `Issue`: key, summary, type, status, status_category, points, epic, priority, labels, flagged, resolved_at, added_after_start.
- `SprintMetrics`: committed, completed, pct_complete, pct_expected, carryover, scope_added, scope_removed, velocity_avg, bug_stats.
- `EpicProgress`: epic name, done, total.
- `Milestone`: name, date, pct_complete, status.
- `RagResult`: status, reasons[], overridden.

## 9. Edge Cases

- No active sprint (between sprints): report on the last closed sprint and note it in the header.
- Unestimated issues: fall back to issue counts and flag the data-quality issue.
- Story points field differs per Jira instance: configurable field ID.
- Sprint just started (under 1 day elapsed): skip the RAG percentage rule and show "Too early to assess."
- Multiple boards or projects: aggregate, with per-project sections (future).
- Issues without an epic: grouped under "Unassigned work."
- Jira outage or auth failure: fail loudly and send nothing.

## 10. Testing and Acceptance Criteria

- Unit tests cover metrics, RAG rules (including boundaries and overrides), and rendering against fixture data.
- Golden-file test for the HTML output.
- Manual acceptance:
  1. `report generate --mode preview` produces a correct report for the current sprint.
  2. The RAG status and reason match a manual calculation.
  3. The email renders correctly in Outlook and Gmail.
  4. No individual names or credentials appear in the output or logs.
  5. Draft mode creates the draft without sending.

## 11. Milestones

1. **M1 (foundation):** project scaffold, config, Jira client, fixtures.
2. **M2 (core logic):** metrics and RAG engine with tests.
3. **M3 (output):** HTML template, preview mode, commentary merge.
4. **M4 (delivery):** email draft/send, archive.
5. **M5 (polish):** scheduling, optional LLM narrative, docs.

## 12. Open Questions

1. Which Jira project key(s) and board ID does this cover?
2. Which email system is used (Outlook/Microsoft 365 or Gmail)? This decides the delivery provider (Graph API, Gmail API, or SMTP).
3. Which milestones or fix versions should be tracked, and what are their dates?
4. Are there compliance restrictions on sending ticket data to an external LLM? Until answered, the LLM stays off.
5. Does the PM have a sample report or format preference to match?
6. Which Jira custom field holds story points?
7. Who is on the distribution list, and should different executives get different versions?
8. Where will scheduled runs execute (laptop, server, or CI)?
