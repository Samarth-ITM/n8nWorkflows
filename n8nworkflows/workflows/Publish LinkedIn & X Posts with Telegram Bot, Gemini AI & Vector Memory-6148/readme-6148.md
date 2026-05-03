Publish LinkedIn & X Posts with Telegram Bot, Gemini AI & Vector Memory

https://n8nworkflows.xyz/workflows/publish-linkedin---x-posts-with-telegram-bot--gemini-ai---vector-memory-6148


# Publish LinkedIn & X Posts with Telegram Bot, Gemini AI & Vector Memory

### 1. Workflow Overview

This workflow implements a Telegram-based AI Social Media Manager bot that receives user inputs via Telegram messages (text, documents, or voice), processes them with Google Gemini AI and a LangChain AI Agent, and assists in creating and publishing social media posts on platforms such as LinkedIn (as a person or company) and X (formerly Twitter). It also maintains a vector memory for long-term knowledge storage and retrieval to enhance context in conversations.

The key logical blocks are:

- **1.1 Bot Input Reception (Telegram Integration):** Receiving updates from Telegram (messages, callback queries, inline queries), preprocessing and standardizing input data for downstream processing.

- **1.2 Input Type Handling and Content Extraction:** Differentiating input types (text, document, voice, callback) and extracting relevant content accordingly, including downloading and encoding files, and using Google Gemini to analyze documents and audio.

- **1.3 AI Agent Processing and Memory Management:** Using a LangChain AI Agent powered by Google Gemini Chat to interpret user requests, generate social media post drafts, manage long-term knowledge via vector memory, and decide on posting actions.

- **1.4 Social Media Posting:** Creating posts on LinkedIn (company or personal profile) and X, triggered by the AI Agent upon user approval.

- **1.5 Response Delivery and Error Handling:** Splitting long AI responses into message-sized parts and sending them back to the user via Telegram; handling errors and invalid callbacks gracefully.

---

### 2. Block-by-Block Analysis

#### 2.1 Bot Input Reception (Telegram Integration)

- **Overview:**  
  This block manages incoming Telegram updates, extracts and normalizes key information such as chat ID, message text, callback data, and file identifiers, preparing a unified data context for further processing.

- **Nodes Involved:**  
  - Telegram Trigger  
  - Set input context  
  - Input type switch

- **Node Details:**

  - **Telegram Trigger**  
    - Type: Telegram Trigger node  
    - Role: Receives all Telegram updates including messages, callback queries, inline queries, etc.  
    - Configuration: Listens to multiple update types; uses a Telegram Bot credential named "Gate bot".  
    - Inputs: Telegram webhook calls from Telegram servers.  
    - Outputs: Raw Telegram update JSON.  
    - Edge Cases: Telegram API limits; missing or malformed updates; bot token expiry.

  - **Set input context**  
    - Type: Set node  
    - Role: Extracts and normalizes relevant fields from Telegram update JSON into easy-to-use variables such as `chat_id`, `callback_data`, `message_text`, `update_type`, and file IDs for documents or voice messages.  
    - Key Expressions: Conditional extraction using optional chaining (`?.`) to handle multiple update types and fallback defaults.  
    - Inputs: Output of Telegram Trigger.  
    - Outputs: JSON with standardized fields.  
    - Edge Cases: Missing fields for some update types; potential expression evaluation errors if Telegram JSON schema changes.

  - **Input type switch**  
    - Type: Switch node  
    - Role: Routes the workflow based on the determined `update_type` (`text`, `document`, `voice`, `callback`, or `extra`).  
    - Configuration: Uses strict string equality on the `update_type` field set previously.  
    - Inputs: Normalized input context JSON.  
    - Outputs: Branches to specialized processing nodes per input type.  
    - Edge Cases: Unknown or unsupported update types; fallback route "extra" is present but not connected downstream.

---

#### 2.2 Input Type Handling and Content Extraction

- **Overview:**  
  This block handles different input types by downloading, encoding, and extracting semantic content from documents and audio via Google Gemini AI, or prepares text prompts directly. It ensures that raw user data is converted into AI-understandable text.

- **Nodes Involved:**  
  - Download document  
  - Encode document  
  - Describe document  
  - Document prompt  
  - Download audio  
  - Encode audio  
  - Describe audio  
  - Audio prompt  
  - Text prompt

- **Node Details:**

  - **Download document**  
    - Type: Telegram node (Download file)  
    - Role: Downloads the user-sent document file from Telegram servers using the extracted `file_id`.  
    - Inputs: `file_id` from Set input context → Input type switch → Download document.  
    - Outputs: Binary file data.  
    - Edge Cases: Telegram API file download errors; invalid or expired file IDs.

  - **Encode document**  
    - Type: Extract from File node  
    - Role: Converts downloaded binary document into Base64 string in JSON property for API consumption.  
    - Inputs: Binary data from Download document.  
    - Outputs: JSON with Base64-encoded document data.  
    - Edge Cases: Large files causing memory issues; encoding failures.

  - **Describe document**  
    - Type: HTTP Request node  
    - Role: Sends Base64 PDF data to Google Gemini AI API to extract company/product information from the document.  
    - Configuration: POST request to Gemini's generateContent endpoint with specific prompt text and embedded PDF data. Uses stored Google Palm API credentials.  
    - Inputs: Encoded document JSON.  
    - Outputs: AI-generated content about the document.  
    - Edge Cases: API rate limits; invalid API key; network timeouts; response parsing errors.

  - **Document prompt**  
    - Type: Set node  
    - Role: Constructs AI Agent input prompt combining document caption and extracted document content for downstream processing.  
    - Inputs: Output from Describe document node.  
    - Outputs: JSON with `chatInput` string.  
    - Edge Cases: Missing caption or empty extracted text.

  - **Download audio**  
    - Type: Telegram node (Download file)  
    - Role: Downloads voice message audio using `voice_file_id`.  
    - Inputs: From Input type switch on voice input.  
    - Outputs: Binary audio data.  
    - Edge Cases: Invalid file IDs; download failures.

  - **Encode audio**  
    - Type: Extract from File node  
    - Role: Converts downloaded audio binary to Base64 string property.  
    - Inputs: Binary audio from Download audio.  
    - Outputs: JSON with Base64-encoded audio data.  
    - Edge Cases: Large audio files; encoding failures.

  - **Describe audio**  
    - Type: HTTP Request node  
    - Role: Sends Base64 audio to Google Gemini AI API with prompt to generate a transcript of speech.  
    - Inputs: Encoded audio property.  
    - Outputs: Transcribed text from audio.  
    - Edge Cases: Same as Describe document; plus audio format support issues.

  - **Audio prompt**  
    - Type: Set node  
    - Role: Sets `chatInput` field with the transcribed text from the audio for AI Agent input.  
    - Inputs: Output of Describe audio node.  
    - Outputs: JSON with `chatInput`.  
    - Edge Cases: Empty or low-quality transcripts.

  - **Text prompt**  
    - Type: Set node  
    - Role: Directly sets `chatInput` from plain text messages received.  
    - Inputs: Output of Input type switch on text branch.  
    - Outputs: JSON with `chatInput`.  
    - Edge Cases: Empty message text, spam or invalid content.

---

#### 2.3 AI Agent Processing and Memory Management

- **Overview:**  
  This core block uses LangChain AI Agent powered by Google Gemini Chat to understand user input, generate social media post drafts, manage vector memory, and decide on actions including posting or requesting clarifications.

- **Nodes Involved:**  
  - AI Agent  
  - Google Gemini Chat Model  
  - Embeddings Google Gemini  
  - Simple Memory  
  - Retrieve knowledge  
  - Save knowledge  
  - Chat Memory Manager  
  - If  
  - Success  
  - Error

- **Node Details:**

  - **AI Agent**  
    - Type: LangChain Agent Node  
    - Role: Central AI decision maker. Processes user input (`chatInput`), uses vector memory tools (`Retrieve knowledge`, `Save knowledge`), generates social media posts drafts or commands for posting.  
    - Configuration: System message sets agent persona as social media content manager.  
    - Inputs: `chatInput` from prompt nodes, vector memory data, language model responses.  
    - Outputs: Textual response to user, commands to post, or memory actions.  
    - Edge Cases: Misinterpretation of input, API limits, memory retrieval errors.

  - **Google Gemini Chat Model**  
    - Type: LangChain Google Gemini Chat Model  
    - Role: Language model backend for AI Agent.  
    - Inputs: User prompt context.  
    - Outputs: Model-generated text responses.  
    - Edge Cases: API authentication errors, rate limits.

  - **Embeddings Google Gemini**  
    - Type: LangChain Embeddings Node  
    - Role: Generates vector embeddings from text for memory retrieval.  
    - Inputs: Text data from AI Agent.  
    - Outputs: Embeddings used for vector store.  
    - Edge Cases: API issues, embedding quality.

  - **Simple Memory**  
    - Type: LangChain Memory Buffer Window  
    - Role: Maintains short-term conversational memory per chat session (`chat_id`).  
    - Inputs: Conversation messages.  
    - Outputs: Contextual memory for AI Agent.  
    - Edge Cases: Memory overflow, session key conflicts.

  - **Retrieve knowledge**  
    - Type: LangChain Vector Store (in-memory)  
    - Role: Retrieves relevant long-term knowledge records by chat session key.  
    - Inputs: Embedding queries, `chat_id` as memory key.  
    - Outputs: Retrieved knowledge documents.  
    - Edge Cases: Empty or outdated memory, retrieval errors.

  - **Save knowledge**  
    - Type: LangChain Tool Workflow Node  
    - Role: Invokes a sub-workflow to save new knowledge records to vector memory.  
    - Inputs: Data to save, `chat_id`.  
    - Outputs: Confirmation or error.  
    - Edge Cases: Sub-workflow failures, data format issues.

  - **Chat Memory Manager**  
    - Type: LangChain Memory Manager (delete mode)  
    - Role: Clears all short-term memory for a chat session on command.  
    - Inputs: Triggered from conditional node.  
    - Outputs: Confirmation message.  
    - Edge Cases: Concurrent access.

  - **If**  
    - Type: If node  
    - Role: Checks if callback data equals "cm" (clear memory command). Routes to clear memory or error.  
    - Inputs: `callback_data` from input context.  
    - Outputs: Branches to Chat Memory Manager or Error nodes.  
    - Edge Cases: Unexpected callback data.

  - **Success**  
    - Type: Telegram node (Send message)  
    - Role: Sends confirmation message "Memory is cleaned" to user after memory reset.  
    - Inputs: Trigger from Chat Memory Manager.  
    - Outputs: Telegram message.  
    - Edge Cases: Telegram message delivery failures.

  - **Error**  
    - Type: Telegram node (Send message)  
    - Role: Sends error message "Invalid callback" to user if callback data is unrecognized.  
    - Inputs: From If node negative branch.  
    - Outputs: Telegram message.  
    - Edge Cases: Telegram API errors.

---

#### 2.4 Social Media Posting

- **Overview:**  
  This block executes social media posting commands initiated by the AI Agent, allowing posting to X (Twitter) and LinkedIn as either a personal profile or an organization.

- **Nodes Involved:**  
  - Create X (Twitter) post  
  - Create post in LinkedIn as a person  
  - Create post in LinkedIn as a company

- **Node Details:**

  - **Create X (Twitter) post**  
    - Type: Twitter Tool node  
    - Role: Posts tweets (X posts) with text content generated by AI Agent.  
    - Configuration: Text input limited to 280 characters; handles "Forbidden" error indicating text length issues. Uses OAuth2 credentials for X account.  
    - Inputs: AI Agent's post text command.  
    - Outputs: Twitter API response.  
    - Edge Cases: API rate limits, authentication issues, text length errors.

  - **Create post in LinkedIn as a person**  
    - Type: LinkedIn Tool node  
    - Role: Posts social media content on LinkedIn personal profile.  
    - Configuration: Requires LinkedIn OAuth2 credentials; configured with personal LinkedIn user ID.  
    - Inputs: AI Agent's post text.  
    - Outputs: LinkedIn API response.  
    - Edge Cases: LinkedIn API quota limits, authentication, malformed content.

  - **Create post in LinkedIn as a company**  
    - Type: LinkedIn Tool node  
    - Role: Posts LinkedIn content as an organization.  
    - Configuration: Requires organization URN; LinkedIn OAuth2 credentials.  
    - Inputs: AI Agent's post text.  
    - Outputs: LinkedIn API response.  
    - Edge Cases: Organization permissions, API limits.

---

#### 2.5 Response Delivery and Error Handling

- **Overview:**  
  This block manages AI Agent responses, including splitting long messages into chunks to avoid Telegram's maximum message length errors, and sending messages back to the user.

- **Nodes Involved:**  
  - Split agent's response  
  - Split Out  
  - Send response

- **Node Details:**

  - **Split agent's response**  
    - Type: Code node (JavaScript)  
    - Role: Splits AI Agent's textual output into chunks of 3000 characters or less to avoid Telegram message length limitations.  
    - Inputs: AI Agent output.  
    - Outputs: JSON array with split text parts.  
    - Edge Cases: Very long outputs; splitting logic correctness.

  - **Split Out**  
    - Type: Split Out node  
    - Role: Splits array of message parts into individual items to send separately.  
    - Inputs: Output from code node with parts array.  
    - Outputs: Multiple individual messages.  
    - Edge Cases: Empty arrays.

  - **Send response**  
    - Type: Telegram node (Send message)  
    - Role: Sends each chunked message to the user with HTML parse mode.  
    - Inputs: Messages from Split Out node; chat ID from context.  
    - Outputs: Telegram messages delivered to user.  
    - Edge Cases: Telegram API rate limits; message delivery failures.

---

### 3. Summary Table

| Node Name                       | Node Type                               | Functional Role                                  | Input Node(s)                 | Output Node(s)                                 | Sticky Note                                                                                                                |
|--------------------------------|---------------------------------------|-------------------------------------------------|------------------------------|-----------------------------------------------|----------------------------------------------------------------------------------------------------------------------------|
| Telegram Trigger               | Telegram Trigger                      | Receives Telegram updates                        | -                            | Set input context                             | ## Bot entrance\n\nSet your Telegram bot credentials here.\n\nOptional: add "Restrict to chat IDs" with your chat ID so only you can use your bot |
| Set input context              | Set                                  | Normalizes Telegram update data                  | Telegram Trigger             | Input type switch                             | ## Request preprocessing\n\nSetting fields from the request to a more convinient format                                      |
| Input type switch              | Switch                               | Routes flow by input type                         | Set input context            | Text prompt, Download document, Download audio, If |                                                                                                                            |
| Download document             | Telegram (Download file)              | Downloads document file from Telegram            | Input type switch            | Encode document                              | ## Extracting data from non-text input\n\nSet up "Describe PDF" and "Describe audio" nodes with Google's AI Studio API key\n\nSet up "Download document" and "Download audio" with Telegram Bot API key |
| Encode document               | Extract from File                    | Converts binary document to Base64               | Download document            | Describe document                            |                                                                                                                            |
| Describe document             | HTTP Request                        | Extracts info from document via Google Gemini AI| Encode document              | Document prompt                              |                                                                                                                            |
| Document prompt               | Set                                  | Prepares prompt combining caption & extracted text| Describe document          | AI Agent                                     |                                                                                                                            |
| Download audio                | Telegram (Download file)              | Downloads voice message audio from Telegram      | Input type switch            | Encode audio                                |                                                                                                                            |
| Encode audio                 | Extract from File                    | Converts binary audio to Base64                   | Download audio               | Describe audio                              |                                                                                                                            |
| Describe audio               | HTTP Request                        | Transcribes audio using Google Gemini AI         | Encode audio                 | Audio prompt                                |                                                                                                                            |
| Audio prompt                 | Set                                  | Prepares prompt with transcribed audio text      | Describe audio               | AI Agent                                     |                                                                                                                            |
| Text prompt                 | Set                                  | Sets prompt from text message                     | Input type switch            | AI Agent                                     |                                                                                                                            |
| AI Agent                   | LangChain Agent                      | Processes input, generates posts, manages memory| Text prompt, Document prompt, Audio prompt, Retrieve knowledge, Save knowledge, Embeddings Google Gemini, Google Gemini Chat Model, Simple Memory | Split agent's response | ## AI Agent Node\n\n**Needs setup**\n\nAdd your Google's AI Studio API key to the "Model" node\n\nHere the agent receives the user's input and decides what to do in response:\n- Ask for clarification\n- Prepare the post and send for verification\n- Create a post in LinkedIn or X\n- Save some information in long-term memory |
| Google Gemini Chat Model    | LangChain LM Chat Google Gemini      | Language model backend for AI Agent              | AI Agent                     | AI Agent                                     |                                                                                                                            |
| Embeddings Google Gemini    | LangChain Embeddings Google Gemini   | Generates embeddings for vector memory           | AI Agent                     | Retrieve knowledge                          |                                                                                                                            |
| Simple Memory              | LangChain Memory Buffer Window        | Maintains short-term chat memory                  | AI Agent                     | AI Agent, Chat Memory Manager                |                                                                                                                            |
| Retrieve knowledge          | LangChain Vector Store In Memory      | Retrieves long-term knowledge by chat session    | Embeddings Google Gemini     | AI Agent                                     |                                                                                                                            |
| Save knowledge             | LangChain Tool Workflow               | Saves new knowledge to vector memory via sub-workflow| AI Agent                 | AI Agent                                     |                                                                                                                            |
| Chat Memory Manager         | LangChain Memory Manager (delete)    | Clears chat session short-term memory             | If                          | Success                                      |                                                                                                                            |
| If                        | If                                  | Checks for clear memory callback command          | Input type switch            | Chat Memory Manager, Error                   |                                                                                                                            |
| Success                   | Telegram                             | Sends success message after memory cleared        | Chat Memory Manager          | -                                           |                                                                                                                            |
| Error                     | Telegram                             | Sends error message for invalid callback          | If                          | -                                           |                                                                                                                            |
| Split agent's response     | Code                               | Splits AI responses into chunks for Telegram      | AI Agent                    | Split Out                                    | ## Response to the user\nHere we optionally split the agent's response into multiple messages not to get "Message too long" error from Telegram |
| Split Out                 | Split Out                           | Splits message array into individual messages     | Split agent's response       | Send response                                |                                                                                                                            |
| Send response             | Telegram                           | Sends messages back to user on Telegram            | Split Out                   | -                                           |                                                                                                                            |
| Create X (Twitter) post    | Twitter Tool                      | Posts tweets on X platform                          | AI Agent                    | -                                           | ## LinkedIn Tools\n\n- Input your organisation URN in the "Create post in LinkedIn as a company" node\n- Select your user name in LinkedIn in the "Create a post in LinkedIn as a person" |
| Create post in LinkedIn as a person | LinkedIn Tool                | Posts on LinkedIn personal profile                  | AI Agent                    | -                                           |                                                                                                                            |
| Create post in LinkedIn as a company | LinkedIn Tool               | Posts on LinkedIn company page                      | AI Agent                    | -                                           |                                                                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node:**  
   - Type: Telegram Trigger  
   - Configure with your Telegram Bot API credentials ("Gate bot").  
   - Set updates to listen to: message, callback_query, inline_query, and all ("*").  
   - No additional restrictions needed, but optionally restrict to your chat ID.

2. **Add Set node ("Set input context"):**  
   - Extract from incoming Telegram JSON:  
     - `chat_id` = `callback_query?.message.chat.id || message?.chat.id`  
     - `callback_data` = `callback_query?.data || 'none'`  
     - `message_text` = `message?.text || 'none'`  
     - `update_type` = computed based on presence of text, callback, document, or voice in message.  
     - Extract file IDs and captions as needed.

3. **Add Switch node ("Input type switch"):**  
   - Routes on `update_type` field with cases: text, document, voice, callback, fallback to extra.

4. **For text input branch:**  
   - Add Set node ("Text prompt") to set `chatInput` from `message_text`.  
   - Connect to AI Agent block (to be created later).

5. **For document input branch:**  
   - Add Telegram node ("Download document") using `file_id` to download file.  
   - Add Extract From File node ("Encode document") to convert binary to Base64 string property.  
   - Add HTTP Request node ("Describe document") configured to POST to Google Gemini API with prompt to extract company/product info, embedding Base64 data. Use Google Palm API credentials.  
   - Add Set node ("Document prompt") to compose prompt with caption and extracted content.  
   - Connect to AI Agent block.

6. **For voice input branch:**  
   - Add Telegram node ("Download audio") using `voice_file_id`.  
   - Add Extract From File node ("Encode audio") to Base64 encode audio.  
   - Add HTTP Request node ("Describe audio") to request transcript from Google Gemini API.  
   - Add Set node ("Audio prompt") with transcribed text.  
   - Connect to AI Agent block.

7. **For callback input branch:**  
   - Add If node checking if `callback_data` equals "cm" (clear memory).  
   - If true, connect to LangChain Memory Manager node ("Chat Memory Manager") in delete mode, clearing all memory.  
   - On success, send Telegram message "Memory is cleaned" ("Success" node).  
   - If false, send Telegram message "Invalid callback" ("Error" node).

8. **AI Agent block:**  
   - Add LangChain Agent node ("AI Agent") with system message defining content manager persona and instructions.  
   - Connect LangChain Google Gemini Chat Model node as LM backend.  
   - Add LangChain Embeddings node ("Embeddings Google Gemini") for vector embedding generation.  
   - Add LangChain Vector Store in-memory node ("Retrieve knowledge") to fetch knowledge by `chat_id`.  
   - Add LangChain Tool Workflow node ("Save knowledge") to invoke external workflow for saving knowledge vectors, accepting `data` and `chat_id` inputs.  
   - Add LangChain Memory Buffer Window node ("Simple Memory") keyed by `chat_id` for short-term memory.  
   - Connect all memory and model nodes to the AI Agent accordingly.

9. **Split AI responses:**  
   - Add Code node ("Split agent's response") with JS code to split long text into 3000-char parts.  
   - Add Split Out node to split the array into separate messages.  
   - Add Telegram node ("Send response") to send each part with HTML parse mode to `chat_id`.

10. **Social media posting nodes:**  
    - Add Twitter Tool node ("Create X (Twitter) post") with OAuth2 credentials; text input from AI Agent commands.  
    - Add LinkedIn Tool nodes:  
      - For personal posts ("Create post in LinkedIn as a person") with user ID and OAuth2 credentials.  
      - For company posts ("Create post in LinkedIn as a company") with organization URN and OAuth2 credentials.  
    - Connect these nodes as AI Agent tools for posting actions.

11. **Add Sticky Notes:**  
    - Add descriptive sticky notes near corresponding nodes for setup instructions and workflow explanation (optional but recommended for clarity).

12. **Set all credentials:**  
    - Telegram Bot API (Telegram Trigger, Download nodes, Send).  
    - Google Palm API key for Gemini nodes (Describe document/audio, Chat Model, Embeddings).  
    - LinkedIn OAuth2 credentials.  
    - X OAuth2 credentials.

13. **Test end-to-end:**  
    - Send various input types to Telegram bot: text, document, voice messages.  
    - Verify AI processing, post draft generation, memory saving, and posting to social media after user approval.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Your personal AI SMM manager inside Telegram bot. Capable of receiving text, voice, or document input and publishing posts with user verification. Saves long-term knowledge in vector memory. Requires API keys for Google AI Studio, Telegram Bot, LinkedIn, and X (Twitter).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | This workflow description sticky note in the original workflow ("Sticky Note6")                   |
| Google's AI Studio API key is required for document/audio description and AI model usage. Get one at: https://aistudio.google.com/app/apikey                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Mentioned in "Sticky Note6" and node credentials                                                  |
| Telegram Bot API key needed to receive messages and download files. Register via @BotFather on Telegram.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | "Sticky Note6" and Telegram Trigger node credentials                                             |
| LinkedIn API key setup instructions: https://docs.n8n.io/integrations/builtin/credentials/linkedin/                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | "Sticky Note6" and LinkedIn Tool nodes                                                           |
| X (Twitter) API key setup instructions: https://docs.n8n.io/integrations/builtin/credentials/twitter/                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | "Sticky Note6" and Twitter Tool node                                                             |
| To avoid "Message too long" errors on Telegram, AI responses are split into chunks of 3000 characters before sending.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | "Sticky Note4"                                                                                   |
| AI Agent Node requires system prompt configuration to behave as a social media content manager and use tools for memory and posting.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | "Sticky Note3"                                                                                   |
| LinkedIn Tools require setting your organization URN or personal user ID in respective nodes to post correctly.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | "Sticky Note5"                                                                                   |

---

**Disclaimer:** The provided text and workflow are generated strictly from an automated n8n workflow export. All processing respects current content policies and contains no illegal, offensive, or protected elements. Data handled are legal and public.