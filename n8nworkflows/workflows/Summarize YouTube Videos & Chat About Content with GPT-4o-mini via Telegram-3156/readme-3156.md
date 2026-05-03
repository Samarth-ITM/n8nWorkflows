Summarize YouTube Videos & Chat About Content with GPT-4o-mini via Telegram

https://n8nworkflows.xyz/workflows/summarize-youtube-videos---chat-about-content-with-gpt-4o-mini-via-telegram-3156


# Summarize YouTube Videos & Chat About Content with GPT-4o-mini via Telegram

### 1. Workflow Overview

This workflow automates the summarization of YouTube video transcripts and enables interactive question-answering about the video content via Telegram, using the GPT-4o-mini AI model. It supports two input methods for YouTube URLs: Telegram messages and webhooks (e.g., from Apple shortcuts). The workflow extracts the transcript using a community node, stores it in Google Docs, generates a structured summary, and sends it back to the user. Additionally, it allows users to ask questions about the video content, retrieving context from the stored transcript and responding via Telegram.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception**: Receives YouTube URLs via Telegram messages or webhooks.
- **1.2 Transcript Extraction and Storage**: Extracts the transcript from YouTube and stores it in Google Docs.
- **1.3 Transcript Processing and Summarization**: Splits, concatenates, and summarizes the transcript using GPT-4o-mini.
- **1.4 Summary Delivery**: Sends the generated summary back to the user via Telegram and webhook response.
- **1.5 Interactive Q&A Handling**: Processes user questions about the video content, retrieves transcript context, and responds via Telegram.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block captures YouTube video URLs from two sources: Telegram chat messages and HTTP webhooks. It normalizes the input to extract the URL for downstream processing.

**Nodes Involved:**  
- Trigger on Telegram Message  
- Receive YouTube URL via Webhook  
- Extract YouTube URL from Input

**Node Details:**

- **Trigger on Telegram Message**  
  - Type: `chatTrigger` (LangChain)  
  - Role: Listens for incoming Telegram messages to start the workflow.  
  - Configuration: Uses a webhook ID to receive messages.  
  - Inputs: None (trigger node)  
  - Outputs: Passes message JSON to "Extract YouTube URL from Input".  
  - Edge Cases: Telegram API downtime, malformed messages, unsupported message types.

- **Receive YouTube URL via Webhook**  
  - Type: `webhook`  
  - Role: Receives HTTP requests containing YouTube URLs (e.g., from Apple shortcuts).  
  - Configuration: Path set to a unique webhook ID, response mode set to "responseNode".  
  - Inputs: External HTTP request  
  - Outputs: Passes request data to "Extract YouTube URL from Input".  
  - Edge Cases: Invalid or missing URL parameters, webhook authentication if enabled.

- **Extract YouTube URL from Input**  
  - Type: `set`  
  - Role: Extracts the YouTube URL from either Telegram message (`chatInput`) or webhook query parameter (`query.url`).  
  - Configuration: Sets a new field `youtubeUrl` using expression `={{ $json.chatInput || $json.query.url }}`.  
  - Inputs: From Telegram trigger or webhook node  
  - Outputs: Passes normalized URL to "Extract Video ID from URL".  
  - Edge Cases: Missing or malformed URL, empty input.

---

#### 1.2 Transcript Extraction and Storage

**Overview:**  
This block extracts the YouTube video ID from the URL, retrieves the transcript using a community node, and manages transcript storage in Google Docs.

**Nodes Involved:**  
- Extract Video ID from URL  
- Extract YouTube Transcript  
- Split Transcript into Segments  
- Concatenate Transcript Segments  
- Update Transcript in Google Docs  
- Retrieve Transcript from Google Docs

**Node Details:**

- **Extract Video ID from URL**  
  - Type: `code` (Python)  
  - Role: Parses the YouTube URL to extract the video ID required for transcript retrieval.  
  - Configuration: Custom Python code (currently placeholder code returning `myNewField=1`, needs actual extraction logic).  
  - Inputs: From "Extract YouTube URL from Input"  
  - Outputs: Passes video ID to "Extract YouTube Transcript".  
  - Edge Cases: Invalid URL format, missing video ID, code errors.  
  - Note: Requires implementation of actual extraction logic.

- **Extract YouTube Transcript**  
  - Type: `youtubeTranscripter` (Community node)  
  - Role: Retrieves the transcript text of the YouTube video by video ID.  
  - Configuration: Uses `videoId` from previous node.  
  - Inputs: Video ID JSON  
  - Outputs: Transcript segments (field `transcript`)  
  - Edge Cases: Transcript unavailable, API failures, rate limits, node compatibility (self-hosted n8n required).

- **Split Transcript into Segments**  
  - Type: `splitOut`  
  - Role: Splits the transcript array into individual segments for processing.  
  - Configuration: Splits on field `transcript`.  
  - Inputs: Transcript from "Extract YouTube Transcript"  
  - Outputs: Individual transcript segments to "Concatenate Transcript Segments".  
  - Edge Cases: Empty transcript, malformed data.

- **Concatenate Transcript Segments**  
  - Type: `summarize`  
  - Role: Concatenates all transcript segments into a single text string.  
  - Configuration: Concatenates `text` fields separated by space.  
  - Inputs: Transcript segments from "Split Transcript into Segments"  
  - Outputs: Concatenated transcript text (`concatenated_text`)  
  - Edge Cases: Large transcripts causing performance issues.

- **Update Transcript in Google Docs**  
  - Type: `googleDocs`  
  - Role: Updates the Google Docs document with the concatenated transcript text.  
  - Configuration: Uses `replaceAll` action to replace content with concatenated transcript. Document URL is dynamic (`={{ $json.documentId }}`).  
  - Inputs: Concatenated transcript text and document ID  
  - Outputs: Confirmation of update  
  - Credentials: Google Docs OAuth2  
  - Edge Cases: Authentication errors, document access issues, API rate limits.

- **Retrieve Transcript from Google Docs**  
  - Type: `googleDocs`  
  - Role: Retrieves the transcript content from a fixed Google Docs document URL for Q&A.  
  - Configuration: Operation `get`, fixed document URL.  
  - Inputs: Triggered during Q&A flow  
  - Outputs: Transcript content for AI processing  
  - Credentials: Google Docs OAuth2  
  - Edge Cases: Document not found, permission errors.

---

#### 1.3 Transcript Processing and Summarization

**Overview:**  
This block uses the GPT-4o-mini model to generate a structured summary of the concatenated transcript text.

**Nodes Involved:**  
- Generate Summary with GPT-4o-mini  
- gpt-4o-mini

**Node Details:**

- **Generate Summary with GPT-4o-mini**  
  - Type: `chainLlm` (LangChain)  
  - Role: Sends the concatenated transcript text to GPT-4o-mini with a detailed prompt to create a structured summary.  
  - Configuration: Prompt instructs the AI to produce a general summary, key moments, and instructions (if any), formatted in markdown with HTML bold tags.  
  - Inputs: Concatenated transcript text (`concatenated_text`)  
  - Outputs: Summary text (`text`)  
  - Edge Cases: API rate limits, prompt failures, large input truncation.

- **gpt-4o-mini**  
  - Type: `lmChatOpenAi` (LangChain)  
  - Role: Language model node configured to use OpenAI API with GPT-4o-mini.  
  - Configuration: Uses OpenAI credentials, no special options.  
  - Inputs: Receives prompt from "Generate Summary with GPT-4o-mini"  
  - Outputs: AI-generated summary text  
  - Credentials: OpenAI API key  
  - Edge Cases: Authentication errors, network timeouts.

---

#### 1.4 Summary Delivery

**Overview:**  
This block sends the generated summary back to the user via Telegram and responds to webhook requests.

**Nodes Involved:**  
- Send Summary via Telegram  
- Send Response to Webhook

**Node Details:**

- **Send Summary via Telegram**  
  - Type: `telegram`  
  - Role: Sends the AI-generated summary text to the Telegram chat.  
  - Configuration: Sends `text` field with HTML parse mode, appends the original YouTube URL for reference.  
  - Inputs: Summary text from "Generate Summary with GPT-4o-mini" and YouTube URL from "Extract YouTube URL from Input"  
  - Outputs: Confirmation of message sent, triggers "Send Response to Webhook"  
  - Credentials: Telegram API OAuth2  
  - Edge Cases: Telegram API limits, message formatting issues.

- **Send Response to Webhook**  
  - Type: `respondToWebhook`  
  - Role: Sends HTTP response back to the webhook caller after summary delivery.  
  - Configuration: Default response mode, no custom data.  
  - Inputs: Triggered after Telegram message sent  
  - Outputs: HTTP 200 response to webhook caller  
  - Edge Cases: Webhook timeout, network errors.

---

#### 1.5 Interactive Q&A Handling

**Overview:**  
This block enables users to ask questions about the video content via Telegram. It retrieves the transcript from Google Docs, uses GPT-4o-mini to answer questions contextually, and sends responses back via Telegram.

**Nodes Involved:**  
- Telegram Trigger  
- Handle User Questions via AI  
- OpenAI Chat Model  
- Window Buffer Memory  
- Google Docs2  
- Send AI Response via Telegram

**Node Details:**

- **Telegram Trigger**  
  - Type: `telegramTrigger`  
  - Role: Listens for incoming Telegram messages (user questions).  
  - Configuration: Listens for "message" updates.  
  - Inputs: None (trigger)  
  - Outputs: Passes message JSON to "Handle User Questions via AI"  
  - Credentials: Telegram API OAuth2  
  - Edge Cases: Telegram API downtime, unsupported message types.

- **Handle User Questions via AI**  
  - Type: `agent` (LangChain)  
  - Role: Processes user question text, uses system prompt to instruct AI to answer based on transcript content.  
  - Configuration: System message emphasizes checking transcript from Google Docs before answering.  
  - Inputs: User message text from Telegram Trigger  
  - Outputs: AI-generated answer text to "Send AI Response via Telegram"  
  - Edge Cases: AI hallucination, transcript retrieval failure.

- **OpenAI Chat Model**  
  - Type: `lmChatOpenAi` (LangChain)  
  - Role: GPT-4o-mini model used to generate answers.  
  - Configuration: Model set to "gpt-4o-mini".  
  - Inputs: Prompt from "Handle User Questions via AI"  
  - Outputs: AI response text  
  - Credentials: OpenAI API key  
  - Edge Cases: API errors, rate limits.

- **Window Buffer Memory**  
  - Type: `memoryBufferWindow` (LangChain)  
  - Role: Maintains conversational context per user session keyed by user message text.  
  - Configuration: Session key set dynamically to user message text.  
  - Inputs: User message text  
  - Outputs: Contextual memory for AI agent  
  - Edge Cases: Memory overflow, session key collisions.

- **Google Docs2**  
  - Type: `googleDocsTool`  
  - Role: Retrieves the transcript content from Google Docs to provide context for AI answers.  
  - Configuration: Operation `get`, fixed document URL.  
  - Inputs: Triggered by AI agent  
  - Outputs: Transcript content for AI processing  
  - Credentials: Google Docs OAuth2  
  - Edge Cases: Document access errors.

- **Send AI Response via Telegram**  
  - Type: `telegram`  
  - Role: Sends the AI-generated answer back to the user via Telegram.  
  - Configuration: Sends `output` field with HTML parse mode, disables attribution.  
  - Inputs: AI answer text from "Handle User Questions via AI"  
  - Outputs: Confirmation of message sent  
  - Credentials: Telegram API OAuth2  
  - Edge Cases: Telegram API failures, message formatting issues.

---

### 3. Summary Table

| Node Name                      | Node Type                          | Functional Role                           | Input Node(s)                     | Output Node(s)                          | Sticky Note                                                                                          |
|--------------------------------|----------------------------------|-----------------------------------------|----------------------------------|---------------------------------------|----------------------------------------------------------------------------------------------------|
| Trigger on Telegram Message     | chatTrigger (LangChain)           | Receives YouTube URL via Telegram       | None                             | Extract YouTube URL from Input         | Get video url via webhook or message. For Apple users, use shortcut to share video directly to n8n. |
| Receive YouTube URL via Webhook | webhook                          | Receives YouTube URL via HTTP webhook   | None                             | Extract YouTube URL from Input         | Get video url via webhook or message. For Apple users, use shortcut to share video directly to n8n. |
| Extract YouTube URL from Input  | set                             | Extracts YouTube URL from input JSON    | Trigger on Telegram Message, Receive YouTube URL via Webhook | Extract Video ID from URL               |                                                                                                    |
| Extract Video ID from URL       | code (Python)                   | Parses YouTube URL to get video ID       | Extract YouTube URL from Input   | Extract YouTube Transcript             |                                                                                                    |
| Extract YouTube Transcript      | youtubeTranscripter (Community) | Retrieves transcript from YouTube video | Extract Video ID from URL        | Split Transcript into Segments         |                                                                                                    |
| Split Transcript into Segments  | splitOut                       | Splits transcript array into segments    | Extract YouTube Transcript       | Concatenate Transcript Segments        |                                                                                                    |
| Concatenate Transcript Segments | summarize                      | Concatenates transcript segments         | Split Transcript into Segments   | Generate Summary with GPT-4o-mini, Retrieve Transcript from Google Docs |                                                                                                    |
| Generate Summary with GPT-4o-mini | chainLlm (LangChain)           | Generates structured summary from transcript | Concatenate Transcript Segments | Send Summary via Telegram              |                                                                                                    |
| gpt-4o-mini                    | lmChatOpenAi (LangChain)         | GPT-4o-mini AI model for summarization  | Generate Summary with GPT-4o-mini | Generate Summary with GPT-4o-mini      |                                                                                                    |
| Send Summary via Telegram       | telegram                       | Sends summary text to Telegram chat     | Generate Summary with GPT-4o-mini | Send Response to Webhook               |                                                                                                    |
| Send Response to Webhook        | respondToWebhook               | Sends HTTP response to webhook caller   | Send Summary via Telegram        | None                                  |                                                                                                    |
| Retrieve Transcript from Google Docs | googleDocs                   | Retrieves transcript content for Q&A    | Concatenate Transcript Segments  | Update Transcript in Google Docs       | Load memory: Upload transcript to Google Docs for Q&A.                                            |
| Update Transcript in Google Docs | googleDocs                    | Updates Google Docs document with transcript | Retrieve Transcript from Google Docs | None                                  | Load memory: Upload transcript to Google Docs for Q&A.                                            |
| Telegram Trigger               | telegramTrigger                 | Listens for user questions on Telegram  | None                             | Handle User Questions via AI           | Ask AI about video                                                                                  |
| Handle User Questions via AI    | agent (LangChain)              | Processes user questions with AI         | Telegram Trigger, OpenAI Chat Model, Google Docs2 | Send AI Response via Telegram          | Ask AI about video                                                                                  |
| OpenAI Chat Model              | lmChatOpenAi (LangChain)         | GPT-4o-mini AI model for Q&A             | Handle User Questions via AI     | Handle User Questions via AI            | Ask AI about video                                                                                  |
| Window Buffer Memory           | memoryBufferWindow (LangChain)   | Maintains conversational context         | Handle User Questions via AI     | Handle User Questions via AI            | Ask AI about video                                                                                  |
| Google Docs2                  | googleDocsTool                  | Retrieves transcript for AI context      | Handle User Questions via AI     | Handle User Questions via AI            | Ask AI about video                                                                                  |
| Send AI Response via Telegram   | telegram                       | Sends AI-generated answers to Telegram   | Handle User Questions via AI     | None                                  | Ask AI about video                                                                                  |
| Sticky Note                   | stickyNote                     | Instructional note                       | None                             | None                                  | Get video url via webhook or message. For Apple users, use shortcut to share video directly to n8n. |
| Sticky Note1                  | stickyNote                     | Instructional note                       | None                             | None                                  | Load memory: Upload transcript to Google Docs for Q&A.                                            |
| Sticky Note2                  | stickyNote                     | Instructional note                       | None                             | None                                  | Ask AI about video                                                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes for Input Reception**  
   - Add a **Telegram Trigger** node (`telegramTrigger`) configured with your Telegram bot credentials to listen for incoming messages.  
   - Add a **Webhook** node configured with a unique path to receive HTTP requests containing YouTube URLs.

2. **Normalize Input URL**  
   - Add a **Set** node named "Extract YouTube URL from Input".  
   - Configure it to set a new string field `youtubeUrl` with expression: `={{ $json.chatInput || $json.query.url }}`.  
   - Connect both Telegram Trigger and Webhook nodes to this Set node.

3. **Extract YouTube Video ID**  
   - Add a **Code** node (Python) named "Extract Video ID from URL".  
   - Implement Python code to parse `youtubeUrl` and extract the video ID (e.g., regex or URL parsing).  
   - Connect "Extract YouTube URL from Input" to this node.

4. **Extract YouTube Transcript**  
   - Install the community node `n8n-nodes-youtube-transcription-kasha` via npm on your self-hosted n8n instance.  
   - Add the **youtubeTranscripter** node named "Extract YouTube Transcript".  
   - Configure it to use the `videoId` from the previous node.  
   - Connect "Extract Video ID from URL" to this node.

5. **Split Transcript into Segments**  
   - Add a **SplitOut** node named "Split Transcript into Segments".  
   - Configure it to split the field `transcript`.  
   - Connect "Extract YouTube Transcript" to this node.

6. **Concatenate Transcript Segments**  
   - Add a **Summarize** node named "Concatenate Transcript Segments".  
   - Configure it to concatenate the `text` fields separated by space.  
   - Connect "Split Transcript into Segments" to this node.

7. **Update Transcript in Google Docs**  
   - Add a **Google Docs** node named "Update Transcript in Google Docs".  
   - Configure operation as `update` with action `replaceAll`.  
   - Set the document URL dynamically from input JSON (`={{ $json.documentId }}`).  
   - Replace text with the concatenated transcript (`={{ $('Concatenate Transcript Segments').item.json.concatenated_text }}`).  
   - Connect "Retrieve Transcript from Google Docs" to this node (see step 10).  
   - Configure Google Docs OAuth2 credentials.

8. **Generate Summary with GPT-4o-mini**  
   - Add a **chainLlm** node named "Generate Summary with GPT-4o-mini".  
   - Use a detailed prompt instructing the AI to produce a structured summary with general summary, key moments, and instructions, formatted in markdown with HTML bold tags.  
   - Connect "Concatenate Transcript Segments" to this node.

9. **Configure GPT-4o-mini Model Node**  
   - Add an **lmChatOpenAi** node named "gpt-4o-mini".  
   - Set model to GPT-4o-mini and connect it as the language model for the chainLlm node.  
   - Provide OpenAI API credentials.

10. **Retrieve Transcript from Google Docs for Q&A**  
    - Add a **Google Docs** node named "Retrieve Transcript from Google Docs".  
    - Configure operation `get` with a fixed document URL containing the transcript.  
    - Connect "Concatenate Transcript Segments" to this node to trigger transcript retrieval.

11. **Send Summary via Telegram**  
    - Add a **Telegram** node named "Send Summary via Telegram".  
    - Configure to send the summary text (`={{ $json.text }}`) with HTML parse mode.  
    - Append the original YouTube URL for reference.  
    - Connect "Generate Summary with GPT-4o-mini" to this node.  
    - Provide Telegram API credentials.

12. **Send Response to Webhook**  
    - Add a **Respond to Webhook** node named "Send Response to Webhook".  
    - Connect "Send Summary via Telegram" to this node to respond to webhook callers.

13. **Setup Interactive Q&A Handling**  
    - Add a **Telegram Trigger** node named "Telegram Trigger" to listen for user questions.  
    - Add an **agent** node named "Handle User Questions via AI".  
      - Configure with system message instructing AI to answer based on transcript content from Google Docs.  
      - Connect "Telegram Trigger" to this node.

14. **Configure OpenAI Chat Model for Q&A**  
    - Add an **lmChatOpenAi** node named "OpenAI Chat Model".  
    - Set model to GPT-4o-mini and connect it as the language model for the agent node.  
    - Provide OpenAI API credentials.

15. **Add Window Buffer Memory**  
    - Add a **memoryBufferWindow** node named "Window Buffer Memory".  
    - Configure session key dynamically based on user message text (`={{ $json.message.text }}`).  
    - Connect it to the agent node to maintain conversational context.

16. **Add Google Docs Tool for Transcript Retrieval**  
    - Add a **googleDocsTool** node named "Google Docs2".  
    - Configure to get transcript content from the fixed Google Docs document URL.  
    - Connect it to the agent node.

17. **Send AI Response via Telegram**  
    - Add a **Telegram** node named "Send AI Response via Telegram".  
    - Configure to send AI-generated answers with HTML parse mode.  
    - Connect "Handle User Questions via AI" to this node.  
    - Provide Telegram API credentials.

18. **Test the Workflow**  
    - Send a YouTube URL via Telegram or webhook.  
    - Verify transcript extraction, summarization, and summary delivery.  
    - Ask questions via Telegram and verify AI responses.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                     |
|----------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| The `youtubeTranscripter` community node requires a self-hosted n8n instance due to external dependencies. | https://www.npmjs.com/package/n8n-nodes-youtube-transcription-kasha                               |
| Apple users can enhance input by using a custom shortcut to share YouTube videos directly to n8n.  | https://routinehub.co/shortcut/21757/                                                             |
| Prompt for summarization uses markdown headers and HTML bold tags for Telegram compatibility.      | Ensures clean display without unsupported Telegram features like nested lists or tables.          |
| The workflow supports multiple input methods (Telegram and webhook) for flexibility.                | Allows integration with various platforms and automation shortcuts.                               |
| Google Docs is used as persistent storage for transcripts to enable Q&A functionality.              | Requires OAuth2 credentials with appropriate document access permissions.                          |

---

This documentation provides a detailed, structured understanding of the workflow, enabling reproduction, modification, and troubleshooting by advanced users and AI agents alike.