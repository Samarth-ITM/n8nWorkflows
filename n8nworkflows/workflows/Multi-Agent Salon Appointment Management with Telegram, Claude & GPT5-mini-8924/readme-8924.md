Multi-Agent Salon Appointment Management with Telegram, Claude & GPT5-mini

https://n8nworkflows.xyz/workflows/multi-agent-salon-appointment-management-with-telegram--claude---gpt5-mini-8924


# Multi-Agent Salon Appointment Management with Telegram, Claude & GPT5-mini

---

## 1. Workflow Overview

This workflow automates appointment management for a nail salon using a multi-agent AI system interacting through Telegram. It integrates Telegram user inputs, rate-limiting controls, Redis for distributed processing locks and message batching, AI agents leveraging Claude and GPT-5-mini for natural language understanding and booking logic, and Google Calendar for scheduling. The workflow also supports multimedia inputs (audio, images, PDF files), extracting and processing content accordingly.

The workflow is logically divided into these blocks:

- **1.1 Input Reception & Authorization:** Telegram message trigger and user role verification (owner vs. client).
- **1.2 Rate Limiting & Message Queueing:** Controls user request rates using Redis and custom logic, batching messages to avoid overload.
- **1.3 Distributed Processing Lock:** Implements Redis-based locking to ensure only one processor handles queued messages at a time.
- **1.4 Message Processing & AI Interaction:** Processes messages via AI agents (GPT-5-mini, Gemini, Claude) with Redis chat memory and multi-agent cooperation.
- **1.5 Multimedia Content Handling:** Downloads and analyzes audio, image, and PDF file attachments from Telegram messages.
- **1.6 Calendar Integration & Reminders:** Fetches Google Calendar events and sends appointment reminders via Telegram on schedule.
- **1.7 Booking Confirmation:** Sends final booking confirmation messages to users and owners.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception & Authorization

**Overview:**  
Receives incoming Telegram messages, differentiates between salon owners and customers, and routes the workflow accordingly.

**Nodes Involved:**  
- Telegram Trigger  
- Is Owner?  
- Execute Airtable Agent (sub-workflow)  

**Node Details:**  

- **Telegram Trigger**  
  - *Type:* Telegram Trigger  
  - *Role:* Entry point; listens for Telegram messages via webhook.  
  - *Config:* Uses Telegram webhook ID for incoming updates.  
  - *Connections:* Outputs to "Is Owner?".  
  - *Failures:* Webhook misconfiguration, Telegram API errors.  

- **Is Owner?**  
  - *Type:* If Node  
  - *Role:* Checks if message sender is the salon owner.  
  - *Config:* Condition based on Telegram user ID or chat ID.  
  - *Inputs:* From Telegram Trigger.  
  - *Outputs:* If true, triggers "Execute Airtable Agent" (likely owner-specific workflow); if false, proceeds to "Set Initial Data".  
  - *Failures:* Incorrect user ID config may misroute users.  

- **Execute Airtable Agent**  
  - *Type:* Execute Workflow  
  - *Role:* Invokes an external workflow for owner commands or data access (e.g., Airtable integration).  
  - *Config:* Points to a sub-workflow for owner operations.  
  - *Failures:* Sub-workflow errors, Airtable API issues, auth errors.  

---

### 2.2 Rate Limiting & Message Queueing

**Overview:**  
Implements rate limits to prevent abuse, uses Redis to batch messages, and checks current load before processing.

**Nodes Involved:**  
- Set Initial Data  
- Rate Limiter  
- Redis Hourly  
- Increment Counter Hourly  
- Check Limit  
- Check Rate Limited  
- Send Mesage?  
- Push (to Redis queue)  

**Node Details:**  

- **Set Initial Data**  
  - *Type:* Set Node  
  - *Role:* Initializes data fields and variables related to rate limiting and message metadata.  
  - *Connections:* Outputs to "Rate Limiter".  
  - *Failures:* Missing or incorrect initial data may break logic downstream.  

- **Rate Limiter**  
  - *Type:* Code Node  
  - *Role:* Custom JavaScript to enforce rate limiting policies (e.g., max requests per hour).  
  - *Inputs:* Data from Set Initial Data.  
  - *Outputs:* Proceeds to Redis Hourly node.  
  - *Failures:* Code errors, Redis connection issues.  

- **Redis Hourly**  
  - *Type:* Redis Node  
  - *Role:* Reads/modifies hourly counters for rate limiting.  
  - *Inputs:* From Rate Limiter.  
  - *Outputs:* To Increment Counter Hourly.  
  - *Failures:* Redis server downtime, connection errors.  

- **Increment Counter Hourly**  
  - *Type:* Redis Node  
  - *Role:* Increments the hourly request counter for the user.  
  - *Outputs:* To Check Limit node.  
  - *Failures:* Redis errors.  

- **Check Limit**  
  - *Type:* Code Node  
  - *Role:* Checks if user has reached the allowed request limit.  
  - *Outputs:* To "Check Rate Limited".  
  - *Failures:* Logic errors or expression failures.  

- **Check Rate Limited**  
  - *Type:* If Node  
  - *Role:* Branches based on whether rate limit is reached.  
  - *Outputs:* If limited, pushes message to Redis queue; otherwise, sends message immediately.  
  - *Failures:* Misconfigured condition may block or flood users.  

- **Send Mesage?**  
  - *Type:* If Node  
  - *Role:* Determines whether to send message now or batch it.  
  - *Outputs:* If yes, sends user message; if no, pushes to queue.  

- **Push**  
  - *Type:* Redis Node  
  - *Role:* Pushes user message to Redis list for batching.  
  - *Outputs:* To "Set Processing Lock".  
  - *Failures:* Redis connectivity issues.  

---

### 2.3 Distributed Processing Lock

**Overview:**  
Manages a distributed lock using Redis to ensure only one instance processes the batch of messages at a time.

**Nodes Involved:**  
- Set Processing Lock  
- Wait  
- Get Current Processing Lock  
- Am I the Processor?  
- Pop All Batched Messages  
- Delete Message List  
- Delete Processing Lock  

**Node Details:**  

- **Set Processing Lock**  
  - *Type:* Redis Node  
  - *Role:* Attempts to set a Redis lock key to claim processing.  
  - *Outputs:* To "Wait".  
  - *Failures:* Lock acquisition failure, Redis errors.  

- **Wait**  
  - *Type:* Wait Node  
  - *Role:* Pauses workflow briefly to allow lock propagation.  
  - *Outputs:* To "Get Current Processing Lock".  
  - *Failures:* Timing issues.  

- **Get Current Processing Lock**  
  - *Type:* Redis Node  
  - *Role:* Reads current lock status.  
  - *Outputs:* To "Am I the Processor?".  
  - *Failures:* Redis errors.  

- **Am I the Processor?**  
  - *Type:* If Node  
  - *Role:* Checks if this instance holds the processing lock.  
  - *Outputs:* If yes, proceeds to pop messages; if no, stops processing.  
  - *Failures:* Incorrect lock key evaluation.  

- **Pop All Batched Messages**  
  - *Type:* Redis Node  
  - *Role:* Retrieves and removes all messages from the Redis list for processing.  
  - *Outputs:* To "Delete Message List".  
  - *Failures:* Redis list empty or connection issues.  

- **Delete Message List**  
  - *Type:* Redis Node  
  - *Role:* Deletes the message list from Redis to clear queue.  
  - *Outputs:* To "Delete Processing Lock".  
  - *Failures:* Redis errors.  

- **Delete Processing Lock**  
  - *Type:* Redis Node  
  - *Role:* Removes the processing lock to allow next processor.  
  - *Outputs:* To "Combine Messages" (rest of processing).  
  - *Failures:* Redis errors, lock persistence issues.  

---

### 2.4 Message Processing & AI Interaction

**Overview:**  
Processes batched messages, interacts with AI language models and agents, maintains conversational memory, and sends booking messages.

**Nodes Involved:**  
- Combine Messages  
- Get many events (Google Calendar)  
- Limit Reached?  
- Switch  
- Text  
- Get Audio Url1  
- Get Image Url1  
- Only PDF File1  
- Not supported1  
- Download Audio1  
- Transcribe Audio1  
- Analyze Image1  
- Download File1  
- Extract from File1  
- Audio1  
- Image1  
- File1  
- gpt-5-mini (OpenAI LM)  
- gemini-2.5-flash (Google Gemini LM)  
- nail-salon-booking-mcp (Claude MCP Client)  
- Redis Chat Memory (Langchain memory node)  
- Booking Agent (Langchain Agent)  
- cancel_agent (Langchain agent tool)  
- send_acknowledgement (Telegram tool)  
- Send Booking Message (Telegram node)  
- Send User Message1 (Telegram node)  
- Send Owner Message1 (Telegram node)  
- Not supported1 (Telegram node)  
- Incorrect format1 (Telegram node)  

**Node Details:**  

- **Combine Messages**  
  - *Type:* Code Node  
  - *Role:* Aggregates all batched messages into a single data structure for processing.  
  - *Outputs:* To "Get many events".  
  - *Failures:* Logic bugs may cause message loss or format errors.  

- **Get many events**  
  - *Type:* Google Calendar Node  
  - *Role:* Retrieves calendar events for appointment availability and scheduling.  
  - *Outputs:* To "Limit Reached?".  
  - *Failures:* Google API auth errors or quota limits.  

- **Limit Reached?**  
  - *Type:* Code Node  
  - *Role:* Checks if booking limit or schedule conflicts exist.  
  - *Outputs:* To "Switch".  
  - *Failures:* Logical errors in limit calculation.  

- **Switch**  
  - *Type:* Switch Node  
  - *Role:* Routes message processing based on input type: text, audio, image, PDF, or unsupported.  
  - *Outputs:* To respective content processing nodes.  
  - *Failures:* Misclassification leads to improper handling.  

- **Text**  
  - *Type:* Set Node  
  - *Role:* Prepares text data for AI agent processing.  
  - *Outputs:* To "Booking Agent".  

- **Get Audio Url1**  
  - *Type:* HTTP Request  
  - *Role:* Downloads audio file URL from Telegram message.  
  - *Outputs:* To "Download Audio1".  

- **Download Audio1**  
  - *Type:* HTTP Request  
  - *Role:* Downloads audio content.  
  - *Outputs:* To "Transcribe Audio1".  

- **Transcribe Audio1**  
  - *Type:* OpenAI Node  
  - *Role:* Uses OpenAI Whisper or similar to transcribe audio to text.  
  - *Outputs:* To "Audio1".  

- **Audio1**  
  - *Type:* Set Node  
  - *Role:* Sets transcribed text for downstream processing.  
  - *Outputs:* To "Booking Agent".  

- **Get Image Url1**  
  - *Type:* HTTP Request  
  - *Role:* Downloads image file URL.  
  - *Outputs:* To "Analyze Image1".  

- **Analyze Image1**  
  - *Type:* OpenAI Node  
  - *Role:* Uses OpenAI vision or similar to analyze image content.  
  - *Outputs:* To "Image1".  

- **Image1**  
  - *Type:* Set Node  
  - *Role:* Sets image analysis results for agent.  
  - *Outputs:* To "Booking Agent".  

- **Only PDF File1**  
  - *Type:* If Node  
  - *Role:* Checks if uploaded file is PDF format; if not, sends incorrect format message.  
  - *Outputs:* To "Get File Url1" or "Incorrect format1".  

- **Get File Url1**  
  - *Type:* HTTP Request  
  - *Role:* Downloads PDF file URL.  
  - *Outputs:* To "Download File1".  

- **Download File1**  
  - *Type:* HTTP Request  
  - *Role:* Downloads file content.  
  - *Outputs:* To "Extract from File1".  

- **Extract from File1**  
  - *Type:* Extract From File Node  
  - *Role:* Extracts text content from PDF.  
  - *Outputs:* To "File1".  

- **File1**  
  - *Type:* Set Node  
  - *Role:* Sets extracted file content for agent processing.  
  - *Outputs:* To "Booking Agent".  

- **Incorrect format1**  
  - *Type:* Telegram Node  
  - *Role:* Sends message to user about unsupported file format.  

- **Not supported1**  
  - *Type:* Telegram Node  
  - *Role:* Responds to unsupported message types.  

- **gpt-5-mini**  
  - *Type:* OpenAI Langchain LM Chat Node  
  - *Role:* Provides AI language model processing for natural language understanding and booking.  
  - *Outputs:* To "Booking Agent" and "cancel_agent".  
  - *Failures:* API key issues, rate limits, model errors.  

- **gemini-2.5-flash**  
  - *Type:* Google Gemini LM Chat Node  
  - *Role:* Alternative language model for agent collaboration.  
  - *Outputs:* To "Booking Agent".  

- **nail-salon-booking-mcp**  
  - *Type:* Claude MCP Client Tool  
  - *Role:* Provides Claude AI multi-agent coordination for complex booking logic.  
  - *Outputs:* To "cancel_agent" and "Booking Agent".  

- **Redis Chat Memory**  
  - *Type:* Redis Chat Memory Node (Langchain)  
  - *Role:* Maintains conversational context stored in Redis for agents.  
  - *Outputs:* To "cancel_agent" and "Booking Agent".  
  - *Failures:* Redis connection or data consistency errors.  

- **Booking Agent**  
  - *Type:* Langchain Agent Node  
  - *Role:* Core AI agent that processes user requests and formulates booking decisions and messages.  
  - *Outputs:* To "Send Booking Message".  

- **cancel_agent**  
  - *Type:* Langchain Agent Tool Node  
  - *Role:* Secondary agent handling cancellations or special commands.  
  - *Outputs:* To "Booking Agent".  

- **send_acknowledgement**  
  - *Type:* Telegram Tool Node  
  - *Role:* Sends acknowledgements back to users post-processing.  
  - *Outputs:* To "Booking Agent".  

- **Send Booking Message**  
  - *Type:* Telegram Node  
  - *Role:* Sends final booking confirmation message to user.  

- **Send User Message1**  
  - *Type:* Telegram Node  
  - *Role:* Sends user-related messages.  
  - *Outputs:* To "Send Owner Message1".  

- **Send Owner Message1**  
  - *Type:* Telegram Node  
  - *Role:* Sends messages to salon owner.  

---

### 2.5 Multimedia Content Handling

**Overview:**  
Dedicated download and analysis nodes handle media attachments from Telegram messages, converting them into text or structured data for the AI agents.

**Nodes Involved:**  
- Get Audio Url1  
- Download Audio1  
- Transcribe Audio1  
- Audio1  
- Get Image Url1  
- Analyze Image1  
- Image1  
- Only PDF File1  
- Get File Url1  
- Download File1  
- Extract from File1  
- File1  
- Incorrect format1  

**Node Details:**  
Covered in section 2.4 above as part of message processing.

---

### 2.6 Calendar Integration & Reminders

**Overview:**  
Periodically triggers scheduled reminders for appointments using Google Calendar data and sends Telegram reminders to clients.

**Nodes Involved:**  
- Schedule Trigger1  
- Calculate Tomorrow1  
- Get Schedule Events1  
- Format Reminder Data1  
- Send Client Reminder1  

**Node Details:**  

- **Schedule Trigger1**  
  - *Type:* Schedule Trigger  
  - *Role:* Periodic trigger (e.g., daily) to initiate reminder workflow.  
  - *Failures:* Scheduling misconfiguration.  

- **Calculate Tomorrow1**  
  - *Type:* Code Node  
  - *Role:* Calculates date/time for the next day or reminder window.  
  - *Outputs:* To "Get Schedule Events1".  

- **Get Schedule Events1**  
  - *Type:* Google Calendar Node  
  - *Role:* Fetches events for the calculated day from Google Calendar.  
  - *Outputs:* To "Format Reminder Data1".  
  - *Failures:* Google API errors.  

- **Format Reminder Data1**  
  - *Type:* Code Node  
  - *Role:* Processes event data to format reminder messages.  
  - *Outputs:* To "Send Client Reminder1".  

- **Send Client Reminder1**  
  - *Type:* Telegram Node  
  - *Role:* Sends the reminder message to clients.  

---

### 2.7 Booking Confirmation

**Overview:**  
Sends final booking confirmation and acknowledgment messages to clients and owners to complete the appointment flow.

**Nodes Involved:**  
- Send Booking Message  
- Send User Message1  
- Send Owner Message1  
- send_acknowledgement  

**Node Details:**  

- **Send Booking Message**  
  - *Type:* Telegram Node  
  - *Role:* Sends the booking confirmation message to the user.  

- **Send User Message1**  
  - *Type:* Telegram Node  
  - *Role:* Sends messages to user, often intermediate responses.  
  - *Outputs:* To "Send Owner Message1".  

- **Send Owner Message1**  
  - *Type:* Telegram Node  
  - *Role:* Sends notification messages to the salon owner.  

- **send_acknowledgement**  
  - *Type:* Telegram Tool Node  
  - *Role:* Sends acknowledgments to users post agent processing.  

---

## 3. Summary Table

| Node Name               | Node Type                         | Functional Role                                   | Input Node(s)                 | Output Node(s)                         | Sticky Note                          |
|-------------------------|----------------------------------|-------------------------------------------------|------------------------------|---------------------------------------|------------------------------------|
| Telegram Trigger        | Telegram Trigger                 | Entry point for Telegram messages                |                              | Is Owner?                             |                                    |
| Is Owner?               | If Node                         | Checks if sender is salon owner                   | Telegram Trigger             | Execute Airtable Agent / Set Initial Data |                                    |
| Execute Airtable Agent  | Execute Workflow                | Runs owner-specific sub-workflow                   | Is Owner?                    |                                       |                                    |
| Set Initial Data        | Set Node                       | Initializes variables for rate limiting            | Is Owner?                    | Rate Limiter                         |                                    |
| Rate Limiter            | Code Node                      | Enforces rate limiting                              | Set Initial Data             | Redis Hourly                        |                                    |
| Redis Hourly            | Redis Node                     | Reads hourly request counter                        | Rate Limiter                 | Increment Counter Hourly             |                                    |
| Increment Counter Hourly| Redis Node                     | Increments hourly counter                           | Redis Hourly                 | Check Limit                        |                                    |
| Check Limit             | Code Node                      | Checks if user exceeded request limit              | Increment Counter Hourly     | Check Rate Limited                 |                                    |
| Check Rate Limited      | If Node                       | Branches based on rate limit status                 | Check Limit                  | Send Mesage? / Push                |                                    |
| Send Mesage?            | If Node                       | Determines immediate send or queue                   | Check Rate Limited           | Send User Message1 / Push           |                                    |
| Push                    | Redis Node                     | Adds message to Redis queue                          | Send Mesage?                 | Set Processing Lock                |                                    |
| Set Processing Lock     | Redis Node                     | Claims Redis lock for processing                     | Push                        | Wait                             |                                    |
| Wait                    | Wait Node                     | Short delay for lock propagation                      | Set Processing Lock          | Get Current Processing Lock        |                                    |
| Get Current Processing Lock | Redis Node                  | Reads current Redis lock status                        | Wait                        | Am I the Processor?                |                                    |
| Am I the Processor?     | If Node                       | Checks if this instance holds the lock                | Get Current Processing Lock  | Pop All Batched Messages / Stop    |                                    |
| Pop All Batched Messages| Redis Node                     | Retrieves all queued messages                         | Am I the Processor?          | Delete Message List               |                                    |
| Delete Message List     | Redis Node                     | Cleans Redis message list                             | Pop All Batched Messages     | Delete Processing Lock            |                                    |
| Delete Processing Lock  | Redis Node                     | Releases processing lock                              | Delete Message List          | Combine Messages                 |                                    |
| Combine Messages        | Code Node                      | Aggregates batched messages for processing            | Delete Processing Lock       | Get many events                  |                                    |
| Get many events         | Google Calendar Node           | Retrieves calendar events                             | Combine Messages             | Limit Reached?                   |                                    |
| Limit Reached?          | Code Node                      | Checks booking limits and conflicts                    | Get many events              | Switch                         |                                    |
| Switch                  | Switch Node                   | Routes messages by content type                        | Limit Reached?               | Text / Audio / Image / PDF / Not supported |                                    |
| Text                    | Set Node                       | Prepares text for AI processing                         | Switch                      | Booking Agent                  |                                    |
| Get Audio Url1          | HTTP Request                  | Downloads audio URL                                    | Switch                      | Download Audio1                |                                    |
| Download Audio1         | HTTP Request                  | Downloads audio content                                | Get Audio Url1              | Transcribe Audio1              |                                    |
| Transcribe Audio1       | OpenAI Node                   | Transcribes audio to text                              | Download Audio1             | Audio1                       |                                    |
| Audio1                  | Set Node                       | Sets transcribed audio text                            | Transcribe Audio1           | Booking Agent                  |                                    |
| Get Image Url1          | HTTP Request                  | Downloads image URL                                    | Switch                      | Analyze Image1               |                                    |
| Analyze Image1          | OpenAI Node                   | Analyzes image content                                 | Get Image Url1              | Image1                       |                                    |
| Image1                  | Set Node                       | Sets image analysis results                            | Analyze Image1              | Booking Agent                  |                                    |
| Only PDF File1          | If Node                       | Checks if file is PDF                                  | Switch                      | Get File Url1 / Incorrect format1 |                                    |
| Get File Url1           | HTTP Request                  | Downloads PDF URL                                     | Only PDF File1              | Download File1               |                                    |
| Download File1          | HTTP Request                  | Downloads PDF content                                 | Get File Url1               | Extract from File1           |                                    |
| Extract from File1      | Extract From File Node         | Extracts text from PDF                                 | Download File1              | File1                        |                                    |
| File1                   | Set Node                       | Sets extracted PDF text                                | Extract from File1           | Booking Agent                  |                                    |
| Incorrect format1       | Telegram Node                 | Sends unsupported format message                       | Only PDF File1              |                              |                                    |
| Not supported1          | Telegram Node                 | Sends unsupported content message                      | Switch                      |                              |                                    |
| gpt-5-mini              | OpenAI LM Chat Node           | AI language model for natural language processing      |                             | Booking Agent / cancel_agent  |                                    |
| gemini-2.5-flash        | Google Gemini LM Chat Node    | Alternative AI language model                           |                             | Booking Agent                 |                                    |
| nail-salon-booking-mcp  | Claude MCP Client Tool        | Claude multi-agent processing                           |                             | cancel_agent / Booking Agent |                                    |
| Redis Chat Memory       | Redis Langchain Memory Node   | Maintains conversational context                        |                             | cancel_agent / Booking Agent |                                    |
| Booking Agent           | Langchain Agent Node          | Core AI agent managing booking logic                    |                             | Send Booking Message         |                                    |
| cancel_agent            | Langchain Agent Tool Node     | Handles cancellations and special commands             | gpt-5-mini / nail-salon-booking-mcp | Booking Agent           |                                    |
| send_acknowledgement    | Telegram Tool Node            | Sends acknowledgements to user                           |                             | Booking Agent                |                                    |
| Send Booking Message    | Telegram Node                 | Sends booking confirmation message                      | Booking Agent               |                             |                                    |
| Send User Message1      | Telegram Node                 | Sends messages to user                                   | Send Mesage?                | Send Owner Message1          |                                    |
| Send Owner Message1     | Telegram Node                 | Sends messages to owner                                  | Send User Message1          |                             |                                    |
| Schedule Trigger1       | Schedule Trigger             | Triggers scheduled reminders                            |                             | Calculate Tomorrow1          |                                    |
| Calculate Tomorrow1     | Code Node                    | Calculates next day for reminders                       | Schedule Trigger1           | Get Schedule Events1         |                                    |
| Get Schedule Events1    | Google Calendar Node         | Fetches calendar events for reminders                   | Calculate Tomorrow1         | Format Reminder Data1        |                                    |
| Format Reminder Data1   | Code Node                    | Formats reminder messages                                | Get Schedule Events1        | Send Client Reminder1        |                                    |
| Send Client Reminder1   | Telegram Node                | Sends appointment reminders to clients                  | Format Reminder Data1       |                             |                                    |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node:**  
   - Type: Telegram Trigger  
   - Configure webhook with Telegram bot token.  
   - Position: Start point.  

2. **Add "Is Owner?" If Node:**  
   - Check Telegram user ID or chat ID against owner ID.  
   - True branch: Connect to "Execute Airtable Agent".  
   - False branch: Connect to "Set Initial Data".  

3. **Add "Execute Airtable Agent" Execute Workflow Node:**  
   - Link to sub-workflow managing owner commands (requires sub-workflow creation).  

4. **Create "Set Initial Data" Node:**  
   - Initialize variables for rate limiting (e.g., user ID, timestamps).  
   - Connect from false branch of "Is Owner?".  

5. **Add "Rate Limiter" Code Node:**  
   - Implement JS logic for rate limiting policies.  
   - Connect from "Set Initial Data".  

6. **Add Redis Nodes for Hourly Counters:**  
   - "Redis Hourly": Read current user hourly count.  
   - "Increment Counter Hourly": Increment counter.  
   - Connect sequentially.  

7. **Add "Check Limit" Code Node:**  
   - Logic to check if limit reached.  
   - Connect from "Increment Counter Hourly".  

8. **Add "Check Rate Limited" If Node:**  
   - Branch based on rate limit status.  
   - If limited: Connect to "Push" Redis Node.  
   - Else: Connect to "Send Mesage?".  

9. **Add "Send Mesage?" If Node:**  
   - Decide whether to send message now or batch it.  
   - If send now: Connect to "Send User Message1".  
   - Else: Connect to "Push".  

10. **Implement "Push" Redis Node:**  
    - Push message to Redis list for batching.  
    - Connect to "Set Processing Lock".  

11. **Add Distributed Lock Nodes:**  
    - "Set Processing Lock" Redis Node: Set lock key.  
    - "Wait" Node: Short delay (e.g., 1-2 seconds).  
    - "Get Current Processing Lock" Redis Node: Read lock.  
    - "Am I the Processor?" If Node: Check lock ownership.  
    - If yes: Continue to "Pop All Batched Messages".  
    - If no: End flow for this trigger.  

12. **Add Redis Nodes for Message Queue:**  
    - "Pop All Batched Messages": Retrieve all batch.  
    - "Delete Message List": Clear batch list.  
    - "Delete Processing Lock": Release lock.  
    - Connect sequentially.  

13. **Add "Combine Messages" Code Node:**  
    - Aggregate messages for AI processing.  
    - Connect from "Delete Processing Lock".  

14. **Add Google Calendar Node "Get many events":**  
    - Fetch calendar events for scheduling context.  
    - Connect from "Combine Messages".  

15. **Add "Limit Reached?" Code Node:**  
    - Check conflict or booking limits.  
    - Connect from "Get many events".  

16. **Add "Switch" Node:**  
    - Route based on message content type (text, audio, image, PDF, unsupported).  
    - Connect from "Limit Reached?".  

17. **For Each Content Type:**  
    - **Text:** Set Node to prepare data, connect to Booking Agent.  
    - **Audio:** HTTP Request to get URL, download, transcribe with OpenAI, set data, then to Booking Agent.  
    - **Image:** HTTP Request to get URL, analyze with OpenAI, set data, then to Booking Agent.  
    - **PDF:** If Node checks format, download, extract text, set data, then to Booking Agent.  
    - **Unsupported:** Send Telegram message to user about unsupported type.  

18. **Add AI Language Model Nodes:**  
    - "gpt-5-mini" (OpenAI LM): Connect outputs to Booking Agent and cancel_agent.  
    - "gemini-2.5-flash" (Google Gemini LM): Connect to Booking Agent.  
    - "nail-salon-booking-mcp" (Claude MCP): Connect outputs to cancel_agent and Booking Agent.  

19. **Add Redis Chat Memory Node:**  
    - Connect to AI agents for maintaining conversation context.  

20. **Set up Langchain Agents:**  
    - "Booking Agent": Core booking logic.  
    - "cancel_agent": Handles cancellations.  
    - Connect AI models and memory nodes accordingly.  

21. **Add Telegram Nodes for Responses:**  
    - "Send Booking Message": Sends final confirmation to user.  
    - "Send User Message1": Intermediate user messages.  
    - "Send Owner Message1": Notifications to salon owner.  
    - "send_acknowledgement": Telegram tool for acknowledgements.  

22. **Set up Scheduled Reminder Workflow:**  
    - "Schedule Trigger1": Set daily or desired frequency.  
    - "Calculate Tomorrow1": Code node to compute reminder date.  
    - "Get Schedule Events1": Google Calendar node to fetch events.  
    - "Format Reminder Data1": Code node for formatting messages.  
    - "Send Client Reminder1": Telegram node to send reminders.  

23. **Configure Credentials:**  
    - Telegram Bot API credentials with webhook enabled.  
    - Redis credentials for locking and message queue.  
    - OpenAI API key for GPT-5-mini and Whisper transcription.  
    - Google Cloud credentials for Gemini and Google Calendar.  
    - Claude API credentials for MCP client.  

24. **Test All Branches:**  
    - Verify owner vs. client routing.  
    - Test rate limiting by sending multiple messages.  
    - Check multimedia handling with audio, image, and PDF attachments.  
    - Validate AI agents respond correctly and bookings are confirmed.  
    - Confirm scheduled reminder messages are sent on time.  

---

## 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                     |
|-----------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| The workflow uses multi-agent AI architecture combining Claude, GPT-5-mini, and Google Gemini for robustness and varied AI capabilities. | AI models integration and multi-agent collaboration.                                              |
| Redis is heavily used for message batching, distributed locking, and chat memory, ensuring scalability and concurrency control. | Redis setup and reliability are critical for workflow stability.                                  |
| Telegram webhook configuration and bot token management are essential for real-time message processing. | Telegram Bot API documentation: https://core.telegram.org/bots/api                                   |
| Google Calendar integration requires OAuth2 credentials with appropriate scopes for event reading.       | Google Calendar API docs: https://developers.google.com/calendar/api                                |
| OpenAI Whisper or similar transcription model is used for audio processing, requiring accurate audio format handling. | OpenAI API docs: https://platform.openai.com/docs/models/whisper                                   |
| The workflow includes a scheduled trigger for sending appointment reminders, improving client engagement. | Scheduling best practices in n8n: https://docs.n8n.io/nodes/n8n-nodes-base.scheduleTrigger/         |
| Sub-workflows (e.g., Airtable Agent) should be created and tested separately before integration.         | Modular workflow design recommended for maintainability.                                          |
| Error handling and monitoring are advised to capture API errors, Redis failures, and AI model rate limit issues. | Consider adding error workflow or retry mechanisms.                                               |

---

**Disclaimer:**  
The provided content is exclusively derived from an automated n8n workflow designed for legal and public data processing. It respects content policies and contains no illegal, offensive, or protected elements.

---