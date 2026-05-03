Capture and enrich leads with GPT-4o, Postgres, Slack, Gmail and your CRM

https://n8nworkflows.xyz/workflows/capture-and-enrich-leads-with-gpt-4o--postgres--slack--gmail-and-your-crm-14687


# Capture and enrich leads with GPT-4o, Postgres, Slack, Gmail and your CRM

## 1. Workflow Overview

This workflow captures inbound leads from a webhook-based form submission, validates the minimum required fields, enriches the lead with AI-generated scoring and recommendations, stores the lead in Postgres, assigns it to a sales representative, syncs it to CRM systems, sends a welcome email, notifies the sales team in Slack, and logs the activity for auditing.

Typical use cases include:

- Lead capture from website forms
- Automated qualification before human follow-up
- Routing leads to sales reps
- Multi-system synchronization across database, CRM, email, and team chat
- Activity logging for reporting and traceability

The workflow is organized into the following logical blocks.

### 1.1 Input Reception and Runtime Configuration
Receives a lead payload through an HTTP webhook and injects configurable runtime values such as CRM endpoint, Slack channel, and the list of sales reps.

### 1.2 Validation and Error Handling
Checks that required lead fields are present, specifically `name` and `email`. If validation fails, it returns a structured error payload instead of continuing.

### 1.3 Data Normalization
Extracts lead data from the webhook body and standardizes the shape of the payload used by downstream nodes.

### 1.4 AI Enrichment and Scoring
Uses an OpenAI chat model through the LangChain agent node to classify the lead as hot/warm/cold and generate structured business insights.

### 1.5 Persistence and Assignment
Stores the enriched lead in Postgres, then assigns a sales representative using round-robin logic backed by execution custom data.

### 1.6 CRM Synchronization
Pushes the lead into Salesforce and also sends a generic CRM API request via HTTP. In the current configuration, both branches run in parallel from the assignment step, but the HTTP CRM request is also chained after Salesforce.

### 1.7 Lead Communication
Sends a welcome email to the lead through Gmail.

### 1.8 Team Notification and Audit Logging
Posts the enriched lead summary to Slack and writes a processing record to an activity log table in Postgres.

---

## 2. Block-by-Block Analysis

## 2.1 Input Reception and Runtime Configuration

**Overview:**  
This block receives inbound lead submissions and adds reusable configuration values that the rest of the workflow references. It acts as the entry point and central parameter source.

**Nodes Involved:**
- Lead Form Webhook
- Workflow Configuration

### Node Details

#### Lead Form Webhook
- **Type and technical role:** `n8n-nodes-base.webhook`  
  Entry-point trigger node that listens for incoming HTTP POST requests.
- **Configuration choices:**
  - HTTP method: `POST`
  - Path: `lead-capture`
  - Response mode: `lastNode`
- **Key expressions or variables used:**
  - No expressions in configuration
  - Downstream nodes expect incoming data in `{{$json.body}}`
- **Input and output connections:**
  - Input: none
  - Output: Workflow Configuration
- **Version-specific requirements:**
  - Type version `2.1`
  - Webhook behavior depends on active workflow / test URL vs production URL in n8n
- **Edge cases or potential failure types:**
  - Invalid HTTP method
  - Empty or malformed JSON body
  - Since response mode is `lastNode`, failures later in the flow can affect the webhook response
  - If multiple branches finish differently, final response behavior may be less predictable depending on execution path
- **Sub-workflow reference:** none

#### Workflow Configuration
- **Type and technical role:** `n8n-nodes-base.set`  
  Injects static configuration values into the workflow execution context.
- **Configuration choices:**
  - Adds:
    - `crmApiUrl`
    - `slackChannel`
    - `salesReps` as an array of emails
  - `includeOtherFields: true`, so the incoming webhook data remains available
- **Key expressions or variables used:**
  - Static placeholders:
    - `<__PLACEHOLDER_VALUE__CRM API endpoint URL (HubSpot or GoHighLevel)__>`
    - `<__PLACEHOLDER_VALUE__Slack channel ID for notifications__>`
    - `["user@example.com", ...]`
- **Input and output connections:**
  - Input: Lead Form Webhook
  - Output: Validate Required Fields
- **Version-specific requirements:**
  - Type version `3.4`
- **Edge cases or potential failure types:**
  - Placeholder values must be replaced before production use
  - Empty `salesReps` array will break the round-robin assignment later
- **Sub-workflow reference:** none

---

## 2.2 Validation and Error Handling

**Overview:**  
This block verifies that the inbound payload contains the minimum lead identity fields. If validation fails, the workflow routes to an error payload rather than proceeding to enrichment and CRM actions.

**Nodes Involved:**
- Validate Required Fields
- Handle Validation Error

### Node Details

#### Validate Required Fields
- **Type and technical role:** `n8n-nodes-base.if`  
  Conditional gate that checks required fields.
- **Configuration choices:**
  - Combines conditions with `and`
  - Requires both:
    - `{{$json.body.name}}` not empty
    - `{{$json.body.email}}` not empty
  - Strict type validation enabled
- **Key expressions or variables used:**
  - `={{ $json.body.name }}`
  - `={{ $json.body.email }}`
- **Input and output connections:**
  - Input: Workflow Configuration
  - True output: Normalize Lead Data
  - False output: Handle Validation Error
- **Version-specific requirements:**
  - Type version `2.3`
- **Edge cases or potential failure types:**
  - If `body` is missing entirely, expressions may resolve to undefined and fail validation
  - Does not validate email format, only presence
  - Does not validate phone/company/message structure
- **Sub-workflow reference:** none

#### Handle Validation Error
- **Type and technical role:** `n8n-nodes-base.set`  
  Creates a structured error object for invalid submissions.
- **Configuration choices:**
  - Sets:
    - `error = "Missing required fields: name or email"`
    - `status = "validation_failed"`
    - `timestamp = {{$now.toISO()}}`
- **Key expressions or variables used:**
  - `={{ $now.toISO() }}`
- **Input and output connections:**
  - Input: Validate Required Fields (false branch)
  - Output: none
- **Version-specific requirements:**
  - Type version `3.4`
- **Edge cases or potential failure types:**
  - This path does not notify admins or persist failed submissions
  - With webhook response mode set to `lastNode`, this node likely becomes the webhook response body on validation failure
- **Sub-workflow reference:** none

---

## 2.3 Data Normalization

**Overview:**  
This block converts the inbound webhook body into a clean, flat data structure and fills optional values with defaults. It ensures downstream AI and storage nodes receive predictable fields.

**Nodes Involved:**
- Normalize Lead Data

### Node Details

#### Normalize Lead Data
- **Type and technical role:** `n8n-nodes-base.set`  
  Transforms raw request payload into normalized lead fields.
- **Configuration choices:**
  - Maps:
    - `name = $json.body.name`
    - `email = $json.body.email`
    - `phone = $json.body.phone || ''`
    - `company = $json.body.company || ''`
    - `message = $json.body.message || ''`
    - `timestamp = $now.toISO()`
  - Keeps all original fields with `includeOtherFields: true`
- **Key expressions or variables used:**
  - `={{ $json.body.name }}`
  - `={{ $json.body.email }}`
  - `={{ $json.body.phone || '' }}`
  - `={{ $json.body.company || '' }}`
  - `={{ $json.body.message || '' }}`
  - `={{ $now.toISO() }}`
- **Input and output connections:**
  - Input: Validate Required Fields (true branch)
  - Output: AI Lead Enrichment & Scoring
- **Version-specific requirements:**
  - Type version `3.4`
- **Edge cases or potential failure types:**
  - Name may be a single word, which affects later first/last name splitting
  - Empty optional fields may reduce AI quality
- **Sub-workflow reference:** none

---

## 2.4 AI Enrichment and Scoring

**Overview:**  
This block performs AI-based lead qualification. It uses a chat model and a structured output schema so later nodes can reliably consume lead score, reasoning, estimated value, insights, and recommended action.

**Nodes Involved:**
- AI Lead Enrichment & Scoring
- OpenAI Chat Model
- Structured Output Parser

### Node Details

#### AI Lead Enrichment & Scoring
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  AI agent node orchestrating prompt execution with a connected language model and output parser.
- **Configuration choices:**
  - Prompt text includes:
    - name
    - email
    - phone
    - company
    - message
  - System message instructs the model to:
    - classify as hot/warm/cold
    - analyze quality and intent
    - provide enrichment insights
    - return structured output per parser schema
  - `promptType: define`
  - `hasOutputParser: true`
- **Key expressions or variables used:**
  - `{{ $json.name }}`
  - `{{ $json.email }}`
  - `{{ $json.phone }}`
  - `{{ $json.company }}`
  - `{{ $json.message }}`
- **Input and output connections:**
  - Main input: Normalize Lead Data
  - AI language model input: OpenAI Chat Model
  - AI output parser input: Structured Output Parser
  - Main output: Store Lead in Database
- **Version-specific requirements:**
  - Type version `3`
  - Requires compatible LangChain nodes in the n8n instance
- **Edge cases or potential failure types:**
  - Model credential issues
  - Parser mismatch if the model returns invalid schema
  - Rate limits or timeout from OpenAI
  - Missing optional fields may lead to weak enrichment
- **Sub-workflow reference:** none

#### OpenAI Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Provides the LLM used by the AI agent.
- **Configuration choices:**
  - Model: `gpt-4o-mini`
  - No additional options configured
- **Key expressions or variables used:** none
- **Input and output connections:**
  - Output: AI Lead Enrichment & Scoring via `ai_languageModel`
- **Version-specific requirements:**
  - Type version `1.3`
  - Requires OpenAI credentials
  - The configured model must be available to the account
- **Edge cases or potential failure types:**
  - Invalid API key
  - Model availability changes
  - Token/rate limit issues
- **Sub-workflow reference:** none

#### Structured Output Parser
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`  
  Forces the AI result into a predefined JSON schema.
- **Configuration choices:**
  - Manual schema with properties:
    - `leadScore`: enum `hot|warm|cold`
    - `scoreReason`: string
    - `estimatedValue`: number
    - `companyInsights`: string
    - `recommendedAction`: string
  - Required fields:
    - `leadScore`
    - `scoreReason`
    - `estimatedValue`
    - `recommendedAction`
- **Key expressions or variables used:** none
- **Input and output connections:**
  - Output: AI Lead Enrichment & Scoring via `ai_outputParser`
- **Version-specific requirements:**
  - Type version `1.3`
- **Edge cases or potential failure types:**
  - If the model returns text not conforming to schema, parsing can fail
  - `companyInsights` is optional in schema, so downstream nodes should tolerate it being missing or empty
- **Sub-workflow reference:** none

---

## 2.5 Persistence and Assignment

**Overview:**  
This block first persists the enriched lead record in Postgres, then assigns the lead to a sales rep using a rotating index stored in execution custom data.

**Nodes Involved:**
- Store Lead in Database
- Round-Robin Sales Assignment

### Node Details

#### Store Lead in Database
- **Type and technical role:** `n8n-nodes-base.postgres`  
  Inserts the processed lead into the `public.leads` table.
- **Configuration choices:**
  - Schema: `public`
  - Table: `leads`
  - Manually mapped columns:
    - `name`
    - `email`
    - `phone`
    - `company`
    - `message`
    - `created_at`
    - `lead_score`
    - `score_reason`
    - `estimated_value`
    - `company_insights`
    - `recommended_action`
- **Key expressions or variables used:**
  - Base fields from normalized data
  - AI fields from `{{$json.output.*}}`
- **Input and output connections:**
  - Input: AI Lead Enrichment & Scoring
  - Output: Round-Robin Sales Assignment
- **Version-specific requirements:**
  - Type version `2.6`
  - Requires Postgres credentials and table schema to exist
- **Edge cases or potential failure types:**
  - DB auth/connectivity failure
  - Table or column mismatch
  - Type mismatch, especially `estimated_value`
  - Nullability constraints if DB schema is stricter than workflow assumptions
- **Sub-workflow reference:** none

#### Round-Robin Sales Assignment
- **Type and technical role:** `n8n-nodes-base.code`  
  Custom JavaScript node that assigns a sales rep in a rotating sequence.
- **Configuration choices:**
  - Loads `salesReps` from `Workflow Configuration`
  - Reads current index from `$execution.customData`
  - Selects `salesReps[currentIndex % salesReps.length]`
  - Increments and stores the index for next execution
  - Returns original JSON plus `assignedSalesRep`
- **Key expressions or variables used:**
  - `$('Workflow Configuration').first().json.salesReps`
  - `$execution.customData.get('salesRepIndex')`
  - `$execution.customData.set('salesRepIndex', currentIndex + 1)`
- **Input and output connections:**
  - Input: Store Lead in Database
  - Outputs:
    - Send to CRM
    - Create Salesforce Lead
- **Version-specific requirements:**
  - Type version `2`
  - Depends on execution custom data support in the deployed n8n version/environment
- **Edge cases or potential failure types:**
  - Empty or missing `salesReps` array causes division/modulo issues or undefined assignments
  - `customData` persistence semantics can vary by environment and scaling mode
  - Parallel workflow executions may create race conditions in round-robin distribution
- **Sub-workflow reference:** none

---

## 2.6 CRM Synchronization

**Overview:**  
This block pushes the lead into CRM destinations. The current design mixes a Salesforce lead creation step and a generic HTTP CRM sync step, which may be intentional for dual-write or may create duplicate downstream records depending on the target systems.

**Nodes Involved:**
- Create Salesforce Lead
- Send to CRM

### Node Details

#### Create Salesforce Lead
- **Type and technical role:** `n8n-nodes-base.salesforce`  
  Creates a lead record in Salesforce.
- **Configuration choices:**
  - `company = {{$json.company || 'Unknown'}}`
  - `lastname = {{$json.name.split(' ').slice(1).join(' ') || $json.name}}`
  - Additional fields:
    - `email`
    - `owner = assignedSalesRep`
    - `phone`
    - `rating` mapped from AI score
    - `status` mapped from AI score
    - `firstname = name first token`
    - `leadSource = Web Form`
    - `description = message`
- **Key expressions or variables used:**
  - `{{$json.company || 'Unknown'}}`
  - `{{$json.name.split(' ')[0]}}`
  - `{{$json.name.split(' ').slice(1).join(' ') || $json.name}}`
  - Score mapping logic for Salesforce `rating` and `status`
- **Input and output connections:**
  - Input: Round-Robin Sales Assignment
  - Output: Send to CRM
- **Version-specific requirements:**
  - Type version `1`
  - Requires Salesforce credentials with lead creation permissions
- **Edge cases or potential failure types:**
  - Single-word names produce fallback behavior for lastname
  - `owner` may not match expected Salesforce owner ID format if email is used instead of Salesforce user ID
  - Salesforce field permissions or validation rules may reject values
- **Sub-workflow reference:** none

#### Send to CRM
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends a POST request to an external CRM API such as HubSpot or GoHighLevel.
- **Configuration choices:**
  - URL from workflow config: `crmApiUrl`
  - Method: `POST`
  - JSON body with `properties` object containing:
    - `email`
    - `firstname`
    - `lastname`
    - `phone`
    - `company`
    - `lead_score`
    - `estimated_value`
    - `hubspot_owner_id`
  - Headers:
    - `Content-Type: application/json`
    - `Authorization: <placeholder>`
- **Key expressions or variables used:**
  - `={{ $('Workflow Configuration').first().json.crmApiUrl }}`
  - `{{ $json.name.split(' ')[0] }}`
  - `{{ $json.name.split(' ').slice(1).join(' ') }}`
  - `{{ $json.output.leadScore }}`
  - `{{ $json.output.estimatedValue }}`
  - `{{ $json.assignedSalesRep }}`
- **Input and output connections:**
  - Inputs:
    - Round-Robin Sales Assignment
    - Create Salesforce Lead
  - Output: Send Welcome Email
- **Version-specific requirements:**
  - Type version `4.3`
- **Edge cases or potential failure types:**
  - Placeholder URL/auth header must be replaced
  - `hubspot_owner_id` usually expects a CRM owner ID, not an email address
  - If both inbound connections execute, the node may run twice for the same lead, potentially duplicating CRM records and downstream actions
  - API-specific payload formats may differ from the generic `properties` structure used here
- **Sub-workflow reference:** none

**Important design note:**  
Because `Round-Robin Sales Assignment` connects directly to `Send to CRM` and also to `Create Salesforce Lead`, and `Create Salesforce Lead` then connects to `Send to CRM`, the HTTP CRM node can receive two executions for the same lead. This can cause:
- duplicate CRM sync calls,
- duplicate welcome emails,
- duplicate Slack notifications,
- duplicate activity log entries.

If the intent is to choose one CRM path, an IF or Switch node should be added. If the intent is dual-write, the downstream notification/email path should be restructured to avoid duplication.

---

## 2.7 Lead Communication

**Overview:**  
This block sends an acknowledgement email to the submitted lead once CRM synchronization is complete.

**Nodes Involved:**
- Send Welcome Email

### Node Details

#### Send Welcome Email
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends an HTML email through Gmail.
- **Configuration choices:**
  - Recipient: `{{$json.email}}`
  - Subject: `Welcome! We received your inquiry`
  - HTML body thanks the lead and mentions the assigned sales rep
- **Key expressions or variables used:**
  - `={{ $json.email }}`
  - `{{ $json.name }}`
  - `{{ $json.assignedSalesRep }}`
- **Input and output connections:**
  - Input: Send to CRM
  - Output: Notify Sales Team
- **Version-specific requirements:**
  - Type version `2.2`
  - Requires Gmail credentials/oauth
- **Edge cases or potential failure types:**
  - Gmail OAuth expiration or scope issue
  - Invalid recipient email
  - `assignedSalesRep` is inserted as-is, which may be an email rather than a friendly name
  - If upstream CRM node runs twice, duplicate emails may be sent
- **Sub-workflow reference:** none

---

## 2.8 Team Notification and Audit Logging

**Overview:**  
This block informs the internal sales team through Slack and writes a final processing record to Postgres for traceability.

**Nodes Involved:**
- Notify Sales Team
- Log Activity

### Node Details

#### Notify Sales Team
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends a formatted Slack message to a configured channel.
- **Configuration choices:**
  - Target by channel ID from workflow config
  - Message includes:
    - lead details
    - AI score and value
    - assignment
    - recommended action
    - company insights
- **Key expressions or variables used:**
  - `={{ $('Workflow Configuration').first().json.slackChannel }}`
  - `{{ $json.output.leadScore.toUpperCase() }}`
  - `{{ $json.name }}`
  - `{{ $json.email }}`
  - `{{ $json.company }}`
  - `{{ $json.phone }}`
  - `{{ $json.output.estimatedValue }}`
  - `{{ $json.output.scoreReason }}`
  - `{{ $json.assignedSalesRep }}`
  - `{{ $json.output.recommendedAction }}`
  - `{{ $json.output.companyInsights }}`
- **Input and output connections:**
  - Input: Send Welcome Email
  - Output: Log Activity
- **Version-specific requirements:**
  - Type version `2.4`
  - Requires Slack credentials and access to the selected channel
- **Edge cases or potential failure types:**
  - Invalid Slack channel ID
  - Missing permissions to post
  - `companyInsights` may be undefined if AI omitted optional field
  - Duplicate message risk if upstream path duplicates executions
- **Sub-workflow reference:** none

#### Log Activity
- **Type and technical role:** `n8n-nodes-base.postgres`  
  Writes an audit record to the `public.activity_log` table.
- **Configuration choices:**
  - Schema: `public`
  - Table: `activity_log`
  - Fields inserted:
    - `timestamp`
    - `crm_synced = true`
    - `email_sent = true`
    - `lead_email`
    - `lead_score`
    - `assigned_rep`
    - `activity_type = 'lead_processed'`
    - `slack_notified = true`
- **Key expressions or variables used:**
  - `={{ $now.toISO() }}`
  - `={{ $json.email }}`
  - `={{ $json.output.leadScore }}`
  - `={{ $json.assignedSalesRep }}`
- **Input and output connections:**
  - Input: Notify Sales Team
  - Output: none
- **Version-specific requirements:**
  - Type version `2.6`
  - Requires Postgres credentials and matching table schema
- **Edge cases or potential failure types:**
  - DB insert failure
  - Booleans or timestamp types may need schema compatibility
  - Duplicate records possible if upstream duplicated execution
- **Sub-workflow reference:** none

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Lead Form Webhook | Webhook | Receives incoming lead form submissions via HTTP POST |  | Workflow Configuration | ## Input Layer<br>Capture leads from webhook form submission |
| Workflow Configuration | Set | Stores reusable runtime configuration values | Lead Form Webhook | Validate Required Fields | ## Input Layer<br>Capture leads from webhook form submission |
| Validate Required Fields | If | Validates presence of required lead fields | Workflow Configuration | Normalize Lead Data; Handle Validation Error | ## Validation<br>Ensure required fields like name and email |
| Normalize Lead Data | Set | Standardizes incoming lead fields for downstream use | Validate Required Fields | AI Lead Enrichment & Scoring | ## Data Preparation<br>Normalize and structure lead information |
| AI Lead Enrichment & Scoring | LangChain Agent | Scores and enriches the lead using AI | Normalize Lead Data; OpenAI Chat Model; Structured Output Parser | Store Lead in Database | ## AI Enrichment<br>Score lead and generate insights |
| OpenAI Chat Model | OpenAI Chat Model | Provides the language model for enrichment |  | AI Lead Enrichment & Scoring | ## AI Enrichment<br>Score lead and generate insights |
| Structured Output Parser | Structured Output Parser | Enforces structured AI output schema |  | AI Lead Enrichment & Scoring | ## AI Enrichment<br>Score lead and generate insights |
| Store Lead in Database | Postgres | Persists enriched lead data in Postgres | AI Lead Enrichment & Scoring | Round-Robin Sales Assignment | ## Data Storage and  Assignment Logic<br>Distribute leads via round-robin system<br>Store enriched lead in database |
| Round-Robin Sales Assignment | Code | Assigns lead owner using rotating round-robin logic | Store Lead in Database | Send to CRM; Create Salesforce Lead | ## Data Storage and  Assignment Logic<br>Distribute leads via round-robin system<br>Store enriched lead in database |
| Create Salesforce Lead | Salesforce | Creates a lead record in Salesforce | Round-Robin Sales Assignment | Send to CRM | ## CRM Sync<br>Push lead data to CRM or Salesforce |
| Send to CRM | HTTP Request | Sends lead data to a generic CRM API endpoint | Round-Robin Sales Assignment; Create Salesforce Lead | Send Welcome Email | ## CRM Sync<br>Push lead data to CRM or Salesforce |
| Send Welcome Email | Gmail | Sends acknowledgment email to the lead | Send to CRM | Notify Sales Team | ## Email Automation<br>Send welcome email to the lead |
| Notify Sales Team | Slack | Posts lead summary and AI insights to Slack | Send Welcome Email | Log Activity | ## Team Notification and Logging<br>Notify sales team via Slack  and <br>Track all actions for audit and analytics |
| Log Activity | Postgres | Logs processing status for analytics and audit | Notify Sales Team |  | ## Team Notification and Logging<br>Notify sales team via Slack  and <br>Track all actions for audit and analytics |
| Handle Validation Error | Set | Returns a validation failure payload | Validate Required Fields |  | ## send error report |
| Sticky Note | Sticky Note | Documentation canvas note |  |  |  |
| Sticky Note1 | Sticky Note | Documentation canvas note |  |  |  |
| Sticky Note3 | Sticky Note | Documentation canvas note |  |  |  |
| Sticky Note4 | Sticky Note | Documentation canvas note |  |  |  |
| Sticky Note5 | Sticky Note | Documentation canvas note |  |  |  |
| Sticky Note6 | Sticky Note | Documentation canvas note |  |  |  |
| Sticky Note7 | Sticky Note | Documentation canvas note |  |  |  |
| Sticky Note8 | Sticky Note | Documentation canvas note |  |  |  |
| Sticky Note10 | Sticky Note | Documentation canvas note |  |  |  |
| Sticky Note11 | Sticky Note | Documentation canvas note |  |  |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add a Webhook node** named **Lead Form Webhook**.
   - Type: `Webhook`
   - HTTP Method: `POST`
   - Path: `lead-capture`
   - Response Mode: `Last Node`
   - Expect incoming JSON in the request body, for example:
     - `name`
     - `email`
     - `phone`
     - `company`
     - `message`

3. **Add a Set node** named **Workflow Configuration** after the webhook.
   - Enable keeping input fields.
   - Add these fields:
     - `crmApiUrl` as string
     - `slackChannel` as string
     - `salesReps` as array, for example:
       - `["rep1@example.com", "rep2@example.com", "rep3@example.com"]`
   - Replace placeholders with real values before use.

4. **Add an If node** named **Validate Required Fields**.
   - Condition mode: `AND`
   - Condition 1: `{{$json.body.name}}` is not empty
   - Condition 2: `{{$json.body.email}}` is not empty

5. **Connect Workflow Configuration → Validate Required Fields**.

6. **Add a Set node** named **Normalize Lead Data** on the true branch.
   - Keep other fields enabled.
   - Map:
     - `name = {{$json.body.name}}`
     - `email = {{$json.body.email}}`
     - `phone = {{$json.body.phone || ''}}`
     - `company = {{$json.body.company || ''}}`
     - `message = {{$json.body.message || ''}}`
     - `timestamp = {{$now.toISO()}}`

7. **Add a Set node** named **Handle Validation Error** on the false branch.
   - Set:
     - `error = Missing required fields: name or email`
     - `status = validation_failed`
     - `timestamp = {{$now.toISO()}}`

8. **Add an AI Agent node** named **AI Lead Enrichment & Scoring**.
   - Type: LangChain Agent
   - Prompt type: define manually
   - Prompt text:
     - Include name, email, phone, company, message from normalized fields
   - System message should instruct the model to:
     - classify lead as hot/warm/cold
     - explain why
     - estimate deal value
     - provide company insights
     - recommend next action
     - return structured output matching the parser schema

9. **Add an OpenAI Chat Model node** named **OpenAI Chat Model**.
   - Select model: `gpt-4o-mini`
   - Configure OpenAI credentials

10. **Add a Structured Output Parser node** named **Structured Output Parser**.
    - Use manual JSON schema
    - Define fields:
      - `leadScore`: string enum `hot`, `warm`, `cold`
      - `scoreReason`: string
      - `estimatedValue`: number
      - `companyInsights`: string
      - `recommendedAction`: string
    - Required:
      - `leadScore`
      - `scoreReason`
      - `estimatedValue`
      - `recommendedAction`

11. **Connect the AI nodes**:
    - Normalize Lead Data → AI Lead Enrichment & Scoring
    - OpenAI Chat Model → AI Lead Enrichment & Scoring using the language model connector
    - Structured Output Parser → AI Lead Enrichment & Scoring using the output parser connector

12. **Add a Postgres node** named **Store Lead in Database**.
    - Operation: insert row
    - Schema: `public`
    - Table: `leads`
    - Configure Postgres credentials
    - Map fields:
      - `name`
      - `email`
      - `phone`
      - `company`
      - `message`
      - `created_at = {{$json.timestamp}}`
      - `lead_score = {{$json.output.leadScore}}`
      - `score_reason = {{$json.output.scoreReason}}`
      - `estimated_value = {{$json.output.estimatedValue}}`
      - `company_insights = {{$json.output.companyInsights}}`
      - `recommended_action = {{$json.output.recommendedAction}}`

13. **Create the `leads` table** in Postgres if it does not exist.
   Suggested columns:
   - `id` serial or UUID
   - `name` text
   - `email` text
   - `phone` text
   - `company` text
   - `message` text
   - `created_at` timestamp or timestamptz
   - `lead_score` text
   - `score_reason` text
   - `estimated_value` numeric
   - `company_insights` text
   - `recommended_action` text

14. **Add a Code node** named **Round-Robin Sales Assignment** after the lead storage node.
   - Paste this logic conceptually:
     - read `salesReps` from Workflow Configuration
     - read current index from execution custom data
     - choose rep by modulo
     - increment stored index
     - return lead data plus `assignedSalesRep`
   - Ensure your n8n environment supports `$execution.customData`

15. **Connect Store Lead in Database → Round-Robin Sales Assignment**.

16. **Add a Salesforce node** named **Create Salesforce Lead**.
   - Configure Salesforce credentials
   - Operation: create lead
   - Map fields:
     - Company: `{{$json.company || 'Unknown'}}`
     - First name: `{{$json.name.split(' ')[0]}}`
     - Last name: `{{$json.name.split(' ').slice(1).join(' ') || $json.name}}`
     - Email: `{{$json.email}}`
     - Phone: `{{$json.phone}}`
     - Owner: `{{$json.assignedSalesRep}}`
     - Lead source: `Web Form`
     - Description: `{{$json.message}}`
     - Rating:
       - Hot → `Hot`
       - Warm → `Warm`
       - Cold → `Cold`
     - Status:
       - Hot → `Working - Contacted`
       - Warm → `Open - Not Contacted`
       - Cold → `Unqualified`

17. **Add an HTTP Request node** named **Send to CRM**.
   - Method: `POST`
   - URL: `{{$('Workflow Configuration').first().json.crmApiUrl}}`
   - Send JSON body
   - Add headers:
     - `Content-Type: application/json`
     - `Authorization: Bearer ...` or CRM API key format required by your platform
   - Body structure:
     - `properties.email = {{$json.email}}`
     - `properties.firstname = {{$json.name.split(' ')[0]}}`
     - `properties.lastname = {{$json.name.split(' ').slice(1).join(' ')}}`
     - `properties.phone = {{$json.phone}}`
     - `properties.company = {{$json.company}}`
     - `properties.lead_score = {{$json.output.leadScore}}`
     - `properties.estimated_value = {{$json.output.estimatedValue}}`
     - `properties.hubspot_owner_id = {{$json.assignedSalesRep}}`

18. **Connect the CRM nodes exactly as in the source workflow**:
   - Round-Robin Sales Assignment → Send to CRM
   - Round-Robin Sales Assignment → Create Salesforce Lead
   - Create Salesforce Lead → Send to CRM

19. **Important implementation warning:**  
   This connection pattern can cause `Send to CRM` to execute twice. If you want only one CRM path:
   - insert an IF or Switch node to choose Salesforce vs HTTP CRM, or
   - merge branches carefully before continuing to email/Slack steps.

20. **Add a Gmail node** named **Send Welcome Email**.
   - Configure Gmail OAuth2 credentials
   - To: `{{$json.email}}`
   - Subject: `Welcome! We received your inquiry`
   - HTML body should thank the lead and mention `{{$json.assignedSalesRep}}`

21. **Connect Send to CRM → Send Welcome Email**.

22. **Add a Slack node** named **Notify Sales Team**.
   - Configure Slack credentials
   - Action: send message to channel
   - Channel ID: `{{$('Workflow Configuration').first().json.slackChannel}}`
   - Compose a message including:
     - lead score
     - contact details
     - estimated value
     - score reason
     - assigned rep
     - recommended action
     - company insights

23. **Connect Send Welcome Email → Notify Sales Team**.

24. **Add another Postgres node** named **Log Activity**.
   - Operation: insert row
   - Schema: `public`
   - Table: `activity_log`
   - Map:
     - `timestamp = {{$now.toISO()}}`
     - `crm_synced = true`
     - `email_sent = true`
     - `lead_email = {{$json.email}}`
     - `lead_score = {{$json.output.leadScore}}`
     - `assigned_rep = {{$json.assignedSalesRep}}`
     - `activity_type = lead_processed`
     - `slack_notified = true`

25. **Create the `activity_log` table** if needed.
   Suggested columns:
   - `id` serial or UUID
   - `timestamp` timestamp or timestamptz
   - `crm_synced` boolean
   - `email_sent` boolean
   - `lead_email` text
   - `lead_score` text
   - `assigned_rep` text
   - `activity_type` text
   - `slack_notified` boolean

26. **Connect Notify Sales Team → Log Activity**.

27. **Set up credentials** for all external services:
   - OpenAI credentials for `OpenAI Chat Model`
   - Postgres credentials for both database nodes
   - Salesforce credentials for `Create Salesforce Lead`
   - Gmail OAuth2 credentials for `Send Welcome Email`
   - Slack credentials for `Notify Sales Team`
   - If using HTTP CRM sync, ensure the authorization header matches the target CRM API format

28. **Replace all placeholders**:
   - CRM API URL
   - CRM authorization token
   - Slack channel ID
   - Sales rep list

29. **Test the webhook** with a valid payload, for example:
   - `name`
   - `email`
   - `phone`
   - `company`
   - `message`

30. **Test invalid payload behavior** by omitting `name` or `email` and confirm the workflow returns the validation error object.

31. **Review duplication risk before production**.
   - If both Salesforce and HTTP CRM branches are intended, add deduplication or branch-specific notifications.
   - If only one CRM should be used, redesign the branch before enabling the workflow.

**Sub-workflow setup:**  
This workflow does not contain any Execute Workflow / sub-workflow nodes. No separate child workflow needs to be created.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow captures incoming leads via a webhook and processes them through validation, enrichment, and automated routing. After validating required fields, the lead data is normalized and analyzed by an AI agent to classify it as hot, warm, or cold. The workflow enriches the lead with insights, estimated deal value, and recommended actions. Leads are stored in a database, assigned to sales reps using round-robin logic, and synced with CRM systems. The workflow sends a welcome email to the lead and notifies the sales team via Slack. All activities are logged for tracking and analytics. | General workflow note |
| Configure webhook endpoint for lead capture | Setup guidance |
| Add OpenAI API credentials | Setup guidance |
| Set CRM API or Salesforce credentials | Setup guidance |
| Configure Slack and Gmail integrations | Setup guidance |
| Add Postgres database connection | Setup guidance |
| Define sales reps for assignment logic | Setup guidance |