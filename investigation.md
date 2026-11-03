# Investigation: How a Claude Code Web Session Loads Context

Date of session: 2026-11-03 (all times UTC, taken from file timestamps where
given). Repo: `claude-code-web-context`, working branch
`claude/compassionate-ptolemy-7tbffc`.

This document records what was tested in one cloud Claude Code session, what
was observed, and what remains unknown. Findings are labeled **Measured**
(directly observed in the session) or **Inferred** (a reasonable explanation
that was not verified). Personal details (email address, mailbox contents,
account identifiers) are deliberately left out.

## 1. Summary of findings

1. The session runs the Claude Code harness inside a disposable VM as `root`
   with `HOME=/root`. The home directory comes from the container setup, not
   from the user's claude.ai account. (Measured)
2. The only account-level content that reaches the session is a set of synced
   skills under `~/.claude/skills/synced/`. The Desktop/Web profile and
   preferences files (`profile.md`, `preferences.md`, `topics/`, `areas/`,
   `people/`, `projects/`) are not present and were not loaded. (Measured)
3. Project context comes from the repo clone. `.claude/CLAUDE.md` was loaded at
   startup. (Measured)
4. A rule file added to `.claude/rules/` on GitHub mid-session was **not**
   loaded after a fetch, **not** loaded immediately after the merge that put
   it on disk, and **was** injected several turns later as a system reminder.
   What triggered the reload is unknown. (Measured; cause Inferred)
5. Auto memory is not operating in this session. There is no memory directory,
   no memory instructions in the system prompt, and "remember X" wrote
   nothing to disk. (Measured)
6. A project agent added to `.claude/agents/` mid-session was **not** added to
   the agent list and the Agent tool rejected it by name. This matches the
   documented restart requirement when the `agents/` directory did not exist
   at session start. (Measured)
7. The Gmail connector is available and a read-only search worked. (Measured)

## 2. Starting context

This is what the session provided before the first user question.

### System prompt
- Identity and harness rules: Claude Code in a cloud container via the Agent
  SDK; how output is displayed; permission modes; how to treat pasted content
  and hook output; preference for dedicated file/search tools.
- Artifact and Claude Docs rules.
- Remote-environment details: ephemeral container, disk allowance,
  pre-installed Chromium/Playwright, outbound traffic via an agent proxy with a
  CA bundle under `/root/.ccr/`.
- GitHub rules: use the GitHub MCP tools instead of `gh` (except `gh api` for
  search), attribution footer on every GitHub post, do not open PRs unless
  asked, rules for handling PR activity events, and a repository scope limited
  to this repo.
- Git rules: develop on the designated branch, push with `-u origin <branch>`,
  retry with backoff on network errors, restart a merged branch from the
  default branch.
- Model-identity rules (check the session tool rather than guess).

### System-reminder blocks in the first user turn
- The project `.claude/CLAUDE.md` contents (`Keyword: Umbrella`).
- The user's email address (identification only; omitted here).
- Attribution lines to append to commits and PR descriptions.

### Environment block
- Primary working directory `/home/user/claude-code-web-context`, a git repo on
  Linux with bash; a scratchpad directory; the date; knowledge cutoff.
- About 30 deferred tools listed by name only (schemas load on demand through
  `ToolSearch`), including task tools, cron tools, `WebFetch`, `WebSearch`,
  and the Claude Docs tools.
- MCP servers still connecting at that moment: Gmail and GitHub.
- Available agent types: `claude`, `claude-code-guide`, `Explore`,
  `general-purpose`, `Plan`, `statusline-setup`.
- MCP server instructions (Claude Docs).
- A skills listing of roughly 25 entries: bundled skills (e.g. `loop`,
  `simplify`, `code-review`, `claude-api`, the `artifact-*` skills,
  `session-start-hook`) and account-synced `anthropic-skills:*` entries
  (docx, pdf, xlsx, pptx, deep-research, browser and computer-use skills, etc.).

### What was not present at startup
- No user-level `~/.claude/CLAUDE.md`, no memory files, no profile or
  preferences files, no rules.
- No repo contents beyond `.claude/CLAUDE.md` (the README was not injected).
- No tool schemas for the Gmail or GitHub MCP servers (still connecting).

## 3. Environment findings

| Item | Observation | Status |
|---|---|---|
| OS user | `root` (uid 0), `HOME=/root`, hostname `vm`, `USER` unset | Measured |
| Other users | A `claude` user with `/home/claude` (some entries in it are root-owned) and an `ubuntu` user (uid 1000) | Measured |
| Git identity | `Claude <noreply@anthropic.com>`, commits SSH-signed via a signing program; the configured key path is under `/home/claude` | Measured |
| GitHub identity | MCP `get_me` returns the user's own GitHub login, so API actions appear as the user | Measured |
| Why root | Unknown; the user's expectation would have been a non-privileged account | Not determined |
| Clone time | Repo cloned at 16:20:43; `main` and the session branch started at the same commit | Measured |

### `/root` layout (names only; credential-bearing dotfiles were not read)
Toolchain directories (bun, cargo, rustup, gradle, nix, npm, uv/pip config,
gcloud config), `.ccr/` (proxy certificates and README), and `.claude/`.

### `/root/.claude/` layout
- Written by the harness in a burst at 16:20:43-45: `launcher-settings.json`,
  three hook scripts, `skills/`, `plugins/`, `backups/`, `cache/`,
  `environment-manager/`, `.last-cleanup`.
- Created during the session: the transcript
  (`projects/-home-user-claude-code-web-context/<session-id>.jsonl`),
  `sessions/<n>.json`, a shell snapshot (created at 16:23 on first Bash use),
  and a small `ccr-tip.json`.
- Hook scripts:
  - `stop-hook-git-check.sh` blocks finishing while there are uncommitted
    changes, untracked files, unpushed commits, or commits not signed as the
    expected identity. It produced the "unpushed commit" feedback seen during
    the session.
  - `session-start-setup-script-failure.sh` reports a failed environment setup
    script; inert here because no failure note exists.
  - `user-prompt-submit-reply-reminder.py` is Slack-specific and does nothing
    unless an environment variable is set; most likely inert here (Inferred).
- `skills/` holds `session-start-hook/` (bundled) and `synced/<org>_<id>/`
  (the account-synced skills).
- Absent: `CLAUDE.md`, `settings.json`, `memory/`, `rules/`, `agents/`.

## 4. Tests and results

### T1. Desktop/Web context files
- **Question:** Does the session have `profile.md`, `preferences.md`, or the
  `topics/`, `areas/`, `people/`, `projects/` directories?
- **Method:** Searched the filesystem and listed MCP resources.
- **Result (Measured):** None exist. The only matches were unrelated
  directories. `ListMcpResourcesTool` returned no resources.
- **Conclusion:** User-level context in this session is whatever the container
  setup placed in `/root/.claude`; the account's Desktop/Web context files are
  not delivered.

### T2. Account identity
- **Result (Measured):** See section 3. The OS, git and GitHub identities are
  three different things.

### T3. Auto memory
- **Method:** The user said "Remember that you know Kung Fu." Then searched
  `/root/.claude` and the filesystem for new files, memory directories, and
  the string.
- **Result (Measured):** Nothing was written to disk. The string appears only
  in the session transcript. No `memory/` directory exists under the project
  directory. The system prompt contained no auto-memory instructions and no
  memory tool was available.
- **Conclusion:** Auto memory is not active in this session. Why (setting off,
  or not enabled by the cloud harness) is not determined. The model could only
  hold the fact for the current conversation.
- **Methodology note:** The model's first instinct was to treat the project
  `CLAUDE.md` as "project memory". The user corrected this: `CLAUDE.md` is
  user-authored, and auto memory is a separate system. Nothing was edited
  before the interruption.

### T4. Do remote changes reach the session automatically?
- **Method:** The user changed the repo on GitHub, then asked the model to
  fetch `main` with the output suppressed (no diff, no reading). The user then
  asked what the model knew about "minimegastructures".
- **Result (Measured):** After `git fetch`, the model knew nothing about the
  term. Fetch updates `origin/main` only; the working tree and loaded context
  were unchanged.
- **Conclusion:** Nothing syncs GitHub changes into the container
  automatically, and a fetch alone does not put content into context.

### T5. Rule file added mid-session
- **Setup:** `.claude/rules/minimegastructures.md` existed on `origin/main`
  but not in the working tree. Its contents were never read by the model.
  Its text is a single line: "Most minimegastructures are ducks."
- **Sequence (Measured):**
  1. Listing `origin/main` showed the file name; the working tree had no
     `.claude/rules/` directory.
  2. A fast-forward merge put the file on disk (16:47). On the next question
     the model still knew nothing about the term.
  3. After a push and another question, still nothing.
  4. On a later turn a system reminder arrived containing the file's contents,
     labeled as instruction files "re-read when this session started" and
     replacing earlier copies. Afterwards the model could state the rule.
- **Conclusion:** A newly added unscoped rule file **was** loaded without the
  model reading it, but not immediately and not at a time the model could
  predict. The `.claude/rules/` directory did not exist at session start, so
  this differs from the agent behavior in T6.
- **Open:** What triggered the reload (a harness re-read, a session resume,
  or something about turn count) is not known. The reminder's wording ("when
  this session started") does not match what actually happened and is
  unexplained. The uploaded reference document (section 5) does not describe
  mid-session loading of rule files.

### T6. Project agent added mid-session
- **Setup:** A `claude-code-guide` subagent was asked how to create a project
  agent. Its answer (from the Subagents docs page) was used to create
  `.claude/agents/goose-expert.md` (frontmatter: `name`, `description`,
  `tools: Read`, and `model` set to the smallest/fastest model tier). The file
  was committed and pushed.
- **Result (Measured):**
  - The agent list in context stayed at the six startup entries.
  - Asked directly, the model confirmed `goose-expert` was not listed.
  - Invoking it through the Agent tool failed with "Agent type 'goose-expert'
    not found" and listed the same six agents.
- **Conclusion:** Matches the documented rule that a restart is required when
  the `agents/` directory did not exist at session start. It is not evidence
  that the directory watcher fails when the directory already existed.
- **Caveat (Not verified):** A silently skipped file (bad frontmatter) was not
  ruled out with `claude --debug` or `claude plugin validate`.

### T7. `claude-code-guide` agent
- **Result (Measured):** The agent exists, ran in the background, used one
  documentation page, and handed back a report. The report said project agents
  live in `.claude/agents/`, the watcher picks up edits without a restart, and
  a restart is needed in three cases, including when the directory did not
  exist at session start. The agent said it did not read two other doc pages.

### T8. Gmail connector
- **Result (Measured):** The Gmail tools appeared after the server finished
  connecting. After loading the tool schemas, a read-only search for unread
  threads returned 6 threads, so the connector is authenticated and working.
  No message was opened or modified. The mailbox contents are not recorded
  here.
- **Note:** The tool set includes send, forward, reply, trash and label
  actions, so connector access is broader than read-only.

### T9. Uploaded reference document
- A document describing Claude Code context loading was uploaded for
  comparison; the comparison is in section 5.

## 5. Comparison with the uploaded reference document

| Area | Document says | Observed | Fit |
|---|---|---|---|
| Project `CLAUDE.md` | Loaded at launch | `.claude/CLAUDE.md` arrived at startup | Matches |
| User `~/.claude/CLAUDE.md` | Loaded first if present | Not present | Matches (absent) |
| Unscoped rule | Loads at launch | Added mid-session; injected several turns later | Not covered |
| Auto memory | `~/.claude/projects/<project>/memory/`, `MEMORY.md` loaded at start; "remember" writes a file | No directory, no instructions, nothing written | Does not apply here |
| Skills | Listed at start; user skills at `~/.claude/skills/`; account-synced skills exist | Listing at start; synced skills present | Matches |
| Agents | Listed at start; directory watched; restart needed if `agents/` was missing at start | Not listed; Agent tool rejected it | Matches |
| Project key | Follows the git toplevel | `projects/-home-user-claude-code-web-context/` | Matches |

Parts of the harness layer are outside the document: hooks, `launcher-settings.json`,
and the injected system context (email, attribution rules, GitHub and
environment instructions).

## 6. Open questions

1. What triggered the mid-session rule reload, and does it also apply to edits
   of an existing `CLAUDE.md`?
2. Why is auto memory absent: a setting, a cloud-harness default, or something
   else?
3. Does a fresh session on this branch list `goose-expert` at startup and
   include the minimegastructures rule in context from the start?
4. Do subagents inherit project rules? The reference document says they do.
5. Why does the session run as root when a `claude` user exists?
6. What does `launcher-settings.json` register (not read yet)?

## 7. Proposed next tests

1. **Restart test:** start a fresh session on this branch and ask about
   minimegastructures and about `goose-expert` without any hints.
2. **Subagent inheritance:** ask `goose-expert` (Read tool only) what
   minimegastructures are. If rules reach subagents it should answer "ducks"
   without reading a file.
3. **Path-scoped rule:** add a rule with a `paths:` glob and check that it loads
   only when a matching file is touched.
4. **Nested `CLAUDE.md`:** add one in a subdirectory and check that it loads
   only after a file in that directory is read.
5. **Existing-file edit:** change `.claude/CLAUDE.md` mid-session and see
   whether the new text is injected.
6. **Auto memory:** in a fresh session, check for a `memory/` directory and any
   memory-related settings.
7. **Validate the agent file** with `claude --debug` or
   `claude plugin validate .claude/agents`.

## 8. Corrections and caveats

- An early guess that a task-tool reminder came from one of the hook scripts
  was wrong; none of the scripts produce it.
- An early statement that nothing had loaded after the merge was true at that
  moment and later superseded by the reminder in T5.
- Timestamps are from file metadata; turn-by-turn timing for events without a
  file (fetch, push) was not recorded.
- Everything about Claude Code behavior beyond the observations above comes
  from the subagent's report on one documentation page and the uploaded
  reference document, neither independently verified.
