---
name: teamwork-tasks-from-desk
version: 1.0.0
description: "Use when the user provides a Teamwork.com Desk ticket URL (https://<workspace>.teamwork.com/desk/tickets/<id>) and asks to 'vytvor tasky z desk ticketu', 'preklop ticket do projects', 'urob task z desku', 'spracuj desk ticket', 'vytvor projektový task z desku', 'create projects task from desk', 'turn desk ticket into projects task', 'desk ticket to task', or invokes '/teamwork-tasks-from-desk'. Fetches the Desk ticket (subject, customer, chronological thread of replies/notes, email attachments) via the Teamwork Desk REST API, asks for a target Teamwork Projects URL (project/tasklist) and an assignee email (asks for an email per role when the task splits into BE/FE/QA/migration), then interactively drafts a main task in the canonical WAME format ([preamble] → HR → Akceptačné kritériá → HR → Cieľ → HR → Technický popis) with optional subtasks and a WAME-methodology estimate. Asks clarifying questions via AskUserQuestion in batches of 4 (max 6 per run) and asks per attachment where it belongs. After a full preview + confirmation creates the task (and subtasks) via /projects/api/v3, uploads attachments via the pending-file flow, and links the main task back to the originating Desk ticket — using the native Teamwork Desk-link attribute when available, falling back to a URL in the description footer. Closes the loop by posting an internal note in the Desk thread containing both the link to the new Projects task and a structured summary of what was prepared (title, AC count, subtasks list, estimate, attachments uploaded, goal), and finally offers to draft a customer-facing reply — the draft is ALWAYS posted as an internal note for review, NEVER sent to the customer (the plugin never touches the Desk replies endpoint). Reuses the shared API token config; Desk uses its own token stored alongside the Projects token. Never replies to the customer, never moves the Desk ticket on its board, never logs time, never moves tasks on the Projects board."
argument-hint: "<desk-ticket-url> [--projects-url=<url>] [--assignee=<email>] [--no-subtasks] [--language=sk|en] [--write-back=ask|auto|never] [--attach-mode=ask|distribute|main] [--notify-desk=ask|true|false] [--draft-reply=ask|true|false] [--draft-tone=formal|casual|empathetic] [--max-questions=N]"
allowed-tools: [Bash, Read, Write, Edit, Grep, Glob, AskUserQuestion]
---

# Teamwork Tasks From Desk

Turn a **Teamwork Desk ticket** into a fully-prepared **Teamwork Projects task**
(plus optional subtasks) without leaving the terminal. The skill fetches the
ticket — subject, customer, the full chronological thread of replies and
internal notes, and every email attachment — drafts a task in the canonical
WAME format, asks clarifying questions, lets you map attachments to the right
task or subtask, creates everything via the Projects REST API, and closes the
loop by posting an internal note back into the Desk ticket so support and
engineering stay in sync.

The user invoked this skill with: `$ARGUMENTS`

This skill is the **bridge between customer support and engineering**. The
support agent who triages a Desk ticket can hand it off in one command; the
engineer who picks it up gets a ready-to-implement Projects task without any
back-and-forth. It is read-only against:

- the Desk ticket's board state (never reopens, closes, reassigns, or moves the
  ticket between statuses)
- the Desk customer thread (never replies, never sends a customer-facing
  message — every Desk write is an `isInternal: true` note)
- the Projects board (never moves the created tasks between columns)
- the Projects timesheet (never logs time)
- the local git working tree (never commits, never pushes)

The only writes are:

1. POSTing the main Projects task + confirmed subtasks
2. Uploading attachments to those tasks
3. (Optionally) PATCHing the main task with a native `deskTicketId` when the
   Projects API exposes one
4. (Optionally) POSTing one or two **internal** notes back into the Desk thread
   — one with a link + summary of what was created, and optionally a DRAFT of a
   customer-facing reply that is **always** posted as an internal note for
   human review, **never** sent to the customer

---

## Arguments

Expected first positional argument: a **Teamwork Desk ticket URL** of the form
`https://<workspace>.teamwork.com/desk/tickets/<id>` (also accepts
`/#/tickets/<id>` and `/tickets/<id>` shapes).

Optional flags (override config for this run only — not persisted):

- `--projects-url=<url>` — skip the Step 4 question, use this Projects project
  or tasklist URL directly
- `--assignee=<email>` — skip the global Step 6 prompt, use this email for the
  main task (per-role prompts may still happen for subtasks)
- `--no-subtasks` — never propose subtasks; create a single main task only
- `--language=sk|en` — language for the rewritten description and the
  clarifying questions (default from config, fallback `sk`)
- `--write-back=ask|auto|never` — gate Projects writes (default `ask`). `auto`
  skips the Step 9 confirmation; `never` stops after the preview (full dry-run)
- `--attach-mode=ask|distribute|main` — how to place email attachments. `ask`
  asks per file (default); `distribute` runs the filename heuristic without
  asking; `main` puts everything on the main task
- `--notify-desk=ask|true|false` — whether to post the Step 13 internal note in
  the Desk thread (default `ask`)
- `--draft-reply=ask|true|false` — whether to offer the customer-reply DRAFT in
  Step 13b (default `ask`)
- `--draft-tone=formal|casual|empathetic` — tone for the customer-reply DRAFT
  (default `formal`)
- `--max-questions=N` — cap for clarifying questions per run (default 6)

If `$ARGUMENTS` is empty or contains no URL, ask via **AskUserQuestion** for the
Desk ticket URL before doing anything else.

---

## Step 1 — Parse the Desk ticket URL

```bash
URL="<the url>"
WORKSPACE=$(echo "$URL"   | sed -nE 's|https?://([^.]+)\.teamwork\.com/.*|\1|p')
TICKET_ID=$(echo "$URL"   | sed -nE 's|.*/(desk/tickets|tickets)/([0-9]+).*|\2|p')
BASE_URL="https://${WORKSPACE}.teamwork.com"
DESK_BASE="${BASE_URL}/desk"
```

If `WORKSPACE` or `TICKET_ID` is empty, ask via **AskUserQuestion** for a
corrected URL. Do not guess.

---

## Step 2 — Load shared config (Projects + Desk tokens)

This skill deliberately shares the config file with the `teamwork-task`,
`teamwork-task-test`, and `teamwork-task-analyze` plugins, so users who already
configured one of those only have to enter the **Desk** token once.

```
~/.claude/plugins/data/teamwork-task-wamesk/config.json
```

Algorithm:

1. `CONFIG_DIR="$HOME/.claude/plugins/data/teamwork-task-wamesk"` — `mkdir -p "$CONFIG_DIR"`.
2. `CONFIG_FILE="$CONFIG_DIR/config.json"`.
3. If `CONFIG_FILE` does not exist, copy this plugin's bundled template from
   `${CLAUDE_PLUGIN_ROOT}/config.example.json` and `chmod 600 "$CONFIG_FILE"`.
4. Validate with `jq . "$CONFIG_FILE" >/dev/null`. If invalid → report and stop.
5. **First-run check** — three keys must be set:
   - `.teamwork.base_url` — workspace base URL (prefill from the parsed Desk URL
     when first-run)
   - `.teamwork.api_token` — **Projects** API token (shared with the rest of the
     Teamwork plugin family); get from
     `https://<workspace>.teamwork.com/launchpad/apikey/manage`
   - `.teamwork.desk_token` — **Desk** API token (this plugin's addition);
     Teamwork Desk uses a separate API key, generate it from the Desk Settings
     → API Keys page

   If any of the three is missing/empty/placeholder, prompt via
   **AskUserQuestion** for the missing values. Write them back atomically with
   `jq` + `mv`, then `chmod 600`.

6. Optional key `.teamwork.desk_base_url` — defaults to `${BASE_URL}/desk`. Some
   workspaces host Desk on a separate hostname; this lets users override.
7. **Apply config migration / defaults for this skill's own keys** — see Step
   2.5 below.
8. **Apply CLI flag overrides** (`--projects-url`, `--assignee`, `--no-subtasks`,
   `--language`, `--write-back`, `--attach-mode`, `--notify-desk`,
   `--draft-reply`, `--draft-tone`, `--max-questions`) to the in-memory config
   — do not persist.
9. **Never echo any token.** Always pass auth to `curl` via `-u "$TOKEN:xxx"`
   (kept out of `ps`), never in the URL or in `set -x` output.

### Step 2.5 — Desk-skill config defaults (idempotent merge)

Add this skill's own keys to the same shared config file, defaulting any
missing keys. Run on every invocation — `//=` is idempotent.

```bash
TMP=$(mktemp)
jq '
  .desk_skill                                       //= {}
| .desk_skill.write_back_mode                       //= "ask_each"
| .desk_skill.description_format                    //= "wame_canonical"
| .desk_skill.sections                              //= {}
| .desk_skill.sections.acceptance_label_sk          //= "Akceptačné kritériá"
| .desk_skill.sections.acceptance_label_en          //= "Acceptance criteria"
| .desk_skill.sections.goal_label_sk                //= "Cieľ"
| .desk_skill.sections.goal_label_en                //= "Goal"
| .desk_skill.sections.tech_label_sk                //= "Technický popis"
| .desk_skill.sections.tech_label_en                //= "Technical plan"
| .desk_skill.sections.source_label_sk              //= "Zdroj"
| .desk_skill.sections.source_label_en              //= "Source"
| .desk_skill.sections.hr_marker                    //= "---"
| .desk_skill.estimate                              //= {}
| .desk_skill.estimate.methodology                  //= "wame_senior_claude_code"
| .desk_skill.estimate.set_if_missing               //= true
| .desk_skill.estimate.buffer_pct_min               //= 15
| .desk_skill.estimate.buffer_pct_max               //= 30
| .desk_skill.estimate.speedup_pct_min              //= 30
| .desk_skill.estimate.speedup_pct_max              //= 50
| .desk_skill.estimate.step_minutes                 //= 15
| .desk_skill.estimate.max_task_minutes             //= 480
| .desk_skill.subtasks                              //= {}
| .desk_skill.subtasks.enabled                      //= true
| .desk_skill.subtasks.propose_split_threshold_minutes //= 240
| .desk_skill.subtasks.min_split_subtasks           //= 2
| .desk_skill.subtasks.max_split_subtasks           //= 6
| .desk_skill.subtasks.role_prefixes                //= ["BE","FE","QA","Migration","DevOps"]
| .desk_skill.assignee                              //= {}
| .desk_skill.assignee.ask_per_role                 //= true
| .desk_skill.assignee.allow_empty                  //= true
| .desk_skill.attachments                           //= {}
| .desk_skill.attachments.attach_mode               //= "ask"
| .desk_skill.attachments.max_attachment_size_mb    //= 25
| .desk_skill.attachments.extensions_allow          //= ["md","txt","pdf","docx","xlsx","csv","json","html","eml","msg","sql","png","jpg","jpeg","zip"]
| .desk_skill.attachments.cleanup                   //= "after_run"
| .desk_skill.link_back                             //= {}
| .desk_skill.link_back.try_native_desk_link        //= true
| .desk_skill.link_back.url_in_description_footer   //= true
| .desk_skill.link_back.notify_desk_default         //= "ask"
| .desk_skill.link_back.notify_desk_include_summary //= true
| .desk_skill.customer_reply_draft                  //= {}
| .desk_skill.customer_reply_draft.enabled          //= true
| .desk_skill.customer_reply_draft.ask_default      //= "ask"
| .desk_skill.customer_reply_draft.tone             //= "formal"
| .desk_skill.customer_reply_draft.language_detection //= "from_ticket"
| .desk_skill.customer_reply_draft.language_fallback //= "sk"
| .desk_skill.customer_reply_draft.signature        //= "S pozdravom,\nTím WAME"
| .desk_skill.customer_reply_draft.include_open_questions //= true
| .desk_skill.customer_reply_draft.post_as_internal_note //= true
| .desk_skill.customer_reply_draft.never_send_as_reply //= true
| .desk_skill.clarifying_questions                  //= {}
| .desk_skill.clarifying_questions.max_per_run      //= 6
| .desk_skill.clarifying_questions.fold_into_description //= true
| .desk_skill.clarifying_questions.leave_unresolved_as_open_marker //= "[OTVORENÉ]"
' "$CONFIG_FILE" > "$TMP" && mv "$TMP" "$CONFIG_FILE" && chmod 600 "$CONFIG_FILE"
```

Load all values into local variables for the rest of the run:

```bash
PROJECTS_TOKEN=$(jq -r '.teamwork.api_token'  "$CONFIG_FILE")
DESK_TOKEN=$(   jq -r '.teamwork.desk_token'  "$CONFIG_FILE")
BASE_URL=$(     jq -r '.teamwork.base_url'    "$CONFIG_FILE")
DESK_BASE=$(    jq -r '(.teamwork.desk_base_url // "") | if . == "" then "'"$BASE_URL"'/desk" else . end' "$CONFIG_FILE")
PROJECTS_AUTH="${PROJECTS_TOKEN}:xxx"
DESK_AUTH="${DESK_TOKEN}:xxx"
```

---

## Step 3 — Fetch the Desk ticket

Teamwork Desk REST uses the same HTTP Basic auth as Projects, with the Desk
token. The version path may be `/desk/api/v2/` on most workspaces; on older
ones it may be `/desk/api/v1/` or unversioned. The skill probes v2 first and
falls back gracefully, recording the working version into the config for
subsequent runs:

```bash
DESK_API_VERSION=$(jq -r '.desk_skill.api_version // ""' "$CONFIG_FILE")
if [ -z "$DESK_API_VERSION" ]; then
  for V in v2 v1; do
    HTTP=$(curl -sS -o /dev/null -w '%{http_code}' -u "$DESK_AUTH" \
      "${DESK_BASE}/api/${V}/tickets/${TICKET_ID}.json")
    if [ "$HTTP" = "200" ]; then DESK_API_VERSION="$V"; break; fi
  done
  if [ -z "$DESK_API_VERSION" ]; then
    echo "❌ Could not reach Desk API at ${DESK_BASE}/api/{v2,v1}/tickets/${TICKET_ID}.json"; exit 1
  fi
  # Persist for next run
  TMP=$(mktemp); jq --arg v "$DESK_API_VERSION" '.desk_skill.api_version = $v' \
    "$CONFIG_FILE" > "$TMP" && mv "$TMP" "$CONFIG_FILE" && chmod 600 "$CONFIG_FILE"
fi
DESK_API="${DESK_BASE}/api/${DESK_API_VERSION}"
```

### Step 3.1 — Ticket details (subject, customer, inbox)

```bash
TICKET_JSON=$(curl -sS -u "$DESK_AUTH" \
  "${DESK_API}/tickets/${TICKET_ID}.json?include=customer,inbox,assignee")
```

Extract:

```bash
SUBJECT=$(   jq -r '.ticket.subject     // .subject     // "(no subject)"'           <<<"$TICKET_JSON")
CUSTOMER_FN=$(jq -r '(.ticket.customer  // .customer    // {}).firstName // ""'      <<<"$TICKET_JSON")
CUSTOMER_LN=$(jq -r '(.ticket.customer  // .customer    // {}).lastName  // ""'      <<<"$TICKET_JSON")
CUSTOMER_EMAIL=$(jq -r '
  (((.ticket.customer // .customer // {}).emailAddresses // []) | .[0].address)
  // ((.ticket.customer // .customer // {}).email)
  // ""' <<<"$TICKET_JSON")
INBOX_NAME=$(jq -r '(.ticket.inbox // .inbox // {}).name // ""' <<<"$TICKET_JSON")
```

If HTTP 401 → re-prompt the Desk token (re-run Step 2 first-run flow). 403/404
→ stop with the failing URL printed.

### Step 3.2 — Chronological thread

```bash
THREADS=$(curl -sS -u "$DESK_AUTH" \
  "${DESK_API}/tickets/${TICKET_ID}/threads.json?page=1&pageSize=100&orderBy=createdAt&orderMode=asc")
```

Page through if `meta.totalPages > 1`. For each thread item collect: `id`,
`body` (HTML), `createdAt`, `author.name`, `channel` (email vs note vs
internal), `isInternal` (bool), `attachments[]`.

Convert each `body` to markdown — prefer `pandoc -f html -t markdown_strict`,
fall back to `python -m html2text` or a `sed`-based stripper. Keep the
chronological order: earliest first.

Detect the **first customer-initiated message** — usually the earliest
`channel == "email"` from the customer (not the agent). This is the source for
the AC extraction in Step 5.

### Step 3.3 — Attachments

Attachments may appear in two places: dedicated `/attachments.json` endpoint
and inline within each thread's `attachments[]`. Collect both, deduplicate by
attachment `id`.

```bash
ATT_INDEX=$(curl -sS -u "$DESK_AUTH" \
  "${DESK_API}/tickets/${TICKET_ID}/attachments.json?page=1&pageSize=100")
```

For each attachment (whether from the index or inline within a thread), record:
`id`, `filename`, `size` (bytes), `mimeType`, `downloadURL`.

Filter by config:

```bash
MAX_MB=$(jq -r '.desk_skill.attachments.max_attachment_size_mb' "$CONFIG_FILE")
ALLOW=( $(jq -r '.desk_skill.attachments.extensions_allow[]' "$CONFIG_FILE") )
```

Skip files larger than `MAX_MB` MB; record them in the report as `⏭ too large`.
Skip extensions not in the allow-list; record them as `⏭ extension blocked`.

Download surviving attachments into a per-run temp directory:

```bash
RUN_ID="$(date +%s)-${RANDOM}"
ATT_DIR="${TMPDIR:-/tmp}/teamwork-tasks-from-desk-${RUN_ID}/attachments"
mkdir -p "$ATT_DIR"
curl -sS -u "$DESK_AUTH" -L -o "${ATT_DIR}/${FILENAME}" "$DOWNLOAD_URL"
```

For binary documents (PDF/DOCX/XLSX/PPTX), also extract text into a sibling
`.txt` so the Step 5 reasoning can see the content:

```bash
case "$EXT" in
  pdf)  pdftotext -layout "$F" "${F%.pdf}.txt"  2>/dev/null || true ;;
  docx) pandoc -f docx -t plain "$F" -o "${F%.docx}.txt" 2>/dev/null || true ;;
  xlsx) python3 -c "<csv extractor>" "$F" > "${F%.xlsx}.txt"      2>/dev/null || true ;;
esac
```

Failures are non-fatal — Step 5 simply works without that file's content.

Cleanup mode `desk_skill.attachments.cleanup`:

- `after_run` (default) — `rm -rf "$ATT_DIR"` at the very end of Step 14
- `keep` — leave the directory; print its path in the final report

---

## Step 4 — Ask for the target Projects URL (project or tasklist)

If `--projects-url` was supplied as a CLI flag, use it directly. Otherwise ask
via **AskUserQuestion**:

> "Paste the Teamwork Projects URL where the task should be created — either a
> project URL (`https://<workspace>.teamwork.com/app/projects/<projectId>`) or
> a tasklist URL (`https://<workspace>.teamwork.com/app/tasklists/<tasklistId>`)."

Parse with the same regex used by the other Teamwork plugins:

```bash
PROJECTS_URL="<answer>"
PROJ_KIND=$(echo "$PROJECTS_URL" | sed -nE 's|.*/(projects|tasklists)/[0-9]+.*|\1|p' | sed 's/s$//')
PROJ_ENTITY_ID=$(echo "$PROJECTS_URL" | sed -nE 's|.*/(projects|tasklists)/([0-9]+).*|\2|p')
```

### Step 4.1 — Project URL flow

```bash
GET ${BASE_URL}/projects/api/v3/projects/${PROJ_ENTITY_ID}.json
GET ${BASE_URL}/projects/api/v3/projects/${PROJ_ENTITY_ID}/tasklists.json
```

If the project has exactly one tasklist → use it without asking.

If the project has multiple tasklists → **AskUserQuestion** with each tasklist
as an option (label = tasklist name, description = `# tasks: N` from the
response metadata). Cap at 12 visible; if more, include a "More…" option that
asks for an exact tasklist name via free-form input.

Set:

```bash
PROJECT_ID="${PROJ_ENTITY_ID}"
PROJECT_NAME=$(jq -r '.project.name' <<<"$PROJECT_JSON")
TASKLIST_ID="${selected}"
TASKLIST_NAME="${selected_name}"
```

### Step 4.2 — Tasklist URL flow

```bash
GET ${BASE_URL}/projects/api/v3/tasklists/${PROJ_ENTITY_ID}.json
```

Extract `project.id` and `project.name` from the response to set `PROJECT_ID`
and `PROJECT_NAME`. `TASKLIST_ID` = `PROJ_ENTITY_ID`.

Stop with a clear error if the user cannot access the target (HTTP 403/404).

---

## Step 5 — Synthesise the task concept

This is the reasoning step. Inputs:

- `SUBJECT`, `CUSTOMER_FN`, `CUSTOMER_LN`, `CUSTOMER_EMAIL`, `INBOX_NAME`
- Chronological threads (markdown bodies + author + channel + isInternal)
- All downloaded attachment files + their extracted `.txt` sidecars
- Repo shape sniff (only when running inside a Laravel/Vue/etc. repo) —
  `composer.json` name, `package.json` name, `wamesk/*` modules, `Modules/*`
- Language from `--language` → `desk_skill.default_language` → `sk`

Produce a draft with:

### 5.1 — Title

Short, business-friendly, derived from the ticket subject + the customer's
main ask. Keep under 90 characters. Always end with `(Desk #<TICKET_ID>)` so
the link to the source ticket is visible in every tasklist view.

Examples:

- `Pridať export faktúr do CSV (Desk #12345)`
- `Oprava overenia telefónneho čísla v registrácii (Desk #12346)`

### 5.2 — Description (canonical WAME format)

```
[optional preamble — the customer's first email verbatim, kept when it carries
context that would otherwise be lost; trim quoted signatures and legal
footers]

---

## Akceptačné kritériá
- [ ] AC 1 — atomic, testable, written from the user's perspective
- [ ] AC 2 — …

---

## Cieľ
2–4 sentences. **Why** (business outcome), not **how** (technical means).

---

## Technický popis

### Stack & scope
What system / module / files this touches. Detected language and framework.

### Step-by-step
Plan-Mode style numbered list — what to build first, what depends on what.

### Data model / schema changes
Tables/columns/migrations or "none".

### Edge cases & risks
Specifically the ones the customer hinted at in the thread.

### Tests to add
Unit / feature / Dusk / Cypress / Playwright as appropriate.

### Zdroj
- Desk ticket: <DESK_BASE>/tickets/<TICKET_ID> (#<TICKET_ID>) — <SUBJECT>
- Customer: <CUSTOMER_FN> <CUSTOMER_LN> <<CUSTOMER_EMAIL>>
- Created via /teamwork-tasks-from-desk on <YYYY-MM-DD>
```

The `### Zdroj` block is **mandatory** even when the Projects API exposes a
native `deskTicketId` field — having the URL visible in the description avoids
relying on UI features.

### 5.3 — Subtasks (optional)

Subtasks are proposed **only** when **all** these are true:

1. `--no-subtasks` is not set
2. The total proposed estimate exceeds `desk_skill.subtasks.propose_split_threshold_minutes` (default 240 min)
3. The AC have natural cut points (different layers, different components,
   migration vs. feature work, BE vs. FE, etc.)

When proposed, each subtask gets:

- **Name** — `[BE]`, `[FE]`, `[QA]`, `[Migration]`, or `[DevOps]` prefix (from
  `desk_skill.subtasks.role_prefixes`), then a short business-friendly name
- **Description** — the same canonical structure as the main task minus the
  preamble (subtasks always begin with AC) and minus the `### Zdroj` block
  (only the main task carries the source link)
- **Estimate** — own value, summed up later to validate against the main task's
  estimate

Count: 2–6. Never force-split smaller tasks.

### 5.4 — Estimate

WAME senior + Claude Code methodology:

- Start from a traditional senior-engineer estimate of the task
- Apply 30–50 % speedup (Claude Code as a force multiplier)
- Then add 15–30 % buffer for unknowns, friction, review feedback
- Round to a multiple of `desk_skill.estimate.step_minutes` (default 15)
- Cap at `desk_skill.estimate.max_task_minutes` (default 480) — anything bigger
  must be split via Step 5.3

For each subtask, run the same methodology independently. The main task's
estimate is the **sum** of its subtask estimates (or the methodology-computed
value when there are no subtasks).

Calibration anchors (in minutes, *post*-speedup, *post*-buffer):

| Task type | Estimate |
|---|---|
| Trivial copy / label change | 15 |
| Bug fix with known repro | 30–60 |
| Single CRUD with form + table | 60–120 |
| New Vue/React component with state | 60–120 |
| New module / domain object end-to-end | 240–360 |
| Migration with backfill + tests | 120–240 |

---

## Step 6 — Resolve assignees by email

### 6.1 — How many emails to ask for

After Step 5, count distinct role prefixes across the subtasks
(deduplicated set of `[BE]`, `[FE]`, `[QA]`, …). Let `ROLE_COUNT` be that size.

- `ROLE_COUNT == 0` (no subtasks) or `ROLE_COUNT == 1` → one **AskUserQuestion**:
  > "Email of the assignee for this task (leave empty for unassigned):"

  If `--assignee=<email>` was passed, use it directly without asking.

- `ROLE_COUNT >= 2` → batched **AskUserQuestion** — one question per role
  (e.g. `Email for BE tasks?`, `Email for FE tasks?`, `Email for QA tasks?`).
  Each can be answered independently with an email or left empty. Batch up to
  4 questions per UI call (Claude Code limit).

The main task's assignee defaults to the **dominant role** (the one with the
highest summed subtask estimate). The user can override in the Step 9 preview.

### 6.2 — Resolve email → userId

For each non-empty email:

```bash
EMAIL_ENCODED=$(python3 -c 'import urllib.parse,sys; print(urllib.parse.quote(sys.argv[1]))' "$EMAIL")
RESPONSE=$(curl -sS -u "$PROJECTS_AUTH" \
  "${BASE_URL}/projects/api/v3/people.json?searchTerm=${EMAIL_ENCODED}&pageSize=10")
MATCHES=$(jq -r '
  [ (.people // []) | .[]
    | select((.email // .emailAddress // "") | ascii_downcase == ("'$EMAIL'" | ascii_downcase))
  ]' <<<"$RESPONSE")
```

- 0 matches → **AskUserQuestion**: `Try another email` / `Leave unassigned` /
  `Cancel run`
- 1 match → `ASSIGNEE_ID_FOR_<ROLE>=<id>`
- 2+ matches → **AskUserQuestion** with each candidate as an option (label =
  `<firstName> <lastName>`, description = `<email> · <jobTitle>`); pick one

Cache resolved emails for the rest of the run so duplicate roles (e.g. BE and
DevOps assigned to the same person) only resolve once.

---

## Step 7 — Clarifying questions

Same batched-`AskUserQuestion` pattern as `teamwork-task-analyze` Step 6.2:

- Max `desk_skill.clarifying_questions.max_per_run` total questions (default 6)
- Batch of 4 per **AskUserQuestion** call (Claude Code UI limit)
- Each question has 2–4 concrete options + free-form `Other` + a `Skip`
- Prioritise questions whose answer materially changes the AC, goal, or
  technical plan
- Skip questions answerable from the threads, attachments, or repo context
  already gathered
- Fold answers back:
  - Behaviour answers → AC items
  - "Why" answers → Goal
  - Implementation choices → Technical plan
  - Data/schema choices → Data model section
- Unanswered + dropped questions → preserve as `[OTVORENÉ] <question>` markers
  in the relevant section so they do not silently disappear

---

## Step 8 — Map attachments to tasks / subtasks

`ATTACH_MODE` comes from `--attach-mode` → `desk_skill.attachments.attach_mode`
(default `ask`).

### Step 8a — `ATTACH_MODE == "ask"`

For each downloaded attachment, run one **AskUserQuestion**:

> "Where does `<filename>` (<size>, <mimeType>) belong?"
>
> Options:
> - **Main task** — attach to the main task
> - **[BE] <subtask name>** — attach to subtask 1
> - **[FE] <subtask name>** — attach to subtask 2
> - … one per subtask, up to 3 shown (with a `More…` to pick from a longer list)
> - **Skip — don't upload**

Batch up to 4 attachment questions per **AskUserQuestion** call.

### Step 8b — `ATTACH_MODE == "distribute"`

Heuristic without asking:

- Filenames containing `mockup`, `wireframe`, `screen`, `ui`, `ux`, `figma`, `*.png`, `*.jpg` → first `[FE]` subtask (or main task)
- Filenames containing `schema`, `migration`, `*.sql`, `db`, `dump` → first `[BE]` or `[Migration]` subtask
- Filenames containing `e2e`, `cypress`, `playwright`, `test`, `scenario` → first `[QA]` subtask
- Filenames containing `deploy`, `infra`, `nginx`, `docker`, `.env.example` → first `[DevOps]` subtask
- Everything else → main task

### Step 8c — `ATTACH_MODE == "main"`

Everything on the main task, no questions.

Record the mapping as a simple structure:

```bash
# Pseudo-shell — store as jq-compatible JSON
{
  "main":    ["spec.pdf", "screenshot.png"],
  "sub_1":   ["schema.sql"],
  "sub_2":   ["mockup.png"],
  "skipped": ["weird-binary.bin"]
}
```

---

## Step 9 — Full preview + confirmation gate

Render a single markdown block summarising **everything** that will be created.
The user must approve before any write happens.

```markdown
## Concept preview

**Source Desk ticket:** <SUBJECT> (#<TICKET_ID>) — <DESK_BASE>/tickets/<TICKET_ID>
**Customer:** <CUSTOMER_FN> <CUSTOMER_LN> <<CUSTOMER_EMAIL>>
**Target project:** <PROJECT_NAME> (#<PROJECT_ID>)
**Target tasklist:** <TASKLIST_NAME> (#<TASKLIST_ID>)

### Main task

- **Title:** <MAIN_NAME>
- **Estimate:** <MAIN_EST> min
- **Assignee:** <name> (or "unassigned")
- **Attachments:** <count> — <filename1>, <filename2>, …
- **Description:**

  <full canonical-format description, fenced as a markdown block>

### Subtasks (<N>)

1. **[BE]** <subtask name>
   - Estimate: <n> min
   - Assignee: <name>
   - Attachments: <count>
   - AC (<n>): <first AC>; <second AC>; …
2. **[FE]** …

### Link-back to Desk

- Native Desk-link attribute (Projects API): probe in Step 10
- URL embedded in description footer (### Zdroj): yes
- Internal note in Desk thread after creation: <yes / no / will ask>
- Customer reply DRAFT (internal note): <yes / no / will ask>
```

Then **AskUserQuestion**:

- **Apply** — proceed to Step 10
- **Edit main task** — user pastes corrections (free-form), re-render
- **Edit subtask N** — same, per subtask
- **Drop subtasks** — keep only the main task with the full estimate
- **Cancel** — stop without any writes

If `--write-back=never` → stop after rendering the preview (full dry-run, no
**AskUserQuestion**).

If `--write-back=auto` → skip the **AskUserQuestion** and proceed to Step 10
directly.

---

## Step 10 — Create the main task in Projects

```bash
MAIN_PAYLOAD=$(jq -n \
  --arg name  "$MAIN_NAME" \
  --arg desc  "$MAIN_DESC" \
  --arg est   "$MAIN_EST" \
  --arg uid   "${ASSIGNEE_ID_FOR_MAIN:-}" \
  '{ task: (
       { name: $name, description: $desc, estimateMinutes: ($est|tonumber) }
       + ( if $uid == "" then {} else { assignees: [{type:"user", id:($uid|tonumber)}] } end )
  )}')

RESP=$(curl -sS -u "$PROJECTS_AUTH" \
  -H "Content-Type: application/json" -H "Accept: application/json" \
  -X POST -d "$MAIN_PAYLOAD" \
  "${BASE_URL}/projects/api/v3/tasklists/${TASKLIST_ID}/tasks.json")

MAIN_TASK_ID=$(jq -r '.task.id // .id // empty' <<<"$RESP")
if [ -z "$MAIN_TASK_ID" ]; then
  echo "❌ Main task creation failed:"; echo "$RESP"; exit 1
fi
MAIN_TASK_URL="${BASE_URL}/app/tasks/${MAIN_TASK_ID}"
```

Non-2xx on the main POST → render the error body, stop. This is the
load-bearing call.

### Step 10.1 — Probe + apply native Desk link (PATCH)

After the main task is created, probe whether the workspace's Projects API
exposes a native Desk-link attribute:

```bash
PROBE=$(curl -sS -u "$PROJECTS_AUTH" \
  "${BASE_URL}/projects/api/v3/tasks/${MAIN_TASK_ID}.json?include=deskTicket,helpDeskTicket")
HAS_NATIVE=$(jq -r '
  ((.task // {}) | has("deskTicket"))
  or ((.task // {}) | has("helpDeskTicket"))
  or ((.task // {}) | has("deskTicketId"))
  or ((.task // {}) | has("helpDeskTicketId"))
' <<<"$PROBE")
```

If `HAS_NATIVE == "true"`, attempt the PATCH with both candidate field names
(idempotent — the API will accept whichever it recognises and ignore the
other):

```bash
LINK_PAYLOAD=$(jq -n --arg tid "$TICKET_ID" \
  '{ task: { deskTicketId: ($tid|tonumber), helpDeskTicketId: ($tid|tonumber) } }')
curl -sS -u "$PROJECTS_AUTH" -X PATCH -d "$LINK_PAYLOAD" \
  -H "Content-Type: application/json" \
  "${BASE_URL}/projects/api/v3/tasks/${MAIN_TASK_ID}.json" > /dev/null
NATIVE_DESK_LINK="applied"
```

If `HAS_NATIVE == "false"`, fall back to the URL in `### Zdroj` (already in
the description from Step 5) + the optional Step 13 internal note. Set
`NATIVE_DESK_LINK="fallback"`.

---

## Step 11 — Create subtasks

For each confirmed subtask, in the order produced by Step 5.3:

```bash
SUB_PAYLOAD=$(jq -n \
  --arg name  "$SUB_NAME" \
  --arg desc  "$SUB_DESC" \
  --arg est   "$SUB_EST" \
  --arg parent "$MAIN_TASK_ID" \
  --arg uid   "${ASSIGNEE_ID_FOR_SUB:-}" \
  '{ task: (
       { name: $name, description: $desc, estimateMinutes: ($est|tonumber),
         parentTaskId: ($parent|tonumber) }
       + ( if $uid == "" then {} else { assignees: [{type:"user", id:($uid|tonumber)}] } end )
  )}')

SUB_RESP=$(curl -sS -u "$PROJECTS_AUTH" \
  -H "Content-Type: application/json" \
  -X POST -d "$SUB_PAYLOAD" \
  "${BASE_URL}/projects/api/v3/tasklists/${TASKLIST_ID}/tasks.json")

SUB_ID=$(jq -r '.task.id // .id // empty' <<<"$SUB_RESP")
if [ -n "$SUB_ID" ]; then
  SUBTASK_IDS+=("$SUB_ID")
  echo "✅ Subtask created: ${SUB_NAME} (#${SUB_ID})"
else
  SUBTASK_FAILURES+=("${SUB_NAME}: $(echo "$SUB_RESP" | jq -c '.errors // .message // .')")
  echo "❌ Subtask failed: ${SUB_NAME}"
fi
```

Per-subtask failure is **non-fatal** — log and continue with the rest.

---

## Step 12 — Upload attachments

Use the **pending-file** upload pattern. For each attachment in the Step 8
mapping that was not skipped:

```bash
# 1) Multipart upload — returns a pending-file ref
PENDING_RESP=$(curl -sS -u "$PROJECTS_AUTH" -X POST \
  -F "file=@${ATT_DIR}/${FILENAME}" \
  -F "fileName=${FILENAME}" \
  "${BASE_URL}/projects/api/v3/files.json?projectId=${PROJECT_ID}")

PENDING_REF=$(jq -r '
  (.pendingFile.ref) // (.pendingFile.id) // (.file.id) // (.id) // empty
' <<<"$PENDING_RESP")

if [ -z "$PENDING_REF" ]; then
  ATTACHMENT_FAILURES+=("${FILENAME}: pending-file upload failed: $(jq -c . <<<"$PENDING_RESP")")
  continue
fi

# 2) PATCH the target task with the pending file ref
# Some workspaces want pendingFileAttachments[], others pendingFileRefs[].
# Try the canonical shape first, fall back to the legacy shape on 4xx.
PATCH1=$(jq -n --arg ref "$PENDING_REF" \
  '{task:{attachments:{pendingFileAttachments:[$ref]}}}')
HTTP=$(curl -sS -o /tmp/tw_patch.json -w '%{http_code}' \
  -u "$PROJECTS_AUTH" -X PATCH -d "$PATCH1" \
  -H "Content-Type: application/json" \
  "${BASE_URL}/projects/api/v3/tasks/${TARGET_TASK_ID}.json")

if [ "$HTTP" != "200" ] && [ "$HTTP" != "204" ]; then
  PATCH2=$(jq -n --arg ref "$PENDING_REF" \
    '{task:{pendingFileRefs:[$ref]}}')
  HTTP=$(curl -sS -o /tmp/tw_patch.json -w '%{http_code}' \
    -u "$PROJECTS_AUTH" -X PATCH -d "$PATCH2" \
    -H "Content-Type: application/json" \
    "${BASE_URL}/projects/api/v3/tasks/${TARGET_TASK_ID}.json")
fi

if [ "$HTTP" = "200" ] || [ "$HTTP" = "204" ]; then
  echo "✅ Uploaded ${FILENAME} → task #${TARGET_TASK_ID}"
else
  ATTACHMENT_FAILURES+=("${FILENAME}: attach failed (HTTP $HTTP)")
fi
```

Per-file failure is non-fatal — log and continue.

---

## Step 13 — Internal note in Desk thread (link + structured summary)

`NOTIFY_DESK` comes from `--notify-desk` → `desk_skill.link_back.notify_desk_default`
(default `ask`).

If `NOTIFY_DESK == "ask"` → run one **AskUserQuestion**:

> "Post an internal note in the Desk ticket with a link to the new Projects
> task + a summary of what was prepared?"
>
> Options:
> - **Yes, post it**
> - **No, skip**
> - **Yes and don't ask next time** — flips `notify_desk_default` to `true` in
>   the shared config

If `NOTIFY_DESK == "false"` → skip the note (still record `⏭ skipped` in the
final report).

The note body is **never** just a link — it always includes a structured
summary of what was prepared so the support agent sees the outcome without
having to open the task:

```markdown
### 📋 Projects task created from this ticket

**Task:** [<MAIN_NAME>](<MAIN_TASK_URL>) (#<MAIN_TASK_ID>)
**Project:** <PROJECT_NAME> · **Tasklist:** <TASKLIST_NAME>
**Estimate:** <MAIN_EST> min (~<HOURS> h)
**Assignee:** <name> (or "unassigned")

**Akceptačné kritériá (<N>):**
- AC 1 — <first 80 chars>
- AC 2 — <first 80 chars>
- AC 3 — <first 80 chars>
- … (max 5 shown, rest as "+ <K> more in the task description")

**Subtasks (<N>):**
- [BE] <name> (#<id>) — <est> min — <assignee or "unassigned">
- [FE] <name> (#<id>) — <est> min — <assignee or "unassigned">

**Attachments uploaded:** <count> z Desk ticketu prenesené:
- <filename1> → main task
- <filename2> → subtask <name>

**Cieľ:** <first 2–3 sentences extracted from the ## Cieľ section>

— Vygenerované cez `/teamwork-tasks-from-desk` na <YYYY-MM-DD>
```

Post as `isInternal: true`:

```bash
NOTE_BODY=$(<the markdown body above, in a heredoc>)
NOTE_PAYLOAD=$(jq -n --arg body "$NOTE_BODY" \
  '{thread: { type: "note", body: $body, isInternal: true }}')
curl -sS -u "$DESK_AUTH" -X POST \
  -H "Content-Type: application/json" \
  -d "$NOTE_PAYLOAD" \
  "${DESK_API}/tickets/${TICKET_ID}/threads.json"
```

`type: "note"` and `isInternal: true` are **non-negotiable** — the plugin
deliberately never constructs a `type: "message"` / `reply` / `customer-facing`
payload, and never uses the `/replies.json` endpoint of the Desk API.

---

## Step 13b — Customer reply DRAFT (internal note ONLY, NEVER sent)

`DRAFT_REPLY` comes from `--draft-reply` → `desk_skill.customer_reply_draft.ask_default`
(default `ask`).

If `DRAFT_REPLY == "ask"` → run one **AskUserQuestion**:

> "Want me to draft a customer-facing reply about this work? It will be posted
> as an **INTERNAL note** in the ticket for your review — never sent to the
> customer directly."
>
> Options:
> - **Yes, draft a reply** — uses the default tone
> - **No, skip**
> - **Yes — formal tone**
> - **Yes — empathetic tone**

If `DRAFT_REPLY == "false"` → skip (record `⏭ skipped` in the final report).

### Draft generation

- **Language** — detected from the first customer-initiated thread body (sk /
  cz / en); fall back to `desk_skill.customer_reply_draft.language_fallback`
  (default `sk`)
- **Tone** — from `--draft-tone` → `desk_skill.customer_reply_draft.tone`
  (default `formal`); supports `formal`, `casual`, `empathetic`
- **Content shape** (sk example, formal tone):

  ```
  Dobrý deň pán/pani <CUSTOMER_LN>,

  ďakujeme za váš podnet týkajúci sa <téma rekapitulovaná z prvej zákazníckej
  správy, 1–2 vety, business-friendly bez technických detailov>.

  Vaš podnet sme zaevidovali a budeme ho riešiť v priebehu cca <estimate
  prepočítaný na pracovné dni — round up to whole or half days>. <Stručný popis
  toho, čo bude doručené — 1–2 vety zo `## Cieľ` sekcie tasku, naozaj
  business-friendly>.

  [Iba ak ostali nevyriešené [OTVORENÉ] body a config.include_open_questions = true:]
  Aby sme prácu mohli začať bez prerušenia, potrebovali by sme od vás ešte
  doplniť:

  - <[OTVORENÉ] question 1>
  - <[OTVORENÉ] question 2>

  <signature from config>
  ```

### Posting the DRAFT

The DRAFT is wrapped in a **clearly visible review banner** so a support agent
who later opens the ticket cannot mistake it for an actual reply:

```markdown
### ✉️ DRAFT — návrh odpovede pre klienta (pred odoslaním skontrolovať)

> **Toto je INTERNÁ POZNÁMKA. Nebola odoslaná klientovi. Skopíruj, uprav podľa
> potreby a odošli ako Reply ručne.**

---

<the generated reply body>

---

— Vygenerované cez `/teamwork-tasks-from-desk` na <YYYY-MM-DD>
```

Post identical to Step 13 — `type: "note"`, `isInternal: true`:

```bash
DRAFT_PAYLOAD=$(jq -n --arg body "$DRAFT_BODY" \
  '{thread: { type: "note", body: $body, isInternal: true }}')
curl -sS -u "$DESK_AUTH" -X POST \
  -H "Content-Type: application/json" \
  -d "$DRAFT_PAYLOAD" \
  "${DESK_API}/tickets/${TICKET_ID}/threads.json"
```

### Hard guarantees

- `type: "note"` and `isInternal: true` are set on **every** Desk POST this
  skill makes. The plugin never constructs any payload of `type: "message"`,
  `type: "reply"`, or anything customer-facing.
- The plugin never calls `POST ${DESK_API}/tickets/<id>/replies.json` — that
  endpoint is the only way to send a customer-facing reply in the Desk API,
  and it is explicitly out of scope.
- The plugin does not collect the customer's email into any `to:`/`cc:`/`bcc:`
  payload field; it has nowhere to *send* a reply even by accident.
- The DRAFT body always begins with the review banner above so it is visually
  distinct from any genuine reply.

---

## Step 14 — Final report

Render a markdown summary into the terminal:

```markdown
## 📨 Desk → Projects task created

**Desk ticket:** <SUBJECT> (#<TICKET_ID>) — <DESK_BASE>/tickets/<TICKET_ID>
**Projects task:** <MAIN_NAME> (#<MAIN_TASK_ID>) — <MAIN_TASK_URL>
**Estimate:** <MAIN_EST> min · **Assignee:** <name or "unassigned">

### Subtasks
- ✅ [BE] <name> (#<id>) — <est> min — assignee: <name>
- ✅ [FE] <name> (#<id>) — <est> min — assignee: <name>
- ❌ [QA] <name> — <reason>

### Attachments
- ✅ <file1> → main task
- ✅ <file2> → subtask [FE] <name>
- ⏭ <file3> skipped (size > 25 MB)
- ❌ <file4> upload failed: <reason>

### Link-back to Desk
- Native Projects API attribute: <applied / fallback>
- URL in description footer (### Zdroj): yes
- Internal Desk note (link + summary): ✅ posted at <DESK_BASE>/tickets/<TICKET_ID> / ⏭ skipped
- Customer reply DRAFT (internal note): ✅ posted for review / ⏭ skipped

### Unresolved
- [OTVORENÉ] <question 1>
- [OTVORENÉ] <question 2>
```

Cleanup (when `desk_skill.attachments.cleanup == "after_run"`):

```bash
rm -rf "${TMPDIR:-/tmp}/teamwork-tasks-from-desk-${RUN_ID}"
```

When `cleanup == "keep"`, print the directory path so the user can inspect it.

---

## Hard boundaries (what this skill **never** does)

- Never posts a customer-facing reply on the Desk ticket
- Never moves the Desk ticket between statuses (open / pending / closed)
- Never reassigns the Desk ticket to a different agent or inbox
- Never sends an email to the customer
- Never moves the new Projects task between board columns
- Never logs time on the new Projects task
- Never marks the new task or subtasks as complete
- Never runs `git commit`, `git push`, or any git command
- Never overwrites an existing Projects task — running on the same Desk ticket
  twice will create a second task (idempotency detection is deferred to a
  future version; if you re-run, cancel at Step 9 and clean up manually)

---

## Failure modes & recovery

| Failure | Behaviour |
|---|---|
| Desk URL unparseable | AskUserQuestion for a corrected URL, stop if still bad |
| Desk 401 | Re-prompt the Desk token (Step 2 first-run flow) |
| Desk 403/404 | Stop with the failing URL printed |
| Desk threads endpoint 404 | Probe v1 → unversioned; record the working one in config |
| Projects URL unparseable | AskUserQuestion for a corrected URL |
| Projects 401 | Re-prompt the Projects token (shared with siblings) |
| Projects 403 on tasklist | Stop with a clear "no access" message |
| Email resolves to 0 people | AskUserQuestion (retry / unassigned / cancel) |
| Email resolves to ≥2 people | AskUserQuestion to pick the right person |
| Main task POST non-2xx | Render body, stop (load-bearing) |
| Subtask POST non-2xx | Log `❌`, continue with the rest |
| Attachment download non-2xx | Log `⏭`, continue |
| Attachment PATCH non-2xx | Try the alternative payload shape, then log `❌` |
| Desk note POST non-2xx | Log `❌`, continue — the task is already created |

---

## Persistent learning

On each successful run, the skill writes back into the shared config:

- `desk_skill.api_version` — the working Desk API version (set in Step 3)
- `desk_skill.link_back.notify_desk_default` — if the user picked "Yes and
  don't ask next time" in Step 13
- `desk_skill.attachments.upload_payload_shape` — the working pending-file
  shape (`pendingFileAttachments` vs `pendingFileRefs`) discovered in Step 12,
  so subsequent runs skip the probe

All writes are atomic (`jq` + `mv` + `chmod 600`).
