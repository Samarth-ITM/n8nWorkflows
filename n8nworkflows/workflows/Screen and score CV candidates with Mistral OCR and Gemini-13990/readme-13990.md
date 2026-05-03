Screen and score CV candidates with Mistral OCR and Gemini

https://n8nworkflows.xyz/workflows/screen-and-score-cv-candidates-with-mistral-ocr-and-gemini-13990


# Screen and score CV candidates with Mistral OCR and Gemini

# 1. Workflow Overview

This workflow automates inbound CV screening for an **AI Engineer** role at **AppStoneLab Technologies**. It listens for incoming application emails, extracts CV text from the attached PDF using **Mistral OCR**, evaluates the candidate against a fixed job description using **Google Gemini**, parses the AI result into structured fields, and then routes the applicant into either a **shortlist path** or a **rejection path**.

Its main use case is high-volume early-stage candidate triage where resumes are submitted by email. It is especially useful because it handles both **digital PDFs and scanned PDFs**, preserves the original attachment, and automatically sends follow-up emails to both HR and candidates.

## 1.1 Inbox Monitoring and CV Intake
The workflow starts when a new email arrives in the monitored mailbox. It receives the email in resolved format and carries the attached resume forward as binary data.

## 1.2 OCR-Based Resume Extraction
The first processing block sends the CV attachment to Mistral OCR so that resume content becomes machine-readable text.

## 1.3 AI-Based Candidate Evaluation
The extracted CV text is embedded into a prompt containing the target role’s job description. Gemini evaluates fit and returns a structured JSON result with score, decision, summary, and skills.

## 1.4 Response Parsing and Data Normalization
A Code node cleans the AI output, parses JSON, extracts sender metadata from the original email, and restores binary attachment access for downstream email nodes.

## 1.5 Decision Routing
An IF node checks whether the AI decision equals `shortlisted`.

## 1.6 Shortlist Handling
If shortlisted, the workflow first notifies HR with candidate details and the original CV attached, then sends an interview invitation to the candidate.

## 1.7 Rejection Handling
If rejected, the workflow sends a polite decline email to the candidate.

---

# 2. Block-by-Block Analysis

## Block 1 — Inbox Monitoring and CV Intake

### Overview
This block is the workflow entry point. It watches an IMAP inbox for newly received emails and passes full email data plus attachment binaries into the automation.

### Nodes Involved
- Watch Inbox for CV Emails

### Node Details

#### Watch Inbox for CV Emails
- **Type and technical role:** `n8n-nodes-base.emailReadImap`  
  IMAP trigger node that starts the workflow when a new message appears in the mailbox.
- **Configuration choices:**
  - Format is set to **resolved**, so the node returns structured email metadata and binary attachment data.
  - `trackLastMessageId` is enabled, which helps prevent repeated processing of already-seen emails.
  - The sticky note states that attachments are expected and the first attachment is used as `attachment_0`.
  - The sticky note also says “Mark as Read”, but that behavior is not explicitly visible in the provided node parameters. If required, it should be verified in the live node configuration.
- **Key expressions or variables used downstream:**
  - `$('Watch Inbox for CV Emails').first().json.from.value[0].address`
  - `$('Watch Inbox for CV Emails').first().json.subject`
  - `$('Watch Inbox for CV Emails').first().binary`
- **Input and output connections:**
  - Input: none, this is the trigger.
  - Output: connected to **OCR: Extract CV Text**
- **Version-specific requirements:**
  - Uses **typeVersion 2.1**
- **Edge cases or potential failure types:**
  - IMAP authentication failure
  - Mailbox/folder mismatch
  - Emails without attachments
  - Attachment not available as `attachment_0`
  - Different sender structure causing expression failures later if `from.value[0].address` is missing
  - Duplicate or missed processing if IMAP state tracking is reset
- **Sub-workflow reference:** none

---

## Block 2 — OCR-Based Resume Extraction

### Overview
This block converts the attached CV PDF into extracted text. It relies on binary file input and is the bridge between raw email intake and AI evaluation.

### Nodes Involved
- OCR: Extract CV Text

### Node Details

#### OCR: Extract CV Text
- **Type and technical role:** `n8n-nodes-base.mistralAi`  
  Sends the binary resume file to Mistral OCR for text extraction.
- **Configuration choices:**
  - Reads binary input from `attachment_0`
  - Uses Mistral Cloud credentials
  - No additional options are configured
  - The sticky note specifies the intended OCR model as `mistral-ocr-latest`
- **Key expressions or variables used:**
  - Binary property: `attachment_0`
  - Downstream expected field: `{{$json.extractedText}}`
- **Input and output connections:**
  - Input: **Watch Inbox for CV Emails**
  - Output: **AI Score CV**
- **Version-specific requirements:**
  - Uses **typeVersion 1**
  - Availability depends on n8n version including the Mistral AI node with OCR support
- **Edge cases or potential failure types:**
  - Missing binary attachment
  - Unsupported file type
  - Corrupted PDF
  - OCR API authentication or quota errors
  - OCR returning empty or low-quality text for scans with poor image quality
  - Large PDF causing processing delay or service limits
- **Sub-workflow reference:** none

---

## Block 3 — AI-Based Candidate Evaluation

### Overview
This block compares the OCR-extracted CV text against a fixed AI Engineer job description. It asks Gemini to return only structured JSON that downstream logic can parse.

### Nodes Involved
- AI Score CV

### Node Details

#### AI Score CV
- **Type and technical role:** `@n8n/n8n-nodes-langchain.googleGemini`  
  LLM evaluation node using Google Gemini through the LangChain-integrated n8n node.
- **Configuration choices:**
  - Model selected: `models/gemini-3-flash-preview`
  - `jsonOutput` is enabled
  - Sampling options:
    - `topK: 20`
    - `temperature: 1`
  - A single prompt message instructs the model to:
    - act as an HR assistant for AppStoneLab Technologies
    - evaluate the CV against the embedded job description
    - return only a valid JSON object
  - The job description is hardcoded for the AI Engineer role in Surat, India, with 0–1 years experience
  - CV content is injected via `{{ $json.extractedText }}`
  - Required response schema:
    - `score`
    - `decision`
    - `candidate_name`
    - `summary`
    - `key_skills`
- **Key expressions or variables used:**
  - `{{ $json.extractedText }}`
- **Input and output connections:**
  - Input: **OCR: Extract CV Text**
  - Output: **Parse AI Response + Pass Binary**
- **Version-specific requirements:**
  - Uses **typeVersion 1.1**
  - Requires the LangChain Gemini node package to be available in the n8n instance
  - Credential type is listed as Google PaLM/Gemini API credential
- **Edge cases or potential failure types:**
  - Google API authentication or quota failure
  - Model availability changes; the sticky note says `gemini-3.1-flash-lite-preview`, while the actual node uses `models/gemini-3-flash-preview`
  - LLM may still return markdown fences or non-JSON text despite prompt instructions
  - Long CV text may approach token limits
  - Extracted OCR text quality directly affects scoring reliability
  - `jsonOutput` helps but does not guarantee fully parseable structure in all cases
- **Sub-workflow reference:** none

---

## Block 4 — Response Parsing and Data Normalization

### Overview
This is the most operationally important transformation block. It converts Gemini output into reliable workflow fields and reattaches the original binary so downstream email nodes can include the CV file.

### Nodes Involved
- Parse AI Response + Pass Binary

### Node Details

#### Parse AI Response + Pass Binary
- **Type and technical role:** `n8n-nodes-base.code`  
  JavaScript transformation node that cleans and parses the AI response.
- **Configuration choices:**
  - Reads the first input item
  - Pulls AI text from `item.json.content.parts[0].text`
  - Removes possible markdown wrappers:
    - leading ```json
    - leading ```
    - trailing ```
  - Parses the cleaned string with `JSON.parse`
  - Builds a new JSON payload containing:
    - `score`
    - `decision`
    - `candidate_name`
    - `summary`
    - `key_skills`
    - `candidate_email`
    - `original_subject`
  - Pulls candidate email from the IMAP trigger sender address
  - Copies binary data from the IMAP trigger output so attachment `attachment_0` is preserved
- **Key expressions or variables used:**
  - `item.json.content.parts[0].text`
  - `$('Watch Inbox for CV Emails').first().json.from.value[0].address`
  - `$('Watch Inbox for CV Emails').first().json.subject`
  - `$('Watch Inbox for CV Emails').first().binary`
- **Input and output connections:**
  - Input: **AI Score CV**
  - Output: **Shortlisted or Rejected?**
- **Version-specific requirements:**
  - Uses **typeVersion 2**
  - Requires Code node JavaScript support enabled in the n8n instance
- **Edge cases or potential failure types:**
  - `content.parts[0].text` path may differ if the Gemini node response shape changes
  - `JSON.parse` fails if the AI output contains stray text, malformed JSON, or missing quotes
  - Sender address expression fails if the incoming email lacks the expected `from.value[0].address` structure
  - Binary copy fails functionally if the original email contains no attachment
  - If multiple attachments exist, only the original IMAP binary set is preserved without explicit file selection logic
- **Sub-workflow reference:** none

---

## Block 5 — Decision Routing

### Overview
This block determines whether the applicant should enter the shortlist branch or the rejection branch. It relies entirely on the parsed AI field `decision`.

### Nodes Involved
- Shortlisted or Rejected?

### Node Details

#### Shortlisted or Rejected?
- **Type and technical role:** `n8n-nodes-base.if`  
  Conditional branching node.
- **Configuration choices:**
  - Strict string equality check
  - Condition: `{{$json.decision}} == shortlisted`
  - Case sensitivity is enabled
  - Type validation is strict
- **Key expressions or variables used:**
  - `={{ $json.decision }}`
- **Input and output connections:**
  - Input: **Parse AI Response + Pass Binary**
  - True output: **Notify HR with CV Summary**
  - False output: **Send Decline Email to Candidate**
- **Version-specific requirements:**
  - Uses **typeVersion 2.3**
  - Uses conditions version 3
- **Edge cases or potential failure types:**
  - If Gemini returns `Shortlisted`, `SHORTLISTED`, or extra whitespace, the condition fails because matching is strict and case-sensitive
  - Missing `decision` field causes false routing or validation issues
  - If alternative values are returned, such as `maybe` or `review`, they will be treated as rejection
- **Sub-workflow reference:** none

---

## Block 6 — Shortlist Handling

### Overview
This block handles successful candidates. It first informs HR with a rich summary and attached CV, then sends an interview invitation to the candidate in sequence.

### Nodes Involved
- Notify HR with CV Summary
- Send Interview Invite to Candidate

### Node Details

#### Notify HR with CV Summary
- **Type and technical role:** `n8n-nodes-base.emailSend`  
  SMTP email node sending an internal HR notification for shortlisted candidates.
- **Configuration choices:**
  - Recipient: `{HR_EMAIL}`
  - Sender: `{SENDER_EMAIL}`
  - Subject includes candidate name and score:
    - `✅ Shortlisted: {{ $json.candidate_name }} | AI Engineer | Score: {{ $json.score }}/100`
  - HTML body includes:
    - score banner
    - candidate name
    - candidate email
    - key skills
    - AI summary
    - role/location context
  - Original CV attached using `attachment_0`
- **Key expressions or variables used:**
  - `{{ $json.score }}`
  - `{{ $json.candidate_name }}`
  - `{{ $json.candidate_email }}`
  - `{{ $json.key_skills }}`
  - `{{ $json.summary }}`
  - attachment option: `attachment_0`
- **Input and output connections:**
  - Input: true branch from **Shortlisted or Rejected?**
  - Output: **Send Interview Invite to Candidate**
- **Version-specific requirements:**
  - Uses **typeVersion 2.1**
- **Edge cases or potential failure types:**
  - SMTP authentication or relay issues
  - `{HR_EMAIL}` and `{SENDER_EMAIL}` are placeholders and must be replaced
  - Missing attachment binary may cause send failure or mail without expected CV
  - HTML rendering differences across email clients
  - Special characters in candidate data may affect rendering if not safely formatted
- **Sub-workflow reference:** none

#### Send Interview Invite to Candidate
- **Type and technical role:** `n8n-nodes-base.emailSend`  
  SMTP email node sending a candidate-facing interview invitation.
- **Configuration choices:**
  - Recipient: candidate email from parsed data
  - Sender: `{HR_EMAIL}`
  - Subject: `Interview Invitation — AI Engineer at AppStoneLab`
  - HTML body includes:
    - congratulatory message
    - scheduling CTA button
    - “What to Expect” section with 3 interview steps
  - Attachment option is set to `attachment_0`
  - Scheduling URL placeholder:
    - `YOUR_CALENDLY_OR_CAL_LINK_HERE`
- **Key expressions or variables used:**
  - `{{ $('Parse AI Response + Pass Binary').item.json.candidate_name }}`
  - `={{ $('Parse AI Response + Pass Binary').item.json.candidate_email }}`
  - attachment option: `attachment_0`
- **Input and output connections:**
  - Input: **Notify HR with CV Summary**
  - Output: none
- **Version-specific requirements:**
  - Uses **typeVersion 2.1**
- **Edge cases or potential failure types:**
  - Placeholder scheduling link must be replaced or the email will contain a nonfunctional CTA
  - `{HR_EMAIL}` must be replaced
  - Depending on business intent, attaching the candidate’s own CV back to them may be unnecessary; this node currently includes the attachment
  - Expression depends on the named Code node output still being available in execution context
  - SMTP delivery and spam filtering issues
- **Sub-workflow reference:** none

---

## Block 7 — Rejection Handling

### Overview
This block sends a polite, branded rejection email to applicants not marked as shortlisted. It avoids attaching the CV and encourages future applications.

### Nodes Involved
- Send Decline Email to Candidate

### Node Details

#### Send Decline Email to Candidate
- **Type and technical role:** `n8n-nodes-base.emailSend`  
  SMTP email node for rejected candidates.
- **Configuration choices:**
  - Recipient: candidate email from parsed data
  - Sender: `{HR_EMAIL}`
  - Subject: `Your Application — AI Engineer at AppStoneLab`
  - HTML body includes:
    - personalized greeting with candidate name
    - empathetic rejection language
    - careers page link to future openings: `https://appstonelab.com/career`
  - No attachments configured
- **Key expressions or variables used:**
  - `{{ $('Parse AI Response + Pass Binary').item.json.candidate_name }}`
  - `={{ $('Parse AI Response + Pass Binary').item.json.candidate_email }}`
- **Input and output connections:**
  - Input: false branch from **Shortlisted or Rejected?**
  - Output: none
- **Version-specific requirements:**
  - Uses **typeVersion 2.1**
- **Edge cases or potential failure types:**
  - `{HR_EMAIL}` must be replaced with a real sender address
  - SMTP or deliverability issues
  - Candidate email missing or malformed
  - Personalized name may be blank if the AI failed to extract it
- **Sub-workflow reference:** none

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Watch Inbox for CV Emails | n8n-nodes-base.emailReadImap | Trigger on new inbound CV emails and expose email metadata plus attachments |  | OCR: Extract CV Text | ### 📥 Watch Inbox<br>**Trigger:** Fires when a new email arrives in INBOX.<br>- Format: `Resolved` (full data + binary)<br>- Attachments: ON → store candidate resume as `attachment_0`<br>- Action: Mark as Read |
| OCR: Extract CV Text | n8n-nodes-base.mistralAi | Run OCR on attached CV PDF | Watch Inbox for CV Emails | AI Score CV | ### 📄 OCR Extraction<br>**Mistral OCR** reads the CV binary.<br>- Model: `mistral-ocr-latest`<br>- Input: Candidate Resume/CV<br>- Output: Extracted info from PDF<br>- Works on both digital & scanned PDFs |
| AI Score CV | @n8n/n8n-nodes-langchain.googleGemini | Score the candidate against the job description using Gemini | OCR: Extract CV Text | Parse AI Response + Pass Binary | ### 🤖 AI CV Scoring<br>**Gemini** evaluates CV vs JD.<br>- Model: `gemini-3.1-flash-lite-preview`<br>- Input: Extracted info of Candidate from Mistral OCR<br>- Returns JSON: score, decision, name, summary, key_skills<br>- JSON Output mode: ON |
| Parse AI Response + Pass Binary | n8n-nodes-base.code | Clean AI output, parse JSON, extract email metadata, preserve attachment binary | AI Score CV | Shortlisted or Rejected? | ### ⚙️ Parse Response<br>**Critical node** — does 3 things:<br>1. Strips Gemini markdown fences (` ```json `)<br>2. Parses JSON into clean fields<br>3. Passes `binary` from IMAP trigger forward so CV attachment survives into email nodes |
| Shortlisted or Rejected? | n8n-nodes-base.if | Route candidate based on AI decision | Parse AI Response + Pass Binary | Notify HR with CV Summary; Send Decline Email to Candidate | ### 🔀 Route Decision<br>**IF**<br>`decision == "shortlisted"`<br>- ✅ TRUE → HR email + candidate invite<br>- ❌ FALSE → polite decline to candidate |
| Notify HR with CV Summary | n8n-nodes-base.emailSend | Send internal shortlist notification with CV attachment | Shortlisted or Rejected? | Send Interview Invite to Candidate | ### 📧 HR Notification<br>**Sends to HR** on shortlist.<br>- Includes: score, name, email, key skills, AI summary<br>- Attachment: `attachment_0` (original CV PDF)<br>- Subject has score in title for quick scanning |
| Send Interview Invite to Candidate | n8n-nodes-base.emailSend | Send interview invitation to shortlisted candidate | Notify HR with CV Summary |  | ### 📧 Interview Invite<br>**Sends to shortlisted candidate.**<br>- Includes scheduling link (update `YOUR_CALENDLY_OR_CAL_LINK_HERE`)<br>- 'What to Expect' section with 3 interview steps<br>- Runs after Node 6 (sequential on TRUE branch) |
| Send Decline Email to Candidate | n8n-nodes-base.emailSend | Send polite rejection email to non-shortlisted candidate | Shortlisted or Rejected? |  | ### 📧 Polite Decline<br>**Sends to rejected candidate.**<br>- Empathetic tone, no harsh language<br>- Links to appstonelab.com/career for future roles<br>- No CV attachment needed here |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation note for inbox trigger block |  |  |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation note for OCR block |  |  |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation note for AI scoring block |  |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documentation note for parsing block |  |  |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Documentation note for routing block |  |  |  |
| Sticky Note5 | n8n-nodes-base.stickyNote | Documentation note for HR notification block |  |  |  |
| Sticky Note6 | n8n-nodes-base.stickyNote | Documentation note for interview invite block |  |  |  |
| Sticky Note7 | n8n-nodes-base.stickyNote | Documentation note for rejection email block |  |  |  |
| Sticky Note8 | n8n-nodes-base.stickyNote | High-level workflow description |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like **Candidate evaluation**.
   - In workflow settings, keep binary handling enabled. This workflow uses attachment binaries later in SMTP nodes.
   - The source workflow uses:
     - `binaryMode: separate`
     - execution order `v1`

2. **Add the IMAP trigger node**
   - Create node: **Email Read IMAP**
   - Rename it to **Watch Inbox for CV Emails**
   - Configure:
     - Format: **Resolved**
     - Enable tracking of last message ID
     - Select your IMAP credentials
   - Ensure the mailbox/folder is the one receiving candidate applications.
   - Make sure attachments are included in the node output.
   - Confirm the resume arrives as `attachment_0` when there is one attached file.

3. **Configure IMAP credentials**
   - Add IMAP host, port, username, and password or app password depending on provider.
   - Use TLS/SSL as required by your email provider.
   - Test by sending a sample email with a PDF attachment to the monitored inbox.

4. **Add the OCR node**
   - Create node: **Mistral AI**
   - Rename it to **OCR: Extract CV Text**
   - Connect **Watch Inbox for CV Emails → OCR: Extract CV Text**
   - Configure:
     - Binary Property: `attachment_0`
   - Select Mistral Cloud credentials.
   - If your n8n version exposes OCR model selection, choose the OCR-capable model intended for CV extraction.

5. **Configure Mistral credentials**
   - Add your Mistral API key in n8n credentials.
   - Verify OCR access is enabled in the account plan.

6. **Add the Gemini scoring node**
   - Create node: **Google Gemini** from the LangChain-integrated node set
   - Rename it to **AI Score CV**
   - Connect **OCR: Extract CV Text → AI Score CV**
   - Configure:
     - Model: `models/gemini-3-flash-preview`  
       If unavailable in your environment, choose the nearest supported Gemini Flash model.
     - Enable JSON output
     - Temperature: `1`
     - Top K: `20`
   - Add a single user message with the HR-evaluation prompt.
   - Insert the OCR text into the prompt using:
     - `{{ $json.extractedText }}`
   - Require the model to return exactly:
     - `score`
     - `decision`
     - `candidate_name`
     - `summary`
     - `key_skills`

7. **Use the prompt content**
   - Include a fixed job description for the target role.
   - In this workflow, the role is:
     - AI Engineer at AppStoneLab Technologies
     - Surat, India
     - 0–1 years
   - The workflow hardcodes responsibilities and requirements directly into the prompt.

8. **Configure Gemini credentials**
   - Add Google Gemini/PaLM API credentials in n8n.
   - Ensure the selected model is accessible for your account and region.

9. **Add the Code node for parsing**
   - Create node: **Code**
   - Rename it to **Parse AI Response + Pass Binary**
   - Connect **AI Score CV → Parse AI Response + Pass Binary**
   - Set language to JavaScript.
   - Add logic that:
     - reads the first item
     - gets AI text from `item.json.content.parts[0].text`
     - strips markdown fences if present
     - parses JSON
     - returns clean fields
     - copies binary from the IMAP trigger node
   - Recreate these output fields:
     - `score`
     - `decision`
     - `candidate_name`
     - `summary`
     - `key_skills`
     - `candidate_email`
     - `original_subject`
   - Pull email sender from:
     - `$('Watch Inbox for CV Emails').first().json.from.value[0].address`
   - Pull binary from:
     - `$('Watch Inbox for CV Emails').first().binary`

10. **Recommended hardening for the Code node**
    - Add try/catch around `JSON.parse`
    - Normalize decision values with `trim().toLowerCase()`
    - Check that `attachment_0` exists before forwarding
    - Fall back gracefully if sender address or candidate name is missing

11. **Add the IF node**
    - Create node: **IF**
    - Rename it to **Shortlisted or Rejected?**
    - Connect **Parse AI Response + Pass Binary → Shortlisted or Rejected?**
    - Configure condition:
      - Left value: `{{ $json.decision }}`
      - Operator: equals
      - Right value: `shortlisted`
    - Keep strict validation on if you want exact behavior matching the source workflow.

12. **Add the HR notification email node**
    - Create node: **Send Email**
    - Rename it to **Notify HR with CV Summary**
    - Connect the **true** output of **Shortlisted or Rejected?** to this node
    - Configure SMTP credentials
    - Set:
      - To: real HR inbox replacing `{HR_EMAIL}`
      - From: real sender replacing `{SENDER_EMAIL}`
      - Subject:
        - `✅ Shortlisted: {{ $json.candidate_name }} | AI Engineer | Score: {{ $json.score }}/100`
      - Attachments: `attachment_0`
    - Build the HTML body to include:
      - candidate name
      - candidate email
      - score
      - key skills
      - summary
      - shortlist indicator

13. **Add the interview invitation email node**
    - Create node: **Send Email**
    - Rename it to **Send Interview Invite to Candidate**
    - Connect **Notify HR with CV Summary → Send Interview Invite to Candidate**
    - Configure:
      - To: `{{ $('Parse AI Response + Pass Binary').item.json.candidate_email }}`
      - From: replace `{HR_EMAIL}` with a real HR sender address
      - Subject: `Interview Invitation — AI Engineer at AppStoneLab`
      - Optionally attach `attachment_0` if you want to preserve source behavior
    - Add HTML body with:
      - congratulatory message
      - shortlist wording
      - scheduling call-to-action
      - “What to Expect” section
    - Replace:
      - `YOUR_CALENDLY_OR_CAL_LINK_HERE`
      with your actual interview booking URL

14. **Add the rejection email node**
    - Create node: **Send Email**
    - Rename it to **Send Decline Email to Candidate**
    - Connect the **false** output of **Shortlisted or Rejected?** to this node
    - Configure:
      - To: `{{ $('Parse AI Response + Pass Binary').item.json.candidate_email }}`
      - From: real HR address replacing `{HR_EMAIL}`
      - Subject: `Your Application — AI Engineer at AppStoneLab`
    - Add HTML body with:
      - personalized greeting using candidate name
      - polite rejection wording
      - future opportunities link:
        - `https://appstonelab.com/career`
    - Do not configure attachment forwarding on this node if you want to match intended business behavior.

15. **Configure SMTP credentials**
    - Add SMTP host, port, username, password, and secure mode according to your email provider.
    - Test sending from the configured sender domains.
    - Ensure SPF/DKIM/DMARC are aligned if production delivery matters.

16. **Add optional sticky notes for maintainability**
    - Add notes describing:
      - inbox watch behavior
      - OCR extraction
      - AI scoring
      - parsing
      - routing
      - HR notification
      - invite email
      - decline email
    - Include operational reminders like replacing placeholder email addresses and scheduling links.

17. **Test with sample candidate emails**
    - Send a test email with:
      - one PDF attachment
      - realistic sender email
      - valid subject line
    - Confirm:
      - OCR returns extracted text
      - Gemini returns parseable JSON
      - Code node preserves binary
      - IF routing behaves correctly
      - shortlisted path sends two emails
      - rejected path sends one email

18. **Validate important assumptions**
    - Confirm the first attachment is always the resume
    - Confirm the sender email is the candidate’s email
    - Confirm the Gemini node output path is still `content.parts[0].text`
    - Confirm all placeholders are replaced:
      - `{HR_EMAIL}`
      - `{SENDER_EMAIL}`
      - `YOUR_CALENDLY_OR_CAL_LINK_HERE`

19. **Activate the workflow**
    - Once tested, activate the IMAP trigger.
    - Monitor first live runs for:
      - parsing errors
      - bad OCR output
      - invalid candidate emails
      - duplicate mail processing

20. **Recommended production improvements**
    - Add a filter before OCR to ignore emails without PDF attachments
    - Add validation for attachment MIME type
    - Store results in Sheets, Airtable, or a database for auditability
    - Add confidence logging or manual-review routing for borderline scores
    - Normalize AI decision values before the IF node
    - Add fallback notifications if OCR or AI parsing fails

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| ## 🤖 AI Hiring Pipeline — Purpose: Automates end-to-end CV screening for a Specific role. Flow: 1. Watches inbox for incoming CV emails 2. Extracts CV text via Mistral OCR (works on scanned + digital PDFs) 3. Scores CV against JD using Gemini AI (returns structured JSON) 4. Parses response + preserves binary attachment 5. Routes: Shortlisted → HR notified + candidate invited \| Rejected → polite decline. Stack: IMAP · Mistral OCR · Gemini 3.1 Flash Lite · SMTP. Threshold: score ≥ shortlisted decision by Gemini | Overall workflow note |
| Careers page referenced in rejection email | https://appstonelab.com/career |
| Scheduling link in interview invite must be replaced before production use | `YOUR_CALENDLY_OR_CAL_LINK_HERE` |
| The sticky notes mention `gemini-3.1-flash-lite-preview`, but the actual configured node uses `models/gemini-3-flash-preview` | Configuration consistency note |
| The sticky note says the inbox trigger marks emails as read, but this is not explicit in the provided node parameters and should be verified in the live workflow | Operational verification note |