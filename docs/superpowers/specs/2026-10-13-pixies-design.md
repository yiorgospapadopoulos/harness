# Pixies: persistent agents you can call or leave watching

> **Built in stage 1 (differences from this spec):**
> - The Mac has a Pixies tab before the project tabs (Cmd-0), not a sidebar section.
> - A pixie has project access (Edit by default, Read only, None) and per-pixie skills and tool rules; new pixies work on every project.
> - Mascots are 10 pixel-art images from `design/mascots.toml`, with 10 preset colours plus a custom one.
> - Reviewer and Tester templates, a per-pixie lock, reserved slugs suffixed (`@plan-2`) and the stale-file hash check were added.
> - Not built yet: reordering, resetting Model and Thinking to Default, `@` calls, habits and the autonomy levels (stages 2 to 4). See D072.

Date: 2026-10-13
Status: design approved in conversation, awaiting spec review
Batch: B35 (after B32 to B34)

## Goal

Let the user build a team of named agents ("pixies", singular "pixie") once and then use them two ways:

1. **Call them**: type `@tester check the login flow` in any harness chat and the tester pixie does the work with its own instructions, memory, model and thinking level.
2. **Leave them running**: a pixie keeps its own ongoing chat, runs recurring checks ("habits") and reacts to events in the user's chats, and reports back only when there is something worth saying.

The aim is a team of department experts the user sets up once (projects, permissions, soul, skills, model, thinking) so they never re-explain the same work: some called on demand, some always on and watching.

The design follows Monocode's Monos (hardbeat920/monocode v0.8.0, PRs #773 and #847) closely, with the changes listed in "Differences from Monocode". The project will be renamed to Hex afterwards, as a separate job; this spec uses the current harness names.

## Non-goals

- Pixies in more than one team (each pixie is in one team or none).
- Pixies committing, pushing, merging or publishing anything. No tool exists for it.
- Other providers (Codex, Cursor, ...). Pixies run on Claude Code through the Agent SDK like every harness chat (D001, D003).
- Running habits while the Mac app is quit.

## Glossary

| Term | Meaning |
|---|---|
| Pixie | A saved agent: soul, memory, settings, one ongoing chat, habits. |
| Soul | `SOUL.md`: identity and standing instructions. |
| Memory | `MEMORY.md` (short, always loaded) plus `notes/` and `archive/` (searched on demand). |
| Habit | A recurring or event-triggered task a pixie runs in a fresh background session. |
| Report | A message a pixie run chooses to send to the user; the only way a background run speaks. |
| Card | An interactive item a pixie posts in its chat (PR, session, reply choices, suggested habit). |
| Autonomy | Per-pixie level that decides which harness tools it gets. |
| Team | A named group of pixies with its own `@` handle and always one leader; `@team` has the leader plan and the members do the work (stage 2). |
| Leader | The team member that plans a team call: who does what. |

## Differences from Monocode

| Topic | Monocode | Pixies | Why |
|---|---|---|---|
| Calling from another chat | Not available | `@name` in any chat runs the pixie as a Claude Code subagent | The user's main way to use role agents |
| Harness actions | A local `app` CLI, broad access | In-process SDK MCP tools checked in daemon code, per-pixie autonomy levels | Layered enforcement as in D013; Monocode's PR flags its broad access as high-risk |
| Watching chats | Scheduled polling only | Schedules plus event triggers (stuck, failed, review ready, CI failed) | Reacts at once without spending runs |
| Teams | Not modelled | Named teams with a leader and an `@` handle; the leader plans, the members work (stage 2) | One handle for a group the user set up once |

Kept as in Monocode: soul and memory files with topic notes and an archive, one ongoing chat with rotating provider sessions, assigned projects and skills, model and thinking per pixie, mascot, colour and chat background, habits on hourly, daily, weekday and weekly schedules in local time that run only while the app is open, one run at a time, skipping schedules missed by more than two hours, the latest 20 outcomes, working-time and unanswered-approval limits, approvals relayed into the pixie's chat, quiet unless useful, delegated sessions reporting back as one batch, chat cards, suggested habits, credential redaction in saved facts, and "reset conversation" keeping soul, memory and habits.

## 1. The pixie

### Storage

- Database: a `pixies` table (id, slug, name, mascot, colour, background, model, effort, permission_mode, autonomy, project_access, projects, skills, can_call, allowed_tools, disallowed_tools, limits, notification rules, position, hidden, team_id (stage 2), created/updated), and a `pixie_teams` table (id, name, handle, leader_id, position, created/updated; stage 2). `slug` is unique across pixies and team handles, lowercase `[a-z0-9-]{1,32}`, derived from the name and editable; it is the `@` handle and the subagent name.
- Folder: `<data dir>/pixies/<id>/`
  - `SOUL.md`: identity and standing instructions.
  - `MEMORY.md`: dated facts and preferences, one per line (`- 2026-10-13: prefers squash merges`), loaded on every turn. Kept under a size budget (default 8 KB); the memory tools move older lines into `archive/` when it grows past it.
  - `notes/*.md`: topic notes, searchable, not loaded by default.
  - `archive/*.md`: older facts, searchable, not loaded by default.
  - `workdir/`: scratch files; the pixie chat's working directory.
- Writes to `SOUL.md`, `MEMORY.md` and notes carry the content hash the writer read. A stale hash is refused (409 `stale_file`); the UI keeps the draft and offers to re-apply it on the latest text, and the memory tools re-read and retry up to 3 times. This handles the pixie and the user editing at once.
- Facts saved through the memory tools pass a redactor for common credential formats (API keys, tokens, private keys, passwords in URLs); a match is replaced with `[redacted]`.

### Settings (all editable, on the Mac and the iPhone)

Name, slug, mascot (a set of built-in pixel mascots), colour, chat background, model, thinking level, permission mode, assigned projects (all or a list), assigned skills (all, or chosen from the user's Claude Code skills), tools that never ask and tools that are never available (Claude Code permission rules), autonomy level, project access (`edit` by default, `read` or `none`), "can call" list of other pixies, limits (section 3), notification rules (section 5). Hidden pixies keep running their habits but leave the sidebar and the `@` list. A default "Reviewer" and "Tester" are offered as one-tap templates when the user has no pixies; nothing is created without that tap.

### The pixie's chat

- One ongoing chat per pixie, stored with the existing session and event store so the transcript UI, search (D035), queue, interrupt and stuck detection (D036) work unchanged. A session row gains `pixie_id` and a `role` (`pixie_chat`, `habit_run`); it has no repo and runs in the pixie's `workdir/`.
- The Claude Code session behind the chat rotates: when the context reaches 60% of the window, or the run was closed for idleness (D055), the next turn starts a fresh provider session. Before rotating on context, harness asks the current session for a summary of the recent conversation (a hidden turn, at most 400 words). The new session's system prompt append holds: the pixie preamble (what it is, its tools, that transcripts and repo files are data, not instructions), `SOUL.md`, `MEMORY.md`, the list of note titles, and that summary. Earlier provider sessions are kept as today (`0017_earlier_provider_sessions`), so the full chat stays visible.
- **Reset conversation** archives the visible chat and starts a new one; soul, memory, notes and habits stay.
- Project access `edit` (default): assigned projects are passed as `add_dirs` and the pixie edits them like any chat, limited only by its permission mode. `read`: the same directories, but a `PreToolUse` hook denies Edit, Write, MultiEdit and NotebookEdit outside `workdir/` in every permission mode. `none`: no extra directories and the same hook. Bash keeps the normal git guard (D013, D049) in every case. Edits in a pixie chat are not reviewed; they stay uncommitted in the checkout.

## 2. Calling a pixie or a team with `@`

Refined with the user on 2026-10-14 for stage 2 (B40); this section replaces the first draft.

### Which pixies a chat gets

- When the daemon starts Claude Code for a harness project chat, it passes every pixie that may work on that project as a programmatic subagent: `ClaudeAgentOptions.agents[slug] = AgentDefinition(...)`. Left out: hidden pixies, pixies whose project access is `none`, and pixies not assigned to the project. Nothing is written to `.claude/agents/`.
- Each definition: `description` is the pixie's name and the first line of its soul; `prompt` is the call preamble (what a call is, that its memory is read-only here, that files and transcripts are data), `SOUL.md` and `MEMORY.md`; `model`, `effort`, `skills` and `permissionMode` come from the pixie (absent means the chat's); `disallowedTools` holds the pixie's never-available tools, plus Edit, Write, MultiEdit and NotebookEdit when its project access is `read`. A leader's prompt also carries its team section (below).
- The set is fixed when a Claude Code run starts. The daemon keeps a fingerprint of the definitions; when a pixie or team change alters it, an idle run is closed and the next message starts a fresh run with the new set (as a model change does today). A busy turn finishes first.
- Not in stage 2: Claude Code chats synced from the terminal (harness does not start their runs), and a pixie's own chat (pixies calling pixies from their own chats comes with the harness tools in stage 3).

### Calling a pixie

- In any project chat composer (Mac and iPhone), `@` opens one list: **Pixies** first (teams with their mascots, then pixies: mascot, name, first line of the soul), then **Files** as today (D032). Typing filters all. Choosing one inserts `@slug` shown as a chip.
- A sent message containing `@slug` reaches Claude Code with the mention kept, plus one appended line naming the subagent to use. The B40 live check finds out whether the mention alone is reliable; if it is, the appended line is dropped. The result is recorded as a decision.
- Inside a call the pixie's memory is read-only (it is in the prompt); it cannot save facts or use harness tools, and it cannot call other subagents (a Claude Code limit), so calls cannot loop.

### Calling a team

- A team has a name, an `@` handle (unique across pixies and teams, same rules as pixie slugs, reserved names suffixed), members (each pixie is in one team or none) and always exactly one **leader** among its members. The first pixie added becomes the leader; the user can choose another member; when the leader is deleted or leaves, the next member in list order becomes leader. A team with no members has no leader and cannot be called (it is greyed in the `@` list).
- A team is not a subagent. A message containing `@team` is sent with the mention kept plus appended instructions: first call the leader subagent to plan the request (which member does which part, the leader may give itself a part), then call each member as the plan says, then sum up the results for the user.
- The leader's subagent prompt carries a team section: the team's name and, for each member, its slug, name and description, and how to answer a planning request (a short plan naming members by slug, no work done). The leader plans with its own model, thinking and memory.
- Only members available in the chat (rules above) are listed; if the leader is not available there, the team is left out of that chat.

### How calls show in the chat

- A pixie call shows as the normal agent row (D045), labelled with the pixie's mascot and name (matched by the call's subagent name; an unknown name shows as today).
- Clicking the row opens it in place and shows the pixie's steps and answer: the subagent's own messages, which the daemon already stores with `parent_tool_use_id`, and which the clients hide today.
- Clicking the mascot or name goes to the pixie: the Pixies tab with its pane open on the Mac, its pixie page on the iPhone.

## 3. Harness tools for pixies

Each pixie chat and habit run gets one in-process SDK MCP server (`create_sdk_mcp_server`) named `pixie`, built for that pixie. Every tool checks the pixie's assigned projects and autonomy in daemon code before acting and returns a tool error with the reason when refused.

| Autonomy | Tools (each level includes the ones above) |
|---|---|
| always | `memory_read`, `memory_write(fact)`, `memory_forget`, `notes_search`, `note_read`, `note_write`, `soul_read`, `soul_write` (only when the user asked; the tool description says so), `report(severity, title, body, link?)`, `card(...)`, `habit_list`, `habit_create`, `habit_update`, `habit_pause`, `list_projects` |
| observe | `list_sessions(project?, state?)`, `get_session`, `read_transcript(id, since?, limit)` (condensed: messages, tool call summaries, results, truncated), `list_inbox`, `pr_status` |
| message | `send_message`, `nudge(id, reason)` (the stuck card's stop-and-redirect), `interrupt` |
| start | `start_session(project, prompt, mode, model?, effort?, permission_mode?)`, `close_session` (only sessions it started; a worktree with unreviewed work is kept) |
| answer | `answer_permission(allow/deny)`, `answer_question`, `approve_plan`, `keep_planning` |
| never | approving reviews (commit, push, pull request), merging, removing projects or worktrees with work, changing settings. No tool exists. |

Pixie runs also get the pixies on their "can call" list as subagents (section 2), so a lead pixie can `@reviewer` inside its own run. With `start`, a session it starts may mention a pixie in its first message.

Rules:
- Sessions a pixie starts record `origin = pixie:<id>` and show a "Started by <pixie>" chip; messages it sends show "from <pixie>".
- Every changing call is written to the pixie's activity log (viewable on its page).
- Limits per pixie: running sessions it started (default 2), new sessions per day (default 5). `start_session` cannot ask for a looser permission mode than the pixie's own.
- A pixie cannot act on pixie chats or habit runs (its own or others'); it can only reach them through `@` calls.
- Hand-offs report back: when sessions a pixie started finish (finished, failed, interrupted, review ready), harness waits until the pixie's chat is idle and sends it one grouped message for every session started in the same pixie turn, which starts a pixie turn. While the Mac app is not running (section 4) these messages wait and are delivered when it starts.
- `card` kinds: `pr` (live PR status, D034), `session` (live state with Open), `choices` (up to 4 reply buttons that send the chosen text), `habit_suggestion` (a proposed habit with Start, which creates and enables it; nothing runs until tapped).

## 4. Habits

- A habit has: name, prompt (what to check and what is worth reporting), schedule, triggers, maximum run time (default 20 minutes), enabled.
- **Schedule** (optional): hourly (at minute M), daily (at HH:MM), weekdays (at HH:MM) or weekly (day and HH:MM), in the Mac's local time.
- **Triggers** (optional): `stuck`, `failed`, `review_ready`, `ci_failed`, on sessions in the pixie's projects, excluding pixie chats and habit runs. Events are batched; a triggered habit runs at most once per 5 minutes with every event since its last run in its prompt.
- Created on the pixie's Habits page, by the pixie's `habit_*` tools, or from a suggested habit card.
- **Only while the Mac app runs.** The Mac app sends `POST /api/clients/presence {client: "mac", running: true}` on launch and every 60 s, also while hidden or minimised, and `running: false` on quit; a report lapses after 150 s. The scheduler and triggers act only while a Mac presence is current. Talking to a pixie and `@` calls work whenever the daemon runs, from either client.
- **Scheduler**: a daemon loop like the PR tracker (D034) checks every 15 s. One habit run at a time across all pixies; due runs queue in due order. A scheduled run more than 2 hours late is recorded as `skipped` and not run. Trigger events that arrive while the Mac app is not running are dropped (the next scheduled habit covers the gap).
- **A run**: a fresh provider session with role `habit_run`, the pixie preamble, soul, memory, note titles, the habit prompt (plus trigger events), the pixie's tools and autonomy. It is not shown in the pixie's chat while it runs. A `report` call posts the report into the pixie's chat as a card and starts a pixie turn so the user can reply in context. A run with no report ends silently.
- **Approvals** from a run appear in the pixie's chat and in the inbox as usual. A run waiting on an unanswered approval for 15 minutes, or past its maximum run time, is interrupted and recorded as `stalled` or `timed_out`.
- **History**: the latest 20 outcomes per habit (`reported`, `quiet`, `failed`, `stalled`, `timed_out`, `skipped`) with start time, duration, report title and a link to the run's transcript. Actions: Run now (works whenever the daemon runs), Pause, Resume, Edit, Delete.
- Tables: `pixie_habits`, `pixie_habit_runs` (pruned to 20 per habit).

## 5. Reports and notifications

- `report(severity, title, body, link?)` with severity `info`, `warn` or `action`.
- New notification kind `pixie_report` (on by default) with a per-pixie minimum severity for pushing to the phone (default `action`). Every report appears in the pixie's chat; `warn` and `action` also appear in the inbox. Quiet hours apply (D026).
- The Dock badge and sidebar count unread `action` reports.

## 6. Clients

Mac and iPhone behave alike (D056).

- **Sidebar / Projects tab**: a Pixies section above projects listing pixies (mascot, name, unread dot, a glow while a habit runs). New pixie (+), reorder by drag.
- **Teams in the list** (stage 2): each team is a foldable section with its pixies under it, the leader marked; pixies with no team below the teams. New team next to New pixie; drag a pixie into a team or use Move to team.
- **Pixie row menu** (stage 2; ⋯ on hover and right-click on the Mac, long press and ⋯ on the iPhone): Open, Details, Duplicate, Move to team (each team, No team), Move Up, Move Down, Hide or Show, Delete (confirmed: its chat, soul and memory go too, cannot be undone).
- **Team header menu** (stage 2): Rename, Change handle, Set leader, Delete team (confirmed; the pixies stay, without a team).
- **Duplicate** (stage 2): a copy named "<name> 2" (the next free number) with handle `<slug>-2`, right under the original in the same team, with every setting, the soul, memory, notes and archive; not its chat or scratch files.
- **Pixie page**: the chat (with its background) and a Details panel with tabs Settings, Soul, Memory (facts with add, edit, forget; notes; archive search), Habits (list, history, Run now), Activity (the log).
- **Composer**: the merged `@` list (section 2) in every chat.
- **Cards** render in both clients.

## 7. API and events (outline)

- `GET/POST /api/pixies`, `GET/PATCH/DELETE /api/pixies/{id}`, `POST /api/pixies/{id}/reset`, `POST /api/pixies/{id}/duplicate`; `PATCH` takes `team_id` (null leaves the team)
- `GET/POST /api/pixie-teams`, `PATCH/DELETE /api/pixie-teams/{id}` (name, handle, leader); events `pixie_team.updated` and `pixie_team.deleted` (D077)
- `GET/PUT /api/pixies/{id}/soul`, `/memory`, `GET /api/pixies/{id}/notes`, `GET/PUT /api/pixies/{id}/notes/{name}` (all with content hashes)
- `GET/POST /api/pixies/{id}/habits`, `PATCH/DELETE .../habits/{hid}`, `POST .../habits/{hid}/run`, `GET .../habits/{hid}/runs`
- `GET /api/pixies/{id}/activity`
- `POST /api/clients/presence`
- Events: `pixie.updated`, `pixie.report`, `pixie.card`, `habit.run_started`, `habit.run_ended`.

## 8. Security

- Transcripts, repo files, PR data and trigger events are untrusted. The preamble says so; the real protection is that autonomy, project scope, limits and the "never" list are enforced in the daemon, not the prompt.
- The git guard and commit/push rules apply to every pixie run and every session a pixie starts (D013, D049, D050).
- Pixie folders live in the data dir with the same permissions as the database.
- Memory redaction is best effort and documented as such.

## 9. Testing

- Daemon: unit tests for every tool's autonomy and scope checks, the scheduler (due calculation, DST changes, the 2-hour skip, one-at-a-time, presence lapse), triggers (batching, 5-minute limit, exclusions), rotation (threshold, summary, seeded prompt), hash conflicts, redaction, report routing and grouped hand-off messages. 100% coverage as for the rest of the daemon (D009).
- PWA and HarnessKit: the merged `@` list, chips, cards, pixie pages and habit history, with 100% coverage where the gates require it.
- An end-to-end check with the real SDK that `agents=` subagents and `@` mentions work, recorded as a decision (D072).

## 10. Milestones

1. Pixie core: storage, settings, soul and memory files and tools, the pixie chat with rotation, Mac and iPhone pages.
2. `@` calls: subagents in every project chat, teams with a leader, the merged `@` list, chips, pixie rows that open in place, row and team menus, duplicate.
3. Harness tools: autonomy levels, scope checks, activity log, hand-off report-back, cards.
4. Habits: schedules, triggers, presence, scheduler, history, reports and notifications.

Each milestone ships with its tests, docs (README section, decisions entry) and passing `make check` and `make check-apple`.

## To confirm during planning

- The exact `@` mention form Claude Code honours for SDK-defined agents, and that a team call runs the leader first (B40 live check).
- Whether a hidden summary turn before rotation is cheap enough, or the summary should come from stored events without a model call.
