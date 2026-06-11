---
name: teamwork-tasks-from-desk
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

### Step 1.1 — Record run start time (for Step 14 duration report)

**New in 1.0.1.** Capture a monotonic-friendly start timestamp **before any
network I/O** so that the final report can show how long the whole run took
(skill invocation → task created → internal note posted). Use UTC ISO-8601 +
epoch seconds — the epoch lets us compute the delta without date parsing
gymnastics on macOS, and the ISO string is human-readable in the final report.

```bash
RUN_START_EPOCH=$(date -u +%s)
RUN_START_ISO=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
```

These two variables are referenced once more in Step 14. Do **not** try to make
them session-global / exported — `bash` subshells in pipeline stages would lose
them. Keep them as plain shell variables in the top scope of the run.

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
9. **Never echo any token.** For older opaque tokens, pass auth via
   `-u "$TOKEN:xxx"` (Basic auth, kept out of `ps`). For newer tokens that
   begin with `tkn.v1_` use HTTP Bearer instead (see Step 2.6 below — Bearer is
   detected automatically and persisted in `.desk_skill.auth_scheme`).
   Never put a token in a URL or in `set -x` output.

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
| .desk_skill.sections.preamble_label               //= ""
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
| .desk_skill.auth_scheme                           //= ""
| .desk_skill.note_payload_shape                    //= ""
| .desk_skill.note_body_format                      //= "html"
| .desk_skill.task_create_estimate_field            //= "estimatedMinutes"
| .desk_skill.attachment_upload_endpoint            //= ""
| .desk_skill.attachment_attach_endpoint            //= ""
| .desk_skill.file_download_endpoint                //= ""
| .desk_skill.timing                                //= {}
| .desk_skill.timing.report_run_duration            //= true
| .desk_skill.timing.duration_format                //= "human"
' "$CONFIG_FILE" > "$TMP" && mv "$TMP" "$CONFIG_FILE" && chmod 600 "$CONFIG_FILE"
```

New keys introduced in **1.0.1** (all empty defaults — populated on first
successful run, idempotently re-used on every subsequent run):

- `desk_skill.auth_scheme` — `"bearer"` or `"basic"`. Probed once in Step 2.6
  and persisted; subsequent runs skip the probe.
- `desk_skill.note_payload_shape` — `"v2_messages_top_level"` (modern Desk
  workspaces) or `"v2_threads_nested"` (older). Probed once in Step 13 by
  posting a no-op note + DELETE if delete is allowed; otherwise probed by
  POST + observing the failure-vs-success contract. Persisted.
- `desk_skill.note_body_format` — `"html"` (default, since markdown is **not**
  rendered by Teamwork Desk) or `"text"`. Step 13/13b render the note body
  with HTML tags (`<h3>`, `<p>`, `<a>`, `<pre>`, `<ul>/<li>`, `<blockquote>`)
  instead of markdown so it displays correctly in the Desk UI.
- `desk_skill.task_create_estimate_field` — request field name for setting the
  estimate at POST time. Defaults to `estimatedMinutes` (Teamwork v3 quirk:
  the **read** field is `estimateMinutes` but the **write** field is
  `estimatedMinutes` with a `d` — they are not interchangeable). Persisted so
  Step 10 can use it directly.
- `desk_skill.attachment_upload_endpoint` — `"v1/pendingFiles"` (proven
  working) or `"v3/files"`. Probed and persisted in Step 12.
- `desk_skill.attachment_attach_endpoint` — `"v1_put_task"` (proven working
  with `pendingFileAttachments: "<ref>"` string) or `"v3_patch_attachments"`.
- `desk_skill.file_download_endpoint` — `"v2_download_json"` for the working
  `/files/{id}/download.json` (303 redirect → signed S3) path. Older versions
  used `/files/{id}.json` (metadata, 403 on most tiers).
- `desk_skill.timing` — controls Step 14 duration reporting (default ON).

Load all values into local variables for the rest of the run:

```bash
PROJECTS_TOKEN=$(jq -r '.teamwork.api_token'  "$CONFIG_FILE")
DESK_TOKEN=$(   jq -r '.teamwork.desk_token'  "$CONFIG_FILE")
BASE_URL=$(     jq -r '.teamwork.base_url'    "$CONFIG_FILE")
DESK_BASE=$(    jq -r '(.teamwork.desk_base_url // "") | if . == "" then "'"$BASE_URL"'/desk" else . end' "$CONFIG_FILE")
PROJECTS_AUTH="${PROJECTS_TOKEN}:xxx"     # Basic auth for Projects (always)
```

`DESK_AUTH` is **not** set here yet — Desk auth scheme is auto-detected in
Step 2.6 below because newer `tkn.v1_*` tokens require Bearer auth, while
older opaque tokens use Basic.

### Step 2.6 — Auto-detect Desk auth scheme (Bearer vs Basic) — NEW in 1.0.1

Teamwork Desk supports two auth schemes:

- **Basic auth** — `-u "${DESK_TOKEN}:xxx"` for legacy opaque tokens.
- **Bearer auth** — `-H "Authorization: Bearer ${DESK_TOKEN}"` for newer tokens
  generated via the modern Desk Settings → API Keys flow. These tokens have a
  recognisable prefix (`tkn.v1_…`) and **Basic auth returns 401** for them.

The skill probes both schemes once and persists the working one in
`desk_skill.auth_scheme`. Subsequent runs skip the probe.

```bash
DESK_AUTH_SCHEME=$(jq -r '.desk_skill.auth_scheme // ""' "$CONFIG_FILE")

probe_desk_auth() {
  # Tries the requested scheme against /me.json (cheap, always available).
  local scheme="$1"
  local probe_url="${DESK_BASE}/api/v2/me.json"
  case "$scheme" in
    bearer) curl -sS -o /dev/null -w '%{http_code}' \
              -H "Authorization: Bearer $DESK_TOKEN" "$probe_url" ;;
    basic)  curl -sS -o /dev/null -w '%{http_code}' \
              -u "${DESK_TOKEN}:xxx" "$probe_url" ;;
  esac
}

if [ -z "$DESK_AUTH_SCHEME" ]; then
  # Probe in priority order: bearer (newer) → basic (legacy).
  for SCHEME in bearer basic; do
    HTTP=$(probe_desk_auth "$SCHEME")
    if [ "$HTTP" = "200" ]; then DESK_AUTH_SCHEME="$SCHEME"; break; fi
  done
  if [ -z "$DESK_AUTH_SCHEME" ]; then
    echo "❌ Could not authenticate to Desk API at ${DESK_BASE}. Tried bearer + basic on /me.json."
    echo "   Re-check the Desk token in ~/.claude/plugins/data/teamwork-task-wamesk/config.json"
    exit 1
  fi
  TMP=$(mktemp); jq --arg s "$DESK_AUTH_SCHEME" '.desk_skill.auth_scheme = $s' \
    "$CONFIG_FILE" > "$TMP" && mv "$TMP" "$CONFIG_FILE" && chmod 600 "$CONFIG_FILE"
fi
```

Build a single `desk_curl` helper used by every subsequent Desk API call so the
auth scheme is applied transparently and never duplicated:

```bash
desk_curl() {
  # Usage: desk_curl <curl-args...>
  # Adds the correct auth header/flag based on $DESK_AUTH_SCHEME.
  case "$DESK_AUTH_SCHEME" in
    bearer) curl -sS -H "Authorization: Bearer $DESK_TOKEN" "$@" ;;
    basic)  curl -sS -u "${DESK_TOKEN}:xxx" "$@" ;;
  esac
}
```

From this point onward in the document, replace every `curl -sS -u "$DESK_AUTH" …`
or `-H "Authorization: Bearer …"` invocation with `desk_curl …`.

---

## Step 3 — Fetch the Desk ticket

The version path may be `/desk/api/v2/` on most workspaces; on older ones it
may be `/desk/api/v1/`. The skill probes v2 first and falls back gracefully,
recording the working version into the config for subsequent runs:

```bash
DESK_API_VERSION=$(jq -r '.desk_skill.api_version // ""' "$CONFIG_FILE")
if [ -z "$DESK_API_VERSION" ]; then
  for V in v2 v1; do
    HTTP=$(desk_curl -o /dev/null -w '%{http_code}' \
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
TICKET_JSON=$(desk_curl \
  "${DESK_API}/tickets/${TICKET_ID}.json?include=customer,inbox,assignee")
```

**Note on `?include=`** — v2 may return an empty `included: []` even when the
include parameter is accepted (workspace-tier dependent). The skill must
**not** assume `included.customers[]` / `included.inboxes[]` are populated.
If they are empty, fetch each by id separately:

```bash
CUSTOMER_ID=$(jq -r '.ticket.customer.id // .customer.id // empty' <<<"$TICKET_JSON")
INBOX_ID=$(   jq -r '.ticket.inbox.id    // .inbox.id    // empty' <<<"$TICKET_JSON")

if [ -n "$CUSTOMER_ID" ] && [ "$(jq -r '.included.customers // {} | length' <<<"$TICKET_JSON")" = "0" ]; then
  CUSTOMER_JSON=$(desk_curl "${DESK_API}/customers/${CUSTOMER_ID}.json")
fi
if [ -n "$INBOX_ID" ] && [ "$(jq -r '.included.inboxes // {} | length' <<<"$TICKET_JSON")" = "0" ]; then
  INBOX_JSON=$(desk_curl "${DESK_API}/inboxes/${INBOX_ID}.json")
fi
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

Try `/threads.json` first (older shape); on 404 fall back to `/messages.json`
(modern Desk workspaces — what the v2 path actually exposes):

```bash
THREADS_URL="${DESK_API}/tickets/${TICKET_ID}/threads.json?page=1&pageSize=100&orderBy=createdAt&orderMode=asc"
HTTP=$(desk_curl -o /tmp/threads.json -w '%{http_code}' "$THREADS_URL")

if [ "$HTTP" != "200" ]; then
  # Modern workspaces only expose /messages.json — same shape but uses
  # `messages[]` instead of `threads[]` and `threadType` enum instead of
  # `channel`. Both schemes carry an `htmlBody`/`textBody` per item.
  THREADS_URL="${DESK_API}/tickets/${TICKET_ID}/messages.json?page=1&pageSize=100&orderBy=createdAt&orderMode=asc"
  desk_curl -o /tmp/threads.json "$THREADS_URL"
fi

THREADS=$(cat /tmp/threads.json)
```

Page through if `pagination.pages > 1` (older: `meta.totalPages`). Item array
key is whichever of `threads[]` or `messages[]` is present. For each item
collect: `id`, `htmlBody`, `textBody`, `createdAt`, `createdBy.id`,
`threadType` (one of `message`, `note`, `eventInfo`), `files[]`.

The `createdBy.type` distinguishes `customers` (the ticket reporter) from
`users` (your agents). Look up the customer's first message by filtering
`threadType == "message" AND createdBy.type == "customers"` and picking the
earliest `createdAt`. This is the source for the AC extraction in Step 5.

**Markdown conversion:** convert each `htmlBody` to markdown — prefer
`pandoc -f html -t markdown_strict`, fall back to `python -m html2text` or a
`sed`-based stripper. Keep chronological order: earliest first.

**Agent identity resolution:** the team-side notes carry only
`createdBy.id` referring to users. To label them (`agent Mário`, `Allcoo Bot`),
fetch each unique user id once via `GET ${DESK_API}/users/${id}.json` (200 OK
on most tiers). Cache the lookups for the rest of the run.

### Step 3.3 — Attachments

Each thread/message item carries a `files: [{id, type: "files"}, …]` array
referencing **file ids** on the ticket. Collect every unique file id across
the chronological items.

The dedicated `/tickets/{id}/attachments.json` endpoint and the
`/files/{id}.json` metadata endpoint are **gated** behind a "You Must Upgrade
Your Account" tier on many workspaces (both return HTTP 403 + error code E5).
Do not rely on them for either listing or metadata.

```bash
# Build file_id set from the threads payload directly — NOT from /attachments.json
FILE_IDS=$(jq -r '
  (.messages // .threads // []) | .[]?.files // [] | .[]?.id
' /tmp/threads.json | sort -u)
```

For each `file_id`, attempt the **download endpoint with the 303-follow
trick** (works on the gated tier where the metadata endpoint does not):

```bash
RUN_ID="$(date +%s)-${RANDOM}"
ATT_DIR="${TMPDIR:-/tmp}/teamwork-tasks-from-desk-${RUN_ID}/attachments"
mkdir -p "$ATT_DIR"

MAX_MB=$(jq -r '.desk_skill.attachments.max_attachment_size_mb' "$CONFIG_FILE")
ALLOW=( $(jq -r '.desk_skill.attachments.extensions_allow[]' "$CONFIG_FILE") )

for FILE_ID in $FILE_IDS; do
  # ★ The working download endpoint (returns 303 → signed S3 URL):
  #   GET ${DESK_API}/files/${FILE_ID}/download.json
  # NOT ${DESK_API}/files/${FILE_ID}.json   ← 403 on gated tiers
  # NOT ${DESK_API}/tickets/${TICKET_ID}/attachments.json  ← 403 on gated tiers
  TMP_PATH="${ATT_DIR}/_pending_${FILE_ID}"
  HTTP=$(desk_curl -L -o "$TMP_PATH" -w '%{http_code}' \
    "${DESK_API}/files/${FILE_ID}/download.json")

  if [ "$HTTP" != "200" ]; then
    ATTACHMENT_FAILURES+=("file_${FILE_ID}: download HTTP ${HTTP}")
    rm -f "$TMP_PATH"
    continue
  fi

  # File-size check — the size is only known post-download here because the
  # /files/{id}.json metadata endpoint is gated. Cheap enough: if oversized,
  # discard and warn.
  SIZE=$(stat -f %z "$TMP_PATH" 2>/dev/null || stat -c %s "$TMP_PATH")
  SIZE_MB=$(( SIZE / 1048576 ))
  if [ "$SIZE_MB" -gt "$MAX_MB" ]; then
    echo "⏭ file_${FILE_ID} too large (${SIZE_MB} MB > ${MAX_MB} MB)"
    rm -f "$TMP_PATH"
    continue
  fi

  # Sniff the real filename from the (S3) Content-Disposition header.
  # Fallback: probe the .eml first message body (contains "File:  XXX")
  # or use file_${FILE_ID} as a last resort.
  FILENAME="$(desk_curl -I -L "${DESK_API}/files/${FILE_ID}/download.json" \
    | grep -i 'content-disposition' \
    | sed -nE 's/.*filename="?([^"]+)"?.*/\1/p' \
    | tr -d '\r' | head -1)"
  [ -z "$FILENAME" ] && FILENAME="file_${FILE_ID}"

  # Extension allow-list check
  EXT="${FILENAME##*.}"; EXT="${EXT,,}"
  if ! printf '%s\n' "${ALLOW[@]}" | grep -qFx "$EXT"; then
    echo "⏭ ${FILENAME}: extension '${EXT}' blocked by config"
    rm -f "$TMP_PATH"
    continue
  fi

  mv "$TMP_PATH" "${ATT_DIR}/${FILENAME}"
done
```

Persist the working file-download endpoint on first success:

```bash
TMP=$(mktemp); jq '.desk_skill.file_download_endpoint = "v2_download_json"' \
  "$CONFIG_FILE" > "$TMP" && mv "$TMP" "$CONFIG_FILE" && chmod 600 "$CONFIG_FILE"
```

For binary documents (PDF/DOCX/XLSX/PPTX), also extract text into a sibling
`.txt` so the Step 5 reasoning can see the content:

```bash
case "$EXT" in
  pdf)  pdftotext -layout "$F" "${F%.pdf}.txt"  2>/dev/null || true ;;
  docx) pandoc -f docx -t plain "$F" -o "${F%.docx}.txt" 2>/dev/null || true ;;
  xlsx) python3 - "$F" > "${F%.xlsx}.txt" 2>/dev/null <<'PY' || true
# Pure-stdlib XLSX -> TSV text extractor (no openpyxl/pandas needed).
# An .xlsx is a zip of XML parts: shared strings live in sharedStrings.xml,
# cell values in xl/worksheets/sheetN.xml. We resolve shared-string indices
# and emit one tab-separated line per row, sheets separated by a blank line.
import sys, zipfile, re
from xml.etree import ElementTree as ET
from xml.parsers import expat

NS = "{http://schemas.openxmlformats.org/spreadsheetml/2006/main}"

def parse(data):
    # Harden against XXE / billion-laughs from a malicious .xlsx. Valid OOXML
    # never contains a DOCTYPE, so reject any DTD outright (this fires BEFORE
    # any entity is declared) and refuse external entity resolution. Build an
    # ElementTree on top of a raw expat parser so the guards actually stick.
    builder = ET.TreeBuilder()
    # namespace_separator -> expat emits "uri\tlocalname"; rewrite to the
    # "{uri}localname" form ElementTree (and the f"{NS}..." lookups) expect.
    p = expat.ParserCreate(namespace_separator="\t")
    def _no_dtd(*a):
        raise ValueError("DTD/DOCTYPE not allowed in xlsx XML")
    def _fix(tag):
        return "{%s}%s" % tuple(tag.split("\t", 1)) if "\t" in tag else tag
    p.StartDoctypeDeclHandler = _no_dtd
    p.EntityDeclHandler = _no_dtd
    p.ExternalEntityRefHandler = lambda *a: False
    p.StartElementHandler = lambda tag, attrs: builder.start(
        _fix(tag), {_fix(k): v for k, v in attrs.items()})
    p.EndElementHandler = lambda tag: builder.end(_fix(tag))
    p.CharacterDataHandler = lambda d: builder.data(d)
    p.Parse(data, True)
    return builder.close()

def col_index(ref):
    m = re.match(r"([A-Z]+)", ref or "")
    if not m:
        return 0
    n = 0
    for ch in m.group(1):
        n = n * 26 + (ord(ch) - ord("A") + 1)
    return n - 1

with zipfile.ZipFile(sys.argv[1]) as z:
    shared = []
    if "xl/sharedStrings.xml" in z.namelist():
        root = parse(z.read("xl/sharedStrings.xml"))
        for si in root.findall(f"{NS}si"):
            shared.append("".join(t.text or "" for t in si.iter(f"{NS}t")))

    sheets = sorted(n for n in z.namelist()
                    if re.match(r"xl/worksheets/sheet\d+\.xml$", n))
    out = []
    for sheet in sheets:
        root = parse(z.read(sheet))
        for row in root.iter(f"{NS}row"):
            cells = {}
            maxc = -1
            for c in row.findall(f"{NS}c"):
                idx = col_index(c.get("r"))
                maxc = max(maxc, idx)
                v = c.find(f"{NS}v")
                if c.get("t") == "s" and v is not None:
                    try:
                        cells[idx] = shared[int(v.text)]
                    except (ValueError, IndexError):
                        cells[idx] = ""
                elif c.get("t") == "inlineStr":
                    t = c.find(f"{NS}is/{NS}t")
                    cells[idx] = t.text or "" if t is not None else ""
                else:
                    cells[idx] = v.text or "" if v is not None else ""
            out.append("\t".join(cells.get(i, "") for i in range(maxc + 1)))
        out.append("")
    sys.stdout.write("\n".join(out))
PY
        ;;
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

### Step 8.1 — Materialise the file lists for the upload loop (Step 12)

Bash has **no nested arrays**, so do **not** try `SUBTASK_FILES[$i][@]`.
Instead materialise the mapping above into one flat array for the main task and
one newline-delimited string variable per subtask index. Subtask index `i` is
0-based and lines up with `SUBTASK_IDS[$i]` from Step 11. Store **absolute**
paths under `$ATT_DIR`.

```bash
# Main task files — a flat bash array of absolute paths.
MAIN_TASK_FILES=()
for f in "${MAP_MAIN[@]}"; do          # MAP_MAIN = filenames mapped to "main"
  MAIN_TASK_FILES+=("${ATT_DIR}/${f}")
done

# Per-subtask files — one newline-delimited string variable per index, named
# SUBTASK_FILES_<i>. Build them from the sub_<n> buckets (sub_1 → index 0, …).
for n in "${!SUBTASK_NAMES[@]}"; do     # n is 0-based; bucket key is sub_$((n+1))
  bucket="MAP_SUB_$((n + 1))"          # e.g. MAP_SUB_1 = array of filenames
  # bash 3.2-safe dynamic array read. `declare -n` (nameref) needs bash 4.3+,
  # but macOS ships bash 3.2.57 where it silently fails and leaves the lists
  # empty — so the subtask attachments would never upload. eval copies the
  # dynamically-named array into ref; an unset/empty bucket yields an empty ref.
  eval "ref=( \"\${${bucket}[@]}\" )" 2>/dev/null || ref=()
  lines=""
  for f in "${ref[@]}"; do
    lines+="${ATT_DIR}/${f}"$'\n'
  done
  printf -v "SUBTASK_FILES_${n}" '%s' "$lines"
done
```

The `eval "ref=( \"\${${bucket}[@]}\" )"` form above is intentional: it reads a
dynamically-named array without `declare -n` (nameref), so it works on macOS
stock bash 3.2 as well as modern bash. `printf -v "SUBTASK_FILES_${n}"` is also
bash 3.1+ safe, so the whole materialisation block is portable.

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

**Critical 1.0.1 lessons:**

1. The Teamwork v3 task POST **rejects multi-line non-ASCII descriptions**
   when encoded by `jq` because `jq -n --arg desc "$MAIN_DESC"` may pass
   embedded control characters through unescaped. **Always encode the JSON
   payload with Python**, which uses `json.dump(..., ensure_ascii=False)` and
   handles every code point correctly.
2. The **read** field for an estimate on a Teamwork v3 task is
   `estimateMinutes`. The **write** field is `estimatedMinutes` (extra `d`).
   They are **not** interchangeable. Sending `estimateMinutes` on POST is
   silently accepted but the value lands as 0. Use the persisted
   `desk_skill.task_create_estimate_field` config key (default
   `estimatedMinutes`).
3. **Idempotency guard:** if the same POST appears to fail (e.g. a downstream
   `jq` view of the response throws a parse error), the task **may have
   actually been created**. Before retrying, query
   `GET /projects/api/v3/tasklists/{id}/tasks.json?searchTerm={MAIN_NAME}` and
   skip the retry if a freshly-created (last ~60 s) task with the same name
   already exists.

```bash
# Write the task NAME and description body to temp files so Python can read
# them raw — this avoids every quoting + escaping pitfall. NEVER splice the
# name into Python source via shell quoting (e.g. ${MAIN_NAME@Q}): shell
# quoting is not valid Python and a crafted name can break out and execute.
NAME_PATH="$(mktemp)"
printf '%s' "$MAIN_NAME" > "$NAME_PATH"
DESC_PATH="$(mktemp)"
printf '%s' "$MAIN_DESC" > "$DESC_PATH"

EST_FIELD=$(jq -r '.desk_skill.task_create_estimate_field // "estimatedMinutes"' "$CONFIG_FILE")

PAYLOAD_PATH="$(mktemp)"
python3 - "$NAME_PATH" "$DESC_PATH" "$PAYLOAD_PATH" <<PY
import json, sys
name_path, desc_path, payload_path = sys.argv[1], sys.argv[2], sys.argv[3]
name = open(name_path).read()
desc = open(desc_path).read()
payload = {
    "task": {
        "name":            name,
        "description":     desc,
        "${EST_FIELD}":    ${MAIN_EST},
    }
}
assignee_id = ${ASSIGNEE_ID_FOR_MAIN:-0}
if assignee_id:
    payload["task"]["assignees"] = {"userIds": [assignee_id]}
parent_id = ${PARENT_TASK_ID:-0}
if parent_id:
    payload["task"]["parentTaskId"] = parent_id
json.dump(payload, open(payload_path, "w"), ensure_ascii=False)
PY

# ★ Idempotency probe BEFORE the POST — only triggered on retry.
#   Skip on the very first attempt (RETRY_ATTEMPT = 0) for speed.
if [ "${RETRY_ATTEMPT:-0}" -gt 0 ]; then
  # URL-encode the task name for the searchTerm query param.
  MAIN_NAME_URL_ENCODED=$(python3 -c \
    "import urllib.parse,sys; print(urllib.parse.quote(sys.argv[1]))" "$MAIN_NAME")
  # Pass the name to Python via argv (never via shell quoting into source).
  EXISTING=$(curl -sS -u "$PROJECTS_AUTH" \
    "${BASE_URL}/projects/api/v3/tasklists/${TASKLIST_ID}/tasks.json?searchTerm=${MAIN_NAME_URL_ENCODED}&pageSize=5" \
    | python3 -c "
import json,sys,datetime
want=sys.argv[1]
d=json.load(sys.stdin); now=datetime.datetime.utcnow()
for t in d.get('tasks',[]):
    if t.get('name') == want:
        # Skip if created in the last 60 s (assume it is our prior attempt).
        ts = t.get('dateCreated') or t.get('dateUpdated') or ''
        try:
            dt = datetime.datetime.strptime(ts.rstrip('Z'), '%Y-%m-%dT%H:%M:%S')
            if (now - dt).total_seconds() < 60:
                print(t['id']); break
        except Exception: pass
" "$MAIN_NAME")
  if [ -n "$EXISTING" ]; then
    MAIN_TASK_ID="$EXISTING"
    echo "↩ Reusing existing task #${MAIN_TASK_ID} (idempotency guard hit)"
  fi
fi

if [ -z "${MAIN_TASK_ID:-}" ]; then
  HTTP=$(curl -sS -u "$PROJECTS_AUTH" \
    -H "Content-Type: application/json" -H "Accept: application/json" \
    -X POST -d @"$PAYLOAD_PATH" \
    -o /tmp/tw_main_resp.json -w '%{http_code}' \
    "${BASE_URL}/projects/api/v3/tasklists/${TASKLIST_ID}/tasks.json")

  MAIN_TASK_ID=$(python3 -c "
import json
d = json.load(open('/tmp/tw_main_resp.json'))
t = d.get('task', d)
print(t.get('id') or '')")
  if [ -z "$MAIN_TASK_ID" ]; then
    echo "❌ Main task creation failed (HTTP ${HTTP}):"
    head -c 1000 /tmp/tw_main_resp.json
    exit 1
  fi
fi

MAIN_TASK_URL="${BASE_URL}/app/tasks/${MAIN_TASK_ID}"
echo "✅ Main task created: ${MAIN_NAME} (#${MAIN_TASK_ID})"
rm -f "$NAME_PATH" "$DESC_PATH" "$PAYLOAD_PATH"
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

For each confirmed subtask, in the order produced by Step 5.3.

**Per-role assignee mapping** — Step 6.1 produced a map of role → userId
(empty for unassigned). The role for each subtask comes from its name prefix
(`[BE]`, `[FE]`, `[QA]`, `[Migration]`, `[DevOps]`). When constructing the
subtask payload, look up the userId for that role from the Step 6 map:

```bash
# ASSIGNEE_MAP is a JSON object built in Step 6:
#   { "BE": "12345", "FE": "67890", "QA": "", ... }
# Parse the role from the subtask name prefix:
SUB_ROLE=$(echo "$SUB_NAME" | sed -nE 's|^\[([A-Za-z]+)\].*|\1|p')
SUB_ASSIGNEE_ID=$(jq -r --arg r "$SUB_ROLE" '.[$r] // ""' <<<"$ASSIGNEE_MAP")

# When the subtask has no role prefix (single-role task, or split disabled),
# fall back to the main task's assignee:
if [ -z "$SUB_ROLE" ]; then
  SUB_ASSIGNEE_ID="${ASSIGNEE_ID_FOR_MAIN:-}"
fi
```

Then POST the subtask using the **same Python-encoded payload + correct write
field** (`estimatedMinutes`) as Step 10:

```bash
# Write the subtask NAME and description to temp files so Python reads them
# raw — never splice them into Python source via shell quoting (${SUB_NAME@Q}).
SUB_NAME_PATH="$(mktemp)"
printf '%s' "$SUB_NAME" > "$SUB_NAME_PATH"
SUB_DESC_PATH="$(mktemp)"
printf '%s' "$SUB_DESC" > "$SUB_DESC_PATH"
SUB_PAYLOAD_PATH="$(mktemp)"
python3 - "$SUB_NAME_PATH" "$SUB_DESC_PATH" "$SUB_PAYLOAD_PATH" <<PY
import json, sys
name = open(sys.argv[1]).read()
desc = open(sys.argv[2]).read()
payload = {
    "task": {
        "name":            name,
        "description":     desc,
        "${EST_FIELD}":    ${SUB_EST},
        "parentTaskId":    ${MAIN_TASK_ID},
    }
}
uid = ${SUB_ASSIGNEE_ID:-0}
if uid:
    payload["task"]["assignees"] = {"userIds": [uid]}
json.dump(payload, open(sys.argv[3], "w"), ensure_ascii=False)
PY

SUB_RESP=$(curl -sS -u "$PROJECTS_AUTH" \
  -H "Content-Type: application/json" \
  -X POST -d @"$SUB_PAYLOAD_PATH" \
  "${BASE_URL}/projects/api/v3/tasklists/${TASKLIST_ID}/tasks.json")

# Feed the raw server response via stdin (like the main-task response is read
# from a file) — never splice it into Python source via ${SUB_RESP@Q}.
SUB_ID=$(printf '%s' "$SUB_RESP" | python3 -c "
import json, sys
d = json.load(sys.stdin)
t = d.get('task', d)
print(t.get('id') or '')")
if [ -n "$SUB_ID" ]; then
  SUBTASK_IDS+=("$SUB_ID")
  echo "✅ Subtask created: ${SUB_NAME} (#${SUB_ID})"
else
  SUBTASK_FAILURES+=("${SUB_NAME}: $SUB_RESP")
  echo "❌ Subtask failed: ${SUB_NAME}"
fi
rm -f "$SUB_NAME_PATH" "$SUB_DESC_PATH" "$SUB_PAYLOAD_PATH"
```

Per-subtask failure is **non-fatal** — log and continue with the rest.

---

## Step 12 — Upload attachments

**Critical 1.0.1 finding:** the v3 file/attachment shapes documented online
return **200 OK + empty task.attachments** — they look successful but **do
not actually attach the file**. The only reliably-working pattern is:

1. **Upload** via **v1** `POST /projects/api/v1/pendingFiles.json` (multipart)
2. **Attach** via **v1** `PUT /projects/api/v1/tasks/{taskId}.json` with body
   `{"task": {"pendingFileAttachments": "<ref>"}}` — note that
   `pendingFileAttachments` is a **comma-separated string** of refs, not an
   array. Single ref → single string. Multiple → `"ref1,ref2,ref3"`.
3. **Verify** via v3 `GET /projects/api/v3/tasks/{taskId}.json?include=attachments`
   — successful attach reports `task.attachments: [{id: <fileId>, type: "files"}]`.
   If empty, the attach silently failed and the file lives orphaned in the
   project file library.

```bash
# Persist working endpoints on first success (or read from config on repeat runs).
UPLOAD_EP=$(jq -r '.desk_skill.attachment_upload_endpoint // "v1/pendingFiles"' "$CONFIG_FILE")
ATTACH_EP=$(jq -r '.desk_skill.attachment_attach_endpoint // "v1_put_task"'     "$CONFIG_FILE")

upload_and_attach() {
  local file_path="$1"
  local task_id="$2"
  local filename="$(basename "$file_path")"

  # 1) Multipart upload to v1 pendingFiles — returns {pendingFile:{ref:"tf_..."}}
  local pending_resp
  pending_resp=$(curl -sS -u "$PROJECTS_AUTH" -X POST \
    -F "file=@${file_path};filename=${filename}" \
    "${BASE_URL}/projects/api/v1/pendingFiles.json")

  local pending_ref
  pending_ref=$(python3 -c "
import json, sys
d = json.loads(sys.argv[1])
print(d.get('pendingFile', {}).get('ref') or '')" "$pending_resp")

  if [ -z "$pending_ref" ]; then
    ATTACHMENT_FAILURES+=("${filename}: pending-file upload failed: $pending_resp")
    return 1
  fi

  # 2) Attach via v1 PUT — pendingFileAttachments is a COMMA-SEPARATED STRING,
  #    not an array. The response confirms with assignedFileIds[].
  local attach_payload
  attach_payload=$(python3 -c "
import json, sys
ref = sys.argv[1]
print(json.dumps({'task': {'pendingFileAttachments': ref}}))" "$pending_ref")

  local http
  local attach_resp
  attach_resp=$(curl -sS -u "$PROJECTS_AUTH" -X PUT \
    -H "Content-Type: application/json" \
    -d "$attach_payload" \
    -o /tmp/tw_attach.json -w '%{http_code}' \
    "${BASE_URL}/projects/api/v1/tasks/${task_id}.json")

  if [ "$attach_resp" != "200" ] && [ "$attach_resp" != "204" ]; then
    ATTACHMENT_FAILURES+=("${filename}: v1 PUT attach failed (HTTP $attach_resp)")
    return 1
  fi

  # 3) Verify the attachment landed on the task (the silent-fail trap).
  local assigned
  assigned=$(python3 -c "
import json
d = json.load(open('/tmp/tw_attach.json'))
print(','.join(d.get('assignedFileIds') or []))")

  if [ -z "$assigned" ]; then
    ATTACHMENT_FAILURES+=("${filename}: PUT returned 200 but assignedFileIds is empty — possible silent-fail")
    return 1
  fi

  echo "✅ Uploaded ${filename} → task #${task_id} (file id ${assigned})"
  return 0
}

# Drive the loop over the Step 8 mapping.
#
# MAIN_TASK_FILES is a flat bash array of absolute file paths for the main task.
# Per-subtask files are NOT a nested array (bash has no nested arrays). Instead,
# each subtask index i has its own newline-delimited string variable
# SUBTASK_FILES_<i> (populated in Step 8 — see "Materialise the file lists").
# We read it back via indirect expansion.
for FILE_PATH in "${MAIN_TASK_FILES[@]}"; do
  [ -n "$FILE_PATH" ] || continue
  upload_and_attach "$FILE_PATH" "$MAIN_TASK_ID" || true
done
for i in "${!SUBTASK_IDS[@]}"; do
  list_var="SUBTASK_FILES_${i}"
  list="${!list_var:-}"
  [ -n "$list" ] || continue
  while IFS= read -r FILE_PATH; do
    [ -n "$FILE_PATH" ] || continue
    upload_and_attach "$FILE_PATH" "${SUBTASK_IDS[$i]}" || true
  done <<< "$list"
done

# Persist on first run when defaults landed
TMP=$(mktemp); jq '
  .desk_skill.attachment_upload_endpoint //= "v1/pendingFiles" |
  .desk_skill.attachment_attach_endpoint //= "v1_put_task"
' "$CONFIG_FILE" > "$TMP" && mv "$TMP" "$CONFIG_FILE" && chmod 600 "$CONFIG_FILE"
```

Per-file failure is non-fatal — log and continue.

### Step 12.1 — Things to NOT do (proven dead ends from 1.0.0 in the wild)

These endpoints / shapes were tried during 1.0.0 → 1.0.1 debugging and **all
silently fail** (HTTP 200/201 but no file ever attached). Do **not** revisit:

- `POST /projects/api/v3/files.json?projectId=X` → 405 Method Not Allowed
- `POST /projects/api/v3/pendingFiles.json` → 404 Not Found
- `POST /projects/api/v3/tasks/{id}/files.json` → 404
- `POST /projects/api/v1/tasks/{id}/files.json` with `{file: {pendingFileRef}}` → 200 OK + STATUS:OK + file not attached
- `PATCH /projects/api/v3/tasks/{id}.json` with `{task: {attachments: {pendingFileAttachments: ["ref"]}}}` → 200 + no attach
- `PATCH /projects/api/v3/tasks/{id}.json` with `{task: {pendingFileAttachments: ["ref"]}}` → 200 + no attach
- `PATCH /projects/api/v3/tasks/{id}.json` with `{task: {attachments: {pendingFiles: ["ref"]}}}` → 200 + no attach
- `POST /projects/api/v3/tasks/{id}/attach.json` → 404

The single working combination is documented in Step 12 above. Persisting the
endpoint names in config prevents future probes.

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

### Step 13.1 — Body format: HTML, not Markdown (1.0.1)

Teamwork Desk **does not render markdown** in note bodies — `**bold**`,
`### headings`, and `[link](url)` show as raw characters in the UI. The note
body must be HTML for it to display correctly. Older versions of this skill
produced markdown notes that the user had to manually re-format; 1.0.1 emits
HTML by default.

The body now uses these HTML elements (all standard, no JS / no custom CSS):

- `<h3>`, `<h4>` — section headings
- `<p>`, `<b>`, `<i>`, `<code>` — inline emphasis + monospace
- `<a href="…">` — links
- `<ul><li>…</li></ul>` — bullet lists
- `<pre>` — code blocks (for diffs)
- `<blockquote>` — quoted text (used in the DRAFT review banner)
- `<hr>` — section dividers

Template:

```html
<h3>📋 Projects task vytvorený z tohto Desk ticketu</h3>

<p><b>Task:</b> <a href="<MAIN_TASK_URL>"><MAIN_NAME></a> (#<MAIN_TASK_ID>)<br>
<b>Project:</b> <PROJECT_NAME> · <b>Tasklist:</b> <TASKLIST_NAME><br>
<b>Estimate:</b> <MAIN_EST> min (~<HOURS> h)<br>
<b>Assignee:</b> <name or "unassigned"></p>

<h4>📎 Attachments</h4>
<ul>
<li><code>filename.xlsx</code> (<size>) → uploaded to task #<MAIN_TASK_ID></li>
</ul>

<h4>✅ Akceptačné kritériá (<N>)</h4>
<ul>
<li><b>AC 1</b> — <first 80 chars></li>
<li><b>AC 2</b> — <first 80 chars></li>
…
</ul>

<h4>📋 Subtasks (<N>)</h4>
<ul>
<li>[BE] <name> (#<id>) — <est> min — <assignee or "unassigned"></li>
</ul>

<h4>🎯 Cieľ</h4>
<p><first 2–3 sentences extracted from the ## Cieľ section></p>

<p><i>— Vygenerované cez <code>/teamwork-tasks-from-desk</code> na <YYYY-MM-DD></i></p>
```

### Step 13.2 — Correct Desk messages POST schema (1.0.1)

The skill's 1.0.0 attempt at `POST /threads.json` with body
`{thread: {type: "note", body: ..., isInternal: true}}` **returns HTTP 403
"You Must Upgrade Your Account"** on modern Desk workspaces — the
`/threads.json` POST endpoint is gated behind a higher tier.

The working endpoint on modern v2 workspaces is `POST /messages.json` with a
**flat** payload — the body is the **top-level `message` string field**,
threadType is the top-level enum, and `isPrivate: true` is the modern name for
the "internal note" flag:

```bash
NOTE_BODY_PATH="$(mktemp)"
printf '%s' "$NOTE_HTML" > "$NOTE_BODY_PATH"

NOTE_PAYLOAD_PATH="$(mktemp)"
python3 - "$NOTE_BODY_PATH" "$NOTE_PAYLOAD_PATH" <<PY
import json, sys
body = open(sys.argv[1]).read()
json.dump({
    "message":     body,
    "threadType":  "note",
    "isPrivate":   True,
    "editMethod":  "html",
}, open(sys.argv[2], "w"), ensure_ascii=False)
PY

HTTP=$(desk_curl \
  -H "Content-Type: application/json" \
  -X POST -d @"$NOTE_PAYLOAD_PATH" \
  -o /tmp/tw_note_resp.json -w '%{http_code}' \
  "${DESK_API}/tickets/${TICKET_ID}/messages.json")

if [ "$HTTP" = "201" ]; then
  echo "✅ Internal note posted"
  TMP=$(mktemp); jq '.desk_skill.note_payload_shape = "v2_messages_top_level"' \
    "$CONFIG_FILE" > "$TMP" && mv "$TMP" "$CONFIG_FILE" && chmod 600 "$CONFIG_FILE"
else
  echo "❌ Internal note failed (HTTP $HTTP):"
  head -c 500 /tmp/tw_note_resp.json
fi
rm -f "$NOTE_BODY_PATH" "$NOTE_PAYLOAD_PATH"
```

**Non-negotiable invariants:**

- `threadType: "note"` — never `"message"` (which would be a customer-facing
  reply on some workspaces) or `"reply"`.
- `isPrivate: true` — the modern equivalent of the legacy `isInternal: true`.
  Without it the note may surface in the customer's email thread.
- `editMethod: "html"` — without this, the body is treated as plain text and
  HTML tags display literally.
- The plugin never POSTs to `/replies.json` (the only Desk endpoint that
  actually sends a customer-facing email reply) — it has no code path that
  constructs such a payload.

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

  Váš podnet sme zaevidovali a budeme ho riešiť v priebehu cca <estimate
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

The DRAFT is wrapped in a **clearly visible HTML review banner** so a support
agent who later opens the ticket cannot mistake it for an actual reply (same
HTML rationale as Step 13.1 — markdown is not rendered):

```html
<h3>✉️ DRAFT — návrh odpovede pre klienta (pred odoslaním skontrolovať)</h3>

<blockquote><b>Toto je INTERNÁ POZNÁMKA. Nebola odoslaná klientovi. Skopíruj, uprav podľa
potreby a odošli ako Reply ručne cez Desk UI.</b></blockquote>

<hr>

<the generated reply body, each paragraph wrapped in &lt;p&gt;…&lt;/p&gt;>

<hr>

<p><i>— Vygenerované cez <code>/teamwork-tasks-from-desk</code> na <YYYY-MM-DD></i></p>
```

Post identical to Step 13.2 — `threadType: "note"`, `isPrivate: true`,
`editMethod: "html"`, body in the top-level `message` field:

```bash
DRAFT_BODY_PATH="$(mktemp)"
printf '%s' "$DRAFT_HTML" > "$DRAFT_BODY_PATH"

DRAFT_PAYLOAD_PATH="$(mktemp)"
python3 - "$DRAFT_BODY_PATH" "$DRAFT_PAYLOAD_PATH" <<PY
import json, sys
body = open(sys.argv[1]).read()
json.dump({
    "message":     body,
    "threadType":  "note",
    "isPrivate":   True,
    "editMethod":  "html",
}, open(sys.argv[2], "w"), ensure_ascii=False)
PY

desk_curl \
  -H "Content-Type: application/json" \
  -X POST -d @"$DRAFT_PAYLOAD_PATH" \
  "${DESK_API}/tickets/${TICKET_ID}/messages.json" > /dev/null
rm -f "$DRAFT_BODY_PATH" "$DRAFT_PAYLOAD_PATH"
```

### Hard guarantees

- `threadType: "note"` and `isPrivate: true` are set on **every** Desk POST
  this skill makes. The plugin never constructs any payload of
  `threadType: "message"`, `threadType: "reply"`, or anything
  customer-facing. (Pre-1.0.1 wrote `type: "note"` + `isInternal: true` under
  a nested `thread` envelope — the modern v2 API rejected that schema with
  400 / 403; the schema above is the only one that produces a note instead
  of an attempted reply.)
- The plugin never calls `POST ${DESK_API}/tickets/<id>/replies.json` — that
  endpoint is the only way to send a customer-facing reply in the Desk API,
  and it is explicitly out of scope.
- The plugin does not collect the customer's email into any `to:`/`cc:`/`bcc:`
  payload field; it has nowhere to *send* a reply even by accident.
- The DRAFT body always begins with the review banner above so it is visually
  distinct from any genuine reply.

---

## Step 14 — Final report

### Step 14.1 — Compute run duration (1.0.1)

The plugin records the **time-to-deliverables** — from skill invocation
(`RUN_START_EPOCH` captured in Step 1.1) to the moment the deliverables are
all in place: the Projects task created **and** the Step 13 internal note
posted (and the DRAFT, when requested). This gives the user a true
end-to-end "how long did this hand-off take?" number so future runs can be
calibrated against it.

```bash
RUN_END_EPOCH=$(date -u +%s)
RUN_END_ISO=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
RUN_DURATION_SECONDS=$(( RUN_END_EPOCH - RUN_START_EPOCH ))
RUN_DURATION_MINUTES=$(( RUN_DURATION_SECONDS / 60 ))
RUN_DURATION_REMAINDER=$(( RUN_DURATION_SECONDS % 60 ))

# Human format like "12 min 34 s" (config: duration_format = "human")
# Compact format like "12:34"        (config: duration_format = "compact")
DURATION_FORMAT=$(jq -r '.desk_skill.timing.duration_format // "human"' "$CONFIG_FILE")
case "$DURATION_FORMAT" in
  compact) RUN_DURATION_HUMAN=$(printf '%d:%02d' "$RUN_DURATION_MINUTES" "$RUN_DURATION_REMAINDER") ;;
  *)       RUN_DURATION_HUMAN="${RUN_DURATION_MINUTES} min ${RUN_DURATION_REMAINDER} s" ;;
esac
```

The duration block is included in the final report **only when**
`desk_skill.timing.report_run_duration == true` (default).

### Step 14.2 — Persist the run for later analysis (optional)

When `desk_skill.timing.report_run_duration == true`, also append a one-line
entry to a per-user history log so the user can compare runs over time:

```bash
HIST_PATH="${HOME}/.claude/plugins/data/teamwork-task-wamesk/runs.tsv"
mkdir -p "$(dirname "$HIST_PATH")"
[ -f "$HIST_PATH" ] || printf 'started_at\tended_at\tduration_s\tticket_id\ttask_id\tsubject\n' > "$HIST_PATH"
printf '%s\t%s\t%s\t%s\t%s\t%s\n' \
  "$RUN_START_ISO" "$RUN_END_ISO" "$RUN_DURATION_SECONDS" \
  "$TICKET_ID" "${MAIN_TASK_ID:-}" "$(printf %s "$SUBJECT" | tr '\t\n' '  ')" \
  >> "$HIST_PATH"
```

### Step 14.3 — Render the markdown summary into the terminal

```markdown
## 📨 Desk → Projects task created

**Desk ticket:** <SUBJECT> (#<TICKET_ID>) — <DESK_BASE>/tickets/<TICKET_ID>
**Projects task:** <MAIN_NAME> (#<MAIN_TASK_ID>) — <MAIN_TASK_URL>
**Estimate:** <MAIN_EST> min · **Assignee:** <name or "unassigned">

### ⏱ Run duration
- Started: <RUN_START_ISO>
- Ended:   <RUN_END_ISO>
- Duration: **<RUN_DURATION_HUMAN>** (from skill invocation to task created + Desk note posted)
- History log: `~/.claude/plugins/data/teamwork-task-wamesk/runs.tsv`

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
- Internal Desk note (link + summary): ✅ posted / ⏭ skipped
- Customer reply DRAFT (internal note): ✅ posted for review / ⏭ skipped

### Unresolved
- [OTVORENÉ] <question 1>
- [OTVORENÉ] <question 2>
```

### Step 14.4 — Cleanup

```bash
# Default behaviour (desk_skill.attachments.cleanup == "after_run"):
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
| Desk 401 on both Bearer + Basic | Re-prompt the Desk token (Step 2 first-run flow) |
| Desk 401 on Basic only | Try Bearer (1.0.1 auto-probe); persist `auth_scheme=bearer` |
| Desk 403 on `/files/{id}.json` | Use `/files/{id}/download.json` 303-follow trick (1.0.1) |
| Desk 403 on `/attachments.json` | Build the file_id set from `messages[].files[]` instead |
| Desk 403 on `/threads.json` POST | Use `/messages.json` POST with top-level fields (1.0.1) |
| Desk 403/404 on other endpoints | Stop with the failing URL printed |
| Desk thread/messages endpoint 404 | Probe v1; record the working one in config |
| Projects URL unparseable | AskUserQuestion for a corrected URL |
| Projects 401 | Re-prompt the Projects token (shared with siblings) |
| Projects 403 on tasklist | Stop with a clear "no access" message |
| Email resolves to 0 people | AskUserQuestion (retry / unassigned / cancel) |
| Email resolves to ≥2 people | AskUserQuestion to pick the right person |
| Main task POST returns no id | Idempotency probe (1.0.1) — search by name + recent timestamp before retrying |
| Subtask POST non-2xx | Log `❌`, continue with the rest |
| Attachment download non-2xx | Log `⏭`, continue |
| Attachment attach returns 200 but `assignedFileIds` empty | Treat as failure — log `❌`; retry once with v1 PUT (1.0.1 default) |
| Desk note POST non-2xx | Log `❌`, continue — the task is already created |
| `jq` parse error when displaying Projects response | Suppress + retry parse via Python; never panic-retry the POST |

---

## Persistent learning

On each successful run, the skill writes back into the shared config so
subsequent runs skip the probes:

- `desk_skill.api_version` — the working Desk API version (set in Step 3)
- `desk_skill.auth_scheme` — `"bearer"` or `"basic"` (set in Step 2.6, 1.0.1)
- `desk_skill.note_payload_shape` — `"v2_messages_top_level"` (set in Step 13.2, 1.0.1)
- `desk_skill.note_body_format` — `"html"` (default in 1.0.1)
- `desk_skill.task_create_estimate_field` — `"estimatedMinutes"` (default in 1.0.1)
- `desk_skill.attachment_upload_endpoint` — `"v1/pendingFiles"` (set in Step 12, 1.0.1)
- `desk_skill.attachment_attach_endpoint` — `"v1_put_task"` (set in Step 12, 1.0.1)
- `desk_skill.file_download_endpoint` — `"v2_download_json"` (set in Step 3.3, 1.0.1)
- `desk_skill.link_back.notify_desk_default` — if the user picked "Yes and
  don't ask next time" in Step 13
- `desk_skill.timing.report_run_duration` — `true` by default (Step 14, 1.0.1)
- `desk_skill.timing.duration_format` — `"human"` or `"compact"` (Step 14, 1.0.1)

A historical TSV log of every run lives at
`~/.claude/plugins/data/teamwork-task-wamesk/runs.tsv` so the user can
compare `duration_s` across runs as the skill matures and the probes become
no-ops.

All writes are atomic (`jq` + `mv` + `chmod 600`).
