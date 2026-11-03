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
   loaded after a fetch, **not** loaded after the merge that put it on disk
   (16:47), and not loaded during the following turns. It was injected 16
   seconds after the session's `claude` process was **resumed** in a new
   process (16:54:19, see T11). This is a re-read on resume, not live file
   watching. (Measured)
5. Auto memory is not operating in this session. There is no memory directory,
   no memory instructions in the system prompt, and "remember X" wrote
   nothing to disk. (Measured)
6. A project agent added to `.claude/agents/` mid-session was **not** added to
   the agent list and the Agent tool rejected it by name. This matches the
   documented restart requirement when the `agents/` directory did not exist
   at session start. (Measured)
7. The Gmail connector is available and a read-only search worked. (Measured)
8. The harness-supplied system prompt text (GitHub rules, git rules,
   environment notes) is not stored in any file that was found. It exists on
   disk only as a `prompt_snapshot` record in the session transcript. (Measured;
   delivery channel not found)
9. The session was resumed in a new `claude` process at 16:54:19, launched by
   the environment manager in "resume" mode. The session ID stayed the same.
   (Measured; the reason for the resume is unknown)

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

### Home directory structure (`/root`)

Names only, listed at about 17:24 UTC. Credential-bearing files
(`.aws/`, `.ssh/`, `.boto`, `.claude.json`, session `.key` files) were not
read. `<org-id>` stands for the account's organization identifier, and
`<session-id>` for this session's identifier.

```
/root
|-- .aws/                      (contents not read)
|-- .bashrc, .profile, .gitconfig
|-- .boto                      (not read)
|-- .bun/                      bin/
|-- .cache/                    Microsoft/, claude-cli-nodejs/, goimports/, uv/
|-- .cargo/                    bin/, env
|-- .ccr/                      README.md, agent-proxy-ca.crt, ca-bundle.crt,
|                              system-trust.ca.pem, nssdb-*.checked-v1 (2 files)
|-- .claude/                   (expanded below)
|-- .claude.json               (not read)
|-- .config/                   gcloud/, pip/, uv/
|-- .gradle/                   native/, notifications/
|-- .local/                    bin/, lib/, share/
|-- .nix-defexpr/              channels -> (link)
|-- .nix-profile -> /nix/var/nix/profiles/per-user/root/profile
|-- .npm/                      _cacache/, _logs/
|-- .pki/                      nssdb/
|-- .rustup/                   downloads/, settings.toml, tmp/, toolchains/,
|                              update-hashes/
`-- .ssh/                      (contents not read)
```

```
/root/.claude
|-- .last-cleanup
|-- backups/                   .claude.json.backup.<epoch>  (2 files: 16:20:43, 16:54:21)
|-- cache/
|   `-- model-catalog/         <org-id>-<hash>-ccr.json
|-- environment-manager/       code-sign -> /opt/env-runner/environment-manager (symlink),
|                              codesign-mcp-config.json
|-- launcher-settings.json     (not read)
|-- plugins/
|   |-- store/                 cc-plugin-agents-md_builtin-<hash>.json
|   `-- synced/<org-id>_<id>/  account-synced plugin bucket (marker file only at top level)
|-- projects/
|   `-- -home-user-claude-code-web-context/
|       |-- <session-id>.jsonl         the session transcript
|       `-- <session-id>/              subagents/, ccr-tip.json
|-- session-env/<session-id>/
|-- session-start-setup-script-failure.sh
|-- sessions/                  103.json, 103.<hash>.key   (90.* from the first process is gone)
|-- shell-snapshots/           snapshot-bash-<epoch>-<id>.sh
|-- skills/
|   |-- session-start-hook/    SKILL.md
|   `-- synced/<org-id>_<id>/  built-in-browser, chrome-browser, computer-use,
|                              deep-research, docs, docx, google-workspace,
|                              import-memory, morning, pdf, pptx, skill-creator,
|                              xlsx, manifest.json
|-- stop-hook-git-check.sh
|-- uploads/<session-id>/      15835037-claude-code.md (the uploaded reference
|                              document), .saved-uploads.jsonl
`-- user-prompt-submit-reply-reminder.py
```

Notes on the layout:
- **Harness-provisioned at 16:20:43-45:** `launcher-settings.json`, the three
  hook scripts, `skills/`, `plugins/`, `cache/`, `environment-manager/`, the
  first backup, and `.last-cleanup`.
- **Created or changed during the session:** the transcript and its
  `<session-id>/` directory, `sessions/` (replaced when the process resumed),
  the second backup (16:54:21), the shell snapshot (regenerated at 16:58:38),
  and `uploads/` (added when the reference document was uploaded).
- **Absent:** `~/.claude/CLAUDE.md`, `settings.json`, `memory/`, `rules/`,
  `agents/`.
- **Hook scripts:**
  - `stop-hook-git-check.sh` blocks finishing while there are uncommitted
    changes, untracked files, unpushed commits, or commits not signed as the
    expected identity. It produced the "unpushed commit" feedback during the
    session.
  - `session-start-setup-script-failure.sh` reports a failed environment setup
    script; inert here because no failure note exists.
  - `user-prompt-submit-reply-reminder.py` is Slack-specific and does nothing
    unless an environment variable is set; most likely inert here (Inferred).
- **Synced skills** are the account-level content that reached the session.
  They match the `anthropic-skills:*` entries in the startup skills listing.

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
- **Conclusion (revised after T11):** The rule file was loaded without the
  model reading it, but only after the session's `claude` process was resumed
  at 16:54:19. In the original process the file sat on disk from 16:47 and was
  never injected. So this was a re-read on resume, not live watching. An
  earlier version of this document treated it as an unexplained mid-session
  reload; that was wrong. The reminder's wording ("re-read when this session
  started") is consistent with a new process start.
- **Not tested:** whether a rule added to a directory that already existed
  would be picked up by live watching within a single process.

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
  that the directory watcher fails when the directory already existed. The
  directory also did not exist when the process was resumed (T11), because the
  agent file was created after the rule reminder arrived.
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

### T10. Where the harness-supplied system prompt text lives
- **Question:** Does the GitHub and environment rule text in the system prompt
  exist as a file in the container?
- **Method:** Searched for seven distinctive phrases in text files under the
  config and system directories (`/root/.claude` excluding the transcript,
  `/root/.ccr`, `/root/.config`, `/etc`, `/home`, `/usr/local`,
  `/opt/claude-code`, `/root/.local/share`), in every running process's
  `cmdline` and `environ` (match counts only), and in the environment-manager
  and `claude` binaries. Then inspected the transcript.
- **Result (Measured):**
  - No file contains the text. No process argument or environment variable
    contains it. The environment-manager binary does not contain it. The
    `claude` binary contains only the tool name `subscribe_pr_activity`, none
    of the rule prose.
  - The transcript records the system prompt in two `prompt_snapshot`
    attachments (16:20:45 and 16:20:57), identical in their `systemPrompt`
    field. That field has 11 blocks: 10 base blocks (with a
    `__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__` marker after the second) and one
    14,846-character block titled "Your current remote execution
    environment". That last block holds the GitHub rules, the git branch rules,
    the repository scope, the environment documentation pointer, and the proxy
    and browser notes.
- **Conclusion (Inferred):** The rules are supplied by the harness at launch as
  an appended prompt block. They are not baked into the CLI and are not kept in
  a config file; the transcript only records them. How they are delivered to
  the CLI was not found.
- **Limits:** A whole-disk text search timed out, so `/nix`, `/opt/node22`,
  package caches and `/tmp` were not searched. Absence of matches does not
  rule out encoded or split text.

### T11. The session's `claude` process was resumed at 16:54:19
- **Evidence (Measured):**
  - The first process left `sessions/90.json`. It is gone, replaced by
    `sessions/103.json`, which records pid 103, the same session ID, start
    time 16:54:21, CLI version 2.1.317, `entrypoint: remote`, and
    `kind: interactive`.
  - `ps` shows `claude` pid 103 started at 16:54:19 with
    `--preload /home/claude/.claude/remote/spare.sock`, one second after
    an `environment-manager task-run --stdin --session <id> --session-mode resume`
    process started (16:54:18).
  - A second `.claude.json` backup was written at 16:54:21. The original shell
    snapshot (16:23:39) is gone and a new one was created at 16:58:38, on the
    first Bash call after the restart.
  - In the transcript, the only `instructions` attachments are the startup one
    (16:20:45) and one at 16:54:35 that contains the rule text, 16 seconds
    after the new process started.
- **Conclusion:** The harness restarted the CLI in resume mode mid-conversation.
  The new process re-read the instruction files and picked up the rule file.
  This explains T5.
- **Inferred:** The `--preload ... spare.sock` argument suggests the new process
  was a pre-started spare, and the `/home/claude` path may be why that user
  directory exists. This was not verified.
- **Not determined:** Why the resume happened. It could be an idle checkpoint,
  a per-message mechanism, or something else. The process stayed the same for
  the rest of the session (still pid 103 at 17:24).
- **Side finding:** The transcript contains 7 `task_reminder` attachments
  (first at 16:28:26) and 28 `stop_hook_summary` entries. The task reminders
  seen during the session are a built-in CLI attachment type, not output from
  the hook scripts.

### T12. Fresh session on the branch (second session, 2026-11-03)

A new session was started after the first one ended. It ran on its own branch,
`claude/amazing-faraday-epuupd`, cut from `main` after PR #1's branch was
pushed. These results come from that session.

- **T12a. Restart test (proposed test 7.1, open question 3). Measured.**
  - `.claude/CLAUDE.md` (`Keyword: Umbrella`) and
    `.claude/rules/minimegastructures.md` were both in context from the first
    turn, before any file was read. The rule is unscoped and arrived in the
    startup system reminder.
  - `goose-expert` was in the startup agent list and the Agent tool accepted it
    by name. This confirms T6: the agent was missing earlier only because
    `.claude/agents/` did not exist when that process started.
  - A project skill (`las-vegas-piece`, in `.claude/skills/`) was listed at
    startup and ran when invoked. Project skills, agents, `CLAUDE.md` and
    rules all load at launch.
- **T12b. Memory test ("granola"). Measured.** At the end of the first session
  the user asked the model to remember "granola" without it being written to a
  file. The new session did not know it. Auto memory is still absent: no memory
  instructions or memory tool appeared. This is the predicted outcome.
- **T12c. Session transcripts as a reference. Measured.**
  - The previous session's transcript is readable from a new session through
    `list_sessions` and `list_events` (claude-code-remote tools). The "granola"
    request was found there.
  - Nothing loads transcripts automatically. The new session found it only
    because the user asked and the model went looking.
  - The raw event list was about 290K characters for 66 events, too large to
    read directly; it had to be parsed with a script.
  - This is a lookup record, not memory.
- **T12d. Subagent inheritance (proposed test 7.2, open question 4). Measured,
  with a caveat.**
  - `goose-expert` has only the `Read` tool. It was asked what
    minimegastructures are, and told not to read files.
  - It answered that it knew of no established meaning but that "the project
    rule already in my context" says "Most minimegastructures are ducks".
  - Project rules in `.claude/rules/` therefore reach subagents, as the
    reference document says.
  - Only the rule text was inherited; the agent had no other knowledge of the
    term.
  - **Caveat:** The usage line showed `tool_uses: 1`. A trivial goose question
    to the same agent also showed 1, so this is most likely the agent's
    hand-back call and not a file read. That was not verified. The claim that no
    file was read is the agent's own. The subagent transcript under
    `projects/.../<session-id>/subagents/` was not inspected.

## 5. Comparison with the uploaded reference document

| Area | Document says | Observed | Fit |
|---|---|---|---|
| Project `CLAUDE.md` | Loaded at launch | `.claude/CLAUDE.md` arrived at startup | Matches |
| User `~/.claude/CLAUDE.md` | Loaded first if present | Not present | Matches (absent) |
| Unscoped rule | Loads at launch | Added mid-session; loaded only after the process was resumed (T11) | Consistent with launch-time loading (resume re-reads) |
| Auto memory | `~/.claude/projects/<project>/memory/`, `MEMORY.md` loaded at start; "remember" writes a file | No directory, no instructions, nothing written | Does not apply here |
| Skills | Listed at start; user skills at `~/.claude/skills/`; account-synced skills exist | Listing at start; synced skills present | Matches |
| Agents | Listed at start; directory watched; restart needed if `agents/` was missing at start | Not listed; Agent tool rejected it | Matches |
| Project key | Follows the git toplevel | `projects/-home-user-claude-code-web-context/` | Matches |

Parts of the harness layer are outside the document: hooks, `launcher-settings.json`,
and the injected system context (email, attribution rules, GitHub and
environment instructions).

## 6. Open questions

1. Why did the harness resume the session in a new process at 16:54:19, and
   does it happen on a pattern (idle time, message count) or at random? Does
   the new process also pick up edits to an existing `CLAUDE.md`?
2. Why is auto memory absent: a setting, a cloud-harness default, or something
   else?
3. ~~Does a fresh session on this branch list `goose-expert` at startup and
   include the minimegastructures rule in context from the start?~~ **Answered
   (T12a):** yes to both.
4. ~~Do subagents inherit project rules? The reference document says they do.~~
   **Answered (T12d):** yes, with the tool-use caveat noted there.
5. Why does the session run as root when a `claude` user exists?
6. What does `launcher-settings.json` register (not read yet)?

## 7. Proposed next tests

1. ~~**Restart test:** start a fresh session on this branch and ask about
   minimegastructures and about `goose-expert` without any hints.~~ **Done
   (T12a).**
2. ~~**Subagent inheritance:** ask `goose-expert` (Read tool only) what
   minimegastructures are. If rules reach subagents it should answer "ducks"
   without reading a file.~~ **Done (T12d).** To close the caveat, inspect the
   subagent transcript for a `Read` call.
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
8. **Same-process rule test:** add a rule file to a rules directory that
   already existed at the start of the current process, without a resume, and
   check whether live watching picks it up.

## 8. Corrections and caveats

- An early guess that a task-tool reminder came from one of the hook scripts
  was wrong; none of the scripts produce it. It is a built-in `task_reminder`
  attachment (T11).
- The first write-up of T5 treated the rule reload as a mid-session event. The
  later finding that the process was resumed (T11) replaced that conclusion.
- An early statement that nothing had loaded after the merge was true at that
  moment and later superseded by the reminder in T5.
- Timestamps are from file metadata; turn-by-turn timing for events without a
  file (fetch, push) was not recorded.
- Everything about Claude Code behavior beyond the observations above comes
  from the subagent's report on one documentation page and the uploaded
  reference document, neither independently verified.
