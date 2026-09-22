# Changelog

All notable changes to the `teamwork-tasks-from-desk` plugin are documented in
this file. The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/)
and the project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.2.0] — 2026-09-22

### Fixed
- **`${EXT,,}` aborted the whole attachment download loop.** That is bash 4 syntax;
  macOS ships bash 3.2 and answers `bad substitution`, as does zsh. The failure is total,
  not partial: every attachment the customer sent — every screenshot — was lost before
  the allow-list was even consulted. The file already knew about this trap; two pages
  later it deliberately avoids `declare -n` for the same reason. Now uses a portable
  `tr`.
- **The attachment allow-list dropped ordinary screenshot formats.** `png`, `jpg` and
  `jpeg` only means a `.gif` screen recording, a `.webp` screenshot or an iPhone `.heic`
  photo is downloaded, deleted and never reaches the task. Widened in both the Step 2.5
  merge block and `config.example.json`, which seeds a first-run config that `//=` then
  leaves alone forever — both copies needed it or the fix was half applied.
- **The customer's own words were droppable and trimmable.** Step 5.2 called the preamble
  `optional`, kept it only "when it carries context that would otherwise be lost", and
  told the skill to trim trailing blocks — which is exactly where a pasted screenshot
  sits in an email reply. The `sed`-based HTML stripper offered as an equal third
  conversion option removed every tag including `<img>`.
- `preamble_strip_signatures` in `config.example.json` was a dead key nothing reads,
  shipping `true` to every new user as if it were policy.

### Changed
- **Two new rules, shared verbatim as the `wame-task-record-v1` block.**
  1. *The estimate lives in the estimate field, and nowhere else.* Minutes never go
     into a task title or description — not in the preamble, not in the technical
     plan, not as a footer line. An estimate gets revised, and a number duplicated
     into prose has to be changed in every copy; the copy somebody misses is the one
     the next reader believes. Previews, confirmation gates, final reports and
     companion documents may still show it — those are read once and thrown away.
  2. *Never lose what the reporter wrote.* When an existing description is rewritten,
     everything already there survives verbatim at the top: inline images, links, the
     reporter's own wording, spelling and punctuation. No diacritics added, no grammar
     fixed, no translation, no tidying.

## [1.1.0] — 2026-09-22

### Changed
- **Estimate methodology replaced — `wame-estimate-v2`.** The old rule produced a
  "traditional" estimate, cut it by 30–50 % for Claude Code, then added a 15–30 %
  buffer on top. Two percentages stacked on a guess give a 0.58×–0.91× band on
  every task, so the same work could legitimately be quoted at 60 or at 95
  minutes and the wider end always won the argument. The methodology now
  estimates **one number directly** against a table of finished-outcome anchors.
  The anchors are tighter (a single figure each, adjust by at most one 15-minute
  step) and the block states explicitly what the number covers — reproduce,
  implement, test, run the suite, self-review, one review round — and what it
  never covers: deployment, production data fixes, client communication, and any
  work behind an unanswered `[OTVORENÉ]` question.
- **Uncertainty is now an open question, not a surcharge.** Where the old text
  told you to pad for "unknown unknowns", the new one tells you to write the
  question into the task, estimate the investigation that answers it, and state
  what the fix costs under each answer.
- **The 240-minute split threshold is now named as the working ceiling**, so it
  no longer contradicts the 480-minute hard cap sitting in the same paragraph.
- The methodology block is byte-identical across `teamwork-task-analyze`,
  `teamwork-tasks-from-dnr`, `teamwork-tasks-from-desk`,
  `teamwork-tasks-from-session` and `dnr-business`, and now carries a version
  marker so a drifted copy is visible.
- Step 5.4 no longer keeps a private copy of the methodology with its own anchor
  table. It points at the shared `## WAME estimate methodology` block, which the
  plugin now carries in full, and keeps only what is specific to this skill: the
  rounding keys and the rule that a main task's estimate is the sum of its
  subtasks.
- Step 5.4 names the failure mode of this particular skill: a Desk ticket is
  written by a customer, so it is usually short one fact you need in order to
  size the work. That is an open question for Step 7, not a reason to round up.

### Removed
- Config keys `estimate.buffer_pct_min`, `estimate.buffer_pct_max`,
  `estimate.speedup_pct_min` and `estimate.speedup_pct_max`. The config
  migration deletes them from files written by earlier versions and renames
  `methodology` from `wame_senior_claude_code` to `wame_estimate_v2` — a stale
  `buffer_pct_max` left in the file reads like a rule somebody still follows.

## [1.0.2] — 2026-06-11

Security and correctness hardening of the task-creation path. No user-facing
workflow changes — the same steps run in the same order — but several silent
breakages and one shell→Python injection vector are now closed. The skill
frontmatter no longer carries a `version` field; `plugin.json` is the single
source of truth for the plugin version.

### Fixed

- **Shell→Python injection / breakage via `${MAIN_NAME@Q}` / `${SUB_NAME@Q}`
  (Step 10, Step 11).** The Teamwork task name was spliced into an unquoted
  `python3` heredoc using shell `@Q` quoting, which is not valid Python
  (`SyntaxError` on apostrophes/control chars) and let a crafted task name
  break out and execute as Python. The name is now written to a temp file and
  read by Python from `argv`/stdin — exactly like the description already was.
- **Raw server JSON spliced into Python via `json.loads(${SUB_RESP@Q})`
  (Step 11).** A multi-line or control-char subtask response broke the parse
  and was injectable. The response is now fed to Python over **stdin**, the
  same way the main-task response is read.
- **Stubbed XLSX text extraction (Step 3).** The `xlsx` sidecar branch was a
  literal `python3 -c "<csv extractor>"` placeholder masked by
  `2>/dev/null || true`, so `.xlsx` attachments silently produced no `.txt`.
  Replaced with a real pure-stdlib XLSX→TSV extractor (`zipfile` +
  `xml.etree`, shared-strings aware) hardened against XXE / entity-expansion
  from a malicious workbook.
- **Subtask attachments never uploaded (Step 12).** The loop used
  `"${SUBTASK_FILES[$i][@]}"`, which is invalid bash (no nested arrays), over
  a `SUBTASK_FILES` variable that was never populated. Replaced with valid
  per-index newline-delimited `SUBTASK_FILES_<i>` string variables, now
  materialised from the Step 8 attachment mapping (new Step 8.1).
- **Weakened duplicate-task guard (Step 10).** The idempotency probe queried
  `searchTerm=${MAIN_NAME_URL_ENCODED}` against a never-assigned variable
  (empty search). `MAIN_NAME_URL_ENCODED` is now derived by URL-encoding
  `MAIN_NAME` before the query.

### Removed

- **`version` field from the SKILL.md YAML frontmatter.** The plugin version
  now lives solely in `.claude-plugin/plugin.json`.

## [1.0.1] — 2026-06-09

First-run lessons from the very first invocation against ticket Desk #9954824
("nečekaná změna ceny") in the WAME workspace. None of the user-facing
behaviour changes — the workflow stays identical — but several silent
failure modes from 1.0.0 are now fixed, and every successful run now reports
how long the hand-off took.

### Added

- **Run-time duration report (Step 14.1).** The skill now records a start
  timestamp the moment it is invoked (Step 1.1) and prints a `### ⏱ Run
  duration` block in the final report. Calibrated against
  *time-to-deliverables*: skill invocation → Projects task created **and**
  Step 13 internal note posted. Configurable via
  `desk_skill.timing.report_run_duration` (default `true`) and
  `desk_skill.timing.duration_format` (`human` / `compact`).
- **Per-user history log.** Every run is appended to a TSV at
  `~/.claude/plugins/data/teamwork-task-wamesk/runs.tsv` so the user can
  compare the duration of past runs as the probes turn into no-ops.
- **Bearer auth auto-detection (Step 2.6).** Probes `Authorization: Bearer
  <token>` against `${DESK_API}/me.json`, falls back to Basic, persists the
  working scheme in `desk_skill.auth_scheme`. Newer Desk tokens prefixed with
  `tkn.v1_` require Bearer; older opaque tokens still use Basic.
- **`desk_curl` helper.** A single shell function that applies the correct
  auth scheme transparently. Every subsequent Desk call (`Step 3.1`, `3.2`,
  `3.3`, `13`, `13b`) goes through it instead of hand-rolling auth flags.
- **Idempotency guard on main-task POST (Step 10).** When the response
  parsing throws an error (rare, but possible with multi-line Slovak
  descriptions), the skill no longer blindly retries — it searches the
  target tasklist by `searchTerm=<MAIN_NAME>&pageSize=5` and skips the
  retry if a freshly-created (last 60 s) match exists. **Prevents the
  duplicate-task bug** that bit the very first 1.0.0 run.
- **Failure-modes table additions.** Each new Desk gating mode (`403 on
  /files/{id}.json` → use `/download.json`; `403 on /threads.json POST` →
  use `/messages.json`; `200 + empty assignedFileIds` → real failure) is
  documented in the recovery table.
- **`### Step 12.1 — Things to NOT do`** — every dead-end pending-file
  shape probed during the first run is listed verbatim so future debugging
  doesn't redo the work.

### Changed

- **Internal-note body format: HTML.** Teamwork Desk does **not** render
  markdown in note bodies — `**bold**` and `### heading` show as raw
  characters in the UI. Both the Step 13 summary note **and** the Step 13b
  customer-reply DRAFT now emit HTML (`<h3>`, `<p>`, `<a>`, `<pre>`,
  `<blockquote>`, `<ul>/<li>`, `<hr>`) with `editMethod: "html"` on the
  payload. The visual review banner around the DRAFT is preserved.
- **Desk note POST schema.** Replaced `POST /threads.json` with
  `{thread:{type:"note", body:"…", isInternal:true}}` (rejected as 403 on
  modern workspaces) with `POST /messages.json` with a **flat top-level**
  body — `{message:"<html>", threadType:"note", isPrivate:true,
  editMethod:"html"}`. This is the only schema modern Desk v2 accepts.
- **Desk attachment download.** Switched from `GET /files/{id}.json`
  (returns 403 + "You Must Upgrade Your Account" on gated tiers) to
  `GET /files/{id}/download.json` (returns 303 → signed S3 URL → file
  content, **works on the gated tier**). Filename is sniffed from the
  Content-Disposition header on the signed URL. Persists
  `desk_skill.file_download_endpoint = "v2_download_json"`.
- **Projects task POST estimate field.** The Teamwork v3 task POST silently
  ignores `estimateMinutes` (the **read** field name) and lands the value
  as 0. The **write** field is `estimatedMinutes` with a `d`. Now persisted
  in `desk_skill.task_create_estimate_field` and applied consistently in
  Step 10 and Step 11.
- **JSON payload encoding for task description.** Replaced
  `jq -n --arg desc "$MAIN_DESC"` (which silently passes through control
  characters and broke `curl` on multi-line non-ASCII descriptions) with a
  `python3 - <<PY ... json.dump(..., ensure_ascii=False)` heredoc that
  writes the payload to a temp file fed to `curl -d @file`. Both main task
  and subtasks use the same encoder. **Prevents the original 1.0.0 bug
  where a description with Slovak / Czech text caused a parse error in
  the response display, which the user then mistook for failure and
  triggered a retry → duplicate task.**
- **Attachment upload schema.** Replaced the v3 attempts
  (`/projects/api/v3/files.json`, `/projects/api/v3/pendingFiles.json`,
  `PATCH /projects/api/v3/tasks/{id}.json` with various
  `attachments.pendingFile*` shapes — **all return 200 OK but no file
  actually attaches**) with the proven v1 flow:
  1. `POST /projects/api/v1/pendingFiles.json` (multipart) → returns
     `{pendingFile:{ref:"tf_…"}}`
  2. `PUT /projects/api/v1/tasks/{id}.json` with body
     `{"task":{"pendingFileAttachments":"<ref>"}}` — note the value is a
     **comma-separated string**, not an array. Response confirms with
     `{assignedFileIds:["<fileId>"]}`.
  3. Verify via `GET /projects/api/v3/tasks/{id}.json?include=attachments`
     — when `assignedFileIds` is empty the attach is treated as a failure.
- **Ticket-thread fetch fallback.** When `/threads.json` 404s on modern
  workspaces the skill now falls back to `/messages.json` automatically
  (same record shape but `messages[]` + `threadType` enum). The customer's
  first message is detected via
  `threadType == "message" AND createdBy.type == "customers"`.
- **`/files/{id}.json` removed from the happy path.** It is documented as
  a gated endpoint to avoid relying on. Attachment metadata (`size`,
  `mimeType`) is sniffed from the downloaded file + Content-Disposition.

### Fixed

- **Duplicate task creation.** Caused by 1.0.0 retrying the POST after a
  parse error on the response display — the underlying API call had
  succeeded. Fixed by the idempotency guard in Step 10 (search before
  retry) AND by replacing `jq` encoding with Python encoding (which doesn't
  throw on the multi-line description in the first place).
- **Estimate landing as 0 minutes on POST.** Fixed by the
  `estimatedMinutes` rename.
- **Attachment "uploaded" but invisible on task.** Fixed by switching to
  the v1 PUT flow and verifying via `assignedFileIds`.
- **Internal-note body shows raw markdown.** Fixed by switching to HTML.

## [1.0.0] — 2026-06-09

### Added
- Initial release. Sibling to `teamwork-task`, `teamwork-task-test` and
  `teamwork-task-analyze`.
- **Fetch:** Desk ticket subject, customer, inbox, chronological thread
  (replies + internal notes), and email attachments via the Teamwork Desk REST
  API (auto-detects v2 vs v1 endpoints on first run and persists the working
  version in the shared config).
- **Shared config:** reuses the Projects API token at
  `~/.claude/plugins/data/teamwork-task-wamesk/config.json` from the rest of
  the Teamwork plugin family; adds a separate `teamwork.desk_token` key for
  the Desk API (Desk uses its own API keys, prompted on first run).
- **Target resolution:** accepts a Projects project URL or tasklist URL;
  picks the tasklist automatically when the project has only one, otherwise
  asks via `AskUserQuestion`.
- **Synthesis:** drafts the main task in the canonical WAME format
  (`[preamble] → HR → ## Akceptačné kritériá → HR → ## Cieľ → HR → ## Technický popis`),
  always with a `### Zdroj` footer linking back to the Desk ticket.
- **Subtasks:** proposes 2–6 atomic subtasks only when the estimate exceeds
  240 min **and** the AC have natural cut points; each subtask carries a role
  prefix (`[BE]`, `[FE]`, `[QA]`, `[Migration]`, `[DevOps]`) and its own
  estimate.
- **Estimates:** uses the WAME senior-engineer-with-Claude-Code methodology —
  30–50 % speedup, 15–30 % buffer, 15 min step, 480 min cap per task.
- **Assignee per role:** when subtasks span multiple roles, asks for an email
  per role via batched `AskUserQuestion`; resolves emails to user IDs via
  `/projects/api/v3/people.json?searchTerm=<email>`; empty email = unassigned.
- **Clarifying questions:** batched `AskUserQuestion` (up to 4 per batch, up
  to 6 per run); answers folded into AC / Cieľ / Technický popis; unresolved
  preserved as `[OTVORENÉ]` markers.
- **Attachment placement:** default `--attach-mode=ask` asks per file; also
  supports `distribute` (filename heuristic) and `main` (everything on the
  main task).
- **Preview + confirmation gate:** Step 9 renders the full concept (target
  project, tasklist, all task/subtask details, attachment mapping, link-back
  plan) and waits for `Apply` / `Edit main` / `Edit subtask N` / `Drop
  subtasks` / `Cancel`. `--write-back=never` stops after preview (dry-run).
- **Native Desk-link probe:** after creating the main task, probes whether
  the Projects API exposes `deskTicketId` / `helpDeskTicketId`; if so,
  PATCHes the task with the Desk ticket ID. Falls back to the URL in the
  `### Zdroj` description footer otherwise.
- **Attachment upload:** pending-file flow with defensive payload-shape
  detection (`pendingFileAttachments` vs `pendingFileRefs`); per-file
  failures are non-fatal.
- **Internal Desk note with summary:** Step 13 posts an `isInternal: true`
  note in the Desk thread containing the link to the new Projects task **plus
  a structured summary** (title, project, tasklist, estimate, assignee, AC
  preview, subtasks list, attachments uploaded, goal) so support agents see
  the outcome without opening the task. Gated by `--notify-desk` (default
  `ask`).
- **Customer-reply DRAFT (internal note only):** Step 13b optionally drafts a
  customer-facing reply in the ticket's detected language (sk/cz/en) with a
  configurable tone (formal/casual/empathetic) — **always** posted as an
  `isInternal: true` note with a clearly visible review banner, **never** sent
  to the customer. The plugin never constructs a `type: "message"` payload
  and never calls the Desk `/replies.json` endpoint.
- **Boundaries:** never replies to the customer, never sends email, never
  moves the Desk ticket between statuses, never reassigns the Desk ticket,
  never moves the new Projects task on its board, never logs time, never
  marks the task or subtasks complete, never runs git.
