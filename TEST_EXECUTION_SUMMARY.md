# Test Execution Summary

**Project:** Mattermost QA — Manual Testing
**Application under test:** Mattermost (Entry Edition), self-hosted via Docker
**Test management:** Jira + Zephyr Scale
**Total test cases executed:** 71
**Overall result:** 58 Pass / 13 Fail

Full per-case detail (objective, precondition, BDD steps, expected result,
actual result) is in
[`test-cases/Test-case-report-with-result.xlsx`](test-cases/Test-case-report-with-result.xlsx).
Full bug detail is in
[`bug-reports/Bug_Report_Combined.xlsx`](bug-reports/Bug_Report_Combined.xlsx).

---

## Results by Epic

| Epic | Area | Test Cases | Pass | Fail | Pass Rate |
|---|---|---|---|---|---|
| 1 | Authentication & Account Management | 19 | 19 | 0 | 100% |
| 2 | Roles & Permissions | 21 | 19 | 2 | 90% |
| 3 | Channels & Messaging Core | 15 | 10 | 5 | 67% |
| 4 | Notifications & Mentions | 8 | 2 | 6 | 25% |
| 5 | Search & File Sharing | 8 | 8 | 0 | 100% |
| **Total** | | **71** | **58** | **13** | **82%** |

**Reading the pattern across epics:** Authentication and Search/File Sharing
were fully reliable. Roles & Permissions was largely solid, with two genuine
findings (one a real permission-enforcement inconsistency, one a UI-staleness
issue with no actual privilege leak). Channels & Messaging and — most
notably — **Notifications & Mentions** were the weakest areas, and the
failures in both areas trace substantially back to the same root cause (see
Systemic Finding below), rather than representing 11 unrelated defects.

---

## Bug Summary

| Key | Priority | Summary |
|---|---|---|
| MMQA-32 | Highest | **Systemic:** real-time client updates are unreliable across multiple event types (new messages, reconnection, deletions, cross-session state) |
| MMQA-27 | Highest | Regular Members can remove other members in some channels but not others — permission enforcement inconsistency |
| MMQA-6 | High | Messages and channel-state changes (e.g. archive) do not render live for an idle open session without manual refresh |
| MMQA-30 | High | Channel creation allows invalid characters and duplicate channel names |
| MMQA-31 | High | Websocket reconnects after a network interruption but does not backfill messages missed during the gap |
| MMQA-33 | High | @mention does not trigger a visible notification badge or alert, even after refresh |
| MMQA-34 | High | Per-channel "All messages" notification override does not take effect |
| MMQA-35 | High | Unread message count badge never renders (bold indicator works, the number does not) |
| MMQA-28 | Low | Channel Members panel offers no role-change control for other users (workaround exists via System Console) |
| MMQA-29 | Low | A demoted Channel Admin's open session still shows a Delete option until refresh — server correctly rejects the action (403); no privilege retained |

**Priority distribution:** 2 Highest, 6 High, 2 Low. No defect was inflated to
Critical/Highest without cause — priority reflects actual impact (e.g. a
confirmed, exploitable permission gap vs. a stale UI element that the server
still correctly blocks).

---

## Systemic Finding: Real-Time Update Reliability (MMQA-32)

Four test cases, run independently across three different epics, surfaced the
same underlying symptom through four different triggers:

| Source | Trigger | Finding |
|---|---|---|
| MMQA-6 | New message sent by another user | Not reflected in an idle open session until refresh |
| MMQA-6 (extended) | Channel archived by another user | Same — no live banner/indicator until refresh |
| MMQA-31 | Network interruption + reconnect | Websocket reconnects correctly, but does not backfill messages missed during the gap |
| (traced to MMQA-32) | Root post deletion by another user | Not reflected live; thread correctly removed only after refresh (data integrity itself was correct) |
| (traced to MMQA-32) | Read-state in one of a user's own sessions | Does not propagate to that same user's other open session |

In every case, the **server-side state was confirmed correct** (via refresh,
and in one case via a direct DevTools-captured HTTP 403 showing the server
properly rejecting a stale-permission action). Only the **live client-side
update mechanism** consistently failed. This was treated as one systemic
issue (MMQA-32) rather than five unrelated bugs, with a specific
recommendation to investigate the websocket event handling / client-side
state reconciliation layer as a single root-cause area.

The fifth data point — cross-session sync failing for the *same* user, with
no other user or cross-permission logic involved — was the piece that ruled
out a "cross-user-specific" explanation and pointed toward a general failure
to apply server-pushed state to idle, already-rendered client UI.

---

## Methodology Notes

- **Traceability:** every test case is linked (via Zephyr's Coverage/Work
  Item fields) to its parent Story, and every failed test case that produced
  a defect is linked to that defect's Jira ticket.
- **Control-user testing:** for RBAC and permission-boundary cases, a regular
  Member was tested alongside the role under test wherever a baseline was
  needed, to distinguish "default permission scheme grants this to everyone"
  from "this role specifically has elevated access."
- **Server vs. UI distinction:** where a test showed unexpected behavior,
  DevTools (Network/Socket panels) was used to confirm whether the server
  itself was enforcing correctly — this is what separated MMQA-29 (UI-only
  staleness, Low priority) from MMQA-27 (an actual executed unauthorized
  action, Highest priority), rather than treating every unexpected UI state
  as equally severe.
- **Known limitation:** DevTools' "Offline" network throttling does not
  always fully terminate a persistent websocket connection the way a real
  network outage would; this is noted explicitly in MMQA-31 as a caveat on
  the test method, not treated as fully conclusive on its own.
