# Test Environment Setup

This project runs against a real, self-hosted instance of Mattermost — not a
public demo server and not a purpose-built QA training app. This document
describes how the environment was built and seeded, so the test results in
this repo are reproducible.

## 1. Running the application

Mattermost was run locally via Docker, using the official Docker Compose
setup:

```bash
git clone https://github.com/mattermost/docker
cd docker
cp env.example .env
docker compose up -d
```

This starts two containers: `mattermost` (the application) and `postgres`
(its database).

**Port mapping note:** the default `docker-compose.yml` did not publish port
8065 to the host by default in this setup. This had to be added explicitly
under the `mattermost` service:

```yaml
services:
  mattermost:
    ports:
      - "8065:8065"
```

After adding this and running `docker compose up -d --force-recreate`, the
app became reachable at `http://localhost:8065`.

## 2. Initial account and workspace creation

The first account created on a fresh instance automatically becomes **System
Admin**. This account (`qa_admin`) was used to:

- Complete the onboarding wizard (organization name, team creation)
- Create the team used for all testing: **QA-Test-Org**
- Enable **Open Server** (System Console → Authentication → Signup) and
  **team open-invite** (System Console → Teams), both disabled by default,
  so additional test accounts could self-register and join the team without
  needing individual email invitations

## 3. Test accounts and roles

Four accounts were created to cover the application's role hierarchy, since
testing permissions properly requires more than one account per role:

| Account | Role | Scope |
|---|---|---|
| `qa_admin` | System Admin | Full server access, all teams and channels |
| `teamadmin` | Team Admin | Full access within QA-Test-Org only |
| `member1` | Member | Baseline/control account — used throughout to establish what a *default* Member can do, separate from any Channel Admin promotions |
| `member2` | Member | Used specifically for negative/visibility testing (e.g. confirming a non-member cannot see or search a private channel) |
| `member3` | Member | Promoted to **Channel Admin** on specific channels only, to test whether elevated permissions correctly stayed scoped to that one channel rather than leaking server- or team-wide |

Role assignment for Team Admin and Channel Admin was done via **System
Console → User Management → Teams/Channels**, since the in-channel Members
side panel was found to be missing a role-change control for other users'
rows (see `MMQA-28` in the bug report) — a finding that came directly out of
needing to do this setup step.

## 4. Channels

| Channel | Visibility | Purpose |
|---|---|---|
| Town Square, Off-Topic | Public | Default channels, used for baseline/control testing |
| `general-discussion-renamed` | Public | Primary channel for messaging, threading, and Channel Admin permission testing (renamed mid-project as part of channel-management test cases) |
| `announcements` | Public | Secondary channel used as a control for cross-channel permission comparisons |
| `private-project-alpha` | Private | Used for all private-channel visibility, membership, and file-access-boundary testing |
| `Teamadmin-test-channel` | Public | Created specifically to test Team Admin-scoped actions |

## 5. Permission scheme

Mattermost's **default System Scheme** was used for the majority of testing,
since the goal was to test the application's out-of-the-box behavior. Where
a test needed to confirm that a permission change actually takes effect
(rather than just documenting the default), specific permissions (e.g.
"Manage Channel Members" for All Members) were temporarily toggled off,
re-tested, and restored — this is called out explicitly in the relevant test
case's actual result rather than left as an undocumented, permanent change
to the environment.

## 6. Browser/tooling setup for cross-session and real-time testing

Several findings in this project (see `key-findings.md`, MMQA-32) required
testing with two or more active sessions of different users simultaneously,
and in one case, two sessions of the *same* user. This was done using:

- A normal browser window for one account
- An Incognito/Private window (or a second browser) for a second account,
  to avoid session/cookie collisions between accounts

For the real-time/websocket findings specifically, Chrome/Edge DevTools was
used directly as a testing tool, not just for inspection:

- **Network tab → Socket filter**, to confirm whether a websocket connection
  was active, dropped, or reconnecting
- **Network throttling (Offline / No throttling)**, to simulate a network
  interruption
- Reading individual request **status codes** (e.g. confirming a `403
  Forbidden` on a rejected `DELETE` request) to distinguish a server-side
  enforcement issue from a client-side UI staleness issue

## Scope note

This project covers test design, execution, and defect reporting only.
There is no development team attached to this project, so filed defects
remain in a reported/triaged state (Jira "To Do") rather than progressing
through a fix-and-retest workflow — that step is outside this project's
scope, not an oversight.
