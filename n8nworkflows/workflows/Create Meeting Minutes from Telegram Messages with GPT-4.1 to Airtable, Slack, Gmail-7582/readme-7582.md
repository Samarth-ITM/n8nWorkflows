Create Meeting Minutes from Telegram Messages with GPT-4.1 to Airtable, Slack, Gmail

https://n8nworkflows.xyz/workflows/create-meeting-minutes-from-telegram-messages-with-gpt-4-1-to-airtable--slack--gmail-7582


# Create Meeting Minutes from Telegram Messages with GPT-4.1 to Airtable, Slack, Gmail

---

### 1. Workflow Overview

This workflow automates the creation and distribution of meeting minutes from Telegram messages or voice notes. It is designed for teams that capture meeting discussions in Telegram and want to generate structured, professional meeting reports automatically sent via email, archived in Airtable, and notified on Slack.

**Use Cases:**
- Convert Telegram text messages or voice notes into formal meeting minutes.
- Automatically parse and extract meeting details such as recipients, topics, participants, decisions, and action items.
- Store meeting minutes in Airtable for archival and tracking.
- Notify team members in Slack and send the report via Gmail.

**Logical Blocks:**

- **1.1 Telegram Input Processing:** Trigger and differentiate between text messages and voice notes from Telegram.
- **1.2 Voice Notes Handling:** Download and transcribe voice notes into text.
- **1.3 Email Extraction and Structuring:** Parse the message text to extract recipient emails and structure meeting content.
- **1.4 Meeting Minutes Generation via GPT-4.1:** Use a GPT-4.1 specialized prompt to generate clean, structured meeting minutes in JSON format.
- **1.5 Data Cleanup and Formatting:** Convert GPT output to Airtable-compatible fields, clean HTML to plain text.
- **1.6 Storage and Distribution:** Save the report to Airtable, send Slack notification, and email the report via Gmail.
- **1.7 Control and Wait Nodes:** Manage workflow timing and conditional branching.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Telegram Input Processing

**Overview:**  
Receives incoming Telegram updates and determines if the message is a text or a voice note to route the processing accordingly.

**Nodes Involved:**  
- Telegram Trigger  
- Message or Voice ? (Code)  
- If (Conditional)  
- Voice note (API call to Telegram to get voice metadata)  
- Content (Code to extract text content)

**Node Details:**

- **Telegram Trigger**  
  - Type: Trigger node  
  - Configuration: Listens to Telegram "message" updates only  
  - Credentials: Telegram API  
  - Inputs: Telegram updates  
  - Outputs: Raw message data forwarded to next node  
  - Potential failures: Telegram API connectivity, webhook issues

- **Message or Voice ? (Code)**  
  - Type: Function (JavaScript) node  
  - Role: Analyzes Telegram update JSON to classify message type as "text" or "voice" and extract relevant content or file_id.  
  - Key expressions: Checks for `updates.message.text` and `updates.message.voice.file_id`  
  - Inputs: Telegram Trigger output  
  - Outputs: JSON with `{type: "text" or "voice", content or file_id}`  
  - Edge cases: Missing fields, unsupported message types result in empty output (workflow ignores)

- **If (Conditional)**  
  - Type: Conditional node  
  - Role: Routes flow based on message type ("voice" or other)  
  - Configuration: Checks if `{{$json.type}}` equals "voice"  
  - Inputs: Message or Voice ? output  
  - Outputs: Two branches - True (voice), False (text)  
  - Edge cases: Unexpected message types skip processing

- **Voice note**  
  - Type: Telegram node (file metadata fetch)  
  - Role: Retrieves metadata for the voice message using Telegram API and file_id  
  - Inputs: If condition true (voice)  
  - Outputs: Voice file metadata, including file_id for download  
  - Credentials: Telegram API  
  - Edge cases: File not found, API failure

- **Content (Code)**  
  - Type: Function (JavaScript) node  
  - Role: Extracts text content from text messages for further processing  
  - Inputs: If condition false (text messages)  
  - Outputs: JSON with `{text: content}`  
  - Edge cases: Empty or malformed text

---

#### 1.2 Voice Notes Handling

**Overview:**  
Downloads voice note files from Telegram and uses OpenAI's transcription capability to convert audio into text.

**Nodes Involved:**  
- Wait (for timing control)  
- Voice note download (Telegram file download)  
- Transcription of voice notes to text (OpenAI audio transcription)  
- WAIT (pause after transcription)

**Node Details:**

- **Wait (first)**  
  - Type: Wait node  
  - Role: Inserts delay after fetching voice message metadata before download  
  - Configuration: Default (no specific time) – likely immediate pass-through  
  - Inputs: Voice note output  
  - Outputs: Triggers voice note download  
  - Edge cases: Delays might be needed if Telegram file availability is delayed

- **Voice note download**  
  - Type: Telegram node (file download)  
  - Role: Downloads the actual voice note file using file_id from metadata  
  - Inputs: Wait output  
  - Credentials: Telegram API  
  - Outputs: Binary audio file data passed to transcription node  
  - Edge cases: API errors, file missing, network issues

- **Transcription of voice notes to text**  
  - Type: OpenAI node (audio resource)  
  - Role: Uses OpenAI's audio transcription to convert voice note to text  
  - Inputs: Binary audio file  
  - Credentials: OpenAI API  
  - Outputs: Transcribed text  
  - Potential failures: Audio format incompatibility, API limits, transcription errors

- **WAIT (second)**  
  - Type: Wait node  
  - Role: Pauses the flow briefly after transcription (to ensure stable processing)  
  - Inputs: Transcription output  
  - Outputs: Continues to next block  
  - Edge cases: None critical

---

#### 1.3 Email Extraction and Structuring

**Overview:**  
Processes the text (from message or transcription) to extract or infer the recipient email, domain hints, and prepare structured input for GPT.

**Nodes Involved:**  
- Code structure (Code)  

**Node Details:**

- **Code structure**  
  - Type: Function (JavaScript) node  
  - Role: Parses the text to detect explicit email addresses or build probable email addresses from text cues like "send to" or "envoyer à", including domain extraction and fallback to `contact@gmail.com`.  
  - Key techniques: Regex matching for emails and domains, normalization of names, fallback heuristics  
  - Inputs: Text content either from Content node (text) or WAIT node (transcription)  
  - Outputs: JSON with `{ input: { text }, email, domain, domainUrl }`  
  - Edge cases: No email detected, ambiguous text, incomplete addresses, non-standard formats  
  - Failure modes: Regex failure, empty input string

---

#### 1.4 Meeting Minutes Generation via GPT-4.1

**Overview:**  
Uses a GPT-4.1-mini model with a detailed prompt specialized for meeting minutes generation, producing a structured JSON with email, subject, and HTML body.

**Nodes Involved:**  
- Generate Meeting Message (Langchain Agent)  
- Enforce Email JSON (Output Parser Structured)  

**Node Details:**

- **Generate Meeting Message**  
  - Type: Langchain Agent node  
  - Role: Calls GPT-4.1-mini to generate meeting minutes, extracting structured data and formatting the email body in HTML according to a comprehensive prompt.  
  - Prompt highlights: Detect recipient email, meeting context, participants, decisions, actions, next steps, risks, open questions, and produce email subject and body as HTML.  
  - Inputs: From Code structure node output (`input.text`)  
  - Outputs: GPT raw JSON output (email, subject, body)  
  - Model: gpt-4.1-mini  
  - Edge cases: API rate limits, malformed output, incomplete response

- **Enforce Email JSON**  
  - Type: Langchain Output Parser Structured node  
  - Role: Validates and enforces GPT output conforms strictly to the JSON schema `{email, subject, body}`  
  - Inputs: GPT output  
  - Outputs: Parsed and validated JSON for downstream use  
  - Edge cases: Parsing failures, invalid JSON structure

---

#### 1.5 Data Cleanup and Formatting

**Overview:**  
Transforms structured GPT output into fields compatible with Airtable and other systems, removing HTML tags to produce a plain-text version for the report.

**Nodes Involved:**  
- Code (Cleanup / Airtable mapping)  

**Node Details:**

- **Code**  
  - Type: Function (JavaScript) node  
  - Role: Extracts fields from GPT output, cleans HTML body into readable plain text, prepares `{Email, subject, Report}` fields for Airtable.  
  - Key expressions: Uses regex to replace `<br>`, `<p>`, and strip HTML tags  
  - Inputs: Generate Meeting Message output JSON  
  - Outputs: JSON with Email, subject, Report (plain text)  
  - Edge cases: Malformed HTML, empty body, missing fields

---

#### 1.6 Storage and Distribution

**Overview:**  
Creates a new record in Airtable with the meeting report, sends a notification message to a Slack channel, and sends the meeting minutes by Gmail.

**Nodes Involved:**  
- Create a record (Airtable)  
- Send a message (Slack)  
- Send Email (Gmail)  

**Node Details:**

- **Create a record**  
  - Type: Airtable node  
  - Role: Inserts a new record with fields Email, subject, Report into specified Airtable base and table  
  - Credentials: Airtable Personal Access Token  
  - Inputs: Code cleanup outputs  
  - Outputs: Airtable record JSON including fields used for email and Slack  
  - Edge cases: API limits, authentication failure, schema mismatch

- **Send a message**  
  - Type: Slack node  
  - Role: Posts a message to a configured Slack channel containing the subject and report text  
  - Credentials: Slack OAuth2  
  - Inputs: Airtable record output fields  
  - Configuration: Channel ID must be set by user (`YOUR CHANNEL`)  
  - Edge cases: Slack API limits, invalid channel ID, auth failure

- **Send Email**  
  - Type: Gmail node  
  - Role: Sends an email with subject and message body to the extracted recipient email  
  - Credentials: Gmail OAuth2  
  - Inputs: Airtable record fields for Email, subject, Report  
  - Edge cases: Email sending failures, invalid recipient, OAuth token expiration

---

#### 1.7 Control and Wait Nodes

**Overview:**  
Manage timing and ensure proper sequencing of asynchronous operations such as file availability and transcription.

**Nodes Involved:**  
- Wait (before voice note download)  
- WAIT (after transcription)  

**Node Details:**  
- Both wait nodes have no explicit delay configured; likely used for internal timing control.  
- Inputs/outputs: Control flow between Telegram voice note metadata retrieval, file download, transcription, and further processing.

---

### 3. Summary Table

| Node Name                   | Node Type                               | Functional Role                                               | Input Node(s)                  | Output Node(s)              | Sticky Note                                                                                              |
|-----------------------------|---------------------------------------|--------------------------------------------------------------|-------------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------|
| Telegram Trigger            | Telegram Trigger                      | Receives Telegram messages                                   | -                             | Message or Voice ?           | ## Message sent to Telegram as voice note or text?                                                    |
| Message or Voice ?          | Code (Function)                      | Classifies input as text or voice message                     | Telegram Trigger              | If                          | ## Message sent to Telegram as voice note or text?                                                    |
| If                         | If (Conditional)                     | Routes flow based on message type                             | Message or Voice ?            | Voice note, Content          | ## Message sent to Telegram as voice note or text?                                                    |
| Voice note                 | Telegram                              | Retrieves voice message metadata                              | If (voice branch)             | Wait                        | ## Message sent to Telegram as voice note or text?                                                    |
| Wait                       | Wait                                 | Controls timing before downloading voice note                | Voice note                   | Voice note download          | ## Message sent to Telegram as voice note or text?                                                    |
| Voice note download        | Telegram                              | Downloads voice note audio file                               | Wait                        | Transcription of voice notes | ## Message sent to Telegram as voice note or text?                                                    |
| Transcription of voice notes to text | OpenAI (Audio transcription)          | Converts voice note to text                                   | Voice note download          | WAIT                        | ## Message sent to Telegram as voice note or text?                                                    |
| WAIT                       | Wait                                 | Pauses flow after transcription                              | Transcription of voice notes | Code structure              |                                                                                                        |
| Content                    | Code (Function)                      | Extracts text from Telegram text messages                    | If (text branch)             | Code structure              | ## Message sent to Telegram as voice note or text?                                                    |
| Code structure             | Code (Function)                      | Extracts/infers email and domain from text                   | Content, WAIT                | Generate Meeting Message     |                                                                                                        |
| Generate Meeting Message   | Langchain Agent                      | Generates structured meeting minutes JSON via GPT-4.1       | Code structure               | Code                        | ## Generate meeting minutes  
- **Node**: Agent (“Generate Meeting Message”)  
- **Prompt**: specialized for “meeting minutes”  
- **Model**: GPT-4.1-mini  
- **Output**: `{ email, subject, body }`  
👉 GPT creates a clean and structured meeting report. |
| Enforce Email JSON          | Langchain Output Parser Structured   | Validates and enforces strict JSON output                     | GPT-4 Email Generator Model  | Generate Meeting Message     | ## Enforce clean JSON  
- **Node**: Output Parser Structured  
- **JSON Example**:  
json {"email": "address@gmail.com","subject":"Subject","body":Email"}  
👉 Ensures the output is always valid JSON.                     |
| GPT-4 Email Generator Model | Langchain LM Chat OpenAI              | GPT-4.1 mini language model for email generation             | Enforce Email JSON           | Generate Meeting Message     |                                                                                                        |
| Code                       | Code (Function)                      | Cleans HTML to plain text and maps fields for Airtable       | Generate Meeting Message     | Create a record              | ## Cleanup / Airtable mapping  
- **Node**: Code  
- **Return**: `{ Email, subject, Report }`  
👉 Prepares the correct fields aligned with your Airtable table. |
| Create a record            | Airtable                             | Saves meeting minutes record                                  | Code                        | Send a message              | ## Store in Airtable  
- **Node**: Airtable (Create Record)  
- **Mapping**:  
  - Email = `{{$json.Email}}`  
  - subject = `{{$json.subject}}`  
  - Report = `{{$json.Report}}`  
👉 Archives each meeting report in your Airtable base.           |
| Send a message             | Slack                               | Sends notification message to Slack channel                  | Create a record             | Send Email                  | ## Notify in Slack  
- **Node**: Slack (Send Message)  
- **Channel**: your team channel  
- **Message**:  {{$json.fields.subject}}{{$json.fields.Report}}     |
| Send Email                 | Gmail                               | Emails meeting minutes to recipient                           | Send a message              | -                           | ## Send the email  
- **Node**: Gmail (Send Email)  
- **sendTo**: `{{$('Create a record').item.json.fields.Email}}`  
- **subject**: `{{$('Create a record').item.json.fields.subject}}`  
- **message**: `{{$('Create a record').item.json.fields.Report}}`  
👉 Sends the complete meeting minutes to the recipients.         |
| Sticky Note                | Sticky Note                         | Informational; message type explanation                       | -                           | -                           | ## Message sent to Telegram as voice note or text?                                                    |
| Sticky Note1               | Sticky Note                         | GIF illustration                                              | -                           | -                           | ![Alt text](https://media.giphy.com/media/v1.Y2lkPTc5MGI3NjExaXpjaWY5ajhyN3Zvc3U2aDhqOGkzOWg4M3hvdjl1ZHRzMGxwYTVrNCZlcD12MV9naWZzX3NlYXJjaCZjdD1n/ya4eevXU490Iw/giphy.gif) |
| Sticky Note2               | Sticky Note                         | GPT meeting minutes explanation                              | -                           | -                           | ## Generate meeting minutes  
- **Node**: Agent (“Generate Meeting Message”)  
- **Prompt**: specialized for “meeting minutes”  
- **Model**: GPT-4.1-mini  
- **Output**: `{ email, subject, body }`  
👉 GPT creates a clean and structured meeting report.             |
| Sticky Note3               | Sticky Note                         | JSON output enforcement explanation                           | -                           | -                           | ## Enforce clean JSON  
- **Node**: Output Parser Structured  
- **JSON Example**:  
json {"email": "address@gmail.com","subject":"Subject","body":Email"}  
👉 Ensures the output is always valid JSON.                     |
| Sticky Note4               | Sticky Note                         | Data cleaning and Airtable mapping explanation                | -                           | -                           | ## Cleanup / Airtable mapping  
- **Node**: Code  
- **Return**: `{ Email, subject, Report }`  
👉 Prepares the correct fields aligned with your Airtable table. |
| Sticky Note5               | Sticky Note                         | Airtable storage explanation                                  | -                           | -                           | ## Store in Airtable  
- **Node**: Airtable (Create Record)  
- **Mapping**:  
  - Email = `{{$json.Email}}`  
  - subject = `{{$json.subject}}`  
  - Report = `{{$json.Report}}`  
👉 Archives each meeting report in your Airtable base.           |
| Sticky Note6               | Sticky Note                         | Slack notification explanation                               | -                           | -                           | ## Notify in Slack  
- **Node**: Slack (Send Message)  
- **Channel**: your team channel  
- **Message**:  {{$json.fields.subject}}{{$json.fields.Report}}     |
| Sticky Note7               | Sticky Note                         | Gmail send email explanation                                 | -                           | -                           | ## Send the email  
- **Node**: Gmail (Send Email)  
- **sendTo**: `{{$('Create a record').item.json.fields.Email}}`  
- **subject**: `{{$('Create a record').item.json.fields.subject}}`  
- **message**: `{{$('Create a record').item.json.fields.Report}}`  
👉 Sends the complete meeting minutes to the recipients.         |
| Sticky Note8               | Sticky Note                         | ChatGPT GIF                                                    | -                           | -                           | ![Alt text](https://media1.tenor.com/m/y98Q1SkqLCAAAAAd/chat-gpt.gif)                                  |
| Sticky Note9               | Sticky Note                         | Airtable project management GIF                               | -                           | -                           | ![Alt text](https://media1.tenor.com/m/WW0Y9Urm5yoAAAAC/airtable-project-management.gif)               |
| Sticky Note10              | Sticky Note                         | Slack GIF                                                     | -                           | -                           | ![Alt text](https://media.tenor.com/go-I8u5XWoAAAAAi/slack-slackhq.gif)                               |
| Sticky Note11              | Sticky Note                         | Email sending GIF                                             | -                           | -                           | ![Alt text](https://media0.giphy.com/media/v1.Y2lkPTc5MGI3NjExOGk3YThucTEycGtvZ3piaWlheW41aXlwYWo0MTllMWpqaWs0c2tnbCZlcD12MV9pbnRlcm5hbF9naWZfYnlfaWQmY3Q9Zw/aOften89vRbG/giphy.gif) |
| Code structure             | Code (Function)                    | Extracts and builds recipient email and domain from text      | Content, WAIT                | Generate Meeting Message     |                                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: Telegram Trigger  
   - Configuration: Listen for "message" updates only  
   - Credentials: Connect your Telegram Bot API credentials

2. **Add a Code node named "Message or Voice ?"**  
   - Purpose: Identify if the message is text or voice note  
   - Code:  
     ```js
     const updates = $json;
     if (updates.message?.text) {
       return [{ json: { type: "text", content: updates.message.text } }];
     }
     if (updates.message?.voice) {
       return [{ json: { type: "voice", file_id: updates.message.voice.file_id } }];
     }
     return [];
     ```
   - Connect Telegram Trigger output to this node

3. **Add an If node**  
   - Purpose: Check if message type equals "voice"  
   - Condition: `{{$json.type}}` equals `"voice"`

4. **On the True branch:**  
   - Add Telegram node "Voice note"  
     - Operation: Get file metadata using `{{$json.file_id}}`  
     - Credentials: Telegram API  
   - Add a Wait node (default delay) to ensure file availability  
   - Add Telegram node "Voice note download"  
     - Operation: Download file using `{{$json.result.file_id}}`  
     - Credentials: Telegram API  
   - Add OpenAI node "Transcription of voice notes to text"  
     - Operation: Audio transcribe  
     - Credentials: OpenAI API  
   - Add Wait node (default delay)  
   - Connect Wait node output to next block

5. **On the False branch:**  
   - Add Code node "Content"  
     - Code: `return [{ text: $json.content }];`

6. **Connect outputs of both Wait and Content nodes to Code node "Code structure"**  
   - Purpose: Parse text to extract or build email address  
   - Code: (Use the detailed email extraction JavaScript provided in the workflow)  
   - Outputs: `{ input: { text }, email, domain, domainUrl }`

7. **Add Langchain Output Parser Structured node "Enforce Email JSON"**  
   - JSON Schema Example:  
     ```json
     {
       "email": "address@gmail.com",
       "subject": "Email subject",
       "body": "<p>Email body in HTML</p>"
     }
     ```
8. **Add Langchain LM Chat OpenAI node "GPT-4 Email Generator Model"**  
   - Model: GPT-4.1-mini  
   - Credentials: OpenAI API  
   - Connect output of "Enforce Email JSON" to this node

9. **Add Langchain Agent node "Generate Meeting Message"**  
   - Prompt: Specialized prompt for meeting minutes generation (as detailed in the workflow)  
   - Connect "GPT-4 Email Generator Model" output to this node  
   - Connect "Enforce Email JSON" to "GPT-4 Email Generator Model" (ai_outputParser relationship)

10. **Add Code node "Code" for cleanup and Airtable mapping**  
    - Code: Convert HTML body to plain text and map fields Email, subject, Report  
    - Connect output of "Generate Meeting Message" to this node

11. **Add Airtable node "Create a record"**  
    - Operation: Create record in your Airtable base and table  
    - Map fields: Email, subject, Report  
    - Credentials: Airtable Personal Access Token

12. **Add Slack node "Send a message"**  
    - Operation: Post message to a team Slack channel  
    - Channel: Set your target channel ID  
    - Message: Combine subject and report fields  
    - Credentials: Slack OAuth2  
    - Connect Airtable record output to this node

13. **Add Gmail node "Send Email"**  
    - Operation: Send email  
    - To: Use Airtable record Email field  
    - Subject: Airtable record subject field  
    - Message: Airtable record Report field  
    - Credentials: Gmail OAuth2  
    - Connect Slack node output to this node

14. **Add Sticky Notes as needed to document each major step**

---

### 5. General Notes & Resources

| Note Content                                                                                                        | Context or Link                                                                                                  |
|--------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Workflow designed for automating meeting minutes from Telegram messages and voice notes using GPT-4.1 and Airtable | Project description                                                                                             |
| Includes detailed prompt engineering for GPT to extract structured meeting details and generate professional emails| Example prompt is embedded in the "Generate Meeting Message" node                                               |
| Slack notification and Gmail integration require proper OAuth2 credentials and channel/email configuration          | Slack channel ID and Gmail OAuth2 credentials must be set up in n8n prior to running                            |
| Airtable schema must include string fields: Email, subject, Report, and optionally Date                             | Airtable base and table configuration must reflect these fields                                                |
| For voice notes, Telegram file download and OpenAI transcription may require handling API rate limits and delays    | Use Wait nodes to avoid race conditions or API throttling                                                      |
| Email extraction includes heuristics to build probable emails from natural language phrases                         | See "Code structure" node for detailed extraction logic                                                        |
| Visual sticky notes in the workflow provide helpful explanations and links to GIFs for illustration                 | E.g., ChatGPT, Airtable, Slack, Gmail usage gifs                                                                |

---

**Disclaimer:**  
The text provided is extracted exclusively from an automated workflow created with n8n, an integration and automation tool. The processing strictly respects existing content policies and contains no illegal, offensive, or protected elements. All data handled are legal and public.

---