Create a Customer Support Voice Agent with GPT-5, ElevenLabs & Calendar Booking

https://n8nworkflows.xyz/workflows/create-a-customer-support-voice-agent-with-gpt-5--elevenlabs---calendar-booking-8074


# Create a Customer Support Voice Agent with GPT-5, ElevenLabs & Calendar Booking

### 1. Workflow Overview

This workflow implements a Customer Support Voice Agent powered by GPT-5, ElevenLabs, and Google Calendar integration. It is designed to handle multilingual voice or chat requests, provide knowledgebase answers, check calendar availability, book appointments, and send email confirmations — all fully automated and optimized for French and other languages with voice interaction capabilities.

The workflow consists of three main logical blocks:

- **1.1 Input Reception & Understanding**  
  Captures incoming user requests (voice or text) via a webhook, transcribes and interprets them using an AI agent (GPT-5), and queries a Google Sheets knowledgebase for factual answers.

- **1.2 Scheduling & Appointment Management**  
  Handles calendar operations by checking availability, proposing free slots, confirming bookings, creating appointments in Google Calendar, and sending email confirmations through Gmail.

- **1.3 Response Delivery & Notifications**  
  Returns the AI-generated response to the user via webhook, supports multilingual voice response generation (ElevenLabs), and enables system logging or notifications (not explicitly configured here but indicated).

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Understanding

**Overview:**  
This block receives the user's voice or chat input, transcribes it, and processes it with GPT-5 to understand intent, fetch knowledgebase data if relevant, or initiate scheduling logic.

**Nodes Involved:**  
- Webhook: Receive User Request (ElevenLabs)  
- AI Agent: Multilingual Vocal Support (GPT-5)  
- OpenAI GPT-5 Model  
- Reasoning Tool (LangChain)  
- Knowledgebase Lookup

**Node Details:**

- **Webhook: Receive User Request (ElevenLabs)**  
  - *Type:* Webhook  
  - *Role:* Entry point receiving POST requests containing user queries (text or voice transcription).  
  - *Configuration:* HTTP POST method on a unique webhook path; response mode set to respond via a dedicated response node.  
  - *Input/Output:* Receives external request → outputs request data to AI Agent.  
  - *Failure Modes:* Missing or malformed requests; network issues; unauthorized requests if no security configured.  
  - *Notes:* This node initiates the workflow and expects data from ElevenLabs or similar voice platforms.

- **AI Agent: Multilingual Vocal Support (GPT-5)**  
  - *Type:* LangChain Agent node integrating OpenAI GPT-5 and other tools.  
  - *Role:* Core AI logic interpreting the last user request, deciding on actions (knowledge query, availability check, booking).  
  - *Configuration:*  
    - Input text uses the transcription field `{{$json.body.searchQuery}}`.  
    - System message defines detailed goals, rules, constraints, and tool usage, including strict timezone conversions (Paris → CST) and response style rules (short, simple, ≤3 sentences).  
    - Tools integrated: Reasoning Tool, Knowledgebase Lookup, Calendar availability check, Appointment creation, Email sending.  
  - *Input/Output:* Receives transcription → outputs AI decision and response data.  
  - *Failure Modes:* Expression evaluation errors in dynamic fields; API errors from OpenAI or integrated tools; logic errors if tools called multiple times incorrectly.  
  - *Version:* Requires LangChain agent support for GPT-5 and tool chaining (v2.2+).  
  - *Notes:* The agent enforces business rules, including booking restrictions and conversation style.

- **OpenAI GPT-5 Model**  
  - *Type:* Language Model (Chat Completion) node  
  - *Role:* Provides GPT-5 language model responses to the agent.  
  - *Configuration:* Model set explicitly to "gpt-5-mini" variant; OpenAI API credentials configured.  
  - *Input/Output:* Receives prompt from the AI Agent → returns generated text.  
  - *Failure Modes:* API rate limits, invalid credentials, network errors.

- **Reasoning Tool (LangChain)**  
  - *Type:* LangChain Tool node ("think" tool)  
  - *Role:* Supports the AI Agent by enabling reasoning or planning steps.  
  - *Configuration:* Minimal, acts as a callable tool within the AI Agent.  
  - *Failure Modes:* N/A as it is internal logic support.

- **Knowledgebase Lookup**  
  - *Type:* Google Sheets Tool node  
  - *Role:* Fetches company or FAQ data from a Google Sheets document used as the knowledgebase.  
  - *Configuration:* Document ID and sheet name configured via expressions or environment variables; OAuth2 credentials provided.  
  - *Input/Output:* Called once per request by the AI Agent for factual answers.  
  - *Failure Modes:* Authentication errors; missing or malformed sheet data; rate limits.  
  - *Notes:* Responses must avoid referencing sheets or tools explicitly.

---

#### 2.2 Scheduling & Appointment Management

**Overview:**  
This block manages calendar availability queries, proposes free time slots, creates bookings upon confirmation, and sends booking confirmation emails.

**Nodes Involved:**  
- Calendar: Check Availability  
- Calendar: Create Appointment  
- Gmail: Send Booking Confirmation

**Node Details:**

- **Calendar: Check Availability**  
  - *Type:* Google Calendar Tool node  
  - *Role:* Queries the calendar for free/busy slots to confirm availability.  
  - *Configuration:* Uses OAuth2 credentials; timeMin and timeMax dynamically set by AI Agent; returns all events in given interval.  
  - *Input/Output:* Called multiple times to find at least two free timeslots for scheduling.  
  - *Failure Modes:* Auth failure; network timeout; invalid time window parameters.

- **Calendar: Create Appointment**  
  - *Type:* Google Calendar Tool node  
  - *Role:* Creates a 1-hour appointment in the calendar after client confirmation.  
  - *Configuration:* Dynamically sets start/end times and event summary/description from AI Agent.  
  - *Input/Output:* Called once per request to book the appointment.  
  - *Failure Modes:* Booking conflicts (slot no longer free); repeated calls causing failure; auth errors.

- **Gmail: Send Booking Confirmation**  
  - *Type:* Gmail node (OAuth2)  
  - *Role:* Sends a confirmation email to the client once the appointment is booked.  
  - *Configuration:* Recipient, subject, and message dynamically provided by AI Agent; OAuth2 credentials for Gmail.  
  - *Input/Output:* Single email sent per booking request confirming details in Paris timezone.  
  - *Failure Modes:* Missing client email requiring fallback; SMTP errors; quota limits.

---

#### 2.3 Response Delivery & Notifications

**Overview:**  
This final block sends the AI-generated response back to the user and supports voice output via ElevenLabs. It also enables system logging or notifications (optional).

**Nodes Involved:**  
- Webhook: Return AI Response (ElevenLabs)

**Node Details:**

- **Webhook: Return AI Response (ElevenLabs)**  
  - *Type:* Respond to Webhook node  
  - *Role:* Sends the AI agent’s response back to the user’s frontend or voice platform.  
  - *Configuration:* Default response mode; no content transformation configured here.  
  - *Input/Output:* Receives processed AI output → returns response over HTTP.  
  - *Failure Modes:* Timeout or network failures; malformed responses; client disconnections.

---

### 3. Summary Table

| Node Name                          | Node Type                       | Functional Role                           | Input Node(s)                           | Output Node(s)                           | Sticky Note                                                                                      |
|-----------------------------------|--------------------------------|-----------------------------------------|---------------------------------------|-----------------------------------------|------------------------------------------------------------------------------------------------|
| Webhook: Receive User Request (ElevenLabs) | Webhook                        | Entry point receiving user requests     | -                                     | AI Agent: Multilingual Vocal Support (GPT-5) | ## 🟢 Step 1 — Receive & Understand Request: Receives input from website/voice platform.         |
| AI Agent: Multilingual Vocal Support (GPT-5) | LangChain Agent                | Processes user input, applies AI logic  | Webhook: Receive User Request (ElevenLabs), OpenAI GPT-5 Model, Reasoning Tool, Knowledgebase Lookup, Calendar nodes, Gmail node | Webhook: Return AI Response (ElevenLabs)     | ## 🟢 Step 1 — Receive & Understand Request: Core AI logic with tools and constraints.           |
| OpenAI GPT-5 Model                 | LangChain LM Chat OpenAI        | Provides GPT-5 language model responses | AI Agent: Multilingual Vocal Support (GPT-5) | AI Agent: Multilingual Vocal Support (GPT-5) | ## 🟢 Step 1 — Receive & Understand Request: GPT-5 model for language generation.                |
| Reasoning Tool (LangChain)         | LangChain Tool Think            | Supports AI reasoning/planning          | AI Agent: Multilingual Vocal Support (GPT-5) | AI Agent: Multilingual Vocal Support (GPT-5) | ## 🟢 Step 1 — Receive & Understand Request: Tool for AI reasoning steps.                        |
| Knowledgebase Lookup               | Google Sheets Tool              | Fetches knowledgebase data               | AI Agent: Multilingual Vocal Support (GPT-5) | AI Agent: Multilingual Vocal Support (GPT-5) | ## 🟢 Step 1 — Receive & Understand Request: Accesses company FAQ database.                      |
| Calendar: Check Availability       | Google Calendar Tool            | Checks calendar free/busy status         | AI Agent: Multilingual Vocal Support (GPT-5) | AI Agent: Multilingual Vocal Support (GPT-5) | ## 🔵 Step 2 — Handle Scheduling & Appointments: Checks real-time calendar availability.        |
| Calendar: Create Appointment       | Google Calendar Tool            | Creates appointment events in calendar   | AI Agent: Multilingual Vocal Support (GPT-5) | AI Agent: Multilingual Vocal Support (GPT-5) | ## 🔵 Step 2 — Handle Scheduling & Appointments: Books confirmed slots.                         |
| Gmail: Send Booking Confirmation   | Gmail Tool                     | Sends confirmation emails to clients     | AI Agent: Multilingual Vocal Support (GPT-5) | AI Agent: Multilingual Vocal Support (GPT-5) | ## 🔵 Step 2 — Handle Scheduling & Appointments: Sends booking confirmation email with details. |
| Webhook: Return AI Response (ElevenLabs) | Respond to Webhook              | Sends AI response back to user           | AI Agent: Multilingual Vocal Support (GPT-5) | -                                       | ## 🟣 Step 3 — Respond to User & Notifications: Returns response and voice output.               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook to Receive User Request**  
   - Node Type: Webhook  
   - HTTP Method: POST  
   - Path: Unique identifier (e.g., `6404fe0f-aa6a-4e5a-a71c-81a6fcb606af`)  
   - Response Mode: `responseNode` (to use a dedicated response node)  
   - Purpose: Entry point for incoming voice or chat requests from ElevenLabs or similar.

2. **Add AI Agent Node (LangChain Agent)**  
   - Node Type: `@n8n/n8n-nodes-langchain.agent`  
   - Parameters:  
     - Text input: `=The call transcription :\n{{ $json.body.searchQuery }}`  
     - System Message: Detailed goals and constraints for handling requests (copy full system prompt logic including tool usage rules, timezone handling, response style).  
   - Tools to connect: Reasoning Tool, Knowledgebase Lookup, Calendar Check, Calendar Create, Gmail Send Email.  
   - Purpose: Central AI logic interpreting requests and orchestrating tool calls.

3. **Add OpenAI GPT-5 Model Node**  
   - Node Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
   - Model: `gpt-5-mini`  
   - Credentials: OpenAI API credentials configured properly.  
   - Connect Output to AI Agent's language model input.

4. **Add Reasoning Tool Node**  
   - Node Type: `@n8n/n8n-nodes-langchain.toolThink`  
   - No special parameters needed.  
   - Connect as a tool input to AI Agent.

5. **Add Knowledgebase Lookup Node**  
   - Node Type: Google Sheets Tool  
   - Parameters:  
     - Document ID: Google Sheets ID containing the knowledgebase  
     - Sheet Name: Specific sheet name or ID within the document  
   - Credentials: Google Sheets OAuth2 configured.  
   - Connect as a tool input to AI Agent.

6. **Add Calendar: Check Availability Node**  
   - Node Type: Google Calendar Tool  
   - Operation: `getAll` events between dynamic `timeMin` and `timeMax` (expressions set dynamically by AI Agent in Paris → CST timezone).  
   - Credentials: Google Calendar OAuth2 configured.  
   - Connect as a tool input to AI Agent.

7. **Add Calendar: Create Appointment Node**  
   - Node Type: Google Calendar Tool  
   - Operation: `create` event with start/end times, summary, and description dynamically set (all dates in CST, converted from Paris).  
   - Credentials: Google Calendar OAuth2 configured.  
   - Connect as a tool input to AI Agent.

8. **Add Gmail: Send Booking Confirmation Node**  
   - Node Type: Gmail Tool  
   - Parameters:  
     - Recipient email dynamically set by AI Agent  
     - Subject and message dynamically set with booking details in Paris timezone  
   - Credentials: Gmail OAuth2 configured.  
   - Connect as a tool input to AI Agent.

9. **Add Webhook: Return AI Response Node**  
   - Node Type: Respond to Webhook  
   - Purpose: Sends final AI response back to the user or frontend.  
   - Connect input from AI Agent's main output.

10. **Connect Workflow Nodes**  
    - Webhook (Receive User Request) → AI Agent  
    - AI Agent → Webhook (Return AI Response)  
    - AI Agent connects to all tool nodes as AI tools (OpenAI GPT-5, Reasoning Tool, Knowledgebase, Calendar Check/Create, Gmail)  
    - Ensure all OAuth2 credentials are properly configured and tested.

11. **Set Timezone Handling and Expression Logic**  
    - Implement timezone conversions (Paris to CST and back) in AI Agent prompts and dynamic node parameters.  
    - Follow strict rules to limit tool calls (e.g., knowledgebase only once per request, create appointment only once).

12. **Optional: Add Sticky Notes**  
    - Add descriptive sticky notes to document workflow steps and logic for maintainers.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                   | Context or Link                                                                                         |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Full video tutorial showing the workflow and setup: click the preview image or visit https://youtu.be/nlwpbXQqNQ4                                                                                                                           | https://youtu.be/nlwpbXQqNQ4 (YouTube)                                                                 |
| Detailed documentation, API config, platform connection guides, and workflow customization tips accessible on Notion.                                                                                                                       | https://automatisation.notion.site/AI-Voice-Agent-25f3d6550fd9807cbf55e202138d5291?source=copy_link    |
| The workflow enforces strict business rules for scheduling and knowledgebase queries, including timezone conversions and limiting API tool calls to avoid failures.                                                                           | Business logic embedded in AI Agent system prompt                                                     |
| The use of ElevenLabs enables multilingual voice synthesis for responses, not explicitly detailed here but implied by webhook naming and context.                                                                                            | ElevenLabs voice platform integration                                                                  |
| The workflow assumes OAuth2 credentials are configured for Google Sheets, Google Calendar, Gmail, and OpenAI services with proper permissions and scopes.                                                                                     | OAuth2 credentials setup required                                                                       |

---

*Disclaimer: The provided text is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and includes no illegal, offensive, or protected elements. All data handled is legal and public.*