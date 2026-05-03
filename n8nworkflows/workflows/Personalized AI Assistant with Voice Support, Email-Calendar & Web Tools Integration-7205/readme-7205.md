Personalized AI Assistant with Voice Support, Email/Calendar & Web Tools Integration

https://n8nworkflows.xyz/workflows/personalized-ai-assistant-with-voice-support--email-calendar---web-tools-integration-7205


# Personalized AI Assistant with Voice Support, Email/Calendar & Web Tools Integration

---

### 1. Workflow Overview

This workflow implements a **Personalized AI Assistant with Voice Support and Integrated Email, Calendar, and Web Tools**. Named "My Funny Assistant," it is designed to interact with users via chat, process both text and audio inputs, and provide responses infused with a distinct sassy, glamorous AI personality called Nova. It supports multiple integrations including Telegram for audio messages, Gmail and Google Calendar for email and scheduling, Airtable for contact management, Google Drive for file search, and web tools like SerpAPI, Wikipedia, Hacker News, and a Calculator for enriched AI responses.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Preprocessing**: Receives chat messages or audio notes, distinguishes input types, and transcribes audio when necessary.
- **1.2 AI Processing & Memory Management**: Uses AI language models and memory buffers to process user input, maintain conversational context, and generate responses with a defined personality.
- **1.3 External Tools & Integrations**: Interfaces with Gmail, Google Calendar, Airtable, Google Drive, SerpAPI, Wikipedia, Hacker News, and Calculator to fulfill user requests related to emails, contacts, events, web search, and calculations.
- **1.4 Response Output & Personality Layer**: Applies a "bitchy luxe" persona to the AI's output, styling responses and managing the user interface experience.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Preprocessing

**Overview:**  
This block captures incoming messages (text or audio) from users and prepares them for AI processing. It routes audio messages for transcription and text messages directly to the AI.

**Nodes Involved:**  
- When chat message received  
- Switch  
- Get a file  
- Transcribe a recording  
- Edit Fields  

**Node Details:**

- **When chat message received**  
  - *Type:* Chat Trigger (LangChain)  
  - *Role:* Entry point webhook receiving public chat messages.  
  - *Configuration:* Public access enabled; custom branding with a sassy assistant introduction and CSS theme for UI styling; allows file uploads.  
  - *Expressions:* Uses sessionId from incoming message for session management.  
  - *Inputs:* Webhook trigger from user chat.  
  - *Outputs:* Connects to Switch node.  
  - *Edge Cases:* File uploads can fail if unsupported formats; webhook downtime affects message reception.

- **Switch**  
  - *Type:* Switch (base node)  
  - *Role:* Routes the workflow based on message content type: audio video_note, text input, or error.  
  - *Configuration:*  
    - Checks if the message contains a `video_note.file_id` → routes to Audio path.  
    - Checks if `chatInput` (text) exists → routes to Text path.  
    - Checks for error flag → routes to Error path (not connected further).  
  - *Inputs:* From "When chat message received" node.  
  - *Outputs:* Audio → Get a file; Text → Edit Fields.  
  - *Edge Cases:* Missing or malformed message content can lead to unhandled branches.

- **Get a file**  
  - *Type:* Telegram node  
  - *Role:* Downloads the audio file from Telegram using the file ID from the video_note.  
  - *Configuration:* Uses Telegram API credentials; fileId dynamically taken from incoming message.  
  - *Inputs:* From Switch node Audio output.  
  - *Outputs:* Transcribe a recording.  
  - *Edge Cases:* File download failures, Telegram API rate limits, invalid file IDs.

- **Transcribe a recording**  
  - *Type:* OpenAI LangChain node (audio transcribe)  
  - *Role:* Converts audio recording to text using OpenAI’s transcription service.  
  - *Configuration:* Uses OpenAI API credentials; operation set to "transcribe."  
  - *Inputs:* From Get a file node.  
  - *Outputs:* AI Agent - Nova node.  
  - *Edge Cases:* Audio quality issues, transcription failures, API timeouts.

- **Edit Fields**  
  - *Type:* Set (base node)  
  - *Role:* Prepares text input by setting the "text" field with chatInput content for AI processing.  
  - *Configuration:* Assigns `text = chatInput`.  
  - *Inputs:* From Switch node Text output.  
  - *Outputs:* AI Agent - Nova node.  
  - *Edge Cases:* Empty or malformed text inputs.

---

#### 1.2 AI Processing & Memory Management

**Overview:**  
This block handles AI language processing, maintaining conversation context, and generating responses with a specific personality. It integrates memory buffers and multiple language models to ensure nuanced, context-aware output.

**Nodes Involved:**  
- Simple Memory  
- AI Agent - Nova  
- Basic LLM Chain  
- OpenAI Chat Model  
- Anthropic Chat Model  
- Sticky Note  

**Node Details:**

- **Simple Memory**  
  - *Type:* LangChain memory buffer (window)  
  - *Role:* Maintains a rolling window of the last 20 conversational messages per session for context.  
  - *Configuration:* Session key derived from chat session ID; contextWindowLength=20.  
  - *Inputs:* Connected to AI Agent - Nova’s memory input.  
  - *Outputs:* AI Agent - Nova.  
  - *Edge Cases:* Session key errors can break context; memory overflow if context window too short.

- **AI Agent - Nova**  
  - *Type:* LangChain agent node  
  - *Role:* Core AI logic node; processes user text, calls external tools as needed, and generates responses.  
  - *Configuration:*  
    - Uses a detailed prompt defining Nova’s sarcastic, glamorous personality and instructions for handling emails, calendar, tasks, and info queries.  
    - Integrates multiple tools (email, calendar, Airtable, web tools) for dynamic response generation.  
    - Accesses conversation memory.  
  - *Inputs:* Receives text from Edit Fields or transcribed audio; memory from Simple Memory; tools outputs.  
  - *Outputs:* Basic LLM Chain node.  
  - *Edge Cases:* API failures, tool integration errors, prompt misinterpretation.

- **Basic LLM Chain**  
  - *Type:* LangChain LLM Chain  
  - *Role:* Takes AI Agent output and applies a final personality layer—Nova’s "bitchy luxe" tone and style.  
  - *Configuration:* Uses GPT-4.1-mini model; prompt text defines personality with examples and formatting instructions.  
  - *Inputs:* From AI Agent - Nova.  
  - *Outputs:* Terminal output (no further nodes).  
  - *Edge Cases:* Model API limits, prompt formatting errors.

- **OpenAI Chat Model**  
  - *Type:* LangChain language model (OpenAI)  
  - *Role:* Alternative language model option for AI Agent usage.  
  - *Configuration:* GPT-4.1-mini selected; linked as AI language model input to AI Agent - Nova.  
  - *Inputs:* Not directly triggered; configured as AI language model option.  
  - *Outputs:* AI Agent - Nova.  
  - *Edge Cases:* API key limits, version compatibility.

- **Anthropic Chat Model**  
  - *Type:* LangChain language model (Anthropic)  
  - *Role:* Alternative AI language model option (Claude 4 Sonnet).  
  - *Configuration:* Uses Claude Sonnet 4; linked as AI language model input to Basic LLM Chain.  
  - *Inputs:* From AI Agent - Nova.  
  - *Outputs:* Basic LLM Chain.  
  - *Edge Cases:* API rate limits, model availability.

- **Sticky Note**  
  - *Type:* Sticky Note (base node)  
  - *Role:* Documentation aid labeling the personality layer as "This layer is the bitchy tone."  
  - *Position:* Adjacent to Basic LLM Chain.  
  - *Edge Cases:* None (documentation only).

---

#### 1.3 External Tools & Integrations

**Overview:**  
This block connects to external services and tools to retrieve or manipulate data as per user requests, including emails, calendar events, contacts, files, and web queries.

**Nodes Involved:**  
- Get many messages in Gmail  
- Get Calendar  
- Contacts (Airtable)  
- Search files and folders in Google Drive  
- SerpAPI  
- Wikipedia  
- Hacker News  
- Calculator  

**Node Details:**

- **Get many messages in Gmail**  
  - *Type:* Gmail Tool  
  - *Role:* Fetches emails from user’s Gmail inbox for reading or summarization.  
  - *Configuration:* Retrieves all messages or filtered subset based on AI parameters; uses OAuth2 credentials.  
  - *Inputs:* AI Agent - Nova ai_tool input.  
  - *Outputs:* Back to AI Agent - Nova.  
  - *Edge Cases:* OAuth token expiration; Gmail API quota; large inbox size.

- **Get Calendar**  
  - *Type:* Google Calendar Tool  
  - *Role:* Retrieves calendar events for specified user calendar.  
  - *Configuration:* Uses OAuth2 Google Calendar credentials; operation to get all events; returnAll flag controlled by AI.  
  - *Inputs:* AI Agent - Nova ai_tool input.  
  - *Outputs:* AI Agent - Nova.  
  - *Edge Cases:* Calendar permission errors; API limits.

- **Contacts (Airtable)**  
  - *Type:* Airtable Tool  
  - *Role:* Creates new contact records in Airtable base from AI-extracted fields.  
  - *Configuration:* Maps AI output fields (Full Name, Phone, Email, Summary, Verification Status) into Airtable columns; uses personal access token.  
  - *Inputs:* AI Agent - Nova ai_tool input.  
  - *Outputs:* AI Agent - Nova.  
  - *Edge Cases:* Airtable API limits; field mapping errors.

- **Search files and folders in Google Drive**  
  - *Type:* Google Drive Tool  
  - *Role:* Searches files or folders in Google Drive matching AI query terms.  
  - *Configuration:* Limits to 50 results; query string dynamically set by AI; OAuth2 credentials.  
  - *Inputs:* AI Agent - Nova ai_tool input.  
  - *Outputs:* AI Agent - Nova.  
  - *Edge Cases:* Drive API quota; query syntax issues.

- **SerpAPI**  
  - *Type:* LangChain SerpAPI Tool  
  - *Role:* Performs Google Search queries to fetch web results for general information.  
  - *Configuration:* Uses SerpAPI credentials; no additional parameters set.  
  - *Inputs:* AI Agent - Nova ai_tool input.  
  - *Outputs:* AI Agent - Nova.  
  - *Edge Cases:* SerpAPI rate limits; query failures.

- **Wikipedia**  
  - *Type:* LangChain Wikipedia Tool  
  - *Role:* Retrieves Wikipedia information for user questions.  
  - *Configuration:* Default parameters; no credentials needed.  
  - *Inputs:* AI Agent - Nova ai_tool input.  
  - *Outputs:* AI Agent - Nova.  
  - *Edge Cases:* Article not found; disambiguation pages.

- **Hacker News**  
  - *Type:* Hacker News Tool  
  - *Role:* Fetches news articles from Hacker News based on AI-supplied article ID or keywords.  
  - *Configuration:* Article ID set dynamically by AI; no additional parameters.  
  - *Inputs:* AI Agent - Nova ai_tool input.  
  - *Outputs:* AI Agent - Nova.  
  - *Edge Cases:* Article ID invalid; network issues.

- **Calculator**  
  - *Type:* LangChain Calculator Tool  
  - *Role:* Performs arithmetic or mathematical calculations on user queries.  
  - *Configuration:* Default settings.  
  - *Inputs:* AI Agent - Nova ai_tool input.  
  - *Outputs:* AI Agent - Nova.  
  - *Edge Cases:* Invalid math expressions; calculation errors.

---

#### 1.4 Response Output & Personality Layer

**Overview:**  
This block finalizes the AI response with a specific "bitchy luxe" persona and delivers the output back to the user.

**Nodes Involved:**  
- Basic LLM Chain  
- Sticky Note  

**Node Details:**

- **Basic LLM Chain** (already detailed above)  
  - *Role:* Applies Nova’s biting, glamorous tone to AI responses.  
  - *Output:* Final message to user interface.

- **Sticky Note**  
  - *Role:* Indicates the personality styling applied in this layer.

---

### 3. Summary Table

| Node Name                   | Node Type                          | Functional Role                                | Input Node(s)                 | Output Node(s)                 | Sticky Note                                     |
|-----------------------------|----------------------------------|-----------------------------------------------|------------------------------|-------------------------------|------------------------------------------------|
| When chat message received  | LangChain Chat Trigger            | Entry webhook for user chat messages          | (Webhook trigger)             | Switch                        |                                                |
| Switch                      | Switch (base node)                | Routes input to audio, text, or error paths   | When chat message received    | Get a file, Edit Fields        |                                                |
| Get a file                  | Telegram node                    | Downloads audio file from Telegram             | Switch (Audio)                | Transcribe a recording         |                                                |
| Transcribe a recording      | OpenAI audio transcribe node     | Transcribes audio to text                       | Get a file                   | AI Agent - Nova                |                                                |
| Edit Fields                 | Set (base node)                  | Sets text field from chat input                 | Switch (Text)                | AI Agent - Nova                |                                                |
| Simple Memory               | LangChain memory buffer          | Maintains conversation context                  | (Connected internally)       | AI Agent - Nova               |                                                |
| AI Agent - Nova             | LangChain agent node             | Core AI processing and orchestration            | Edit Fields, Transcribe a recording, Simple Memory, Tools | Basic LLM Chain            |                                                |
| Basic LLM Chain             | LangChain LLM chain              | Applies final personality tone to response     | AI Agent - Nova              | (Terminal output)             | This layer is the bitchy tone                   |
| OpenAI Chat Model           | LangChain LM (OpenAI)            | Language model option for AI Agent              | (Configured as AI languageModel for AI Agent - Nova) | AI Agent - Nova             |                                                |
| Anthropic Chat Model        | LangChain LM (Anthropic)         | Alternative language model for final LLM chain | AI Agent - Nova              | Basic LLM Chain               |                                                |
| Get many messages in Gmail  | Gmail Tool                      | Fetches emails from Gmail                        | AI Agent - Nova (ai_tool)    | AI Agent - Nova               |                                                |
| Get Calendar                | Google Calendar Tool             | Retrieves calendar events                        | AI Agent - Nova (ai_tool)    | AI Agent - Nova               |                                                |
| Contacts                    | Airtable Tool                   | Creates contact records in Airtable             | AI Agent - Nova (ai_tool)    | AI Agent - Nova               |                                                |
| Search files and folders in Google Drive | Google Drive Tool        | Searches Google Drive files/folders             | AI Agent - Nova (ai_tool)    | AI Agent - Nova               |                                                |
| SerpAPI                     | LangChain SerpAPI Tool           | Performs Google Search                           | AI Agent - Nova (ai_tool)    | AI Agent - Nova               |                                                |
| Wikipedia                   | LangChain Wikipedia Tool         | Retrieves Wikipedia info                         | AI Agent - Nova (ai_tool)    | AI Agent - Nova               |                                                |
| Hacker News                 | Hacker News Tool                | Fetches Hacker News articles                     | AI Agent - Nova (ai_tool)    | AI Agent - Nova               |                                                |
| Calculator                  | LangChain Calculator Tool        | Performs mathematical calculations               | AI Agent - Nova (ai_tool)    | AI Agent - Nova               |                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "When chat message received" node (LangChain Chat Trigger):**  
   - Set as public webhook.  
   - Customize welcome screen with Nova’s introduction and sassy personality text.  
   - Add custom CSS for the chat UI theme as provided.  
   - Enable file uploads.  
   - Save sessionId from incoming messages.

2. **Create "Switch" node:**  
   - Add three outputs: Audio, Text, Error.  
   - Audio: Condition - message contains `video_note.file_id`.  
   - Text: Condition - `chatInput` exists.  
   - Error: Condition - `error` field exists (optional).  
   - Connect input from "When chat message received."

3. **Create "Get a file" node (Telegram):**  
   - Set resource to "file."  
   - File ID sourced dynamically from `message.video_note.file_id` of incoming Telegram message.  
   - Use Telegram API credentials with valid token.  
   - Connect input from Switch Audio output.

4. **Create "Transcribe a recording" node (OpenAI):**  
   - Set resource to "audio" and operation to "transcribe."  
   - Use OpenAI API credentials.  
   - Connect input from "Get a file."

5. **Create "Edit Fields" node (Set):**  
   - Assign `text` field from `chatInput` property.  
   - Connect input from Switch Text output.

6. **Create "Simple Memory" node (LangChain Memory Buffer Window):**  
   - Set session key to the sessionId from "When chat message received."  
   - Set context window length to 20 messages.  
   - Connect to AI Agent - Nova’s memory input.

7. **Create "AI Agent - Nova" node (LangChain Agent):**  
   - Configure prompt with detailed instructions defining Nova’s personality and tool usage guidelines.  
   - Set text input as `text` field from "Edit Fields" or transcribed text.  
   - Connect ai_tool inputs to external tools (see below).  
   - Connect ai_memory input from "Simple Memory."  
   - Connect ai_languageModel input optionally from "OpenAI Chat Model."  
   - Connect main output to "Basic LLM Chain."

8. **Create "Basic LLM Chain" node (LangChain LLM Chain):**  
   - Use GPT-4.1-mini model.  
   - Define prompt with Nova’s sarcastic, glamorous tone and example phrases for final output styling.  
   - Connect input from "AI Agent - Nova."  
   - No further outputs.

9. **Create "OpenAI Chat Model" node (LangChain LM):**  
   - Model: GPT-4.1-mini.  
   - Connect output to AI Agent - Nova’s ai_languageModel input.

10. **Create "Anthropic Chat Model" node (LangChain LM):**  
    - Model: Claude Sonnet 4.  
    - Connect output to "Basic LLM Chain" ai_languageModel input.

11. **Create external tool nodes and connect to AI Agent - Nova (ai_tool inputs):**  
    - **Get many messages in Gmail:** Set operation to get all emails; connect output back to AI Agent.  
    - **Get Calendar:** Connect to Google Calendar with OAuth2; set to get all events or filtered by AI.  
    - **Contacts (Airtable):** Configure Airtable base and table; map AI output fields to Airtable columns; connect back.  
    - **Search files and folders in Google Drive:** Setup Google Drive OAuth2; dynamic query from AI; connect back.  
    - **SerpAPI:** Configure API key; no extra parameters; connect back.  
    - **Wikipedia:** Default parameters; connect back.  
    - **Hacker News:** Use dynamic article ID from AI; connect back.  
    - **Calculator:** Default settings; connect back.

12. **Add a "Sticky Note" node near "Basic LLM Chain":**  
    - Content: "This layer is the bitchy tone."

13. **Connections:**  
    - Connect "When chat message received" main output to "Switch."  
    - Connect Switch Audio → Get a file → Transcribe a recording → AI Agent - Nova.  
    - Connect Switch Text → Edit Fields → AI Agent - Nova.  
    - Connect Simple Memory to AI Agent memory input.  
    - Connect AI Agent output to Basic LLM Chain input.  
    - Connect all external tool nodes ai_tool inputs and outputs to and from AI Agent - Nova as per tool usage.  
    - Connect OpenAI Chat Model as AI language model input to AI Agent - Nova.  
    - Connect Anthropic Chat Model as AI language model input to Basic LLM Chain.

14. **Credentials Setup:**  
    - Telegram API with valid bot token.  
    - OpenAI API key for transcription and language models.  
    - Google OAuth2 for Gmail, Google Calendar, and Google Drive.  
    - Airtable Personal Access Token.  
    - SerpAPI API key.  
    - Anthropic API key.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                    | Context or Link                                                                                                                        |
|-------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| This workflow features a custom chat UI theme with the "NOVA — Bitchy Luxe" style, including CSS animations and color palettes for a glamorous AI. | Embedded in "When chat message received" node's custom CSS configuration.                                                             |
| The AI personality "Nova" is defined with a detailed prompt emphasizing sarcasm, sass, and a glamorous attitude, inspired by pop culture icons. | Prompt text inside "AI Agent - Nova" and "Basic LLM Chain" nodes.                                                                     |
| Uses LangChain nodes extensively for AI orchestration, memory, and tool integration, demonstrating advanced n8n AI workflow capabilities.       | Workflow design leverages LangChain ecosystem within n8n.                                                                             |
| The workflow supports audio input transcription via Telegram voice notes, enhancing multimodal interaction.                                    | Audio path via "Get a file" and "Transcribe a recording" nodes.                                                                       |
| Integration with multiple external APIs (Google, Airtable, SerpAPI, Hacker News) requires proper OAuth2 and API key credential setup.          | Credentials must be configured prior to execution for all external services.                                                          |

---

**Disclaimer:** The text above is generated from an automated n8n workflow export. It strictly complies with content policies and contains no illegal or offensive elements. All processed data is legal and publicly accessible.