Send pre-meeting Slack briefings using Google Calendar, Notion, GitHub, and Jira

https://n8nworkflows.xyz/workflows/send-pre-meeting-slack-briefings-using-google-calendar--notion--github--and-jira-13224


# Send pre-meeting Slack briefings using Google Calendar, Notion, GitHub, and Jira

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** Pre-Meeting Context Bot  
**Template title:** Send pre-meeting Slack briefings using Google Calendar, Notion, GitHub, and Jira

**Purpose:**  
When a new Google Calendar event is created, the workflow prepares a “pre-meeting context” briefing: it pulls the most recent past meeting notes from Notion, waits until **15 minutes before** the meeting starts, finds relevant work items in **GitHub** and **Jira** related to the meeting title, and then sends a **Slack DM** to each attendee (matched by email).

**Primary use cases:**
- Automatically brief meeting participants with agenda topics (PRs/tickets) and previous meeting notes.
- Reduce time spent searching for context right before recurring/related meetings.

### Logical Blocks
1.1 **Trigger & Event Normalization (Google Calendar)**  
1.2 **Previous Meeting Notes (Notion) + Validation**  
1.3 **Timing Control (T-15 minutes) + Keyword Extraction**  
1.4 **Work Item Discovery (GitHub PRs + Jira issues) & Aggregation**  
1.5 **Slack Message Construction + Attendee Fan-out**  
1.6 **Slack User Lookup & DM Delivery (with safe skip)**  
1.7 **Documentation/Annotations (Sticky Notes)**

---

## 2. Block-by-Block Analysis

### 1.1 Trigger & Event Normalization (Google Calendar)

**Overview:**  
Detects new calendar events and extracts the minimal event payload needed downstream (start time, title, attendees).

**Nodes involved:**
- **Capture New Google Calendar Event**
- **Format Calenda Event Payload**

#### Node: Capture New Google Calendar Event
- **Type / role:** `Google Calendar Trigger` — polls for newly created events.
- **Key configuration:**
  - Trigger: `eventCreated`
  - Polling: every minute
  - Calendar: `user@example.com` (list selection)
- **Inputs/outputs:**
  - Entry node; outputs event JSON from Google Calendar API.
- **Credentials:** Google Calendar OAuth2.
- **Edge cases / failures:**
  - OAuth token expiration / revoked consent.
  - Polling delays: “created” events might be picked up up to 1 minute later.
  - Calendar permissions (insufficient scope to read attendees).
- **Version notes:** TypeVersion `1`.

#### Node: Format Calenda Event Payload
- **Type / role:** `Code` — normalizes calendar event fields into a compact object.
- **Key configuration choices:**
  - Reads only the **first** trigger item: `$input.first()`.
  - Derives `eventStart` from `start.date` (all-day) or `start.dateTime` (timed).
- **Key variables/fields produced:**
  - `eventStart` (Date object)
  - `summary` (meeting title)
  - `attendees` (array)
- **Connections:**
  - Input: Capture New Google Calendar Event
  - Output: Get Last Meeting Notes
- **Edge cases / failures:**
  - `start` can be missing or in unexpected format → `new Date(undefined)` becomes invalid.
  - `attendees` can be undefined for events without guests or restricted visibility.
- **Version notes:** TypeVersion `2`.

---

### 1.2 Previous Meeting Notes (Notion) + Validation

**Overview:**  
Fetches the latest meeting note from a Notion database and ensures it occurred before the current calendar event. Then it normalizes the payload for later use.

**Nodes involved:**
- **Get Last Meeting Notes**
- **Is Previous Meeting**
- **Normalize Calendar and Notion Payloads**

#### Node: Get Last Meeting Notes
- **Type / role:** `Notion` — reads from a Notion database.
- **Key configuration choices:**
  - Resource: `databasePage`
  - Operation: `getAll`
  - Limit: `1`
  - Sort: `Meeting Date|date` descending (most recent first)
  - Database: “Meeting Notes”
- **Connections:**
  - Input: Format Calenda Event Payload
  - Output: Is Previous Meeting
- **Credentials:** Notion API.
- **Edge cases / failures:**
  - Database schema mismatch (property name “Meeting Date” must exist and be a date).
  - No results → downstream nodes may fail if they assume `properties` exist.
  - Rate limits from Notion.
- **Version notes:** TypeVersion `2.2`.

#### Node: Is Previous Meeting
- **Type / role:** `Filter` — keeps Notion page only if it is before current event.
- **Key configuration choices:**
  - Condition: Notion `properties["Meeting Date"].date.start` **before** current `eventStart`
  - Right value pulled via expression: `$('Format Calenda Event Payload').item.json.eventStart`
- **Connections:**
  - Input: Get Last Meeting Notes
  - Output (pass): Normalize Calendar and Notion Payloads
  - Output (fail): **Not connected** (items filtered out end the branch)
- **Edge cases / failures:**
  - If Notion “Meeting Date” is empty, expression resolves to null → filter may behave unexpectedly.
  - Timezone differences: Notion date vs Calendar dateTime could cause borderline comparisons.
- **Version notes:** TypeVersion `2.3`.

#### Node: Normalize Calendar and Notion Payloads
- **Type / role:** `Set` — builds a unified “meeting context” object.
- **Key configuration choices / fields:**
  - `previousMeetingNotes` from: `$json.properties.Notes.rich_text[0].plain_text`
  - `eventStart`, `summary`, `attendees` copied from “Format Calenda Event Payload”
- **Connections:**
  - Input: Is Previous Meeting
  - Output: Add 15 minute Early Time
- **Edge cases / failures:**
  - `rich_text[0]` can be missing if Notes is empty → expression error.
  - Assumes Notion property is named exactly `Notes` and is `rich_text`.
- **Important integration issue:**
  - Later logic expects `previousMeetingDate`, but **this node does not set it**. The PR filtering node references:
    - `$('Normalize Calendar and Notion Payloads').first().json.previousMeetingDate`
    - This will be `null/undefined` unless added. (See Block 1.4)
- **Version notes:** TypeVersion `3.4`.

---

### 1.3 Timing Control (T-15 minutes) + Keyword Extraction

**Overview:**  
Computes the notification time (15 minutes before start), waits until that time, then extracts keywords from the meeting title to drive GitHub/Jira relevance filtering.

**Nodes involved:**
- **Add 15 minute Early Time**
- **Wait for Time to Come**
- **Extract Meeting Keywords**

#### Node: Add 15 minute Early Time
- **Type / role:** `Code` — computes `notifyAt`.
- **Key configuration choices:**
  - `notifyAt = eventStart - 15 minutes`
  - Returns merged JSON with `notifyAt`
- **Connections:**
  - Input: Normalize Calendar and Notion Payloads
  - Output: Wait for Time to Come
- **Edge cases / failures:**
  - Invalid `eventStart` → invalid date math.
  - If meeting starts in <15 minutes, `notifyAt` is in the past; Wait node behavior depends on n8n version (often resumes immediately).
- **Version notes:** TypeVersion `2`.

#### Node: Wait for Time to Come
- **Type / role:** `Wait` — pauses execution until a specific time.
- **Key configuration choices:**
  - Resume mode: `specificTime`
  - DateTime expression: `={{ $json.notifyAt }}`
- **Connections:**
  - Input: Add 15 minute Early Time
  - Output: Extract Meeting Keywords
- **Edge cases / failures:**
  - If n8n restarts while waiting, wait persistence depends on platform configuration.
  - If `notifyAt` is invalid, node will error.
- **Version notes:** TypeVersion `1.1`.

#### Node: Extract Meeting Keywords
- **Type / role:** `Code` — generates searchable keywords from meeting title.
- **Key configuration choices:**
  - Lowercases title, removes symbols, splits by spaces
  - Keeps words length > 2
  - Takes first 5 words and joins into a single string
- **Outputs:**
  - `meetingTitle`
  - `keywords` (space-separated)
- **Connections:**
  - Input: Wait for Time to Come
  - Output: Get Jira Issues Related to Meeting **and** Get PRs from Repo (fan-out)
- **Edge cases / failures:**
  - Titles with only short words produce empty keywords → PR filtering returns none; Jira JQL may become too broad/invalid depending on implementation.
- **Version notes:** TypeVersion `2`.

---

### 1.4 Work Item Discovery (GitHub PRs + Jira issues) & Aggregation

**Overview:**  
Fetches candidate PRs from GitHub and issues from Jira, filters PRs for relevance, merges both sources, and normalizes them into a single list.

**Nodes involved:**
- **Get PRs from Repo**
- **Filter PRs Related to Meeting**
- **Get Jira Issues Related to Meeting**
- **Merge Jira Tickets and Github PR**
- **Prepare PRs and Jira Ticket List**

#### Node: Get PRs from Repo
- **Type / role:** `GitHub` — fetches repository items (configured as “repository” resource).
- **Key configuration choices:**
  - Authentication: OAuth2
  - Repository selected: `TestRepo`
  - **Owner parameter is blank** (`value":""`) which can break requests unless n8n derives it from repository selection.
- **Connections:**
  - Input: Extract Meeting Keywords
  - Output: Filter PRs Related to Meeting
- **Edge cases / failures:**
  - Owner not set → API request failure.
  - Token missing `repo` scopes for private repos.
  - Depending on node operation defaults, it may fetch issues not PRs unless operation is explicitly set (not shown in parameters).
- **Version notes:** TypeVersion `1.1`.

#### Node: Filter PRs Related to Meeting
- **Type / role:** `Code` — filters PR list by date and keyword relevance.
- **Key configuration choices:**
  - `keywords` loaded from `Extract Meeting Keywords`
  - `previousMeetingDate` expected from `Normalize Calendar and Notion Payloads` but **not actually provided**
  - Filters out PRs created before/at last meeting date (if available)
  - Filters in PRs whose `title` or `body` contains any keyword
- **Connections:**
  - Input: Get PRs from Repo
  - Output: Merge Jira Tickets and Github PR (input 0)
- **Edge cases / failures:**
  - If `keywords` is empty, `keywords.some(...)` is always false → returns no PRs.
  - If GitHub node output structure differs (e.g., not `created_at`), filter may remove everything.
- **Version notes:** TypeVersion `2`.

#### Node: Get Jira Issues Related to Meeting
- **Type / role:** `Jira Software` — searches issues.
- **Key configuration choices:**
  - Operation: `getAll`
  - Uses JQL in options:
    - `=text ~ "{{ $json.meetingTitle }}" ORDER BY updated DESC`
- **Connections:**
  - Input: Extract Meeting Keywords
  - Output: Merge Jira Tickets and Github PR (input 1)
- **Edge cases / failures:**
  - **Expression likely incorrect:** In n8n, JQL should typically be set with an expression like:
    - `={{ 'text ~ "' + $json.meetingTitle + '" ORDER BY updated DESC' }}`
    - As written, it may literally send `{{ $json.meetingTitle }}` to Jira.
  - Jira permissions / project access restrictions.
  - JQL “text ~” can be slow on large instances; rate limiting.
- **Version notes:** TypeVersion `1`.

#### Node: Merge Jira Tickets and Github PR
- **Type / role:** `Merge` — combines two input streams.
- **Key configuration choices:**
  - Default merge behavior (not explicitly set). In n8n v3 Merge, default is typically “Append” but should be verified.
- **Connections:**
  - Input 0: Filter PRs Related to Meeting
  - Input 1: Get Jira Issues Related to Meeting
  - Output: Prepare PRs and Jira Ticket List
- **Edge cases / failures:**
  - If one branch returns no items, merge behavior depends on mode (append vs combine by position).
- **Version notes:** TypeVersion `3.2`.

#### Node: Prepare PRs and Jira Ticket List
- **Type / role:** `Code` — normalizes merged items into `tickets[]`.
- **Key configuration choices:**
  - If no input items: returns `{ tickets: [] }`
  - Detects GitHub PRs by `html_url` + `number`
  - Detects Jira issues by `key` + `fields`
  - Outputs one item with `json.tickets`
- **Connections:**
  - Input: Merge Jira Tickets and Github PR
  - Output: Build Message for Slack
- **Edge cases / failures:**
  - Jira issue URL uses `self` (API URL) not the browse URL; less user-friendly in Slack.
  - Unknown item shapes yield `{}` entries; later filtering in Slack message removes invalid ones.
- **Version notes:** TypeVersion `2`.

---

### 1.5 Slack Message Construction + Attendee Fan-out

**Overview:**  
Builds Slack Block Kit message containing meeting title/time, topics (tickets), and previous notes; then prepares attendees list and fans out one item per attendee.

**Nodes involved:**
- **Build Message for Slack**
- **Normalize Attendees and Slack Message**
- **Fan Out Attendees**

#### Node: Build Message for Slack
- **Type / role:** `Code` — composes Slack Block Kit payload.
- **Key configuration choices:**
  - Reads `tickets` from current input (`Prepare PRs and Jira Ticket List`)
  - Reads `previousMeetingNotes` from `Normalize Calendar and Notion Payloads`
  - Reads `summary` and `eventStart` from `Format Calenda Event Payload`
  - Formats meeting time using locale `en-IN`
  - Filters tickets to those having `source`, `id`, `title`
- **Output:**
  - `{ blocks: [...] }` (Slack blocks array)
- **Connections:**
  - Input: Prepare PRs and Jira Ticket List
  - Output: Normalize Attendees and Slack Message
- **Edge cases / failures:**
  - Uses emoji in header text (`🧠`), which is fine for Slack blocks, but message payload must be valid JSON.
  - If previous notes are long, Slack block limits may be exceeded (section text max length).
- **Version notes:** TypeVersion `2`.

#### Node: Normalize Attendees and Slack Message
- **Type / role:** `Set` — packages attendees and message together.
- **Key configuration choices:**
  - `attendees` from `Add 15 minute Early Time` (original calendar attendees)
  - `message` set to the entire Slack block object from previous node (`={{ $json }}`)
- **Connections:**
  - Input: Build Message for Slack
  - Output: Fan Out Attendees
- **Edge cases / failures:**
  - If attendees missing, fan-out returns empty (no Slack messages).
- **Version notes:** TypeVersion `3.4`.

#### Node: Fan Out Attendees
- **Type / role:** `Code` — converts attendee array to per-attendee items.
- **Key configuration choices:**
  - Filters out attendees with `responseStatus === 'declined'`
  - Emits items shaped as `{ email, message }`
- **Connections:**
  - Input: Normalize Attendees and Slack Message
  - Output: Get User Slack Info from Email
- **Edge cases / failures:**
  - Attendee entries may not contain `email` (resource calendars, groups).
  - If event is private, attendees list might be hidden.
- **Version notes:** TypeVersion `2`.

---

### 1.6 Slack User Lookup & DM Delivery (with safe skip)

**Overview:**  
Looks up Slack users by attendee email, verifies a user exists, then sends the Slack DM with blocks.

**Nodes involved:**
- **Get User Slack Info from Email**
- **Check Slack User Found**
- **Send Meeting Context in Slack DM**

#### Node: Get User Slack Info from Email
- **Type / role:** `HTTP Request` — calls Slack Web API `users.lookupByEmail`.
- **Key configuration choices:**
  - URL: `https://slack.com/api/users.lookupByEmail`
  - Query parameter: `email={{ $json.email }}`
  - Header: `Authorization: Bearer {{ slack oauth token }}`
- **Connections:**
  - Input: Fan Out Attendees
  - Output: Check Slack User Found
- **Edge cases / failures:**
  - Token placeholder is not a real credential binding; should use n8n credentials or environment variable.
  - Slack scopes required: typically `users:read.email` (and possibly `users:read`).
  - Slack may return `{ ok:false, error:"users_not_found" }` for guests, mismatched emails, or external attendees.
- **Version notes:** TypeVersion `4.3`.

#### Node: Check Slack User Found
- **Type / role:** `IF` — verifies Slack lookup success.
- **Key configuration choices:**
  - Condition: `$json.ok === true`
- **Connections:**
  - Input: Get User Slack Info from Email
  - True output: Send Meeting Context in Slack DM
  - False output: **Not connected** (implicitly skips)
- **Edge cases / failures:**
  - If Slack API returns non-JSON or a 429, `$json.ok` may not exist.
- **Version notes:** TypeVersion `2.3`.

#### Node: Send Meeting Context in Slack DM
- **Type / role:** `Slack` — sends a DM with Block Kit message.
- **Key configuration choices:**
  - Message type: `block`
  - Recipient: `user.id` from Slack lookup response (`={{ $json.user.id }}`)
  - Blocks UI: `={{ $('Fan Out Attendees').item.json.message }}`
  - “includeLinkToWorkflow”: false
- **Connections:**
  - Input: Check Slack User Found (true branch)
  - Output: end
- **Credentials:** Slack API credential (OAuth).
- **Edge cases / failures:**
  - Blocks expression relies on paired item mapping; if item linkage breaks, blocks may be empty.
  - Slack block validation errors if content exceeds limits.
  - Missing `chat:write` scope prevents sending DMs.
- **Version notes:** TypeVersion `2.4`.

---

### 1.7 Documentation/Annotations (Sticky Notes)

**Overview:**  
Non-executing nodes providing visual documentation and setup instructions.

**Nodes involved:**
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5
- Sticky Note6
- Sticky Note7
- Sticky Note8

**Edge cases / failures:** None (non-executing).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Capture New Google Calendar Event | googleCalendarTrigger | Entry trigger: detect new events | — | Format Calenda Event Payload | ## The Trigger<br>- Triggers when a new Google Calendar event is created<br>- Captures essential meeting details (title, start time, attendees)<br>- Formats the event data for downstream processing |
| Format Calenda Event Payload | code | Normalize calendar payload | Capture New Google Calendar Event | Get Last Meeting Notes | ## The Trigger<br>- Triggers when a new Google Calendar event is created<br>- Captures essential meeting details (title, start time, attendees)<br>- Formats the event data for downstream processing |
| Get Last Meeting Notes | notion | Fetch latest meeting note from Notion DB | Format Calenda Event Payload | Is Previous Meeting | ## Previous Meeting Context<br>- Automatically pulls notes from the latest past meeting saved in Notion<br>- Ensures only meetings that happened **before the current one** are considered<br>- Cleans and organizes the information so it’s clear, consistent, and ready to use in the workflow |
| Is Previous Meeting | filter | Keep only notes before the current meeting | Get Last Meeting Notes | Normalize Calendar and Notion Payloads | ## Previous Meeting Context<br>- Automatically pulls notes from the latest past meeting saved in Notion<br>- Ensures only meetings that happened **before the current one** are considered<br>- Cleans and organizes the information so it’s clear, consistent, and ready to use in the workflow |
| Normalize Calendar and Notion Payloads | set | Build unified meeting context | Is Previous Meeting | Add 15 minute Early Time | ## Previous Meeting Context<br>- Automatically pulls notes from the latest past meeting saved in Notion<br>- Ensures only meetings that happened **before the current one** are considered<br>- Cleans and organizes the information so it’s clear, consistent, and ready to use in the workflow |
| Add 15 minute Early Time | code | Compute notifyAt = start - 15 min | Normalize Calendar and Notion Payloads | Wait for Time to Come | ## Meeting Preparation Timing<br>- Setting a variable which consist reminder time **15 minutes before the meeting starts**<br>- Waiting until the correct time to continue the workflow<br>- Extracting key words from the meeting title to understand what the meeting is about |
| Wait for Time to Come | wait | Pause until notifyAt | Add 15 minute Early Time | Extract Meeting Keywords | ## Meeting Preparation Timing<br>- Setting a variable which consist reminder time **15 minutes before the meeting starts**<br>- Waiting until the correct time to continue the workflow<br>- Extracting key words from the meeting title to understand what the meeting is about |
| Extract Meeting Keywords | code | Extract title keywords for search | Wait for Time to Come | Get Jira Issues Related to Meeting; Get PRs from Repo | ## Meeting Preparation Timing<br>- Setting a variable which consist reminder time **15 minutes before the meeting starts**<br>- Waiting until the correct time to continue the workflow<br>- Extracting key words from the meeting title to understand what the meeting is about |
| Get PRs from Repo | github | Fetch PRs/issues from GitHub repo | Extract Meeting Keywords | Filter PRs Related to Meeting | ## Find Relevant Pull Requests<br>- Fetching recent pull requests from the connected GitHub repository<br>- Filtering only the pull requests that are relevant to the current meeting topic |
| Filter PRs Related to Meeting | code | Filter PRs by keywords/date | Get PRs from Repo | Merge Jira Tickets and Github PR | ## Find Relevant Pull Requests<br>- Fetching recent pull requests from the connected GitHub repository<br>- Filtering only the pull requests that are relevant to the current meeting topic |
| Get Jira Issues Related to Meeting | jira | Search Jira issues via JQL | Extract Meeting Keywords | Merge Jira Tickets and Github PR | ## Find Relevant Jira Tickets<br>- Fetching Jira tickets related to the meeting topic<br>- Ensuring only relevant work items are included in the meeting summary |
| Merge Jira Tickets and Github PR | merge | Combine Jira + GitHub items | Filter PRs Related to Meeting; Get Jira Issues Related to Meeting | Prepare PRs and Jira Ticket List | ## Prepare Meeting Work Items<br>- Combining relevant GitHub pull requests and Jira tickets<br>- Organizing them into a structured list for sharing |
| Prepare PRs and Jira Ticket List | code | Normalize to unified tickets[] | Merge Jira Tickets and Github PR | Build Message for Slack | ## Prepare Meeting Work Items<br>- Combining relevant GitHub pull requests and Jira tickets<br>- Organizing them into a structured list for sharing |
| Build Message for Slack | code | Build Slack Block Kit payload | Prepare PRs and Jira Ticket List | Normalize Attendees and Slack Message | ## Prepare and Send Slack Messages<br>- Builds a structured Slack message containing meeting details, topics, and previous notes<br>- Aligns the message with the attendee list from the calendar event<br>- Splits attendees into individual recipients so each person can be messaged separately |
| Normalize Attendees and Slack Message | set | Package attendees + message together | Build Message for Slack | Fan Out Attendees | ## Prepare and Send Slack Messages<br>- Builds a structured Slack message containing meeting details, topics, and previous notes<br>- Aligns the message with the attendee list from the calendar event<br>- Splits attendees into individual recipients so each person can be messaged separately |
| Fan Out Attendees | code | Create one item per attendee | Normalize Attendees and Slack Message | Get User Slack Info from Email | ## Prepare and Send Slack Messages<br>- Builds a structured Slack message containing meeting details, topics, and previous notes<br>- Aligns the message with the attendee list from the calendar event<br>- Splits attendees into individual recipients so each person can be messaged separately |
| Get User Slack Info from Email | httpRequest | Slack lookup by email | Fan Out Attendees | Check Slack User Found | ## Send Meeting Context via Slack DM<br>- Looks up the Slack user using the attendee’s email address<br>- Verifies whether a matching Slack user exists<br>- Sends the meeting context as a direct message to the user if found<br>- Safely skips sending if no Slack user is available for the email |
| Check Slack User Found | if | Gate: only send if ok=true | Get User Slack Info from Email | Send Meeting Context in Slack DM (true) | ## Send Meeting Context via Slack DM<br>- Looks up the Slack user using the attendee’s email address<br>- Verifies whether a matching Slack user exists<br>- Sends the meeting context as a direct message to the user if found<br>- Safely skips sending if no Slack user is available for the email |
| Send Meeting Context in Slack DM | slack | Send DM with blocks | Check Slack User Found (true) | — | ## Send Meeting Context via Slack DM<br>- Looks up the Slack user using the attendee’s email address<br>- Verifies whether a matching Slack user exists<br>- Sends the meeting context as a direct message to the user if found<br>- Safely skips sending if no Slack user is available for the email |
| Sticky Note | stickyNote | Visual documentation | — | — | ## The Trigger<br>- Triggers when a new Google Calendar event is created<br>- Captures essential meeting details (title, start time, attendees)<br>- Formats the event data for downstream processing |
| Sticky Note1 | stickyNote | Visual documentation | — | — | ## Previous Meeting Context<br>- Automatically pulls notes from the latest past meeting saved in Notion<br>- Ensures only meetings that happened **before the current one** are considered<br>- Cleans and organizes the information so it’s clear, consistent, and ready to use in the workflow |
| Sticky Note2 | stickyNote | Visual documentation | — | — | ## Meeting Preparation Timing<br>…(as above)… |
| Sticky Note3 | stickyNote | Visual documentation | — | — | ## Find Relevant Pull Requests<br>…(as above)… |
| Sticky Note4 | stickyNote | Visual documentation | — | — | ## Find Relevant Jira Tickets<br>…(as above)… |
| Sticky Note5 | stickyNote | Visual documentation | — | — | ## Prepare Meeting Work Items<br>…(as above)… |
| Sticky Note6 | stickyNote | Visual documentation | — | — | ## Prepare and Send Slack Messages<br>…(as above)… |
| Sticky Note7 | stickyNote | Visual documentation | — | — | ## Send Meeting Context via Slack DM<br>…(as above)… |
| Sticky Note8 | stickyNote | Visual documentation / full workflow description | — | — | # 🧠 Pre-Meeting Context Automation<br>…(full note content in section 5)… |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n named **“Pre-Meeting Context Bot”**.

2) **Add Trigger: Google Calendar Trigger**
   - Node: **Google Calendar Trigger** (name it *Capture New Google Calendar Event*)
   - Configure:
     - Trigger On: **Event Created**
     - Poll interval: **Every minute**
     - Calendar: select the desired calendar
   - Credentials: create/connect **Google Calendar OAuth2** credential.

3) **Add Code node: format event payload**
   - Node: **Code** (name *Format Calenda Event Payload*)
   - Paste logic equivalent to:
     - Output `eventStart` from `start.date` or `start.dateTime`
     - Output `summary`
     - Output `attendees`
   - Connect: Trigger → Format node.

4) **Add Notion node: fetch last meeting note**
   - Node: **Notion** (name *Get Last Meeting Notes*)
   - Resource: **Database Page**
   - Operation: **Get All**
   - Database: select your “Meeting Notes” database
   - Sort: `Meeting Date` descending
   - Limit: `1`
   - Credentials: connect **Notion API** credential.
   - Connect: Format node → Notion node.

5) **Add Filter node: ensure note is before current event**
   - Node: **Filter** (name *Is Previous Meeting*)
   - Condition:
     - Left: Notion page date start (your property must match)
     - Operator: **before**
     - Right: expression referencing the calendar event start from the format node
   - Connect: Notion → Filter.

6) **Add Set node: normalize meeting context**
   - Node: **Set** (name *Normalize Calendar and Notion Payloads*)
   - Add fields:
     - `previousMeetingNotes` from Notion notes property (guard for empty rich_text if needed)
     - `eventStart`, `summary`, `attendees` from the formatted calendar node
   - **Recommended fix:** also add
     - `previousMeetingDate` from Notion “Meeting Date” (so PR filtering works)
   - Connect: Filter (true) → Set.

7) **Add Code node: compute notifyAt**
   - Node: **Code** (name *Add 15 minute Early Time*)
   - Subtract 15 minutes from `eventStart` and output `notifyAt`.
   - Connect: Normalize → Add 15 minute Early Time.

8) **Add Wait node: pause until notifyAt**
   - Node: **Wait** (name *Wait for Time to Come*)
   - Resume: **Specific time**
   - Date/Time: `={{ $json.notifyAt }}`
   - Connect: Add 15 minute Early Time → Wait.

9) **Add Code node: extract keywords**
   - Node: **Code** (name *Extract Meeting Keywords*)
   - Output:
     - `meetingTitle` (raw title)
     - `keywords` (cleaned/searchable terms)
   - Connect: Wait → Extract keywords.

10) **Add GitHub node: fetch PRs**
   - Node: **GitHub** (name *Get PRs from Repo*)
   - Authentication: **OAuth2**
   - Select repository (and ensure **Owner** is correctly set)
   - Ensure operation returns PRs (configure explicitly if needed in your n8n version).
   - Credentials: connect **GitHub OAuth2** credential.
   - Connect: Extract keywords → GitHub.

11) **Add Code node: filter PRs**
   - Node: **Code** (name *Filter PRs Related to Meeting*)
   - Filter:
     - created after `previousMeetingDate` (if provided)
     - contains any keyword in title/body
   - Connect: GitHub → Filter PRs.

12) **Add Jira node: search issues**
   - Node: **Jira Software** (name *Get Jira Issues Related to Meeting*)
   - Operation: **Get All**
   - JQL: build it using an n8n expression that inserts meetingTitle safely.
   - Credentials: connect **Jira Software Cloud API** credential.
   - Connect: Extract keywords → Jira.

13) **Add Merge node: append GitHub + Jira results**
   - Node: **Merge** (name *Merge Jira Tickets and Github PR*)
   - Mode: **Append** (recommended to avoid positional pairing issues)
   - Connect:
     - Filter PRs → Merge input 1
     - Jira → Merge input 2

14) **Add Code node: normalize to tickets[]**
   - Node: **Code** (name *Prepare PRs and Jira Ticket List*)
   - Output a single item: `{ tickets: [...] }`
   - Connect: Merge → Prepare list.

15) **Add Code node: build Slack blocks**
   - Node: **Code** (name *Build Message for Slack*)
   - Compose Slack blocks with:
     - meeting title/time
     - topics from `tickets`
     - previous notes
   - Connect: Prepare list → Build message.

16) **Add Set node: package attendees + message**
   - Node: **Set** (name *Normalize Attendees and Slack Message*)
   - Fields:
     - `attendees` from earlier calendar payload
     - `message` set to the Slack blocks object from the previous node
   - Connect: Build message → Set.

17) **Add Code node: fan out attendees**
   - Node: **Code** (name *Fan Out Attendees*)
   - Emit one item per attendee: `{ email, message }` (skip declined)
   - Connect: Normalize attendees/message → Fan out.

18) **Add HTTP Request node: Slack lookup by email**
   - Node: **HTTP Request** (name *Get User Slack Info from Email*)
   - GET `https://slack.com/api/users.lookupByEmail`
   - Query param: `email={{$json.email}}`
   - Header: `Authorization: Bearer <token>`
   - **Recommended:** store token in an n8n credential or env var, not hardcoded text.
   - Connect: Fan out → HTTP Request.

19) **Add IF node: proceed only if Slack user found**
   - Node: **IF** (name *Check Slack User Found*)
   - Condition: `={{ $json.ok }}` equals `true`
   - Connect: HTTP Request → IF.

20) **Add Slack node: send DM**
   - Node: **Slack** (name *Send Meeting Context in Slack DM*)
   - Send message as **Block Kit**
   - Recipient: user ID from lookup response
   - Blocks: use the `message` blocks from the attendee item
   - Credentials: connect **Slack OAuth** credential with required scopes:
     - `chat:write`
     - `users:read.email` (for lookup)
   - Connect: IF true → Slack send.

21) **(Optional but recommended) Handle “IF false” path**
   - Add a no-op/log node to record missing Slack users, or silently end.

22) **Activate workflow**, create a new calendar event with attendees, and verify DMs are sent ~15 minutes before start.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| # 🧠 Pre-Meeting Context Automation (full explanation + setup steps) | From Sticky Note8 (embedded in the workflow canvas) |
| Setup reminder: attendee emails in Calendar must match Slack emails | Sticky Note8 step 6 |
| Operational behavior: workflow waits until 15 minutes before meeting | Sticky Note2 |
| Safety behavior: if Slack user not found, workflow skips sending | Sticky Note7 |
| Important implementation gap: `previousMeetingDate` is referenced later but not set | PR filter uses `previousMeetingDate`; add it in “Normalize Calendar and Notion Payloads” |
| Jira JQL expression likely needs correction to actually interpolate meeting title | Current JQL contains `{{ $json.meetingTitle }}` literal; rebuild using n8n expression formatting |
| Slack lookup token should be managed via credentials or environment variables | HTTP Request node currently shows a placeholder token string |

If you want, I can propose concrete corrected expressions for the Notion notes extraction (null-safe), Jira JQL building, and adding `previousMeetingDate` so GitHub filtering behaves as intended.