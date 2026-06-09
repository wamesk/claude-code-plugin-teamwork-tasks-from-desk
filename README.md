# teamwork-tasks-from-desk

The **bridge between customer support and engineering**. Hands off a Teamwork
**Desk** ticket — subject, customer, full chronological thread, email
attachments — into a fully-prepared Teamwork **Projects** task (plus optional
subtasks) without leaving the terminal.

Goal: after this skill finishes, an engineer who picks up the new Projects
task can start implementing immediately, and the support agent who triaged the
Desk ticket sees what was prepared in a structured internal note inside the
original ticket.

## Install

```
/plugin marketplace add wamesk/claude-code
/plugin install teamwork-tasks-from-desk@wame
```

The **Projects** API token is shared with the `teamwork-task`,
`teamwork-task-test` and `teamwork-task-analyze` plugins — if any of those is
already configured, this plugin uses the same token. The **Desk** API token is
separate (Teamwork Desk uses its own keys); the skill prompts for it on first
run and stores it alongside in the shared config.

Both tokens live in `~/.claude/plugins/data/teamwork-task-wamesk/config.json`
with `chmod 600`. They survive plugin updates.

## Usage

```
/teamwork-tasks-from-desk https://<workspace>.teamwork.com/desk/tickets/12345
```

The skill then asks (via `AskUserQuestion`):

1. The target Teamwork **Projects** URL — project or tasklist
2. The assignee email (or one email per role when the task naturally splits
   into BE/FE/QA/migration); empty = unassigned
3. Any clarifying questions raised by the ticket content (max 6 per run)
4. Per-attachment placement — main task vs. specific subtask vs. skip
5. Final preview + confirmation before any write
6. Whether to post an internal note in the Desk thread with a link + summary
7. Whether to draft a customer-facing reply (always posted as **internal note**
   for review — never sent to the customer)

### Optional flags

- `--projects-url=<url>` — skip the Projects URL question
- `--assignee=<email>` — skip the main-task assignee prompt
- `--no-subtasks` — never propose subtasks; single main task only
- `--language=sk|en` — language for the rewritten description (default from
  config, fallback `sk`)
- `--write-back=ask|auto|never` — gate Projects writes (default `ask`); `never`
  stops after the preview (full dry-run)
- `--attach-mode=ask|distribute|main` — how to place attachments (default
  `ask`); `distribute` uses a filename heuristic, `main` puts everything on
  the main task
- `--notify-desk=ask|true|false` — whether to post the internal note in the
  Desk thread (default `ask`)
- `--draft-reply=ask|true|false` — whether to offer the customer-reply DRAFT
  (default `ask`)
- `--draft-tone=formal|casual|empathetic` — tone for the customer-reply DRAFT
  (default `formal`)
- `--max-questions=N` — cap for clarifying questions per run (default 6)

## What it does

1. **Fetches** the Desk ticket — subject, customer, inbox, chronological thread
   of replies and internal notes, every email attachment — via the Teamwork
   Desk REST API. Auto-detects whether the workspace runs Desk API v2 or v1.
2. **Asks for the target Projects URL** (project or tasklist).
3. **Synthesises a draft main task** in the canonical WAME format:
   `[preamble] → HR → ## Akceptačné kritériá → HR → ## Cieľ → HR → ## Technický popis`,
   ending with a `### Zdroj` block that links back to the Desk ticket.
4. **Proposes subtasks** only when the estimate exceeds 240 min and the AC
   have natural cut points (BE/FE/QA/migration/DevOps role prefixes).
5. **Asks for an assignee email per role** when the task splits — each role
   can be answered independently or left empty.
6. **Estimates** via the WAME senior-engineer-with-Claude-Code methodology
   (30–50 % speedup, 15–30 % buffer, 15 min step, 480 min cap per task).
7. **Pauses with clarifying questions** via `AskUserQuestion` in batches of 4,
   max 6 per run; folds answers back into AC / Cieľ / Technický popis;
   preserves anything unresolved as `[OTVORENÉ]` markers.
8. **Maps email attachments** to the right task or subtask — per-file question
   by default; filename heuristic or "everything to main" available via flags.
9. **Previews the full concept** + waits for confirmation. `--write-back=never`
   stops here as a dry-run.
10. **Creates the main task** and probes for a native Desk-link attribute on
    the Projects API (`deskTicketId` / `helpDeskTicketId`); applies it when
    supported, otherwise relies on the URL footer in the description.
11. **Creates subtasks** with `parentTaskId` and per-role assignees.
12. **Uploads attachments** via the pending-file flow; tries both
    `pendingFileAttachments` and `pendingFileRefs` payload shapes.
13. **Posts an internal note in the Desk thread** with a structured summary of
    what was created — task link, AC count, subtasks list, estimate,
    attachments uploaded, goal. Not just a bare link.
14. **Offers a customer-reply DRAFT** — language detected from the ticket
    (sk/cz/en), tone configurable. Always posted as an **internal note** for
    human review; the plugin never sends it to the customer, never touches
    the Desk replies endpoint.
15. **Renders a final report** summarising everything that was created,
    skipped, or failed.

## What it does **not** do

- Never replies to the Desk customer
- Never sends an email
- Never moves the Desk ticket between statuses (open / pending / closed)
- Never reassigns the Desk ticket
- Never moves the new Projects task between board columns
- Never logs time
- Never marks the new task or subtasks as complete
- Never runs `git commit` / `git push`

It is strictly **read-only against the boards and the timesheet** — the only
writes are the new Projects task + subtasks + attachments, and the optional
internal Desk notes.

## Config

Stored at `~/.claude/plugins/data/teamwork-task-wamesk/config.json` (shared
with the rest of the Teamwork plugin family). This plugin adds a
`desk_skill.*` namespace and a `teamwork.desk_token` key — see
`config.example.json` for the full list of keys (description section labels,
estimate methodology parameters, subtask split thresholds, role prefixes,
clarifying-question cap, attachment handling, link-back configuration,
customer-reply draft tone and signature).

The shared config survives plugin updates. Tokens are `chmod 600`.

## License

MIT — © 2026 Stanislav Červeňák
