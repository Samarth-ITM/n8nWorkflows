📊 AI Token Tracker for WhatsApp & Telegram – Save AI Usage to Google Sheets

https://n8nworkflows.xyz/workflows/---ai-token-tracker-for-whatsapp---telegram---save-ai-usage-to-google-sheets-3963


# 📊 AI Token Tracker for WhatsApp & Telegram – Save AI Usage to Google Sheets

---

### 1. Workflow Overview

This workflow, **AI Token Tracker for WhatsApp & Telegram – Save AI Usage to Google Sheets**, is designed to monitor and log AI token consumption and related metadata from conversations conducted via WhatsApp (using Evolution API) or Telegram bots. It captures detailed token usage (prompt, completion, total), timestamps, user/channel identifiers, and optionally estimates cost in USD. The data is appended to a structured Google Sheet, enabling ongoing tracking and cost control of AI usage.

The workflow is logically organized into the following blocks:

- **1.1 Input Reception and Source Identification:** Handles incoming messages from WhatsApp Evolution API webhook and Telegram trigger, distinguishes message types (audio/text).
- **1.2 Media Processing and Transcription:** For audio inputs, fetches media, converts to file, and transcribes audio to text.
- **1.3 Text Preparation and Filtering:** Normalizes and maps input text, filters valid inputs for AI processing.
- **1.4 AI Processing:** Sends the prepared input to an AI language model (LangChain agent integrated with OpenRouter GPT-4o), handles AI responses, and error management.
- **1.5 Usage Logging and Cleanup:** Calculates token usage, cleans data, and appends records to Google Sheets for both successful and error cases.
- **1.6 Response Handling:** Sends AI responses or error notifications back to Telegram or WhatsApp users.
- **1.7 Optional Tools and Disabled Nodes:** Some nodes like email sending, calendar event creation, and contacts fetching are included but disabled, possibly for future expansions or alternative workflows.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Source Identification

- **Overview:**  
Receives incoming messages from WhatsApp (Evolution API webhook) and Telegram bot trigger. It distinguishes the input source and branches accordingly for processing.

- **Nodes Involved:**  
  - Evolution API (Webhook)  
  - Telegram (Telegram Trigger)  
  - Map (for WhatsApp message mapping)  
  - Map1 (for Telegram message mapping)  
  - Input Source (Switch node)  
  - Input Source1 (Switch node for error routing)

- **Node Details:**

  - **Evolution API**  
    - Type: Webhook (HTTP trigger for WhatsApp messages via Evolution API)  
    - Config: Listens for incoming HTTP POST requests from WhatsApp messages.  
    - Input: HTTP POST from Evolution API.  
    - Output: Passes raw WhatsApp message data to **Map** node.  
    - Edge cases: Could fail with missing or malformed webhook data, Evolution API downtime.

  - **Telegram**  
    - Type: Telegram Trigger (listens for messages from Telegram bot)  
    - Config: Configured with Telegram Bot Token credential; triggers on new Telegram messages.  
    - Input: Telegram messages.  
    - Output: Passes data to **Map1** for normalization.  
    - Edge cases: Invalid bot token, Telegram API issues, message format variations.

  - **Map** (WhatsApp)  
    - Type: Set node  
    - Role: Normalizes and extracts relevant WhatsApp message fields (e.g., text, user, timestamp) for downstream processing.  
    - Input: Raw webhook data from Evolution API node.  
    - Output: To **Audio or Text Separation** node.

  - **Map1** (Telegram)  
    - Type: Set node  
    - Role: Normalizes and extracts relevant Telegram message fields (text, user ID, chat ID, timestamp).  
    - Input: Telegram trigger node output.  
    - Output: To **Text Mapping** node.

  - **Input Source**  
    - Type: Switch node  
    - Role: Routes data based on source or message content to either WhatsApp HTTP request node or Telegram response node.  
    - Input: From **Log** node (token logging).  
    - Output: To **WhatsApp** HTTP Request node or **Response** Telegram node.  
    - Edge cases: Misrouting if source fields are ambiguous or missing.

  - **Input Source1**  
    - Type: Switch node  
    - Role: Routes error log entries to either WhatsApp error HTTP request or Telegram error response.  
    - Input: From **Errors** Google Sheets node.  
    - Output: To **WhatsApp Erro** HTTP Request or **Error Response** Telegram node.

#### 1.2 Media Processing and Transcription

- **Overview:**  
Handles media messages by separating audio from text inputs, fetching media files, converting to file format for processing, and transcribing audio to text using OpenAI's Whisper via LangChain node.

- **Nodes Involved:**  
  - Audio or Text Separation (If node)  
  - GET Media (HTTP Request)  
  - Convert to File (ConvertToFile node)  
  - Audio Transcription (LangChain OpenAI node)

- **Node Details:**

  - **Audio or Text Separation**  
    - Type: If node  
    - Role: Checks if incoming message contains audio (voice note) or text.  
    - Input: From **Map** node (WhatsApp normalized message).  
    - Outputs: Audio messages routed to **GET Media**; text messages routed to **Text Mapping**.  
    - Edge cases: Ambiguous or missing media type fields.

  - **GET Media**  
    - Type: HTTP Request  
    - Role: Downloads audio media file from WhatsApp Evolution API URL.  
    - Input: Media URL from **Audio or Text Separation**.  
    - Output: Binary audio file data to **Convert to File**.  
    - Edge cases: HTTP errors, expired media URLs, authentication errors.

  - **Convert to File**  
    - Type: ConvertToFile  
    - Role: Converts downloaded binary data into a file format n8n can process.  
    - Input: Binary data from **GET Media**.  
    - Output: File data to **Audio Transcription**.  
    - Edge cases: File conversion failures, incorrect mime types.

  - **Audio Transcription**  
    - Type: LangChain OpenAI node (OpenAI Whisper)  
    - Role: Transcribes audio file content to text string using OpenAI Whisper or equivalent.  
    - Input: Audio file from **Convert to File**.  
    - Output: Transcribed text to **Text Mapping**.  
    - Edge cases: Transcription API limits, timeouts, language detection failures.

#### 1.3 Text Preparation and Filtering

- **Overview:**  
Prepares text input (either from transcription or direct text messages), maps necessary fields, and filters messages to determine if they qualify for AI processing.

- **Nodes Involved:**  
  - Text Mapping (Set node)  
  - Filter (Filter node)

- **Node Details:**

  - **Text Mapping**  
    - Type: Set node  
    - Role: Standardizes text message data structure: input text, user info, timestamp, etc.  
    - Input: From **Audio Transcription** or **Map1** (Telegram).  
    - Output: To **Filter** node.  
    - Edge cases: Missing text or user fields.

  - **Filter**  
    - Type: Filter node  
    - Role: Applies criteria to decide if the message should be sent to AI processing (e.g., non-empty text).  
    - Input: From **Text Mapping**.  
    - Output: Yes branch to **AI Agent**, No branch discarded.  
    - Edge cases: Incorrect filter conditions could block valid inputs.

#### 1.4 AI Processing

- **Overview:**  
Sends filtered text inputs to the AI Agent (LangChain agent) which uses OpenRouter GPT-4o or other configured language model to generate responses. Handles error continuation and branches accordingly.

- **Nodes Involved:**  
  - AI Agent (LangChain agent node)  
  - Chat Model (LM Chat OpenRouter node)

- **Node Details:**

  - **AI Agent**  
    - Type: LangChain Agent node  
    - Role: Core AI processing unit; sends prompt to configured AI model, receives completion, calculates token usage.  
    - Config: Uses LangChain agent with OpenRouter GPT-4o or other LLM. On error, continues with error output channel.  
    - Input: From **Filter** node (valid text inputs).  
    - Output: Success to **Clean Up**, error to **Clean_Up** node.  
    - Edge cases: API errors, token limits, model unavailability.

  - **Chat Model**  
    - Type: LM Chat OpenRouter  
    - Role: Language model node linked as AI backend for the AI Agent.  
    - Input: AI Agent node's language model interface.  
    - Output: Returns AI responses and token usage metadata.  
    - Edge cases: Rate limits, credential issues.

#### 1.5 Usage Logging and Cleanup

- **Overview:**  
Processes AI responses and errors to clean data, format logging entries, and append them to respective Google Sheets tabs for usage and error tracking.

- **Nodes Involved:**  
  - Clean Up (Code node)  
  - Clean_Up (Code node)  
  - Log (Google Sheets node)  
  - Errors (Google Sheets node)

- **Node Details:**

  - **Clean Up**  
    - Type: Code node  
    - Role: Processes successful AI response data, extracts token counts, calculates totals and costs, formats data for spreadsheet.  
    - Input: Success output from **AI Agent**.  
    - Output: To **Log** node.  
    - Edge cases: Script errors, missing data fields.

  - **Clean_Up**  
    - Type: Code node  
    - Role: Processes error outputs from AI Agent, formats error details for logging.  
    - Input: Error output from **AI Agent**.  
    - Output: To **Errors** node.  
    - Edge cases: Incomplete error data.

  - **Log**  
    - Type: Google Sheets node  
    - Role: Appends structured AI usage data (tokens, timestamps, user, cost estimates) to the main Google Sheet tab.  
    - Input: From **Clean Up** node.  
    - Output: To **Input Source** switch for response routing.  
    - Edge cases: Google API auth errors, quota exceeded.

  - **Errors**  
    - Type: Google Sheets node  
    - Role: Appends error information to a separate Google Sheets tab for error monitoring.  
    - Input: From **Clean_Up** node.  
    - Output: To **Input Source1** switch.  
    - Edge cases: Google Sheets write failures.

#### 1.6 Response Handling

- **Overview:**  
Sends the AI-generated response or error message back to users on Telegram or WhatsApp channels.

- **Nodes Involved:**  
  - Response (Telegram)  
  - WhatsApp (HTTP Request)  
  - Error Response (Telegram)  
  - WhatsApp Erro (HTTP Request)

- **Node Details:**

  - **Response**  
    - Type: Telegram node  
    - Role: Sends AI response text messages back to Telegram users.  
    - Input: From **Input Source** switch (success path).  
    - Output: None further downstream.  
    - Edge cases: Telegram API errors, invalid chat IDs.

  - **WhatsApp**  
    - Type: HTTP Request  
    - Role: Sends AI response messages back to WhatsApp via Evolution API HTTP endpoint.  
    - Input: From **Input Source** switch (success path).  
    - Output: None further downstream.  
    - Edge cases: HTTP errors, API authentication failures.

  - **Error Response**  
    - Type: Telegram node  
    - Role: Sends error messages to Telegram users when AI processing fails.  
    - Input: From **Input Source1** switch (error path).  
    - Output: None further downstream.  
    - Edge cases: Telegram API errors.

  - **WhatsApp Erro**  
    - Type: HTTP Request  
    - Role: Sends error messages back to WhatsApp users.  
    - Input: From **Input Source1** switch (error path).  
    - Output: None further downstream.  
    - Edge cases: HTTP request failures.

#### 1.7 Optional Tools and Disabled Nodes

- **Overview:**  
Nodes for additional AI tools (Send Email, Create Event, Get Contacts) and sticky notes are present but disabled or empty; these may be placeholders for extended functionality or notes for users.

- **Nodes Involved:**  
  - Send Email (disabled)  
  - Create Event (disabled)  
  - Get Contacts (disabled)  
  - Multiple Sticky Notes (various positions)

- **Node Details:**  
Disabled nodes are not active in the workflow execution. Sticky notes contain no content in provided data but might hold instructions or comments in n8n editor.

---

### 3. Summary Table

| Node Name            | Node Type                          | Functional Role                         | Input Node(s)                    | Output Node(s)                         | Sticky Note                  |
|----------------------|----------------------------------|--------------------------------------|---------------------------------|--------------------------------------|------------------------------|
| Evolution API        | Webhook                          | Receives WhatsApp messages            | —                               | Map                                  |                              |
| Telegram             | Telegram Trigger                 | Receives Telegram messages            | —                               | Map1                                 |                              |
| Map                  | Set                             | Normalizes WhatsApp message           | Evolution API                   | Audio or Text Separation              |                              |
| Map1                 | Set                             | Normalizes Telegram message           | Telegram                       | Text Mapping                         |                              |
| Audio or Text Separation | If                            | Separates audio from text messages    | Map                            | GET Media (audio) / Text Mapping (text) |                              |
| GET Media             | HTTP Request                    | Downloads audio media from WhatsApp   | Audio or Text Separation       | Convert to File                      |                              |
| Convert to File       | ConvertToFile                   | Converts binary to file format        | GET Media                     | Audio Transcription                  |                              |
| Audio Transcription   | LangChain OpenAI                | Transcribes audio to text             | Convert to File               | Text Mapping                        |                              |
| Text Mapping          | Set                             | Standardizes text message data        | Audio Transcription / Map1    | Filter                             |                              |
| Filter                | Filter                          | Filters messages for AI processing    | Text Mapping                  | AI Agent (yes)                     |                              |
| AI Agent              | LangChain Agent                 | Sends prompt to AI model, processes response | Filter                     | Clean Up (success), Clean_Up (error) |                              |
| Chat Model            | LM Chat OpenRouter              | Language model backend for AI Agent  | AI Agent (lmChat)             | AI Agent                           |                              |
| Clean Up              | Code                            | Processes AI responses for logging   | AI Agent (success)            | Log                               |                              |
| Clean_Up              | Code                            | Processes AI errors for logging      | AI Agent (error)              | Errors                            |                              |
| Log                   | Google Sheets                   | Appends token usage logs              | Clean Up                     | Input Source                      |                              |
| Errors                 | Google Sheets                   | Appends error logs                    | Clean_Up                     | Input Source1                     |                              |
| Input Source          | Switch                         | Routes logged data to WhatsApp or Telegram response | Log                        | WhatsApp / Response               |                              |
| Input Source1         | Switch                         | Routes error logs to WhatsApp or Telegram error response | Errors                     | WhatsApp Erro / Error Response    |                              |
| WhatsApp              | HTTP Request                   | Sends AI responses to WhatsApp       | Input Source                 | —                                 |                              |
| Response               | Telegram                       | Sends AI responses to Telegram       | Input Source                 | —                                 |                              |
| WhatsApp Erro          | HTTP Request                   | Sends error messages to WhatsApp     | Input Source1                | —                                 |                              |
| Error Response         | Telegram                       | Sends error messages to Telegram     | Input Source1                | —                                 |                              |
| Send Email             | Gmail Tool (disabled)           | (Disabled) Send emails as AI tool    | AI Agent (ai_tool)           | AI Agent                         | Disabled nodes not active    |
| Create Event           | Google Calendar Tool (disabled) | (Disabled) Create calendar events    | AI Agent (ai_tool)           | AI Agent                         | Disabled nodes not active    |
| Get Contacts           | Airtable Tool (disabled)        | (Disabled) Get contacts data          | AI Agent (ai_tool)           | AI Agent                         | Disabled nodes not active    |
| Sticky Notes           | Sticky Note                    | Comments and notes                    | —                           | —                                 | Empty in provided data       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node for WhatsApp Evolution API**  
   - Node Type: Webhook  
   - Configure HTTP Method: POST  
   - Set path for receiving Evolution API messages.  
   - No authentication by default; secure via external Evolution API config.

2. **Create Telegram Trigger Node**  
   - Node Type: Telegram Trigger  
   - Configure with Telegram Bot Token credential.  
   - Trigger on new messages.

3. **Add Set Node "Map" for WhatsApp Message Normalization**  
   - Extract message text, user ID, timestamp, media type, media URL.  
   - Output standardized JSON for downstream nodes.

4. **Add Set Node "Map1" for Telegram Message Normalization**  
   - Extract message text, chat ID, user ID, timestamp.  
   - Output standardized JSON.

5. **Add If Node "Audio or Text Separation"**  
   - Condition: Check if message contains audio media.  
   - True branch: Audio path; False branch: Text path.

6. **Add HTTP Request Node "GET Media"**  
   - HTTP GET to download audio file from WhatsApp media URL.  
   - Use credentials or headers if needed.

7. **Add ConvertToFile Node "Convert to File"**  
   - Convert downloaded binary response to file object.

8. **Add LangChain OpenAI Node "Audio Transcription"**  
   - Use OpenAI Whisper model or equivalent.  
   - Input: Audio file from previous node.  
   - Output: Transcribed text.

9. **Add Set Node "Text Mapping"**  
   - Standardize message text and metadata (user, timestamp).  
   - Input: From transcription or Telegram mapping.

10. **Add Filter Node "Filter"**  
    - Condition: Check if text input is non-empty and valid.  
    - Yes branch proceeds; No branch discards.

11. **Add LangChain Agent Node "AI Agent"**  
    - Configure agent with OpenRouter GPT-4o or preferred model.  
    - Set onError to continue and route error output.  
    - Input: Filter node (valid texts).

12. **Add LM Chat OpenRouter Node "Chat Model"**  
    - Configure with OpenRouter API credentials.  
    - Connect as language model backend for AI Agent.

13. **Add Code Node "Clean Up"**  
    - Extract token usage details (prompt, completion, total).  
    - Calculate cost if cost per 1K tokens is provided.  
    - Format data for logging.

14. **Add Code Node "Clean_Up"**  
    - Format AI errors for logging.

15. **Add Google Sheets Node "Log"**  
    - Append data to tokens usage spreadsheet tab.  
    - Configure with Google Sheets OAuth2 credentials.  
    - Specify spreadsheet ID and worksheet name.

16. **Add Google Sheets Node "Errors"**  
    - Append error records to dedicated error tab in spreadsheet.

17. **Add Switch Node "Input Source"**  
    - Route log entries to WhatsApp or Telegram response nodes based on message source.

18. **Add Switch Node "Input Source1"**  
    - Route error log entries to WhatsApp or Telegram error response nodes.

19. **Add HTTP Request Node "WhatsApp"**  
    - POST response messages back to WhatsApp via Evolution API endpoint.

20. **Add Telegram Node "Response"**  
    - Send AI response to Telegram users.

21. **Add HTTP Request Node "WhatsApp Erro"**  
    - POST error messages back to WhatsApp users.

22. **Add Telegram Node "Error Response"**  
    - Send error messages to Telegram users.

23. **Optional:** Add disabled nodes for email, calendar, contacts if needed.

24. **Add Sticky Notes** as necessary for workflow documentation in n8n editor.

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                                                                     |
|---------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Free Google Sheets template for logging AI token usage, including columns for date, input/output, tokens, cost estimates.      | [Google Sheets Template](https://docs.google.com/spreadsheets/d/1_HHTTT_gXEaH-LtZxNycSXeGGYE7dcS9vbuad0dZcnw/edit?usp=sharing) |
| Workflow designed by Amanda from iloveflows.com, specializing in AI automations.                                               | [https://iloveflows.com](https://iloveflows.com)                                                   |
| Recommended n8n Cloud platform link for easy deployment of workflows.                                                          | [https://n8n.partnerlinks.io/amanda](https://n8n.partnerlinks.io/amanda)                           |
| This workflow requires OpenAI GPT-4o access and appropriate API credentials for Telegram, Evolution API, and Google Sheets.  |                                                                                                   |
| The workflow supports both Telegram and WhatsApp inputs, with flexible routing and error handling for robust operation.       |                                                                                                   |

---

This structured reference document offers detailed insight into the workflow’s purpose, node-by-node analysis, reproduction steps, and key contextual resources for efficient use, modification, or automation with n8n.