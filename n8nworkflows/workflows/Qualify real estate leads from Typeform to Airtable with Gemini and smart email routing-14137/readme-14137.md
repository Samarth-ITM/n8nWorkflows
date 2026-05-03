Qualify real estate leads from Typeform to Airtable with Gemini and smart email routing

https://n8nworkflows.xyz/workflows/qualify-real-estate-leads-from-typeform-to-airtable-with-gemini-and-smart-email-routing-14137


# Qualify real estate leads from Typeform to Airtable with Gemini and smart email routing

# 1. Workflow Overview

This workflow receives real estate inquiries submitted through Typeform, extracts the submitted answers, uses Google Gemini to qualify the lead, stores both the original inquiry and the AI assessment in Airtable, and then sends an email tailored to the lead priority.

Its main use case is automated inbound lead triage for real estate or leasing teams that want to avoid manual review of every form submission. The workflow classifies each lead into `High`, `Medium`, or `Low`, logs the result in a CRM-style Airtable base, and routes the contact into either an immediate priority response or a softer nurture response.

## 1.1 Input Reception and Form Parsing
This block receives the Typeform webhook payload and converts Typeform’s field-reference-based answer structure into normalized business fields such as name, email, property type, budget, and requirements.

## 1.2 AI Lead Qualification
This block sends the normalized inquiry to an AI agent powered by Google Gemini. The agent is prompted to return a strict JSON object containing lead score, intent, timeline, and notes.

## 1.3 AI Output Normalization
This block parses the AI response defensively, handling malformed JSON, markdown fences, or extra text, and merges the AI output with the original form fields.

## 1.4 CRM Persistence
This block creates a new Airtable record containing both the submitted inquiry and the AI-generated qualification fields.

## 1.5 Lead Routing and Email Response
This block checks whether the lead score is `High`. High-priority leads receive an urgent response email; all others receive a nurture-style response.

## 1.6 Embedded Documentation
The workflow includes multiple sticky notes that document setup, assumptions, and maintenance tasks such as webhook setup, Typeform field reference updates, and email/Airtable credential preparation.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Form Parsing

### Overview
This block accepts a Typeform webhook POST request and extracts the relevant inquiry values from Typeform’s nested `answers` array. Because Typeform identifies fields by reference IDs rather than friendly names, the extraction logic explicitly maps known refs to business fields.

### Nodes Involved
- `Webhook1`
- `Extract Typeform Fields1`

### Node Details

#### Webhook1
- **Type and technical role:** `n8n-nodes-base.webhook`; entry-point trigger node that receives HTTP POST requests.
- **Configuration choices:**
  - HTTP method: `POST`
  - Path: a generated webhook path matching the node webhook ID
  - No special webhook options are enabled
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input: none; this is a trigger
  - Output: `Extract Typeform Fields1`
- **Version-specific requirements:** `typeVersion 2.1`
- **Edge cases or potential failure types:**
  - Typeform not configured with the production URL
  - Typeform sending payload in an unexpected shape
  - Testing with sample payloads that differ from production payload structure
  - Webhook path changes if node is recreated manually
- **Sub-workflow reference:** None

#### Extract Typeform Fields1
- **Type and technical role:** `n8n-nodes-base.code`; custom JavaScript normalization node.
- **Configuration choices:**
  - Reads the webhook payload from `$input.first().json`
  - Detects Typeform data from several possible locations:
    - `input.body.form_response`
    - `input.form_response`
    - `input.body`
    - `input`
  - Builds a `refMap` keyed by Typeform `field.ref`
  - Uses hardcoded `REF_...` constants to map field IDs to business fields
  - Extracts answer values depending on answer type:
    - `text`
    - `email`
    - `phone_number`
    - `choice`
    - `choices`
    - `boolean`
    - `number`
  - Outputs a normalized JSON object with:
    - `name`
    - `email`
    - `phone`
    - `property_type`
    - `purpose`
    - `location`
    - `budget`
    - `requirements`
    - `consent`
    - `submit_date`
- **Key expressions or variables used:**
  - `const fr = input.body?.form_response || input.form_response || input.body || input;`
  - Typeform field refs such as:
    - `REF_NAME`
    - `REF_PHONE`
    - `REF_EMAIL`
    - `REF_PROP_TYPE`
    - `REF_PURPOSE`
    - `REF_LOCATION`
    - `REF_BUDGET`
    - `REF_REQUIREMENTS`
    - `REF_CONSENT`
- **Input and output connections:**
  - Input: `Webhook1`
  - Output: `AI Lead Qualifier2`
- **Version-specific requirements:** `typeVersion 2`
- **Edge cases or potential failure types:**
  - Typeform form rebuilt or duplicated, causing field refs to change
  - Missing answers for optional fields
  - Unsupported Typeform answer types returning empty strings
  - Unexpected payload nesting if Typeform changes schema
  - Logging may reveal data in execution logs, so access control matters
- **Sub-workflow reference:** None

---

## 2.2 AI Lead Qualification

### Overview
This block sends the normalized real estate inquiry to an AI agent that evaluates lead quality and returns a structured JSON assessment. The agent relies on a connected Google Gemini chat model.

### Nodes Involved
- `AI Lead Qualifier2`
- `Google Gemini Chat Model2`

### Node Details

#### AI Lead Qualifier2
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; LangChain-based AI agent node that constructs a prompt and uses a connected chat model.
- **Configuration choices:**
  - Prompt type: defined directly in the node
  - Main prompt includes:
    - role: AI leasing assistant
    - instruction to return only raw JSON
    - exact JSON format
    - strict allowed values for `lead_score`
    - embedded lead data from prior node
    - explicit scoring rules
  - System message reinforces:
    - raw JSON only
    - no markdown
    - no code blocks
    - no explanation
- **Key expressions or variables used:**
  - `{{ $json.name }}`
  - `{{ $json.email }}`
  - `{{ $json.phone }}`
  - `{{ $json.property_type }}`
  - `{{ $json.purpose }}`
  - `{{ $json.location }}`
  - `{{ $json.budget }}`
  - `{{ $json.requirements }}`
- **Input and output connections:**
  - Main input: `Extract Typeform Fields1`
  - AI language model input: `Google Gemini Chat Model2`
  - Main output: `Parse AI Output2`
- **Version-specific requirements:** `typeVersion 3.1`; requires n8n installation with LangChain/AI nodes enabled.
- **Edge cases or potential failure types:**
  - Gemini credential missing or invalid
  - Model returns text that is not strict JSON despite prompt instructions
  - Rate limits or API quota exhaustion
  - Hallucinated fields or wrong enum values
  - Prompt behavior may vary across model versions
- **Sub-workflow reference:** None

#### Google Gemini Chat Model2
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`; supplies the LLM used by the AI agent.
- **Configuration choices:**
  - No custom options set in this export
  - Expected to use a configured Google Gemini credential
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Output via AI model connection: `AI Lead Qualifier2`
- **Version-specific requirements:** `typeVersion 1`
- **Edge cases or potential failure types:**
  - Missing API key/credential
  - Unsupported model or region restrictions depending on environment
  - API timeouts or temporary provider errors
- **Sub-workflow reference:** None

---

## 2.3 AI Output Normalization

### Overview
This block converts the AI agent’s response into reliable structured fields, even if the model wraps the JSON in markdown or adds extra text. It also merges the parsed AI assessment with the original Typeform data into a single clean object.

### Nodes Involved
- `Parse AI Output2`

### Node Details

#### Parse AI Output2
- **Type and technical role:** `n8n-nodes-base.code`; JavaScript cleanup and merge node.
- **Configuration choices:**
  - Reads AI raw output from `$input.first().json.output || ''`
  - Reads original form data from `$('Extract Typeform Fields1').first().json`
  - Attempts four parsing strategies:
    1. Direct `JSON.parse`
    2. Strip markdown fences and parse
    3. Extract first JSON object block with regex and parse
    4. Regex fallback for individual fields
  - Ensures `lead_score` is one of:
    - `High`
    - `Medium`
    - `Low`
    - otherwise defaults to `Medium`
  - Builds final output including original form fields and AI fields
- **Key expressions or variables used:**
  - `$input.first().json.output`
  - `$('Extract Typeform Fields1').first().json`
  - `parsed.lead_score`
  - `parsed.intent`
  - `parsed.timeline`
  - `parsed.notes`
- **Input and output connections:**
  - Input: `AI Lead Qualifier2`
  - Output: `Save to Airtable2`
- **Version-specific requirements:** `typeVersion 2`
- **Edge cases or potential failure types:**
  - AI response missing `output`
  - AI returns severely malformed text that regex fallback cannot recover
  - Cross-node reference to `Extract Typeform Fields1` can fail if node names change during rebuild
  - Defaulting to `Medium` may hide underlying AI formatting issues if not monitored
- **Sub-workflow reference:** None

---

## 2.4 CRM Persistence

### Overview
This block stores the enriched lead record in Airtable. It maps each normalized field into a target table structure suitable for CRM-style lead tracking.

### Nodes Involved
- `Save to Airtable2`

### Node Details

#### Save to Airtable2
- **Type and technical role:** `n8n-nodes-base.airtable`; creates a new Airtable record.
- **Configuration choices:**
  - Operation: `create`
  - Base: selected by ID
  - Table: selected by ID
  - Mapping mode: manually defined
  - Field mappings:
    - `Full Name` ← `{{$json.full_name}}`
    - `Email Address` ← `{{$json.email}}`
    - `Mobile Phone Number` ← `{{$json.phone}}`
    - `Property Type` ← `{{$json.property_type}}`
    - `Purpose` ← `{{$json.purpose}}`
    - `Preferred Location` ← `{{$json.location}}`
    - `Budget Range` ← `{{$json.budget}}`
    - `Requirements` ← `{{$json.requirements}}`
    - `Submit Date` ← `{{$json.submit_date}}`
    - `Lead Score` ← `{{$json.lead_score}}`
    - `Intent` ← `{{$json.intent}}`
    - `Timeline` ← `{{$json.timeline}}`
    - `Notes` ← `{{$json.notes}}`
  - `typecast: true` enabled to tolerate field type coercion
- **Key expressions or variables used:**
  - All mappings pull from current item JSON produced by `Parse AI Output2`
- **Input and output connections:**
  - Input: `Parse AI Output2`
  - Output: `High Lead?2`
- **Version-specific requirements:** `typeVersion 2.1`
- **Edge cases or potential failure types:**
  - Airtable credential missing or unauthorized
  - Base ID or table ID invalid
  - Airtable field names/schema not matching expected columns
  - Typecast may not fix all incompatible types, especially if select fields have constrained values
  - API rate limits for high submission volume
- **Sub-workflow reference:** None

---

## 2.5 Lead Routing and Email Response

### Overview
This block decides whether the lead is high priority and sends the corresponding email. High leads get a faster, more urgent promise; medium and low leads receive a general acknowledgment and softer follow-up message.

### Nodes Involved
- `High Lead?2`
- `Priority Email — High Lead2`
- `Nurture Email — Medium/Low Lead2`

### Node Details

#### High Lead?2
- **Type and technical role:** `n8n-nodes-base.if`; conditional router.
- **Configuration choices:**
  - Compares `lead_score` to the exact string `High`
  - Condition type: string equals
  - Strict validation enabled
- **Key expressions or variables used:**
  - `{{ $('Parse AI Output2').first().json.lead_score }}`
- **Input and output connections:**
  - Input: `Save to Airtable2`
  - True output: `Priority Email — High Lead2`
  - False output: `Nurture Email — Medium/Low Lead2`
- **Version-specific requirements:** `typeVersion 2.1`
- **Edge cases or potential failure types:**
  - If node names change, expression references must be updated
  - If `lead_score` is blank or malformed, it routes to false path
  - Depends on exact capitalization `High`
- **Sub-workflow reference:** None

#### Priority Email — High Lead2
- **Type and technical role:** `n8n-nodes-base.emailSend`; sends HTML email to high-priority leads.
- **Configuration choices:**
  - Subject personalized with lead name
  - Recipient set to parsed lead email
  - Sender hardcoded as `user@example.com` and must be replaced
  - Attribution footer disabled
  - Rich HTML body includes:
    - greeting
    - inquiry summary
    - purpose, location, budget, timeline
    - AI notes
    - callback promise within 2 hours
- **Key expressions or variables used:**
  - `{{ $('Parse AI Output2').first().json.full_name }}`
  - `{{ $('Parse AI Output2').first().json.email }}`
  - `{{ $('Parse AI Output2').first().json.property_type }}`
  - `{{ $('Parse AI Output2').first().json.purpose }}`
  - `{{ $('Parse AI Output2').first().json.location }}`
  - `{{ $('Parse AI Output2').first().json.budget }}`
  - `{{ $('Parse AI Output2').first().json.timeline }}`
  - `{{ $('Parse AI Output2').first().json.notes }}`
- **Input and output connections:**
  - Input: true branch of `High Lead?2`
  - Output: none
- **Version-specific requirements:** `typeVersion 2.1`
- **Edge cases or potential failure types:**
  - Email credential not configured
  - `fromEmail` invalid or not authorized by SMTP/provider
  - Missing recipient email causing send failure
  - HTML rendering differences across mail clients
  - Promise of response within 2 hours creates an operational dependency on staff availability
- **Sub-workflow reference:** None

#### Nurture Email — Medium/Low Lead2
- **Type and technical role:** `n8n-nodes-base.emailSend`; sends acknowledgment/nurture email to non-high leads.
- **Configuration choices:**
  - Subject personalized with lead name
  - Recipient set to parsed lead email
  - Sender hardcoded as `user@example.com` and must be replaced
  - Attribution footer disabled
  - HTML body includes:
    - thank-you message
    - inquiry summary
    - invitation to schedule a showing
    - preparation tips
    - response expectation of 1–2 business days
- **Key expressions or variables used:**
  - `{{ $('Parse AI Output2').first().json.full_name }}`
  - `{{ $('Parse AI Output2').first().json.email }}`
  - `{{ $('Parse AI Output2').first().json.property_type }}`
  - `{{ $('Parse AI Output2').first().json.location }}`
  - `{{ $('Parse AI Output2').first().json.budget }}`
- **Input and output connections:**
  - Input: false branch of `High Lead?2`
  - Output: none
- **Version-specific requirements:** `typeVersion 2.1`
- **Edge cases or potential failure types:**
  - Same SMTP/provider issues as the high-priority email node
  - Missing email address causing failure
  - Medium and low leads are combined; if different messaging is needed, a second condition is required
- **Sub-workflow reference:** None

---

## 2.6 Embedded Documentation

### Overview
These sticky notes provide operational guidance directly inside the canvas. They describe setup tasks, maintenance caveats, and behavioral expectations for each functional section.

### Nodes Involved
- `Overview`
- `Webhook note`
- `Extractor note`
- `AI note`
- `Model note`
- `Parse note`
- `Airtable note`
- `Route note`
- `High email note`
- `Nurture email note`

### Node Details

#### Overview
- **Type and technical role:** `n8n-nodes-base.stickyNote`; visual documentation.
- **Configuration choices:** Large top-level summary note with deployment checklist.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** `typeVersion 1`
- **Edge cases or potential failure types:** None; informational only.
- **Sub-workflow reference:** None

#### Webhook note
- **Type and technical role:** `n8n-nodes-base.stickyNote`; explains webhook setup.
- **Configuration choices:** Positioned beside webhook node.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** `typeVersion 1`
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None

#### Extractor note
- **Type and technical role:** `n8n-nodes-base.stickyNote`; explains Typeform ref mapping dependency.
- **Configuration choices:** Positioned beside extractor code node.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** `typeVersion 1`
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None

#### AI note
- **Type and technical role:** `n8n-nodes-base.stickyNote`; explains AI output fields and prompt maintenance.
- **Configuration choices:** Positioned beside AI agent.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** `typeVersion 1`
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None

#### Model note
- **Type and technical role:** `n8n-nodes-base.stickyNote`; indicates model credential setup and interchangeable providers.
- **Configuration choices:** Positioned below model node.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** `typeVersion 1`
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None

#### Parse note
- **Type and technical role:** `n8n-nodes-base.stickyNote`; documents defensive parsing logic.
- **Configuration choices:** Positioned beside parse code node.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** `typeVersion 1`
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None

#### Airtable note
- **Type and technical role:** `n8n-nodes-base.stickyNote`; documents Airtable schema expectations.
- **Configuration choices:** Positioned beside Airtable node.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** `typeVersion 1`
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None

#### Route note
- **Type and technical role:** `n8n-nodes-base.stickyNote`; explains branching logic.
- **Configuration choices:** Positioned beside IF node.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** `typeVersion 1`
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None

#### High email note
- **Type and technical role:** `n8n-nodes-base.stickyNote`; explains operational expectation for high-priority email.
- **Configuration choices:** Positioned above high-lead email node.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** `typeVersion 1`
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None

#### Nurture email note
- **Type and technical role:** `n8n-nodes-base.stickyNote`; explains medium/low lead email behavior.
- **Configuration choices:** Positioned beside nurture email node.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** `typeVersion 1`
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook1 | webhook | Receives Typeform submission via HTTP POST |  | Extract Typeform Fields1 | **Webhook**  Typeform POSTs here on every submission. Copy the production URL from this node and paste it into: Typeform → Connect → Webhooks |
| Extract Typeform Fields1 | code | Maps Typeform field refs to normalized lead fields | Webhook1 | AI Lead Qualifier2 | **Extract Typeform Fields**  Typeform sends field *reference IDs*, not names. This node maps refs → values.  ⚠️ If you rebuild the form, update the REF_ constants at the top of this node. Log a test webhook payload to find the new refs. |
| AI Lead Qualifier2 | @n8n/n8n-nodes-langchain.agent | Prompts AI to score the lead and return structured assessment JSON | Extract Typeform Fields1; Google Gemini Chat Model2 | Parse AI Output2 | **AI Lead Qualifier**  Gemini reads the lead and returns JSON: • `lead_score` — High / Medium / Low • `intent` — what they actually want • `timeline` — how urgent • `notes` — summary for sales  Edit scoring rules in the prompt. |
| Google Gemini Chat Model2 | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Provides Gemini chat model to the AI agent |  | AI Lead Qualifier2 | Connect your Gemini credential here. Can swap for GPT-4o or Claude |
| Parse AI Output2 | code | Parses AI output robustly and merges it with original form data | AI Lead Qualifier2 | Save to Airtable2 | **Parse AI Output**  Gemini sometimes wraps JSON in backticks or adds extra text. This node tries 4 approaches to extract clean JSON — direct parse, strip fences, regex, field-by-field fallback.  If score is wrong, check the execution log here first. |
| Save to Airtable2 | airtable | Creates a CRM record in Airtable | Parse AI Output2 | High Lead?2 | **Save to Airtable**  Creates a new CRM record with all lead data + AI assessment.  Your base needs these fields: Full Name, Email, Phone, Property Type, Purpose, Location, Budget, Requirements, Submit Date, Lead Score, Intent, Timeline, Notes.  `typecast: true` handles field type mismatches. |
| High Lead?2 | if | Routes leads based on whether the AI score is High | Save to Airtable2 | Priority Email — High Lead2; Nurture Email — Medium/Low Lead2 | **Route by score**  High → priority email Anything else → nurture email  Want a third path for Low leads? Add another IF after this one. |
| Priority Email — High Lead2 | emailSend | Sends immediate high-priority response email | High Lead?2 |  | **Priority email — High leads**  Goes out immediately. Promises callback within 2 hours — make sure your team is ready when this fires.  Update `fromEmail` before going live. |
| Nurture Email — Medium/Low Lead2 | emailSend | Sends standard acknowledgment to medium and low leads | High Lead?2 |  | **Nurture email — Medium & Low**  Warmer tone, no urgency. Promises 1–2 business days.  Same SMTP credential as the priority email above. |
| Overview | stickyNote | Canvas-level setup and deployment guidance |  |  | ## 🏠 Real Estate Lead Qualifier  Typeform submission → AI scores the lead (High / Medium / Low) → saves to Airtable CRM → sends the right email automatically. No manual triage needed.  **Setup checklist:** - Paste the Webhook production URL into Typeform → Connect → Webhooks - Add your Google Gemini API key to the Gemini Chat Model node (or swap for GPT-4o / Claude) - Connect your Airtable credential and replace the Base ID + Table ID in the Airtable node - Update `fromEmail` in both Send Email nodes before going live - If you rebuild the Typeform, update the `REF_` constants in the Extract Fields node |
| Webhook note | stickyNote | Visual note for webhook configuration |  |  | **Webhook**  Typeform POSTs here on every submission. Copy the production URL from this node and paste it into: Typeform → Connect → Webhooks |
| Extractor note | stickyNote | Visual note for Typeform ref maintenance |  |  | **Extract Typeform Fields**  Typeform sends field *reference IDs*, not names. This node maps refs → values.  ⚠️ If you rebuild the form, update the REF_ constants at the top of this node. Log a test webhook payload to find the new refs. |
| AI note | stickyNote | Visual note for AI scoring behavior |  |  | **AI Lead Qualifier**  Gemini reads the lead and returns JSON: • `lead_score` — High / Medium / Low • `intent` — what they actually want • `timeline` — how urgent • `notes` — summary for sales  Edit scoring rules in the prompt. |
| Model note | stickyNote | Visual note for Gemini credential setup |  |  | Connect your Gemini credential here. Can swap for GPT-4o or Claude |
| Parse note | stickyNote | Visual note for AI output parsing behavior |  |  | **Parse AI Output**  Gemini sometimes wraps JSON in backticks or adds extra text. This node tries 4 approaches to extract clean JSON — direct parse, strip fences, regex, field-by-field fallback.  If score is wrong, check the execution log here first. |
| Airtable note | stickyNote | Visual note for Airtable schema requirements |  |  | **Save to Airtable**  Creates a new CRM record with all lead data + AI assessment.  Your base needs these fields: Full Name, Email, Phone, Property Type, Purpose, Location, Budget, Requirements, Submit Date, Lead Score, Intent, Timeline, Notes.  `typecast: true` handles field type mismatches. |
| Route note | stickyNote | Visual note for branch behavior |  |  | **Route by score**  High → priority email Anything else → nurture email  Want a third path for Low leads? Add another IF after this one. |
| High email note | stickyNote | Visual note for high-priority email expectations |  |  | **Priority email — High leads**  Goes out immediately. Promises callback within 2 hours — make sure your team is ready when this fires.  Update `fromEmail` before going live. |
| Nurture email note | stickyNote | Visual note for nurture email behavior |  |  | **Nurture email — Medium & Low**  Warmer tone, no urgency. Promises 1–2 business days.  Same SMTP credential as the priority email above. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
   - Give it the title: **Qualify real estate leads from Typeform to Airtable with Gemini and smart email routing**.

2. **Add a Webhook node** named `Webhook1`.
   - Type: `Webhook`
   - HTTP Method: `POST`
   - Path: choose a stable custom path or let n8n generate one
   - Save the node
   - Copy the **production webhook URL** later into Typeform

3. **Configure Typeform** to call the webhook.
   - In Typeform, open your form
   - Go to **Connect → Webhooks**
   - Paste the production URL from `Webhook1`
   - Ensure Typeform sends full submission payloads

4. **Add a Code node** named `Extract Typeform Fields1`.
   - Type: `Code`
   - Connect `Webhook1` → `Extract Typeform Fields1`
   - Set it to JavaScript mode
   - Paste logic equivalent to the workflow:
     - Read the incoming payload
     - Resolve `form_response`
     - Loop through `answers`
     - Build a `refMap` keyed by `answer.field.ref`
     - Define known Typeform field refs as constants
     - Extract values according to answer type
     - Return normalized fields:
       - `name`
       - `email`
       - `phone`
       - `property_type`
       - `purpose`
       - `location`
       - `budget`
       - `requirements`
       - `consent`
       - `submit_date`
   - Important: set your actual Typeform field refs in the `REF_...` constants
   - If you rebuild the Typeform, update those constants

5. **Add a Google Gemini Chat Model node** named `Google Gemini Chat Model2`.
   - Type: `Google Gemini Chat Model`
   - Create or select your Gemini credential/API key
   - Leave model options at default unless you need a specific Gemini model
   - This node will not be in the main chain; it connects to the AI node as a model input

6. **Add an AI Agent node** named `AI Lead Qualifier2`.
   - Type: `AI Agent`
   - Connect `Extract Typeform Fields1` → `AI Lead Qualifier2`
   - Connect `Google Gemini Chat Model2` to the AI Agent using the **AI language model** connector
   - Set prompt mode to define the prompt directly
   - Use a prompt that:
     - identifies the AI as a real estate leasing assistant
     - instructs it to return only raw JSON
     - defines exact expected output:
       - `lead_score`
       - `intent`
       - `timeline`
       - `notes`
     - includes the lead fields from the incoming JSON:
       - name
       - email
       - phone
       - property type
       - purpose
       - location
       - budget
       - requirements
     - states that `lead_score` must be exactly `High`, `Medium`, or `Low`
     - provides scoring rules
   - Add a system message reinforcing:
     - raw JSON only
     - no markdown
     - no explanation
     - no code block

7. **Add a Code node** named `Parse AI Output2`.
   - Connect `AI Lead Qualifier2` → `Parse AI Output2`
   - Use JavaScript that:
     - reads the AI result from the agent output
     - reads original form data from `Extract Typeform Fields1`
     - tries multiple parsing strategies:
       1. direct JSON parse
       2. remove markdown code fences
       3. extract the first JSON object with regex
       4. regex fallback for individual fields
     - validates `lead_score`
       - if not one of `High`, `Medium`, `Low`, force `Medium`
     - returns a single merged JSON object with:
       - `full_name`
       - `email`
       - `phone`
       - `property_type`
       - `purpose`
       - `location`
       - `budget`
       - `requirements`
       - `submit_date`
       - `lead_score`
       - `intent`
       - `timeline`
       - `notes`
   - Keep the reference to `Extract Typeform Fields1` aligned with the exact node name, or adjust the expression if you rename the node

8. **Prepare Airtable** before adding the next node.
   - Create or choose a base
   - Create a table for incoming leads
   - Add these fields:
     - Full Name
     - Email Address
     - Mobile Phone Number
     - Property Type
     - Purpose
     - Preferred Location
     - Budget Range
     - Requirements
     - Submit Date
     - Lead Score
     - Intent
     - Timeline
     - Notes

9. **Add an Airtable node** named `Save to Airtable2`.
   - Connect `Parse AI Output2` → `Save to Airtable2`
   - Type: `Airtable`
   - Credential: select your Airtable credential
   - Operation: `Create`
   - Choose the target Base and Table
   - Use manual field mapping
   - Map fields as follows:
     - Full Name ← `full_name`
     - Email Address ← `email`
     - Mobile Phone Number ← `phone`
     - Property Type ← `property_type`
     - Purpose ← `purpose`
     - Preferred Location ← `location`
     - Budget Range ← `budget`
     - Requirements ← `requirements`
     - Submit Date ← `submit_date`
     - Lead Score ← `lead_score`
     - Intent ← `intent`
     - Timeline ← `timeline`
     - Notes ← `notes`
   - Enable `typecast: true`

10. **Add an IF node** named `High Lead?2`.
    - Connect `Save to Airtable2` → `High Lead?2`
    - Configure one condition:
      - left value: `lead_score`
      - operator: string equals
      - right value: `High`
    - In the original workflow, the condition references `$('Parse AI Output2').first().json.lead_score`
    - A cleaner rebuild can simply test the current item’s `lead_score` if available in the IF node input

11. **Add an Email Send node** named `Priority Email — High Lead2`.
    - Connect the **true** output of `High Lead?2` to this node
    - Configure SMTP or your email credential
    - Set:
      - `toEmail` = lead email
      - `fromEmail` = your real sender address
      - subject = personalized high-priority subject
      - HTML body = personalized template including:
        - full name
        - property type
        - purpose
        - location
        - budget
        - timeline
        - notes
    - Disable attribution if desired
    - Make sure the body promises only what your sales team can actually deliver

12. **Add another Email Send node** named `Nurture Email — Medium/Low Lead2`.
    - Connect the **false** output of `High Lead?2` to this node
    - Use the same email credential as above
    - Set:
      - `toEmail` = lead email
      - `fromEmail` = your real sender address
      - subject = thank-you / acknowledgment subject
      - HTML body = softer template including:
        - full name
        - property type
        - location
        - budget
        - reassurance of follow-up in 1–2 business days
    - Disable attribution if desired

13. **Set all email expressions carefully.**
    - If you copy the original logic, the templates reference:
      - `$('Parse AI Output2').first().json.full_name`
      - `$('Parse AI Output2').first().json.email`
      - and similar fields
    - If you rebuild with simpler current-item expressions, ensure the fields still exist on the incoming item

14. **Test the workflow with a real or mock Typeform submission.**
    - Confirm that:
      - the webhook receives the payload
      - field refs map correctly
      - Gemini returns parsable output
      - Airtable record is created
      - the right email branch is selected

15. **Inspect execution logs for parser reliability.**
    - The parse code logs raw AI output and the final parsed object
    - If lead scoring seems inconsistent, review `Parse AI Output2` first

16. **Replace all placeholder operational values before activation.**
    - `fromEmail` in both email nodes
    - Airtable base/table selection
    - Gemini credential
    - Typeform field refs in extractor code

17. **Activate the workflow** once all end-to-end tests pass.

### Credential Configuration Summary
- **Google Gemini Chat Model**
  - Requires a valid Gemini API credential in n8n
- **Airtable**
  - Requires Airtable personal access token or supported credential type
  - Must have access to the selected base/table
- **Email Send nodes**
  - Require SMTP or supported mail transport credentials
  - `fromEmail` must usually match an authorized sender/domain

### Input/Output Expectations
- **Webhook input:** Typeform submission payload including `form_response.answers`
- **Extractor output:** normalized lead JSON
- **AI Agent output:** JSON-like text with qualification fields
- **Parser output:** complete structured lead object
- **Airtable output:** Airtable record metadata plus current item context
- **Email nodes output:** delivery attempt result or provider error

### Sub-workflow Setup
- This workflow does **not** include any sub-workflow nodes.
- It has a single entry point: `Webhook1`.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Real estate lead qualifier flow: Typeform submission → AI scoring → Airtable save → email routing | Workflow purpose |
| Paste the Webhook production URL into Typeform → Connect → Webhooks | Typeform webhook setup |
| Add your Google Gemini API key to the Gemini Chat Model node, or swap to GPT-4o / Claude if preferred | Model configuration |
| Connect your Airtable credential and replace the Base ID + Table ID in the Airtable node | Airtable configuration |
| Update `fromEmail` in both Send Email nodes before going live | Email deployment requirement |
| If the Typeform is rebuilt, update the `REF_` constants in the extraction code | Typeform maintenance |
| `typecast: true` is enabled in Airtable to reduce field type mismatch issues | Airtable behavior |
| High leads receive an email promising callback within 2 hours; ensure operations can support that SLA | Business operations note |
| Medium and low leads are combined into one nurture branch; add another IF if you want a dedicated Low-lead path | Routing enhancement option |

