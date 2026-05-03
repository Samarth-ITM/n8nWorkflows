Capture and nurture AI agency leads with Google Gemini, Outlook and Sheets

https://n8nworkflows.xyz/workflows/capture-and-nurture-ai-agency-leads-with-google-gemini--outlook-and-sheets-11616


# Capture and nurture AI agency leads with Google Gemini, Outlook and Sheets

## 1. Workflow Overview

**Workflow name:** AI Agent Outbound + Follow-up  
**Provided title:** Capture and nurture AI agency leads with Google Gemini, Outlook and Sheets

This workflow captures leads via an n8n Form, optionally performs an AI image edit using Google Gemini, logs lead data to Google Sheets, sends an AI-personalized outreach email via Microsoft Outlook, waits 48 hours, then sends an AI-written follow-up if no reply is detected (or tags the lead as “Interested” if they replied—currently placeholder logic).

### 1.1 Lead Capture + Image Delivery
- Receives lead info + an image file from an n8n form.
- Uses Gemini “image edit” to produce an edited image and emails it back.
- Stores the lead in Google Sheets.

### 1.2 AI Outreach Generation
- Prepares a small “automation ideas” library + lead identity fields.
- Uses a LangChain Agent (Gemini chat model) to select an idea and write a short personalized email (structured JSON output).
- Sends outreach email via Outlook with a booking link.

### 1.3 Follow-up System (48h)
- Waits 48 hours after outreach.
- Runs “check for reply” logic (currently hard-coded to **no reply**).
- If no reply: generates a short follow-up email with Gemini and sends it.
- If replied: updates the lead row in Google Sheets with an “Interested - Replied” status.

---

## 2. Block-by-Block Analysis

### Block 1 — Lead Capture + Image Editing + Logging
**Overview:** Collects lead details and an image, edits the image with Gemini, emails the edited result back, and logs lead details to Google Sheets.

**Nodes involved:**
- Property Photo Upload Form
- Edit an image
- Send a message
- Append or update row in sheet

#### Node: Property Photo Upload Form
- **Type / role:** `Form Trigger` (`n8n-nodes-base.formTrigger`) — entry point; hosts a public form and outputs submitted fields + binary file.
- **Key configuration (interpreted):**
  - Form path: `magic-editor` (the endpoint users open to submit).
  - Title/description: “Magic Editor”; prompts user to upload an image and describe edits.
  - Fields:
    - **Image** (file, required, single file) → becomes binary data under property name `Image`.
    - **Description:** (text, required) → JSON key is literally `Description:`
    - **Email:** (text, required) → JSON key `Email:`
    - Name (text, required)
    - **Company Name?** (text, required) → JSON key `Company Name?`
- **Inputs/outputs:** No input; outputs to three parallel branches:
  1) Edit an image  
  2) Append or update row in sheet  
  3) Automation Ideas Library
- **Edge cases / failures:**
  - Users submitting invalid email format (no validation is configured here).
  - File upload limits (n8n instance limits; large images can exceed memory/binary constraints).
  - Field names include punctuation (`Description:`, `Email:`, `Company Name?`)—must be referenced exactly in expressions.

#### Node: Edit an image
- **Type / role:** `Google Gemini (LangChain) Image` (`@n8n/n8n-nodes-langchain.googleGemini`) — performs an AI image edit operation.
- **Configuration choices:**
  - Resource: `image`, Operation: `edit`
  - Input image: binary property **`Image`** from the form.
  - Prompt (expression):  
    `BASED OFF OF THIS DESCRIPTION, EDIT THE IMAGE: {{ $json['Description:'] }} ...`
- **Credentials:** Google PaLM/Gemini (`googlePalmApi`)
- **Inputs/outputs:** Receives form item; outputs to “Send a message”.
- **Edge cases / failures:**
  - Gemini credential/auth issues; quota limits.
  - Image edit may fail for unsupported formats or size constraints.
  - Output binary is expected later as `edited` (see Outlook attachment). If Gemini outputs a different binary property name in your n8n version, the attachment step will fail.

#### Node: Send a message
- **Type / role:** `Microsoft Outlook` (`n8n-nodes-base.microsoftOutlook`) — emails the edited image to the submitter.
- **Configuration choices:**
  - Subject: `Image Is ready!`
  - Body uses expressions referencing the **form node by name**:
    - `{{ $('Property Photo Upload Form').item.json.Name }}`
    - `{{ $('Property Photo Upload Form').item.json['Company Name?'] }}`
  - Recipient: `={{ $('Property Photo Upload Form').item.json['Email:'] }}`
  - Attachment: binary property name **`edited`**
- **Credentials:** Outlook OAuth2 (`microsoftOutlookOAuth2Api`)
- **Inputs/outputs:** Input from “Edit an image”; terminal node for this branch.
- **Edge cases / failures:**
  - Attachment property mismatch (`edited` not present) → message send fails.
  - Outlook OAuth token expiry/consent scope issues.
  - Email sending throttles or blocked attachments.

#### Node: Append or update row in sheet
- **Type / role:** `Google Sheets` (`n8n-nodes-base.googleSheets`) — stores lead record.
- **Configuration choices:**
  - Operation: **Append or update**
  - Document: “Warm Leads” (Spreadsheet ID `1bxEOqa0...`)
  - Sheet: `Sheet1` (gid=0)
  - Mapped columns:
    - Name, Time (`submittedAt`), Email, Company
  - Matching column: **Name**
- **Credentials:** Google Sheets OAuth2
- **Inputs/outputs:** Input from form; terminal for this branch.
- **Edge cases / failures:**
  - Matching on **Name** can overwrite/update the wrong person if names repeat. In practice, **Email** is safer as a unique key.
  - Sheet must already have compatible headers/columns. If missing, mapping fails.
  - Permission issues on the spreadsheet.

---

### Block 2 — AI Outreach Generation (Idea selection + email writing)
**Overview:** Prepares a list of automation ideas, has Gemini generate a concise personalized outreach email in structured JSON, parses it, and sends it via Outlook.

**Nodes involved:**
- Automation Ideas Library
- Google Gemini Chat Model
- Structured Output Parser
- AI Sales Agent - Idea Picker & Email Writer
- Parse AI Output
- Send Personalized Outreach Email

#### Node: Automation Ideas Library
- **Type / role:** `Set` (`n8n-nodes-base.set`) — enriches the item with an idea library and normalized lead fields.
- **Configuration choices:**
  - Adds `automationIdeas` (multi-line string list)
  - Copies/normalizes:
    - `name = {{$json.Name}}`
    - `email = {{$json['Email:']}}`
    - `company = {{$json['Company Name?']}}`
  - “Include other fields” enabled (keeps original form fields too).
- **Inputs/outputs:** Input from form; output to AI Sales Agent.
- **Edge cases / failures:**
  - If form field names change, these expressions break.
  - Long idea libraries may push token usage; keep concise.

#### Node: Google Gemini Chat Model
- **Type / role:** `LangChain Chat Model (Google Gemini)` (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`) — provides the LLM for the sales agent.
- **Configuration choices:** default options.
- **Credentials:** `googlePalmApi`
- **Connections:** Connected to the agent via **`ai_languageModel`** channel.
- **Edge cases / failures:** Credential/quota issues; model output variability.

#### Node: Structured Output Parser
- **Type / role:** `LangChain Structured Output Parser` (`@n8n/n8n-nodes-langchain.outputParserStructured`) — enforces JSON structure.
- **Configuration choices:**
  - Manual JSON schema requiring:
    - `chosenIdea` (string)
    - `emailBody` (string)
- **Connections:** Connected to the agent via **`ai_outputParser`** channel.
- **Edge cases / failures:** If the model output can’t be parsed to schema, the agent step fails.

#### Node: AI Sales Agent - Idea Picker & Email Writer
- **Type / role:** `LangChain Agent` (`@n8n/n8n-nodes-langchain.agent`) — chooses an automation idea and writes outreach email.
- **Configuration choices:**
  - Input text includes company, contact name, and `automationIdeas`.
  - System message constraints:
    - Under 150 words, casual-professional, not marketing-y
    - Return JSON with `chosenIdea` and `emailBody`
  - Output parser enabled (`hasOutputParser: true`)
- **Inputs/outputs:**
  - Main input from “Automation Ideas Library”
  - `ai_languageModel` from “Google Gemini Chat Model”
  - `ai_outputParser` from “Structured Output Parser”
  - Main output to “Parse AI Output”
- **Edge cases / failures:**
  - Parser failure if the agent returns non-JSON or extra text.
  - Hallucinated claims (“already built”) are explicitly required by prompt—ensure this matches your business reality.

#### Node: Parse AI Output
- **Type / role:** `Code` (`n8n-nodes-base.code`) — normalizes the structured agent output into flat fields used by email + follow-up.
- **Logic (interpreted):**
  - Reads `$input.first().json.output`
  - Emits:
    - `chosenIdea`, `emailBody`
    - plus `name/email/company` taken from the **Automation Ideas Library** node via `$('Automation Ideas Library').first().json...`
- **Inputs/outputs:** Input from agent; output to “Send Personalized Outreach Email”.
- **Edge cases / failures:**
  - If agent output key is not `output` in your n8n/LangChain node version, code breaks.
  - If multiple items are processed, using `.first()` may mismatch lead data (design assumes single-item flow).

#### Node: Send Personalized Outreach Email
- **Type / role:** `Microsoft Outlook` — sends the outreach email with booking CTA.
- **Configuration choices:**
  - Subject: `Quick automation idea for {{ $json.company }}`
  - Body includes `{{ $json.emailBody }}` plus calendar link: `https://cal.com/quartersmart/intro`
  - To: `{{ $json.email }}`
- **Inputs/outputs:** Input from Parse AI Output; output to Wait 48 Hours.
- **Edge cases / failures:**
  - Outlook auth/sending limits.
  - If `emailBody` already includes greeting/signoff, your wrapper greeting may duplicate tone.

---

### Block 3 — Follow-up System (wait, reply check, follow-up or tag)
**Overview:** Waits 48 hours, checks whether a reply was received (currently stubbed), then either sends a Gemini-written follow-up or marks the lead as replied/interested.

**Nodes involved:**
- Wait 48 Hours
- Check for Reply Logic
- Check If Reply Received
- AI Follow-up Email Writer
- Google Gemini Chat Model1
- Send Follow-up Email
- Tag as Interested

#### Node: Wait 48 Hours
- **Type / role:** `Wait` (`n8n-nodes-base.wait`) — delays workflow continuation.
- **Configuration choices:** 48 hours.
- **Inputs/outputs:** From outreach email; to “Check for Reply Logic”.
- **Edge cases / failures:**
  - Wait node behavior depends on n8n execution mode (main vs queue). Ensure your instance supports long waits and execution persistence.

#### Node: Check for Reply Logic
- **Type / role:** `Code` — placeholder for real reply detection.
- **Logic (interpreted):**
  - Copies each incoming item, adds `replyReceived: false`.
  - Comments indicate intended future: query Outlook for replies.
- **Inputs/outputs:** From Wait; to IF node.
- **Edge cases / failures:**
  - As written, it *always* triggers the “no reply” path.
  - If you later implement Outlook querying, handle:
    - conversation/thread IDs
    - “Re:” subject matching ambiguity
    - time windows and mailbox folders

#### Node: Check If Reply Received
- **Type / role:** `IF` (`n8n-nodes-base.if`) — routes based on reply status.
- **Configuration choices:**
  - Condition checks `{{ $('Check for Reply Logic').item.json.replyReceived }}` equals `false`.
  - **True branch (no reply)** → AI Follow-up Email Writer  
  - **False branch (reply received)** → Tag as Interested
- **Edge cases / failures:**
  - The condition references another node using `$('Check for Reply Logic').item...`; this can be fragile with multiple items or changed execution context. Prefer `$json.replyReceived` since the flag is on the current item.

#### Node: AI Follow-up Email Writer
- **Type / role:** `LangChain Agent` — generates a short follow-up email body.
- **Configuration choices:**
  - Input text: company, name, original chosen idea.
  - System message: max 60 words, polite, not pushy, return only body text.
  - Uses external model connection from “Google Gemini Chat Model1”.
- **Inputs/outputs:**
  - Main input from IF (no reply path)
  - `ai_languageModel` from Google Gemini Chat Model1
  - Main output to “Send Follow-up Email”
- **Edge cases / failures:**
  - Output is expected at `$json.output` downstream; confirm your agent node outputs there.
  - If the original `chosenIdea` is missing (earlier steps failed), follow-up quality degrades.

#### Node: Google Gemini Chat Model1
- **Type / role:** Gemini chat model provider for follow-up agent.
- **Configuration choices:** default options; uses `googlePalmApi`.
- **Connections:** to AI Follow-up Email Writer via `ai_languageModel`.

#### Node: Send Follow-up Email
- **Type / role:** Outlook — sends follow-up message.
- **Configuration choices:**
  - Subject: `Re: Quick automation idea for {{ $json.company }}`
  - Body: `{{ $json.output }}` + sign-off
  - To: `{{ $json.email }}`
- **Inputs/outputs:** Terminal for no-reply branch.
- **Edge cases / failures:**
  - Not actually threading in the same email conversation unless you set Outlook-specific headers (Message-ID/In-Reply-To) which this node does not show.
  - If `$json.output` is not a plain string, email content may be wrong.

#### Node: Tag as Interested
- **Type / role:** Google Sheets — updates lead status when reply is detected.
- **Configuration choices:**
  - Operation: appendOrUpdate
  - Matching column: **Email**
  - Writes:
    - Email
    - Status = `Interested - Replied`
- **Inputs/outputs:** From IF (reply received branch); terminal.
- **Edge cases / failures:**
  - This assumes the sheet has a `Status` column.
  - If the lead was initially matched/updated by **Name** (earlier node) and later by **Email** (this node), you can end up with inconsistent rows if duplicates exist.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Property Photo Upload Form | Form Trigger | Lead intake (form + file upload) | — | Edit an image; Append or update row in sheet; Automation Ideas Library |  |
| Edit an image | Google Gemini (LangChain) Image | AI image editing based on prompt | Property Photo Upload Form | Send a message |  |
| Send a message | Microsoft Outlook | Send edited image to lead | Edit an image | — |  |
| Append or update row in sheet | Google Sheets | Store lead data (Name-matched) | Property Photo Upload Form | — |  |
| Automation Ideas Library | Set | Add automation ideas + normalize lead fields | Property Photo Upload Form | AI Sales Agent - Idea Picker & Email Writer | ## 🤖 PART 2: AI Outreach\n\nAI analyzes company context, selects relevant automation idea, and generates personalized outreach email automatically. |
| AI Sales Agent - Idea Picker & Email Writer | LangChain Agent | Pick idea + write outreach email (structured) | Automation Ideas Library | Parse AI Output | ## 🤖 PART 2: AI Outreach\n\nAI analyzes company context, selects relevant automation idea, and generates personalized outreach email automatically. |
| Google Gemini Chat Model | Gemini Chat Model (LangChain) | LLM provider for outreach agent | — | AI Sales Agent - Idea Picker & Email Writer (ai_languageModel) | ## 🤖 PART 2: AI Outreach\n\nAI analyzes company context, selects relevant automation idea, and generates personalized outreach email automatically. |
| Structured Output Parser | Structured Output Parser (LangChain) | Enforce JSON schema for agent output | — | AI Sales Agent - Idea Picker & Email Writer (ai_outputParser) | ## 🤖 PART 2: AI Outreach\n\nAI analyzes company context, selects relevant automation idea, and generates personalized outreach email automatically. |
| Parse AI Output | Code | Flatten agent output + carry lead fields | AI Sales Agent - Idea Picker & Email Writer | Send Personalized Outreach Email | ## 📋 PART 1: Lead Capture\n\nCaptures leads through form submission, processes images with AI, and stores contact info in Google Sheets. |
| Send Personalized Outreach Email | Microsoft Outlook | Send AI-personalized outreach + booking link | Parse AI Output | Wait 48 Hours | ## 📬 PART 3: Follow-up System\n\nWaits 48 hours, checks for replies, sends AI-generated follow-up if no response, or tags lead as interested if they replied. |
| Wait 48 Hours | Wait | Delay before follow-up | Send Personalized Outreach Email | Check for Reply Logic | ## 📬 PART 3: Follow-up System\n\nWaits 48 hours, checks for replies, sends AI-generated follow-up if no response, or tags lead as interested if they replied. |
| Check for Reply Logic | Code | Placeholder reply detection (currently always false) | Wait 48 Hours | Check If Reply Received | ## 📬 PART 3: Follow-up System\n\nWaits 48 hours, checks for replies, sends AI-generated follow-up if no response, or tags lead as interested if they replied. |
| Check If Reply Received | IF | Branch: no reply vs replied | Check for Reply Logic | AI Follow-up Email Writer; Tag as Interested | ## 📬 PART 3: Follow-up System\n\nWaits 48 hours, checks for replies, sends AI-generated follow-up if no response, or tags lead as interested if they replied. |
| AI Follow-up Email Writer | LangChain Agent | Generate short follow-up body | Check If Reply Received (no-reply path) | Send Follow-up Email | ## 📬 PART 3: Follow-up System\n\nWaits 48 hours, checks for replies, sends AI-generated follow-up if no response, or tags lead as interested if they replied. |
| Google Gemini Chat Model1 | Gemini Chat Model (LangChain) | LLM provider for follow-up agent | — | AI Follow-up Email Writer (ai_languageModel) | ## 📬 PART 3: Follow-up System\n\nWaits 48 hours, checks for replies, sends AI-generated follow-up if no response, or tags lead as interested if they replied. |
| Send Follow-up Email | Microsoft Outlook | Send follow-up email | AI Follow-up Email Writer | — | ## 📬 PART 3: Follow-up System\n\nWaits 48 hours, checks for replies, sends AI-generated follow-up if no response, or tags lead as interested if they replied. |
| Tag as Interested | Google Sheets | Update lead status when reply received | Check If Reply Received (replied path) | — | ## 📬 PART 3: Follow-up System\n\nWaits 48 hours, checks for replies, sends AI-generated follow-up if no response, or tags lead as interested if they replied. |
| 📋 PART 1: Lead Capture | Sticky Note | Comment block label | — | — | ## 📋 PART 1: Lead Capture\n\nCaptures leads through form submission, processes images with AI, and stores contact info in Google Sheets. |
| 🤖 PART 2: AI Outreach | Sticky Note | Comment block label | — | — | ## 🤖 PART 2: AI Outreach\n\nAI analyzes company context, selects relevant automation idea, and generates personalized outreach email automatically. |
| 📬 PART 3: Follow-up System | Sticky Note | Comment block label | — | — | ## 📬 PART 3: Follow-up System\n\nWaits 48 hours, checks for replies, sends AI-generated follow-up if no response, or tags lead as interested if they replied. |
| 🔄 How It Works | Sticky Note | Global explanation + setup list | — | — | ## 🔄 How It Works\n\n**Step 1:** Lead fills form → Image edited + contact saved\n**Step 2:** AI picks automation idea → Writes personalized email\n**Step 3:** Email sent with calendar link\n**Step 4:** Wait 48 hours\n**Step 5:** Check for reply:\n  • No reply → AI writes follow-up → Send\n  • Reply received → Tag as \"Interested\" in sheet\n\n**Result:** Automated lead qualification + nurturing at scale\n\n## 📖 Setup Instructions\n\n1. Add Google Gemini API credentials\n2. Connect Microsoft Outlook account\n3. Connect Google Sheets (create a sheet with columns: Name, Company, Email, Time, Status)\n4. Update calendar booking link in outreach email (currently: https://cal.com/quartersmart/intro)\n5. Customize automation ideas list in the \"📚 Automation Ideas Library\" node\n6. Update email signatures with your name and company\n7. Test with a sample form submission |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named **“AI Agent Outbound + Follow-up”** (or your preferred name).

2. **Add node: Form Trigger**
   - Node type: **Form Trigger**
   - Path: `magic-editor`
   - Title: `Magic Editor`
   - Description: “Upload any image… receive it in an email!”
   - Fields (ensure labels match exactly, including punctuation):
     - File field label **Image** (required, single file)
     - Text field label **Description:** (required)
     - Text field label **Email:** (required)
     - Text field label **Name** (required)
     - Text field label **Company Name?** (required)

3. **Add node: Google Gemini Image → Edit**
   - Node type: **Google Gemini (LangChain) → Image**
   - Operation: **Edit**
   - Image input: binary property name **`Image`**
   - Prompt:
     - `BASED OFF OF THIS DESCRIPTION, EDIT THE IMAGE: {{ $json['Description:'] }}`
     - Add the “keep camera angle…” line
   - Credentials: create/select **Google Gemini/PaLM API** credential.

4. **Add node: Microsoft Outlook → Send Email (edited image delivery)**
   - Node type: **Microsoft Outlook**
   - Operation: send message/email
   - To: `{{ $('Property Photo Upload Form').item.json['Email:'] }}`
   - Subject: `Image Is ready!`
   - Body: reference Name and Company via the form node expressions.
   - Attachments: add one attachment using binary property **`edited`** (adjust if your Gemini node outputs a different binary name).
   - Credentials: connect **Microsoft Outlook OAuth2**.

5. **Add node: Google Sheets → Append or update row**
   - Node type: **Google Sheets**
   - Spreadsheet: create/select your spreadsheet (e.g., “Warm Leads”)
   - Sheet: `Sheet1`
   - Operation: **Append or update**
   - Ensure the sheet has columns: **Name, Company, Email, Time, Status**
   - Map:
     - Name = form Name
     - Company = form Company Name?
     - Email = form Email:
     - Time = form submittedAt
   - Matching column: **Name** (or change to **Email** to avoid duplicates).

6. **Add node: Set (Automation Ideas Library)**
   - Node type: **Set**
   - Add field `automationIdeas` (string) with your list (multi-line).
   - Add normalized fields:
     - `name = {{$json.Name}}`
     - `email = {{$json['Email:']}}`
     - `company = {{$json['Company Name?']}}`
   - Enable **Include other fields**.

7. **Add node: Gemini Chat Model (for outreach)**
   - Node type: **LangChain → Chat Model → Google Gemini**
   - Credentials: same Gemini credential.

8. **Add node: Structured Output Parser**
   - Node type: **LangChain → Output Parser → Structured**
   - Schema with `chosenIdea` and `emailBody` as strings.

9. **Add node: LangChain Agent (Outreach)**
   - Node type: **LangChain Agent**
   - Prompt/input text containing company, name, and `automationIdeas`.
   - System message: enforce <150 words and JSON output format.
   - Connect:
     - Chat Model node → Agent via **ai_languageModel**
     - Structured Output Parser → Agent via **ai_outputParser**

10. **Add node: Code (Parse AI Output)**
   - Node type: **Code**
   - Read agent output and return JSON with:
     - `chosenIdea`, `emailBody`
     - plus `name/email/company` from the Set node (or from current item).

11. **Add node: Microsoft Outlook (Send Personalized Outreach Email)**
   - To: `{{ $json.email }}`
   - Subject: `Quick automation idea for {{ $json.company }}`
   - Body: include `{{ $json.emailBody }}` and your booking link (e.g. `https://cal.com/quartersmart/intro`)

12. **Add node: Wait**
   - Node type: **Wait**
   - Duration: **48 hours**

13. **Add node: Code (Check for Reply Logic)**
   - Initially replicate placeholder: add `replyReceived: false`
   - (Later replace with real Outlook reply detection.)

14. **Add node: IF (Check If Reply Received)**
   - Condition: `replyReceived is false`
   - True output (no reply) → follow-up agent
   - False output (replied) → Tag as Interested

15. **Add node: Gemini Chat Model (for follow-up)**
   - Node type: **LangChain → Chat Model → Google Gemini**
   - Credentials: Gemini.

16. **Add node: LangChain Agent (AI Follow-up Email Writer)**
   - Input includes company, name, and chosenIdea.
   - System message: max 60 words; return only body text.
   - Connect follow-up Gemini model via **ai_languageModel**.

17. **Add node: Outlook (Send Follow-up Email)**
   - To: `{{ $json.email }}`
   - Subject: `Re: Quick automation idea for {{ $json.company }}`
   - Body: `{{ $json.output }}` + signature

18. **Add node: Google Sheets (Tag as Interested)**
   - Operation: appendOrUpdate
   - Match on **Email**
   - Set `Status = Interested - Replied`

19. **Connect nodes in this order**
   - Form Trigger → (3 branches)
     - → Edit an image → Send a message
     - → Append/update row in sheet
     - → Automation Ideas Library → Outreach Agent → Parse AI Output → Send Personalized Outreach Email → Wait → Check for Reply Logic → IF
       - IF (no reply) → Follow-up Agent → Send Follow-up Email
       - IF (replied) → Tag as Interested

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques. | Disclaimer (provided by requester) |
| Update calendar booking link in outreach email (currently: https://cal.com/quartersmart/intro) | From sticky note “🔄 How It Works” |
| Create a sheet with columns: Name, Company, Email, Time, Status | From sticky note “🔄 How It Works” |
| Reply-check is placeholder (always `replyReceived: false`); production version should query Outlook to detect replies | Implement implied by “Check for Reply Logic” node comments |