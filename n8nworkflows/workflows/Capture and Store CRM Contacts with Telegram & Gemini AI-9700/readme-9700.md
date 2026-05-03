Capture and Store CRM Contacts with Telegram & Gemini AI

https://n8nworkflows.xyz/workflows/capture-and-store-crm-contacts-with-telegram---gemini-ai-9700


# Capture and Store CRM Contacts with Telegram & Gemini AI

### 1. Workflow Overview

This workflow implements a Telegram bot designed to capture and store CRM contact information using AI-powered multi-modal data extraction. It supports inputs via text messages, voice recordings, and images (e.g., business cards). The bot conducts a natural language conversation to collect contact details, confirms extracted data with the user, and manages contacts in a Google Sheets-based CRM by searching, creating, or updating entries.

The workflow is logically organized into the following blocks:

- **1.1 Input Reception & Session Management**: Receives Telegram messages, manages conversation sessions using session IDs stored in a data table, and handles new session initialization.
- **1.2 Input Type Processing**: Differentiates between text, audio, and image inputs and processes each accordingly (transcription, image analysis, or direct text).
- **1.3 AI Processing with Google Gemini & Memory**: Uses Google Gemini AI models for natural language understanding, transcription, and image analysis; maintains conversation context with memory buffers.
- **1.4 CRM Contact Management**: Searches for existing contacts, creates new contacts, or updates existing contacts in Google Sheets based on AI-extracted data.
- **1.5 Telegram Response Handling**: Sends messages and typing indicators back to the user on Telegram to maintain natural interaction.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Session Management

- **Overview:**  
  Handles incoming Telegram updates, manages user session IDs to track conversation states, and supports session reset via the `/new` command.

- **Nodes Involved:**  
  Telegram Trigger, Send "typing...", parameters, Get sessionID, If no sessionID or /new, Generate sessionID, Upsert row(s), Add sessionID to input, Sticky Note1, Sticky Note3

- **Node Details:**

  - **Telegram Trigger**  
    - *Type:* Telegram Trigger  
    - *Role:* Entry point for Telegram messages (text, voice, photos). Configured to download images at large size.  
    - *Input:* Incoming Telegram webhook events  
    - *Output:* Message JSON with content and metadata  
    - *Credentials:* Telegram bot API token  
    - *Potential Failures:* Webhook misconfiguration, invalid Telegram token, rate limits  

  - **Send "typing..."**  
    - *Type:* Telegram node (sendChatAction)  
    - *Role:* Sends "typing" indicator to Telegram chat to improve UX during processing  
    - *Input:* Chat ID from Telegram Trigger  
    - *Output:* None (side effect)  
    - *Credentials:* Telegram bot API token  
    - *Potential Failures:* Telegram API limits or connectivity issues  

  - **parameters**  
    - *Type:* Set node  
    - *Role:* Holds user-configurable parameters: Google Sheets document ID and sheet name  
    - *Input:* Telegram Trigger JSON  
    - *Output:* JSON with parameters for downstream nodes  
    - *Configuration:* Spreadsheet document ID (empty by default), sheet name default "Sheet1"  
    - *Edge Cases:* Missing or invalid spreadsheet ID leads to failed Google Sheets operations  

  - **Get sessionID**  
    - *Type:* Data Table node (get operation)  
    - *Role:* Retrieves stored session ID for the current chat from a data table for conversation continuity  
    - *Input:* Chat ID from Telegram message  
    - *Output:* Session ID if exists  
    - *Edge Cases:* No session found, empty result leads to new session creation  

  - **If no sessionID or /new**  
    - *Type:* If node  
    - *Role:* Checks if session ID is missing or user sent `/new` command to start fresh conversation  
    - *Input:* Output of Get sessionID and Telegram message text  
    - *Output:* Two branches: true (no session or new command), false (continue existing session)  
    - *Edge Cases:* Expression errors if input JSON malformed  

  - **Generate sessionID**  
    - *Type:* Crypto node (generate)  
    - *Role:* Generates a new unique session ID when needed  
    - *Input:* Trigger from If node true branch  
    - *Output:* New session ID string  
    - *Edge Cases:* Rare failures in UUID generation are possible  

  - **Upsert row(s)**  
    - *Type:* Data Table node (upsert operation)  
    - *Role:* Stores or updates session ID alongside chat ID in data table to maintain session persistence  
    - *Input:* Session ID from Generate sessionID node, chat ID from Telegram  
    - *Output:* Confirmation of upserted row  
    - *Edge Cases:* Data Table connectivity or permission failures  

  - **Add sessionID to input**  
    - *Type:* Merge node (combine)  
    - *Role:* Combines session ID with incoming message data to pass complete context downstream  
    - *Input:* Session ID and Telegram message data  
    - *Output:* Single JSON object with session ID and message content  
    - *Edge Cases:* Merge failures if data missing  

  - **Sticky Notes 1 & 3**  
    - Provide user instructions about session management and parameter configuration  

---

#### 1.2 Input Type Processing

- **Overview:**  
  Distinguishes message type (text, audio, image), triggering appropriate processing paths: transcribe audio, analyze image, or handle text directly.

- **Nodes Involved:**  
  Switch, Get audio file, Transcribe audio, Process audio transcript, Analyze image, Process extracted data, Process text input, Sticky Note2

- **Node Details:**

  - **Switch**  
    - *Type:* Switch node  
    - *Role:* Routes incoming messages based on content: audio voice message, photo, or text  
    - *Input:* Merged sessionID + Telegram message JSON  
    - *Output:* One of three outputs: audio input, image input, text input  
    - *Edge Cases:* Messages with no recognized type go to fallback (extra), which is unconnected and ignored  

  - **Get audio file**  
    - *Type:* Telegram node  
    - *Role:* Downloads voice message file from Telegram for transcription  
    - *Input:* File ID from Telegram voice message  
    - *Output:* Binary audio file data  
    - *Credentials:* Telegram API  
    - *Edge Cases:* Missing or expired file links, Telegram API errors  

  - **Transcribe audio**  
    - *Type:* Google Gemini AI (audio transcription)  
    - *Role:* Converts audio binary to text transcript using Gemini model "gemini-2.5-flash"  
    - *Input:* Binary audio file  
    - *Output:* JSON with transcribed text  
    - *Credentials:* Google Gemini (Google PaLM API key)  
    - *Edge Cases:* Audio quality issues, API rate limits, transcription errors  

  - **Process audio transcript**  
    - *Type:* Code node (JavaScript)  
    - *Role:* Wraps transcribed text in XML-like tags `<transcribed_audio>` for AI context  
    - *Input:* Transcription result JSON  
    - *Output:* JSON with userMessage property containing marked transcript  
    - *Edge Cases:* Malformed transcription JSON  

  - **Analyze image**  
    - *Type:* Google Gemini AI (image analysis)  
    - *Role:* Extracts text and contact info from image binary using Gemini model  
    - *Input:* Binary image data from Telegram photo  
    - *Output:* AI response JSON with extracted text content  
    - *Credentials:* Google Gemini  
    - *Edge Cases:* Poor image quality, OCR failures, API limits  

  - **Process extracted data**  
    - *Type:* Code node  
    - *Role:* Wraps extracted image content in XML tags `<extracted_image_content>`, appends photo caption if present  
    - *Input:* Gemini AI response and Telegram message caption  
    - *Output:* JSON with userMessage property containing combined text for AI agent  
    - *Edge Cases:* Missing caption, malformed AI response  

  - **Process text input**  
    - *Type:* Code node  
    - *Role:* Passes text message content directly as userMessage for AI agent  
    - *Input:* Telegram text message JSON  
    - *Output:* userMessage string  
    - *Edge Cases:* Empty or malformed text messages  

  - **Sticky Note2**  
    - Describes the multi-modal message processing strategy  

---

#### 1.3 AI Processing with Google Gemini & Memory

- **Overview:**  
  Uses Google Gemini chat model integrated via LangChain agent node to interpret userMessage input, extract contact details, maintain conversation context with session memory, and generate confirmation prompts or CRM update instructions.

- **Nodes Involved:**  
  AI Agent, Google Gemini Chat Model, Simple Memory, Sticky Note4

- **Node Details:**

  - **AI Agent**  
    - *Type:* LangChain Agent node  
    - *Role:* Central AI processing unit interpreting userMessage, managing dialog, extracting and confirming contact info, invoking CRM operations as tools  
    - *Input:* userMessage strings containing text or annotated audio/image content, session memory context  
    - *Output:* Text output for Telegram reply and calls to AI tools (Search/Create/Update contact)  
    - *Configuration:*  
      - System message defines role as concise personal assistant for CRM data capture.  
      - Mandatory extraction fields: first name, last name, email, company. Optional: phone, job title, meeting notes.  
      - Up to 3 clarifying questions per contact.  
      - Handles `/new` command by resetting conversation.  
      - Prepares phone numbers starting with "+" by prefixing with single quote for CRM compatibility.  
    - *Version:* 2.2  
    - *Potential Failures:* API limits, malformed input, model errors, conversation state loss, session key mismatches  

  - **Google Gemini Chat Model**  
    - *Type:* LangChain Google Gemini LM Chat node  
    - *Role:* Provides the underlying language model for AI Agent's chat processing  
    - *Input:* Passed from AI Agent as language model backend  
    - *Credentials:* Google PaLM API key  
    - *Edge Cases:* API key invalid, quota exceeded  

  - **Simple Memory**  
    - *Type:* LangChain Memory Buffer (windowed)  
    - *Role:* Maintains conversational context limited to latest 10 exchanges keyed by sessionID  
    - *Input:* sessionID and AI Agent interactions  
    - *Output:* Context to AI Agent  
    - *Edge Cases:* Memory loss on node failure, sessionID mismatch causes context loss  

  - **Sticky Note4**  
    - Notes the AI Agent as the core logic  

---

#### 1.4 CRM Contact Management

- **Overview:**  
  Implements CRM operations using Google Sheets as a backend: searches for existing contacts by email, creates new entries, or updates existing ones with AI-extracted data.

- **Nodes Involved:**  
  Search for contact, Create new contact, Update existing contact

- **Node Details:**

  - **Search for contact**  
    - *Type:* Google Sheets Tool node  
    - *Role:* Looks up an existing contact by email in Google Sheets  
    - *Input:* Email extracted by AI Agent, sheet and document IDs from parameters  
    - *Output:* Search results for contact row(s)  
    - *Credentials:* Google Sheets service account  
    - *Edge Cases:* Spreadsheet access errors, no matching contact  

  - **Create new contact**  
    - *Type:* Google Sheets Tool node  
    - *Role:* Appends a new contact row with extracted data into Google Sheets  
    - *Input:* AI Agent output fields (Full name, Email, Phone, Company, Job title, Meeting notes)  
    - *Output:* Confirmation of row append  
    - *Credentials:* Google Sheets service account  
    - *Edge Cases:* Data validation errors, duplicate emails, API limits  

  - **Update existing contact**  
    - *Type:* Google Sheets Tool node  
    - *Role:* Updates an existing contact row by matching email, merging new data  
    - *Input:* AI Agent output fields, sheet/document IDs  
    - *Output:* Confirmation of update  
    - *Credentials:* Google Sheets service account  
    - *Edge Cases:* Concurrent edits, missing email match, permission errors  

---

#### 1.5 Telegram Response Handling

- **Overview:**  
  Sends AI Agent's response text back to the user via Telegram, completing the conversational loop.

- **Nodes Involved:**  
  Send a text message

- **Node Details:**

  - **Send a text message**  
    - *Type:* Telegram node (send message)  
    - *Role:* Sends text output generated by AI Agent back to the Telegram chat  
    - *Input:* Text output from AI Agent node, chat ID from Telegram Trigger  
    - *Credentials:* Telegram bot API token  
    - *Edge Cases:* Message length limits, network errors, Telegram API failures  

---

### 3. Summary Table

| Node Name            | Node Type                      | Functional Role                              | Input Node(s)                    | Output Node(s)                | Sticky Note                                  |
|----------------------|--------------------------------|---------------------------------------------|---------------------------------|------------------------------|----------------------------------------------|
| Telegram Trigger      | Telegram Trigger               | Entry point for Telegram updates             | -                               | parameters, Send "typing..."  |                                              |
| Send "typing..."      | Telegram                      | Sends typing indicator to Telegram chat      | Telegram Trigger                | -                            |                                              |
| parameters           | Set                           | Stores Google Sheets config parameters       | Telegram Trigger                | Add sessionID to input        | ## Configure ➤ set spreadsheet document ID ➤ set sheet name |
| Get sessionID         | Data Table (get)              | Retrieves existing sessionID for chat        | parameters                     | If no sessionID or /new       | ## Get existing sessionID or create a new one sessionID is used to store conversation history |
| If no sessionID or /new | If                           | Checks if sessionID missing or `/new` sent   | Get sessionID                  | Generate sessionID or Add sessionID to input | ## Get existing sessionID or create a new one sessionID is used to store conversation history |
| Generate sessionID    | Crypto (generate)             | Generates new sessionID                       | If no sessionID or /new        | Upsert row(s)                | ## Get existing sessionID or create a new one sessionID is used to store conversation history |
| Upsert row(s)         | Data Table (upsert)           | Stores sessionID associated with chat        | Generate sessionID             | Add sessionID to input        | ## Get existing sessionID or create a new one sessionID is used to store conversation history |
| Add sessionID to input| Merge (combine)               | Combines sessionID with incoming message     | Upsert row(s), parameters      | Switch                      |                                              |
| Switch                | Switch                       | Routes by input type: audio, image, or text | Add sessionID to input          | Get audio file, Analyze image, Process text input | ## Process various types of user messages - Transcribe audio - Analyse image - Prepare text message |
| Get audio file        | Telegram                      | Downloads voice message audio file           | Switch (audio input)            | Transcribe audio             |                                              |
| Transcribe audio      | Google Gemini (audio)         | Transcribes audio to text                     | Get audio file                 | Process audio transcript      |                                              |
| Process audio transcript | Code                        | Wraps transcript text for AI context         | Transcribe audio               | AI Agent                    |                                              |
| Analyze image         | Google Gemini (image)         | Extracts contact info from images             | Switch (image input)            | Process extracted data        |                                              |
| Process extracted data| Code                         | Prepares extracted image text for AI          | Analyze image                  | AI Agent                    |                                              |
| Process text input    | Code                         | Passes text messages as userMessage           | Switch (text input)             | AI Agent                    |                                              |
| AI Agent              | LangChain Agent              | Central AI for extracting and confirming data| Process audio transcript, Process extracted data, Process text input, Simple Memory | Send a text message          | ## AI Agent                                   |
| Google Gemini Chat Model | LangChain LM Chat           | Language model backend for AI Agent           | AI Agent (ai_languageModel)    | AI Agent                    | ## AI Agent                                   |
| Simple Memory         | LangChain Memory Buffer       | Maintains conversation context                 | Add sessionID to input         | AI Agent (ai_memory)         | ## AI Agent                                   |
| Search for contact    | Google Sheets Tool            | Searches CRM for contact by email              | AI Agent (ai_tool)             | AI Agent (ai_tool)           |                                              |
| Create new contact    | Google Sheets Tool            | Appends new contact to CRM                      | AI Agent (ai_tool)             | AI Agent (ai_tool)           |                                              |
| Update existing contact | Google Sheets Tool          | Updates existing CRM contact                     | AI Agent (ai_tool)             | AI Agent (ai_tool)           |                                              |
| Send a text message   | Telegram                      | Sends AI Agent response text back to user      | AI Agent                      | -                            |                                              |
| Sticky Note           | Sticky Note                  | Documentation notes                             | -                             | -                            | ## Telegram bot for capturing contacts ... (full content in node) |
| Sticky Note1          | Sticky Note                  | Documentation notes                             | -                             | -                            | ## Get existing sessionID or create a new one sessionID is used to store conversation history |
| Sticky Note2          | Sticky Note                  | Documentation notes                             | -                             | -                            | ## Process various types of user messages ... |
| Sticky Note3          | Sticky Note                  | Documentation notes                             | -                             | -                            | ## Configure ➤ set spreadsheet document ID ➤ set sheet name |
| Sticky Note4          | Sticky Note                  | Documentation notes                             | -                             | -                            | ## AI Agent                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: Telegram Trigger  
   - Configure to listen for "message" updates including images with "large" size download option  
   - Supply Telegram bot API credentials with your bot token  
   - Note webhook URL for Telegram bot webhook setup  

2. **Create "Send typing..." node**  
   - Type: Telegram  
   - Operation: sendChatAction  
   - Chat ID: from Telegram Trigger message.chat.id  
   - Credentials: Telegram bot API  

3. **Create parameters node**  
   - Type: Set  
   - Define two parameters:  
     - `spreadsheet_document_id` (string) — your Google Sheets document ID  
     - `sheet_name` (string) — default "Sheet1"  
   - Connect Telegram Trigger main output to parameters input  

4. **Create Get sessionID node**  
   - Type: Data Table (get operation)  
   - Filter by chatID equals Telegram message.chat.id  
   - Connect parameters output to this node  

5. **Create If no sessionID or /new node**  
   - Type: If  
   - Conditions (OR):  
     - Input JSON empty  
     - sessionID not exists or empty  
     - Telegram message.text starts with "/new"  
   - Connect Get sessionID output to this node  

6. **Create Generate sessionID node**  
   - Type: Crypto (generate)  
   - Action: generate UUID string  
   - Connect If node "true" output to this node  

7. **Create Upsert row(s) node**  
   - Type: Data Table (upsert operation)  
   - Columns: chatID (number), sessionID (string)  
   - Match on sessionID  
   - Filter by chatID = Telegram chat.id  
   - Connect Generate sessionID output to this node  

8. **Create Add sessionID to input node**  
   - Type: Merge (combine)  
   - Mode: combine by position  
   - Connect Upsert row(s) output (or If node "false" output) and parameters output to this node  

9. **Create Switch node**  
   - Type: Switch  
   - Rules:  
     - Audio input if message.voice exists  
     - Image input if message.photo array exists  
     - Text input if message.text exists  
   - Connect Add sessionID to input output to Switch input  

10. **Create Get audio file node**  
    - Type: Telegram  
    - Operation: file  
    - File ID: message.voice.file_id  
    - Connect Switch "audio input" output to this node  

11. **Create Transcribe audio node**  
    - Type: Google Gemini (audio)  
    - Model: models/gemini-2.5-flash  
    - Input type: binary audio from previous node  
    - Connect Get audio file output to this node  

12. **Create Process audio transcript node**  
    - Type: Code  
    - JS code: wrap transcript text in `<transcribed_audio>` tags and output userMessage  
    - Connect Transcribe audio output to this node  

13. **Create Analyze image node**  
    - Type: Google Gemini (image)  
    - Model: models/gemini-2.5-flash  
    - Prompt: Extract contact info (first name, last name, email, phone, company, job title)  
    - Input type: binary image from Telegram photo  
    - Connect Switch "image input" output to this node  

14. **Create Process extracted data node**  
    - Type: Code  
    - JS code: wrap extracted text in `<extracted_image_content>` tags, append photo caption if exists  
    - Connect Analyze image output to this node  

15. **Create Process text input node**  
    - Type: Code  
    - JS code: output message.text as userMessage  
    - Connect Switch "text input" output to this node  

16. **Create AI Agent node**  
    - Type: LangChain Agent  
    - Input: userMessage from audio, image, or text processing nodes  
    - System message: as specified, instructing to extract contact data and manage CRM entries  
    - Configure AI tools: Google Sheets nodes for Search/Create/Update contact  
    - Set session key from sessionID for memory context  
    - Connect processed input nodes outputs (Process audio transcript, Process extracted data, Process text input) to AI Agent input  
    - Connect Simple Memory node to AI Agent for context maintenance  

17. **Create Google Gemini Chat Model node**  
    - Type: LangChain LM Chat Google Gemini  
    - Model: models/gemini-2.5-flash  
    - Credentials: Google Palm API key  
    - Connect to AI Agent as language model  

18. **Create Simple Memory node**  
    - Type: LangChain Memory Buffer Window  
    - Session key: sessionID from Add sessionID to input  
    - Context window length: 10  
    - Connect to AI Agent as memory input/output  

19. **Create Search for contact node**  
    - Type: Google Sheets Tool (search)  
    - Filter: by Email field equal to AI Agent extracted email  
    - Connect AI Agent as AI tool input/output  

20. **Create Create new contact node**  
    - Type: Google Sheets Tool (append)  
    - Columns: Full name, Email, Phone, Company, Job title, Meeting notes from AI Agent variables  
    - Connect AI Agent as AI tool input/output  

21. **Create Update existing contact node**  
    - Type: Google Sheets Tool (appendOrUpdate)  
    - Columns: same as Create new contact  
    - Matching column: Email  
    - Connect AI Agent as AI tool input/output  

22. **Create Send a text message node**  
    - Type: Telegram  
    - Text: AI Agent output  
    - Chat ID: Telegram message.chat.id  
    - Connect AI Agent main output to this node  

23. **Optional: Add Sticky Notes**  
    - Add documentation sticky notes at relevant places to describe configuration and logical blocks  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Context or Link                                                                                         |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| This workflow is an AI agent Telegram bot designed to capture CRM contacts via multi-modal inputs (text, audio, images). It uses Google Gemini AI for natural language processing and Google Sheets as a CRM backend. Users can send contact data, receive clarifications, and confirm before saving. The `/new` command resets the session.                                                                                                                                                                                                                                                            | See full description in Sticky Note attached to initial Telegram Trigger node                          |
| For Telegram bot setup, create a bot via BotFather and get its Access Token. Set the webhook URL to the n8n Telegram Trigger webhook URL using Telegram API call: `curl -X POST "https://api.telegram.org/bot{TOKEN}/setWebhook?url={WEBHOOK_URL}"`                                                                                                                                                                                                                                                                                                                                              | https://docs.n8n.io/integrations/builtin/credentials/telegram/ and https://core.telegram.org/bots/features |
| The Google Sheets document should have columns: Full name, Email, Phone, Company, Job title, Meeting notes. The workflow uses a service account for Google Sheets API credentials.                                                                                                                                                                                                                                                                                                                                                                                                                   | Setup Google Sheets accordingly with correct permissions                                               |
| Google Gemini API key (PaLM) is required for AI nodes: AI Agent, Transcribe audio, Analyze image, and Chat model nodes.                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Obtain API key from Google Cloud console                                                               |
| The phone number field is prefixed with a single quote `'` if it starts with a plus `+` symbol to ensure Google Sheets stores it as text and preserves formatting.                                                                                                                                                                                                                                                                                                                                                                                                                                  | Implemented in AI Agent system message instructions                                                   |
| The AI Agent prompts and system messages are carefully designed to extract mandatory and optional contact details and confirm with the user using formatted checkmarks and question marks. Up to 3 clarifying questions are allowed per contact.                                                                                                                                                                                                                                                                                                                                                  | See AI Agent system message configuration                                                             |
| This example uses Google Sheets as a CRM placeholder but can be replaced with other CRM integrations like HubSpot, Pipedrive, or Monday.com by changing the contact management nodes.                                                                                                                                                                                                                                                                                                                                                                                                               | Workflow modularity                                                                                     |

---

**Disclaimer:**  
The provided text is exclusively from an automated workflow built in n8n, an integration and automation tool. It strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.