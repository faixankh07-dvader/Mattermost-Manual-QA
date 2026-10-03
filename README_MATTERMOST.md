# Mattermost QA — Manual Testing Project

A real-application manual QA project: full test planning, execution, and defect
reporting against a self-hosted, production-grade application — not a QA
demo/sandbox app. Built to practice the complete manual testing lifecycle the
way it's actually done on a real team: Jira for work tracking, Zephyr Scale
for test case management and execution, and a genuine multi-role environment
(System Admin / Team Admin / Members) rather than a single test account.

## Why a real app instead of a demo site

Most manual-QA portfolio projects test the same handful of purpose-built demo
apps (OrangeHRM, Demoblaze, etc.) — applications designed to be tested, with
shallow, predictable bugs. This project instead targets **Mattermost**, a
real, actively-developed, open-source team collaboration platform, self-hosted
locally via Docker. That means real complexity: a 3-layer role system (System
Admin / Team Admin / Channel Admin), a live websocket-based messaging layer,
real permission schemes, and genuine, previously-unknown defects — not bugs
seeded into a training app.

## Key Findings

- **71 test cases** executed across 5 epics, covering authentication, RBAC,
  channel/messaging functionality, notifications, and search/file sharing
- **58 passed / 13 failed**
- **10 bugs filed** in Jira, each with full repro steps, environment details,
  and expected vs. actual results
- **One systemic finding ([MMQA-32](bug-reports/key-findings.md#mmqa-32)):**
  rather than filing four unrelated "doesn't update live" bugs, testing traced
  four independent failures — new message delivery, websocket reconnect
  backfill, deletion propagation, and cross-session unread-state sync — to a
  single likely root cause in the client's real-time update handling. Each
  failure was isolated with its own repro and evidence (including DevTools
  Network/Socket captures) before being tied together, so the pattern is
  demonstrated, not asserted.
- **Notifications proved to be the least reliable area of the app**, with 6
  of 8 test cases failing — ranging from a missing @mention notification
  badge to a non-functional unread-count badge.
- **A real permission inconsistency ([MMQA-27](bug-reports/key-findings.md#mmqa-27),
  Highest priority):** a regular Member was able to remove another member from
  some channels but not others, within the same team and permission scheme —
  confirmed as an actual executed action, not just a UI display issue.

See [`TEST_EXECUTION_SUMMARY.md`](TEST_EXECUTION_SUMMARY.md) for the full
per-epic breakdown and bug list.

## Scope

| Epic | Area | Test Cases | Pass | Fail |
|---|---|---|---|---|
| 1 | Authentication & Account Management | 19 | 19 | 0 |
| 2 | Roles & Permissions | 21 | 19 | 2 |
| 3 | Channels & Messaging Core | 15 | 10 | 5 |
| 4 | Notifications & Mentions | 8 | 2 | 6 |
| 5 | Search & File Sharing | 8 | 8 | 0 |
| **Total** | | **71** | **58** | **13** |

## Tools & Environment

- **Application under test:** [Mattermost](https://mattermost.com/) (Entry
  Edition), self-hosted via Docker Compose, running locally
  (`http://localhost:8065`)
- **Test management:** Jira + Zephyr Scale (test cases, test cycles,
  execution tracking, traceability)
- **Defect tracking:** Jira (linked to both the originating test case and the
  relevant Epic/Story)
- **Test environment:** 4 seeded accounts across 3 roles (System Admin, Team
  Admin, regular Member), multiple public/private channels, and
  cross-session/multi-browser setups for concurrency and permission testing
- **Browser tooling:** Chrome/Edge DevTools (Network and Socket panels) used
  directly as testing evidence for the real-time/websocket findings — not
  just UI observation

## Repository Structure

```
.
├── README.md                     — you are here
├── TEST_EXECUTION_SUMMARY.md     — detailed results, full bug list, findings
├── test-cases/
│   └── Test-case-report-with-result.xlsx   — all 71 test cases: objective,
│         preconditions, BDD steps, expected/actual result, pass/fail status
├── bug-reports/
│   ├── Bug_Report_Combined.xlsx  — all 10 filed defects, full detail
│   └── key-findings.md           — the 3 strongest bugs, in readable form
├── screenshots/
│   └── (Jira/Zephyr evidence — project structure, bug detail, test
│         execution, traceability, test cycles)
└── environment-setup/
    └── setup.md                  — how the test environment was built
```

## How This Project Differs From a Checklist

Beyond running test cases to pass/fail, this project emphasizes:

- **Traceability** — every test case links to its Story and, where it failed,
  to the resulting defect ticket, forming a real requirement → test →
  execution → defect chain (a gap identified and deliberately fixed from
  earlier practice projects).
- **Root-cause synthesis over isolated bug counts** — see MMQA-32 above.
- **Distinguishing server-side correctness from UI staleness** — several
  "failures" (e.g. [MMQA-29](bug-reports/key-findings.md)) were confirmed via
  DevTools to be UI-only staleness, with the server correctly rejecting the
  underlying action (HTTP 403) — a meaningfully different, lower-severity
  class of defect than a real privilege leak, and reported as such rather
  than over-stated.
- **Negative and boundary testing** throughout, not just happy-path
  verification — permission boundaries, input limits, non-member visibility
  checks, and cross-role comparisons.
