# Implementation Backlog: Weekly Status Report Generator

Derived from [project_spec.md](project_spec.md). Spec references (FR-n, NFR, §n) point to that file.

## Approach

**Build an end-to-end thin slice first:** Jira fetch → metrics → RAG → preview HTML. Email delivery and extras come after the slice works.

**Priority tags**
- **P0:** required for the thin slice (the first usable preview report).
- **P1:** required for the full MVP (delivery, commentary, hardening).
- **P2:** nice to have, after MVP.

**Estimates:** S ≈ under 1 hour, M ≈ 1–3 hours, L ≈ half a day or more.

**Phase mapping.** Because the thin slice needs live Jira data early, the Jira client sits in *Core Features*, not *Integration*:
- **Setup:** environment, config, models, fixtures.
- **Core Features:** the thin slice (Jira client, metrics, RAG, narrative, HTML preview, CLI).
- **Integration:** email delivery, commentary merge, LLM, milestones, scheduling.
- **Testing:** unit, golden-file, and acceptance tests. Write the P0 tests for metrics and RAG as each module lands, not at the end.
- **Documentation:** README, runbook, and so on.

**Assumptions (defaults I chose; please confirm or change)**
- Python environment setup comes first and is currently blocked (see S-1).
- Tasks are fine-grained, with acceptance notes.
- LLM narrative, scheduled runs, Outlook/Gmail drafts, and fix-version milestones are post-slice. Milestones and the plain-text fallback are P1, and the LLM and scheduling are P2.
- Tests are in the Testing phase, but P0 modules get tests as they're built.

---

## Phase 1: Setup

- [ ] **S-1 (P0, M): Get Python 3.11+ and a virtualenv working.** *Blocked:* only Python 3.9.6 is installed. Choose between installing `uv`, installing Python 3.12 from python.org, or lowering the requirement. *Done when:* `python --version` in the venv reports 3.11 or newer.
- [ ] **S-2 (P0, S): Install the project with dev dependencies.** Run `pip install -e ".[dev]"`. *Done when:* `import status_report` works and `pytest` runs (even with zero tests).
- [x] **S-3 (P0, S): Create `pyproject.toml`.** Dependencies and the `report` entry point (§6).
- [x] **S-4 (P0, S): Create `.gitignore`.** Covers `.venv`, `.env`, `config.yaml`, `archive/` (NFR Security).
- [x] **S-5 (P0, S): Create `.env.example` and `config.example.yaml`.** Match §7.
- [x] **S-6 (P0, S): Create the package and test skeleton.** `src/status_report/` and `tests/`.
- [ ] **S-7 (P0, S): Resolve the Jira inputs.** Get the base URL, project key(s), board ID, and the story-points custom field ID (Open Questions 1 and 6). *Done when:* the values are in the local `config.yaml`, which is git-ignored.
- [ ] **S-8 (P0, S): Create a Jira API token and put it in `.env`.** Use a read-only account or scope where possible (NFR Security).
- [ ] **S-9 (P0, M): Implement config loading (`config.py`).** Load `config.yaml` and `.env`, validate with pydantic, and give clear errors for missing keys. Apply the defaults from §7 and keep secrets out of the config model's `repr`.
- [ ] **S-10 (P0, M): Define the data models (`models.py`).** `Sprint`, `Issue`, `SprintMetrics`, `EpicProgress`, `Milestone`, `RagResult` (§8).
- [ ] **S-11 (P0, M): Capture the Jira fixtures.** Save sanitized sample JSON for an active sprint, a closed sprint, and issues with and without epics, points, and blockers into `tests/fixtures/` (§10). *Done when:* the fixtures contain no real names, emails, or tokens.
- [ ] **S-12 (P1, S): Add structured logging and a `--verbose` flag** (NFR Observability). Make sure no secrets are ever logged.
- [ ] **S-13 (P1, S): Add `ruff` config and a lint/test command** (for example in the README or a `Makefile`).

## Phase 2: Core Features (thin slice)

### Jira client (FR-1)
- [ ] **C-1 (P0, M): Build the Jira HTTP client (`jira_client.py`).** Use basic auth with email and token, a base URL from config, and timeouts.
- [ ] **C-2 (P0, M): Add pagination helpers** for the Agile API (`startAt`/`maxResults`) and the search API (token-based paging where applicable).
- [ ] **C-3 (P0, M): Add retry with backoff.** Handle HTTP 429 using `Retry-After`, plus 5xx and network errors. Fail loudly on 401 and 403 with a clear message, and never return partial data (NFR Reliability).
- [ ] **C-4 (P0, M): Fetch the active sprint** for the board: name, goal, start, end, state. If none is active, fall back to the last closed sprint and flag it (§9).
- [ ] **C-5 (P0, L): Fetch sprint issues.** Include status, status category, type, points, epic, priority, labels, flagged state, resolution date, and fix versions. Map them into `Issue` models.
- [ ] **C-6 (P0, M): Derive `added_after_start`** for scope change, using the sprint's changelog or sprint report data. *Done when:* the issues added after sprint start are identified in the fixtures.
- [ ] **C-7 (P0, M): Fetch the previous N closed sprints** and their completed points for velocity (default 3).
- [ ] **C-8 (P0, S): Fetch issues resolved in the last 7 days** (for the "Completed this week" section).
- [ ] **C-9 (P1, M): Fetch bug counts:** opened and closed in the last 7 days, and open Critical/High bugs.
- [ ] **C-10 (P0, S): Add an offline mode** that loads fixtures instead of calling Jira (FR-7 dry-run).

### Metrics (FR-2)
- [ ] **C-11 (P0, M): Committed vs. completed points and percent complete (`metrics.py`).**
- [ ] **C-12 (P0, M): Expected percent from elapsed working days.** Exclude weekends and handle sprints under 1 day old, which show "Too early to assess" (§9).
- [ ] **C-13 (P0, S): Carryover count.**
- [ ] **C-14 (P0, S): Scope change:** points added and removed after sprint start.
- [ ] **C-15 (P0, S): Velocity:** average of the last N sprints compared with the current sprint.
- [ ] **C-16 (P0, M): Blocker detection:** flagged issues, configured blocked labels, and configured blocked statuses.
- [ ] **C-17 (P0, M): Epic progress:** points done over total per epic, with "Unassigned work" for issues without an epic.
- [ ] **C-18 (P1, S): Unestimated-issue fallback.** Use issue counts and surface a data-quality note (§9).
- [ ] **C-19 (P1, S): Bug statistics** from C-9.

### RAG (FR-3)
- [ ] **C-20 (P0, M): Implement the RAG rules (`rag.py`).** Green, Amber, and Red from the thresholds in config, where the worst applicable condition wins.
- [ ] **C-21 (P0, S): Generate the one-line reason** for the chosen status.
- [ ] **C-22 (P0, S): Support a manual override** with a reason, and record `overridden=True`.
- [ ] **C-23 (P0, S): Handle "Too early to assess"** and missing data without crashing.

### Narrative and rendering (FR-4, FR-5)
- [ ] **C-24 (P0, M): Write the deterministic summary generator (`narrative.py`).** Produce a 3–5 sentence executive summary from the metrics and RAG result.
- [ ] **C-25 (P0, L): Build the HTML template (`templates/report.html.j2`).** Follow the section order in FR-4 and the layout in [reports/template.md](reports/template.md), and keep it to about one page.
- [ ] **C-26 (P0, M): Render with Jinja2 (`render.py`).** Hide ticket keys and individual names by default (`show_ticket_keys`, `show_individuals`). Escape all Jira text to prevent injection.
- [ ] **C-27 (P0, S): Write the HTML to a local file and open it** (`preview` mode).
- [ ] **C-28 (P1, M): Inline the CSS** with `premailer` for email-client compatibility.
- [ ] **C-29 (P1, S): Generate the plain-text fallback** (FR-6).

### CLI (FR-7)
- [ ] **C-30 (P0, M): Build the CLI (`cli.py`).** `report generate [--sprint ID] [--mode preview] [--offline]`.
- [ ] **C-31 (P0, S): Handle errors cleanly.** Print a clear message and exit non-zero on any failure, and send nothing on partial data.
- [ ] **C-32 (P0, M): Thin-slice checkpoint.** Run it against the real Jira board and compare the preview with a manual calculation. *Done when:* the PM confirms the numbers and RAG status are right.

## Phase 3: Integration

- [ ] **I-1 (P1, S): Decide the email provider** (Open Question 2). Choose SMTP, Microsoft Graph, or Gmail API, and record the decision in the spec.
- [ ] **I-2 (P1, L): Implement the email delivery module (`delivery.py`).** Compose a multipart message (HTML and plain text) with subject, sender, and recipients from config.
- [ ] **I-3 (P1, M): Implement `draft` mode:** create a draft in the chosen system without sending (default mode).
- [ ] **I-4 (P1, M): Implement `send` mode,** with a confirmation prompt and a guard that blocks sending when the data fetch failed.
- [ ] **I-5 (P1, M): Archive each report** as HTML plus a JSON metrics snapshot in `archive/` (FR-6).
- [ ] **I-6 (P1, M): Add `commentary.md` merging.** Merge PM-written risks, asks, and commentary into the Blockers, Asks, and Summary sections (FR-5).
- [ ] **I-7 (P1, S): Wire the RAG override** to the CLI (for example `--rag AMBER --rag-reason "..."`).
- [ ] **I-8 (P1, M): Milestone tracking** from fix versions: date, percent complete, and on-track or at-risk status. *Depends on* Open Question 3 (which versions to track).
- [ ] **I-9 (P1, S): Add the `--mode draft|send` options** to the CLI.
- [ ] **I-10 (P2, L): Optional LLM narrative.** Add an `--llm` flag, send only aggregated metrics and sanitized titles, and add a redaction hook. Keep it off by default. *Blocked on* Open Question 4 (compliance).
- [ ] **I-11 (P2, M): Scheduling.** Run weekly (Fri 09:00, configurable timezone) in draft mode, via cron or GitHub Actions. *Depends on* Open Question 8 (where it runs).
- [ ] **I-12 (P2, S): Support multiple projects or boards** with per-project sections (§9, marked future).

## Phase 4: Testing

- [ ] **T-1 (P0, M): Unit tests for metrics** (`test_metrics.py`): committed vs. completed, expected percent, carryover, scope change, velocity, blockers, and epic grouping. Include edge cases: no estimates, no epic, a sprint under 1 day old, and weekends.
- [ ] **T-2 (P0, M): Unit tests for RAG** (`test_rag.py`): every threshold boundary (exactly 80% and exactly 60%), blocker and critical-blocker rules, worst-condition-wins, and the override.
- [ ] **T-3 (P0, S): Unit tests for the Jira client** with mocked HTTP: pagination, 429 with `Retry-After`, 5xx retry, and 401 failing loudly.
- [ ] **T-4 (P0, M): Render tests** (`test_render.py`) for ticket-key and name hiding, escaping of hostile Jira text, and a missing-data case.
- [ ] **T-5 (P1, M): Golden-file test for the HTML output** against fixtures (§10).
- [ ] **T-6 (P1, S): No-leak test.** Assert that no tokens, emails, or individual names appear in the output or logs (NFR Privacy and Security).
- [ ] **T-7 (P1, M): Delivery tests** with a fake SMTP or provider: draft creates but doesn't send, and a failed fetch sends nothing.
- [ ] **T-8 (P1, S): Performance check.** A report for about 100 issues finishes in under 60 seconds (NFR Performance).
- [ ] **T-9 (P1, M): Manual acceptance run** of all five criteria in §10, including rendering in Outlook and Gmail.
- [ ] **T-10 (P2, S): Add CI** (GitHub Actions) to run lint and tests on every push.

## Phase 5: Documentation

- [ ] **D-1 (P1, M): Write the README.** Include what it does, setup, a quick start, and CLI usage.
- [ ] **D-2 (P1, S): Document configuration.** Cover every key in `config.example.yaml`, the environment variables, and how to find the story-points field ID.
- [ ] **D-3 (P1, S): Document how to create a Jira API token** and which scope it needs.
- [ ] **D-4 (P1, S): Document the RAG rules** and how to tune the thresholds, linking [reports/instructions.md](reports/instructions.md).
- [ ] **D-5 (P1, S): Write the weekly runbook.** Describe how to generate, review, add commentary, override the RAG status, and send, plus troubleshooting for common Jira and email errors.
- [ ] **D-6 (P2, S): Document the architecture,** updating the diagram and layout in the spec if they changed.
- [ ] **D-7 (P1, S): Update `project_spec.md`.** Fold in the answers to the open questions and any decisions made during the build.

---

## Decisions Needed (from the spec's open questions)

| # | Question | Blocks |
|---|---|---|
| 1 | Jira project key(s) and board ID | S-7, C-4, C-32 |
| 2 | Email system (Outlook/M365 or Gmail) | I-1 to I-4 |
| 3 | Which fix versions to track, and their dates | I-8 |
| 4 | Whether ticket data may be sent to an external LLM | I-10 |
| 5 | A sample report or format to match | C-25 (template polish) |
| 6 | Story-points custom field ID | S-7, C-5 |
| 7 | Distribution list, and whether versions differ per executive | I-2, I-3 |
| 8 | Where scheduled runs execute | I-11 |

## Suggested Order of Work

1. **Unblock:** S-1, S-2, S-7, S-8.
2. **Foundation:** S-9 to S-11.
3. **Thin slice:** C-1 to C-8, C-10 to C-17, C-20 to C-27, C-30 to C-32, with T-1 to T-4 as each module lands.
4. **Delivery and hardening:** I-1 to I-9, C-9, C-18, C-19, C-28, C-29, T-5 to T-9.
5. **Extras and docs:** I-10 to I-12, T-10, D-1 to D-7.
