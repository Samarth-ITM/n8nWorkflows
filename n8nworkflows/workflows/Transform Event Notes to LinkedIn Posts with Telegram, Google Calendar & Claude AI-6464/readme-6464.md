Transform Event Notes to LinkedIn Posts with Telegram, Google Calendar & Claude AI

https://n8nworkflows.xyz/workflows/transform-event-notes-to-linkedin-posts-with-telegram--google-calendar---claude-ai-6464


# Transform Event Notes to LinkedIn Posts with Telegram, Google Calendar & Claude AI

### 1. Workflow Overview

This workflow automates the transformation of informal event notes sent via Telegram into polished, LinkedIn-ready posts, enriched with calendar event data and saved for future reference. It is ideal for professionals who want to quickly document and share insights from events they attend, leveraging AI to generate engaging social media content.

The workflow comprises the following logical blocks:

- **1.1 Input Reception:** Captures user messages from Telegram, expecting a format "Event Name: Personal Notes".
- **1.2 Message Parsing:** Extracts the event name and personal notes from the Telegram message.
- **1.3 Calendar Data Retrieval:** Fetches Google Calendar events from the past 7 days for matching.
- **1.4 Event Matching:** Matches the parsed event name against calendar events using fuzzy logic.
- **1.5 Prompt Formatting:** Combines matched event details and personal notes into a prompt for AI.
- **1.6 AI Content Generation:** Uses Claude Opus 4 AI model to create a professional LinkedIn post.
- **1.7 Data Formatting:** Post-processes event info and AI output for consistency and readability.
- **1.8 Data Persistence:** Stores the event data and LinkedIn post into Supabase database.
- **1.9 Confirmation Message:** Sends a Telegram confirmation upon successful saving.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Listens for incoming Telegram messages that trigger the workflow.
- **Nodes Involved:** `Telegram Trigger: receive a message`
- **Node Details:**
  - Type: Telegram Trigger
  - Role: Entry point capturing Telegram bot messages with update type "message".
  - Configuration: Listens for new messages indefinitely.
  - Inputs: Telegram user messages.
  - Outputs: Raw message JSON.
  - Credentials: Telegram API credentials (OAuth token).
  - Edge Cases: Missing or malformed messages; Telegram API downtime.

#### 2.2 Message Parsing

- **Overview:** Extracts event name and personal notes from the Telegram message text.
- **Nodes Involved:** `Format the message`
- **Node Details:**
  - Type: Code node (JavaScript)
  - Role: Parses text using colon `:` as delimiter; if absent, treats whole message as notes.
  - Key Expressions: Uses `$input.first().json?.message?.text` to access Telegram message text.
  - Inputs: Output from Telegram Trigger.
  - Outputs: JSON with `event_name` and `notes`.
  - Edge Cases: Messages without colon, empty messages, unexpected format.

#### 2.3 Calendar Data Retrieval

- **Overview:** Queries Google Calendar for events in the last 7 days to find matching events.
- **Nodes Involved:** `Search for Google Calendar`
- **Node Details:**
  - Type: Google Calendar node
  - Role: Fetches all calendar events from past 7 days up to now.
  - Parameters: 
    - `timeMin`: current time minus 7 days
    - `timeMax`: current time
    - Calendar ID: predefined calendar email.
  - Credentials: Google Calendar OAuth2 credentials.
  - Outputs: List of calendar events with metadata.
  - Edge Cases: API rate limits, authentication errors, empty calendar.

#### 2.4 Event Matching

- **Overview:** Performs fuzzy matching between parsed event name and calendar events.
- **Nodes Involved:** `Merge the message and events`, `Match the message and event`
- **Node Details:**
  - `Merge the message and events`
    - Type: Merge node
    - Role: Combines outputs of message parsing and calendar events.
    - Inputs: Parsed message and calendar events.
  - `Match the message and event`
    - Type: Code node (JavaScript)
    - Role: Implements multi-level fuzzy matching:
      - Exact summary match (case-insensitive)
      - Substring match in either direction
      - Word-based partial match (words longer than 2 characters)
    - Extracts matched event details including date, location, attendees.
    - Outputs a JSON flagging if event found, and enriched event data.
  - Edge Cases: No matching event found returns default placeholders.

#### 2.5 Prompt Formatting

- **Overview:** Creates an AI prompt combining matched event info and personal notes.
- **Nodes Involved:** `Format the matched event`
- **Node Details:**
  - Type: Code node (JavaScript)
  - Role: Constructs a multi-line prompt string for AI input.
  - Content: Includes event title, date, location, personal notes.
  - Outputs: Prompt string (`chatInput`) plus original event data.
  - Edge Cases: Missing event info replaced with fallback texts.

#### 2.6 AI Content Generation

- **Overview:** Uses Anthropic Claude Opus 4 model to generate a professional LinkedIn post.
- **Nodes Involved:** `Anthropic Chat Model`, `AI Agent`
- **Node Details:**
  - `Anthropic Chat Model`
    - Type: LangChain Anthropic LM node
    - Role: Provides Claude Opus 4 AI model backend.
    - Parameters: Model set to "claude-opus-4-20250514".
    - Credentials: Anthropic API key.
  - `AI Agent`
    - Type: LangChain Agent node
    - Role: Implements prompt handling with system message guiding tone and style.
    - Input: Prompt from previous node.
    - Output: Parsed AI response with generated LinkedIn post.
  - Edge Cases: API limits, response parsing errors, network timeouts.

#### 2.7 Data Formatting

- **Overview:** Cleans and formats event info and AI-generated post for database insertion and display.
- **Nodes Involved:** `Merge the event info and LinkedIn post`, `Format the event info and LinkedIn post`
- **Node Details:**
  - `Merge the event info and LinkedIn post`
    - Type: Merge node
    - Role: Combines AI output and original event info by position.
  - `Format the event info and LinkedIn post`
    - Type: Code node (JavaScript)
    - Role: Formats dates to MM/DD/YYYY HH:mm, cleans newlines and whitespace from texts.
    - Outputs structured JSON with fields: Event Date, Event Title, Location, Personal Notes, LinkedIn Post, Created Date.
  - Edge Cases: Invalid date formats, unexpected null fields.

#### 2.8 Data Persistence

- **Overview:** Saves the fully formatted event and LinkedIn post data to Supabase database.
- **Nodes Involved:** `Save to Supabase`
- **Node Details:**
  - Type: Supabase node
  - Role: Inserts or updates record in table "Event notes neu".
  - Parameters: Auto maps input data fields to database columns.
  - Credentials: Supabase API key and project URL.
  - Edge Cases: Database connectivity issues, permission errors.

#### 2.9 Confirmation Message

- **Overview:** Sends Telegram confirmation message upon successful database save.
- **Nodes Involved:** `Send conformation to Telegram`
- **Node Details:**
  - Type: Telegram node (message sender)
  - Role: Sends a templated confirmation message with event title, date, and timestamp.
  - Parameters: Uses chatId fixed to a specific Telegram user.
  - Credentials: Telegram API.
  - Edge Cases: Telegram API errors, chat ID invalid or blocked bot.

---

### 3. Summary Table

| Node Name                      | Node Type                               | Functional Role                                 | Input Node(s)                   | Output Node(s)                  | Sticky Note                                                                                                                           |
|--------------------------------|---------------------------------------|------------------------------------------------|--------------------------------|--------------------------------|---------------------------------------------------------------------------------------------------------------------------------------|
| Telegram Trigger: receive a message | Telegram Trigger                     | Receives Telegram messages to start workflow   |                                | Format the message             | ## 1. Telegram message trigger: When you send a message with the format "Event Name: Your personal notes" to your Telegram bot, it triggers this workflow. |
| Format the message              | Code                                  | Parses message text into event_name and notes  | Telegram Trigger               | Merge the message and events   | ## 2. Format your message: This code extracts the event name and your personal notes from the message.                                |
| Search for Google Calendar     | Google Calendar                       | Fetches last 7 days calendar events             |                                | Merge the message and events   | ## 3. Get events in the last 7 days                                                                                                   |
| Merge the message and events   | Merge                                 | Combines parsed message and calendar events    | Format the message, Search for Google Calendar | Match the message and event | ## 4. Match events in the calendar: Searches Google Calendar for matching events from past 7 days.                                    |
| Match the message and event    | Code                                  | Matches event name fuzzily to calendar events  | Merge the message and events   | Format the matched event       |                                                                                                                                       |
| Format the matched event       | Code                                  | Prepares AI prompt with event details & notes  | Match the message and event    | AI Agent, Merge the event info and LinkedIn post | ## 5. Format matched event as prompt: Combines notes and event details for AI input.                                                  |
| Anthropic Chat Model           | LangChain Anthropic LM                | AI model backend for generating LinkedIn post  | AI Agent                      | AI Agent                      | ## 6. Draft LinkedIn post: Uses Claude Opus 4 to generate professional LinkedIn post with hashtags.                                   |
| AI Agent                      | LangChain Agent                       | Handles AI prompt and output parsing            | Format the matched event, Anthropic Chat Model | Merge the event info and LinkedIn post |                                                                                                                                       |
| Merge the event info and LinkedIn post | Merge                           | Combines AI output with event info              | Format the matched event, AI Agent | Format the event info and LinkedIn post | ## 7. Merge and format LinkedIn post and event info.                                                                                  |
| Format the event info and LinkedIn post | Code                          | Cleans and formats all data fields for storage | Merge the event info and LinkedIn post | Save to Supabase             |                                                                                                                                       |
| Save to Supabase              | Supabase                              | Saves event and post data to database           | Format the event info and LinkedIn post | Send conformation to Telegram | ## 8. Save event info and LinkedIn post to Supabase database.                                                                         |
| Send conformation to Telegram  | Telegram                             | Sends confirmation back to Telegram user        | Save to Supabase              |                                | ## 9. Send confirmation message via Telegram after successful save.                                                                   |
| Sticky Note                   | Sticky Note                          | Documentation and instructions                   |                                |                                | See detailed notes below.                                                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**
   - Type: Telegram Trigger
   - Configure with Telegram Bot credentials.
   - Set update type to "message".
   - This node starts the workflow when a user sends a message.

2. **Add Code Node "Format the message"**
   - Purpose: Parse Telegram message text to extract event name and notes.
   - JavaScript code:
     ```js
     const message = $input.first().json?.message?.text;
     if (!message) {
       return { event_name: '', notes: 'No message received' };
     }
     const colonIndex = message.indexOf(':');
     if (colonIndex === -1) {
       return { event_name: '', notes: message.trim() };
     }
     return {
       event_name: message.substring(0, colonIndex).trim(),
       notes: message.substring(colonIndex + 1).trim()
     };
     ```
   - Connect from Telegram Trigger.

3. **Add Google Calendar Node "Search for Google Calendar"**
   - Type: Google Calendar
   - Set operation to "getAll".
   - Set `timeMin` to 7 days ago (`{{$now.minus({ days: 7 })}}`).
   - Set `timeMax` to now (`{{$now}}`).
   - Choose the target calendar by its ID/email.
   - Use Google Calendar OAuth2 credentials.
   - This node fetches recent events to match.

4. **Add Merge Node "Merge the message and events"**
   - Merge Mode: Append or combine inputs by position.
   - Connect inputs from "Format the message" and "Search for Google Calendar".

5. **Add Code Node "Match the message and event"**
   - Purpose: Fuzzy match parsed event name to calendar events.
   - Use the provided JavaScript that searches exact, contains, and word-based matches.
   - Outputs enriched event data or fallback placeholders.
   - Connect input from "Merge the message and events".

6. **Add Code Node "Format the matched event"**
   - Purpose: Create AI prompt string combining event info and notes.
   - Use JavaScript to build prompt text including event title, date, location, personal notes.
   - Connect input from "Match the message and event".

7. **Add LangChain Anthropic Chat Model Node "Anthropic Chat Model"**
   - Select model: `claude-opus-4-20250514`.
   - Provide Anthropic API credentials.
   - Connect input from "AI Agent" node (see next step).

8. **Add LangChain Agent Node "AI Agent"**
   - Set prompt type to "define".
   - System message: "You are my personal event notes organizer..."
   - Input: Use expression `{{$json.chatInput}}` for AI prompt.
   - Connect input from "Format the matched event".
   - Connect AI model node "Anthropic Chat Model" as language model resource.

9. **Add Merge Node "Merge the event info and LinkedIn post"**
   - Mode: Combine by position.
   - Connect inputs from "Format the matched event" and "AI Agent".

10. **Add Code Node "Format the event info and LinkedIn post"**
    - Purpose: Format dates and clean text outputs.
    - Implement helpers to format date strings and clean whitespace.
    - Output structured JSON with keys: Event Date, Event Title, Location, Personal Notes, LinkedIn Post, Created Date.
    - Connect input from "Merge the event info and LinkedIn post".

11. **Add Supabase Node "Save to Supabase"**
    - Configure with Supabase credentials.
    - Set target table: "Event notes neu".
    - Data mapping: Auto-map input data.
    - Connect input from "Format the event info and LinkedIn post".

12. **Add Telegram Node "Send conformation to Telegram"**
    - Configure with Telegram API credentials.
    - Set `chatId` to your Telegram user ID.
    - Message text includes event title, date, and current time.
    - Connect input from "Save to Supabase".

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Context or Link                                                                                                                                                                                                                              |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow transforms your quick event notes into polished LinkedIn posts automatically. Simply send a Telegram message formatted as "Event Name: Your personal notes". The system matches the event to your Google Calendar, generates a professional post using AI, and stores the data in Supabase for future access. It helps build a personal library of professional networking insights. Requirements include Telegram Bot, Google Calendar API OAuth2, Anthropic API access, and Supabase credentials. Customization can be done by modifying the AI system prompt or adding more event details. | See sticky note node content at workflow start for full detailed instructions.                                                                                                                                                               |
| The AI prompt system message instructs the AI to create a post with a tone mixing formal and fun, keeping paragraphs short and to the point, suitable for a professional LinkedIn audience.                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Found in AI Agent node configuration under systemMessage parameter.                                                                                                                                                                         |
| For Google Calendar API, ensure OAuth2 credentials have read access to the target calendar, and the calendar ID is correctly configured in the workflow.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Google Calendar node configuration.                                                                                                                                                                                                         |
| Telegram bot must be created via @BotFather, and the chatId used for confirmation messages must be validated to ensure messages reach the intended Telegram user.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Telegram Trigger and Send conformation to Telegram nodes.                                                                                                                                                                                   |
| Anthropic API key with access to Claude Opus 4 must be correctly set in credentials; monitor usage quotas and response times for stability.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Anthropic Chat Model node credentials.                                                                                                                                                                                                       |
| Supabase table "Event notes neu" should have columns matching the data fields sent by the workflow: Event Date, Event Title, Location, Personal Notes, LinkedIn Post, Created Date. Adjust table schema if extending data fields.                                                                                                                                                                                                                                                                                                                                                                                                                             | Supabase node configuration.                                                                                                                                                                                                                 |

---

**Disclaimer:** This document is based exclusively on an n8n workflow automation and complies with all applicable content policies. It contains no illegal or offensive content and handles only legal and public data.