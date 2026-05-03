Capture, score and route Gmail leads with Groq Llama 3.3, Supabase and Slack

https://n8nworkflows.xyz/workflows/capture--score-and-route-gmail-leads-with-groq-llama-3-3--supabase-and-slack-14362


# Capture, score and route Gmail leads with Groq Llama 3.3, Supabase and Slack

# 1. Workflow Overview

This workflow captures inbound Gmail messages, normalizes and validates lead data, stores the lead in Supabase, waits for a delay, then runs two AI-driven follow-up processes:

1. **Lead scoring** using Groq Llama 3.3 to assign a numeric quality score.
2. **Lead classification/routing** to determine whether the message should go to Sales, Support, Billing, or Other.

It also sends Slack notifications both when a new lead is first received and when the classified lead is routed to the appropriate team channel.

## 1.1 Input Reception and Lead Normalization
The workflow starts from a **Gmail Trigger**, fetches the message details, extracts normalized fields such as sender, message body, source, and thread ID, then validates that required values are present.

## 1.2 Lead Notification and Database Storage
If validation passes, the workflow sends an initial Slack notification and stores the lead in a Supabase `leads` table with initial metadata such as `score = 0` and `status = new`.

## 1.3 Delayed Post-Processing
A **Wait** node pauses execution for 1 hour. After the delay, the workflow branches into:
- an **AI scoring branch** for leads with `score = 0`
- a **classification/routing branch** for leads with `status = update`

## 1.4 AI Lead Scoring
The scoring branch retrieves unscored leads from Supabase, sends each lead to a Groq-backed LangChain agent, parses structured JSON output, maps the score fields, and updates the lead record in Supabase.

## 1.5 AI Classification and Slack Routing
The routing branch retrieves leads whose status is `update`, verifies a message exists, classifies the message into a fixed category using another Groq-backed agent, then routes the record through a Switch node to the appropriate Slack channel.

---

# 2. Block-by-Block Analysis

## 2.1 Block: Lead Capture and Normalization

**Overview:**  
This block detects new Gmail activity, fetches the email payload, converts it into a simplified internal lead structure, and validates that the minimum required fields exist before further processing.

**Nodes Involved:**  
- Gmail Trigger1
- Get a message1
- Normalize Incoming Lead Data1
- Validate Lead Data1

### Node: Gmail Trigger1
- **Type and technical role:** `n8n-nodes-base.gmailTrigger`  
  Polling trigger for new Gmail events.
- **Configuration choices:**  
  - Polls Gmail **every minute**
  - No additional filters are configured
- **Key expressions or variables used:**  
  None in parameters.
- **Input and output connections:**  
  - Entry point node
  - Outputs to **Get a message1**
- **Version-specific requirements:**  
  Uses `typeVersion: 1.3`; requires a valid Gmail account connection in n8n.
- **Edge cases or potential failure types:**  
  - Gmail OAuth authentication failure
  - Polling limits or quota issues
  - Trigger may pick up emails that are not actual leads because no filter is defined
- **Sub-workflow reference:**  
  None

### Node: Get a message1
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Fetches full Gmail message details.
- **Configuration choices:**  
  - Operation: **get**
  - `simple: false`, so raw/full message data is retained
  - `messageId` is set from `={{ $json.threadId }}`
- **Key expressions or variables used:**  
  - `{{$json.threadId}}`
- **Input and output connections:**  
  - Input from **Gmail Trigger1**
  - Output to **Normalize Incoming Lead Data1**
- **Version-specific requirements:**  
  Uses `typeVersion: 2.2`
- **Edge cases or potential failure types:**  
  - Possible mismatch between `threadId` and a true Gmail `messageId`
  - Missing permissions to read message contents
  - Message retrieval may fail if the referenced item no longer exists
- **Sub-workflow reference:**  
  None

### Node: Normalize Incoming Lead Data1
- **Type and technical role:** `n8n-nodes-base.set`  
  Reshapes Gmail data into a standardized lead payload.
- **Configuration choices:**  
  Creates the following fields:
  - `name` = sender email address from `from.value[0].address`
  - `message` = email plain text from `text`
  - `threadId` = Gmail thread ID
  - `source` = constant `"gmail"`
  - `gmail` = sender email address
- **Key expressions or variables used:**  
  - `={{ $json.from.value[0].address }}`
  - `={{ $json.text }}`
  - `={{ $json.threadId }}`
- **Input and output connections:**  
  - Input from **Get a message1**
  - Output to **Validate Lead Data1**
- **Version-specific requirements:**  
  Uses `typeVersion: 3.4`
- **Edge cases or potential failure types:**  
  - `from.value[0]` may not exist for unusual email payloads
  - `text` may be empty if the email is HTML-only
  - The workflow stores sender email into both `name` and `gmail`, so no human-readable name is extracted
- **Sub-workflow reference:**  
  None

### Node: Validate Lead Data1
- **Type and technical role:** `n8n-nodes-base.if`  
  Gatekeeper node to ensure required lead fields are present.
- **Configuration choices:**  
  Requires all of the following to be non-empty:
  - `name`
  - `message`
  - `threadId`
  - `source`
  - `gmail`
- **Key expressions or variables used:**  
  - `={{ $json.name }}`
  - `={{ $json.message }}`
  - `={{ $json.threadId }}`
  - `={{ $json.source }}`
  - `={{ $json.gmail }}`
- **Input and output connections:**  
  - Input from **Normalize Incoming Lead Data1**
  - True output goes to **Notify New Lead (Slack)1**
  - False output is unused
- **Version-specific requirements:**  
  Uses `typeVersion: 2.3`
- **Edge cases or potential failure types:**  
  - Leads failing validation are silently dropped because the false branch is not connected
  - If `message` is blank due to email formatting, a valid lead may be lost
- **Sub-workflow reference:**  
  None

---

## 2.2 Block: Lead Notification and Database Storage

**Overview:**  
This block sends an immediate operational alert to Slack and stores the normalized lead in Supabase with initial scoring and status values.

**Nodes Involved:**  
- Notify New Lead (Slack)1
- Store Lead in Database1

### Node: Notify New Lead (Slack)1
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends a Slack message to a notification channel when a lead arrives.
- **Configuration choices:**  
  - Sends a formatted message containing:
    - name/from
    - email
    - source
    - message
    - thread ID
  - Uses channel mode with placeholder channel ID
  - `includeLinkToWorkflow` disabled
  - Authentication via OAuth2
- **Key expressions or variables used:**  
  - `{{ $json.name }}`
  - `{{ $json.email }}`
  - `{{ $json.source }}`
  - `{{ $json.message }}`
  - `{{ $json.threadId }}`
- **Input and output connections:**  
  - Input from **Validate Lead Data1**
  - Output to **Store Lead in Database1**
- **Version-specific requirements:**  
  Uses `typeVersion: 2.4`
- **Edge cases or potential failure types:**  
  - The message references `{{ $json.email }}`, but the normalization block creates `gmail`, not `email`
  - Slack OAuth/token issues
  - Invalid channel ID placeholder not replaced
- **Sub-workflow reference:**  
  None

### Node: Store Lead in Database1
- **Type and technical role:** `n8n-nodes-base.supabase`  
  Inserts a new lead row into Supabase.
- **Configuration choices:**  
  Inserts into table `leads` with fields:
  - `name` = normalized `name`
  - `email` = `{{$json.email}}`
  - `source` = normalized `source`
  - `score` = `"0"`
  - `status` = `"new"`
  - `created_at` = current ISO timestamp
  - `message` = normalized message
  - `thread_id` = normalized thread ID
- **Key expressions or variables used:**  
  - `={{ $json.name }}`
  - `={{ $json.email }}`
  - `={{ $json.source }}`
  - `={{ new Date().toISOString() }}`
  - `={{ $json.message }}`
  - `={{ $json.threadId }}`
- **Input and output connections:**  
  - Input from **Notify New Lead (Slack)1**
  - Output to **Wait1**
- **Version-specific requirements:**  
  Uses `typeVersion: 1`
- **Edge cases or potential failure types:**  
  - `email` may be null/empty because upstream normalization creates `gmail`, not `email`
  - Supabase credential or table permission errors
  - Table schema mismatch if fields do not exist exactly as configured
- **Sub-workflow reference:**  
  None

---

## 2.3 Block: Delayed Post-Processing

**Overview:**  
This block inserts a 1-hour pause before downstream enrichment. After the wait, the workflow fans out into scoring and classification branches in parallel.

**Nodes Involved:**  
- Wait1

### Node: Wait1
- **Type and technical role:** `n8n-nodes-base.wait`  
  Delays workflow execution for asynchronous follow-up processing.
- **Configuration choices:**  
  - Wait duration: **1 hour**
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Input from **Store Lead in Database1**
  - Outputs to:
    - **Fetch Unscored Leads1**
    - **Fetch Scored Leads1**
- **Version-specific requirements:**  
  Uses `typeVersion: 1.1`
- **Edge cases or potential failure types:**  
  - Delayed jobs depend on n8n’s execution persistence being enabled
  - Large workflow volume may accumulate waiting executions
  - Both downstream branches query broad sets of records, not just the lead that triggered the wait
- **Sub-workflow reference:**  
  None

---

## 2.4 Block: AI Lead Scoring Pipeline

**Overview:**  
This block pulls leads with `score = 0`, scores them through a structured AI agent using Groq Llama 3.3, then updates the lead row with score and a `scored` status.

**Nodes Involved:**  
- Fetch Unscored Leads1
- AI Lead Scoring Engine1
- LLM (Scoring)1
- Structured Output Parser1
- Map AI Score Output1
- Update Lead Score1

### Node: Fetch Unscored Leads1
- **Type and technical role:** `n8n-nodes-base.supabase`  
  Reads all unscored lead records from Supabase.
- **Configuration choices:**  
  - Table: `leads`
  - Operation: `getAll`
  - Filter: `score = 0`
  - `returnAll: true`
- **Key expressions or variables used:**  
  None in the filter value beyond literal `0`.
- **Input and output connections:**  
  - Input from **Wait1**
  - Outputs to:
    - **AI Lead Scoring Engine1**
    - **Map AI Score Output1**
- **Version-specific requirements:**  
  Uses `typeVersion: 1`
- **Edge cases or potential failure types:**  
  - Fetches all unscored leads globally, not just the current lead
  - Can reprocess multiple rows every time a wait completes
  - Large datasets may impact runtime
- **Sub-workflow reference:**  
  None

### Node: AI Lead Scoring Engine1
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  LangChain agent that prompts the LLM to produce structured scoring JSON.
- **Configuration choices:**  
  - Prompt includes lead fields:
    - `name`
    - `email`
    - `source`
    - `message`
  - System message defines scoring criteria and strict JSON-only output
  - `hasOutputParser: true`
  - Prompt type: defined manually
- **Key expressions or variables used:**  
  - `{{$json.name}}`
  - `{{$json.email}}`
  - `{{$json.source}}`
  - `{{$json.message}}`
- **Input and output connections:**  
  - Main input from **Fetch Unscored Leads1**
  - AI model input from **LLM (Scoring)1**
  - Output parser input from **Structured Output Parser1**
  - Main output to **Map AI Score Output1**
- **Version-specific requirements:**  
  Uses `typeVersion: 3`; requires LangChain-compatible setup in the n8n version being used.
- **Edge cases or potential failure types:**  
  - If `email` is missing in DB, score quality may degrade
  - LLM may still produce invalid output despite parser constraints
  - Groq model access/credential issues
  - Prompt relies on CRM assumptions that may not fit all lead types
- **Sub-workflow reference:**  
  None

### Node: LLM (Scoring)1
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatGroq`  
  Groq-hosted chat model used by the scoring agent.
- **Configuration choices:**  
  - No explicit model set in parameters
  - Options left default
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - AI language model output to **AI Lead Scoring Engine1**
- **Version-specific requirements:**  
  Uses `typeVersion: 1`; requires Groq API credentials.
- **Edge cases or potential failure types:**  
  - Missing model selection may rely on defaults or fail depending on node version
  - API quota/rate limits
  - Timeout on long prompts or service instability
- **Sub-workflow reference:**  
  None

### Node: Structured Output Parser1
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`  
  Enforces a JSON schema for the scoring output.
- **Configuration choices:**  
  Manual JSON schema requiring:
  - `score` as number
  - `note` as string
  - both fields required
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Output parser connection to **AI Lead Scoring Engine1**
- **Version-specific requirements:**  
  Uses `typeVersion: 1.3`
- **Edge cases or potential failure types:**  
  - Parser failure if model output is malformed
  - `score` could still be outside 1–100 even if it is numeric unless additionally validated downstream
- **Sub-workflow reference:**  
  None

### Node: Map AI Score Output1
- **Type and technical role:** `n8n-nodes-base.set`  
  Renames AI output fields to the format used by the database update node.
- **Configuration choices:**  
  Creates:
  - `Score` from `score`
  - `message` from `note`
- **Key expressions or variables used:**  
  - `={{ $json.score }}`
  - `={{ $json.note }}`
- **Input and output connections:**  
  - Receives input from **Fetch Unscored Leads1** and **AI Lead Scoring Engine1**
  - Outputs to **Update Lead Score1**
- **Version-specific requirements:**  
  Uses `typeVersion: 3.4`
- **Edge cases or potential failure types:**  
  - Because it also receives direct input from **Fetch Unscored Leads1**, it may execute on items that do not yet contain `score`/`note`
  - This can create empty values and lead to broken updates
- **Sub-workflow reference:**  
  None

### Node: Update Lead Score1
- **Type and technical role:** `n8n-nodes-base.supabase`  
  Updates the lead row with the AI score and marks it as `scored`.
- **Configuration choices:**  
  - Table: `leads`
  - Operation: `update`
  - Filter by `id` using the current item from **Fetch Unscored Leads1**
  - Updates:
    - `score` = `{{$json.Score}}`
    - `status` = `scored`
- **Key expressions or variables used:**  
  - `={{ $('Fetch Unscored Leads1').item.json.id }}`
  - `={{ $json.Score }}`
- **Input and output connections:**  
  - Input from **Map AI Score Output1**
- **Version-specific requirements:**  
  Uses `typeVersion: 1`
- **Edge cases or potential failure types:**  
  - Depends on item linking to **Fetch Unscored Leads1**; can break if execution item pairing is altered
  - Status becomes `scored`, but the classification branch later expects `status = update`, creating a likely logic mismatch
- **Sub-workflow reference:**  
  None

---

## 2.5 Block: Lead Qualification, Classification, and Routing

**Overview:**  
This block retrieves leads marked for routing, confirms a message exists, classifies each message with Groq Llama 3.3, then routes the item to the relevant Slack destination.

**Nodes Involved:**  
- Fetch Scored Leads1
- Check Message Exists1
- AI Lead Classification Engine1
- LLM (Classification)1
- Route Lead by Category1
- Send to Sales Channel1
- Send to Support Channel1
- Send to Billing Channel1
- End1

### Node: Fetch Scored Leads1
- **Type and technical role:** `n8n-nodes-base.supabase`  
  Reads leads from Supabase for routing.
- **Configuration choices:**  
  - Table: `leads`
  - Operation: `getAll`
  - Filter: `status = update`
  - `returnAll: true`
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Input from **Wait1**
  - Output to **Check Message Exists1**
- **Version-specific requirements:**  
  Uses `typeVersion: 1`
- **Edge cases or potential failure types:**  
  - No rows will be returned if no process ever sets `status` to `update`
  - Current scoring branch sets status to `scored`, so this branch may never run
- **Sub-workflow reference:**  
  None

### Node: Check Message Exists1
- **Type and technical role:** `n8n-nodes-base.if`  
  Ensures the lead has a non-empty message before classification.
- **Configuration choices:**  
  Condition:
  - `message` is not empty
- **Key expressions or variables used:**  
  - `={{ $json.message }}`
- **Input and output connections:**  
  - Input from **Fetch Scored Leads1**
  - True output to **AI Lead Classification Engine1**
  - False output unused
- **Version-specific requirements:**  
  Uses `typeVersion: 2.3`
- **Edge cases or potential failure types:**  
  - False branch silently discards records
- **Sub-workflow reference:**  
  None

### Node: AI Lead Classification Engine1
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  Uses an LLM prompt to classify a message into one of four fixed categories.
- **Configuration choices:**  
  - Prompt text: classify `{{$json.message}}`
  - System message strictly limits outputs to:
    - `sales`
    - `support`
    - `billing`
    - `other`
  - Output must be one lowercase word with no punctuation
- **Key expressions or variables used:**  
  - `{{$json.message}}`
- **Input and output connections:**  
  - Main input from **Check Message Exists1**
  - AI model input from **LLM (Classification)1**
  - Main output to **Route Lead by Category1**
- **Version-specific requirements:**  
  Uses `typeVersion: 3`
- **Edge cases or potential failure types:**  
  - Output format may vary depending on node version; the downstream switch expects `output`
  - If the agent returns text in a different field, routing fails
  - Ambiguous messages may be misclassified
- **Sub-workflow reference:**  
  None

### Node: LLM (Classification)1
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatGroq`  
  Groq-hosted model for classification.
- **Configuration choices:**  
  - Explicit model: `llama-3.3-70b-versatile`
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - AI language model output to **AI Lead Classification Engine1**
- **Version-specific requirements:**  
  Uses `typeVersion: 1`; requires Groq credentials.
- **Edge cases or potential failure types:**  
  - API/authentication/rate-limit issues
  - Classification latency or service errors
- **Sub-workflow reference:**  
  None

### Node: Route Lead by Category1
- **Type and technical role:** `n8n-nodes-base.switch`  
  Branches the execution based on classification result.
- **Configuration choices:**  
  Named outputs:
  - `sales` when `{{$json.output}} == "sales"`
  - `support` when `{{$json.output}} == "support"`
  - `billing` when `{{$json.output}} == "billing"`
  - `other` when `{{$json.output}} == "other"`
- **Key expressions or variables used:**  
  - `={{ $json.output }}`
- **Input and output connections:**  
  - Input from **AI Lead Classification Engine1**
  - Output 1 to **Send to Sales Channel1**
  - Output 2 to **Send to Support Channel1**
  - Output 3 to **Send to Billing Channel1**
  - Output 4 to **End1**
- **Version-specific requirements:**  
  Uses `typeVersion: 3.4`
- **Edge cases or potential failure types:**  
  - If classification output is not in `output`, all routing can fail
  - If model returns unexpected text, no intended branch will match
- **Sub-workflow reference:**  
  None

### Node: Send to Sales Channel1
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends routed lead data to the sales Slack channel.
- **Configuration choices:**  
  Message includes:
  - name
  - email
  - message
  - source
  - OAuth2 auth explicitly enabled
- **Key expressions or variables used:**  
  - `{{ $json.name }}`
  - `{{ $json.email }}`
  - `{{ $json.message }}`
  - `{{ $json.source }}`
- **Input and output connections:**  
  - Input from **Route Lead by Category1** (`sales`)
- **Version-specific requirements:**  
  Uses `typeVersion: 2.4`
- **Edge cases or potential failure types:**  
  - Placeholder channel ID must be replaced
  - Email may be empty if not stored correctly upstream
- **Sub-workflow reference:**  
  None

### Node: Send to Support Channel1
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends routed lead data to the support Slack channel.
- **Configuration choices:**  
  Similar formatted message to sales.
- **Key expressions or variables used:**  
  - `{{ $json.name }}`
  - `{{ $json.email }}`
  - `{{ $json.message }}`
  - `{{ $json.source }}`
- **Input and output connections:**  
  - Input from **Route Lead by Category1** (`support`)
- **Version-specific requirements:**  
  Uses `typeVersion: 2.4`
- **Edge cases or potential failure types:**  
  - Channel placeholder not configured
  - Slack auth issues
- **Sub-workflow reference:**  
  None

### Node: Send to Billing Channel1
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends routed lead data to the billing Slack channel.
- **Configuration choices:**  
  Similar formatted message to sales/support.
- **Key expressions or variables used:**  
  - `{{ $json.name }}`
  - `{{ $json.email }}`
  - `{{ $json.message }}`
  - `{{ $json.source }}`
- **Input and output connections:**  
  - Input from **Route Lead by Category1** (`billing`)
- **Version-specific requirements:**  
  Uses `typeVersion: 2.4`
- **Edge cases or potential failure types:**  
  - Channel placeholder not configured
  - Slack auth issues
- **Sub-workflow reference:**  
  None

### Node: End1
- **Type and technical role:** `n8n-nodes-base.noOp`  
  Terminates the `other` route cleanly.
- **Configuration choices:**  
  No parameters.
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Input from **Route Lead by Category1** (`other`)
- **Version-specific requirements:**  
  Uses `typeVersion: 1`
- **Edge cases or potential failure types:**  
  None significant
- **Sub-workflow reference:**  
  None

---

## 2.6 Block: Documentation Sticky Notes

**Overview:**  
These nodes are not operational logic. They provide visual guidance for users inside the n8n canvas.

**Nodes Involved:**  
- Sticky Note6
- Sticky Note7
- Sticky Note8
- Sticky Note9
- Sticky Note10
- Sticky Note11

### Node: Sticky Note6
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  General workflow description and setup guidance.
- **Configuration choices:**  
  Documents the workflow purpose, setup tips, and integration list.
- **Input and output connections:**  
  None
- **Edge cases or potential failure types:**  
  None
- **Sub-workflow reference:**  
  None

### Node: Sticky Note7
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Labels Step 1.
- **Configuration choices:**  
  Documents capture and normalization stage.
- **Input and output connections:**  
  None

### Node: Sticky Note8
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Labels Step 2.
- **Configuration choices:**  
  Documents storage and notification stage.
- **Input and output connections:**  
  None

### Node: Sticky Note9
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Labels Step 5.
- **Configuration choices:**  
  Documents routing and team notifications.
- **Input and output connections:**  
  None

### Node: Sticky Note10
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Labels Step 4.
- **Configuration choices:**  
  Documents qualification and classification stage.
- **Input and output connections:**  
  None

### Node: Sticky Note11
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Labels Step 3.
- **Configuration choices:**  
  Documents AI scoring stage.
- **Input and output connections:**  
  None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Gmail Trigger1 | Gmail Trigger | Poll Gmail for incoming emails every minute |  | Get a message1 | ## Step 1 – Lead capture and normalization<br>Captures incoming emails, extracts key fields (name, email, message, thread), and validates required data before processing. |
| Get a message1 | Gmail | Retrieve full message content from Gmail | Gmail Trigger1 | Normalize Incoming Lead Data1 | ## Step 1 – Lead capture and normalization<br>Captures incoming emails, extracts key fields (name, email, message, thread), and validates required data before processing. |
| Normalize Incoming Lead Data1 | Set | Map raw Gmail fields into lead schema | Get a message1 | Validate Lead Data1 | ## Step 1 – Lead capture and normalization<br>Captures incoming emails, extracts key fields (name, email, message, thread), and validates required data before processing. |
| Validate Lead Data1 | If | Ensure required lead fields are present | Normalize Incoming Lead Data1 | Notify New Lead (Slack)1 | ## Step 1 – Lead capture and normalization<br>Captures incoming emails, extracts key fields (name, email, message, thread), and validates required data before processing. |
| Notify New Lead (Slack)1 | Slack | Send immediate Slack alert for new lead | Validate Lead Data1 | Store Lead in Database1 | ## Step 2 – Lead storage and notification<br>Sends a real-time Slack alert for new leads and stores structured lead data in Supabase with status tracking. |
| Store Lead in Database1 | Supabase | Insert new lead into `leads` table | Notify New Lead (Slack)1 | Wait1 | ## Step 2 – Lead storage and notification<br>Sends a real-time Slack alert for new leads and stores structured lead data in Supabase with status tracking. |
| Wait1 | Wait | Delay follow-up processing by 1 hour | Store Lead in Database1 | Fetch Unscored Leads1; Fetch Scored Leads1 |  |
| Fetch Unscored Leads1 | Supabase | Retrieve all leads with `score = 0` | Wait1 | AI Lead Scoring Engine1; Map AI Score Output1 | ## Step 3 – AI lead scoring pipeline<br>Fetches unscored leads, evaluates them using AI scoring logic, and updates the database with score and status. |
| AI Lead Scoring Engine1 | LangChain Agent | Generate lead score and note in structured JSON | Fetch Unscored Leads1; LLM (Scoring)1; Structured Output Parser1 | Map AI Score Output1 | ## Step 3 – AI lead scoring pipeline<br>Fetches unscored leads, evaluates them using AI scoring logic, and updates the database with score and status. |
| LLM (Scoring)1 | Groq Chat Model | Language model backend for scoring agent |  | AI Lead Scoring Engine1 | ## Step 3 – AI lead scoring pipeline<br>Fetches unscored leads, evaluates them using AI scoring logic, and updates the database with score and status. |
| Structured Output Parser1 | Structured Output Parser | Enforce JSON schema for scoring output |  | AI Lead Scoring Engine1 | ## Step 3 – AI lead scoring pipeline<br>Fetches unscored leads, evaluates them using AI scoring logic, and updates the database with score and status. |
| Map AI Score Output1 | Set | Rename score fields for DB update | Fetch Unscored Leads1; AI Lead Scoring Engine1 | Update Lead Score1 | ## Step 3 – AI lead scoring pipeline<br>Fetches unscored leads, evaluates them using AI scoring logic, and updates the database with score and status. |
| Update Lead Score1 | Supabase | Update lead score and set status to `scored` | Map AI Score Output1 |  | ## Step 3 – AI lead scoring pipeline<br>Fetches unscored leads, evaluates them using AI scoring logic, and updates the database with score and status. |
| Fetch Scored Leads1 | Supabase | Retrieve leads with `status = update` for routing | Wait1 | Check Message Exists1 | ## Step 4 – Lead qualification and classification<br>Filters scored leads, ensures valid messages exist, and classifies each lead into categories like sales, support, or billing. |
| Check Message Exists1 | If | Ensure lead has a message before classification | Fetch Scored Leads1 | AI Lead Classification Engine1 | ## Step 4 – Lead qualification and classification<br>Filters scored leads, ensures valid messages exist, and classifies each lead into categories like sales, support, or billing. |
| AI Lead Classification Engine1 | LangChain Agent | Classify lead message into routing category | Check Message Exists1; LLM (Classification)1 | Route Lead by Category1 | ## Step 4 – Lead qualification and classification<br>Filters scored leads, ensures valid messages exist, and classifies each lead into categories like sales, support, or billing. |
| LLM (Classification)1 | Groq Chat Model | Language model backend for classification agent |  | AI Lead Classification Engine1 | ## Step 4 – Lead qualification and classification<br>Filters scored leads, ensures valid messages exist, and classifies each lead into categories like sales, support, or billing. |
| Route Lead by Category1 | Switch | Route classified lead to team-specific branch | AI Lead Classification Engine1 | Send to Sales Channel1; Send to Support Channel1; Send to Billing Channel1; End1 | ## Step 5 – Smart routing and team notifications<br>Routes leads based on AI classification and sends them to the correct Slack channel for faster response and handling. |
| Send to Sales Channel1 | Slack | Send sales-classified lead to sales Slack channel | Route Lead by Category1 |  | ## Step 5 – Smart routing and team notifications<br>Routes leads based on AI classification and sends them to the correct Slack channel for faster response and handling. |
| Send to Support Channel1 | Slack | Send support-classified lead to support Slack channel | Route Lead by Category1 |  | ## Step 5 – Smart routing and team notifications<br>Routes leads based on AI classification and sends them to the correct Slack channel for faster response and handling. |
| Send to Billing Channel1 | Slack | Send billing-classified lead to billing Slack channel | Route Lead by Category1 |  | ## Step 5 – Smart routing and team notifications<br>Routes leads based on AI classification and sends them to the correct Slack channel for faster response and handling. |
| End1 | No Operation | End branch for `other` classifications | Route Lead by Category1 |  | ## Step 5 – Smart routing and team notifications<br>Routes leads based on AI classification and sends them to the correct Slack channel for faster response and handling. |
| Sticky Note6 | Sticky Note | Canvas documentation |  |  | ### AI lead scoring and smart routing with Gmail, Supabase, OpenAI, and Slack<br>Automatically capture leads from Gmail, score them using AI, and route them to the correct team channels.<br><br>**What it does**<br>• Captures inbound leads from Gmail<br>• Stores leads in Supabase database<br>• Uses AI to score and classify leads<br>• Routes leads to Sales, Support, or Billing<br>• Sends real-time Slack notifications<br><br>**Setup**<br>• Connect Gmail, Supabase, Slack, and OpenAI/Groq<br>• Configure Slack channel IDs<br>• Do not hardcode credentials<br><br>**Tip**<br>Use scoring + classification together to prioritize and route leads automatically. |
| Sticky Note7 | Sticky Note | Canvas documentation for step 1 |  |  | ## Step 1 – Lead capture and normalization<br>Captures incoming emails, extracts key fields (name, email, message, thread), and validates required data before processing. |
| Sticky Note8 | Sticky Note | Canvas documentation for step 2 |  |  | ## Step 2 – Lead storage and notification<br>Sends a real-time Slack alert for new leads and stores structured lead data in Supabase with status tracking. |
| Sticky Note9 | Sticky Note | Canvas documentation for step 5 |  |  | ## Step 5 – Smart routing and team notifications<br>Routes leads based on AI classification and sends them to the correct Slack channel for faster response and handling. |
| Sticky Note10 | Sticky Note | Canvas documentation for step 4 |  |  | ## Step 4 – Lead qualification and classification<br>Filters scored leads, ensures valid messages exist, and classifies each lead into categories like sales, support, or billing. |
| Sticky Note11 | Sticky Note | Canvas documentation for step 3 |  |  | ## Step 3 – AI lead scoring pipeline<br>Fetches unscored leads, evaluates them using AI scoring logic, and updates the database with score and status. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it:  
   `Capture, score and route Gmail leads with Groq Llama 3.3, Supabase and Slack`

2. **Add a Gmail Trigger node**
   - Node type: **Gmail Trigger**
   - Name: `Gmail Trigger1`
   - Configure polling to **every minute**
   - Connect Gmail OAuth2 credentials with permission to read mailbox contents

3. **Add a Gmail node to fetch message details**
   - Node type: **Gmail**
   - Name: `Get a message1`
   - Operation: **Get**
   - Set **Simple = false**
   - Set **Message ID** to `={{ $json.threadId }}`
   - Connect it after `Gmail Trigger1`
   - Important: in a robust rebuild, verify whether the trigger returns `messageId` or `threadId`; if available, prefer the actual message ID

4. **Add a Set node for normalization**
   - Node type: **Set**
   - Name: `Normalize Incoming Lead Data1`
   - Create fields:
     - `name` → `={{ $json.from.value[0].address }}`
     - `message` → `={{ $json.text }}`
     - `threadId` → `={{ $json.threadId }}`
     - `source` → `gmail`
     - `gmail` → `={{ $json.from.value[0].address }}`
   - Connect after `Get a message1`

5. **Add an If node for validation**
   - Node type: **If**
   - Name: `Validate Lead Data1`
   - Build an `AND` condition set requiring these fields to be non-empty:
     - `={{ $json.name }}`
     - `={{ $json.message }}`
     - `={{ $json.threadId }}`
     - `={{ $json.source }}`
     - `={{ $json.gmail }}`
   - Connect after `Normalize Incoming Lead Data1`

6. **Add a Slack node for initial notifications**
   - Node type: **Slack**
   - Name: `Notify New Lead (Slack)1`
   - Authentication: **OAuth2**
   - Choose **Send message to channel**
   - Configure channel ID with your notification channel
   - Message text:
     ```text
     New lead received

     Name/from: {{ $json.name }}
     Email: {{ $json.email }}
     Source: {{ $json.source }}
     Message : {{ $json.message }}
     threadvid/id : {{ $json.threadId }}
     ```
   - Connect the **true** output of `Validate Lead Data1` to this node
   - Recommended fix while rebuilding: use `{{ $json.gmail }}` or create an explicit `email` field upstream

7. **Prepare Supabase**
   - Create a Supabase project
   - Create a table named `leads`
   - Include at least these columns:
     - `id` (primary key)
     - `name`
     - `email`
     - `source`
     - `score`
     - `status`
     - `created_at`
     - `message`
     - `thread_id`
   - In n8n, create Supabase credentials with URL and API key/service role key as appropriate

8. **Add a Supabase insert node**
   - Node type: **Supabase**
   - Name: `Store Lead in Database1`
   - Operation: **Insert**
   - Table: `leads`
   - Map fields:
     - `name` → `={{ $json.name }}`
     - `email` → `={{ $json.email }}`
     - `source` → `={{ $json.source }}`
     - `score` → `0`
     - `status` → `new`
     - `created_at` → `={{ new Date().toISOString() }}`
     - `message` → `={{ $json.message }}`
     - `thread_id` → `={{ $json.threadId }}`
   - Connect after `Notify New Lead (Slack)1`
   - Recommended fix while rebuilding: set `email` from `gmail` unless you add a separate `email` field in normalization

9. **Add a Wait node**
   - Node type: **Wait**
   - Name: `Wait1`
   - Wait for:
     - Unit: **hours**
     - Amount: **1**
   - Connect after `Store Lead in Database1`

10. **Add a Supabase node to fetch unscored leads**
    - Node type: **Supabase**
    - Name: `Fetch Unscored Leads1`
    - Operation: **Get Many / Get All**
    - Table: `leads`
    - Filter:
      - `score` equals `0`
    - Set **Return All = true**
    - Connect from `Wait1`

11. **Add the Groq scoring model node**
    - Node type: **Groq Chat Model**
    - Name: `LLM (Scoring)1`
    - Attach Groq API credentials
    - Leave defaults as in the source workflow, or explicitly choose `llama-3.3-70b-versatile` for consistency
    - This node will connect as AI model input, not regular main flow

12. **Add the structured output parser**
    - Node type: **Structured Output Parser**
    - Name: `Structured Output Parser1`
    - Schema type: **Manual**
    - Schema:
      - object with required fields:
        - `score` number
        - `note` string

13. **Add the scoring agent**
    - Node type: **AI Agent**
    - Name: `AI Lead Scoring Engine1`
    - Prompt text:
      ```text
      Analyze the following lead:

      Name: {{$json.name}}
      Email: {{$json.email}}
      Source: {{$json.source}}
      Message: {{$json.message}}

      Return JSON:
      {
        "score": number,
        "note": "short summary"
      }
      ```
    - System message:
      ```text
      You are a lead scoring AI system used in a CRM pipeline.

      Your job is to evaluate the quality of an incoming lead and assign a score between 1 and 100.

      Scoring Criteria:
      - Email Quality:
        - Corporate domain → higher score
        - Free email (gmail, yahoo, outlook) → lower score
      - Name Validity:
        - Real full name → higher score
        - Fake or incomplete → lower score
      - Message Intent:
        - Clear business intent → higher score
        - Generic or vague → lower score
      - Source:
        - Website form or referral → higher score
        - Unknown → lower score

      Rules:
      - Always return valid JSON
      - Score must be an integer between 1 and 100
      - Note should be short (max 10 words)
      - Do not include explanations outside JSON
      ```
    - Enable output parser
    - Connect:
      - Main input from `Fetch Unscored Leads1`
      - AI model input from `LLM (Scoring)1`
      - Parser input from `Structured Output Parser1`

14. **Add a Set node to map scoring output**
    - Node type: **Set**
    - Name: `Map AI Score Output1`
    - Fields:
      - `Score` → `={{ $json.score }}`
      - `message` → `={{ $json.note }}`
    - Connect from `AI Lead Scoring Engine1`
    - Do **not** connect `Fetch Unscored Leads1` directly to this node if you want cleaner logic; the source workflow does, but that can create malformed updates

15. **Add a Supabase update node**
    - Node type: **Supabase**
    - Name: `Update Lead Score1`
    - Operation: **Update**
    - Table: `leads`
    - Filter:
      - `id` equals `={{ $('Fetch Unscored Leads1').item.json.id }}`
    - Update fields:
      - `score` → `={{ $json.Score }}`
      - `status` → `scored`
    - Connect after `Map AI Score Output1`

16. **Add a Supabase node for records awaiting routing**
    - Node type: **Supabase**
    - Name: `Fetch Scored Leads1`
    - Operation: **Get Many / Get All**
    - Table: `leads`
    - Filter:
      - `status` equals `update`
    - Return all rows
    - Connect from `Wait1`
    - Important: this is exactly how the source workflow is configured, but it is logically inconsistent with the scoring branch
    - Recommended fix while rebuilding: filter on `status = scored`, or update the scoring branch to set `status = update`

17. **Add an If node to confirm a message exists**
    - Node type: **If**
    - Name: `Check Message Exists1`
    - Condition:
      - `={{ $json.message }}` is not empty
    - Connect after `Fetch Scored Leads1`

18. **Add the Groq classification model node**
    - Node type: **Groq Chat Model**
    - Name: `LLM (Classification)1`
    - Model: `llama-3.3-70b-versatile`
    - Connect using Groq credentials

19. **Add the classification agent**
    - Node type: **AI Agent**
    - Name: `AI Lead Classification Engine1`
    - Prompt:
      ```text
      Classify this customer message:

      {{$json.message}}
      ```
    - System message:
      ```text
      You are a lead classification system for customer support and sales routing.

      Classify each message into exactly one category:

      - sales → interest in product, demo, pricing
      - support → technical issue or help request
      - billing → payment, invoice, subscription issue
      - other → anything else

      Rules:
      - Output only one word
      - Output must be lowercase
      - No punctuation or explanation
      ```
    - Connect:
      - Main input from `Check Message Exists1`
      - AI model input from `LLM (Classification)1`

20. **Add a Switch node for routing**
    - Node type: **Switch**
    - Name: `Route Lead by Category1`
    - Create four named outputs:
      - `sales` if `={{ $json.output }}` equals `sales`
      - `support` if `={{ $json.output }}` equals `support`
      - `billing` if `={{ $json.output }}` equals `billing`
      - `other` if `={{ $json.output }}` equals `other`
    - Connect after `AI Lead Classification Engine1`
    - Verify the actual output field of your agent; if your n8n version returns a different property, adjust the switch accordingly

21. **Add the sales Slack node**
    - Node type: **Slack**
    - Name: `Send to Sales Channel1`
    - OAuth2 authentication
    - Channel ID: your sales channel
    - Text:
      ```text
      🚀 New Lead Received

      Name: {{ $json.name }}
      Email: {{ $json.email }}
      Message: {{ $json.message }}
      Source: {{ $json.source }}
      ```
    - Connect from the `sales` output of the switch

22. **Add the support Slack node**
    - Node type: **Slack**
    - Name: `Send to Support Channel1`
    - Channel ID: your support channel
    - Use the same message structure
    - Connect from the `support` output

23. **Add the billing Slack node**
    - Node type: **Slack**
    - Name: `Send to Billing Channel1`
    - Channel ID: your billing channel
    - Use the same message structure
    - Connect from the `billing` output

24. **Add a No Operation node**
    - Node type: **No Op**
    - Name: `End1`
    - Connect from the `other` output of the switch

25. **Configure credentials**
    - **Gmail OAuth2**
      - Must allow trigger polling and message reading
    - **Supabase**
      - Must have permission to read and write the `leads` table
    - **Slack OAuth2**
      - Must have permission to post to all configured channels
    - **Groq**
      - Must support the selected chat model(s)

26. **Add optional sticky notes**
    - Create notes matching the workflow sections if you want the same visual organization:
      - General workflow description
      - Step 1 capture/normalization
      - Step 2 storage/notification
      - Step 3 scoring
      - Step 4 classification
      - Step 5 routing

27. **Test the full workflow**
    - Send a Gmail message into the monitored inbox
    - Confirm:
      - Gmail trigger fires
      - Message details are fetched
      - Lead passes validation
      - Slack notification is sent
      - Lead row appears in Supabase
      - Wait resumes after 1 hour
      - Unscored lead is scored
      - Routing branch actually picks up the scored row

28. **Apply recommended corrections before production**
    - Normalize an explicit `email` field, not only `gmail`
    - Use the correct Gmail message identifier
    - Remove the direct connection from `Fetch Unscored Leads1` to `Map AI Score Output1`
    - Align statuses:
      - either set updated rows to `update`
      - or change routing query from `status = update` to `status = scored`
    - Consider filtering the post-wait branches to process only the inserted lead rather than querying all matching rows

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| AI lead scoring and smart routing with Gmail, Supabase, OpenAI, and Slack. Automatically capture leads from Gmail, score them using AI, and route them to the correct team channels. | Canvas note |
| What it does: Captures inbound leads from Gmail; Stores leads in Supabase database; Uses AI to score and classify leads; Routes leads to Sales, Support, or Billing; Sends real-time Slack notifications. | Canvas note |
| Setup: Connect Gmail, Supabase, Slack, and OpenAI/Groq; Configure Slack channel IDs; Do not hardcode credentials. | Canvas note |
| Tip: Use scoring + classification together to prioritize and route leads automatically. | Canvas note |