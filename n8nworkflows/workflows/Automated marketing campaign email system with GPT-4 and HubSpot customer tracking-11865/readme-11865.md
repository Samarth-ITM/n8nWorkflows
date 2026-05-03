Automated marketing campaign email system with GPT-4 and HubSpot customer tracking

https://n8nworkflows.xyz/workflows/automated-marketing-campaign-email-system-with-gpt-4-and-hubspot-customer-tracking-11865


# Automated marketing campaign email system with GPT-4 and HubSpot customer tracking

## 1. Workflow Overview

**Purpose:** This workflow automates a marketing email campaign system driven by GPT-4-class models, using Google Sheets as a campaign/customer source of truth, HubSpot for customer tracking/CRM synchronization, Gmail for outbound and reply handling, and Google Calendar for event creation based on inbound replies.

**Target use cases:**
- Running recurring outbound campaign emails to selected customers
- Logging outbound/inbound message history for personalization and compliance
- Detecting and categorizing customer replies (e.g., sentiment/intent)
- Creating calendar events when replies indicate meetings or scheduling intent
- Synchronizing customer records with HubSpot

**Logical blocks (based on node-to-node dependencies):**
1.1 **Campaign & Customer Intake (Hourly)** — pulls customers/campaigns from Google Sheets, filters, enriches customer info, and creates HubSpot contacts if needed.  
1.2 **Outbound Campaign Execution (Daily)** — skips weekends, selects eligible customers/campaign steps, fetches history, generates email copy via an AI Agent, sends Gmail, logs history, and updates campaign state back to Sheets.  
1.3 **Inbound Reply Processing (Gmail Trigger)** — on incoming email, pulls message history, stores reply, classifies (sentiment/category), optionally creates calendar events, generates AI reply, sends reply, and logs it.

> Note: Many nodes have empty `parameters` in the provided export, so the document describes the **intended configuration** inferred from node types, names, and connections. You will need to fill in concrete resource IDs (Sheet IDs, Data Table names, HubSpot object fields, Gmail labels, etc.) when rebuilding.

---

## 2. Block-by-Block Analysis

### 1.1 Campaign & Customer Intake (Hourly)

**Overview:** Runs every hour to pull customers and campaign definitions from Google Sheets. Filters the customer list, constructs a customer payload, uses an AI Agent to normalize/prepare the data, converts it to a JSON object, creates the customer in HubSpot, and updates the originating sheet row.

**Nodes involved:**
- Every hour
- Get customers
- Filter
- Customer information
- AI Agent1
- OpenAI Model1
- Convert to JSON object
- Create new Customer
- Update row in sheet1
- Get campaign
- If campaign does not exist1
- Insert new campaign

#### Node details

**Every hour**
- **Type/role:** Schedule Trigger; entry point for hourly sync.
- **Configuration (intended):** Runs hourly (cron-like schedule; exact interval not shown).
- **Outputs:** Fan-out to **Get customers** and **Get campaign**.
- **Edge cases:** Workflow concurrency if processing is long; consider enabling execution limits.

**Get customers**
- **Type/role:** Google Sheets node; reads customer rows.
- **Configuration (intended):** “Read” operation from a Customers sheet/range.
- **Input:** Every hour.
- **Output:** Rows representing customers.
- **Failures:** OAuth/permission errors; sheet/range not found; quota limits.

**Filter**
- **Type/role:** Filter node; keeps only customers matching criteria.
- **Configuration (intended):** Conditions such as “opt-in = true”, “email not empty”, “not already synced”, etc. (not provided).
- **Input:** Get customers.
- **Output:** Filtered customers.
- **Edge cases:** If expressions reference missing columns, the node can evaluate to false or error depending on setup.

**Customer information**
- **Type/role:** Code (JavaScript); transforms sheet row into a structured customer object.
- **Configuration (intended):** Build fields like `email`, `firstname`, `lastname`, `company`, `persona`, `campaignStage`, etc.
- **Input:** Filter.
- **Output:** Structured JSON per customer.
- **Failures:** JS runtime errors when accessing undefined properties; inconsistent sheet columns.

**AI Agent1**
- **Type/role:** LangChain Agent; AI-assisted normalization/enrichment of customer payload (or mapping to HubSpot schema).
- **Configuration (intended):** Prompt/tools not shown. Uses **OpenAI Model1** as language model input via the `ai_languageModel` connection.
- **Input:** Customer information.
- **Output:** AI-produced content/fields, likely including HubSpot-ready properties.
- **Failures:** Model auth issues; token limits; inconsistent output format.

**OpenAI Model1**
- **Type/role:** `lmChatOpenAi`; provides chat model to AI Agent1.
- **Configuration (intended):** Model selection (e.g., GPT-4/4.1), temperature, max tokens (not provided).
- **Connection:** Feeds AI Agent1 through `ai_languageModel`.
- **Failures:** API key invalid; model not available; rate limits.

**Convert to JSON object**
- **Type/role:** Code (JavaScript); parses AI output into a strict JSON object for HubSpot creation.
- **Configuration (intended):** Extract JSON from AI text, validate required fields.
- **Input:** AI Agent1.
- **Output:** JSON object for HubSpot node.
- **Edge cases:** AI returns non-JSON or partial JSON; parsing failures; missing required properties.

**Create new Customer**
- **Type/role:** HubSpot node; creates a contact/customer record.
- **Configuration (intended):** “Create Contact” with properties mapped from the parsed JSON; dedupe by email if supported.
- **Input:** Convert to JSON object.
- **Output:** HubSpot contact record/ID.
- **Failures:** OAuth/private app token errors; validation errors; duplicate contact conflicts.

**Update row in sheet1**
- **Type/role:** Google Sheets node; updates customer row after HubSpot creation.
- **Configuration (intended):** Write back `hubspotId`, `syncedAt`, status flags.
- **Input:** Create new Customer.
- **Failures:** Sheet write permissions; row lookup mismatch.

**Get campaign**
- **Type/role:** Google Sheets node; reads campaign definitions.
- **Configuration (intended):** Read “Campaigns” sheet/range.
- **Input:** Every hour.
- **Output:** Campaign rows.
- **Failures:** Same as other Sheets nodes.

**If campaign does not exist1** (Data Table node, despite name)
- **Type/role:** Data Table node; appears to be used as an existence check/cache.
- **Configuration (intended):** Lookup campaign by key (campaignId/name) to determine if already present in internal Data Table storage.
- **Input:** Get campaign.
- **Output:** To Insert new campaign if missing (single output shown).
- **Edge cases:** If lookup keys don’t match; if Data Table empty; if configured incorrectly, could insert duplicates.

**Insert new campaign**
- **Type/role:** Data Table node; inserts campaign records.
- **Configuration (intended):** Store campaign metadata and step definitions.
- **Input:** If campaign does not exist1.
- **Output:** No downstream connections shown.
- **Failures:** Data Table not configured/created; schema mismatch.

---

### 1.2 Outbound Campaign Execution (Daily)

**Overview:** Runs daily, avoids weekends, selects new campaign targets, finds the matching HubSpot customer, filters to eligible recipients, retrieves message history, prepares prompt context in JS, uses an AI Agent to generate the email content, sends via Gmail, stores message history, and updates campaign progression and/or Google Sheets.

**Nodes involved:**
- Daily
- Don’t email on weekends
- Get new campaign
- Get customer (HubSpot)
- Filter customer
- Get histories
- Code in JavaScript
- AI Agent
- OpenAI Model
- Convert to JSON object1
- Send a message
- Insert message history
- Update Campaign
- Update row in sheet

#### Node details

**Daily**
- **Type/role:** Schedule Trigger; entry point for daily send flow.
- **Configuration (intended):** Daily at a fixed hour (not shown).
- **Output:** Don’t email on weekends.
- **Edge cases:** Ensure timezone matches business requirements.

**Don’t email on weekends**
- **Type/role:** Filter node; blocks Saturday/Sunday execution.
- **Configuration (intended):** `dayOfWeek` check via n8n expression.
- **Input:** Daily.
- **Output:** Get new campaign.
- **Edge cases:** Timezone; holiday handling is not included.

**Get new campaign**
- **Type/role:** Data Table node; retrieves campaign instances/next steps to send.
- **Configuration (intended):** Query campaigns/customers due for outreach today.
- **Input:** Don’t email on weekends.
- **Output:** Get customer (HubSpot).
- **Failures:** Data Table misconfiguration; empty results (should safely end).

**Get customer** (HubSpot)
- **Type/role:** HubSpot node; fetch contact data for personalization and validation.
- **Configuration (intended):** “Get Contact by ID/email” based on Data Table campaign target.
- **Input:** Get new campaign.
- **Output:** Filter customer.
- **Failures:** Auth; contact not found; API limits.

**Filter customer**
- **Type/role:** Filter node; ensures customer is eligible (opt-in, has email, not bounced, not already contacted today, etc.).
- **Input:** Get customer.
- **Output:** Get histories.
- **Edge cases:** Missing properties; inconsistent HubSpot fields.

**Get histories**
- **Type/role:** Data Table node; fetches prior message history for the customer/campaign.
- **Input:** Filter customer.
- **Output:** Code in JavaScript.
- **Always output:** true (so downstream nodes run even if no history exists).
- **Edge cases:** History table empty; ensure downstream JS handles empty arrays.

**Code in JavaScript**
- **Type/role:** Code node; builds AI context and/or computes campaign step updates.
- **Configuration (intended):**
  - Merge customer data + campaign step + message history
  - Produce a prompt input for AI Agent (email objective, tone, constraints)
  - Produce “campaign update” data for Update Campaign
- **Input:** Get histories.
- **Outputs:** Two branches:
  - To **AI Agent** (for generation)
  - To **Update Campaign** (to persist campaign state changes)
- **Failures:** Runtime errors; incorrect date arithmetic; missing keys.

**AI Agent**
- **Type/role:** LangChain Agent; generates outbound email content.
- **Model:** Uses **OpenAI Model** via `ai_languageModel`.
- **Input:** Code in JavaScript.
- **Output:** Convert to JSON object1.
- **Edge cases:** Non-deterministic output; must be forced into a strict schema for Gmail.

**OpenAI Model**
- **Type/role:** `lmChatOpenAi`; provides the chat model for AI Agent.
- **Failures:** Rate limits; response too large; model availability.

**Convert to JSON object1**
- **Type/role:** Code node; converts AI output to Gmail “send email” fields.
- **Configuration (intended):** Parse `{to, subject, bodyHtml/bodyText, cc, bcc}`.
- **Input:** AI Agent.
- **Output:** Send a message.
- **Edge cases:** Invalid email addresses; missing subject; HTML sanitization.

**Send a message**
- **Type/role:** Gmail node; sends the outbound campaign email.
- **Configuration (intended):** Gmail OAuth2 credentials; “Send” operation.
- **Input:** Convert to JSON object1.
- **Output:** Insert message history.
- **Failures:** OAuth revoked; Gmail API quota; sending limits; invalid RFC headers.

**Insert message history**
- **Type/role:** Data Table node; logs outbound email metadata.
- **Configuration (intended):** Store `messageId`, timestamp, campaignId, customerId, subject/body summary.
- **Input:** Send a message.
- **Output:** none shown.
- **Edge cases:** Duplicate message IDs; storage schema drift.

**Update Campaign**
- **Type/role:** Data Table node; updates campaign status (step advanced, lastSentAt, nextSendAt).
- **Input:** Code in JavaScript.
- **Output:** Update row in sheet.
- **Edge cases:** Race conditions if multiple sends in parallel.

**Update row in sheet**
- **Type/role:** Google Sheets node; reflects campaign progress back to Sheets.
- **Input:** Update Campaign.
- **Failures:** Row identification problems; partial updates.

---

### 1.3 Inbound Reply Processing (Gmail Trigger)

**Overview:** When a reply arrives in Gmail, the workflow retrieves prior history, stores the reply, categorizes it (including sentiment), optionally generates calendar event details and creates an event, fetches HubSpot customer context, generates an AI reply, sends the reply, and logs the sent message.

**Nodes involved:**
- Gmail Trigger
- Get message histories
- Update customer’s reply
- ネガポジ判定 (sentiment/positive-negative judgment)
- Update reply’s category
- Create calendar info
- Convert to JSON Object
- Create an event
- Get Customer (HubSpot)
- reply to customer (OpenAI)
- Convert to JSON object2
- Reply to a message
- Insert new mesage

#### Node details

**Gmail Trigger**
- **Type/role:** Gmail Trigger; entry point for inbound emails.
- **Configuration (intended):** Watch inbox/label/search query (not provided).
- **Output:** Get message histories.
- **Failures:** OAuth issues; watch expiration; missing Pub/Sub setup depending on n8n/Gmail trigger mode.

**Get message histories**
- **Type/role:** Data Table node; fetch existing conversation/campaign history by thread/message identifiers.
- **Input:** Gmail Trigger.
- **Output:** Update customer’s reply.
- **Failures:** Lookup key mismatch; no history found (should still proceed depending on design).

**Update customer’s reply**
- **Type/role:** Data Table node; persists inbound reply content and metadata.
- **Input:** Get message histories.
- **Outputs:** Fan-out to:
  - ネガポジ判定 (categorization)
  - Create calendar info (event extraction)
  - Get Customer (HubSpot enrichment)
- **Failures:** Storage issues; oversized email body; encoding.

**ネガポジ判定**
- **Type/role:** OpenAI node (LangChain OpenAI wrapper); classifies sentiment/intent.
- **Configuration (intended):** Prompt to label reply (e.g., positive/neutral/negative, interested/not interested).
- **Input:** Update customer’s reply.
- **Output:** Update reply’s category.
- **Edge cases:** Ambiguous replies; non-text replies; language variance.

**Update reply’s category**
- **Type/role:** Data Table node; writes classification results back to stored reply record.
- **Input:** ネガポジ判定.
- **Output:** none shown.

**Create calendar info**
- **Type/role:** OpenAI node; extracts calendar event details from reply when relevant.
- **Configuration (intended):** Prompt to output structured event JSON (title, start/end, attendees, timezone).
- **Input:** Update customer’s reply.
- **Output:** Convert to JSON Object.
- **Edge cases:** Missing dates/times; timezone ambiguity; hallucinated times—should validate before creating events.

**Convert to JSON Object**
- **Type/role:** Code node; parses/validates event JSON for Google Calendar.
- **Input:** Create calendar info.
- **Output:** Create an event.
- **Failures:** JSON parsing; invalid RFC3339 timestamps.

**Create an event**
- **Type/role:** Google Calendar node; creates event.
- **Configuration (intended):** Calendar ID; event fields mapped from parsed JSON.
- **Input:** Convert to JSON Object.
- **Output:** none shown.
- **Failures:** Calendar permissions; invalid time ranges; attendee permission restrictions.

**Get Customer** (HubSpot)
- **Type/role:** HubSpot node; fetches contact context for personalized reply.
- **Input:** Update customer’s reply.
- **Output:** reply to customer.
- **Failures:** Contact not found; auth; rate limits.

**reply to customer**
- **Type/role:** OpenAI node; generates reply email text given inbound content + customer context + history.
- **Input:** Get Customer.
- **Output:** Convert to JSON object2.
- **Edge cases:** Must enforce safe email format; ensure it references correct thread.

**Convert to JSON object2**
- **Type/role:** Code node; produces Gmail “reply” payload and a log record payload.
- **Input:** reply to customer.
- **Outputs:** Two branches:
  - Reply to a message (send)
  - Insert new mesage (log)
- **Failures:** Missing threadId/messageId; invalid headers.

**Reply to a message**
- **Type/role:** Gmail node; sends reply in-thread.
- **Input:** Convert to JSON object2.
- **Output:** none shown.
- **Failures:** Gmail thread/message not found; insufficient scopes.

**Insert new mesage**
- **Type/role:** Data Table node; logs the outbound reply.
- **Input:** Convert to JSON object2.
- **Output:** none shown.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Every hour | Schedule Trigger | Hourly entry point | — | Get customers; Get campaign |  |
| Get customers | Google Sheets | Read customers list | Every hour | Filter |  |
| Filter | Filter | Select eligible/sync-needed customers | Get customers | Customer information |  |
| Customer information | Code | Transform customer row to structured object | Filter | AI Agent1 |  |
| OpenAI Model1 | LM Chat OpenAI | Language model for Agent1 | — | AI Agent1 (ai_languageModel) |  |
| AI Agent1 | LangChain Agent | Normalize/enrich customer payload | Customer information | Convert to JSON object |  |
| Convert to JSON object | Code | Parse AI output to strict JSON for HubSpot | AI Agent1 | Create new Customer |  |
| Create new Customer | HubSpot | Create contact record | Convert to JSON object | Update row in sheet1 |  |
| Update row in sheet1 | Google Sheets | Write HubSpot sync back to sheet | Create new Customer | — |  |
| Get campaign | Google Sheets | Read campaign definitions | Every hour | If campaign does not exist1 |  |
| If campaign does not exist1 | Data Table | Campaign existence check/lookup | Get campaign | Insert new campaign |  |
| Insert new campaign | Data Table | Store new campaign definition | If campaign does not exist1 | — |  |
| Daily | Schedule Trigger | Daily entry point for sending | — | Don’t email on weekends |  |
| Don’t email on weekends | Filter | Block weekend sends | Daily | Get new campaign |  |
| Get new campaign | Data Table | Select due campaign targets | Don’t email on weekends | Get customer |  |
| Get customer | HubSpot | Fetch target contact context | Get new campaign | Filter customer |  |
| Filter customer | Filter | Final eligibility filter before send | Get customer | Get histories |  |
| Get histories | Data Table | Fetch conversation history | Filter customer | Code in JavaScript |  |
| Code in JavaScript | Code | Build AI prompt context + campaign updates | Get histories | AI Agent; Update Campaign |  |
| OpenAI Model | LM Chat OpenAI | Language model for outbound Agent | — | AI Agent (ai_languageModel) |  |
| AI Agent | LangChain Agent | Generate outbound email | Code in JavaScript | Convert to JSON object1 |  |
| Convert to JSON object1 | Code | Map AI output to Gmail send fields | AI Agent | Send a message |  |
| Send a message | Gmail | Send outbound campaign email | Convert to JSON object1 | Insert message history |  |
| Insert message history | Data Table | Log outbound email | Send a message | — |  |
| Update Campaign | Data Table | Advance/update campaign state | Code in JavaScript | Update row in sheet |  |
| Update row in sheet | Google Sheets | Persist campaign progress to Sheets | Update Campaign | — |  |
| Gmail Trigger | Gmail Trigger | Entry point for inbound replies | — | Get message histories |  |
| Get message histories | Data Table | Fetch related thread/history | Gmail Trigger | Update customer’s reply |  |
| Update customer’s reply | Data Table | Store inbound reply content | Get message histories | ネガポジ判定; Create calendar info; Get Customer |  |
| ネガポジ判定 | OpenAI | Classify sentiment/intent | Update customer’s reply | Update reply’s category |  |
| Update reply’s category | Data Table | Store classification results | ネガポジ判定 | — |  |
| Create calendar info | OpenAI | Extract event details from reply | Update customer’s reply | Convert to JSON Object |  |
| Convert to JSON Object | Code | Validate/parse event JSON | Create calendar info | Create an event |  |
| Create an event | Google Calendar | Create calendar event | Convert to JSON Object | — |  |
| Get Customer | HubSpot | Fetch customer for reply personalization | Update customer’s reply | reply to customer |  |
| reply to customer | OpenAI | Generate reply email content | Get Customer | Convert to JSON object2 |  |
| Convert to JSON object2 | Code | Prepare Gmail reply payload + log payload | reply to customer | Reply to a message; Insert new mesage |  |
| Reply to a message | Gmail | Send email reply | Convert to JSON object2 | — |  |
| Insert new mesage | Data Table | Log outbound reply | Convert to JSON object2 | — |  |
| Get histories (duplicate name is unique node) | Data Table | (Already listed above) | Filter customer | Code in JavaScript |  |
| Sticky Note / Sticky Note1…13 | Sticky Note | Comments (all empty content) | — | — |  |

> Sticky notes exist but their `content` is empty in the export, so no comments can be associated to covered nodes.

---

## 4. Reproducing the Workflow from Scratch

1) **Create workflow**
1. Create a new n8n workflow named: **“AI-Driven Campaign Email Automation”**.
2. Ensure settings are compatible with your environment (execution order can remain default).

### A. Hourly sync: Customers + Campaign definitions

2) **Add trigger**
- Add **Schedule Trigger** → name **Every hour** → set to run hourly.

3) **Read customers from Google Sheets**
- Add **Google Sheets** → name **Get customers**
  - Credential: Google OAuth2 with Sheets scope
  - Operation: Read/Get Many Rows
  - Select Spreadsheet + Customers sheet/range
- Connect: **Every hour → Get customers**

4) **Filter customer rows**
- Add **Filter** → name **Filter**
  - Conditions (example): `email is not empty`, `optIn == true`, `hubspotId is empty` (adapt to your sheet columns)
- Connect: **Get customers → Filter**

5) **Transform customer row**
- Add **Code (JavaScript)** → name **Customer information**
  - Build a normalized object from the row fields (email, name, company, tags, etc.)
- Connect: **Filter → Customer information**

6) **AI Agent for normalization**
- Add **AI Agent (LangChain Agent)** → name **AI Agent1**
  - Provide system/user instructions to output *strict JSON* for HubSpot contact creation (properties object).
- Add **OpenAI Chat Model** → name **OpenAI Model1**
  - Credential: OpenAI API key
  - Choose a suitable model (GPT-4-class), set temperature low (e.g., 0–0.3) for structured output
- Connect:
  - **Customer information → AI Agent1**
  - **OpenAI Model1 (ai_languageModel) → AI Agent1**

7) **Parse AI output**
- Add **Code (JavaScript)** → name **Convert to JSON object**
  - Parse AI text into JSON; validate required fields (at minimum: email)
- Connect: **AI Agent1 → Convert to JSON object**

8) **Create HubSpot contact**
- Add **HubSpot** → name **Create new Customer**
  - Credential: HubSpot Private App token or OAuth
  - Operation: Create Contact
  - Map properties from parsed JSON
- Connect: **Convert to JSON object → Create new Customer**

9) **Update customer row in Sheets**
- Add **Google Sheets** → name **Update row in sheet1**
  - Operation: Update Row (by row number or by lookup key like email)
  - Write back HubSpot ID + sync timestamp/status
- Connect: **Create new Customer → Update row in sheet1**

10) **Read campaigns from Google Sheets**
- Add **Google Sheets** → name **Get campaign**
  - Operation: Read/Get Many Rows
  - Select Spreadsheet + Campaigns sheet/range
- Connect: **Every hour → Get campaign**

11) **Check/insert campaigns into Data Table**
- Create a **Data Table** (n8n Data Tables feature) for campaigns (e.g., `campaigns`), with columns like:
  - `campaignId` (unique), `name`, `steps`, `active`, `updatedAt`
- Add **Data Table** → name **If campaign does not exist1**
  - Configure as “Lookup” (by `campaignId`/name) or an equivalent existence check
- Add **Data Table** → name **Insert new campaign**
  - Configure as “Insert” when not found
- Connect: **Get campaign → If campaign does not exist1 → Insert new campaign**

### B. Daily outbound campaign sending

12) **Daily trigger**
- Add **Schedule Trigger** → name **Daily** → set once per day (weekday mornings).

13) **Weekend suppression**
- Add **Filter** → name **Don’t email on weekends**
  - Condition: day-of-week is Mon–Fri (use n8n date expressions; ensure timezone)
- Connect: **Daily → Don’t email on weekends**

14) **Get due campaign targets**
- Create a **Data Table** for campaign assignments/executions (e.g., `campaign_runs`) with:
  - `customerKey` (email/hubspotId), `campaignId`, `step`, `nextSendAt`, `status`, `lastSentAt`
- Add **Data Table** → name **Get new campaign**
  - Query rows due to send today (`nextSendAt <= now`, `status == active`)
- Connect: **Don’t email on weekends → Get new campaign**

15) **Fetch customer from HubSpot**
- Add **HubSpot** → name **Get customer**
  - Operation: Get/Search Contact by email or ID
- Connect: **Get new campaign → Get customer**

16) **Eligibility filter**
- Add **Filter** → name **Filter customer**
  - Conditions: has email, opt-in true, not unsubscribed, etc.
- Connect: **Get customer → Filter customer**

17) **Fetch message history**
- Add **Data Table** → name **Get histories**
  - Query history table by customer + campaign/thread
  - Enable “Always Output Data” (so it returns an empty set instead of stopping)
- Connect: **Filter customer → Get histories**

18) **Prepare AI context + campaign update**
- Add **Code (JavaScript)** → name **Code in JavaScript**
  - Build:
    - `promptContext` for the outbound email (customer info + campaign step + recent history)
    - `campaignUpdate` object (advance step, set lastSentAt/nextSendAt)
- Connect: **Get histories → Code in JavaScript**

19) **Generate email with AI Agent**
- Add **AI Agent** → name **AI Agent**
  - Prompt: generate subject + body, return strict JSON
- Add **OpenAI Chat Model** → name **OpenAI Model**
- Connect:
  - **Code in JavaScript → AI Agent**
  - **OpenAI Model (ai_languageModel) → AI Agent**

20) **Parse to Gmail payload**
- Add **Code (JavaScript)** → name **Convert to JSON object1**
  - Output fields for Gmail send node: `to`, `subject`, `message` (text/html), optional headers
- Connect: **AI Agent → Convert to JSON object1**

21) **Send email**
- Add **Gmail** → name **Send a message**
  - Credential: Google OAuth2 with Gmail scopes
  - Operation: Send
- Connect: **Convert to JSON object1 → Send a message**

22) **Log outbound message**
- Add **Data Table** → name **Insert message history**
  - Insert `messageId`, `threadId`, timestamp, subject, customer, campaignId, step
- Connect: **Send a message → Insert message history**

23) **Update campaign state**
- Add **Data Table** → name **Update Campaign**
  - Update `campaign_runs` record using `campaignUpdate`
- Connect: **Code in JavaScript → Update Campaign**

24) **Write campaign progress back to Sheets**
- Add **Google Sheets** → name **Update row in sheet**
  - Operation: Update row in campaign/customer sheet with step/lastSent/nextSend
- Connect: **Update Campaign → Update row in sheet**

### C. Inbound replies: classify, schedule, respond

25) **Inbound trigger**
- Add **Gmail Trigger** → name **Gmail Trigger**
  - Configure watch (label/query such as “in:inbox -from:me” or campaign label)
- Connect: **Gmail Trigger → Get message histories**

26) **Lookup message history**
- Add **Data Table** → name **Get message histories**
  - Lookup by `threadId`/`messageId` from Gmail trigger output
- Connect: **Gmail Trigger → Get message histories**

27) **Store inbound reply**
- Add **Data Table** → name **Update customer’s reply**
  - Insert/update inbound reply record (body, from, date, threadId)
- Connect: **Get message histories → Update customer’s reply**

28) **Sentiment/intent classification**
- Add **OpenAI** → name **ネガポジ判定**
  - Prompt: return category labels (positive/neutral/negative, interested/not interested, etc.) as JSON
- Add **Data Table** → name **Update reply’s category**
  - Update reply record with classification
- Connect: **Update customer’s reply → ネガポジ判定 → Update reply’s category**

29) **Calendar extraction + creation**
- Add **OpenAI** → name **Create calendar info**
  - Prompt: if scheduling intent exists, return event JSON; otherwise return an explicit “no_event” flag
- Add **Code (JavaScript)** → name **Convert to JSON Object**
  - Validate; only proceed when event is present
- Add **Google Calendar** → name **Create an event**
  - Operation: Create
- Connect: **Update customer’s reply → Create calendar info → Convert to JSON Object → Create an event**

30) **Fetch HubSpot customer for reply**
- Add **HubSpot** → name **Get Customer**
  - Search by email from inbound message
- Connect: **Update customer’s reply → Get Customer**

31) **Generate AI reply**
- Add **OpenAI** → name **reply to customer**
  - Prompt uses inbound email + HubSpot context + history
  - Output strict JSON with reply body + subject/thread handling hints
- Connect: **Get Customer → reply to customer**

32) **Prepare Gmail reply payload + log payload**
- Add **Code (JavaScript)** → name **Convert to JSON object2**
  - Produce:
    - Gmail reply fields (threadId, inReplyTo, references, subject, body)
    - Log object for Data Table
- Connect: **reply to customer → Convert to JSON object2**

33) **Send reply + log**
- Add **Gmail** → name **Reply to a message** (operation: Reply)
- Add **Data Table** → name **Insert new mesage**
- Connect:
  - **Convert to JSON object2 → Reply to a message**
  - **Convert to JSON object2 → Insert new mesage**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sticky notes are present but empty in the provided workflow export. | No additional embedded documentation was available. |
| Several nodes have empty parameter blocks in the export; you must supply Sheet IDs/ranges, Data Table names/schemas, HubSpot operations/fields, Gmail trigger query/labels, and OpenAI prompts/models. | Applies to most integration nodes. |
| Add robust JSON validation after AI steps (all “Convert to JSON…” nodes) to prevent malformed payloads from reaching Gmail/Calendar/HubSpot. | Best practice for AI-structured output. |

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.