# Changelog

All notable changes to the `teamwork-tasks-from-desk` plugin are documented in
this file. The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/)
and the project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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
