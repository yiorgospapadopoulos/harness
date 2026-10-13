# Pixies: persistent agents you can call or leave watching

Date: 2026-10-13
Status: design approved in conversation, awaiting spec review
Batch: B34 (after B32/B33)

## Goal

Let the user build a team of named agents ("pixies", singular "pixie") once and then use them two ways:

1. **Call them**: type `@tester check the login flow` in any harness chat and the tester pixie does the work with its own instructions, memory, model and thinking level.
2. **Leave them running**: a pixie keeps its own ongoing chat, runs recurring checks ("habits") and reacts to events in the user's chats, and reports back only when there is something worth saying.

The aim is a team of department experts the user sets up once (projects, permissions, soul, skills, model, thinking) so they never re-explain the same work: some called on demand, some always on and watching.

The design follows Monocode's Monos (hardbeat920/monocode v0.8.0, PRs #773 and #847) closely, with the changes listed in "Differences from Monocode". The project will be renamed to Hex afterwards, as a separate job; this spec uses the current harness names.

## Non-goals

- Named teams as a separate object. A team is just pixies that may call each other (a per-pixie "can call" list); a lead is a pixie whose soul tells it to delegate.
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

## Differences from Monocode

| Topic | Monocode | Pixies | Why |
|---|---|---|---|
| Calling from another chat | Not available | `@name` in any chat runs the pixie as a Claude Code subagent | The user's main way to use role agents |
| Harness actions | A local `app` CLI, broad access | In-process SDK MCP tools checked in daemon code, per-pixie autonomy levels | Layered enforcement as in D013; Monocode's PR flags its broad access as high-risk |
| Watching chats | Scheduled polling only | Schedules plus event triggers (stuck, failed, review ready, CI failed) | Reacts at once without spending runs |
| Teams | Not modelled | Pixies may call the pixies on their "can call" list | Lets a lead pixie delegate |

Kept as in Monocode: soul and memory files with topic notes and an archive, one ongoing chat with rotating provider sessions, assigned projects and skills, model and thinking per pixie, mascot, colour and chat background, habits on hourly, daily, weekday and weekly schedules in local time that run only while the app is open, one run at a time, skipping schedules missed by more than two hours, the latest 20 outcomes, working-time and unanswered-approval limits, approvals relayed into the pixie's chat, quiet unless useful, delegated sessions reporting back as one batch, chat cards, suggested habits, credential redaction in saved facts, and "reset conversation" keeping soul, memory and habits.

## 1. The pixie

### Storage

- Database: a `pixies` table (id, slug, name, mascot, colour, background, model, effort, permission_mode, autonomy, project_access, projects, skills, can_call, allowed_tools, disallowed_tools, limits, notification rules, position, hidden, created/updated). `slug` is unique, lowercase `[a-z0-9-]{1,32}`, derived from the name and editable; it is the `@` handle and the subagent name.
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

## 2. Calling a pixie with `@`

- In any chat composer (Mac and iPhone), `@` opens one list: a **Pixies** section first (mascot, name, first line of the soul), then **Files** as today (D032). Typing filters both. Choosing a pixie inserts `@slug` shown as a chip.
- Every harness chat's run passes the pixies assigned to its project as programmatic subagents: `ClaudeAgentOptions.agents[slug] = AgentDefinition(description, prompt, model, effort, skills, permissionMode)`, where `prompt` is the preamble, `SOUL.md` and `MEMORY.md`. Agent SDK 0.2.163 supports all of these fields. Nothing is written to `.claude/agents/`.
- A sent message containing `@slug` reaches Claude Code with that mention kept, plus an appended line that names the agent to use, so the chat's Claude hands that part to the subagent. Planning confirms the exact mention syntax Claude Code honours through the SDK and drops the appended line if the mention alone is reliable.
- Inside an `@` call the pixie's memory is read-only (it is in the prompt); it cannot save facts or use harness tools, because the subagent runs in the calling chat's process and permissions. Claude Code subagents cannot start further subagents, so calls cannot loop.
- Its work shows in the calling chat as a normal agent row (D045), labelled with the pixie's mascot and name.

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
- **Pixie page**: the chat (with its background) and a Details panel with tabs Settings, Soul, Memory (facts with add, edit, forget; notes; archive search), Habits (list, history, Run now), Activity (the log).
- **Composer**: the merged `@` list (section 2) in every chat.
- **Cards** render in both clients.

## 7. API and events (outline)

- `GET/POST /api/pixies`, `GET/PATCH/DELETE /api/pixies/{id}`, `POST /api/pixies/{id}/reset`
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
- An end-to-end check with the real SDK that `agents=` subagents and `@` mentions work, recorded as a decision (D057).

## 10. Milestones

1. Pixie core: storage, settings, soul and memory files and tools, the pixie chat with rotation, Mac and iPhone pages.
2. `@` calls: subagents in every chat, the merged `@` list, chips.
3. Harness tools: autonomy levels, scope checks, activity log, hand-off report-back, cards.
4. Habits: schedules, triggers, presence, scheduler, history, reports and notifications.

Each milestone ships with its tests, docs (README section, decisions entry) and passing `make check` and `make check-apple`.

## To confirm during planning

- The exact `@` mention form Claude Code honours for SDK-defined agents.
- Whether a hidden summary turn before rotation is cheap enough, or the summary should come from stored events without a model call.
