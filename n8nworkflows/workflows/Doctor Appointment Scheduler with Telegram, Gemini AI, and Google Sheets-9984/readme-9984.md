Doctor Appointment Scheduler with Telegram, Gemini AI, and Google Sheets

https://n8nworkflows.xyz/workflows/doctor-appointment-scheduler-with-telegram--gemini-ai--and-google-sheets-9984


# Doctor Appointment Scheduler with Telegram, Gemini AI, and Google Sheets

---
### 1. Workflow Overview

This workflow automates doctor appointment scheduling via Telegram chat, leveraging AI (Google Gemini and LangChain AI Agent) and Google Sheets for doctor information and booking records. It supports both text and voice inputs, processes user queries about doctor availability, and facilitates appointment booking by interacting with dedicated Google Sheets for each doctor.

**Target use cases:**  
- Patients requesting doctor information and appointments through Telegram messages (text or voice).  
- Hospital assistants automating appointment scheduling without manual intervention.  

**Logical blocks:**  
- **1.1 Input Reception and Preprocessing:** Captures Telegram messages (text/voice), distinguishes input type, and transcribes voice messages.  
- **1.2 AI Processing & Memory:** Uses Google Gemini Chat Model and LangChain AI Agent to understand user requests, maintain session context, and generate responses based on doctor data.  
- **1.3 Doctor Information Lookup:** Queries Google Sheets storing doctor info and individual doctor schedules.  
- **1.4 Appointment Booking:** Updates the relevant doctor's Google Sheet to append new appointment records after confirmation.  
- **1.5 User Interaction:** Sends messages and chat actions back to the user on Telegram.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Preprocessing

- **Overview:**  
  Receives incoming Telegram messages, determines if input is text or voice, transcribes voice messages to text for further processing.

- **Nodes Involved:**  
  - Telegram Trigger  
  - Switch  
  - Send a chat action  
  - Get a file  
  - Transcribe a recording  
  - Edit Fields (for transcribed text)  
  - Edit Fields1 (for direct text input)

- **Node Details:**

  - **Telegram Trigger**  
    - Type: Telegram Trigger Node  
    - Role: Entry point, listens for incoming Telegram messages (only "message" updates).  
    - Configuration: Linked to Telegram bot credentials; triggers workflow on new messages.  
    - Inputs: Telegram updates  
    - Outputs: Raw Telegram message JSON  
    - Failures: Telegram API downtime, webhook misconfiguration.

  - **Switch**  
    - Type: Switch Node  
    - Role: Routes flow based on message content type: text or voice.  
    - Configuration: Checks if `$json.message.text` exists for text; `$json.message.voice` exists for voice.  
    - Inputs: Telegram Trigger output  
    - Outputs: Two outputs: "Text" (text messages), "Voice" (voice messages)  
    - Edge cases: Messages without text or voice (ignored); malformed message JSON.

  - **Send a chat action**  
    - Type: Telegram Node (sendChatAction)  
    - Role: Sends "typing" status to user to indicate processing.  
    - Config: Uses chat ID from Telegram message.  
    - Inputs: Switch node (for text path)  
    - Outputs: None (side-effect node)  
    - Failures: Telegram API errors.

  - **Get a file**  
    - Type: Telegram Node (file resource)  
    - Role: Retrieves voice message file using file_id from Telegram message.  
    - Config: Uses voice file_id from message JSON.  
    - Inputs: Switch node (voice path)  
    - Outputs: Binary audio file data  
    - Failures: File not found, Telegram file API errors.

  - **Transcribe a recording**  
    - Type: Google Gemini (audio transcription)  
    - Role: Converts voice audio to text using Google Gemini Speech-to-Text model.  
    - Config: Model "models/gemini-2.5-flash", input type binary.  
    - Inputs: Get a file (binary audio)  
    - Outputs: Transcribed text in JSON candidates array  
    - Failures: API limits, audio quality issues.

  - **Edit Fields**  
    - Type: Set Node  
    - Role: Extracts transcribed text from Gemini response, sets `text` field for AI processing.  
    - Inputs: Transcribe a recording output  
    - Outputs: JSON with `text` field  
    - Edge cases: Empty or null transcription.

  - **Edit Fields1**  
    - Type: Set Node  
    - Role: For direct text messages, sets `text` field from Telegram message text for AI input.  
    - Inputs: Switch output (Text)  
    - Outputs: JSON with `text` field

---

#### 1.2 AI Processing & Memory

- **Overview:**  
  Processes user input text with an AI agent that references doctor info and appointment data, maintaining session memory to handle multi-turn conversations.

- **Nodes Involved:**  
  - Simple Memory  
  - Google Gemini Chat Model  
  - Doctor_Info (Google Sheets read)  
  - Dr Karan Singh (Google Sheets read)  
  - Dr Arjun Mehta (Google Sheets read)  
  - AI Agent  
  - Dr Karan Singh Append Row (Google Sheets append)  
  - Dr Arjun Mehta Append Row (Google Sheets append)

- **Node Details:**

  - **Simple Memory**  
    - Type: LangChain MemoryBufferWindow  
    - Role: Maintains conversational session memory keyed by Telegram chat ID, enabling context in AI interactions.  
    - Config: Session key is Telegram chat ID from incoming message.  
    - Inputs: None (used internally by AI Agent)  
    - Outputs: Provides memory context to AI Agent.

  - **Google Gemini Chat Model**  
    - Type: LangChain LM Chat Google Gemini  
    - Role: Language model for chat completions, used internally by AI Agent.  
    - Config: Uses Google Palm API credentials.  
    - Inputs: Text input from Edit Fields or Edit Fields1, plus memory context.  
    - Outputs: AI-generated response content.  
    - Failures: API quota limits, network errors.

  - **Doctor_Info**  
    - Type: Google Sheets Tool (read)  
    - Role: Reads doctor info sheet containing doctor name, department, fees.  
    - Config: Reads from specific Google Sheet and tab "Doctor_Info".  
    - Inputs: None (used as AI tool input)  
    - Outputs: Doctor info data for AI Agent reference.  
    - Failures: Google Sheets API auth errors, sheet missing.

  - **Dr Karan Singh** and **Dr Arjun Mehta**  
    - Type: Google Sheets Tool (read)  
    - Role: Reads individual doctor schedules to check booked appointments.  
    - Config: Each reads from respective doctor's sheet tab.  
    - Inputs: None (used as AI tool input)  
    - Outputs: Doctor schedule data for AI Agent.  
    - Failures: Sheet access issues.

  - **AI Agent**  
    - Type: LangChain Agent Node  
    - Role: Core logic that interprets user requests, cross-references doctor info and schedules, confirms availability, and handles booking logic.  
    - Config:  
      - System message defines assistant behavior: polite, references only sheet data, handles booking confirmations, appends bookings.  
      - Inputs: Text from Edit Fields/ Edit Fields1, memory context, doctor info sheets, Google Gemini Chat model.  
      - Outputs: Text response to user, also triggers append row nodes for booking.  
    - Edge cases: Missing doctor info, unavailable appointment slot, incomplete user data, AI prompt failures.

  - **Dr Karan Singh Append Row** and **Dr Arjun Mehta Append Row**  
    - Type: Google Sheets Tool (append)  
    - Role: After booking confirmation, appends new appointment row to respective doctor's schedule sheet.  
    - Config: Maps fields: Doctor Name, Patient Name, Booked Date, Booked Time from AI Agent output variables.  
    - Inputs: AI Agent node (ai_tool output)  
    - Outputs: Confirmation of append operation  
    - Failures: Sheet write permission errors, data mapping issues.

---

#### 1.3 User Interaction & Messaging

- **Overview:**  
  Sends AI-generated text responses back to the user on Telegram, providing confirmations, info, or error messages.

- **Nodes Involved:**  
  - Send a text message

- **Node Details:**

  - **Send a text message**  
    - Type: Telegram Node (sendMessage)  
    - Role: Sends AI Agent’s output text back to Telegram user.  
    - Config:  
      - Text dynamically set from AI Agent's output JSON field `output`.  
      - Chat ID extracted from Telegram Trigger message.  
      - Attribution disabled for cleaner messages.  
    - Inputs: AI Agent main output  
    - Outputs: None (end node)  
    - Failures: Telegram API errors, invalid chat ID.

---

#### Additional Notes on Sticky Notes

- Sticky Note near workflow start explains the overall workflow purpose emphasizing appointment scheduling logic, polite AI responses, and sheet integration.  
- Sticky Note near Google Sheets nodes documents schema of doctor info and appointment sheets:  
  - Doctor_Info Tool Schema: Doctor name, Department, Fees  
  - Doctor Scheduler Schema: Doctor Name, Patient Name, Booked Date, Booked Time

---

### 3. Summary Table

| Node Name                 | Node Type                                | Functional Role                              | Input Node(s)                    | Output Node(s)                           | Sticky Note                                                                                                         |
|---------------------------|-----------------------------------------|----------------------------------------------|---------------------------------|-----------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| Telegram Trigger           | Telegram Trigger                        | Entry point for Telegram messages             | —                               | Switch, Send a chat action               |                                                                                                                     |
| Switch                    | Switch                                 | Routes between text and voice message paths  | Telegram Trigger                | Edit Fields1 (text), Get a file (voice) |                                                                                                                     |
| Send a chat action         | Telegram                               | Sends "typing" action to Telegram user        | Switch (Text path)              | —                                       |                                                                                                                     |
| Get a file                 | Telegram                               | Fetches voice message audio file               | Switch (Voice path)             | Transcribe a recording                   |                                                                                                                     |
| Transcribe a recording     | Google Gemini                          | Transcribes voice audio to text                | Get a file                     | Edit Fields                             |                                                                                                                     |
| Edit Fields                | Set                                   | Extracts text from transcription               | Transcribe a recording          | AI Agent                               |                                                                                                                     |
| Edit Fields1               | Set                                   | Extracts text from Telegram text message       | Switch (Text path)              | AI Agent                               |                                                                                                                     |
| Simple Memory              | LangChain MemoryBufferWindow           | Maintains conversational context               | —                              | AI Agent                               |                                                                                                                     |
| Google Gemini Chat Model   | LangChain LM Chat Google Gemini        | Provides language model completions             | AI Agent                       | AI Agent                               |                                                                                                                     |
| Doctor_Info                | Google Sheets Tool (read)               | Provides doctor info data                        | —                              | AI Agent (ai_tool)                     | See Sticky Note1: Doctor_Info Tool Schema: Doctor name, Department, Fees; Doctor Scheduler Schema: Doctor Name, etc. |
| Dr Karan Singh             | Google Sheets Tool (read)               | Provides schedule data for Dr. Karan Singh     | —                              | AI Agent (ai_tool)                     |                                                                                                                     |
| Dr Arjun Mehta             | Google Sheets Tool (read)               | Provides schedule data for Dr. Arjun Mehta     | —                              | AI Agent (ai_tool)                     |                                                                                                                     |
| AI Agent                  | LangChain Agent                        | Core logic for interpreting requests and booking | Edit Fields, Edit Fields1, Simple Memory, Doctor Sheets, Gemini LM | Send a text message, Append Row nodes | See Sticky Note: Doctor’s Appointment Scheduler Agent explanation                                                   |
| Dr Karan Singh Append Row  | Google Sheets Tool (append)             | Appends new appointment for Dr. Karan Singh    | AI Agent (ai_tool)             | AI Agent                               |                                                                                                                     |
| Dr Arjun Mehta Append Row  | Google Sheets Tool (append)             | Appends new appointment for Dr. Arjun Mehta    | AI Agent (ai_tool)             | AI Agent                               |                                                                                                                     |
| Send a text message        | Telegram                               | Sends AI response back to Telegram user        | AI Agent                      | —                                       |                                                                                                                     |
| Sticky Note                | Sticky Note                           | Provides workflow overview and logic details   | —                              | —                                       | "Doctor’s Appointment Scheduler Agent" explanation                                                                 |
| Sticky Note1               | Sticky Note                           | Documents Google Sheets schema                   | —                              | —                                       | "Doctor_Info Tool Schema" and "Doctor Scheduler Schema"                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Set to trigger on "message" updates only.  
   - Connect your Telegram bot credentials.  
   - This node receives incoming messages (text or voice).

2. **Add Switch Node**  
   - Type: Switch  
   - Configure two outputs:  
     - Output "Text": condition checks if `$json.message.text` exists.  
     - Output "Voice": condition checks if `$json.message.voice` exists.  
   - Connect Telegram Trigger to Switch node.

3. **Add Send a Chat Action Node**  
   - Type: Telegram (sendChatAction)  
   - Chat ID: `{{$json.message.chat.id}}`  
   - Operation: "sendChatAction" (e.g., typing)  
   - Connect Switch Text output to this node.

4. **Add Edit Fields1 Node for Text Messages**  
   - Type: Set  
   - Add field `text` with value `{{$json.message.text}}`  
   - Connect Switch Text output to this node.

5. **Add Get a File Node for Voice Messages**  
   - Type: Telegram (file resource)  
   - File ID: `{{$json.message.voice.file_id}}`  
   - Connect Switch Voice output to this node.

6. **Add Transcribe a Recording Node**  
   - Type: LangChain Google Gemini (audio transcription)  
   - Model ID: `models/gemini-2.5-flash`  
   - Input Type: binary (audio file)  
   - Connect Get a File output to this node.

7. **Add Edit Fields Node for Transcription**  
   - Type: Set  
   - Add field `text` with value `{{$json.candidates[0].content.parts[0].text}}` (extract first transcription text)  
   - Connect Transcribe a Recording output to this node.

8. **Add Simple Memory Node**  
   - Type: LangChain MemoryBufferWindow  
   - Session key: `{{$('Telegram Trigger').item.json.message.chat.id}}` (to keep session per user)  
   - No inputs; this node is referenced by AI Agent.

9. **Add Google Gemini Chat Model Node**  
   - Type: LangChain LM Chat Google Gemini  
   - Connect to AI Agent (used internally)  
   - Configure with Google Palm API credentials.

10. **Add Google Sheets Tool Nodes for Doctor Data**  
    - **Doctor_Info:** Read mode from Google Sheet tab containing doctor info (name, department, fees).  
    - **Dr Karan Singh:** Read mode from Google Sheet tab with Dr. Karan Singh's appointments.  
    - **Dr Arjun Mehta:** Read mode from Google Sheet tab with Dr. Arjun Mehta's appointments.  
    - Use Google Sheets OAuth2 credentials.  
    - These nodes are linked as AI tools for the AI Agent.

11. **Add AI Agent Node**  
    - Type: LangChain Agent  
    - Configure with system prompt directing polite assistant behavior, referencing doctor info sheets, booking logic, and appointment confirmations.  
    - Inputs: `text` field from Edit Fields or Edit Fields1 node, Simple Memory node (ai_memory), Doctor_Info and individual doctor sheets (ai_tool), Google Gemini Chat Model (ai_languageModel).  
    - Outputs: text response and triggers append row nodes.

12. **Add Google Sheets Append Row Nodes**  
    - **Dr Karan Singh Append Row:** Append new appointment row to Dr. Karan Singh sheet.  
    - **Dr Arjun Mehta Append Row:** Append new appointment row to Dr. Arjun Mehta sheet.  
    - Map columns: Doctor Name, Patient Name, Booked Date, Booked Time from AI Agent output variables.  
    - Use same OAuth2 credentials.  
    - Connect AI Agent’s ai_tool output to these append nodes.

13. **Add Send a Text Message Node**  
    - Type: Telegram (sendMessage)  
    - Text: Use AI Agent output JSON field `output`.  
    - Chat ID: `{{$('Telegram Trigger').item.json.message.chat.id}}`  
    - Connect AI Agent main output to this node.

14. **Connect all nodes as per above flow:**  
    - Telegram Trigger → Switch  
    - Switch Text → Edit Fields1 → AI Agent  
    - Switch Text → Send a chat action (parallel)  
    - Switch Voice → Get a file → Transcribe a recording → Edit Fields → AI Agent  
    - AI Agent → Send a text message  
    - AI Agent → Append Row nodes for doctor schedules

15. **Credentials Setup:**  
    - Telegram API credentials for Telegram nodes.  
    - Google Palm API credentials for Gemini nodes.  
    - Google Sheets OAuth2 credentials for reading and appending sheets.

---

### 5. General Notes & Resources

| Note Content                                                                                                                          | Context or Link                                                                                          |
|----------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| The workflow uses Google Gemini (PaLM) AI model for both text generation and speech-to-text transcription.                             | Google Gemini API (Google Palm API) credentials required.                                               |
| Doctor information and appointment bookings are stored and managed via separate Google Sheets tabs for modularity and clarity.         | Google Sheets document: https://docs.google.com/spreadsheets/d/16z9zoORxtT1PzydJSKsF5Szlu9x7QewX-5UBITjn_Kg |
| The AI Agent system prompt strictly instructs to avoid hallucination, only respond based on sheet data, and confirm bookings politely. | See system message in AI Agent node parameters.                                                         |
| Voice messages are transcribed to text to unify processing via AI Agent.                                                                | Voice transcription uses Gemini model "models/gemini-2.5-flash".                                        |
| The “Send a chat action” node improves user experience by showing typing status during AI processing.                                  | Telegram feature for better UX.                                                                          |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.