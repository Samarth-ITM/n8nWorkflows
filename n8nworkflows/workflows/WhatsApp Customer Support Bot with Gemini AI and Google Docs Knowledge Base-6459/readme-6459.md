WhatsApp Customer Support Bot with Gemini AI and Google Docs Knowledge Base

https://n8nworkflows.xyz/workflows/whatsapp-customer-support-bot-with-gemini-ai-and-google-docs-knowledge-base-6459


# WhatsApp Customer Support Bot with Gemini AI and Google Docs Knowledge Base

---

### 1. Workflow Overview

This workflow implements a **WhatsApp Customer Support Bot** that leverages **Google Gemini AI** and a **Google Docs knowledge base** to provide automated, context-aware responses to customer inquiries received via WhatsApp. It is designed to:

- Receive incoming WhatsApp messages in real time.
- Retrieve a company's knowledge base stored in Google Docs.
- Format and send relevant context and user queries to Google Gemini AI to generate answers.
- Clean and process AI responses for readability.
- Enforce a 24-hour response window policy for message relevance.
- Log all interactions into a Google Sheets spreadsheet for record-keeping.
- Send appropriate follow-up or expiry messages when the 24-hour window has passed.

The workflow is logically divided into these main blocks:

- **1.1 Message Reception & Initial Data Retrieval:** Receive WhatsApp messages and fetch Google Docs knowledge base.
- **1.2 Prompt Preparation & AI Processing:** Format prompt combining date, knowledge base, user question, and send to Google Gemini via LangChain agent.
- **1.3 AI Response Cleaning & Delivery:** Clean the AI-generated response and send it back via WhatsApp.
- **1.4 Interaction Logging & 24-Hour Window Enforcement:** Append conversation records to Google Sheets and check if the message falls within a 24-hour window to decide on sending the AI answer or an expiration notice.

---

### 2. Block-by-Block Analysis

#### 2.1 Message Reception & Initial Data Retrieval

- **Overview:**  
  This block triggers on incoming WhatsApp messages, retrieves the company knowledge base from Google Docs, and passes data forward for prompt preparation.

- **Nodes Involved:**  
  - `when message received` (WhatsApp Trigger)  
  - `company's knowledge` (Google Docs)  

- **Node Details:**  

  - **when message received**  
    - Type: WhatsApp Trigger  
    - Role: Listens for new incoming WhatsApp messages (update event: "messages").  
    - Configuration: Uses WhatsApp OAuth credentials; webhook enabled.  
    - Inputs: External WhatsApp message event.  
    - Outputs: WhatsApp message JSON including sender info and message content.  
    - Edge Cases: Possible webhook connectivity issues, OAuth token expiration.  

  - **company's knowledge**  
    - Type: Google Docs  
    - Role: Retrieves the content of a specific Google Document used as a knowledge base.  
    - Configuration: Document URL configured via expression (Google Docs document ID).  
    - Credentials: Google Docs OAuth2.  
    - Inputs: Triggered by WhatsApp message node output.  
    - Outputs: Full content of the document in JSON form.  
    - Edge Cases: Access permission errors, document not found, API rate limits.  

---

#### 2.2 Prompt Preparation & AI Processing

- **Overview:**  
  This block constructs the AI prompt by combining the current date, the knowledge base text, and the user's WhatsApp question, then sends it to the Google Gemini AI via LangChain agent with memory buffer support.

- **Nodes Involved:**  
  - `Prepare Prompt` (Code Transformation)  
  - `Simple Memory` (LangChain Memory Buffer)  
  - `Google Gemini Chat Model` (Google Gemini AI)  
  - `AI Agent` (LangChain Agent)  

- **Node Details:**  

  - **Prepare Prompt**  
    - Type: AI Transform (JavaScript Code)  
    - Role: Extracts and formats the prompt to send to AI.  
    - Configuration:  
      - Extracts today's date as "Month Day, Year".  
      - Joins Google Docs content into plain text.  
      - Extracts user's WhatsApp message text.  
      - Constructs a prompt string with date, doc content, and user question.  
      - Returns `finalPrompt`.  
    - Inputs: Outputs from `company's knowledge` and `when message received`.  
    - Outputs: JSON with `finalPrompt`.  
    - Edge Cases: Missing document content or WhatsApp message could cause empty prompt parts; expression failures.  

  - **Simple Memory**  
    - Type: LangChain Memory Buffer Window  
    - Role: Keeps a conversation history buffer keyed by WhatsApp user ID for context continuity.  
    - Configuration: Session key set to WhatsApp sender ID extracted from incoming message.  
    - Inputs: None (invoked as AI memory for `AI Agent`).  
    - Outputs: Memory context to the agent.  
    - Edge Cases: Session key missing or malformed; memory overflow if many messages; LangChain version compatibility.  

  - **Google Gemini Chat Model**  
    - Type: LangChain Language Model Node (Google Gemini)  
    - Role: Provides the AI language model interface to Google Gemini (PaLM).  
    - Credentials: Google Palm API key configured.  
    - Inputs: Connected as AI language model for `AI Agent`.  
    - Outputs: AI-generated text response.  
    - Edge Cases: API quota exceeded, authentication errors, network timeouts.  

  - **AI Agent**  
    - Type: LangChain Agent  
    - Role: Orchestrates the chat AI process with system messages and output parsing.  
    - Configuration:  
      - System message defines the AI as a customer assistant for "TechFlow Yazılım Şirketi" with instructions to act human-like.  
      - Takes `finalPrompt` as input text.  
      - Max 5 retries allowed on failure.  
      - Output parser enabled for structured output.  
    - Inputs: `finalPrompt` from `Prepare Prompt`, AI memory from `Simple Memory`, AI language model from `Google Gemini Chat Model`.  
    - Outputs: AI response with structured JSON containing the answer.  
    - Edge Cases: API failures, parsing errors, retry exhaustion.  

---

#### 2.3 AI Response Cleaning & Delivery

- **Overview:**  
  Cleans up the raw AI-generated text to remove formatting artifacts and sends the cleaned answer back to the user via WhatsApp.

- **Nodes Involved:**  
  - `cleanAnswer` (Code)  
  - `Send AI Agent's Answer` (WhatsApp Sender)  

- **Node Details:**  

  - **cleanAnswer**  
    - Type: Code Node (JavaScript)  
    - Role: Cleans AI output by stripping markdown characters, converting markdown links to plain text + URL, collapsing excessive blank lines, and removing an unwanted preamble.  
    - Key transformations:  
      - Remove `*`, `_`, `~` markdown markers.  
      - Convert `[text](url)` to `text url`.  
      - Collapse 3+ newlines to 2 newlines.  
      - Remove leading phrases like "based on the document you provided".  
    - Inputs: AI Agent output JSON (`output` field).  
    - Outputs: JSON with cleaned `answer` string.  
    - Edge Cases: Empty or unexpected AI output format, regex failures.  

  - **Send AI Agent's Answer**  
    - Type: WhatsApp Node (Send Message)  
    - Role: Sends the cleaned answer text back to the WhatsApp user.  
    - Configuration:  
      - Text body set to cleaned answer from `cleanAnswer`.  
      - Recipient phone number extracted from incoming WhatsApp message contact `wa_id`.  
      - Phone number ID configured.  
    - Credentials: WhatsApp API account.  
    - Inputs: Cleaned answer JSON.  
    - Outputs: WhatsApp message sent confirmation.  
    - Edge Cases: WhatsApp API failure, invalid recipient number, rate limiting.  

---

#### 2.4 Interaction Logging & 24-Hour Window Enforcement

- **Overview:**  
  Logs the entire conversation interaction into a Google Sheets spreadsheet and enforces a 24-hour message response window. If the message timestamp is older than 24 hours, a timeout message is sent instead of the AI answer.

- **Nodes Involved:**  
  - `Date & Time` (Timestamp node)  
  - `Append or update row in sheet` (Google Sheets)  
  - `24-hour window check` (Code)  
  - `If` (Conditional Branch)  
  - `Send message` (WhatsApp Sender - Timeout Notice)  

- **Node Details:**  

  - **Date & Time**  
    - Type: Date & Time node  
    - Role: Adds current timestamp to the workflow data (used for logging).  
    - Inputs: AI Agent output.  
    - Outputs: JSON with current date/time.  
    - Edge Cases: None significant.  

  - **Append or update row in sheet**  
    - Type: Google Sheets  
    - Role: Logs the user message, AI response, user ID, and timestamp into a Google Sheet.  
    - Configuration:  
      - Operation: Append or update.  
      - Sheet: Named "Sayfa1" (gid=0), linked to a specific Google Sheets document ID.  
      - Columns: User (sender ID), Message (incoming text), Response (AI output), Timestamp (current date/time).  
    - Credentials: Google Sheets OAuth2.  
    - Inputs: Date & Time output, AI Agent output, WhatsApp message data.  
    - Outputs: Confirmation of row append/update.  
    - Edge Cases: Sheet access errors, schema mismatch, API limits.  

  - **24-hour window check**  
    - Type: Code Node (JavaScript)  
    - Role: Checks if incoming WhatsApp message timestamp is within the last 24 hours compared to current time.  
    - Logic: Converts WhatsApp message UNIX timestamp to ms, compares with Date.now() - 24h.  
    - Outputs: Boolean `withinWindow`, plus carries forward answer and userId.  
    - Inputs: Google Sheets append data (with current date), WhatsApp message timestamp.  
    - Edge Cases: Missing timestamps, incorrect data types.  

  - **If**  
    - Type: Conditional node  
    - Role: Branches workflow depending on `withinWindow` boolean.  
    - Condition: Passes if `withinWindow == true`.  
    - Inputs: Output of `24-hour window check`.  
    - Outputs:  
      - True branch → `cleanAnswer` → `Send AI Agent's Answer`  
      - False branch → `Send message` (timeout notice)  
    - Edge Cases: Condition evaluation errors.  

  - **Send message**  
    - Type: WhatsApp Node (Send Message)  
    - Role: Sends a fallback message when the 24-hour window has expired, instructing the user to start a new conversation.  
    - Configuration:  
      - Static text: "Mesajımızı 24 saat cevaplayamadığınız için süresi dolmuştur. Lütfen tekrar sohbet başlatın."  
      - Recipient: WhatsApp user ID.  
      - Phone number ID set.  
    - Credentials: WhatsApp API account.  
    - Inputs: From false branch of `If` node.  
    - Edge Cases: WhatsApp API failures, invalid phone number.  

---

### 3. Summary Table

| Node Name                  | Node Type                              | Functional Role                         | Input Node(s)                 | Output Node(s)              | Sticky Note                                            |
|----------------------------|--------------------------------------|---------------------------------------|------------------------------|-----------------------------|--------------------------------------------------------|
| when message received       | WhatsApp Trigger                     | Entry point, receives WhatsApp messages| -                            | company's knowledge          |                                                        |
| company's knowledge         | Google Docs                         | Fetch company knowledge base text     | when message received         | Prepare Prompt              |                                                        |
| Prepare Prompt             | AI Transform (Code)                  | Formats AI prompt combining date, docs, message | company's knowledge, when message received | AI Agent                  |                                                        |
| Simple Memory              | LangChain Memory Buffer              | Maintains conversation context memory | -                            | AI Agent                   |                                                        |
| Google Gemini Chat Model   | LangChain Language Model             | Provides Google Gemini AI model        | -                            | AI Agent                   |                                                        |
| AI Agent                  | LangChain Agent                      | Runs AI processing with prompt & memory| Prepare Prompt, Simple Memory, Google Gemini Chat Model | Date & Time               |                                                        |
| Date & Time               | Date & Time                         | Adds current timestamp                 | AI Agent                     | Append or update row in sheet |                                                        |
| Append or update row in sheet | Google Sheets                      | Logs conversation data                 | Date & Time                  | 24-hour window check         |                                                        |
| 24-hour window check       | Code                               | Checks if message is within 24 hours  | Append or update row in sheet | If                         |                                                        |
| If                        | Conditional                        | Branches on 24-hour window check      | 24-hour window check          | cleanAnswer, Send message   |                                                        |
| cleanAnswer               | Code                               | Cleans AI output text                  | If (true branch)              | Send AI Agent's Answer       |                                                        |
| Send AI Agent's Answer     | WhatsApp Sender                    | Sends cleaned AI response to user     | cleanAnswer                  | Date & Time                 |                                                        |
| Send message              | WhatsApp Sender                    | Sends timeout message if >24h          | If (false branch)             | -                           |                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create WhatsApp Trigger Node**  
   - Type: `WhatsApp Trigger`  
   - Configure webhook with OAuth credentials for WhatsApp API.  
   - Set updates to listen for `messages`.  
   - Position: Entry point.  

2. **Add Google Docs Node**  
   - Type: `Google Docs`  
   - Operation: `Get document content`  
   - Document URL: Set to your Google Docs knowledge base document ID or URL.  
   - Use Google Docs OAuth2 credentials.  
   - Connect input from WhatsApp Trigger.  

3. **Add Code Node for Prompt Preparation**  
   - Type: `AI Transform (Code)`  
   - Paste code to:  
     - Get current date formatted "Month Day, Year".  
     - Join Google Docs content text runs into a plain text string.  
     - Extract WhatsApp message text from incoming data.  
     - Construct `finalPrompt` with date, docs text, and user question.  
   - Connect input from Google Docs node.  

4. **Add LangChain Memory Buffer Node**  
   - Type: `Simple Memory` (Memory Buffer Window)  
   - Configure session key: `={{ $('when message received').item.json.contacts[0].wa_id }}`  
   - No direct input connections; used as AI memory for the agent.  

5. **Add Google Gemini Chat Model Node**  
   - Type: `LangChain Language Model` configured for Google Gemini AI.  
   - Add Google Palm API key credentials.  

6. **Add LangChain Agent Node**  
   - Type: `LangChain Agent`  
   - Set `text` parameter to `={{ $json.finalPrompt }}`  
   - Set system message to the customer assistant instructions (in Turkish).  
   - Enable output parser.  
   - Set max retries to 5.  
   - Connect inputs:  
     - Prompt from `Prepare Prompt`  
     - AI memory from `Simple Memory`  
     - Language Model from `Google Gemini Chat Model`  

7. **Add Date & Time Node**  
   - Type: `Date & Time`  
   - Purpose: Add current timestamp for logging.  
   - Connect input from `AI Agent` output.  

8. **Add Google Sheets Node**  
   - Type: `Google Sheets`  
   - Operation: Append or update row.  
   - Document ID: Your Google Sheets document for logging.  
   - Sheet Name: e.g., "Sayfa1" or appropriate sheet.  
   - Map columns:  
     - User → WhatsApp sender ID  
     - Message → Incoming WhatsApp message text  
     - Response → AI Agent's output text  
     - Timestamp → date from Date & Time node  
   - Use Google Sheets OAuth2 credentials.  
   - Connect input from Date & Time node.  

9. **Add Code Node for 24-hour Window Check**  
   - Type: Code Node  
   - Code logic:  
     - Extract WhatsApp message timestamp (convert to ms).  
     - Compare with current time minus 24 hours.  
     - Return boolean `withinWindow`.  
   - Connect input from Google Sheets node.  

10. **Add If Node for Branching**  
    - Condition: `withinWindow` equals true.  
    - Connect input from 24-hour window check node.  

11. **Add Code Node for Cleaning AI Answer**  
    - Type: Code Node  
    - Code to:  
      - Remove markdown formatting (`*`, `_`, `~`)  
      - Convert markdown links `[text](url)` to `text url`  
      - Collapse multiple blank lines  
      - Remove unwanted preamble text  
    - Connect true branch from If node.  

12. **Add WhatsApp Node to Send AI Answer**  
    - Type: WhatsApp Send Message  
    - Text Body: Set to cleaned answer output.  
    - Recipient Phone Number: Extract from WhatsApp trigger contact ID.  
    - Use WhatsApp API credentials.  
    - Connect input from cleanAnswer node.  

13. **Add WhatsApp Node to Send Timeout Message**  
    - Type: WhatsApp Send Message  
    - Text Body: Static message indicating timeout after 24 hours.  
    - Recipient Phone Number: Extract from WhatsApp trigger contact ID.  
    - Use WhatsApp API credentials.  
    - Connect false branch from If node.  

14. **Connect final nodes accordingly:**  
    - The `Send AI Agent's Answer` node connects back to `Date & Time` node for timestamping and logging.  
    - The `Append or update row in sheet` node connects to the `24-hour window check`.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                      | Context or Link                                                                                                  |
|----------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| The AI system acts as a customer assistant for "TechFlow Yazılım Şirketi", instructed to behave human-like in Turkish.          | System message in `AI Agent` node configuration.                                                                |
| WhatsApp API credentials require OAuth2 setup and webhook URL registration for real-time message triggers.                      | WhatsApp Trigger and WhatsApp Send nodes credentials.                                                           |
| Google Gemini AI (PaLM) requires valid API key and LangChain integration configured in n8n.                                      | Google Gemini Chat Model node credentials.                                                                       |
| Google Docs document must be accessible with appropriate OAuth2 credentials and permissions for reading document content.        | Google's knowledge node settings.                                                                                 |
| Google Sheets used for logging conversation history must have columns matching the mapping schema exactly to avoid errors.      | Append or update row in sheet node configuration.                                                                |
| The 24-hour window check is critical to maintain compliance with WhatsApp messaging policies and avoid sending outdated messages.| See 24-hour window check node and conditional branching.                                                         |
| For more information on WhatsApp Business API and message templates, consult: https://developers.facebook.com/docs/whatsapp    | Official WhatsApp Business API documentation.                                                                     |
| LangChain and n8n integration documentation: https://n8n.io/integrations/n8n-nodes-langchain                                  | Useful for understanding LangChain agent and memory nodes usage.                                                |

---

**Disclaimer:** This document is generated solely based on an n8n workflow JSON export. It does not contain illegal or offensive content and fully respects content policies. All data manipulated are legal and publicly accessible.

---