Build a Multi-Modal Telegram AI Assistant with Gemini, Voice & Image Generation

https://n8nworkflows.xyz/workflows/build-a-multi-modal-telegram-ai-assistant-with-gemini--voice---image-generation-6003


# Build a Multi-Modal Telegram AI Assistant with Gemini, Voice & Image Generation

### 1. Workflow Overview

This workflow implements a multi-modal AI assistant on Telegram, integrating Google Gemini models for natural language understanding and response generation, voice transcription via AssemblyAI, image generation via external APIs, and user memory management with MongoDB. It supports text and voice inputs, intent detection with routing to appropriate handlers (chat, image generation, reminders, memory updates), and dynamic "GF Mode" for affectionate conversational style. The workflow is structured into several logical blocks:

- **1.1 Telegram Input Reception & Mode Toggle:** Captures incoming Telegram messages (text or voice), detects commands to toggle "GF Mode" (a romantic/affectionate assistant persona).

- **1.2 Voice Message Transcription Pipeline:** Handles voice messages by fetching audio, submitting transcription jobs to AssemblyAI, polling for completion, and extracting transcription text and metadata.

- **1.3 Intent Analysis & Routing:** Uses Google Gemini LLM chains to analyze user messages and classify intent into image generation, reminder setting, memory updating, or general chat.

- **1.4 Context Building & Memory Retrieval:** Retrieves user conversation summaries and memories from MongoDB, builds context strings for AI agents to maintain conversational continuity.

- **1.5 Chat & Image Generation Agents:** Invokes Google Gemini chat models and custom LangChain agents to generate chat responses or detailed image prompts, including cleaning and formatting output.

- **1.6 Reminder Processing:** Parses natural language reminders into structured JSON, creates Google Calendar events and Google Tasks entries, then confirms status to the user.

- **1.7 Memory Summarization & Storage:** Summarizes conversations and stores user-specific memory documents in MongoDB for future context enrichment.

- **1.8 Response Delivery:** Sends final replies back to Telegram, supporting text, images, and voice outputs (via TTS).

---

### 2. Block-by-Block Analysis

---

#### 2.1 Telegram Input Reception & GF Mode Toggle

**Overview:**  
Captures Telegram messages, distinguishes text vs voice, and allows users to toggle a special "GF Mode" (emotional, affectionate assistant persona).

**Nodes Involved:**  
- Telegram Trigger1  
- Toggle GF Mode1  
- Set GF Mode TRUE1  
- Set GF Mode FALSE1  
- Send Mode Confirmation : /gf_on1  
- Send Mode Confirmation : /gf_off1  

**Node Details:**  

- **Telegram Trigger1**  
  - Type: Telegram Trigger  
  - Role: Entry point for all Telegram messages (text/voice).  
  - Config: Listens for "message" updates only.  
  - Outputs raw Telegram message JSON.  
  - Edge cases: Messages without text or voice may be filtered downstream.

- **Toggle GF Mode1**  
  - Type: Switch  
  - Role: Detects commands "/gf on" or "/gf off" in message text to toggle GF Mode.  
  - Config: Compares message text string to these commands, routes accordingly.  
  - Outputs to Set GF Mode TRUE1 or Set GF Mode FALSE1 or continues normal processing.  
  - Edge cases: Case sensitivity is enabled; unrecognized commands default to normal flow.

- **Set GF Mode TRUE1 / Set GF Mode FALSE1**  
  - Type: Set  
  - Role: Prepare data to update GF Mode status in Google Sheets.  
  - Config: Sets gfMode to true or false, includes user's chat ID.  
  - Output feeds into Google Sheets nodes.  

- **Google Sheets: Set GF Mode TRUE1 / Google Sheets:Set GF Mode FALSE1**  
  - Type: Google Sheets  
  - Role: Persist GF Mode status per user in a Google Sheet (sheet and doc ID placeholders to be configured).  
  - Config: Append or update row keyed by chat_id.  
  - Edge cases: Requires valid Google Sheets credentials and correct sheet/document IDs.

- **Send Mode Confirmation : /gf_on1 / Send Mode Confirmation : /gf_off1**  
  - Type: Telegram  
  - Role: Sends confirmation message to user upon mode toggle, e.g., "Okay love, I’m in romantic mode now 😘" or "Got it. Back to assistant mode 😊".  
  - Config: Uses chat ID from Telegram Trigger.  
  - Edge cases: Ensures no attribution appended; message formatting is plain text.

---

#### 2.2 Voice Message Transcription Pipeline

**Overview:**  
Processes incoming voice messages by downloading audio file from Telegram, submitting transcription jobs to AssemblyAI, polling until completion, and formatting transcription results.

**Nodes Involved:**  
- Check Voice Message1  
- HTTP Request Fetch Voice Message Audio1  
- Submit Audio Transcription Job1  
- Store Transcript ID1  
- Wait 5 Seconds1  
- Check Transcription Status1  
- Is Completed?1  
- Is Error?1  
- Reply: Sorry, Didn’t Hear You1  
- Format Results1  

**Node Details:**  

- **Check Voice Message1**  
  - Type: If  
  - Role: Checks if the message includes a voice object to branch voice processing.  
  - Config: Condition tests existence of `message.voice` object.  
  - Edge cases: Non-voice messages skip this branch.

- **HTTP Request Fetch Voice Message Audio1**  
  - Type: HTTP Request  
  - Role: Retrieves Telegram file path for voice message using Telegram Bot API with token credential.  
  - Config: GET request to Telegram API endpoint `/getFile` with voice file_id.  
  - Outputs file path for later audio download.

- **Submit Audio Transcription Job1**  
  - Type: HTTP Request  
  - Role: Submits transcription job to AssemblyAI with audio URL.  
  - Config: POST to AssemblyAI transcript endpoint, sending audio_url from Telegram file path, plus transcription options like language detection, punctuation, sentiment analysis.  
  - Requires AssemblyAI API key as HTTP header authentication.  
  - Outputs transcript job ID and status.

- **Store Transcript ID1**  
  - Type: Set  
  - Role: Stores transcript job ID, status, audio URL, and timestamp for polling.  
  - Used to track transcription job progress.

- **Wait 5 Seconds1**  
  - Type: Wait  
  - Role: Delays workflow 5 seconds before polling transcription status to allow processing.

- **Check Transcription Status1**  
  - Type: HTTP Request  
  - Role: Polls AssemblyAI transcript endpoint to get current job status and transcription results.  
  - Config: GET request with transcript_id.  
  - Outputs full transcription JSON data.

- **Is Completed?1**  
  - Type: If  
  - Role: Checks if transcription status is "completed".  
  - Routes to next steps or to error handling.

- **Is Error?1**  
  - Type: If  
  - Role: Checks if transcription status is "error".  
  - If error, triggers reply node to inform user.

- **Reply: Sorry, Didn’t Hear You1**  
  - Type: Telegram  
  - Role: Sends error message to user if transcription fails.

- **Format Results1**  
  - Type: Set  
  - Role: Extracts key transcription fields (transcription text, confidence, audio duration, language, sentiment, entities) into standardized properties for downstream use.

---

#### 2.3 Intent Analysis & Routing (Chat Input and Voice Input)

**Overview:**  
Analyzes user message text or transcription text to detect intent (generate_image, remember, reminder, chat), parses JSON output, routes to appropriate workflow branches.

**Nodes Involved:**  
- Extract & Validate Data1  
- Check GF Mode Status1  
- Google Cloud Natural Language1  
- Detect Mood1  
- Fetch Auto Memory1  
- Fetch User Memory2  
- Build Context2  
- Intent Analysis2  
- Parse Intent2  
- intent string2  
- Route Intent2  
- AI Agent FOR Generate Image Prompt1  
- Clean Prompt Text1  
- HTTP Request (Image Generation API)  
- Edit Fields4  
- Convert to File  
- Send Image for Text1  
- Chat Agent2  
- Extract Memory Info1  
- Check If Worth Remembering1  
- Chat Agent Output Cleaner1  
- Send reply1  
- Summarize Chat2  
- Edit Fields7  
- Save Conversation Memory on memory_auto3  
- Save Conversation Memory on user_memory1  

**Node Details:**  

- **Extract & Validate Data1**  
  - Type: Code  
  - Role: Extracts and sanitizes Telegram message text and user metadata (userId, chatId, username, etc.).  
  - Limits messageText length to 4000 chars.

- **Check GF Mode Status1**  
  - Type: Google Sheets  
  - Role: Looks up the user's GF mode status in Google Sheets by chat ID to customize assistant persona.  
  - Outputs gfMode boolean.

- **Google Cloud Natural Language1**  
  - Type: Google Cloud Natural Language API  
  - Role: Performs sentiment analysis on the user message text to detect mood.

- **Detect Mood1**  
  - Type: Code  
  - Role: Interprets sentiment score to categorize mood as happy, sad, neutral, etc.

- **Fetch Auto Memory1 / Fetch User Memory2**  
  - Type: MongoDB  
  - Role: Fetches conversation summaries and memory documents for the user from MongoDB collections "memory_auto" and "conversation_summaries".  
  - Sorted by most recent.

- **Build Context2**  
  - Type: Code  
  - Role: Builds a string that consolidates recent conversation summaries or raw messages as context for AI agents.

- **Intent Analysis2**  
  - Type: LangChain chainLlm (Google Gemini)  
  - Role: Analyzes user message or transcription text to detect intent, returns JSON with intent and fields.  
  - Intent options: generate_image, remember, reminder, chat.

- **Parse Intent2**  
  - Type: Code  
  - Role: Parses JSON output from Intent Analysis, defaults to "chat" intent on failure.

- **intent string2**  
  - Type: Code  
  - Role: Maps intent string to numeric intentCode for routing (1=generate_image, 2=remember, 3=chat, 4=reminder).

- **Route Intent2**  
  - Type: Switch  
  - Role: Routes workflow based on intentCode to image generation, chat, or reminder branches.

- **AI Agent FOR Generate Image Prompt1**  
  - Type: LangChain agent (Google Gemini)  
  - Role: Generates detailed image generation prompt based on conversation context and user message.

- **Clean Prompt Text1**  
  - Type: Code  
  - Role: Cleans prompt text by removing quotes and extra spaces before sending to image generation API.

- **HTTP Request (Image Generation API)**  
  - Type: HTTP Request  
  - Role: Calls Together.xyz image generation API with cleaned prompt, requests base64-encoded image.

- **Edit Fields4**  
  - Type: Set  
  - Role: Extracts base64 image string from API response.

- **Convert to File**  
  - Type: Convert To File  
  - Role: Converts base64 string to binary file for Telegram photo sending.

- **Send Image for Text1**  
  - Type: Telegram  
  - Role: Sends generated image photo to Telegram chat.

- **Chat Agent2**  
  - Type: LangChain agent (Google Gemini)  
  - Role: Generates chat response considering GF Mode, mood, memory context, and user message.

- **Extract Memory Info1**  
  - Type: LangChain chainLlm  
  - Role: Extracts new information from the conversation turn that may be worth remembering.

- **Check If Worth Remembering1**  
  - Type: If  
  - Role: Checks if extracted memory info is not "nothing" before saving.

- **Chat Agent Output Cleaner1**  
  - Type: Code  
  - Role: Sanitizes chat agent output for Telegram sending (escapes special chars).

- **Send reply1**  
  - Type: Telegram  
  - Role: Sends cleaned chat response back to user.

- **Summarize Chat2**  
  - Type: LangChain chainLlm  
  - Role: Summarizes the user-assistant exchange into a concise human-readable sentence or "nothing" if trivial.

- **Edit Fields7**  
  - Type: Set  
  - Role: Prepares document fields with timestamp, userID, summary, user message, and assistant response.

- **Save Conversation Memory on memory_auto3 / Save Conversation Memory on user_memory1**  
  - Type: MongoDB  
  - Role: Persists conversation summaries and memory info in MongoDB collections for future context.

---

#### 2.4 Reminder Processing

**Overview:**  
Parses natural language reminders, creates calendar events and tasks, confirms reminder status back to user.

**Nodes Involved:**  
- intent from route intent1  
- intent match checker1  
- Parse Natural Language Reminders to JSON1  
- Clean & Parse1  
- Edit Fields6  
- Google Calendar  
- Google Tasks  
- reminder status confirmation1  
- Telegram1  

**Node Details:**  

- **intent from route intent1**  
  - Type: If  
  - Role: Checks if intentCode equals 4 (reminder), routes accordingly.

- **intent match checker1**  
  - Type: Merge (fuzzy compare)  
  - Role: Combines intent data streams to ensure consistent routing.

- **Parse Natural Language Reminders to JSON1**  
  - Type: LangChain chainLlm  
  - Role: Converts user reminder message and current time into structured JSON with task title, start and end times, and timezone.  
  - Strict JSON output only.

- **Clean & Parse1**  
  - Type: Code  
  - Role: Cleans and parses reminder JSON.

- **Edit Fields6**  
  - Type: Set  
  - Role: Maps parsed reminder JSON fields into parameters for Google Calendar event creation.

- **Google Calendar**  
  - Type: Google Calendar  
  - Role: Creates calendar event with reminder details (title, start/end time, description).

- **Google Tasks**  
  - Type: Google Tasks  
  - Role: Creates a Google Task for the reminder with notes and due date.

- **reminder status confirmation1**  
  - Type: LangChain chainLlm  
  - Role: Generates a concise, emoji-rich Telegram message confirming reminder creation status.

- **Telegram1**  
  - Type: Telegram  
  - Role: Sends the confirmation message to the user chat.

---

#### 2.5 Voice Input Chat Processing & TTS Output

**Overview:**  
Processes voice transcriptions through chat AI agent, converts reply to speech via Edge-TTS, sends as audio message, and summarizes conversation memory.

**Nodes Involved:**  
- Fetch User Memory3  
- Build Context3  
- Chat Agent3  
- Run Edge-TTS1  
- Read MP3 File1  
- Send Image for Voice msg1  
- Summarize Voice Message1  
- Prepare Voice Summary Document1  
- Save Conversation Memory on memory_auto5  

**Node Details:**  

- **Fetch User Memory3**  
  - Type: MongoDB  
  - Role: Retrieves user memory documents for voice input conversations.

- **Build Context3**  
  - Type: Code  
  - Role: Builds memory context string and conversation history for voice input.

- **Chat Agent3**  
  - Type: LangChain agent (Google Gemini)  
  - Role: Generates chat reply in affectionate style limited to plain text, based on transcription and memory.

- **Run Edge-TTS1**  
  - Type: Execute Command  
  - Role: Runs Edge-TTS CLI to generate MP3 audio from chat reply text, using 'en-US-JennyNeural' voice.

- **Read MP3 File1**  
  - Type: Read Binary File  
  - Role: Loads generated MP3 audio into binary format for sending.

- **Send Image for Voice msg1**  
  - Type: Telegram  
  - Role: Sends generated MP3 audio file as voice message to Telegram chat.

- **Summarize Voice Message1**  
  - Type: LangChain chainLlm  
  - Role: Summarizes the voice message user-assistant exchange concisely for memory storage.

- **Prepare Voice Summary Document1**  
  - Type: Set  
  - Role: Prepares MongoDB document with timestamp, userID, summary, user message, and assistant response.

- **Save Conversation Memory on memory_auto5**  
  - Type: MongoDB  
  - Role: Stores voice message conversation summary for future context.

---

### 3. Summary Table

| Node Name                          | Node Type                            | Functional Role                                | Input Node(s)                      | Output Node(s)                            | Sticky Note                                            |
|-----------------------------------|------------------------------------|------------------------------------------------|----------------------------------|------------------------------------------|--------------------------------------------------------|
| Telegram Trigger1                 | Telegram Trigger                   | Entry point for incoming Telegram messages      | -                                | Toggle GF Mode1, Check Voice Message1    | Telegram Request Handling "🔔 Trigger → Route" "🎙️ Voice Pipeline" "💬 Chat Pipeline" |
| Toggle GF Mode1                  | Switch                            | Detects "/gf on" or "/gf off" commands          | Telegram Trigger1                | Set GF Mode TRUE1, Set GF Mode FALSE1, Check Voice Message1 | Update GF Mode → Notify User                            |
| Set GF Mode TRUE1                | Set                               | Prepares data to set GF Mode = true              | Toggle GF Mode1                 | Google Sheets: Set GF Mode TRUE1          | Update GF Mode → Notify User                            |
| Set GF Mode FALSE1               | Set                               | Prepares data to set GF Mode = false             | Toggle GF Mode1                 | Google Sheets:Set GF Mode FALSE1          | Update GF Mode → Notify User                            |
| Google Sheets: Set GF Mode TRUE1 | Google Sheets                    | Persists GF Mode true in Google Sheets           | Set GF Mode TRUE1               | Send Mode Confirmation : /gf_on1          | Update GF Mode → Notify User                            |
| Google Sheets:Set GF Mode FALSE1 | Google Sheets                    | Persists GF Mode false in Google Sheets          | Set GF Mode FALSE1              | Send Mode Confirmation : /gf_off1         | Update GF Mode → Notify User                            |
| Send Mode Confirmation : /gf_on1 | Telegram                        | Sends confirmation for GF Mode ON                 | Google Sheets: Set GF Mode TRUE1 | -                                        |                                                        |
| Send Mode Confirmation : /gf_off1| Telegram                        | Sends confirmation for GF Mode OFF                | Google Sheets:Set GF Mode FALSE1 | -                                        |                                                        |
| Check Voice Message1             | If                                | Checks if message contains voice                   | Toggle GF Mode1                 | HTTP Request Fetch Voice Message Audio1, Extract & Validate Data1 | Segment (Voice input) "Intent Analysis → Context Building → Routing" |
| HTTP Request Fetch Voice Message Audio1 | HTTP Request               | Retrieves Telegram voice file path                 | Check Voice Message1            | Submit Audio Transcription Job1            |                                                        |
| Submit Audio Transcription Job1  | HTTP Request                     | Submits transcription job to AssemblyAI           | HTTP Request Fetch Voice Message Audio1 | Store Transcript ID1                      |                                                        |
| Store Transcript ID1             | Set                               | Stores transcription job metadata                  | Submit Audio Transcription Job1 | Wait 5 Seconds1                            |                                                        |
| Wait 5 Seconds1                 | Wait                              | Delays 5 seconds before polling transcription     | Store Transcript ID1            | Check Transcription Status1                 |                                                        |
| Check Transcription Status1      | HTTP Request                     | Polls transcription status from AssemblyAI        | Wait 5 Seconds1                | Is Completed?1                             |                                                        |
| Is Completed?1                  | If                                | Checks for transcription completion                | Check Transcription Status1    | Intent Analysis3 (if completed), Is Error?1 (otherwise) |                                                        |
| Is Error?1                     | If                                | Checks for transcription errors                     | Check Transcription Status1    | Reply: Sorry, Didn’t Hear You1 (if error), Wait 5 Seconds1 (retry) |                                                        |
| Reply: Sorry, Didn’t Hear You1  | Telegram                        | Sends error message if transcription fails          | Is Error?1                    | -                                        |                                                        |
| Format Results1                 | Set                               | Extracts transcription results and metadata        | Is Completed?1                | Fetch User Memory3                         |                                                        |
| Extract & Validate Data1        | Code                              | Extracts and sanitizes Telegram text and metadata  | Check Voice Message1, Toggle GF Mode1 | Check GF Mode Status1                     | Segment of "Intent Analysis → Context Building → Routing" |
| Check GF Mode Status1           | Google Sheets                    | Retrieves GF Mode flag from Google Sheets           | Extract & Validate Data1        | Merge3                                    |                                                        |
| Google Cloud Natural Language1  | Google Cloud API                | Performs sentiment analysis on user message         | Extract & Validate Data1        | Detect Mood1                              |                                                        |
| Detect Mood1                   | Code                              | Converts sentiment score into mood category         | Google Cloud Natural Language1 | Merge3                                    |                                                        |
| Fetch Auto Memory1             | MongoDB                           | Retrieves auto memory conversation summaries        | Extract & Validate Data1        | Build Auto Memory Context1                 |                                                        |
| Build Auto Memory Context1      | Code                              | Builds memory context string from auto memory       | Fetch Auto Memory1             | AI Agent FOR Generate Image Prompt1, AI Agents (memory input) |                                                        |
| Fetch User Memory2              | MongoDB                           | Retrieves user conversation summaries                | Extract & Validate Data1        | Build Context2                            |                                                        |
| Build Context2                 | Code                              | Consolidates recent conversation summaries           | Fetch User Memory2, Merge1      | Route Intent2                             | Segment of "Intent Analysis → Context Building → Routing" |
| Intent Analysis2               | LangChain chainLlm              | Detects intent from user message or transcription    | Merge1                        | Parse Intent2                            | Segment of "Intent Analysis → Context Building → Routing" |
| Parse Intent2                 | Code                              | Parses JSON output from intent analysis               | Intent Analysis2              | intent string2                           |                                                        |
| intent string2                | Code                              | Maps intent string to numeric code for routing        | Parse Intent2                 | Route Intent2, intent from string1       |                                                        |
| Route Intent2                | Switch                           | Routes workflow to image generation, chat, or reminder | intent string2               | AI Agent FOR Generate Image Prompt1, Merge3, intent from route intent1 | ## Set Reminder workflow                               |
| AI Agent FOR Generate Image Prompt1 | LangChain agent             | Generates detailed prompt for image generation        | Route Intent2, Build Auto Memory Context1 | Clean Prompt Text1                       | ## image generation workflow (for chat input)          |
| Clean Prompt Text1           | Code                              | Cleans generated image prompt text                     | AI Agent FOR Generate Image Prompt1 | HTTP Request (Image Generation API)      |                                                        |
| HTTP Request (Image Generation API) | HTTP Request               | Sends prompt to image generation API, requests image  | Clean Prompt Text1             | Edit Fields4                             |                                                        |
| Edit Fields4                | Set                               | Extracts base64 image data from API response          | HTTP Request                  | Convert to File                          |                                                        |
| Convert to File             | Convert To File                  | Converts base64 string to binary file for Telegram    | Edit Fields4                  | Send Image for Text1                     |                                                        |
| Send Image for Text1         | Telegram                        | Sends generated image to user                          | Convert to File               | Summarize Chat3                         |                                                        |
| Summarize Chat3             | LangChain chainLlm              | Summarizes user-assistant exchange                     | Send Image for Text1           | Edit Fields for "Build Memory_auto1 Document1" | ## Main Chat workflow                                  |
| Edit Fields for "Build Memory_auto1 Document1" | Set               | Prepares memory document with summary & messages       | Summarize Chat3               | Save Conversation Memory on memory_auto4 |                                                        |
| Save Conversation Memory on memory_auto4 | MongoDB                   | Saves summarized conversation to memory_auto collection | Edit Fields for "Build Memory_auto1 Document1" | -                                    |                                                        |
| Chat Agent2                | LangChain agent                 | Generates chat responses considering mood & context    | Code1                         | Extract Memory Info1, Summarize Chat2    | ## Main Chat workflow                                  |
| Extract Memory Info1        | LangChain chainLlm              | Extracts new info worth remembering from conversation  | Chat Agent2                   | Check If Worth Remembering1, Chat Agent Output Cleaner1 |                                                        |
| Check If Worth Remembering1 | If                               | Checks if extracted memory info is significant          | Extract Memory Info1          | Edit Fields5                            |                                                        |
| Edit Fields5               | Set                               | Prepares memory document for storage                     | Check If Worth Remembering1   | Save Conversation Memory on user_memory1 |                                                        |
| Save Conversation Memory on user_memory1 | MongoDB                   | Saves detailed conversation memory                       | Edit Fields5                 | -                                        |                                                        |
| Chat Agent Output Cleaner1 | Code                              | Sanitizes chat agent output for Telegram                 | Extract Memory Info1          | Send reply1                            |                                                        |
| Send reply1               | Telegram                        | Sends chat response to user                               | Chat Agent Output Cleaner1    | -                                        |                                                        |
| Summarize Chat2            | LangChain chainLlm              | Summarizes chat exchange for memory storage             | Chat Agent2                  | Edit Fields7                           |                                                        |
| Edit Fields7               | Set                               | Prepares summary document for memory_auto collection    | Summarize Chat2              | Save Conversation Memory on memory_auto3 |                                                        |
| Save Conversation Memory on memory_auto3 | MongoDB                   | Saves summarized memory document                          | Edit Fields7                 | -                                        |                                                        |
| intent from route intent1  | If                               | Checks if intentCode is reminder (4)                      | Route Intent2                | intent match checker1                   | ## Set Reminder workflow                               |
| intent match checker1      | Merge                            | Combines intent match inputs for reminder processing     | intent from string1, intent from route intent1 | Parse Natural Language Reminders to JSON1 |                                                        |
| Parse Natural Language Reminders to JSON1 | LangChain chainLlm       | Parses user reminder message into structured JSON        | intent match checker1        | Clean & Parse1                         | ## Set Reminder workflow                               |
| Clean & Parse1             | Code                              | Cleans and parses reminder JSON                           | Parse Natural Language Reminders to JSON1 | Edit Fields6                            |                                                        |
| Edit Fields6               | Set                               | Maps parsed JSON to Google Calendar parameters           | Clean & Parse1               | Google Calendar                       |                                                        |
| Google Calendar            | Google Calendar                  | Creates calendar event for reminder                        | Edit Fields6                 | Google Tasks                         |                                                        |
| Google Tasks               | Google Tasks                    | Creates Google Task for reminder                           | Google Calendar              | reminder status confirmation1          |                                                        |
| reminder status confirmation1 | LangChain chainLlm            | Generates a Telegram-friendly reminder confirmation       | Google Tasks                 | Telegram1                            |                                                        |
| Telegram1                  | Telegram                        | Sends reminder confirmation message                        | reminder status confirmation1 | -                                      |                                                        |
| Fetch User Memory3         | MongoDB                           | Retrieves user memory for voice input processing           | Format Results1              | Build Context3                       | Segment (Voice input) "Fetch memory → Context Building → Ai agent → TTS → Reply as voice message → Summarize memory → save in Mongodb" |
| Build Context3             | Code                              | Builds memory and transcription context for voice input   | Fetch User Memory3           | Chat Agent3                         |                                                        |
| Chat Agent3                | LangChain agent                 | Generates affectionate chat response for voice input      | Build Context3               | Run Edge-TTS1                      |                                                        |
| Run Edge-TTS1              | Execute Command                 | Converts chat text reply to MP3 audio using Edge-TTS      | Chat Agent3                  | Read MP3 File1                    |                                                        |
| Read MP3 File1             | Read Binary File               | Reads generated MP3 audio file                             | Run Edge-TTS1                | Send Image for Voice msg1           |                                                        |
| Send Image for Voice msg1  | Telegram                        | Sends generated voice audio message to Telegram chat      | Read MP3 File1               | Summarize Voice Message1         |                                                        |
| Summarize Voice Message1   | LangChain chainLlm             | Summarizes voice message exchange for memory storage      | Send Image for Voice msg1    | Prepare Voice Summary Document1   |                                                        |
| Prepare Voice Summary Document1 | Set                         | Prepares memory document for voice conversation            | Summarize Voice Message1     | Save Conversation Memory on memory_auto5 |                                                        |
| Save Conversation Memory on memory_auto5 | MongoDB                   | Saves voice conversation memory summary                     | Prepare Voice Summary Document1 | -                                  |                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node:**  
   - Type: Telegram Trigger  
   - Parameters: Listen to "message" updates.  
   - Credentials: Telegram Bot token.  

2. **Add Switch Node "Toggle GF Mode1":**  
   - Type: Switch  
   - Parameters: Check if message text equals "/gf on" or "/gf off".  
   - Outputs: 3 outputs - "/gf on", "/gf off", default.  

3. **Add Set Nodes "Set GF Mode TRUE1" and "Set GF Mode FALSE1":**  
   - Type: Set  
   - TRUE: Set field `gfMode` = true, `chat_id` = user chat ID.  
   - FALSE: Set field `gfMode` = false, `chat_id` = user chat ID.  

4. **Add Google Sheets Nodes "Google Sheets: Set GF Mode TRUE1" and "Google Sheets:Set GF Mode FALSE1":**  
   - Type: Google Sheets  
   - Operation: Append or update row keyed by `chat_id` in your Google Sheet.  
   - Credentials: Google Sheets OAuth2.  
   - Sheet: Provide your Google Sheet ID and sheet name ("gid=0").  

5. **Add Telegram Nodes "Send Mode Confirmation : /gf_on1" and "Send Mode Confirmation : /gf_off1":**  
   - Type: Telegram  
   - Parameters: Send confirmation messages to user chat ID, disable attribution.  

6. **Add If Node "Check Voice Message1":**  
   - Type: If  
   - Condition: Check if incoming message has `voice` object.  
   - Output 1: Process voice pipeline.  
   - Output 2: Process normal text pipeline.  

7. **Voice Pipeline:**  
   a. HTTP Request "HTTP Request Fetch Voice Message Audio1": GET Telegram API `/getFile` with voice file_id.  
   b. HTTP Request "Submit Audio Transcription Job1": POST to AssemblyAI with audio URL, language detection, punctuation, etc.  
   c. Set "Store Transcript ID1": Store transcript_id, status, audio_url, timestamp.  
   d. Wait 5 Seconds "Wait 5 Seconds1".  
   e. HTTP Request "Check Transcription Status1": Poll AssemblyAI transcript status with transcript_id.  
   f. If "Is Completed?1": Check status == "completed".  
       - If completed: Proceed to Intent Analysis3.  
       - Else if error: Send error reply "Reply: Sorry, Didn’t Hear You1".  
       - Else: Wait and poll again.  
   g. Set "Format Results1": Extract transcription text, confidence, sentiment, etc.  
   h. Fetch MongoDB "Fetch User Memory3" for user memory documents.  
   i. Code "Build Context3": Build memory context and conversation history strings.  
   j. LangChain agent "Chat Agent3": Generate chat reply text in affectionate style.  
   k. Execute Command "Run Edge-TTS1": Generate MP3 audio from reply text using Edge-TTS CLI.  
   l. Read Binary File "Read MP3 File1": Load MP3 audio for sending.  
   m. Telegram "Send Image for Voice msg1": Send audio file as voice message.  
   n. LangChain chainLlm "Summarize Voice Message1": Summarize voice conversation for memory.  
   o. Set "Prepare Voice Summary Document1": Prepare DB document with summary.  
   p. MongoDB "Save Conversation Memory on memory_auto5": Save summary document.  

8. **Text Pipeline:**  
   a. Code "Extract & Validate Data1": Extract message text and user info.  
   b. Google Sheets "Check GF Mode Status1": Lookup GF mode flag for user.  
   c. Google Cloud Natural Language "Google Cloud Natural Language1": Sentiment analysis on message.  
   d. Code "Detect Mood1": Classify mood from sentiment score.  
   e. MongoDB "Fetch Auto Memory1": Load conversation summaries from "memory_auto".  
   f. MongoDB "Fetch User Memory2": Load user memory from "conversation_summaries".  
   g. Code "Build Context2": Build memory context string.  
   h. LangChain chainLlm "Intent Analysis2": Detect intent from message text.  
   i. Code "Parse Intent2": Parse JSON intent output.  
   j. Code "intent string2": Map intent string to numeric code.  
   k. Switch "Route Intent2": Route by intent code: image generation, chat, or reminder.  

9. **Image Generation Branch:**  
   a. LangChain agent "AI Agent FOR Generate Image Prompt1": Generate detailed image prompt.  
   b. Code "Clean Prompt Text1": Clean prompt text for API.  
   c. HTTP Request (Together.xyz): Send prompt to image generation API, request base64 image.  
   d. Set "Edit Fields4": Extract base64 image data.  
   e. Convert To File "Convert to File": Convert base64 to binary file.  
   f. Telegram "Send Image for Text1": Send photo to Telegram chat.  
   g. LangChain chainLlm "Summarize Chat3": Summarize exchange.  
   h. Set "Edit Fields for Build Memory_auto1 Document1": Prepare memory document.  
   i. MongoDB "Save Conversation Memory on memory_auto4": Save summary.  

10. **Chat Branch:**  
    a. Merge inputs including mood, GF mode, context.  
    b. LangChain agent "Chat Agent2": Generate chat reply with context and mood.  
    c. LangChain chainLlm "Extract Memory Info1": Extract new info to remember.  
    d. If "Check If Worth Remembering1": Only proceed if info is significant.  
    e. Set "Edit Fields5": Prepare memory document.  
    f. MongoDB "Save Conversation Memory on user_memory1": Save detailed memory.  
    g. Code "Chat Agent Output Cleaner1": Sanitize reply text.  
    h. Telegram "Send reply1": Send chat reply to user.  
    i. LangChain chainLlm "Summarize Chat2": Summarizes the exchange for memory.  
    j. Set "Edit Fields7": Prepare summary document.  
    k. MongoDB "Save Conversation Memory on memory_auto3": Save summary.  

11. **Reminder Branch:**  
    a. If "intent from route intent1": Filters intentCode == 4.  
    b. Merge "intent match checker1": Combine intent data.  
    c. LangChain chainLlm "Parse Natural Language Reminders to JSON1": Parse natural language reminder to structured JSON.  
    d. Code "Clean & Parse1": Clean and parse JSON.  
    e. Set "Edit Fields6": Map reminder data to calendar event fields.  
    f. Google Calendar: Create event.  
    g. Google Tasks: Create task with notes and due date.  
    h. LangChain chainLlm "reminder status confirmation1": Generate confirmation text.  
    i. Telegram "Telegram1": Send confirmation message.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                                                                  |
|---------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow integrates Google Gemini LLM models via LangChain nodes for natural language understanding and generation.           | Gemini model references: "models/gemini-2.0-flash", "models/gemini-2.0-flash-lite"               |
| Voice transcription uses AssemblyAI API; requires API key for authentication.                                                   | AssemblyAI API: https://www.assemblyai.com/                                                    |
| Image generation uses Together.xyz API with model "black-forest-labs/FLUX.1-schnell".                                            | API docs: https://together.xyz/                                                                |
| GF Mode is a custom persona toggle stored in Google Sheets to switch assistant style.                                            | Google Sheets document ID and sheet name must be configured for your setup.                     |
| Edge-TTS CLI is used for text-to-speech synthesis with voice "en-US-JennyNeural".                                               | Edge-TTS repo: https://github.com/rany2/edge-tts                                               |
| MongoDB is used for storing user memories and conversation summaries, supporting long-term context.                             | Collections: "user_memory", "memory_auto", "conversation_summaries"                             |
| Telegram nodes require Bot API token with full permissions for message read/send and file retrieval.                            | Telegram Bot API docs: https://core.telegram.org/bots/api                                    |
| Sentiment analysis and mood detection influence chat agent tone and response style.                                             | Mood categories: happy, neutral_positive, neutral, sad, very_sad                               |
| All AI output nodes expect plain text responses without markdown or HTML, except sanitized Telegram replies using HTML parse mode. | Telegram messages sent with parse_mode: 'HTML'                                                |

---

This structured documentation provides a complete understanding of the workflow architecture, detailed node roles and configurations, and clear instructions to recreate or modify the workflow. It highlights potential failure points such as transcription errors, API authentication issues, and JSON parsing errors, ensuring robustness in production deployment.