# How to Fill Out the Weekly Status Report

Use [template.md](template.md) as the starting point and see [example.md](example.md) for a completed report. The audience is executive management, so keep the report short (about one page), lead with status, and write in business terms.

## General Rules

- **Copy the template, don't edit it.** Save each report as `YYYY-MM-DD-status.md` (the report date).
- **Replace every `{{placeholder}}`.** Delete any section that truly doesn't apply, except Blockers and Asks, which should say "None" instead.
- **Write for executives.** Describe outcomes and impact, not tickets. Leave out Jira keys, technical jargon, and implementation detail.
- **No individual names** in the body. Use roles or teams (for example "Backend team", "Product Owner"). The "Prepared by" line is the only exception.
- **Be specific and honest.** Use numbers and dates. Bad news early is better than bad news late.
- **Data cutoff.** Pull Jira numbers at one point in time and record that time in the footer.

## Section by Section

### Header
- **Sprint:** the active sprint name and its start and end dates. If the team is between sprints, use the last closed sprint and say so.
- **Report date:** the day you send it, in `YYYY-MM-DD`.

### Overall Status (RAG)
Pick one value and give a one-line reason. Use these default rules, and apply the worst condition that matches:

| Status | When to use it |
|---|---|
| **GREEN** | Completion is at least 80% of expected, and there are no open blockers. |
| **AMBER** | Completion is 60–80% of expected, or any open blocker, or a milestone is at risk. |
| **RED** | Completion is below 60% of expected, or a critical blocker, or a missed milestone. |

"Expected completion" is the share of the sprint's working days that have elapsed. For example, on day 6 of a 10-day sprint it is 60%. If the sprint is under one day old, write "Too early to assess" instead of a colour.

You may override the calculated colour when you have context the numbers don't. If you do, say so in the reason line (for example "AMBER (manual override): ...").

### Executive Summary
Three to five sentences covering:
1. Where the sprint stands against its goal.
2. The most important change since last week.
3. The one thing leadership should know or do.

### Key Metrics
- **Committed points:** story points in the sprint at sprint start.
- **Completed points:** points for issues in a Done status category, with the percentage of committed.
- **Expected completion:** see the RAG rules above.
- **Velocity:** points completed this sprint compared with the average of the last 3 closed sprints.
- **Scope change:** points added to and removed from the sprint after it started.
- **Carryover:** issues carried over from the previous sprint.
- **Bugs:** opened and closed in the last 7 days.

If the team doesn't estimate with points, use issue counts and note "counts, not points" under the table.

### Completed This Week
Two to five bullets, grouped by epic or product area. Describe what users or the business can now do, not which tickets were closed.

### In Progress / Planned Next Week
Two to five bullets with the same grouping. Mention anything that may slip.

### Blockers and Risks
One row per item. Include only things that threaten scope, dates, or quality.
- **Impact:** what happens if it isn't resolved, and by when.
- **Owner:** a role or team, not a person.
- **Severity:** Low, Med, High, or Critical. Any Critical item makes the overall status RED.
- Write "No open blockers or risks." if there are none.

### Asks of Leadership
List decisions, approvals, or resources you need, each with a needed-by date. Write "None this week." if there are none. Don't bury asks in the summary.

### Milestones and Releases
One row per tracked release or fix version. Set the status like this:
- **On track:** percent complete is in line with the time elapsed.
- **At risk:** behind the plan, but recoverable.
- **Late:** the target date will be missed without intervention.
- **Done:** released.

### Epic Progress
Points done over total points for each active epic, with a percentage. Put work with no epic under "Unassigned work."

### Footer
Name the Jira board or project and the date and time you pulled the data, with timezone.

## Pre-Send Checklist

- [ ] No `{{placeholders}}` remain.
- [ ] The RAG colour matches the rules, or an override is explained.
- [ ] Every blocker has an owner (role or team) and a next step.
- [ ] Every ask has a needed-by date.
- [ ] No individual names, Jira keys, or credentials appear in the body.
- [ ] Numbers add up (completed ≤ committed, percentages are consistent).
- [ ] The report fits on about one page.
- [ ] The subject line follows the format: `[Weekly Status] <Project> | <Sprint> | <RAG> | <Date>`.
