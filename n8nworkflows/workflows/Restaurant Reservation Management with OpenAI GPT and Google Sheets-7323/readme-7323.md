Restaurant Reservation Management with OpenAI GPT and Google Sheets

https://n8nworkflows.xyz/workflows/restaurant-reservation-management-with-openai-gpt-and-google-sheets-7323


# Restaurant Reservation Management with OpenAI GPT and Google Sheets

### 1. Workflow Overview

This workflow automates restaurant table reservation management using conversational AI powered by OpenAI GPT and Google Sheets as a backend database. It is designed to handle customer interactions for making new reservations, updating existing bookings, and canceling reservations through chat messages.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception**: Receives incoming chat messages from customers via a webhook-based chat trigger.
- **1.2 AI Processing**: Uses an AI agent (OpenAI GPT model) to understand user intents, manage dialogue context, and orchestrate data retrieval, calculation, and updates.
- **1.3 Data Retrieval**: Queries Google Sheets to fetch table information, availability, and existing reservations.
- **1.4 Reservation Management**: Updates the Google Sheets data to create, modify, or delete reservation records and update table availability accordingly.
- **1.5 Calculation**: Uses a calculator tool to generate new unique Reservation IDs by incrementing the highest existing ID.
- **1.6 Memory**: Maintains conversational context between turns for multi-turn dialogue coherence.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for incoming chat messages from customers to initiate or continue reservation interactions.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**  
  - **When chat message received**  
    - *Type & Role:* Chat trigger node (Langchain chatTrigger)  
    - *Configuration:* Public webhook listening for chat messages with no additional options set.  
    - *Input/Output:* No input; outputs to AI Agent node.  
    - *Edge Cases:* Failures can occur if webhook is not properly set or if incoming messages are malformed.  
    - *Version:* 1.1  
    - *Sub-workflow:* None

#### 2.2 AI Processing

- **Overview:**  
  Processes the chat message using an AI agent that interprets the user's intent, maintains dialogue context, and coordinates calls to data and update tools.

- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model  
  - Simple Memory  
  - Calculator

- **Node Details:**  
  - **AI Agent**  
    - *Type & Role:* Langchain agent node responsible for orchestrating AI logic, integrating tools, and managing conversation flow.  
    - *Configuration:*  
      - System message defines the assistant's role as a reservation concierge.  
      - Lists tools available to the AI (data lookup, memory, calculation, write/modify).  
      - Contains detailed instructions for making, updating, and canceling reservations.  
      - Enforces behavior rules such as no past bookings, exact Slot ID usage, and incrementing Reservation ID via Calculator only.  
    - *Key Expressions:* Uses `$now` for current date/time, `$fromAI` expressions to parse AI-generated data for updates.  
    - *Input:* Receives chat trigger messages.  
    - *Output:* Controls flow to all other nodes via ai_tool, ai_memory, and ai_languageModel connections.  
    - *Edge Cases:* Expression failures if AI output does not match expected format; logic errors if AI does not follow instructions; OpenAI API rate limits or errors.  
    - *Version:* 2.1  
    - *Sub-workflow:* None

  - **OpenAI Chat Model**  
    - *Type & Role:* Language model node providing GPT-based chat completions.  
    - *Configuration:* Uses model "gpt-4.1-mini" with default options.  
    - *Input:* Takes messages from AI Agent.  
    - *Output:* Provides AI-generated text back to AI Agent.  
    - *Edge Cases:* API authentication errors, rate limits, or service outages.  
    - *Version:* 1.2  
    - *Credentials:* OpenAI API key required.

  - **Simple Memory**  
    - *Type & Role:* Memory buffer node to store conversation history for multi-turn dialogue.  
    - *Configuration:* Default settings, no explicit parameters.  
    - *Input:* Receives messages from AI Agent.  
    - *Output:* Returns conversation history to AI Agent.  
    - *Edge Cases:* Memory overflow if conversation is too long; loss of context on restart.  
    - *Version:* 1.3

  - **Calculator**  
    - *Type & Role:* Calculation tool used exclusively to compute next Reservation ID by incrementing the highest existing ID.  
    - *Configuration:* No parameters set, serves as a generic calculator for addition.  
    - *Input:* Called by AI Agent.  
    - *Output:* Returns incremented Reservation ID to AI Agent.  
    - *Edge Cases:* Calculation errors if input is missing or malformed.  
    - *Version:* 1

#### 2.3 Data Retrieval

- **Overview:**  
  Fetches required data from Google Sheets to provide the AI agent with current restaurant table information, availability, and existing reservations.

- **Nodes Involved:**  
  - Get Table Information  
  - Get Table Availability  
  - Get Table Reservations

- **Node Details:**  
  - **Get Table Information**  
    - *Type & Role:* Google Sheets Tool node to read table feature data.  
    - *Configuration:* Reads from the "Tables" sheet (gid=1179954207) in the designated Google Sheets document.  
    - *Credentials:* Uses OAuth2 credential to access Google Sheets.  
    - *Input:* Called by AI Agent.  
    - *Output:* Provides table features such as capacity, smoking allowed, near window, outdoor seating, high chair availability, and notes.  
    - *Edge Cases:* OAuth token expiration, API quota limits, missing or corrupted sheet data.  
    - *Version:* 4.6

  - **Get Table Availability**  
    - *Type & Role:* Google Sheets Tool node to read current table slot availability.  
    - *Configuration:* Reads from "Availability" sheet (gid=0) in the same Google Sheets document.  
    - *Credentials:* Same as above.  
    - *Input:* Called by AI Agent.  
    - *Output:* Provides slot IDs, dates, times, table IDs, capacities, and status (Available/Reserved).  
    - *Edge Cases:* Same as above.  
    - *Version:* 4.6

  - **Get Table Reservations**  
    - *Type & Role:* Google Sheets Tool node to read all current reservations.  
    - *Configuration:* Reads from "Reservations" sheet (gid=1246645050) in the Google Sheets document.  
    - *Credentials:* Same as above.  
    - *Input:* Called by AI Agent.  
    - *Output:* Provides reservation records including IDs, slot IDs, customer info, party size, status, notes, and booking timestamps.  
    - *Edge Cases:* Same as above.  
    - *Version:* 4.6

#### 2.4 Reservation Management

- **Overview:**  
  Updates reservation records and availability status in Google Sheets according to AI agent decisions.

- **Nodes Involved:**  
  - Update Reservations  
  - Cancel Reservations  
  - Update Table Availabilty

- **Node Details:**  
  - **Update Reservations**  
    - *Type & Role:* Google Sheets Tool node to append or modify reservation rows.  
    - *Configuration:* Appends new rows to the "Reservations" sheet with mapped columns for Reservation ID, Slot ID, Table ID, Customer Name, Contact, Party Size, Status, Booked At, Requested Features, and Notes.  
    - *Input:* Called by AI Agent with data parsed from AI output variables via `$fromAI`.  
    - *Output:* Confirms write success or failure.  
    - *Edge Cases:* Data type mismatches, missing fields, appending to protected sheets, OAuth issues.  
    - *Version:* 4.6

  - **Cancel Reservations**  
    - *Type & Role:* Google Sheets Tool node to delete reservation rows.  
    - *Configuration:* Deletes a row in the "Reservations" sheet by start index parsed from AI output.  
    - *Input:* Called by AI Agent based on user cancellation request.  
    - *Output:* Confirms deletion.  
    - *Edge Cases:* Incorrect row index, deletion of wrong entries, OAuth errors.  
    - *Version:* 4.6

  - **Update Table Availabilty**  
    - *Type & Role:* Google Sheets Tool node to update availability status of a specific slot.  
    - *Configuration:* Updates row matching Slot ID in the "Availability" sheet. Fields such as Date, Time, Table ID, Capacity, and Status (Reserved or Available) are updated.  
    - *Input:* Called by AI Agent after reservation or cancellation to reflect current status.  
    - *Output:* Confirms update success.  
    - *Edge Cases:* Failure to find matching Slot ID, partial updates, concurrency issues.  
    - *Version:* 4.6

---

### 3. Summary Table

| Node Name               | Node Type                         | Functional Role            | Input Node(s)                | Output Node(s)             | Sticky Note                                     |
|-------------------------|----------------------------------|----------------------------|------------------------------|----------------------------|------------------------------------------------|
| When chat message received | Langchain chatTrigger           | Input Reception            | None                         | AI Agent                   |                                                |
| AI Agent                | Langchain agent                  | AI Processing & Orchestration | When chat message received, Calculator, Simple Memory, OpenAI Chat Model, Get Table Information, Get Table Availability, Get Table Reservations, Update Reservations, Cancel Reservations, Update Table Availabilty | None (endpoint node)        |                                                |
| OpenAI Chat Model       | Langchain lmChatOpenAi           | AI Language Model          | AI Agent (ai_languageModel)  | AI Agent                   |                                                |
| Simple Memory           | Langchain memoryBufferWindow     | Conversation Memory        | AI Agent (ai_memory)         | AI Agent                   |                                                |
| Calculator              | Langchain toolCalculator         | Calculate next Reservation ID | AI Agent (ai_tool)           | AI Agent                   |                                                |
| Get Table Information   | Google Sheets Tool               | Fetch table features       | AI Agent (ai_tool)            | AI Agent                   |                                                |
| Get Table Availability  | Google Sheets Tool               | Fetch availability slots   | AI Agent (ai_tool)            | AI Agent                   |                                                |
| Get Table Reservations  | Google Sheets Tool               | Fetch existing reservations | AI Agent (ai_tool)            | AI Agent                   |                                                |
| Update Reservations     | Google Sheets Tool               | Append/modify reservations | AI Agent (ai_tool)            | AI Agent                   |                                                |
| Cancel Reservations     | Google Sheets Tool               | Delete reservation rows    | AI Agent (ai_tool)            | AI Agent                   |                                                |
| Update Table Availabilty | Google Sheets Tool               | Update slot availability status | AI Agent (ai_tool)            | AI Agent                   |                                                |
| Sticky Note             | Sticky Note                     | Documentation              | None                         | None                       | ## Handle Reservation                          |
| Sticky Note1            | Sticky Note                     | Documentation              | None                         | None                       | ## Get Reservation Data                        |
| Sticky Note2            | Sticky Note                     | Documentation              | None                         | None                       | ## AI                                          |
| Sticky Note3            | Sticky Note                     | Documentation              | None                         | None                       | ## Restaurant Table Reservation AI Agent      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Chat Trigger Node:**  
   - Node: *When chat message received* (Langchain chatTrigger)  
   - Configuration: Set webhook to public, no additional options.  
   - Purpose: To receive incoming chat messages from customers.

2. **Create the AI Agent Node:**  
   - Node: *AI Agent* (Langchain agent)  
   - Configuration:  
     - Paste the full system prompt defining assistant role, tools, tasks, and behavior guidelines (as per system message in the workflow).  
     - Connect ai_tool, ai_memory, ai_languageModel inputs and outputs accordingly.  
   - Purpose: Orchestrates conversation, tool usage, and response generation.

3. **Create the OpenAI Chat Model Node:**  
   - Node: *OpenAI Chat Model* (Langchain lmChatOpenAi)  
   - Configuration:  
     - Select model `gpt-4.1-mini`.  
     - Use your OpenAI API credentials.  
   - Connect output to AI Agent node’s ai_languageModel input.

4. **Create the Simple Memory Node:**  
   - Node: *Simple Memory* (Langchain memoryBufferWindow)  
   - Configuration: Default.  
   - Connect to AI Agent node’s ai_memory input.

5. **Create the Calculator Node:**  
   - Node: *Calculator* (Langchain toolCalculator)  
   - Configuration: Default, used by AI Agent for calculating next Reservation ID.  
   - Connect to AI Agent node’s ai_tool input.

6. **Create Google Sheets Nodes for Data Retrieval:**  
   - Nodes: *Get Table Information*, *Get Table Availability*, *Get Table Reservations* (all Google Sheets Tool nodes)  
   - Configuration for each:  
     - Set *Document ID* to your Google Sheets document ID.  
     - Set *Sheet Name* to the respective sheets:  
       - "Tables" for Table Information (gid=1179954207)  
       - "Availability" for Table Availability (gid=0)  
       - "Reservations" for Table Reservations (gid=1246645050)  
     - Use Google Sheets OAuth2 credentials.  
   - Connect outputs to AI Agent node’s ai_tool input.

7. **Create Google Sheets Nodes for Data Updates:**  
   - Nodes: *Update Reservations*, *Cancel Reservations*, *Update Table Availabilty* (Google Sheets Tool nodes)  
   - Configuration for each:  
     - Set *Document ID* and *Sheet Name* as above.  
     - *Update Reservations*: Operation = append, define column mappings exactly as per schema (Reservation ID, Slot ID, Table ID, Customer Name, Contact, Party Size, Status, Booked At, Requested Features, Notes).  
     - *Cancel Reservations*: Operation = delete, row index provided dynamically from AI output.  
     - *Update Table Availabilty*: Operation = update, matching on Slot ID, update fields like Date, Time, Status, Table ID, Capacity.  
   - Use Google Sheets OAuth2 credentials.  
   - Connect outputs to AI Agent node’s ai_tool input.

8. **Connect Workflow:**  
   - From *When chat message received* node main output to AI Agent main input.  
   - From AI Agent ai_languageModel output to OpenAI Chat Model input and back.  
   - From AI Agent ai_memory output to Simple Memory input and back.  
   - From AI Agent ai_tool output to Google Sheets nodes and Calculator node inputs as needed.

9. **Set Credentials:**  
   - Google Sheets OAuth2 with appropriate scope for reading and writing to the specified sheets.  
   - OpenAI API key with access to GPT-4 models.

10. **Test the Workflow:**  
    - Send chat messages simulating reservation requests.  
    - Verify AI agent responses and check updates in Google Sheets reflect bookings, updates, or cancellations correctly.

---

### 5. General Notes & Resources

| Note Content                                                                                                                | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| The AI agent is named "Julian, the Reservation Concierge" and responds with a friendly, professional tone.                  | AI Agent system message                                                                              |
| Reservation ID must always be incremented via the Calculator tool; Slot IDs are never generated or modified by AI.           | AI Agent system message                                                                              |
| Booking requests in the past are rejected automatically based on current date/time (`{{ $now }}`).                           | AI Agent system message                                                                              |
| Use these Google Sheets tabs for data storage: Tables (gid=1179954207), Availability (gid=0), Reservations (gid=1246645050). | Google Sheets node configurations                                                                    |
| For multi-turn conversations, Simple Memory node buffers prior user messages to maintain context.                            | Simple Memory node description                                                                       |

---

**Disclaimer:**  
The text provided is extracted exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.