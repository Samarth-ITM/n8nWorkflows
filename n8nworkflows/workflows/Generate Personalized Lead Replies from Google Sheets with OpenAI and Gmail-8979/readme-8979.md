Generate Personalized Lead Replies from Google Sheets with OpenAI and Gmail

https://n8nworkflows.xyz/workflows/generate-personalized-lead-replies-from-google-sheets-with-openai-and-gmail-8979


# Generate Personalized Lead Replies from Google Sheets with OpenAI and Gmail

### 1. Workflow Overview

This workflow automates the generation and sending of personalized email replies to leads listed in a Google Sheet. It reads lead details from the sheet, leverages OpenAI's language model to craft tailored email responses, and sends these replies via Gmail with proper sender identification.

The workflow is logically divided into four main blocks:

- **1.1 Lead Intake (Google Sheets):** Fetch lead data including email, first name, intent, and message context from a specified Google Sheet.
- **1.2 Sender Identity Retrieval (Gmail API):** Retrieve the Gmail account's "sendAs" settings to obtain the display name used in the email signature.
- **1.3 Draft Generation (OpenAI Model):** Generate a professional, friendly, and personalized HTML email reply using the OpenAI language model, incorporating lead data and sender display name.
- **1.4 Email Dispatch (Gmail):** Send the generated email reply to the lead's email address with a subject line referencing the lead's intent.

---

### 2. Block-by-Block Analysis

#### 2.1 Lead Intake (Google Sheets)

- **Overview:**  
  This block reads one or more rows from a specified Google Sheet to retrieve lead information including email address, first name, intent, and the original message content. This data forms the basis for personalized response generation.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Get row(s) in sheet (Google Sheets)

- **Node Details:**  

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point to manually start the workflow.  
    - Configuration: No parameters, triggers once on manual execution.  
    - Inputs: None  
    - Outputs: Connects to "Get row(s) in sheet"  
    - Edge Cases: None specific; user must manually trigger.

  - **Get row(s) in sheet**  
    - Type: Google Sheets node  
    - Role: Reads rows from the Google Sheet containing lead data.  
    - Configuration:  
      - Document ID and Sheet Name placeholders must be replaced with actual Google Sheet identifiers.  
      - Reads all rows or specific ones as configured (default is reading all rows).  
    - Key Expressions:  
      - DocumentId: `{YOUR_GOOGLE_DOCUMENT_ID}` (to be replaced)  
      - SheetName: `{YOUR_SHEET_NAME}` (to be replaced)  
    - Inputs: Trigger from manual node  
    - Outputs: Emits each row as an item with JSON fields: Email ID, First Name, Intent, Why They Sent Email  
    - Edge Cases:  
      - Authentication failure if Google Sheets OAuth2 credentials are invalid or expired.  
      - Empty sheet or missing expected columns will cause downstream failures.  
    - Credential: Google Sheets OAuth2 account  
    - Sticky Note: Explains fields used and replacement of placeholder IDs.

#### 2.2 Sender Identity Retrieval (Gmail API)

- **Overview:**  
  Retrieves the Gmail account's "sendAs" settings, specifically to extract the display name used in the email signature. This ensures personalized and consistent sender identity in replies.

- **Nodes Involved:**  
  - HTTP Request

- **Node Details:**  

  - **HTTP Request**  
    - Type: HTTP Request node  
    - Role: Calls Gmail API endpoint to fetch sendAs settings.  
    - Configuration:  
      - URL: `https://gmail.googleapis.com/gmail/v1/users/me/settings/sendAs`  
      - Authentication: Uses predefined Gmail OAuth2 credentials.  
      - Always outputs data to ensure downstream nodes receive response even if empty.  
    - Inputs: From "Get row(s) in sheet" node (i.e., per lead)  
    - Outputs: JSON response containing sendAs array, including displayName.  
    - Edge Cases:  
      - OAuth token issues (expired, revoked).  
      - API rate limits or network failures.  
      - Unexpected response format.  
    - Credential: Gmail OAuth2 account  
    - Sticky Note: Notes usage of sendAs[0].displayName for signature.

#### 2.3 Draft Generation (OpenAI Model)

- **Overview:**  
  Generates a personalized, professional, and friendly email reply in HTML format using OpenAI's language model. The prompt includes lead-specific data and the sender's display name for signature.

- **Nodes Involved:**  
  - Message a model (OpenAI)

- **Node Details:**  

  - **Message a model**  
    - Type: OpenAI Language Model node (LangChain)  
    - Role: Generates the email draft reply.  
    - Configuration:  
      - Model ID must be specified (e.g., GPT-4 or GPT-3.5) — placeholder to be replaced.  
      - Messages array includes a single prompt string with embedded expressions:  
        ```
        Write a professional and friendly email reply to {{First Name}}. Their intent is "{{Intent}}". They wrote: "{{Why They Sent Email}}". Make the response specific to their message and helpful.
        No need to add the subject line. Generate in HTML formatting.
        The footer signature should be of the following format
        Thanks,
        {{displayName}}
        ```  
      - Uses data from the Google Sheet node and HTTP Request node (`sendAs[0].displayName`).  
    - Inputs: From HTTP Request node (sendAs info)  
    - Outputs: JSON with `message.content` containing the generated HTML email body.  
    - Edge Cases:  
      - API key invalid or quota exceeded.  
      - Model misinterpretation causing inappropriate or off-topic replies.  
      - HTML formatting issues in output.  
    - Credential: OpenAI API account  
    - Sticky Note: Emphasizes output should be HTML and excludes subject line.

#### 2.4 Email Dispatch (Gmail)

- **Overview:**  
  Sends the personalized email generated by OpenAI to the lead’s email address using Gmail. The email subject is prefixed with "Re:" followed by the lead's intent. The email body is sent as HTML.

- **Nodes Involved:**  
  - Send Personalized emails (Gmail)

- **Node Details:**  

  - **Send Personalized emails**  
    - Type: Gmail node  
    - Role: Sends the final email to the lead.  
    - Configuration:  
      - Recipient email address from Google Sheet field `Email ID`.  
      - Subject line: `"Re:" + Intent` field from the Google Sheet.  
      - Message body: OpenAI generated HTML content (`message.content`).  
      - Option to disable attribution line appended by Gmail node.  
      - **Important:** Node is configured to send email as HTML to preserve formatting.  
    - Inputs: From OpenAI node (email draft)  
    - Outputs: Email send operation result.  
    - Edge Cases:  
      - Authentication failure or revoked Gmail OAuth2 credentials.  
      - Invalid recipient email address.  
      - Gmail API limits or quota exceeded.  
      - Formatting errors if HTML is malformed.  
    - Credential: Gmail OAuth2 account  
    - Sticky Note: Reminder to set emailType = html for formatting.

---

### 3. Summary Table

| Node Name                | Node Type                   | Functional Role                          | Input Node(s)              | Output Node(s)               | Sticky Note                                                                                      |
|--------------------------|-----------------------------|----------------------------------------|----------------------------|-----------------------------|------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger              | Workflow entry trigger                  | —                          | Get row(s) in sheet          |                                                                                                |
| Get row(s) in sheet      | Google Sheets               | Reads lead data from Google Sheet      | When clicking ‘Execute workflow’ | HTTP Request                 | ## STEP 1 · Lead Intake (Google Sheets) Reads rows from your sheet. Fields used: • Email ID → recipient • First Name / Intent / Why They Sent Email → prompt context Tip: Replace the placeholder Document/Sheet IDs before running. |
| HTTP Request             | HTTP Request                | Fetches Gmail sendAs display name      | Get row(s) in sheet          | Message a model              | ## STEP 2 · Sender Identity (Gmail sendAs) Fetches Gmail sendAs to get your display name for the signature. Output used in LLM prompt: • sendAs[0].displayName |
| Message a model          | OpenAI (LangChain)          | Generates personalized email draft     | HTTP Request                | Send Personalized emails     | ## STEP 3 · Draft Generation (LLM) Creates a personalized HTML reply. Prompt inputs: • First Name, Intent, Why They Sent Email • Signature uses Gmail displayName Note: Model should output **HTML** (no subject). |
| Send Personalized emails | Gmail                      | Sends the personalized email            | Message a model             | —                           | ## STEP 4 · Send Email (Gmail) To: Email ID from sheet Subject: "Re:" + Intent Body: LLM HTML ⚠️ Set Gmail node **emailType = html** so the formatting renders. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node**  
   - Name: `When clicking ‘Execute workflow’`  
   - Purpose: Start workflow manually. No configuration needed.

2. **Create Google Sheets node**  
   - Name: `Get row(s) in sheet`  
   - Operation: Read rows from Google Sheet  
   - Parameters:  
     - Document ID: Replace `{YOUR_GOOGLE_DOCUMENT_ID}` with your actual Google Sheet ID.  
     - Sheet Name: Replace `{YOUR_SHEET_NAME}` with your actual sheet tab name.  
   - Credentials: Configure Google Sheets OAuth2 credentials with appropriate access.  
   - Connect output of Manual Trigger to this node.

3. **Create HTTP Request node**  
   - Name: `HTTP Request`  
   - Purpose: Fetch Gmail sendAs settings to get displayName.  
   - Parameters:  
     - URL: `https://gmail.googleapis.com/gmail/v1/users/me/settings/sendAs`  
     - Authentication: Use predefined Gmail OAuth2 Credential.  
     - Set "Always Output Data" to true.  
   - Credentials: Configure Gmail OAuth2 credentials.  
   - Connect output of `Get row(s) in sheet` to this node.

4. **Create OpenAI node (LangChain)**  
   - Name: `Message a model`  
   - Purpose: Generate personalized email draft.  
   - Parameters:  
     - Model ID: Replace `{YOUR_OPENAI_MODEL}` with your OpenAI model ID (e.g., `gpt-4`).  
     - Messages: Single message with content:  
       ```
       Write a professional and friendly email reply to {{ $('Get row(s) in sheet').item.json['First Name'] }}. Their intent is "{{ $('Get row(s) in sheet').item.json.Intent }}". They wrote: "{{ $('Get row(s) in sheet').item.json['Why They Sent Email'] }}". Make the response specific to their message and helpful.
       No need to add the subject line. Generate in HTML formatting.
       The footer signature should be of the following format
       Thanks,
       {{ $json.sendAs[0].displayName }}
       ```  
   - Credentials: Configure OpenAI API credentials.  
   - Connect output of `HTTP Request` node to this node.

5. **Create Gmail node**  
   - Name: `Send Personalized emails`  
   - Purpose: Send the generated email.  
   - Parameters:  
     - Send To: Expression `={{ $('Get row(s) in sheet').item.json['Email ID'] }}`  
     - Subject: Expression `=Re:{{ $('Get row(s) in sheet').item.json.Intent }}`  
     - Message: Expression `={{ $json.message.content }}`  
     - Options: Disable "Append Attribution" to prevent extra signature lines.  
     - Important: Set email format type to HTML to preserve formatting.  
   - Credentials: Configure Gmail OAuth2 credentials.  
   - Connect output of `Message a model` node to this node.

6. **Ensure proper connections:**  
   - Manual Trigger → Google Sheets  
   - Google Sheets → HTTP Request  
   - HTTP Request → OpenAI Model  
   - OpenAI Model → Gmail Send

7. **Replace all placeholders:**  
   - Google Sheet document and sheet IDs  
   - OpenAI model ID  
   - Gmail OAuth2 and Google Sheets OAuth2 credentials  
   - Webhook ID in Gmail node if used

8. **Test the workflow manually:**  
   - Trigger with Manual Trigger node  
   - Verify row data is read correctly  
   - Check sendAs output includes displayName  
   - Confirm OpenAI generates valid HTML email content  
   - Ensure Gmail sends email with correct recipient, subject, and HTML body

---

### 5. General Notes & Resources

| Note Content                                                                                                                    | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Ensure Gmail node is configured to send emails in HTML format to preserve the formatting generated by OpenAI.                 | Critical for email appearance                                                                    |
| Replace placeholder IDs (`{YOUR_GOOGLE_DOCUMENT_ID}`, `{YOUR_SHEET_NAME}`, `{YOUR_OPENAI_MODEL}`, `{YOUR_WEBHOOK_ID}`) before running. | Setup instruction                                                                                |
| Workflow uses OAuth2 credentials for Google Sheets and Gmail, requiring prior authentication and consent.                      | Credential setup                                                                                 |
| OpenAI prompt is designed to output HTML only (no subject) and includes a custom signature using Gmail displayName.            | Prompt design detail                                                                             |
| Source: Workflow generated in n8n, template ID: 7163                                                                           | Template metadata                                                                               |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.