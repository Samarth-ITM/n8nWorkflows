LINE Assistant with Google Calendar and Gmail Integration

https://n8nworkflows.xyz/workflows/line-assistant-with-google-calendar-and-gmail-integration-2671


# LINE Assistant with Google Calendar and Gmail Integration

### 1. Workflow Overview

This workflow, titled **"LINE Assistant with Google Calendar and Gmail Integration"**, is designed to automate communication and scheduling tasks for users who rely on LINE messaging, Google Calendar, and Gmail. It targets small business owners, personal assistants, and project managers who need to manage conversations, schedule events, and retrieve emails efficiently without manual switching between platforms.

The workflow logically consists of the following blocks:

- **1.1 Input Reception:** Listens to incoming LINE messages via webhook and filters message types.
- **1.2 AI Processing & Memory Management:** Uses LangChain AI agent with OpenAI GPT and Wikipedia tools, supported by session memory, to interpret user input and generate responses.
- **1.3 External Data Retrieval:** Interacts with Google Calendar and Gmail APIs to fetch or create calendar events and retrieve emails based on AI-parsed user queries.
- **1.4 Response Handling & Delivery:** Cleans and formats AI-generated responses and sends replies back to LINE users, handling both ordinary and error cases.
- **1.5 Error Management:** Implements fallback logic for unrecognized inputs or AI response issues to maintain graceful user experience.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block receives incoming webhook requests from LINE platform, validating and routing messages based on type (text or others).

- **Nodes Involved:**  
  - Line Receiving  
  - Switch Between Text and Others

- **Node Details:**  

  - **Line Receiving**  
    - Type: Webhook  
    - Role: Entry point; receives HTTP POST requests from LINE messaging API.  
    - Config: Path set to `linechatbotagent`, HTTP method POST, secured by webhookId.  
    - Input: External LINE platform message payload.  
    - Output: Raw JSON event data, including userId, message text, and reply token.  
    - Edge Cases: Invalid payloads, unsupported message types, webhook security failures.

  - **Switch Between Text and Others**  
    - Type: Switch  
    - Role: Routes flow based on message type (only processes text messages).  
    - Config: Checks if `body.events[0].message.type` equals `"text"`; routes text messages to AI agent, others to error handler.  
    - Input: Output from Line Receiving.  
    - Output: Two branches — text messages proceed to AI Agent; other types lead to Line Answering (Error Case).  
    - Edge Cases: New or unsupported message types might bypass logic; case sensitivity enforced.

---

#### 2.2 AI Processing & Memory Management

- **Overview:**  
  Interprets user messages using an AI agent powered by OpenAI GPT and LangChain tools, while maintaining conversational context with window buffer memory.

- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model  
  - Window Buffer Memory  
  - Wikipedia  
  - OpenAI

- **Node Details:**  

  - **AI Agent**  
    - Type: LangChain Agent Node  
    - Role: Central AI interpreter that receives user text and decides on actions/tools to invoke.  
    - Config: Receives input text from LINE message JSON path `body.events[0].message.text`. Sets a system message with current date context. Uses "define" promptType for structured interaction.  
    - Input: Text from Switch node; tool outputs (Wikipedia, Gmail, Calendar).  
    - Output: AI-generated response and structured commands to tools.  
    - Edge Cases: Empty or malformed text, API rate limits, unexpected tool responses.

  - **OpenAI Chat Model**  
    - Type: Language Model (OpenAI GPT)  
    - Role: Provides language generation capabilities for AI Agent.  
    - Config: Default OpenAI options; no explicit model override here.  
    - Input/Output: Connected as ai_languageModel to AI Agent.  
    - Edge Cases: Model unavailability, quota exceeded.

  - **Window Buffer Memory**  
    - Type: Memory Buffer (LangChain)  
    - Role: Maintains conversational context keyed by LINE userId.  
    - Config: Session key derived from `body.events[0].source.userId`.  
    - Input/Output: ai_memory interface with AI Agent.  
    - Edge Cases: Memory overflow, session key mismatches.

  - **Wikipedia**  
    - Type: LangChain Wikipedia Tool  
    - Role: Provides factual knowledge retrieval when requested by AI Agent.  
    - Config: Default Wikipedia API usage.  
    - Input/Output: ai_tool interface with AI Agent.  
    - Edge Cases: Network failures, ambiguous queries.

  - **OpenAI**  
    - Type: LangChain OpenAI Node  
    - Role: Post-processes AI Agent output to produce concise, user-friendly answers without extraneous details or links.  
    - Config: Uses model `gpt-4o-mini`, with a system prompt instructing answer condensation and forbidding links.  
    - Input: AI Agent output.  
    - Output: Cleaned AI response.  
    - Edge Cases: Model errors, prompt failures.

---

#### 2.3 External Data Retrieval

- **Overview:**  
  Fetches or creates calendar events and retrieves emails based on user queries parsed by AI Agent.

- **Nodes Involved:**  
  - Google Calendar Read  
  - Google Calendar Create  
  - Gmail Read

- **Node Details:**  

  - **Google Calendar Read**  
    - Type: Google Calendar Tool Node  
    - Role: Retrieves upcoming or specified events from Google Calendar.  
    - Config: Limits to 5 events; filters by `timeMin` and `timeMax` dynamically extracted from AI Agent using `$fromAI()` helper for "startdate" and "enddate". Uses OAuth2 credentials linked to configured Google Calendar account.  
    - Edge Cases: OAuth token expiration, invalid date ranges.

  - **Google Calendar Create**  
    - Type: Google Calendar Tool Node  
    - Role: Creates new calendar events with details extracted dynamically via AI.  
    - Config: Uses `$fromAI()` to get event start and end times and summary. Uses same OAuth2 credentials as Calendar Read.  
    - Edge Cases: Invalid event details, permission errors.

  - **Gmail Read**  
    - Type: Gmail Tool Node  
    - Role: Retrieves recent emails filtered by date extracted from AI Agent (`receiveddate` parameter).  
    - Config: Limits to 5 emails; filters by receivedAfter date dynamically from AI. Uses OAuth2 credentials for Gmail account.  
    - Edge Cases: Authentication failures, invalid filter formats.

---

#### 2.4 Response Handling & Delivery

- **Overview:**  
  Cleans AI-generated text responses, then sends them back to LINE users. Handles both normal and error reply cases.

- **Nodes Involved:**  
  - Error Handling from AI Response  
  - Text Cleansing  
  - Line Answering (Ordinary Case)  
  - Line Answering (Error Case)

- **Node Details:**  

  - **Error Handling from AI Response**  
    - Type: Switch  
    - Role: Determines if AI response is valid or empty, routing accordingly.  
    - Config: Checks existence of `message.content` in JSON output.  
    - Input: Output from OpenAI node.  
    - Output: Valid responses go to Text Cleansing; invalid to Line Answering (Error Case).  
    - Edge Cases: False negatives on valid content.

  - **Text Cleansing**  
    - Type: Set  
    - Role: Sanitizes AI response text by removing newlines, markdown, HTML tags, and quotes for clean LINE message.  
    - Config: Uses multiple chained string replacements and removals on `message.content`.  
    - Input: Valid AI response JSON.  
    - Output: Cleaned text to be sent.  
    - Edge Cases: Unexpected text formats causing expression failures.

  - **Line Answering (Ordinary Case)**  
    - Type: HTTP Request  
    - Role: Sends cleaned text response back to LINE platform using reply API.  
    - Config: POST to `https://api.line.me/v2/bot/message/reply` with JSON body containing replyToken and message text; Authorization header uses `LINE_API_TOKEN` environment variable.  
    - Input: Cleaned text from Text Cleansing.  
    - Output: None (end node for success path).  
    - Edge Cases: Network failures, invalid tokens, replyToken expiration.

  - **Line Answering (Error Case)**  
    - Type: HTTP Request  
    - Role: Sends default error message to LINE user when input or AI processing fails.  
    - Config: Same API endpoint as ordinary case; message text is fixed Thai phrase "กรุณาส่งอย่างอื่นเถอะนะเตงอัว" (Please send something else).  
    - Input: Routed from Switch nodes on failure conditions.  
    - Output: None (end node for failure path).  
    - Edge Cases: Same as ordinary case.

---

### 3. Summary Table

| Node Name                     | Node Type                        | Functional Role                         | Input Node(s)                  | Output Node(s)                  | Sticky Note                                         |
|-------------------------------|---------------------------------|---------------------------------------|-------------------------------|--------------------------------|-----------------------------------------------------|
| Line Receiving                | Webhook                         | Entry point for LINE webhook messages | n/a                           | Switch Between Text and Others |                                                     |
| Switch Between Text and Others | Switch                         | Routes text vs non-text messages       | Line Receiving                | AI Agent, Line Answering (Error Case) |                                                     |
| AI Agent                     | LangChain Agent                 | Processes user input with AI tools     | Switch Between Text and Others | OpenAI                        |                                                     |
| OpenAI Chat Model             | LangChain LM Chat OpenAI        | Language model for AI Agent            | AI Agent                      | AI Agent                      |                                                     |
| Window Buffer Memory          | LangChain Memory Buffer Window  | Maintains session memory               | AI Agent                      | AI Agent                      |                                                     |
| Wikipedia                    | LangChain Wikipedia Tool         | Provides knowledge retrieval           | AI Agent                      | AI Agent                      |                                                     |
| OpenAI                      | LangChain OpenAI Node            | Post-processes AI output                | AI Agent                      | Error Handling from AI Response |                                                     |
| Error Handling from AI Response| Switch                         | Checks for valid AI response           | OpenAI                       | Text Cleansing, Line Answering (Error Case) |                                                     |
| Text Cleansing               | Set                             | Cleans AI response text                 | Error Handling from AI Response | Line Answering (Ordinary Case) |                                                     |
| Line Answering (Ordinary Case)| HTTP Request                   | Sends reply to LINE for valid responses| Text Cleansing                | n/a                           |                                                     |
| Line Answering (Error Case)  | HTTP Request                   | Sends generic error reply to LINE      | Switch Between Text and Others, Error Handling from AI Response | n/a |                                                     |
| Google Calendar Read         | Google Calendar Tool            | Retrieves calendar events               | AI Agent                      | AI Agent                      |                                                     |
| Google Calendar Create       | Google Calendar Tool            | Creates calendar events                 | AI Agent                      | AI Agent                      |                                                     |
| Gmail Read                  | Gmail Tool                      | Retrieves emails                        | AI Agent                      | AI Agent                      |                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Webhook Node**  
   - Name: `Line Receiving`  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `linechatbotagent`  
   - Save the webhook URL and configure it in LINE Developer Console for messaging API.

2. **Add Switch Node to Filter Message Types**  
   - Name: `Switch Between Text and Others`  
   - Type: Switch  
   - Add a rule: If expression `{{$json.body.events[0].message.type}}` equals `"text"`, route to AI Agent; else route to error reply.  
   - Connect `Line Receiving` main output to this node.

3. **Setup AI Agent Node**  
   - Name: `AI Agent`  
   - Type: LangChain Agent  
   - Parameters:  
     - Text input: `{{$json.body.events[0].message.text}}`  
     - System message: `"You are a helpful assistant.\n\nHere is the current date {{ $now }}"`  
     - Prompt type: define  
   - Connect the "text" output from Switch node here.

4. **Configure AI Language Model Node**  
   - Name: `OpenAI Chat Model`  
   - Type: LangChain LM Chat OpenAI  
   - Use default parameters.  
   - Connect as `ai_languageModel` to `AI Agent`.

5. **Add Memory Buffer Node**  
   - Name: `Window Buffer Memory`  
   - Type: LangChain Memory Buffer Window  
   - Session Key: `{{$json.body.events[0].source.userId}}`  
   - Connect as `ai_memory` to `AI Agent`.

6. **Add Wikipedia Tool Node**  
   - Name: `Wikipedia`  
   - Type: LangChain Wikipedia Tool  
   - Connect as `ai_tool` to `AI Agent`.

7. **Add Google Calendar Read Node**  
   - Name: `Google Calendar Read`  
   - Type: Google Calendar Tool  
   - Operation: Get All  
   - Limit: 5  
   - TimeMin: `{{$fromAI("startdate","start date user mentioned about")}}`  
   - TimeMax: `{{$fromAI("enddate","end date user mentioned about")}}`  
   - Select your Google Calendar OAuth2 credential.  
   - Connect as `ai_tool` to `AI Agent`.

8. **Add Google Calendar Create Node**  
   - Name: `Google Calendar Create`  
   - Type: Google Calendar Tool  
   - Operation: Create Event  
   - Start: `{{$fromAI("createstartdate","start date and time to create event")}}`  
   - End: `{{$fromAI("createenddate","end date and time to create event")}}`  
   - Summary: `{{$fromAI("event_name","Name of an Event")}}`  
   - Use same Google Calendar OAuth2 credential.  
   - Connect as `ai_tool` to `AI Agent`.

9. **Add Gmail Read Node**  
   - Name: `Gmail Read`  
   - Type: Gmail Tool  
   - Operation: Get All  
   - Limit: 5  
   - Filter `receivedAfter`: `{{$fromAI("receiveddate","the date email received")}}`  
   - Use Gmail OAuth2 credential.  
   - Connect as `ai_tool` to `AI Agent`.

10. **Add OpenAI Post-Processing Node**  
    - Name: `OpenAI`  
    - Type: LangChain OpenAI  
    - Model: `gpt-4o-mini`  
    - System prompt: `"Your task is to extract and condense the answer into an easily readable format. Don't provide a link or details such as \"ดูเพิ่มเติม\" or \"ดูรายละเอียดได้ที่นี่.\""`  
    - Input content: `{{$json.output}}` (from AI Agent output)  
    - Connect `AI Agent` main output to this node.

11. **Add Switch Node for AI Response Validity**  
    - Name: `Error Handling from AI Response`  
    - Type: Switch  
    - Rule: Check if `{{$json.message.content}}` exists and is not empty.  
    - If valid: route to Text Cleansing.  
    - Else: route to Line Answering (Error Case).

12. **Add Text Cleansing Node**  
    - Name: `Text Cleansing`  
    - Type: Set  
    - Assignment:  
      - Field: `message.content`  
      - Value expression: `{{$json.message.content.replaceAll("\n","\\n").replaceAll("\n","").removeMarkdown().removeTags().replaceAll('"',"")}}`  
    - Connect valid output from Error Handling node here.

13. **Add Line Answering (Ordinary Case) Node**  
    - Name: `Line Answering (Ordinary Case)`  
    - Type: HTTP Request  
    - Method: POST  
    - URL: `https://api.line.me/v2/bot/message/reply`  
    - Headers:  
      - Authorization: `Bearer {{ $env.LINE_API_TOKEN }}`  
      - Content-Type: `application/json`  
    - Body (JSON):  
      ```json
      {
        "replyToken": "{{ $json.body.events[0].replyToken }}",
        "messages": [{
          "type": "text",
          "text": "{{ $json.message.content }}"
        }]
      }
      ```  
    - Connect output from Text Cleansing here.

14. **Add Line Answering (Error Case) Node**  
    - Name: `Line Answering (Error Case)`  
    - Type: HTTP Request  
    - Method: POST  
    - URL: `https://api.line.me/v2/bot/message/reply`  
    - Headers:  
      - Authorization: `Bearer {{ $env.LINE_API_TOKEN }}`  
      - Content-Type: `application/json`  
    - Body (JSON):  
      ```json
      {
        "replyToken": "{{ $json.body.events[0].replyToken }}",
        "messages": [{
          "type": "text",
          "text": "กรุณาส่งอย่างอื่นเถอะนะเตงอัว"
        }]
      }
      ```  
    - Connect both the "others" output of Switch Between Text and Others node and the error output of Error Handling from AI Response here.

15. **Credential Setup**  
    - Configure LINE API token as environment variable `LINE_API_TOKEN`.  
    - Setup Google Calendar OAuth2 credentials with proper scopes (calendar read/write).  
    - Setup Gmail OAuth2 credentials with read-only Gmail scopes.  
    - Setup OpenAI API key in n8n credentials.

16. **Test the Workflow**  
    - Send a text message via LINE to the configured webhook.  
    - Observe AI-generated response, calendar or email data retrieval, and reply delivery.

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                   |
|------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Sticky notes can be used inline in n8n to provide detailed instructions or reminders for each node configuration and usage.         | n8n UI feature                                                                                   |
| Video walkthrough or step-by-step documentation is recommended for first-time users to understand deployment and testing.          | Suggested enhancement for onboarding                                                             |
| Localization of AI replies can be done by modifying system messages or prompt templates in the AI Agent node to match target audience language. | Customization recommendation                                                                     |
| Adding integrations like Slack or Microsoft Teams can extend notification capabilities beyond LINE.                               | Suggested feature expansion                                                                       |
| Use centralized Edit Fields node to manage dynamic inputs such as tokens, emails, or thresholds for easier maintenance.            | Optimization tip                                                                                  |
| Implementing logging nodes can help with debugging and monitoring message flows and AI responses.                                  | Best practice for reliability                                                                     |
| Project inspired by bridging conversational AI with productivity platforms for small business automation.                         | Context                                                                                          |

---

This document fully describes the workflow's structure, logic, nodes configuration, and reproduction steps, providing clear guidance for users or automation agents to understand, modify, or rebuild the workflow.