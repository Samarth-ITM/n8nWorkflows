Plan voice-based travel and calendar bookings with Claude AI

https://n8nworkflows.xyz/workflows/plan-voice-based-travel-and-calendar-bookings-with-claude-ai-14914


# Plan voice-based travel and calendar bookings with Claude AI

# Technical Analysis: Voice-Activated Travel Assistant with Claude AI

This document provides a comprehensive technical breakdown of the n8n workflow designed to process voice-based travel requests via WhatsApp and Telegram, utilizing AI for intent extraction, multi-source travel searching, and automated calendar booking.

---

### 1. Workflow Overview

The workflow serves as a hands-free travel concierge. It transforms a voice note into a structured travel itinerary by orchestrating audio transcription, LLM-based entity extraction, real-time API searches for travel services, and final delivery through the original messaging platform.

**Logical Blocks:**
- **1.1 Voice Reception & Transcription:** Captures webhooks from WhatsApp/Telegram, downloads audio files, and converts speech to text using OpenAI Whisper.
- **1.2 NLP Understanding & Context Enrichment:** Uses Claude AI to classify intent and extract travel entities, then merges this with user preferences and conversation history from Google Sheets.
- **1.3 Multi-Source Travel Search & Smart Ranking:** Queries flights, hotels, and activities in parallel and applies a scoring algorithm to rank the best options based on budget and preferences.
- **1.4 Response Generation, Calendar & Delivery:** Generates a conversational voice-ready response via Claude AI, creates a Google Calendar event, and sends the final response back to the user.

---

### 2. Block-by-Block Analysis

#### 2.1 Voice Reception & Transcription
**Overview:** This block handles the entry point of the workflow, normalizing data from different messaging platforms and converting audio binaries into text.

- **Nodes Involved:** `WhatsApp Voice Message Webhook`, `Telegram Voice Note Webhook`, `Normalize Message Input`, `Download Audio File`, `Transcribe Audio with OpenAI Whisper`, `Extract and Validate Transcription`.
- **Node Details:**
    - **Webhooks (WhatsApp/Telegram):** Receive POST requests containing voice message metadata.
    - **Normalize Message Input (Code):** A JavaScript node that standardizes payloads from different platforms into a single `normalizedMessage` object (senderId, audioUrl, platform).
    - **Download Audio File (HTTP Request):** Fetches the raw audio file from the provided URL. Uses `httpHeaderAuth` for authentication.
    - **Transcribe Audio (HTTP Request):** Sends the audio binary to OpenAI's `whisper-1` model. Returns verbose JSON.
    - **Extract and Validate Transcription (Code):** Cleans the Whisper output, calculates a confidence score, and validates that the transcription is not empty.
- **Edge Cases:** Audio files too short to transcribe; missing `audioUrl` in the webhook payload; OpenAI API timeouts.

#### 2.2 NLP Understanding & Context Enrichment
**Overview:** Converts raw text into a structured travel request and injects user-specific data to fill in missing gaps.

- **Nodes Involved:** `Claude AI Intent Classification & Entity Extraction`, `Claude Sonnet 4 Model1`, `Fetch User Profile & Preferences`, `Fetch Recent Conversation Context`, `Merge Intent, Profile & Context`.
- **Node Details:**
    - **Claude AI Intent Classification (Agent):** A LangChain agent that parses the transcription. It extracts IATA codes, dates, budget, and passenger counts. It is strictly configured to output JSON.
    - **Fetch User Profile/Context (Google Sheets):** Retrieves rows from `user_profiles` and `conversation_history` based on the `senderId`.
    - **Merge Intent, Profile & Context (Code):** Logic that fills missing fields (e.g., if the user didn't specify a departure city, it uses the `defaultDepartureCity` from the Google Sheet profile).
- **Edge Cases:** User not found in Google Sheets (defaults are applied); Claude AI returning markdown instead of raw JSON (handled by regex cleaning in the Merge node).

#### 2.3 Multi-Source Travel Search & Smart Ranking
**Overview:** Executes parallel searches across travel APIs and ranks results using a custom weighted algorithm.

- **Nodes Involved:** `Search Flights`, `Search Hotels`, `Search Activities & Tours`, `Aggregate and Rank Results`.
- **Node Details:**
    - **Search Nodes (HTTP Request):** Three parallel requests to Skyscanner/Kiwi, Booking.com, and Viator.
    - **Aggregate and Rank Results (Code):** The "brain" of the search. It normalizes different API responses and applies a `calculateScore` function:
        - **Flights:** Weights price (40%), duration (30%), and stops (30%).
        - **Hotels:** Weights price (30%), rating (40%), and star match (30%).
        - **Activities:** Weights rating (50%), review count (20%), and price (30%).
- **Edge Cases:** One of the three APIs failing (handled by `continueOnFail: true`); no results found for a specific destination.

#### 2.4 Response Generation, Calendar & Delivery
**Overview:** Finalizes the user experience by creating a calendar entry and delivering a friendly, conversational response.

- **Nodes Involved:** `Claude AI Response Generation`, `Claude Sonnet 4 Model`, `Parse Response & Create Calendar Event`, `Add Event to Google Calendar`, `Send WhatsApp Response`, `Send Telegram Response`, `Send Interactive Action Buttons`, `Log Conversation to History`, `Webhook Response - Success`.
- **Node Details:**
    - **Claude AI Response Generation (Agent):** Converts the ranked search results and user profile into a warm, voice-optimized script.
    - **Parse Response (Code):** Extracts the JSON from Claude's response and formats it specifically for the Google Calendar API.
    - **Add Event (Google Calendar):** Creates a travel event with specific colors and reminders.
    - **Delivery Nodes (HTTP/Telegram):** Sends the text and interactive buttons back to the user via the original platform.
    - **Log Conversation (Google Sheets):** Appends the final interaction to the history sheet for future context.
- **Edge Cases:** Calendar API authentication errors; oversized messages exceeding WhatsApp/Telegram character limits.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| WhatsApp Voice Message Webhook | Webhook | Entry point | - | Normalize Message Input | 1. Voice Reception & Transcription |
| Telegram Voice Note Webhook | Webhook | Entry point | - | Normalize Message Input | 1. Voice Reception & Transcription |
| Normalize Message Input | Code | Data standardization | Webhooks | Download Audio File | 1. Voice Reception & Transcription |
| Download Audio File | HTTP Request | File retrieval | Normalize Message Input | Transcription, Claude Intent, Sheets | 1. Voice Reception & Transcription |
| Transcribe Audio... | HTTP Request | Speech-to-Text | Download Audio File | Extract and Validate Trans. | 1. Voice Reception & Transcription |
| Extract and Validate Trans. | Code | Text cleaning | Transcription | Merge Intent, Profile & Context | 1. Voice Reception & Transcription |
| Claude AI Intent Class... | Agent | NLP Entity Extraction | Download Audio File | Extract and Validate Trans. | 2. NLP Understanding & Context Enrichment |
| Claude Sonnet 4 Model1 | LM Chat | LLM Engine | - | Claude AI Intent Class... | 2. NLP Understanding & Context Enrichment |
| Fetch User Profile... | Google Sheets | User data retrieval | Download Audio File | Extract and Validate Trans. | 2. NLP Understanding & Context Enrichment |
| Fetch Recent Conv... | Google Sheets | Context retrieval | Download Audio File | Extract and Validate Trans. | 2. NLP Understanding & Context Enrichment |
| Merge Intent, Profile... | Code | Data unification | Extract and Validate Trans. | Search Nodes | 2. NLP Understanding & Context Enrichment |
| Search Flights... | HTTP Request | Flight API query | Merge Intent... | Aggregate and Rank Results | 3. Multi-Source Travel Search & Smart Ranking |
| Search Hotels... | HTTP Request | Hotel API query | Merge Intent... | Aggregate and Rank Results | 3. Multi-Source Travel Search & Smart Ranking |
| Search Activities... | HTTP Request | Activity API query | Merge Intent... | Aggregate and Rank Results | 3. Multi-Source Travel Search & Smart Ranking |
| Aggregate and Rank Results | Code | Ranking Algorithm | Search Nodes | Claude AI Response Gen. | 3. Multi-Source Travel Search & Smart Ranking |
| Claude AI Response Gen. | Agent | Conversational Writing | Aggregate and Rank | Parse Response | 4. Response Generation, Calendar & Delivery |
| Claude Sonnet 4 Model | LM Chat | LLM Engine | - | Claude AI Response Gen. | 4. Response Generation, Calendar & Delivery |
| Parse Response... | Code | Calendar formatting | Claude AI Response Gen. | Calendar, Delivery Nodes | 4. Response Generation, Calendar & Delivery |
| Add Event to Google Cal. | Google Calendar | Booking event | Parse Response | Send WhatsApp Response | 4. Response Generation, Calendar & Delivery |
| Send WhatsApp Response | HTTP Request | User Delivery | Add Event... | Log Conversation | 4. Response Generation, Calendar & Delivery |
| Send Telegram Response | Telegram | User Delivery | Parse Response | Log Conversation | 4. Response Generation, Calendar & Delivery |
| Send Interactive Buttons | HTTP Request | User Delivery | Parse Response | Log Conversation | 4. Response Generation, Calendar & Delivery |
| Log Conversation... | Google Sheets | Audit/History | Delivery Nodes | Webhook Response | 4. Response Generation, Calendar & Delivery |
| Webhook Response - Success | Respond to Webhook | API Confirmation | Log Conversation | - | 4. Response Generation, Calendar & Delivery |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Setup Webhooks and Input
1. Create two **Webhook** nodes: one for WhatsApp and one for Telegram. Set paths to `whatsapp-voice` and `telegram-voice`.
2. Add a **Code** node (`Normalize Message Input`) to map different payload structures (e.g., `body.entry` for WhatsApp vs `body.message` for Telegram) to a consistent JSON object.
3. Add an **HTTP Request** node to download the audio file using the URL extracted in the previous step.

#### Step 2: Audio Processing
1. Create an **HTTP Request** node targeting `https://api.openai.com/v1/audio/transcriptions`. 
   - Set method to `POST`.
   - Content-Type: `multipart-form-data`.
   - Parameters: `file` (binary), `model: whisper-1`.
2. Add a **Code** node to extract the `text` and `language` fields from the OpenAI response and add a validation check for length.

#### Step 3: Intelligence and Context
1. Add an **AI Agent** node (`Claude AI Intent Classification`).
   - Connect a **Chat Anthropic** model (Claude 3.5 Sonnet).
   - Provide a detailed system prompt instructing the AI to extract travel entities and return **only valid JSON**.
2. Create two **Google Sheets** nodes:
   - One to fetch the `user_profiles` sheet.
   - One to fetch the `conversation_history` sheet.
3. Add a **Code** node (`Merge Intent, Profile & Context`) to combine the AI-extracted JSON with the Google Sheet data, filling in missing information (like default city).

#### Step 4: Travel Search
1. Create three parallel **HTTP Request** nodes for:
   - Skyscanner (Flights)
   - Booking.com (Hotels)
   - Viator (Activities)
   - *Enable "Continue on Fail" for all three.*
2. Add a **Code** node (`Aggregate and Rank Results`) that takes the outputs of these three nodes and implements the weighted scoring logic (Price, Rating, Duration).

#### Step 5: Final Delivery
1. Add an **AI Agent** node (`Claude AI Response Generation`) connected to the Claude model. Provide a prompt to turn the ranked data into a friendly voice-optimized script.
2. Add a **Code** node to parse the AI's JSON and format a `calendarEvent` object.
3. Create a **Google Calendar** node to "Create an Event" using the formatted dates and descriptions.
4. Create three delivery nodes:
   - **HTTP Request** for WhatsApp text.
   - **Telegram** node for Telegram text.
   - **HTTP Request** for WhatsApp interactive buttons.
5. Add a **Google Sheets** node to append the session details to the `conversation_history` sheet.
6. End with a **Respond to Webhook** node to close the HTTP connection with a success status.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Requires Anthropic API Key | Claude AI Processing |
| Requires OpenAI API Key | Whisper Audio Transcription |
| Requires Google OAuth2 | Sheets & Calendar Integration |
| Requires WhatsApp Business API | Message Reception & Delivery |
| Requires Telegram Bot Token | BotFather Setup |
| User Profiles Sheet | Must contain columns: `phoneNumber`, `telegramId`, `defaultDepartureCity`, `budgetLevel` |
| Conversation History Sheet | Must contain columns: `userId`, `timestamp`, `intent`, `searchSummary` |