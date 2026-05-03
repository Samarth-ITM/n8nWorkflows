Track & Query Expenses via Telegram with GPT-4.1 Mini & Google Sheets

https://n8nworkflows.xyz/workflows/track---query-expenses-via-telegram-with-gpt-4-1-mini---google-sheets-10290


# Track & Query Expenses via Telegram with GPT-4.1 Mini & Google Sheets

### 1. Workflow Overview

This workflow automates expense tracking and querying via Telegram, supporting both voice and text inputs. It leverages GPT-4.1 Mini AI to analyze user messages, integrates with Google Sheets for data storage and querying, uses AssemblyAI for voice transcription, and sends notifications or alerts through Telegram and Gmail. The workflow is designed to let users log expenses or ask about their spending history conversationally, with voice messages transcribed automatically.

**Logical Blocks:**

- **1.1 Input Reception and Routing:** Receives Telegram messages (voice or text), determines message type, and routes accordingly.
- **1.2 Voice Message Processing:** Handles voice quality check, downloads voice file, uploads it to AssemblyAI, transcribes, waits for results, and retrieves the transcription.
- **1.3 Text Message Processing:** Sends immediate processing notification and proceeds to read transaction history.
- **1.4 Expense Data Management:** Reads Google Sheets transaction history, calculates starting balance, appends new transactions, and reads data for queries.
- **1.5 AI Analysis and Response Generation:** Uses GPT-4.1 Mini and LangChain agents with conversation memory to analyze expenses, answer queries, and track API costs.
- **1.6 Response Delivery:** Generates voice responses, sends voice and text replies back to Telegram.
- **1.7 Alerts and Notifications:** Sends low balance alerts via Gmail.
- **1.8 Cost Tracking:** Tracks API usage costs and logs them into Google Sheets.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Routing

**Overview:**  
Receives incoming Telegram messages and determines whether the message is voice or text to route it properly.

**Nodes Involved:**  
- Telegram Input  
- Route by Message Type

**Node Details:**  

- **Telegram Input**  
  - Type: Telegram Trigger  
  - Role: Entry point listening for all incoming Telegram messages (voice or text).  
  - Config: Default Telegram bot webhook trigger, no filters specified.  
  - Input: Telegram user message  
  - Output: Passes message data to routing node  
  - Failure types: Telegram API downtime, webhook misconfiguration.

- **Route by Message Type**  
  - Type: Switch  
  - Role: Checks message type (voice or text) to route accordingly.  
  - Config: Switch conditions likely based on message properties (e.g., presence of voice data).  
  - Input: Telegram Input node output  
  - Output: Two branches - one for voice, one for text.  
  - Failure types: Incorrect message metadata causing wrong routing.

---

#### 1.2 Voice Message Processing

**Overview:**  
Processes voice messages: verifies voice quality, downloads the audio file, uploads it to AssemblyAI for transcription, waits for transcription, and retrieves results.

**Nodes Involved:**  
- Check Voice Quality  
- Send Processing Notification (Voice)  
- 🎙️ Download Voice File  
- Upload to AssemblyAI  
- Start Transcription  
- Wait for Transcription  
- Get Transcription Result  
- Check Transcription Status

**Node Details:**  

- **Check Voice Quality**  
  - Type: Switch  
  - Role: Validates if the voice message meets quality criteria before processing.  
  - Config: Conditions based on audio file properties, e.g., duration or format.  
  - Input: Output from Route by Message Type (voice branch)  
  - Output: Two branches: one sends a processing notification, the other proceeds to download voice file.  
  - Failure: Poor-quality or unsupported voice files.

- **Send Processing Notification (Voice)**  
  - Type: Telegram node  
  - Role: Sends a message to the user indicating that voice processing has started.  
  - Config: Sends predefined text notification.  
  - Input: Check Voice Quality (quality branch)  
  - Output: None further.  
  - Failure: Telegram API errors.

- **🎙️ Download Voice File**  
  - Type: Telegram node  
  - Role: Downloads the voice message file from Telegram servers.  
  - Config: Uses Telegram file ID from voice message metadata.  
  - Input: Check Voice Quality (successful quality branch)  
  - Output: Passes audio file to Upload to AssemblyAI.  
  - Failure: Network errors, Telegram file access issues.

- **Upload to AssemblyAI**  
  - Type: HTTP Request  
  - Role: Uploads the voice file to AssemblyAI for transcription.  
  - Config: HTTP POST with audio file payload, includes AssemblyAI API key in headers.  
  - Input: Downloaded audio file  
  - Output: Receives transcription job ID, forwards to Start Transcription.  
  - Failure: API auth errors, upload timeouts.

- **Start Transcription**  
  - Type: HTTP Request  
  - Role: Initiates transcription request at AssemblyAI with job ID.  
  - Config: HTTP POST or PATCH to start transcription.  
  - Input: Upload to AssemblyAI output  
  - Output: Forwards to Wait for Transcription node.  
  - Failure: API errors, invalid job ID.

- **Wait for Transcription**  
  - Type: Wait  
  - Role: Pauses workflow until transcription is likely completed.  
  - Config: Fixed or dynamic wait time (e.g., polling interval).  
  - Input: Start Transcription output  
  - Output: Triggers Get Transcription Result.  
  - Failure: Timeout if transcription takes too long.

- **Get Transcription Result**  
  - Type: HTTP Request  
  - Role: Retrieves transcription results from AssemblyAI.  
  - Config: HTTP GET with job ID to fetch status and result.  
  - Input: Wait for Transcription node  
  - Output: Passes full transcription data to Check Transcription Status.  
  - Failure: API errors, incomplete transcription data.

- **Check Transcription Status**  
  - Type: If  
  - Role: Checks if transcription is complete or still processing.  
  - Config: Condition on transcription status field (e.g., "completed").  
  - Input: Get Transcription Result output  
  - Output: If complete, proceeds to Read Transaction History; if not, loops back to Wait for Transcription.  
  - Failure: Infinite loop risk if transcription never completes.

---

#### 1.3 Text Message Processing

**Overview:**  
Handles incoming text messages by notifying the user that processing has started and reading the transaction history for further AI analysis.

**Nodes Involved:**  
- Send Processing Notification (Text)  
- Read Transaction History

**Node Details:**  

- **Send Processing Notification (Text)**  
  - Type: Telegram node  
  - Role: Sends a notification to the user indicating processing of their text message.  
  - Config: Predefined text sent immediately upon receiving a text message.  
  - Input: Route by Message Type (text branch)  
  - Output: Triggers Read Transaction History.  
  - Failure: Telegram API issues.

- **Read Transaction History**  
  - Type: Google Sheets  
  - Role: Reads all transaction records from the configured Google Sheet.  
  - Config: Reads full or filtered range of rows representing expense transactions.  
  - Input: Send Processing Notification (Text) or Check Transcription Status (voice branch)  
  - Output: Passes data to Calculate Starting Balance.  
  - Failure: Google Sheets API limits, permission issues.

---

#### 1.4 Expense Data Management

**Overview:**  
Processes expense data by calculating balances, appending new transactions, and reading data for AI queries.

**Nodes Involved:**  
- Calculate Starting Balance  
- Append Transaction to Sheet  
- Read Sheet for Queries

**Node Details:**  

- **Calculate Starting Balance**  
  - Type: Code  
  - Role: Computes the starting balance based on the transaction history data.  
  - Config: Custom JavaScript code to sum or manipulate transaction amounts.  
  - Input: Read Transaction History output  
  - Output: Feeds AI Expense Analyzer for contextual analysis.  
  - Failure: Code errors, unexpected data formats.

- **Append Transaction to Sheet**  
  - Type: Google Sheets Tool  
  - Role: Adds new transaction data into the expense tracking Google Sheet.  
  - Config: Appends rows with transaction details parsed from user input or transcription.  
  - Input: AI Expense Analyzer output (likely new expense entries)  
  - Output: Confirms data written, may feed back to AI.  
  - Failure: API write limits, data format mismatches.

- **Read Sheet for Queries**  
  - Type: Google Sheets Tool  
  - Role: Reads transaction data to answer user queries about past expenses or patterns.  
  - Config: Reads all rows or filtered subsets for AI query context.  
  - Input: AI Expense Analyzer requests data  
  - Output: Provides data to AI for query answering.  
  - Notes: Sticky Note attached: "Use this tool to read transaction data from the expense tracker sheet. You can retrieve all rows to answer questions about past transactions, spending patterns, categories, etc"  
  - Failure: API read limits, credential errors.

---

#### 1.5 AI Analysis and Response Generation

**Overview:**  
Uses GPT-4.1 Mini model with LangChain agents and conversation memory to analyze user input, generate insights or answers, and track API usage cost.

**Nodes Involved:**  
- GPT-4.1 Mini Model  
- Conversation Memory  
- AI Expense Analyzer  
- Track API Costs

**Node Details:**  

- **GPT-4.1 Mini Model**  
  - Type: LangChain OpenAI Chat Language Model  
  - Role: Processes user messages, providing AI-generated responses.  
  - Config: OpenAI GPT-4.1 Mini configured with relevant prompt/context.  
  - Input: Conversation Memory output  
  - Output: Feeds AI Expense Analyzer.  
  - Failure: OpenAI API auth errors, rate limits.

- **Conversation Memory**  
  - Type: LangChain Memory Buffer Window  
  - Role: Maintains conversation context over multiple interactions for coherent AI responses.  
  - Config: Sliding window memory with configurable size.  
  - Input: AI Expense Analyzer or GPT-4.1 Mini model  
  - Output: Feeds GPT-4.1 Mini model  
  - Failure: Memory overflows or data corruption.

- **AI Expense Analyzer**  
  - Type: LangChain Agent  
  - Role: Coordinates AI tasks such as analyzing expenses, formulating queries, or deciding actions like appending transactions or sending alerts.  
  - Config: Agent configured with tools (Google Sheets reading/appending, Gmail alerts).  
  - Input: Calculate Starting Balance and GPT-4.1 Mini model  
  - Output: Generates commands to append transactions, send alerts, or generate responses.  
  - Failure: Agent misconfiguration, tool integration errors.

- **Track API Costs**  
  - Type: Code  
  - Role: Calculates and tracks API usage cost for transparency and monitoring.  
  - Config: Custom code to parse usage metrics and estimate costs.  
  - Input: AI Expense Analyzer output  
  - Output: Logs cost data to Google Sheets.  
  - Failure: Calculation errors, missing usage data.

---

#### 1.6 Response Delivery

**Overview:**  
Generates voice responses from AI output and sends both voice and text messages back to the user via Telegram.

**Nodes Involved:**  
- Generate Voice Response  
- Send Voice to Telegram  
- Send Text to Telegram

**Node Details:**  

- **Generate Voice Response**  
  - Type: HTTP Request  
  - Role: Converts AI-generated text response into voice/audio format (likely via TTS API).  
  - Config: Sends text to TTS service, receives audio file.  
  - Input: AI Expense Analyzer output  
  - Output: Audio file for Telegram voice message  
  - Failure: TTS API errors, audio format mismatches.

- **Send Voice to Telegram**  
  - Type: Telegram node  
  - Role: Sends generated voice message back to the user.  
  - Config: Uses Telegram API to upload and send audio file.  
  - Input: Generate Voice Response output  
  - Output: None further.  
  - Failure: Telegram API errors or file size limits.

- **Send Text to Telegram**  
  - Type: Telegram node  
  - Role: Sends textual response message back to the user.  
  - Config: Sends text message with AI-generated response.  
  - Input: Generate Voice Response output (parallel)  
  - Output: None further.  
  - Failure: Telegram API issues.

---

#### 1.7 Alerts and Notifications

**Overview:**  
Sends email alerts (e.g., low balance warnings) using Gmail when triggered by AI analysis.

**Nodes Involved:**  
- Send Low Balance Alert

**Node Details:**  

- **Send Low Balance Alert**  
  - Type: Gmail Tool  
  - Role: Sends an email alert to predefined recipients when balance is low.  
  - Config: Uses Gmail OAuth2 credentials, predefined email template or dynamic content.  
  - Input: AI Expense Analyzer (alert trigger)  
  - Output: None further.  
  - Failure: Gmail API quota limits, auth errors.

---

#### 1.8 Cost Tracking

**Overview:**  
Logs API usage costs into Google Sheets for tracking and monitoring purposes.

**Nodes Involved:**  
- Log Cost to Sheet

**Node Details:**  

- **Log Cost to Sheet**  
  - Type: Google Sheets  
  - Role: Appends API cost data to a dedicated sheet for record-keeping.  
  - Config: Target Google Sheet and range specified for cost logging.  
  - Input: Track API Costs output  
  - Output: None further.  
  - Failure: Google Sheets API write limits, permission errors.

---

### 3. Summary Table

| Node Name                      | Node Type                             | Functional Role                         | Input Node(s)                       | Output Node(s)                           | Sticky Note                                                                                                  |
|--------------------------------|-------------------------------------|---------------------------------------|-----------------------------------|-----------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Telegram Input                 | Telegram Trigger                    | Entry point for Telegram messages     | -                                 | Route by Message Type                    |                                                                                                              |
| Route by Message Type          | Switch                             | Routes message by voice or text       | Telegram Input                    | Check Voice Quality, Send Processing Notification (Text) |                                                                                                              |
| Check Voice Quality            | Switch                             | Validates voice message quality       | Route by Message Type             | Send Processing Notification (Voice), 🎙️ Download Voice File |                                                                                                              |
| Send Processing Notification (Voice) | Telegram                         | Notifies user voice processing start  | Check Voice Quality              | -                                       |                                                                                                              |
| 🎙️ Download Voice File          | Telegram                          | Downloads voice message audio file    | Check Voice Quality              | Upload to AssemblyAI                    |                                                                                                              |
| Upload to AssemblyAI           | HTTP Request                      | Uploads audio for transcription       | 🎙️ Download Voice File           | Start Transcription                     |                                                                                                              |
| Start Transcription            | HTTP Request                      | Initiates transcription job           | Upload to AssemblyAI             | Wait for Transcription                  |                                                                                                              |
| Wait for Transcription         | Wait                             | Waits for transcription completion    | Start Transcription              | Get Transcription Result                |                                                                                                              |
| Get Transcription Result       | HTTP Request                     | Fetches transcription result          | Wait for Transcription           | Check Transcription Status              |                                                                                                              |
| Check Transcription Status     | If                               | Checks if transcription is complete   | Get Transcription Result         | Read Transaction History, Wait for Transcription |                                                                                                              |
| Send Processing Notification (Text) | Telegram                         | Notifies user text processing start   | Route by Message Type            | Read Transaction History                |                                                                                                              |
| Read Transaction History       | Google Sheets                    | Reads expense transactions             | Send Processing Notification (Text), Check Transcription Status | Calculate Starting Balance              |                                                                                                              |
| Calculate Starting Balance     | Code                            | Calculates starting balance            | Read Transaction History         | AI Expense Analyzer                     |                                                                                                              |
| GPT-4.1 Mini Model            | LangChain LM Chat OpenAI          | AI language model for analysis        | Conversation Memory              | AI Expense Analyzer                     |                                                                                                              |
| Conversation Memory            | LangChain Memory Buffer Window    | Maintains conversation context        | AI Expense Analyzer              | GPT-4.1 Mini Model                      |                                                                                                              |
| AI Expense Analyzer            | LangChain Agent                  | Coordinates AI expense analysis        | Calculate Starting Balance, GPT-4.1 Mini Model, Conversation Memory | Generate Voice Response, Track API Costs, Append Transaction to Sheet, Read Sheet for Queries, Send Low Balance Alert |                                                                                                              |
| Append Transaction to Sheet    | Google Sheets Tool               | Adds new expense transactions          | AI Expense Analyzer             | -                                       |                                                                                                              |
| Read Sheet for Queries         | Google Sheets Tool               | Reads data for AI queries               | AI Expense Analyzer             | AI Expense Analyzer                     | Use this tool to read transaction data from the expense tracker sheet. You can retrieve all rows to answer questions about past transactions, spending patterns, categories, etc |
| Send Low Balance Alert         | Gmail Tool                      | Sends low balance email alerts         | AI Expense Analyzer             | -                                       |                                                                                                              |
| Generate Voice Response        | HTTP Request                    | Converts text responses to voice       | AI Expense Analyzer             | Send Voice to Telegram, Send Text to Telegram |                                                                                                              |
| Send Voice to Telegram         | Telegram                        | Sends voice message back to user       | Generate Voice Response         | -                                       |                                                                                                              |
| Send Text to Telegram          | Telegram                        | Sends text message back to user        | Generate Voice Response         | -                                       |                                                                                                              |
| Track API Costs                | Code                            | Tracks API usage costs                  | AI Expense Analyzer             | Log Cost to Sheet                       |                                                                                                              |
| Log Cost to Sheet              | Google Sheets                  | Logs API usage cost data                | Track API Costs                 | -                                       |                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Input Node**  
   - Type: Telegram Trigger  
   - Configure with your Telegram Bot credentials  
   - Default settings to listen to all incoming messages (voice and text)  

2. **Add Switch Node "Route by Message Type"**  
   - Connect Telegram Input → Route by Message Type  
   - Configure switch to detect if incoming message includes a voice message or text  
   - One output for voice, one for text  

3. **Voice Message Branch**:  

   3.1 **Add Switch Node "Check Voice Quality"**  
       - Connect Route by Message Type (voice) → Check Voice Quality  
       - Configure conditions on voice message quality parameters (e.g., length, format)  
       - Two outputs: Low quality and acceptable quality  

   3.2 **Send Processing Notification (Voice)**  
       - Telegram node connected to Check Voice Quality low-quality branch  
       - Message to user: "Processing your voice message..."  

   3.3 **🎙️ Download Voice File**  
       - Telegram node connected to Check Voice Quality acceptable-quality branch  
       - Configure to download voice file using file ID from message  

   3.4 **Upload to AssemblyAI**  
       - HTTP Request node connected to Download Voice File  
       - Configure POST to AssemblyAI upload endpoint  
       - Include AssemblyAI API key in headers  

   3.5 **Start Transcription**  
       - HTTP Request node connected to Upload to AssemblyAI  
       - Configure to start transcription job with returned upload URL/job ID  

   3.6 **Wait for Transcription**  
       - Wait node connected to Start Transcription  
       - Set appropriate wait time (e.g., 10-15 seconds)  

   3.7 **Get Transcription Result**  
       - HTTP Request node connected to Wait for Transcription  
       - Configure GET request to AssemblyAI transcription endpoint with job ID  

   3.8 **Check Transcription Status**  
       - If node connected to Get Transcription Result  
       - Condition: If transcription status is "completed"  
       - True branch → Read Transaction History (step 5)  
       - False branch → Loop back to Wait for Transcription node  

4. **Text Message Branch**:  

   4.1 **Send Processing Notification (Text)**  
       - Telegram node connected to Route by Message Type (text branch)  
       - Message: "Processing your text message..."  

5. **Read Transaction History**  
   - Google Sheets node connected to both Send Processing Notification (Text) and Check Transcription Status (completed branch)  
   - Configure with Google Sheets credentials  
   - Read all rows from the expenses tracking sheet  

6. **Calculate Starting Balance**  
   - Code node connected to Read Transaction History  
   - JavaScript logic to sum transactions and compute starting balance  

7. **Setup AI Components:**  

   7.1 **Conversation Memory**  
       - LangChain Memory Buffer Window node configured with desired window size for conversation context  

   7.2 **GPT-4.1 Mini Model**  
       - LangChain LM Chat OpenAI node  
       - Configure with OpenAI API key and GPT-4.1 Mini model  
       - Connect Conversation Memory output to this node  

   7.3 **AI Expense Analyzer**  
       - LangChain Agent node  
       - Configure with tools:  
          - Google Sheets Tool nodes for reading sheet (queries) and appending transactions  
          - Gmail Tool for alerts  
       - Connect inputs from Calculate Starting Balance and GPT-4.1 Mini Model  

8. **Expense Data Management:**  

   8.1 **Append Transaction to Sheet**  
       - Google Sheets Tool node  
       - Configure to append rows to expenses sheet  
       - Connect from AI Expense Analyzer  

   8.2 **Read Sheet for Queries**  
       - Google Sheets Tool node  
       - Configure to read transaction data for AI queries  
       - Connect as AI Expense Analyzer tool  

   8.3 **Send Low Balance Alert**  
       - Gmail Tool node  
       - Configure with Gmail OAuth2 credentials  
       - Set up email template for low balance alerts  
       - Connect as AI Expense Analyzer tool  

9. **Response Generation and Delivery:**  

   9.1 **Generate Voice Response**  
       - HTTP Request node  
       - Configure to use a TTS API, sending AI-generated text and receiving audio file  

   9.2 **Send Voice to Telegram**  
       - Telegram node  
       - Configure to send voice message back to user  
       - Connect from Generate Voice Response  

   9.3 **Send Text to Telegram**  
       - Telegram node  
       - Configure to send textual message reply  
       - Connect from Generate Voice Response (in parallel with voice send)  

10. **API Cost Tracking:**  

    10.1 **Track API Costs**  
        - Code node  
        - Implement logic to calculate API usage costs from AI Expense Analyzer output  

    10.2 **Log Cost to Sheet**  
        - Google Sheets node  
        - Append cost data to a dedicated sheet for monitoring  
        - Connect from Track API Costs  

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                      |
|-----------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| Use "Read Sheet for Queries" node to retrieve all transaction rows for answering spending questions | Sticky Note attached to "Read Sheet for Queries" node |
| Workflow supports both voice and text input via Telegram for flexible expense tracking               | -                                                   |
| AssemblyAI used for accurate voice transcription with API key required                              | AssemblyAI API docs: https://www.assemblyai.com/docs |
| GPT-4.1 Mini model via LangChain provides conversational AI expense analysis                        | OpenAI API documentation: https://platform.openai.com/docs/models |
| Gmail node requires OAuth2 setup with Google credentials for sending alerts                          | Google OAuth2 docs: https://developers.google.com/identity/protocols/oauth2 |
| Telegram Bot must be configured with proper webhook and permissions for voice file download/send    | Telegram Bot API: https://core.telegram.org/bots/api   |

---

This documentation fully describes the workflow’s structure, logic, and integration points, enabling reproduction, modification, and error anticipation by developers and AI agents alike.

---

**Disclaimer:**  
The provided text is exclusively from an n8n automated workflow. It complies strictly with content policies and includes only legal, public data.