Personalized Follow-Up Emails for Zoom Attendees with GPT-4 and Gmail

https://n8nworkflows.xyz/workflows/personalized-follow-up-emails-for-zoom-attendees-with-gpt-4-and-gmail-8058


# Personalized Follow-Up Emails for Zoom Attendees with GPT-4 and Gmail

### 1. Workflow Overview

This workflow automates the process of sending personalized follow-up emails to attendees of a Zoom meeting using GPT-4 for content generation and Gmail for email delivery. It is designed to trigger automatically when a Zoom meeting ends, extract participant details, generate customized email content leveraging OpenAI's GPT-4, and send individual emails to each participant.

Logical blocks:

- **1.1 Input Reception:** Capture Zoom meeting ended event with participant details via webhook.
- **1.2 Data Normalization:** Extract and structure participant and meeting data into individual records.
- **1.3 AI Processing:** Use OpenAI GPT-4 to generate personalized follow-up email content per participant.
- **1.4 Email Dispatch:** Send the generated emails via Gmail to each participant.
- **1.5 Setup Instructions:** Guidance for configuring integrations and prerequisites.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block receives the Zoom meeting ended event as a webhook POST request containing meeting and participant data.

- **Nodes Involved:**  
  - Zoom Meeting Webhook

- **Node Details:**

  - **Zoom Meeting Webhook**  
    - Type: Webhook  
    - Role: Entry point capturing the Zoom meeting ended event.  
    - Configuration:  
      - HTTP method: POST  
      - Path: `/zoom-meeting-ended` (custom webhook endpoint)  
      - Expects Zoom to send meeting ended event with participant data included in payload.  
    - Inputs: External HTTP POST request from Zoom.  
    - Outputs: JSON payload forwarded to next node.  
    - Edge cases:  
      - Missing or incomplete participant data if Zoom app is misconfigured.  
      - Unauthorized requests if webhook URL is exposed.  
      - Network or timeout errors in receiving webhook.  
    - No sub-workflows involved.

#### 1.2 Data Normalization

- **Overview:**  
  Processes the raw Zoom webhook JSON to extract participant information and relevant meeting details, then outputs one item per participant for individualized processing.

- **Nodes Involved:**  
  - Normalize Participants

- **Node Details:**

  - **Normalize Participants**  
    - Type: Code (JavaScript)  
    - Role: Parse Zoom payload to create structured participant records.  
    - Configuration:  
      - Extracts the `participants` array from the Zoom webhook payload (`e.payload.object.participants`).  
      - For each participant, outputs an object containing:  
        - `meeting_id`, `topic`, `host` (host email), `participant_name`, `participant_email`, `transcript` (if available), and current timestamp.  
      - Returns an array with one output item per participant, enabling parallel processing downstream.  
    - Inputs: JSON from Zoom Meeting Webhook.  
    - Outputs: Array of participant objects.  
    - Key expressions: Uses `$input.first().json` to access webhook data.  
    - Edge cases:  
      - Empty or missing `participants` array results in zero outputs — workflow stops.  
      - Missing participant emails will later cause email sending failure.  
      - Transcript may be empty or undefined, handled with default empty string.  
    - No sub-workflows involved.

#### 1.3 AI Processing

- **Overview:**  
  Generates a personalized follow-up email content for each participant using OpenAI’s GPT-4 chat completion API.

- **Nodes Involved:**  
  - Generate Follow-Up Email

- **Node Details:**

  - **Generate Follow-Up Email**  
    - Type: OpenAI node (Chat Completion)  
    - Role: Invoke GPT-4 for drafting individualized follow-up email text.  
    - Configuration:  
      - Resource: Chat  
      - Operation: Create (Chat Completion)  
      - Model implicit: GPT-4 (as per setup instructions)  
      - No additional requestOptions configured here, implying default settings.  
      - Input to this node is one participant’s structured data from prior node.  
    - Inputs: Participant data JSON.  
    - Outputs: GPT-generated email content as text in the response.  
    - Key expressions: Not explicitly shown, but typically prompts use participant_name, topic, transcript, etc.  
    - Edge cases:  
      - API rate limits or quota exhaustion.  
      - API key misconfiguration or invalid key.  
      - Network errors or timeouts.  
      - GPT output may occasionally be irrelevant or incomplete.  
    - No sub-workflows involved.

#### 1.4 Email Dispatch

- **Overview:**  
  Sends the generated personalized follow-up email to each participant via Gmail.

- **Nodes Involved:**  
  - Send Email

- **Node Details:**

  - **Send Email**  
    - Type: Gmail node  
    - Role: Delivers the generated email content.  
    - Configuration:  
      - Subject: `"Follow-Up: {{ $('Normalize Participants').item.json.topic }}"` — dynamically uses meeting topic in subject line.  
      - Recipient email is implicitly expected to be set in the node’s parameters (likely from participant_email, although exact field mapping not shown in the JSON snippet).  
      - Uses OAuth2 Gmail credentials.  
    - Inputs: Output from GPT-4 node containing email body and participant details.  
    - Outputs: Email sent confirmation or error.  
    - Edge cases:  
      - Gmail OAuth token expiration or revocation.  
      - Email sending limits reached (Gmail quota).  
      - Invalid recipient email addresses.  
      - Network or service outages.  
    - No sub-workflows involved.

#### 1.5 Setup Instructions

- **Overview:**  
  Provides a sticky note within the workflow to help users configure necessary apps and credentials.

- **Nodes Involved:**  
  - Setup Instructions (Sticky Note)

- **Node Details:**

  - **Setup Instructions**  
    - Type: Sticky Note  
    - Role: Documentation and setup guidance embedded in workflow canvas.  
    - Content Summary:  
      - Zoom app: Enable `meeting.ended` event, include participant email/name, paste webhook URL.  
      - Gmail: Connect OAuth credentials for sending emails.  
      - OpenAI: Provide API key and use GPT-4 model.  
    - Inputs/Outputs: None (informational only).  
    - Edge cases: None.

---

### 3. Summary Table

| Node Name             | Node Type          | Functional Role                      | Input Node(s)          | Output Node(s)          | Sticky Note                                           |
|-----------------------|--------------------|------------------------------------|-----------------------|-------------------------|-------------------------------------------------------|
| Zoom Meeting Webhook   | Webhook            | Receive Zoom meeting ended event   | (External HTTP POST)  | Normalize Participants  | Setup instructions cover webhook configuration steps.|
| Normalize Participants | Code               | Extract and normalize participant data | Zoom Meeting Webhook  | Generate Follow-Up Email | Setup instructions describe data requirements.       |
| Generate Follow-Up Email | OpenAI (Chat Completion) | Generate personalized email content using GPT-4 | Normalize Participants | Send Email               | Setup instructions include OpenAI API key setup.     |
| Send Email            | Gmail              | Send personalized follow-up email | Generate Follow-Up Email | (End)                   | Setup instructions include Gmail OAuth setup.         |
| Setup Instructions    | Sticky Note        | Setup and configuration guidance  | None                  | None                    | Contains detailed setup steps for Zoom, Gmail, OpenAI.|

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Name: `Zoom Meeting Webhook`  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `zoom-meeting-ended`  
   - Purpose: Receive Zoom meeting ended event with participant data.

2. **Create Code Node**  
   - Name: `Normalize Participants`  
   - Type: Code (JavaScript)  
   - Connect input from `Zoom Meeting Webhook`.  
   - Code snippet:  
     ```javascript
     const e = $input.first().json;
     const participants = e.payload.object.participants || [];
     return participants.map(p => ({
       json: {
         meeting_id: e.payload.object.id,
         topic: e.payload.object.topic,
         host: e.payload.object.host_email,
         participant_name: p.user_name,
         participant_email: p.user_email,
         transcript: e.payload.object.transcript || '',
         timestamp: new Date().toISOString()
       }
     }));
     ```
   - Purpose: Normalize Zoom payload into individual participant records.

3. **Create OpenAI Node**  
   - Name: `Generate Follow-Up Email`  
   - Type: OpenAI (Chat Completion)  
   - Connect input from `Normalize Participants`.  
   - Configuration:  
     - Resource: Chat  
     - Operation: Create  
     - Set model to GPT-4 (via credentials or request options).  
     - Construct prompt using participant data (e.g., participant_name, topic, transcript).  
   - Credentials: OpenAI API key with GPT-4 access.

4. **Create Gmail Node**  
   - Name: `Send Email`  
   - Type: Gmail  
   - Connect input from `Generate Follow-Up Email`.  
   - Parameters:  
     - Subject: `Follow-Up: {{ $('Normalize Participants').item.json.topic }}`  
     - Recipient Email: Set to `{{ $json.participant_email }}` from normalized data.  
     - Email Body: Use content from OpenAI node output.  
   - Credentials: Gmail OAuth2 connected account.

5. **Create Sticky Note (Optional)**  
   - Name: `Setup Instructions`  
   - Content:  
     ```
     ## 🛠️ Setup Steps
     ### 1. Zoom App
     - Enable `meeting.ended` event.
     - Include participant email/name in webhook payload.
     - Paste workflow webhook URL.

     ### 2. Gmail
     - Connect Gmail OAuth in n8n.
     - Emails are sent automatically per participant.

     ### 3. OpenAI
     - Add your OpenAI API key.
     - Uses GPT-4 for personalized drafting.
     ```
   - Purpose: Provide setup guidance within the workflow editor.

6. **Connect Nodes Sequentially**:  
   - `Zoom Meeting Webhook` → `Normalize Participants` → `Generate Follow-Up Email` → `Send Email`.

7. **Credential Setup:**  
   - Configure Zoom webhook in Zoom app to send meeting ended events with participant info to the webhook URL generated by `Zoom Meeting Webhook` node.  
   - Add OpenAI API credentials with GPT-4 access in n8n.  
   - Connect Gmail OAuth2 credentials with send email permissions.

8. **Test Workflow:**  
   - Trigger a Zoom meeting end event with participants to test end-to-end email sending.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                          |
|----------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Ensure Zoom webhook payload includes participant email and name to enable personalized emails.                 | Zoom App webhook configuration.                                                                         |
| OpenAI GPT-4 usage requires appropriate API key and quota; monitor usage to avoid unexpected failures.         | OpenAI documentation: https://platform.openai.com/docs/models/gpt-4                                      |
| Gmail OAuth2 credentials must have send email permissions and be refreshed before expiration to prevent errors.| Gmail API and OAuth2 setup: https://developers.google.com/gmail/api/quickstart/js                          |
| Sticky note content inside the workflow provides concise setup instructions for Zoom, Gmail, and OpenAI setup. | Visible directly on the workflow canvas for user guidance.                                              |

---

**Disclaimer:**  
The provided content is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.