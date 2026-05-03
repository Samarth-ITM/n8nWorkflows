Get AI meeting briefs from Google Calendar with SerpAPI, Azure OpenAI and Slack

https://n8nworkflows.xyz/workflows/get-ai-meeting-briefs-from-google-calendar-with-serpapi--azure-openai-and-slack-13519


# Get AI meeting briefs from Google Calendar with SerpAPI, Azure OpenAI and Slack

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensif ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title (given):** Get AI meeting briefs from Google Calendar with SerpAPI, Azure OpenAI and Slack  
**Workflow name (in JSON):** Who Am I Meeting - Research Assistant  
**Purpose:** Monitor Google Calendar for newly created events, extract attendee/company info, run parallel web research via SerpAPI, generate an AI meeting brief using Azure OpenAI (LangChain Agent), and deliver the brief via Slack DM.

### Logical blocks
1.1 **Event Detection & Filtering**  
Detect new events and ignore events without attendees.

1.2 **Attendee/Company Data Extraction**  
Normalize event data into `person.name` and `company.name` used by search queries.

1.3 **Parallel Web Research (SerpAPI)**  
Run two Google searches in parallel (person + company) and post-process results.

1.4 **AI Briefing Generation (Azure OpenAI Agent)**  
Merge and prepare research context, then produce a concise meeting brief.

1.5 **Slack Delivery**  
Format output into a Slack-ready message and send it via DM.

---

## 2. Block-by-Block Analysis

### 2.1 Block: Event Detection & Filtering
**Overview:** Polls Google Calendar every minute for newly created events. Filters out events that do not contain attendees before doing any downstream work.

**Nodes involved:**
- New Calendar Event Trigger
- Filter: Has Attendees

#### Node: New Calendar Event Trigger
- **Type / role:** `googleCalendarTrigger` — polling trigger for new Google Calendar events.
- **Key configuration:**
  - Trigger: **eventCreated**
  - Poll schedule: **every minute**
  - Calendar ID: `user@example.com` (selected from list)
- **Inputs/outputs:**
  - **Input:** none (trigger)
  - **Output:** event payload to **Filter: Has Attendees**
- **Credentials:** Google Calendar OAuth2 (`Google Calendar account -anuj`)
- **Version notes:** v1 trigger node. Polling frequency can be constrained by n8n instance settings and Google API quotas.
- **Edge cases / failures:**
  - OAuth token expiration / revoked consent → authentication failures.
  - Calendar ID mismatch or lack of permissions → empty results or API errors.
  - High event creation volume → frequent executions and potential quota/rate limiting.

#### Node: Filter: Has Attendees
- **Type / role:** `if` — gate to ensure only events with attendees proceed.
- **Key configuration:**
  - Condition uses expression: `={{ $json.attendees.length }}`
  - Operation: `isNotEmpty`
- **Inputs/outputs:**
  - **Input:** event JSON from trigger
  - **True output:** to **Extract Attendee & Company Info**
  - **False output:** not connected (events dropped)
- **Version notes:** IF node v1.
- **Edge cases / failures:**
  - If `attendees` is missing (not just empty), `$json.attendees.length` can error. Safer pattern would be something like `{{$json.attendees?.length || 0}}` and compare numerically.
  - Some calendar events represent attendees differently (organizer vs attendees), causing false negatives.

---

### 2.2 Block: Attendee/Company Data Extraction
**Overview:** Transforms the event payload into a normalized structure (`person.name`, `company.name`) used by the two SerpAPI queries.

**Nodes involved:**
- Extract Attendee & Company Info

#### Node: Extract Attendee & Company Info
- **Type / role:** `set` — constructs/reshapes fields for downstream nodes.
- **Key configuration:**
  - The node has **no explicit field mappings in the JSON** (only `options:{}`), meaning it currently does not create `person.name` / `company.name` by itself.
  - However, downstream nodes expect:
    - `{{$json.person.name}}`
    - `{{$json.company.name}}`
- **Inputs/outputs:**
  - **Input:** filtered calendar event
  - **Output:** splits to both:
    - **Search Person on Google**
    - **Search Company on Google**
- **Version notes:** Set node v2.
- **Edge cases / failures:**
  - As configured, this node likely leaves data unchanged; if the incoming event does not already contain `person.name` and `company.name`, both SerpAPI nodes will search for `undefined` / empty queries.
  - Multi-attendee events: you must decide which attendee to research (first external attendee, all attendees, exclude your own domain, etc.).

---

### 2.3 Block: Parallel Web Research (SerpAPI)
**Overview:** Performs two parallel SerpAPI Google searches (person and company). Each result set is then “parsed” by a Code node (currently placeholder logic).

**Nodes involved:**
- Search Person on Google
- Parse Person Search Results
- Search Company on Google
- Parse Company Search Results
- Merge Person & Company Data

#### Node: Search Person on Google
- **Type / role:** `serpApi` — executes a SerpAPI query for the person.
- **Key configuration:**
  - Query (`q`): `={{ $json.person.name }}`
- **Inputs/outputs:**
  - **Input:** output of Extract node
  - **Output:** to **Parse Person Search Results**
- **Credentials:** SerpAPI (`SerpApi account Vivek`)
- **Version notes:** node v1.
- **Edge cases / failures:**
  - Missing/invalid API key → 401/403 errors.
  - Rate limits / quota exhaustion (explicitly warned in sticky note).
  - Empty or low-quality query string → irrelevant results.

#### Node: Parse Person Search Results
- **Type / role:** `code` — post-process person search results.
- **Key configuration:**
  - Current JS code is placeholder; it sets `item.json.myNewField = 1` for every item and returns inputs unchanged otherwise.
- **Inputs/outputs:**
  - **Input:** SerpAPI results
  - **Output:** to **Merge Person & Company Data** (Input 1)
- **Version notes:** Code node v2.
- **Edge cases / failures:**
  - If SerpAPI returns nested structures, you likely want to extract/clean fields (titles, snippets, links). Current code does not.
  - Large result payloads can increase memory usage; consider trimming.

#### Node: Search Company on Google
- **Type / role:** `serpApi` — executes a SerpAPI query for the company.
- **Key configuration:**
  - Query (`q`): `={{ $json.company.name }}`
- **Inputs/outputs:**
  - **Input:** output of Extract node
  - **Output:** to **Parse Company Search Results**
- **Credentials:** SerpAPI (`SerpApi account Vivek`)
- **Edge cases / failures:** same as person search.

#### Node: Parse Company Search Results
- **Type / role:** `code` — post-process company search results.
- **Key configuration:** placeholder logic identical to person parse node (`myNewField = 1`).
- **Inputs/outputs:**
  - **Input:** SerpAPI results
  - **Output:** to **Merge Person & Company Data** (Input 2)
- **Edge cases / failures:** same as person parse node.

#### Node: Merge Person & Company Data
- **Type / role:** `merge` — combines the two research branches.
- **Key configuration:**
  - No explicit mode shown; Merge node defaults vary by version/config. With v3.2, typical modes include “Combine”, “Merge By Index”, etc. Here, configuration is empty, so behavior depends on default in your n8n version/UI state.
- **Inputs/outputs:**
  - **Input 1:** Parse Person Search Results
  - **Input 2:** Parse Company Search Results
  - **Output:** to **Prepare Combined Research Data**
- **Version notes:** Merge node v3.2.
- **Edge cases / failures:**
  - If one branch returns 0 items and the other returns items, output behavior depends on merge mode (may output nothing).
  - If both branches return different item counts, index-based merging may mismatch.

---

### 2.4 Block: AI Briefing Generation
**Overview:** Prepares a combined context payload and asks an Azure OpenAI-backed LangChain Agent to generate a meeting brief.

**Nodes involved:**
- Prepare Combined Research Data
- Azure OpenAI Chat Model1
- AI Meeting Briefing Agent

#### Node: Prepare Combined Research Data
- **Type / role:** `code` — prepares final prompt/context input.
- **Key configuration:**
  - Placeholder JS sets `myNewField = 1` and returns items. No real shaping/summarization occurs.
- **Inputs/outputs:**
  - **Input:** Merge Person & Company Data
  - **Output:** AI Meeting Briefing Agent
- **Version notes:** Code node v2.
- **Edge cases / failures:**
  - If you intend to pass a single combined text blob into the agent, you likely need to concatenate/serialize search results into a manageable context window.
  - Oversized context can cause LLM token limit issues or increased cost/latency.

#### Node: Azure OpenAI Chat Model1
- **Type / role:** `lmChatAzureOpenAi` — provides the language model backend for the agent.
- **Key configuration:**
  - Model deployment: `gpt-4o-mini`
- **Connections:**
  - Connected to the agent via **AI languageModel** channel.
- **Credentials:** Azure OpenAI (`Azure Open AI account`)
- **Version notes:** v1. Requires Azure OpenAI resource + deployment name matching `model`.
- **Edge cases / failures:**
  - Wrong deployment name, region, or API version → request failure.
  - Azure content filtering or policy blocks → partial/empty responses.
  - Network/timeout issues under heavy load.

#### Node: AI Meeting Briefing Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — generates the briefing text.
- **Key configuration:**
  - Prompt type: `define`
  - Prompt text: `Summarise the company and the profile name`
  - Uses Azure OpenAI Chat Model via `ai_languageModel` connection.
- **Inputs/outputs:**
  - **Main input:** combined research data from Prepare Combined Research Data
  - **Main output:** to Format Slack Briefing Message
- **Version notes:** Agent node v3 (LangChain integration).
- **Edge cases / failures:**
  - If upstream data is not mapped into the agent’s expected input fields, the model may only see minimal context.
  - If multiple items are passed, agent behavior may run per-item; ensure you understand item batching and resulting Slack messages count.
- **Sub-workflow reference:** none.

---

### 2.5 Block: Slack Delivery
**Overview:** Converts the AI output into a Slack-ready text field and sends it as a DM.

**Nodes involved:**
- Format Slack Briefing Message
- Send Briefing via Slack DM

#### Node: Format Slack Briefing Message
- **Type / role:** `set` — prepares a `text` field used by Slack node.
- **Key configuration:**
  - No explicit mappings present in JSON; likely should map agent output into `$json.text`.
- **Inputs/outputs:**
  - **Input:** AI Meeting Briefing Agent output
  - **Output:** Send Briefing via Slack DM
- **Version notes:** Set node v2.
- **Edge cases / failures:**
  - Slack node uses `={{ $json.text }}`; if this Set node does not create `text`, message will be empty or expression will resolve to undefined.
  - Consider adding formatting (meeting title/time, attendee, bullet points).

#### Node: Send Briefing via Slack DM
- **Type / role:** `slack` — sends a message to a Slack channel/user.
- **Key configuration:**
  - Authentication: OAuth2
  - Channel: `YOUR_SLACK_USER_ID` (placeholder)
  - Text: `={{ $json.text }}`
- **Inputs/outputs:**
  - **Input:** Slack message payload from Format node
  - **Output:** end
- **Credentials:** Slack OAuth2 (`Slack account 5`)
- **Version notes:** Slack node v1.
- **Edge cases / failures:**
  - Placeholder user ID causes delivery failure (explicitly warned in sticky note). Some failures can appear “silent” depending on Slack API response handling/logging.
  - Missing scopes (e.g., `chat:write`, possibly `im:write`) can prevent DM sending.
  - If using user IDs, ensure the app can open/send to IM channel; in some setups you must use `conversations.open` first.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| 📋 Overview | stickyNote | Documentation / overview | — | — | ## 🔍 Who Am I Meeting — Research Assistant / How it works / Setup steps / Customization (contains SerpAPI/Azure/Slack setup guidance) |
| Section: Event Detection & Filtering | stickyNote | Documentation for event block | — | — | ## 📅 Event Detection & Filtering Polls Google Calendar every minute... |
| Section: Parallel Web Research | stickyNote | Documentation for research block | — | — | ## 🔎 Parallel Web Research Runs two SerpAPI Google searches simultaneously... |
| Section: AI Briefing Generation | stickyNote | Documentation for AI block | — | — | ## 🤖 AI Briefing Generation Merges person and company research data... |
| Section: Slack Delivery | stickyNote | Documentation for Slack block | — | — | ## 📨 Slack Delivery Formats the AI-generated brief into a Slack message... |
| ⚠️ Warning: SerpAPI | stickyNote | Warning about SerpAPI creds/limits | — | — | ⚠️ **SerpAPI Credentials Required** Both search nodes require a valid SerpAPI key... |
| ⚠️ Warning: Slack DM Config | stickyNote | Warning about Slack user/channel | — | — | ⚠️ **Slack Channel Required** Replace `YOUR_SLACK_USER_ID` with a real Slack member ID... |
| New Calendar Event Trigger | googleCalendarTrigger | Detect new calendar events | — | Filter: Has Attendees | ## 📅 Event Detection & Filtering Polls Google Calendar every minute... |
| Filter: Has Attendees | if | Skip events with no attendees | New Calendar Event Trigger | Extract Attendee & Company Info | ## 📅 Event Detection & Filtering Polls Google Calendar every minute... |
| Extract Attendee & Company Info | set | Normalize attendee/company fields | Filter: Has Attendees | Search Person on Google; Search Company on Google | ## 📅 Event Detection & Filtering Polls Google Calendar every minute... |
| Search Person on Google | serpApi | Person research query | Extract Attendee & Company Info | Parse Person Search Results | ## 🔎 Parallel Web Research Runs two SerpAPI Google searches simultaneously... / ⚠️ **SerpAPI Credentials Required**... |
| Search Company on Google | serpApi | Company research query | Extract Attendee & Company Info | Parse Company Search Results | ## 🔎 Parallel Web Research Runs two SerpAPI Google searches simultaneously... / ⚠️ **SerpAPI Credentials Required**... |
| Parse Person Search Results | code | Clean/shape person results (placeholder) | Search Person on Google | Merge Person & Company Data | ## 🔎 Parallel Web Research Runs two SerpAPI Google searches simultaneously... |
| Parse Company Search Results | code | Clean/shape company results (placeholder) | Search Company on Google | Merge Person & Company Data | ## 🔎 Parallel Web Research Runs two SerpAPI Google searches simultaneously... |
| Merge Person & Company Data | merge | Combine both research streams | Parse Person Search Results; Parse Company Search Results | Prepare Combined Research Data | ## 🤖 AI Briefing Generation Merges person and company research data... |
| Prepare Combined Research Data | code | Build combined AI input (placeholder) | Merge Person & Company Data | AI Meeting Briefing Agent | ## 🤖 AI Briefing Generation Merges person and company research data... |
| AI Meeting Briefing Agent | langchain.agent | Generate meeting brief | Prepare Combined Research Data | Format Slack Briefing Message | ## 🤖 AI Briefing Generation Merges person and company research data... |
| Azure OpenAI Chat Model1 | lmChatAzureOpenAi | LLM backend for agent | — | AI Meeting Briefing Agent (ai_languageModel) | ## 🤖 AI Briefing Generation Merges person and company research data... |
| Format Slack Briefing Message | set | Map AI output to `text` | AI Meeting Briefing Agent | Send Briefing via Slack DM | ## 📨 Slack Delivery Formats the AI-generated brief... |
| Send Briefing via Slack DM | slack | Send DM with the brief | Format Slack Briefing Message | — | ## 📨 Slack Delivery Formats the AI-generated brief... / ⚠️ **Slack Channel Required** Replace `YOUR_SLACK_USER_ID`... |

---

## 4. Reproducing the Workflow from Scratch

1) **Create workflow**
- Name it: **Who Am I Meeting - Research Assistant** (or your preferred name).
- Ensure workflow setting **Execution Order** is set to **v1** (matches JSON).

2) **Add Google Calendar trigger**
- Add node: **Google Calendar Trigger**
- Configure:
  - Trigger event: **Event Created**
  - Poll interval: **Every Minute**
  - Calendar: select your target calendar (e.g., your email)
- Credentials:
  - Create/attach **Google Calendar OAuth2** credentials with access to the calendar.

3) **Add attendee filter**
- Add node: **IF**
- Condition (String):
  - Value 1 expression: `{{ $json.attendees.length }}`
  - Operation: **isNotEmpty**
- Connect: **Google Calendar Trigger → IF**

4) **Add extraction/normalization node**
- Add node: **Set** (v2)
- Configure it to output at least:
  - `person.name` (string)
  - `company.name` (string)
- Example mapping approach (you must adapt to your calendar payload):
  - `person.name`: from the first attendee display name or email local-part
  - `company.name`: parse from attendee email domain, or from event title/description
- Connect: **IF (true) → Set**

5) **Add parallel SerpAPI searches**
- Add node: **SerpAPI**
  - Name: **Search Person on Google**
  - Query `q`: `{{ $json.person.name }}`
  - Attach **SerpAPI credentials** (API key).
- Add another **SerpAPI**
  - Name: **Search Company on Google**
  - Query `q`: `{{ $json.company.name }}`
  - Attach same SerpAPI credentials.
- Connect the Set node to both SerpAPI nodes (two outgoing connections).

6) **Add parsing Code nodes (person + company)**
- Add node: **Code** (v2) after each SerpAPI node:
  - **Parse Person Search Results** connected from **Search Person on Google**
  - **Parse Company Search Results** connected from **Search Company on Google**
- Initially you can keep placeholder code, but for a working brief you should:
  - Extract top N organic results (title/snippet/link)
  - Reduce payload size
  - Produce fields like `personResearch` and `companyResearch`

7) **Merge the two branches**
- Add node: **Merge** (v3.2)
- Connect:
  - Parse Person Search Results → Merge (Input 1)
  - Parse Company Search Results → Merge (Input 2)
- Set merge mode explicitly in UI (recommended) to avoid default ambiguity (e.g., “Merge By Index” if both return 1 item each).

8) **Prepare combined research payload**
- Add node: **Code** (v2) named **Prepare Combined Research Data**
- Build a single structured object for the agent, e.g.:
  - `person`: `{ name, researchSummary/links }`
  - `company`: `{ name, researchSummary/links }`
  - `event`: `{ title, start, attendees }` (optional but useful)
- Connect: **Merge → Prepare Combined Research Data**

9) **Add Azure OpenAI Chat Model**
- Add node: **Azure OpenAI Chat Model** (LangChain)
- Configure:
  - Deployment/model name: **gpt-4o-mini** (must match your Azure deployment)
- Credentials:
  - Azure OpenAI API key/endpoint credentials in n8n.

10) **Add the LangChain Agent**
- Add node: **AI Agent** (LangChain Agent)
- Configure:
  - Prompt type: **Define**
  - Prompt text: similar to: “Summarise the company and the profile name”
  - (Recommended) Expand prompt to reference the structured fields you created.
- Connect:
  - **Prepare Combined Research Data → AI Agent** (main)
  - **Azure OpenAI Chat Model → AI Agent** via **AI languageModel** connection.

11) **Format Slack message**
- Add node: **Set** (v2) named **Format Slack Briefing Message**
- Map agent output into `text` (required by Slack node), for example:
  - `text`: use the agent’s generated text field (depends on agent output schema in your n8n version).
- Connect: **AI Agent → Format Slack Briefing Message**

12) **Send Slack DM**
- Add node: **Slack**
- Operation: send message (default “Post” behavior)
- Configure:
  - Text: `{{ $json.text }}`
  - Channel: replace `YOUR_SLACK_USER_ID` with a real member ID like `U0XXXXXXX`
  - Authentication: **OAuth2**
- Credentials:
  - Slack OAuth2 with scopes allowing posting messages/DMs.
- Connect: **Format Slack Briefing Message → Slack**

13) **(Optional but recommended) Add sticky notes**
- Add sticky notes mirroring the sections/warnings from the workflow to document setup and pitfalls.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow researches people you are about to meet by monitoring Google Calendar for newly created events, running parallel SerpAPI searches, generating a brief with Azure OpenAI, and sending it via Slack DM. | From “📋 Overview” sticky note |
| Setup requires: Google Calendar OAuth2, SerpAPI key (both search nodes), Azure OpenAI credentials (deployment `gpt-4o-mini`), Slack OAuth2, and replacing `YOUR_SLACK_USER_ID`. | From “📋 Overview” + Slack warning note |
| SerpAPI rate limits apply; high-volume calendars can exhaust monthly quota quickly. | From “⚠️ Warning: SerpAPI” sticky note |
| Slack DM requires a real Slack member ID (format `U0XXXXXXX`); leaving placeholder may fail delivery. | From “⚠️ Warning: Slack DM Config” sticky note |

