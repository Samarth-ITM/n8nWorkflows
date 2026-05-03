Send women’s safety emergency alerts with GPT-4o-mini, Telegram and Google Sheets

https://n8nworkflows.xyz/workflows/send-women-s-safety-emergency-alerts-with-gpt-4o-mini--telegram-and-google-sheets-14359


# Send women’s safety emergency alerts with GPT-4o-mini, Telegram and Google Sheets

# 1. Workflow Overview

This workflow implements a real-time emergency alert pipeline for women’s safety scenarios. It accepts an incoming HTTP POST request from a panic button, safety app, or frontend, enriches the payload with a Google Maps link and localized timestamp, uses GPT-4o-mini to format a clean emergency message, then sends the alert to a Telegram group and logs the incident in Google Sheets.

Typical use cases:
- Mobile safety app panic button
- Wearable emergency trigger
- Web-based emergency form
- Internal rapid-response alerting system

## 1.1 Input Reception

The workflow starts from a single webhook entry point. It waits for an HTTP POST request containing `name`, `phone`, `lat`, and `lng`.

## 1.2 Data Preparation

The incoming payload is normalized, missing values are defaulted, and a Google Maps URL is generated from the coordinates. A second preparation step creates a timestamp in India Standard Time and builds a preliminary alert message.

## 1.3 AI Formatting

The prepared data is sent into an AI Agent backed by the OpenAI Chat Model (`gpt-4o-mini`). The prompt strictly instructs the model to output a concise, emoji-tagged emergency alert in a fixed format.

## 1.4 Dispatch and Logging

Once the AI-generated message is available, the workflow branches in parallel:
- One branch sends the final alert to a Telegram group
- One branch appends an incident record to Google Sheets

## 1.5 Security and Configuration Notes

The workflow relies on three external credential sets:
- OpenAI API credential
- Telegram Bot credential
- Google Sheets OAuth2 credential

It also requires replacing placeholder values such as the Telegram `chatId` and Google Sheet `documentId`.

---

# 2. Block-by-Block Analysis

## Block 1 — Input Reception

### Overview
This block receives the emergency trigger from an external app or button. It is the only execution entry point in the workflow and begins the entire processing chain when a POST request is received.

### Nodes Involved
- Receive Emergency Alert

### Node Details

#### Receive Emergency Alert
- **Type and technical role:** `n8n-nodes-base.webhook`  
  Acts as the HTTP trigger for the workflow.
- **Configuration choices:**  
  - HTTP method: `POST`
  - Path: `emergency-alert`
  - No extra webhook options are configured
  - Webhook ID is defined internally as `emergency-alert-webhook`
- **Key expressions or variables used:**  
  None in node parameters. Downstream nodes expect the request body to contain:
  - `name`
  - `phone`
  - `lat`
  - `lng`
- **Input and output connections:**  
  - Input: none, this is the trigger
  - Output: `Generate Maps Link`
- **Version-specific requirements (if any):**  
  Uses webhook node type version `1`
- **Edge cases or potential failure types:**  
  - If the caller sends malformed JSON, the webhook may reject or produce unexpected structure
  - If the caller sends data outside the body, downstream logic may miss fields
  - No authentication or signature validation is configured, so the endpoint is publicly callable unless protected externally
  - No schema validation is present, so invalid or missing coordinates are still accepted
- **Sub-workflow reference:**  
  None

---

## Block 2 — Data Preparation

### Overview
This block extracts the emergency fields from the webhook payload, applies fallback values, constructs a Google Maps link, and creates a timestamped alert payload. It prepares both machine-usable structured data and a human-readable draft message before AI formatting.

### Nodes Involved
- Generate Maps Link
- Build Timestamped Message

### Node Details

#### Generate Maps Link
- **Type and technical role:** `n8n-nodes-base.function`  
  Custom JavaScript node that normalizes input and creates a Google Maps URL.
- **Configuration choices:**  
  The function reads from either:
  - `$input.first().json.body`, or
  - `$input.first().json`  
  This makes it resilient to slight differences in webhook payload shape.
- **Key expressions or variables used:**  
  JavaScript variables:
  - `body`
  - `name = body.name || 'Unknown'`
  - `phone = body.phone || 'Unknown'`
  - `lat = body.lat || '0'`
  - `lng = body.lng || '0'`
  - `mapsLink = https://www.google.com/maps?q=${lat},${lng}`
- **Input and output connections:**  
  - Input: `Receive Emergency Alert`
  - Output: `Build Timestamped Message`
- **Version-specific requirements (if any):**  
  Uses Function node version `1`, which depends on the legacy Function node model available in n8n
- **Edge cases or potential failure types:**  
  - Missing fields are silently replaced with fallback values
  - Invalid latitude/longitude still produce a Google Maps link, but that link may be useless
  - If the input is not an object, the code may fail
  - Coordinates are not sanitized or validated
- **Sub-workflow reference:**  
  None

#### Build Timestamped Message
- **Type and technical role:** `n8n-nodes-base.function`  
  Custom JavaScript node that enriches the data with a localized timestamp and draft emergency text.
- **Configuration choices:**  
  - Uses `new Date()`
  - Formats timestamp with locale `en-IN`
  - Forces timezone `Asia/Kolkata`
  - Includes `dateStyle: 'full'` and `timeStyle: 'long'`
  - Builds a multi-line alert message
- **Key expressions or variables used:**  
  - `const data = $input.first().json`
  - `timestamp = now.toLocaleString('en-IN', { timeZone: 'Asia/Kolkata', dateStyle: 'full', timeStyle: 'long' })`
  - `message = ...`
- **Input and output connections:**  
  - Input: `Generate Maps Link`
  - Output: `AI Agent: Format Emergency Alert`
- **Version-specific requirements (if any):**  
  Uses Function node version `1`
- **Edge cases or potential failure types:**  
  - Locale/timezone formatting depends on Node.js runtime support
  - The draft `message` is not used for Telegram delivery; Telegram uses AI output instead
  - If input fields are missing, the draft message will still be generated with fallback values
- **Sub-workflow reference:**  
  None

---

## Block 3 — AI Formatting

### Overview
This block passes structured emergency data to an AI Agent that formats it into a standardized alert message. The agent depends on an OpenAI Chat Model node and is designed to produce a concise, visually readable alert with minimal variation.

### Nodes Involved
- AI Agent: Format Emergency Alert
- OpenAI Chat Model

### Node Details

#### AI Agent: Format Emergency Alert
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  LangChain-based AI Agent node that receives workflow data and generates formatted text.
- **Configuration choices:**  
  - Prompt type: `define`
  - Prompt is fully custom and embedded directly in the node
  - The prompt includes:
    - contextual role definition
    - input field interpolation
    - formatting instructions
    - a strict output template
    - explicit prohibitions against extra text
- **Key expressions or variables used:**  
  Inline expressions in the prompt:
  - `{{ $json.name }}`
  - `{{ $json.phone }}`
  - `{{ $json.lat }}`
  - `{{ $json.lng }}`
  - `{{ $json.mapsLink }}`
  - `{{ $json.timestamp }}`
  
  The downstream Telegram node uses:
  - `{{ $json.output }}`  
  which implies the agent returns its generated message in the `output` field.
- **Input and output connections:**  
  - Main input: `Build Timestamped Message`
  - AI language model input: `OpenAI Chat Model`
  - Main outputs:
    - `Send Alert to Telegram Group`
    - `Log Incident to Google Sheets`
- **Version-specific requirements (if any):**  
  Uses node type version `3`; requires n8n with LangChain AI nodes installed/enabled
- **Edge cases or potential failure types:**  
  - OpenAI credential issues will prevent execution
  - Model quota/rate limits can interrupt alert generation
  - LLMs may occasionally deviate from formatting despite prompt constraints
  - If AI output contains Markdown-special characters, Telegram rendering may be affected because parse mode is enabled
  - If the AI node fails, neither Telegram nor Sheets branch will run
- **Sub-workflow reference:**  
  None

#### OpenAI Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Provides the language model used by the AI Agent.
- **Configuration choices:**  
  - Model: `gpt-4o-mini`
  - No additional options set
  - No built-in tools configured
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Output: connected to `AI Agent: Format Emergency Alert` as `ai_languageModel`
- **Version-specific requirements (if any):**  
  Node type version `1.3`; requires compatible n8n AI/LangChain support and valid OpenAI credentials
- **Edge cases or potential failure types:**  
  - Invalid API key
  - Account billing issues
  - Model unavailability in the connected OpenAI account
  - Timeout/network failures
- **Sub-workflow reference:**  
  None

---

## Block 4 — Dispatch and Logging

### Overview
This block sends the AI-formatted alert to responders through Telegram while also recording the incident in Google Sheets. The two nodes execute in parallel from the AI Agent output, but they do not use identical data sources.

### Nodes Involved
- Send Alert to Telegram Group
- Log Incident to Google Sheets

### Node Details

#### Send Alert to Telegram Group
- **Type and technical role:** `n8n-nodes-base.telegram`  
  Sends the final emergency alert to a Telegram chat or group using a bot account.
- **Configuration choices:**  
  - Text is set from AI output: `{{ $json.output }}`
  - `chatId` is a placeholder: `YOUR_TELEGRAM_CHAT_ID`
  - Additional field `parse_mode` is set to `Markdown`
- **Key expressions or variables used:**  
  - `={{ $json.output }}`
- **Input and output connections:**  
  - Input: `AI Agent: Format Emergency Alert`
  - Output: none
- **Version-specific requirements (if any):**  
  Uses Telegram node version `1`
- **Edge cases or potential failure types:**  
  - Placeholder `chatId` must be replaced or the node will fail
  - Bot must be added to the group and have permission to post
  - Markdown parse mode can break if the AI output contains unescaped Markdown syntax
  - Invalid bot token or revoked credential will fail authentication
  - Telegram group IDs are often negative integers; wrong format causes delivery errors
- **Sub-workflow reference:**  
  None

#### Log Incident to Google Sheets
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends a new row to a Google Sheet for persistent incident tracking.
- **Configuration choices:**  
  - Operation: `append`
  - Spreadsheet document ID is a placeholder: `YOUR_GOOGLE_SHEET_ID`
  - Sheet/tab selected as `gid=0`, cached display name `safety data`
  - Mapping mode: define below
  - Explicitly maps columns:
    - `name`
    - `phone`
    - `message`
    - `mapsLink`
    - `timestamp`
  - Uses expressions referencing the earlier `Build Timestamped Message` node rather than the AI output
- **Key expressions or variables used:**  
  - `{{ $('Build Timestamped Message').item.json.name }}`
  - `{{ $('Build Timestamped Message').item.json.phone }}`
  - `{{ $('Build Timestamped Message').item.json.message }}`
  - `{{ $('Build Timestamped Message').item.json.mapsLink }}`
  - `{{ $('Build Timestamped Message').item.json.timestamp }}`
- **Input and output connections:**  
  - Input: `AI Agent: Format Emergency Alert`
  - Output: none
- **Version-specific requirements (if any):**  
  Uses Google Sheets node version `4`
- **Edge cases or potential failure types:**  
  - Placeholder document ID must be replaced
  - OAuth2 credential must have access to the spreadsheet
  - Sheet column names must match the configured schema
  - Since the node logs `Build Timestamped Message.message` rather than AI output, the sheet record may differ from the Telegram message
  - If the referenced upstream node name is changed, expressions break
  - Append failures may happen due to permission, quota, or spreadsheet structure changes
- **Sub-workflow reference:**  
  None

---

## Block 5 — Documentation and Security Notes

### Overview
This workflow includes several sticky notes that document purpose, setup, processing stages, and credential handling directly on the canvas. These notes are important because they clarify configuration expectations and intended operation.

### Nodes Involved
- Sticky Note: Overview
- Sticky Note: Webhook
- Sticky Note: Data Prep
- Sticky Note: AI Formatting
- Sticky Note: Dispatch & Logging
- Sticky Note: Security

### Node Details

#### Sticky Note: Overview
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual documentation note summarizing the workflow and setup.
- **Configuration choices:**  
  Contains high-level description and setup checklist for webhook, OpenAI, Telegram, and Google Sheets.
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  None
- **Version-specific requirements (if any):**  
  Sticky note version `1`
- **Edge cases or potential failure types:**  
  None operational
- **Sub-workflow reference:**  
  None

#### Sticky Note: Webhook
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Documents that the webhook expects `name`, `phone`, `lat`, and `lng` and is the sole entry point.
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  None
- **Version-specific requirements (if any):**  
  Sticky note version `1`
- **Edge cases or potential failure types:**  
  None operational
- **Sub-workflow reference:**  
  None

#### Sticky Note: Data Prep
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Describes field extraction, map link generation, and IST timestamp handling.
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  None
- **Version-specific requirements (if any):**  
  Sticky note version `1`
- **Edge cases or potential failure types:**  
  None operational
- **Sub-workflow reference:**  
  None

#### Sticky Note: AI Formatting
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Explains that GPT-4o-mini formats a concise emergency alert in a stable structure.
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  None
- **Version-specific requirements (if any):**  
  Sticky note version `1`
- **Edge cases or potential failure types:**  
  None operational
- **Sub-workflow reference:**  
  None

#### Sticky Note: Dispatch & Logging
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Documents parallel alert dispatch to Telegram and row logging to Google Sheets.
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  None
- **Version-specific requirements (if any):**  
  Sticky note version `1`
- **Edge cases or potential failure types:**  
  None operational
- **Sub-workflow reference:**  
  None

#### Sticky Note: Security
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Advises use of OAuth2 and stored n8n credentials, and warns against sharing tokens and IDs.
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  None
- **Version-specific requirements (if any):**  
  Sticky note version `1`
- **Edge cases or potential failure types:**  
  None operational
- **Sub-workflow reference:**  
  None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note: Overview | stickyNote | Canvas documentation with workflow summary and setup steps |  |  | ## 🚨 Women's Safety Emergency Alert System<br>### How it works<br>When a safety app or button sends a POST request to this webhook, the workflow immediately extracts the person's name, phone number, and GPS coordinates, builds a Google Maps link, and passes everything through an AI agent that formats a clear, urgent alert message. That message is sent simultaneously to a Telegram group and logged to a Google Sheet for a permanent safety record.<br>This is designed for real-time personal safety — no delay, no manual steps. One trigger fires everything at once.<br>### Setup steps<br>1. **Webhook** — Copy the webhook URL and configure your safety app or frontend to POST `name`, `phone`, `lat`, and `lng` in the request body.<br>2. **OpenAI** — Connect your OpenAI credential to the `OpenAI Chat Model` node. Uses `gpt-4o-mini` by default.<br>3. **Telegram** — Connect a Telegram Bot credential and update the `chatId` in `Send Alert to Telegram Group` with your safety group's chat ID.<br>4. **Google Sheets** — Connect your Google Sheets OAuth2 credential and update the `documentId` in `Log Incident to Google Sheets` with your own tracking spreadsheet ID.<br>5. **Test** — Send a test POST payload with sample name, phone, lat, and lng values. Confirm the Telegram message arrives and a row is added to the sheet. |
| Sticky Note: Webhook | stickyNote | Canvas documentation for webhook expectations |  |  | ## 📡 Webhook Trigger<br>Receives an emergency POST request from a safety app or panic button. Expects `name`, `phone`, `lat`, and `lng` in the body. This is the only entry point — nothing runs until this fires. |
| Sticky Note: Data Prep | stickyNote | Canvas documentation for preparation logic |  |  | ## 🗺️ Data Preparation<br>Extracts the incoming fields, builds a Google Maps link from the GPS coordinates, and assembles a timestamped message (IST timezone). These two steps prepare all the data before the AI formats the final alert. |
| Sticky Note: AI Formatting | stickyNote | Canvas documentation for AI formatting stage |  |  | ## 🤖 AI Alert Formatting<br>Passes the structured data to GPT-4o-mini, which formats a concise, emoji-tagged emergency alert. The prompt enforces a strict output format so the Telegram message is always readable at a glance — no filler, no variation. |
| Sticky Note: Dispatch & Logging | stickyNote | Canvas documentation for parallel output stage |  |  | ## 📬 Alert Dispatch & Logging<br>The formatted alert fires to both destinations simultaneously: a Telegram group for immediate human response, and a Google Sheet for a permanent incident record with name, phone, location, and timestamp. |
| Sticky Note: Security | stickyNote | Canvas documentation for credential hygiene |  |  | ## 🔐 Credentials & Security<br>Use OAuth2 for Google Sheets. Store Telegram and OpenAI keys as named n8n credentials — never paste tokens directly into nodes. Replace all sheet and chat IDs before sharing this template. |
| Receive Emergency Alert | webhook | Receives incoming emergency POST request |  | Generate Maps Link | ## 📡 Webhook Trigger<br>Receives an emergency POST request from a safety app or panic button. Expects `name`, `phone`, `lat`, and `lng` in the body. This is the only entry point — nothing runs until this fires. |
| Generate Maps Link | function | Normalizes incoming data and builds Google Maps URL | Receive Emergency Alert | Build Timestamped Message | ## 🗺️ Data Preparation<br>Extracts the incoming fields, builds a Google Maps link from the GPS coordinates, and assembles a timestamped message (IST timezone). These two steps prepare all the data before the AI formats the final alert. |
| Build Timestamped Message | function | Adds IST timestamp and draft emergency message | Generate Maps Link | AI Agent: Format Emergency Alert | ## 🗺️ Data Preparation<br>Extracts the incoming fields, builds a Google Maps link from the GPS coordinates, and assembles a timestamped message (IST timezone). These two steps prepare all the data before the AI formats the final alert. |
| AI Agent: Format Emergency Alert | @n8n/n8n-nodes-langchain.agent | Formats the emergency alert via LLM prompt | Build Timestamped Message; OpenAI Chat Model | Send Alert to Telegram Group; Log Incident to Google Sheets | ## 🤖 AI Alert Formatting<br>Passes the structured data to GPT-4o-mini, which formats a concise, emoji-tagged emergency alert. The prompt enforces a strict output format so the Telegram message is always readable at a glance — no filler, no variation. |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Supplies GPT-4o-mini model to the AI Agent |  | AI Agent: Format Emergency Alert | ## 🤖 AI Alert Formatting<br>Passes the structured data to GPT-4o-mini, which formats a concise, emoji-tagged emergency alert. The prompt enforces a strict output format so the Telegram message is always readable at a glance — no filler, no variation. |
| Send Alert to Telegram Group | telegram | Sends formatted alert to Telegram responders | AI Agent: Format Emergency Alert |  | ## 📬 Alert Dispatch & Logging<br>The formatted alert fires to both destinations simultaneously: a Telegram group for immediate human response, and a Google Sheet for a permanent incident record with name, phone, location, and timestamp. |
| Log Incident to Google Sheets | googleSheets | Appends emergency record to spreadsheet | AI Agent: Format Emergency Alert |  | ## 📬 Alert Dispatch & Logging<br>The formatted alert fires to both destinations simultaneously: a Telegram group for immediate human response, and a Google Sheet for a permanent incident record with name, phone, location, and timestamp.<br>## 🔐 Credentials & Security<br>Use OAuth2 for Google Sheets. Store Telegram and OpenAI keys as named n8n credentials — never paste tokens directly into nodes. Replace all sheet and chat IDs before sharing this template. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `AI-Powered Women Safety Emergency Alert System`
   - Keep it inactive until testing is complete.

2. **Add a Webhook node**
   - Node type: `Webhook`
   - Name: `Receive Emergency Alert`
   - Set:
     - HTTP Method: `POST`
     - Path: `emergency-alert`
   - Leave advanced options empty unless you want authentication or response customization.
   - This node will become the only trigger.

3. **Prepare the caller payload**
   - Ensure your external app posts JSON in the request body with:
     - `name`
     - `phone`
     - `lat`
     - `lng`
   - Example shape:
     - `name`: person’s name
     - `phone`: phone number
     - `lat`: latitude
     - `lng`: longitude

4. **Add a Function node for map link generation**
   - Node type: `Function`
   - Name: `Generate Maps Link`
   - Connect `Receive Emergency Alert` → `Generate Maps Link`
   - Use logic equivalent to:
     - Read from `json.body` if present, otherwise from `json`
     - Default missing values to:
       - `name = 'Unknown'`
       - `phone = 'Unknown'`
       - `lat = '0'`
       - `lng = '0'`
     - Build:
       - `mapsLink = https://www.google.com/maps?q=<lat>,<lng>`
   - Return one item containing:
     - `name`
     - `phone`
     - `lat`
     - `lng`
     - `mapsLink`

5. **Add a second Function node for timestamp and draft message**
   - Node type: `Function`
   - Name: `Build Timestamped Message`
   - Connect `Generate Maps Link` → `Build Timestamped Message`
   - Configure logic to:
     - Read the previous node’s JSON
     - Generate current date/time
     - Format timestamp using:
       - locale: `en-IN`
       - timezone: `Asia/Kolkata`
       - date style: `full`
       - time style: `long`
     - Build a draft message with:
       - emergency title
       - name
       - phone
       - static distress sentence
       - map link
       - timestamp
   - Return:
     - original data
     - `message`
     - `timestamp`

6. **Add the OpenAI Chat Model node**
   - Node type: `OpenAI Chat Model`
   - Name: `OpenAI Chat Model`
   - Choose model: `gpt-4o-mini`
   - Leave other options at defaults unless you want temperature or token customization.
   - Create or attach an OpenAI API credential.
   - Credential requirement:
     - valid OpenAI API key
     - account with access to `gpt-4o-mini`

7. **Add the AI Agent node**
   - Node type: `AI Agent`
   - Name: `AI Agent: Format Emergency Alert`
   - Connect:
     - `Build Timestamped Message` → `AI Agent: Format Emergency Alert`
     - `OpenAI Chat Model` → AI Agent’s `ai_languageModel` input
   - Set prompt type to a manually defined prompt.
   - Configure the prompt to include:
     - role: women’s safety emergency formatting assistant
     - input data fields:
       - `{{ $json.name }}`
       - `{{ $json.phone }}`
       - `{{ $json.lat }}`
       - `{{ $json.lng }}`
       - `{{ $json.mapsLink }}`
       - `{{ $json.timestamp }}`
     - instructions:
       - urgent but clear
       - short and easy to read
       - use emojis
       - make location obvious
       - no unnecessary explanation
     - strict output template
       - header
       - name
       - phone
       - distress line
       - Google Maps link
       - timestamp
     - explicit guardrails:
       - always return formatted message
       - no extra text
   - Expect the AI Agent to emit the generated text in an `output` field.

8. **Add the Telegram node**
   - Node type: `Telegram`
   - Name: `Send Alert to Telegram Group`
   - Connect `AI Agent: Format Emergency Alert` → `Send Alert to Telegram Group`
   - Configure:
     - Message text: `{{ $json.output }}`
     - Chat ID: replace `YOUR_TELEGRAM_CHAT_ID` with your real group/chat ID
     - Parse mode: `Markdown`
   - Create or attach a Telegram Bot credential.
   - Credential/setup requirements:
     - create a Telegram bot via BotFather
     - copy the bot token into n8n credentials
     - add the bot to the target group
     - allow the bot to send messages
   - Constraint:
     - if using Markdown, avoid malformed formatting characters in generated text

9. **Add the Google Sheets node**
   - Node type: `Google Sheets`
   - Name: `Log Incident to Google Sheets`
   - Connect `AI Agent: Format Emergency Alert` → `Log Incident to Google Sheets`
   - Set:
     - Operation: `Append`
     - Document ID: replace `YOUR_GOOGLE_SHEET_ID`
     - Sheet/tab: select the intended sheet, in this workflow it is the first tab (`gid=0`)
   - Use column mapping mode where you define fields manually.
   - Prepare the spreadsheet with columns:
     - `name`
     - `phone`
     - `mapsLink`
     - `message`
     - `timestamp`
   - Map values using expressions that reference `Build Timestamped Message`:
     - `name` → `{{ $('Build Timestamped Message').item.json.name }}`
     - `phone` → `{{ $('Build Timestamped Message').item.json.phone }}`
     - `message` → `{{ $('Build Timestamped Message').item.json.message }}`
     - `mapsLink` → `{{ $('Build Timestamped Message').item.json.mapsLink }}`
     - `timestamp` → `{{ $('Build Timestamped Message').item.json.timestamp }}`
   - Create or attach a Google Sheets OAuth2 credential.
   - Credential/setup requirements:
     - authorize the Google account that owns or can edit the sheet
     - ensure the spreadsheet is accessible to that account

10. **Understand the branching behavior**
    - The AI Agent fans out to two destinations in parallel:
      - Telegram for immediate response
      - Google Sheets for record keeping
    - Note that:
      - Telegram uses AI output
      - Google Sheets logs the earlier draft `message`, not the AI output
    - If you want both outputs to match exactly, change the `message` column mapping to `{{ $json.output }}` from the AI Agent instead.

11. **Optionally add sticky notes for maintainability**
    - Add notes for:
      - overall purpose
      - webhook expectations
      - data prep logic
      - AI formatting intent
      - dispatch/logging behavior
      - credential security
    - This is not operationally required but helps future maintainers.

12. **Test the webhook**
    - In n8n, copy the test webhook URL from `Receive Emergency Alert`
    - Send a POST request with JSON such as:
      - `name`: `Anita`
      - `phone`: `+911234567890`
      - `lat`: `28.6139`
      - `lng`: `77.2090`
    - Confirm:
      - `Generate Maps Link` outputs a valid Google Maps URL
      - `Build Timestamped Message` adds a readable IST timestamp
      - AI Agent returns a formatted message in `output`
      - Telegram receives the alert
      - Google Sheets appends a row

13. **Harden the workflow before production**
    - Add webhook authentication or proxy protection if the endpoint should not be public
    - Consider validating `lat` and `lng`
    - Consider adding fallback routing if OpenAI fails
    - Consider replacing Markdown parse mode with plain text if formatting errors occur
    - Consider logging AI output to Sheets for consistency

14. **Activate the workflow**
    - Once all credentials, IDs, and tests are complete, activate the workflow.
   - Use the production webhook URL in the safety app or panic button integration.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Women’s Safety Emergency Alert System: receives panic POST requests, formats urgent alerts with GPT-4o-mini, sends to Telegram, and logs to Google Sheets. | Workflow purpose |
| Setup expects request body fields `name`, `phone`, `lat`, and `lng`. | Webhook input contract |
| Replace placeholder values before use: Telegram `chatId` and Google Sheets `documentId`. | Required configuration |
| Use stored n8n credentials for OpenAI, Telegram, and Google Sheets; do not paste secrets directly into node fields. | Security practice |
| Google Sheets logging currently stores the draft message from `Build Timestamped Message`, not the AI-generated `output`. | Important implementation note |
| Time formatting is localized to India Standard Time using `Asia/Kolkata` and `en-IN`. | Timestamp behavior |

