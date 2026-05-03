Automated Property & Restaurant Bookings with AI Voice Calls via Telegram

https://n8nworkflows.xyz/workflows/automated-property---restaurant-bookings-with-ai-voice-calls-via-telegram-9850


# Automated Property & Restaurant Bookings with AI Voice Calls via Telegram

### 1. Workflow Overview

This automated workflow facilitates property viewing appointments and restaurant reservations through AI-powered voice calls initiated by Telegram commands. It leverages AI language models for natural language understanding and scheduling logic, integrates with Retell AI for voice call automation, and uses Google Calendar to manage and verify event availability.

The workflow is organized into the following logical blocks:

- **1.1 Triggers**: Receives commands from users via Telegram.
- **1.2 Parsing & Routing**: Interprets Telegram commands and routes them to the appropriate booking agent (property or restaurant).
- **1.3 Agent – Property Viewings**: Processes property viewing requests, checks calendar availability, prepares call instructions, and initiates calls.
- **1.4 Agent – Restaurant Reservations**: Handles restaurant booking requests similarly by extracting party size, time, and initiating calls.
- **1.5 Retell AI Calls**: Creates outbound calls through Retell AI HTTP API using JSON prepared by agents.
- **1.6 Webhook & Post-call Analysis**: Listens to Retell AI call end events, filters completed calls, analyzes transcripts with AI to confirm appointments, and creates Google Calendar events if confirmed.
- **1.7 Notifications**: Sends Telegram notifications upon successful call processing and appointment confirmation.

---

### 2. Block-by-Block Analysis

#### 2.1 Block: Triggers

- **Overview:**  
  Captures incoming Telegram messages as commands to start either property viewing or restaurant reservation workflows.

- **Nodes Involved:**  
  - Telegram Trigger  
  - Send a text message  
  - Switch  
  - Sticky Note: Section: Triggers

- **Node Details:**  
  - **Telegram Trigger**  
    - Type: Telegram Trigger  
    - Role: Listens for new Telegram messages of type "message" to initiate workflow.  
    - Configuration: No filters, captures all messages.  
    - Outputs: Triggers downstream parsing and routing.  
    - Edge Cases: Telegram API downtime or invalid credentials may block triggers.  
  - **Send a text message**  
    - Type: Telegram  
    - Role: Sends initial acknowledgement "Comenzando llamada" to user.  
    - Config: Sends text to chat ID extracted from Telegram Trigger message.  
    - Input: Telegram Trigger output.  
    - Output: Connects to Switch for routing.  
    - Edge Cases: Failures if chat ID invalid or bot token revoked.  
  - **Switch**  
    - Type: Switch  
    - Role: Routes commands based on whether they start with "/reserva" (restaurant) or not (property viewing).  
    - Conditions:  
      - "pisos" output if message text does not start with "/reserva"  
      - "Reserva restaurante" output if message text starts with "/reserva"  
    - Input: Send a text message output.  
    - Output: Routes to different AI agents accordingly.

---

#### 2.2 Block: Parsing & Routing

- **Overview:**  
  Splits user commands into two pathways: property viewing or restaurant reservation, preparing input for respective AI agents.

- **Nodes Involved:**  
  - Switch (from Triggers)  
  - Sticky Note: Section: Parsing & Routing

- **Node Details:**  
  - Switch node as detailed above routes commands.  
  - No additional nodes in this block; routing is main function.

---

#### 2.3 Block: Agent – Property Viewings

- **Overview:**  
  An AI agent extracts necessary details from property viewing commands, checks calendar availability, generates available slots, prepares call instructions, and triggers the call creation request.

- **Nodes Involved:**  
  - AI Agent: Property Viewings (Langchain Agent)  
  - Google Gemini Chat Model (LLM)  
  - Structured Output Parser (Langchain Output Parser)  
  - Get an event in Google Calendar (Google Calendar Tool)  
  - Retell Create Call (HTTP Request)  
  - Sticky Notes: Section: Agent – Property Viewings, Section: Retell AI Calls

- **Node Details:**  
  - **AI Agent: Property Viewings**  
    - Type: Langchain Agent  
    - Role: Parses `/cita <telefono> [direccion] [notas]` commands, normalizes Spanish phone numbers, validates, extracts address and notes, consults Google Calendar for busy slots, generates available time slots, prepares Retell AI call JSON.  
    - Configuration:  
      - Uses system message defining task, data extraction rules, date/time normalization (Europe/Madrid timezone), and slot generation logic with buffers and intervals.  
      - Input text includes Telegram command text and current date/time.  
      - Output: JSON with `from_number`, `to_number`, `retell_llm_dynamic_variables` (including address and slots).  
    - Input Connections: From `Structured Output Parser` and Google Calendar Tool (for availability).  
    - Output Connection: To `Retell Create Call`.  
    - Failure Modes: Incorrect phone format, calendar API errors, AI misinterpretation.  
  - **Google Gemini Chat Model**  
    - Type: LLM Chat Model (Google Gemini)  
    - Role: Generates structured JSON output from the AI agent prompt.  
    - Credentials: Google Palm API.  
    - Inputs: Text from AI Agent node.  
    - Outputs: JSON to `Structured Output Parser`.  
  - **Structured Output Parser**  
    - Type: Langchain Output Parser  
    - Role: Validates and auto-fixes AI JSON output, ensuring it matches schema with phone numbers, address, and slots.  
    - Input: LLM output JSON.  
    - Output: Cleaned JSON to AI Agent.  
  - **Get an event in Google Calendar**  
    - Type: Google Calendar Tool  
    - Role: Checks calendar for existing events by event ID (obtained dynamically from AI).  
    - Credentials: Google Calendar OAuth2.  
    - Input: Event ID from AI.  
    - Output: Calendar events to AI Agent for availability logic.  
  - **Retell Create Call**  
    - Type: HTTP Request  
    - Role: Sends prepared JSON to Retell AI API to initiate outbound voice call.  
    - URL: `https://api.retellai.com/v2/create-phone-call`  
    - Method: POST  
    - Headers: Content-Type application/json, Bearer authentication with Retell AI token.  
    - Input: Call JSON from AI Agent.  
    - Output: API response.  
    - Failure Modes: API authentication failure, network errors, malformed JSON.

---

#### 2.4 Block: Agent – Restaurant Reservations

- **Overview:**  
  Similar to property viewing agent, this AI agent handles `/reserva` commands, extracting phone, party size, desired time, and prepares call instructions for restaurant bookings.

- **Nodes Involved:**  
  - AI Agent Reserva Restaurantes (Langchain Agent)  
  - Google Gemini Chat Model1 (LLM)  
  - Structured Output Parser1 (Langchain Output Parser)  
  - Get an event in Google Calendar1 (Google Calendar Tool)  
  - Retell Create Call1 (HTTP Request)  
  - Sticky Note: Section: Agent – Restaurant Reservations1

- **Node Details:**  
  - **AI Agent Reserva Restaurantes**  
    - Type: Langchain Agent  
    - Role:  
      - Parses `/reserva <telefono> [direccion] [notas]` commands.  
      - Normalizes Spanish phone numbers, validates mobile format.  
      - Extracts party size and desired time.  
      - Prepares call JSON for Retell AI including override agent ID and dynamic variables.  
    - Configuration: System prompt specifies JSON output format, timezone Europe/Madrid.  
    - Input: Telegram command text routed via Switch.  
    - Output: JSON for call creation.  
  - **Google Gemini Chat Model1**  
    - Same role as Gemini 2.0 Flash but for restaurant reservation parsing.  
  - **Structured Output Parser1**  
    - Validates and fixes AI JSON output specific to restaurant reservation schema.  
  - **Get an event in Google Calendar1**  
    - Checks calendar availability for restaurant booking time slot.  
  - **Retell Create Call1**  
    - Sends call creation request to Retell AI API.  
    - Similar configuration as Retell Create Call node.

---

#### 2.5 Block: Retell AI Calls

- **Overview:**  
  Handles creation of outbound calls to customers using Retell AI API with JSON instructions generated by AI agents.

- **Nodes Involved:**  
  - Retell Create Call  
  - Retell Create Call1  
  - Sticky Note: Section: Retell AI Calls1

- **Node Details:**  
  - **Retell Create Call / Retell Create Call1**  
    - Type: HTTP Request  
    - Role: Submit call instruction JSON to Retell AI outbound call endpoint.  
    - Authentication: HTTP Bearer with Retell AI API token.  
    - Input: JSON from respective AI agents.  
    - Output: Call creation API responses.

---

#### 2.6 Block: Webhook & Post-call Analysis

- **Overview:**  
  Receives webhook callbacks from Retell AI upon call completion, filters relevant "call_ended" events, analyzes call transcripts with an AI agent to confirm appointments, and creates Google Calendar events if confirmed.

- **Nodes Involved:**  
  - Retell Webhook (Webhook)  
  - Filter  
  - AI Agent Parse Call (Langchain Agent)  
  - Gemini 2.0 Flash 2 (LLM)  
  - Create an event in Google Calendar  
  - Notify Success (Telegram)  
  - Sticky Note: Section: Webhook & Post-call Analysis

- **Node Details:**  
  - **Retell Webhook**  
    - Type: Webhook  
    - Role: Listens for POST requests from Retell AI on call events.  
    - Path: `retell-webhook-agent`  
    - Input: JSON body containing call data including transcript, status, disconnection reason.  
  - **Filter**  
    - Filters events where `body.event == "call_ended"` to process only completed calls.  
  - **AI Agent Parse Call**  
    - Type: Langchain Agent  
    - Role:  
      - Analyzes call transcript and metadata.  
      - Applies detailed rules to determine if appointment is truly confirmed based on transcript content, disconnection reason, time slots, and address.  
      - Extracts structured data: date, time, address, contact info, who hung up, summary.  
      - Attempts to create Google Calendar event if confirmed, otherwise reports failure reasons.  
    - System message contains comprehensive logic for confirmation criteria, date/time normalization, output formatting, and calendar event creation.  
  - **Gemini 2.0 Flash 2**  
    - LLM engine used by AI Agent Parse Call for analysis.  
  - **Create an event in Google Calendar**  
    - Creates the calendar event with details extracted by AI Agent if confirmation criteria met.  
    - Uses OAuth2 credentials.  
  - **Notify Success**  
    - Sends Telegram message with AI agent output summary to admin chat ID (`245284777`).  
    - Confirms successful processing or reports issues.  

---

### 3. Summary Table

| Node Name                  | Node Type                      | Functional Role                                  | Input Node(s)               | Output Node(s)                 | Sticky Note                                                                                     |
|----------------------------|--------------------------------|-------------------------------------------------|-----------------------------|-------------------------------|------------------------------------------------------------------------------------------------|
| Telegram Trigger            | Telegram Trigger               | Receives Telegram commands                       |                             | Send a text message            | **Triggers** - Telegram Trigger                                                                |
| Send a text message         | Telegram                      | Sends initial "Comenzando llamada" reply        | Telegram Trigger            | Switch                        | **Parsing & Routing** - Switch routes commands, initial Telegram reply                          |
| Switch                     | Switch                        | Routes commands to property or restaurant agent | Send a text message          | AI Agent: Property Viewings, AI Agent Reserva Restaurantes |                                                                                               |
| AI Agent: Property Viewings | Langchain Agent               | Parses property viewing commands, checks calendar, prepares call JSON | Structured Output Parser, Get an event in Google Calendar | Retell Create Call                | **Agent: Property Viewings** - LLM generates call JSON, calendar availability logic             |
| Google Gemini Chat Model    | LLM Chat Model (Google Gemini) | Generates structured JSON for property viewings | AI Agent: Property Viewings  | Structured Output Parser      |                                                                                               |
| Structured Output Parser    | Langchain Output Parser       | Validates and fixes JSON output                  | Google Gemini Chat Model     | AI Agent: Property Viewings   |                                                                                               |
| Get an event in Google Calendar | Google Calendar Tool          | Checks calendar event availability               | AI Agent: Property Viewings  | AI Agent: Property Viewings   |                                                                                               |
| Retell Create Call          | HTTP Request                  | Creates outbound call via Retell AI API          | AI Agent: Property Viewings  |                               | **Retell AI Calls** - Create outbound calls via HTTP                                          |
| AI Agent Reserva Restaurantes | Langchain Agent               | Parses restaurant reservation commands, prepares call JSON | Structured Output Parser1, Get an event in Google Calendar1 | Retell Create Call1              | **Agent: Restaurant Reservations** - LLM prepares booking call, collects party size and time    |
| Google Gemini Chat Model1   | LLM Chat Model (Google Gemini) | Generates structured JSON for restaurant bookings| AI Agent Reserva Restaurantes | Structured Output Parser1     |                                                                                               |
| Structured Output Parser1   | Langchain Output Parser       | Validates and fixes JSON output                  | Google Gemini Chat Model1    | AI Agent Reserva Restaurantes |                                                                                               |
| Get an event in Google Calendar1 | Google Calendar Tool          | Checks calendar event availability for restaurant| AI Agent Reserva Restaurantes | AI Agent Reserva Restaurantes |                                                                                               |
| Retell Create Call1         | HTTP Request                  | Creates outbound call via Retell AI API          | AI Agent Reserva Restaurantes |                               |                                                                                               |
| Retell Webhook             | Webhook                      | Receives Retell AI call event callbacks          |                             | Filter                        | **Webhook & Post-call Analysis** - Retell Webhook, filter call_ended events, AI analyzes transcript |
| Filter                     | Filter                       | Filters "call_ended" events                       | Retell Webhook              | AI Agent Parse Call           |                                                                                               |
| AI Agent Parse Call         | Langchain Agent               | Analyzes transcript, confirms appointment, creates event | Filter                       | Notify Success, Create an event in Google Calendar |                                                                                               |
| Gemini 2.0 Flash 2          | LLM Chat Model (Google Gemini) | AI model used by AI Agent Parse Call              | AI Agent Parse Call          |                               |                                                                                               |
| Create an event in Google Calendar | Google Calendar Tool          | Creates confirmed appointment event               | AI Agent Parse Call           |                               |                                                                                               |
| Notify Success              | Telegram                     | Sends appointment confirmation notification      | AI Agent Parse Call           |                               |                                                                                               |
| Section: Webhook & Post-call Analysis | Sticky Note                 | Describes webhook reception and call analysis     |                             |                               | **Webhook & Post-call Analysis** - Retell Webhook, filter call_ended events, AI analyzes transcript |
| Section: Triggers          | Sticky Note                  | Highlights Telegram trigger node                   |                             |                               | **Triggers** - Telegram Trigger                                                               |
| Section: Parsing & Routing  | Sticky Note                  | Highlights command routing and initial Telegram reply |                             |                               | **Parsing & Routing** - Switch routes commands, initial Telegram reply                          |
| Section: Agent – Property Viewings | Sticky Note                  | Explains LLM call JSON generation and calendar logic |                             |                               | **Agent: Property Viewings** - LLM generates call JSON, calendar availability logic            |
| Section: Agent – Restaurant Reservations1 | Sticky Note                  | Explains LLM restaurant booking preparation       |                             |                               | **Agent: Restaurant Reservations** - LLM prepares booking call, collects party size and time   |
| Section: Retell AI Calls1   | Sticky Note                  | Explains HTTP-based call creation via Retell AI  |                             |                               | **Retell AI Calls** - Create outbound calls via HTTP                                          |
| Setup Notes1                | Sticky Note                  | Setup instructions for RetellAI, Telegram, Google Calendar integrations |                             |                               | **QUICK SETUP: RetellAI, Telegram & Google Calendar** - Step-by-step integration and testing instructions |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure with Telegram API credentials (bot token).  
   - Set updates to listen for "message" events.  

2. **Create 'Send a text message' Node**  
   - Type: Telegram  
   - Connect input from Telegram Trigger.  
   - Configure with same Telegram API credentials.  
   - Set text: "Comenzando llamada"  
   - Set chatId to `{{$json.message.chat.id}}`.  

3. **Create Switch Node**  
   - Connect input from 'Send a text message'.  
   - Add two outputs with conditions:  
     - Output 1 "pisos": message text does NOT start with "/reserva".  
     - Output 2 "Reserva restaurante": message text starts with "/reserva".  

4. **Create Agent - Property Viewings Sub-Workflow**  
   - Components:  
     - Google Gemini Chat Model node (Google Palm API credentials).  
     - Structured Output Parser node with JSON schema for phone, address, and slots.  
     - Google Calendar Tool node (OAuth2 credentials) to fetch events.  
     - Langchain Agent node configured with system prompt for property viewing commands, timezone Europe/Madrid, date/time normalization, slot generation rules, and JSON output format.  
     - HTTP Request node to Retell AI API to create call, with HTTP Bearer authentication.  
   - Connect nodes in order: Gemini Chat Model → Output Parser → Agent → Google Calendar → Retell Create Call.  
   - Connect Switch output "pisos" to the first Gemini Chat Model node.  

5. **Create Agent - Restaurant Reservations Sub-Workflow**  
   - Components:  
     - Similar structure as property agent but with prompts tailored for restaurant reservations (extract party size, time).  
     - Google Gemini Chat Model1.  
     - Structured Output Parser1.  
     - Google Calendar Tool1.  
     - HTTP Request node for Retell AI API call creation.  
   - Connect Switch output "Reserva restaurante" to Gemini Chat Model1.  

6. **Create Retell Webhook Node**  
   - Type: Webhook  
   - Configure path as "retell-webhook-agent".  
   - HTTP method: POST.  
   - No authentication (public endpoint).  

7. **Create Filter Node**  
   - Connect input from Retell Webhook.  
   - Condition: Only proceed if `body.event == "call_ended"`.  

8. **Create AI Agent Parse Call Node**  
   - Langchain Agent node configured with comprehensive system prompt to analyze transcript and metadata, confirm appointment per complex rules, convert natural language date/time/address, and create Google Calendar event if confirmed.  
   - Uses Google Gemini Chat Model2 (Gemini 2.0 Flash 2) as AI engine.  
   - Connect Filter output to this agent.  

9. **Create Gemini 2.0 Flash 2 Node**  
   - Google Palm API credentials.  
   - Connect as AI language model for AI Agent Parse Call.  

10. **Create Google Calendar Create Event Node**  
    - Connected to AI Agent Parse Call as AI tool.  
    - Configure with OAuth2 credentials and target calendar ID.  

11. **Create Notify Success Node (Telegram)**  
    - Type: Telegram  
    - Sends message to admin chat ID with confirmation output from AI Agent Parse Call.  
    - Connect output from AI Agent Parse Call.  

12. **Connect Retell Create Call nodes**  
    - From AI Agents (property and restaurant) to respective Retell Create Call HTTP nodes.  
    - Configure HTTP headers with Retell AI API authentication (HTTP Bearer).  

13. **Create all necessary credentials:**  
    - Telegram API credentials with bot token.  
    - Retell AI HTTP Bearer credentials with API token.  
    - Google Palm API key for Gemini models.  
    - Google Calendar OAuth2 credentials with enabled API and authorized redirect URLs.  

14. **Replace placeholders:**  
    - Phone numbers (e.g., `+34949960018`) with your own outbound number.  
    - Calendar IDs with your Google Calendar ID.  
    - Chat IDs for Telegram notifications.  

15. **Test Workflow:**  
    - Send commands in Telegram to test property viewing and restaurant reservation flows.  
    - Ensure calls are created via Retell AI.  
    - Confirm webhook receives call ended events.  
    - Verify AI analysis and calendar event creation.  
    - Check Telegram notifications.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                         | Context or Link                                                                                     |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| **QUICK SETUP: RetellAI, Telegram & Google Calendar**: Steps to create credentials, configure APIs, assign tokens in n8n, and test full flow.                                                                                                                        | Setup Notes sticky note in workflow.                                                              |
| Retell AI webhook URL must be set in Retell AI dashboard to point to `https://<your-n8n-domain>/webhook/retell-webhook-agent`.                                                                                                                                          | Retell AI platform configuration.                                                                |
| Google Calendar API requires OAuth 2.0 credentials with n8n callback URL authorized in Google Cloud Console.                                                                                                                                                         | Google Cloud Console configuration.                                                              |
| Telegram Bot token and chat IDs must be correctly configured to enable message sending and receiving.                                                                                                                                                                | Telegram BotFather and Telegram user chat ID retrieval.                                           |
| AI Agents use Google Gemini (PaLM API) models, requiring Google Cloud project and Palm API key.                                                                                                                                                                       | Google Cloud AI API setup.                                                                         |
| The AI agent parsing logic includes complex natural language understanding for Spanish language date/time normalization, address normalization with accents and number conversion, and strict criteria for confirming appointments.                                   | Embedded in Langchain Agent system prompts.                                                      |
| Calendar availability logic includes slot filtering by day of week and time ranges, buffer times, and intervals to avoid overlapping events.                                                                                                                         | Embedded in AI Agent: Property Viewings prompt.                                                  |
| Retell AI call creation uses JSON with dynamic variables for address, slots, party size, and times to generate natural voice call prompts.                                                                                                                           | Retell AI API documentation.                                                                     |

---

**Disclaimer:** The provided text derives exclusively from an automated workflow constructed with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled are legal and public.