WhatsApp Support Assistant with GPT-4 Mini & Google Sheets Data Training

https://n8nworkflows.xyz/workflows/whatsapp-support-assistant-with-gpt-4-mini---google-sheets-data-training-6452


# WhatsApp Support Assistant with GPT-4 Mini & Google Sheets Data Training

### 1. Workflow Overview

This workflow implements a **WhatsApp AI HelpDesk Support Bot** that leverages GPT-4 Mini (via OpenAI) and dynamic training data stored in Google Sheets. Its primary purpose is to provide smart, context-aware customer support for WhatsApp inquiries using WasenderAPI for message sending. It integrates live conversational memory and company/product/service data to generate relevant answers while logging all customer issues for tracking.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives incoming WhatsApp messages through a webhook.
- **1.2 AI Processing:** Processes messages using a LangChain AI Agent powered by OpenAI GPT-4 Mini, enriched with conversational memory and Google Sheets data tools.
- **1.3 Data Logging:** Logs customer issues into Google Sheets for record-keeping.
- **1.4 Response Dispatch:** Sends the AI-generated response back to the customer via HTTP request to WasenderAPI.
- **1.5 Auxiliary Data Reads:** Reads company, product, and service information from Google Sheets to provide contextual knowledge to the AI Agent.
- **1.6 Post-Processing:** Executes a code node for any custom message formatting or data manipulation before sending.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Captures incoming customer messages from WhatsApp via a webhook to trigger the workflow.
- **Nodes Involved:**  
  - **Webhook**

- **Node Details:**

  - **Webhook**  
    - *Type & Role:* HTTP Webhook node; entry point for incoming WhatsApp messages.  
    - *Configuration:* Uses a fixed webhook ID (`2039cc8d-2632-4302-9677-fc16b0900002`). Set to listen for POST requests from the WhatsApp integration.  
    - *Expressions/Variables:* Typically captures incoming message payload including sender ID, message content, timestamps, etc.  
    - *Connections:* Output connects to "AI Agent - Customer Support Agent".  
    - *Version:* 2  
    - *Failure Types:* Possible webhook auth failures, invalid POST payloads, or network timeouts.

#### 2.2 AI Processing

- **Overview:** This core block handles message understanding and response generation using a LangChain AI Agent with GPT-4 Mini, enriched by conversational memory and external data sources.

- **Nodes Involved:**  
  - AI Agent - Customer Support Agent  
  - OpenAI Model1  
  - Conversation Memory  
  - Read Company Basic Information  
  - Read Product Sheet  
  - Read Service Sheet  
  - Log Customer Issues

- **Node Details:**

  - **AI Agent - Customer Support Agent**  
    - *Type & Role:* LangChain AI Agent node; orchestrates the AI processing by integrating language model, memory, and external tools.  
    - *Configuration:* Links to OpenAI GPT-4 Mini model, conversation memory, and multiple Google Sheets tools for enriched context during response generation. No internal parameters set, uses linked inputs.  
    - *Expressions:* Uses AI language model input via `ai_languageModel`, conversational memory via `ai_memory`, and tools via `ai_tool`.  
    - *Connections:* Input from Webhook; outputs to "Code" node.  
    - *Version:* 1.7  
    - *Failure Types:* Model API rate limits, memory overflow, data read errors, or expression evaluation failures.

  - **OpenAI Model1**  
    - *Type & Role:* OpenAI Chat Completion node; provides GPT-4 Mini language model.  
    - *Configuration:* Default chat model settings, connected as `ai_languageModel` input to AI Agent. Credentials for OpenAI required.  
    - *Failure Types:* Authentication errors, API limits, or connection timeouts.

  - **Conversation Memory**  
    - *Type & Role:* LangChain memory buffer; maintains recent conversation context for continuity.  
    - *Configuration:* Default buffer window size; connected to AI Agent as `ai_memory`.  
    - *Failure Types:* Memory overflow or context loss if window size is too small.

  - **Read Company Basic Information**  
    - *Type & Role:* Google Sheets Tool node; fetches company data for AI context.  
    - *Configuration:* Reads specific Google Sheet containing company info, connected to AI Agent as `ai_tool`. Requires Google Sheets OAuth2 credentials.  
    - *Failure Types:* Sheet access errors, invalid ranges, or permission issues.

  - **Read Product Sheet**  
    - *Type & Role:* Google Sheets Tool node; provides product-related data for training AI responses.  
    - *Configuration:* Reads product data sheet, connected to AI Agent as `ai_tool`.  
    - *Failure Types:* Similar to above.

  - **Read Service Sheet**  
    - *Type & Role:* Google Sheets Tool node; reads service-related info.  
    - *Configuration:* Connected to AI Agent as `ai_tool`.  
    - *Failure Types:* Same as other Google Sheets nodes.

  - **Log Customer Issues**  
    - *Type & Role:* Google Sheets Tool node; logs each customer interaction or issue for audit.  
    - *Configuration:* Connected as `ai_tool` to AI Agent, which can log issues dynamically.  
    - *Failure Types:* Write permission errors or API quota limits.

#### 2.3 Post-Processing

- **Overview:** Applies any necessary formatting or preparation of the AI-generated response before sending.

- **Nodes Involved:**  
  - Code

- **Node Details:**

  - **Code**  
    - *Type & Role:* JavaScript code execution node; processes AI output for formatting, sanitization, or custom logic.  
    - *Configuration:* No preset parameters; user-defined code logic.  
    - *Connections:* Input from AI Agent; output to HTTP Request node.  
    - *Failure Types:* Script errors or runtime exceptions.

#### 2.4 Response Dispatch

- **Overview:** Sends the finalized AI-generated message back to the customer on WhatsApp via WasenderAPI HTTP request.

- **Nodes Involved:**  
  - Send Message Using HTTP Request

- **Node Details:**

  - **Send Message Using HTTP Request**  
    - *Type & Role:* HTTP Request node; posts message payload to WasenderAPI endpoint to deliver WhatsApp messages.  
    - *Configuration:* Configured with WasenderAPI endpoint URL, method POST, authorization headers, and dynamically populated message body from Code node output.  
    - *Connections:* Input from Code node; no further outputs.  
    - *Failure Types:* Network errors, API authentication failures, invalid payloads, or WhatsApp service outages.

---

### 3. Summary Table

| Node Name                    | Node Type                           | Functional Role                      | Input Node(s)                      | Output Node(s)                    | Sticky Note                 |
|------------------------------|-----------------------------------|------------------------------------|----------------------------------|----------------------------------|-----------------------------|
| Webhook                      | Webhook                           | Input reception of WhatsApp message | —                                | AI Agent - Customer Support Agent |                             |
| AI Agent - Customer Support Agent | LangChain Agent                  | AI message processing and response generation | Webhook, OpenAI Model1, Conversation Memory, Read Sheets nodes | Code                             |                             |
| OpenAI Model1                | LangChain LM Chat OpenAI          | Provides GPT-4 Mini language model | —                                | AI Agent - Customer Support Agent |                             |
| Conversation Memory          | LangChain Memory Buffer Window    | Maintains conversation context     | —                                | AI Agent - Customer Support Agent |                             |
| Read Company Basic Information | Google Sheets Tool                | Reads company data for AI context  | —                                | AI Agent - Customer Support Agent |                             |
| Read Product Sheet           | Google Sheets Tool                 | Reads product data for AI context  | —                                | AI Agent - Customer Support Agent |                             |
| Read Service Sheet           | Google Sheets Tool                 | Reads service data for AI context  | —                                | AI Agent - Customer Support Agent |                             |
| Log Customer Issues          | Google Sheets Tool                 | Logs customer issues                | —                                | AI Agent - Customer Support Agent |                             |
| Code                        | Code                              | Post-processes AI output            | AI Agent - Customer Support Agent | Send Message Using HTTP Request |                             |
| Send Message Using HTTP Request | HTTP Request                     | Sends response to WhatsApp via API | Code                            | —                                |                             |
| Sticky Note1                | Sticky Note                       | (Empty content)                     | —                                | —                                |                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Webhook Node:**
   - Type: Webhook  
   - Configure with a unique webhook ID (e.g., `2039cc8d-2632-4302-9677-fc16b0900002`).  
   - Set to listen for HTTP POST requests.  
   - This node will receive incoming WhatsApp messages.

2. **Add OpenAI Model Node:**
   - Type: LangChain LM Chat OpenAI  
   - Select GPT-4 Mini or appropriate model.  
   - Set up OpenAI credentials (API key).  
   - No special parameters needed beyond defaults.

3. **Add Conversation Memory Node:**
   - Type: LangChain Memory Buffer Window  
   - Use default buffer window size or customize as needed.  
   - No credentials required.

4. **Add Google Sheets Tool Nodes:**
   - Create three Google Sheets Tool nodes:  
     - "Read Company Basic Information"  
     - "Read Product Sheet"  
     - "Read Service Sheet"  
   - Configure each to read the relevant Google Sheet tab/range.  
   - Set up Google OAuth2 credentials for Sheets access.

5. **Add Log Customer Issues Node:**
   - Type: Google Sheets Tool  
   - Configure to append rows to a Google Sheet dedicated to customer issue logs.  
   - Use same Google Sheets credentials.

6. **Add AI Agent Node:**
   - Type: LangChain Agent  
   - Link inputs:  
     - `ai_languageModel` to OpenAI Model1 node  
     - `ai_memory` to Conversation Memory node  
     - `ai_tool` to all four Google Sheets Tool nodes (three reading data + logging)  
   - Connect Webhook output to AI Agent input.  
   - No additional parameters required.

7. **Add Code Node:**
   - Type: Code (JavaScript)  
   - Connect AI Agent output to Code input.  
   - Implement any custom logic needed to prepare the AI response for sending (e.g., JSON formatting, sanitizing).  
   - No credentials needed.

8. **Add HTTP Request Node:**
   - Type: HTTP Request  
   - Configure to POST to WasenderAPI endpoint for sending WhatsApp messages.  
   - Set headers including authorization (API key/token).  
   - Use the Code node output as the body of the POST request.  
   - Connect Code node output to HTTP Request node input.

9. **Connect Workflow Nodes:**
   - Webhook → AI Agent  
   - AI Agent → Code  
   - Code → Send Message Using HTTP Request

10. **Activate Workflow:**
    - Ensure all credentials (OpenAI, Google Sheets, WasenderAPI) are properly set and tested.  
    - Activate the workflow to listen on the webhook URL.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                                                       |
|-------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| The workflow uses WasenderAPI for WhatsApp message sending. Ensure correct API endpoint and auth. | WasenderAPI documentation: https://wasender.com/api-docs/                                           |
| Google Sheets OAuth2 credentials must have appropriate read/write permissions for targeted sheets.| Google Sheets API setup guide: https://developers.google.com/sheets/api/quickstart/js               |
| LangChain AI Agent orchestrates multiple data sources and conversational memory for context-aware AI.| LangChain official site: https://python.langchain.com/en/latest/index.html                           |
| OpenAI GPT-4 Mini is used for cost-efficient yet powerful language model capabilities.           | OpenAI API docs: https://platform.openai.com/docs/models/gpt-4                                       |
| Conversation Memory maintains short-term chat context to keep responses coherent and relevant.   |                                                                                                     |
| The code node offers flexibility for customizing message payload before sending.                  |                                                                                                     |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or proprietary material. All data processed is legal and public.