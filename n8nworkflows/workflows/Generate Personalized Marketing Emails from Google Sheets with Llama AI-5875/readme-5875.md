Generate Personalized Marketing Emails from Google Sheets with Llama AI

https://n8nworkflows.xyz/workflows/generate-personalized-marketing-emails-from-google-sheets-with-llama-ai-5875


# Generate Personalized Marketing Emails from Google Sheets with Llama AI

---

### 1. Workflow Overview

This workflow automates the creation and sending of personalized marketing emails using promotional offer data and client information stored in Google Sheets. It leverages an AI language model (Llama 3.2 via LangChain) to generate persuasive, emotionally engaging sales copy tailored to each marketing offer. The generated content is then formatted and sent via Gmail to the list of clients.

**Target Use Case:**  
Businesses wanting to automatically generate compelling marketing emails based on dynamic offer data and distribute them to a client list without manual copywriting or email assembly.

**Logical Blocks:**  
- **1.1 Input Reception:** Trigger on changes in the marketing offer sheet, then fetch the client list from another sheet.  
- **1.2 AI Processing:** Use a Llama 3.2 model to generate promotional marketing content based on the offer data.  
- **1.3 Content Formatting:** Prepare the generated marketing message and compile client emails into a string.  
- **1.4 Email Dispatch:** Send the personalized marketing email to all clients via Gmail.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception

- **Overview:**  
  This block monitors updates in the marketing offer Google Sheet and retrieves the client email list from another Google Sheet to prepare for email generation and dispatch.

- **Nodes Involved:**  
  - Track Offer Sheet Updates (Sheet 1)  
  - Fetch Client List (Sheet 2)

- **Node Details:**  

  1. **Track Offer Sheet Updates (Sheet 1)**  
     - Type: Google Sheets Trigger  
     - Role: Watches for any updates every minute on the specified sheet containing marketing offer details.  
     - Configuration: Poll mode every minute on a specific worksheet (Sheet2 tab in the document).  
     - Inputs: Triggers workflow when a change occurs.  
     - Outputs: Passes updated offer data downstream.  
     - Edge cases: May miss changes if polling interval is too long; requires valid Google OAuth2 credentials; handles Google API rate limits.

  2. **Fetch Client List (Sheet 2)**  
     - Type: Google Sheets  
     - Role: Reads the client information (emails) from another sheet in the same Google Sheets document.  
     - Configuration: Reads a defined sheet (Sheet1 tab) using service account authentication.  
     - Inputs: Triggered by the output of the AI content generation node (indirectly).  
     - Outputs: Provides client emails for formatting and sending.  
     - Edge cases: Errors if sheet ID or name changes; authentication failures; empty or malformed email fields.

---

#### Block 1.2: AI Processing

- **Overview:**  
  Generates personalized, emotionally engaging marketing messages using a large language model based on the latest offer details.

- **Nodes Involved:**  
  - Llama 3.2 - Promo Content Model  
  - Generate Marketing Content with AI

- **Node Details:**  

  1. **Llama 3.2 - Promo Content Model**  
     - Type: LangChain LM Chat (Ollama API)  
     - Role: Provides AI language generation capability using the Llama 3.2 model.  
     - Configuration: Connects to Ollama API with specified credentials; model "llama3.2-16000:latest".  
     - Inputs: Receives prompt definition from "Generate Marketing Content with AI".  
     - Outputs: AI-generated marketing message content.  
     - Edge cases: API call failures, model unavailability, rate limits, invalid prompt syntax.

  2. **Generate Marketing Content with AI**  
     - Type: LangChain Agent Node  
     - Role: Defines the prompt and instructs the AI to generate a persuasive marketing message based on input offer data.  
     - Configuration: Complex prompt with detailed formatting instructions, personalization tokens referencing offer fields (e.g., {{ $json.title }}).  
     - Inputs: Triggered by the Google Sheets Trigger node.  
     - Outputs: AI-generated text output with marketing content.  
     - Key Expressions: Uses mustache expressions to inject dynamic offer data into prompt.  
     - Edge Cases: Expression resolution failures if input data missing; malformed prompt causing poor AI output; AI hallucinations or irrelevant content.

---

#### Block 1.3: Content Formatting

- **Overview:**  
  Aggregates the generated marketing content and client emails into a format suitable for mass email sending.

- **Nodes Involved:**  
  - Format Personalized Email

- **Node Details:**  

  1. **Format Personalized Email**  
     - Type: Code (JavaScript) Node  
     - Role: Extracts client emails from fetched data and combines them into a single comma-separated string; also passes the AI-generated marketing message forward.  
     - Configuration: JavaScript code accessing input items to collect emails under the assumed property `Emial` (likely a typo, see edge cases).  
     - Inputs: Receives client list and AI content.  
     - Outputs: Object with concatenated emails (`allEmails`) and the marketing message (`output`).  
     - Edge Cases: Property name typo (`Emial` instead of `Email`) could cause empty email list; no validation of email formats; empty client list results in blank recipients.

---

#### Block 1.4: Email Dispatch

- **Overview:**  
  Sends the formatted marketing email to all clients using Gmail.

- **Nodes Involved:**  
  - Send Marketing Email to Client

- **Node Details:**  

  1. **Send Marketing Email to Client**  
     - Type: Gmail Node  
     - Role: Sends an email with marketing content to the compiled list of client emails.  
     - Configuration: Uses OAuth2 credentials for Gmail account; sends plain text email with subject "Offer".  
     - Inputs: Receives concatenated emails and AI-generated message text.  
     - Outputs: Email delivery confirmation or error.  
     - Edge Cases: Gmail API quota limits; invalid or blank recipient emails; sending failure due to auth expiration; email content formatting issues.

---

### 3. Summary Table

| Node Name                        | Node Type                      | Functional Role                          | Input Node(s)                      | Output Node(s)                      | Sticky Note                                                                                              |
|---------------------------------|--------------------------------|----------------------------------------|----------------------------------|-----------------------------------|--------------------------------------------------------------------------------------------------------|
| Track Offer Sheet Updates (Sheet 1) | Google Sheets Trigger          | Watches marketing offer updates        | —                                | Generate Marketing Content with AI | This workflow auto-generates and sends personalized marketing emails using data from two Google Sheets: Sheet 1: Marketing Offer Details Sheet 2: Client Information It uses an AI model to create compelling content, formats it, and sends it to each client via email. |
| Generate Marketing Content with AI | LangChain Agent Node           | Generates AI-based marketing content   | Track Offer Sheet Updates (Sheet 1) | Fetch Client List (Sheet 2)        |                                                                                                        |
| Fetch Client List (Sheet 2)      | Google Sheets                  | Retrieves client emails                 | Generate Marketing Content with AI | Format Personalized Email           |                                                                                                        |
| Format Personalized Email        | Code (JavaScript)              | Formats emails and combines marketing text | Fetch Client List (Sheet 2)        | Send Marketing Email to Client      |                                                                                                        |
| Send Marketing Email to Client   | Gmail                         | Sends marketing email to clients       | Format Personalized Email          | —                                 |                                                                                                        |
| Llama 3.2 - Promo Content Model  | LangChain LM Chat (Ollama API) | Provides AI language model for content | Generate Marketing Content with AI (ai_languageModel input) | Generate Marketing Content with AI (ai_languageModel output) |                                                                                                        |
| Sticky Note                     | Sticky Note                   | Documentation note                     | —                                | —                                 | This workflow auto-generates and sends personalized marketing emails using data from two Google Sheets: Sheet 1: Marketing Offer Details Sheet 2: Client Information It uses an AI model to create compelling content, formats it, and sends it to each client via email. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets Trigger Node:**  
   - Name: "Track Offer Sheet Updates (Sheet 1)"  
   - Type: Google Sheets Trigger  
   - Credential: Google OAuth2 (with access to the marketing offer sheet)  
   - Configuration: Poll every minute on the specific sheet tab containing offer data (Sheet2) in your Google Sheets document.  
   - Purpose: Detect changes to marketing offers.

2. **Create LangChain LM Chat Node:**  
   - Name: "Llama 3.2 - Promo Content Model"  
   - Type: LangChain LM Chat (Ollama API)  
   - Credential: Ollama API credentials  
   - Model: "llama3.2-16000:latest"  
   - Purpose: Provides the AI language model for content generation.

3. **Create LangChain Agent Node:**  
   - Name: "Generate Marketing Content with AI"  
   - Type: LangChain Agent Node  
   - Prompt: Use the detailed prompt text specifying the marketing message structure, including dynamic placeholders for offer fields such as title, discount, validity, etc.  
   - Link AI Model Input to "Llama 3.2 - Promo Content Model" node.  
   - Input: Receive data from "Track Offer Sheet Updates (Sheet 1)" node.  
   - Purpose: Generate emotional, persuasive marketing copy.

4. **Create Google Sheets Node:**  
   - Name: "Fetch Client List (Sheet 2)"  
   - Type: Google Sheets  
   - Credential: Google Service Account with read access to client list sheet  
   - Configuration: Read client emails from the specified sheet tab (Sheet1).  
   - Input: Triggered by the output of "Generate Marketing Content with AI".  
   - Purpose: Retrieve client emails for sending.

5. **Create Code Node:**  
   - Name: "Format Personalized Email"  
   - Type: Code (JavaScript) Node  
   - Code:  
     ```javascript
     const emails = items.map(item => item.json.Email); // Ensure property name matches your sheet
     return [
       {
         json: {
           allEmails: emails.join(", "),
           output: $('Generate Marketing Content with AI').first().json.output,
         }
       }
     ];
     ```  
   - Input: Receives client list items.  
   - Output: Concatenated email string and AI-generated content.  
   - Purpose: Prepare email addresses and content for sending.

6. **Create Gmail Node:**  
   - Name: "Send Marketing Email to Client"  
   - Type: Gmail  
   - Credential: Gmail OAuth2 account with send permissions  
   - Parameters:  
     - Send To: `={{ $json.allEmails }}`  
     - Subject: "Offer"  
     - Message: `={{ $json.output }}` (plain text)  
   - Input: From "Format Personalized Email" node.  
   - Purpose: Dispatch emails to all clients.

7. **Connect Nodes:**  
   - Connect "Track Offer Sheet Updates (Sheet 1)" → "Generate Marketing Content with AI"  
   - Connect "Generate Marketing Content with AI" (ai_languageModel output) → "Llama 3.2 - Promo Content Model"  
   - Connect "Generate Marketing Content with AI" (main output) → "Fetch Client List (Sheet 2)"  
   - Connect "Fetch Client List (Sheet 2)" → "Format Personalized Email"  
   - Connect "Format Personalized Email" → "Send Marketing Email to Client"

8. **Test Workflow:**  
   - Update the offer sheet to trigger the workflow.  
   - Verify AI generates appropriate marketing content.  
   - Confirm client emails are fetched and concatenated correctly.  
   - Ensure Gmail node sends emails successfully.

---

### 5. General Notes & Resources

| Note Content                                                                                                                    | Context or Link                                                                                          |
|---------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| The workflow uses dynamic prompt injection with mustache syntax to personalize AI output per offer.                            | Prompt in "Generate Marketing Content with AI" node                                                     |
| Ensure the client email property in Google Sheets matches the code node's expected field name to avoid empty recipient lists.  | Code Node "Format Personalized Email" expects `Email` field (typo observed as `Emial` in original)      |
| Gmail OAuth2 credentials must have permissions to send emails on behalf of the user.                                            | Gmail node credential setup                                                                              |
| Polling interval for Google Sheets Trigger is set to every minute; adjust as needed for use case and API quota considerations. | "Track Offer Sheet Updates (Sheet 1)" node settings                                                     |
| Ollama API credentials required for Llama 3.2 model usage; ensure model availability and API limits.                            | "Llama 3.2 - Promo Content Model" node credential setup                                                 |
| This workflow automates marketing at scale, avoiding manual copywriting and email sending.                                      | Sticky Note content                                                                                       |

---

**Disclaimer:** The text provided is extracted exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.