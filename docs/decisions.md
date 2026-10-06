# Decisions

Why harness is built the way it is. Newest entries at the bottom. Superseded decisions stay, marked as such.

## D001 (2026-10-06): Orchestrator around the official Claude Code engine

- **Question:** Build a custom agent loop, or orchestrate the official one?
- **Decision:** harness is the app around the agent (sessions, UI, approvals, git, config). The agent loop stays Claude Code's, so agents behave exactly like Claude Code with the user's own setup.
- **Rationale:** Keeps agent quality identical, reuses CLAUDE.md, skills, plugins, MCP and hooks, and is the only way to run on a personal subscription for free. A custom loop would need billed API calls.
- **Scope:** Agent behavior is customized only through supported SDK options (MCP tools, subagents, system prompt appends, hooks, per-repo instructions).

## D002 (2026-10-06): Run everything on the MacBook Air

- **Question:** Which machine hosts the daemon and agents?
- **Decision:** The MacBook Air itself, left on.
- **Rationale:** No extra hardware or cost; same environment as the user's current Claude Code setup.
- **Consequence:** Agents pause while the Mac sleeps. Accepted by the user. This supersedes the original note that work should continue with the laptop closed.

## D003 (2026-10-06): Drive agents through the Claude Agent SDK (Python)

- **Options:** A) Agent SDK; B) interactive CLI in tmux with screen scraping; C) detached headless `claude -p` stream-json processes.
- **Decision:** A.
- **Rationale:** Structured events, permission prompts via callback, per-session model, effort, permission mode and attachments as options, loads user settings. B is fragile; C rebuilds what the SDK provides.
- **Consequence:** A daemon restart interrupts an in-flight turn; the session is resumable. The daemon is separate from the Mac app so app restarts never affect agents.
- **Follow-up:** B01 spike verifies settings, plugin and hook loading, resume, the permission callback and claude.ai connectors before building on them.

## D004 (2026-10-06): The daemon performs all git and GitHub actions after approval

- **Options:** A) daemon commits, pushes, opens PRs and merges after UI approval; B) agents commit, only push and PR are approved; C) intercept agent git commands via the permission callback.
- **Decision:** A.
- **Rationale:** Deterministic, testable git logic; the agent can never push regardless of permission mode; review happens before anything is committed. The user's global Claude settings only deny `git push`, so harness enforces its own per-session block of `git commit`, `git push`, `gh pr create` and `gh pr merge`.
- **Flow:** Approve (commit, push, PR), reject, or send feedback into the same agent session. Merge is a separate approval once CI is green.

## D005 (2026-10-06): Existing checkouts, worktree per task by default

- **Options:** A) register existing local checkouts, worktree or in place per task; B) daemon-owned clones.
- **Decision:** A, with worktree as the default for new repos.
- **Rationale:** No duplicate clones to sync; worktrees isolate parallel agents and the user's own edits, as in other parallel-agent tools.
- **Details:** Per-repo TOML in `~/.config/harness/repos/` with `link` (gitignored files such as `.env`, symlinked from the main checkout, never copied) and `setup` (commands run after worktree creation). On repo registration the app suggests link candidates. Worktrees are removed after merge or discard. In-place tasks warn on a dirty checkout.

## D006 (2026-10-06): Free only; native SwiftUI Mac app plus iPhone PWA

- **Options:** A) one SwiftUI app for Mac and iPhone; B) SwiftUI Mac app plus PWA on iPhone; C) wrapped web UI on the Mac.
- **Decision:** B.
- **Rationale:** The user wants a real Mac app and wants the project to be free. A native iPhone app without the paid Apple Developer Program expires every 7 days and cannot receive Apple push. C was excluded because the Mac client must not be a web app.
- **Consequences:** Two client codebases. iPhone notifications use web push (requires the PWA on the Home Screen) with ntfy as a fallback. iPhone voice input uses keyboard dictation.

## D007 (2026-10-06): React, TypeScript and Vite for the PWA

- **Options:** A) React + TypeScript + Vite; B) Svelte + TypeScript; C) server-rendered HTMX.
- **Decision:** A.
- **Rationale:** Strongest agent fluency and mature libraries for diffs, command palette, long streaming lists and PWA support. C is a poor fit for live streaming and is not a typed frontend.
- **Related:** Design tokens live in one file (`design/tokens.toml`) and generate CSS variables and Swift constants so both clients share one look.

## D008 (2026-10-06): Tailscale plus per-device pairing

- **Options:** A) per-device pairing tokens; B) passkeys with Face ID; C) password and session.
- **Decision:** A. Passkeys recorded as a later improvement.
- **Details:** The daemon listens only on localhost and the Tailscale interface. The Mac app reads its token from the Keychain. The iPhone pairs once with a short-lived one-time code shown in the Mac app. Devices are individually revocable. Secrets live in the macOS Keychain; GitHub uses the existing `gh` login.

## D009 (2026-10-06): Coverage scope

- **Decision:** 100% of logic. Daemon: line and branch, excluding only the entry point. PWA: lines, branches, functions and statements including components, excluding only the bootstrap file and generated tokens. Swift: 100% of the `HarnessKit` package; SwiftUI views verified by screenshots and interactive checks.
- **Rationale:** Line-level tests of SwiftUI views are brittle and prove little; keeping logic out of views keeps the guarantee meaningful.

## D010 (2026-10-06): Single repository, config and data locations

- **Decision:** One repository with `daemon/`, `web/`, `apple/`, `design/` and `docs/`. User config in `~/.config/harness/` (TOML). Data in `~/Library/Application Support/harness/`. Modules (providers, integrations, notification channels) are internal and toggled in config; no third-party plugin loading.
- **Rationale:** Simplest layout that keeps the pieces separate; config stays readable and versionable by the user; YAGNI on dynamic plugins.

## D011 (2026-10-06): Development workflow for this project

- **Decision:** Agents: Opus 5.5 medium for implementation, Opus 5.5 high for difficult work, review and coordination. Blocks are large; after the user accepts a block, the coordinator commits it locally as several small conventional commits. The user pushes. Planning files (`plans/`, `AGENTS.md`, `CLAUDE.md`) are never committed; this decisions log and the README are.

## D012 (2026-10-06): Agent SDK spike findings (B01)

Spike run with `claude-agent-sdk` 0.2.163 and Claude Code 2.1.285, mostly on `claude-haiku-4-5` to keep subscription use small.

- **Auth:** works through the existing local Claude Code login (`apiKeySource: none`); no API key or `setup-token` needed.
- **User setup:** with `setting_sources=["user", "project", "local"]` the session loads all enabled plugins, 83 skills, user hooks, MCP servers including claude.ai connectors (Gmail, Calendar, Drive and others connect), and the project CLAUDE.md.
- **Permissions:** `can_use_tool` can hold a prompt open (tested 20 s per prompt) and resolve allow or deny from outside; the deny message reaches the agent.
- **Resume:** resuming by session id works after a clean close and after a hard kill of the CLI process mid-turn.
- **Controls:** `set_model` and `set_permission_mode` work mid-session; `effort` is an option; image content blocks in streamed user messages work; `include_partial_messages` yields token-level stream events.
- **Usage:** every turn emits a `RateLimitEvent` with five-hour and seven-day utilization and reset times. The usage meter (B15) is feasible from this data.
- **User hooks:** the user's iTerm `cc-status` hooks fire without a terminal and cause no errors.
- **Git blocking:** permission rules alone (`disallowed_tools` with `Bash(git commit:*)` etc.) are bypassable: `sh -c 'git commit ...'` and `git -c k=v commit ...` succeeded. See D013.

## D013 (2026-10-06): Layered enforcement of the agent git block

- **Context:** D004 requires that agents never commit, push or open PRs. D012 showed permission rules alone are insufficient.
- **Decision:** Four layers in every agent session:
  1. `disallowed_tools` rules for `git commit`, `git push`, `gh pr create`, `gh pr merge`.
  2. A PreToolUse hook (SDK callback) that denies any Bash command matching git commit or push or gh pr create or merge anywhere in the command, including wrappers and git options. Verified in the spike to block all tested variants.
  3. Agent processes get an environment without git credential helpers or a GitHub token, so a push that slips through cannot authenticate.
  4. After each turn the daemon audits the repository refs; any commit or ref change it did not make raises an alert event and blocks approval until the user decides.
- **Rationale:** No single layer is complete (an agent could write a script that runs git), but together they make an accidental or casual bypass detectable and a push practically impossible. The threat model is a cooperative agent, not an adversary.

## D014 (2026-10-06): Daemon binds to localhost only; Tailscale reaches it via `tailscale serve`

- **Supersedes:** the binding detail in D008 (localhost plus the Tailscale interface).
- **Decision:** The daemon listens only on `127.0.0.1`. Remote devices reach it through `tailscale serve`, which terminates HTTPS on the tailnet name and proxies to localhost.
- **Rationale:** One listening address, no interface detection, HTTPS for free (needed for web push and secure cookies), and nothing reachable from the local Wi-Fi. Device tokens still guard every request.

## D015 (2026-10-06): Daemon platform libraries

- **Decision:** FastAPI with uvicorn for HTTP and websockets; pydantic for config and API models; `aiosqlite` with plain SQL migrations tracked by `PRAGMA user_version`; `keyring` for the macOS Keychain; `tomli-w` to write TOML (reading uses stdlib `tomllib`); `httpx` for tests.
- **Rationale:** Small, widely used, typed libraries; no ORM, since the schema is small and plain SQL is the most readable option.

## D016 (2026-10-06): Git guard matching and the SSH push gap

- **Decision:** The PreToolUse guard matches `commit` and `push` only in git's subcommand position (after global options such as `-C`, `-c`, `--git-dir`), anywhere in a command line, plus `gh pr create|merge`. This avoids blocking harmless commands such as `git log --grep commit` or `git stash push`.
- **Accepted false positives:** text that merely contains a git command (`echo "git commit"`), `git commit-tree`.
- **Known misses:** git aliases and commands assembled from variables or scripts; pushes over SSH remotes, since removing credential helpers does not affect ssh-agent keys and blocking SSH would also break `git fetch`.
- **Mitigation:** the layer 4 ref audit (D013) reports any ref change, including remote-tracking refs updated by a push, and blocks approval until the user decides.

## D017 (2026-10-06): Agent task list and question tool

- **Findings (live, Claude Code 2.1.285):** the CLI has no TodoWrite; agents track work with `TaskCreate`, `TaskUpdate`, `TaskList`, `TaskGet`. Subagents are started with the `Agent` tool (input `subagent_type`, `description`, `prompt`); their messages carry `parent_tool_use_id` pointing at that call. `TaskCreate` input has `subject`, `description`, `activeForm`; its result text is `Task #<id> created successfully: <subject>`; `TaskUpdate` input has `taskId` and `status` (`in_progress`, `completed`). `AskUserQuestion` is available only when a permission callback is set; its input carries questions with options, and Claude marks its preferred option with "(Recommended)" in the label. Answers are returned through `updated_input` with an `answers` map.
- **Decision:** the "Current work" panel is fed from the agent's task list tool calls; the "Agents" panel from `Agent` calls and subagent messages; questions join the approval queue.

## D018 (2026-10-06): Questions must reach the user in every permission mode

- **Context:** `can_use_tool` is not invoked in `bypassPermissions` or for auto-allowed tools, so a question could be skipped silently.
- **Decision:** intercept `AskUserQuestion` with a PreToolUse hook (which runs in every mode), hold it in the approval queue, and return the user's answers as updated input. Verify live in B05.

## D019 (2026-10-06): Approval queue and review rules (B05)

- **One path:** permissions and questions are resolved through `/api/approvals`; reviews through the session's approve, reject and feedback actions. The earlier per-session permission endpoint was removed.
- **Always allow:** allows that tool by name for the rest of the session (persisted). Allowing Bash allows all Bash; the git guard hook still applies.
- **Reviews:** created when a turn ends idle with changes since the last approved commit; at most one pending per session; a new message supersedes it.
- **Git failures:** a failed git or gh step returns 502 with the step and stderr; retries skip steps already done.
- **Tracking:** task list and agents are rebuilt lazily from persisted events after a restart.
- **Questions:** the PreToolUse hook waits up to 7 days, because the SDK default of 60 s would abandon a question.
- **Live evidence:** see B05 in the plan (real PR on a private sandbox repo, questions in two permission modes).

## D020 (2026-10-06): Mac app design locked

- **Decision:** The user locked the "project workspace" design (mockup version 4, kept locally at `plans/design/mac-workspace.html`). It supersedes the four exploratory directions; the user started from the Board idea and reshaped it.
- **Layout:** project tabs like browser tabs (a background project that needs attention glows amber with a count); per project a chat sidebar (pinned, needs you, running, idle, done, archived groups; one click opens a chat; up to four chats side by side in a split workspace, the fourth click replaces the bottom-right pane; drag to split or swap; resizable; maximize) that folds into a rail; a right panel with three cards (Agents, Needs you, Goals) that open and close independently and share the height equally, also foldable into a rail; a Board view per project.
- **Details:** composer with attach, mic and quiet text selectors (model, thinking, permission mode) that open lists; chat actions (pin, rename, continue in Claude Code, archive, delete); project name menu (open in VS Code, Finder, Terminal, copy path); account row at bottom left opening Connectors, gear opening Settings (appearance with theme, accent palette, bubble color and wallpaper; agent defaults; notifications; account); usage popover with 5 hour and weekly limits, monthly sessions and tokens, and a daily token calendar.
- **Visual language:** Liquid Glass panels; neutral graphite default accent (no purple), user-selectable accents (Graphite, Ocean, Teal, Moss, Clay, Rose) and wallpapers (Stone, Sage, Sky, Slate); fixed semantic status colors (running blue, needs you amber, review and goals green, failed red, interrupted grey-brown); minimal text.

## D021 (2026-10-06): iPhone PWA design locked

- **Decision:** The user approved the iPhone companion design (kept locally at `plans/design/iphone.html`), in the same visual language as D020.
- **Layout:** a floating glass tab bar with Inbox (badge with the total), Projects and Account. Inbox gathers approvals, questions and reviews across all projects, grouped by project; cards swipe right to approve and left to reject, with buttons as the accessible path; questions are answered with buttons (Recommended preselected, free text allowed); reviews open a full-screen unified diff. Projects has project chips with attention glow and the chat list grouped like the Mac sidebar, with long-press or swipe actions; a chat opens full screen with a Goals strip, an Agents sheet and quiet selectors opening action sheets. Account holds usage (limits, monthly sessions and tokens, daily calendar), connectors, notifications and appearance.

## D022 (2026-10-06): Chat management, titles, usage and local apps (B20)

- **Titles:** sessions created without a title get a short one from the prompt (first sentence, leading filler words removed, at most 48 characters at a word boundary); branch slugs use it (at most 40 characters plus a short hex suffix).
- **Purge:** deleting with purge removes the run, worktree, branch, attachments, approvals, events and the session row. Token usage rows are kept, because those tokens were really spent and monthly totals should not drop.
- **Usage:** every turn's token counts (input, output, cache creation, cache read) are stored; "total" is the sum of all four. The latest 5 hour and weekly limits are persisted.
- **Connectors:** reported from Claude Code itself; harness shows status and points to claude.ai or Claude Code for sign-in, since it cannot authenticate connectors on the user's behalf.
- **Continue in Claude Code:** opens a generated `.command` launcher in Terminal (`claude --resume <id>` in the session folder), which needs no Automation permission. A warning is returned while harness still holds the session's run.

## D023 (2026-10-06): Design token pipeline (B06)

- **Decision:** `design/tokens.toml` is the single source for colors (light and dark), status colors, the six accents, glass, wallpapers, radii, spacing, typography and motion. `harness design generate` writes `web/src/generated/tokens.css` and `apple/HarnessKit/Sources/HarnessKit/Generated/DesignTokens.swift`; both generated files are committed, and a test fails the gates when they drift from the TOML.
- **Swift shape:** Foundation only (`RGBA`, `ThemedColor`), no SwiftUI import; the Mac app maps tokens to SwiftUI. Durations in seconds, saturation as a factor.
- **Values:** glass blur 20 (cards) and 30 (main panels), saturation 1.85; extra radii for panels, chips and the larger iPhone shapes.
