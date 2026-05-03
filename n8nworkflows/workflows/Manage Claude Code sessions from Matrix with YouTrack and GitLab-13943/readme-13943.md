Manage Claude Code sessions from Matrix with YouTrack and GitLab

https://n8nworkflows.xyz/workflows/manage-claude-code-sessions-from-matrix-with-youtrack-and-gitlab-13943


# Manage Claude Code sessions from Matrix with YouTrack and GitLab

# 1. Workflow Overview

This workflow implements a chat-ops gateway that lets a team interact with an AI coding assistant from a Matrix room while keeping visibility into YouTrack issues, GitLab project status, and Claude session state stored in SQLite on a remote server.

Its main use case is operational control of AI-assisted coding sessions from chat. Team members can send normal messages or issue-specific prefixed messages to Claude Code, inspect or manage sessions, query issue data from YouTrack, inspect GitLab-related status, and check host health. The workflow relies on Matrix polling, SSH-based remote command execution, a SQLite session store, and Matrix message posting.

## 1.1 Configuration

A single Set node defines all environment-style values used throughout the workflow: Matrix homeserver and room, SSH target host, Claude binary path, SQLite DB path, issue prefix, and related paths.

## 1.2 Matrix Polling and Message Extraction

A Schedule Trigger runs every 30 seconds, retrieves the previous Matrix sync token from workflow static data, calls Matrix `/sync`, extracts new chat messages from the configured room, ignores bot messages, and updates static state for the next poll.

## 1.3 Command Detection and Routing

Incoming Matrix content is normalized into one of several command families:
- plain message
- issue-prefixed message like `PROJ-4: fix this`
- `!session`
- `!issue`
- `!pipeline`
- `!system`
- `!help`
- unknown/empty input

A Switch node dispatches each request to the corresponding handler.

## 1.4 Claude Message Handling

For plain messages or issue-prefixed messages, the workflow checks whether a gateway lock exists, optionally notifies Matrix that Claude is busy, otherwise acquires the lock, reads the current session from SQLite, resumes that Claude session over SSH, parses Claude output, posts the response back to Matrix, and finally releases the lock.

## 1.5 Command Handlers

Dedicated handlers process:
- session inspection and session lifecycle commands
- YouTrack issue lookups
- GitLab-related project/pipeline overview
- system status
- help
- fallback for unknown commands

Most of these handlers execute SSH commands remotely and then format the response into Matrix-compatible message bodies.

## 1.6 Session Completion and Cleanup

After command responses are posted, the workflow checks whether the `!session done` path returned a session payload. If so, it runs cleanup over SSH: asks Claude for a brief summary, archives the session in SQLite, clears queue and lock artifacts, creates a cooldown marker, and posts a final notice to Matrix.

---

# 2. Block-by-Block Analysis

## Block 1 — Configuration

### Overview
This block centralizes user-editable workflow values so the rest of the workflow can reference them consistently. It acts as the configuration source for Matrix, SSH paths, Claude runtime paths, YouTrack and GitLab base URLs, and SQLite storage.

### Nodes Involved
- Gateway Config

### Node Details

#### Gateway Config
- **Type and technical role:** `n8n-nodes-base.set`; defines constant configuration fields as JSON properties.
- **Configuration choices:**
  - Sets:
    - `SSH_HOST`
    - `MATRIX_HOMESERVER`
    - `MATRIX_ROOM_ID`
    - `MATRIX_BOT_USER`
    - `YOUTRACK_URL`
    - `GITLAB_URL`
    - `CLAUDE_PROJECT_PATH`
    - `CLAUDE_BINARY`
    - `DB_PATH`
    - `CONTEXT_DIR`
    - `ISSUE_PREFIX`
    - `COOLDOWN_TTL`
  - These are placeholders and must be replaced with real values.
- **Key expressions or variables used:** None internally, but many later nodes reference it using `$('Gateway Config').first().json`.
- **Input and output connections:** No incoming connection; referenced by expression from other nodes rather than flowing in the main chain.
- **Version-specific requirements:** Set node version `3.4`.
- **Edge cases or potential failure types:**
  - Invalid paths break SSH commands later.
  - Wrong Matrix room/server values cause HTTP request failures.
  - Wrong YouTrack/GitLab URLs produce empty or error SSH command outputs.
  - `COOLDOWN_TTL` is defined but not actively used in downstream logic.
  - `SSH_HOST` is stored but not explicitly interpolated in SSH node commands; actual host comes from SSH credentials.
- **Sub-workflow reference:** None.

---

## Block 2 — Matrix Polling and New Message Detection

### Overview
This block runs every 30 seconds, loads the last Matrix sync position, polls the configured Matrix room, extracts new human-authored messages, and stores updated sync state in workflow static storage.

### Nodes Involved
- Poll Every 30s
- Get Sync Token
- Poll Matrix Sync
- Extract Messages
- Has Messages?

### Node Details

#### Poll Every 30s
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`; entry point for periodic polling.
- **Configuration choices:**
  - Interval rule set to seconds.
  - Because only `field: "seconds"` appears, this is effectively an every-30-seconds poll as named, though the exact interval should be verified in the UI after import because the JSON does not show a numeric `30`.
- **Key expressions or variables used:** None.
- **Input and output connections:** Entry node → `Get Sync Token`.
- **Version-specific requirements:** Schedule Trigger version `1.2`.
- **Edge cases or potential failure types:**
  - If import changes interval interpretation, polling frequency may differ from the label.
  - Very frequent polling can hit Matrix rate limits.
- **Sub-workflow reference:** None.

#### Get Sync Token
- **Type and technical role:** `n8n-nodes-base.code`; retrieves persisted Matrix sync position from workflow global static data.
- **Configuration choices:**
  - Reads `staticData.matrixSinceToken || ''`.
  - Returns one item containing `sinceToken` merged with all config values from `Gateway Config`.
- **Key expressions or variables used:**
  - `$getWorkflowStaticData('global')`
  - `$('Gateway Config').first().json`
- **Input and output connections:** Input from `Poll Every 30s`; output to `Poll Matrix Sync`.
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or potential failure types:**
  - If static data is unavailable or cleared, initial sync runs without `since`, potentially returning recent room history.
  - If `Gateway Config` is misconfigured, downstream errors occur immediately.
- **Sub-workflow reference:** None.

#### Poll Matrix Sync
- **Type and technical role:** `n8n-nodes-base.httpRequest`; calls Matrix client `/sync`.
- **Configuration choices:**
  - GET request to:
    - `{{ MATRIX_HOMESERVER }}/_matrix/client/v3/sync`
    - `timeout=0`
    - room filter limited to configured room
    - timeline limit `10`
    - optional `since` token
  - Request timeout set to 15 seconds.
  - Uses generic credential type `httpHeaderAuth`.
- **Key expressions or variables used:**
  - `{{ $json.MATRIX_HOMESERVER }}`
  - `{{ $json.MATRIX_ROOM_ID }}`
  - `{{ $json.sinceToken ? '&since=' + $json.sinceToken : '' }}`
- **Input and output connections:** Input from `Get Sync Token`; output to `Extract Messages`.
- **Version-specific requirements:** HTTP Request version `4.2`.
- **Edge cases or potential failure types:**
  - Invalid Matrix token returns 401/403.
  - Bad homeserver URL or network issues cause timeout/DNS failures.
  - Inline filter JSON inside URL may be sensitive to encoding behavior.
  - If more than 10 new events arrive between polls, some messages may be missed unless Matrix sync state compensates.
- **Sub-workflow reference:** None.

#### Extract Messages
- **Type and technical role:** `n8n-nodes-base.code`; parses Matrix sync response and filters message events.
- **Configuration choices:**
  - Saves `next_batch` to `staticData.matrixSinceToken`.
  - Reads the configured room or falls back to the first joined room found.
  - Extracts `m.room.message` events only.
  - Ignores messages sent by `MATRIX_BOT_USER`.
  - Filters out events older than `staticData.lastProcessedTimestamp`.
  - Returns only the first extracted message, merged with config.
- **Key expressions or variables used:**
  - `$('Gateway Config').first().json`
  - `$getWorkflowStaticData('global')`
  - `syncData.rooms?.join`
  - `origin_server_ts`
- **Input and output connections:** Input from `Poll Matrix Sync`; output to `Has Messages?`.
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or potential failure types:**
  - Only the first new message in a batch is forwarded; additional messages in the same poll are dropped.
  - Timestamp-based deduplication can skip messages with equal timestamps.
  - If Matrix history arrives out of order, `lastProcessedTimestamp` may suppress valid messages.
  - Fallback to first joined room could process the wrong room if `MATRIX_ROOM_ID` is incorrect.
- **Sub-workflow reference:** None.

#### Has Messages?
- **Type and technical role:** `n8n-nodes-base.if`; prevents empty executions from going forward.
- **Configuration choices:**
  - Checks whether `messageText` is not empty.
- **Key expressions or variables used:**
  - `={{ $json.messageText }}`
- **Input and output connections:** Input from `Extract Messages`; true branch to `Detect Command`; false branch unused.
- **Version-specific requirements:** If node version `2`.
- **Edge cases or potential failure types:**
  - Since `Extract Messages` already returns empty array when nothing is found, this node mainly acts as a safety guard.
- **Sub-workflow reference:** None.

---

## Block 3 — Command Parsing and Routing

### Overview
This block interprets chat input and maps it into normalized command metadata. It also supports direct issue-prefixed routing such as `PROJ-4: do something`, legacy aliases like `!done`, and generation of notice payloads for empty input.

### Nodes Involved
- Detect Command
- Command Router

### Node Details

#### Detect Command
- **Type and technical role:** `n8n-nodes-base.code`; parses message syntax and emits command metadata.
- **Configuration choices:**
  - Trims and lowercases input.
  - Base64-encodes original text for safe SSH transport.
  - Detects format `PREFIX-NUMBER: message`.
  - For non-command text:
    - empty → `command: empty` with prebuilt `matrixBody`
    - non-empty → `command: message`
  - For `!commands`:
    - extracts `command`, `sub`, and `args`
    - applies legacy aliases:
      - `!done` → `!session done`
      - `!cancel` → `!session cancel`
      - `!status` → `!session current`
- **Key expressions or variables used:**
  - `$input.first().json.messageText`
  - `$('Gateway Config').first().json`
  - `Buffer.from(...).toString('base64')`
  - regex `^([A-Z]+-\d+):\s*([\s\S]*)$`
- **Input and output connections:** Input from `Has Messages?`; output to `Command Router`.
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or potential failure types:**
  - Prefix detection is hardcoded to uppercase issue IDs and does not use `ISSUE_PREFIX`.
  - Messages like `proj-4: text` are not treated as prefixed issues.
  - Empty prefixed input returns a notice but routes as unknown/fallback later.
  - Base64 encoding prevents shell quoting issues, but very large Matrix messages may still stress CLI limits.
- **Sub-workflow reference:** None.

#### Command Router
- **Type and technical role:** `n8n-nodes-base.switch`; routes normalized commands to dedicated handlers.
- **Configuration choices:**
  - Named outputs:
    - `message`
    - `session`
    - `help`
    - `issue`
    - `pipeline`
    - `system`
  - Fallback output `extra`
- **Key expressions or variables used:**
  - `={{ $json.command }}`
- **Input and output connections:**
  - Input from `Detect Command`
  - Outputs to:
    - `Check Lock`
    - `Handle Session Command`
    - `Handle Help`
    - `Handle Issue Command`
    - `Handle Pipeline Command`
    - `Handle System Command`
    - `Handle Unknown Command`
- **Version-specific requirements:** Switch version `3`.
- **Edge cases or potential failure types:**
  - Commands like `empty` intentionally fall to fallback and are handled by `Handle Unknown Command`, which preserves provided `matrixBody`.
  - Unsupported subcommands are not filtered here; they are handled inside command-specific SSH/code nodes.
- **Sub-workflow reference:** None.

---

## Block 4 — Claude Message Handling

### Overview
This block processes ordinary user messages for Claude Code. It serializes access using a filesystem lock, resumes the active Claude session stored in SQLite, posts Claude’s reply to Matrix, and releases the lock.

### Nodes Involved
- Check Lock
- Is Locked?
- Post Busy Notice
- Read Session & Acquire Lock
- Resume Claude Session
- Parse Claude Response
- Post Response to Matrix
- Release Lock

### Node Details

#### Check Lock
- **Type and technical role:** `n8n-nodes-base.ssh`; remotely checks whether a lock file exists and is fresh.
- **Configuration choices:**
  - Uses `CONTEXT_DIR/gateway.lock`.
  - If lock file exists and is newer than 600 seconds, outputs `LOCKED`.
  - Otherwise removes stale lock and outputs `FREE`.
- **Key expressions or variables used:**
  - `{{ $json.CONTEXT_DIR }}`
  - shell `stat -c %Y`
- **Input and output connections:** Input from `Command Router` message branch; output to `Is Locked?`.
- **Version-specific requirements:** SSH node version `1`.
- **Edge cases or potential failure types:**
  - Requires GNU `stat -c`; may fail on BSD/macOS remotes.
  - SSH authentication or connectivity failures stop the message path.
  - Stale lock cleanup is time-based only; an active Claude process running longer than 10 minutes may be treated as stale.
- **Sub-workflow reference:** None.

#### Is Locked?
- **Type and technical role:** `n8n-nodes-base.if`; branches based on lock state.
- **Configuration choices:**
  - Tests whether trimmed stdout equals `LOCKED`.
- **Key expressions or variables used:**
  - `={{ $json.stdout.trim() }}`
- **Input and output connections:**
  - Input from `Check Lock`
  - true branch → `Post Busy Notice`
  - false branch → `Read Session & Acquire Lock`
- **Version-specific requirements:** If version `2`.
- **Edge cases or potential failure types:**
  - Unexpected SSH output format can bypass or mis-trigger the lock branch.
- **Sub-workflow reference:** None.

#### Post Busy Notice
- **Type and technical role:** `n8n-nodes-base.httpRequest`; posts a Matrix notice telling users Claude is busy.
- **Configuration choices:**
  - PUT request to Matrix send endpoint with event ID `busy-{{ Date.now() }}`
  - Sends JSON notice: “Claude is busy processing a previous message. Your message has been queued.”
- **Key expressions or variables used:**
  - `$('Gateway Config').first().json.MATRIX_HOMESERVER`
  - `$('Gateway Config').first().json.MATRIX_ROOM_ID`
  - `Date.now()`
- **Input and output connections:** Input from locked branch of `Is Locked?`; no downstream connection.
- **Version-specific requirements:** HTTP Request version `4.2`.
- **Edge cases or potential failure types:**
  - Message says “queued” but no actual queue insertion occurs in this workflow branch.
  - Matrix auth/network errors prevent notification.
- **Sub-workflow reference:** None.

#### Read Session & Acquire Lock
- **Type and technical role:** `n8n-nodes-base.ssh`; atomically writes the lock and fetches current session metadata from SQLite.
- **Configuration choices:**
  - Writes `locked` into `gateway.lock`.
  - Queries current session:
    - `sessionId`
    - `issueId`
    - `issueTitle`
  - `continueOnFail` enabled.
- **Key expressions or variables used:**
  - `$('Gateway Config').first().json.DB_PATH`
  - `$('Gateway Config').first().json.CONTEXT_DIR`
- **Input and output connections:** Input from unlocked branch of `Is Locked?`; output to `Resume Claude Session`.
- **Version-specific requirements:** SSH version `1`.
- **Edge cases or potential failure types:**
  - If SQLite CLI is missing, stdout may be empty and stderr populated.
  - Lock is acquired even if DB query fails.
  - Because `continueOnFail` is true, downstream nodes must tolerate malformed output.
- **Sub-workflow reference:** None.

#### Resume Claude Session
- **Type and technical role:** `n8n-nodes-base.ssh`; resumes an active Claude Code session and sends the user message.
- **Configuration choices:**
  - Reads:
    - `CLAUDE_BINARY`
    - `CLAUDE_PROJECT_PATH`
    - session JSON from previous node
    - base64-encoded message from `Detect Command`
  - Returns `NO_SESSION` if no current session exists or no session ID can be parsed.
  - Runs:
    - `timeout 300`
    - `claude -r <sessionId> -p "<message>" --output-format json --dangerously-skip-permissions`
  - `continueOnFail` enabled.
- **Key expressions or variables used:**
  - `$("Gateway Config").first().json.CLAUDE_BINARY`
  - `$("Gateway Config").first().json.CLAUDE_PROJECT_PATH`
  - `$("Read Session & Acquire Lock").first().json.stdout`
  - `$("Detect Command").first().json.base64Message`
  - Python JSON parsing on remote host
- **Input and output connections:** Input from `Read Session & Acquire Lock`; output to `Parse Claude Response`.
- **Version-specific requirements:** SSH version `1`.
- **Edge cases or potential failure types:**
  - Requires `timeout`, `python3`, and `base64` on remote host.
  - Claude binary path must be correct and executable.
  - Long-running tasks are cut off at 300 seconds.
  - `--dangerously-skip-permissions` may be incompatible with some Claude installations or security postures.
  - If Claude outputs mixed logs and JSON, parsing may fail downstream.
- **Sub-workflow reference:** None.

#### Parse Claude Response
- **Type and technical role:** `n8n-nodes-base.code`; transforms Claude CLI output into Matrix message JSON.
- **Configuration choices:**
  - Uses stdout, falling back to stderr.
  - If output is `NO_SESSION`, sends notice prompting user to start a session.
  - Otherwise tries to parse JSON and read `parsed.result`.
  - Falls back to raw output truncated to 4000 chars.
  - Emits Matrix `m.text`.
- **Key expressions or variables used:**
  - `$input.first().json.stdout`
  - `$input.first().json.stderr`
  - `$('Gateway Config').first().json`
- **Input and output connections:** Input from `Resume Claude Session`; output to `Post Response to Matrix`.
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or potential failure types:**
  - Assumes Claude JSON contains `result`.
  - Raw truncation may cut structured output mid-line.
  - Errors from SSH transport may be posted verbatim to Matrix.
- **Sub-workflow reference:** None.

#### Post Response to Matrix
- **Type and technical role:** `n8n-nodes-base.httpRequest`; sends Claude output back to Matrix.
- **Configuration choices:**
  - PUT request to Matrix send endpoint with event ID `resp-{{ Date.now() }}`
  - JSON body comes from `matrixBody`
- **Key expressions or variables used:**
  - `={{ $json.MATRIX_HOMESERVER }}`
  - `={{ $json.MATRIX_ROOM_ID }}`
  - `={{ $json.matrixBody }}`
- **Input and output connections:** Input from `Parse Claude Response`; output to `Release Lock`.
- **Version-specific requirements:** HTTP Request version `4.2`.
- **Edge cases or potential failure types:**
  - Matrix API failures leave the lock to be released only if node execution continues successfully; since `continueOnFail` is not set, failed post may stop before unlock.
- **Sub-workflow reference:** None.

#### Release Lock
- **Type and technical role:** `n8n-nodes-base.ssh`; removes the gateway lock file.
- **Configuration choices:**
  - Executes `rm -f "<CONTEXT_DIR>/gateway.lock"`
  - `continueOnFail` enabled
- **Key expressions or variables used:**
  - `$('Gateway Config').first().json.CONTEXT_DIR`
- **Input and output connections:** Input from `Post Response to Matrix`; no downstream connection.
- **Version-specific requirements:** SSH version `1`.
- **Edge cases or potential failure types:**
  - If earlier nodes fail before reaching this node, lock may remain.
  - File permission errors can leave the gateway blocked.
- **Sub-workflow reference:** None.

---

## Block 5 — Command Handlers

### Overview
This block handles all non-message command families. Each command either runs remote shell logic through SSH or formats a static response, then posts a Matrix notice. Some commands are partially implemented versus the sticky note claims.

### Nodes Involved
- Handle Session Command
- Format Session Response
- Handle Issue Command
- Format Issue Response
- Handle Pipeline Command
- Format Pipeline Response
- Handle Help
- Handle System Command
- Format System Response
- Handle Unknown Command
- Post Command Response
- Is Session Done?

### Node Details

#### Handle Session Command
- **Type and technical role:** `n8n-nodes-base.ssh`; manages session inspection and lifecycle operations in SQLite and the filesystem.
- **Configuration choices:**
  - Reads subcommand from `Detect Command` with default `current`
  - Reads first argument as optional issue ID and uppercases it
  - Supports:
    - `current`: current session JSON prefixed with `SESSION:`
    - `list`: JSON array of sessions
    - `done`: returns current or targeted session JSON
    - `cancel`: kills Claude processes, clears lock, deletes session and queue rows
    - `pause`: sets `paused=1`
    - `resume`: sets `paused=0`
    - default: `UNKNOWN_SUB:<sub>`
  - `continueOnFail` enabled.
- **Key expressions or variables used:**
  - `$("Detect Command").first().json.sub`
  - `$("Detect Command").first().json.args[0]`
  - `$("Gateway Config").first().json.DB_PATH`
  - `$("Gateway Config").first().json.CONTEXT_DIR`
- **Input and output connections:** Input from `Command Router`; output to `Format Session Response`.
- **Version-specific requirements:** SSH version `1`.
- **Edge cases or potential failure types:**
  - Requires SQLite CLI on remote host.
  - `cancel` uses `pkill -f claude`, which may terminate unrelated Claude processes.
  - `done` only returns session JSON; actual archival happens later in another node.
  - No validation that targeted issue ID matches `ISSUE_PREFIX`.
  - Assumes `sessions` and `queue` tables exist.
- **Sub-workflow reference:** None.

#### Format Session Response
- **Type and technical role:** `n8n-nodes-base.code`; converts session command output into user-readable Matrix notices.
- **Configuration choices:**
  - Uses `sub` from `Detect Command`.
  - Formats:
    - `current`
    - `list`
    - `done`
    - `cancel`
    - `pause`
    - `resume`
  - Emits `m.notice`.
- **Key expressions or variables used:**
  - `$('Detect Command').first().json.sub`
  - `$input.first().json.stdout`
  - `$('Gateway Config').first().json`
- **Input and output connections:** Input from `Handle Session Command`; output to `Post Command Response`.
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or potential failure types:**
  - For `done`, it only announces “Ending session...” and does not display the eventual Claude summary.
  - Parsing errors are surfaced as generic error text or fallback strings.
  - `current` formatter ignores `issue_title` and `started_at` even if available.
- **Sub-workflow reference:** None.

#### Handle Issue Command
- **Type and technical role:** `n8n-nodes-base.ssh`; queries YouTrack over HTTP from the remote server.
- **Configuration choices:**
  - Reads `sub` default `status`
  - Uses remote env var `YT_TOKEN`
  - Supports:
    - `status`: fetch issues in state `In Progress`
    - `info <id>`: fetch issue details
    - default: help string
  - `continueOnFail` enabled.
- **Key expressions or variables used:**
  - `$("Detect Command").first().json.sub`
  - `$("Detect Command").first().json.args[0]`
  - `$("Gateway Config").first().json.YOUTRACK_URL`
  - shell env `$YT_TOKEN`
- **Input and output connections:** Input from `Command Router`; output to `Format Issue Response`.
- **Version-specific requirements:** SSH version `1`.
- **Edge cases or potential failure types:**
  - Sticky note lists `start`, `verify`, `done`, `comment`, but these are not implemented.
  - If `YT_TOKEN` is not set in the SSH environment, YouTrack returns auth errors.
  - SSH non-login shells may not load `.bashrc`, so env vars may be unavailable.
  - Network ACLs from remote host to YouTrack may block requests.
- **Sub-workflow reference:** None.

#### Format Issue Response
- **Type and technical role:** `n8n-nodes-base.code`; formats YouTrack API responses.
- **Configuration choices:**
  - If output is a JSON array, lists `idReadable: summary`
  - If output is a single issue, prints ID, summary, and first 500 chars of description
  - Otherwise returns raw output or fallback text
  - Emits `m.notice`
- **Key expressions or variables used:**
  - `$input.first().json.stdout`
  - `$('Gateway Config').first().json`
- **Input and output connections:** Input from `Handle Issue Command`; output to `Post Command Response`.
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or potential failure types:**
  - Does not inspect HTTP status codes because the query happens inside SSH.
  - Large descriptions are truncated.
- **Sub-workflow reference:** None.

#### Handle Pipeline Command
- **Type and technical role:** `n8n-nodes-base.ssh`; queries GitLab from the remote server.
- **Configuration choices:**
  - Reads `sub` default `status`
  - Uses remote env var `GL_TOKEN`
  - Supports only:
    - `status`: list up to 10 accessible projects with default branch
    - default: help string
  - `continueOnFail` enabled.
- **Key expressions or variables used:**
  - `$("Detect Command").first().json.sub`
  - `$("Gateway Config").first().json.GITLAB_URL`
  - shell env `$GL_TOKEN`
- **Input and output connections:** Input from `Command Router`; output to `Format Pipeline Response`.
- **Version-specific requirements:** SSH version `1`.
- **Edge cases or potential failure types:**
  - Sticky note lists `logs` and `retry`, but they are not implemented.
  - This is not true pipeline status; it is closer to project membership overview.
  - Requires Python 3 on remote host for formatting.
- **Sub-workflow reference:** None.

#### Format Pipeline Response
- **Type and technical role:** `n8n-nodes-base.code`; wraps pipeline/project output into Matrix notice JSON.
- **Configuration choices:**
  - Uses stdout directly, fallback notice if empty.
- **Key expressions or variables used:**
  - `$input.first().json.stdout`
  - `$('Gateway Config').first().json`
- **Input and output connections:** Input from `Handle Pipeline Command`; output to `Post Command Response`.
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or potential failure types:**
  - No semantic parsing of GitLab response; raw command errors may be posted as-is.
- **Sub-workflow reference:** None.

#### Handle Help
- **Type and technical role:** `n8n-nodes-base.code`; returns a static command reference.
- **Configuration choices:**
  - Builds a multiline help notice describing supported commands.
- **Key expressions or variables used:**
  - `$('Gateway Config').first().json`
- **Input and output connections:** Input from `Command Router`; output to `Post Command Response`.
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or potential failure types:**
  - Help text reflects current implementation better than sticky notes, but still mentions `Prefix routing: PROJ-4: your message` with hardcoded example.
- **Sub-workflow reference:** None.

#### Handle System Command
- **Type and technical role:** `n8n-nodes-base.ssh`; gathers host load, memory, and Claude process count.
- **Configuration choices:**
  - Runs `uptime`, `free -h`, and `pgrep -c -f claude`
  - `continueOnFail` enabled
- **Key expressions or variables used:** Shell commands only.
- **Input and output connections:** Input from `Command Router`; output to `Format System Response`.
- **Version-specific requirements:** SSH version `1`.
- **Edge cases or potential failure types:**
  - Supports only effective `!system status`, even though no explicit subcommand switch exists.
  - Depends on Linux utilities `uptime`, `free`, and `pgrep`.
- **Sub-workflow reference:** None.

#### Format System Response
- **Type and technical role:** `n8n-nodes-base.code`; converts raw system status text into a Matrix notice.
- **Configuration choices:**
  - Uses stdout directly, fallback `No system data.`
- **Key expressions or variables used:**
  - `$input.first().json.stdout`
  - `$('Gateway Config').first().json`
- **Input and output connections:** Input from `Handle System Command`; output to `Post Command Response`.
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or potential failure types:** Minimal formatting means shell errors may be posted directly.
- **Sub-workflow reference:** None.

#### Handle Unknown Command
- **Type and technical role:** `n8n-nodes-base.code`; fallback responder for unknown or prebuilt notice cases.
- **Configuration choices:**
  - If input already contains `matrixBody`, it passes that through.
  - Otherwise posts `Unknown command: !<command>` and suggests `!help`.
- **Key expressions or variables used:**
  - `$json.matrixBody`
  - `$json.command`
  - `$('Gateway Config').first().json`
- **Input and output connections:** Input from `Command Router` fallback; output to `Post Command Response`.
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or potential failure types:**
  - This is how `empty` command notices are delivered.
  - Unknown non-bang text does not reach here because plain messages route to Claude branch.
- **Sub-workflow reference:** None.

#### Post Command Response
- **Type and technical role:** `n8n-nodes-base.httpRequest`; posts command results to Matrix.
- **Configuration choices:**
  - PUT request to Matrix send endpoint with event ID `cmd-{{ Date.now() }}`
  - Sends JSON body from `matrixBody`
  - `continueOnFail` enabled
- **Key expressions or variables used:**
  - `={{ $json.MATRIX_HOMESERVER }}`
  - `={{ $json.MATRIX_ROOM_ID }}`
  - `={{ $json.matrixBody }}`
- **Input and output connections:** Inputs from all formatter/help/unknown nodes; output to `Is Session Done?`.
- **Version-specific requirements:** HTTP Request version `4.2`.
- **Edge cases or potential failure types:**
  - Because `continueOnFail` is enabled, session cleanup can proceed even if Matrix post fails.
- **Sub-workflow reference:** None.

#### Is Session Done?
- **Type and technical role:** `n8n-nodes-base.if`; detects whether the session command output corresponds to `!session done`.
- **Configuration choices:**
  - Checks whether `$('Handle Session Command').first().json.stdout` contains `session_id`
  - True branch goes to cleanup
- **Key expressions or variables used:**
  - `={{ $('Handle Session Command').first().json.stdout || '' }}`
- **Input and output connections:** Input from `Post Command Response`; true branch to `Clean Up & End Session`; false branch unused.
- **Version-specific requirements:** If version `2`.
- **Edge cases or potential failure types:**
  - This can be influenced by stale `Handle Session Command` data if multiple paths are executed in unexpected ways.
  - It relies on string containment rather than confirming current command is `session done`.
  - A session JSON from another subcommand could accidentally trigger cleanup if it included `session_id`.
- **Sub-workflow reference:** None.

---

## Block 6 — Session Finalization and Archival

### Overview
This block runs after a session completion command has been recognized. It asks Claude for a brief summary, archives the session into SQLite logs, removes current state artifacts, creates a cooldown marker, and notifies Matrix.

### Nodes Involved
- Clean Up & End Session
- Post Session Ended

### Node Details

#### Clean Up & End Session
- **Type and technical role:** `n8n-nodes-base.ssh`; finalizes a completed Claude session on the remote host.
- **Configuration choices:**
  - Reads session JSON from `Handle Session Command`
  - Extracts `session_id` and `issue_id` using Python
  - If no session exists, emits `NO_SESSION`
  - Runs Claude one final time to request:
    - “Summarize what you worked on in 2-3 sentences”
  - Archives current session into `session_log` with outcome `done`
  - Deletes rows from `sessions` and `queue`
  - Removes lock file
  - Creates cooldown marker `gateway.cooldown.<ISSUE>`
  - Echoes `SESSION_ENDED:<ISSUE>`
  - `continueOnFail` enabled
- **Key expressions or variables used:**
  - `$("Gateway Config").first().json.DB_PATH`
  - `$("Gateway Config").first().json.CONTEXT_DIR`
  - `$("Gateway Config").first().json.CLAUDE_PROJECT_PATH`
  - `$("Gateway Config").first().json.CLAUDE_BINARY`
  - `$("Handle Session Command").first().json.stdout`
- **Input and output connections:** Input from `Is Session Done?`; output to `Post Session Ended`.
- **Version-specific requirements:** SSH version `1`.
- **Edge cases or potential failure types:**
  - The Claude-generated summary is not actually posted to YouTrack in this node despite the final notice claiming it is.
  - `COOLDOWN_TTL` is not used to expire cooldown files.
  - If the Claude summary step fails, archival still may or may not continue depending on shell execution flow.
  - Requires `session_log` table to exist.
- **Sub-workflow reference:** None.

#### Post Session Ended
- **Type and technical role:** `n8n-nodes-base.httpRequest`; posts final session completion notice to Matrix.
- **Configuration choices:**
  - PUT request with event ID `end-{{ Date.now() }}`
  - Sends notice: `Session ended. Summary posted to YouTrack.`
  - `continueOnFail` enabled
- **Key expressions or variables used:**
  - `$('Gateway Config').first().json.MATRIX_HOMESERVER`
  - `$('Gateway Config').first().json.MATRIX_ROOM_ID`
- **Input and output connections:** Input from `Clean Up & End Session`; no downstream connection.
- **Version-specific requirements:** HTTP Request version `4.2`.
- **Edge cases or potential failure types:**
  - Notice is misleading because no YouTrack update occurs in the provided workflow.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Gateway Config | Set | Central configuration values for Matrix, Claude, SQLite, YouTrack, GitLab |  | Referenced by expressions only | ## Manage AI coding sessions from Matrix<br>A chat-ops bridge between Matrix, Claude Code,<br>YouTrack, and GitLab. Your team talks to an AI<br>coding assistant from a chat room — with issue<br>tracking and CI/CD visibility built in.<br><br>## How it works<br>1. Polls a Matrix room every 30s for messages<br>2. Routes `!commands` to the matching handler<br>3. Forwards messages to Claude Code via SSH<br>4. Posts Claude's response back to Matrix<br>5. Syncs session state with YouTrack issues<br><br>## Setup steps<br>1. Import and edit the **Gateway Config** node<br>2. Create n8n credentials: SSH + Matrix token<br>3. Set `YOUTRACK_TOKEN` and `GITLAB_TOKEN`<br>4. Create the SQLite database (see description)<br>5. Activate the workflow<br>### 1. Configuration<br>User-configurable variables. |
| Poll Every 30s | Schedule Trigger | Periodic workflow entry point for Matrix polling |  | Get Sync Token | ### 2. Matrix polling<br>Polls Matrix /sync every 30 seconds, extracts new messages, filters empty batches. |
| Get Sync Token | Code | Loads previous Matrix sync token from workflow static data | Poll Every 30s | Poll Matrix Sync | ### 2. Matrix polling<br>Polls Matrix /sync every 30 seconds, extracts new messages, filters empty batches. |
| Poll Matrix Sync | HTTP Request | Calls Matrix `/sync` API for the configured room | Get Sync Token | Extract Messages | ### 2. Matrix polling<br>Polls Matrix /sync every 30 seconds, extracts new messages, filters empty batches. |
| Extract Messages | Code | Parses sync response, updates tokens, extracts latest non-bot message | Poll Matrix Sync | Has Messages? | ### 2. Matrix polling<br>Polls Matrix /sync every 30 seconds, extracts new messages, filters empty batches. |
| Has Messages? | If | Ensures a non-empty message is present before routing | Extract Messages | Detect Command | ### 2. Matrix polling<br>Polls Matrix /sync every 30 seconds, extracts new messages, filters empty batches. |
| Detect Command | Code | Parses Matrix input into message or command metadata | Has Messages? | Command Router | ### 3. Command routing<br>Parses `!commands`, routes to handlers. |
| Command Router | Switch | Dispatches commands to message or management handlers | Detect Command | Check Lock; Handle Session Command; Handle Help; Handle Issue Command; Handle Pipeline Command; Handle System Command; Handle Unknown Command | ### 3. Command routing<br>Parses `!commands`, routes to handlers. |
| Check Lock | SSH | Checks remote gateway lock state before sending to Claude | Command Router | Is Locked? | ### 4. Message handler<br>Checks lock, reads session, resumes Claude via SSH, posts response to Matrix. |
| Is Locked? | If | Branches on busy vs available Claude gateway state | Check Lock | Post Busy Notice; Read Session & Acquire Lock | ### 4. Message handler<br>Checks lock, reads session, resumes Claude via SSH, posts response to Matrix. |
| Post Busy Notice | HTTP Request | Posts busy notice to Matrix when lock is active | Is Locked? |  | ### 4. Message handler<br>Checks lock, reads session, resumes Claude via SSH, posts response to Matrix. |
| Read Session & Acquire Lock | SSH | Writes lock file and loads current session from SQLite | Is Locked? | Resume Claude Session | ### 4. Message handler<br>Checks lock, reads session, resumes Claude via SSH, posts response to Matrix. |
| Resume Claude Session | SSH | Resumes active Claude session and sends user message | Read Session & Acquire Lock | Parse Claude Response | ### 4. Message handler<br>Checks lock, reads session, resumes Claude via SSH, posts response to Matrix. |
| Parse Claude Response | Code | Converts Claude CLI output into Matrix message JSON | Resume Claude Session | Post Response to Matrix | ### 4. Message handler<br>Checks lock, reads session, resumes Claude via SSH, posts response to Matrix. |
| Post Response to Matrix | HTTP Request | Sends Claude response back into the Matrix room | Parse Claude Response | Release Lock | ### 4. Message handler<br>Checks lock, reads session, resumes Claude via SSH, posts response to Matrix. |
| Release Lock | SSH | Removes remote gateway lock file | Post Response to Matrix |  | ### 4. Message handler<br>Checks lock, reads session, resumes Claude via SSH, posts response to Matrix. |
| Handle Session Command | SSH | Executes session current/list/done/cancel/pause/resume logic in SQLite | Command Router | Format Session Response | ### 5. Command handlers<br>**!session** current, list, done, cancel, pause, resume<br>**!issue** status, info, start, verify, done, comment<br>**!pipeline** status, logs, retry<br>**!system** status \| **!help** reference |
| Format Session Response | Code | Formats session command output for Matrix | Handle Session Command | Post Command Response | ### 5. Command handlers<br>**!session** current, list, done, cancel, pause, resume<br>**!issue** status, info, start, verify, done, comment<br>**!pipeline** status, logs, retry<br>**!system** status \| **!help** reference |
| Handle Issue Command | SSH | Queries YouTrack from the remote host | Command Router | Format Issue Response | ### 5. Command handlers<br>**!session** current, list, done, cancel, pause, resume<br>**!issue** status, info, start, verify, done, comment<br>**!pipeline** status, logs, retry<br>**!system** status \| **!help** reference |
| Format Issue Response | Code | Formats YouTrack API results into Matrix notices | Handle Issue Command | Post Command Response | ### 5. Command handlers<br>**!session** current, list, done, cancel, pause, resume<br>**!issue** status, info, start, verify, done, comment<br>**!pipeline** status, logs, retry<br>**!system** status \| **!help** reference |
| Handle Pipeline Command | SSH | Queries GitLab-related project data from remote host | Command Router | Format Pipeline Response | ### 5. Command handlers<br>**!session** current, list, done, cancel, pause, resume<br>**!issue** status, info, start, verify, done, comment<br>**!pipeline** status, logs, retry<br>**!system** status \| **!help** reference |
| Format Pipeline Response | Code | Formats GitLab/project output into Matrix notices | Handle Pipeline Command | Post Command Response | ### 5. Command handlers<br>**!session** current, list, done, cancel, pause, resume<br>**!issue** status, info, start, verify, done, comment<br>**!pipeline** status, logs, retry<br>**!system** status \| **!help** reference |
| Handle Help | Code | Returns static help text for supported commands | Command Router | Post Command Response | ### 5. Command handlers<br>**!session** current, list, done, cancel, pause, resume<br>**!issue** status, info, start, verify, done, comment<br>**!pipeline** status, logs, retry<br>**!system** status \| **!help** reference |
| Handle System Command | SSH | Returns host load, memory, and Claude process count | Command Router | Format System Response | ### 5. Command handlers<br>**!session** current, list, done, cancel, pause, resume<br>**!issue** status, info, start, verify, done, comment<br>**!pipeline** status, logs, retry<br>**!system** status \| **!help** reference |
| Format System Response | Code | Formats host status output into Matrix notices | Handle System Command | Post Command Response | ### 5. Command handlers<br>**!session** current, list, done, cancel, pause, resume<br>**!issue** status, info, start, verify, done, comment<br>**!pipeline** status, logs, retry<br>**!system** status \| **!help** reference |
| Handle Unknown Command | Code | Handles unsupported commands or prebuilt notice payloads | Command Router | Post Command Response | ### 5. Command handlers<br>**!session** current, list, done, cancel, pause, resume<br>**!issue** status, info, start, verify, done, comment<br>**!pipeline** status, logs, retry<br>**!system** status \| **!help** reference |
| Post Command Response | HTTP Request | Posts command responses to Matrix | Format Session Response; Format Issue Response; Format Pipeline Response; Handle Help; Format System Response; Handle Unknown Command | Is Session Done? | ### 6. Response and session end<br>Posts output to Matrix. Archives session to SQLite on done. |
| Is Session Done? | If | Detects whether the session command should trigger final cleanup | Post Command Response | Clean Up & End Session | ### 6. Response and session end<br>Posts output to Matrix. Archives session to SQLite on done. |
| Clean Up & End Session | SSH | Summarizes, archives, and removes current session state | Is Session Done? | Post Session Ended | ### 6. Response and session end<br>Posts output to Matrix. Archives session to SQLite on done. |
| Post Session Ended | HTTP Request | Posts final session-ended notice to Matrix | Clean Up & End Session |  | ### 6. Response and session end<br>Posts output to Matrix. Archives session to SQLite on done. |
| Sticky Note | Sticky Note | Documentation note |  |  | ## Manage AI coding sessions from Matrix<br>A chat-ops bridge between Matrix, Claude Code,<br>YouTrack, and GitLab. Your team talks to an AI<br>coding assistant from a chat room — with issue<br>tracking and CI/CD visibility built in.<br><br>## How it works<br>1. Polls a Matrix room every 30s for messages<br>2. Routes `!commands` to the matching handler<br>3. Forwards messages to Claude Code via SSH<br>4. Posts Claude's response back to Matrix<br>5. Syncs session state with YouTrack issues<br><br>## Setup steps<br>1. Import and edit the **Gateway Config** node<br>2. Create n8n credentials: SSH + Matrix token<br>3. Set `YOUTRACK_TOKEN` and `GITLAB_TOKEN`<br>4. Create the SQLite database (see description)<br>5. Activate the workflow |
| Sticky Note 4fa9 | Sticky Note | Documentation note |  |  | ### 1. Configuration<br>User-configurable variables. |
| Sticky Note 7417 | Sticky Note | Documentation note |  |  | ### 2. Matrix polling<br>Polls Matrix /sync every 30 seconds, extracts new messages, filters empty batches. |
| Sticky Note 782c | Sticky Note | Documentation note |  |  | ### 3. Command routing<br>Parses `!commands`, routes to handlers. |
| Sticky Note c9e3 | Sticky Note | Documentation note |  |  | ### 4. Message handler<br>Checks lock, reads session, resumes Claude via SSH, posts response to Matrix. |
| Sticky Note 6e7b | Sticky Note | Documentation note |  |  | ### 5. Command handlers<br>**!session** current, list, done, cancel, pause, resume<br>**!issue** status, info, start, verify, done, comment<br>**!pipeline** status, logs, retry<br>**!system** status \| **!help** reference |
| Sticky Note dbbd | Sticky Note | Documentation note |  |  | ### 6. Response and session end<br>Posts output to Matrix. Archives session to SQLite on done. |
| Sticky Note Credentials | Sticky Note | Documentation note for credential setup |  |  | ### Credentials Setup<br><br>**Do NOT store API tokens in workflow nodes.**<br>Use n8n's credential system instead:<br><br>1. **Matrix Bot Token** — HTTP Header Auth<br>`Authorization: Bearer <token>`<br><br>2. **SSH Private Key** — SSH credential<br>for your Claude Code server<br><br>**API tokens for YouTrack & GitLab:**<br>SSH command nodes reference `$YT_TOKEN` and<br>`$GL_TOKEN` environment variables. Set these<br>in `~/.bashrc` on the remote server:<br>```<br>export YT_TOKEN=perm-YOUR-TOKEN<br>export GL_TOKEN=glpat-YOUR-TOKEN<br>```<br><br>This keeps tokens out of the workflow JSON<br>and uses the server's environment instead. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named something like:
   - `Manage AI coding sessions from Matrix with YouTrack and GitLab`

2. **Add a Set node** named `Gateway Config`.
   - Create string fields:
     1. `SSH_HOST`
     2. `MATRIX_HOMESERVER`
     3. `MATRIX_ROOM_ID`
     4. `MATRIX_BOT_USER`
     5. `YOUTRACK_URL`
     6. `GITLAB_URL`
     7. `CLAUDE_PROJECT_PATH`
     8. `CLAUDE_BINARY`
     9. `DB_PATH`
     10. `CONTEXT_DIR`
     11. `ISSUE_PREFIX`
     12. `COOLDOWN_TTL`
   - Fill them with your real values, for example:
     - `MATRIX_HOMESERVER`: `https://matrix.example.com`
     - `MATRIX_ROOM_ID`: `!roomId:matrix.example.com`
     - `MATRIX_BOT_USER`: `@bot:matrix.example.com`
     - `DB_PATH`: absolute path to your SQLite DB
     - `CONTEXT_DIR`: directory where lock/cooldown files live

3. **Create the Matrix credential**.
   - Credential type: **HTTP Header Auth**
   - Name: e.g. `Matrix Bot Token`
   - Header name: `Authorization`
   - Header value: `Bearer <your_matrix_access_token>`

4. **Create the SSH credential**.
   - Credential type: **SSH**
   - Name: e.g. `Server SSH Key`
   - Use the hostname, username, port, and private key for the remote Claude/SQLite host.
   - Make sure the remote user can:
     - run `sqlite3`
     - run `python3`
     - run `curl`
     - run `timeout`
     - execute the Claude CLI
     - write to `CONTEXT_DIR`

5. **Prepare the remote environment**.
   - Install or verify:
     - `sqlite3`
     - `python3`
     - `curl`
     - GNU coreutils-compatible commands
   - Ensure the remote shell exposes:
     - `YT_TOKEN`
     - `GL_TOKEN`
   - The workflow expects those variables inside SSH sessions. If `.bashrc` is not loaded for non-interactive SSH sessions, configure them in a profile that your SSH execution actually loads, or inject them another way.

6. **Create the SQLite database schema** on the remote server.
   - At minimum, this workflow expects tables:
     - `sessions`
     - `queue`
     - `session_log`
   - The SQL implied by the workflow requires columns such as:
     - `sessions`: `issue_id`, `issue_title`, `session_id`, `started_at`, `message_count`, `paused`, `is_current`
     - `queue`: `issue_id`
     - `session_log`: `issue_id`, `issue_title`, `session_id`, `started_at`, `message_count`, `outcome`
   - Also ensure there is at most one row with `is_current=1`.

7. **Add a Schedule Trigger** named `Poll Every 30s`.
   - Configure it to run every 30 seconds.
   - Connect it to `Get Sync Token`.

8. **Add a Code node** named `Get Sync Token`.
   - Paste logic equivalent to:
     - get global static data
     - read `matrixSinceToken`
     - return one item merging `sinceToken` with `Gateway Config`
   - Connect `Poll Every 30s` → `Get Sync Token`.

9. **Add an HTTP Request node** named `Poll Matrix Sync`.
   - Method: `GET`
   - Authentication: Generic Credential Type → HTTP Header Auth
   - Credential: `Matrix Bot Token`
   - URL expression should call:
     - `{{MATRIX_HOMESERVER}}/_matrix/client/v3/sync`
     - include `timeout=0`
     - include room filter for `MATRIX_ROOM_ID`
     - include timeline limit `10`
     - append `&since=<token>` only if `sinceToken` exists
   - Timeout: 15000 ms
   - Connect `Get Sync Token` → `Poll Matrix Sync`.

10. **Add a Code node** named `Extract Messages`.
    - Implement logic to:
      - read Matrix sync response
      - save `next_batch` to global static data
      - inspect `rooms.join`
      - select the configured room, or fallback to first joined room
      - take only `m.room.message` events
      - ignore events sent by `MATRIX_BOT_USER`
      - ignore events older than `lastProcessedTimestamp`
      - map events to:
        - `messageText`
        - `sender`
        - `timestamp`
      - update `lastProcessedTimestamp`
      - return the first message merged with config, or no items if none found
    - Connect `Poll Matrix Sync` → `Extract Messages`.

11. **Add an If node** named `Has Messages?`.
    - Condition:
      - `messageText` is not empty
    - Connect `Extract Messages` → `Has Messages?`.

12. **Add a Code node** named `Detect Command`.
    - Implement parsing logic for:
      - issue-prefixed message format like `PROJ-4: text`
      - plain message → `command=message`
      - `!command sub args...`
      - aliases:
        - `!done` → `session done`
        - `!cancel` → `session cancel`
        - `!status` → `session current`
      - base64-encode the original or stripped message for SSH-safe transport
      - generate `matrixBody` for empty input cases
    - Connect `Has Messages?` true output → `Detect Command`.

13. **Add a Switch node** named `Command Router`.
    - Route by `command`.
    - Create outputs:
      - `message`
      - `session`
      - `help`
      - `issue`
      - `pipeline`
      - `system`
    - Set fallback output for anything else.
    - Connect `Detect Command` → `Command Router`.

14. **Build the message/Claude path: add SSH node `Check Lock`.**
    - SSH credential: `Server SSH Key`
    - Command:
      - check if `CONTEXT_DIR/gateway.lock` exists
      - if it exists and is newer than 600 seconds, output `LOCKED`
      - else remove stale lock and output `FREE`
    - Connect `Command Router` `message` output → `Check Lock`.

15. **Add If node `Is Locked?`.**
    - Condition: `stdout.trim() == LOCKED`
    - Connect `Check Lock` → `Is Locked?`.

16. **Add HTTP Request node `Post Busy Notice`.**
    - Method: `PUT`
    - Authentication: Matrix header credential
    - URL:
      - `{{MATRIX_HOMESERVER}}/_matrix/client/v3/rooms/{{MATRIX_ROOM_ID}}/send/m.room.message/busy-{{Date.now()}}`
    - Send JSON body:
      - `{"msgtype":"m.notice","body":"Claude is busy processing a previous message. Your message has been queued."}`
    - Connect `Is Locked?` true branch → `Post Busy Notice`.

17. **Add SSH node `Read Session & Acquire Lock`.**
    - Enable `Continue On Fail`
    - Command should:
      - write `locked` into `CONTEXT_DIR/gateway.lock`
      - query SQLite for the current session:
        - `sessionId`
        - `issueId`
        - `issueTitle`
    - Connect `Is Locked?` false branch → `Read Session & Acquire Lock`.

18. **Add SSH node `Resume Claude Session`.**
    - Enable `Continue On Fail`
    - Use expressions to pull:
      - Claude binary path
      - project path
      - session JSON from previous SSH node
      - base64 message from `Detect Command`
    - Logic should:
      - normalize session JSON to one line
      - return `NO_SESSION` if empty
      - parse `sessionId` with `python3`
      - decode message from base64
      - run:
        - `timeout 300 <claude binary> -r <sessionId> -p "<message>" --output-format json --dangerously-skip-permissions`
    - Connect `Read Session & Acquire Lock` → `Resume Claude Session`.

19. **Add Code node `Parse Claude Response`.**
    - Logic:
      - prefer stdout, fallback to stderr
      - if output is `NO_SESSION`, create a Matrix notice telling user there is no active session
      - else try JSON parse and read `result`
      - else truncate raw output to about 4000 chars
      - output `matrixBody`
    - Connect `Resume Claude Session` → `Parse Claude Response`.

20. **Add HTTP Request node `Post Response to Matrix`.**
    - Method: `PUT`
    - Authentication: Matrix header credential
    - URL:
      - `{{MATRIX_HOMESERVER}}/_matrix/client/v3/rooms/{{MATRIX_ROOM_ID}}/send/m.room.message/resp-{{Date.now()}}`
    - JSON body from `matrixBody`
    - Connect `Parse Claude Response` → `Post Response to Matrix`.

21. **Add SSH node `Release Lock`.**
    - Enable `Continue On Fail`
    - Command:
      - remove `CONTEXT_DIR/gateway.lock`
    - Connect `Post Response to Matrix` → `Release Lock`.

22. **Build the session command path: add SSH node `Handle Session Command`.**
    - Enable `Continue On Fail`
    - Use `sub` from `Detect Command`, default `current`
    - Use `args[0]` as optional issue ID
    - Implement shell cases:
      - `current`
      - `list`
      - `done`
      - `cancel`
      - `pause`
      - `resume`
    - Use SQLite against `DB_PATH`
    - For `cancel`, also:
      - `pkill -f claude`
      - remove gateway lock
      - delete from `sessions` and `queue`
    - Connect `Command Router` `session` output → `Handle Session Command`.

23. **Add Code node `Format Session Response`.**
    - Parse outputs according to the requested session subcommand.
    - Produce readable notices such as:
      - current session info
      - list of sessions
      - ending message
      - paused/resumed/cancelled confirmation
    - Output `matrixBody`
    - Connect `Handle Session Command` → `Format Session Response`.

24. **Build the issue command path: add SSH node `Handle Issue Command`.**
    - Enable `Continue On Fail`
    - Use `sub` default `status`
    - Use remote env var `YT_TOKEN`
    - Support:
      - `status`: YouTrack issues in progress
      - `info <id>`: issue details
      - default: usage/help string
    - Connect `Command Router` `issue` output → `Handle Issue Command`.

25. **Add Code node `Format Issue Response`.**
    - Try JSON parse.
    - If array: join issue IDs and summaries
    - If single object: print ID, summary, and truncated description
    - Else show raw output
    - Connect `Handle Issue Command` → `Format Issue Response`.

26. **Build the pipeline path: add SSH node `Handle Pipeline Command`.**
    - Enable `Continue On Fail`
    - Use `sub` default `status`
    - Use remote env var `GL_TOKEN`
    - Implement current behavior:
      - `status`: list accessible GitLab projects and default branches
      - default: usage/help string
    - Connect `Command Router` `pipeline` output → `Handle Pipeline Command`.

27. **Add Code node `Format Pipeline Response`.**
    - Wrap stdout in a Matrix notice.
    - Connect `Handle Pipeline Command` → `Format Pipeline Response`.

28. **Build the help path: add Code node `Handle Help`.**
    - Create static help text listing:
      - `!session current`
      - `!session list`
      - `!session done`
      - `!session cancel`
      - `!session pause`
      - `!session resume`
      - `!issue status`
      - `!issue info <id>`
      - `!pipeline status`
      - `!system status`
      - prefixed issue syntax example
    - Connect `Command Router` `help` output → `Handle Help`.

29. **Build the system path: add SSH node `Handle System Command`.**
    - Enable `Continue On Fail`
    - Shell command should output:
      - load average
      - memory usage
      - count of Claude processes
    - Connect `Command Router` `system` output → `Handle System Command`.

30. **Add Code node `Format System Response`.**
    - Wrap stdout in a Matrix notice.
    - Connect `Handle System Command` → `Format System Response`.

31. **Add Code node `Handle Unknown Command`.**
    - If incoming item already contains `matrixBody`, pass it through.
    - Otherwise generate:
      - `Unknown command: !<command>`
      - `Type !help to see available commands.`
    - Connect `Command Router` fallback output → `Handle Unknown Command`.

32. **Add HTTP Request node `Post Command Response`.**
    - Method: `PUT`
    - Authentication: Matrix header credential
    - Enable `Continue On Fail`
    - URL:
      - `{{MATRIX_HOMESERVER}}/_matrix/client/v3/rooms/{{MATRIX_ROOM_ID}}/send/m.room.message/cmd-{{Date.now()}}`
    - Body: JSON from `matrixBody`
    - Connect all of these to it:
      - `Format Session Response`
      - `Format Issue Response`
      - `Format Pipeline Response`
      - `Handle Help`
      - `Format System Response`
      - `Handle Unknown Command`

33. **Add If node `Is Session Done?`.**
    - Condition:
      - `$('Handle Session Command').first().json.stdout` contains `session_id`
    - Connect `Post Command Response` → `Is Session Done?`

34. **Add SSH node `Clean Up & End Session`.**
    - Enable `Continue On Fail`
    - Read session JSON from `Handle Session Command`
    - Parse `session_id` and `issue_id`
    - If no session ID, output `NO_SESSION`
    - Run one final Claude prompt asking for a 2–3 sentence summary
    - Archive session into `session_log`
    - Delete corresponding rows from `sessions` and `queue`
    - Remove gateway lock
    - Create `gateway.cooldown.<ISSUE>`
    - Echo `SESSION_ENDED:<ISSUE>`
    - Connect `Is Session Done?` true branch → `Clean Up & End Session`

35. **Add HTTP Request node `Post Session Ended`.**
    - Method: `PUT`
    - Authentication: Matrix header credential
    - Enable `Continue On Fail`
    - URL:
      - `{{MATRIX_HOMESERVER}}/_matrix/client/v3/rooms/{{MATRIX_ROOM_ID}}/send/m.room.message/end-{{Date.now()}}`
    - Body:
      - `{"msgtype":"m.notice","body":"Session ended. Summary posted to YouTrack."}`
    - Connect `Clean Up & End Session` → `Post Session Ended`.

36. **Optionally add sticky notes** for maintainability.
    - Add notes for:
      - overview/setup
      - configuration
      - Matrix polling
      - command routing
      - message handler
      - command handlers
      - session ending
      - credential guidance

37. **Test credentials and remote dependencies** before activating.
    - Validate Matrix send and sync manually.
    - Validate SSH access and command availability manually.
    - Confirm Claude CLI can resume an existing session ID.
    - Confirm DB schema matches expected SQL.

38. **Run controlled tests in this order**:
    1. `!help`
    2. `!system status`
    3. `!issue status`
    4. `!pipeline status`
    5. `!session current`
    6. plain non-command message with an active session
    7. `!session done`

39. **Important implementation caveats to preserve or improve during rebuild**:
    - The workflow currently processes only the first new Matrix message per polling batch.
    - Busy notice claims queuing, but no actual queue insertion happens on locked-message path.
    - Sticky notes mention more `!issue` and `!pipeline` subcommands than are implemented.
    - Session completion notice says summary is posted to YouTrack, but no YouTrack update occurs.
    - Lock release is vulnerable if Matrix response posting fails before `Release Lock`.

40. **Recommended hardening if you extend it**:
    - Add an error workflow or `try/finally`-style unlock strategy.
    - Process all extracted Matrix messages instead of only the first one.
    - Replace timestamp deduplication with event ID tracking.
    - Implement real queue storage when locked.
    - Implement actual YouTrack comment/update for end summaries.
    - Use `ISSUE_PREFIX` consistently in parsing and validation.

**Sub-workflow setup:**  
This workflow does **not** invoke any sub-workflows and does not require any `Execute Workflow` node configuration.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Do not store API tokens directly in nodes; use n8n credentials for Matrix and SSH. | Operational security guidance from the workflow notes |
| Matrix credential should be HTTP Header Auth with `Authorization: Bearer <token>`. | Matrix API authentication |
| YouTrack and GitLab tokens are expected as remote environment variables: `YT_TOKEN` and `GL_TOKEN`. | Remote server shell environment |
| Example environment variables shown in notes: `export YT_TOKEN=perm-YOUR-TOKEN` and `export GL_TOKEN=glpat-YOUR-TOKEN` | Remote host setup |
| The workflow claims “Create the SQLite database (see description)”, but the provided workflow JSON does not include the schema creation step. | Rebuild prerequisite |
| The workflow title says “Manage Claude Code sessions from Matrix with YouTrack and GitLab”, while the internal workflow name is “Manage AI coding sessions from Matrix with YouTrack and GitLab”. | Naming discrepancy |
| Several sticky-note-listed command capabilities are not implemented in the actual nodes: `!issue start`, `verify`, `done`, `comment`, and `!pipeline logs`, `retry`. | Functional gap to account for during extension |
| The final session-end notice says the summary was posted to YouTrack, but the provided nodes do not perform a YouTrack write-back. | Behavior mismatch |