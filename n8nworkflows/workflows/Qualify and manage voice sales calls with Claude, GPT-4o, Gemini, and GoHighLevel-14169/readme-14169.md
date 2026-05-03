Qualify and manage voice sales calls with Claude, GPT-4o, Gemini, and GoHighLevel

https://n8nworkflows.xyz/workflows/qualify-and-manage-voice-sales-calls-with-claude--gpt-4o--gemini--and-gohighlevel-14169


# Qualify and manage voice sales calls with Claude, GPT-4o, Gemini, and GoHighLevel

# 1. Workflow Overview

This workflow automates two related sales operations around AI voice calls and GoHighLevel (GHL):

1. **Inbound call qualification**: It receives end-of-call webhooks from **Vapi** or **Retell**, normalizes the payload, analyzes the transcript with multiple LLMs, scores the lead using **BANT**, decides whether the lead is qualified, creates or updates CRM records in **GoHighLevel**, generates objection-handling guidance and CRM notes, and logs the result to **Supabase** and **Google Sheets**.
2. **Outbound call scheduling**: It runs on a weekday schedule, pulls nurturing leads from GHL, uses AI to rank them by outbound priority, initiates calls through **Vapi**, and logs those outbound attempts to Supabase and Google Sheets.

The workflow is structured into these logical blocks:

## 1.1 Configuration and setup notes
Sticky notes provide required credentials, schema, placeholder IDs, and external setup steps for Supabase, GHL, Vapi, and Google Sheets.

## 1.2 Inbound reception and payload normalization
A webhook receives call-ended payloads from Vapi or Retell, then a code node standardizes the payload into a unified internal schema.

## 1.3 Call validation and BANT qualification
Short or low-content calls are filtered out. Valid calls are analyzed by Claude Haiku using a structured BANT schema.

## 1.4 Qualified inbound lead handling
If BANT score is at least 6, the workflow detects booking intent with GPT-4o, upserts the contact in GHL, creates a hot opportunity, generates objection-handling guidance and a CRM note, updates GHL, and logs results.

## 1.5 Non-qualified inbound lead handling
If BANT score is below 6, the workflow upserts the contact in GHL, creates a nurturing opportunity, triggers a GHL nurture workflow via API, and logs the result.

## 1.6 Scheduled outbound nurture calling
Every weekday at 9:00 AM, the workflow fetches nurturing leads from GHL, ranks them with GPT-4o Mini, batches them, initiates outbound Vapi calls, and logs each outbound initiation.

---

# 2. Block-by-Block Analysis

## 2.1 Configuration and setup notes

**Overview:**  
This block contains documentation embedded directly in the canvas. These nodes do not execute business logic, but they are essential for deployment because they define credentials, table schema, sheet columns, placeholder IDs, and operating assumptions.

**Nodes Involved:**  
- Setup Guide
- Supabase Schema
- Google Sheets Columns
- HighLevel IDs
- INBOUND FLOW
- OUTBOUND FLOW1
- All underscore-prefixed sticky notes associated with operational nodes

### Node Details

#### Setup Guide
- **Type and role:** Sticky Note; deployment instructions.
- **Configuration choices:** Lists required credentials and environment variables:
  - HighLevel OAuth2
  - Anthropic
  - OpenAI
  - Google Gemini (PaLM) API
  - Supabase
  - Google Sheets OAuth2
  - `VAPI_API_KEY`
  - `GHL_API_KEY`
- **Key expressions or variables used:** Mentions placeholder values such as `YOUR_PIPELINE_ID`, `YOUR_HOT_STAGE_ID`, `YOUR_NURTURING_STAGE_ID`, `YOUR_NURTURE_WORKFLOW_ID`, `YOUR_SPREADSHEET_ID`, `YOUR_VAPI_PHONE_NUMBER_ID`, `YOUR_VAPI_ASSISTANT_ID`.
- **Input/output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or failure types:** Misconfigured credentials or missing env vars will break API nodes later.
- **Sub-workflow reference:** None.

#### Supabase Schema
- **Type and role:** Sticky Note; database schema definition.
- **Configuration choices:** Defines table `voice_call_logs` and indexes.
- **Key expressions or variables used:** None.
- **Input/output connections:** None.
- **Version-specific requirements:** Requires Supabase project permissions sufficient to run SQL and use service role key.
- **Edge cases or failure types:** If schema differs from note, insert nodes may fail on missing columns or datatype mismatch.
- **Sub-workflow reference:** None.

#### Google Sheets Columns
- **Type and role:** Sticky Note; spreadsheet schema.
- **Configuration choices:** Requires a sheet named `Voice Call Log` with exact headers.
- **Edge cases:** If headers differ and mapping is strict, appended data may not land in expected columns.

#### HighLevel IDs
- **Type and role:** Sticky Note; placeholder resolution guide.
- **Configuration choices:** Explains where to find pipeline, stage, workflow, and Vapi IDs.
- **Edge cases:** Wrong IDs cause GHL creation/update failures or outbound call failures.

#### INBOUND FLOW / OUTBOUND FLOW1 / underscore-prefixed sticky notes
- **Type and role:** Sticky Notes; visual documentation for neighboring nodes.
- **Configuration choices:** Each note labels a functional step.
- **Edge cases:** None at runtime.
- **Sub-workflow reference:** None.

---

## 2.2 Inbound reception and payload normalization

**Overview:**  
This block receives inbound call-ended webhooks from voice providers and converts different provider payload formats into a consistent internal object. That normalized object becomes the single source of truth for downstream AI analysis and CRM updates.

**Nodes Involved:**  
- Inbound Call Webhook
- Normalize Call Payload
- Valid Call?

### Node Details

#### Inbound Call Webhook
- **Type and technical role:** `n8n-nodes-base.webhook`; entry point for inbound HTTP POST requests.
- **Configuration choices:**
  - Method: `POST`
  - Path: `voice-sales-inbound`
  - No custom response handling shown
- **Key expressions or variables used:** None.
- **Input and output connections:** Starts the inbound flow; outputs to **Normalize Call Payload**.
- **Version-specific requirements:** Webhook node version 2.
- **Edge cases or potential failure types:**
  - If the workflow is inactive, production webhook will not receive events.
  - Provider may send payload in unexpected structure.
  - Depending on n8n defaults, provider may time out if execution is slow; the note implies “auto 200 response”, but actual behavior depends on webhook mode and runtime configuration.
- **Sub-workflow reference:** None.

#### Normalize Call Payload
- **Type and technical role:** `Code` node; canonical payload mapper.
- **Configuration choices:** JavaScript inspects:
  - **Vapi** payloads where `body.message.type` is `end-of-call-report` or `end-of-call`
  - **Retell** payloads where `body.event` is `call_ended` or `call_analyzed`
  - fallback generic format
- **Key expressions or variables used:**
  - Produces:
    - `call_id`
    - `phone_number`
    - `transcript`
    - `recording_url`
    - `duration_sec`
    - `provider`
    - `raw_body`
  - Phone normalization to E.164-ish format:
    - if no leading `+`, strips non-digits
    - 10 digits become `+1##########`
    - otherwise `+` plus all digits
- **Input and output connections:** Input from **Inbound Call Webhook**; output to **Valid Call?**
- **Version-specific requirements:** Code node version 2.
- **Edge cases or potential failure types:**
  - International numbers may be incorrectly normalized if local formatting is ambiguous.
  - If transcript is an unexpected object rather than string/array, downstream `.length` checks may misbehave.
  - If no payload body exists, fallback fields may become empty or default values.
- **Sub-workflow reference:** None.

#### Valid Call?
- **Type and technical role:** `IF` node; quality gate before spending AI tokens and CRM operations.
- **Configuration choices:**
  - Condition 1: `duration_sec >= 30`
  - Condition 2: `transcript.length > 50`
  - Both conditions are required.
- **Key expressions or variables used:**
  - `{{ $json.duration_sec }}`
  - `{{ $json.transcript.length }}`
- **Input and output connections:** Input from **Normalize Call Payload**; true branch goes to **BANT Qualifier**. No false branch is connected.
- **Version-specific requirements:** IF node version 2.
- **Edge cases or potential failure types:**
  - Short but valuable calls are dropped entirely.
  - Calls failing this test are not logged anywhere in this design.
  - If `transcript` is null instead of string, the expression could fail, though Normalize Call Payload tries to ensure a string.
- **Sub-workflow reference:** None.

---

## 2.3 Call validation and BANT qualification

**Overview:**  
This block uses Anthropic Claude Haiku and a structured output parser to score the call against the BANT framework. A follow-up code node makes the AI result safe and consistent for downstream branching.

**Nodes Involved:**  
- BANT Qualifier
- Claude Haiku (BANT)
- BANT Output Parser
- Extract BANT
- Qualified?

### Node Details

#### BANT Qualifier
- **Type and technical role:** LangChain Agent node; prompts the AI to analyze the transcript and produce structured BANT output.
- **Configuration choices:**
  - Prompt includes phone, duration, provider, and transcript.
  - System message defines exact scoring rubric:
    - Budget: 0–3
    - Authority: 0–3
    - Need: 0–2
    - Timeline: 0–2
    - Qualified threshold: score >= 6
  - `hasOutputParser: true`
- **Key expressions or variables used:**
  - `{{ $json.phone_number }}`
  - `{{ $json.duration_sec }}`
  - `{{ $json.provider }}`
  - `{{ $json.transcript }}`
- **Input and output connections:**
  - Main input from **Valid Call?**
  - AI language model input from **Claude Haiku (BANT)**
  - AI parser input from **BANT Output Parser**
  - Main output to **Extract BANT**
- **Version-specific requirements:** LangChain Agent version 1.8.
- **Edge cases or potential failure types:**
  - Transcript may be too noisy for consistent BANT inference.
  - AI may still output malformed JSON; parser reduces but does not eliminate risk.
  - Provider-specific transcript artifacts may affect accuracy.
- **Sub-workflow reference:** None.

#### Claude Haiku (BANT)
- **Type and technical role:** Anthropic chat model connector.
- **Configuration choices:**
  - Model: `claude-3-haiku-20240307`
  - Temperature: `0.1`
- **Key expressions or variables used:** None.
- **Input and output connections:** Supplies LLM to **BANT Qualifier**.
- **Version-specific requirements:** Anthropic chat node version 1.3; requires Anthropic credentials.
- **Edge cases or potential failure types:**
  - Authentication failure
  - Model availability or quota issues
  - Latency/timeouts
- **Sub-workflow reference:** None.

#### BANT Output Parser
- **Type and technical role:** Structured output parser for AI response validation.
- **Configuration choices:** Manual JSON schema requiring:
  - `score` number
  - `budget_confirmed` boolean
  - `authority_confirmed` boolean
  - `need_identified` boolean
  - `timeline_defined` boolean
  - `summary` string
  - `key_objections` array of strings
- **Input and output connections:** Feeds parser configuration into **BANT Qualifier**.
- **Version-specific requirements:** Output parser version 1.2.
- **Edge cases or potential failure types:** If model returns schema-incompatible output, agent may fail or produce parser errors upstream.
- **Sub-workflow reference:** None.

#### Extract BANT
- **Type and technical role:** Code node; defensive parsing and normalization of AI output.
- **Configuration choices:**
  - Reads from `item.json.output` or `item.json`
  - Strips markdown code fences if present
  - Applies fallback default object on parse failure
  - Computes:
    - `bant_score`
    - `budget_confirmed`
    - `authority_confirmed`
    - `need_identified`
    - `timeline_defined`
    - `bant_summary`
    - `key_objections`
    - `qualified = bant_score >= 6`
- **Input and output connections:** Input from **BANT Qualifier**; output to **Qualified?**
- **Version-specific requirements:** Code node version 2.
- **Edge cases or potential failure types:**
  - If agent output shape changes, fallback parse path may silently produce zero score.
  - Parse errors become low-score defaults rather than hard failures.
- **Sub-workflow reference:** None.

#### Qualified?
- **Type and technical role:** IF node; branches qualified vs non-qualified lead handling.
- **Configuration choices:**
  - Condition: `bant_score >= 6`
- **Key expressions or variables used:**
  - `{{ $json.bant_score }}`
- **Input and output connections:**
  - Input from **Extract BANT**
  - True branch to **Booking Intent Detector**
  - False branch to **Search GHL Contact NQ**
- **Version-specific requirements:** IF node version 2.
- **Edge cases or potential failure types:** A malformed BANT parse defaults to 0, which sends calls to nurturing even if transcript was actually promising.
- **Sub-workflow reference:** None.

---

## 2.4 Qualified inbound lead handling

**Overview:**  
This block handles high-intent leads. It detects booking intent, upserts the contact into GoHighLevel, creates a hot opportunity, generates objection-handling guidance and a CRM note, updates CRM metadata, and logs the qualified call externally.

**Nodes Involved:**  
- Booking Intent Detector
- GPT-4o (Booking)
- Booking Output Parser
- Extract Booking
- Search GHL Contact
- Contact Exists?
- Update Contact
- Create Contact
- Merge Contact Paths
- Create Hot Opportunity
- Objection Handler
- Claude Sonnet (Objection)
- Objection Output Parser
- Extract Objection Response
- CRM Note Writer
- Gemini Flash (CRM Note)
- CRM Note Output Parser
- Extract CRM Note
- Update Contact Notes
- Update Stage: Hot Lead
- Log to Supabase
- Log to Google Sheets

### Node Details

#### Booking Intent Detector
- **Type and technical role:** LangChain Agent; detects appointment intent and objections for qualified calls.
- **Configuration choices:**
  - Prompt includes phone, BANT score, summary, objections, transcript.
  - System message asks for:
    - `appointment_requested`
    - `preferred_time`
    - `confidence`
    - `objections`
- **Key expressions or variables used:** Uses `phone_number`, `bant_score`, `bant_summary`, `key_objections`, `transcript`.
- **Input and output connections:**
  - Main input from **Qualified?** true branch
  - LLM from **GPT-4o (Booking)**
  - Parser from **Booking Output Parser**
  - Main output to **Extract Booking**
- **Version-specific requirements:** LangChain Agent version 1.8.
- **Edge cases or potential failure types:** Implicit booking signals are subjective and may vary by transcript quality.
- **Sub-workflow reference:** None.

#### GPT-4o (Booking)
- **Type and technical role:** OpenAI chat model connector.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Max tokens: `800`
  - Temperature: `0.1`
- **Input/output connections:** Feeds **Booking Intent Detector**.
- **Version-specific requirements:** OpenAI chat node version 1.2; requires OpenAI credentials.
- **Edge cases:** API limits, auth issues, model availability.

#### Booking Output Parser
- **Type and technical role:** Structured output parser.
- **Configuration choices:** Schema requires:
  - `appointment_requested` boolean
  - `preferred_time` string
  - `confidence` number
  - `objections` array
- **Input/output connections:** Parser input for **Booking Intent Detector**.
- **Edge cases:** Parser mismatch can break or degrade extraction.

#### Extract Booking
- **Type and technical role:** Code node; defensive extraction of booking analysis.
- **Configuration choices:**
  - Parses agent output from string/object
  - Fallback object if parse fails
  - Produces:
    - `appointment_requested`
    - `preferred_time`
    - `booking_confidence`
    - `detected_objections`
- **Input/output connections:** Input from **Booking Intent Detector**; output to **Search GHL Contact**
- **Version-specific requirements:** Code node version 2.
- **Edge cases:** Parse failure keeps the lead qualified but with no appointment intent and no objections beyond fallback.

#### Search GHL Contact
- **Type and technical role:** HighLevel node; contact lookup.
- **Configuration choices:**
  - Operation: `getAll`
  - Query filter: phone number
  - Limit: `1`
- **Key expressions or variables used:** `{{ $json.phone_number }}`
- **Input/output connections:** Input from **Extract Booking**; output to **Contact Exists?**
- **Version-specific requirements:** HighLevel node version 2; requires HighLevel OAuth2 credential.
- **Edge cases or potential failure types:**
  - Phone format mismatch may prevent matching.
  - Multiple matching contacts are reduced to one result.
- **Sub-workflow reference:** None.

#### Contact Exists?
- **Type and technical role:** IF node; upsert branch selector.
- **Configuration choices:**
  - Checks if `id` is not empty
- **Input/output connections:**
  - True → **Update Contact**
  - False → **Create Contact**
- **Edge cases:** If GHL response shape changes or `id` missing, the workflow may create duplicates.

#### Update Contact
- **Type and technical role:** HighLevel node; updates an existing contact.
- **Configuration choices:**
  - Operation: `update`
  - `contactId = {{ $json.id }}`
  - Tags set to `voice-lead,qualified`
- **Input/output connections:** Output to **Merge Contact Paths**
- **Edge cases:**
  - Existing tags may be overwritten rather than merged, depending on node/API behavior.
  - No note content is actually appended here despite sticky note wording.
- **Sub-workflow reference:** None.

#### Create Contact
- **Type and technical role:** HighLevel node; creates a new contact.
- **Configuration choices:**
  - Phone from **Extract Booking**
  - Additional fields:
    - `tags: voice-lead,qualified`
    - `source: Voice AI`
    - `firstName: Voice Lead`
- **Key expressions or variables used:** `{{ $('Extract Booking').item.json.phone_number }}`
- **Input/output connections:** Output to **Merge Contact Paths**
- **Edge cases:**
  - Contacts are created with minimal profile data.
  - Duplicate creation possible if lookup misses existing records.

#### Merge Contact Paths
- **Type and technical role:** Merge node in `chooseBranch` mode; rejoins update/create paths.
- **Configuration choices:** Selects whichever branch produced a contact object.
- **Input/output connections:** Inputs from **Update Contact** and **Create Contact**; output to **Create Hot Opportunity**
- **Version-specific requirements:** Merge node version 3.
- **Edge cases:** If both branches somehow produce data unexpectedly, branch selection behavior should be tested.

#### Create Hot Opportunity
- **Type and technical role:** HighLevel node; creates an opportunity for a qualified lead.
- **Configuration choices:**
  - Resource: `opportunity`
  - Name: `Voice Lead — {phone}`
  - Contact ID from merged contact
  - Pipeline ID: `YOUR_PIPELINE_ID`
  - Stage ID: `YOUR_HOT_STAGE_ID`
  - Monetary value: `0`
- **Input/output connections:** Output to **Objection Handler**
- **Edge cases:**
  - Placeholder IDs must be replaced.
  - Creating duplicate opportunities is possible if the same call is processed multiple times.

#### Objection Handler
- **Type and technical role:** LangChain Agent; generates a follow-up objection-handling script.
- **Configuration choices:**
  - Inputs include BANT data, booking data, and extracted objections.
  - System message instructs model to choose one technique such as:
    - feel-felt-found
    - acknowledge-clarify-respond
    - reframe
    - isolate-and-solve
    - social-proof
  - Structured response includes script, technique, confidence.
- **Input/output connections:**
  - Main input from **Create Hot Opportunity**
  - LLM from **Claude Sonnet (Objection)**
  - Parser from **Objection Output Parser**
  - Output to **Extract Objection Response**
- **Version-specific requirements:** LangChain Agent version 1.8.
- **Edge cases:** If there are no objections, the model still must choose a technique and may generate generic output.

#### Claude Sonnet (Objection)
- **Type and technical role:** Anthropic chat model connector.
- **Configuration choices:**
  - Model: `claude-3-5-sonnet-20241022`
  - Temperature: `0.3`
- **Input/output connections:** Supplies LLM for **Objection Handler**
- **Version-specific requirements:** Anthropic node version 1.3.
- **Edge cases:** Model availability/auth/quota issues.

#### Objection Output Parser
- **Type and technical role:** Structured output parser.
- **Configuration choices:** Requires:
  - `response_script`
  - `technique`
  - `confidence`
- **Input/output connections:** Parser for **Objection Handler**
- **Edge cases:** Malformed AI output.

#### Extract Objection Response
- **Type and technical role:** Code node; consolidates previous context and parsed objection response into one working payload.
- **Configuration choices:**
  - Defensive parse with fallback defaults
  - Explicitly reloads upstream data from:
    - `$('Extract BANT').item.json`
    - `$('Extract Booking').item.json`
    - `$('Merge Contact Paths').item.json`
    - `$('Create Hot Opportunity').item.json`
  - Produces a unified object including GHL contact/opportunity IDs and objection-handling fields
- **Input/output connections:** Input from **Objection Handler**; output to **CRM Note Writer**
- **Edge cases:**
  - Cross-node expression dependence means renaming upstream nodes can break this node.
  - If `Create Hot Opportunity` fails or returns unexpected fields, `ghl_opp_id` may be empty.

#### CRM Note Writer
- **Type and technical role:** LangChain Agent; writes a structured CRM note.
- **Configuration choices:**
  - Prompt includes call data, BANT, appointment status, objections, and objection response.
  - System message enforces a specific note structure:
    - CALL SUMMARY
    - QUALIFICATION STATUS
    - NEXT STEPS
    - OBJECTIONS ON FILE
    - RECOMMENDED APPROACH
- **Input/output connections:**
  - Main input from **Extract Objection Response**
  - LLM from **Gemini Flash (CRM Note)**
  - Parser from **CRM Note Output Parser**
  - Output to **Extract CRM Note**
- **Version-specific requirements:** LangChain Agent version 1.8.
- **Edge cases:** If note is too long/too short, output may still parse but not meet ideal formatting intent.

#### Gemini Flash (CRM Note)
- **Type and technical role:** Google Gemini chat model connector.
- **Configuration choices:**
  - Temperature: `0.2`
  - Max output tokens: `800`
  - `modelName` is blank in the JSON and must be selected manually
- **Input/output connections:** Supplies model to **CRM Note Writer**
- **Version-specific requirements:** Gemini node version 1; requires Google Gemini API credential.
- **Edge cases:**
  - Blank model selection is a deployment risk.
  - Wrong Google credential type or deprecated model names can fail execution.

#### CRM Note Output Parser
- **Type and technical role:** Structured output parser.
- **Configuration choices:** Requires one field:
  - `crm_note`
- **Input/output connections:** Parser for **CRM Note Writer**
- **Edge cases:** None beyond malformed output.

#### Extract CRM Note
- **Type and technical role:** Code node; finalizes the CRM note string.
- **Configuration choices:**
  - Defensive parse from string/object/output object
  - Fallback note if parsing fails
  - Merges note into payload from **Extract Objection Response**
- **Input/output connections:** Output to **Update Contact Notes**
- **Edge cases:** Cross-node references may break on rename.

#### Update Contact Notes
- **Type and technical role:** HighLevel node; updates contact tags.
- **Configuration choices:**
  - Operation: `update`
  - Contact ID: `{{ $json.ghl_contact_id }}`
  - Tags: `voice-lead,qualified,hot`
- **Input/output connections:** Output to **Update Stage: Hot Lead**
- **Important implementation note:** Despite its name and sticky note, this node does **not** write `crm_note` into the contact record in the shown configuration. It only updates tags.
- **Edge cases:**
  - Possible mismatch between intended behavior and actual config.
  - Existing tags may be overwritten.
- **Sub-workflow reference:** None.

#### Update Stage: Hot Lead
- **Type and technical role:** HighLevel node; updates opportunity stage.
- **Configuration choices:**
  - Resource: `opportunity`
  - Operation: `update`
  - Opportunity ID from `ghl_opp_id`
  - Stage ID: `YOUR_HOT_STAGE_ID`
- **Input/output connections:** Fans out to:
  - **Log to Supabase**
  - **Log to Google Sheets**
- **Edge cases:**
  - If opportunity creation failed earlier, this node will fail.
  - Placeholder stage ID must be replaced.

#### Log to Supabase
- **Type and technical role:** Supabase node; persists qualified inbound call details.
- **Configuration choices:**
  - Table: `voice_call_logs`
  - Inserts extensive mapped fields including:
    - call metadata
    - BANT values
    - qualification
    - objections
    - booking info
    - GHL IDs
    - CRM note
    - objection response
- **Key expressions or variables used:** Uses many `{{ $json.* }}` expressions and `JSON.stringify($json.key_objections)`.
- **Input/output connections:** Input from **Update Stage: Hot Lead**.
- **Version-specific requirements:** Supabase node version 1.
- **Edge cases:**
  - Table schema mismatch
  - credential/permission issues
  - JSONB field insertion failures if serialization is malformed

#### Log to Google Sheets
- **Type and technical role:** Google Sheets node; appends a reporting row.
- **Configuration choices:**
  - Operation: `append`
  - Sheet: `Voice Call Log`
  - Spreadsheet ID: `YOUR_SPREADSHEET_ID`
  - Maps selected fields such as date, call ID, provider, qualified, BANT score, GHL IDs, CRM note, objection response, appointment requested
- **Input/output connections:** Input from **Update Stage: Hot Lead**.
- **Version-specific requirements:** Google Sheets node version 4.4; requires OAuth2 credential.
- **Edge cases:**
  - Spreadsheet ID placeholder must be replaced.
  - Sheet name and headers must exist exactly.
  - Append order may be affected by concurrent executions.

---

## 2.5 Non-qualified inbound lead handling

**Overview:**  
This block handles leads below the qualification threshold. It still creates or updates the contact in GHL, places the lead into a nurturing opportunity stage, triggers a nurture workflow through the LeadConnector API, and records the event externally.

**Nodes Involved:**  
- Search GHL Contact NQ
- Contact Exists NQ?
- Update Contact NQ
- Create Contact NQ
- Merge Contact NQ
- Create Nurturing Opportunity
- Trigger Nurture Workflow
- Log to Supabase NQ
- Log to Google Sheets NQ

### Node Details

#### Search GHL Contact NQ
- **Type and technical role:** HighLevel node; looks up existing contact by phone.
- **Configuration choices:** Same pattern as qualified path:
  - `getAll`
  - query filter by phone
  - limit 1
- **Input/output connections:** Input from **Qualified?** false branch; output to **Contact Exists NQ?**
- **Edge cases:** Same as qualified lookup.

#### Contact Exists NQ?
- **Type and technical role:** IF node; determines update vs create branch.
- **Configuration choices:** Checks whether `id` is non-empty.
- **Input/output connections:**
  - True → **Update Contact NQ**
  - False → **Create Contact NQ**

#### Update Contact NQ
- **Type and technical role:** HighLevel node; updates existing nurture contact.
- **Configuration choices:**
  - Contact ID from `id`
  - Tags: `voice-lead,nurturing`
- **Input/output connections:** Output to **Merge Contact NQ**
- **Edge cases:** Sticky note says “not-qualified” but actual configured tags are `voice-lead,nurturing`; note and implementation differ.

#### Create Contact NQ
- **Type and technical role:** HighLevel node; creates nurture contact.
- **Configuration choices:**
  - Phone from **Extract BANT**
  - Additional fields:
    - `tags: voice-lead,nurturing`
    - `source: Voice AI`
    - `firstName: Voice Lead`
- **Input/output connections:** Output to **Merge Contact NQ**
- **Edge cases:** Duplicate creation possible if lookup misses.

#### Merge Contact NQ
- **Type and technical role:** Merge node in choose-branch mode.
- **Configuration choices:** Rejoins create/update branch.
- **Input/output connections:** Output to **Create Nurturing Opportunity**

#### Create Nurturing Opportunity
- **Type and technical role:** HighLevel node; creates a nurture-stage opportunity.
- **Configuration choices:**
  - Opportunity name: `Voice Lead (Nurturing) — {phone}`
  - Resource: `opportunity`
  - Pipeline ID: `YOUR_PIPELINE_ID`
  - Stage ID: `YOUR_NURTURING_STAGE_ID`
  - Monetary value: `0`
- **Input/output connections:** Output to **Trigger Nurture Workflow**
- **Edge cases:** Wrong IDs or duplicate opportunities.

#### Trigger Nurture Workflow
- **Type and technical role:** HTTP Request node; directly invokes LeadConnector workflow enrollment API.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://services.leadconnectorhq.com/contacts/{{ $json.id }}/workflow/YOUR_NURTURE_WORKFLOW_ID`
  - Headers:
    - `Authorization: Bearer {{$env.GHL_API_KEY}}`
    - `Content-Type: application/json`
    - `Version: 2021-04-15`
  - `neverError: true`
- **Input/output connections:** Fans out to:
  - **Log to Supabase NQ**
  - **Log to Google Sheets NQ**
- **Version-specific requirements:** HTTP Request node version 4.2.
- **Edge cases or potential failure types:**
  - Requires `GHL_API_KEY` env var, separate from OAuth2 credential used by HighLevel nodes.
  - `neverError: true` means failed API responses may not stop the flow; bad enrollments can be silently logged as if successful unless response is inspected.
  - Placeholder workflow ID must be replaced.
- **Sub-workflow reference:** External GHL workflow, not an n8n sub-workflow.

#### Log to Supabase NQ
- **Type and technical role:** Supabase node; logs non-qualified inbound calls.
- **Configuration choices:** Inserts a smaller subset than the qualified branch, including:
  - call metadata
  - BANT details
  - `qualified = false`
  - `ghl_contact_id`
- **Input/output connections:** Input from **Trigger Nurture Workflow**
- **Edge cases:** Same schema and auth risks as the qualified branch.

#### Log to Google Sheets NQ
- **Type and technical role:** Google Sheets node; appends a simplified non-qualified row.
- **Configuration choices:**
  - `CRM Note` hardcoded as `Not qualified — added to nurturing pipeline`
  - `Qualified` hardcoded `false`
  - no opportunity ID or objection response
- **Input/output connections:** Input from **Trigger Nurture Workflow**
- **Edge cases:** Same spreadsheet/header issues as qualified logging.

---

## 2.6 Scheduled outbound nurture calling

**Overview:**  
This block runs every weekday morning and automates outbound dialing for nurturing leads. It retrieves relevant contacts from GHL, asks GPT-4o Mini to rank them, batches them to reduce provider pressure, initiates Vapi outbound calls, and logs each initiated attempt.

**Nodes Involved:**  
- Daily Outbound Schedule1
- Fetch GHL Nurturing Leads1
- Priority Ranker1
- GPT-4o Mini (Ranker)1
- Priority Ranker Output Parser1
- Extract Ranked Leads1
- Split in Batches1
- Initiate Vapi Outbound Call1
- Log Outbound to Supabase1
- Log Outbound to Sheets1

### Node Details

#### Daily Outbound Schedule1
- **Type and technical role:** Schedule Trigger; starts outbound campaign.
- **Configuration choices:**
  - Cron: `0 9 * * 1-5`
  - Runs at 9:00 AM Monday to Friday
- **Input/output connections:** Starts flow to **Fetch GHL Nurturing Leads1**
- **Version-specific requirements:** Schedule Trigger version 1.2.
- **Edge cases:** Time zone depends on n8n instance settings.

#### Fetch GHL Nurturing Leads1
- **Type and technical role:** HighLevel node; fetches contacts matching nurture-related query.
- **Configuration choices:**
  - Operation: `getAll`
  - Return all: true
  - Filter query: `nurturing`
- **Input/output connections:** Output to **Priority Ranker1**
- **Edge cases:**
  - This is a broad text query, not an explicit stage filter.
  - It may return contacts tagged or text-matched with “nurturing,” not necessarily only opportunity-stage nurture leads.
  - Large result sets may affect performance.

#### Priority Ranker1
- **Type and technical role:** LangChain Agent; ranks outbound call priority.
- **Configuration choices:**
  - Prompt passes all fetched lead data as JSON array.
  - System message defines weighting:
    - recency 40%
    - engagement 35%
    - timing/day-of-week 25%
  - Requires valid phone number
  - Limit top 50
- **Key expressions or variables used:**
  - `{{ $now.toISO() }}`
  - `{{ $now.toFormat('EEEE') }}`
  - `{{ JSON.stringify($input.all().map(...), null, 2) }}`
- **Input/output connections:**
  - Main input from **Fetch GHL Nurturing Leads1**
  - LLM from **GPT-4o Mini (Ranker)1**
  - Parser from **Priority Ranker Output Parser1**
  - Output to **Extract Ranked Leads1**
- **Version-specific requirements:** LangChain Agent version 1.8.
- **Edge cases:** Ranking quality depends on returned GHL fields actually containing usable dates/tags.

#### GPT-4o Mini (Ranker)1
- **Type and technical role:** OpenAI chat model.
- **Configuration choices:**
  - Model: `gpt-4o-mini`
  - Max tokens: `4000`
  - Temperature: `0.1`
- **Input/output connections:** Supplies model for **Priority Ranker1**
- **Version-specific requirements:** OpenAI node version 1.2.
- **Edge cases:** Token pressure if many contacts are returned.

#### Priority Ranker Output Parser1
- **Type and technical role:** Structured output parser.
- **Configuration choices:** Requires object with `ranked_leads`, each containing:
  - `contact_id`
  - `phone`
  - `priority_score`
  - `reason`
- **Input/output connections:** Parser for **Priority Ranker1**

#### Extract Ranked Leads1
- **Type and technical role:** Code node; validates, sorts, filters, and explodes ranked leads into items.
- **Configuration choices:**
  - Parses agent output defensively
  - Sorts descending by `priority_score`
  - Filters invalid or empty phone values
  - Limits to 50
  - If none remain, returns one item:
    - `{ skip: true, message: 'No leads with phone numbers to call today.' }`
  - Otherwise returns one item per lead
- **Input/output connections:** Output to **Split in Batches1**
- **Edge cases:**
  - The `skip: true` item is still forwarded downstream; there is no IF node to stop outbound requests. That means later nodes may try to place a call with missing phone data unless Vapi request body was intentionally left blank for later manual completion.
  - Sorting duplicates the AI’s expected sorting but provides extra safety.

#### Split in Batches1
- **Type and technical role:** Split In Batches; throttles outbound requests.
- **Configuration choices:**
  - Batch size: `5`
- **Input/output connections:** Output to **Initiate Vapi Outbound Call1**
- **Version-specific requirements:** Version 3.
- **Edge cases:**
  - No loop-back connection is shown, so this node may only emit the first batch depending on n8n execution pattern. In most batch-based designs, an explicit loop is added to fetch the next batch. As configured, this is a likely implementation gap.
- **Sub-workflow reference:** None.

#### Initiate Vapi Outbound Call1
- **Type and technical role:** HTTP Request node; initiates an outbound phone call via Vapi.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://api.vapi.ai/call/phone`
  - Headers:
    - `Authorization: Bearer {{$env.VAPI_API_KEY}}`
    - `Content-Type: application/json`
  - `neverError: true`
  - `sendBody: true`, but the body parameters are effectively empty in the JSON
- **Input/output connections:** Fans out to:
  - **Log Outbound to Supabase1**
  - **Log Outbound to Sheets1**
- **Version-specific requirements:** HTTP Request node version 4.2.
- **Edge cases or potential failure types:**
  - This node appears incomplete as shown: Vapi outbound calls typically require a request body containing target phone number and assistant/phone number configuration.
  - Because `neverError: true` is enabled, unsuccessful API calls may still proceed to logging.
  - Requires `VAPI_API_KEY` env var.
- **Sub-workflow reference:** External Vapi API, not an n8n sub-workflow.

#### Log Outbound to Supabase1
- **Type and technical role:** Supabase node; logs outbound call initiation.
- **Configuration choices:**
  - Table: `voice_call_logs`
  - Direction: `outbound`
  - Provider: `vapi`
  - Duration: `0`
  - `qualified: false`
  - Uses outbound Vapi response `id` when available; fallback `outbound-{timestamp}`
  - Includes CRM note with priority score and ranking reason
- **Input/output connections:** Input from **Initiate Vapi Outbound Call1**
- **Edge cases:** If Vapi call failed but still returned a response, outbound log may not reflect actual success state.

#### Log Outbound to Sheets1
- **Type and technical role:** Google Sheets node; appends outbound initiation row.
- **Configuration choices:**
  - Direction: `outbound`
  - Provider: `vapi`
  - CRM Note includes priority score and reason
  - Duration 0, BANT empty
- **Input/output connections:** Input from **Initiate Vapi Outbound Call1**
- **Edge cases:** Same logging caveat as Supabase.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Setup Guide | Sticky Note | Deployment/setup instructions |  |  | ## 🚀 Voice AI + GHL Sales Agent — Setup Guide<br>**STEP 1 — Credentials**<br>- Add `HighLevel OAuth2` credential (OAuth2 flow via GHL marketplace app)<br>- Add `Anthropic` API key credential<br>- Add `OpenAI` API key credential<br>- Add `Google Gemini (PaLM) API` credential<br>- Add `Supabase` credential (URL + Service Role Key)<br>- Add `Google Sheets OAuth2` credential<br><br>**STEP 2 — Supabase**<br>- Run the CREATE TABLE SQL shown in the Supabase Schema note<br>- Replace `YOUR_SUPABASE_CRED_ID` in all Supabase nodes<br><br>**STEP 3 — GoHighLevel**<br>- Replace `YOUR_HL_CRED_ID` with your GHL credential ID<br>- Replace `YOUR_PIPELINE_ID` with your GHL pipeline ID<br>- Replace `YOUR_HOT_STAGE_ID` with the Hot Lead stage ID<br>- Replace `YOUR_NURTURING_STAGE_ID` with the Nurturing stage ID<br>- Replace `YOUR_NURTURE_WORKFLOW_ID` with your GHL automation workflow ID<br><br>**STEP 4 — Voice Provider (Vapi)**<br>- Set env var `VAPI_API_KEY` in n8n Settings → Environment Variables<br>- Replace `YOUR_VAPI_PHONE_NUMBER_ID` with your Vapi phone number ID<br>- Replace `YOUR_VAPI_ASSISTANT_ID` with your Vapi assistant ID<br>- Inbound webhook URL: `https://YOUR_N8N_URL/webhook/voice-sales-inbound`<br>- Configure Vapi End-of-Call webhook to POST to the inbound URL<br>- For Retell: configure `call_ended` webhook to POST to same URL<br><br>**STEP 5 — Google Sheets**<br>- Create a spreadsheet with sheet named `Voice Call Log`<br>- Add all column headers listed in the Sheets Columns note<br>- Replace `YOUR_SPREADSHEET_ID` with the spreadsheet ID from the URL<br><br>**STEP 6 — GHL API Key (for Trigger Nurture Workflow)**<br>- Set env var `GHL_API_KEY` in n8n Settings → Environment Variables<br>- This is your GHL Location API key (Settings → API Keys)<br><br>**STEP 7 — Activate**<br>- Activate the workflow<br>- Test with a short test call to verify BANT scoring<br>- Monitor the Voice Call Log sheet for entries |
| Supabase Schema | Sticky Note | Database schema note |  |  | ## 🗄️ Supabase Table Schema<br>```sql<br>CREATE TABLE voice_call_logs ( ... )<br>``` |
| Google Sheets Columns | Sticky Note | Spreadsheet column note |  |  | ## 📊 Google Sheets Columns<br>Sheet name: **Voice Call Log**<br>Headers: Call ID, Date, Phone Number, Direction, Provider, Duration (sec), BANT Score, Qualified, Appointment Requested, GHL Contact ID, GHL Opp ID, CRM Note, Objection Response |
| HighLevel IDs | Sticky Note | Placeholder ID reference |  |  | ## 🔑 HighLevel Placeholder IDs<br>`YOUR_HL_CRED_ID`, `YOUR_PIPELINE_ID`, `YOUR_HOT_STAGE_ID`, `YOUR_NURTURING_STAGE_ID`, `YOUR_NURTURE_WORKFLOW_ID`<br>Also includes Vapi IDs and where to find them |
| INBOUND FLOW | Sticky Note | Section label |  |  | ## 📞 INBOUND FLOW — Vapi / Retell Webhook → BANT → Booking → HighLevel CRM |
| Inbound Call Webhook | Webhook | Receives inbound call-ended webhook |  | Normalize Call Payload | Receives end-of-call report from Vapi or Retell — auto 200 response |
| Normalize Call Payload | Code | Normalizes provider payloads | Inbound Call Webhook | Valid Call? | Normalises Vapi + Retell payloads → unified: call_id, phone, transcript, duration_sec, provider |
| Valid Call? | IF | Filters short/low-content calls | Normalize Call Payload | BANT Qualifier | Skip calls < 30 s or transcript < 50 chars |
| BANT Qualifier | LangChain Agent | BANT scoring and objection extraction | Valid Call? | Extract BANT | Claude Haiku scores Budget · Authority · Need · Timeline (0–10) from transcript |
| Claude Haiku (BANT) | Anthropic Chat Model | LLM for BANT analysis |  | BANT Qualifier | Claude Haiku scores Budget · Authority · Need · Timeline (0–10) from transcript |
| BANT Output Parser | Structured Output Parser | Enforces BANT JSON schema |  | BANT Qualifier | Claude Haiku scores Budget · Authority · Need · Timeline (0–10) from transcript |
| Extract BANT | Code | Parses and normalizes BANT result | BANT Qualifier | Qualified? | Merges BANT score + objections into working object |
| Qualified? | IF | Routes qualified vs nurture path | Extract BANT | Booking Intent Detector; Search GHL Contact NQ | score ≥ 6 → Booking path  \|  score < 6 → Nurture path |
| Booking Intent Detector | LangChain Agent | Detects appointment intent and objections | Qualified? | Extract Booking | GPT-4o detects appointment intent + preferred time + confidence |
| GPT-4o (Booking) | OpenAI Chat Model | LLM for booking detection |  | Booking Intent Detector | GPT-4o detects appointment intent + preferred time + confidence |
| Booking Output Parser | Structured Output Parser | Enforces booking JSON schema |  | Booking Intent Detector | GPT-4o detects appointment intent + preferred time + confidence |
| Extract Booking | Code | Parses booking output | Booking Intent Detector | Search GHL Contact | Extracts appointment_requested, preferred_time, objections[] |
| Search GHL Contact | HighLevel | Lookup contact by phone | Extract Booking | Contact Exists? | Search HighLevel contact by phone number |
| Contact Exists? | IF | Selects update vs create contact path | Search GHL Contact | Update Contact; Create Contact | TRUE → update existing  \|  FALSE → create new |
| Update Contact | HighLevel | Updates existing qualified contact | Contact Exists? | Merge Contact Paths | Tag existing HL contact: voice-lead, qualified |
| Create Contact | HighLevel | Creates new qualified contact | Contact Exists? | Merge Contact Paths | Create HL contact with phone + source = Voice AI |
| Merge Contact Paths | Merge | Rejoins qualified contact upsert branches | Update Contact; Create Contact | Create Hot Opportunity | Rejoin update + create branches with resolved contact ID |
| Create Hot Opportunity | HighLevel | Creates hot opportunity in GHL | Merge Contact Paths | Objection Handler | Open opportunity in Hot Lead pipeline stage |
| Objection Handler | LangChain Agent | Generates objection-handling script | Create Hot Opportunity | Extract Objection Response | Claude Sonnet generates rebuttal script using feel-felt-found / Challenger Sale |
| Claude Sonnet (Objection) | Anthropic Chat Model | LLM for objection response generation |  | Objection Handler | Claude Sonnet generates rebuttal script using feel-felt-found / Challenger Sale |
| Objection Output Parser | Structured Output Parser | Enforces objection-response schema |  | Objection Handler | Claude Sonnet generates rebuttal script using feel-felt-found / Challenger Sale |
| Extract Objection Response | Code | Consolidates qualified lead context | Objection Handler | CRM Note Writer | Extracts response_script, technique, confidence |
| CRM Note Writer | LangChain Agent | Generates structured CRM note | Extract Objection Response | Extract CRM Note | Gemini 2.0 Flash writes professional CRM note from call summary |
| Gemini Flash (CRM Note) | Google Gemini Chat Model | LLM for CRM note generation |  | CRM Note Writer | Gemini 2.0 Flash writes professional CRM note from call summary |
| CRM Note Output Parser | Structured Output Parser | Enforces CRM note schema |  | CRM Note Writer | Gemini 2.0 Flash writes professional CRM note from call summary |
| Extract CRM Note | Code | Parses final CRM note string | CRM Note Writer | Update Contact Notes | Extracts crm_note string from agent output |
| Update Contact Notes | HighLevel | Updates qualified/hot contact tags | Extract CRM Note | Update Stage: Hot Lead | Append AI CRM note to HL contact record |
| Update Stage: Hot Lead | HighLevel | Moves opportunity to hot stage | Update Contact Notes | Log to Supabase; Log to Google Sheets | Move HL opportunity to Hot Lead stage |
| Log to Supabase | Supabase | Logs qualified inbound call | Update Stage: Hot Lead |  | Persist qualified inbound call to voice_call_logs |
| Log to Google Sheets | Google Sheets | Logs qualified inbound row | Update Stage: Hot Lead |  | Append summary row to Voice Call Log sheet |
| Search GHL Contact NQ | HighLevel | Lookup non-qualified contact by phone | Qualified? | Contact Exists NQ? | Search HL contact by phone before nurture upsert |
| Contact Exists NQ? | IF | Selects non-qualified update/create path | Search GHL Contact NQ | Update Contact NQ; Create Contact NQ | TRUE → update  \|  FALSE → create |
| Update Contact NQ | HighLevel | Updates existing nurture contact | Contact Exists NQ? | Merge Contact NQ | Tag contact: not-qualified, add to nurture queue |
| Create Contact NQ | HighLevel | Creates new nurture contact | Contact Exists NQ? | Merge Contact NQ | Create HL contact, flag for nurture sequence |
| Merge Contact NQ | Merge | Rejoins non-qualified upsert branches | Update Contact NQ; Create Contact NQ | Create Nurturing Opportunity | Rejoin upsert paths with resolved contact ID |
| Create Nurturing Opportunity | HighLevel | Creates nurture-stage opportunity | Merge Contact NQ | Trigger Nurture Workflow | Open opportunity in Nurturing pipeline stage |
| Trigger Nurture Workflow | HTTP Request | Enrolls contact in GHL nurture automation | Create Nurturing Opportunity | Log to Supabase NQ; Log to Google Sheets NQ | Fire HL automation to enrol contact in nurture email/SMS sequence |
| Log to Supabase NQ | Supabase | Logs non-qualified inbound call | Trigger Nurture Workflow |  | Persist not-qualified call to voice_call_logs |
| Log to Google Sheets NQ | Google Sheets | Logs non-qualified inbound row | Trigger Nurture Workflow |  | Append not-qualified row to sheet |
| OUTBOUND FLOW1 | Sticky Note | Section label |  |  | ## 📤 OUTBOUND FLOW — Daily Prioritised Outbound Calls via Vapi |
| Daily Outbound Schedule1 | Schedule Trigger | Starts outbound campaign weekdays at 9 AM |  | Fetch GHL Nurturing Leads1 | 9:00 AM Mon–Fri · cron: 0 9 * * 1-5 |
| Fetch GHL Nurturing Leads1 | HighLevel | Retrieves nurturing leads | Daily Outbound Schedule1 | Priority Ranker1 | Retrieve all Nurturing stage contacts from HighLevel |
| Priority Ranker1 | LangChain Agent | Ranks leads for outbound calling | Fetch GHL Nurturing Leads1 | Extract Ranked Leads1 | GPT-4o Mini ranks leads by conversion priority to maximise call ROI |
| GPT-4o Mini (Ranker)1 | OpenAI Chat Model | LLM for lead ranking |  | Priority Ranker1 | GPT-4o Mini ranks leads by conversion priority to maximise call ROI |
| Priority Ranker Output Parser1 | Structured Output Parser | Enforces ranked lead schema |  | Priority Ranker1 | GPT-4o Mini ranks leads by conversion priority to maximise call ROI |
| Extract Ranked Leads1 | Code | Sorts, filters, limits ranked leads | Priority Ranker1 | Split in Batches1 | Sort ranked_leads by priority_score desc · limit 50 |
| Split in Batches1 | Split In Batches | Throttles outbound requests in groups of 5 | Extract Ranked Leads1 | Initiate Vapi Outbound Call1 | 5 leads per batch — respects Vapi rate limits |
| Initiate Vapi Outbound Call1 | HTTP Request | Starts outbound Vapi call | Split in Batches1 | Log Outbound to Supabase1; Log Outbound to Sheets1 | POST outbound call to Vapi API for each prioritised lead |
| Log Outbound to Supabase1 | Supabase | Logs outbound initiation | Initiate Vapi Outbound Call1 |  | Record outbound initiation in voice_call_logs |
| Log Outbound to Sheets1 | Google Sheets | Logs outbound initiation row | Initiate Vapi Outbound Call1 |  | Append outbound row to Voice Call Log sheet |
| _Inbound Call Webhook | Sticky Note | Node comment |  |  | Receives end-of-call report from Vapi or Retell — auto 200 response |
| _Normalize Call Payload | Sticky Note | Node comment |  |  | Normalises Vapi + Retell payloads → unified: call_id, phone, transcript, duration_sec, provider |
| _Valid Call? | Sticky Note | Node comment |  |  | Skip calls < 30 s or transcript < 50 chars |
| _BANT Qualifier | Sticky Note | Node comment |  |  | Claude Haiku scores Budget · Authority · Need · Timeline (0–10) from transcript |
| _Extract BANT | Sticky Note | Node comment |  |  | Merges BANT score + objections into working object |
| _Qualified? | Sticky Note | Node comment |  |  | score ≥ 6 → Booking path  \|  score < 6 → Nurture path |
| _Booking Intent Detector | Sticky Note | Node comment |  |  | GPT-4o detects appointment intent + preferred time + confidence |
| _Extract Booking | Sticky Note | Node comment |  |  | Extracts appointment_requested, preferred_time, objections[] |
| _Search GHL Contact | Sticky Note | Node comment |  |  | Search HighLevel contact by phone number |
| _Contact Exists? | Sticky Note | Node comment |  |  | TRUE → update existing  \|  FALSE → create new |
| _Update Contact | Sticky Note | Node comment |  |  | Tag existing HL contact: voice-lead, qualified |
| _Create Contact | Sticky Note | Node comment |  |  | Create HL contact with phone + source = Voice AI |
| _Merge Contact Paths | Sticky Note | Node comment |  |  | Rejoin update + create branches with resolved contact ID |
| _Create Hot Opportunity | Sticky Note | Node comment |  |  | Open opportunity in Hot Lead pipeline stage |
| _Objection Handler | Sticky Note | Node comment |  |  | Claude Sonnet generates rebuttal script using feel-felt-found / Challenger Sale |
| _Extract Objection Response | Sticky Note | Node comment |  |  | Extracts response_script, technique, confidence |
| _CRM Note Writer | Sticky Note | Node comment |  |  | Gemini 2.0 Flash writes professional CRM note from call summary |
| _Extract CRM Note | Sticky Note | Node comment |  |  | Extracts crm_note string from agent output |
| _Update Contact Notes | Sticky Note | Node comment |  |  | Append AI CRM note to HL contact record |
| _Update Stage: Hot Lead | Sticky Note | Node comment |  |  | Move HL opportunity to Hot Lead stage |
| _Log to Supabase | Sticky Note | Node comment |  |  | Persist qualified inbound call to voice_call_logs |
| _Log to Google Sheets | Sticky Note | Node comment |  |  | Append summary row to Voice Call Log sheet |
| _Search GHL Contact NQ | Sticky Note | Node comment |  |  | Search HL contact by phone before nurture upsert |
| _Contact Exists NQ? | Sticky Note | Node comment |  |  | TRUE → update  \|  FALSE → create |
| _Update Contact NQ | Sticky Note | Node comment |  |  | Tag contact: not-qualified, add to nurture queue |
| _Create Contact NQ | Sticky Note | Node comment |  |  | Create HL contact, flag for nurture sequence |
| _Merge Contact NQ | Sticky Note | Node comment |  |  | Rejoin upsert paths with resolved contact ID |
| _Create Nurturing Opportunity | Sticky Note | Node comment |  |  | Open opportunity in Nurturing pipeline stage |
| _Trigger Nurture Workflow | Sticky Note | Node comment |  |  | Fire HL automation to enrol contact in nurture email/SMS sequence |
| _Log to Supabase NQ | Sticky Note | Node comment |  |  | Persist not-qualified call to voice_call_logs |
| _Log to Google Sheets NQ | Sticky Note | Node comment |  |  | Append not-qualified row to sheet |
| _Daily Outbound Schedule | Sticky Note | Node comment |  |  | 9:00 AM Mon–Fri · cron: 0 9 * * 1-5 |
| _Fetch GHL Nurturing Leads | Sticky Note | Node comment |  |  | Retrieve all Nurturing stage contacts from HighLevel |
| _Priority Ranker | Sticky Note | Node comment |  |  | GPT-4o Mini ranks leads by conversion priority to maximise call ROI |
| _Extract Ranked Leads | Sticky Note | Node comment |  |  | Sort ranked_leads by priority_score desc · limit 50 |
| _Split in Batches | Sticky Note | Node comment |  |  | 5 leads per batch — respects Vapi rate limits |
| _Initiate Vapi Outbound Call | Sticky Note | Node comment |  |  | POST outbound call to Vapi API for each prioritised lead |
| _Log Outbound to Supabase | Sticky Note | Node comment |  |  | Record outbound initiation in voice_call_logs |
| _Log Outbound to Sheets | Sticky Note | Node comment |  |  | Append outbound row to Voice Call Log sheet |

---

# 4. Reproducing the Workflow from Scratch

1. **Create required credentials in n8n**
   1. Add a **HighLevel OAuth2** credential.
   2. Add an **Anthropic API** credential.
   3. Add an **OpenAI API** credential.
   4. Add a **Google Gemini / PaLM API** credential.
   5. Add a **Supabase** credential using project URL and service role key.
   6. Add a **Google Sheets OAuth2** credential.
   7. In n8n environment variables, define:
      - `VAPI_API_KEY`
      - `GHL_API_KEY`

2. **Prepare Supabase**
   1. Create table `voice_call_logs` with the schema shown in the workflow note.
   2. Ensure these columns exist exactly:
      - `call_id`, `phone_number`, `direction`, `provider`, `duration_sec`, `transcript`, `recording_url`, `bant_score`, `qualified`, `budget_confirmed`, `authority_confirmed`, `need_identified`, `timeline_defined`, `bant_summary`, `key_objections`, `appointment_requested`, `preferred_time`, `ghl_contact_id`, `ghl_opp_id`, `crm_note`, `objection_response`, `outbound_call_id`, `created_at`
   3. Create indexes if desired.

3. **Prepare Google Sheets**
   1. Create a spreadsheet.
   2. Create a sheet named **Voice Call Log**.
   3. Add headers in row 1:
      - Call ID
      - Date
      - Phone Number
      - Direction
      - Provider
      - Duration (sec)
      - BANT Score
      - Qualified
      - Appointment Requested
      - GHL Contact ID
      - GHL Opp ID
      - CRM Note
      - Objection Response
   4. Copy the spreadsheet ID from the URL.

4. **Collect GoHighLevel and Vapi identifiers**
   1. Get:
      - pipeline ID
      - hot stage ID
      - nurturing stage ID
      - nurture workflow ID
   2. Also collect:
      - Vapi phone number ID
      - Vapi assistant ID
   3. Replace all placeholders in later nodes.

---

## Build the inbound flow

5. **Add a Webhook node**
   - Name: `Inbound Call Webhook`
   - Type: `Webhook`
   - Method: `POST`
   - Path: `voice-sales-inbound`

6. **Add a Code node**
   - Name: `Normalize Call Payload`
   - Connect from `Inbound Call Webhook`
   - Paste logic that:
     - detects Vapi payloads
     - detects Retell payloads
     - maps them into:
       - `call_id`
       - `phone_number`
       - `transcript`
       - `recording_url`
       - `duration_sec`
       - `provider`
       - `raw_body`
     - normalizes phone numbers to `+` prefixed format

7. **Add an IF node**
   - Name: `Valid Call?`
   - Connect from `Normalize Call Payload`
   - Conditions:
     - `duration_sec >= 30`
     - `transcript.length > 50`
   - Only the true branch is used.

8. **Add an Anthropic Chat Model node**
   - Name: `Claude Haiku (BANT)`
   - Model: `claude-3-haiku-20240307`
   - Temperature: `0.1`
   - Assign Anthropic credential.

9. **Add a Structured Output Parser node**
   - Name: `BANT Output Parser`
   - Manual schema:
     - `score` number
     - `budget_confirmed` boolean
     - `authority_confirmed` boolean
     - `need_identified` boolean
     - `timeline_defined` boolean
     - `summary` string
     - `key_objections` array of strings

10. **Add a LangChain Agent node**
    - Name: `BANT Qualifier`
    - Connect main input from `Valid Call?`
    - Connect AI language model from `Claude Haiku (BANT)`
    - Connect AI output parser from `BANT Output Parser`
    - Prompt should include call phone, duration, provider, and transcript.
    - System message must define the BANT scoring rubric and require valid JSON.

11. **Add a Code node**
    - Name: `Extract BANT`
    - Connect from `BANT Qualifier`
    - Parse `item.json.output` defensively.
    - Strip markdown code fences if needed.
    - Output:
      - `bant_score`
      - `budget_confirmed`
      - `authority_confirmed`
      - `need_identified`
      - `timeline_defined`
      - `bant_summary`
      - `key_objections`
      - `qualified`

12. **Add an IF node**
    - Name: `Qualified?`
    - Connect from `Extract BANT`
    - Condition: `bant_score >= 6`
    - True branch = qualified path
    - False branch = nurturing path

---

## Build the qualified inbound path

13. **Add an OpenAI Chat Model node**
    - Name: `GPT-4o (Booking)`
    - Model: `gpt-4o`
    - Temperature: `0.1`
    - Max tokens: `800`

14. **Add a Structured Output Parser**
    - Name: `Booking Output Parser`
    - Schema:
      - `appointment_requested` boolean
      - `preferred_time` string
      - `confidence` number
      - `objections` array of strings

15. **Add a LangChain Agent**
    - Name: `Booking Intent Detector`
    - Connect from `Qualified?` true branch
    - Connect AI model from `GPT-4o (Booking)`
    - Connect parser from `Booking Output Parser`
    - Prompt should include BANT score, summary, objections, transcript.
    - Require valid JSON.

16. **Add a Code node**
    - Name: `Extract Booking`
    - Connect from `Booking Intent Detector`
    - Parse output and emit:
      - `appointment_requested`
      - `preferred_time`
      - `booking_confidence`
      - `detected_objections`

17. **Add a HighLevel node**
    - Name: `Search GHL Contact`
    - Connect from `Extract Booking`
    - Operation: `getAll`
    - Query filter: phone number
    - Limit: `1`

18. **Add an IF node**
    - Name: `Contact Exists?`
    - Connect from `Search GHL Contact`
    - Condition: `id` is not empty

19. **Add a HighLevel update node**
    - Name: `Update Contact`
    - Connect from `Contact Exists?` true branch
    - Operation: `update`
    - Contact ID from `{{$json.id}}`
    - Set tags to `voice-lead,qualified`

20. **Add a HighLevel create node**
    - Name: `Create Contact`
    - Connect from `Contact Exists?` false branch
    - Create contact with:
      - phone from `Extract Booking`
      - first name `Voice Lead`
      - source `Voice AI`
      - tags `voice-lead,qualified`

21. **Add a Merge node**
    - Name: `Merge Contact Paths`
    - Mode: `chooseBranch`
    - Connect `Update Contact` and `Create Contact`
    - Output goes to the next node.

22. **Add a HighLevel node**
    - Name: `Create Hot Opportunity`
    - Connect from `Merge Contact Paths`
    - Resource: `opportunity`
    - Operation: create
    - Contact ID from merged contact
    - Pipeline ID: your actual GHL pipeline ID
    - Stage ID: your actual hot lead stage ID
    - Name pattern: `Voice Lead — {phone}`
    - Monetary value: `0`

23. **Add an Anthropic Chat Model**
    - Name: `Claude Sonnet (Objection)`
    - Model: `claude-3-5-sonnet-20241022`
    - Temperature: `0.3`

24. **Add a Structured Output Parser**
    - Name: `Objection Output Parser`
    - Schema:
      - `response_script`
      - `technique`
      - `confidence`

25. **Add a LangChain Agent**
    - Name: `Objection Handler`
    - Connect from `Create Hot Opportunity`
    - Connect model from `Claude Sonnet (Objection)`
    - Connect parser from `Objection Output Parser`
    - Prompt should include objections, BANT summary, appointment intent, preferred time.
    - Require a concise objection-handling script.

26. **Add a Code node**
    - Name: `Extract Objection Response`
    - Connect from `Objection Handler`
    - Parse response.
    - Also pull data from upstream nodes using expressions:
      - `$('Extract BANT').item.json`
      - `$('Extract Booking').item.json`
      - `$('Merge Contact Paths').item.json`
      - `$('Create Hot Opportunity').item.json`
    - Build one unified JSON object for the remainder of the path.

27. **Add a Google Gemini Chat Model node**
    - Name: `Gemini Flash (CRM Note)`
    - Choose a valid Gemini Flash model manually.
    - Temperature: `0.2`
    - Max output tokens: `800`

28. **Add a Structured Output Parser**
    - Name: `CRM Note Output Parser`
    - Schema:
      - `crm_note`

29. **Add a LangChain Agent**
    - Name: `CRM Note Writer`
    - Connect from `Extract Objection Response`
    - Connect model from `Gemini Flash (CRM Note)`
    - Connect parser from `CRM Note Output Parser`
    - Prompt should include all call details and require a CRM note with sections:
      - CALL SUMMARY
      - QUALIFICATION STATUS
      - NEXT STEPS
      - OBJECTIONS ON FILE
      - RECOMMENDED APPROACH

30. **Add a Code node**
    - Name: `Extract CRM Note`
    - Connect from `CRM Note Writer`
    - Parse out `crm_note`
    - Merge back with payload from `Extract Objection Response`

31. **Add a HighLevel node**
    - Name: `Update Contact Notes`
    - Connect from `Extract CRM Note`
    - Operation: `update`
    - Contact ID from `ghl_contact_id`
    - Tags: `voice-lead,qualified,hot`
    - If you want true note-writing behavior, extend this node or add another GHL/API node because the shown configuration only updates tags.

32. **Add a HighLevel node**
    - Name: `Update Stage: Hot Lead`
    - Connect from `Update Contact Notes`
    - Resource: `opportunity`
    - Operation: `update`
    - Opportunity ID from `ghl_opp_id`
    - Stage ID: your hot lead stage ID

33. **Add a Supabase node**
    - Name: `Log to Supabase`
    - Connect from `Update Stage: Hot Lead`
    - Table: `voice_call_logs`
    - Map all qualified inbound fields including transcript, score, objections, booking info, IDs, note, objection response.

34. **Add a Google Sheets node**
    - Name: `Log to Google Sheets`
    - Connect from `Update Stage: Hot Lead`
    - Operation: `append`
    - Spreadsheet ID: your spreadsheet ID
    - Sheet name: `Voice Call Log`
    - Map columns:
      - Date
      - Call ID
      - Phone Number
      - Direction = inbound
      - Provider
      - Duration (sec)
      - BANT Score
      - Qualified
      - Appointment Requested
      - GHL Contact ID
      - GHL Opp ID
      - CRM Note
      - Objection Response

---

## Build the non-qualified inbound path

35. **Add a HighLevel node**
    - Name: `Search GHL Contact NQ`
    - Connect from `Qualified?` false branch
    - Same contact lookup by phone.

36. **Add an IF node**
    - Name: `Contact Exists NQ?`
    - Condition: contact `id` is not empty

37. **Add a HighLevel update node**
    - Name: `Update Contact NQ`
    - Connect from true branch
    - Update tags to `voice-lead,nurturing`

38. **Add a HighLevel create node**
    - Name: `Create Contact NQ`
    - Connect from false branch
    - Create contact with:
      - phone from `Extract BANT`
      - first name `Voice Lead`
      - source `Voice AI`
      - tags `voice-lead,nurturing`

39. **Add a Merge node**
    - Name: `Merge Contact NQ`
    - Mode: `chooseBranch`
    - Merge create/update outputs

40. **Add a HighLevel node**
    - Name: `Create Nurturing Opportunity`
    - Connect from `Merge Contact NQ`
    - Resource: `opportunity`
    - Create opportunity in your pipeline
    - Stage ID: your nurturing stage ID
    - Name: `Voice Lead (Nurturing) — {phone}`

41. **Add an HTTP Request node**
    - Name: `Trigger Nurture Workflow`
    - Connect from `Create Nurturing Opportunity`
    - Method: `POST`
    - URL:
      `https://services.leadconnectorhq.com/contacts/{{ $json.id }}/workflow/YOUR_NURTURE_WORKFLOW_ID`
    - Headers:
      - `Authorization: Bearer {{ 'Bearer ' + $env.GHL_API_KEY }}` or simply `Bearer {{$env.GHL_API_KEY}}`
      - `Content-Type: application/json`
      - `Version: 2021-04-15`
    - Enable `neverError: true` if you want non-blocking behavior, though inspecting the response is recommended.

42. **Add a Supabase node**
    - Name: `Log to Supabase NQ`
    - Connect from `Trigger Nurture Workflow`
    - Insert non-qualified inbound record with `qualified = false`.

43. **Add a Google Sheets node**
    - Name: `Log to Google Sheets NQ`
    - Connect from `Trigger Nurture Workflow`
    - Append row to `Voice Call Log`
    - Set CRM Note to `Not qualified — added to nurturing pipeline`

---

## Build the outbound flow

44. **Add a Schedule Trigger**
    - Name: `Daily Outbound Schedule1`
    - Cron: `0 9 * * 1-5`

45. **Add a HighLevel node**
    - Name: `Fetch GHL Nurturing Leads1`
    - Connect from schedule
    - Operation: `getAll`
    - Return all: true
    - Query filter: `nurturing`

46. **Add an OpenAI Chat Model**
    - Name: `GPT-4o Mini (Ranker)1`
    - Model: `gpt-4o-mini`
    - Temperature: `0.1`
    - Max tokens: `4000`

47. **Add a Structured Output Parser**
    - Name: `Priority Ranker Output Parser1`
    - Schema:
      - `ranked_leads` array
      - each item contains `contact_id`, `phone`, `priority_score`, `reason`

48. **Add a LangChain Agent**
    - Name: `Priority Ranker1`
    - Connect from `Fetch GHL Nurturing Leads1`
    - Connect model from `GPT-4o Mini (Ranker)1`
    - Connect parser from `Priority Ranker Output Parser1`
    - Pass the fetched leads as JSON array to the prompt.
    - Include ranking logic for recency, engagement, timing, valid phone numbers, and top 50 limit.

49. **Add a Code node**
    - Name: `Extract Ranked Leads1`
    - Connect from `Priority Ranker1`
    - Parse ranked results
    - Sort descending by `priority_score`
    - Filter invalid phones
    - Limit to 50
    - Return one item per lead

50. **Add a Split In Batches node**
    - Name: `Split in Batches1`
    - Connect from `Extract Ranked Leads1`
    - Batch size: `5`
    - If you want all batches to process, add the usual batch loop pattern; the provided JSON does not include it.

51. **Add an HTTP Request node**
    - Name: `Initiate Vapi Outbound Call1`
    - Connect from `Split in Batches1`
    - Method: `POST`
    - URL: `https://api.vapi.ai/call/phone`
    - Headers:
      - `Authorization: Bearer {{$env.VAPI_API_KEY}}`
      - `Content-Type: application/json`
    - Add the required Vapi JSON body. The shared workflow JSON leaves this empty, but in practice you should include at minimum:
      - destination phone number
      - assistant ID
      - phone number ID
      - any metadata you want returned on callback
    - Consider disabling `neverError: true` or inspecting the HTTP response.

52. **Add a Supabase node**
    - Name: `Log Outbound to Supabase1`
    - Connect from `Initiate Vapi Outbound Call1`
    - Insert:
      - `direction = outbound`
      - `provider = vapi`
      - `duration_sec = 0`
      - `qualified = false`
      - `ghl_contact_id`
      - `outbound_call_id`
      - CRM note containing priority score and reason

53. **Add a Google Sheets node**
    - Name: `Log Outbound to Sheets1`
    - Connect from `Initiate Vapi Outbound Call1`
    - Append outbound row to `Voice Call Log`

---

## Final deployment steps

54. **Replace placeholders everywhere**
   - `YOUR_PIPELINE_ID`
   - `YOUR_HOT_STAGE_ID`
   - `YOUR_NURTURING_STAGE_ID`
   - `YOUR_NURTURE_WORKFLOW_ID`
   - `YOUR_SPREADSHEET_ID`

55. **Assign credentials to every applicable node**
   - HighLevel nodes → HighLevel OAuth2 credential
   - Anthropic model nodes → Anthropic credential
   - OpenAI model nodes → OpenAI credential
   - Gemini node → Google Gemini credential
   - Supabase nodes → Supabase credential
   - Google Sheets nodes → Google Sheets OAuth2 credential

56. **Configure voice providers**
   - In Vapi, set end-of-call webhook URL to:
     `https://YOUR_N8N_URL/webhook/voice-sales-inbound`
   - For Retell, configure the `call_ended` webhook to the same URL.

57. **Test inbound**
   - Send a sample Vapi payload.
   - Send a sample Retell payload.
   - Verify:
     - webhook reception
     - BANT scoring
     - GHL contact/opportunity creation
     - Supabase insert
     - Google Sheets append

58. **Test outbound**
   - Trigger the schedule manually or execute downstream nodes.
   - Confirm ranked leads are produced.
   - Confirm Vapi request body is valid.
   - Confirm outbound attempts are logged.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Inbound webhook path is `voice-sales-inbound` | Configure Vapi and Retell end-of-call callbacks to this endpoint |
| Vapi API key must be stored as `VAPI_API_KEY` | n8n Settings → Environment Variables |
| GHL Location API key must be stored as `GHL_API_KEY` | Used by `Trigger Nurture Workflow` HTTP request |
| The Gemini model node has no model selected in the provided workflow JSON | Must be configured manually before activation |
| `Update Contact Notes` does not actually write the generated CRM note in its current configuration | It only updates tags; add a dedicated note-writing step if required |
| Outbound Vapi call node appears incomplete | The HTTP body is empty in the provided JSON and should be completed with phone number, assistant ID, and phone number ID |
| `Split in Batches1` has no visible loop-back branch | Verify whether all batches are processed in your n8n version; otherwise add an explicit loop |
| `Trigger Nurture Workflow` and `Initiate Vapi Outbound Call1` both use `neverError: true` | Useful for non-blocking execution, but can hide failed API calls unless responses are inspected |