Generate Client Proposals from Call Transcripts with AI, Google Slides and Airtable

https://n8nworkflows.xyz/workflows/generate-client-proposals-from-call-transcripts-with-ai--google-slides-and-airtable-7558


# Generate Client Proposals from Call Transcripts with AI, Google Slides and Airtable

### 1. Workflow Overview

This workflow automates the generation and distribution of client proposals based on call transcripts using AI, Google Slides, and Airtable. It is designed for agencies or consultancies that want to streamline proposal creation from discovery calls, reducing manual data entry and improving turnaround time.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives call transcript input via a chat message trigger.
- **1.2 AI Processing:** Uses an AI model (OpenAI GPT-4.1-MINI) to extract structured proposal data from the transcript.
- **1.3 Proposal Document Generation:** Copies a Google Slides proposal template and replaces placeholder texts with extracted data.
- **1.4 Proposal Sharing:** Shares the generated proposal file with the client via Google Drive permissions.
- **1.5 Airtable Update:** Updates the client's record status in Airtable to "Proposal sent".
- **1.6 Setup and Guidance:** Provides notes and instructions for API keys and credential setup.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block captures incoming chat messages that contain call transcripts or discovery call scripts, triggering the workflow.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**  
  - **Name:** When chat message received  
  - **Type:** `@n8n/n8n-nodes-langchain.chatTrigger` (Chat Trigger)  
  - **Configuration:**  
    - Webhook ID set to receive external chat messages.  
    - No special filters configured, accepts all incoming messages.  
  - **Expressions/Variables:** Receives raw chat input as `chatInput` in JSON.  
  - **Input/Output:** No input; outputs the chat message payload to the next node.  
  - **Version Requirements:** v1.3 of the node.  
  - **Potential Failures:** Webhook downtime, malformed incoming data, or unauthorized access.  
  - **Sub-workflow:** None.

#### 1.2 AI Processing

- **Overview:**  
  Uses OpenAI to extract structured data from the transcript text, specifically targeting key proposal variables.

- **Nodes Involved:**  
  - Message a model

- **Node Details:**  
  - **Name:** Message a model  
  - **Type:** `@n8n/n8n-nodes-langchain.openAi` (OpenAI Node)  
  - **Configuration:**  
    - Model: GPT-4.1-MINI  
    - System prompt instructs strict JSON extraction of: email, company, client name, project title, goals, deliverables, timeline (weeks), and budget (USD).  
    - Inputs chat transcript from prior node as user message.  
    - Output parsed as JSON (`jsonOutput=true`).  
  - **Expressions/Variables:** Uses expression `{{$json.chatInput}}` to feed transcript. Outputs structured JSON in `message.content`.  
  - **Input/Output:** Input is chat transcript; output is structured proposal data JSON.  
  - **Version Requirements:** v1.8 of the node, compatibility with OpenAI API keys.  
  - **Potential Failures:** API key issues, rate limits, malformed input causing parsing failure, or incomplete data extraction.  
  - **Sub-workflow:** None.

#### 1.3 Proposal Document Generation

- **Overview:**  
  Copies a Google Slides template file and replaces placeholder text with extracted proposal details.

- **Nodes Involved:**  
  - Copy file  
  - Replace text in a presentation

- **Node Details:**  

  - **Copy file**  
    - **Type:** `n8n-nodes-base.googleDrive` (Google Drive)  
    - **Role:** Copies a predefined Google Slides template to create a new proposal file named after the client company.  
    - **Config:**  
      - File ID of the template presentation hardcoded.  
      - Output file named dynamically using `{{$json.message.content.company}} proposal`.  
    - **Input/Output:** Input is the AI output JSON; outputs new file metadata including id.  
    - **Potential Failures:** Permission errors, invalid file ID, Google Drive API quota issues.  
  - **Replace text in a presentation**  
    - **Type:** `n8n-nodes-base.googleSlides` (Google Slides)  
    - **Role:** Replaces placeholders in the copied presentation with values from AI output JSON.  
    - **Config:**  
      - Presentation ID dynamically set from copied file metadata.  
      - Replacements include: `{Company Name}`, `{client}`, `{project_title}`, `{Goals}`, `{deliverables}`, `{timeline}`, `{budget}`.  
      - Each placeholder maps to corresponding JSON fields from AI output.  
      - PageObjectIds target specific slides/text boxes.  
    - **Input/Output:** Input is copied file info; output is presentation metadata with replaced text.  
    - **Potential Failures:** Incorrect or missing placeholders, API permission errors, text replacement failures if placeholders missing in template.  

#### 1.4 Proposal Sharing

- **Overview:**  
  Shares the generated proposal file with the client's email as a read-only user in Google Drive.

- **Nodes Involved:**  
  - Share file

- **Node Details:**  
  - **Type:** `n8n-nodes-base.googleDrive`  
  - **Role:** Adds sharing permissions to the generated proposal file, granting read access to the client.  
  - **Config:**  
    - File ID resolved from presentation metadata (`presentationId`).  
    - Permission role: reader  
    - Permission type: user  
    - Email address dynamically sourced from AI output JSON.  
  - **Input/Output:** Input is presentation metadata; outputs sharing confirmation.  
  - **Potential Failures:** Invalid email, permission API quota, Google Drive sharing restrictions.  

#### 1.5 Airtable Update

- **Overview:**  
  Updates the lead's status in an Airtable base to mark the proposal as sent.

- **Nodes Involved:**  
  - Update record

- **Node Details:**  
  - **Type:** `n8n-nodes-base.airtable`  
  - **Role:** Updates the Airtable record matching the client email to set LeadStatus to "Proposal sent".  
  - **Config:**  
    - Airtable base and table selected via IDs.  
    - Matching column: Email, value from AI output JSON.  
    - Updates LeadStatus field to "Proposal sent".  
  - **Input/Output:** Input is AI output JSON; output is update confirmation.  
  - **Potential Failures:** Incorrect API token, email not found in Airtable, network errors, field schema mismatches.  

#### 1.6 Setup and Guidance

- **Overview:**  
  Provides setup instructions and links for API keys and credential configuration.

- **Nodes Involved:**  
  - Sticky Note

- **Node Details:**  
  - **Type:** `n8n-nodes-base.stickyNote`  
  - **Role:** Visual guide for users to set up OpenAI API keys, Google OAuth credentials, and Airtable tokens.  
  - **Content:**  
    - Links for OpenAI API keys, Google Cloud console for OAuth/scopes, Airtable token creation.  
    - Reminder to have a "Lead Status" field in Airtable for updates.  
  - **Input/Output:** None; purely informational.  
  - **Potential Failures:** N/A.

---

### 3. Summary Table

| Node Name                 | Node Type                              | Functional Role                       | Input Node(s)               | Output Node(s)                | Sticky Note                                                                                                           |
|---------------------------|--------------------------------------|------------------------------------|-----------------------------|------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger | Receives chat message input          | -                           | Message a model               |                                                                                                                       |
| Message a model            | @n8n/n8n-nodes-langchain.openAi      | Extracts structured data via AI     | When chat message received   | Copy file                    |                                                                                                                       |
| Copy file                 | n8n-nodes-base.googleDrive            | Copies Slides template              | Message a model              | Replace text in a presentation |                                                                                                                       |
| Replace text in a presentation | n8n-nodes-base.googleSlides         | Replaces placeholders with data    | Copy file                   | Share file                   |                                                                                                                       |
| Share file                | n8n-nodes-base.googleDrive            | Shares proposal with client email  | Replace text in a presentation | Update record              |                                                                                                                       |
| Update record             | n8n-nodes-base.airtable               | Updates lead status in Airtable    | Share file                  | -                            |                                                                                                                       |
| Sticky Note               | n8n-nodes-base.stickyNote             | Setup and API key guidance          | -                           | -                            | Create your OpenAI API key here: https://platform.openai.com/settings/organization/api-keys Setup credentials, OAuth, and scopes for Google Drive / Slides here: https://console.cloud.google.com/ Create Airtable Token here: https://airtable.com/create/tokens Also ensure a field for Lead Status exists in Airtable to update after proposal is sent.|

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Chat Trigger Node**  
   - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Configure a webhook to receive chat messages containing call transcripts.  
   - No additional options required.  

2. **Add OpenAI Node ("Message a model")**  
   - Type: `@n8n/n8n-nodes-langchain.openAi`  
   - Credentials: Configure OpenAI API key with appropriate access.  
   - Model: Select GPT-4.1-MINI or equivalent.  
   - Message configuration:  
     - System prompt instructing extraction of JSON with keys: email, company, client, project_title, goals, deliverables, timeline_weeks, budget_usd.  
     - User message: Use expression to pass chat transcript from previous node (`{{$json.chatInput}}`).  
   - Enable JSON output parsing.  
   - Connect output of chat trigger to this node.  

3. **Add Google Drive Node ("Copy file")**  
   - Type: `n8n-nodes-base.googleDrive`  
   - Credentials: Set up Google Drive OAuth2 with read/write access.  
   - Operation: Copy file.  
   - File ID: Use the template Google Slides presentation ID (must be a valid Google Slides file).  
   - Name: Use expression `{{$json.message.content.company}} proposal` to name the copied file dynamically.  
   - Connect output of OpenAI node to this node.  

4. **Add Google Slides Node ("Replace text in a presentation")**  
   - Type: `n8n-nodes-base.googleSlides`  
   - Credentials: Use Google Slides OAuth2 (may share credentials with Google Drive).  
   - Operation: replaceText.  
   - Presentation ID: Use the ID from the copied file output (`{{$json.id}}`).  
   - Configure text replacements: For each placeholder (e.g., `{Company Name}`, `{client}`, `{project_title}`, etc.), map to corresponding JSON fields from the AI output, e.g., `{{$json.message.content.company}}`.  
   - Specify PageObjectIds corresponding to slides/text boxes in the template where replacement should happen.  
   - Connect output of the "Copy file" node here.  

5. **Add Google Drive Node ("Share file")**  
   - Type: `n8n-nodes-base.googleDrive`  
   - Operation: Share file.  
   - File ID: Use the presentation ID from the previous node (presentation with replaced text).  
   - Permissions: Set role to "reader", type to "user", and email to client email extracted from AI output (`{{$json.message.content.email}}`).  
   - Connect output of "Replace text in a presentation" node here.  

6. **Add Airtable Node ("Update record")**  
   - Type: `n8n-nodes-base.airtable`  
   - Credentials: Configure Airtable Personal Access Token with read/write access to your base.  
   - Base: Select the Airtable base ID where leads are stored.  
   - Table: Select the table name or ID for leads.  
   - Operation: Update record.  
   - Matching field: Use "Email" field to find the record matching the client email (`{{$json.message.content.email}}`).  
   - Update field: Set "LeadStatus" to the value "Proposal sent".  
   - Connect output of "Share file" node here.  

7. **Add Sticky Note** (optional but recommended)  
   - Type: `n8n-nodes-base.stickyNote`  
   - Content: Add instructions for setting up OpenAI keys, Google OAuth credentials, and Airtable tokens including relevant URLs.  
   - Place visually near the workflow start for user reference.  

8. **Connect all nodes in sequence:**  
   - When chat message received → Message a model → Copy file → Replace text in a presentation → Share file → Update record  

9. **Test the workflow:**  
   - Send a valid chat message with a discovery call script to the webhook.  
   - Verify that the AI extracts data correctly, the proposal is generated and shared, and the Airtable record updates accordingly.  

---

### 5. General Notes & Resources

| Note Content                                                                                                             | Context or Link                                                                                                     |
|--------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| Create your OpenAI API key here: https://platform.openai.com/settings/organization/api-keys                                | OpenAI API key creation                                                                                             |
| Setup credentials, OAuth, and scopes for Google Drive / Slides here: https://console.cloud.google.com/                    | Google Cloud Console for OAuth2 credential setup                                                                    |
| Create Airtable Token here: https://airtable.com/create/tokens                                                             | Airtable Personal Access Token creation                                                                             |
| Ensure your Airtable base includes a "LeadStatus" field to update after the proposal is sent                               | Airtable schema requirement                                                                                         |

---

This documentation fully describes the workflow logic, node details, setup instructions, and potential points of failure to allow advanced users or AI agents to understand, reproduce, and maintain the automated proposal generation process.