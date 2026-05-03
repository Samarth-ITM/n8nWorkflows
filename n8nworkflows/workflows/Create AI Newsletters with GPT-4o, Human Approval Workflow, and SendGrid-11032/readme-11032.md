Create AI Newsletters with GPT-4o, Human Approval Workflow, and SendGrid

https://n8nworkflows.xyz/workflows/create-ai-newsletters-with-gpt-4o--human-approval-workflow--and-sendgrid-11032


# Create AI Newsletters with GPT-4o, Human Approval Workflow, and SendGrid

### 1. Workflow Overview

This workflow automates the creation, review, and distribution of personalized newsletters using AI (OpenAI GPT-4o), human approval, and email delivery via SendGrid. It is designed for digital marketers, content creators, and small businesses who want to streamline newsletter generation while maintaining control over content through a human-in-the-loop approval step.

**Logical Blocks:**

- **1.1 Input Reception:** Collect newsletter parameters (topic, target audience, sender, approval email) via an interactive form.
- **1.2 Data Logging:** Store submitted form data into Google Sheets for record-keeping.
- **1.3 AI Content Generation:** Use GPT-4o to generate a newsletter subject and body in Markdown format based on the inputs.
- **1.4 Content Conversion:** Convert the AI-generated Markdown body to HTML.
- **1.5 Approval Process:** Send an approval request email containing a preview and an approval link to the specified approver.
- **1.6 Wait for Approval:** Pause workflow execution until the approver clicks the approval link.
- **1.7 Newsletter Distribution:** Upon approval, send the final newsletter HTML to a subscriber list via SendGrid.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Collects input from the user about the newsletter topic, target audience, sender name, and approval email address through a user-friendly form.
- **Nodes Involved:** `Newsletter Input Form`

**Node Details:**

- **Newsletter Input Form**
  - Type: Form Trigger
  - Role: Entry point that triggers workflow upon form submission.
  - Configuration:  
    - Form title in Japanese ("メルマガ生成フォーム" = "Newsletter Generation Form")
    - Fields:  
      - 今週のトピック (Topic) — required text  
      - ターゲット読者層 (Target Audience) — required text  
      - 送信者名 (Sender Name) — required text  
      - 承認用メールアドレス (Approval Email) — required email field  
    - No appended attribution on form submissions.
  - Inputs: External form submission.
  - Outputs: JSON containing form field values.
  - Edge Cases: Missing required fields; invalid email format for approval email.
  - Notes: Webhook URL auto-generated; ensure form is accessible to intended users.

#### 2.2 Data Logging

- **Overview:** Saves all form responses to a Google Sheet for auditing and review.
- **Nodes Involved:** `Workflow Configuration`, `Store Form Responses`

**Node Details:**

- **Workflow Configuration**
  - Type: Set node
  - Role: Contains static or semi-static configuration values.
  - Configuration:  
    - Assigns a string variable `subscriber_list` that holds comma-separated subscriber emails (placeholder to replace).
    - Includes all other incoming fields for continuity.
  - Inputs: Output from `Newsletter Input Form`
  - Outputs: Same data plus added `subscriber_list`.
  - Edge Cases: Placeholder not replaced with actual subscriber emails.

- **Store Form Responses**
  - Type: Google Sheets node
  - Role: Appends or updates form response data into a Google Sheet.
  - Configuration:  
    - Maps four columns: topic, sender, target, admin_email.  
    - Operation: appendOrUpdate (adds new rows or updates existing).  
    - Requires Google Sheets Document ID and Sheet Name (placeholders to replace).
  - Inputs: Output from `Workflow Configuration`.
  - Outputs: Data passed to next node.
  - Edge Cases: Google Sheets API authentication errors, invalid document ID or sheet name, quota limits.

#### 2.3 AI Content Generation

- **Overview:** Uses GPT-4o to generate a newsletter subject and body in Markdown format, structured as JSON.
- **Nodes Involved:** `Generate Newsletter Content`, `OpenAI GPT-4o Model`, `JSON Output Parser`

**Node Details:**

- **Generate Newsletter Content**
  - Type: LangChain Agent node
  - Role: Sends prompt to AI model and receives structured output.
  - Configuration:  
    - Input text composed dynamically from form fields: topic, target audience, sender name.  
    - System message instructs AI to behave as a professional email marketer and generate only JSON output with keys `subject` and `body_markdown`.  
    - Output is parsed using a JSON schema.
  - Inputs: Output from `Store Form Responses`.
  - Outputs: Parsed JSON with newsletter subject and body_markdown.
  - Edge Cases: AI response not conforming to JSON schema, model API rate limits, prompt injection risks.
  - Version: Requires n8n version supporting LangChain nodes.

- **OpenAI GPT-4o Model**
  - Type: LangChain OpenAI Chat Model node
  - Role: Executes the GPT-4o model call.
  - Configuration:  
    - Model ID set to `gpt-4o`.  
    - Credentials: OpenAI API key required.
  - Inputs: From `Generate Newsletter Content` (language model input).
  - Outputs: AI raw response forwarded to output parser.
  - Edge Cases: API key invalid, network timeout, model unavailability.

- **JSON Output Parser**
  - Type: LangChain Output Parser Structured
  - Role: Validates and extracts structured JSON from AI output.
  - Configuration:  
    - Manual schema requiring properties `subject` (string) and `body_markdown` (string).
  - Inputs: AI model output.
  - Outputs: Clean JSON usable in subsequent nodes.
  - Edge Cases: Parsing failures due to malformed AI output.

#### 2.4 Content Conversion

- **Overview:** Converts AI-generated newsletter body from Markdown to HTML for email formatting.
- **Nodes Involved:** `Convert Markdown to HTML`

**Node Details:**

- **Convert Markdown to HTML**
  - Type: Markdown node
  - Role: Converts markdown string to HTML string.
  - Configuration:  
    - Mode set to `markdownToHtml`.  
    - Input markdown taken from `body_markdown` field of AI output.
  - Inputs: Output of `Generate Newsletter Content` (JSON).
  - Outputs: HTML content as string in `data` field.
  - Edge Cases: Invalid Markdown content causing conversion errors.

#### 2.5 Approval Process

- **Overview:** Sends an email containing the newsletter preview and an approval link to the specified approval email.
- **Nodes Involved:** `Send Approval Email`

**Node Details:**

- **Send Approval Email**
  - Type: Gmail node
  - Role: Sends HTML email requesting human approval.
  - Configuration:  
    - Recipient email set from `admin_email` field stored in `Workflow Configuration` node.  
    - Subject prefixed with "【承認依頼】メルマガ下書き" ("Approval Request: Newsletter Draft") plus generated subject.  
    - HTML body includes:  
      - Subject preview (bold)  
      - Newsletter body preview wrapped in a styled div  
      - Sender and target info  
      - Approval button linking to n8n workflow resume URL (valid for 24 hours)  
    - Uses Gmail OAuth2 credentials.
  - Inputs: HTML converted content plus subject and config data.
  - Outputs: Triggers next `Wait for Approval` node.
  - Edge Cases: Gmail API quota errors, invalid OAuth token, broken approval link.

#### 2.6 Wait for Approval

- **Overview:** Pauses workflow execution awaiting the user to click the approval link in the email.
- **Nodes Involved:** `Wait for Approval`

**Node Details:**

- **Wait for Approval**
  - Type: Wait node (webhook resume)
  - Role: Suspends workflow until resume URL is called.
  - Configuration:  
    - Resume triggered by webhook.  
    - Maximum wait time: 24 hours (after which workflow times out or ends).
  - Inputs: From `Send Approval Email`.
  - Outputs: Upon resume, triggers newsletter distribution.
  - Edge Cases: No click within 24 hours, broken webhook URL, multiple clicks.

#### 2.7 Newsletter Distribution

- **Overview:** Sends the approved newsletter to all subscribers using SendGrid.
- **Nodes Involved:** `Send Newsletter to Subscribers`

**Node Details:**

- **Send Newsletter to Subscribers**
  - Type: SendGrid node
  - Role: Sends bulk email newsletter.
  - Configuration:  
    - Subject: Uses AI-generated subject.  
    - To: Uses `subscriber_list` from `Workflow Configuration` (comma-separated emails).  
    - From name: Sender name from config.  
    - From email: Must be a verified SendGrid sender (placeholder to replace).  
    - Content type: `text/html` with newsletter HTML body.  
    - Credentials: SendGrid API key.
  - Inputs: From `Wait for Approval`.
  - Outputs: Final node, ends workflow.
  - Edge Cases: SendGrid API errors, invalid verified sender, invalid recipient addresses, rate limits.

---

### 3. Summary Table

| Node Name                  | Node Type                      | Functional Role                        | Input Node(s)               | Output Node(s)                  | Sticky Note                                                                                                                                                                                                                                           |
|----------------------------|--------------------------------|--------------------------------------|-----------------------------|--------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Newsletter Input Form       | Form Trigger                   | Collect newsletter inputs            | (Trigger)                   | Workflow Configuration          | 1. **Input Collection:** Starts with a user-friendly n8n Form where you input the newsletter topic, target audience, and sender details.                                                                                                            |
| Workflow Configuration      | Set                           | Store subscriber list and pass data | Newsletter Input Form       | Store Form Responses            | 2. **Data Logging:** Automatically saves the form responses to a Google Sheet for your records.                                                                                                                                                      |
| Store Form Responses        | Google Sheets                  | Log form data into Google Sheets     | Workflow Configuration      | Generate Newsletter Content     | 2. **Data Logging:** Automatically saves the form responses to a Google Sheet for your records.                                                                                                                                                      |
| Generate Newsletter Content | LangChain Agent                | Generate newsletter content with AI | Store Form Responses        | Convert Markdown to HTML        | 3. **AI Generation:** Uses OpenAI (GPT-4o) to draft a catchy subject line and valuable body content formatted in Markdown based on your inputs.                                                                                                    |
| OpenAI GPT-4o Model         | LangChain OpenAI Chat Model    | Calls GPT-4o model                   | Generate Newsletter Content | JSON Output Parser              | 3. **AI Generation:** Uses OpenAI (GPT-4o) to draft a catchy subject line and valuable body content formatted in Markdown based on your inputs.                                                                                                    |
| JSON Output Parser          | LangChain Output Parser        | Parse AI JSON output                 | OpenAI GPT-4o Model         | Generate Newsletter Content     | 3. **AI Generation:** Uses OpenAI (GPT-4o) to draft a catchy subject line and valuable body content formatted in Markdown based on your inputs.                                                                                                    |
| Convert Markdown to HTML    | Markdown                      | Convert Markdown newsletter to HTML | Generate Newsletter Content | Send Approval Email             | 4. **Approval Process:** Sends a "Review Request" email to your inbox (via Gmail). This email contains a preview of the newsletter and a special link to approve it.                                                                                 |
| Send Approval Email         | Gmail                         | Send approval request email         | Convert Markdown to HTML    | Wait for Approval              | 4. **Approval Process:** Sends a "Review Request" email to your inbox (via Gmail). This email contains a preview of the newsletter and a special link to approve it.                                                                                 |
| Wait for Approval           | Wait                          | Pause workflow until approval click | Send Approval Email         | Send Newsletter to Subscribers  | 5. **Execution Wait:** The workflow pauses and waits for you to click the approval link.                                                                                                                                                              |
| Send Newsletter to Subscribers | SendGrid                    | Send final newsletter to subscribers | Wait for Approval           | (End)                         | 6. **Distribution:** Once approved, it converts the Markdown to HTML and sends the final newsletter to your subscriber list using SendGrid.                                                                                                         |
| Sticky Note                 | Sticky Note                   | Documentation note                  |                             |                                | See detailed workflow explanation and setup instructions in the main sticky note (positioned top-left).                                                                                                                                              |
| Sticky Note1                | Sticky Note                   | Documentation note                  |                             |                                | 1. **Input Collection:** Starts with a user-friendly n8n Form where you input the newsletter topic, target audience, and sender details.                                                                                                            |
| Sticky Note2                | Sticky Note                   | Documentation note                  |                             |                                | 2. **Data Logging:** Automatically saves the form responses to a Google Sheet for your records.                                                                                                                                                      |
| Sticky Note3                | Sticky Note                   | Documentation note                  |                             |                                | 3. **AI Generation:** Uses OpenAI (GPT-4o) to draft a catchy subject line and valuable body content formatted in Markdown based on your inputs.                                                                                                    |
| Sticky Note4                | Sticky Note                   | Documentation note                  |                             |                                | 4. **Approval Process:** Sends a "Review Request" email to your inbox (via Gmail). This email contains a preview of the newsletter and a special link to approve it.                                                                                 |
| Sticky Note5                | Sticky Note                   | Documentation note                  |                             |                                | 5. **Execution Wait:** The workflow pauses and waits for you to click the approval link.                                                                                                                                                              |
| Sticky Note6                | Sticky Note                   | Documentation note                  |                             |                                | 6. **Distribution:** Once approved, it converts the Markdown to HTML and sends the final newsletter to your subscriber list using SendGrid.                                                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node:**
   - Name: `Newsletter Input Form`
   - Configure form with these required fields:  
     - Text: "今週のトピック" (Topic)  
     - Text: "ターゲット読者層" (Target Audience)  
     - Text: "送信者名" (Sender Name)  
     - Email: "承認用メールアドレス" (Approval Email)  
   - Set form title and description as per original.
   - Save webhook ID for external form submission.

2. **Create a Set Node:**
   - Name: `Workflow Configuration`
   - Add a string variable `subscriber_list` with your comma-separated subscriber emails.
   - Enable "Include Other Fields" to pass form data downstream.

3. **Create a Google Sheets Node:**
   - Name: `Store Form Responses`
   - Operation: `appendOrUpdate`
   - Map columns: topic, sender, target, admin_email from incoming JSON.
   - Configure Google Sheets credentials.
   - Set Google Sheets Document ID and Sheet Name (must exist and match).
   
4. **Create a LangChain Agent Node:**
   - Name: `Generate Newsletter Content`
   - Text input:  
     ```
     トピック: {{ $json.topic }}
     ターゲット読者層: {{ $json.target }}
     送信者名: {{ $json.sender }}
     ```
   - System message: instruct AI to produce JSON with subject and body_markdown, using Markdown only.
   - Enable output parser.

5. **Create a LangChain OpenAI Chat Model Node:**
   - Name: `OpenAI GPT-4o Model`
   - Model ID: `gpt-4o`
   - Attach OpenAI API credentials.

6. **Create a LangChain Output Parser Node:**
   - Name: `JSON Output Parser`
   - Schema:  
     ```json
     {
       "type": "object",
       "properties": {
         "subject": { "type": "string" },
         "body_markdown": { "type": "string" }
       }
     }
     ```
   - Connect OpenAI model output to this parser.

7. **Create a Markdown Node:**
   - Name: `Convert Markdown to HTML`
   - Mode: `markdownToHtml`
   - Input markdown: `={{ $json.body_markdown }}`

8. **Create a Gmail Node:**
   - Name: `Send Approval Email`
   - To: `={{ $('Workflow Configuration').first().json.admin_email }}`
   - Subject: `=【承認依頼】メルマガ下書き: {{ $json.subject }}`
   - Message (HTML): Include subject preview, HTML newsletter preview inside styled div, sender and target info, and a clickable approval link using `{{ $execution.resumeUrl }}`.
   - Use Gmail OAuth2 credentials.

9. **Create a Wait Node:**
   - Name: `Wait for Approval`
   - Resume Mode: webhook
   - Resume webhook ID: auto-generated, use in approval email link.
   - Limit wait time: 24 hours.

10. **Create a SendGrid Node:**
    - Name: `Send Newsletter to Subscribers`
    - Subject: `={{ $('Convert Markdown to HTML').first().json.subject }}`
    - To Email: `={{ $('Workflow Configuration').first().json.subscriber_list }}`
    - From Name: `={{ $('Workflow Configuration').first().json.sender }}`
    - From Email: Set to your verified SendGrid sender email address.
    - Content Type: `text/html`
    - Content Value: `={{ $('Convert Markdown to HTML').first().json.data }}`
    - Configure SendGrid API key credentials.

11. **Connect Nodes in Sequence:**
    - `Newsletter Input Form` → `Workflow Configuration` → `Store Form Responses` → `Generate Newsletter Content` → `OpenAI GPT-4o Model` → `JSON Output Parser` → `Convert Markdown to HTML` → `Send Approval Email` → `Wait for Approval` → `Send Newsletter to Subscribers`

12. **Test Workflow:**
    - Deploy workflow.
    - Submit form.
    - Check approval email.
    - Click approval link to trigger sending.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                                                                                                                                                              |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Streamline your email marketing process with this AI-powered "Human-in-the-Loop" workflow. It allows you to generate high-quality, targeted newsletters from a simple form input, review them via email, and automatically distribute them to your subscribers upon approval.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Overview from main sticky note in the workflow.                                                                                                                                                                                             |
| Requirements: OpenAI API key for GPT-4o, Google Cloud account for Sheets and Gmail, SendGrid account for sending newsletters, and n8n version supporting LangChain nodes.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Main sticky note instructions.                                                                                                                                                                                                              |
| How to customize: Change AI model (e.g., Anthropic Claude), adjust prompt tone, or swap email providers (Mailchimp, Outlook, AWS SES) by replacing corresponding nodes.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Main sticky note instructions.                                                                                                                                                                                                              |
| Approval link in email is valid for 24 hours and clicking it triggers immediate newsletter distribution. Ensure to monitor approval emails promptly to avoid workflow timeouts.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Send Approval Email and Wait for Approval node documentation.                                                                                                                                                                               |
| Verified sender email in SendGrid must be configured to avoid delivery failures.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Send Newsletter to Subscribers node configuration note.                                                                                                                                                                                     |
| Form fields and email contents are in Japanese, matching the target audience's language context. Modify accordingly if used in a different language environment.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Workflow form and email content notes.                                                                                                                                                                                                       |
| For detailed API setup and credential configuration, consult n8n official documentation for Google Sheets, Gmail OAuth2, OpenAI API, and SendGrid integration.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | n8n official docs: https://docs.n8n.io/                                                                                                                                                                                                      |

---

**Disclaimer:**  
The provided text is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.