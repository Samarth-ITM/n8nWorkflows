Monitor GitHub repo access and push events with GitHub and Slack alerts

https://n8nworkflows.xyz/workflows/monitor-github-repo-access-and-push-events-with-github-and-slack-alerts-12336


# Monitor GitHub repo access and push events with GitHub and Slack alerts

## 1. Workflow Overview

**Purpose:** Monitor a specific GitHub repository for potentially unauthorized or high-risk actions (membership changes, repository made public, and pushes) and send **Slack alerts** when activity violates predefined rules.

**Primary use cases**
- Detect unexpected access changes (member/team access events).
- Detect **repository visibility changes** (private → public).
- Detect pushes by users **not in an allowed whitelist**.

### 1.1 Scheduling & Event Retrieval
The workflow runs on a time interval and pulls the latest repository events via GitHub’s REST API.

### 1.2 Security Event Filtering
Only “critical” event types are kept for further checks (MemberEvent, PublicEvent, TeamAddEvent, PushEvent—though TeamAddEvent is filtered but not routed downstream).

### 1.3 Identity & Whitelist Lookup
For each remaining event, the workflow determines the relevant GitHub username and looks it up in an n8n **Data Table** (`it_whitelist`) to fetch the user’s role.

### 1.4 Policy Routing & Enforcement
A Switch routes by event type into dedicated branches. Each branch applies a rule (via IF nodes) to decide if the event is a violation.

### 1.5 Alerting (Slack)
Violations are posted to a Slack channel with details about the event and actor.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & GitHub Event Polling
**Overview:** Triggers the workflow on a schedule and fetches repository events from GitHub using an authenticated HTTP request.

**Nodes involved:**
- Schedule
- Github Events (HTTP)

#### Node: Schedule
- **Type / role:** `Schedule Trigger` — periodic trigger.
- **Config choices:** Runs every **1 hour** (`interval` with `field: hours`).  
  *Note:* A sticky note mentions “every 10 minutes”, but the node is configured for hourly polling.
- **Connections:**
  - Output → `Github Events (HTTP)`
- **Edge cases / failures:**
  - Time-based drift is minimal; but event volume between polls can be large for active repos.
  - If frequency is too low, older events may be missed depending on GitHub event API retention/order.

#### Node: Github Events (HTTP)
- **Type / role:** `HTTP Request` — calls GitHub REST API events endpoint.
- **Config choices:**
  - **URL:** `https://api.github.com/repos/<user_name>/<repository>/events` (must be customized).
  - **Auth:** Predefined credential type `githubApi` (PAT/OAuth via n8n GitHub credentials).
  - **Headers:**
    - `Accept: application/vnd.github+json`
    - `X-GitHub-Api-Version: 2022-11-28`
- **Input / Output:**
  - Input: schedule trigger
  - Output: array of GitHub event objects (one per item in n8n terms).
- **Version-specific requirements:** Node typeVersion `4.3` (HTTP Request node).
- **Edge cases / failures:**
  - 401/403 if token invalid or lacks scopes (private repo events typically need appropriate repo scopes).
  - 404 if repo path incorrect.
  - GitHub rate limiting (403 with rate-limit message).
  - Payload size and pagination: this endpoint returns a limited set of most recent events; older events may not be available.

---

### Block 2 — Security Event Filtering
**Overview:** Reduces noise by retaining only event types considered security-relevant.

**Nodes involved:**
- Filter Member Events (Code)

#### Node: Filter Member Events
- **Type / role:** `Code` — filters incoming GitHub events.
- **Config choices:**
  - Defines `criticalEvents`:
    - `MemberEvent`
    - `PublicEvent`
    - `TeamAddEvent`
    - `PushEvent`
  - Returns only items whose `item.json.type` is in that list.
- **Key variables/expressions:**
  - Uses `$input.all()` and filters by `item.json.type`.
- **Connections:**
  - Input ← `Github Events (HTTP)`
  - Output → `Check Whitelist`
- **Edge cases / failures:**
  - If GitHub response structure changes or `type` is missing, events may be silently dropped.
  - **TeamAddEvent is allowed through here but is not handled later in the Switch**, so it will fall through with no explicit path (see Switch block).

---

### Block 3 — Identity & Whitelist Lookup
**Overview:** Determines which GitHub username to check (member being added vs actor) and looks up that username in a Data Table to retrieve a role (e.g., `admin`, `developer`).

**Nodes involved:**
- Check Whitelist (Data Table)

#### Node: Check Whitelist
- **Type / role:** `Data Table` — retrieves matching whitelist rows from `it_whitelist`.
- **Config choices:**
  - **Operation:** `get`
  - **Filter condition:** column `github_username` equals:
    - If `MemberEvent`: `payload.member.login`
    - Else: `actor.login`
  - `alwaysOutputData: true` ensures downstream nodes run even when no match is found.
- **Key expressions:**
  - `{{ $json.type === 'MemberEvent' ? $json.payload.member.login : $json.actor.login }}`
- **Connections:**
  - Input ← `Filter Member Events`
  - Output → `Switch`
- **Edge cases / failures:**
  - If the Data Table doesn’t exist, is renamed, or columns don’t match (`github_username`, `role`), lookup fails.
  - If the event is `MemberEvent` but `payload.member.login` is missing (unusual but possible for malformed events), the lookup expression can evaluate to `undefined` and return no match.
  - Multiple matches for the same username may produce multiple rows/items; downstream checks may behave unexpectedly unless uniqueness is enforced in the table.
- **Data model expectation:**
  - Data Table name/id: `it_whitelist`
  - Columns: `github_username` (string), `role` (string; expected values include `admin`, others allowed)

---

### Block 4 — Routing & Security Rule Enforcement
**Overview:** Routes events by type and applies event-specific authorization logic. The intent is “True path = violation → alert”.

**Nodes involved:**
- Switch
- MemberEvent (IF)
- PublicEvent (IF)
- PushEvent (IF)
- No Operation, do nothing (NoOp)

#### Node: Switch
- **Type / role:** `Switch` — routes items to a branch based on event type.
- **Config choices:**
  - Three rules:
    - If type equals `MemberEvent` → output 0
    - If type equals `PublicEvent` → output 1
    - If type equals `PushEvent` → output 2
  - **Left value expression in each rule:** `{{ $('Filter Member Events').item.json.type }}`
    - This references the “Filter Member Events” node’s current item.
- **Connections:**
  - Input ← `Check Whitelist`
  - Outputs:
    - Output 0 → `MemberEvent`
    - Output 1 → `PublicEvent`
    - Output 2 → `PushEvent`
- **Edge cases / failures:**
  - **TeamAddEvent is not routed** (filtered earlier but not matched here): those items will not go anywhere after Switch.
  - Using `$('Filter Member Events').item.json.type` instead of `$json.type` can be fragile if execution context changes; best practice is usually `$json.type` at this point, since the item is the event item.

#### Node: MemberEvent (IF)
- **Type / role:** `IF` — intended to flag unauthorized member changes.
- **Config choices (as implemented):**
  - Condition: `{{ $('Filter Member Events').item.json.type }}` **notEquals** `"admin"`
- **Connections:**
  - Input ← `Switch` (MemberEvent route)
  - True → `Member Event Alert`
- **Important issue (logic bug):**
  - This compares **event type** (e.g., `"MemberEvent"`) against `"admin"`. It will almost always be “not equals admin” → **always true**, causing alerts for every MemberEvent.
  - Intended check (per sticky note) is: **user role is not admin**. That implies the IF should check something like `{{ $json.role }}` or `{{ $json[0]?.role }}` depending on the Data Table output shape.
- **Edge cases / failures:**
  - Even after fixing, you must clarify whether you’re checking the **actor** or the **added member**. The Data Table lookup currently uses `payload.member.login` for MemberEvent (the *added/removed member*), not the *actor* who performed it—this may not match your intended security policy.

#### Node: PublicEvent (IF)
- **Type / role:** `IF` — flags repo visibility change if not performed by an admin.
- **Config choices:**
  - Condition: `$json.role` **notEquals** `admin`
- **Connections:**
  - Input ← `Switch` (PublicEvent route)
  - True → `Public Event Alert`
- **Edge cases / failures:**
  - Depends on Data Table output: if no match, `$json.role` may be empty/undefined, which is “not admin” → triggers an alert (likely desired).
  - Ensure `role` is actually present at this stage; if the Data Table returns a different structure (e.g., wraps results), adjust accordingly.

#### Node: PushEvent (IF)
- **Type / role:** `IF` — flags pushes from unknown users (not in whitelist).
- **Config choices:**
  - Condition: `$json.role` is **empty**
- **Connections:**
  - Input ← `Switch` (PushEvent route)
  - True → `Push Event Alert`
  - False → `No Operation, do nothing`
- **Edge cases / failures:**
  - If Data Table returns multiple rows, role may not be a simple string.
  - If role exists but is whitespace or unexpected casing, “empty” may not behave as intended.

#### Node: No Operation, do nothing
- **Type / role:** `NoOp` — sink for non-violations in push branch.
- **Config choices:** None.
- **Connections:**
  - Input ← `PushEvent` (False path)
- **Edge cases / failures:** None; used to terminate the branch cleanly.

---

### Block 5 — Slack Alerting
**Overview:** Sends Slack messages for each violation type to a specified channel.

**Nodes involved:**
- Member Event Alert (Slack)
- Public Event Alert (Slack)
- Push Event Alert (Slack)

#### Node: Member Event Alert
- **Type / role:** `Slack` — posts alert to channel.
- **Config choices:**
  - Channel: `content-creation-agent` (channelId `C0917N0QN2C`)
  - Message text includes:
    - Actor: `{{ $json.actor.login }}`
    - Added member: `{{ $json.payload.member.login }}`
- **Connections:**
  - Input ← `MemberEvent` (True)
- **Edge cases / failures:**
  - Slack credential/auth errors (token revoked, missing scopes like `chat:write`).
  - Message may fail if payload fields are missing (e.g., payload.member not present).

#### Node: Public Event Alert
- **Type / role:** `Slack`
- **Config choices:**
  - Channel: `content-creation-agent`
  - Text: `"epository {{ $json.repository.name }} was made PUBLIC ..."` (typo: missing leading “R” in “Repository”)
- **Connections:**
  - Input ← `PublicEvent` (True)
- **Edge cases / failures:**
  - `repository.name` may not exist depending on event payload; GitHub events often have `repo.name` (like `owner/repo`) rather than `repository.name`. Verify actual event schema from GitHub response.

#### Node: Push Event Alert
- **Type / role:** `Slack`
- **Config choices:**
  - Channel: `content-creation-agent`
  - Text includes:
    - Event type: `{{ $('Filter Member Events').item.json.type }}`
    - Actor: `{{ $json.actor.login }}`
  - The message starts with an emoji (as configured).
- **Connections:**
  - Input ← `PushEvent` (True)
- **Edge cases / failures:**
  - The expression references `Filter Member Events` again; safer to use `$json.type` if available at this stage.
  - If GitHub event payload doesn’t include `actor.login` (rare), message becomes incomplete.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule | scheduleTrigger | Time-based trigger | — | Github Events (HTTP) | Monitor GitHub Repositories for Unauthorized Actions (how it works, setup steps, whitelist + rules). |
| Github Events (HTTP) | httpRequest | Fetch GitHub repo events | Schedule | Filter Member Events | GitHub Events & Identity Check: Poll events; filter; check whitelist. |
| Filter Member Events | code | Filter to critical event types | Github Events (HTTP) | Check Whitelist | GitHub Events & Identity Check: Poll events; filter; check whitelist. |
| Check Whitelist | dataTable | Lookup username/role in it_whitelist | Filter Member Events | Switch | GitHub Events & Identity Check: Poll events; filter; check whitelist. |
| Switch | switch | Route by event type | Check Whitelist | MemberEvent; PublicEvent; PushEvent | Switch and If Nodes to apply Security rules (True path = violation). |
| MemberEvent | if | Flag unauthorized member changes | Switch | Member Event Alert | Switch and If Nodes to apply Security rules (True path = violation). |
| PublicEvent | if | Flag repo made public by non-admin | Switch | Public Event Alert | Switch and If Nodes to apply Security rules (True path = violation). |
| PushEvent | if | Flag pushes by unknown users | Switch | Push Event Alert (True); No Operation, do nothing (False) | Switch and If Nodes to apply Security rules (True path = violation). |
| No Operation, do nothing | noOp | End branch for non-violations | PushEvent (False) | — | Switch and If Nodes to apply Security rules (True path = violation). |
| Member Event Alert | slack | Post Slack alert for MemberEvent violation | MemberEvent (True) | — | Slack Node (Alert System): posts message with event type and username; only on violation. |
| Public Event Alert | slack | Post Slack alert for PublicEvent violation | PublicEvent (True) | — | Slack Node (Alert System): posts message with event type and username; only on violation. |
| Push Event Alert | slack | Post Slack alert for PushEvent violation | PushEvent (True) | — | Slack Node (Alert System): posts message with event type and username; only on violation. |
| Sticky Note | stickyNote | Documentation/comment | — | — | (same content as note) |
| Sticky Note1 | stickyNote | Documentation/comment | — | — | (same content as note) |
| Sticky Note3 | stickyNote | Documentation/comment | — | — | (same content as note) |
| Sticky Note4 | stickyNote | Documentation/comment | — | — | (same content as note) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **“Monitor GitHub Repositories for Unauthorized Actions”** (or your desired title).

2. **Add Schedule Trigger**
   - Add node: **Schedule Trigger**
   - Set interval to **Every 1 hour** (or change to 10 minutes if desired to match the note).

3. **Add GitHub Events fetch (HTTP Request)**
   - Add node: **HTTP Request**
   - Method: `GET`
   - URL: `https://api.github.com/repos/<owner>/<repo>/events`
   - Authentication: **Predefined Credential Type**
   - Credential Type: **GitHub API**
     - Configure a GitHub credential (PAT or OAuth).
     - Ensure scopes cover repo access as needed (private repos require `repo` scope on PAT).
   - Headers:
     - `Accept = application/vnd.github+json`
     - `X-GitHub-Api-Version = 2022-11-28`
   - Connect: **Schedule → HTTP Request**

4. **Add filtering logic (Code node)**
   - Add node: **Code**
   - Paste logic that filters items by `$json.type` to keep: `MemberEvent`, `PublicEvent`, `TeamAddEvent`, `PushEvent`.
   - Connect: **HTTP Request → Code**

5. **Create the whitelist Data Table**
   - In n8n, create a **Data Table** named **`it_whitelist`**
   - Add columns:
     - `github_username` (text)
     - `role` (text; e.g., `admin`, `developer`)
   - Insert rows:
     - Your own GitHub username with role `admin` (prevents self-alerts).
     - Other authorized users as needed.

6. **Add Data Table lookup node**
   - Add node: **Data Table**
   - Operation: **Get**
   - Select Data Table: `it_whitelist`
   - Add filter condition:
     - Key: `github_username`
     - Value expression:
       - `{{ $json.type === 'MemberEvent' ? $json.payload.member.login : $json.actor.login }}`
   - Enable: **Always Output Data** (so routing continues even if no match)
   - Connect: **Code → Data Table**

7. **Add Switch node (route by event type)**
   - Add node: **Switch**
   - Rules (string equals):
     - `{{ $json.type }}` equals `MemberEvent` → Output 0
     - `{{ $json.type }}` equals `PublicEvent` → Output 1
     - `{{ $json.type }}` equals `PushEvent` → Output 2
   - Connect: **Data Table → Switch**
   - (Optional) Add a default branch to a NoOp/logging node for unhandled events like TeamAddEvent.

8. **Add IF nodes for enforcement**
   - Add node: **IF** named `MemberEvent`
     - Condition (recommended intent): role **not equals** `admin`
       - Expression typically: `{{ $json.role }}` notEquals `admin`
   - Add node: **IF** named `PublicEvent`
     - Condition: `{{ $json.role }}` notEquals `admin`
   - Add node: **IF** named `PushEvent`
     - Condition: `{{ $json.role }}` is empty
   - Connect Switch outputs:
     - Output 0 → MemberEvent IF
     - Output 1 → PublicEvent IF
     - Output 2 → PushEvent IF

9. **Add Slack alert nodes**
   - Create Slack credential: Slack Bot Token with `chat:write` to the target channel.
   - Add node: **Slack** named `Member Event Alert`
     - Post to channel (e.g., `content-creation-agent`)
     - Message template referencing `actor.login` and `payload.member.login`
     - Connect: MemberEvent IF **True** → Member Event Alert
   - Add node: **Slack** named `Public Event Alert`
     - Message about repo made public
     - Connect: PublicEvent IF **True** → Public Event Alert
   - Add node: **Slack** named `Push Event Alert`
     - Message about unauthorized push
     - Connect: PushEvent IF **True** → Push Event Alert

10. **Add a sink for non-violations**
   - Add node: **No Operation**
   - Connect: PushEvent IF **False** → No Operation

11. **Validate with test executions**
   - Execute the workflow manually.
   - Inspect actual GitHub event fields (`repo.name` vs `repository.name`, etc.).
   - Confirm Data Table lookup returns a `role` field in the same item that enters the IF nodes.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques. | Provided disclaimer (FR). |
| Sticky note indicates polling “every 10 minutes”, but Schedule is configured for “every hour”. | Align Schedule node with your intended monitoring frequency. |
| TeamAddEvent is filtered as critical but not routed in Switch. | Add a Switch rule + enforcement/alert branch if you want TeamAddEvent coverage. |
| MemberEvent IF node compares event type to “admin” (likely unintended). | Adjust to compare **role** to `admin`, and confirm whether you intend to check the actor vs the member being added. |
| Public alert message contains a typo (“epository”). | Fix Slack message text if needed. |