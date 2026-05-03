Classify cold email replies and notify via Telegram with OpenAI and Instantly

https://n8nworkflows.xyz/workflows/classify-cold-email-replies-and-notify-via-telegram-with-openai-and-instantly-14220


# Classify cold email replies and notify via Telegram with OpenAI and Instantly

# 1. Workflow Overview

This workflow receives reply events from Instantly, validates and normalizes the incoming payload, classifies the reply sentiment/intent with OpenAI, routes the lead into HOT/WARM/COLD branches, sends Telegram notifications, auto-acknowledges engaged leads via Gmail, and logs all processed replies to Google Sheets.

Typical use cases:
- Prioritizing inbound replies from cold email campaigns
- Alerting a sales team in real time when a lead shows intent
- Sending fast acknowledgment emails to interested leads
- Keeping a structured reply log for follow-up tracking

## 1.1 Input Reception and Validation
The workflow starts with an authenticated webhook endpoint that receives Instantly reply events. It immediately returns `200 OK`, then validates that the payload contains both a sender email and a reply body.

## 1.2 Lead Data Normalization
If the payload is valid, the workflow extracts and standardizes key lead fields such as email, name, subject, campaign, timestamp, and message ID so downstream nodes can use a consistent schema.

## 1.3 AI Classification
The normalized reply is sent to OpenAI (`gpt-4o-mini`) with explicit HOT/WARM/COLD classification criteria. The model is instructed to return strict JSON containing the classification, reasoning, and key signals.

## 1.4 Routing and Actions
The workflow routes the result first through a HOT check, then a WARM check, leaving all remaining replies as COLD. Each branch sends a Telegram alert; HOT and WARM also send a Gmail auto-acknowledgment.

## 1.5 Logging
All three branches eventually converge into a Google Sheets append operation that records the lead, classification, reply snippet, reasoning, and whether an automatic acknowledgment was sent.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Validation

### Overview
This block exposes the webhook endpoint for Instantly, acknowledges the request immediately, and validates required payload fields. It ensures the workflow does not proceed unless there is enough data to classify a reply.

### Nodes Involved
- Instantly Reply Webhook
- Respond 200 OK
- Validate Payload
- Stop - Invalid Payload

### Node Details

#### Instantly Reply Webhook
- **Type and technical role:** `n8n-nodes-base.webhook` — entry point for external HTTP POST requests from Instantly.
- **Configuration choices:**
  - HTTP method: `POST`
  - Path: `instantly-reply`
  - Response mode: `responseNode`, meaning a dedicated response node sends the HTTP response
  - Authentication: `headerAuth`
- **Key expressions or variables used:** None in parameters, but incoming data is expected under `$json.body`.
- **Input and output connections:**
  - Input: none, this is the workflow trigger
  - Output: `Respond 200 OK`
- **Version-specific requirements:** Type version `2`
- **Edge cases or potential failure types:**
  - Missing or incorrect header authentication configuration
  - Instantly sending to the wrong URL
  - Payload shape differing from expected schema
  - If the webhook is in test mode instead of production mode when Instantly calls it
- **Sub-workflow reference:** None

#### Respond 200 OK
- **Type and technical role:** `n8n-nodes-base.respondToWebhook` — returns an HTTP response to the webhook caller.
- **Configuration choices:**
  - Responds with text
  - Response body: `OK`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input: `Instantly Reply Webhook`
  - Output: `Validate Payload`
- **Version-specific requirements:** Type version `1.1`
- **Edge cases or potential failure types:**
  - If placed incorrectly or not connected after the webhook, the caller may timeout
  - If webhook response mode is changed away from `responseNode`, this node becomes unnecessary or invalid
- **Sub-workflow reference:** None

#### Validate Payload
- **Type and technical role:** `n8n-nodes-base.if` — validates that minimum required data exists.
- **Configuration choices:**
  - Uses two conditions combined with `and`
  - Checks that sender email is not empty using:
    - `{{$json.body.from_address_email || $json.body.from_email || $json.body.email}}`
  - Checks that reply content is not empty using:
    - `{{$json.body.reply_body || $json.body.text_body || $json.body.body}}`
- **Key expressions or variables used:**
  - Fallback field mapping for sender email
  - Fallback field mapping for reply body
- **Input and output connections:**
  - Input: `Respond 200 OK`
  - True output: `Extract Lead Fields`
  - False output: `Stop - Invalid Payload`
- **Version-specific requirements:** Type version `2.2`
- **Edge cases or potential failure types:**
  - Payload fields may exist but contain whitespace-only values; `isNotEmpty` does not trim automatically
  - Unexpected payload nesting could make all expressions resolve empty
  - Strict type validation may behave differently if incoming values are non-strings
- **Sub-workflow reference:** None

#### Stop - Invalid Payload
- **Type and technical role:** `n8n-nodes-base.stopAndError` — intentionally halts execution for invalid inputs.
- **Configuration choices:** Default stop/error behavior; no custom message configured.
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input: false branch of `Validate Payload`
  - Output: none
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:**
  - Execution will appear failed for invalid payloads, which may be acceptable for monitoring but can increase noise in execution logs
- **Sub-workflow reference:** None

---

## 2.2 Lead Data Normalization

### Overview
This block converts the incoming Instantly reply payload into a predictable internal schema. It simplifies downstream prompts, routing, notifications, and logging.

### Nodes Involved
- Extract Lead Fields

### Node Details

#### Extract Lead Fields
- **Type and technical role:** `n8n-nodes-base.set` — creates a normalized data object with selected fields.
- **Configuration choices:**
  - Sets:
    - `lead_email`
    - `lead_name`
    - `reply_body`
    - `subject`
    - `campaign_name`
    - `received_at`
    - `message_id`
  - Uses field fallbacks to support variations in Instantly payload naming
  - Uses `$now.toISO()` for processing timestamp
- **Key expressions or variables used:**
  - `{{$json.body.from_address_email || $json.body.from_email || $json.body.email}}`
  - `{{$json.body.from_address_name || $json.body.from_name || $json.body.first_name || 'Unknown'}}`
  - `{{$json.body.reply_body || $json.body.text_body || $json.body.body || ''}}`
  - `{{$json.body.subject || 'No Subject'}}`
  - `{{$json.body.campaign_name || $json.body.campaign || 'Unknown Campaign'}}`
  - `{{$now.toISO()}}`
  - `{{$json.body.message_id || $json.body.id || ''}}`
- **Input and output connections:**
  - Input: true branch of `Validate Payload`
  - Output: `Classify Reply - OpenAI`
- **Version-specific requirements:** Type version `3.4`
- **Edge cases or potential failure types:**
  - If fields exist but are malformed objects instead of strings, later nodes may fail
  - `received_at` reflects workflow processing time, not necessarily original reply receipt time from Instantly
  - If `reply_body` is extremely long, it may increase token usage in the AI node
- **Sub-workflow reference:** None

---

## 2.3 AI Classification

### Overview
This block submits the normalized lead reply to OpenAI and expects a machine-readable JSON classification. It is the decision engine for the entire workflow.

### Nodes Involved
- Classify Reply - OpenAI
- Parse Classification

### Node Details

#### Classify Reply - OpenAI
- **Type and technical role:** `@n8n/n8n-nodes-langchain.openAi` — sends a chat-style prompt to OpenAI and expects JSON output.
- **Configuration choices:**
  - Model: `gpt-4o-mini`
  - JSON output enabled
  - `onError: continueRegularOutput`, so downstream execution continues even if the node has an issue
  - System prompt defines HOT/WARM/COLD categories with explicit examples
  - User message includes lead metadata and reply content
- **Key expressions or variables used:**
  - Prompt variables:
    - `{{$json.lead_name}}`
    - `{{$json.lead_email}}`
    - `{{$json.subject}}`
    - `{{$json.campaign_name}}`
    - `{{$json.reply_body}}`
  - Required output shape:
    - `{"classification":"HOT|WARM|COLD","reasoning":"...","key_signals":"..."}`
- **Input and output connections:**
  - Input: `Extract Lead Fields`
  - Output: `Parse Classification`
- **Version-specific requirements:** Type version `1.8`
- **Edge cases or potential failure types:**
  - Missing or invalid OpenAI credentials
  - Model unavailability or API quota/rate limit issues
  - Non-JSON or malformed JSON output despite instruction
  - Long reply bodies causing token overages or truncation
  - Because `continueRegularOutput` is enabled, downstream nodes may receive incomplete or error-shaped data
- **Sub-workflow reference:** None

#### Parse Classification
- **Type and technical role:** `n8n-nodes-base.set` — extracts AI output fields and reattaches normalized lead data.
- **Configuration choices:**
  - Reads:
    - `classification` from `$json.message.content.classification`
    - `reasoning` from `$json.message.content.reasoning`
    - `key_signals` from `$json.message.content.key_signals`
  - Applies defaults:
    - `classification`: `WARM`
    - `reasoning`: `Classification unavailable`
    - `key_signals`: empty string
  - Pulls all lead fields from `Extract Lead Fields` by node reference
- **Key expressions or variables used:**
  - `{{$json.message.content.classification || 'WARM'}}`
  - `{{$json.message.content.reasoning || 'Classification unavailable'}}`
  - `{{$json.message.content.key_signals || ''}}`
  - `{{ $('Extract Lead Fields').item.json.<field> }}`
- **Input and output connections:**
  - Input: `Classify Reply - OpenAI`
  - Output: `Is HOT?`
- **Version-specific requirements:** Type version `3.4`
- **Edge cases or potential failure types:**
  - If the OpenAI node output shape changes, `message.content` may not exist
  - Defaulting to `WARM` means AI failures may generate false positives for moderate-interest leads
  - Cross-node reference assumes one-to-one item alignment with `Extract Lead Fields`
- **Sub-workflow reference:** None

---

## 2.4 Routing and Notifications

### Overview
This block routes replies into HOT, WARM, or COLD branches. It then sends Telegram alerts for every classification and Gmail auto-acks for HOT and WARM only.

### Nodes Involved
- Is HOT?
- Is WARM?
- Telegram - HOT Lead
- Telegram - WARM Lead
- Telegram - COLD Lead
- Auto-Ack HOT Gmail
- Auto-Ack WARM Gmail

### Node Details

#### Is HOT?
- **Type and technical role:** `n8n-nodes-base.if` — checks whether the classification is `HOT`.
- **Configuration choices:**
  - Condition: `{{$json.classification}} === 'HOT'`
- **Key expressions or variables used:**
  - `{{$json.classification}}`
- **Input and output connections:**
  - Input: `Parse Classification`
  - True output: `Telegram - HOT Lead`
  - False output: `Is WARM?`
- **Version-specific requirements:** Type version `2.2`
- **Edge cases or potential failure types:**
  - Case-sensitive comparison means `hot` or `Hot` would not match
  - Unexpected labels from the AI result will fall through to later branches
- **Sub-workflow reference:** None

#### Is WARM?
- **Type and technical role:** `n8n-nodes-base.if` — checks whether the classification is `WARM`.
- **Configuration choices:**
  - Condition: `{{$json.classification}} === 'WARM'`
- **Key expressions or variables used:**
  - `{{$json.classification}}`
- **Input and output connections:**
  - Input: false branch of `Is HOT?`
  - True output: `Telegram - WARM Lead`
  - False output: `Telegram - COLD Lead`
- **Version-specific requirements:** Type version `2.2`
- **Edge cases or potential failure types:**
  - Any classification other than exact `WARM` becomes COLD by default
  - This includes malformed outputs such as `UNKNOWN`, empty string, or lowercase values
- **Sub-workflow reference:** None

#### Telegram - HOT Lead
- **Type and technical role:** `n8n-nodes-base.telegram` — sends a Telegram message for high-priority leads.
- **Configuration choices:**
  - Chat ID is a placeholder: `YOUR_TELEGRAM_CHAT_ID`
  - Parse mode: `Markdown`
- **Key expressions or variables used:** No message text is explicitly configured in the provided JSON.
- **Input and output connections:**
  - Input: true branch of `Is HOT?`
  - Output: `Auto-Ack HOT Gmail`
- **Version-specific requirements:** Type version `1.2`
- **Edge cases or potential failure types:**
  - This node appears incomplete because a Telegram message/body field is not configured
  - Invalid bot credentials or unauthorized bot/chat access
  - Markdown parse errors if a message is later added with unescaped characters
  - Wrong chat ID will prevent delivery
- **Sub-workflow reference:** None

#### Telegram - WARM Lead
- **Type and technical role:** `n8n-nodes-base.telegram` — sends a Telegram message for medium-priority leads.
- **Configuration choices:**
  - Chat ID placeholder: `YOUR_TELEGRAM_CHAT_ID`
  - Parse mode: `Markdown`
- **Key expressions or variables used:** No message text is explicitly configured in the provided JSON.
- **Input and output connections:**
  - Input: true branch of `Is WARM?`
  - Output: `Auto-Ack WARM Gmail`
- **Version-specific requirements:** Type version `1.2`
- **Edge cases or potential failure types:**
  - Same as HOT Telegram node: likely missing message content
  - Credential, permission, or chat ID issues
- **Sub-workflow reference:** None

#### Telegram - COLD Lead
- **Type and technical role:** `n8n-nodes-base.telegram` — sends a Telegram message for low-priority or negative replies.
- **Configuration choices:**
  - Chat ID placeholder: `YOUR_TELEGRAM_CHAT_ID`
  - Parse mode: `Markdown`
- **Key expressions or variables used:** No message text is explicitly configured in the provided JSON.
- **Input and output connections:**
  - Input: false branch of `Is WARM?`
  - Output: `Log Reply to Sheet`
- **Version-specific requirements:** Type version `1.2`
- **Edge cases or potential failure types:**
  - Same likely incompleteness as the other Telegram nodes
  - Delivery failures due to credentials or incorrect chat ID
- **Sub-workflow reference:** None

#### Auto-Ack HOT Gmail
- **Type and technical role:** `n8n-nodes-base.gmail` — sends an automatic acknowledgment email to HOT leads.
- **Configuration choices:**
  - `onError: continueRegularOutput`
  - Recipient: `{{$json.lead_email}}`
  - Subject: `Re: {{$json.subject}}`
  - Plain text email
  - Message:
    - `Hi {{ $json.lead_name }},`
    - `Thanks for getting back to me! I'll have a detailed response for you shortly.`
- **Key expressions or variables used:**
  - `{{$json.lead_email}}`
  - `{{$json.subject}}`
  - `{{$json.lead_name}}`
- **Input and output connections:**
  - Input: `Telegram - HOT Lead`
  - Output: `Log Reply to Sheet`
- **Version-specific requirements:** Type version `2.1`
- **Edge cases or potential failure types:**
  - Missing Gmail OAuth credentials
  - Sending limits or restricted mailbox permissions
  - Risk of threading issues if `Re:` subject does not correspond to original Gmail thread
  - Auto-responding to automated or undesirable messages if classification is wrong
  - Because `continueRegularOutput` is enabled, logging continues even if send fails
- **Sub-workflow reference:** None

#### Auto-Ack WARM Gmail
- **Type and technical role:** `n8n-nodes-base.gmail` — sends an automatic acknowledgment email to WARM leads.
- **Configuration choices:**
  - `onError: continueRegularOutput`
  - Recipient: `{{$json.lead_email}}`
  - Subject: `Re: {{$json.subject}}`
  - Plain text email
  - Message:
    - `Hi {{ $json.lead_name }},`
    - `Appreciate the reply! Let me get back to you with more details.`
- **Key expressions or variables used:**
  - `{{$json.lead_email}}`
  - `{{$json.subject}}`
  - `{{$json.lead_name}}`
- **Input and output connections:**
  - Input: `Telegram - WARM Lead`
  - Output: `Log Reply to Sheet`
- **Version-specific requirements:** Type version `2.1`
- **Edge cases or potential failure types:**
  - Same credential and deliverability risks as HOT Gmail
  - False WARM default from the parse node could trigger unintended acknowledgments
- **Sub-workflow reference:** None

---

## 2.5 Logging

### Overview
This block appends a record of each processed reply into Google Sheets. It centralizes classification outcomes and operational tracking fields for later human follow-up.

### Nodes Involved
- Log Reply to Sheet

### Node Details

#### Log Reply to Sheet
- **Type and technical role:** `n8n-nodes-base.googleSheets` — appends a row to a spreadsheet.
- **Configuration choices:**
  - Operation: `append`
  - Document ID placeholder: `YOUR_GOOGLE_SHEET_ID`
  - Sheet name: `Sheet1`
  - Explicit column mapping
  - Tracks:
    - subject
    - campaign
    - lead_name
    - reasoning
    - timestamp
    - lead_email
    - auto_ack_sent
    - reply_snippet
    - classification
    - manual_reply_at
    - manual_reply_sent
- **Key expressions or variables used:**
  - Values are mostly pulled from `Parse Classification`
  - `auto_ack_sent`:
    - `{{ ['HOT', 'WARM'].includes($('Parse Classification').item.json.classification) ? 'Yes' : 'No' }}`
  - `reply_snippet`:
    - `{{ $('Parse Classification').item.json.reply_body.substring(0, 200) }}`
- **Input and output connections:**
  - Inputs:
    - `Auto-Ack HOT Gmail`
    - `Auto-Ack WARM Gmail`
    - `Telegram - COLD Lead`
  - Output: none
- **Version-specific requirements:** Type version `4.5`
- **Edge cases or potential failure types:**
  - Missing Google Sheets credentials or access to the document
  - Sheet/tab name mismatch
  - Column names in the spreadsheet not matching expectations
  - If `reply_body` is null instead of string, `substring` could fail; currently normalization makes it an empty string fallback
  - Converging branches may make debugging harder if one branch passes a slightly different item shape, though this design relies on `Parse Classification` references rather than direct branch payload consistency
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Instantly Reply Webhook | Webhook | Receives POST reply events from Instantly with header-based authentication |  | Respond 200 OK | ### Receive & validate<br>Instantly sends a webhook on reply. Respond 200 immediately, validate the payload has an email and reply body, then normalize lead fields. Invalid payloads stop here.<br>**Dont Forget To** Replace YOUR_TELEGRAM_CHAT_ID in all three Telegram nodes and YOUR_GOOGLE_SHEET_ID in the Log Reply node. Configure the Instantly webhook to point to this workflow's URL. |
| Respond 200 OK | Respond to Webhook | Immediately acknowledges the webhook call with HTTP 200 text response | Instantly Reply Webhook | Validate Payload | ### Receive & validate<br>Instantly sends a webhook on reply. Respond 200 immediately, validate the payload has an email and reply body, then normalize lead fields. Invalid payloads stop here. |
| Validate Payload | If | Verifies presence of sender email and reply body | Respond 200 OK | Extract Lead Fields; Stop - Invalid Payload | ### Receive & validate<br>Instantly sends a webhook on reply. Respond 200 immediately, validate the payload has an email and reply body, then normalize lead fields. Invalid payloads stop here. |
| Extract Lead Fields | Set | Normalizes lead and reply fields into a stable schema | Validate Payload | Classify Reply - OpenAI | ### Receive & validate<br>Instantly sends a webhook on reply. Respond 200 immediately, validate the payload has an email and reply body, then normalize lead fields. Invalid payloads stop here. |
| Stop - Invalid Payload | Stop and Error | Halts execution when the incoming payload is incomplete | Validate Payload |  | ### Receive & validate<br>Instantly sends a webhook on reply. Respond 200 immediately, validate the payload has an email and reply body, then normalize lead fields. Invalid payloads stop here. |
| Classify Reply - OpenAI | OpenAI (LangChain) | Classifies the reply as HOT, WARM, or COLD with reasoning and key signals | Extract Lead Fields | Parse Classification | ### Classify with AI<br>Sends lead data to GPT-4o-mini with classification criteria. Returns HOT, WARM, or COLD with reasoning and detected key signals. |
| Parse Classification | Set | Extracts AI output and restores normalized lead fields for routing | Classify Reply - OpenAI | Is HOT? | ### Classify with AI<br>Sends lead data to GPT-4o-mini with classification criteria. Returns HOT, WARM, or COLD with reasoning and detected key signals. |
| Is HOT? | If | Routes HOT replies to the hot-lead branch | Parse Classification | Telegram - HOT Lead; Is WARM? | ### Route, notify & log<br>Routes by classification. Each tier gets a formatted Telegram alert. HOT and WARM leads receive an instant auto-ack email via Gmail. All replies are logged to Google Sheets. |
| Is WARM? | If | Routes WARM replies; non-WARM items become COLD | Is HOT? | Telegram - WARM Lead; Telegram - COLD Lead | ### Route, notify & log<br>Routes by classification. Each tier gets a formatted Telegram alert. HOT and WARM leads receive an instant auto-ack email via Gmail. All replies are logged to Google Sheets. |
| Telegram - HOT Lead | Telegram | Sends Telegram alert for HOT replies | Is HOT? | Auto-Ack HOT Gmail | ### Route, notify & log<br>Routes by classification. Each tier gets a formatted Telegram alert. HOT and WARM leads receive an instant auto-ack email via Gmail. All replies are logged to Google Sheets.<br>**Dont Forget To** Replace YOUR_TELEGRAM_CHAT_ID in all three Telegram nodes and YOUR_GOOGLE_SHEET_ID in the Log Reply node. Configure the Instantly webhook to point to this workflow's URL. |
| Telegram - WARM Lead | Telegram | Sends Telegram alert for WARM replies | Is WARM? | Auto-Ack WARM Gmail | ### Route, notify & log<br>Routes by classification. Each tier gets a formatted Telegram alert. HOT and WARM leads receive an instant auto-ack email via Gmail. All replies are logged to Google Sheets.<br>**Dont Forget To** Replace YOUR_TELEGRAM_CHAT_ID in all three Telegram nodes and YOUR_GOOGLE_SHEET_ID in the Log Reply node. Configure the Instantly webhook to point to this workflow's URL. |
| Telegram - COLD Lead | Telegram | Sends Telegram alert for COLD replies | Is WARM? | Log Reply to Sheet | ### Route, notify & log<br>Routes by classification. Each tier gets a formatted Telegram alert. HOT and WARM leads receive an instant auto-ack email via Gmail. All replies are logged to Google Sheets.<br>**Dont Forget To** Replace YOUR_TELEGRAM_CHAT_ID in all three Telegram nodes and YOUR_GOOGLE_SHEET_ID in the Log Reply node. Configure the Instantly webhook to point to this workflow's URL. |
| Auto-Ack HOT Gmail | Gmail | Sends automatic acknowledgment email to HOT leads | Telegram - HOT Lead | Log Reply to Sheet | ### Route, notify & log<br>Routes by classification. Each tier gets a formatted Telegram alert. HOT and WARM leads receive an instant auto-ack email via Gmail. All replies are logged to Google Sheets. |
| Auto-Ack WARM Gmail | Gmail | Sends automatic acknowledgment email to WARM leads | Telegram - WARM Lead | Log Reply to Sheet | ### Route, notify & log<br>Routes by classification. Each tier gets a formatted Telegram alert. HOT and WARM leads receive an instant auto-ack email via Gmail. All replies are logged to Google Sheets. |
| Log Reply to Sheet | Google Sheets | Appends reply metadata and classification to a spreadsheet log | Auto-Ack HOT Gmail; Auto-Ack WARM Gmail; Telegram - COLD Lead |  | ### Route, notify & log<br>Routes by classification. Each tier gets a formatted Telegram alert. HOT and WARM leads receive an instant auto-ack email via Gmail. All replies are logged to Google Sheets.<br>**Dont Forget To** Replace YOUR_TELEGRAM_CHAT_ID in all three Telegram nodes and YOUR_GOOGLE_SHEET_ID in the Log Reply node. Configure the Instantly webhook to point to this workflow's URL. |
| Section - Intake | Sticky Note | Visual documentation for intake/validation section |  |  |  |
| Section - Classify | Sticky Note | Visual documentation for AI classification section |  |  |  |
| Section - Route | Sticky Note | Visual documentation for routing/notification/logging section |  |  |  |
| Warning | Sticky Note | Visual warning for placeholder replacement and webhook setup |  |  |  |
| Main Description | Sticky Note | Global workflow description, setup notes, and customization ideas |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: `Classify Cold Email Replies and Notify via Telegram with OpenAI and Instantly`.

2. **Add the webhook trigger**
   - Node type: **Webhook**
   - Name: `Instantly Reply Webhook`
   - Method: `POST`
   - Path: `instantly-reply`
   - Response mode: `Using Respond to Webhook node`
   - Authentication: `Header Auth`
   - Create and attach a Header Auth credential suitable for requests coming from Instantly.
   - Save the workflow so the production webhook URL is generated.

3. **Add the response node**
   - Node type: **Respond to Webhook**
   - Name: `Respond 200 OK`
   - Response type: `Text`
   - Response body: `OK`
   - Connect:
     - `Instantly Reply Webhook` → `Respond 200 OK`

4. **Add payload validation**
   - Node type: **If**
   - Name: `Validate Payload`
   - Create two conditions with `AND` logic:
     1. Left value:
        `{{ $json.body.from_address_email || $json.body.from_email || $json.body.email }}`
        Operator: `is not empty`
     2. Left value:
        `{{ $json.body.reply_body || $json.body.text_body || $json.body.body }}`
        Operator: `is not empty`
   - Connect:
     - `Respond 200 OK` → `Validate Payload`

5. **Add the invalid-payload stop node**
   - Node type: **Stop and Error**
   - Name: `Stop - Invalid Payload`
   - Connect the **false** output of `Validate Payload` to this node.

6. **Add the lead normalization node**
   - Node type: **Set**
   - Name: `Extract Lead Fields`
   - Add these string fields:
     - `lead_email` = `{{ $json.body.from_address_email || $json.body.from_email || $json.body.email }}`
     - `lead_name` = `{{ $json.body.from_address_name || $json.body.from_name || $json.body.first_name || 'Unknown' }}`
     - `reply_body` = `{{ $json.body.reply_body || $json.body.text_body || $json.body.body || '' }}`
     - `subject` = `{{ $json.body.subject || 'No Subject' }}`
     - `campaign_name` = `{{ $json.body.campaign_name || $json.body.campaign || 'Unknown Campaign' }}`
     - `received_at` = `{{ $now.toISO() }}`
     - `message_id` = `{{ $json.body.message_id || $json.body.id || '' }}`
   - Connect the **true** output of `Validate Payload` to this node.

7. **Add the OpenAI classification node**
   - Node type: **OpenAI** from the LangChain/OpenAI integration
   - Name: `Classify Reply - OpenAI`
   - Model: `gpt-4o-mini`
   - Enable **JSON output**
   - Set node error handling to **Continue on Fail / continue regular output**
   - Add an OpenAI credential
   - Configure messages:
     - **System message**:
       ```
       You are a cold email reply classifier for a B2B outbound sales team. Analyze the email reply and classify the lead buying temperature.

       Classification criteria:

       HOT - Strong buying signals:
       - Asking about pricing, packages, or next steps
       - Requesting a call, demo, or meeting
       - Expressing clear interest in the service
       - Asking detailed questions about capabilities
       - Yes let us talk or similar affirmative responses

       WARM - Moderate interest:
       - Asking general questions without commitment
       - Requesting more information
       - Polite interest but no urgency
       - Tell me more or Send me details type responses
       - Forwarding to a colleague or decision maker

       COLD - No buying intent:
       - Unsubscribe requests
       - Not interested responses
       - Out of office replies
       - Angry or negative responses
       - Already have a solution or no need
       - Asking to be removed from list

       Respond ONLY with valid JSON: {"classification": "HOT", "reasoning": "Brief explanation", "key_signals": "The specific phrases or signals detected"}
       ```
     - **User message**:
       ```
       From: {{ $json.lead_name }} <{{ $json.lead_email }}>
       Subject: {{ $json.subject }}
       Campaign: {{ $json.campaign_name }}

       Reply:
       {{ $json.reply_body }}
       ```
   - Connect:
     - `Extract Lead Fields` → `Classify Reply - OpenAI`

8. **Add the classification parsing node**
   - Node type: **Set**
   - Name: `Parse Classification`
   - Add these fields:
     - `classification` = `{{ $json.message.content.classification || 'WARM' }}`
     - `reasoning` = `{{ $json.message.content.reasoning || 'Classification unavailable' }}`
     - `key_signals` = `{{ $json.message.content.key_signals || '' }}`
     - `lead_email` = `{{ $('Extract Lead Fields').item.json.lead_email }}`
     - `lead_name` = `{{ $('Extract Lead Fields').item.json.lead_name }}`
     - `reply_body` = `{{ $('Extract Lead Fields').item.json.reply_body }}`
     - `subject` = `{{ $('Extract Lead Fields').item.json.subject }}`
     - `campaign_name` = `{{ $('Extract Lead Fields').item.json.campaign_name }}`
     - `received_at` = `{{ $('Extract Lead Fields').item.json.received_at }}`
     - `message_id` = `{{ $('Extract Lead Fields').item.json.message_id }}`
   - Connect:
     - `Classify Reply - OpenAI` → `Parse Classification`

9. **Add the HOT routing node**
   - Node type: **If**
   - Name: `Is HOT?`
   - Condition:
     - Left value: `{{ $json.classification }}`
     - Operator: `equals`
     - Right value: `HOT`
   - Connect:
     - `Parse Classification` → `Is HOT?`

10. **Add the WARM routing node**
    - Node type: **If**
    - Name: `Is WARM?`
    - Condition:
      - Left value: `{{ $json.classification }}`
      - Operator: `equals`
      - Right value: `WARM`
    - Connect:
      - False output of `Is HOT?` → `Is WARM?`

11. **Add the HOT Telegram notification**
    - Node type: **Telegram**
    - Name: `Telegram - HOT Lead`
    - Add Telegram Bot credentials
    - Operation: configure it to send a message to a chat
    - Chat ID: replace with your real chat ID instead of `YOUR_TELEGRAM_CHAT_ID`
    - Parse mode: `Markdown`
    - Important: the provided workflow JSON does **not** include a message body, so you must add one manually.
    - Suggested message:
      ```markdown
      *HOT Lead Reply*
      *Lead:* {{ $json.lead_name }} <{{ $json.lead_email }}>
      *Campaign:* {{ $json.campaign_name }}
      *Subject:* {{ $json.subject }}
      *Reasoning:* {{ $json.reasoning }}
      *Signals:* {{ $json.key_signals }}

      *Reply:*
      {{ $json.reply_body }}
      ```
    - Connect:
      - True output of `Is HOT?` → `Telegram - HOT Lead`

12. **Add the WARM Telegram notification**
    - Node type: **Telegram**
    - Name: `Telegram - WARM Lead`
    - Use the same Telegram credential
    - Chat ID: your real Telegram chat ID
    - Parse mode: `Markdown`
    - Add a message body, for example:
      ```markdown
      *WARM Lead Reply*
      *Lead:* {{ $json.lead_name }} <{{ $json.lead_email }}>
      *Campaign:* {{ $json.campaign_name }}
      *Subject:* {{ $json.subject }}
      *Reasoning:* {{ $json.reasoning }}
      *Signals:* {{ $json.key_signals }}

      *Reply:*
      {{ $json.reply_body }}
      ```
    - Connect:
      - True output of `Is WARM?` → `Telegram - WARM Lead`

13. **Add the COLD Telegram notification**
    - Node type: **Telegram**
    - Name: `Telegram - COLD Lead`
    - Same Telegram credential
    - Chat ID: your real Telegram chat ID
    - Parse mode: `Markdown`
    - Add a message body, for example:
      ```markdown
      *COLD Lead Reply*
      *Lead:* {{ $json.lead_name }} <{{ $json.lead_email }}>
      *Campaign:* {{ $json.campaign_name }}
      *Subject:* {{ $json.subject }}
      *Reasoning:* {{ $json.reasoning }}
      *Signals:* {{ $json.key_signals }}

      *Reply:*
      {{ $json.reply_body }}
      ```
    - Connect:
      - False output of `Is WARM?` → `Telegram - COLD Lead`

14. **Add the HOT Gmail auto-acknowledgment**
    - Node type: **Gmail**
    - Name: `Auto-Ack HOT Gmail`
    - Add Gmail OAuth2 credentials
    - Operation: send email
    - To: `{{ $json.lead_email }}`
    - Subject: `Re: {{ $json.subject }}`
    - Email type: plain text
    - Body:
      ```
      Hi {{ $json.lead_name }},

      Thanks for getting back to me! I'll have a detailed response for you shortly.

      Best regards
      ```
    - Enable continue on fail / continue regular output
    - Connect:
      - `Telegram - HOT Lead` → `Auto-Ack HOT Gmail`

15. **Add the WARM Gmail auto-acknowledgment**
    - Node type: **Gmail**
    - Name: `Auto-Ack WARM Gmail`
    - Same Gmail credential
    - To: `{{ $json.lead_email }}`
    - Subject: `Re: {{ $json.subject }}`
    - Email type: plain text
    - Body:
      ```
      Hi {{ $json.lead_name }},

      Appreciate the reply! Let me get back to you with more details.

      Best regards
      ```
    - Enable continue on fail / continue regular output
    - Connect:
      - `Telegram - WARM Lead` → `Auto-Ack WARM Gmail`

16. **Prepare the Google Sheet**
    - Create a spreadsheet and a tab named `Sheet1`
    - Add these columns in the header row:
      - `timestamp`
      - `lead_email`
      - `lead_name`
      - `classification`
      - `campaign`
      - `subject`
      - `reply_snippet`
      - `reasoning`
      - `auto_ack_sent`
      - `manual_reply_sent`
      - `manual_reply_at`

17. **Add the Google Sheets logging node**
    - Node type: **Google Sheets**
    - Name: `Log Reply to Sheet`
    - Add Google Sheets credentials
    - Operation: `Append`
    - Spreadsheet ID: replace `YOUR_GOOGLE_SHEET_ID`
    - Sheet name: `Sheet1`
    - Map columns manually:
      - `subject` = `{{ $('Parse Classification').item.json.subject }}`
      - `campaign` = `{{ $('Parse Classification').item.json.campaign_name }}`
      - `lead_name` = `{{ $('Parse Classification').item.json.lead_name }}`
      - `reasoning` = `{{ $('Parse Classification').item.json.reasoning }}`
      - `timestamp` = `{{ $('Parse Classification').item.json.received_at }}`
      - `lead_email` = `{{ $('Parse Classification').item.json.lead_email }}`
      - `auto_ack_sent` = `{{ ['HOT', 'WARM'].includes($('Parse Classification').item.json.classification) ? 'Yes' : 'No' }}`
      - `reply_snippet` = `{{ $('Parse Classification').item.json.reply_body.substring(0, 200) }}`
      - `classification` = `{{ $('Parse Classification').item.json.classification }}`
      - `manual_reply_at` = empty string
      - `manual_reply_sent` = empty string

18. **Connect all logging branches**
    - `Auto-Ack HOT Gmail` → `Log Reply to Sheet`
    - `Auto-Ack WARM Gmail` → `Log Reply to Sheet`
    - `Telegram - COLD Lead` → `Log Reply to Sheet`

19. **Add optional sticky notes for documentation**
    - Add notes describing:
      - intake and validation
      - AI classification
      - routing, notifications, and logging
      - setup reminders for Telegram chat ID, Google Sheet ID, and Instantly webhook URL

20. **Configure credentials**
    - **Header Auth** for the webhook:
      - Match the header/value that Instantly will send
    - **OpenAI**:
      - Use an API key with access to `gpt-4o-mini`
    - **Telegram Bot**:
      - Create a bot with BotFather
      - Add the bot to the target chat/channel if needed
      - Retrieve the chat ID
    - **Gmail OAuth2**:
      - Authorize the Gmail account that should send acknowledgments
    - **Google Sheets OAuth2 / service auth**:
      - Ensure the target spreadsheet is accessible to the credential identity

21. **Configure Instantly**
    - In Instantly, open webhook/integration settings
    - Configure the reply event endpoint to point to the production URL of:
      - `POST /webhook/instantly-reply` or the exact n8n production path shown in your instance
    - Include the correct authentication header expected by the webhook

22. **Test with sample payloads**
    - Test a valid HOT reply like “Yes, interested. Can we book a demo?”
    - Test a WARM reply like “Can you send more details?”
    - Test a COLD reply like “Please remove me from your list.”
    - Test an invalid payload missing sender email or reply body

23. **Review behavior before activation**
    - Confirm the OpenAI node returns JSON in the expected shape
    - Confirm Telegram nodes actually include message text
    - Confirm Gmail sends only for HOT and WARM
    - Confirm Google Sheets receives one row per processed reply

24. **Activate the workflow**
    - Turn the workflow on only after production credentials and IDs are in place.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Classifies incoming cold email replies from Instantly as HOT, WARM, or COLD using OpenAI, sends Telegram notifications by priority, and auto-acknowledges engaged leads via Gmail. | Workflow purpose |
| Setup reminder: connect OpenAI, Gmail, Telegram Bot, Google Sheets, and Header Auth credentials. | Operational setup |
| Replace `YOUR_TELEGRAM_CHAT_ID` in all three Telegram nodes. | Required manual configuration |
| Create a Google Sheet with columns: `timestamp, lead_email, lead_name, classification, campaign, subject, reply_snippet, reasoning, auto_ack_sent, manual_reply_sent, manual_reply_at`. | Spreadsheet schema |
| Replace `YOUR_GOOGLE_SHEET_ID` in the Google Sheets node. | Required manual configuration |
| Configure Instantly webhook: `Settings > Integrations > Webhooks > reply_received > paste n8n webhook URL`. | Instantly integration |
| Customization idea: edit the OpenAI system prompt to adjust classification criteria. | Prompt tuning |
| Customization idea: modify the Gmail auto-ack message copy. | Response personalization |
| Customization idea: swap Telegram notifications for Slack if preferred. | Alternative notification channel |

## Additional implementation notes
- There are **no sub-workflows** in this workflow.
- There is **one entry point**: `Instantly Reply Webhook`.
- The Telegram nodes appear structurally incomplete in the provided JSON because no message body is configured. To make the workflow functional, add explicit Telegram message text.
- Because the OpenAI node continues on error and the parse node defaults classification to `WARM`, AI failures may cause unintended WARM handling and auto-ack emails. A safer variant would default to `COLD` or route failures into a dedicated review path.