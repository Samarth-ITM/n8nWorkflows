Personal Google Calendar & Reminder Bot on Telegram using Gemini & Whisper

https://n8nworkflows.xyz/workflows/personal-google-calendar---reminder-bot-on-telegram-using-gemini---whisper-5784


# Personal Google Calendar & Reminder Bot on Telegram using Gemini & Whisper

### 1. Workflow Overview

This n8n workflow implements a **Personal Google Calendar & Reminder Bot on Telegram**, integrating Google Calendar, Telegram messaging, OpenAI Whisper transcription, and Google Gemini AI models. It allows users to interact with their Google Calendar via Telegram using both text and voice messages. The bot can create, update, delete, and list calendar events with natural language commands, and automatically sends personalized event reminders.

The workflow is logically divided into three primary blocks:

- **1.1 Reminder Agent Block:** Periodically checks for upcoming Google Calendar events and sends personalized reminders to a Telegram chat.
- **1.2 Calendar Agent Block:** Handles user interactions from Telegram (text or voice), processes natural language commands for calendar management via the Google Calendar API, and responds with formatted Telegram messages.
- **1.3 AI Processing & Integration Block:** Manages AI-related processing including transcription of voice messages using OpenAI Whisper, natural language understanding and response generation via Google Gemini model, and memory management for conversational context.

---

### 2. Block-by-Block Analysis

#### 2.1 Reminder Agent Block

- **Overview:**  
  This block triggers every 15 minutes to check for Google Calendar events starting in 45-60 minutes, filters duplicates, generates warm and engaging reminder messages with AI assistance, and sends reminders via Telegram.

- **Nodes Involved:**  
  - Reminder Schedule  
  - Get Upcoming Events  
  - Filter Duplicate Reminders  
  - Reminder Message Agent  
  - Send Reminder  

- **Node Details:**

  - **Reminder Schedule**  
    - *Type:* Schedule Trigger  
    - *Role:* Initiates reminder checking every 15 minutes.  
    - *Configuration:* Interval set to 15 minutes.  
    - *Inputs:* None (trigger).  
    - *Outputs:* Triggers "Get Upcoming Events".  
    - *Edge Cases:* Missed triggers if workflow paused; time zone mismatches.  

  - **Get Upcoming Events**  
    - *Type:* Google Calendar  
    - *Role:* Retrieves all calendar events starting between 45 and 60 minutes from current time.  
    - *Configuration:* Uses dynamic `timeMin` and `timeMax` expressions relative to now; uses OAuth2 credential for the Google calendar "contactmuhtadin@gmail.com".  
    - *Key Expressions:*  
      - `timeMin = $now.plus(45, 'minutes')`  
      - `timeMax = $now.plus(60, 'minutes')`  
    - *Inputs:* Trigger from schedule.  
    - *Outputs:* Event list to "Filter Duplicate Reminders".  
    - *Edge Cases:* API quota limits; empty event list; timezone inconsistencies.  

  - **Filter Duplicate Reminders**  
    - *Type:* Remove Duplicates  
    - *Role:* Prevents sending reminders for the same event multiple times by removing previously sent event IDs.  
    - *Configuration:* Deduplication based on event `id` persisted across executions.  
    - *Inputs:* Events from "Get Upcoming Events".  
    - *Outputs:* Unique events to "Reminder Message Agent".  
    - *Edge Cases:* Loss of execution history can cause duplicate reminders.  

  - **Reminder Message Agent**  
    - *Type:* LangChain Agent (AI language model integration)  
    - *Role:* Generates personalized, warm reminder messages with rich contextual understanding of the event details.  
    - *Configuration:* Uses Google Gemini 2.5 flash model with temperature 0.2 and max tokens 1000.  
    - *Prompt Details:* Complex system prompt with modules for event classification, personalization, emoji intelligence, context analysis, error handling, and message templates.  
    - *Inputs:* Filtered events.  
    - *Outputs:* Reminder text messages.  
    - *Edge Cases:* AI service errors, prompt processing failures, or unexpected empty event fields.  

  - **Send Reminder**  
    - *Type:* Telegram node  
    - *Role:* Sends the formatted reminder message to the Telegram chat.  
    - *Configuration:* Sends message with HTML parse mode, disables web page preview, uses a fixed chat ID ("YOUR_CHAT_ID") which must be replaced with actual chat ID.  
    - *Inputs:* Reminder message from AI agent.  
    - *Outputs:* None.  
    - *Edge Cases:* Telegram API errors, invalid chat ID, message length limits.  

---

#### 2.2 Calendar Agent Block

- **Overview:**  
  This block handles incoming Telegram messages (text or voice), including voice transcription, natural language understanding, and interaction with Google Calendar for event management commands. It supports creating, updating, deleting, and listing events.

- **Nodes Involved:**  
  - Start (Telegram Trigger)  
  - Switch  
  - Send Typing Indicator  
  - 📥 Download Voice  
  - OpenAI (Whisper Transcription)  
  - Edit Fields  
  - Merge  
  - Typing 2  
  - Calendar Manage! (AI Agent)  
  - Send AI Response  

- **Node Details:**

  - **Start (Telegram Trigger)**  
    - *Type:* Telegram Trigger  
    - *Role:* Entry point for incoming Telegram messages.  
    - *Configuration:* Listens for message updates; uses Telegram API credentials named "Khaisa Assistant".  
    - *Inputs:* Telegram webhook.  
    - *Outputs:* To "Switch".  
    - *Edge Cases:* Telegram webhook misconfiguration; malformed updates.  

  - **Switch**  
    - *Type:* Switch (Routing)  
    - *Role:* Routes incoming messages as either text or voice message.  
    - *Configuration:* Checks if `message.text` exists for text; `message.voice.file_id` exists for voice.  
    - *Inputs:* From Start node.  
    - *Outputs:* Two branches:  
      - Text: To "Send Typing Indicator"  
      - Voice: To "📥 Download Voice"  
    - *Edge Cases:* Unsupported message types; missing fields.  

  - **Send Typing Indicator**  
    - *Type:* Telegram node  
    - *Role:* Sends "typing..." chat action to Telegram for user feedback.  
    - *Configuration:* Uses chat ID from incoming message; operation is "sendChatAction".  
    - *Inputs:* Text messages branch.  
    - *Outputs:* To "Edit Fields".  
    - *Edge Cases:* Telegram API errors.  

  - **📥 Download Voice**  
    - *Type:* Telegram node  
    - *Role:* Downloads voice message file from Telegram for transcription.  
    - *Configuration:* Uses `voice.file_id` from Telegram message.  
    - *Inputs:* Voice messages branch from Switch.  
    - *Outputs:* To "OpenAI" node.  
    - *Edge Cases:* File download failures; large file size.  

  - **OpenAI (Whisper Transcription)**  
    - *Type:* LangChain OpenAI node  
    - *Role:* Transcribes voice message to text using OpenAI Whisper model.  
    - *Configuration:* Operation: transcribe audio; language set to Indonesian ("ID"); uses OpenAI API credentials.  
    - *Inputs:* Audio file from "Download Voice".  
    - *Outputs:* Transcription text to "Merge".  
    - *Edge Cases:* API rate limits; transcription inaccuracies; unsupported audio format.  

  - **Edit Fields**  
    - *Type:* Set node  
    - *Role:* Prepares message text for AI processing.  
    - *Configuration:* Sets `message` field to the original text message from Telegram.  
    - *Inputs:* From "Send Typing Indicator".  
    - *Outputs:* To "Merge".  
    - *Edge Cases:* Missing text message.  

  - **Merge**  
    - *Type:* Merge node  
    - *Role:* Combines transcription output and text input into a single data stream.  
    - *Configuration:* Combines all inputs.  
    - *Inputs:* From "Edit Fields" (text) and "OpenAI" (voice transcription).  
    - *Outputs:* To "Typing 2".  
    - *Edge Cases:* Synchronization delays between inputs.  

  - **Typing 2**  
    - *Type:* Telegram node  
    - *Role:* Sends a second "typing..." indicator to Telegram while processing AI response.  
    - *Configuration:* Uses chat ID from original message.  
    - *Inputs:* From "Merge".  
    - *Outputs:* To "Calendar Manage!".  
    - *Edge Cases:* API failures.  

  - **Calendar Manage!**  
    - *Type:* LangChain Agent node  
    - *Role:* Core AI agent that interprets the user's natural language commands and manages calendar events accordingly.  
    - *Configuration:* Uses Google Gemini 2.5 flash model; system prompt extensively defines calendar management behaviors including event creation, deletion, updates, listing, confirmation, error handling, and Telegram HTML formatting. Uses memory buffer for session context keyed by chat ID.  
    - *Inputs:* Message text from "Typing 2" (merged text or transcription). Also receives AI memory and AI tool responses.  
    - *Outputs:* Formatted Telegram response text to "Send AI Response".  
    - *Edge Cases:* AI errors, incomplete commands, ambiguous instructions, API failures.  

  - **Send AI Response**  
    - *Type:* Telegram node  
    - *Role:* Sends the AI-generated calendar management response back to the Telegram user.  
    - *Configuration:* Sends text with HTML parse mode, disables web page preview, uses chat ID from original message.  
    - *Inputs:* Response from "Calendar Manage!".  
    - *Outputs:* None.  
    - *Edge Cases:* Telegram API errors, message length limits.  

---

#### 2.3 AI Processing & Integration Block

- **Overview:**  
  This block integrates AI capabilities (Google Gemini model and OpenAI Whisper) for natural language understanding, transcription, and memory management to enable conversational calendar interactions.

- **Nodes Involved:**  
  - Google Gemini Model  
  - Simple Memory  
  - Create Event (Google Calendar Tool)  
  - Update Event (Google Calendar Tool)  
  - Delete Event (Google Calendar Tool)  
  - Get Events (Google Calendar Tool)  

- **Node Details:**

  - **Google Gemini Model**  
    - *Type:* LangChain Google Gemini LM Chat node  
    - *Role:* Provides AI language model capabilities for both Reminder Agent and Calendar Manage! agents.  
    - *Configuration:* Model "models/gemini-2.5-flash", temperature 0.2, max tokens 1000; uses Google Palm API credentials.  
    - *Inputs:* Connected as AI language model for Reminder Message Agent and Calendar Manage! nodes.  
    - *Outputs:* AI-generated text outputs.  
    - *Edge Cases:* API quota, latency, request failures.  

  - **Simple Memory**  
    - *Type:* LangChain Memory Buffer Window  
    - *Role:* Maintains conversational context by storing last 20 messages for each Telegram chat session keyed by chat ID.  
    - *Configuration:* Session key based on Telegram chat ID.  
    - *Inputs:* Memory input for Calendar Manage! node.  
    - *Outputs:* Provides context for AI agent.  
    - *Edge Cases:* Memory overflow, session key mismatches.  

  - **Create Event**  
    - *Type:* Google Calendar Tool (Create operation)  
    - *Role:* Creates new calendar events based on AI agent parameters.  
    - *Configuration:* Parameters `event_title`, `start_date`, `end_date` required; optional `location`, `description`; uses OAuth2 credentials.  
    - *Inputs:* AI tool input from Calendar Manage!.  
    - *Outputs:* Event creation confirmation to AI agent.  
    - *Edge Cases:* Missing required fields, API errors.  

  - **Update Event**  
    - *Type:* Google Calendar Tool (Update operation)  
    - *Role:* Updates existing calendar events based on AI instructions.  
    - *Configuration:* Requires `event_id`; optional updated fields; uses OAuth2 credentials.  
    - *Inputs:* AI tool input from Calendar Manage!.  
    - *Outputs:* Event update confirmation to AI agent.  
    - *Edge Cases:* Invalid event ID, missing fields, API errors.  

  - **Delete Event**  
    - *Type:* Google Calendar Tool (Delete operation)  
    - *Role:* Deletes calendar events identified by AI agent.  
    - *Configuration:* Requires `event_id`; confirmation handled by AI agent prompt; uses OAuth2 credentials.  
    - *Inputs:* AI tool input from Calendar Manage!.  
    - *Outputs:* Deletion confirmation to AI agent.  
    - *Edge Cases:* Event not found, permission errors.  

  - **Get Events**  
    - *Type:* Google Calendar Tool (Get operation)  
    - *Role:* Retrieves calendar events within specified date ranges for AI agent processing.  
    - *Configuration:* Dynamic `start_date` and `end_date` from AI input or defaults; OAuth2 credentials.  
    - *Inputs:* AI tool input from Calendar Manage!.  
    - *Outputs:* Event data to AI agent.  
    - *Edge Cases:* API limits, empty results.  

---

### 3. Summary Table

| Node Name               | Node Type                           | Functional Role                         | Input Node(s)                 | Output Node(s)              | Sticky Note                                                                                 |
|-------------------------|-----------------------------------|---------------------------------------|------------------------------|-----------------------------|---------------------------------------------------------------------------------------------|
| Reminder Schedule       | Schedule Trigger                   | Triggers periodic reminder checks     | None                         | Get Upcoming Events          | # Reminder Agent                                                                           |
| Get Upcoming Events     | Google Calendar                   | Fetches upcoming events from calendar | Reminder Schedule             | Filter Duplicate Reminders   | # Reminder Agent                                                                           |
| Filter Duplicate Reminders | Remove Duplicates                | Filters out already reminded events   | Get Upcoming Events           | Reminder Message Agent       | # Reminder Agent                                                                           |
| Reminder Message Agent  | LangChain Agent (AI)               | Generates personalized reminder text  | Filter Duplicate Reminders    | Send Reminder               | # Reminder Agent                                                                           |
| Send Reminder           | Telegram                          | Sends reminder message to Telegram    | Reminder Message Agent        | None                        | # Reminder Agent                                                                           |
| Start                   | Telegram Trigger                  | Entry point for Telegram messages     | Telegram                     | Switch                      | # Calendar Agent                                                                           |
| Switch                  | Switch                           | Routes text or voice Telegram messages | Start                        | Send Typing Indicator, 📥 Download Voice | # Calendar Agent                                                                           |
| Send Typing Indicator   | Telegram                         | Sends "typing..." indicator for text  | Switch (text branch)          | Edit Fields                 | # Calendar Agent                                                                           |
| 📥 Download Voice       | Telegram                         | Downloads voice message audio file    | Switch (voice branch)         | OpenAI                      | # Calendar Agent                                                                           |
| OpenAI                  | LangChain OpenAI (Whisper)       | Transcribes voice to text              | 📥 Download Voice             | Merge                       | # Calendar Agent                                                                           |
| Edit Fields             | Set                             | Prepares text message for AI processing | Send Typing Indicator         | Merge                       | # Calendar Agent                                                                           |
| Merge                   | Merge                           | Combines text and voice transcription | Edit Fields, OpenAI           | Typing 2                    | # Calendar Agent                                                                           |
| Typing 2                | Telegram                        | Sends second "typing..." indicator    | Merge                        | Calendar Manage!            | # Calendar Agent                                                                           |
| Calendar Manage!        | LangChain Agent (AI)             | Processes calendar commands via AI    | Typing 2, Get events, Create Event, Update Event, Delete Event, Simple Memory, Google Gemini Model | Send AI Response            | # Calendar Agent, # AI Processor                                                         |
| Send AI Response        | Telegram                        | Sends AI-generated calendar response  | Calendar Manage!             | None                        | # Calendar Agent                                                                           |
| Google Gemini Model     | LangChain Google Gemini LM Chat | AI language model for agents           | Reminder Message Agent, Calendar Manage! | Reminder Message Agent, Calendar Manage! | # AI Processor                                                                           |
| Simple Memory           | LangChain Memory Buffer Window  | Maintains conversational context      | Calendar Manage!             | Calendar Manage!            | # AI Processor                                                                           |
| Create Event            | Google Calendar Tool (Create)    | Creates calendar events                | Calendar Manage! (AI tool)   | Calendar Manage! (AI tool)  | # AI Processor                                                                           |
| Update Event            | Google Calendar Tool (Update)    | Updates calendar events                | Calendar Manage! (AI tool)   | Calendar Manage! (AI tool)  | # AI Processor                                                                           |
| Delete Event            | Google Calendar Tool (Delete)    | Deletes calendar events                | Calendar Manage! (AI tool)   | Calendar Manage! (AI tool)  | # AI Processor                                                                           |
| Get Events              | Google Calendar Tool (Get)       | Retrieves calendar events              | Calendar Manage! (AI tool)   | Calendar Manage! (AI tool)  | # AI Processor                                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node ("Start")**  
   - Type: Telegram Trigger  
   - Listen to message updates  
   - Set credentials with your Telegram bot API (e.g., "Khaisa Assistant")  

2. **Add Switch Node ("Switch")**  
   - Configure two outputs:  
     - Text branch: condition if `message.text` exists  
     - Voice branch: condition if `message.voice.file_id` exists  
   - Connect "Start" to "Switch"  

3. **Text Branch: Send Typing Indicator ("Send Typing Indicator")**  
   - Type: Telegram  
   - Operation: sendChatAction ("typing")  
   - Chat ID: `{{$json.message.chat.id}}`  
   - Connect "Switch" text branch to this node  

4. **Text Branch: Edit Fields ("Edit Fields")**  
   - Type: Set node  
   - Set field `message` to `{{$json.message.text}}`  
   - Connect "Send Typing Indicator" to "Edit Fields"  

5. **Voice Branch: Download Voice ("📥 Download Voice")**  
   - Type: Telegram  
   - Download file by `{{$json.message.voice.file_id}}`  
   - Connect "Switch" voice branch to this node  

6. **Voice Branch: OpenAI Whisper Transcription ("OpenAI")**  
   - Type: LangChain OpenAI node  
   - Operation: transcribe audio  
   - Language: Indonesian ("ID")  
   - Use OpenAI API credentials  
   - Connect "📥 Download Voice" to "OpenAI"  

7. **Merge Node ("Merge")**  
   - Type: Merge node  
   - Mode: Combine all inputs  
   - Connect "Edit Fields" and "OpenAI" nodes to "Merge"  

8. **Typing Indicator ("Typing 2")**  
   - Type: Telegram, sendChatAction "typing"  
   - Chat ID: from original message  
   - Connect "Merge" to "Typing 2"  

9. **Create AI Agent Node ("Calendar Manage!")**  
   - Type: LangChain Agent  
   - Use Google Gemini 2.5 flash model with temperature 0.2  
   - Set system prompt for Google Calendar assistant with detailed instructions for create, update, delete, list, and error handling  
   - Connect "Typing 2" as main input  
   - Connect AI tool nodes ("Create Event", "Update Event", "Delete Event", "Get Events") and "Simple Memory" as AI tool and memory inputs  
   - Use Google Gemini API credentials and Google Calendar OAuth2 credentials  

10. **Send AI Response (Telegram Send Message)**  
    - Type: Telegram  
    - Text: `{{$json}}` from "Calendar Manage!" output  
    - Chat ID: from original message  
    - Use HTML parse mode, disable web page preview  
    - Connect "Calendar Manage!" output to this node  

11. **Add Google Calendar Tool Nodes for AI Tool Integration:**  
    - **Create Event:** with required fields (event_title, start_date, end_date), optional fields (location, description)  
    - **Update Event:** requires event_id, optional updated fields  
    - **Delete Event:** requires event_id  
    - **Get Events:** accepts start_date and end_date, returns events  
    - Use Google Calendar OAuth2 credentials  

12. **Add Memory Node ("Simple Memory")**  
    - Type: LangChain Memory Buffer Window  
    - Session key: Telegram chat ID  
    - Context window length: 20 messages  

13. **Reminder Agent Periodic Flow:**  
    - **Reminder Schedule:** Schedule trigger every 15 minutes  
    - **Get Upcoming Events:** Fetch events starting 45-60 minutes from now  
    - **Filter Duplicate Reminders:** Remove events already reminded  
    - **Reminder Message Agent:** LangChain Agent node with Google Gemini model, prompt for warm personalized reminders  
    - **Send Reminder:** Telegram node sends reminder message (use your chat ID)  

14. **Credentials Setup:**  
    - Telegram API: for bot interactions  
    - Google Calendar OAuth2 API: for calendar access  
    - OpenAI API: for voice transcription  
    - Google Palm API (Google Gemini): for AI language models  

15. **Additional Setup:**  
    - Replace placeholder chat IDs with actual Telegram chat IDs  
    - Configure OAuth2 credentials correctly  
    - Customize system prompts as desired for personality and language  
    - Test with sample voice and text messages  

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| Main bot features: Create, view, update, delete events; voice messages in Indonesian. | Sticky Note3 (How to Use the Bot) |
| Setup requirements: Telegram Bot via @BotFather, Google Calendar OAuth2, OpenAI API, Google Gemini AI. | Sticky Note4 (Bot Setup Requirements) |
| Useful links: [n8n Documentation](https://docs.n8n.io), [Telegram Bot API](https://core.telegram.org/bots/api), [Google Calendar API](https://developers.google.com/calendar), [OpenAI API Docs](https://platform.openai.com/docs) | Sticky Note4 (Useful Resources) |
| Auto reminder feature sends notifications 45-60 minutes before events. | Sticky Note4 |
| Example user commands for creating, viewing, updating, deleting events provided for user guidance. | Sticky Note5 (Example Commands) |

---

**Disclaimer:**  
The text provided is generated from an automated n8n workflow, fully compliant with current content policies and containing no illegal or offensive material. All data processed are lawful and publicly accessible.