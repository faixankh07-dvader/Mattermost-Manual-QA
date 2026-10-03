# Key Findings

The full defect list (10 bugs) is in
[`Bug_Report_Combined.xlsx`](Bug_Report_Combined.xlsx). This document pulls
out the four most significant findings in readable form — the ones that best
demonstrate the testing approach behind this project, not just its output.

---

### MMQA-32

**Systemic: Real-time client updates are unreliable across multiple event
types** · Priority: Highest · Parent: Channels & Messaging Core

Rather than filing isolated "doesn't update live" bugs each time the symptom
appeared, this ticket ties together five independently-executed test cases
that surfaced the same underlying behavior through different triggers:

| Trigger | Where it surfaced |
|---|---|
| New message from another user | MMQA-6 |
| Channel archived by another user | MMQA-6 (extended scope) |
| Network interruption + reconnect | MMQA-31 |
| Root post deletion with replies | Epic 3 threading test |
| Read-state across a single user's own two sessions | Epic 4 unread-state test |

In every case, server-side state was confirmed correct after a manual
refresh — only the **live push to an already-open, idle client** failed. The
fifth data point (same user, two tabs, no other user or permission logic
involved) was deliberately run to rule out a cross-user-specific explanation;
it still failed, which narrowed the likely cause to a general client-side
state-reconciliation gap rather than anything tied to permissions or
cross-user event delivery specifically.

**Why this matters:** identifying a shared root cause across four unrelated-
looking symptoms, and designing a fifth test specifically to test that
theory, is a different kind of finding than reporting four separate bugs —
it gives a development team one place to look instead of four.

---

### MMQA-27

**Regular Members can remove other members from a channel in some channels
but not others** · Priority: Highest · Parent: Roles & Permissions

**Steps to Reproduce:**
1. Log in as a regular Member (no Channel Admin role in any of the affected
   channels)
2. Open "Off-Topic" → Members panel → "Remove from Channel" is visible next
   to another member's name
3. Open "Teamadmin-test-channel" → Members panel → clicking "Remove from
   Channel" **successfully removes** the target member
4. Open "Town Square" → Members panel → no "Remove from Channel" option
   appears for any member (correct, expected behavior)

**Expected:** Per Mattermost's own in-app guidance ("Only channel admins can
manage other members" — shown correctly in Town Square), a regular Member
should never see or be able to use this option, in any channel.

**Actual:** The action was blocked correctly in one channel and fully
executed in another, by the same user holding the same role throughout — not
just a UI display inconsistency, since the removal in
Teamadmin-test-channel actually went through.

**Why this matters:** this is a genuine RBAC boundary failure. The same
user, same role, same permission scheme, produced two different enforcement
outcomes depending only on which channel they were in — exactly the kind of
inconsistency that's easy to miss if you test a permission once and assume
the result generalizes across the whole app.

---

### MMQA-33

**@mention does not trigger a visible notification badge, even after
refresh** · Priority: High · Parent: Notifications & Mentions

**Steps to Reproduce:**
1. User B has a channel open and idle
2. User A posts a message containing "@UserB"
3. Observe User B's @ mentions icon for a badge update
4. Refresh User B's page
5. Check the @ mentions icon again

**Expected:** A visible badge or alert on the @ mentions icon, at minimum
after a refresh.

**Actual:** No badge appeared at any point — not live, and not after
refresh. The mention was only discoverable by manually opening the Recent
Mentions panel, where it **was** correctly listed and highlighted. Browser
notification permission and the user's own notification preferences were
both confirmed enabled before filing, ruling out configuration as the cause.

**Why this matters:** this is explicitly *not* the same defect family as
MMQA-32 — a refresh fixes every issue in that group, but it does not fix
this one. Distinguishing "the client hasn't caught up yet" from "this
feature doesn't work at all" mattered here, since conflating the two would
have hidden a more serious, unrelated defect inside the systemic ticket.

---

### MMQA-29

**A demoted Channel Admin's open session still shows a Delete option until
refresh** · Priority: Low · Parent: Roles & Permissions

**Steps to Reproduce:**
1. Promote a user to Channel Admin in a channel; confirm they can see a
   Delete option on another user's post
2. Demote them back to Member, without refreshing their open session
3. In that still-open session, click Delete on the post

**Expected:** The Delete option disappears once the role changes, or at
minimum the action is rejected if attempted.

**Actual:** The Delete option remained visible in the stale session.
Clicking it sent `DELETE /api/v4/posts/{id}`, which the server correctly
returned as **403 Forbidden** — confirmed via DevTools Network tab — and the
post was not deleted.

**Why this matters, and why it's Low priority, not Critical:** at first
glance this looks like a privilege-escalation bug. Checking the actual
network request before filing showed the server was enforcing permissions
correctly the whole time; only the UI hadn't caught up. That distinction —
confirmed with evidence, not assumed — is what separates this from MMQA-27,
where the unauthorized action actually succeeded.
