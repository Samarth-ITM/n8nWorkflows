Automated Post-Purchase Product Delivery & Upsell with Jotform,  GDrive, Gemini

https://n8nworkflows.xyz/workflows/automated-post-purchase-product-delivery---upsell-with-jotform---gdrive--gemini-9582


# Automated Post-Purchase Product Delivery & Upsell with Jotform,  GDrive, Gemini

### 1. Workflow Overview

This workflow automates the entire post-purchase process following a customer’s submission of an order via a JotForm. It targets online sellers who want to immediately deliver digital products, log sales data, and engage customers with personalized thank-you emails enhanced by AI-generated content. The workflow is logically divided into four main blocks:

- **1.1 Input Reception:** Triggered by a new completed form submission from JotForm.
- **1.2 Digital Product Delivery & Sales Logging:** Shares the purchased product file via Google Drive and records the sale details in a Google Sheet.
- **1.3 AI-Powered Email Content Generation:** Uses Google Gemini and a LangChain AI Agent to create a personalized, HTML-formatted thank-you email that invites customers to a Discord community.
- **1.4 Email Dispatch:** Sends the AI-generated email to the customer’s email address using Gmail.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block listens for new submissions on a specified JotForm. When a customer completes the purchase form, the workflow is triggered, capturing all submitted data for downstream processing.

- **Nodes Involved:**  
  - JotForm Trigger

- **Node Details:**

  - **JotForm Trigger**  
    - Type: Trigger node for JotForm form submissions  
    - Configuration: Listens to form ID `252855528344060` using the user’s JotForm API credentials  
    - Key Expressions/Variables: Captures entire form payload, including customer name, email, phone, and product details  
    - Input: None (trigger)  
    - Output: Emits form submission data to next node  
    - Version: 1  
    - Potential Failures: API authentication errors, webhook registration issues, form ID changes, data format inconsistencies  
    - Notes: Serves as the workflow entry point, activating all subsequent actions on form submission

#### 1.2 Digital Product Delivery & Sales Logging

- **Overview:**  
  Shares the purchased digital product document with the customer via Google Drive and records the transaction in a Google Sheet for sales tracking.

- **Nodes Involved:**  
  - Share file (Google Drive)  
  - Append or update row in sheet (Google Sheets)

- **Node Details:**

  - **Share file**  
    - Type: Google Drive operation node  
    - Configuration: Shares a specific Google Doc (`fileId=1u73RpV-HJhNPtJHIwPZH3NPnjeaeFqAFbkgP_fRvkZM`) with viewer permission assigned to the customer's email extracted from the form submission  
    - Key Expressions: Uses `{{$json['Email Address']}}` from JotForm data for the email to share with  
    - Input: Receives form data from JotForm Trigger  
    - Output: Passes data forward after successful share  
    - Version: 3  
    - Potential Failures: Permissions errors, invalid file ID, service account authentication failure, email parsing errors

  - **Append or update row in sheet**  
    - Type: Google Sheets operation node  
    - Configuration: Appends or updates a row based on unique matching column `email` in Google Sheet `Sales - n8n workflow` (Sheet1)  
    - Mapped Columns:  
      - `name`: Concatenates first and last name from form data  
      - `email`: Customer email  
      - `phone`: Phone number (full format)  
      - `products`: First product name from submitted products list  
      - `amount of sale`: Price + currency of first product  
    - Input: Receives output from Share file node  
    - Output: Passes data to AI Agent node  
    - Version: 4.7  
    - Potential Failures: OAuth token expiry, sheet access issues, data format mismatch, concurrency issues on update

#### 1.3 AI-Powered Email Content Generation

- **Overview:**  
  Generates a personalized thank-you email with a warm subject line and an HTML body inviting the customer to join a Discord community, using Google Gemini and LangChain’s AI Agent.

- **Nodes Involved:**  
  - AI Agent  
  - Google Gemini Chat Model  
  - Structured Output Parser

- **Node Details:**

  - **Google Gemini Chat Model**  
    - Type: AI language model node using Google Gemini (PaLM) API  
    - Configuration: Default options, authenticated with Google Palm API credentials  
    - Input: Text prompt from previous node (via AI Agent chaining)  
    - Output: AI-generated response text  
    - Version: 1  
    - Potential Failures: API quota limits, authentication errors, network issues

  - **Structured Output Parser**  
    - Type: LangChain output parser for structured JSON  
    - Configuration: Parses AI output strictly into JSON object with `subject` and `body` fields, matching a defined schema example  
    - Input: AI chat model output  
    - Output: Structured JSON for email content  
    - Version: 1.3  
    - Potential Failures: Parsing errors if response deviates from schema, malformed JSON

  - **AI Agent**  
    - Type: LangChain AI agent node  
    - Configuration:  
      - Uses a prompt instructing the AI to create a thank-you email subject and HTML body  
      - Inserts placeholders dynamically: `{{$json.name}}`, `{{$json.products}}` from sales data  
      - Specifies tone, content, and formatting details, including a Discord invite link  
      - Expects and outputs structured JSON parsed by Structured Output Parser  
    - Input: Receives customer & product data from Google Sheets node  
    - Output: Sends parsed JSON to “Send a message” node  
    - Version: 2.2  
    - Potential Failures: Expression evaluation errors, AI generation failures, prompt misinterpretation

#### 1.4 Email Dispatch

- **Overview:**  
  Sends the personalized thank-you email created by the AI Agent to the customer's email address using Gmail with OAuth2 credentials.

- **Nodes Involved:**  
  - Send a message (Gmail)

- **Node Details:**

  - **Send a message**  
    - Type: Gmail sending node  
    - Configuration:  
      - `sendTo`: Customer email from Google Sheets node output  
      - `subject`: AI Agent’s generated email subject  
      - `message`: AI Agent’s generated HTML body as email content  
      - Credentials: Gmail OAuth2 account configured for sending  
    - Input: Receives structured email fields from AI Agent  
    - Output: None (terminal node)  
    - Version: 2.1  
    - Potential Failures: OAuth token expiry, Gmail API quota or permission errors, invalid email address, HTML formatting issues

---

### 3. Summary Table

| Node Name               | Node Type                      | Functional Role                                  | Input Node(s)                 | Output Node(s)            | Sticky Note                                                                                                           |
|-------------------------|--------------------------------|-------------------------------------------------|------------------------------|---------------------------|-----------------------------------------------------------------------------------------------------------------------|
| JotForm Trigger          | n8n-nodes-base.jotFormTrigger  | Receive new purchase submissions from JotForm   | None                         | Share file                | This automation handles what happens right after a customer makes a purchase on your online form. It automatically shares a document with them, records the sale in a spreadsheet, uses AI to write a personalized thank-you email, and then sends it to their inbox. |
| Share file               | n8n-nodes-base.googleDrive      | Share purchased product with customer via Drive | JotForm Trigger              | Append or update row in sheet |                                                                                                                       |
| Append or update row in sheet | n8n-nodes-base.googleSheets   | Log sale details in Google Sheet                  | Share file                   | AI Agent                  |                                                                                                                       |
| AI Agent                 | @n8n/n8n-nodes-langchain.agent | Generate personalized thank-you email content    | Append or update row in sheet | Send a message            |                                                                                                                       |
| Google Gemini Chat Model | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | AI language model powering the Agent             | Structured Output Parser (via AI Agent) | AI Agent (ai_languageModel input) |                                                                                                                       |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Parse AI agent output into structured JSON       | Google Gemini Chat Model      | AI Agent (ai_outputParser) |                                                                                                                       |
| Send a message           | n8n-nodes-base.gmail           | Send the personalized email via Gmail             | AI Agent                     | None                      |                                                                                                                       |
| Sticky Note              | n8n-nodes-base.stickyNote      | Documentation note on workflow purpose            | None                         | None                      | This automation handles what happens right after a customer makes a purchase on your online form. It automatically shares a document with them, records the sale in a spreadsheet, uses AI to write a personalized thank-you email, and then sends it to their inbox. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a JotForm Trigger Node**  
   - Type: `JotForm Trigger`  
   - Configure with your JotForm API credentials  
   - Select form ID `252855528344060` or your relevant form  
   - This node triggers on new form submissions

2. **Add a Google Drive Node (Share file)**  
   - Type: `Google Drive`  
   - Operation: `share`  
   - Configure with Google Service Account credentials  
   - Set `fileId` to the ID of the digital product document (e.g., `1u73RpV-HJhNPtJHIwPZH3NPnjeaeFqAFbkgP_fRvkZM`)  
   - Set permissions to `role: reader`, `type: user`  
   - Set `emailAddress` dynamically from the JotForm submission: `={{ $json["Email Address"] }}`  
   - Connect output of JotForm Trigger node to this node’s input

3. **Add a Google Sheets Node (Append or update row in sheet)**  
   - Type: `Google Sheets`  
   - Operation: `appendOrUpdate`  
   - Authenticate using Google Sheets OAuth2 credentials  
   - Set `documentId` to your sales tracking sheet ID (e.g., `1-obvaY2DHnSBXloq8zLr8Ky0sSuBgGNF8eOF9fv8ucE`)  
   - Set `sheetName` to `gid=0` or your sheet tab  
   - Define columns with mappings:  
     - `name`: `={{ $('JotForm Trigger').item.json['Full Name'].first }} {{ $('JotForm Trigger').item.json['Full Name'].last }}`  
     - `email`: `={{ $('JotForm Trigger').item.json['Email Address'] }}`  
     - `phone`: `={{ $('JotForm Trigger').item.json['Phone Number'].full }}`  
     - `products`: `={{ $('JotForm Trigger').item.json['My Products'].products[0].productName }}`  
     - `amount of sale`: `={{ $('JotForm Trigger').item.json['My Products'].products[0].subTotal }} {{ $('JotForm Trigger').item.json['My Products'].products[0].currency }}`  
   - Set `matchingColumns` to `email` to update existing entries  
   - Connect output of Share file node to this node’s input

4. **Set up Google Gemini Chat Model Node**  
   - Type: `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
   - Authenticate with Google Palm API credentials  
   - No special parameters needed beyond defaults  
   - This node will be used by AI Agent internally

5. **Add Structured Output Parser Node**  
   - Type: `@n8n/n8n-nodes-langchain.outputParserStructured`  
   - Paste the JSON schema example for email subject and body as shown in the example  
   - Connect output of Google Gemini Chat Model to this parser node

6. **Add AI Agent Node**  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - Configure prompt to:  
     - Generate a warm, inviting subject line thanking the customer by name  
     - Generate an HTML email body including placeholders for `{{ $json.name }}`, `{{ $json.products }}`  
     - Invite to Discord with clickable link: https://discord.gg/sARWmykk  
     - Use tone: appreciative, friendly, enthusiastic  
     - Sign as “IAMVAAR”  
   - Enable output parsing with Structured Output Parser node  
   - Connect input from Append or update row in sheet node  
   - Connect AI Agent’s `ai_languageModel` input to Google Gemini Chat Model node  
   - Connect AI Agent’s `ai_outputParser` input to Structured Output Parser node

7. **Add Gmail Send a message Node**  
   - Type: `Gmail`  
   - Authenticate with Gmail OAuth2 credentials for sending emails  
   - Set `sendTo`: `={{ $('Append or update row in sheet').item.json.email }}`  
   - Set `subject`: `={{ $json.output.subject }}` (from AI Agent output)  
   - Set `message`: `={{ $json.output.body }}` (HTML)  
   - Connect input from AI Agent’s output

8. **Connect the Nodes Sequentially:**  
   - JotForm Trigger → Share file → Append or update row in sheet → AI Agent → Send a message  
   - AI Agent internally connects to Google Gemini Chat Model and Structured Output Parser as per LangChain specification

9. **Add a Sticky Note (Optional)**  
   - Add a sticky note node with the description explaining the entire workflow’s purpose for documentation

---

### 5. General Notes & Resources

| Note Content                                                                                                             | Context or Link                                                                                   |
|--------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow automates post-purchase product delivery, sales logging, and personalized email communication with AI.     | Core project purpose                                                                              |
| AI-generated emails include an invitation to a Discord community to foster customer engagement and support.              | Discord invite link: https://discord.gg/sARWmykk                                                |
| Uses Google Gemini (PaLM) for AI natural language generation, integrated via LangChain agent nodes in n8n.                | https://developers.generativeai.google/                                                        |
| Google Drive file sharing uses a Service Account for secure permissions management on product delivery documents.         | https://developers.google.com/identity/protocols/oauth2/service-account                            |
| Google Sheets used as a CRM-lite tool for sales tracking and record maintenance.                                           |                                                                                                 |
| Gmail node requires OAuth2 credentials authorized to send emails on behalf of the workflow owner.                         |                                                                                                 |

---

**Disclaimer:** This document is based exclusively on an n8n workflow automation. All data processed is legal and public. The workflow complies fully with content policies and contains no illegal or offensive content.