Extract & Approve Tasks from Telegram to Notion with Gemini AI & Voice Support

https://n8nworkflows.xyz/workflows/extract---approve-tasks-from-telegram-to-notion-with-gemini-ai---voice-support-9424


# Extract & Approve Tasks from Telegram to Notion with Gemini AI & Voice Support

### 1. Workflow Overview

This workflow is designed to automate task management by extracting task information from Telegram messages (both text and voice), validating and approving the extracted tasks via Telegram, and then creating corresponding task entries in Notion. It leverages Google Gemini AI for natural language processing (NLP) tasks such as transcription and information extraction.

**Target Use Cases:**  
- Teams or individuals managing tasks via Telegram messages  
- Voice note users who want to convert spoken tasks into structured task entries  
- Automated task approval workflows with real-time user interaction via Telegram  
- Integration with Notion for centralized task tracking  

**Logical Blocks:**  
- **1.1 Input Reception:** Triggered by incoming Telegram messages (text or voice).  
- **1.2 Message Type Branching:** Determines if the input is text or voice and routes accordingly.  
- **1.3 Voice Processing Chain:** Downloads voice messages and transcribes them to text using Google Gemini.  
- **1.4 Text Preparation:** Prepares plain text messages for AI processing.  
- **1.5 AI Extraction:** Extracts task name and due date from the text using Google Gemini and LangChain nodes.  
- **1.6 Extraction Validation:** Checks if the extracted task data is valid.  
- **1.7 User Approval:** Sends task details back to the user on Telegram for approval or rejection.  
- **1.8 Approval Check:** Processes user approval response and routes to task creation or rejection notification.  
- **1.9 Notion Task Creation:** Creates a new task page in Notion and notifies the user of success or failure.  

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Listens for new messages on Telegram (both text and voice) to start the workflow.  
- **Nodes Involved:**  
  - Telegram: Receive Message  
  - Sticky Note (description node)

- **Node Details:**  
  - **Telegram: Receive Message**  
    - Type: Telegram Trigger  
    - Role: Initiates workflow on new Telegram message event.  
    - Configuration: Listens for message updates only.  
    - Inputs: Telegram message payload.  
    - Outputs: Raw Telegram message JSON.  
    - Credentials: Telegram API Bot Token.  
    - Edge Cases: Network issues, invalid webhook setup, or Telegram API downtime.  

---

#### 1.2 Message Type Branching

- **Overview:** Determines if the received Telegram message is text or voice to route to the appropriate processing pipeline.  
- **Nodes Involved:**  
  - Switch: Text or Voice  
  - Sticky Note (description node)

- **Node Details:**  
  - **Switch: Text or Voice**  
    - Type: Switch  
    - Role: Routes messages based on presence of `message.text`.  
    - Configuration:  
      - Condition 1: Exists `message.text` → output key: "text"  
      - Condition 2: No `message.text` (implied voice) → output key: "voice"  
    - Inputs: Telegram message JSON  
    - Outputs: Either routes to text processing or voice processing branches.  
    - Edge Cases: Messages without text or voice, or unexpected message formats.  

---

#### 1.3 Voice Processing Chain

- **Overview:** Downloads voice message file from Telegram and uses Google Gemini to transcribe it to text.  
- **Nodes Involved:**  
  - Telegram: Download Voice File  
  - Gemini: Transcribe Voice  
  - Set: Put Transcript into Text  
  - Sticky Note (description node)

- **Node Details:**  
  - **Telegram: Download Voice File**  
    - Type: Telegram node  
    - Role: Downloads voice file binary using `file_id` from Telegram.  
    - Configuration: Uses `message.voice.file_id` to fetch file.  
    - Inputs: Voice message JSON from Switch output.  
    - Outputs: Binary audio file.  
    - Credentials: Telegram API.  
    - Edge Cases: Invalid file ID, file not found, download failures.  

  - **Gemini: Transcribe Voice**  
    - Type: Google Gemini (LangChain node)  
    - Role: Transcribes the binary audio file to text using Gemini 2.5 Flash model.  
    - Configuration: Input type set to binary, model `models/gemini-2.5-flash`.  
    - Inputs: Binary audio from previous node.  
    - Outputs: Transcription result JSON with text content.  
    - Credentials: Google Palm API (Gemini).  
    - Edge Cases: Transcription failures, unsupported audio format, API quota exhaustion.  

  - **Set: Put Transcript into Text**  
    - Type: Set node  
    - Role: Extracts the transcribed text from Gemini output and stores it as a new field `text`.  
    - Configuration: Expression to assign `text = $json.content.parts[0].text`.  
    - Inputs: Gemini transcription JSON.  
    - Outputs: JSON with `text` property ready for AI extraction.  
    - Edge Cases: Empty transcription, malformed Gemini response.  

---

#### 1.4 Text Preparation

- **Overview:** Extracts text directly from Telegram text messages for AI processing.  
- **Nodes Involved:**  
  - Set: Prepare Text  
  - Sticky Note (description node)

- **Node Details:**  
  - **Set: Prepare Text**  
    - Type: Set node  
    - Role: Copies `message.text` from Telegram message JSON into a `text` field for downstream use.  
    - Configuration: Assign `text = $json.message.text`.  
    - Inputs: Telegram message JSON from Switch output (text branch).  
    - Outputs: JSON with `text` property.  
    - Edge Cases: Empty text messages.  

---

#### 1.5 AI Extraction

- **Overview:** Uses Google Gemini Chat Model and LangChain information extractor to parse `TaskName` and `TaskDue` date from text.  
- **Nodes Involved:**  
  - Google Gemini Chat Model  
  - AI Extractor: TaskName & TaskDue  
  - Sticky Note (description node)

- **Node Details:**  
  - **Google Gemini Chat Model**  
    - Type: LangChain Google Gemini Chat Model node  
    - Role: Provides AI language model for downstream extractor.  
    - Configuration: Model `models/gemini-2.5-flash-lite`.  
    - Inputs: None (used as AI model resource node).  
    - Outputs: Connected as AI model input to extractor.  
    - Credentials: Google Palm API.  
    - Edge Cases: API authentication, model unavailability.  

  - **AI Extractor: TaskName & TaskDue**  
    - Type: LangChain Information Extractor  
    - Role: Extracts structured attributes from freeform text.  
    - Configuration:  
      - Input text: `{{$json.text}}`  
      - Attributes:  
        - TaskName (string, optional)  
        - TaskDue (date, optional)  
    - Inputs: JSON with `text` field.  
    - Outputs: JSON with `output.TaskName` and `output.TaskDue`.  
    - AI Model Input: Receives AI model from "Google Gemini Chat Model".  
    - Edge Cases: Inaccurate extraction, ambiguous dates, empty or irrelevant text.  

---

#### 1.6 Extraction Validation

- **Overview:** Validates that both task name and due date were successfully extracted before proceeding.  
- **Nodes Involved:**  
  - If: Extraction Valid?  
  - Telegram: Notify - Extraction Failed  
  - Sticky Note (description node)

- **Node Details:**  
  - **If: Extraction Valid?**  
    - Type: If node  
    - Role: Checks existence of `output.TaskName` and `output.TaskDue` fields.  
    - Configuration:  
      - Condition: String exists for TaskName AND datetime exists for TaskDue.  
    - Inputs: Output from AI Extractor node.  
    - Outputs:  
      - True: Proceed to approval step  
      - False: Notify extraction failure  
    - Edge Cases: Partial extraction, malformed date formats.  

  - **Telegram: Notify - Extraction Failed**  
    - Type: Telegram node (send message)  
    - Role: Notifies user that task extraction failed.  
    - Configuration: Message: "Title or due date cannot be extracted. Please try again."  
    - Inputs: False branch of validation node.  
    - Credentials: Telegram API.  
    - Edge Cases: Message delivery failure, invalid chat ID.  

---

#### 1.7 User Approval

- **Overview:** Sends a Telegram message to the user displaying the extracted task details with Approve and Decline buttons.  
- **Nodes Involved:**  
  - Telegram: Ask Approve / Decline  
  - Sticky Note (description node)

- **Node Details:**  
  - **Telegram: Ask Approve / Decline**  
    - Type: Telegram node (sendAndWait with approval options)  
    - Role: Requests user confirmation for the extracted task.  
    - Configuration:  
      - Chat ID sourced dynamically from the original Telegram message.  
      - Message includes TaskName and TaskDue formatted as a date string.  
      - Approval type: double approval with "Approve" and "Decline" buttons.  
    - Inputs: Validated extraction data.  
    - Outputs: User response to approval prompt.  
    - Credentials: Telegram API.  
    - Edge Cases: User ignores or delays response, Telegram API downtime.  

---

#### 1.8 Approval Check

- **Overview:** Processes the user's approval response to either create the task in Notion or notify the user that the task was not created.  
- **Nodes Involved:**  
  - Approval Check (If Approved?)  
  - Telegram: Notify - Task Not Created  
  - Sticky Note (description node)

- **Node Details:**  
  - **Approval Check (If Approved?)**  
    - Type: If node  
    - Role: Checks if user approved (boolean `data.approved` is true).  
    - Inputs: User response from approval node.  
    - Outputs:  
      - True: Proceed to Notion task creation.  
      - False: Notify user task was not created.  
    - Edge Cases: Missing or malformed approval data.  

  - **Telegram: Notify - Task Not Created**  
    - Type: Telegram node (send message)  
    - Role: Notifies user the task was declined.  
    - Configuration: Message: "❌ Task not created in Notion."  
    - Inputs: False branch from approval check.  
    - Credentials: Telegram API.  
    - Edge Cases: Message delivery issues.  

---

#### 1.9 Notion Task Creation

- **Overview:** Creates a new task page in a Notion database using the extracted task name and due date, then notifies the user of success.  
- **Nodes Involved:**  
  - Notion: Create Task Page  
  - Telegram: Notify - Task Created  
  - Sticky Note (description node)

- **Node Details:**  
  - **Notion: Create Task Page**  
    - Type: Notion node  
    - Role: Creates a database page with mapped properties.  
    - Configuration:  
      - Database ID: User must replace `"YOUR_DATABASE_ID_HERE"` with actual ID.  
      - Title property mapped to extracted `TaskName`.  
      - Date property mapped to extracted `TaskDue` (without time).  
    - Inputs: Data from approval check (approved tasks).  
    - Credentials: Notion API integration token.  
    - Edge Cases: Invalid database ID, API quota limits, permission issues.  

  - **Telegram: Notify - Task Created**  
    - Type: Telegram node (send message)  
    - Role: Sends confirmation message to user upon successful task creation.  
    - Configuration: Message: "✅ Task created in Notion."  
    - Inputs: Output of Notion node.  
    - Credentials: Telegram API.  
    - Edge Cases: Delivery failure or user unreachable.  

---

### 3. Summary Table

| Node Name                         | Node Type                                  | Functional Role                               | Input Node(s)                   | Output Node(s)                       | Sticky Note                                                                                          |
|----------------------------------|--------------------------------------------|----------------------------------------------|--------------------------------|------------------------------------|----------------------------------------------------------------------------------------------------|
| Telegram: Receive Message         | Telegram Trigger                           | Trigger workflow on new Telegram message     | -                              | Switch: Text or Voice              | ## 💬 Telegram: Receive Message  Triggers when a new Telegram message arrives either text or voice.|
| Switch: Text or Voice             | Switch                                    | Routes message based on type (text or voice) | Telegram: Receive Message       | Set: Prepare Text, Telegram: Download Voice File | ## 🔀 Switch: Text or Voice  Checks if the message is text or voice and routes it to matching branch. |
| Set: Prepare Text                 | Set                                       | Extracts text from Telegram message           | Switch: Text or Voice           | AI Extractor: TaskName & TaskDue   | ## ✏️ Set: Prepare Text  Extracts the text message from Telegram and stores it for later processing. |
| Telegram: Download Voice File     | Telegram node                             | Downloads voice file from Telegram            | Switch: Text or Voice           | Gemini: Transcribe Voice           | ## 🎙️ Voice Processing Chain  1. Download voice file 2. Transcribe with Gemini 3. Store transcript  |
| Gemini: Transcribe Voice          | Google Gemini (LangChain)                  | Transcribes voice audio to text                | Telegram: Download Voice File   | Set: Put Transcript into Text      | (See above)                                                                                         |
| Set: Put Transcript into Text    | Set                                       | Saves transcribed text to `text` field         | Gemini: Transcribe Voice        | AI Extractor: TaskName & TaskDue   | (See above)                                                                                         |
| Google Gemini Chat Model          | LangChain Google Gemini Chat Model         | AI model resource for extraction               | -                              | AI Extractor: TaskName & TaskDue   | ## 🧠 Task Information Extraction  Powers the AI extraction with Gemini 2.5 Flash Lite.            |
| AI Extractor: TaskName & TaskDue | LangChain Information Extractor            | Extracts TaskName and TaskDue from text        | Set: Prepare Text / Set: Put Transcript into Text | If: Extraction Valid?            | (See above)                                                                                         |
| If: Extraction Valid?             | If node                                   | Validates extracted TaskName and TaskDue      | AI Extractor: TaskName & TaskDue | Telegram: Ask Approve / Decline, Telegram: Notify - Extraction Failed | ## ✅ Validate Task Extraction  Checks if TaskName and TaskDue exist; notifies if extraction failed. |
| Telegram: Notify - Extraction Failed | Telegram node (send message)              | Notifies user extraction failed                | If: Extraction Valid? (false)   | -                                  | (See above)                                                                                        |
| Telegram: Ask Approve / Decline  | Telegram node (sendAndWait approval)       | Sends approval request with Approve/Decline buttons | If: Extraction Valid? (true)    | Approval Check (If Approved?)       | ## 📩 Ask for Task Approval  Sends extracted task details for user approval via Telegram.          |
| Approval Check (If Approved?)    | If node                                   | Checks if user approved the task                | Telegram: Ask Approve / Decline | Notion: Create Task Page, Telegram: Notify - Task Not Created | ## ✅ Check Task Approval  Checks if user approved; routes accordingly.                            |
| Telegram: Notify - Task Not Created | Telegram node (send message)              | Notifies user that task was declined            | Approval Check (If Approved?) (false) | -                                  | (See above)                                                                                        |
| Notion: Create Task Page         | Notion node                               | Creates task page in Notion with extracted data | Approval Check (If Approved?) (true) | Telegram: Notify - Task Created    | ## 📝 Create Task in Notion  Creates task in Notion and confirms to user.                          |
| Telegram: Notify - Task Created  | Telegram node (send message)               | Notifies user that task was created successfully | Notion: Create Task Page        | -                                  | (See above)                                                                                        |
| Sticky Note                      | Sticky Note                               | Descriptive notes for workflow clarity          | -                              | -                                  | Various notes throughout the workflow explaining each block and node.                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node:**  
   - Node Type: Telegram Trigger  
   - Configure webhook for updates: select "message" update type.  
   - Set Telegram API credentials with Bot Token.  
   - This node starts the workflow on new incoming Telegram messages.

2. **Add Switch Node to Branch Message Type:**  
   - Node Type: Switch  
   - Add two conditions:  
     - Condition 1: Check if `$json.message.text` exists (output key "text")  
     - Condition 2: Else (output key "voice")  
   - Connect Telegram Trigger output to this Switch node.

3. **Text Branch: Prepare Text Node:**  
   - Node Type: Set  
   - Add assignment: `text = {{$json.message.text}}`  
   - Connect Switch "text" output to this node.

4. **Voice Branch: Download Voice File Node:**  
   - Node Type: Telegram node (Get File)  
   - Resource: File  
   - File ID: `{{$json.message.voice.file_id}}`  
   - Download file enabled (to get binary).  
   - Connect Switch "voice" output to this node.

5. **Voice Branch: Gemini Transcribe Voice Node:**  
   - Node Type: LangChain Google Gemini Node  
   - Model ID: `models/gemini-2.5-flash` (latest fluent transcription model)  
   - Input type: Binary  
   - Connect output of Download Voice File node to this node.

6. **Voice Branch: Set Transcript into Text Node:**  
   - Node Type: Set  
   - Assign `text = {{$json.content.parts[0].text}}` (extract from Gemini transcription)  
   - Connect Gemini Transcribe Voice node output here.

7. **Merge Text and Voice Branches:**  
   - Connect both "Set: Prepare Text" node output and "Set: Put Transcript into Text" node output to a single AI extractor node.

8. **Add Google Gemini Chat Model Node:**  
   - Node Type: LangChain Google Gemini Chat Model  
   - Model: `models/gemini-2.5-flash-lite`  
   - Set up Google Palm API credentials.  
   - This node serves as the AI model resource for the extractor.

9. **Add AI Extractor Node:**  
   - Node Type: LangChain Information Extractor  
   - Input text: `{{$json.text}}`  
   - Attributes:  
     - TaskName (string, optional)  
     - TaskDue (date, optional)  
   - Connect "Google Gemini Chat Model" as AI model input to this node.  
   - Connect output of prepared text nodes to this node.

10. **Add If Node to Validate Extraction:**  
    - Conditions:  
      - TaskName exists (string)  
      - TaskDue exists (date)  
    - Connect AI Extractor output to this node.

11. **Add Telegram Notify Node for Extraction Failure:**  
    - Type: Telegram node (send message)  
    - Message: "Title or due date cannot be extracted. Please try again."  
    - Chat ID: `{{$json.message.from.id}}` from original Telegram message.  
    - Connect false branch of validation If node here.

12. **Add Telegram Ask Approve/Decline Node:**  
    - Type: Telegram node (sendAndWait with approval buttons)  
    - Chat ID: `{{$json.message.chat.id}}` from original message.  
    - Message:  
      ```
      Task Name: {{$json.output.TaskName}}
      Due Date: {{ new Date($json.output.TaskDue).toLocaleDateString('en-US', { year: 'numeric', month: 'short', day: 'numeric' }) }}
      ```  
    - Approval options: double approval with labels "Approve" and "Decline".  
    - Connect true branch of validation If node here.

13. **Add If Node to Check User Approval:**  
    - Condition: `$json.data.approved` equals true (boolean)  
    - Connect output of Telegram Ask Approve/Decline node to this node.

14. **Add Telegram Notify Node for Declined Task:**  
    - Type: Telegram node (send message)  
    - Message: "❌ Task not created in Notion."  
    - Chat ID: `{{$json.message.from.id}}` from original message.  
    - Connect false branch of approval check here.

15. **Add Notion Create Task Page Node:**  
    - Type: Notion node  
    - Resource: Database Page  
    - Database ID: Replace `"YOUR_DATABASE_ID_HERE"` with actual Notion database ID.  
    - Map properties:  
      - Title → `{{$json.output.TaskName}}`  
      - Date → `{{$json.output.TaskDue}}` (no time included)  
    - Credentials: Notion Integration Token.  
    - Connect true branch of approval check here.

16. **Add Telegram Notify Node for Task Created:**  
    - Type: Telegram node (send message)  
    - Message: "✅ Task created in Notion."  
    - Chat ID: `{{$json.message.from.id}}` from original message.  
    - Connect output of Notion Create Task Page node here.

17. **Final Setup:**  
    - Confirm all Telegram, Google Gemini (Google Palm), and Notion credentials are configured in n8n.  
    - Replace placeholders (e.g., Notion Database ID).  
    - Test with text and voice inputs to ensure proper flow.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Context or Link                                                                                                                               |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------|
| Quick Setup Checklist for Telegram → Transcribe → Extract → Notion covering: Telegram Bot creation, obtaining chat ID, Google Gemini API setup, Notion Integration and database setup, mapping extracted fields to Notion properties, audio transcription notes, required credentials, and testing instructions. Includes tips for troubleshooting date mapping and transcription binary input.                                                                                               | Present as Sticky Note at the workflow’s start; highly recommended for initial setup.                                                        |
| Workflow uses Google Gemini models for both chat-based text extraction (`gemini-2.5-flash-lite`) and audio transcription (`gemini-2.5-flash`). Credentials require Google Cloud project with GenAI/PaLM API enabled.                                                                                                                                                                                                                                                         | Google Gemini API info: https://developers.generativeai.google/                                                                              |
| Telegram Bot API requires Bot Token from BotFather and enables webhook-based message reception. Chat IDs can be extracted by sending a message and inspecting the payload in the Telegram Trigger node.                                                                                                                                                                                                                                                              | Telegram Bot API docs: https://core.telegram.org/bots/api                                                                                     |
| Notion Integration requires creating an integration token, sharing the target database with the integration, and retrieving the database ID from the URL. Database must have at least Title and Date properties configured.                                                                                                                                                                                                                                         | Notion API docs: https://developers.notion.com/docs                                                                                          |
| Voice notes are processed entirely via Telegram file download and Google Gemini transcription, no external audio codecs or converters needed. Ensure Telegram node downloads file binary to pass to Gemini node.                                                                                                                                                                                                                                                        | Gemini handles transcription formats automatically.                                                                                          |
| Approval uses Telegram's "send and wait" with inline buttons for user confirmation, enabling a smooth two-step approval before task creation.                                                                                                                                                                                                                                                                                                                   |                                                                                                |

---

**Disclaimer:** The provided text is the output of an automated workflow created using n8n, adhering strictly to current content policies and containing no illegal, offensive, or proprietary information. All data handled is legal and publicly accessible.