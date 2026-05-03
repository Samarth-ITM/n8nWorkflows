Invite the Fireflies recording bot to meetings from a one-click form

https://n8nworkflows.xyz/workflows/invite-the-fireflies-recording-bot-to-meetings-from-a-one-click-form-14987


# Invite the Fireflies recording bot to meetings from a one-click form

The user wants a comprehensive, structured, self-contained reference document analyzing the n8n workflow JSON they provided. I must follow the document structure with sections: 1. Workflow Overview, 2. Block-by-Block Analysis, 3. Summary Table, 4. Reproducing the Workflow from Scratch, 5. General Notes & Resources.

Let's parse the JSON to extract nodes and connections.

Nodes:
- "5. Set — Build Success Response" (id: 3ff42eda-d35f-4baa-b4c9-6e0ac6d615d1, type: n8n-nodes-base.set)
- "6. Respond — Bot Invited Successfully" (id: fc84d244-ddf0-4181-9cdf-7a641110c522, type: n8n-nodes-base.respondToWebhook)
- "Overview1" (id: fad4ccc3-d401-4071-a44e-1cd10eac432c, type: n8n-nodes-base.stickyNote)
- "Section — Form Input and Link Validation1" (id: 30ed9178-2679-4169-9fef-64742636d4bd, type: n8n-nodes-base.stickyNote)
- "Section — Fireflies API Call and Success Response1" (id: 76e58703-3e58-46bf-a365-2b6216360f41, type: n8n-nodes-base.stickyNote)
- "Note — Fireflies API Key Required1" (id: c1109d3a-a245-4be0-a6a6-f045efa512b3, type: n8n-nodes-base.stickyNote)
- "1. Form — Invite Fireflies Bot" (id: 2100ecbd-7e06-4c8e-b5eb-5038b4b84c95, type: n8n-nodes-base.formTrigger)
- "2. Set — Validate and Prepare Inputs" (id: 6739a7e5-61fb-4cd3-86e1-9323ef8a6bdf, type: n8n-nodes-base.set)
- "3. IF — Valid Meeting Link?" (id: 58876f51-d0d0-4dd3-b3bd-1e1ac6cb9c46, type: n8n-nodes-base.if)
- "4. HTTP — Send to Fireflies API" (id: 42280d1f-ea15-4476-96fb-f65b259a2728, type: n8n-nodes-base.httpRequest)

Connections:
- "3. IF — Valid Meeting Link?" -> true output -> "4. HTTP — Send to Fireflies API". false output -> not connected (empty)
- "1. Form — Invite Fireflies Bot" -> "2. Set — Validate and Prepare Inputs"
- "4. HTTP — Send to Fireflies API" -> "5. Set — Build Success Response"
- "5. Set — Build Success Response" -> "6. Respond — Bot Invited Successfully"
- "2. Set — Validate and Prepare Inputs" -> "3. IF — Valid Meeting Link?"

So the flow is: Form -> Set -> IF -> (true) HTTP -> Set (success HTML) -> Respond (webhook). There's also error path (IF false) that is not connected (should route to nodes 7 and 8, which are missing in JSON). The sticky notes mention nodes 7 and 8, but they are not present in nodes.

Now need to produce:

1. Workflow Overview: high-level explanation, target use cases, logical blocks.

2. Block-by-Block Analysis: group by functional blocks. We need to define blocks:

- Block 1: Input Reception (Form)
- Block 2: Input Validation and Normalization (Set)
- Block 3: Conditional Routing (IF)
- Block 4: API Call (HTTP)
- Block 5: Success Response Construction (Set)
- Block 6: Success Response Delivery (Respond)

The error path (IF false) not connected. Might mention it's unconnected.

We need for each node: type and technical role; configuration choices; key expressions or variables; input and output connections; version-specific requirements; edge cases; sub-workflow reference.

3. Summary Table: columns Node Name, Node Type, Functional Role, Input Node(s), Output Node(s), Sticky Note.

We need to list all nodes and sticky notes. The sticky notes have content. Some sticky notes visually cover multiple nodes. For each node, if a sticky note covers it, we duplicate its content. Let's find which nodes each sticky note covers. It can be determined by position and size.

Positions and sizes:

- Node positions: formTrigger at [-2208,272]; Set at [-1952,272]; IF at [-1696,272]; HTTP at [-1264,192]; Set success at [-912,192]; Respond at [-608,192].

Sticky notes:
- Overview1: position [-2848,-48], width 508, height 928 => covers from x -2848 to -2848+508 = -2340, y -48 to -48+928 = 880. So covers roughly x -2848 to -2340. The formTrigger at x -2208 is outside this x range. So Overview1 likely covers area before the actual flow. It is a big overview note probably covering the left side.

- Section — Form Input and Link Validation1: position [-2272,144], width 772, height 356 => x range -2272 to -2272+772 = -1500, y range 144 to 144+356 = 500. This likely covers nodes formTrigger, Set, IF. Let's see:

formTrigger at [-2208,272] (within x -2272 to -1500, y 272 within 144-500). Yes.
Set node at [-1952,272] (within x -2272 to -1500). Yes.
IF node at [-1696,272] (within x -2272 to -1500). Yes.
HTTP node at [-1264,192] is at x -1264 > -1500, so not included. So sticky note "Section — Form Input and Link Validation1" covers Form, Set, IF.

- Section — Fireflies API Call and Success Response1: position [-1408,80], width 1028, height 388 => x range -1408 to -1408+1028 = -380, y range 80 to 80+388 = 468. It likely covers HTTP, Set success, Respond. Let's check:

HTTP at [-1264,192] (x -1264 within -1408 to -380, y 192 within 80-468). Yes.
Set success at [-912,192] (within x range). Yes.
Respond at [-608,192] (within x range). Yes.

So that sticky note covers nodes 4,5,6.

- Note — Fireflies API Key Required1: position [-1920,592], width 1004, height 140 => x range -1920 to -1920+1004 = -916, y range 592 to 732. Does this cover any node? Node HTTP is at x -1264, y 192, which is below the sticky note (y 192 < 592). So not overlapping. Set node at y 272 < 592. So none of the functional nodes overlap with this note. It is separate, perhaps just a warning note placed below the flow.

Thus we assign sticky notes:

- "1. Form — Invite Fireflies Bot": sticky note "Section — Form Input and Link Validation1".
- "2. Set — Validate and Prepare Inputs": same sticky note.
- "3. IF — Valid Meeting Link?": same sticky note.
- "4. HTTP — Send to Fireflies API": sticky note "Section — Fireflies API Call and Success Response1".
- "5. Set — Build Success Response": same sticky note.
- "6. Respond — Bot Invited Successfully": same sticky note.
- Also "Note — Fireflies API Key Required1" does not cover any node? It is below the flow, but might be considered covering the HTTP node region? However, based on the coordinates, it does not overlap any node. We could still mention it separately in a separate row but with no input/output nodes.

- "Overview1" is likely separate and covers the whole workflow area. It might visually cover the flow partially? Coordinates: x -2848 to -2340, y -48 to 880. Our nodes start at x -2208 which is > -2340? Wait -2208 > -2340? Actually -2208 is greater (less negative) than -2340, meaning -2208 is to the right of -2340. So Overview1 ends at -2340, but nodes start at -2208, which is outside that x range. So Overview1 does not cover any functional nodes. It's a general note on left side.

Thus sticky notes for nodes are as above.

Now need to fill table.

Next, block-by-block analysis with nodes.

We need to note that the HTTP node uses placeholders "YOUR_TOKEN_HERE" which need replacement. Provide details.

Edge cases: IF check for meeting link not empty and starts with http; else false path not wired leads to no response (the description says nodes 7 and 8 error response not connected). We should note that currently false path is not connected.

Potential errors: API key missing or wrong; invalid meeting link; Fireflies API errors; expression parsing errors; missing fields.

Now let's produce the detailed document.

Will incorporate the description text for context, but must be self-contained.

Also note that workflow uses form trigger with responseMode "lastNode", meaning final respond node will send HTML.

Need to capture versions: formTrigger typeVersion 2.2, set typeVersion 3.4, if typeVersion 2, httpRequest typeVersion 4.2, respond typeVersion 1.1.

Let's craft the documentation.

---

We'll write sections.

### 1. Workflow Overview

Describe purpose: one-click form invites Fireflies recording bot to meetings. Logical blocks: Form input, Validation/Preparation, Conditional check, API call, Success response. Mention error path not wired.

Target use cases: sales teams, consultants etc.

---

### 2. Block-by-Block Analysis

Block 1: Input Reception (Form)
- Node: 1. Form — Invite Fireflies Bot
- Provide details.

Block 2: Input Validation & Preparation (Set)
- Node: 2. Set — Validate and Prepare Inputs

Block 3: Conditional Routing (IF)
- Node: 3. IF — Valid Meeting Link?

Block 4: Fireflies API Call (HTTP)
- Node: 4. HTTP — Send to Fireflies API

Block 5: Success Response Construction (Set)
- Node: 5. Set — Build Success Response

Block 6: Success Response Delivery (Respond)
- Node: 6. Respond — Bot Invited Successfully

Also note missing error path (nodes 7 and 8) not present.

Include sticky notes.

Now, we must ensure the sticky note content is preserved. Let's copy sticky note content.

- Overview1 content: "## Fireflies Bot Meeting Inviter — One-Click Form\n\nFor sales teams, account managers, and consultants who manually invite the Fireflies recording bot to every meeting and keep forgetting in the rush before calls. Bookmark this form URL. Before any sales call, open it, paste the meeting link, and submit. The bot joins instantly. No Fireflies dashboard needed, no login, no navigation — just the form.\n\nTitle defaults to Sales Meeting - Incrementors if left blank. Duration is clamped between 15 and 120 minutes. Invalid meeting links get an immediate error response instead of silently failing.\n\n## How it works\n- **1. Form — Invite Fireflies Bot** collects the meeting link, title, password, duration, and language\n- **2. Set — Validate and Prepare Inputs** cleans and normalises all five fields with sensible defaults\n- **3. IF — Valid Meeting Link?** checks the link is non-empty and starts with http — invalid links exit with an error message\n- **4. HTTP — Send to Fireflies API** calls the Fireflies GraphQL addToLiveMeeting mutation\n- **5. Set — Build Success Response** prepares the confirmation message shown on screen\n- **6. Respond — Bot Invited Successfully** returns the confirmation page to the submitter\n- **7. Set — Build Error Response** prepares the error message for invalid links\n- **8. Respond — Invalid Link Error** returns the error page to the submitter\n\n## Set up steps\n1. In **4. HTTP — Send to Fireflies API** — replace YOUR_FIREFLIES_API_KEY in the Authorization header with your actual key from app.fireflies.ai under Integrations then API\n2. In **2. Set — Validate and Prepare Inputs** — update the default title from Sales Meeting - Incrementors to your own default\n3. Activate the workflow and bookmark the Form URL from node 1\n4. Share the form URL with your sales team"

- Section — Form Input and Link Validation1 content: "## Form Input and Link Validation\nForm collects meeting link, title, password, duration, and language. Inputs are cleaned with defaults applied. The IF node checks the link is non-empty and starts with http — invalid links route to the error path immediately."

- Section — Fireflies API Call and Success Response1 content: "## Fireflies API Call and Success Response\nGraphQL mutation sends the meeting link to Fireflies. On success, a confirmation page is shown to the submitter with the meeting title and a reminder to check the Fireflies dashboard."

- Note — Fireflies API Key Required1 content: "## ⚠️ Fireflies API Key Required\nReplace YOUR_FIREFLIES_API_KEY in the Authorization header. Get your key from app.fireflies.ai — go to Integrations, then API, then copy your Personal API Key."

Will include these in sticky note column for nodes they cover.

Now the summary table: list all 6 functional nodes plus sticky notes as separate rows? Possibly include sticky notes as nodes but we can include them but they are not data nodes. The requirement: "List all workflow nodes in a single table." Sticky notes are nodes. So include them. Provide columns: Node Name, Node Type, Functional Role, Input Node(s), Output Node(s), Sticky Note. For sticky notes, Input/Output may be blank. Provide Sticky Note content for each node they cover.

Now produce step-by-step reproduction instructions.

Will need to include details for each node: type, version, configuration.

Let's plan.

---

Now produce final output. Use clear headings, tables, bullet points.

Make sure we do not skip any node.

Make sure we include notes: edge cases, version specifics.

Will also mention that the error response nodes (7 and 8) are referenced in the sticky notes but not present in JSON. We'll note that they are missing.

Now write. 1. Workflow Overview  

**Purpose** – Provide a single‑click web form that instantly invites the Fireflies.ai recording bot to a scheduled meeting. The user pastes a meeting link, optionally fills in a title, password, duration and language, submits the form, and receives a confirmation page – all without ever opening the Fireflies dashboard.  

**Target users** – Sales teams, account managers, consultants, or anyone who runs back‑to‑back calls and needs to ensure the Fireflies bot joins every meeting.  

**Logical blocks**  

| Block | Functional role | Key nodes |
|-------|-----------------|-----------|
| 1.1 Input Reception | Host a public form that collects the meeting link and optional metadata | 1. Form — Invite Fireflies Bot |
| 1.2 Input Validation & Preparation | Trim whitespace, apply sensible defaults, clamp duration, set language | 2. Set — Validate and Prepare Inputs |
| 1.3 Conditional Routing | Verify that the meeting link is non‑empty and starts with `http` | 3. IF — Valid Meeting Link? |
| 1.4 API Call | Send a GraphQL mutation to Fireflies to add the bot to the meeting | 4. HTTP — Send to Fireflies API |
| 1.5 Success Response Construction | Build an HTML confirmation card with title & duration | 5. Set — Build Success Response |
| 1.6 Success Response Delivery | Return the HTML page to the submitter’s browser | 6. Respond — Bot Invited Successfully |
| *(Planned but not wired)* | 7. Build Error Response & 8. Return Error Page – currently disconnected | – |

**Data flow** – Form → Set → IF → (true) HTTP → Set → Respond. The false branch of the IF node is **not connected** in the current JSON, so invalid links silently end without a response (nodes 7 and 8 mentioned in the sticky notes are missing).  

---

## 2. Block‑by‑Block Analysis  

### Block 1.1 – Input Reception  

**Overview** – Hosts an n8n‑managed form that is bookmarkable and shareable. On submission the workflow is triggered and the form data is passed downstream.  

| Node | Type & Technical Role | Key Configuration | Expressions / Variables | Input | Output | Edge Cases | Version |
|------|----------------------|-------------------|------------------------|-------|--------|------------|---------|
| **1. Form — Invite Fireflies Bot** | `n8n-nodes-base.formTrigger` – webhook‑based form entry point | • Title: *Invite Fireflies Bot to Sales Meeting*  <br>• Description: *Enter your meeting details — the bot will automatically join and begin recording instantly.* <br>• Fields (5): <br>  1. **Meeting Link** (text, required, placeholder `https://meet.google.com/xxx-xxxx-xxx`) <br>  2. **Meeting Title** (text, optional, placeholder `e.g. Sales Call - ClientName`) <br>  3. **Meeting Password (if any)** (text, optional) <br>  4. **Duration in Minutes** (number, placeholder `60 (min 15, max 120)`) <br>  5. **Meeting Language** (dropdown, options `en, hi, es, fr, pt`) <br>• Response Mode: `lastNode` (the final Respond node delivers HTML) | No expressions – raw field values are passed to next node | No previous node (form trigger) | `2. Set — Validate and Prepare Inputs` | • User can leave optional fields empty <br>• Non‑numeric duration entered → handled downstream <br>• Invalid links (e.g. missing `https://`) → caught in IF node | formTrigger v2.2 |

**Sticky note covering this node** – *Section — Form Input and Link Validation1*:  

> “## Form Input and Link Validation  
> Form collects meeting link, title, password, duration, and language. Inputs are cleaned with defaults applied. The IF node checks the link is non-empty and starts with http — invalid links route to the error path immediately.”

---

### Block 1.2 – Input Validation & Preparation  

**Overview** – Normalises raw form data: trims the link, fills defaults, clamps duration, defaults language. This guarantees that downstream nodes always receive a predictable payload.  

| Node | Type & Technical Role | Key Configuration | Expressions / Variables | Input | Output | Edge Cases | Version |
|------|----------------------|-------------------|------------------------|-------|--------|------------|---------|
| **2. Set — Validate and Prepare Inputs** | `n8n-nodes-base.set` – transforms incoming JSON into a clean, normalised payload | Five assignments: <br>• `meeting_link` = `{{ $json['Meeting Link'].trim() }}` <br>• `meeting_title` = `{{ $json['Meeting Title'] \|\| 'Sales Meeting - Incrementors' }}` <br>• `meeting_password` = `{{ $json['Meeting Password (if any)'] \|\| '' }}` <br>• `meeting_duration` = `{{ Math.min(Math.max(parseInt($json['Duration in Minutes']) \|\| 60, 15), 120) }}` <br>• `meeting_language` = `{{ $json['Meeting Language'] \|\| 'en' }}` | All values reference incoming form fields. Duration is parsed as integer, default 60, then clamped between 15 and 120. | `1. Form — Invite Fireflies Bot` | `3. IF — Valid Meeting Link?` | • Empty `Meeting Link` → results in empty `meeting_link` → caught by IF node <br>• Non‑numeric duration → `parseInt` yields `NaN` → fallback `60` <br>• Duration out of bounds → clamped <br>• Missing title/password/language → defaults applied | set v3.4 |

**Sticky note covering this node** – *Section — Form Input and Link Validation1* (same as above).

---

### Block 1.3 – Conditional Routing  

**Overview** – Ensures only valid meeting links proceed to the API call. Invalid links should be routed to an error branch (currently disconnected).  

| Node | Type & Technical Role | Key Configuration | Expressions / Variables | Input | Output | Edge Cases | Version |
|------|----------------------|-------------------|------------------------|-------|--------|------------|---------|
| **3. IF — Valid Meeting Link?** | `n8n-nodes-base.if` – conditional split | Two conditions combined with **AND**: <br>1. `{{ $json.meeting_link }}` **not equal** to empty string <br>2. `{{ $json.meeting_link.startsWith('http') }}` **equals** `true` | Uses cleaned `meeting_link` from previous Set node. | `2. Set — Validate and Prepare Inputs` | **True** → `4. HTTP — Send to Fireflies API` <br>**False** → currently **no node connected** (error path not wired) | • Empty link after trimming → false branch <br>• Link missing `http`/`https` → false branch <br>• Because the false branch is unconnected, users submitting invalid links receive no response (workflow ends silently). | if v2 |

**Sticky note covering this node** – *Section — Form Input and Link Validation1* (same as above).  

---

### Block 1.4 – Fireflies API Call  

**Overview** – Sends a GraphQL `addToLiveMeeting` mutation to Fireflies, passing the normalised meeting data. Successful API call results in the bot being invited to the meeting.  

| Node | Type & Technical Role | Key Configuration | Expressions / Variables | Input | Output | Edge Cases | Version |
|------|----------------------|-------------------|------------------------|-------|--------|------------|---------|
| **4. HTTP — Send to Fireflies API** | `n8n-nodes-base.httpRequest` – outbound HTTP POST to Fireflies GraphQL endpoint | • Method: `POST` <br>• URL: `https://api.fireflies.ai/graphql` <br>• Headers: <br>  - `Content-Type: application/json` <br>  - `Authorization: Bearer YOUR_TOKEN_HERE` (must be replaced with real Fireflies personal API key) <br>• Body (JSON, built with `specifyBody: json`): <br>```json<br>{<br>  "query": "mutation AddToLiveMeeting($meetingLink: String!, $title: String, $duration: Int, $meetingPassword: String, $language: String) { addToLiveMeeting(meeting_link: $meetingLink, title: $title, duration: $duration, meeting_password: $meetingPassword, language: $language) { success } }",<br>  "variables": {<br>    "meetingLink": "{{ $json.meeting_link }}",<br>    "title": "{{ $json.meeting_title }}",<br>    "duration": {{ $json.meeting_duration }},<br>    "meetingPassword": "{{ $json.meeting_password }}",<br>    "language": "{{ $json.meeting_language }}"<br>  }<br>}<br>``` | Uses the five normalised fields from node 2. The `duration` variable is injected as a raw number (no quotes) – it must be an integer. | `3. IF — Valid Meeting Link?` (true branch) | `5. Set — Build Success Response` | • Invalid/expired API key → 401/403 response, workflow continues with error payload (no error handling node) <br>• Fireflies API unreachable (network error) → node throws, workflow stops <br>• Meeting link points to unsupported platform → API may return `success: false` (no downstream handling) <br>• Password field ignored by Fireflies for platforms that don’t need it | httpRequest v4.2 |

**Sticky note covering this node** – *Section — Fireflies API Call and Success Response1*:  

> “## Fireflies API Call and Success Response  
> GraphQL mutation sends the meeting link to Fireflies. On success, a confirmation page is shown to the submitter with the meeting title and a reminder to check the Fireflies dashboard.”

**Additional note** – *Note — Fireflies API Key Required1* (positioned below the flow):  

> “## ⚠️ Fireflies API Key Required  
> Replace YOUR_FIREFLIES_API_KEY in the Authorization header. Get your key from app.fireflies.ai — go to Integrations, then API, then copy your Personal API Key.”

---

### Block 1.5 – Success Response Construction  

**Overview** – Generates a styled HTML confirmation card that references the normalized title and duration for immediate user feedback.  

| Node | Type & Technical Role | Key Configuration | Expressions / Variables | Input | Output | Edge Cases | Version |
|------|----------------------|-------------------|------------------------|-------|--------|------------|---------|
| **5. Set — Build Success Response** | `n8n-nodes-base.set` – builds a single string field containing HTML | • One assignment: `confirmationHtml` = <br>```html<br><div style="font-family:sans-serif;max-width:520px;margin:48px auto;padding:32px;border:1px solid #e0e0e0;border-radius:12px;text-align:center"><h2 style="color:#22c55e">&#9989; Fireflies Bot Invited</h2><p><strong>{{ $('2. Set — Validate and Prepare Inputs').item.json.meeting_title }}</strong></p><p>The bot will join your meeting automatically.<br>Duration: {{ $('2. Set — Validate and Prepare Inputs').item.json.meeting_duration }} minutes</p><p style="color:#888;font-size:13px">Transcript and summary will appear in your Fireflies dashboard after the call ends.</p></div><br>``` | Pulls `meeting_title` and `meeting_duration` from node 2 (using `$(` node reference). | `4. HTTP — Send to Fireflies API` | `6. Respond — Bot Invited Successfully` | • If node 2 is not executed (e.g., IF false branch) the expression will fail – not applicable because this node is only reached on the true branch. <br>• HTML injection risk: title is user‑provided text rendered inside HTML without escaping – in a trusted internal tool this is acceptable, but for public forms consider sanitising. | set v3.4 |

**Sticky note covering this node** – *Section — Fireflies API Call and Success Response1* (same as above).

---

### Block 1.6 – Success Response Delivery  

**Overview** – Returns the HTML confirmation page to the submitter’s browser, completing the webhook interaction.  

| Node | Type & Technical Role | Key Configuration | Expressions / Variables | Input | Output | Edge Cases | Version |
|------|----------------------|-------------------|------------------------|-------|--------|------------|---------|
| **6. Respond — Bot Invited Successfully** | `n8n-nodes-base.respondToWebhook` – sends the webhook response back to the browser | • Respond With: `html` <br>• Body (implicit from previous node’s `confirmationHtml` field) | No custom expressions – the node reads `confirmationHtml` from the incoming payload. | `5. Set — Build Success Response` | End of workflow (responds to client) | • If no `confirmationHtml` field exists, the node will return an empty response. <br>• If the form trigger’s `responseMode` is not `lastNode`, the response would be sent earlier – currently set correctly. | respondToWebhook v1.1 |

**Sticky note covering this node** – *Section — Fireflies API Call and Success Response1* (same as above).

---

### Planned but Missing – Error Response Path  

| Intended Node | Type | Intended Role | Current Status |
|---------------|------|---------------|----------------|
| **7. Set — Build Error Response** | `n8n-nodes-base.set` | Generate an HTML error message for invalid links (referenced in Overview sticky note) | **Not present** in the JSON. |
| **8. Respond — Invalid Link Error** | `n8n-nodes-base.respondToWebhook` | Return the error page to the submitter (referenced in Overview sticky note) | **Not present** in the JSON. |

**Impact** – The false branch of the IF node is unconnected, so invalid links result in a silent termination. To enable user‑visible error messages, add nodes 7 and 8 and connect the false output of the IF node to them.

---

## 3. Summary Table  

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|-----------|-----------|-----------------|----------------|-----------------|--------------|
| 1. Form — Invite Fireflies Bot | `n8n-nodes-base.formTrigger` | Collects meeting link, title, password, duration, language via hosted form | – | 2. Set — Validate and Prepare Inputs | Section — Form Input and Link Validation1: “## Form Input and Link Validation …” |
| 2. Set — Validate and Prepare Inputs | `n8n-nodes-base.set` | Normalises & defaults all fields, clamps duration | 1. Form — Invite Fireflies Bot | 3. IF — Valid Meeting Link? | Section — Form Input and Link Validation1 (same) |
| 3. IF — Valid Meeting Link? | `n8n-nodes-base.if` | Routes valid links to API call; invalid links should go to error (currently unconnected) | 2. Set — Validate and Prepare Inputs | True → 4. HTTP — Send to Fireflies API  <br> False → (none) | Section — Form Input and Link Validation1 (same) |
| 4. HTTP — Send to Fireflies API | `n8n-nodes-base.httpRequest` | Sends GraphQL mutation to Fireflies to invite bot | 3. IF — Valid Meeting Link? (true) | 5. Set — Build Success Response | Section — Fireflies API Call and Success Response1: “## Fireflies API Call and Success Response …” |
| 5. Set — Build Success Response | `n8n-nodes-base.set` | Builds HTML confirmation card | 4. HTTP — Send to Fireflies API | 6. Respond — Bot Invited Successfully | Section — Fireflies API Call and Success Response1 (same) |
| 6. Respond — Bot Invited Successfully | `n8n-nodes-base.respondToWebhook` | Returns confirmation HTML page to browser | 5. Set — Build Success Response | – (end) | Section — Fireflies API Call and Success Response1 (same) |
| Overview1 | `n8n-nodes-base.stickyNote` | Workflow overview & instructions | – | – | Overview1: “## Fireflies Bot Meeting Inviter — One‑Click Form …” |
| Section — Form Input and Link Validation1 | `n8n-nodes-base.stickyNote` | Explains Form + Validation block | – | – | Same as above (covers nodes 1‑3) |
| Section — Fireflies API Call and Success Response1 | `n8n-nodes-base.stickyNote` | Explains API + Response block | – | – | Same as above (covers nodes 4‑6) |
| Note — Fireflies API Key Required1 | `n8n-nodes-base.stickyNote` | Reminder to replace API key placeholder | – | – | “## ⚠️ Fireflies API Key Required …” |

---

## 4. Reproducing the Workflow from Scratch  

Follow these steps in a new n8n instance to recreate the workflow exactly.  

1. **Create the workflow**  
   - Open a new workflow.  
   - Name it *Invite the Fireflies recording bot to meetings from a one-click form*.  

2. **Add the Form Trigger node**  
   - Click **+** → *Form Trigger*.  
   - Configure:  
     - **Form Title**: `Invite Fireflies Bot to Sales Meeting`  
     - **Form Description**: `Enter your meeting details — the bot will automatically join and begin recording instantly.`  
     - **Response Mode**: `lastNode` (so the final Respond node sends HTML).  
     - **Form Fields** (add in this order):  
       1. **Meeting Link** – Type: Text, Required: ✅, Placeholder: `https://meet.google.com/xxx-xxxx-xxx`  
       2. **Meeting Title** – Type: Text, Required: ❌, Placeholder: `e.g. Sales Call - ClientName`  
       3. **Meeting Password (if any)** – Type: Text, Required: ❌, Placeholder: `Leave blank if no password`  
       4. **Duration in Minutes** – Type: Number, Required: ❌, Placeholder: `60 (min 15, max 120)`  
       5. **Meeting Language** – Type: Dropdown, Options: `en`, `hi`, `es`, `fr`, `pt`  
   - Leave the Webhook Path unchanged (auto‑generated).  
   - Version: formTrigger v2.2.  

3. **Add the Set node – Validate and Prepare Inputs**  
   - Click **+** → *Set*.  
   - Rename: `2. Set — Validate and Prepare Inputs`.  
   - Under *Assignments*, add:  

     | Name | Type | Value |
     |------|------|-------|
     | `meeting_link` | String | `={{ $json['Meeting Link'].trim() }}` |
     | `meeting_title` | String | `={{ $json['Meeting Title'] || 'Sales Meeting - Incrementors' }}` |
     | `meeting_password` | String | `={{ $json['Meeting Password (if any)'] || '' }}` |
     | `meeting_duration` | Number | `={{ Math.min(Math.max(parseInt($json['Duration in Minutes']) || 60, 15), 120) }}` |
     | `meeting_language` | String | `={{ $json['Meeting Language'] || 'en' }}` |

   - Version: set v3.4.  

4. **Add the IF node – Valid Meeting Link?**  
   - Click **+** → *IF*.  
   - Rename: `3. IF — Valid Meeting Link?`.  
   - Conditions (Combinator: **AND**):  

     1. Left value: `={{ $json.meeting_link }}`  <br> Operator: **does not equal**  <br> Right value: *(empty string)*  
     2. Left value: `={{ $json.meeting_link.startsWith('http') }}` <br> Operator: **equals** <br> Right value: `true`  

   - Version: if v2.  

5. **Add the HTTP Request node – Send to Fireflies API**  
   - Click **+** → *HTTP Request*.  
   - Rename: `4. HTTP — Send to Fireflies API`.  
   - Configuration:  

     - **Method**: `POST`  
     - **URL**: `https://api.fireflies.ai/graphql`  
     - **Authentication**: None (header‑based token).  
     - **Headers**: Add two headers:  

       - `Content-Type`: `application/json`  
       - `Authorization`: `Bearer YOUR_FIREFLIES_API_KEY` (replace `YOUR_FIREFLIES_API_KEY` with your real key)  

     - **Body Specification**: Choose *JSON* (`specifyBody: json`).  
     - **JSON Body** (paste as expression):  

       ```json
       ={
         "query": "mutation AddToLiveMeeting($meetingLink: String!, $title: String, $duration: Int, $meetingPassword: String, $language: String) { addToLiveMeeting(meeting_link: $meetingLink, title: $title, duration: $duration, meeting_password: $meetingPassword, language: $language) { success } }",
         "variables": {
           "meetingLink": "{{ $json.meeting_link }}",
           "title": "{{ $json.meeting_title }}",
           "duration": {{ $json.meeting_duration }},
           "meetingPassword": "{{ $json.meeting_password }}",
           "language": "{{ $json.meeting_language }}"
         }
       }
       ```

   - Version: httpRequest v4.2.  

6. **Add the Set node – Build Success Response**  
   - Click **+** → *Set*.  
   - Rename: `5. Set — Build Success Response`.  
   - Add a single assignment:  

     | Name | Type | Value |
     |------|------|-------|
     | `confirmationHtml` | String | (paste the HTML expression below) |

     Expression (use *Expression* toggle):

     ```html
     <div style="font-family:sans-serif;max-width:520px;margin:48px auto;padding:32px;border:1px solid #e0e0e0;border-radius:12px;text-align:center">
       <h2 style="color:#22c55e">&#9989; Fireflies Bot Invited</h2>
       <p><strong>{{ $('2. Set — Validate and Prepare Inputs').item.json.meeting_title }}</strong></p>
       <p>The bot will join your meeting automatically.<br>Duration: {{ $('2. Set — Validate and Prepare Inputs').item.json.meeting_duration }} minutes</p>
       <p style="color:#888;font-size:13px">Transcript and summary will appear in your Fireflies dashboard after the call ends.</p>
     </div>
     ```

   - Version: set v3.4.  

7. **Add the Respond to Webhook node – Bot Invited Successfully**  
   - Click **+** → *Respond to Webhook*.  
   - Rename: `6. Respond — Bot Invited Successfully`.  
   - Configuration:  

     - **Respond With**: `html`  
     - The node automatically sends the `confirmationHtml` field from the incoming payload.  

   - Version: respondToWebhook v1.1.  

8. **Wire the nodes** (drag connections):  

   - `1. Form — Invite Fireflies Bot` → `2. Set — Validate and Prepare Inputs`  
   - `2. Set — Validate and Prepare Inputs` → `3. IF — Valid Meeting Link?`  
   - `3. IF — Valid Meeting Link?` **true** output → `4. HTTP — Send to Fireflies API`  
   - `4. HTTP — Send to Fireflies API` → `5. Set — Build Success Response`  
   - `5. Set — Build Success Response` → `6. Respond — Bot Invited Successfully`  

   - *False* output of the IF node: **currently unconnected**. To add error handling, create a Set node (`7. Set — Build Error Response`) and a Respond to Webhook node (`8. Respond — Invalid Link Error`) and connect the false path to them.  

9. **Configure Credentials**  

   - The HTTP node uses an **Authorization header** with a static token. There is no n8n credential object; simply replace the placeholder `YOUR_FIREFLIES_API_KEY` in the header value with the real key obtained from **app.fireflies.ai → Integrations → API → Personal API Key**.  
   - No other credentials are required (the Form Trigger uses n8n’s built‑in webhook).  

10. **Activate the workflow**  

    - Click **Activate** (top‑right).  
    - Copy the **Form URL** from node 1 (displayed in the node’s “Webhook URL” field). Bookmark this URL or share it with teammates.  

11. **Optional customisations**  

    - **Default meeting title** – Edit node 2 assignment `meeting_title` to change `Sales Meeting - Incrementors` to your preferred default.  
    - **Language list** – In node 1’s dropdown, add additional language codes (e.g., `de`, `it`, `ja`).  
    - **Maximum duration** – Adjust the `120` value in node 2’s `meeting_duration` expression if you need longer meetings.  
    - **Error handling** – Create nodes 7 and 8 (Set + Respond) and connect the IF false output to display an HTML error page.  

---

## 5. General Notes & Resources  

| Note Content | Context or Link |
|--------------|-----------------|
| **Fireflies API documentation** – GraphQL endpoint `https://api.fireflies.ai/graphql`, mutation `addToLiveMeeting`. | https://developers.fireflies.ai/ (or the internal Fireflies API docs) |
| **Fireflies dashboard** – where transcripts and meeting summaries appear after the call ends. | https://app.fireflies.ai |
| **Author support** – Need help or a custom version for your team/agency? | Email: info@isawow.com · Website: https://isawow.com/ |
| **HTML injection warning** – The `meeting_title` from user input is rendered directly inside the HTML confirmation page without escaping. For a public‑facing form, consider sanitising the title to prevent XSS. | N/A |
| **Error path** – Nodes 7 and 8 are described in the workflow overview sticky note but are **not present** in the JSON. Connect the false branch of the IF node to custom Set + Respond nodes to surface a friendly error page. | N/A |
| **Workflow version compatibility** – Node versions used: FormTrigger v2.2, Set v3.4, IF v2, HTTP Request v4.2, Respond to Webhook v1.1. Ensure your n8n instance supports these versions. | N/A |
| **Sticky note: Overview1** – Full overview and setup steps for the workflow. | N/A |
| **Sticky note: Note — Fireflies API Key Required1** – Reminds you to replace the placeholder token. | N/A |

--- 

*End of reference document.*