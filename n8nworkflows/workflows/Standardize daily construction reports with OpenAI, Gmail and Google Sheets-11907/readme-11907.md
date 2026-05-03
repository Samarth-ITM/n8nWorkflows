Standardize daily construction reports with OpenAI, Gmail and Google Sheets

https://n8nworkflows.xyz/workflows/standardize-daily-construction-reports-with-openai--gmail-and-google-sheets-11907


# Standardize daily construction reports with OpenAI, Gmail and Google Sheets

## 1. Workflow Overview

**Purpose:**  
This workflow collects raw Daily Construction Report (DCR) notes from field staff via an n8n Chat Interface, validates the submission against jailbreak/adversarial patterns, and—if safe—uses OpenAI to standardize the DCR into a strict template. It then logs the standardized report to Google Sheets and emails it to stakeholders via Gmail. If the submission is flagged, it alerts stakeholders by email and blocks processing, notifying the user in-chat.

**Target use cases:**
- Standardizing inconsistent field notes into a uniform DCR format
- Guarding AI workflows against prompt injection/jailbreak attempts
- Creating an auditable log of daily reports in Google Sheets
- Automated distribution of daily reports by email

### 1.1 Input Reception & Configuration
Receives DCR text via chat and sets runtime/config variables used across the workflow.

### 1.2 Input Safety Validation (Guardrails)
Runs an AI guardrail check against jailbreak patterns. Branches to either:
- Standardization (safe) or
- Alert + block (unsafe)

### 1.3 AI Standardization
Uses an LLM agent with a fixed system template to produce a standardized DCR output.

### 1.4 Distribution & Logging
Appends the standardized DCR to Google Sheets, emails it via Gmail, then confirms success in chat.

---

## 2. Block-by-Block Analysis

### Block 1 — Capture and validate daily site input
**Overview:**  
Collects raw DCR notes via an embedded/public chat interface, then initializes configuration values (timestamp, recipients, spreadsheet IDs) used by later nodes.

**Nodes involved:**
- Sticky Note (section header)
- Sticky Note4 (workflow overview/setup note)
- Chat: Capture DCR Input
- Config

#### Node: Sticky Note
- **Type / role:** Sticky Note (documentation)
- **Configuration choices:** Title: “## 1. Capture and validate daily site input”
- **Connections:** None
- **Failure types:** None

#### Node: Sticky Note4
- **Type / role:** Sticky Note (documentation)
- **Configuration choices:** Contains “How It Works” and “Setup” checklist for operators.
- **Connections:** None
- **Failure types:** None

#### Node: Chat: Capture DCR Input
- **Type / role:** `@n8n/n8n-nodes-langchain.chatTrigger` — Entry point (public chat webhook/UI)
- **Key configuration:**
  - **Public:** enabled (`public: true`) → anyone with the chat link can submit
  - **Response mode:** `responseNodes` (responses are produced by downstream “Chat” response nodes)
  - **UI customization:** extensive Custom CSS + header background image URL
  - **Title/subtitle:** “Chat Interface for Daily Construction Report (DCR)” with creator credit link
  - **Initial message:** prompts user to include 9 categories of DCR info
  - **Input placeholder:** “Enter the information here...”
- **Key data used later:**
  - `$('Chat: Capture DCR Input').item.json.chatInput` is the raw text passed to guardrails
- **Connections:**
  - **Output →** Config
- **Edge cases / failures:**
  - Public endpoint may receive spam/non-DCR submissions
  - Very long inputs may cause LLM token issues later (guardrail/model)
  - If the chat UI is unpublished/inaccessible, no executions start
- **Version notes:** TypeVersion `1.4` (ensure your n8n supports the Chat Trigger node in this version range)

#### Node: Config
- **Type / role:** `n8n-nodes-base.set` — Central configuration/runtime variables
- **Configuration choices (assignments):**
  - `runTimestamp`: `DateTime.now().setZone('America/Sao_Paulo').toFormat('dd/MM/yyyy HH:mm:ss')`
  - `reportLanguage`: `"English"` (currently informational; not actively used in prompts)
  - `emailTo`: `"user@example.com"` (recipient(s) for both alert and success emails)
  - `emailErrorSubjectPrefix`: `"Alert – Jailbreak Attempt"`
  - `emailSuccessSubjectPrefix`: `"=Daily Construction Report"` (leading `=` is literal in the value as configured)
  - `logSpreadsheetId`: `"REPLACE_WITH_SPREADSHEET_ID"`
  - `logSheetName`: `"Página1"`
- **Connections:**
  - **Input ←** Chat: Capture DCR Input
  - **Output →** Guardrail: Validate Input
- **Edge cases / failures:**
  - Invalid timezone string would break timestamp expression (Luxon `DateTime`)
  - Spreadsheet ID placeholder not replaced → Google Sheets node will fail
  - Wrong sheet name (“Página1”) not found → append fails later
- **Version notes:** TypeVersion `3.4`

---

### Block 2 — AI-Powered DCR standardization
**Overview:**  
Validates the incoming text with a jailbreak detector (guardrails). If safe, an AI agent reformats the content into a strict DCR template.

**Nodes involved:**
- Sticky Note2 (section header)
- Guardrail Chat Model: OpenAI
- Guardrail: Validate Input
- AI Chat Model: OpenAI
- AI: Standardize DCR

#### Node: Sticky Note2
- **Type / role:** Sticky Note (documentation)
- **Configuration choices:** “## 2. AI-Powered DCR standardization”
- **Connections:** None

#### Node: Guardrail Chat Model: OpenAI
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — LLM powering the guardrail node
- **Configuration choices:**
  - Model: `gpt-4.1-mini`
  - Built-in tools: none
- **Connections:**
  - **Output (ai_languageModel) →** Guardrail: Validate Input
- **Credentials required:** OpenAI API credential
- **Edge cases / failures:**
  - Invalid API key / revoked credential
  - Model name not available in the account/region
  - Rate limiting/timeouts from OpenAI
- **Version notes:** TypeVersion `1.3`

#### Node: Guardrail: Validate Input
- **Type / role:** `@n8n/n8n-nodes-langchain.guardrails` — Safety filter (jailbreak detection)
- **Configuration choices:**
  - **Text to validate:** `={{ $('Chat: Capture DCR Input').item.json.chatInput }}`
  - **Guardrails enabled:** `jailbreak` with `threshold: 0.7`
  - **Customize prompt:** enabled (`customizePrompt: true`)
  - **Customize system message:** enabled
- **Behavior / outputs:**
  - Produces `guardrailsInput` (the validated text echo used in alert message)
  - Branching is implemented via two outgoing connections from this node:
    - **Path A (safe) →** AI: Standardize DCR
    - **Path B (flagged) →** Gmail: Jailbreak Alert
- **Connections:**
  - **Input ←** Config (main) and Guardrail Chat Model (ai_languageModel)
  - **Outputs →** AI: Standardize DCR (main) AND Gmail: Jailbreak Alert (main)
- **Important implementation note:**  
  The JSON shows two simultaneous outgoing connections from the guardrail node. In many designs, you would add an explicit IF/Switch based on the guardrail result. Here, the workflow relies on the guardrail node’s internal routing behavior (or its output content) to ensure only the appropriate branch effectively runs. If your n8n version executes both branches regardless, you must add a conditional node to prevent sending false alerts or processing unsafe text.
- **Edge cases / failures:**
  - False positives/negatives depending on threshold
  - Guardrail prompt customization mistakes could reduce detection quality
  - Missing `chatInput` (empty submission) may produce unpredictable scoring
- **Version notes:** TypeVersion `2`

#### Node: AI Chat Model: OpenAI
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — LLM used by the standardization agent
- **Configuration choices:**
  - Model: `gpt-4.1-mini`
- **Connections:**
  - **Output (ai_languageModel) →** AI: Standardize DCR
- **Credentials required:** OpenAI API credential
- **Edge cases / failures:** same as other OpenAI model node
- **Version notes:** TypeVersion `1.3`

#### Node: AI: Standardize DCR
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — Agent that produces the standardized DCR text
- **Configuration choices:**
  - **System message:** A detailed role + mandatory DCR template with strict rules:
    - Preserve facts/numbers
    - No invention/inference
    - Fixed section order
    - Use “Not informed” when missing
    - No emojis except the section title emojis provided
  - **Prompt type:** `guardrails` (indicates it’s intended to operate after validation)
- **Inputs:**
  - Main input: from Guardrail: Validate Input
  - LLM input: from AI Chat Model: OpenAI via `ai_languageModel` connection
- **Outputs:**
  - Produces `output` used downstream:
    - `{{ $json.output }}` in Google Sheets append
    - `{{ $('AI: Standardize DCR').item.json.output }}` in Gmail report email
- **Connections:**
  - **Output →** Sheets: Append DCR Log
- **Edge cases / failures:**
  - Output may not follow the template if model drifts; add post-validation if strict formatting is critical
  - Very long inputs can exceed token limits → truncated context → missing details
- **Version notes:** TypeVersion `3`

---

### Block 3 — Distribute and log work performance data
**Overview:**  
Logs the standardized DCR into Google Sheets, emails the report to recipients, and returns a success message in the chat interface. If guardrail flags input, sends an alert email and notifies the user that the submission was blocked.

**Nodes involved:**
- Sticky Note3 (section header)
- Sheets: Append DCR Log
- Gmail: Send DCR Report
- Chat: Success Notice
- End: Success
- Gmail: Jailbreak Alert
- Chat: Block Notice
- End: Guardrail Block

#### Node: Sticky Note3
- **Type / role:** Sticky Note (documentation)
- **Configuration choices:** “## 3. Distribute and log work performance data”
- **Connections:** None

#### Node: Sheets: Append DCR Log
- **Type / role:** `n8n-nodes-base.googleSheets` — Append standardized report to a logging sheet
- **Configuration choices:**
  - Operation: **Append**
  - Document ID: `={{ $('Config').item.json.logSpreadsheetId }}`
  - Sheet name: `={{ $('Config').item.json.logSheetName }}`
  - Columns mapped (“defineBelow”):
    - `Date/Time`: `={{ $('Config').item.json.runTimestamp }}`
    - `Report`: `={{ $json.output }}`
  - Type conversion disabled (`attemptToConvertTypes: false`)
- **Connections:**
  - **Input ←** AI: Standardize DCR
  - **Output →** Gmail: Send DCR Report
- **Credentials required:** Google Sheets OAuth2
- **Edge cases / failures:**
  - Spreadsheet not shared with credential account
  - Wrong sheet/tab name
  - Missing header columns “Date/Time” and “Report” (append may still work but mapping can fail depending on sheet structure)
  - Google API quota/rate limits
- **Version notes:** TypeVersion `4.7`

#### Node: Gmail: Send DCR Report
- **Type / role:** `n8n-nodes-base.gmail` — Email distribution of the standardized DCR
- **Configuration choices:**
  - To: `={{ $('Config').item.json.emailTo }}`
  - Subject: `={{ $('Config').item.json.emailSuccessSubjectPrefix }} - {{ $('Config').item.json.runTimestamp }}`
  - Body: `={{ $('AI: Standardize DCR').item.json.output }}`
  - Email type: `text`
- **Connections:**
  - **Input ←** Sheets: Append DCR Log
  - **Output →** Chat: Success Notice
- **Credentials required:** Gmail OAuth2
- **Edge cases / failures:**
  - OAuth token expired/revoked
  - Gmail sending limits (quota) or blocked “from” identity
  - Multiple recipients formatting (if you later change `emailTo` to a list)
- **Version notes:** TypeVersion `2.2`

#### Node: Chat: Success Notice
- **Type / role:** `@n8n/n8n-nodes-langchain.chat` — Chat response node (success path)
- **Configuration choices:**
  - Message: “✅ Your information has been successfully recorded!”
  - `waitUserReply: false`
- **Connections:**
  - **Input ←** Gmail: Send DCR Report
  - **Output →** End: Success
- **Edge cases / failures:**
  - If the workflow runs outside a chat-trigger context, chat response nodes may not behave as expected
- **Version notes:** TypeVersion `1`

#### Node: End: Success
- **Type / role:** `n8n-nodes-base.noOp` — End marker
- **Connections:**
  - **Input ←** Chat: Success Notice

---

#### Node: Gmail: Jailbreak Alert
- **Type / role:** `n8n-nodes-base.gmail` — Sends an alert when input is flagged
- **Configuration choices:**
  - To: `={{ $('Config').item.json.emailTo }}`
  - Subject: `={{ $('Config').item.json.emailErrorSubjectPrefix }}`
  - Message body:  
    `Message sent:\n\n{{ $json.guardrailsInput }}`
- **Connections:**
  - **Input ←** Guardrail: Validate Input
  - **Output →** Chat: Block Notice
- **Credentials required:** Gmail OAuth2
- **Edge cases / failures:**
  - Same Gmail auth/quota concerns as the success email
  - If `$json.guardrailsInput` is not present (node output shape differs by version), the email body may be blank or expression may fail
- **Version notes:** TypeVersion `2.2`

#### Node: Chat: Block Notice
- **Type / role:** `@n8n/n8n-nodes-langchain.chat` — Chat response node (blocked path)
- **Configuration choices:**
  - Message includes the received text:  
    `{{ $('Guardrail: Validate Input').item.json.guardrailsInput }}`
  - `waitUserReply: false`
- **Connections:**
  - **Input ←** Gmail: Jailbreak Alert
  - **Output →** End: Guardrail Block
- **Edge cases / failures:**
  - If the guardrail node does not return `guardrailsInput`, the interpolation fails or becomes empty
- **Version notes:** TypeVersion `1`

#### Node: End: Guardrail Block
- **Type / role:** `n8n-nodes-base.noOp` — End marker for blocked executions
- **Connections:**
  - **Input ←** Chat: Block Notice

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Section label |  |  | ## 1. Capture and validate daily site input |
| Sticky Note2 | Sticky Note | Section label |  |  | ## 2. AI-Powered DCR standardization |
| Sticky Note3 | Sticky Note | Section label |  |  | ## 3. Distribute and log work performance data |
| Sticky Note4 | Sticky Note | Workflow description & setup checklist |  |  | ## AI-driven daily construction report; How It Works; Setup checklist |
| Chat: Capture DCR Input | LangChain Chat Trigger | Public chat entry point for raw DCR submission |  | Config | ## 1. Capture and validate daily site input |
| Config | Set | Defines timestamp, recipients, sheet IDs/names | Chat: Capture DCR Input | Guardrail: Validate Input | ## 1. Capture and validate daily site input |
| Guardrail Chat Model: OpenAI | OpenAI Chat Model (LangChain) | LLM used by guardrail validation |  | Guardrail: Validate Input (ai_languageModel) | ## 2. AI-Powered DCR standardization |
| Guardrail: Validate Input | Guardrails | Detects jailbreak/adversarial patterns; routes to safe vs blocked handling | Config; Guardrail Chat Model: OpenAI (ai_languageModel) | AI: Standardize DCR; Gmail: Jailbreak Alert | ## 2. AI-Powered DCR standardization |
| Gmail: Jailbreak Alert | Gmail | Sends alert email when guardrail flags input | Guardrail: Validate Input | Chat: Block Notice | ## 3. Distribute and log work performance data |
| Chat: Block Notice | LangChain Chat | In-chat warning/blocked response | Gmail: Jailbreak Alert | End: Guardrail Block | ## 3. Distribute and log work performance data |
| End: Guardrail Block | NoOp | End marker for blocked path | Chat: Block Notice |  | ## 3. Distribute and log work performance data |
| AI Chat Model: OpenAI | OpenAI Chat Model (LangChain) | LLM used by DCR standardization agent |  | AI: Standardize DCR (ai_languageModel) | ## 2. AI-Powered DCR standardization |
| AI: Standardize DCR | LangChain Agent | Produces standardized DCR output in strict template | Guardrail: Validate Input; AI Chat Model: OpenAI (ai_languageModel) | Sheets: Append DCR Log | ## 2. AI-Powered DCR standardization |
| Sheets: Append DCR Log | Google Sheets | Appends standardized DCR to log sheet | AI: Standardize DCR | Gmail: Send DCR Report | ## 3. Distribute and log work performance data |
| Gmail: Send DCR Report | Gmail | Emails standardized DCR to stakeholders | Sheets: Append DCR Log | Chat: Success Notice | ## 3. Distribute and log work performance data |
| Chat: Success Notice | LangChain Chat | In-chat success confirmation | Gmail: Send DCR Report | End: Success | ## 3. Distribute and log work performance data |
| End: Success | NoOp | End marker for success path | Chat: Success Notice |  | ## 3. Distribute and log work performance data |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**  
   - Name: “04.AI-driven daily construction report” (or your preferred name)
   - Ensure n8n has access to **LangChain nodes**, **Gmail**, and **Google Sheets** nodes.

2. **Add the entry node: “Chat: Capture DCR Input”**
   - Node type: **LangChain → Chat Trigger**
   - Set **Public = true**
   - Set:
     - Title: “Chat Interface for Daily Construction Report (DCR)”
     - Subtitle: “Alysson Neves (https://n8n.io/creators/alysson)”
     - Response mode: **responseNodes**
     - Input placeholder: “Enter the information here...”
     - Initial message: prompt for the 9 DCR categories (as in the workflow)
   - (Optional) Paste the provided **Custom CSS** to match the UI/branding.

3. **Add “Config” node**
   - Node type: **Set**
   - Add fields:
     - `runTimestamp` (String): `DateTime.now().setZone('America/Sao_Paulo').toFormat('dd/MM/yyyy HH:mm:ss')`
     - `reportLanguage` (String): `English`
     - `emailTo` (String): your recipient(s)
     - `emailErrorSubjectPrefix` (String): `Alert – Jailbreak Attempt`
     - `emailSuccessSubjectPrefix` (String): `Daily Construction Report` (remove leading `=` unless you intentionally want it)
     - `logSpreadsheetId` (String): your Google Sheet file ID
     - `logSheetName` (String): your tab name (e.g., “Sheet1”)
   - Connect: **Chat Trigger → Config**

4. **Add OpenAI model for guardrails: “Guardrail Chat Model: OpenAI”**
   - Node type: **LangChain → OpenAI Chat Model**
   - Model: `gpt-4.1-mini` (or any suitable small/fast model)
   - Configure **OpenAI credentials** (API key).

5. **Add “Guardrail: Validate Input”**
   - Node type: **LangChain → Guardrails**
   - Text to validate (expression):  
     - `$('Chat: Capture DCR Input').item.json.chatInput`
   - Enable **Jailbreak** guardrail:
     - Threshold: `0.7`
     - Customize prompt: enabled
     - Customize system message: enabled (use your policy text as needed)
   - Connect:
     - **Config (main) → Guardrail**
     - **Guardrail Chat Model (ai_languageModel) → Guardrail**

6. **Add the blocked/alert path**
   1) Node: **“Gmail: Jailbreak Alert”**
   - Node type: **Gmail → Send**
   - Credentials: Gmail OAuth2 (connect the sending account)
   - To: `$('Config').item.json.emailTo`
   - Subject: `$('Config').item.json.emailErrorSubjectPrefix`
   - Body: include `{{$json.guardrailsInput}}` (or adjust to actual guardrail output in your version)
   2) Node: **“Chat: Block Notice”**
   - Node type: **LangChain → Chat**
   - Message includes the received content from the guardrail node
   - `waitUserReply = false`
   3) Node: **“End: Guardrail Block”**
   - Node type: **NoOp**
   - Connect: **Guardrail → Gmail Alert → Chat Block Notice → End: Guardrail Block**

7. **Add OpenAI model for standardization: “AI Chat Model: OpenAI”**
   - Node type: **LangChain → OpenAI Chat Model**
   - Model: `gpt-4.1-mini`
   - Credentials: OpenAI API

8. **Add “AI: Standardize DCR”**
   - Node type: **LangChain → Agent**
   - Set the **System Message** to the strict DCR template and rules (as in the workflow)
   - Connect:
     - **Guardrail (main) → AI Agent**
     - **AI Chat Model (ai_languageModel) → AI Agent**

9. **Add logging node: “Sheets: Append DCR Log”**
   - Node type: **Google Sheets**
   - Credentials: Google Sheets OAuth2 (account with access to the spreadsheet)
   - Operation: **Append**
   - Document ID: `$('Config').item.json.logSpreadsheetId`
   - Sheet name: `$('Config').item.json.logSheetName`
   - Map columns:
     - `Date/Time` = `$('Config').item.json.runTimestamp`
     - `Report` = `{{$json.output}}`
   - Connect: **AI Agent → Sheets Append**

10. **Add email distribution: “Gmail: Send DCR Report”**
   - Node type: **Gmail → Send**
   - Credentials: Gmail OAuth2
   - To: `$('Config').item.json.emailTo`
   - Subject: `$('Config').item.json.emailSuccessSubjectPrefix + " - " + $('Config').item.json.runTimestamp`
   - Body: `$('AI: Standardize DCR').item.json.output`
   - Connect: **Sheets Append → Gmail Send**

11. **Add chat confirmation and end**
   - Node: **“Chat: Success Notice”** (LangChain Chat)
     - Message: “Your information has been successfully recorded!”
     - `waitUserReply = false`
   - Node: **“End: Success”** (NoOp)
   - Connect: **Gmail Send → Chat Success Notice → End: Success**

12. **(Recommended hardening) Add an explicit branch condition**
   - If your guardrail node does not automatically stop the “safe” path on flagged input, insert an **IF** node after guardrails based on a boolean/score field produced by the guardrail node, and route explicitly:
     - IF flagged → alert path
     - ELSE → standardize path

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Chat Interface for Daily Construction Report (DCR)” includes creator credit | Alysson Neves (https://n8n.io/creators/alysson) |
| The chat UI uses a custom header background image | https://user-images.githubusercontent.com/10284570/173569848-c624317f-42b1-45a6-ab09-f0ea3c247648.png |
| Setup checklist included in the workflow | Publish chat interface; connect OpenAI; connect Google Sheets and select log spreadsheet; connect Gmail; set recipients/subject prefixes; adapt DCR template; test blocked + success flows |
| Disclaimer (provided) | Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n… (content compliance statement) |