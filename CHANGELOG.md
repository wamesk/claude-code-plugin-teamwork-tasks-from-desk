# Changelog

All notable changes to the `teamwork-tasks-from-desk` plugin are documented in
this file. The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/)
and the project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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
