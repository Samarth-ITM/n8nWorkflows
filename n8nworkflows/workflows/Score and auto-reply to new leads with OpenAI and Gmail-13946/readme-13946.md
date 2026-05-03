Score and auto-reply to new leads with OpenAI and Gmail

https://n8nworkflows.xyz/workflows/score-and-auto-reply-to-new-leads-with-openai-and-gmail-13946


# Score and auto-reply to new leads with OpenAI and Gmail

## 1. Workflow Overview

This workflow captures inbound leads through an n8n form, enriches the submission with business-specific settings, sends the lead details to OpenAI for qualification and email drafting, logs the result to Google Sheets, and automatically replies via Gmail based on the lead’s quality tier.

Typical use cases:
- Small businesses that want immediate lead follow-up
- Agencies qualifying project requests automatically
- Freelancers prioritizing high-value inquiries
- Teams wanting a lightweight CRM-style intake + scoring pipeline without a full CRM

### 1.1 Input Reception and Business Context Injection
The workflow starts with a hosted n8n form that collects lead details such as name, email, budget, and project description. A Set node then injects static business context like the business name, owner name, owner email, description, and booking link.

### 1.2 AI Lead Qualification and Email Generation
The enriched lead data is sent to OpenAI. The prompt instructs the model to return strict JSON containing a numerical score, a tier classification, internal reasoning, and a personalized HTML email draft.

### 1.3 Response Parsing and Data Normalization
A Code node extracts and parses the AI output, normalizes the fields, and assembles a clean payload containing both original lead information and AI-generated values.

### 1.4 Logging and Tier-Based Routing
The parsed result is sent in parallel to Google Sheets for recordkeeping and to a Switch node that routes the item into Hot, Warm, or Cold branches based on the AI-generated tier.

### 1.5 Automated Email Response
Each route ends in a Gmail node that sends the AI-generated response email to the lead, preserving tier-specific messaging.

---

## 2. Block-by-Block Analysis

## 2.1 Input Reception and Business Context Injection

**Overview:**  
This block collects the lead submission and adds static business metadata required by the AI prompt and outbound email generation. It establishes the base dataset used throughout the rest of the workflow.

**Nodes Involved:**  
- Lead Intake Form
- Configure Your Settings

### Node: Lead Intake Form
- **Type and technical role:** `n8n-nodes-base.formTrigger`  
  Entry-point trigger node that exposes an n8n-hosted form and starts the workflow on submission.
- **Configuration choices:**
  - Form title: `Contact Us`
  - Description: “Tell us about your project and we'll get back to you within minutes.”
  - Fields:
    - Name — required
    - Email — required
    - Company — optional
    - Budget Range — required
    - Project Description — required
    - Referral Source — optional
- **Key expressions or variables used:**  
  None in the node itself. It outputs form fields as JSON properties, including keys with spaces such as `Budget Range` and `Project Description`.
- **Input and output connections:**
  - Input: none
  - Output: Configure Your Settings
- **Version-specific requirements:**  
  Uses type version `2.5`. The Form Trigger must be supported by the n8n instance and exposed appropriately if public access is required.
- **Edge cases or potential failure types:**
  - Form not publicly reachable
  - Required fields left blank if client-side validation is bypassed
  - Field naming changes breaking downstream expressions
  - Email format is not strongly validated beyond form behavior unless additional checks are added
- **Sub-workflow reference:**  
  None

### Node: Configure Your Settings
- **Type and technical role:** `n8n-nodes-base.set`  
  Adds static configuration values to each incoming form submission.
- **Configuration choices:**
  - `businessName`: `Your Business Name`
  - `ownerName`: `Your Name`
  - `ownerEmail`: `user@example.com`
  - `calendarLink`: `https://calendly.com/your-link`
  - `businessDescription`: `We help businesses automate their workflows and grow faster.`
  - `includeOtherFields: true` keeps all original form fields
- **Key expressions or variables used:**  
  No expressions; static assignments only.
- **Input and output connections:**
  - Input: Lead Intake Form
  - Output: Score Lead & Draft Reply
- **Version-specific requirements:**  
  Uses Set node type version `3.4`.
- **Edge cases or potential failure types:**
  - Placeholder values left unchanged, causing poor AI responses and incorrect sender identity
  - Missing or invalid calendar URL
  - If `includeOtherFields` were disabled, downstream prompt fields would break
- **Sub-workflow reference:**  
  None

---

## 2.2 AI Lead Qualification and Email Generation

**Overview:**  
This block sends the full lead and business context to OpenAI, asking it to score the lead, assign a tier, explain the reasoning, and write a personalized HTML email reply.

**Nodes Involved:**  
- Score Lead & Draft Reply

### Node: Score Lead & Draft Reply
- **Type and technical role:** `@n8n/n8n-nodes-langchain.openAi`  
  OpenAI chat/responses node used to generate structured lead qualification output.
- **Configuration choices:**
  - Model: `gpt-4o-mini`
  - Uses a system instruction that:
    - Defines the assistant as a lead qualification AI
    - Requires output as valid JSON only
    - Enforces exact response schema:
      - `score`
      - `tier`
      - `reasoning`
      - `emailSubject`
      - `emailBody`
    - Defines scoring criteria:
      - Budget alignment
      - Project clarity
      - Overall fit
    - Defines tier ranges:
      - Hot: 70–100
      - Warm: 40–69
      - Cold: 1–39
    - Specifies email style and HTML formatting rules
  - User content is dynamically built from form data and settings
- **Key expressions or variables used:**
  - `{{ $json.businessName }}`
  - `{{ $json.businessDescription }}`
  - `{{ $json.ownerName }}`
  - `{{ $json.calendarLink }}`
  - `{{ $json.Name }}`
  - `{{ $json.Email }}`
  - `{{ $json.Company }}`
  - `{{ $json["Budget Range"] }}`
  - `{{ $json["Project Description"] }}`
  - `{{ $json["Referral Source"] }}`
- **Input and output connections:**
  - Input: Configure Your Settings
  - Output: Parse AI Response
- **Version-specific requirements:**  
  Uses type version `2.1` of the LangChain OpenAI node. Behavior depends on the installed n8n AI package and available OpenAI integration features.
- **Edge cases or potential failure types:**
  - Invalid or missing OpenAI credentials
  - Model access restrictions
  - Non-JSON or malformed JSON response despite prompt constraints
  - Timeout or API quota errors
  - Prompt changes causing schema drift
  - Long or unusual lead descriptions increasing token usage
- **Sub-workflow reference:**  
  None

---

## 2.3 Response Parsing and Data Normalization

**Overview:**  
This block extracts the model output, attempts to isolate JSON even if wrapped in extra formatting, parses it, and rebuilds a clean object used for routing, logging, and replying.

**Nodes Involved:**  
- Parse AI Response

### Node: Parse AI Response
- **Type and technical role:** `n8n-nodes-base.code`  
  JavaScript transformation node that parses the OpenAI response into structured fields.
- **Configuration choices:**
  - Mode: `runOnceForEachItem`
  - Parsing logic:
    1. Detects AI text in one of several possible fields:
       - `output[0].content[0].text`
       - `text`
       - `content`
       - fallback to stringified JSON
    2. Tries to extract JSON from:
       - fenced ```json blocks
       - or the first `{ ... }` object found
    3. Runs `JSON.parse()`
    4. Pulls original input data from `$('Configure Your Settings').first().json`
    5. Returns normalized fields including score, tier, reasoning, email subject/body, and timestamp
- **Key expressions or variables used:**
  - `$('Configure Your Settings').first().json`
  - Output fields:
    - `leadName`
    - `leadEmail`
    - `company`
    - `budgetRange`
    - `projectDescription`
    - `referralSource`
    - `businessName`
    - `ownerName`
    - `ownerEmail`
    - `calendarLink`
    - `score`
    - `tier`
    - `reasoning`
    - `emailSubject`
    - `emailBody`
    - `timestamp`
- **Input and output connections:**
  - Input: Score Lead & Draft Reply
  - Outputs:
    - Route by Lead Quality
    - Log Lead with Score
- **Version-specific requirements:**  
  Uses Code node type version `2`.
- **Edge cases or potential failure types:**
  - `JSON.parse` failure if the model returns invalid JSON
  - Response format mismatch if OpenAI node output structure changes
  - Empty `output[0].content[0].text`
  - Referencing `Configure Your Settings` by name means renaming that node may break the code
  - `.first()` assumes only one relevant upstream item context
- **Sub-workflow reference:**  
  None

---

## 2.4 Logging and Tier-Based Routing

**Overview:**  
This block splits the normalized payload into two independent actions: persistence in Google Sheets and branching logic for email delivery based on lead quality.

**Nodes Involved:**  
- Log Lead with Score
- Route by Lead Quality

### Node: Log Lead with Score
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends each processed lead to a Google Sheet as a historical log.
- **Configuration choices:**
  - Operation: `append`
  - Mapping mode: `autoMapInputData`
  - `documentId`: currently blank placeholder, intended to be supplied via Google Sheet URL or selected file
  - `sheetName`: blank placeholder, must be selected
- **Key expressions or variables used:**  
  No custom expressions visible; auto-maps incoming JSON keys to matching headers.
- **Input and output connections:**
  - Input: Parse AI Response
  - Output: none
- **Version-specific requirements:**  
  Uses Google Sheets node type version `4.7`.
- **Edge cases or potential failure types:**
  - Missing Google credentials
  - Empty document ID or sheet name
  - Header mismatch preventing correct mapping
  - Insufficient permissions on the spreadsheet
  - HTML in `emailBody` making the sheet harder to inspect, though still storable
- **Sub-workflow reference:**  
  None

### Node: Route by Lead Quality
- **Type and technical role:** `n8n-nodes-base.switch`  
  Routes each lead into one of three outputs based on the exact `tier` string.
- **Configuration choices:**
  - Output 1 renamed to `Hot` when `{{$json.tier}} === "Hot"`
  - Output 2 renamed to `Warm` when `{{$json.tier}} === "Warm"`
  - Output 3 renamed to `Cold` when `{{$json.tier}} === "Cold"`
  - Fallback output enabled as `extra`
  - Strict type validation
  - Case sensitive matching enabled
- **Key expressions or variables used:**
  - `={{ $json.tier }}`
- **Input and output connections:**
  - Input: Parse AI Response
  - Outputs:
    - Hot → Reply to Hot Lead
    - Warm → Reply to Warm Lead
    - Cold → Reply to Cold Lead
    - Fallback `extra` is configured but not connected
- **Version-specific requirements:**  
  Uses Switch node type version `3.4`.
- **Edge cases or potential failure types:**
  - Any unexpected tier value like `hot`, `HOT`, `Qualified`, or empty string goes to fallback and is effectively dropped because fallback is unconnected
  - Trailing spaces in AI output would fail exact matching
- **Sub-workflow reference:**  
  None

---

## 2.5 Automated Email Response

**Overview:**  
This block sends the personalized HTML reply to the lead via Gmail. The three Gmail nodes are functionally similar and exist mainly to preserve distinct tier branches.

**Nodes Involved:**  
- Reply to Hot Lead
- Reply to Warm Lead
- Reply to Cold Lead

### Node: Reply to Hot Lead
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends the AI-generated email to leads classified as Hot.
- **Configuration choices:**
  - Recipient: `={{ $json.leadEmail }}`
  - Subject: `={{ $json.emailSubject }}`
  - Message body: `={{ $json.emailBody }}`
  - Sender name: `={{ $json.ownerName }}`
- **Key expressions or variables used:**
  - `{{ $json.leadEmail }}`
  - `{{ $json.emailSubject }}`
  - `{{ $json.emailBody }}`
  - `{{ $json.ownerName }}`
- **Input and output connections:**
  - Input: Route by Lead Quality (Hot)
  - Output: none
- **Version-specific requirements:**  
  Uses Gmail node type version `2.2`.
- **Edge cases or potential failure types:**
  - Gmail OAuth2 not configured
  - Sending limits or rate limits
  - Invalid recipient address
  - HTML formatting rendering unexpectedly in some email clients
  - From account may differ from `ownerName` branding expectations
- **Sub-workflow reference:**  
  None

### Node: Reply to Warm Lead
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends the AI-generated email to leads classified as Warm.
- **Configuration choices:**  
  Same as the Hot branch:
  - Recipient from `leadEmail`
  - Subject from `emailSubject`
  - HTML body from `emailBody`
  - Sender name from `ownerName`
- **Key expressions or variables used:**  
  Same as above
- **Input and output connections:**
  - Input: Route by Lead Quality (Warm)
  - Output: none
- **Version-specific requirements:**  
  Gmail node version `2.2`.
- **Edge cases or potential failure types:**  
  Same as other Gmail nodes.
- **Sub-workflow reference:**  
  None

### Node: Reply to Cold Lead
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends the AI-generated email to leads classified as Cold.
- **Configuration choices:**  
  Same as the other Gmail branches.
- **Key expressions or variables used:**  
  Same as above
- **Input and output connections:**
  - Input: Route by Lead Quality (Cold)
  - Output: none
- **Version-specific requirements:**  
  Gmail node version `2.2`.
- **Edge cases or potential failure types:**  
  Same as other Gmail nodes.
- **Sub-workflow reference:**  
  None

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Lead Intake Form | n8n-nodes-base.formTrigger | Receives lead submissions through a hosted form |  | Configure Your Settings | ## 1. Trigger & Configuration<br><br>The **Lead Intake Form** creates a web form that captures lead information. When submitted, data flows to **Configure Your Settings** where your business details are added.<br><br>**Setup:** Update the Set node with your business name, email, calendar link, and a brief business description. |
| Configure Your Settings | n8n-nodes-base.set | Adds business metadata to each lead item | Lead Intake Form | Score Lead & Draft Reply | ## 1. Trigger & Configuration<br><br>The **Lead Intake Form** creates a web form that captures lead information. When submitted, data flows to **Configure Your Settings** where your business details are added.<br><br>**Setup:** Update the Set node with your business name, email, calendar link, and a brief business description. |
| Score Lead & Draft Reply | @n8n/n8n-nodes-langchain.openAi | Scores the lead and drafts a personalized response | Configure Your Settings | Parse AI Response | ## 2. AI Lead Scoring<br><br>OpenAI analyzes each lead based on budget, project clarity, and fit. It returns a score (1-100), tier (Hot/Warm/Cold), reasoning, and a fully personalized email draft.<br><br>**Customize:** Edit the system prompt in the OpenAI node to match your industry and ideal customer profile. |
| Parse AI Response | n8n-nodes-base.code | Parses AI output into normalized structured fields | Score Lead & Draft Reply | Route by Lead Quality; Log Lead with Score | ## 3. Smart Auto-Reply<br><br>Leads are routed by quality tier. Each path sends a tailored email:<br>- **Hot (70+):** Enthusiastic reply with calendar booking link<br>- **Warm (40-69):** Helpful response with next steps<br>- **Cold (0-39):** Polite acknowledgment<br><br>**Customize:** Edit each Gmail node to add CC recipients, attachments, or custom behavior per tier. |
| Route by Lead Quality | n8n-nodes-base.switch | Sends leads to Hot, Warm, or Cold email branches | Parse AI Response | Reply to Hot Lead; Reply to Warm Lead; Reply to Cold Lead | ## 3. Smart Auto-Reply<br><br>Leads are routed by quality tier. Each path sends a tailored email:<br>- **Hot (70+):** Enthusiastic reply with calendar booking link<br>- **Warm (40-69):** Helpful response with next steps<br>- **Cold (0-39):** Polite acknowledgment<br><br>**Customize:** Edit each Gmail node to add CC recipients, attachments, or custom behavior per tier. |
| Reply to Hot Lead | n8n-nodes-base.gmail | Sends email to Hot leads | Route by Lead Quality |  | ## 4. Lead Logging<br>Every lead is logged to Google Sheets with score, tier, reasoning, and all form data.<br><br>**Setup:** Create a Google Sheet and add these exact column headers in Row 1:<br><br>`timestamp` · `leadName` · `leadEmail` · `company` · `budgetRange` · `projectDescription` · `referralSource` · `score` · `tier` · `reasoning` · `emailSubject` · `emailBody`<br><br>Then paste your Sheet URL into the Log Lead with Score node. |
| Reply to Warm Lead | n8n-nodes-base.gmail | Sends email to Warm leads | Route by Lead Quality |  | ## 4. Lead Logging<br>Every lead is logged to Google Sheets with score, tier, reasoning, and all form data.<br><br>**Setup:** Create a Google Sheet and add these exact column headers in Row 1:<br><br>`timestamp` · `leadName` · `leadEmail` · `company` · `budgetRange` · `projectDescription` · `referralSource` · `score` · `tier` · `reasoning` · `emailSubject` · `emailBody`<br><br>Then paste your Sheet URL into the Log Lead with Score node. |
| Reply to Cold Lead | n8n-nodes-base.gmail | Sends email to Cold leads | Route by Lead Quality |  | ## 4. Lead Logging<br>Every lead is logged to Google Sheets with score, tier, reasoning, and all form data.<br><br>**Setup:** Create a Google Sheet and add these exact column headers in Row 1:<br><br>`timestamp` · `leadName` · `leadEmail` · `company` · `budgetRange` · `projectDescription` · `referralSource` · `score` · `tier` · `reasoning` · `emailSubject` · `emailBody`<br><br>Then paste your Sheet URL into the Log Lead with Score node. |
| Log Lead with Score | n8n-nodes-base.googleSheets | Appends processed lead data to Google Sheets | Parse AI Response |  | ## 4. Lead Logging<br>Every lead is logged to Google Sheets with score, tier, reasoning, and all form data.<br><br>**Setup:** Create a Google Sheet and add these exact column headers in Row 1:<br><br>`timestamp` · `leadName` · `leadEmail` · `company` · `budgetRange` · `projectDescription` · `referralSource` · `score` · `tier` · `reasoning` · `emailSubject` · `emailBody`<br><br>Then paste your Sheet URL into the Log Lead with Score node. |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation and setup guidance |  |  | ## Score and Auto-Reply to New Leads with OpenAI and Gmail<br><br>### Who is this for<br>Small business owners, freelancers, and agencies who receive inbound leads and want instant, personalized follow-up.<br><br>### What it does<br>1. Scores the lead 1-100 using OpenAI<br>2. Classifies as Hot, Warm, or Cold<br>3. Sends a personalized email reply<br>4. Logs everything to Google Sheets<br><br>### How to set up<br>1. Fill in the **Configure Your Settings** node with your business name, email, calendar link, and description<br>2. Connect your Gmail and Google Sheets credentials<br>3. Add your OpenAI API key<br>4. Create a Google Sheet with these column headers in Row 1:<br>   `timestamp` · `leadName` · `leadEmail` · `company` · `budgetRange` · `projectDescription` · `referralSource` · `score` · `tier` · `reasoning` · `emailSubject` · `emailBody`<br>5. Paste your Sheet URL into the **Log Lead with Score** node<br>6. Activate the workflow and share the form URL<br><br>### Requirements<br>- n8n account (cloud or self-hosted)<br>- OpenAI API key<br>- Gmail account (OAuth2)<br>- Google Sheets (one blank sheet)<br><br>### How to customize<br>- Edit the OpenAI system prompt to adjust scoring criteria for your industry<br>- Modify the tier thresholds (currently Hot 70+, Warm 40-69, Cold 0-39)<br>- Add or remove form fields to match your intake needs |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas documentation for trigger/config block |  |  |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas documentation for AI block |  |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas documentation for reply block |  |  |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas documentation for logging block |  |  |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: `Score and auto-reply to new leads with OpenAI and Gmail`.

2. **Add a Form Trigger node**
   - Node type: `Form Trigger`
   - Set form title to `Contact Us`
   - Set description to: `Tell us about your project and we'll get back to you within minutes.`
   - Add the following fields:
     1. `Name` — required, placeholder `Your full name`
     2. `Email` — required, placeholder `your@email.com`
     3. `Company` — optional, placeholder `Your company name (optional)`
     4. `Budget Range` — required, placeholder `e.g. $1,000 - $5,000`
     5. `Project Description` — required, placeholder `Describe your project, goals, and timeline`
     6. `Referral Source` — optional, placeholder `How did you hear about us?`

3. **Add a Set node after the form**
   - Name it `Configure Your Settings`
   - Enable keeping the incoming fields (`Include Other Input Fields` / equivalent option)
   - Add these string fields:
     - `businessName`
     - `ownerName`
     - `ownerEmail`
     - `calendarLink`
     - `businessDescription`
   - Fill them with your real business values, for example:
     - business name
     - sender/owner display name
     - owner email
     - Calendly or booking URL
     - short business description
   - Connect: `Lead Intake Form` → `Configure Your Settings`

4. **Add an OpenAI node**
   - Node type: `OpenAI` from the n8n AI/LangChain nodes
   - Name it `Score Lead & Draft Reply`
   - Connect OpenAI credentials using your OpenAI API key
   - Select model `gpt-4o-mini`
   - Add a system message with these instructions:
     - The AI is a lead qualification assistant
     - It must return only valid JSON
     - Required JSON keys:
       - `score`
       - `tier`
       - `reasoning`
       - `emailSubject`
       - `emailBody`
     - Include scoring criteria:
       - budget alignment
       - project clarity
       - fit for business
     - Tier thresholds:
       - Hot: 70–100
       - Warm: 40–69
       - Cold: 1–39
     - Email requirements:
       - address the lead by first name
       - reference their submission details
       - 3–5 short paragraphs
       - include calendar link for Hot leads
       - clean HTML with inline styles
       - sign off with owner name
   - Add a user message containing interpolated lead and business context:
     - Business name
     - Business description
     - Owner name
     - Calendar link
     - Lead name
     - Lead email
     - Company
     - Budget Range
     - Project Description
     - Referral Source
   - Use expressions exactly against the Set node output fields, such as:
     - `{{$json.businessName}}`
     - `{{$json["Budget Range"]}}`
     - `{{$json["Project Description"]}}`
   - Connect: `Configure Your Settings` → `Score Lead & Draft Reply`

5. **Add a Code node**
   - Name it `Parse AI Response`
   - Set mode to `Run Once for Each Item`
   - Paste JavaScript logic that:
     - reads the text from the OpenAI response
     - extracts JSON from plain text or fenced code blocks
     - parses it
     - reads original form/config data from `Configure Your Settings`
     - outputs a normalized JSON object
   - The output fields should be:
     - `leadName`
     - `leadEmail`
     - `company`
     - `budgetRange`
     - `projectDescription`
     - `referralSource`
     - `businessName`
     - `ownerName`
     - `ownerEmail`
     - `calendarLink`
     - `score`
     - `tier`
     - `reasoning`
     - `emailSubject`
     - `emailBody`
     - `timestamp`
   - Important implementation detail:
     - If you reference the Set node by name in code, use the exact node name `Configure Your Settings`
   - Connect: `Score Lead & Draft Reply` → `Parse AI Response`

6. **Add a Google Sheets node**
   - Name it `Log Lead with Score`
   - Node type: `Google Sheets`
   - Operation: `Append`
   - Connect Google Sheets credentials
   - Select or paste the spreadsheet URL/ID
   - Select the sheet tab name
   - Use automatic column mapping
   - In the target sheet, create Row 1 headers exactly as:
     - `timestamp`
     - `leadName`
     - `leadEmail`
     - `company`
     - `budgetRange`
     - `projectDescription`
     - `referralSource`
     - `score`
     - `tier`
     - `reasoning`
     - `emailSubject`
     - `emailBody`
   - Connect: `Parse AI Response` → `Log Lead with Score`

7. **Add a Switch node**
   - Name it `Route by Lead Quality`
   - Configure 3 outputs with renamed branches:
     - `Hot`
     - `Warm`
     - `Cold`
   - Add conditions:
     - Output `Hot` when `{{$json.tier}}` equals `Hot`
     - Output `Warm` when `{{$json.tier}}` equals `Warm`
     - Output `Cold` when `{{$json.tier}}` equals `Cold`
   - Keep strict matching enabled if you want exact control
   - Optionally keep a fallback output, but ideally connect it to an error-handling branch
   - Connect: `Parse AI Response` → `Route by Lead Quality`

8. **Add three Gmail nodes**
   - Create:
     - `Reply to Hot Lead`
     - `Reply to Warm Lead`
     - `Reply to Cold Lead`
   - For each:
     - Node type: `Gmail`
     - Connect Gmail OAuth2 credentials
     - Recipient: `{{$json.leadEmail}}`
     - Subject: `{{$json.emailSubject}}`
     - Message body: `{{$json.emailBody}}`
     - Sender name: `{{$json.ownerName}}`
   - Connect:
     - `Route by Lead Quality (Hot)` → `Reply to Hot Lead`
     - `Route by Lead Quality (Warm)` → `Reply to Warm Lead`
     - `Route by Lead Quality (Cold)` → `Reply to Cold Lead`

9. **Optionally add sticky notes**
   - Add a general note with setup requirements and sheet header list
   - Add block notes for:
     - Trigger & Configuration
     - AI Lead Scoring
     - Smart Auto-Reply
     - Lead Logging

10. **Configure credentials**
   - **OpenAI**
     - Add API key with access to `gpt-4o-mini`
   - **Gmail**
     - Use OAuth2
     - Ensure send permissions are granted
   - **Google Sheets**
     - Use Google OAuth2 or service-account-style access depending on your environment
     - Ensure the selected spreadsheet is editable by the credential identity

11. **Test the workflow**
   - Submit the form with sample lead data
   - Confirm:
     - OpenAI returns valid JSON
     - Code node parses correctly
     - Row is appended to Google Sheets
     - The correct Gmail branch is triggered
     - Email renders properly as HTML

12. **Activate the workflow**
   - Once validated, activate it
   - Share the hosted form URL with prospective leads

### Recommended hardening improvements when rebuilding
- Add a fallback branch from the Switch node for unexpected AI tiers
- Add validation for `leadEmail`
- Add try/catch logic in the Code node to prevent total failure on malformed model output
- Normalize tier values with trimming and capitalization before routing
- Consider logging Gmail send failures separately

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Score and Auto-Reply to New Leads with OpenAI and Gmail | Workflow title |
| Who this is for: Small business owners, freelancers, and agencies who receive inbound leads and want instant, personalized follow-up. | General usage context |
| What it does: scores leads, classifies Hot/Warm/Cold, sends personalized email replies, and logs to Google Sheets. | General workflow behavior |
| Setup note: Fill in the Configure Your Settings node with your business name, email, calendar link, and description. | Internal setup guidance |
| Setup note: Connect Gmail and Google Sheets credentials and add your OpenAI API key. | Credential requirements |
| Setup note: Create a Google Sheet with the required headers before using the logging node. | Google Sheets preparation |
| Activation note: Activate the workflow and share the form URL. | Deployment guidance |
| Customization note: Edit the OpenAI system prompt to adjust scoring criteria for your industry. | Prompt customization |
| Customization note: Modify the tier thresholds if your qualification model differs. | Business logic customization |
| Customization note: Add or remove intake form fields to match your lead capture process. | Form customization |