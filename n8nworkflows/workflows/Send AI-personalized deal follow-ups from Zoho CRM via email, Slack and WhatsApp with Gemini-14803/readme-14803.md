Send AI-personalized deal follow-ups from Zoho CRM via email, Slack and WhatsApp with Gemini

https://n8nworkflows.xyz/workflows/send-ai-personalized-deal-follow-ups-from-zoho-crm-via-email--slack-and-whatsapp-with-gemini-14803


# Send AI-personalized deal follow-ups from Zoho CRM via email, Slack and WhatsApp with Gemini

# 1. Workflow Overview

## Purpose

This workflow automates follow-up handling for Zoho CRM deals that are overdue for contact. It scans deals on a schedule, identifies those that are eligible for follow-up, uses Gemini to generate a personalized outreach message, decides the best outreach channel based on inactivity, sends the message through email, Slack, or WhatsApp, then creates a Zoho CRM task and updates the deal follow-up status.

## Target use cases

- Sales teams managing a Zoho CRM pipeline
- Automated re-engagement of stale or pending deals
- AI-assisted follow-up drafting with channel selection logic
- Multi-channel outbound follow-up linked back to CRM operations

## Logical blocks

### 1.1 Scheduled Deal Retrieval
The workflow starts on a cron schedule, computes a date cutoff value, and retrieves all deals from Zoho CRM.

### 1.2 Follow-up Eligibility Evaluation
Each deal is checked to determine whether automated follow-up is enabled, the status is still pending, and the next due date has passed.

### 1.3 AI Message Generation
For eligible deals, Gemini generates structured follow-up content in JSON format, including subject, message, channel suggestion, priority, and next action.

### 1.4 Decision and Channel Selection
A code node combines AI output with CRM activity data to determine the actual follow-up channel, overriding the AI suggestion when inactivity thresholds are exceeded.

### 1.5 Outbound Messaging
Depending on the selected channel, the workflow sends the follow-up by Gmail, Slack, or WhatsApp.

### 1.6 CRM Task Creation and Deal Update
After sending the message, the workflow creates a follow-up task in Zoho CRM and marks the deal’s follow-up status as sent.

---

# 2. Block-by-Block Analysis

## 2.1 Scheduled Deal Retrieval

### Overview
This block initiates the workflow on a time schedule and prepares the initial context before retrieving deals from Zoho CRM. Although a cutoff date is computed, it is not actually consumed downstream in the current version.

### Nodes Involved
- Daily Follow-up Scan Trigger
- Set Inactivity Cutoff Window
- Fetch Active Deals from Zoho CRM

### Node Details

#### Daily Follow-up Scan Trigger
- **Type and role:** `n8n-nodes-base.cron`; scheduled entry point.
- **Configuration choices:** Runs at hour `9` each day.
- **Key expressions or variables used:** None.
- **Input and output connections:** Entry node → outputs to `Set Inactivity Cutoff Window`.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - Workflow must be active for the trigger to execute.
  - Timezone behavior depends on workflow/server timezone settings.
- **Sub-workflow reference:** None.

#### Set Inactivity Cutoff Window
- **Type and role:** `n8n-nodes-base.set`; computes a derived date value.
- **Configuration choices:** Assigns a string field named `cutoffDate`.
- **Key expressions or variables used:**
  - `={{ new Date(Date.now() - 7 * 24 * 60 * 60 * 1000).toISOString().slice(0,19) }}`
- **Interpretation:** Generates a timestamp representing 7 days before the current moment, trimmed to seconds.
- **Input and output connections:** Input from `Daily Follow-up Scan Trigger` → output to `Fetch Active Deals from Zoho CRM`.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - The generated value is unused later, so this node currently has no functional effect.
  - Time formatting may not match Zoho field expectations if later reused.
- **Sub-workflow reference:** None.

#### Fetch Active Deals from Zoho CRM
- **Type and role:** `n8n-nodes-base.zohoCrm`; retrieves deal records.
- **Configuration choices:**
  - Resource: `deal`
  - Operation: `getAll`
  - Return All: enabled
- **Key expressions or variables used:** None.
- **Interpretation:** Fetches all deals, not only active deals despite the node name.
- **Input and output connections:** Input from `Set Inactivity Cutoff Window` → output to `Evaluate Follow-up Eligibility`.
- **Version-specific requirements:** Type version `1`; requires Zoho OAuth2 credentials.
- **Edge cases or potential failure types:**
  - OAuth token expiration or insufficient Zoho scopes
  - Large deal volumes may increase execution time or hit API paging/limit constraints
  - No server-side filter is applied, so filtering is entirely downstream
- **Sub-workflow reference:** None.

---

## 2.2 Follow-up Eligibility Evaluation

### Overview
This block inspects each deal individually and marks whether it is due for automated follow-up. Only deals that are enabled, pending, and already due continue to the AI stage.

### Nodes Involved
- Evaluate Follow-up Eligibility
- Is Follow-up Due?

### Node Details

#### Evaluate Follow-up Eligibility
- **Type and role:** `n8n-nodes-base.code`; per-item decision logic.
- **Configuration choices:** `runOnceForEachItem`.
- **Key expressions or variables used:**
  - Reads current item via `const d = $input.item.json;`
  - Checks:
    - `d.Auto_Follow_up_Enabled`
    - `d.Follow_up_Status !== 'Pending'`
    - `d.Next_Follow_up_Due`
  - Computes:
    - `dueDate = new Date(d.Next_Follow_up_Due)`
    - `due: dueDate <= now`
  - Returns:
    - `dealId: d.id`
    - `dealName: d.Deal_Name`
    - `raw: d`
- **Interpretation:** Produces a simplified payload for downstream processing while retaining the original deal under `raw`.
- **Input and output connections:** Input from `Fetch Active Deals from Zoho CRM` → output to `Is Follow-up Due?`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - Missing or malformed `Next_Follow_up_Due` can yield `Invalid Date`
  - Unexpected field names if Zoho schema differs from assumptions
  - Boolean coercion issues if `Auto_Follow_up_Enabled` is stored as text rather than boolean
- **Sub-workflow reference:** None.

#### Is Follow-up Due?
- **Type and role:** `n8n-nodes-base.if`; filters eligible deals.
- **Configuration choices:**
  - Single boolean condition
  - Checks whether `{{$json.due}}` is `true`
- **Key expressions or variables used:**
  - `={{$json.due}}`
- **Input and output connections:**
  - Input from `Evaluate Follow-up Eligibility`
  - True branch → `Generate Personalized Follow-up (AI)`
  - False branch → no connection
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**
  - If `due` is missing or not boolean, strict validation may fail or route unexpectedly
  - All ineligible deals are silently dropped
- **Sub-workflow reference:** None.

---

## 2.3 AI Message Generation

### Overview
This block asks Gemini to generate follow-up content in a strict JSON structure, then parses the output into usable fields. It uses a structured output parser to reduce formatting inconsistency.

### Nodes Involved
- Generate Personalized Follow-up (AI)
- AI Model (Gemini)
- Parse AI Response (Structured JSON)

### Node Details

#### Generate Personalized Follow-up (AI)
- **Type and role:** `@n8n/n8n-nodes-langchain.chainLlm`; orchestrates prompt + model + parser.
- **Configuration choices:**
  - Prompt type: defined manually
  - Has output parser: enabled
  - Includes a system-style message instructing the model to be short, professional, business tone, no emojis
- **Key expressions or variables used:**
  - Prompt includes:
    - `{{ $json.dealName }}`
    - `{{ $json.raw.Account_Name.name }}`
    - `{{ $json.raw.Stage }}`
    - `{{ $json.raw.Last_Follow_up_At }}`
  - Requires output JSON:
    - `subject`
    - `message`
    - `channel`
    - `priority`
    - `next_action`
- **Input and output connections:**
  - Main input from `Is Follow-up Due?`
  - Language model input from `AI Model (Gemini)`
  - Output parser input from `Parse AI Response (Structured JSON)`
  - Main output to `Decision Engine`
- **Version-specific requirements:** Type version `1.7`; requires compatible LangChain nodes installed in the n8n instance.
- **Edge cases or potential failure types:**
  - Missing nested fields such as `raw.Account_Name.name` can break expressions
  - LLM may still return malformed JSON despite instructions
  - Prompt/content mismatch if CRM fields are null
  - Gemini credential or API quota issues
- **Sub-workflow reference:** None.

#### AI Model (Gemini)
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`; language model provider.
- **Configuration choices:** Uses default options.
- **Key expressions or variables used:** None.
- **Input and output connections:** Connected as `ai_languageModel` to `Generate Personalized Follow-up (AI)`.
- **Version-specific requirements:** Type version `1`; requires Google Gemini/PaLM API credentials.
- **Edge cases or potential failure types:**
  - Authentication errors
  - Quota exhaustion
  - Safety filtering or model-side refusals
  - Regional availability issues
- **Sub-workflow reference:** None.

#### Parse AI Response (Structured JSON)
- **Type and role:** `@n8n/n8n-nodes-langchain.outputParserStructured`; validates/structures LLM output.
- **Configuration choices:** JSON schema example defines expected keys:
  - `subject`
  - `message`
  - `channel`
  - `priority`
  - `next_action`
- **Key expressions or variables used:** None.
- **Input and output connections:** Connected as `ai_outputParser` to `Generate Personalized Follow-up (AI)`.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:**
  - Parser failure if model output is not valid JSON
  - Schema drift if prompt asks for fields not aligned with parser expectations
- **Sub-workflow reference:** None.

---

## 2.4 Decision and Channel Selection

### Overview
This block applies business rules on top of the AI result. It calculates inactivity duration from CRM timestamps and can override the AI-suggested channel to WhatsApp or call based on how stale the deal is.

### Nodes Involved
- Decision Engine
- Channel Routing

### Node Details

#### Decision Engine
- **Type and role:** `n8n-nodes-base.code`; applies deterministic routing logic.
- **Configuration choices:** `runOnceForEachItem`.
- **Key expressions or variables used:**
  - Retrieves original deal:
    - `const deal = $('Is Follow-up Due?').item.json.raw;`
  - Retrieves AI result:
    - `const ai = $json.output || {};`
  - Resolves activity timestamp from:
    - `deal.Last_Activity_Time`
    - `deal.Last_Follow_up_At`
    - `deal.Created_Time`
  - Calculates:
    - `daysSinceLastActivity`
  - Routing logic:
    - default `ai.channel || "email"`
    - if `daysSinceLastActivity > 14` → `"call"`
    - else if `daysSinceLastActivity > 7` → `"whatsapp"`
  - Returns `followUpDecision` with:
    - `lastActivityTime`
    - `followUpType`
    - `daysSinceLastActivity`
    - `aiSuggestedChannel`
    - `priority`
    - `nextAction`
- **Input and output connections:** Input from `Generate Personalized Follow-up (AI)` → output to `Channel Routing`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - `$('Is Follow-up Due?')` item-linking assumptions can break if item pairing changes
  - `call` may be selected, but no switch branch exists for it
  - Invalid date parsing may produce `NaN`
  - If `output` is absent, defaults are used
- **Sub-workflow reference:** None.

#### Channel Routing
- **Type and role:** `n8n-nodes-base.switch`; routes each item to a delivery channel.
- **Configuration choices:**
  - Rule 1: output `Email` when `followUpType == email`
  - Rule 2: output `slack` when `followUpType == slack`
  - Rule 3: output `Whatsapp` when `followUpType == whatsapp`
  - Loose type validation enabled
- **Key expressions or variables used:**
  - `={{ $json.followUpDecision.followUpType }}`
- **Input and output connections:**
  - Input from `Decision Engine`
  - Output 0 → `Send follow-up email`
  - Output 1 → `Send Slack follow-up`
  - Output 2 → `Send WhatsApp Follow-up`
- **Version-specific requirements:** Type version `3.3`.
- **Edge cases or potential failure types:**
  - No route for `call`, even though Decision Engine can choose it
  - Unrecognized channels are dropped silently
  - Case mismatches would fail routing
- **Sub-workflow reference:** None.

---

## 2.5 Outbound Messaging

### Overview
This block sends the AI-generated message through the channel selected by the switch. All three channel nodes converge to the CRM task creation step.

### Nodes Involved
- Send follow-up email
- Send Slack follow-up
- Send WhatsApp Follow-up

### Node Details

#### Send follow-up email
- **Type and role:** `n8n-nodes-base.gmail`; sends an email.
- **Configuration choices:**
  - Subject: `={{ $json.output.subject }}`
  - Message: `={{ $json.output.message }}`
  - Recipient (`sendTo`): empty expression `"="`
- **Key expressions or variables used:**
  - `{{$json.output.subject}}`
  - `{{$json.output.message}}`
- **Input and output connections:** Input from `Channel Routing` email branch → output to `Create CRM Follow-up Task`.
- **Version-specific requirements:** Type version `2.1`; requires Gmail OAuth2 credentials.
- **Edge cases or potential failure types:**
  - This node is currently incomplete because `sendTo` is blank
  - Gmail auth/scope errors
  - Missing subject/message if parser output failed upstream
- **Sub-workflow reference:** None.

#### Send Slack follow-up
- **Type and role:** `n8n-nodes-base.slack`; sends a Slack DM/message to a selected user.
- **Configuration choices:**
  - Authentication: OAuth2
  - Select mode: `user`
  - User ID: placeholder `{userid}`
  - Text: `={{ $json.output.message }}`
- **Key expressions or variables used:**
  - `{{$json.output.message}}`
- **Input and output connections:** Input from `Channel Routing` Slack branch → output to `Create CRM Follow-up Task`.
- **Version-specific requirements:** Type version `2.3`; requires Slack OAuth2 credentials.
- **Edge cases or potential failure types:**
  - Placeholder `{userid}` must be replaced with an actual Slack user ID
  - Bot/user permissions may block direct message delivery
  - No mapping from Zoho owner/contact to Slack user is implemented
- **Sub-workflow reference:** None.

#### Send WhatsApp Follow-up
- **Type and role:** `n8n-nodes-base.whatsApp`; sends a WhatsApp message.
- **Configuration choices:**
  - Operation: `send`
  - Text body: `={{ $json.output.message }}`
- **Key expressions or variables used:**
  - `{{$json.output.message}}`
- **Input and output connections:** Input from `Channel Routing` WhatsApp branch → output to `Create CRM Follow-up Task`.
- **Version-specific requirements:** Type version `1.1`; requires a supported WhatsApp integration setup in n8n.
- **Edge cases or potential failure types:**
  - Recipient configuration is missing in the shown parameters
  - WhatsApp provider/business account setup may be required
  - Template restrictions may apply depending on provider/session state
- **Sub-workflow reference:** None.

---

## 2.6 CRM Task Creation and Deal Update

### Overview
After a follow-up is sent, the workflow records a task in Zoho CRM and updates the deal to mark the follow-up as sent. This closes the automation loop inside the CRM.

### Nodes Involved
- Create CRM Follow-up Task
- Update Deal Follow-up Status

### Node Details

#### Create CRM Follow-up Task
- **Type and role:** `n8n-nodes-base.httpRequest`; manual HTTP call to Zoho CRM Tasks API.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://www.zohoapis.com/crm/v2/Tasks`
  - Authentication: predefined credential type using `zohoOAuth2Api`
  - Body type: JSON
- **Key expressions or variables used:**
  - Subject uses `$('Is Follow-up Due?').item.json.dealName`
  - Due date uses `{{ new Date().toISOString().split('T')[0] }}`
  - What_Id uses `$('Is Follow-up Due?').item.json.dealId`
  - Owner ID uses `$('Is Follow-up Due?').item.json.raw.Owner.id`
- **Interpretation:** Creates a high-priority task linked to the Zoho Deal.
- **Input and output connections:** Input from all three message nodes → output to `Update Deal Follow-up Status`.
- **Version-specific requirements:** Type version `4.3`.
- **Edge cases or potential failure types:**
  - Item-linking assumptions across branches may fail in some execution patterns
  - Owner may be missing
  - Zoho API may reject payload if module link fields are incorrect
  - Regional Zoho domain may differ from `.com`
- **Sub-workflow reference:** None.

#### Update Deal Follow-up Status
- **Type and role:** `n8n-nodes-base.zohoCrm`; updates the deal after task creation.
- **Configuration choices:**
  - Resource: `deal`
  - Operation: `update`
  - Deal ID: `={{ $('Is Follow-up Due?').item.json.dealId }}`
  - Custom field update:
    - `Follow_up_Status = Sent`
- **Key expressions or variables used:**
  - `{{$('Is Follow-up Due?').item.json.dealId}}`
- **Input and output connections:** Input from `Create CRM Follow-up Task` → no downstream nodes.
- **Version-specific requirements:** Type version `1`; requires Zoho OAuth2 credentials.
- **Edge cases or potential failure types:**
  - Custom field API name must exactly match `Follow_up_Status`
  - Deal ID linkage depends on item pairing
  - Update may fail if task creation succeeded but item context is lost
- **Sub-workflow reference:** None.

---

## 2.7 Documentation / Sticky Notes Block

### Overview
These nodes provide in-canvas documentation for the workflow. They do not affect execution but are important for understanding setup intent and expected configuration.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4

### Node Details

#### Sticky Note
- **Type and role:** `n8n-nodes-base.stickyNote`; general workflow documentation.
- **Configuration choices:** Large note containing setup steps and workflow summary.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None operational.
- **Sub-workflow reference:** None.

#### Sticky Note1
- **Type and role:** `n8n-nodes-base.stickyNote`; block label for deal collection.
- **Configuration choices:** Describes trigger and Zoho deal retrieval.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None operational.
- **Sub-workflow reference:** None.

#### Sticky Note2
- **Type and role:** `n8n-nodes-base.stickyNote`; block label for eligibility detection.
- **Configuration choices:** Describes inactivity calculation and filtering.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None operational.
- **Sub-workflow reference:** None.

#### Sticky Note3
- **Type and role:** `n8n-nodes-base.stickyNote`; block label for AI and follow-up decisioning.
- **Configuration choices:** Describes AI evaluation block.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None operational.
- **Sub-workflow reference:** None.

#### Sticky Note4
- **Type and role:** `n8n-nodes-base.stickyNote`; block label for routing and CRM actions.
- **Configuration choices:** Describes channel routing and task creation.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None operational.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily Follow-up Scan Trigger | Cron | Scheduled workflow entry point |  | Set Inactivity Cutoff Window | ## Deal Data Collection<br>Triggers the workflow on schedule and fetches active Deals from Zoo CRM for processing. |
| Set Inactivity Cutoff Window | Set | Computes a 7-day-back cutoff timestamp | Daily Follow-up Scan Trigger | Fetch Active Deals from Zoho CRM | ## Deal Data Collection<br>Triggers the workflow on schedule and fetches active Deals from Zoo CRM for processing. |
| Fetch Active Deals from Zoho CRM | Zoho CRM | Retrieves all deal records from Zoho CRM | Set Inactivity Cutoff Window | Evaluate Follow-up Eligibility | ## Deal Data Collection<br>Triggers the workflow on schedule and fetches active Deals from Zoo CRM for processing. |
| Evaluate Follow-up Eligibility | Code | Determines whether each deal is eligible and due for follow-up | Fetch Active Deals from Zoho CRM | Is Follow-up Due? | ## Deal Processing & Inactivity Detection<br>Splits deals, calculates inactivity and filters only stalled deals that need follow-up. |
| Is Follow-up Due? | If | Filters to only deals whose due flag is true | Evaluate Follow-up Eligibility | Generate Personalized Follow-up (AI) | ## Deal Processing & Inactivity Detection<br>Splits deals, calculates inactivity and filters only stalled deals that need follow-up. |
| Generate Personalized Follow-up (AI) | LangChain LLM Chain | Builds prompt, invokes model, and parses structured AI response | Is Follow-up Due?; AI Model (Gemini); Parse AI Response (Structured JSON) | Decision Engine | ## AI Evaluation & Follow-Up Decision<br>Splits deals, calculates inactivity and filters only stalled deals that need follow-up. |
| AI Model (Gemini) | LangChain Google Gemini Chat Model | Provides the LLM used for content generation |  | Generate Personalized Follow-up (AI) | ## AI Evaluation & Follow-Up Decision<br>Splits deals, calculates inactivity and filters only stalled deals that need follow-up. |
| Parse AI Response (Structured JSON) | LangChain Structured Output Parser | Enforces expected JSON output shape from AI |  | Generate Personalized Follow-up (AI) | ## AI Evaluation & Follow-Up Decision<br>Splits deals, calculates inactivity and filters only stalled deals that need follow-up. |
| Decision Engine | Code | Applies inactivity-based channel override and decision metadata | Generate Personalized Follow-up (AI) | Channel Routing | ## AI Evaluation & Follow-Up Decision<br>Splits deals, calculates inactivity and filters only stalled deals that need follow-up. |
| Channel Routing | Switch | Routes items to email, Slack, or WhatsApp branches | Decision Engine | Send follow-up email; Send Slack follow-up; Send WhatsApp Follow-up | ## Channel Routing & CRM Actions<br>Routes deals to Email/WhatsApp/Call and creates alerts & CRM tasks for action. |
| Send follow-up email | Gmail | Sends follow-up by email | Channel Routing | Create CRM Follow-up Task | ## Channel Routing & CRM Actions<br>Routes deals to Email/WhatsApp/Call and creates alerts & CRM tasks for action. |
| Send Slack follow-up | Slack | Sends follow-up via Slack | Channel Routing | Create CRM Follow-up Task | ## Channel Routing & CRM Actions<br>Routes deals to Email/WhatsApp/Call and creates alerts & CRM tasks for action. |
| Send WhatsApp Follow-up | WhatsApp | Sends follow-up via WhatsApp | Channel Routing | Create CRM Follow-up Task | ## Channel Routing & CRM Actions<br>Routes deals to Email/WhatsApp/Call and creates alerts & CRM tasks for action. |
| Create CRM Follow-up Task | HTTP Request | Creates a Zoho CRM task linked to the deal | Send follow-up email; Send Slack follow-up; Send WhatsApp Follow-up | Update Deal Follow-up Status | ## Channel Routing & CRM Actions<br>Routes deals to Email/WhatsApp/Call and creates alerts & CRM tasks for action. |
| Update Deal Follow-up Status | Zoho CRM | Marks the deal follow-up status as sent | Create CRM Follow-up Task |  | ## Channel Routing & CRM Actions<br>Routes deals to Email/WhatsApp/Call and creates alerts & CRM tasks for action. |
| Sticky Note | Sticky Note | Canvas documentation |  |  | ## Zoho CRM – Smart Follow-up Sequence Optinizer<br>**How it works**<br>This workflow runs weekly and Automatically detect inactive deals and trigger intelligent, AI-driven follow-ups across multiple channels to improve deal conversion and prevent pipeline leakage.<br><br>**Setup steps**<br>1. Configure Zoho CRM credentials in all Zoho nodes (OAuth2 recommended).<br><br>2. Verify Deal Fields Mapping, Add Custom fields in Zoho Deal : Last Activity Time, Last Follow-up Date, Created Date<br><br>3. Configure AI Node (OpenAI/Gemini)<br><br>4. Validate Function Node logic: correctly calculates daysSinceLastActivity<br><br>5. Set up Switch Node conditions : Email, WhatsApp,Slack or Call<br><br>6. Configure Email, WhatsApp, Slack or Call credentials.<br><br>7. Configure Zoho CRM Task Node |
| Sticky Note1 | Sticky Note | Canvas documentation for deal collection block |  |  | ## Deal Data Collection<br>Triggers the workflow on schedule and fetches active Deals from Zoo CRM for processing. |
| Sticky Note2 | Sticky Note | Canvas documentation for deal processing block |  |  | ## Deal Processing & Inactivity Detection<br>Splits deals, calculates inactivity and filters only stalled deals that need follow-up. |
| Sticky Note3 | Sticky Note | Canvas documentation for AI block |  |  | ## AI Evaluation & Follow-Up Decision<br>Splits deals, calculates inactivity and filters only stalled deals that need follow-up. |
| Sticky Note4 | Sticky Note | Canvas documentation for routing/CRM block |  |  | ## Channel Routing & CRM Actions<br>Routes deals to Email/WhatsApp/Call and creates alerts & CRM tasks for action. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: `Zoho CRM – Smart Follow-up Sequence Optimizer`.

2. **Add a Cron node**
   - Node type: `Cron`
   - Name: `Daily Follow-up Scan Trigger`
   - Set it to run once per day at hour `9`.
   - This is the workflow entry point.

3. **Add a Set node**
   - Node type: `Set`
   - Name: `Set Inactivity Cutoff Window`
   - Add one field:
     - `cutoffDate` as string
     - Value:
       ```javascript
       {{ new Date(Date.now() - 7 * 24 * 60 * 60 * 1000).toISOString().slice(0,19) }}
       ```
   - Connect `Daily Follow-up Scan Trigger` → `Set Inactivity Cutoff Window`.

4. **Add a Zoho CRM node to fetch deals**
   - Node type: `Zoho CRM`
   - Name: `Fetch Active Deals from Zoho CRM`
   - Resource: `Deal`
   - Operation: `Get Many` / `Get All`
   - Enable `Return All`.
   - Configure Zoho OAuth2 credentials.
   - Connect `Set Inactivity Cutoff Window` → `Fetch Active Deals from Zoho CRM`.

5. **Configure Zoho credentials**
   - Create or select a `Zoho OAuth2` credential.
   - Ensure access to CRM Deals and Tasks APIs.
   - If your Zoho account is regional, confirm the correct API domain.

6. **Add a Code node for eligibility logic**
   - Node type: `Code`
   - Name: `Evaluate Follow-up Eligibility`
   - Mode: `Run Once for Each Item`
   - Paste this logic:
     ```javascript
     const d = $input.item.json;
     const now = new Date();

     if (!d.Auto_Follow_up_Enabled) return { json: { due: false } };
     if (d.Follow_up_Status !== 'Pending') return { json: { due: false } };
     if (!d.Next_Follow_up_Due) return { json: { due: false } };

     const dueDate = new Date(d.Next_Follow_up_Due);

     return {
       json: {
         due: dueDate <= now,
         dealId: d.id,
         dealName: d.Deal_Name,
         raw: d
       }
     };
     ```
   - Connect `Fetch Active Deals from Zoho CRM` → `Evaluate Follow-up Eligibility`.

7. **Add an If node**
   - Node type: `If`
   - Name: `Is Follow-up Due?`
   - Condition:
     - Value 1: `{{ $json.due }}`
     - Operation: `is true`
   - Connect `Evaluate Follow-up Eligibility` → `Is Follow-up Due?`.

8. **Add the AI chain node**
   - Node type: `Basic LLM Chain` / `LangChain LLM Chain`
   - Name: `Generate Personalized Follow-up (AI)`
   - Prompt type: defined text
   - Enable structured output parsing
   - Prompt text:
     ```text
     You are a CRM assistant.

     Generate a personalized follow-up message for this deal.

     Deal Name: {{ $json.dealName }}
     Account: {{ $json.raw.Account_Name.name }}
     Stage: {{ $json.raw.Stage }}
     Last Follow-up: {{ $json.raw.Last_Follow_up_At }}

     Return ONLY valid JSON in the following format:

     {
       "subject": "...",
       "message": "...",
       "channel": "email",
       "priority": "High",
       "next_action": "send_followup"
     }
     Return ONLY valid JSON.Do not add explanations.
     Do not add markdown.
     ```
   - Add an instruction message:
     - `You are a CRM follow-up assistant. Generate short, professional follow-up content. No emojis. No fluff. Business tone.`

9. **Add a Gemini chat model node**
   - Node type: `Google Gemini Chat Model`
   - Name: `AI Model (Gemini)`
   - Keep default options unless needed.
   - Configure Google Gemini / PaLM API credentials.
   - Connect this node to the AI chain as the language model input.

10. **Configure Gemini credentials**
    - Create/select Google Gemini-compatible credentials.
    - Ensure your API key/project has model access and sufficient quota.

11. **Add a structured output parser node**
    - Node type: `Structured Output Parser`
    - Name: `Parse AI Response (Structured JSON)`
    - Schema example:
      ```json
      {
        "subject": "...",
        "message": "...",
        "channel": "email",
        "priority": "High",
        "next_action": "send_followup"
      }
      ```
    - Connect this node to the AI chain as the output parser input.

12. **Connect the due branch to the AI chain**
    - Connect the `true` output of `Is Follow-up Due?` → `Generate Personalized Follow-up (AI)`.

13. **Add a Code node for decision logic**
    - Node type: `Code`
    - Name: `Decision Engine`
    - Mode: `Run Once for Each Item`
    - Paste:
      ```javascript
      const deal = $('Is Follow-up Due?').item.json.raw;
      const ai = $json.output || {};

      const lastActivityTime =
        deal.Last_Activity_Time ||
        deal.Last_Follow_up_At ||
        deal.Created_Time ||
        null;

      let daysSinceLastActivity = null;

      if (lastActivityTime) {
        const lastActivity = new Date(lastActivityTime);
        daysSinceLastActivity = Math.floor(
          (Date.now() - lastActivity.getTime()) / (1000 * 60 * 60 * 24)
        );
      }

      let followUpType = ai.channel || "email";

      if (daysSinceLastActivity !== null) {
        if (daysSinceLastActivity > 14) {
          followUpType = "call";
        } else if (daysSinceLastActivity > 7) {
          followUpType = "whatsapp";
        }
      }

      return {
        ...$json,
        followUpDecision: {
          lastActivityTime,
          followUpType,
          daysSinceLastActivity,
          aiSuggestedChannel: ai.channel || null,
          priority: ai.priority || "Normal",
          nextAction: ai.next_action || null,
        }
      };
      ```
   - Connect `Generate Personalized Follow-up (AI)` → `Decision Engine`.

14. **Add a Switch node**
   - Node type: `Switch`
   - Name: `Channel Routing`
   - Route on expression:
     - `{{ $json.followUpDecision.followUpType }}`
   - Create rules:
     - `email` → output named `Email`
     - `slack` → output named `slack`
     - `whatsapp` → output named `Whatsapp`
   - Enable loose type validation if available.
   - Connect `Decision Engine` → `Channel Routing`.

15. **Add a Gmail node**
   - Node type: `Gmail`
   - Name: `Send follow-up email`
   - Operation: send email
   - Subject: `{{ $json.output.subject }}`
   - Body/message: `{{ $json.output.message }}`
   - Recipient: configure a real target email field or static address
   - Important: the provided workflow JSON has an empty `sendTo`; you must fix this to make the node usable.
   - Connect the email output of `Channel Routing` → `Send follow-up email`.

16. **Configure Gmail credentials**
   - Create/select Gmail OAuth2 credentials.
   - Confirm send-mail permissions.

17. **Add a Slack node**
   - Node type: `Slack`
   - Name: `Send Slack follow-up`
   - Authentication: OAuth2
   - Message text: `{{ $json.output.message }}`
   - Target type: user
   - User ID: replace placeholder `{userid}` with a real Slack user ID, or dynamically map it from CRM data.
   - Connect the Slack output of `Channel Routing` → `Send Slack follow-up`.

18. **Configure Slack credentials**
   - Create/select Slack OAuth2 credentials.
   - Ensure the app can message the intended recipient/user/channel.

19. **Add a WhatsApp node**
   - Node type: `WhatsApp`
   - Name: `Send WhatsApp Follow-up`
   - Operation: send
   - Text body: `{{ $json.output.message }}`
   - Configure recipient details required by your WhatsApp provider/node version.
   - Connect the WhatsApp output of `Channel Routing` → `Send WhatsApp Follow-up`.

20. **Configure WhatsApp access**
   - Set up the supported WhatsApp credential/provider in n8n.
   - Provide the phone number or dynamic recipient mapping.
   - Check business/template requirements.

21. **Add an HTTP Request node for Zoho task creation**
   - Node type: `HTTP Request`
   - Name: `Create CRM Follow-up Task`
   - Method: `POST`
   - URL: `https://www.zohoapis.com/crm/v2/Tasks`
   - Authentication: predefined credential type
   - Credential type: `Zoho OAuth2`
   - Send body as JSON
   - JSON body:
     ```json
     {
       "data": [
         {
           "Subject": "Follow-up: {{ $('Is Follow-up Due?').item.json.dealName }}",
           "Due_Date": "{{ new Date().toISOString().split('T')[0] }}",
           "Status": "Not Started",
           "Priority": "High",
           "What_Id": "{{ $('Is Follow-up Due?').item.json.dealId }}",
           "$se_module": "Deals",
           "Owner": {
             "id": "{{ $('Is Follow-up Due?').item.json.raw.Owner.id }}"
           },
           "Description": "Auto-generated follow-up task for overdue deal."
         }
       ]
     }
     ```
   - Connect all three messaging nodes into this node:
     - `Send follow-up email` → `Create CRM Follow-up Task`
     - `Send Slack follow-up` → `Create CRM Follow-up Task`
     - `Send WhatsApp Follow-up` → `Create CRM Follow-up Task`

22. **Add a Zoho CRM update node**
   - Node type: `Zoho CRM`
   - Name: `Update Deal Follow-up Status`
   - Resource: `Deal`
   - Operation: `Update`
   - Deal ID:
     - `{{ $('Is Follow-up Due?').item.json.dealId }}`
   - Update custom field:
     - `Follow_up_Status = Sent`
   - Connect `Create CRM Follow-up Task` → `Update Deal Follow-up Status`.

23. **Optionally add sticky notes**
   - Add notes for:
     - Overall setup
     - Deal data collection
     - Deal processing
     - AI evaluation
     - Channel routing and CRM actions

24. **Test with sample deals**
   - Ensure the Zoho deals contain:
     - `Auto_Follow_up_Enabled`
     - `Follow_up_Status`
     - `Next_Follow_up_Due`
     - `Deal_Name`
     - `Account_Name`
     - `Stage`
     - `Last_Follow_up_At`
     - `Last_Activity_Time`
     - `Created_Time`
     - `Owner.id`

25. **Fix the incomplete parts before production**
   - Provide an actual email recipient
   - Provide Slack user mapping
   - Provide WhatsApp recipient mapping
   - Add a `call` route if keeping the `call` override in the decision logic
   - Optionally filter deals directly in Zoho rather than retrieving all deals

26. **Activate the workflow**
   - Save and enable it once all credentials and recipient mappings are complete.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Workflow title in sticky note is written as “Zoho CRM – Smart Follow-up Sequence Optinizer”, which differs slightly from the workflow name and includes a typo. | General documentation consistency |
| The sticky note says the workflow “runs weekly”, but the Cron node is configured to run daily at 09:00. | Scheduling mismatch |
| The sticky note says routing includes Call, but no Call branch exists in the Switch node. | Logic/design mismatch |
| The `Set Inactivity Cutoff Window` node currently computes a value that is not used anywhere downstream. | Optimization / cleanup note |
| `Fetch Active Deals from Zoho CRM` does not actually filter active deals; it retrieves all deals and relies on later filtering. | Zoho retrieval behavior |
| Gmail recipient is blank, Slack user is a placeholder, and WhatsApp recipient data is not configured in the shown node settings. | Required before production use |
| The workflow relies on specific Zoho custom fields existing with exact API names, especially `Follow_up_Status`, `Auto_Follow_up_Enabled`, `Next_Follow_up_Due`, `Last_Follow_up_At`, and possibly `Last_Activity_Time`. | Zoho CRM schema dependency |
| The workflow uses item references like `$('Is Follow-up Due?').item...` across multiple downstream nodes. This works only if item linking remains stable; it should be tested carefully after any changes. | n8n data-linking consideration |