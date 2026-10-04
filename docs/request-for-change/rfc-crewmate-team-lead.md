---
title: Crewmate teams with a lead — the whole session tree, a team board, and an App SDK team capability
status: draft
author: iamwhatever
created: 2026-10-04
last-audited: 2026-10-04
audited-at: b3bb213d03
doc-pr:
implementation-prs: []
tracking-issues: []
supersedes: []
superseded-by: []
---

# RFC: Crewmate teams with a lead — the whole session tree, a team board, and an App SDK team capability

**Status:** draft — proposed, nothing built. This document amends screen 09 ("Teams and the team view") of [rfc-crewmates-launch.md](rfc-crewmates-launch.md). Acceptance is the product owner's call, recorded by moving `status` to `accepted`; until then screen 09's three-block decision stands. Every "exists today" claim below was read at main `b3bb213d03` (2026-10-04); citations name files and symbols.

- Author: iamwhatever
- Amends: [rfc-crewmates-launch.md](rfc-crewmates-launch.md) § 4 screen 09.
- Related: [rfc-conductor-work-ledger.md](rfc-conductor-work-ledger.md) (the ledger a lead's lanes keep), [dashboard-templates.md](../system-specs/modules/dashboard-templates.md) (the template registry the team board is the first entry of), [rfc-everything-is-an-app.md](rfc-everything-is-an-app.md) (why the app half is a generic SDK capability and not a built-in).

## 1. Summary

Screen 09 made a team a name plus members and fixed the team view at three blocks. That shape holds for a team of peers. It breaks for a team that one crewmate runs: the crewmate opens conductor sessions, those open workers, and none of that work is visible on the team view. This RFC decides four things:

1. **A team may name one lead.** The lead is a crewmate. Only the owner sets or changes it.
2. **The team view shows the lead's whole session tree.** Lanes and workers under the lead appear in the status strip, and **Needs you** covers every descendant's questions and pending approvals.
3. **A fourth block, the team board.** Its numbers come from a crew-log `team` fold that nobody can write to; its judgment fields are published by the lead.
4. **A generic App SDK team capability.** An app may propose a team in its manifest, read its own team's tree, goals and fold, and embed the host's `TeamView` and `TeamBoard` components. Creating the team, trusting it, approving its tool calls and seeding its lead all stay with the owner.

The first intended user is the harness-rsi app, which runs one lead with several lanes. Nothing in core names it: every piece below is usable by any team and any app.

## 2. Current state

| Part | Today | Where | Gap for a led team |
|---|---|---|---|
| Team record | `{id, name, members}`; written only by owner routes and the CLI, under a file lock | `src/kiro_crew/crew_teams.py` `Team`, `write_teams` | No lead |
| Team view | Three blocks: status strip, Needs you, This week; each reads a member's own pinned thread and activity | `website/src/pages/members/TeamView.tsx` | Sessions a member opens are invisible; their questions and approvals never reach Needs you |
| Crewmate Sessions tab | Direct children whose `created_by` is the crewmate's thread | `website/src/pages/members/MembersPage.tsx` `drivingSessions` | Shows lanes, not the workers under them |
| Session tree | `session/opened.parent` edges, folded per tree | `src/kiro_crew/crew_log/session_tree.py` `fold_tree` | Already has what the tree route needs |
| Crew-log folds | Named folds served from the projection; `radar` is a slot fold one app relies on | `src/kiro_crew/crew_log/projection.py` `FOLD_NAMES`, `SLOT_PROJECTION_NAMES` | No per-team fold; no team mark on a session |
| Dashboard templates | Registry mechanism with no entries | `src/kiro_crew/dashboard_templates/registry.py` `REGISTRY` | Needs a first entry |
| Board contract precedent | Numbers from a fold the publisher cannot touch; judgment fields published by the conductor | `src/kiro_crew/pipeline_board_contract.py` | The split the team board copies |
| Teams routes for apps | Every route calls `_deny_app_caller`; an app token gets 404 | `src/kiro_crew/dashboard/handlers/teams.py`, `handlers/members.py` `_deny_app_caller` | An app cannot see any team |
| Work ledger board for apps | `require_owner_dashboard_request` | `src/kiro_crew/dashboard/handlers/work_ledger_board.py` | An app cannot read a lead's goals |
| App backend context | `cron`, `events`, `storage`, `spawn`, `job`, `audit`, … — no team | `src/kiro_crew/apps/context.py` `AppContext` | No team handle |
| App page SDK | Host components exist (`ChatEmbed`, `ChatPanel`) | `website/src/app-sdk/index.ts` | Precedent for embedding host UI; no team component |
| `permissions.sessionApproval` | Lets an app act on the user's own sessions after a consent re-prompt | `src/kiro_crew/apps/manifest.py` `Permissions`, `apps/manager.py` | Must not reach a team's sessions (§ 4.4) |

Built-in apps avoid these walls by importing `kiro_crew` in the gateway process. An external app cannot, and a core change written for one app is the thing [rfc-everything-is-an-app.md](rfc-everything-is-an-app.md) rules out.

## 3. Goals and non-goals

Goals:

- A person can see, from one team view, everything a led team is doing and everything waiting on them.
- Team-board numbers are computed, not reported, so a lead cannot overstate its own progress.
- An app can ship a led team without core code written for that app, and without any authority the owner did not click.

Non-goals:

- More than one lead per team, or a lead that is not a crewmate.
- Letting a lead, lane or app edit team membership, trust, or approvals.
- Letting an app register its own dashboard template or add fields to the team board.
- Changing screen 09's rule that a crewmate is on at most one team.

## 4. Design

### 4.1 A team may name one lead

- `crew_teams.Team` gains an optional `lead: {name}`. The name must be one of the team's members and must be a crewmate. A team without a lead behaves exactly as screen 09 describes.
- Only the owner writes it, through the existing owner routes and the CLI. `crew-teams/` stays sealed from the sandbox and from agent file tools, as it is today.
- The team dialog gains a "Led by" picker. The roster shows the lead's face beside the team header.
- The record also gains an optional `app` (the app that proposed the team, § 4.4). It is stamped by the host when the owner confirms the team and is never writable through an app token.

### 4.2 The team view shows the whole tree

- A new owner route, `GET /api/teams/{id}/tree`, walks down from the lead's thread with `session_tree.py` and returns every descendant: session key, role (lead, lane, worker), depth, state, and counts of open questions and pending approvals. It returns no transcript content.
- `TeamView.tsx` reads it. The status strip indents lead → lanes → workers. **Needs you** includes every descendant's questions and pending approvals, each with the session it came from.
- A team without a lead keeps today's per-member reading.

### 4.3 The team board, a fourth block

- `session/opened` gains an optional `team` field: when the root of a new session's chain is a team's lead, the host stamps the team id. A later membership change does not rewrite history.
- `projection.py` gains a generic `team` fold over those entries: sessions opened, rounds, credits, host auto-denials, per lane. The fold knows nothing about any app.
- `dashboard_templates/registry.py` gets its first entry, `team-board`: page, contract, provider and an alignment test, following `pipeline_board_contract.py`. Numeric fields come from the `team` fold and are not writable by the lead; judgment fields (round label, per-lane note, next step) are published by the lead.
- The team view shows the board above the status strip when the team has a lead. Screen 09's three blocks are otherwise unchanged; this is the fourth.

### 4.4 App SDK team capability

One rule covers every item: a team is visible to an app only if its record carries that app's `app` stamp, set by the host when the owner confirmed it. An app sees only its own teams — never another team, another crewmate, or any conversation text.

The stamp follows the install, not the name. Uninstalling the app clears `app` from every team it proposed, so the teams turn into ordinary owner teams. A later install under the same name gets no access to them: it shows its create card again, and the owner either links an existing team or creates a new one. While the app is disabled, `ctx.team` is not issued and its page is not mounted.

| Capability | Seam | What the app author gets |
|---|---|---|
| Manifest `contributes.teams[]`: `{id, name, lead: {agent, crewName}, lanes: [{id, agent}]}`, where `agent` names an agent JSON the app ships | `apps/manifest.py` validation; `apps/bridges.py` installs the agents as today; `crew_teams.py` `app` field | On enable, the host shows an owner-confirmed "Create team" card (same consent pattern as `sessionApproval` in `apps/manager.py`). Nothing is created until the owner clicks; crewmate and team are created through the owner routes |
| Read the tree | `handlers/teams.py`: only the tree route swaps `_deny_app_caller` for an own-team check; every other teams and members route keeps `_deny_app_caller` | `useTeamTree(teamId)` on the page; `ctx.team.tree(team_id)` in the backend |
| Read goals | `handlers/work_ledger_board.py`: a read-only entry admitted only when the ledger's session is in the app's own team tree | `useTeamGoals` / `ctx.team.goals(team_id)`: lane goals, item titles, states, acceptance kinds — no worker report text |
| Read the fold | New `apps/team_sdk.py`; `apps/context.py` adds `team: TeamSDK \| None`, set only when `permissions.team == "read"` and the manifest declares a team | `ctx.team.fold(team_id)`: the `team` fold, read-only |
| Host components | `website/src/app-sdk/index.ts` exports `TeamView` and `TeamBoard`, following `ChatEmbed` | One line embeds the tree, Needs you and the board, **read-only**. These components run in the app's page and call through `useAppApi()` with the app token, as `ChatEmbed` does, so they hold no owner authority. Each Needs you item carries an "Open in Kiro Crew" link to the host team view, where the owner answers or approves under the owner session. The app cannot add board fields |
| Propose a seed | `ctx.team.propose_seed(team_id, text)` adds a card to the team's Needs you | "App X wants to send the lead: …". It is sent only when the owner clicks Send, through the existing thread path |

Not in the SDK, by design: creating or editing a team, setting the lead, changing members, turning trust on or off, approving a tool call in a team session, sending to the lead directly. `permissions.sessionApproval` does not extend to sessions in a team that carries an `app`.

Each implementation PR for this section updates `docs/app-kit/manifest-reference.md` and `docs/app-kit/api-reference.md` in the same change.

## 5. Migration plan

Each phase is one PR. The core phases come first; each app phase follows the core phase it reads.

| Phase | Change | Exit criteria |
|---|---|---|
| 1 Lead | `Team.lead` and `Team.app` in `crew_teams.py`; owner-only write; "Led by" in the team dialog | A team round-trips with and without `lead`; a lead outside the members list is refused; an app token gets 404 on every write |
| 2 Tree | `GET /api/teams/{id}/tree`; `TeamView.tsx` reads it | A worker two levels under the lead shows in the strip; its pending approval shows in Needs you; the response holds no transcript text |
| 3 Goals | The lead's Goals tab reads the work-ledger board across the lead and its lanes | Goals lists each lane's goal and items |
| 4 Team mark and fold | `session/opened.team`; `team` fold in `projection.py` | A session opened under a lead carries the team id; removing a member later leaves the fold's history unchanged |
| 5 Team board | `team-board` in `registry.py` with contract, provider, alignment test | A lead publish that writes a numeric field is refused; the board renders above the strip |
| A1 Manifest + create card | `contributes.teams`; owner-confirmed create card | No team or crewmate exists until the owner clicks; the record carries `app` |
| A2 Tree for apps | Own-team admission on the tree route; `useTeamTree`, `ctx.team.tree`, read-only `TeamView` export; stamp cleared on uninstall | An app reading another app's team, or a team with no `app`, gets 404; an app-token approve or answer call on a team session is refused; after uninstall and reinstall under the same name, the old team is not visible |
| A3 Goals and seed | Own-team read entry on the ledger board; `propose_seed` card | A proposed seed is not delivered until the owner clicks Send |
| A4 Fold and board for apps | `apps/team_sdk.py`, `ctx.team.fold`, `TeamBoard` export | `ctx.team` is `None` without `permissions.team: read` |

Order: 1 → 2 → 3; 4 may run beside 1–3; 5 waits for 2 and 4; each A phase waits for its core phase, and A2 waits for A1.

## 6. Backward compatibility

- `lead`, `app` and `session/opened.team` are optional. Existing `teams.json` documents and crew-log entries read unchanged; a team without a lead renders exactly as screen 09 decided.
- No existing route changes for app tokens except the tree route, which is new.
- No existing app gains any capability without declaring `permissions.team` and being re-enabled.

## 7. Security considerations

- The tree route opened to apps exposes session state and counts for that app's own team only — no conversation text. Every other teams and members route keeps `_deny_app_caller`.
- Team creation, lead, members and trust stay owner-only: `_require_owner` routes, sealed `crew-teams/`, no SDK call.
- Approval stays in host-owned UI. The embedded components are read-only and carry only the app token; answering a question or approving a tool call in a team session happens on the host team view. Two tests pin it: an app-token approve or answer call on a team session is refused, and an app holding `sessionApproval` is refused when it tries to switch a team session to Trust.
- The `app` stamp is cleared on uninstall, so a later app reusing the name inherits no team.
- Board numbers come from the fold; a lead cannot publish them.

## 8. Alternatives considered

- **A lead that is a conductor session, not a crewmate.** Rejected: a conductor has no standing identity or memory, and the team view is keyed on crewmates.
- **Let each app read the crew log directly**, as built-in apps do. Rejected: it only works in-process and turns each app's needs into core code.
- **Let apps register their own dashboard templates.** Rejected for now: the board's value is that its numbers are not the publisher's to set; an app area beside the embedded board covers app-specific numbers.
- **Extend `sessionApproval` to team sessions.** Rejected: it would let an app approve tool calls in sessions the owner trusted for a different purpose.

## 9. Open questions

1. Whether a session may appear in two teams' trees. This RFC assumes no: a session belongs to the tree of the one lead its chain starts from.
2. Whether the create card should also offer to turn on trust for the lead, or keep trust as a separate owner click. This RFC keeps it separate.
