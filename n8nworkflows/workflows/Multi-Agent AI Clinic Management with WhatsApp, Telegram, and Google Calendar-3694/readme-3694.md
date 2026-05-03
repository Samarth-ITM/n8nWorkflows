Multi-Agent AI Clinic Management with WhatsApp, Telegram, and Google Calendar

https://n8nworkflows.xyz/workflows/multi-agent-ai-clinic-management-with-whatsapp--telegram--and-google-calendar-3694


# Multi-Agent AI Clinic Management with WhatsApp, Telegram, and Google Calendar

### 1. Workflow Overview

This workflow automates comprehensive clinic management by integrating AI-powered assistants with multi-channel communication platforms (WhatsApp and Telegram) and Google services (Calendar and Tasks). It targets healthcare clinics aiming to streamline patient communication, appointment scheduling, confirmations, reminders, and internal task management.

The workflow is logically divided into the following blocks:

- **1.1 WhatsApp Patient Interaction Flow**: Handles incoming patient messages via WhatsApp, classifies message types (text, image, audio, document), processes media (OCR, transcription), and generates AI-driven responses sent back to patients.

- **1.2 Telegram Staff Management Flow**: Manages internal clinic staff commands via Telegram for appointment rescheduling and shopping list management, using AI agents and Google Calendar/Tasks integration.

- **1.3 Appointment Reminder System**: A scheduled daily trigger queries next-day appointments, sends confirmation requests to patients via WhatsApp, and processes patient confirmations or rescheduling requests.

- **1.4 Core AI Agent Components and Memory**: Provides persistent chat memory, language models, and tool integrations (MCP system, Google Calendar, Google Tasks) that support all AI agents.

- **1.5 Media Processing Subsystem**: Specialized nodes for handling audio transcription and image OCR to convert media messages into text for AI processing.

---

### 2. Block-by-Block Analysis

#### 2.1 WhatsApp Patient Interaction Flow

**Overview:**  
This block receives incoming WhatsApp messages via a webhook, extracts relevant fields, classifies message types, processes media content (images, audio), and routes text or processed content to an AI assistant that generates patient responses. Responses are formatted and sent back through WhatsApp.

**Nodes Involved:**  
- Webhook1  
- Edit Fields1  
- Switch  
- OpenAI1 (Image OCR)  
- AI Agent2 (Image text analysis)  
- Evolution API (Media download)  
- Convert to File  
- OpenAI (Audio transcription)  
- Assistente Clínica (AI assistant for patient interaction)  
- AI Agent (Message formatting)  
- Evolution API2 (Send WhatsApp message)  
- Postgres Chat Memory1  
- MCP Google Calendar2  

**Node Details:**

- **Webhook1**  
  - Type: Webhook (HTTP POST)  
  - Role: Entry point for WhatsApp messages via Evolution API webhook.  
  - Config: Path `evolutionAPIKORE`, listens for POST requests.  
  - Inputs: Incoming WhatsApp message JSON.  
  - Outputs: Raw message data forwarded to Edit Fields1.  
  - Edge Cases: Missing or malformed webhook data; ensure credentials and webhook URL are correct.

- **Edit Fields1**  
  - Type: Set node  
  - Role: Extracts structured fields from raw webhook data (e.g., phone number, message text, media URLs).  
  - Key Expressions: Extracts `remoteJid`, `pushName`, `message.conversation`, image/audio/document URLs.  
  - Inputs: Webhook1 output.  
  - Outputs: Structured JSON for Switch node.  
  - Edge Cases: Missing fields for certain message types; null checks required.

- **Switch**  
  - Type: Switch  
  - Role: Classifies message type into text, image, audio, or document.  
  - Outputs: Routes flow based on non-empty fields.  
  - Inputs: Edit Fields1 output.  
  - Outputs:  
    - Text → Assistente Clínica  
    - Image → OpenAI1  
    - Audio → Evolution API → Convert to File → OpenAI (transcription) → Assistente Clínica  
    - Document → (Currently unused)  
  - Edge Cases: Empty or unsupported message types; ensure fallback or error handling.

- **OpenAI1**  
  - Type: OpenAI Image Analysis  
  - Role: Performs OCR and image description on received images.  
  - Config: Uses Vision model `chatgpt-4o-latest`.  
  - Inputs: Image URL from Switch.  
  - Outputs: Textual description and transcription to AI Agent2.  
  - Edge Cases: Low-quality images may yield poor OCR results.

- **AI Agent2**  
  - Type: Langchain Agent  
  - Role: Analyzes OCR text and image description to prepare AI response context.  
  - Inputs: OpenAI1 output.  
  - Outputs: Text passed to Assistente Clínica.  
  - Edge Cases: Misinterpretation of image content; ensure prompt clarity.

- **Evolution API**  
  - Type: Evolution API (media download)  
  - Role: Downloads audio media in base64 format from WhatsApp.  
  - Inputs: Audio URL from Switch.  
  - Outputs: Base64 audio data to Convert to File.  
  - Edge Cases: Large files or unsupported formats may cause failures.

- **Convert to File**  
  - Type: Convert to File  
  - Role: Converts base64 audio to binary file for transcription.  
  - Inputs: Evolution API output.  
  - Outputs: Binary audio file to OpenAI.  
  - Edge Cases: Conversion errors if base64 is corrupted.

- **OpenAI**  
  - Type: OpenAI Whisper (Audio transcription)  
  - Role: Transcribes audio to text.  
  - Inputs: Binary audio file.  
  - Outputs: Transcribed text to Assistente Clínica.  
  - Edge Cases: Poor audio quality or accents may reduce transcription accuracy.

- **Assistente Clínica**  
  - Type: Langchain Agent  
  - Role: Core AI assistant for patient WhatsApp interactions; handles scheduling, queries, cancellations.  
  - Config: System message defines clinic policies, tone, available tools (MCP Calendar, CallToHuman, etc.).  
  - Inputs: Text from Switch (text), AI Agent2 (image), OpenAI (audio).  
  - Outputs: AI-generated response text to AI Agent.  
  - Edge Cases: Ambiguous patient requests, escalation triggers CallToHuman.

- **AI Agent**  
  - Type: Langchain Agent (formatter)  
  - Role: Formats AI response text for WhatsApp markdown compliance (e.g., replacing ** with *).  
  - Inputs: Assistente Clínica output.  
  - Outputs: Formatted text to Evolution API2.  
  - Edge Cases: Formatting errors causing message delivery issues.

- **Evolution API2**  
  - Type: Evolution API (message send)  
  - Role: Sends formatted WhatsApp messages back to patients.  
  - Inputs: AI Agent output, recipient phone number from webhook data.  
  - Outputs: Message delivery confirmation.  
  - Edge Cases: API rate limits, invalid phone numbers.

- **Postgres Chat Memory1**  
  - Type: Postgres Chat Memory  
  - Role: Stores conversation context keyed by message session ID for continuity.  
  - Inputs: Webhook message ID as session key.  
  - Outputs: Context for Assistente Clínica.  
  - Edge Cases: Database connectivity issues, session key collisions.

- **MCP Google Calendar2**  
  - Type: MCP Client Tool (Calendar)  
  - Role: Provides calendar data access for Assistente Clínica (e.g., checking availability).  
  - Inputs/Outputs: Used internally by Assistente Clínica.  
  - Edge Cases: API connectivity, calendar permission issues.

---

#### 2.2 Telegram Staff Management Flow

**Overview:**  
This block enables clinic staff to manage appointments and shopping lists via Telegram commands. An AI agent interprets staff messages, interacts with Google Calendar and Google Tasks, and sends confirmation messages back to staff.

**Nodes Involved:**  
- Receber Mensagem Telegram  
- Assistente clinica interno  
- MCP Google Calendar  
- Google Tasks  
- Telegram  
- OpenAI Chat Model1  
- Postgres Chat Memory  
- MCP GMAIL  
- MCP CALENDAR  
- Enviar alerta de cancelamento  

**Node Details:**

- **Receber Mensagem Telegram**  
  - Type: Telegram Trigger  
  - Role: Entry point for staff messages via Telegram bot.  
  - Config: Listens for message updates.  
  - Inputs: Telegram messages from staff.  
  - Outputs: Message JSON to Assistente clinica interno.  
  - Edge Cases: Bot permissions, message format variations.

- **Assistente clinica interno**  
  - Type: Langchain Agent  
  - Role: AI assistant specialized in internal clinic management tasks: appointment rescheduling and shopping list updates.  
  - Config: System message defines roles, responsibilities, tone, and available tools (MCP Google Calendar, Google Tasks, Reagendar no WhatsApp).  
  - Inputs: Telegram message text.  
  - Outputs: Commands to Google Tasks, MCP Google Calendar, and Telegram nodes.  
  - Edge Cases: Ambiguous staff requests, unauthorized patient contact attempts.

- **MCP Google Calendar**  
  - Type: MCP Client Tool (Calendar)  
  - Role: Accesses and manipulates Google Calendar events for rescheduling.  
  - Inputs: Requests from Assistente clinica interno.  
  - Outputs: Event data for rescheduling or cancellation.  
  - Edge Cases: Calendar API limits, event conflicts.

- **Google Tasks**  
  - Type: Google Tasks Tool  
  - Role: Adds shopping list reminders as tasks based on staff input.  
  - Inputs: Task title and notes from Assistente clinica interno.  
  - Outputs: Confirmation of task creation.  
  - Edge Cases: OAuth token expiration, task duplication.

- **Telegram**  
  - Type: Telegram Node (send message)  
  - Role: Sends confirmation or status messages back to staff.  
  - Inputs: Text from Assistente clinica interno, chat ID from Receber Mensagem Telegram.  
  - Outputs: Message delivery confirmation.  
  - Edge Cases: Chat ID mismatches, message formatting.

- **OpenAI Chat Model1**  
  - Type: OpenAI Language Model (GPT-4.1-nano)  
  - Role: Provides natural language understanding and generation for Assistente clinica interno.  
  - Inputs: Telegram message text.  
  - Outputs: AI-generated responses and commands.  
  - Edge Cases: API rate limits, prompt misinterpretation.

- **Postgres Chat Memory**  
  - Type: Postgres Chat Memory  
  - Role: Maintains conversation context for internal assistant sessions.  
  - Inputs: Session key "100" (fixed).  
  - Outputs: Context for Assistente clinica interno.  
  - Edge Cases: Database connectivity.

- **MCP GMAIL**  
  - Type: MCP Client Tool (Gmail)  
  - Role: Integrated but not explicitly detailed; likely for email notifications or escalations.  
  - Inputs/Outputs: Connected to Assistente de confirmação.  
  - Edge Cases: Gmail API permissions.

- **MCP CALENDAR**  
  - Type: MCP Client Tool (Calendar)  
  - Role: Used by Assistente de confirmação and Assistente clinica interno for calendar operations.  
  - Edge Cases: Same as MCP Google Calendar.

- **Enviar alerta de cancelamento**  
  - Type: Telegram Tool  
  - Role: Sends cancellation alerts to clinic managers via Telegram after appointment deletions.  
  - Inputs: Text and chat ID from Assistente Clínica.  
  - Outputs: Telegram message confirmation.  
  - Edge Cases: Telegram API issues.

---

#### 2.3 Appointment Reminder System

**Overview:**  
This block triggers every weekday morning to check next-day appointments, sends confirmation requests to patients via WhatsApp, and delegates patient responses to other agents.

**Nodes Involved:**  
- Gatilho diário (Daily Trigger)  
- Assistente de confirmação  
- MCP CALENDAR  
- REMINDER (Evolution API message send)  
- OpenAI Chat Model2  

**Node Details:**

- **Gatilho diário**  
  - Type: Schedule Trigger  
  - Role: Fires at 08:00 AM Monday through Friday.  
  - Outputs: Trigger signal to Assistente de confirmação.  
  - Edge Cases: Timezone misconfigurations.

- **Assistente de confirmação**  
  - Type: Langchain Agent  
  - Role: Specialized agent that:  
    1. Lists next-day appointments from MCP Calendar.  
    2. Extracts patient phone numbers from event descriptions.  
    3. Sends confirmation messages via the “relembraAGENDAMENTO” tool.  
  - Inputs: Trigger from Gatilho diário.  
  - Outputs: Commands to MCP Calendar and REMINDER nodes.  
  - Edge Cases: Missing or malformed event descriptions; ensure phone numbers are present.

- **MCP CALENDAR**  
  - Type: MCP Client Tool (Calendar)  
  - Role: Provides event data for Assistente de confirmação.  
  - Edge Cases: API limits, calendar sync issues.

- **REMINDER**  
  - Type: Evolution API (message send)  
  - Role: Sends WhatsApp confirmation requests to patients.  
  - Inputs: Message text from Assistente de confirmação, patient phone number.  
  - Edge Cases: Message delivery failures, invalid numbers.

- **OpenAI Chat Model2**  
  - Type: OpenAI Language Model (GPT-4.1-mini)  
  - Role: Supports Assistente de confirmação with natural language processing.  
  - Edge Cases: API limits.

---

#### 2.4 Core AI Agent Components and Memory

**Overview:**  
This block contains foundational components used by multiple agents: language models, persistent chat memory, and MCP tools for calendar and email integration.

**Nodes Involved:**  
- OpenAI Chat Model  
- OpenRouter Chat Model1  
- OpenRouter Chat Model2  
- Postgres Chat Memory  
- Postgres Chat Memory1  
- MCP Google Calendar  
- MCP Google Calendar2  
- MCP GMAIL  
- MCP CALENDAR  
- CallToHuman (Sub-workflow)  

**Node Details:**

- **OpenAI Chat Model / OpenRouter Chat Models**  
  - Type: Language Models (OpenAI GPT-4.1 variants, Google Gemini)  
  - Role: Provide natural language understanding and generation for agents.  
  - Edge Cases: API rate limits, model availability.

- **Postgres Chat Memory / Postgres Chat Memory1**  
  - Type: Persistent memory nodes  
  - Role: Store and retrieve conversation context keyed by session IDs.  
  - Edge Cases: Database connectivity, data consistency.

- **MCP Google Calendar / MCP Google Calendar2 / MCP CALENDAR**  
  - Type: MCP Client Tools  
  - Role: Interface with Google Calendar for event management.  
  - Edge Cases: API permissions, quota limits.

- **MCP GMAIL**  
  - Type: MCP Client Tool (Gmail)  
  - Role: Email integration for notifications or escalations.  
  - Edge Cases: OAuth token expiration.

- **CallToHuman**  
  - Type: Tool Workflow (Sub-workflow)  
  - Role: Escalates conversations to human operators for urgent or out-of-scope cases.  
  - Inputs: Patient name, phone, last message.  
  - Edge Cases: Workflow availability, input validation.

---

#### 2.5 Media Processing Subsystem

**Overview:**  
Handles specialized processing of media messages (audio and images) sent by patients to convert them into text for AI understanding.

**Nodes Involved:**  
- Evolution API (media download)  
- Convert to File  
- OpenAI (audio transcription)  
- OpenAI1 (image OCR and description)  
- AI Agent2 (image text analysis)  

**Node Details:**

- **Evolution API**  
  - Downloads media files (audio) in base64 format from WhatsApp.  
  - Edge Cases: Large files, unsupported formats.

- **Convert to File**  
  - Converts base64 audio to binary for transcription.  
  - Edge Cases: Conversion errors.

- **OpenAI (audio transcription)**  
  - Transcribes audio to text using Whisper.  
  - Edge Cases: Audio quality.

- **OpenAI1 (image OCR)**  
  - Performs OCR and image description using Vision model.  
  - Edge Cases: Image clarity.

- **AI Agent2**  
  - Analyzes OCR output to prepare AI response context.  
  - Edge Cases: Misinterpretation.

---

### 3. Summary Table

| Node Name                  | Node Type                                  | Functional Role                               | Input Node(s)                 | Output Node(s)                  | Sticky Note                                                                                     |
|----------------------------|--------------------------------------------|-----------------------------------------------|------------------------------|---------------------------------|------------------------------------------------------------------------------------------------|
| Webhook1                   | Webhook                                    | Receives WhatsApp messages                      | External HTTP POST            | Edit Fields1                    | Incoming WhatsApp Webhook and Message Type Handling                                            |
| Edit Fields1               | Set                                        | Extracts structured fields from webhook data  | Webhook1                     | Switch                         | Incoming WhatsApp Webhook and Message Type Handling                                            |
| Switch                     | Switch                                     | Classifies message type                         | Edit Fields1                 | Assistente Clínica, OpenAI1, Evolution API | Incoming WhatsApp Webhook and Message Type Handling                                            |
| OpenAI1                    | OpenAI Image Analysis                       | OCR and image description                       | Switch (image)               | AI Agent2                     | Extract Text from Images                                                                        |
| AI Agent2                  | Langchain Agent                            | Analyzes OCR text for AI response context      | OpenAI1                      | Assistente Clínica             | Extract Text from Images                                                                        |
| Evolution API              | Evolution API (media download)              | Downloads audio media                           | Switch (audio)               | Convert to File                | Download Audio and Convert to MP4                                                             |
| Convert to File            | Convert to File                            | Converts base64 audio to binary                 | Evolution API                | OpenAI (audio transcription)  | Download Audio and Convert to MP4                                                             |
| OpenAI                     | OpenAI Whisper (audio transcription)       | Transcribes audio to text                        | Convert to File              | Assistente Clínica             | Download Audio and Convert to MP4                                                             |
| Assistente Clínica         | Langchain Agent                            | AI assistant for patient WhatsApp interaction  | Switch (text), AI Agent2, OpenAI | AI Agent                   | Agent Core Components (Tools, MCP, Memory, LLM Model)                                         |
| AI Agent                   | Langchain Agent (formatter)                 | Formats AI response for WhatsApp                | Assistente Clínica           | Evolution API2                | Processing and Sending WhatsApp Responses                                                     |
| Evolution API2             | Evolution API (message send)                 | Sends WhatsApp messages                         | AI Agent                    | -                             | Processing and Sending WhatsApp Responses                                                     |
| Postgres Chat Memory1      | Postgres Chat Memory                       | Stores conversation context for patient chat   | Webhook1                    | Assistente Clínica             | Agent Core Components (Tools, MCP, Memory, LLM Model)                                         |
| MCP Google Calendar2       | MCP Client Tool (Calendar)                  | Calendar access for patient assistant           | -                           | Assistente Clínica             | Agent Core Components (Tools, MCP, Memory, LLM Model)                                         |
| Receber Mensagem Telegram  | Telegram Trigger                           | Receives staff messages                         | Telegram                    | Assistente clinica interno     | Internal Clinic Assistant                                                                     |
| Assistente clinica interno | Langchain Agent                            | AI assistant for internal clinic management     | Receber Mensagem Telegram   | Telegram, MCP Google Calendar, Google Tasks | Internal Clinic Assistant                                                                     |
| MCP Google Calendar        | MCP Client Tool (Calendar)                  | Calendar access for internal assistant          | Assistente clinica interno  | Assistente clinica interno     | Internal Clinic Assistant                                                                     |
| Google Tasks               | Google Tasks Tool                          | Adds shopping list reminders                    | Assistente clinica interno  | Assistente clinica interno     | Internal Clinic Assistant                                                                     |
| Telegram                   | Telegram Node (send message)                 | Sends messages to staff                         | Assistente clinica interno  | -                             | Internal Clinic Assistant                                                                     |
| OpenAI Chat Model1         | OpenAI Language Model (GPT-4.1-nano)         | NLP for internal assistant                      | Assistente clinica interno  | Assistente clinica interno     | Agent Core Components (Tools, MCP, Memory, LLM Model)                                         |
| Postgres Chat Memory       | Postgres Chat Memory                       | Stores conversation context for internal chat   | -                           | Assistente clinica interno     | Agent Core Components (Tools, MCP, Memory, LLM Model)                                         |
| MCP GMAIL                  | MCP Client Tool (Gmail)                      | Email integration                               | -                           | Assistente de confirmação      | Agent Core Components (Tools, MCP, Memory, LLM Model)                                         |
| MCP CALENDAR               | MCP Client Tool (Calendar)                  | Calendar access for confirmation assistant      | -                           | Assistente de confirmação      | Agent Core Components (Tools, MCP, Memory, LLM Model)                                         |
| Enviar alerta de cancelamento | Telegram Tool                              | Sends cancellation alerts to staff             | Assistente Clínica          | -                             | Internal Clinic Assistant                                                                     |
| Gatilho diário             | Schedule Trigger                           | Triggers appointment confirmation daily         | -                           | Assistente de confirmação      | Appointment Confirmation Assistant                                                           |
| Assistente de confirmação  | Langchain Agent                            | Confirms next-day appointments with patients    | Gatilho diário              | REMINDER                      | Appointment Confirmation Assistant                                                           |
| REMINDER                   | Evolution API (message send)                 | Sends WhatsApp confirmation messages           | Assistente de confirmação   | -                             | Appointment Confirmation Assistant                                                           |
| OpenAI Chat Model2         | OpenAI Language Model (GPT-4.1-mini)          | NLP for confirmation assistant                  | Assistente de confirmação   | Assistente de confirmação      | Appointment Confirmation Assistant                                                           |
| OpenAI Chat Model          | OpenAI Language Model (GPT-4.1-mini)          | NLP for patient assistant                        | Assistente Clínica          | Assistente Clínica             | Agent Core Components (Tools, MCP, Memory, LLM Model)                                         |
| OpenRouter Chat Model1     | OpenRouter Language Model (Gemini)            | NLP for patient assistant                        | AI Agent                    | AI Agent                     | Agent Core Components (Tools, MCP, Memory, LLM Model)                                         |
| OpenRouter Chat Model2     | OpenRouter Language Model (Gemini)            | NLP for image analysis agent                     | AI Agent2                   | AI Agent2                    | Agent Core Components (Tools, MCP, Memory, LLM Model)                                         |
| CallToHuman                | Tool Workflow (Sub-workflow)                 | Escalates urgent or out-of-scope conversations  | Assistente Clínica          | Assistente Clínica             | Agent Core Components (Tools, MCP, Memory, LLM Model)                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node (Webhook1)**  
   - Type: Webhook (POST)  
   - Path: `evolutionAPIKORE`  
   - Purpose: Receive WhatsApp messages from Evolution API webhook.

2. **Create Set Node (Edit Fields1)**  
   - Extract fields: `number` (remoteJid), `name` (pushName), `key_id`, `text` (conversation), `type` (image mimetype), `image.url`, `audio.url`, `document.url`  
   - Connect Webhook1 → Edit Fields1.

3. **Create Switch Node (Switch)**  
   - Conditions:  
     - Text: non-empty `text` field  
     - Image: non-empty `image.url`  
     - Audio: non-empty `audio.url`  
     - Document: non-empty `document.url`  
   - Connect Edit Fields1 → Switch.

4. **Create OpenAI Image Analysis Node (OpenAI1)**  
   - Model: Vision model `chatgpt-4o-latest`  
   - Operation: Analyze image, transcribe text and describe image  
   - Input: `image.url` from Switch (image output)  
   - Connect Switch (image) → OpenAI1.

5. **Create Langchain Agent (AI Agent2)**  
   - System message: Analyze OCR text and image description to guide response  
   - Input: OpenAI1 output  
   - Connect OpenAI1 → AI Agent2.

6. **Create Evolution API Node (Evolution API)**  
   - Operation: Download media base64 for audio  
   - Input: `audio.url` from Switch (audio output)  
   - Connect Switch (audio) → Evolution API.

7. **Create Convert to File Node (Convert to File)**  
   - Operation: Convert base64 to binary file  
   - Input: Evolution API output  
   - Connect Evolution API → Convert to File.

8. **Create OpenAI Whisper Node (OpenAI)**  
   - Resource: Audio  
   - Operation: Transcribe  
   - Input: Convert to File output  
   - Connect Convert to File → OpenAI.

9. **Create Langchain Agent (Assistente Clínica)**  
   - System message: Clinic patient assistant with detailed SOP, tone, tools (MCP Calendar, CallToHuman, etc.)  
   - Input: Switch (text), AI Agent2 (image), OpenAI (audio) outputs  
   - Connect Switch (text) → Assistente Clínica  
   - Connect AI Agent2 → Assistente Clínica  
   - Connect OpenAI → Assistente Clínica.

10. **Create Langchain Agent (AI Agent)**  
    - System message: Format messages for WhatsApp markdown compliance  
    - Input: Assistente Clínica output  
    - Connect Assistente Clínica → AI Agent.

11. **Create Evolution API Node (Evolution API2)**  
    - Operation: Send WhatsApp message  
    - Input: AI Agent output, recipient number from webhook data  
    - Connect AI Agent → Evolution API2.

12. **Create Postgres Chat Memory Node (Postgres Chat Memory1)**  
    - Session Key: Message ID from webhook  
    - Context Window: 50  
    - Connect Webhook1 → Postgres Chat Memory1 → Assistente Clínica.

13. **Create MCP Google Calendar Node (MCP Google Calendar2)**  
    - SSE Endpoint: MCP calendar URL  
    - Connect to Assistente Clínica as tool.

14. **Create Telegram Trigger Node (Receber Mensagem Telegram)**  
    - Listen for staff messages  
    - Connect to Assistente clinica interno.

15. **Create Langchain Agent (Assistente clinica interno)**  
    - System message: Internal assistant for rescheduling and shopping list management  
    - Tools: MCP Google Calendar, Google Tasks, Reagendar no WhatsApp  
    - Connect Receber Mensagem Telegram → Assistente clinica interno.

16. **Create MCP Google Calendar Node (MCP Google Calendar)**  
    - SSE Endpoint: MCP calendar URL  
    - Connect to Assistente clinica interno.

17. **Create Google Tasks Node (Google Tasks)**  
    - Task list ID configured  
    - Connect to Assistente clinica interno.

18. **Create Telegram Node (Telegram)**  
    - Sends messages to staff  
    - Connect Assistente clinica interno → Telegram.

19. **Create OpenAI Chat Model Node (OpenAI Chat Model1)**  
    - Model: GPT-4.1-nano  
    - Connect to Assistente clinica interno.

20. **Create Postgres Chat Memory Node (Postgres Chat Memory)**  
    - Session Key: fixed "100"  
    - Connect to Assistente clinica interno.

21. **Create MCP GMAIL Node (MCP GMAIL)**  
    - SSE Endpoint: Gmail MCP URL  
    - Connect to Assistente de confirmação.

22. **Create MCP CALENDAR Node (MCP CALENDAR)**  
    - SSE Endpoint: MCP calendar URL  
    - Connect to Assistente de confirmação.

23. **Create Telegram Tool Node (Enviar alerta de cancelamento)**  
    - Sends cancellation alerts to staff  
    - Connect Assistente Clínica → Enviar alerta de cancelamento.

24. **Create Schedule Trigger Node (Gatilho diário)**  
    - Cron: 08:00 AM Monday to Friday  
    - Connect to Assistente de confirmação.

25. **Create Langchain Agent (Assistente de confirmação)**  
    - System message: Appointment confirmation assistant  
    - Tasks: List next-day appointments, extract phone numbers, send confirmation messages  
    - Connect Gatilho diário → Assistente de confirmação.

26. **Create Evolution API Node (REMINDER)**  
    - Sends WhatsApp confirmation messages  
    - Connect Assistente de confirmação → REMINDER.

27. **Create OpenAI Chat Model Node (OpenAI Chat Model2)**  
    - Model: GPT-4.1-mini  
    - Connect to Assistente de confirmação.

28. **Create OpenAI Chat Model Node (OpenAI Chat Model)**  
    - Model: GPT-4.1-mini  
    - Connect to Assistente Clínica.

29. **Create OpenRouter Chat Model Nodes (OpenRouter Chat Model1 and 2)**  
    - Models: Google Gemini variants  
    - Connect OpenRouter Chat Model1 → AI Agent  
    - Connect OpenRouter Chat Model2 → AI Agent2.

30. **Create Tool Workflow Node (CallToHuman)**  
    - Workflow ID: human escalation workflow  
    - Inputs: patient name, phone, last message  
    - Connect to Assistente Clínica.

---

### 5. General Notes & Resources

| Note Content                                                                                                                           | Context or Link                                                                                                                  |
|----------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| Workflow version 1.0.0 designed for n8n version 1.88.0+                                                                                 | Version compatibility                                                                                                           |
| Evolution API credentials and instance name must be configured for WhatsApp integration                                                 | Sensitive configuration step                                                                                                   |
| Telegram API credentials required for staff communication bot                                                                          | Sensitive configuration step                                                                                                   |
| Google Calendar and Google Tasks OAuth2 credentials required for appointment and task management                                        | Sensitive configuration step                                                                                                   |
| OpenAI and OpenRouter API keys required for AI language models and media processing                                                     | Sensitive configuration step                                                                                                   |
| PostgreSQL database used for persistent chat memory                                                                                     | Database setup required                                                                                                         |
| Sub-workflow "CallToHuman" handles escalation to human operators for urgent or out-of-scope conversations                               | Workflow ID: A95kslcW4H82nJuR                                                                                                  |
| Appointment confirmation messages sent daily at 08:00 AM Monday to Friday                                                               | Cron schedule: `0 8 * * 1-5`                                                                                                   |
| Clinic calendar link for patient scheduling: https://calendar.google.com/calendar/embed?src=a57a3781407f42b1ad7fe24ce76f558dc6c86fea5f349b7fd39747a2294c1654%40group.calendar.google.com&ctz=America%2FArgentina%2FBuenos_Aires | Used in system messages and SOP                                                                                                 |
| AI assistants use empathetic, professional, and clear language tones without emojis or slang                                            | Communication style guideline                                                                                                  |
| Media processing supports image OCR and audio transcription for richer patient interaction                                              | Enhances AI understanding                                                                                                      |
| Escalation to human operators triggered on urgent, out-of-scope, or dissatisfied patient messages                                       | Ensures quality of care                                                                                                        |
| All sensitive data such as phone numbers and patient details are stored securely in appointment descriptions                            | Data privacy best practice                                                                                                     |

---

This document provides a complete, structured reference for understanding, reproducing, and maintaining the "Multi-Agent AI Clinic Management" workflow integrating WhatsApp, Telegram, and Google Calendar with AI assistance.