Daily WhatsApp Group Intelligence with GPT-4.1 Analysis & Voice Message Transcription

https://n8nworkflows.xyz/workflows/daily-whatsapp-group-intelligence-with-gpt-4-1-analysis---voice-message-transcription-8442


# Daily WhatsApp Group Intelligence with GPT-4.1 Analysis & Voice Message Transcription

---

### 1. Workflow Overview

This n8n workflow, titled **"Daily WhatsApp Group Intelligence with GPT-4.1 Analysis & Voice Message Transcription"**, automates the process of capturing, transcribing, storing, and analyzing WhatsApp group conversations. It targets users such as tech teams, business intelligence professionals, community managers, and research teams who want to transform group chats into actionable business intelligence through daily AI-powered summaries.

**Logical Blocks:**

- **1.1 Real-Time Message Capture & Processing**  
  Captures incoming WhatsApp messages from multiple groups via a webhook, filters by group, transcribes voice messages, and stores all messages in corresponding Google Sheets tabs.

- **1.2 Daily Intelligence Generation & AI Analysis**  
  Runs on a schedule to extract the previous day’s messages from Google Sheets, normalizes and aggregates the data, sends it to an AI agent that generates a structured summary focusing on trends and opportunities, formats the output, splits it into WhatsApp-friendly message fragments, and sends the final summary back to a WhatsApp group.

- **1.3 Configuration & Customization**  
  Includes setup instructions, group ID configurations, Google Sheets setup, and AI prompt customization.

---

### 2. Block-by-Block Analysis

#### 2.1 Real-Time Message Capture & Processing

**Overview:**  
This block receives WhatsApp messages in real time through a webhook integrated with the Evolution API (WhatsApp Business). It filters messages from specified groups, transcribes voice messages using OpenAI, and stores all messages in Google Sheets organized by group.

**Nodes Involved:**  
- Webhook-EVO  
- Set Info  
- Check if the message is from the Group (If node)  
- If (to detect audio messages)  
- Download audio  
- Convert audio  
- OpenAI (for transcription)  
- Audio content (Set node)  
- Set Info1  
- Switch (routes messages by group)  
- Save Messages (3 separate Google Sheets nodes for each group)  
- No hacer nada (No Operation node for messages outside monitored groups)

**Node Details:**

- **Webhook-EVO**  
  - Type: Webhook (HTTP POST)  
  - Role: Entry point receiving WhatsApp message payloads from Evolution API.  
  - Configuration: Path set to uniquely identify webhook; expects POST.  
  - Inputs: Incoming HTTP request with message data.  
  - Outputs: Passes message JSON to next node.  
  - Failure Modes: Missing webhook setup in Evolution API; auth errors; malformed payloads.

- **Set Info**  
  - Type: Set  
  - Role: Extracts and assigns variables from the incoming webhook JSON, including group IDs, message content, sender name, timestamps, captions, and message type.  
  - Key Expressions: Uses expressions like `={{ $json.body.data.key.remoteJid }}` to extract fields.  
  - Inputs: Webhook-EVO output.  
  - Outputs: Structured JSON with extracted fields for filtering.

- **Check if the message is from the Group (If node)**  
  - Type: If  
  - Role: Filters messages so only those from configured groups continue.  
  - Condition: Compares received group ID with stored group IDs `grupo_1`, `grupo_2`, `grupo_3`.  
  - Failure Modes: Misconfigured group IDs cause valid messages to be ignored.

- **If (audio detection)**  
  - Type: If  
  - Role: Checks if incoming message is an audio message (`messageType == "audioMessage"`).  
  - Outputs: Audio messages go to transcription branch; others go directly to storage.

- **Download audio**  
  - Type: HTTP Request  
  - Role: Downloads audio file from Evolution API server using message ID and API key.  
  - Configuration: POST request with message key and disables MP4 conversion.  
  - Inputs: Audio message detected by If node.  
  - Failure Modes: Network errors, API key invalid, URL changes.

- **Convert audio**  
  - Type: Convert to File  
  - Role: Converts downloaded base64 audio to binary for transcription.  
  - Configuration: MIME type from downloaded data.  
  - Inputs: Download audio output.

- **OpenAI (transcription)**  
  - Type: OpenAI Audio Transcription  
  - Role: Transcribes audio file to text using OpenAI API.  
  - Credentials: Requires valid OpenAI API key.  
  - Inputs: Binary audio from Convert audio.  
  - Failure Modes: API rate limits, malformed audio, transcription errors.

- **Audio content (Set node)**  
  - Type: Set  
  - Role: Stores transcribed text in `content` field for further processing.  
  - Inputs: OpenAI transcription output.

- **Set Info1**  
  - Type: Set  
  - Role: Re-assigns message variables, replacing audio content with transcribed text.  
  - Inputs: Audio content or direct text messages.  
  - Outputs: Prepares data for group routing.

- **Switch**  
  - Type: Switch  
  - Role: Routes messages to one of three Google Sheets nodes based on group ID.  
  - Configuration: Checks `received_group` against `grupo_1`, `grupo_2`, `grupo_3`.  
  - Failure Modes: Incorrect routing if group IDs are misconfigured.

- **Save Messages / Save Messages1 / Save Messages2**  
  - Type: Google Sheets Append  
  - Role: Append message data to the corresponding Google Sheet tab for each group.  
  - Configuration: Maps fields such as Date, Time, Name, Caption, Message, Replied Message.  
  - Credentials: Google Sheets OAuth2 API with proper permissions.  
  - Inputs: Switched group message.  
  - Failure Modes: Permissions errors, document ID or sheet name misconfiguration, API quota limits.

- **No hacer nada**  
  - Type: No Operation  
  - Role: Terminates flow for messages outside monitored groups.

---

#### 2.2 Daily Intelligence Generation & AI Analysis

**Overview:**  
Triggered by a schedule at 00:01 daily, this block extracts the previous day’s conversations from Google Sheets, normalizes and aggregates the data, sends it to the AI agent (WhatsOn 🕵🏾‍♂️) for insightful analysis, formats the AI output into WhatsApp-compatible messages, and sends the summary to a designated WhatsApp group.

**Nodes Involved:**  
- Send Summary every day at 00.00h (Schedule Trigger)  
- Extract today's conversations (Google Sheets) x3  
- Group conversations (Aggregate) x3  
- Normalize data (Code) x3  
- Merge (Merge)  
- Group all conversations (Aggregate)  
- AI Agent (Langchain Agent)  
- Set Resumen (Set)  
- OpenAI 4.1 Mini1 (Language Model)  
- JSON parse (Output Parser Autofixing)  
- Output Formatter (Chain LLM)  
- Set the JSON Return Type (Output Parser Structured)  
- Split Out (Split Out)  
- Loop Over Items (Split In Batches)  
- Send message (Evolution API)  
- Wait (Wait)

**Node Details:**

- **Send Summary every day at 00.00h**  
  - Type: Schedule Trigger  
  - Role: Initiates daily extraction process.  
  - Configuration: Runs at 00:01 am Europe/Madrid timezone.

- **Extract today's conversations / Extract today's conversations1 / Extract today's conversations2**  
  - Type: Google Sheets Read  
  - Role: Extracts messages from each group’s sheet for the previous day (filter by date).  
  - Configuration: Filter by date `={{ $now.minus({ days: 1 }).format('dd/LL/yyyy') }}`.  
  - Inputs: Trigger output.  
  - Failure Modes: Google Sheets API errors, incorrect document ID or sheet name.

- **Group conversations / Group conversations1 / Group conversations2**  
  - Type: Aggregate  
  - Role: Aggregates all extracted rows into a single array for each group.

- **Normalize data / Normalize data1 / Normalize data2**  
  - Type: Code  
  - Role: Normalizes data arrays to ensure consistent structure; converts message fields into formatted text lines.  
  - Logic: Handles missing or invalid user names, replies, and missing fields with defaults.  
  - Outputs: Single string summary per group.  
  - Failure Modes: Code errors on unexpected data shape.

- **Merge**  
  - Type: Merge (3 inputs)  
  - Role: Combines normalized summaries from all three groups into a single array.

- **Group all conversations**  
  - Type: Aggregate  
  - Role: Aggregates merged data into one item for AI processing.

- **AI Agent**  
  - Type: Langchain Agent (OpenRouter GPT-4.1)  
  - Role: Analyzes combined group messages, generating a structured, insightful report focused on AI, automation, industry trends, and opportunities.  
  - Configuration: Custom system prompt guiding the AI’s personality, filtering, and output format.  
  - Inputs: Aggregated message summaries from groups.  
  - Failure Modes: API rate limiting, prompt or model errors.

- **Set Resumen**  
  - Type: Set  
  - Role: Stores AI agent's output text for downstream formatting.

- **OpenAI 4.1 Mini1**  
  - Type: Langchain Language Model (OpenRouter)  
  - Role: Secondary AI processing step for refining or mini-summary generation (optional, parallel).  
  - Configuration: Temperature 0.  
  - Inputs: AI Agent output or aggregated text.

- **JSON parse**  
  - Type: Output Parser Autofixing  
  - Role: Parses AI output ensuring valid JSON structure; auto-corrects common issues.  
  - Failure Modes: Persistent invalid JSON outputs.

- **Output Formatter**  
  - Type: Chain LLM  
  - Role: Splits the full AI summary into WhatsApp-friendly message fragments following strict formatting rules (bold headings, bullet preservation, no content modification).  
  - Configuration: Detailed prompt to convert Markdown into WhatsApp format messages, split by H2 titles.  
  - Outputs: JSON object with array of messages.

- **Set the JSON Return Type**  
  - Type: Output Parser Structured  
  - Role: Enforces output JSON schema for messages array.

- **Split Out**  
  - Type: Split Out  
  - Role: Splits the array of messages into individual items for sequential sending.

- **Loop Over Items**  
  - Type: Split In Batches  
  - Role: Sends messages one by one with controlled pacing.

- **Send message**  
  - Type: Evolution API (message send)  
  - Role: Sends each formatted message to the designated WhatsApp group ID.  
  - Configuration: Requires setting the target group ID and Evolution API instance name.  
  - Failure Modes: API errors, invalid group ID, message length limits.

- **Wait**  
  - Type: Wait  
  - Role: Adds delay (0.1 seconds) between messages to avoid flooding or rate limits.

---

#### 2.3 Configuration & Customization

**Overview:**  
Contains sticky notes and disabled nodes providing crucial setup information, group discovery, and customization tips.

**Nodes Involved:**  
- Sticky Note1 (Workflow Overview & Purpose)  
- Sticky Note2 (Critical Group ID Configuration)  
- Sticky Note3 (Real-Time Processing Overview)  
- Sticky Note4 (Group Discovery Instructions)  
- Sticky Note5 (Daily Intelligence Generation Overview)  
- Sticky Note6 (Google Sheets Template Link)  
- Sticky Note7 (Intelligent Message Delivery & Formatting)  
- Sticky Note8 (Webhook Setup Instructions)  
- Sticky Note9 (Voice Message Handling Explanation)  
- Sticky Note10 (AI Analysis Engine Explanation)  
- Sticky Note11 (Customization Options)  
- Sticky Note12 (Message Routing Explanation)  
- Sticky Note15 (Author Contact & Consulting Offers)  
- Find your Groups (Disabled Evolution API node for group discovery)

**Key Points:**  
- Users must replace placeholder group IDs with their own.  
- Google Sheets must be prepared with three tabs, one per group, with specified columns.  
- Evolution API webhook must be properly configured for message upsert and participant updates.  
- AI system prompt can be customized to adjust focus.  
- Voice message transcription can be disabled by removing audio processing branch.  
- Daily summary sending group ID must be updated in the Send message node.

---

### 3. Summary Table

| Node Name                      | Node Type                                    | Functional Role                             | Input Node(s)                     | Output Node(s)                  | Sticky Note                                                                                                                                                                                                                                                                                                    |
|--------------------------------|---------------------------------------------|--------------------------------------------|----------------------------------|--------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Sticky Note1                   | Sticky Note                                 | Workflow overview and introduction          | -                                | -                              | ## WHATSAPP GROUP INTELLIGENCE SYSTEM ... (Full content describing purpose, features, requirements, and audience)                                                                                                                                                                                           |
| Sticky Note2                   | Sticky Note                                 | Critical WhatsApp group ID configuration    | -                                | -                              | 🔴 CRITICAL CONFIGURATION ... Must update group IDs; run "Find your Groups" node if unknown.                                                                                                                                                                                                                   |
| Sticky Note3                   | Sticky Note                                 | Real-time message processing overview       | -                                | -                              | 🔄 - REAL-TIME MESSAGE PROCESSING ... Explains webhook, filtering, audio transcription, storage.                                                                                                                                                                                                              |
| Sticky Note4                   | Sticky Note                                 | Instruction to find WhatsApp groups         | -                                | -                              | ## Find your Groups ... Run node to find group IDs.                                                                                                                                                                                                                                                          |
| Sticky Note5                   | Sticky Note                                 | Daily intelligence generation overview      | -                                | -                              | 📊 - DAILY INTELLIGENCE GENERATION ... Scheduled extraction, AI analysis, formatting, and delivery.                                                                                                                                                                                                           |
| Sticky Note6                   | Sticky Note                                 | Google Sheets template link                  | -                                | -                              | ## Sheets Template ... https://docs.google.com/spreadsheets/d/1REnD1Sac8O2vnyWOIpB4WqMZbWNMq3zxJ-n8a-LSwms/edit?usp=sharing                                                                                                                                                                                |
| Sticky Note7                   | Sticky Note                                 | Intelligent message delivery explanation    | -                                | -                              | 📱 INTELLIGENT MESSAGE DELIVERY ... Formatting and sending summary messages; update target group ID.                                                                                                                                                                                                          |
| Sticky Note8                   | Sticky Note                                 | Evolution API webhook setup instructions    | -                                | -                              | 🔴 WEBHOOK SETUP REQUIRED ... Configure webhook URL in Evolution API with specified events enabled.                                                                                                                                                                                                           |
| Sticky Note9                   | Sticky Note                                 | Voice message processing explanation        | -                                | -                              | 🎙️ SMART AUDIO HANDLING ... Detects, downloads, converts, transcribes voice messages.                                                                                                                                                                                                                        |
| Sticky Note10                  | Sticky Note                                 | AI analysis engine explanation               | -                                | -                              | 🤖 AI ANALYSIS ENGINE ... WhatsOn AI agent focus, filtering, customization notes.                                                                                                                                                                                                                            |
| Sticky Note11                  | Sticky Note                                 | Customization options and instructions       | -                                | -                              | ⚙️ CUSTOMIZATION OPTIONS ... Schedule, groups, AI prompt, audio processing adjustments.                                                                                                                                                                                                                       |
| Sticky Note12                  | Sticky Note                                 | Message routing explanation                   | -                                | -                              | 🔄 SMART MESSAGE ROUTING ... Switch node routes messages by group for storage.                                                                                                                                                                                                                               |
| Sticky Note15                  | Sticky Note                                 | Author contact, consulting offers             | -                                | -                              | ## Was this helpful? ... Links to schedule calls and follow on LinkedIn.                                                                                                                                                                                                                                     |
| Webhook-EVO                   | Webhook (POST)                              | Receive WhatsApp messages from Evolution API | -                                | Set Info                       | 🔴 WEBHOOK SETUP REQUIRED (Sticky Note8)                                                                                                                                                                                                                                                                      |
| Set Info                      | Set                                         | Extract and assign variables from webhook    | Webhook-EVO                     | Check if the message is from the Group | 🔴 CRITICAL CONFIGURATION (Sticky Note2)                                                                                                                                                                                                                                                                      |
| Check if the message is from the Group | If                                   | Filter messages to monitored groups           | Set Info                       | If (audio detection), No hacer nada | 🔄 REAL-TIME MESSAGE PROCESSING (Sticky Note3)                                                                                                                                                                                                                                                               |
| If                           | If                                          | Detect if message is audio                      | Check if the message is from the Group | Download audio (audio branch), Switch (text branch) | 🎙️ SMART AUDIO HANDLING (Sticky Note9)                                                                                                                                                                                                                                                                       |
| Download audio               | HTTP Request                                | Download audio file from Evolution API         | If                            | Convert audio                 | 🎙️ SMART AUDIO HANDLING (Sticky Note9)                                                                                                                                                                                                                                                                       |
| Convert audio               | Convert to File                             | Convert base64 audio to binary                   | Download audio                | OpenAI (transcribe)           | 🎙️ SMART AUDIO HANDLING (Sticky Note9)                                                                                                                                                                                                                                                                       |
| OpenAI                      | OpenAI Audio Transcription                  | Transcribe audio to text                         | Convert audio                 | Audio content                 | Requires OpenAI API credentials                                                                                                                                                                                                                                                                                |
| Audio content               | Set                                         | Store transcribed text                           | OpenAI                       | Set Info1                    | 🎙️ SMART AUDIO HANDLING (Sticky Note9)                                                                                                                                                                                                                                                                       |
| Set Info1                   | Set                                         | Assign variables replacing audio with text     | Audio content                | Switch                      | 🔄 REAL-TIME MESSAGE PROCESSING (Sticky Note3)                                                                                                                                                                                                                                                               |
| Switch                      | Switch                                      | Route messages to Google Sheets by group        | Set Info1                   | Save Messages, Save Messages1, Save Messages2 | 🔄 SMART MESSAGE ROUTING (Sticky Note12)                                                                                                                                                                                                                                                                      |
| Save Messages               | Google Sheets Append                        | Append group 1 messages to sheet                 | Switch (Group 1)             | -                            | Requires Google Sheets credentials; sheet tab Grupo_1                                                                                                                                                                                                                                                        |
| Save Messages1              | Google Sheets Append                        | Append group 2 messages to sheet                 | Switch (Group 2)             | -                            | Requires Google Sheets credentials; sheet tab Grupo_2                                                                                                                                                                                                                                                        |
| Save Messages2              | Google Sheets Append                        | Append group 3 messages to sheet                 | Switch (Group 3)             | -                            | Requires Google Sheets credentials; sheet tab Grupo_3                                                                                                                                                                                                                                                        |
| No hacer nada               | No Operation                               | Terminates flow for messages outside groups      | Check if the message is from the Group (false branch) | -                            |                                                                                                                                                                                                                                                                                                               |
| Send Summary every day at 00.00h | Schedule Trigger                         | Trigger daily summary process                     | -                            | Extract today's conversations nodes | ⚙️ CUSTOMIZATION OPTIONS (Sticky Note11)                                                                                                                                                                                                                                                                      |
| Extract today's conversations | Google Sheets Read                         | Extract previous day’s messages from Group 1     | Send Summary every day at 00.00h | Group conversations          | Requires Google Sheets credentials                                                                                                                                                                                                                                                                            |
| Extract today's conversations1 | Google Sheets Read                        | Extract previous day’s messages from Group 2     | Send Summary every day at 00.00h | Group conversations1         | Requires Google Sheets credentials                                                                                                                                                                                                                                                                            |
| Extract today's conversations2 | Google Sheets Read                        | Extract previous day’s messages from Group 3     | Send Summary every day at 00.00h | Group conversations2         | Requires Google Sheets credentials                                                                                                                                                                                                                                                                            |
| Group conversations          | Aggregate                                   | Aggregate Group 1 data into array                  | Extract today's conversations | Normalize data1              |                                                                                                                                                                                                                                                                                                               |
| Group conversations1         | Aggregate                                   | Aggregate Group 2 data into array                  | Extract today's conversations1 | Normalize data               |                                                                                                                                                                                                                                                                                                               |
| Group conversations2         | Aggregate                                   | Aggregate Group 3 data into array                  | Extract today's conversations2 | Normalize data2             |                                                                                                                                                                                                                                                                                                               |
| Normalize data               | Code                                        | Normalize and format Group 2 messages              | Group conversations1          | Merge (input 1)              | Contains logic to handle missing names, messages, replies                                                                                                                                                                                                                                                    |
| Normalize data1              | Code                                        | Normalize and format Group 1 messages              | Group conversations           | Merge (input 0)              | Same logic as Normalize data                                                                                                                                                                                                                                                                                   |
| Normalize data2              | Code                                        | Normalize and format Group 3 messages              | Group conversations2          | Merge (input 2)              | Same logic as Normalize data                                                                                                                                                                                                                                                                                   |
| Merge                       | Merge (3 inputs)                            | Combine normalized group summaries                  | Normalize data1, Normalize data, Normalize data2 | Group all conversations     |                                                                                                                                                                                                                                                                                                               |
| Group all conversations     | Aggregate                                   | Aggregate merged group summaries into single item  | Merge                        | AI Agent                    |                                                                                                                                                                                                                                                                                                               |
| AI Agent                    | Langchain Agent (GPT-4.1)                   | Generate structured intelligence report             | Group all conversations       | Set Resumen                 | Custom system prompt defining AI personality and output format (Sticky Note10)                                                                                                                                                                                                                               |
| Set Resumen                 | Set                                         | Store AI Agent summary text                          | AI Agent                    | Output Formatter            |                                                                                                                                                                                                                                                                                                               |
| OpenAI 4.1 Mini1            | Langchain Language Model                     | Optional refinement of AI summary                     | AI Agent                    | Output Formatter, JSON parse | Requires OpenRouter API credentials                                                                                                                                                                                                                                                                           |
| JSON parse                  | Output Parser Autofixing                     | Validate and fix JSON output from AI                   | OpenAI 4.1 Mini1             | Output Formatter            |                                                                                                                                                                                                                                                                                                               |
| Output Formatter            | Chain LLM                                    | Split AI summary into WhatsApp message fragments       | Set Resumen, JSON parse      | Split Out                  | Uses strict prompt to maintain message content unaltered and format for WhatsApp                                                                                                                                                                                                                              |
| Set the JSON Return Type    | Output Parser Structured                     | Enforce JSON schema on formatted output                 | Output Formatter             | JSON parse                 |                                                                                                                                                                                                                                                                                                               |
| Split Out                   | Split Out                                    | Split array of WhatsApp messages into separate items    | Output Formatter             | Loop Over Items            |                                                                                                                                                                                                                                                                                                               |
| Loop Over Items             | Split In Batches                             | Sequentially send messages with pacing                  | Split Out                   | Send message               | Controls message flow to avoid rate limits                                                                                                                                                                                                                                                                    |
| Send message                | Evolution API message send                   | Send each fragment as WhatsApp message to target group | Loop Over Items             | Wait                       | Requires correct group ID and Evolution API instance setup; final delivery step                                                                                                                                                                                                                               |
| Wait                       | Wait                                         | Delay between sending messages                           | Send message                | Loop Over Items            | Prevents flooding; 0.1 seconds wait configured                                                                                                                                                                                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node "Webhook-EVO"**  
   - Type: Webhook (POST)  
   - Path: Unique string (e.g., "0c2a31c8-5291-4fe2-b598-ec6508ff97dc")  
   - Purpose: Receive WhatsApp messages from Evolution API.

2. **Create "Set Info" Node**  
   - Type: Set  
   - Assign variables extracting from webhook JSON:  
     - grupo_1, grupo_2, grupo_3: Your WhatsApp group IDs (replace placeholders)  
     - grupo_recibido: `={{ $json.body.data.key.remoteJid }}`  
     - mensaje: `={{ $json.body.data.message.conversation }}`  
     - nombre: `={{ $json.body.data.pushName }}`  
     - hora: `={{ $json.body.date_time }}`  
     - mensaje_respondido: `={{ $json.body.data.contextInfo.quotedMessage.conversation }}`  
     - caption: `={{ $json.body.data.message.imageMessage.caption }}`  
     - audio: `={{ $json.body.data.messageType }}`

3. **Add "Check if the message is from the Group" (If node)**  
   - Condition: Check if `grupo_recibido` equals any of grupo_1, grupo_2, or grupo_3.

4. **Add "If" Node for Audio Detection**  
   - Condition: Check if `$json.messageType == "audioMessage"`.

5. **For Audio Branch:**  
   - Add "Download audio" HTTP Request node:  
     - URL: Use webhook JSON fields for server URL and instance, with POST body parameters for message key ID and `convertToMp4=false`.  
     - Header: Include API key from webhook data.

   - Add "Convert audio" node:  
     - Convert downloaded base64 audio to binary, use MIME type from downloaded data.

   - Add "OpenAI" node for transcription:  
     - Resource: Audio  
     - Operation: Transcribe  
     - Provide OpenAI API credentials.

   - Add "Audio content" (Set node) to store transcription text in `content` field.

   - Add "Set Info1" node:  
     - Similar fields as "Set Info" but replace `mensaje` with transcribed text from `content`.

6. **For Text Branch:** Connect "Set Info" to "Switch" node.

7. **Create "Switch" node** to route messages by group ID (`received_group`):  
   - Outputs correspond to grupo_1, grupo_2, grupo_3.

8. **Create three Google Sheets "Save Messages" nodes:**  
   - Append operation to sheets tabs Grupo_1, Grupo_2, Grupo_3 respectively.  
   - Map fields: Date (today’s date), Time (format message time in Europe/Madrid timezone), Name, Caption, Message, Replied Message.  
   - Use Google Sheets OAuth2 credentials.  
   - Document ID: Your Google Sheet ID with three tabs.

9. **Add "No hacer nada" NoOp node** for messages outside groups.

10. **Connect Webhook-EVO → Set Info → Check if the message is from the Group → If (audio detection) → Audio branch (Download audio → Convert audio → OpenAI → Audio content → Set Info1) → Switch → Save Messages nodes**  
    **Text branch:** Check if the message is from the group → Switch → Save Messages nodes

---

11. **Daily Summary Generation:**

12. **Add "Send Summary every day at 00.00h" Schedule Trigger:**  
    - Configure to run daily at 00:01 Europe/Madrid time.

13. **Add three Google Sheets "Extract today's conversations" nodes:**  
    - Read previous day's messages from each group tab (filter by date column with yesterday’s date).  
    - Use Google Sheets OAuth2 credentials.

14. **Add three "Group conversations" Aggregate nodes:**  
    - Aggregate all rows into arrays for each group.

15. **Add three "Normalize data" Code nodes:**  
    - Use provided JavaScript snippet to normalize and format data into text lines.  
    - Inputs: Corresponding Group conversations output.

16. **Add "Merge" node (3 inputs):**  
    - Combine normalized data from all groups.

17. **Add "Group all conversations" Aggregate node:**  
    - Aggregate merged data into a single item for AI processing.

18. **Add "AI Agent" node:**  
    - Langchain agent using OpenRouter GPT-4.1 model.  
    - Configure system prompt as provided (WhatsOn 🕵🏾‍♂️ agent with filtering and output format).  
    - Credentials: OpenRouter API key.

19. **Add "Set Resumen" node:**  
    - Store AI summary text.

20. **Add "OpenAI 4.1 Mini1" Langchain node (optional):**  
    - Refines or processes AI output further.  
    - Connect AI Agent output as input.  
    - Credentials: OpenRouter API.

21. **Add "JSON parse" Output Parser Autofixing node:**  
    - Parses AI output to ensure valid JSON.

22. **Add "Output Formatter" Chain LLM node:**  
    - Splits summary into WhatsApp-friendly message fragments following strict formatting rules (preserving content exactly, converting Markdown to WhatsApp formatting).  
    - Use provided prompt instructions.

23. **Add "Set the JSON Return Type" Output Parser Structured node:**  
    - Enforce JSON schema with a messages array.

24. **Add "Split Out" node:**  
    - Splits messages array into individual items.

25. **Add "Loop Over Items" Split In Batches node:**  
    - Sequentially processes messages with batch size 1.

26. **Add "Send message" Evolution API node:**  
    - Sends each message to designated WhatsApp group ID (replace `<YOUR GROUP>` and `<INSTANCE NAME>` with your setup).  
    - Requires Evolution API credentials.

27. **Add "Wait" node:**  
    - Delay 0.1 seconds between messages.

28. **Connect nodes:**  
    - Schedule Trigger → Extract today’s conversations nodes (3) → Group conversations nodes (3) → Normalize data nodes (3) → Merge → Group all conversations → AI Agent → Set Resumen → OpenAI 4.1 Mini1 → JSON parse → Output Formatter → Set JSON Return Type → Split Out → Loop Over Items → Send message → Wait → Loop Over Items (loop continues)

---

### 5. General Notes & Resources

| Note Content                                                                                                                   | Context or Link                                                                                          |
|-------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Google Sheets template for message storage with tabs per group.                                                                | https://docs.google.com/spreadsheets/d/1REnD1Sac8O2vnyWOIpB4WqMZbWNMq3zxJ-n8a-LSwms/edit?usp=sharing    |
| Evolution API webhook setup instructions: enable MESSAGES_UPSERT and GROUP_PARTICIPANTS_UPDATE, disable IGNORE_GROUPS.         | Sticky Note8                                                                                           |
| AI Agent system prompt governs analysis focus on AI, automation, business opportunities, and filters out chatter.              | Included in AI Agent node parameters and Sticky Note10                                                 |
| Voice message transcription preserves valuable audio information for analysis.                                                  | Sticky Note9                                                                                           |
| Daily summary sent to a WhatsApp group requires updating the group ID in the Send message node.                                | Sticky Note7                                                                                           |
| To add more groups: add group variables, create Google Sheets tabs, update Switch node, add Save Messages nodes accordingly.    | Sticky Note11                                                                                          |
| Contact and consulting offers by Daniel Lianes for automation help.                                                             | Sticky Note15, including links to schedule calls and LinkedIn profile                                  |
| If group IDs are unknown, use the disabled "Find your Groups" node with Evolution API to fetch groups.                          | Disabled node "Find your Groups"                                                                        |

---

This documentation provides a detailed, step-by-step understanding of the workflow’s architecture, node roles, key configurations, and customization points, enabling reproduction or modification with anticipation of potential pitfalls such as API errors, misconfiguration, or data inconsistencies.

---