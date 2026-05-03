WhatsApp Appointment Scheduling with Google Calendar and Gemini AI

https://n8nworkflows.xyz/workflows/whatsapp-appointment-scheduling-with-google-calendar-and-gemini-ai-5855


# WhatsApp Appointment Scheduling with Google Calendar and Gemini AI

### 1. Workflow Overview

This workflow enables WhatsApp appointment scheduling integrated with Google Calendar, augmented by conversational AI powered by Google's Gemini model. It handles WhatsApp inbound messages, interprets user intent through an AI agent, interacts with Google Calendar to create, update, delete, or retrieve calendar events, and sends WhatsApp replies accordingly.

The workflow is logically structured into the following blocks:

- **1.1 Input Reception:** Captures incoming WhatsApp messages or external workflow triggers to initiate processing.
- **1.2 Initialization & Routing:** Sets initial variables and directs flow based on message type and intent.
- **1.3 AI Processing:** Uses LangChain AI Agent with Google Gemini Chat Model and short-term memory to understand and respond to user inputs.
- **1.4 Calendar Operations:** Executes Google Calendar API calls to get, create, update, or delete events based on AI instructions.
- **1.5 Output Delivery:** Sends responses back to WhatsApp users.
  
---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block listens for WhatsApp incoming messages or external workflow triggers to start the appointment scheduling process or other related operations.

**Nodes Involved:**  
- WhatsApp Trigger  
- When chat message received (LangChain Chat Trigger)  
- When Executed by Another Workflow

**Node Details:**

- **WhatsApp Trigger**  
  - Type: WhatsApp inbound webhook trigger  
  - Role: Receives WhatsApp messages, starting point for WhatsApp-originated requests  
  - Configuration: Linked to specific WhatsApp webhookId for channel integration  
  - Input: External WhatsApp message events  
  - Output: Passes message data to next node for variable extraction  
  - Edge cases: Possible webhook downtime or message format inconsistencies

- **When chat message received**  
  - Type: LangChain chat trigger  
  - Role: Alternative trigger for chat messages, possibly for testing or AI-specific input  
  - Configuration: Uses a separate webhookId, connected to the “Edit Fields” node  
  - Input: Incoming chat messages from LangChain environment  
  - Output: Passes data for field adjustments  
  - Edge cases: AI service unavailability, webhook failures

- **When Executed by Another Workflow**  
  - Type: Execute Workflow trigger  
  - Role: Allows this workflow to be triggered programmatically from other workflows  
  - Configuration: No parameters, triggers Initialization node  
  - Input: External workflow execution calls  
  - Output: Starts the Initialization node  
  - Edge cases: Misconfiguration of calling workflows, parameter mismatch

---

#### 2.2 Initialization & Routing

**Overview:**  
Sets initial variables and controls message routing based on whether the message is a start command or an appointment-related request.

**Nodes Involved:**  
- Variables  
- Initialization  
- Is start? (If node)  
- Define Type (Switch node)  
- Edit Fields  
- Welcome message

**Node Details:**

- **Variables**  
  - Type: Set node  
  - Role: Initializes or resets workflow variables before processing incoming input  
  - Configuration: Likely sets placeholders or default values for managing state  
  - Input: From WhatsApp Trigger  
  - Output: Passes to Initialization node  
  - Edge cases: Missing or invalid initial values may cause downstream errors

- **Initialization**  
  - Type: Set node  
  - Role: Further preparation of variables or context  
  - Configuration: Could define flags or temporary values for routing  
  - Input: From Variables or Execute Workflow trigger  
  - Output: Passes to “Is start?” decision node  
  - Edge cases: Failure to properly set flags could misroute messages

- **Is start?**  
  - Type: If node  
  - Role: Determines if the incoming message is a “start” command (e.g., greetings or session initiation)  
  - Configuration: Checks a condition on the message content or variable  
  - Input: From Initialization  
  - Output: If true → Welcome message; if false → Define Type  
  - Edge cases: Incorrect condition evaluation may misclassify message type

- **Welcome message**  
  - Type: WhatsApp node  
  - Role: Sends a predefined welcome or greeting message to the user  
  - Configuration: Contains the welcome text and WhatsApp webhookId  
  - Input: From Is start? (true branch)  
  - Output: Ends flow for start command  
  - Edge cases: WhatsApp API errors or message delivery failures

- **Define Type**  
  - Type: Switch node  
  - Role: Determines the type of user request (e.g., create event, update event, delete event) based on message content or AI agent output  
  - Configuration: Switches over various conditions mapped to AI intents or keywords  
  - Input: From Is start? (false branch)  
  - Output: If matched → AI Agent; else no output (empty branch)  
  - Edge cases: Ambiguous user input causing no matching case, leading to no action

- **Edit Fields**  
  - Type: Set node  
  - Role: Adjusts or normalizes fields from chat message for AI processing  
  - Configuration: Maps or transforms raw input data to structured format  
  - Input: From “When chat message received”  
  - Output: To Initialization node (for further processing)  
  - Edge cases: Missing fields or unexpected input formats

---

#### 2.3 AI Processing

**Overview:**  
Processes user input with an AI agent to understand intent and generate appropriate calendar-related commands, utilizing memory and the Google Gemini chat model.

**Nodes Involved:**  
- AI Agent (LangChain Agent)  
- Google Gemini Chat Model  
- Simple Memory

**Node Details:**

- **AI Agent**  
  - Type: LangChain AI Agent node  
  - Role: Core AI interpreter that receives processed input, consults memory, and decides on calendar actions  
  - Configuration: No explicit parameters shown, connected to AI Language Model, AI Memory, and AI Tools (Google Calendar nodes)  
  - Inputs: Receives from Define Type node (user intent), Simple Memory (context), Google Gemini Chat Model (language model), and Google Calendar tool nodes for action execution  
  - Outputs: Sends structured responses to “Send Answer” node  
  - Edge cases: AI service downtime, timeout, or unclear user instructions causing failure to generate valid commands

- **Google Gemini Chat Model**  
  - Type: LangChain Language Model node for Google Gemini  
  - Role: Provides natural language understanding and generation capabilities for the AI Agent  
  - Configuration: No parameters shown; presumably uses configured API credentials for Google Gemini  
  - Inputs: Connected as AI language model to AI Agent  
  - Edge cases: API limits, authentication errors, response latency

- **Simple Memory**  
  - Type: LangChain Memory Buffer Window  
  - Role: Maintains conversational context with a sliding window of recent exchanges to support coherent AI responses  
  - Configuration: Default memory window size, no custom parameters shown  
  - Inputs: Feeds AI Agent with memory context  
  - Edge cases: Memory overflow or stale context impacting AI accuracy

---

#### 2.4 Calendar Operations

**Overview:**  
Performs operations on Google Calendar such as fetching, creating, updating, or deleting events based on AI agent instructions.

**Nodes Involved:**  
- Get Calendar Event  
- Create Calendar Event  
- Update Calendar Event  
- Delete Calendar Event

**Node Details:**

- **Get Calendar Event**  
  - Type: Google Calendar Tool node  
  - Role: Retrieves event details from Google Calendar to confirm availability or existing appointments  
  - Configuration: Uses OAuth2 credentials for Google Calendar; parameters dynamically set by AI Agent  
  - Inputs: Connected as AI tool to AI Agent  
  - Outputs: Event data passed back to AI Agent for decision-making  
  - Edge cases: Authentication failures, permission issues, event not found

- **Create Calendar Event**  
  - Type: Google Calendar Tool node  
  - Role: Creates a new event based on user request  
  - Configuration: Parameters such as date, time, description provided dynamically by AI Agent  
  - Inputs: Connected as AI tool to AI Agent  
  - Outputs: Confirmation data to AI Agent  
  - Edge cases: Conflicting events, invalid datetime formats, quota limits

- **Update Calendar Event**  
  - Type: Google Calendar Tool node  
  - Role: Modifies existing events as per user updates  
  - Configuration: Event ID and new details passed by AI Agent  
  - Inputs: Connected as AI tool to AI Agent  
  - Outputs: Updated event confirmation  
  - Edge cases: Event already deleted, concurrency conflicts

- **Delete Calendar Event**  
  - Type: Google Calendar Tool node  
  - Role: Removes events upon user cancellation requests  
  - Configuration: Event ID provided by AI Agent  
  - Inputs: Connected as AI tool to AI Agent  
  - Outputs: Deletion confirmation  
  - Edge cases: Event not found, permission denied

---

#### 2.5 Output Delivery

**Overview:**  
Sends the AI agent’s response or calendar confirmation back to the WhatsApp user.

**Nodes Involved:**  
- Send Answer

**Node Details:**

- **Send Answer**  
  - Type: WhatsApp node  
  - Role: Sends textual replies to users on WhatsApp, completing the interaction loop  
  - Configuration: Uses WhatsApp webhookId linked to the incoming channel  
  - Inputs: Output from AI Agent after processing user request and calendar actions  
  - Outputs: Final message dispatch  
  - Edge cases: WhatsApp API message rejection or delays

---

### 3. Summary Table

| Node Name                 | Node Type                            | Functional Role                      | Input Node(s)                    | Output Node(s)                 | Sticky Note                          |
|---------------------------|------------------------------------|-----------------------------------|---------------------------------|-------------------------------|------------------------------------|
| WhatsApp Trigger          | WhatsApp Trigger                   | Receives WhatsApp messages         | —                               | Variables                     |                                    |
| When chat message received | LangChain Chat Trigger             | Alternative chat message input     | —                               | Edit Fields                   |                                    |
| When Executed by Another Workflow | Execute Workflow Trigger          | Allows external workflow start     | —                               | Initialization                |                                    |
| Variables                 | Set                                | Initialize variables               | WhatsApp Trigger                | Initialization                |                                    |
| Edit Fields               | Set                                | Normalize input fields             | When chat message received      | Initialization                |                                    |
| Initialization            | Set                                | Prepare flags and context          | Variables, Edit Fields, When Executed by Another Workflow | Is start?                    |                                    |
| Is start?                 | If                                 | Check if message is a start command | Initialization                 | Welcome message, Define Type  |                                    |
| Welcome message           | WhatsApp                           | Send greeting message              | Is start?                      | —                             |                                    |
| Define Type               | Switch                            | Determine request type             | Is start?                      | AI Agent                      |                                    |
| AI Agent                  | LangChain Agent                   | AI decision-making and tool control | Define Type                   | Send Answer                  |                                    |
| Google Gemini Chat Model  | LangChain Language Model          | Natural language model             | AI Agent (ai_languageModel)    | AI Agent                     |                                    |
| Simple Memory             | LangChain Memory Buffer Window    | Maintains conversation context    | AI Agent (ai_memory)           | AI Agent                     |                                    |
| Get Calendar Event        | Google Calendar Tool              | Fetch event details                | AI Agent (ai_tool)             | AI Agent                     |                                    |
| Create Calendar Event     | Google Calendar Tool              | Create new calendar event          | AI Agent (ai_tool)             | AI Agent                     |                                    |
| Update Calendar Event     | Google Calendar Tool              | Update existing event              | AI Agent (ai_tool)             | AI Agent                     |                                    |
| Delete Calendar Event     | Google Calendar Tool              | Delete calendar event              | AI Agent (ai_tool)             | AI Agent                     |                                    |
| Send Answer               | WhatsApp                         | Send response to WhatsApp user     | AI Agent                      | —                             |                                    |
| Sticky Note2              | Sticky Note                      | (Empty placeholder note)           | —                             | —                             |                                    |
| Sticky Note               | Sticky Note                      | (Empty placeholder note)           | —                             | —                             |                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create WhatsApp Trigger Node**  
   - Type: WhatsApp Trigger  
   - Configure with your WhatsApp webhook ID to receive incoming messages.

2. **Create ‘Variables’ Set Node**  
   - Initialize necessary workflow variables (e.g., user context, flags) with default values.  
   - Connect input from WhatsApp Trigger.

3. **Create ‘When chat message received’ LangChain Chat Trigger Node**  
   - Configure with a LangChain-compatible webhook ID.  
   - Connect output to ‘Edit Fields’ node for pre-processing.

4. **Create ‘Edit Fields’ Set Node**  
   - Map and normalize incoming message fields to a structured format for AI processing.  
   - Connect output to ‘Initialization’ node.

5. **Create ‘When Executed by Another Workflow’ Node**  
   - Configure to allow external workflows to trigger this workflow.  
   - Connect output to ‘Initialization’ node.

6. **Create ‘Initialization’ Set Node**  
   - Perform additional variable setup or state initialization.  
   - Connect input from ‘Variables’, ‘Edit Fields’, and ‘When Executed by Another Workflow’.  
   - Connect output to ‘Is start?’ node.

7. **Create ‘Is start?’ If Node**  
   - Configure condition to check if incoming message is a start command (e.g., contains "start", "hello").  
   - True branch connects to ‘Welcome message’.  
   - False branch connects to ‘Define Type’.

8. **Create ‘Welcome message’ WhatsApp Node**  
   - Configure with WhatsApp webhook ID.  
   - Set static welcome text like “Welcome to the Appointment Scheduler!”.  
   - Connect input from ‘Is start?’ true branch.

9. **Create ‘Define Type’ Switch Node**  
   - Set cases to detect user intent types (e.g., create, update, delete, query).  
   - Connect input from ‘Is start?’ false branch.  
   - Connect matched intent to ‘AI Agent’.

10. **Create ‘AI Agent’ LangChain Agent Node**  
    - Configure with API credentials and link to:  
      - Google Gemini Chat Model (AI language model)  
      - Simple Memory (AI memory buffer)  
      - Google Calendar Tool nodes (as AI tools)  
    - Connect input from ‘Define Type’.

11. **Create ‘Google Gemini Chat Model’ Node**  
    - Configure API credentials for Google Gemini.  
    - Connect as language model input to ‘AI Agent’.

12. **Create ‘Simple Memory’ LangChain Memory Node**  
    - Configure memory buffer size (default is fine).  
    - Connect as memory input to ‘AI Agent’.

13. **Create Google Calendar Tool Nodes:**  
    - **Get Calendar Event**: Configure with Google OAuth2 credentials, set for event retrieval.  
    - **Create Calendar Event**: Configure for event creation.  
    - **Update Calendar Event**: Configure for event updates.  
    - **Delete Calendar Event**: Configure for event deletion.  
    - Connect each as AI tools input to ‘AI Agent’.

14. **Create ‘Send Answer’ WhatsApp Node**  
    - Configure with WhatsApp webhook ID for outbound messages.  
    - Connect input from ‘AI Agent’ output.

15. **Create Sticky Notes** (Optional)  
    - Add any you find helpful for documentation or comments inside the editor.

16. **Connect all nodes as per the logical flow:**  
    WhatsApp Trigger → Variables → Initialization → Is start? → (True) Welcome message  
    Is start? → (False) Define Type → AI Agent → Send Answer  
    When chat message received → Edit Fields → Initialization  
    When Executed by Another Workflow → Initialization  
    AI Agent ↔ Google Gemini Chat Model, Simple Memory, Calendar Tool Nodes

17. **Set workflow execution order to “v1” in settings.**

18. **Test the workflow end-to-end with WhatsApp messages to verify calendar interactions and AI responses.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                   | Context or Link                                                |
|-------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| The workflow uses Google Gemini AI via LangChain for advanced conversational AI capabilities in appointment scheduling.        | AI integration with Google Gemini via LangChain               |
| Google Calendar nodes require OAuth2 credentials with appropriate scopes for reading and writing calendar events.              | Google Calendar API and OAuth2 setup                           |
| WhatsApp nodes depend on configured WhatsApp Business API webhook IDs for message reception and sending.                       | WhatsApp Business API documentation                            |
| The AI Agent node coordinates between language model, memory, and external tools for complex decision making.                  | LangChain AI Agent pattern                                    |
| For detailed Google Calendar API limits and error handling, consult Google developer documentation.                            | https://developers.google.com/calendar                          |

---

**Disclaimer:**  
The provided text is solely generated from an automated workflow created with n8n, an integration and automation tool. This processing fully complies with applicable content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.