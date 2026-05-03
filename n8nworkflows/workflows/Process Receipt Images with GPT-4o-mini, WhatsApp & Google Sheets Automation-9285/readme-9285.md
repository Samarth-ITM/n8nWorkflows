Process Receipt Images with GPT-4o-mini, WhatsApp & Google Sheets Automation

https://n8nworkflows.xyz/workflows/process-receipt-images-with-gpt-4o-mini--whatsapp---google-sheets-automation-9285


# Process Receipt Images with GPT-4o-mini, WhatsApp & Google Sheets Automation

### 1. Workflow Overview

This workflow automates the processing of receipt images sent via WhatsApp. It targets use cases where users submit invoice or receipt images through WhatsApp messages, and the system extracts structured data from these images using an AI model, stores the receipt on Google Drive, logs extracted data into a Google Sheet, and replies back to the user with a concise, user-friendly summary of the receipt.

The workflow is logically divided into the following blocks:

- **1.1 WhatsApp Input Reception:** Listens for incoming WhatsApp messages containing receipt images and downloads the media.
- **1.2 Media Handling & Storage:** Downloads the image from WhatsApp, uploads it to Google Drive, and shares the file publicly.
- **1.3 AI-Based Image Analysis:** Downloads the stored image from Google Drive, sends it to an AI model for structured data extraction, and parses the AI response.
- **1.4 Data Logging:** Appends the structured invoice data into a Google Sheet for record-keeping.
- **1.5 Summary Generation & Response:** Generates a human-readable summary of the receipt using an AI agent and sends it back to the user on WhatsApp.

---

### 2. Block-by-Block Analysis

#### 2.1 WhatsApp Input Reception

**Overview:**  
This block triggers the workflow on new WhatsApp messages and downloads the media content (invoice image) attached in the message.

**Nodes Involved:**  
- WhatsApp Trigger  
- Download media  
- HTTP Request

**Node Details:**  

- **WhatsApp Trigger**  
  - *Type:* WhatsApp Trigger node  
  - *Role:* Listens to WhatsApp messages in real-time using webhook integration.  
  - *Config:* Monitors "messages" update events; uses OAuth credentials for WhatsApp API authentication.  
  - *Expressions:* Extracts media ID from the first message image attachment (`{{$json.messages[0].image.id}}`).  
  - *Input/Output:* No input; outputs the incoming message JSON.  
  - *Failure Modes:* OAuth token expiration, webhook misconfiguration, missing image attachment in message.  
  - *Version:* v1

- **Download media**  
  - *Type:* WhatsApp node  
  - *Role:* Downloads media content from WhatsApp servers using media ID.  
  - *Config:* Operation is "mediaUrlGet" with media ID from trigger. Uses WhatsApp API credentials.  
  - *Input:* Trigger output (message JSON).  
  - *Output:* JSON containing media URL.  
  - *Failures:* Media ID invalid or expired, WhatsApp API rate limits, network issues.  
  - *Version:* v1

- **HTTP Request**  
  - *Type:* HTTP Request node  
  - *Role:* Retrieves the actual media content from the URL provided by the previous node.  
  - *Config:* URL dynamically set from the previous node’s JSON property `{{$json.url}}`. Uses HTTP header authentication credentials.  
  - *Input:* JSON with media URL.  
  - *Output:* Binary data of the image.  
  - *Failures:* URL invalid or expired, credential expiration, request timeout.  
  - *Version:* v4.2

---

#### 2.2 Media Handling & Storage

**Overview:**  
Uploads the downloaded receipt image to Google Drive in a designated folder and sets the file's sharing permissions to be publicly readable.

**Nodes Involved:**  
- Upload file  
- Share file  
- Download file

**Node Details:**  

- **Upload file**  
  - *Type:* Google Drive node  
  - *Role:* Uploads the binary image data as a file named "data" into a specific Drive folder ("Invoices").  
  - *Config:* Drive: "My Drive"; Folder ID: specific folder for invoices; Filename fixed as "data".  
  - *Input:* Binary image data from HTTP Request.  
  - *Output:* Metadata JSON of uploaded file including file ID and webViewLink.  
  - *Failures:* Permission errors, quota limits, invalid folder ID.  
  - *Version:* v3

- **Share file**  
  - *Type:* Google Drive node  
  - *Role:* Shares the uploaded file publicly by setting permission role to "reader" and type "anyone".  
  - *Config:* File ID dynamically from upload node output (`{{$json.id}}`).  
  - *Input:* File metadata from Upload file node.  
  - *Output:* Confirmation of sharing permissions.  
  - *Failures:* Permission API errors, invalid file ID.  
  - *Version:* v3

- **Download file**  
  - *Type:* Google Drive node  
  - *Role:* Downloads the uploaded file from Drive for use in image analysis.  
  - *Config:* File ID from Upload file node output.  
  - *Input:* File metadata after sharing.  
  - *Output:* Binary image data for AI analysis.  
  - *Failures:* File not found, access denied, network issues.  
  - *Version:* v3

---

#### 2.3 AI-Based Image Analysis

**Overview:**  
Uses an OpenAI-powered LangChain node to analyze the image content and extract structured invoice data in JSON. A subsequent Code node cleans and parses this AI output.

**Nodes Involved:**  
- Analyze image  
- Code

**Node Details:**  

- **Analyze image**  
  - *Type:* LangChain OpenAI node  
  - *Role:* Sends the binary image (base64 encoded) to GPT-4o-mini for structured data extraction.  
  - *Config:* Model: GPT-4o-mini; Task: extract store_name, description, image_url, payment mode, and total amount formatted as JSON only.  
  - *Input:* Binary image from Drive download.  
  - *Output:* Raw text response with JSON embedded in markdown.  
  - *Failures:* API quota exceeded, invalid image format, AI model errors.  
  - *Version:* v1.8

- **Code**  
  - *Type:* Code node (JavaScript)  
  - *Role:* Cleans the AI model’s text output by removing markdown wrappers and newline escapes, then parses to JSON.  
  - *Config:* Uses regex to strip ```json blocks and parses cleaned string.  
  - *Input:* Raw AI text output.  
  - *Output:* Parsed JSON object with structured invoice data.  
  - *Failures:* JSON parse errors if AI output malformed, expression errors.  
  - *Version:* v2

---

#### 2.4 Data Logging

**Overview:**  
Appends the extracted structured data as a new row into a Google Sheet to maintain a log of all processed invoices.

**Nodes Involved:**  
- Append row in sheet

**Node Details:**  

- **Append row in sheet**  
  - *Type:* Google Sheets node  
  - *Role:* Inserts a new row with fields extracted by AI: store name, description, image URL (Google Drive link), payment, and total.  
  - *Config:* Document and sheet IDs set to the target spreadsheet; columns mapped from parsed JSON fields; mapping mode is explicit.  
  - *Input:* Parsed JSON from Code node; also uses upload node’s webViewLink for image URL.  
  - *Output:* Confirmation of row insertion.  
  - *Failures:* API permission errors, sheet locked or deleted, invalid data types.  
  - *Version:* v4.7

---

#### 2.5 Summary Generation & Response

**Overview:**  
Generates a human-readable summary of the receipt using an AI agent, then sends this summary back to the user on WhatsApp.

**Nodes Involved:**  
- AI Agent  
- Send message  
- OpenAI Chat Model (used as LM for AI Agent)

**Node Details:**  

- **AI Agent**  
  - *Type:* LangChain Agent node  
  - *Role:* Composes a short, emoji-rich summary text from the invoice data to improve readability and user experience.  
  - *Config:* Prompt defines output format with examples; input fields include store name, description, payment, total, and invoice link.  
  - *Input:* Parsed invoice data and Google Drive invoice link.  
  - *Output:* Plain text summary.  
  - *Failures:* AI service unavailability, prompt formatting errors.  
  - *Version:* v2.2  
  - *Sub-workflow:* Uses OpenAI Chat Model as underlying LM.

- **OpenAI Chat Model**  
  - *Type:* LangChain LM Chat OpenAI node  
  - *Role:* Provides the language model (GPT-4o-mini) backend for the AI Agent.  
  - *Config:* Model set to gpt-4o-mini; no additional options.  
  - *Input:* AI Agent prompts.  
  - *Output:* Text completions to AI Agent.  
  - *Failures:* API quota, network issues.  
  - *Version:* v1.2

- **Send message**  
  - *Type:* WhatsApp node  
  - *Role:* Sends the generated summary text back to the user’s WhatsApp number.  
  - *Config:* Text body dynamically set to AI Agent output; uses configured WhatsApp API credentials; hardcoded recipient phone number (can be parameterized).  
  - *Input:* Summary text from AI Agent.  
  - *Output:* Confirmation of message sent.  
  - *Failures:* Invalid phone number, WhatsApp API rate limits, credential issues.  
  - *Version:* v1

---

### 3. Summary Table

| Node Name           | Node Type                        | Functional Role                         | Input Node(s)         | Output Node(s)     | Sticky Note                                                                                                    |
|---------------------|---------------------------------|---------------------------------------|-----------------------|--------------------|---------------------------------------------------------------------------------------------------------------|
| WhatsApp Trigger    | n8n-nodes-base.whatsAppTrigger  | Entry point: Receive WhatsApp messages | —                     | Download media      | # 🧾 WhatsApp Receipt Bot<br>**Purpose:** Automates receipt handling via WhatsApp.<br>**Flow:**<br>1. 📥 User sends receipt<br>2. ☁️ Uploads to Google Drive<br>3. 🔍 Extracts store, items, payment, total<br>4. 📊 Sends back summary |
| Download media      | n8n-nodes-base.whatsApp          | Download media from WhatsApp           | WhatsApp Trigger       | HTTP Request        |                                                                                                               |
| HTTP Request        | n8n-nodes-base.httpRequest      | Download image binary from media URL  | Download media         | Upload file         |                                                                                                               |
| Upload file         | n8n-nodes-base.googleDrive      | Upload image to Google Drive           | HTTP Request           | Share file          |                                                                                                               |
| Share file          | n8n-nodes-base.googleDrive      | Make uploaded file publicly accessible | Upload file            | Download file       |                                                                                                               |
| Download file       | n8n-nodes-base.googleDrive      | Download file from Drive for analysis | Share file             | Analyze image       |                                                                                                               |
| Analyze image       | @n8n/n8n-nodes-langchain.openAi| Extract structured data from image    | Download file          | Code                |                                                                                                               |
| Code                | n8n-nodes-base.code             | Parse AI JSON output                   | Analyze image          | Append row in sheet |                                                                                                               |
| Append row in sheet | n8n-nodes-base.googleSheets     | Log invoice data in Google Sheets     | Code                   | AI Agent            |                                                                                                               |
| AI Agent            | @n8n/n8n-nodes-langchain.agent | Generate human-readable summary       | Append row in sheet    | Send message        |                                                                                                               |
| OpenAI Chat Model   | @n8n/n8n-nodes-langchain.lmChatOpenAi | Language model backend for AI Agent | AI Agent (ai_languageModel) | AI Agent (ai_languageModel) |                                                                                                               |
| Send message        | n8n-nodes-base.whatsApp          | Send summary reply to WhatsApp user   | AI Agent               | —                   |                                                                                                               |
| Sticky Note         | n8n-nodes-base.stickyNote       | Workflow description                   | —                     | —                   | # 🧾 WhatsApp Receipt Bot<br>**Purpose:** Automates receipt handling via WhatsApp.<br>**Flow:**<br>1. 📥 User sends receipt<br>2. ☁️ Uploads to Google Drive<br>3. 🔍 Extracts store, items, payment, total<br>4. 📊 Sends back summary |

---

### 4. Reproducing the Workflow from Scratch

1. **Create WhatsApp Trigger node**  
   - Node type: WhatsApp Trigger  
   - Configure webhook to listen for "messages" updates.  
   - Assign WhatsApp OAuth credentials.  
   - Position: (480,160)

2. **Create Download media node**  
   - Node type: WhatsApp  
   - Operation: "mediaUrlGet"  
   - MediaGetId expression: `{{$json.messages[0].image.id}}` (extract media ID from trigger message)  
   - Assign WhatsApp API credentials.  
   - Connect output of WhatsApp Trigger to input of Download media.  
   - Position: (704,160)

3. **Create HTTP Request node**  
   - Node type: HTTP Request  
   - URL expression: `{{$json.url}}` (from Download media output)  
   - Authentication: HTTP Header Authentication  
   - Assign relevant HTTP Header Auth credentials.  
   - Connect Download media to HTTP Request.  
   - Position: (928,160)

4. **Create Upload file node**  
   - Node type: Google Drive  
   - Operation: upload file  
   - Drive: "My Drive"  
   - Folder ID: Target folder for invoices (e.g., "1IP1KFkWq0SGE6vITKxueGzJcvTU4PIVx")  
   - File name: "data" (can be customized)  
   - Assign Google Drive OAuth2 credentials.  
   - Connect HTTP Request output to Upload file input.  
   - Position: (1152,160)

5. **Create Share file node**  
   - Node type: Google Drive  
   - Operation: share  
   - File ID: `{{$json.id}}` (from Upload file output)  
   - Permissions: role = "reader", type = "anyone"  
   - Assign Google Drive OAuth2 credentials.  
   - Connect Upload file to Share file node.  
   - Position: (1376,160)

6. **Create Download file node**  
   - Node type: Google Drive  
   - Operation: download  
   - File ID: `{{$json.id}}` (from Upload file output)  
   - Assign Google Drive OAuth2 credentials.  
   - Connect Share file to Download file node.  
   - Position: (1600,160)

7. **Create Analyze image node**  
   - Node type: LangChain OpenAI  
   - Operation: analyze image  
   - Model: GPT-4o-mini  
   - Input type: base64 (binary from Download file)  
   - Prompt: instruct model to extract store_name, description, image_url, payment, total in JSON only format.  
   - Assign OpenAI API credentials.  
   - Connect Download file to Analyze image.  
   - Position: (480,384)

8. **Create Code node**  
   - Node type: Code (JavaScript)  
   - Code: strip ```json markdown and parse JSON string from AI output.  
   - Connect Analyze image output to Code input.  
   - Position: (704,384)

9. **Create Append row in sheet node**  
   - Node type: Google Sheets  
   - Operation: append  
   - Document ID: target Google Sheet for invoices (e.g., "1cG8U5EpRXKXkM3eN74byIhQBkmfF59mLgvDc3S2xT0s")  
   - Sheet name: "Sheet1" or "gid=0"  
   - Columns: Map from Code output fields: "store name", "discription" (note spelling as in workflow), "image_url" (from Upload file node webViewLink), "payment", "total".  
   - Assign Google Sheets OAuth2 credentials.  
   - Connect Code node to Append row in sheet.  
   - Position: (928,384)

10. **Create AI Agent node**  
    - Node type: LangChain Agent  
    - Prompt: Generate short invoice summary with emojis and invoice link; structured text output (not JSON).  
    - Input: fields from Append row (store name, discription, total, payment) and output link (webViewLink from Upload file).  
    - Assign no specific credentials (uses LM credentials).  
    - Connect Append row in sheet to AI Agent.  
    - Position: (1152,384)

11. **Create OpenAI Chat Model node**  
    - Node type: LangChain LM Chat OpenAI  
    - Model: GPT-4o-mini  
    - Assign OpenAI API credentials.  
    - Connect AI Agent (language model input) to OpenAI Chat Model.  
    - Position: (1104,560)

12. **Create Send message node**  
    - Node type: WhatsApp  
    - Operation: send  
    - Text body: `{{$json.output}}` from AI Agent output (the summary text)  
    - Phone Number ID: WhatsApp phone number ID for sending  
    - Recipient phone number: hardcoded or dynamic user number (currently +919827442000)  
    - Assign WhatsApp API credentials.  
    - Connect AI Agent output to Send message.  
    - Position: (1504,384)

13. **Add Sticky Note**  
    - Node type: Sticky Note  
    - Content: Workflow description: "WhatsApp Receipt Bot" with purpose and flow steps.  
    - Position: (464, -80)

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                             | Context or Link                                                   |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| The workflow uses GPT-4o-mini model for image analysis and text summarization, balancing power and cost.                                                               | AI Model selection                                                |
| The Google Drive folder and Google Sheet IDs must be pre-configured with appropriate permissions for the OAuth credentials used.                                      | Google Drive & Sheets setup                                       |
| The WhatsApp recipient phone number is currently hardcoded; for production, consider dynamic handling based on incoming message sender to reply correctly.             | WhatsApp Send message node                                        |
| The AI output parsing relies on the AI consistently returning valid JSON inside markdown - this is an important potential failure point.                               | AI output handling                                                |
| The sheet column "discription" is misspelled but must be kept consistent for schema mapping unless corrected in all places.                                           | Data schema detail                                               |
| The workflow includes a sticky note summarizing the purpose and flow, useful for quick reference inside the n8n editor.                                                | Sticky note content                                               |
| OpenAI API credentials and WhatsApp OAuth credentials must be properly configured with valid tokens and scopes before running.                                        | Credential setup                                                  |

---

**Disclaimer:**  
The provided text is exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.