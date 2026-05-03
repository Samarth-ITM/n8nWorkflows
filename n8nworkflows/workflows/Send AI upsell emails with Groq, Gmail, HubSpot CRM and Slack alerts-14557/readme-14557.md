Send AI upsell emails with Groq, Gmail, HubSpot CRM and Slack alerts

https://n8nworkflows.xyz/workflows/send-ai-upsell-emails-with-groq--gmail--hubspot-crm-and-slack-alerts-14557


# Send AI upsell emails with Groq, Gmail, HubSpot CRM and Slack alerts

# 1. Workflow Overview

This workflow detects when a user has exceeded their plan usage, generates an AI-based upsell email, sends that email through Gmail, logs the activity in HubSpot, and notifies the internal team in Slack.

Its main use case is automated revenue expansion: when a customer crosses a usage threshold, the workflow reacts immediately to encourage an upgrade and creates internal visibility for follow-up.

## 1.1 Trigger & Data Preparation

The workflow starts from a webhook that receives usage data from an external system. A Set node then normalizes the incoming payload into a clean structure used by downstream nodes.

## 1.2 AI Email Generation

An AI Agent uses a Groq chat model to generate the body of a personalized upsell email. The generated text is then embedded into an HTML email template and sent via Gmail.

## 1.3 CRM Update

After the email is sent, the workflow searches HubSpot contacts by email. If a matching contact is found, it creates a HubSpot note associated with that contact to record the upsell opportunity.

## 1.4 Team Notification

Finally, the workflow posts a Slack message summarizing the user, plan, usage status, and the fact that an upsell email was sent, enabling manual follow-up by the team.

---

# 2. Block-by-Block Analysis

## Block 1 — Trigger & Data Preparation

### Overview

This block receives the event payload and extracts the fields needed throughout the workflow. It acts as the input normalization layer so the rest of the workflow can use predictable variable names.

### Nodes Involved

- Webhook
- Edit Fields

### Node Details

#### Webhook

- **Type and technical role:** `n8n-nodes-base.webhook`  
  Entry-point node that receives HTTP requests from an external application or service.

- **Configuration choices:**
  - Webhook path is set to `YOUR_N8N_WEBHOOK_ID`
  - No additional options are configured
  - Uses webhook node version `2.1`

- **Key expressions or variables used:**
  - Exposes incoming request body through `$json.body`

- **Input and output connections:**
  - Input: none, this is a trigger
  - Output: `Edit Fields`

- **Version-specific requirements:**
  - Uses webhook node `typeVersion: 2.1`
  - Exact webhook URL depends on the n8n instance base URL and whether test or production mode is used

- **Edge cases or potential failure types:**
  - Invalid request payload shape
  - Missing `body` fields
  - Caller using wrong HTTP method or wrong webhook URL
  - Authentication/security not configured at the webhook level
  - No built-in validation that usage actually exceeds the limit

- **Sub-workflow reference:** none

---

#### Edit Fields

- **Type and technical role:** `n8n-nodes-base.set`  
  Maps fields from the incoming webhook payload into a flatter, cleaner object.

- **Configuration choices:**
  - Creates the following fields:
    - `user_id`
    - `name`
    - `email`
    - `plan`
    - `usage_limit`
    - `usage`
  - String and number types are explicitly assigned

- **Key expressions or variables used:**
  - `{{ $json.body.user_id }}`
  - `{{ $json.body.name }}`
  - `{{ $json.body.email }}`
  - `{{ $json.body.plan }}`
  - `{{ $json.body.usage_limit }}`
  - `{{ $json.body.usage }}`

- **Input and output connections:**
  - Input: `Webhook`
  - Output: `AI Agent`

- **Version-specific requirements:**
  - Uses Set node `typeVersion: 3.4`

- **Edge cases or potential failure types:**
  - Missing fields in the webhook body produce empty or invalid values
  - Numeric conversion issues if `usage` or `usage_limit` arrive as non-numeric strings
  - No validation or conditional branch if values are null or malformed

- **Sub-workflow reference:** none

---

## Block 2 — AI Email Generation

### Overview

This block generates personalized upsell text with Groq and sends it to the customer by email. The AI Agent is connected to a Groq language model, and the resulting output is inserted into an HTML email body.

### Nodes Involved

- Groq Chat Model
- AI Agent
- Send a message5

### Node Details

#### Groq Chat Model

- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatGroq`  
  Provides the LLM used by the AI Agent.

- **Configuration choices:**
  - Model: `llama-3.3-70b-versatile`
  - No extra options configured

- **Key expressions or variables used:**
  - No direct expressions in this node
  - Supplies the language model to the AI Agent through the AI connection

- **Input and output connections:**
  - Input: none via main connection
  - Output: connected to `AI Agent` through `ai_languageModel`

- **Version-specific requirements:**
  - Uses node `typeVersion: 1`
  - Requires Groq credentials configured in n8n
  - LangChain-compatible AI nodes must be available in the n8n instance

- **Edge cases or potential failure types:**
  - Invalid or missing Groq API credentials
  - Rate limiting or model availability issues
  - Timeout on model generation
  - Token/context limitations depending on prompt design

- **Sub-workflow reference:** none

---

#### AI Agent

- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  Orchestrates AI text generation using the attached language model.

- **Configuration choices:**
  - No visible prompt or system instructions are configured in the provided JSON
  - No tools or advanced options are configured
  - This means the node is structurally incomplete for meaningful output unless prompt content exists elsewhere in the UI or is added later

- **Key expressions or variables used:**
  - No explicit expressions configured in the exported parameters
  - Downstream Gmail node expects this node to output a field named `output`

- **Input and output connections:**
  - Input: `Edit Fields`
  - AI model input: `Groq Chat Model`
  - Output: `Send a message5`

- **Version-specific requirements:**
  - Uses node `typeVersion: 3`
  - Requires compatibility between LangChain agent node and the selected model node

- **Edge cases or potential failure types:**
  - Missing instructions/prompt may cause empty, generic, or invalid output
  - If the node does not emit `output`, the Gmail node’s HTML expression will fail
  - LLM hallucination or poor personalization if no structured prompt is defined
  - API quota or provider errors

- **Sub-workflow reference:** none

---

#### Send a message5

- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends the AI-generated upsell email using Gmail.

- **Configuration choices:**
  - Recipient: `{{ $('Edit Fields').item.json.email }}`
  - Subject: `You’ve exceeded your plan limit — upgrade to stay uninterrupted`
  - Message body is HTML
  - The HTML template wraps the AI output in a styled email layout
  - Footer uses `© 2026 Your Company`

- **Key expressions or variables used:**
  - Recipient: `{{ $('Edit Fields').item.json.email }}`
  - Email body: `{{ $json["output"].replace(/\n/g, "<br>") }}`
  - The expression assumes the current item contains an `output` property from `AI Agent`

- **Input and output connections:**
  - Input: `AI Agent`
  - Output: `Search contacts`

- **Version-specific requirements:**
  - Uses Gmail node `typeVersion: 2.2`
  - Requires Gmail OAuth2 credentials
  - A valid authenticated Gmail account must be connected

- **Edge cases or potential failure types:**
  - Missing or invalid email address
  - Gmail OAuth authentication failure
  - Sending limits or anti-abuse restrictions from Google
  - HTML render issues if AI output is empty or malformed
  - Expression failure if `$json.output` is undefined

- **Sub-workflow reference:** none

---

## Block 3 — CRM Update

### Overview

This block locates the recipient in HubSpot and records that an upsell action occurred. It uses the contact email as the lookup key, then creates a CRM note associated with the found contact.

### Nodes Involved

- Search contacts
- HTTP Request

### Node Details

#### Search contacts

- **Type and technical role:** `n8n-nodes-base.hubspot`  
  Searches HubSpot CRM contacts for a record matching the email from the payload.

- **Configuration choices:**
  - Operation: `search`
  - Authentication: app token
  - Filter:
    - Property: `email|string`
    - Value: `{{ $('Edit Fields').item.json.email }}`

- **Key expressions or variables used:**
  - `{{ $('Edit Fields').item.json.email }}`

- **Input and output connections:**
  - Input: `Send a message5`
  - Output: `HTTP Request`

- **Version-specific requirements:**
  - Uses HubSpot node `typeVersion: 2.2`
  - Requires HubSpot app token credentials configured in n8n

- **Edge cases or potential failure types:**
  - No matching contact found
  - Multiple matching records depending on CRM data quality
  - HubSpot authentication/token issues
  - Search property syntax may vary by node version; `email|string` should be validated in the UI
  - If no item is returned, downstream note creation will fail

- **Sub-workflow reference:** none

---

#### HTTP Request

- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls the HubSpot CRM API directly to create a note and associate it with the contact found in the previous step.

- **Configuration choices:**
  - Method: `POST`
  - URL: `https://api.hubapi.com/crm/v3/objects/notes`
  - Sends JSON body
  - Custom headers:
    - `Authorization: YOUR_HUBSPOT_ACCESS_TOKEN`
    - `Content-Type: application/json`
  - Note body:
    - Timestamp hardcoded as `2026-11-12T15:48:22Z`
    - Text: `Upsell Opportunity. User have exceeded the limit.`
  - Association target:
    - Contact ID from `{{ $json.id }}`
    - Association type ID `202`

- **Key expressions or variables used:**
  - `{{ $json.id }}` for the HubSpot contact ID

- **Input and output connections:**
  - Input: `Search contacts`
  - Output: `Send a message6`

- **Version-specific requirements:**
  - Uses HTTP Request node `typeVersion: 4.3`
  - Requires a valid HubSpot private app token or bearer token format
  - Association type ID `202` assumes a standard HubSpot-defined note-to-contact association

- **Edge cases or potential failure types:**
  - Hardcoded authorization header is not in standard `Bearer <token>` format as shown; must be corrected
  - Hardcoded timestamp may be undesirable and misleading
  - If `$json.id` is missing, note association will fail
  - HubSpot API may reject malformed body or unauthorized request
  - Token expiration or insufficient scopes
  - If search returns zero items, this node may not run or may fail depending on item flow

- **Sub-workflow reference:** none

---

## Block 4 — Team Notification

### Overview

This block sends a Slack alert summarizing the upsell event. It provides internal visibility so the team can follow up if the user does not upgrade soon after receiving the email.

### Nodes Involved

- Send a message6

### Node Details

#### Send a message6

- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends a message to a Slack channel using OAuth2 authentication.

- **Configuration choices:**
  - Sends a formatted text message
  - Message includes:
    - User name
    - Email
    - Usage and limit
    - Current plan
    - Action taken
    - Next step follow-up guidance
  - Slack destination mode is channel selection
  - `channelId` is currently empty in the exported workflow

- **Key expressions or variables used:**
  - `{{ $('Webhook').item.json.body.name }}`
  - `{{ $('Webhook').item.json.body.email }}`
  - `{{ $('Webhook').item.json.body.usage }}`
  - `{{ $('Webhook').item.json.body.usage_limit }}`
  - `{{ $('Webhook').item.json.body.plan }}`

- **Input and output connections:**
  - Input: `HTTP Request`
  - Output: none

- **Version-specific requirements:**
  - Uses Slack node `typeVersion: 2.4`
  - Requires Slack OAuth2 credentials
  - A valid target channel must be selected

- **Edge cases or potential failure types:**
  - Empty `channelId` means the node is not fully configured and will fail until a channel is selected
  - Slack OAuth permission scopes may be insufficient
  - Expressions depend on original webhook payload existing in expected form
  - Notification is sent only after successful HubSpot note creation because of node order

- **Sub-workflow reference:** none

---

## Additional Non-Executable Nodes

These nodes are documentation/comment nodes and do not participate in execution.

### Sticky Note2
Content:
- `## Step 2: AI Email Generation`
- `Generates a personalized upsell email and sends it via Gmail.`

### Sticky Note3
Content:
- `## Step 3: CRM Update`
- `Finds the contact in HubSpot and logs the upsell activity.`

### Sticky Note4
Content:
- `## Step 4: Team Notification`
- `Sends a Slack alert to notify the team for follow-up.`

### Sticky Note5
Content:
- `# Automated Upsell Trigger Based on Usage Data`
- Describes the workflow purpose, how it works, setup steps, and customization tips

### Sticky Note6
Content:
- `## Step 1: Trigger & Data Prep`
- `Receives webhook data and formats key fields for processing.`

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook | Webhook | Receives usage event payload from an external system |  | Edit Fields | # Automated Upsell Trigger Based on Usage Data<br>This workflow automatically detects when a customer exceeds their usage limits and triggers a personalized upsell process. It helps teams act at the right moment, improving upgrade conversions and customer experience without manual monitoring.<br>### How it works<br>- A webhook receives real-time usage data<br>- Data is structured and prepared for processing<br>- AI generates a personalized upsell message<br>- Email is sent automatically to the customer<br>- Activity is logged inside HubSpot CRM<br>- The team is notified via Slack for follow-up<br>### Setup steps<br>- Configure the webhook to receive usage data<br>- Ensure payload includes email, usage, and limit fields<br>- Connect your AI model (Groq/OpenAI)<br>- Authenticate Gmail, HubSpot, and Slack nodes<br>- Add your HubSpot API token and Slack credentials<br>- Test the workflow using sample webhook data<br>### Customization tips<br>- Adjust usage thresholds based on your pricing model<br>- Modify AI prompts for different tones or offers<br>- Add segmentation logic for different customer tiers<br>## Step 1: Trigger & Data Prep<br>Receives webhook data and formats key fields for processing. |
| Edit Fields | Set | Extracts and normalizes webhook fields | Webhook | AI Agent | # Automated Upsell Trigger Based on Usage Data<br>This workflow automatically detects when a customer exceeds their usage limits and triggers a personalized upsell process. It helps teams act at the right moment, improving upgrade conversions and customer experience without manual monitoring.<br>### How it works<br>- A webhook receives real-time usage data<br>- Data is structured and prepared for processing<br>- AI generates a personalized upsell message<br>- Email is sent automatically to the customer<br>- Activity is logged inside HubSpot CRM<br>- The team is notified via Slack for follow-up<br>### Setup steps<br>- Configure the webhook to receive usage data<br>- Ensure payload includes email, usage, and limit fields<br>- Connect your AI model (Groq/OpenAI)<br>- Authenticate Gmail, HubSpot, and Slack nodes<br>- Add your HubSpot API token and Slack credentials<br>- Test the workflow using sample webhook data<br>### Customization tips<br>- Adjust usage thresholds based on your pricing model<br>- Modify AI prompts for different tones or offers<br>- Add segmentation logic for different customer tiers<br>## Step 1: Trigger & Data Prep<br>Receives webhook data and formats key fields for processing. |
| Groq Chat Model | Groq Chat Model | Provides the LLM used by the AI Agent |  | AI Agent | ## Step 2: AI Email Generation<br>Generates a personalized upsell email and sends it via Gmail. |
| AI Agent | AI Agent | Generates upsell email content from prepared customer data | Edit Fields | Send a message5 | ## Step 2: AI Email Generation<br>Generates a personalized upsell email and sends it via Gmail. |
| Send a message5 | Gmail | Sends the upsell email to the customer | AI Agent | Search contacts | ## Step 2: AI Email Generation<br>Generates a personalized upsell email and sends it via Gmail. |
| Search contacts | HubSpot | Searches HubSpot contacts by email | Send a message5 | HTTP Request | ## Step 3: CRM Update<br>Finds the contact in HubSpot and logs the upsell activity. |
| HTTP Request | HTTP Request | Creates and associates a HubSpot note for the upsell event | Search contacts | Send a message6 | ## Step 3: CRM Update<br>Finds the contact in HubSpot and logs the upsell activity. |
| Send a message6 | Slack | Notifies the team in Slack for follow-up | HTTP Request |  | ## Step 4: Team Notification<br>Sends a Slack alert to notify the team for follow-up. |
| Sticky Note2 | Sticky Note | Documentation/comment node |  |  | ## Step 2: AI Email Generation<br>Generates a personalized upsell email and sends it via Gmail. |
| Sticky Note3 | Sticky Note | Documentation/comment node |  |  | ## Step 3: CRM Update<br>Finds the contact in HubSpot and logs the upsell activity. |
| Sticky Note4 | Sticky Note | Documentation/comment node |  |  | ## Step 4: Team Notification<br>Sends a Slack alert to notify the team for follow-up. |
| Sticky Note5 | Sticky Note | Documentation/comment node |  |  | # Automated Upsell Trigger Based on Usage Data<br>This workflow automatically detects when a customer exceeds their usage limits and triggers a personalized upsell process. It helps teams act at the right moment, improving upgrade conversions and customer experience without manual monitoring.<br>### How it works<br>- A webhook receives real-time usage data<br>- Data is structured and prepared for processing<br>- AI generates a personalized upsell message<br>- Email is sent automatically to the customer<br>- Activity is logged inside HubSpot CRM<br>- The team is notified via Slack for follow-up<br>### Setup steps<br>- Configure the webhook to receive usage data<br>- Ensure payload includes email, usage, and limit fields<br>- Connect your AI model (Groq/OpenAI)<br>- Authenticate Gmail, HubSpot, and Slack nodes<br>- Add your HubSpot API token and Slack credentials<br>- Test the workflow using sample webhook data<br>### Customization tips<br>- Adjust usage thresholds based on your pricing model<br>- Modify AI prompts for different tones or offers<br>- Add segmentation logic for different customer tiers |
| Sticky Note6 | Sticky Note | Documentation/comment node |  |  | ## Step 1: Trigger & Data Prep<br>Receives webhook data and formats key fields for processing. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a Webhook node**
   - Add a **Webhook** node.
   - Set the path to a unique endpoint such as `YOUR_N8N_WEBHOOK_ID`.
   - Keep default options unless you need authentication or a specific HTTP method.
   - This node should accept a JSON body containing at least:
     - `user_id`
     - `name`
     - `email`
     - `plan`
     - `usage_limit`
     - `usage`

2. **Create a Set node named “Edit Fields”**
   - Add a **Set** node after the Webhook.
   - Create these fields:
     - `user_id` → string → `{{ $json.body.user_id }}`
     - `name` → string → `{{ $json.body.name }}`
     - `email` → string → `{{ $json.body.email }}`
     - `plan` → string → `{{ $json.body.plan }}`
     - `usage_limit` → number → `{{ $json.body.usage_limit }}`
     - `usage` → number → `{{ $json.body.usage }}`
   - Connect **Webhook → Edit Fields**.

3. **Create a Groq Chat Model node**
   - Add a **Groq Chat Model** node.
   - Select model `llama-3.3-70b-versatile`.
   - Configure Groq credentials with your API key.
   - This node will not use a main connection; it will be attached to the AI Agent as a language model.

4. **Create an AI Agent node**
   - Add an **AI Agent** node.
   - Connect **Edit Fields → AI Agent**.
   - Connect **Groq Chat Model → AI Agent** using the language model connector.
   - The provided workflow export does not include a prompt, so you should add one manually in the AI Agent configuration.
   - Recommended prompt content:
     - Explain that the user exceeded their usage limit
     - Mention current plan and usage
     - Encourage upgrading
     - Keep tone professional and concise
     - Output plain text suitable for email
   - Make sure the node returns a text field compatible with downstream usage, ideally `output`.

5. **Create a Gmail node named “Send a message5”**
   - Add a **Gmail** node after the AI Agent.
   - Configure Gmail OAuth2 credentials.
   - Set recipient to:
     - `{{ $('Edit Fields').item.json.email }}`
   - Set subject to:
     - `You’ve exceeded your plan limit — upgrade to stay uninterrupted`
   - Set message body to HTML.
   - Paste an HTML template similar to the exported one, inserting AI output with:
     - `{{ $json["output"].replace(/\n/g, "<br>") }}`
   - Connect **AI Agent → Send a message5**.
   - If your AI Agent returns a different property name, update the expression accordingly.

6. **Create a HubSpot node named “Search contacts”**
   - Add a **HubSpot** node after the Gmail node.
   - Configure HubSpot app token credentials.
   - Set operation to **Search**.
   - Search contact records using email.
   - Use the value:
     - `{{ $('Edit Fields').item.json.email }}`
   - Connect **Send a message5 → Search contacts**.

7. **Create an HTTP Request node for HubSpot notes**
   - Add an **HTTP Request** node after the HubSpot search.
   - Set:
     - Method: `POST`
     - URL: `https://api.hubapi.com/crm/v3/objects/notes`
   - Enable JSON body.
   - Add headers:
     - `Authorization: Bearer YOUR_HUBSPOT_ACCESS_TOKEN`
     - `Content-Type: application/json`
   - Build the JSON body so it:
     - Creates a note
     - Includes a timestamp
     - Includes note text about the upsell opportunity
     - Associates the note with the contact ID from the search node
   - Use the contact ID from the current item:
     - `{{ $json.id }}`
   - Prefer using a dynamic timestamp rather than the hardcoded 2026 date from the export.
   - Connect **Search contacts → HTTP Request**.

8. **Create a Slack node named “Send a message6”**
   - Add a **Slack** node after the HTTP Request node.
   - Configure Slack OAuth2 credentials.
   - Select a destination channel.
   - Set the message text to include:
     - User name
     - Email
     - Usage vs. limit
     - Plan
     - Confirmation that upsell email was sent
     - Suggested follow-up within 24 hours
   - Example expressions:
     - `{{ $('Webhook').item.json.body.name }}`
     - `{{ $('Webhook').item.json.body.email }}`
     - `{{ $('Webhook').item.json.body.usage }}`
     - `{{ $('Webhook').item.json.body.usage_limit }}`
     - `{{ $('Webhook').item.json.body.plan }}`
   - Connect **HTTP Request → Send a message6**.

9. **Add optional sticky notes for documentation**
   - Add sticky notes to label the blocks:
     - Step 1: Trigger & Data Prep
     - Step 2: AI Email Generation
     - Step 3: CRM Update
     - Step 4: Team Notification
   - Add a general note describing purpose, setup steps, and customization ideas.

10. **Test the workflow with sample payload**
    - Send a POST request to the webhook with sample JSON such as:
      - `user_id`
      - `name`
      - `email`
      - `plan`
      - `usage_limit`
      - `usage`
    - Confirm:
      - Webhook receives data
      - Set node maps fields correctly
      - AI Agent generates usable output
      - Gmail sends successfully
      - HubSpot search finds a contact
      - Note is created successfully
      - Slack message posts to the selected channel

11. **Recommended improvements before production**
    - Add an **IF** node after `Edit Fields` to verify `usage > usage_limit`
    - Add error branches or an **Error Trigger** workflow
    - Replace hardcoded HubSpot token in HTTP headers with credentials or secure expressions
    - Replace fixed note timestamp with the current execution time
    - Add fallback behavior if no HubSpot contact is found
    - Configure the AI Agent with an explicit prompt and guardrails
    - Complete the Slack channel selection, which is currently blank in the export

### Sub-workflow setup

This workflow does **not** invoke any sub-workflows and does not contain any Execute Workflow node. There are no sub-workflow parameters to configure.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Automated Upsell Trigger Based on Usage Data | General workflow purpose |
| A webhook receives real-time usage data | Workflow behavior |
| Data is structured and prepared for processing | Workflow behavior |
| AI generates a personalized upsell message | Workflow behavior |
| Email is sent automatically to the customer | Workflow behavior |
| Activity is logged inside HubSpot CRM | Workflow behavior |
| The team is notified via Slack for follow-up | Workflow behavior |
| Configure the webhook to receive usage data | Setup guidance |
| Ensure payload includes email, usage, and limit fields | Setup guidance |
| Connect your AI model (Groq/OpenAI) | Setup guidance |
| Authenticate Gmail, HubSpot, and Slack nodes | Setup guidance |
| Add your HubSpot API token and Slack credentials | Setup guidance |
| Test the workflow using sample webhook data | Setup guidance |
| Adjust usage thresholds based on your pricing model | Customization guidance |
| Modify AI prompts for different tones or offers | Customization guidance |
| Add segmentation logic for different customer tiers | Customization guidance |