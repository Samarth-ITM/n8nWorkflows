Process Multiple Media Files in Telegram with Gemini AI & PostgreSQL Database

https://n8nworkflows.xyz/workflows/process-multiple-media-files-in-telegram-with-gemini-ai---postgresql-database-7455


# Process Multiple Media Files in Telegram with Gemini AI & PostgreSQL Database

### 1. Workflow Overview

This workflow automates the processing of multiple media file types received via Telegram, leveraging Google Gemini AI for content analysis and a PostgreSQL database for structured storage and queue management. It is designed to handle diverse media inputs including texts, voice messages, images, videos, documents (in multiple formats), and grouped media messages. The workflow facilitates descriptive analysis, data extraction, and conversation context management through AI, while orchestrating media grouping and message dispatch efficiently.

Logical blocks:

- **1.1 Database Setup:** Initializes PostgreSQL tables for storing media groups, media queue, and chat histories.
- **1.2 Telegram Message Reception & Classification:** Listens to incoming Telegram messages, routes them by media type.
- **1.3 Media Download & MIME Fixing:** Downloads media files from Telegram and normalizes MIME types.
- **1.4 Document Processing:** Extracts and normalizes content from various document types with fallback AI analysis for PDFs.
- **1.5 Media Group & Caption Handling:** Stores media group data, captions, and manages queue insertion logic based on presence of captions and grouping.
- **1.6 Media Queue Trigger & Aggregation:** Detects new media group entries, aggregates file descriptions, and prepares unified messages.
- **1.7 AI Processing & Response Formatting:** Sends processed messages to AI agent with PostgreSQL-backed chat memory; formats AI outputs safely for Telegram MarkdownV2.
- **1.8 Output Delivery:** Returns AI-generated text back to the correct Telegram chat, handling message chunking.

---

### 2. Block-by-Block Analysis

#### 1.1 Database Setup

- **Overview:** Creates necessary PostgreSQL tables for storing media groups, media queues, and chat histories to persist state and context.
- **Nodes Involved:**  
  - `When clicking ‘Execute workflow’` (Manual Trigger)  
  - `Create Tables` (PostgreSQL Execute Query)  
  - `Sticky Note6` (Documentation)
- **Node Details:**
  - *Create Tables*: Executes SQL to create three tables:  
    - `media_group`: stores file descriptions grouped by Telegram `media_group_id`.  
    - `media_queue`: tracks unique media groups queued for processing, with chat IDs and captions.  
    - `chat_histories`: stores AI conversation histories indexed by session ID.  
  - *Sticky Note6*: Explains table purposes and rationale.
- **Potential Failures:** Database connection errors, SQL syntax errors if table exists with incompatible schema.
- **Version Requirements:** PostgreSQL node v2.6+ recommended for compatibility.
- **Sub-Workflow:** None.

#### 1.2 Telegram Message Reception & Classification

- **Overview:** Listens for new Telegram messages, detects the type of message (text, voice, video note, image, audio, video, document, or unsupported), then routes accordingly.
- **Nodes Involved:**  
  - `Telegram Trigger`  
  - `Typing…` (Telegram Chat Action)  
  - `Input Message Router1` (Switch)  
  - `Sticky Note8` (Documentation)
- **Node Details:**  
  - *Telegram Trigger*: Listens for `message` updates, does not auto-download files.  
  - *Typing…*: Sends "typing" chat action to Telegram to indicate bot is processing.  
  - *Input Message Router1*: Checks Telegram message JSON fields to determine message type using conditions on message properties (e.g., existence of `text`, `voice`, `video_note.file_id`, etc.), outputting to named branches like "Text", "Voice Message", "Video note", "Image", "Audio", "Video", "Document", or "extra".  
  - *Sticky Note8*: Describes this as the Blue Section handling message type routing and initial processing.
- **Expressions:** Uses JSON path expressions like `{{ $json.message.voice.file_id }}` to detect presence of various media types.  
- **Potential Failures:** Missing fields in Telegram message JSON, malformed updates, Telegram API downtime.
- **Version Requirements:** Telegram nodes v1.2+ for webhook and message fields compatibility.
- **Sub-Workflow:** None.

#### 1.3 Media Download & MIME Fixing

- **Overview:** Downloads Telegram media files based on detected type, then normalizes MIME types using code nodes to ensure correct file handling downstream.
- **Nodes Involved:**  
  - Download nodes: `Download Voice Message`, `Download VIDEO NOTE`, `Download IMAGE`, `Download AUDIO`, `Download VIDEO`, `Download CSV`, `Download HTML`, `Download ICS`, `Download JSON`, `Download ODS`, `Download PDF`, `Download RTF`, `Download TEXT FILE`, `Download XML`, `Download XLSX`  
  - MIME Fix nodes: `Fix mime`, `Fix mime1`, `Fix mime4`, `Fix mime5`, `Fix mime6` (all Code nodes)  
  - Extraction nodes for document types (e.g., `Extract from CSV`, `Extract from ICS`, etc.)
  - *Sticky Note10* explains this Yellow Section for file and caption handling.
- **Node Details:**  
  - Each download node uses file IDs from Telegram message JSON to fetch files.  
  - MIME fix code nodes map file extensions to correct MIME types and update the binary data accordingly.  
  - Extraction nodes parse files to extract structured content (CSV, ICS, ODS, JSON, PDF, RTF, TXT, XML, XLSX).  
- **Key Expressions:** File ID extraction uses fallback logic (e.g., for photos with multiple sizes, chooses first available).  
- **Potential Failures:**  
  - Telegram file download failures (expired file IDs, network errors).  
  - MIME map mismatches or missing extensions.  
  - Extraction module failures if file is corrupt or unsupported content.  
- **Version Requirements:** Telegram nodes v1.2+, Extract nodes v1+.  
- **Sub-Workflow:** None.

#### 1.4 Document Processing

- **Overview:** Processes supported document formats with extraction and normalization. For PDFs, if text extraction yields empty, falls back to AI analysis.
- **Nodes Involved:**  
  - Extraction nodes (`Extract from CSV`, `Extract from ICS`, etc.)  
  - Normalization nodes (`Normalize CSV`, `Normalize ICS`, `Normalize ODS`, `Normalize PDF`, `Normalize RTF`, `Normalize XML`, `Normalize HTML`, `Normalize JSON`, `Normalize XLSX`, `Normalize text file`)  
  - AI analysis nodes (`Analyze document`, `Normalize PDF (AI)`)  
  - Error handling nodes (`get_error_message`, `get_error_message1`)  
  - `Merge` node to join extraction and AI fallback  
  - `Sticky Note9` (Documentation)
- **Node Details:**  
  - Extraction nodes produce raw data from files.  
  - Normalization nodes wrap extracted data into a consistent `data` field string for downstream use.  
  - For PDFs, after initial extraction, a conditional node checks if extracted text exists; if not, it uses AI to analyze the PDF content.  
  - Error nodes generate standardized error message responses for unsupported files, ensuring AI agent receives meaningful feedback.  
  - *Sticky Note9* clarifies this Red Section document processing workflow.
- **Potential Failures:** Extraction failure due to file corruption, AI API errors, timeout for large PDFs.
- **Version Requirements:** Extract nodes v1+, Langchain AI nodes v1+.
- **Sub-Workflow:** None.

#### 1.5 Media Group & Caption Handling

- **Overview:** Handles storage of media files and captions in the PostgreSQL `media_group` and `media_queue` tables, managing different cases for presence/absence of captions and media group IDs.
- **Nodes Involved:**  
  - Conditional nodes `Captions?1`, `Media_group?2`, `Media_group?3` (If nodes)  
  - Postgres nodes `Insert documents in media_group`, `Insert documents in media_group1`, `Insert media_queue with captions (Trigger)`, `Insert media_queue (Trigger)`  
  - `Wait` node for timing control  
  - `Get_file_and_captions`, `Get_only_file` (Set nodes)  
  - `Sticky Note10` (Documentation)
- **Node Details:**  
  - *Captions?1* checks if caption text exists on the first Telegram message in the group.  
  - *Media_group?2* and *Media_group?3* check for presence of `media_group_id`.  
  - Depending on these checks, inserts are performed into `media_group` and `media_queue` tables accordingly:  
    1. Captions + Media Group: Store file description, then queue media group with captions.  
    2. Captions only (no Media Group): Combine captions and file description, send to AI immediately.  
    3. No Captions + Media Group: Insert file description, wait 2 seconds, then insert media group into queue if not present.  
    4. No Captions + No Media Group: Send file description directly to AI agent.  
  - `Wait` node introduces delay to ensure captions are processed before queue insertion.  
  - Set nodes format the messages and chat IDs properly for AI consumption.  
- **Potential Failures:**  
  - Inconsistent or missing `media_group_id` could cause misgrouping.  
  - Postgres insert conflicts or connection errors.  
  - Timing issues if wait duration insufficient.  
- **Version Requirements:** Postgres node v2.6+, If node v2.2+.  
- **Sub-Workflow:** None.

#### 1.6 Media Queue Trigger & Aggregation

- **Overview:** Listens for new entries in the `media_queue` table, waits briefly to ensure all media files are stored, fetches all related file descriptions, aggregates into unified messages, then sends to AI.
- **Nodes Involved:**  
  - `Media_queue Trigger` (Postgres Trigger)  
  - `get_chat_id` (Set node)  
  - `Wait for all the files` (Wait node)  
  - `Get all files from group_id` (Postgres Select)  
  - `unified_variables` (Code node)  
  - `Get_message (multiple files)` (Set node)  
  - `Sticky Note7` (Documentation)
- **Node Details:**  
  - *Media_queue Trigger* listens for inserts in media_queue table.  
  - *get_chat_id* extracts `chat_id` and `media_group_id` from trigger payload for downstream queries.  
  - *Wait for all the files* introduces a short pause to ensure database consistency.  
  - *Get all files from group_id* retrieves all file descriptions linked to the media group.  
  - *unified_variables* compiles a combined message with caption and all file descriptions.  
  - *Get_message (multiple files)* prepares the final message and chat ID for AI input.  
  - *Sticky Note7* describes this Green Section as the media queue listener and aggregator.
- **Potential Failures:**  
  - Trigger may miss rapid inserts or duplicates.  
  - Database read/write race conditions.  
  - Timeout if media files incomplete or database slow.  
- **Version Requirements:** Postgres Trigger node v1+.  
- **Sub-Workflow:** None.

#### 1.7 AI Processing & Response Formatting

- **Overview:** Sends processed messages to the AI agent (Google Gemini via LangChain n8n nodes) with PostgreSQL memory backend, then formats AI output safely for Telegram MarkdownV2 and splits long messages.
- **Nodes Involved:**  
  - `AI Agent1` (LangChain Agent node)  
  - `Postgres Chat Memory` (LangChain Memory Postgres node)  
  - `MarkdownV2` (Code node for Telegram-safe formatting and chunking)  
  - `Send a text message1` (Telegram node for sending messages)  
  - `Sticky Note11` (Documentation)
- **Node Details:**  
  - *AI Agent1* receives unified messages, sends to Google Gemini AI model (model varies by input type), stores conversation context in PostgreSQL chat_histories via *Postgres Chat Memory*.  
  - *MarkdownV2* escapes characters restricted by Telegram MarkdownV2, validates and normalizes URLs, converts headings, and splits messages into Telegram-compatible chunks (max 4096 chars).  
  - *Send a text message1* sends the processed, formatted chunks back to the originating Telegram chat, disables attribution and uses MarkdownV2 parse mode.  
  - *Sticky Note11* explains this Purple Section's focus on AI integration and output formatting.
- **Expressions:** Uses expressions like `{{ $json.message }}` and `$json.chat_id` to route text and chat context.  
- **Potential Failures:**  
  - AI API rate limits, network errors.  
  - Markdown formatting errors causing Telegram message rejections.  
  - Large message chunking edge cases.  
- **Version Requirements:** LangChain nodes v1+, Telegram node v1.2+.  
- **Sub-Workflow:** None.

#### 1.8 Output Delivery

- **Overview:** Final step of sending AI-generated messages back to the Telegram user in chunks, respecting chat ID and formatting constraints.
- **Nodes Involved:**  
  - `Send a text message1` (Telegram node)  
- **Node Details:**  
  - Sends messages to Telegram chat using safe MarkdownV2 formatting.  
  - Handles multiple message part indexes to send large responses sequentially.  
- **Potential Failures:** Telegram API errors, chat ID mismatches.  
- **Version Requirements:** Telegram node v1.2+.  
- **Sub-Workflow:** None.

---

### 3. Summary Table

| Node Name                          | Node Type                          | Functional Role                          | Input Node(s)                                 | Output Node(s)                                | Sticky Note                                                      |
|-----------------------------------|----------------------------------|----------------------------------------|-----------------------------------------------|-----------------------------------------------|-----------------------------------------------------------------|
| When clicking ‘Execute workflow’  | Manual Trigger                   | Start manual workflow                   |                                               | Create Tables                                 |                                                                 |
| Create Tables                     | Postgres (Execute Query)         | DB schema setup                        | When clicking ‘Execute workflow’              |                                               |                                                                 |
| Telegram Trigger                  | Telegram Trigger                 | Listen to Telegram messages             |                                               | Typing…, Input Message Router1                 | Blue Section – Telegram Trigger & Message Type Handling         |
| Typing…                          | Telegram (Send Chat Action)       | Show typing indicator                   | Telegram Trigger                              |                                               |                                                                 |
| Input Message Router1             | Switch                          | Route messages by type                   | Telegram Trigger                              | Multiple download or processing nodes         | Blue Section – Telegram Trigger & Message Type Handling         |
| Download Voice Message            | Telegram (Download File)          | Download voice message                   | Input Message Router1                         | Fix mime                                      |                                                                 |
| Fix mime                         | Code                            | Normalize MIME type for voice           | Download Voice Message                        | Analyze voice message                         |                                                                 |
| Analyze voice message            | Google Gemini AI (LangChain)      | Analyze voice content                    | Fix mime                                      | get_message (Audio/Video message)              |                                                                 |
| get_message (Audio/Video message)| Set                             | Prepare message for AI                   | Analyze voice message / Analyze video note / Analyze image / Analyze audio | Captions?1                                   |                                                                 |
| Download VIDEO NOTE              | Telegram (Download File)           | Download video note                      | Input Message Router1                         | Fix mime1                                     |                                                                 |
| Fix mime1                       | Code                            | Normalize MIME type for video note      | Download VIDEO NOTE                           | Analyze video note                            |                                                                 |
| Analyze video note              | Google Gemini AI (LangChain)      | Analyze video note content               | Fix mime1                                     | get_message (Audio/Video message)              |                                                                 |
| Download IMAGE                  | Telegram (Download File)           | Download image                          | Input Message Router1                         | Fix mime5                                     |                                                                 |
| Fix mime5                      | Code                            | Normalize MIME type for image            | Download IMAGE                               | Analyze image                                |                                                                 |
| Analyze image                  | Google Gemini AI (LangChain)      | Analyze image content                    | Fix mime5                                     | get_message (Audio/Video message)              |                                                                 |
| Download AUDIO                 | Telegram (Download File)           | Download audio                          | Input Message Router1                         | Fix mime6                                     |                                                                 |
| Fix mime6                     | Code                            | Normalize MIME type for audio            | Download AUDIO                               | Analyze audio                                |                                                                 |
| Analyze audio                 | Google Gemini AI (LangChain)      | Analyze audio content                    | Fix mime6                                     | get_message (Audio/Video message)              |                                                                 |
| Download VIDEO                | Telegram (Download File)           | Download video                         | Input Message Router1                         | Fix mime4                                     |                                                                 |
| Fix mime4                    | Code                            | Normalize MIME type for video            | Download VIDEO                               | Analyze video                                |                                                                 |
| Analyze video                | Google Gemini AI (LangChain)      | Analyze video content                    | Fix mime4                                     | get_message (Audio/Video message)              |                                                                 |
| Group Similar Documents       | Code                            | Categorize documents by file type        | Input Message Router1                         | Switch                                       |                                                                 |
| Switch                      | Switch                          | Route documents by type                   | Group Similar Documents                       | Various Download nodes / Error message        |                                                                 |
| Download CSV                | Telegram (Download File)           | Download CSV document                     | Switch                                        | Extract from CSV                              |                                                                 |
| Extract from CSV           | Extract from File                | Extract CSV content                      | Download CSV                                  | Aggregate                                     |                                                                 |
| Aggregate                 | Aggregate                       | Aggregate CSV data                        | Extract from CSV                              | Normalize CSV                                 |                                                                 |
| Normalize CSV             | Set                            | Normalize CSV extraction                  | Aggregate                                     | get_message (File message)                     |                                                                 |
| Download HTML             | Telegram (Download File)           | Download HTML document                    | Switch                                        | HTML Extract Generic1                          |                                                                 |
| HTML Extract Generic1      | HTML Extract                    | Extract HTML content                      | Download HTML                                 | Normalize HTML                                |                                                                 |
| Normalize HTML            | Set                            | Normalize extracted HTML                  | HTML Extract Generic1                          | get_message (File message)                     |                                                                 |
| Download ICS              | Telegram (Download File)           | Download ICS document                     | Switch                                        | Extract from ICS                              |                                                                 |
| Extract from ICS          | Extract from File                | Extract ICS content                       | Download ICS                                  | Normalize ICS                                 |                                                                 |
| Normalize ICS             | Set                            | Normalize extracted ICS                   | Extract from ICS                              | get_message (File message)                     |                                                                 |
| Download JSON             | Telegram (Download File)           | Download JSON document                    | Switch                                        | Extract from JSON                             |                                                                 |
| Extract from JSON         | Extract from File                | Extract JSON content                      | Download JSON                                 | Normalize JSON                                |                                                                 |
| Normalize JSON            | Set                            | Normalize extracted JSON                  | Extract from JSON                             | get_message (File message)                     |                                                                 |
| Download ODS              | Telegram (Download File)           | Download ODS spreadsheet                  | Switch                                        | Extract from ODS                              |                                                                 |
| Extract from ODS          | Extract from File                | Extract ODS content                       | Download ODS                                  | Get ODS data                                  |                                                                 |
| Get ODS data              | Code                           | Format ODS extraction data                | Extract from ODS                              | Normalize ODS                                 |                                                                 |
| Normalize ODS             | Set                            | Normalize extracted ODS                   | Get ODS data                                  | get_message (File message)                     |                                                                 |
| Download PDF              | Telegram (Download File)           | Download PDF document                     | Switch                                        | Extract from PDF, Merge                        |                                                                 |
| Extract from PDF          | Extract from File                | Extract PDF content                       | Download PDF                                  | Text?                                          |                                                                 |
| Text?                    | If                             | Check if PDF extraction has text          | Extract from PDF                              | Normalize PDF / Merge                          |                                                                 |
| Normalize PDF            | Set                            | Normalize extracted PDF                   | Text? (true branch)                           | get_message (File message)                     |                                                                 |
| Merge                    | Merge                          | Merge fallback AI analysis with extract  | Extract from PDF / Analyze document           | Normalize PDF (AI)                            |                                                                 |
| Analyze document         | Google Gemini AI (LangChain)      | AI fallback for PDF text extraction       | Merge                                         | Normalize PDF (AI)                            |                                                                 |
| Normalize PDF (AI)       | Set                            | Normalize AI-extracted PDF text           | Analyze document                              | get_message (File message)                     |                                                                 |
| Download RTF             | Telegram (Download File)           | Download RTF document                     | Switch                                        | Extract from RTF                              |                                                                 |
| Extract from RTF         | Extract from File                | Extract RTF content                       | Download RTF                                  | Get RTF data                                  |                                                                 |
| Get RTF data             | Code                           | Format RTF extraction data                | Extract from RTF                              | Normalize RTF                                 |                                                                 |
| Normalize RTF            | Set                            | Normalize extracted RTF                   | Get RTF data                                  | get_message (File message)                     |                                                                 |
| Download TEXT FILE       | Telegram (Download File)           | Download text file                        | Switch                                        | Extract from File                             |                                                                 |
| Extract from File        | Extract from File                | Extract text content                      | Download TEXT FILE                            | Normalize text file                           |                                                                 |
| Normalize text file      | Set                            | Normalize extracted text                   | Extract from File                             | get_message (File message)                     |                                                                 |
| Download XML             | Telegram (Download File)           | Download XML document                     | Switch                                        | Extract from XML                              |                                                                 |
| Extract from XML         | Extract from File                | Extract XML content                       | Download XML                                  | Normalize XML                                 |                                                                 |
| Normalize XML            | Set                            | Normalize extracted XML                   | Extract from XML                              | get_message (File message)                     |                                                                 |
| Download XLSX            | Telegram (Download File)           | Download XLSX spreadsheet                 | Switch                                        | Extract from XLSX                             |                                                                 |
| Extract from XLSX        | Extract from File                | Extract XLSX content                      | Download XLSX                                 | Get RTF data1                                 |                                                                 |
| Get RTF data1            | Code                           | Format XLSX extraction data                | Extract from XLSX                             | Normalize XLSX                                |                                                                 |
| Normalize XLSX           | Set                            | Normalize extracted XLSX                   | Get RTF data1                                 | get_message (File message)                     |                                                                 |
| get_error_message        | Set                            | Generate error message for unsupported files | Switch (fallback)                            | get_message (File message)                     |                                                                 |
| get_error_message1       | Set                            | Generate error message for unsupported media | Input Message Router1 fallback               | get_message (Media message)                    |                                                                 |
| get_message (File message) | Set                           | Prepare file message for AI                | Normalization nodes / error messages          | Captions?1                                   |                                                                 |
| Captions?1               | If                             | Checks if captions exist                   | get_message (File message) / get_message (Media message) | Media_group?2 / Media_group?3                  |                                                                 |
| Media_group?2            | If                             | Checks for media_group_id presence         | Captions?1 (true branch)                      | Insert documents in media_group, Get_file_and_captions |                                                                 |
| Insert documents in media_group | Postgres (Insert)            | Insert file descriptions with captions     | Media_group?2                                 | Insert media_queue with captions (Trigger)    |                                                                 |
| Insert media_queue with captions (Trigger) | Postgres (Insert)   | Add media group to processing queue with captions | Insert documents in media_group               |                                               |                                                                 |
| Get_file_and_captions    | Set                            | Format combined file and caption message   | Media_group?2                                 | AI Agent1                                    |                                                                 |
| Media_group?3            | If                             | Checks for media_group_id presence (no captions) | Captions?1 (false branch)                     | Insert documents in media_group1, Get_only_file |                                                                 |
| Insert documents in media_group1 | Postgres (Insert)            | Insert file descriptions without captions   | Media_group?3                                 | Wait                                          |                                                                 |
| Wait                     | Wait                           | Delay to allow captions insertion            | Insert documents in media_group1              | Insert media_queue (Trigger)                    |                                                                 |
| Insert media_queue (Trigger) | Postgres (Insert)             | Insert media group into queue without captions | Wait                                          |                                               |                                                                 |
| Get_only_file            | Set                            | Format message for single file no captions  | Media_group?3                                 | AI Agent1                                    |                                                                 |
| Media_queue Trigger      | Postgres Trigger               | Trigger on new media_queue entries           |                                               | get_chat_id                                   | Green Section – Media Queue Trigger                              |
| get_chat_id              | Set                            | Extract chat_id and media_group_id           | Media_queue Trigger                           | Wait for all the files                         |                                                                 |
| Wait for all the files   | Wait                           | Delay to ensure all media files inserted      | get_chat_id                                  | Get all files from group_id                    |                                                                 |
| Get all files from group_id | Postgres (Select)             | Retrieve all file descriptions for group     | Wait for all the files                        | unified_variables                             |                                                                 |
| unified_variables        | Code                           | Aggregate captions and file descriptions      | Get all files from group_id                   | Get_message (multiple files)                    |                                                                 |
| Get_message (multiple files) | Set                          | Prepare unified message for AI                | unified_variables                             | AI Agent1                                    |                                                                 |
| AI Agent1                | LangChain Agent (Google Gemini) | AI processing of all message types            | Multiple Set nodes (text, audio/video, file) | MarkdownV2                                    | Purple Section – AI Agent & Output Formatting                   |
| Postgres Chat Memory     | LangChain Memory Postgres      | Store and retrieve conversation context       | Connected to AI Agent1                        |                                               |                                                                 |
| MarkdownV2               | Code                           | Escape Telegram MarkdownV2 and chunk messages | AI Agent1                                     | Send a text message1                          |                                                                 |
| Send a text message1     | Telegram (Send Message)        | Send AI response to Telegram chat              | MarkdownV2                                    |                                               |                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create PostgreSQL Tables:**  
   - Add a **Manual Trigger** node named `When clicking ‘Execute workflow’`.  
   - Add **Postgres** node `Create Tables` connected from the manual trigger.  
   - Configure with SQL to create tables: `media_group`, `media_queue`, `chat_histories` with appropriate schemas and indexes.  
   - Configure Postgres credentials.

2. **Telegram Listener and Router:**  
   - Add **Telegram Trigger** node `Telegram Trigger`, listen for `message` updates, disable download.  
   - Connect to **Telegram** node `Typing…` to send "typing" action to chat.  
   - Connect to **Switch** node `Input Message Router1` with conditions detecting message types (text, voice, video_note, image, audio, video, document).  
   - Use JSON path expressions to detect existence of media fields.

3. **Media Download & MIME Fix:**  
   - For each media type branch from the router, add a corresponding **Telegram** node to download the file using the correct file ID expression.  
   - Add code nodes (`Fix mime`, `Fix mime1`, `Fix mime4`, `Fix mime5`, `Fix mime6`) to map file extensions to MIME types and update binary data accordingly.  
   - Connect each MIME fix node to the appropriate Google Gemini AI analyze node (audio, video, image, voice message, video note).

4. **Document Download & Extraction:**  
   - For document branch, add Telegram download nodes for each supported file type (CSV, HTML, ICS, JSON, ODS, PDF, RTF, TEXT, XML, XLSX).  
   - For each, add corresponding extract nodes (`Extract from CSV`, `Extract from ICS`, etc.) configured for the correct operation (e.g., `fromIcs`, `pdf`, `xlsx`).  
   - Add normalization Set nodes wrapping extraction results into a `data` field.  
   - For PDF, add conditional node `Text?` to check if extracted text exists, if not, connect to AI analysis node `Analyze document` and normalize afterwards.  
   - Connect all processed document outputs to `get_message (File message)` Set node to format message for AI input.  
   - Add error handling Set nodes to produce error messages for unsupported files.

5. **Media Group & Caption Handling:**  
   - Add If nodes `Captions?1`, `Media_group?2`, `Media_group?3` to check presence of captions and media group ID using expressions on Telegram message data.  
   - Add Postgres insert nodes to `media_group` and `media_queue` tables as per the four cases described:  
     1. Captions + Media Group: insert media_group then media_queue with captions.  
     2. Captions only: combine captions and file description, send directly to AI.  
     3. No Captions + Media Group: insert media_group, wait 2 seconds, then insert media_queue.  
     4. No Captions + No Media Group: send file description directly to AI.  
   - Use Set nodes to format messages and chat IDs for AI.

6. **Media Queue Trigger & Aggregation:**  
   - Add **Postgres Trigger** node `Media_queue Trigger` watching `media_queue` table.  
   - Connect to `get_chat_id` Set node extracting `chat_id` and `media_group_id`.  
   - Connect to `Wait for all the files` node (delay 5 seconds).  
   - Connect to Postgres select node `Get all files from group_id` querying `media_group` table by `media_group_id`.  
   - Add code node `unified_variables` to aggregate captions and file descriptions into a single message string.  
   - Add Set node `Get_message (multiple files)` preparing final message and chat ID.  
   - Connect to AI Agent node.

7. **AI Agent & Memory Setup:**  
   - Add LangChain `AI Agent1` node configured with Google Gemini model (e.g., `models/gemini-2.5-pro`), input text from formatted messages.  
   - Add LangChain `Postgres Chat Memory` node linked to `AI Agent1` to store conversation history, configured with chat_histories table and session key from chat_id.  
   - Connect AI Agent output to `MarkdownV2` code node for Telegram-safe formatting and chunking.

8. **Output to Telegram:**  
   - Connect `MarkdownV2` node to a Telegram node `Send a text message1` to send the AI response back to the user chat with MarkdownV2 parse mode enabled.  
   - Configure credentials for Telegram API and Postgres accordingly.

9. **Error Handling:**  
   - Add fallback branches from switches or extraction nodes to error message Set nodes that create user-friendly failure messages.  
   - Ensure these messages are sent to AI Agent or directly to Telegram to inform user of unsupported file types.

10. **Testing & Validation:**  
    - Test each media type input via Telegram.  
    - Validate database inserts and AI responses.  
    - Adjust wait times if files or captions are missed.  
    - Verify message chunking and Telegram Markdown formatting.

---

### 5. General Notes & Resources

| Note Content                                                                                                                   | Context or Link                                                                                         |
|-------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| **Gray Section – Database Setup:** Explains purpose of PostgreSQL tables for media grouping, queueing, and chat history.       | Sticky Note6                                                                                           |
| **Green Section – Media Queue Trigger:** Describes how new media queue entries trigger aggregation and AI processing.         | Sticky Note7                                                                                           |
| **Blue Section – Telegram Trigger & Message Type Handling:** Details routing by Telegram message type and initial processing. | Sticky Note8                                                                                           |
| **Red Section – Document Processing:** Covers document extraction and fallback AI analysis for PDFs.                          | Sticky Note9                                                                                           |
| **Yellow Section – File & Caption Handling:** Explains conditional inserts into media_group and media_queue based on captions.| Sticky Note10                                                                                          |
| **Purple Section – AI Agent & Output Formatting:** Describes AI integration, chat memory, and output formatting for Telegram. | Sticky Note11                                                                                          |
| **Special Thanks:** Credit to Ezema Gingsley Chibuzo for the original workflow inspiration.                                   | Sticky Note12: [Create a Multi-Modal Telegram Support Bot with GPT-4 and Supabase RAG](https://n8n.io/workflows/5589-create-a-multi-modal-telegram-support-bot-with-gpt-4-and-supabase-rag/) |

---

**Disclaimer:** The provided text is derived exclusively from an n8n automated workflow. All processed data is legal, public, and handled in accordance with relevant content policies. No illegal or offensive content is involved.