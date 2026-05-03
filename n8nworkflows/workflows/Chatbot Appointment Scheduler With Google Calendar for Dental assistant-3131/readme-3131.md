Chatbot Appointment Scheduler With Google Calendar for Dental assistant

https://n8nworkflows.xyz/workflows/chatbot-appointment-scheduler-with-google-calendar-for-dental-assistant-3131


# Chatbot Appointment Scheduler With Google Calendar for Dental assistant

### 1. Workflow Overview

This workflow is a **Chatbot Appointment Scheduler** designed specifically for dental assistants or similar medical office staff to automate appointment booking. It integrates **Google Calendar** to check and book appointment slots and **Google Sheets** to log patient information for record-keeping.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures incoming chat messages from users requesting appointments.
- **1.2 AI Processing:** Uses an AI agent (powered by OpenAI GPT-4o-mini) to interpret user requests, manage dialogue, and orchestrate booking logic.
- **1.3 Memory Management:** Maintains session context to handle multi-turn conversations.
- **1.4 Availability Checking:** Queries Google Calendar to verify if requested appointment times are free.
- **1.5 Appointment Booking:** Creates calendar events for confirmed appointments.
- **1.6 Data Logging:** Records appointment details into Google Sheets for tracking and future reference.
- **1.7 User Guidance:** Sticky notes provide setup instructions and contextual help for users configuring the workflow.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for incoming chat messages from users to initiate the appointment scheduling process.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**  
  - **When chat message received**  
    - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
    - Role: Entry point webhook node that triggers the workflow when a chat message is received.  
    - Configuration: Default options, webhook ID assigned for external chatbot integration.  
    - Inputs: External chat messages via webhook.  
    - Outputs: Passes message data to the AI Agent node.  
    - Edge Cases: Webhook connectivity issues, malformed messages, or missing session IDs could disrupt flow.

#### 2.2 AI Processing

- **Overview:**  
  This block processes user input using an AI agent that manages the conversation, checks availability, confirms bookings, and instructs downstream nodes.

- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model  
  - Window Buffer Memory

- **Node Details:**  
  - **AI Agent**  
    - Type: `@n8n/n8n-nodes-langchain.agent`  
    - Role: Central conversational AI orchestrator that interprets user requests and controls workflow logic.  
    - Configuration:  
      - System prompt defines the assistant’s role as a dental appointment scheduler with office hours, booking rules, and interaction guidelines.  
      - Uses sub-nodes/tools: "Check Availability", "Create event", and "Add Data" for calendar and sheet operations.  
      - Does not confirm booking without explicit user confirmation.  
    - Inputs: Chat messages from "When chat message received" and session memory.  
    - Outputs: Commands to check availability, create events, and add data.  
    - Edge Cases: AI misinterpretation, prompt failures, API rate limits, or unexpected user inputs.  
    - Version: 1.7  

  - **OpenAI Chat Model**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
    - Role: Provides GPT-4o-mini language model responses for the AI Agent.  
    - Configuration: Model set to "gpt-4o-mini", linked to OpenAI API credentials.  
    - Inputs: Prompts from AI Agent.  
    - Outputs: Text responses back to AI Agent.  
    - Edge Cases: API key errors, network timeouts, or model unavailability.  
    - Version: 1.2  

  - **Window Buffer Memory**  
    - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
    - Role: Maintains conversational context per user session using session ID from incoming chat.  
    - Configuration: Session key derived from the chat message’s sessionId field.  
    - Inputs: Session ID and conversation data.  
    - Outputs: Contextual memory to AI Agent.  
    - Edge Cases: Missing or invalid session IDs, memory overflow.  
    - Version: 1.3  

#### 2.3 Availability Checking

- **Overview:**  
  This block queries Google Calendar to verify if the requested appointment slot is available.

- **Nodes Involved:**  
  - Check Availability

- **Node Details:**  
  - **Check Availability**  
    - Type: `googleCalendarTool`  
    - Role: Checks calendar events between requested start and end times to determine availability.  
    - Configuration:  
      - Calendar selected is the dental office’s Google Calendar.  
      - Time range dynamically set from AI Agent outputs (`Start_Time` and `End_Time`).  
      - Uses OAuth2 credentials for Google Calendar access.  
    - Inputs: Time window from AI Agent.  
    - Outputs: Availability status back to AI Agent.  
    - Edge Cases: OAuth token expiration, API quota limits, invalid time formats.  
    - Version: 1.3  

#### 2.4 Appointment Booking

- **Overview:**  
  This block creates a new event in Google Calendar once the appointment is confirmed by the patient.

- **Nodes Involved:**  
  - Creat event

- **Node Details:**  
  - **Creat event**  
    - Type: `googleCalendarTool`  
    - Role: Creates a calendar event with patient name and phone number as the event title.  
    - Configuration:  
      - Calendar selected is the same dental office calendar.  
      - Start and end times dynamically set from AI Agent outputs (`Start` and `End`).  
      - OAuth2 credentials used for authentication.  
    - Inputs: Confirmed appointment details from AI Agent.  
    - Outputs: Confirmation of event creation.  
    - Edge Cases: Conflicts with existing events, invalid date/time, OAuth errors.  
    - Version: 1.3  

#### 2.5 Data Logging

- **Overview:**  
  This block appends patient and appointment details to a Google Sheets document for record-keeping.

- **Nodes Involved:**  
  - Add data

- **Node Details:**  
  - **Add data**  
    - Type: `googleSheetsTool`  
    - Role: Appends a new row with patient full name, phone number, and appointment date/time.  
    - Configuration:  
      - Target Google Sheets document and sheet specified.  
      - Columns mapped explicitly: "Nom complet", "Numéro de téléphone", "Date / heure".  
      - Uses OAuth2 credentials for Google Sheets access.  
    - Inputs: Patient and appointment data from AI Agent.  
    - Outputs: Confirmation of data append operation.  
    - Edge Cases: Sheet access permissions, quota limits, malformed data.  
    - Version: 4.5  

#### 2.6 User Guidance (Sticky Notes)

- **Overview:**  
  Sticky notes provide setup instructions and contextual help for users configuring the workflow.

- **Nodes Involved:**  
  - Sticky Note, Sticky Note1, Sticky Note2, Sticky Note3, Sticky Note4

- **Node Details:**  
  - Sticky notes contain instructions such as:  
    - Reminder to add OpenAI API key.  
    - Connect Google Calendar and Google Sheets credentials.  
    - Setup session ID for memory node.  
    - Guidance on AI Agent prompt usage.  
    - Link to setup video: https://youtu.be/FKCmGhdP8oE  
  - These nodes have no inputs or outputs and serve purely as documentation aids.

---

### 3. Summary Table

| Node Name                 | Node Type                                | Functional Role                  | Input Node(s)               | Output Node(s)              | Sticky Note                                                                                  |
|---------------------------|-----------------------------------------|--------------------------------|-----------------------------|-----------------------------|----------------------------------------------------------------------------------------------|
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger    | Entry point for chat messages   | External webhook             | AI Agent                    |                                                                                              |
| AI Agent                  | @n8n/n8n-nodes-langchain.agent           | Conversational AI orchestrator  | When chat message received, Window Buffer Memory, OpenAI Chat Model | Check Availability, Creat event, Add data | **AI Agent 👇** The Prompt is already there, You just need to setup the prompt user message with your text message. |
| OpenAI Chat Model          | @n8n/n8n-nodes-langchain.lmChatOpenAi   | Language model for AI Agent     | AI Agent                    | AI Agent                    | **Chat Model ☝️** Add your Open Ai API Key                                                  |
| Window Buffer Memory       | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintains session context       | AI Agent                    | AI Agent                    | **Memory ☝️** Add the Session ID                                                            |
| Check Availability         | googleCalendarTool                      | Checks appointment availability | AI Agent                    | AI Agent                    | **Gpoogle Calendar ☝️** Connect to Google Calendar                                          |
| Creat event                | googleCalendarTool                      | Books appointment in calendar   | AI Agent                    | AI Agent                    | **Gpoogle Calendar ☝️** Connect to Google Calendar                                          |
| Add data                   | googleSheetsTool                       | Logs appointment data           | AI Agent                    |                             | **Google Sheets ☝️** Connect to Google Sheets                                               |
| Sticky Note               | n8n-nodes-base.stickyNote                | Setup instructions              | None                       | None                       | See individual sticky note contents                                                        |
| Sticky Note1              | n8n-nodes-base.stickyNote                | Setup instructions              | None                       | None                       | See individual sticky note contents                                                        |
| Sticky Note2              | n8n-nodes-base.stickyNote                | Setup instructions              | None                       | None                       | See individual sticky note contents                                                        |
| Sticky Note3              | n8n-nodes-base.stickyNote                | Setup instructions              | None                       | None                       | See individual sticky note contents                                                        |
| Sticky Note4              | n8n-nodes-base.stickyNote                | Setup instructions              | None                       | None                       | See individual sticky note contents                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node**  
   - Add a **When chat message received** node (`@n8n/n8n-nodes-langchain.chatTrigger`).  
   - Configure webhook with a unique ID to receive chat messages.  
   - Leave default options.

2. **Add AI Agent Node**  
   - Add an **AI Agent** node (`@n8n/n8n-nodes-langchain.agent`).  
   - Configure the system prompt with the detailed assistant role, office hours, booking rules, and instructions as per the provided prompt text.  
   - Set tools for "Check Availability", "Create event", and "Add Data" to be called by this agent.  
   - Connect input from "When chat message received".  

3. **Add OpenAI Chat Model Node**  
   - Add **OpenAI Chat Model** node (`@n8n/n8n-nodes-langchain.lmChatOpenAi`).  
   - Set model to `gpt-4o-mini`.  
   - Link OpenAI API credentials (OpenAI API Key).  
   - Connect output to AI Agent’s language model input.

4. **Add Window Buffer Memory Node**  
   - Add **Window Buffer Memory** node (`@n8n/n8n-nodes-langchain.memoryBufferWindow`).  
   - Configure session key as `={{ $('When chat message received').item.json.sessionId }}` to maintain per-user conversation context.  
   - Connect input and output to AI Agent’s memory interface.

5. **Add Check Availability Node**  
   - Add **Google Calendar Tool** node named "Check Availability".  
   - Select the dental office Google Calendar.  
   - Configure `timeMin` and `timeMax` parameters to be dynamically set from AI Agent outputs (`Start_Time` and `End_Time`).  
   - Connect input from AI Agent’s tool output.  
   - Use Google Calendar OAuth2 credentials.

6. **Add Create Event Node**  
   - Add **Google Calendar Tool** node named "Creat event".  
   - Select the same Google Calendar.  
   - Configure `start` and `end` parameters dynamically from AI Agent outputs (`Start` and `End`).  
   - Connect input from AI Agent’s tool output.  
   - Use Google Calendar OAuth2 credentials.

7. **Add Add Data Node**  
   - Add **Google Sheets Tool** node named "Add data".  
   - Select the Google Sheets document and sheet for logging appointments.  
   - Map columns:  
     - "Nom complet" → from AI Agent output `Nom_complet`  
     - "Numéro de téléphone" → from AI Agent output `Num_ro_de_telephone`  
     - "Date / heure" → from AI Agent output `Date___heure_`  
   - Set operation to "append".  
   - Connect input from AI Agent’s tool output.  
   - Use Google Sheets OAuth2 credentials.

8. **Connect Nodes**  
   - Connect "When chat message received" → "AI Agent" (main).  
   - Connect "AI Agent" → "OpenAI Chat Model" (ai_languageModel).  
   - Connect "AI Agent" → "Window Buffer Memory" (ai_memory).  
   - Connect "AI Agent" → "Check Availability" (ai_tool).  
   - Connect "AI Agent" → "Creat event" (ai_tool).  
   - Connect "AI Agent" → "Add data" (ai_tool).

9. **Add Sticky Notes for Guidance**  
   - Add sticky notes near relevant nodes with instructions:  
     - Add OpenAI API key near OpenAI Chat Model node.  
     - Connect Google Calendar credentials near calendar nodes.  
     - Connect Google Sheets credentials near Add data node.  
     - Add session ID note near Window Buffer Memory.  
     - Add AI Agent prompt usage note near AI Agent node.  
     - Include link to setup video: https://youtu.be/FKCmGhdP8oE.

10. **Test the Workflow**  
    - Run test chat messages simulating appointment requests.  
    - Verify AI responses, calendar availability checks, event creation, and data logging.  
    - Adjust prompt or credentials as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                |
|----------------------------------------------------------------------------------------------------------------------|------------------------------------------------|
| Setup video explaining how to run and customize the workflow                                                        | https://youtu.be/FKCmGhdP8oE                    |
| Sample Google Sheets template provided for patient data logging                                                     | Refer to attached image "Google sheets TMP.JPG"|
| Office hours and booking rules are embedded in the AI Agent system prompt                                           | See AI Agent systemMessage parameter            |
| Ensure Google OAuth2 credentials have appropriate scopes for Calendar and Sheets access                             | Google Calendar and Sheets OAuth2 setup          |
| The AI Agent uses sub-tools named "Check Availability", "Creat event", and "Add Data" to modularize booking steps  | Internal workflow design                          |

---

This documentation provides a complete, structured understanding of the "Chatbot Appointment Scheduler With Google Calendar for Dental assistant" workflow, enabling reproduction, modification, and troubleshooting by advanced users or automation agents.