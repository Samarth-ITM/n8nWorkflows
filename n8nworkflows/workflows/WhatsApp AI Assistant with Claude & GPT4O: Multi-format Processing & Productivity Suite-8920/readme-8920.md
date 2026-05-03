WhatsApp AI Assistant with Claude & GPT4O: Multi-format Processing & Productivity Suite

https://n8nworkflows.xyz/workflows/whatsapp-ai-assistant-with-claude---gpt4o--multi-format-processing---productivity-suite-8920


# WhatsApp AI Assistant with Claude & GPT4O: Multi-format Processing & Productivity Suite

### 1. Workflow Overview

This n8n workflow implements a comprehensive WhatsApp AI Assistant integrating Claude and GPT-4O language models. It is designed to handle multi-format inputs (text, audio, images, PDF documents) sent through WhatsApp and deliver intelligent, context-aware responses while interfacing with various productivity tools.

**Target Use Cases:**
- Receiving and classifying WhatsApp messages by format
- Transcribing voice messages
- Analyzing image content with AI vision models
- Extracting text from PDF documents
- Processing user queries with advanced AI agents augmented by external tools and databases
- Managing contacts, emails, calendar events, and document searches
- Delivering responses in text or synthesized audio

**Logical Blocks:**

- **1.1 Message Reception and Classification:** Capture WhatsApp messages and route them based on content type (text, audio, image, document).
- **1.2 Content Processing by Format:** Download, convert, or extract data from the received media (audio transcription, image analysis, PDF extraction).
- **1.3 AI Assistant Processing:** Process the formatted content through a personal AI assistant leveraging multiple language models, memory, and external tools (Airtable, Gmail, Google Calendar, SerpAPI, Discord, Calculator).
- **1.4 Response Delivery:** Format and send the AI-generated response back to the user via WhatsApp, including optional text-to-speech conversion for audio replies.
- **1.5 Auxiliary Utilities:** MIME type correction and error handling for unsupported formats.

---

### 2. Block-by-Block Analysis

#### 2.1 Message Reception and Classification

**Overview:**  
This block listens to incoming WhatsApp messages, detects the message format, and routes the flow accordingly. It ensures only supported content types are processed.

**Nodes Involved:**  
- WhatsApp Trigger  
- Input type (Switch)  
- Sticky Note7 (visual label)

**Node Details:**

- **WhatsApp Trigger**  
  - Type: WhatsApp webhook trigger  
  - Role: Entry point capturing incoming WhatsApp messages, configured to listen for message updates  
  - Input: Incoming WhatsApp message webhook  
  - Output: Raw message JSON with various possible content types (text, audio, image, document)  
  - Credentials: WhatsApp OAuth2  
  - Edge Cases: Missing message data, unsupported message types, webhook delivery failure  

- **Input type**  
  - Type: Switch node  
  - Role: Classifies the message into one of Text, Voice, Image, Document, or fallback (extra) based on presence of respective JSON properties  
  - Configuration: Checks existence of keys in message JSON to route flow  
  - Output connections: Text → Text node, Voice → Get Audio Url, Image → Get Image Url, Document → Only PDF File node  
  - Edge Cases: Messages with multiple media types, missing media IDs, fallback routing for unsupported types  

- **Sticky Note7**  
  - Visual label: "Phase 1: Message Reception and Classification"  

---

#### 2.2 Content Processing by Format

**Overview:**  
Processes each media type accordingly: text is passed as-is, audio files are downloaded and transcribed, images are downloaded and analyzed with AI vision, documents are validated (PDF only), downloaded, and text extracted.

**Nodes Involved:**  
- Text  
- Get Audio Url → Download Audio → Transcribe Audio → Audio  
- Get Image Url → Download Image → Analyze Image → Image  
- Only PDF File → Get File Url → Download File → Extract from File → File  
- Incorrect format (error message node)  
- Sticky Notes 1, 2, 3, 4, 5, 12, 14 (content explanations)

**Node Details:**

- **Text**  
  - Type: Set node  
  - Role: Extracts and formats text message content for AI agent  
  - Key expression: `={{ $json.messages[0].text.body }}` assigned to "text" field  
  - Input: From Input type (Text)  
  - Output: To Agent personnel node  
  - Edge Cases: Empty or malformed text body  

- **Get Audio Url**  
  - Type: WhatsApp node (media URL retrieval)  
  - Role: Retrieves temporary download URL for audio message via WhatsApp API  
  - Input: Media ID extracted from incoming audio message JSON  
  - Output: To Download Audio node  
  - Edge Cases: Expired or invalid media URL, API failures  

- **Download Audio**  
  - Type: HTTP Request  
  - Role: Downloads audio file with authentication  
  - Config: Uses generic HTTP header auth with WhatsApp Growth AI Agent IA credentials  
  - Input: URL from Get Audio Url  
  - Output: To Transcribe Audio node  
  - Edge Cases: Network errors, invalid URLs, authentication failures  

- **Transcribe Audio**  
  - Type: OpenAI node (Langchain)  
  - Role: Transcribes audio to text using OpenAI Whisper model, language set to English  
  - Input: Binary audio downloaded  
  - Output: To Audio node (Set)  
  - Edge Cases: Low-quality audio, unsupported formats, transcription errors  

- **Audio**  
  - Type: Set node  
  - Role: Formats transcribed text for AI agent  
  - Key expression: `={{ $json.text }}` assigned to "text" field  
  - Input: From Transcribe Audio  
  - Output: To Agent personnel node  

- **Get Image Url**  
  - Type: WhatsApp node (media URL retrieval)  
  - Role: Retrieves image URL from WhatsApp API  
  - Input: Image media ID from message JSON  
  - Output: To Download Image node  

- **Download Image**  
  - Type: HTTP Request  
  - Role: Downloads image file with authentication  
  - Input: URL from Get Image Url  
  - Output: To Analyze Image node  

- **Analyze Image**  
  - Type: OpenAI node (Langchain)  
  - Role: Performs detailed image analysis and description using GPT-4O-mini model, input is base64 image  
  - Configuration: Custom prompt in French instructing detailed image description  
  - Output: To Image (Set) node  

- **Image**  
  - Type: Set node  
  - Role: Combines user caption and AI image analysis into formatted text for AI agent  
  - Input: From Analyze Image  
  - Output: To Agent personnel node  

- **Only PDF File**  
  - Type: If node  
  - Role: Checks if the document MIME type is "application/pdf" to filter out unsupported formats  
  - Output: True → Get File Url, False → Incorrect format node  

- **Get File Url**  
  - Type: WhatsApp node (media URL retrieval)  
  - Role: Retrieves document URL  
  - Input: PDF media ID  
  - Output: To Download File node  

- **Download File**  
  - Type: HTTP Request  
  - Role: Downloads PDF document with authentication  
  - Output: To Extract from File node  

- **Extract from File**  
  - Type: Extract From File node  
  - Role: Extracts text content from PDF document  
  - Output: To File (Set) node  

- **File**  
  - Type: Set node  
  - Role: Formats extracted PDF text with user caption for AI agent  
  - Input: Extracted text  
  - Output: To Agent personnel node  

- **Incorrect format**  
  - Type: WhatsApp node (send message)  
  - Role: Sends error message "Sorry but you can only send PDF files" to user when document format is invalid  
  - Input: From Only PDF File (false branch)  

- **Sticky Notes (1,2,3,4,5,12,14)**  
  - Provide detailed explanations and instructions for message types and processing steps  

---

#### 2.3 AI Assistant Processing

**Overview:**  
Processes the formatted input text using a personal AI assistant node that integrates Claude Sonnet 4 and GPT-4O models, enriched with memory and external tool integrations to provide advanced, context-aware responses.

**Nodes Involved:**  
- Agent personnel  
- Postgres Chat Memory  
- SerpAPI  
- BDD mails (Airtable)  
- Ajouter un mail (Airtable)  
- Search drive (Google Drive)  
- Create event Google Calendar  
- Get many events in Google Calendar  
- Send a mail (Gmail)  
- Search mails (Gmail)  
- MP Discord (Discord)  
- Calculator  
- Anthropic Chat Model  
- Sticky Note9, 15

**Node Details:**

- **Agent personnel**  
  - Type: Langchain Agent node  
  - Role: Central AI processing node interpreting user prompt, managing context, invoking tools, and generating responses  
  - Configuration:  
    - System message defines the assistant as a WhatsApp personal assistant for Allan, specifying tasks and tools available  
    - Input text formatted with the user request and current date/time  
  - Inputs: Receives formatted text from all content processing Set nodes (Text, Audio, Image, File)  
  - AI Language Models: Uses Claude Sonnet 4 (Anthropic) and GPT-4O mini models  
  - AI Memory: Connected to Postgres Chat Memory node for conversation context persistence keyed by WhatsApp user ID  
  - Tools: Integrated with external productivity tools (SerpAPI, Airtable, Gmail, Google Calendar, Google Drive, Discord, Calculator) for dynamic query execution  
  - Outputs:  
    - Main: AI text response  
    - ai_tool, ai_languageModel, ai_memory: For tool calls and sub-model interactions  

- **Postgres Chat Memory**  
  - Type: Langchain Postgres Memory node  
  - Role: Maintains chat history to provide conversation continuity  
  - Configuration: Custom session key based on WhatsApp contact ID, context window length 20 messages  
  - Credentials: Postgres database  

- **SerpAPI**  
  - Type: Langchain tool node  
  - Role: Performs web searches on user request  

- **BDD mails (Airtable)**  
  - Type: Airtable Tool node  
  - Role: Searches contact emails in Airtable base  

- **Ajouter un mail (Airtable)**  
  - Type: Airtable Tool node  
  - Role: Adds new email contacts to Airtable base  

- **Search drive (Google Drive)**  
  - Type: Google Drive Tool node  
  - Role: Searches documents/files in Google Drive  

- **Create event Google Calendar**  
  - Type: Google Calendar Tool node  
  - Role: Creates calendar events  

- **Get many events in Google Calendar**  
  - Type: Google Calendar Tool node  
  - Role: Retrieves calendar events within a time range  

- **Send a mail (Gmail)**  
  - Type: Gmail Tool node  
  - Role: Sends emails  

- **Search mails (Gmail)**  
  - Type: Gmail Tool node  
  - Role: Searches received emails  

- **MP Discord**  
  - Type: Discord Tool node  
  - Role: Sends private messages on Discord  

- **Calculator**  
  - Type: Langchain Calculator tool  
  - Role: Performs mathematical calculations  

- **Anthropic Chat Model**  
  - Type: Langchain language model node  
  - Role: Provides additional AI language model capabilities (Claude Sonnet 4)  

- **Sticky Notes 9, 15**  
  - Visual labels describing AI assistant capabilities and tools integration  

---

#### 2.4 Response Delivery

**Overview:**  
Delivers the AI-generated response to the user via WhatsApp. For audio inputs, the text response is converted to speech; otherwise, text is sent directly. MIME-type fixes ensure WhatsApp audio compatibility.

**Nodes Involved:**  
- From audio to audio? (If node)  
- Generate Audio Response (OpenAI TTS)  
- Fix mimeType for Audio (Code node)  
- Send audio (WhatsApp)  
- Send message (WhatsApp)  
- Sticky Note10, 16

**Node Details:**

- **From audio to audio?**  
  - Type: If node  
  - Role: Checks if original message was audio to determine if response should be audio  
  - Condition: Existence of audio object in incoming message JSON  

- **Generate Audio Response**  
  - Type: OpenAI node (Langchain)  
  - Role: Converts AI text response to audio using OpenAI text-to-speech (voice "fable")  
  - Input: AI response text from Agent personnel node  

- **Fix mimeType for Audio**  
  - Type: Code node (JavaScript)  
  - Role: Corrects MIME type from "audio/mp3" to "audio/mpeg" for WhatsApp compatibility  
  - Input: Binary audio data from Generate Audio Response  

- **Send audio**  
  - Type: WhatsApp node (send media)  
  - Role: Sends audio message back to user via WhatsApp  
  - Credentials: WhatsApp Growth AI  

- **Send message**  
  - Type: WhatsApp node (send text)  
  - Role: Sends text message response back to user  
  - Input: AI text output  
  - Credentials: WhatsApp Growth AI  

- **Sticky Notes 10, 16**  
  - Visual labels explaining response delivery logic and formats  

---

#### 2.5 Auxiliary Utilities

**Overview:**  
Handles MIME type fixes and error messaging for unsupported document formats.

**Nodes Involved:**  
- Fix mimeType for Audio (Code node)  
- Incorrect format (WhatsApp send message)  

---

### 3. Summary Table

| Node Name               | Node Type                          | Functional Role                                    | Input Node(s)               | Output Node(s)              | Sticky Note                                                                                  |
|-------------------------|----------------------------------|---------------------------------------------------|-----------------------------|-----------------------------|----------------------------------------------------------------------------------------------|
| WhatsApp Trigger        | WhatsApp Trigger                 | Entry point capturing WhatsApp messages           | -                           | Input type                  | See Sticky Note7: Phase 1: Message Reception and Classification                              |
| Input type              | Switch                          | Classify message by type (Text, Audio, Image, Doc)| WhatsApp Trigger            | Text, Get Audio Url, Get Image Url, Only PDF File | See Sticky Note7                                                                           |
| Text                    | Set                             | Format text message for AI                         | Input type (Text)            | Agent personnel             | See Sticky Note4: Text message processing                                                   |
| Get Audio Url           | WhatsApp mediaUrlGet            | Retrieve audio download URL                         | Input type (Voice)           | Download Audio              | See Sticky Note1: Audio message processing                                                  |
| Download Audio          | HTTP Request                    | Download audio file                                | Get Audio Url                | Transcribe Audio            | See Sticky Note1                                                                            |
| Transcribe Audio        | OpenAI (Langchain)              | Transcribe audio to text                           | Download Audio               | Audio                      | See Sticky Note1                                                                            |
| Audio                   | Set                             | Format transcribed audio text for AI              | Transcribe Audio             | Agent personnel             | See Sticky Note1                                                                            |
| Get Image Url           | WhatsApp mediaUrlGet            | Retrieve image download URL                        | Input type (Image)           | Download Image              | See Sticky Note2, 12: Image processing                                                     |
| Download Image          | HTTP Request                    | Download image file                               | Get Image Url                | Analyze Image               | See Sticky Note2, 12                                                                        |
| Analyze Image           | OpenAI (Langchain)              | Analyze image content with GPT-4O-mini            | Download Image               | Image                      | See Sticky Note2, 12                                                                        |
| Image                   | Set                             | Combine user caption and AI image analysis         | Analyze Image                | Agent personnel             | See Sticky Note2, 12                                                                        |
| Only PDF File           | If                             | Validate document MIME type is PDF                 | Input type (Document)        | Get File Url / Incorrect format | See Sticky Note3, 14: PDF document processing                                               |
| Get File Url            | WhatsApp mediaUrlGet            | Retrieve document download URL                     | Only PDF File (true)         | Download File               | See Sticky Note3, 14                                                                        |
| Download File           | HTTP Request                    | Download PDF document                              | Get File Url                 | Extract from File           | See Sticky Note3, 14                                                                        |
| Extract from File       | Extract From File               | Extract text from PDF                              | Download File                | File                       | See Sticky Note3, 14                                                                        |
| File                    | Set                             | Format extracted PDF text for AI                   | Extract from File            | Agent personnel             | See Sticky Note3, 14                                                                        |
| Incorrect format        | WhatsApp send message           | Send error message for unsupported document format| Only PDF File (false)        | -                          | See Sticky Note3, 14                                                                        |
| Agent personnel         | Langchain Agent                | AI personal assistant processing and integration  | Text, Audio, Image, File     | From audio to audio?, AI tools, memory, languageModel | See Sticky Note9, 15: AI assistant processing and tools integration                         |
| Postgres Chat Memory    | Langchain Postgres Memory       | Chat history memory                               | -                           | Agent personnel (ai_memory) | See Sticky Note9, 15                                                                       |
| SerpAPI                 | Langchain tool SerpAPI          | Internet search tool                               | -                           | Agent personnel (ai_tool)   | See Sticky Note9, 15                                                                       |
| BDD mails               | Airtable Tool                  | Search email contacts                              | -                           | Agent personnel (ai_tool)   | See Sticky Note9, 15                                                                       |
| Ajouter un mail         | Airtable Tool                  | Add email contacts                                | -                           | Agent personnel (ai_tool)   | See Sticky Note9, 15                                                                       |
| Search drive            | Google Drive Tool               | Search Google Drive                               | -                           | Agent personnel (ai_tool)   | See Sticky Note9, 15                                                                       |
| Create event Google Calendar | Google Calendar Tool         | Create calendar events                            | -                           | Agent personnel (ai_tool)   | See Sticky Note9, 15                                                                       |
| Get many events in Google Calendar | Google Calendar Tool         | Retrieve calendar events                          | -                           | Agent personnel (ai_tool)   | See Sticky Note9, 15                                                                       |
| Send a mail             | Gmail Tool                     | Send emails                                       | -                           | Agent personnel (ai_tool)   | See Sticky Note9, 15                                                                       |
| Search mails            | Gmail Tool                     | Search received emails                            | -                           | Agent personnel (ai_tool)   | See Sticky Note9, 15                                                                       |
| MP Discord              | Discord Tool                   | Send private message on Discord                   | -                           | Agent personnel (ai_tool)   | See Sticky Note9, 15                                                                       |
| Calculator              | Langchain Calculator Tool       | Perform math calculations                         | -                           | Agent personnel (ai_tool)   | See Sticky Note9, 15                                                                       |
| Anthropic Chat Model    | Langchain LM Anthropic          | Additional AI language model                       | -                           | Agent personnel (ai_languageModel) | See Sticky Note9, 15                                                                       |
| From audio to audio?    | If                             | Check if original input was audio                 | Agent personnel             | Generate Audio Response / Send message | See Sticky Note10, 16: Response delivery                                               |
| Generate Audio Response | OpenAI (Langchain)              | Convert text response to speech                   | From audio to audio?         | Fix mimeType for Audio      | See Sticky Note10, 16                                                                      |
| Fix mimeType for Audio  | Code                            | Fix MIME type for audio compatibility             | Generate Audio Response      | Send audio                 | See Sticky Note10, 16                                                                      |
| Send audio              | WhatsApp send media            | Send audio message                                 | Fix mimeType for Audio       | -                          | See Sticky Note10, 16                                                                      |
| Send message            | WhatsApp send message          | Send text message                                  | From audio to audio? (false) | -                          | See Sticky Note10, 16                                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create WhatsApp Trigger node**  
   - Type: WhatsApp Trigger  
   - Configure webhook to listen for message updates  
   - Set credentials with WhatsApp OAuth2  

2. **Add Switch node (Input type) to classify message**  
   - Configure branches with conditions:  
     - Text: check if `messages[0].text.body` exists  
     - Voice: check if `messages[0].audio` exists  
     - Image: check if `messages[0].image` exists  
     - Document: check if `messages[0].document` exists  
   - Connect WhatsApp Trigger output to this node  

3. **Text Processing Path**  
   - Add Set node (Text)  
   - Configure to assign `text` = `{{$json.messages[0].text.body}}`  
   - Connect Input type Text output to this node  

4. **Audio Processing Path**  
   - Add WhatsApp node (Get Audio Url)  
     - Configure resource: media, operation: mediaUrlGet  
     - Use mediaGetId from input: `{{$json.messages[0].audio.id}}`  
     - Connect Input type Voice output to this node  
   - Add HTTP Request node (Download Audio)  
     - URL from previous node output  
     - Use generic HTTP header authentication with WhatsApp Growth AI Agent IA credentials  
   - Add OpenAI node (Transcribe Audio)  
     - Resource: audio, operation: transcribe  
     - Language: English  
     - Use OpenAI credentials  
   - Add Set node (Audio)  
     - Assign `text` = `{{$json.text}}`  
   - Connect nodes sequentially: Get Audio Url -> Download Audio -> Transcribe Audio -> Audio  

5. **Image Processing Path**  
   - Add WhatsApp node (Get Image Url)  
     - MediaGetId from `{{$json.messages[0].image.id}}`  
   - Add HTTP Request node (Download Image)  
     - Use generic HTTP header auth  
   - Add OpenAI Langchain node (Analyze Image)  
     - Resource: image, operation: analyze  
     - Model: gpt-4o-mini  
     - Input type: base64 image  
     - Use prompt in French that instructs detailed image description (as per original)  
   - Add Set node (Image)  
     - Combine user caption and AI description, assign to `text`  
   - Connect nodes sequentially: Get Image Url -> Download Image -> Analyze Image -> Image  

6. **Document Processing Path (PDF only)**  
   - Add If node (Only PDF File)  
     - Condition: check if `messages[0].document.mime_type` equals `application/pdf`  
   - For True branch:  
     - Add WhatsApp node (Get File Url)  
       - MediaGetId from document ID  
     - Add HTTP Request node (Download File)  
       - Use generic HTTP header auth  
     - Add Extract From File node  
       - Operation: pdf  
     - Add Set node (File)  
       - Format text combining user caption and extracted content  
     - Connect: Only PDF File -> Get File Url -> Download File -> Extract From File -> File  
   - For False branch:  
     - Add WhatsApp node (Incorrect format)  
       - Send text message "Sorry but you can only send PDF files" to message sender  

7. **Connect all Set nodes (Text, Audio, Image, File) to AI Agent node**  
   - Add Langchain Agent node (Agent personnel)  
   - Configure system prompt defining assistant capabilities, tools, and instructions  
   - Use OpenAI and Anthropic credentials for GPT-4O-mini and Claude Sonnet 4 models  
   - Connect memory node (Postgres Chat Memory) with session key as WhatsApp user ID  
   - Connect Langchain tool nodes: SerpAPI, Airtable (BDD mails, Ajouter un mail), Gmail (Send a mail, Search mails), Google Calendar (Create event, Get many events), Google Drive (Search drive), Discord (MP Discord), Calculator  
   - Connect these tools to respective ai_tool or ai_languageModel inputs of Agent personnel  

8. **Response Delivery**  
   - Add If node (From audio to audio?)  
     - Condition: check if original message contains audio  
   - For True branch:  
     - Add OpenAI Langchain node (Generate Audio Response)  
       - Resource: audio, voice: "fable"  
     - Add Code node (Fix mimeType for Audio)  
       - JavaScript to fix MIME type from audio/mp3 to audio/mpeg  
     - Add WhatsApp node (Send audio)  
       - Send audio message to original sender using WhatsApp Growth AI credentials  
     - Connect: Agent personnel -> From audio to audio? -> Generate Audio Response -> Fix mimeType for Audio -> Send audio  
   - For False branch:  
     - Add WhatsApp node (Send message)  
       - Send text message using AI response text  
     - Connect: Agent personnel -> From audio to audio? (false) -> Send message  

9. **Add Sticky Notes** at appropriate positions with content describing phases and node roles as per original workflow for clarity and documentation.

10. **Configure Credentials:**  
    - WhatsApp OAuth2 for trigger and send nodes  
    - HTTP Header Auth for media downloads with WhatsApp Growth AI Agent IA credentials  
    - OpenAI API for GPT and transcription  
    - Anthropic API for Claude model  
    - Postgres credentials for chat memory  
    - Airtable personal access token  
    - Gmail OAuth2 for email operations  
    - Google Calendar OAuth2  
    - Google Drive OAuth2  
    - Discord Bot API  

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                 |
|----------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow supports multi-format WhatsApp messages: text, audio, image, PDF documents with AI analysis and productivity tool integrations. | Workflow description and design overview                                                     |
| For audio transcription, OpenAI Whisper model is used supporting multi-language transcription.     | Audio processing phase                                                                         |
| Image analysis uses GPT-4O-mini vision capabilities with French prompt for detailed image description. | Image processing phase                                                                        |
| AI assistant integrates Claude Sonnet 4 and GPT-4O models with PostgreSQL chat memory and multiple productivity tools including Airtable, Gmail, Google Calendar, Google Drive, Discord, SerpAPI, Calculator. | AI assistant processing phase                                                                |
| Document processing only accepts PDFs; others trigger friendly error message.                       | Document processing phase                                                                      |
| Audio response generation uses OpenAI TTS with MIME type fix for WhatsApp compatibility.            | Response delivery phase                                                                        |
| Workflow built with n8n, leveraging Langchain nodes and WhatsApp official APIs.                     | Platform and technology notes                                                                 |
| For assistance or enterprise customizations, contact Growth-AI.fr via LinkedIn:                     | https://www.linkedin.com/in/allanvaccarizi/ and https://www.linkedin.com/in/hugo-marinier-%F0%9F%A7%B2-6537b633/ |

---

**Disclaimer:** The provided text is exclusively extracted from an automated workflow created with n8n, adhering strictly to content policies and containing no illegal or protected elements. All data processed is legal and public.