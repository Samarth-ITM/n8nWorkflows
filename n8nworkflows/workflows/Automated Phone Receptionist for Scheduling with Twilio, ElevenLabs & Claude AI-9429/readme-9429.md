Automated Phone Receptionist for Scheduling with Twilio, ElevenLabs & Claude AI

https://n8nworkflows.xyz/workflows/automated-phone-receptionist-for-scheduling-with-twilio--elevenlabs---claude-ai-9429


# Automated Phone Receptionist for Scheduling with Twilio, ElevenLabs & Claude AI

---

## 1. Workflow Overview

This workflow implements an **Automated Phone Receptionist for Scheduling** using Twilio for telephony, ElevenLabs for voice input/output, Claude AI (Anthropic) for natural language understanding and reasoning, and Google Calendar for appointment management. It is designed to handle voice interactions for booking, modifying, and confirming appointments with calendar integration and conversational memory.

The workflow consists of the following logical blocks:

- **1.1 Input Reception:** Receives incoming user voice requests via a secured webhook endpoint connected to ElevenLabs.
- **1.2 Conversational Memory:** Maintains session context using Redis to enable coherent multi-turn conversations.
- **1.3 AI Processing and Reasoning:** Uses Claude AI as the primary conversational model, supplemented by a LangChain "think" tool for internal reasoning and calendar-related tools for availability checks and appointment creation or updates.
- **1.4 Calendar Integration:** Interacts with Google Calendar to check availability, create new appointments, and update existing events.
- **1.5 Response Delivery:** Sends the AI-generated voice response back to ElevenLabs for real-time phone interaction.
- **1.6 Configuration and Documentation:** Includes multiple sticky notes detailing setup instructions and important parameters for timezone, appointment duration, calendar IDs, Redis memory, and webhook setup.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception

**Overview:**  
This block receives real-time voice input from users through ElevenLabs via a webhook. It serves as the entry point for the workflow, capturing the caller’s utterance, session details, and call metadata.

**Nodes Involved:**  
- Webhook: Receive User Request (ElevenLabs)

**Node Details:**

- **Webhook: Receive User Request (ElevenLabs)**
  - **Type:** Webhook (HTTP POST endpoint)
  - **Role:** Accepts incoming POST requests from ElevenLabs with user utterance and call session data.
  - **Configuration:**
    - HTTP Method: POST
    - Webhook Path: User must replace `"REPLACE ME"` with their custom endpoint path.
    - Response Mode: Sends response from a downstream respond node.
  - **Key Variables:** Receives JSON payload containing sessionId, utterance text, caller phone number (`system_caller_id`), and call log.
  - **Connections:** Output connected directly to the "Voice AI Agent" node.
  - **Edge Cases:**  
    - Missing or invalid webhook path configuration will cause request failures.  
    - Payload format errors or missing expected fields can break the AI logic downstream.
    - Unauthorized or unexpected requests due to lack of authentication might occur unless handled externally.
  - **Notes:** Sticky Note #2 advises proper webhook path replacement and testing before production.

---

### 2.2 Conversational Memory

**Overview:**  
Maintains the conversational context and state per user session by storing message history in Redis. This enables the AI to remember prior interactions for coherent multi-turn conversations.

**Nodes Involved:**  
- Redis Chat Memory

**Node Details:**

- **Redis Chat Memory**
  - **Type:** LangChain Redis Chat Memory node
  - **Role:** Stores and retrieves conversation history using session ID as the key.
  - **Configuration:**
    - Session Key: Extracted from webhook payload’s `sessionId`.
    - Context Window Length: 20 messages (configurable).
  - **Credentials:** Requires configured Redis credentials.
  - **Connections:** Output connected to "Voice AI Agent" node as AI memory input.
  - **Edge Cases:**  
    - Redis connection failures or misconfiguration will cause loss of context, resulting in degraded conversational quality and lag.
    - Session ID missing or malformed may cause context collisions or loss.
  - **Notes:** Sticky Note #4 highlights the critical importance of Redis memory for performance and real-time experience.

---

### 2.3 AI Processing and Reasoning

**Overview:**  
This block orchestrates the AI reasoning and language model to understand user input, analyze context, and generate scheduling decisions or responses. It employs a combination of Claude AI chat model, a LangChain reasoning tool ("think"), and an AI agent configured with scheduling logic.

**Nodes Involved:**  
- Voice AI Agent  
- Anthropic Chat Model  
- Reasoning Tool (LangChain)

**Node Details:**

- **Voice AI Agent**
  - **Type:** LangChain Agent node
  - **Role:** Core AI orchestrator that integrates multiple tools and manages dialogue flow using defined prompt and rules.
  - **Configuration:**
    - Prompt defines context variables (current time, sessionId, utterance, caller phone, call log).
    - Defines tools available: think (reasoning), get_availability, create_appointment, update_appointment.
    - Embeds detailed booking flow rules, voice response constraints (short sentences, spelling out emails and phone numbers), and critical operational rules.
    - Uses expressions to dynamically pull variables from webhook payload.
    - Enforces business logic: times_allowed, blocked_days, minimum lead time, required fields, confirmation steps.
  - **Connections:**  
    - Receives input from webhook and Redis memory as AI memory.  
    - Uses Anthropic Chat Model as language model.  
    - Uses Reasoning Tool and calendar tools as AI tools.  
    - Outputs to Webhook Respond node.
  - **Edge Cases:**  
    - Incorrect prompt variables or missing configuration placeholders may break logic.  
    - AI model errors or timeouts.  
    - Logical errors in tool chaining could cause appointment conflicts or incomplete info collection.
  - **Notes:** Sticky Note #1 details the prompt variables and configuration options to be customized per deployment.

- **Anthropic Chat Model**
  - **Type:** LangChain Chat Model node (Claude AI)
  - **Role:** Provides natural language understanding and generation.
  - **Configuration:**
    - Model: "claude-3-5-sonnet-20241022" preset.
    - Requires Anthropic API credentials.
  - **Connections:** Output connects to Voice AI Agent’s language model input.
  - **Edge Cases:** API authentication failures, rate limits, or model unavailability.
  - **Notes:** Sticky Note #1 recommends Claude 3.5 Sonnet for best reasoning performance.

- **Reasoning Tool (LangChain)**
  - **Type:** LangChain tool (Think tool)
  - **Role:** Performs internal state analysis and reasoning before other tools.
  - **Configuration:** Defaults; invoked by Voice AI Agent.
  - **Connections:** Output connects to Voice AI Agent’s AI tool input.
  - **Edge Cases:** Failure to respond can stall the AI agent or cause incorrect reasoning steps.
  - **Notes:** Critical for maintaining conversation state and logic flow.

---

### 2.4 Calendar Integration

**Overview:**  
Manages calendar operations by checking availability, creating new appointments, and updating existing events in Google Calendar. It uses OAuth2 credentials linked to a Google account.

**Nodes Involved:**  
- Calendar: Check Availability  
- Calendar: Create Appointment  
- Update an event in Google Calendar

**Node Details:**

- **Calendar: Check Availability**
  - **Type:** Google Calendar Tool node
  - **Role:** Queries calendar events within a specified time window to check slot availability.
  - **Configuration:**
    - Operation: Get All events.
    - Inputs: `timeMin` & `timeMax` dynamically supplied by AI agent overrides.
    - Calendar ID: Must be replaced with user’s calendar ID (e.g., `xxxxx@group.calendar.google.com`).
    - Return All: Boolean override.
  - **Credentials:** Google OAuth2 credentials linked.
  - **Connections:** Tool input for Voice AI Agent.
  - **Edge Cases:**  
    - Incorrect calendar ID or permissions cause failures.  
    - API quota limits or network errors.
  - **Notes:** Sticky Note #3 provides calendar setup instructions.

- **Calendar: Create Appointment**
  - **Type:** Google Calendar Tool node
  - **Role:** Creates a new calendar event with appointment details.
  - **Configuration:**
    - Inputs: Start/end times, summary, attendees (customer email), description, send updates set to "all".
    - Calendar ID: Must be replaced.
  - **Credentials:** Same Google OAuth2 credentials.
  - **Connections:** Tool input for Voice AI Agent.
  - **Edge Cases:**  
    - Conflicting events, invalid times, or missing attendee emails.  
    - Permissions and quota issues.
  - **Notes:** Sticky Note #3 applies here as well.

- **Update an event in Google Calendar**
  - **Type:** Google Calendar Tool node
  - **Role:** Updates existing appointment details (e.g., time changes).
  - **Configuration:**  
    - Requires event ID provided by AI agent.  
    - Calendar ID must be set.  
    - Uses default reminders.
  - **Credentials:** Google OAuth2 credentials.
  - **Connections:** Tool input for Voice AI Agent.
  - **Edge Cases:**  
    - Invalid event ID or permission denials.  
    - Concurrent updates causing conflicts.
  - **Notes:** Same calendar setup notes apply.

---

### 2.5 Response Delivery

**Overview:**  
After AI processing, the workflow sends the generated voice response back to ElevenLabs via the webhook response node, enabling real-time interaction.

**Nodes Involved:**  
- Webhook: Return AI Response (ElevenLabs)

**Node Details:**

- **Webhook: Return AI Response (ElevenLabs)**
  - **Type:** Respond to Webhook node
  - **Role:** Sends the AI response payload back as the HTTP response to ElevenLabs.
  - **Configuration:** Default response options.
  - **Connections:** Input from Voice AI Agent node.
  - **Edge Cases:**  
    - Failure to respond or network issues will cause timeout on caller side.  
    - Incorrect response format may break ElevenLabs integration.
  - **Notes:** Sticky Note #2 reminds to test webhook integration carefully.

---

### 2.6 Configuration and Documentation

**Overview:**  
This block consists of sticky notes providing critical setup instructions and best practices for prompt customization, webhook setup, calendar integration, Redis memory, and AI model selection.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1  
- Sticky Note2  
- Sticky Note3  
- Sticky Note4

**Node Details:**

- Sticky Note (Prompt Configuration)
  - Details all prompt variables to customize such as timezone, appointment duration, business hours, required fields, service type, and notes.
  - Emphasizes preserving dynamic variables (`{{ }}`).
- Sticky Note1 (AI Model Configuration)
  - Recommends Claude 3.5 Sonnet and warns against removing the think tool.
- Sticky Note2 (Webhook Setup)
  - Instructions for setting webhook path and testing.
- Sticky Note3 (Google Calendar Setup)
  - Calendar ID format, permissions, and OAuth2 credential setup.
- Sticky Note4 (Redis Memory Setup)
  - Importance of Redis memory and configuration details.

---

## 3. Summary Table

| Node Name                          | Node Type                    | Functional Role                       | Input Node(s)                     | Output Node(s)                    | Sticky Note                                                                                   |
|-----------------------------------|------------------------------|-------------------------------------|----------------------------------|----------------------------------|----------------------------------------------------------------------------------------------|
| Webhook: Receive User Request (ElevenLabs) | Webhook                      | Receives user voice requests         | -                                | Voice AI Agent                   | ## WEBHOOK SETUP<br>1. Replace "REPLACE ME" with your endpoint<br>2. Test endpoint properly.  |
| Voice AI Agent                    | LangChain Agent               | Core AI logic and orchestration      | Webhook: Receive User Request, Redis Chat Memory, Anthropic Chat Model, Reasoning Tool, Calendar nodes | Webhook: Return AI Response (ElevenLabs) | ## PROMPT CONFIGURATION REQUIRED<br>Set timezone, business hours, required fields, etc.      |
| Redis Chat Memory                 | Redis Chat Memory             | Maintains session conversation       | -                                | Voice AI Agent                   | ## REDIS MEMORY SETUP<br>Required for performance and context maintenance.                    |
| Anthropic Chat Model             | LangChain Chat Model          | Natural language understanding       | -                                | Voice AI Agent                   | ## AI MODEL CONFIGURATION<br>Recommended Claude 3.5 Sonnet for best results.                  |
| Reasoning Tool (LangChain)       | LangChain Tool (Think)        | Reasoning and state analysis         | -                                | Voice AI Agent                   | ## AI MODEL CONFIGURATION<br>Do not remove this tool; critical for logic flow.               |
| Calendar: Check Availability     | Google Calendar Tool          | Checks calendar availability         | -                                | Voice AI Agent                   | ## GOOGLE CALENDAR SETUP<br>Replace calendar ID and set correct permissions.                  |
| Calendar: Create Appointment     | Google Calendar Tool          | Creates new appointment               | -                                | Voice AI Agent                   | ## GOOGLE CALENDAR SETUP<br>Same as above.                                                   |
| Update an event in Google Calendar | Google Calendar Tool          | Updates existing appointment          | -                                | Voice AI Agent                   | ## GOOGLE CALENDAR SETUP<br>Same as above.                                                   |
| Webhook: Return AI Response (ElevenLabs) | Respond To Webhook            | Sends AI response back to ElevenLabs | Voice AI Agent                   | -                                | ## WEBHOOK SETUP<br>Test response format and delivery.                                       |
| Sticky Note                      | Sticky Note                  | Prompt configuration instructions    | -                                | -                                | See content in section 2.6                                                                   |
| Sticky Note1                     | Sticky Note                  | AI Model selection guidance           | -                                | -                                | See content in section 2.6                                                                   |
| Sticky Note2                     | Sticky Note                  | Webhook setup instructions            | -                                | -                                | See content in section 2.6                                                                   |
| Sticky Note3                     | Sticky Note                  | Google Calendar setup instructions    | -                                | -                                | See content in section 2.6                                                                   |
| Sticky Note4                     | Sticky Note                  | Redis memory setup instructions       | -                                | -                                | See content in section 2.6                                                                   |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node: "Webhook: Receive User Request (ElevenLabs)"**
   - Type: Webhook
   - HTTP Method: POST
   - Path: Set your desired unique webhook path (replace `"REPLACE ME"`).
   - Response Mode: Use "Response Node" mode.
   - Purpose: Receive ElevenLabs JSON payload with session and utterance data.

2. **Create Redis Chat Memory Node: "Redis Chat Memory"**
   - Type: LangChain Redis Chat Memory
   - Session Key: Set expression `{{$json.body.sessionId}}` to use session ID from webhook.
   - Context Window Length: Set to 20 messages (default).
   - Credentials: Configure Redis credentials (host, port, password).
   - Purpose: Store conversation history per session.

3. **Create Anthropic Chat Model Node: "Anthropic Chat Model"**
   - Type: LangChain Chat Model
   - Model: Select "claude-3-5-sonnet-20241022" or equivalent Claude 3.5 model.
   - Credentials: Configure with Anthropic API credentials.
   - Purpose: Provide AI language capabilities.

4. **Create Reasoning Tool Node: "Reasoning Tool (LangChain)"**
   - Type: LangChain Tool (think)
   - No special parameters needed.
   - Purpose: Used by agent for internal reasoning.

5. **Create Google Calendar Nodes:**
   - **Calendar: Check Availability**
     - Type: Google Calendar Tool
     - Operation: Get All Events
     - Parameters: Use dynamic expressions for `timeMin` and `timeMax` (to be supplied by AI agent).
     - Calendar: Set your Google Calendar ID (replace `"REPLACE ME"`).
     - Credentials: Set Google OAuth2 credentials.
   - **Calendar: Create Appointment**
     - Type: Google Calendar Tool
     - Operation: Create Event
     - Parameters: Provide `start`, `end`, `summary`, `attendees` (email), `description`, and enable sending updates.
     - Calendar: Use same calendar ID.
     - Credentials: Same as above.
   - **Update an event in Google Calendar**
     - Type: Google Calendar Tool
     - Operation: Update Event
     - Parameters: Event ID (dynamic), update fields as needed.
     - Calendar: Use same calendar ID.
     - Credentials: Same as above.

6. **Create AI Agent Node: "Voice AI Agent"**
   - Type: LangChain Agent
   - Text Prompt: Use the detailed scheduling assistant prompt as provided.
   - Include dynamic variables for current time, sessionId, utterance, caller phone number, call log.
   - Register tools: Reasoning Tool, Calendar Check Availability, Create Appointment, Update Appointment.
   - Set language model input to Anthropic Chat Model node.
   - Set AI memory input to Redis Chat Memory node.
   - Purpose: Orchestrate conversation, enforce booking logic, and generate responses.

7. **Create Webhook Respond Node: "Webhook: Return AI Response (ElevenLabs)"**
   - Type: Respond to Webhook
   - No special parameters.
   - Connect input from Voice AI Agent node.
   - Purpose: Return AI-generated response to caller via ElevenLabs.

8. **Link Nodes:**
   - Webhook Receive → Voice AI Agent (main input)
   - Redis Memory → Voice AI Agent (ai_memory input)
   - Anthropic Chat Model → Voice AI Agent (ai_languageModel input)
   - Reasoning Tool → Voice AI Agent (ai_tool input)
   - Calendar Check Availability → Voice AI Agent (ai_tool input)
   - Calendar Create Appointment → Voice AI Agent (ai_tool input)
   - Calendar Update Appointment → Voice AI Agent (ai_tool input)
   - Voice AI Agent → Webhook Respond

9. **Add Sticky Notes:**
   - Add notes for prompt configuration, AI model recommendations, webhook setup, Google Calendar setup, and Redis memory setup with content as detailed in section 2.6.

10. **Configure Credentials:**
    - Anthropic API credentials for Claude model.
    - Redis credentials for chat memory.
    - Google OAuth2 credentials with calendar access permissions.

11. **Test Workflow:**
    - Test webhook endpoint with ElevenLabs webhook tester.
    - Verify Redis connectivity and session persistence.
    - Validate Google Calendar integration with sample availability checks, event creation, and updates.
    - Simulate voice requests and confirm AI responses.

---

## 5. General Notes & Resources

| Note Content                                                                                                                   | Context or Link                                   |
|-------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------|
| The workflow depends critically on Redis memory for conversational context; without it, real-time voice interaction suffers.  | Sticky Note4 - Redis Memory Setup                 |
| Claude 3.5 Sonnet is the recommended AI model for best reasoning and conversational flow in voice applications.              | Sticky Note1 - AI Model Configuration            |
| Webhook path must be replaced with a unique endpoint and tested with ElevenLabs before live deployment.                        | Sticky Note2 - Webhook Setup                       |
| Google Calendar integration requires proper calendar ID and OAuth2 credentials with necessary permissions.                    | Sticky Note3 - Google Calendar Setup              |
| Prompt variables such as timezone, appointment duration, business hours, and required fields must be customized before use.   | Sticky Note - Prompt Configuration                 |
| The voice assistant enforces critical rules: no booking without confirmation, no overlapping appointments, and full data collection. | Voice AI Agent prompt and booking flow rules      |
| For email addresses and phone numbers, the AI spells out characters and digits with hyphens to improve voice clarity.          | Voice AI Agent prompt (Voice Rules section)       |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---