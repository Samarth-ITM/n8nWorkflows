Self-Learning AI Assistant with Permanent Memory | GPT,Telegram & Pinecone RAG

https://n8nworkflows.xyz/workflows/self-learning-ai-assistant-with-permanent-memory---gpt-telegram---pinecone-rag-3183


# Self-Learning AI Assistant with Permanent Memory | GPT,Telegram & Pinecone RAG

### 1. Workflow Overview

This workflow implements a **Self-Learning AI Assistant with Permanent Memory** that integrates Telegram, OpenAI, Pinecone vector database, and Google Docs to create a personal AI secretary. It captures and processes multi-modal inputs (text, audio, images) from Telegram, stores knowledge permanently in a vector database, and continuously learns from daily accumulated data.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Preprocessing:** Receives messages from Telegram (text, audio, images), routes them by type, and preprocesses each accordingly (transcription for audio, content extraction for images, direct text handling).

- **1.2 AI Processing and Response Generation:** Uses OpenAI models and an AI Agent with memory and vector store tools to understand queries, search knowledge base, and generate context-aware responses.

- **1.3 Knowledge Storage and Vectorization:** Stores new knowledge statements into Google Docs ("memory palace") and converts daily accumulated data into vector embeddings stored in Pinecone for retrieval.

- **1.4 Daily Self-Learning Automation:** Scheduled trigger that loads the daily Google Docs content, splits text, generates embeddings, and updates the Pinecone vector store to enable continuous learning.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Preprocessing

**Overview:**  
This block captures incoming Telegram messages, distinguishes between text, audio, and images, and preprocesses each type for downstream AI processing.

**Nodes Involved:**  
- Telegram Trigger  
- Switch  
- DownloadAudio  
- OpenAI (for audio transcription)  
- OpenAI1 (for image analysis)  
- Edit Fields_Text  
- Edit Fields_Audio  
- Edit Fields_Image  

**Node Details:**

- **Telegram Trigger**  
  - Type: Telegram Trigger  
  - Role: Entry point listening for all Telegram messages sent to the bot.  
  - Configuration: Default webhook, no filters (captures all message types).  
  - Inputs: Telegram messages from users.  
  - Outputs: Routes to Switch node.  
  - Edge Cases: Telegram API downtime, webhook misconfiguration, message format changes.

- **Switch**  
  - Type: Switch  
  - Role: Routes incoming Telegram messages based on content type (text, audio, image).  
  - Configuration: Conditions to detect message type (text message, audio note, image).  
  - Inputs: Telegram Trigger output.  
  - Outputs:  
    - Text messages → Edit Fields_Text  
    - Audio messages → DownloadAudio  
    - Images → OpenAI1  
  - Edge Cases: Unrecognized message types, missing media content.

- **DownloadAudio**  
  - Type: Telegram node (file download)  
  - Role: Downloads audio files from Telegram messages for transcription.  
  - Configuration: Uses Telegram credentials to fetch audio file.  
  - Inputs: Audio messages from Switch.  
  - Outputs: OpenAI node for transcription.  
  - Edge Cases: Audio file missing or inaccessible, Telegram API errors.

- **OpenAI (Audio Transcription)**  
  - Type: OpenAI (Langchain)  
  - Role: Transcribes audio files into text using OpenAI Whisper or similar model.  
  - Configuration: Uses OpenAI API key, transcription model selected.  
  - Inputs: Audio file from DownloadAudio.  
  - Outputs: Edit Fields_Audio for formatting transcription output.  
  - Edge Cases: API rate limits, transcription errors, unsupported audio formats.

- **OpenAI1 (Image Analysis)**  
  - Type: OpenAI (Langchain)  
  - Role: Analyzes images to extract text or relevant content.  
  - Configuration: Uses OpenAI vision or image analysis model.  
  - Inputs: Image messages from Switch.  
  - Outputs: Edit Fields_Image for formatting extracted data.  
  - Edge Cases: Image quality issues, unsupported image formats, API errors.

- **Edit Fields_Text**  
  - Type: Set node  
  - Role: Formats and prepares text messages for AI Agent input.  
  - Configuration: Sets standardized fields (e.g., message content, metadata).  
  - Inputs: Text messages from Switch.  
  - Outputs: AI Agent node.  
  - Edge Cases: Missing or malformed text data.

- **Edit Fields_Audio**  
  - Type: Set node  
  - Role: Formats transcribed audio text for AI Agent input.  
  - Configuration: Sets fields with transcription text and metadata.  
  - Inputs: Transcribed text from OpenAI.  
  - Outputs: AI Agent node.  
  - Edge Cases: Empty transcription, transcription inaccuracies.

- **Edit Fields_Image**  
  - Type: Set node  
  - Role: Formats image analysis output for AI Agent input.  
  - Configuration: Sets extracted text or description fields.  
  - Inputs: Image analysis output from OpenAI1.  
  - Outputs: AI Agent node.  
  - Edge Cases: No text extracted, irrelevant content.

---

#### 2.2 AI Processing and Response Generation

**Overview:**  
This block processes the formatted input using an AI Agent that leverages OpenAI language models, vector store querying, memory buffers, and Google Docs tools to generate intelligent, context-aware responses.

**Nodes Involved:**  
- AI Agent  
- Deepseek (OpenAI Chat model)  
- Window Buffer Memory  
- Pinecone Vector Store  
- Google Docs  

**Node Details:**

- **AI Agent**  
  - Type: Langchain Agent  
  - Role: Core AI logic node that interprets input, queries vector store, uses memory, and generates responses.  
  - Configuration: Uses OpenAI API, connected to Pinecone vector store and Google Docs as tools, integrates Window Buffer Memory for short-term context.  
  - Inputs: Formatted text/audio/image from Edit Fields nodes.  
  - Outputs: Telegram node for response delivery.  
  - Edge Cases: API failures, memory overflow, vector store query errors, malformed input.

- **Deepseek**  
  - Type: Langchain OpenAI Chat Model  
  - Role: Provides language model capabilities to AI Agent for chat completions.  
  - Configuration: OpenAI chat model with specified parameters (temperature, max tokens).  
  - Inputs: AI Agent language model requests.  
  - Outputs: AI Agent.  
  - Edge Cases: API rate limits, model unavailability.

- **Window Buffer Memory**  
  - Type: Langchain Memory Buffer  
  - Role: Maintains recent conversation history for context-aware responses.  
  - Configuration: Sliding window size configured to limit memory length.  
  - Inputs: AI Agent memory input.  
  - Outputs: AI Agent memory output.  
  - Edge Cases: Memory size limits, context truncation.

- **Pinecone Vector Store**  
  - Type: Langchain Vector Store (Pinecone)  
  - Role: Provides retrieval of relevant knowledge vectors for AI Agent queries.  
  - Configuration: Pinecone API key, environment, index, and namespace configured.  
  - Inputs: Embeddings from AI Agent queries.  
  - Outputs: AI Agent tool input for retrieval.  
  - Edge Cases: Pinecone API errors, network issues, empty index.

- **Google Docs**  
  - Type: Google Docs Tool  
  - Role: Allows AI Agent to read/write knowledge statements to the "memory palace" document.  
  - Configuration: Google OAuth2 credentials, target Google Doc ID configured.  
  - Inputs: AI Agent tool input.  
  - Outputs: AI Agent tool output.  
  - Edge Cases: Google API quota exceeded, permission errors, document not found.

---

#### 2.3 Knowledge Storage and Vectorization

**Overview:**  
This block handles the permanent storage of knowledge statements into Google Docs and the conversion of daily accumulated knowledge into vector embeddings stored in Pinecone.

**Nodes Involved:**  
- Schedule Trigger  
- Google Docs1  
- Edit Fields  
- Pinecone Vector Store1  
- Embeddings OpenAI1  
- Default Data Loader  
- Recursive Character Text Splitter  
- Google Docs2  

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Initiates daily self-learning process at a configured time.  
  - Configuration: Cron or interval schedule (e.g., daily at midnight).  
  - Inputs: None (trigger only).  
  - Outputs: Google Docs1.  
  - Edge Cases: Missed triggers due to downtime.

- **Google Docs1**  
  - Type: Google Docs  
  - Role: Reads the daily "memory palace" document content.  
  - Configuration: Uses Google credentials and document ID.  
  - Inputs: Schedule Trigger.  
  - Outputs: Edit Fields.  
  - Edge Cases: Document access errors, empty document.

- **Edit Fields**  
  - Type: Set node  
  - Role: Prepares Google Docs content for embedding generation.  
  - Configuration: Sets or formats text fields for downstream nodes.  
  - Inputs: Google Docs1 output.  
  - Outputs: Pinecone Vector Store1.  
  - Edge Cases: Empty or malformed text.

- **Pinecone Vector Store1**  
  - Type: Langchain Vector Store (Pinecone)  
  - Role: Stores newly generated embeddings into Pinecone index.  
  - Configuration: Pinecone API credentials, index, and namespace.  
  - Inputs: Embeddings OpenAI1 output.  
  - Outputs: Google Docs2.  
  - Edge Cases: API errors, network issues.

- **Embeddings OpenAI1**  
  - Type: Langchain Embeddings (OpenAI)  
  - Role: Converts text into vector embeddings for Pinecone storage.  
  - Configuration: OpenAI API key, embedding model selected.  
  - Inputs: Text from Default Data Loader or Edit Fields.  
  - Outputs: Pinecone Vector Store1.  
  - Edge Cases: API rate limits, malformed text.

- **Default Data Loader**  
  - Type: Langchain Document Default Data Loader  
  - Role: Loads documents for embedding generation.  
  - Configuration: Configured to load Google Docs content or other sources.  
  - Inputs: Recursive Character Text Splitter output.  
  - Outputs: Embeddings OpenAI1.  
  - Edge Cases: Document loading errors.

- **Recursive Character Text Splitter**  
  - Type: Langchain Text Splitter  
  - Role: Splits large text into manageable chunks for embedding.  
  - Configuration: Recursive character splitting with chunk size and overlap.  
  - Inputs: Google Docs2 output.  
  - Outputs: Default Data Loader.  
  - Edge Cases: Improper splitting causing context loss.

- **Google Docs2**  
  - Type: Google Docs  
  - Role: Reads additional documents or updated content for embedding.  
  - Configuration: Google credentials, document ID.  
  - Inputs: Pinecone Vector Store1 output.  
  - Outputs: Recursive Character Text Splitter.  
  - Edge Cases: Access errors, empty content.

---

#### 2.4 Response Delivery

**Overview:**  
Delivers AI-generated responses back to the user on Telegram.

**Nodes Involved:**  
- Telegram  

**Node Details:**

- **Telegram**  
  - Type: Telegram node  
  - Role: Sends messages back to Telegram users with AI responses.  
  - Configuration: Uses Telegram bot credentials, sends text messages.  
  - Inputs: AI Agent output.  
  - Outputs: None (end of flow).  
  - Edge Cases: Telegram API errors, message delivery failures.

---

### 3. Summary Table

| Node Name                  | Node Type                               | Functional Role                         | Input Node(s)           | Output Node(s)           | Sticky Note                                                                 |
|----------------------------|---------------------------------------|---------------------------------------|------------------------|--------------------------|-----------------------------------------------------------------------------|
| Telegram Trigger            | Telegram Trigger                      | Entry point for Telegram messages     | -                      | Switch                   |                                                                             |
| Switch                     | Switch                               | Routes messages by type                | Telegram Trigger       | Edit Fields_Text, DownloadAudio, OpenAI1 |                                                                             |
| DownloadAudio              | Telegram (file download)              | Downloads audio files                  | Switch                 | OpenAI                   |                                                                             |
| OpenAI                     | OpenAI (Langchain)                    | Transcribes audio to text              | DownloadAudio          | Edit Fields_Audio        |                                                                             |
| OpenAI1                    | OpenAI (Langchain)                    | Analyzes images                       | Switch                 | Edit Fields_Image        |                                                                             |
| Edit Fields_Text           | Set                                  | Formats text messages                  | Switch                 | AI Agent                 |                                                                             |
| Edit Fields_Audio          | Set                                  | Formats transcribed audio text         | OpenAI                  | AI Agent                 |                                                                             |
| Edit Fields_Image          | Set                                  | Formats image analysis output          | OpenAI1                 | AI Agent                 |                                                                             |
| AI Agent                   | Langchain Agent                      | Core AI processing and response        | Edit Fields_Text, Edit Fields_Audio, Edit Fields_Image | Telegram                  |                                                                             |
| Telegram                   | Telegram                             | Sends AI responses back to user        | AI Agent               | -                        |                                                                             |
| Deepseek                   | Langchain OpenAI Chat Model          | Provides language model for AI Agent   | AI Agent (ai_languageModel) | AI Agent               |                                                                             |
| Window Buffer Memory       | Langchain Memory Buffer              | Maintains recent conversation context | AI Agent (ai_memory)   | AI Agent                 |                                                                             |
| Pinecone Vector Store      | Langchain Vector Store (Pinecone)    | Retrieves relevant knowledge vectors   | Embeddings OpenAI      | AI Agent (ai_tool)       |                                                                             |
| Google Docs                | Google Docs Tool                     | Reads/writes knowledge statements      | AI Agent (ai_tool)     | AI Agent                 |                                                                             |
| Schedule Trigger           | Schedule Trigger                    | Triggers daily self-learning process   | -                      | Google Docs1             |                                                                             |
| Google Docs1               | Google Docs                         | Reads daily memory palace document     | Schedule Trigger       | Edit Fields              |                                                                             |
| Edit Fields                | Set                                  | Prepares text for embedding            | Google Docs1           | Pinecone Vector Store1   |                                                                             |
| Pinecone Vector Store1     | Langchain Vector Store (Pinecone)    | Stores new embeddings                   | Embeddings OpenAI1     | Google Docs2             |                                                                             |
| Embeddings OpenAI1         | Langchain Embeddings (OpenAI)        | Generates vector embeddings             | Default Data Loader    | Pinecone Vector Store1   |                                                                             |
| Default Data Loader        | Langchain Document Default Data Loader| Loads documents for embedding           | Recursive Character Text Splitter | Embeddings OpenAI1 |                                                                             |
| Recursive Character Text Splitter | Langchain Text Splitter         | Splits text into chunks for embedding  | Google Docs2           | Default Data Loader      |                                                                             |
| Google Docs2               | Google Docs                         | Reads additional documents              | Pinecone Vector Store1 | Recursive Character Text Splitter |                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure with your Telegram bot API credentials.  
   - Set to listen for all message types (text, audio, images).  
   - Connect output to Switch node.

2. **Create Switch Node**  
   - Type: Switch  
   - Add conditions to detect message type:  
     - Text message → route to Edit Fields_Text  
     - Audio message → route to DownloadAudio  
     - Image message → route to OpenAI1  
   - Connect Telegram Trigger output to Switch input.

3. **Create DownloadAudio Node**  
   - Type: Telegram node (file download)  
   - Configure with Telegram credentials.  
   - Connect Switch audio output to DownloadAudio input.  
   - Connect output to OpenAI node for transcription.

4. **Create OpenAI Node (Audio Transcription)**  
   - Type: OpenAI (Langchain)  
   - Configure with OpenAI API key.  
   - Select transcription model (e.g., Whisper).  
   - Connect DownloadAudio output to OpenAI input.  
   - Connect output to Edit Fields_Audio.

5. **Create OpenAI1 Node (Image Analysis)**  
   - Type: OpenAI (Langchain)  
   - Configure with OpenAI API key.  
   - Select image analysis or vision model.  
   - Connect Switch image output to OpenAI1 input.  
   - Connect output to Edit Fields_Image.

6. **Create Edit Fields_Text Node**  
   - Type: Set node  
   - Configure to set standardized fields for text messages (e.g., message content, user ID).  
   - Connect Switch text output to Edit Fields_Text input.  
   - Connect output to AI Agent.

7. **Create Edit Fields_Audio Node**  
   - Type: Set node  
   - Configure to set fields with transcribed text and metadata.  
   - Connect OpenAI output to Edit Fields_Audio input.  
   - Connect output to AI Agent.

8. **Create Edit Fields_Image Node**  
   - Type: Set node  
   - Configure to set fields with extracted image text or description.  
   - Connect OpenAI1 output to Edit Fields_Image input.  
   - Connect output to AI Agent.

9. **Create AI Agent Node**  
   - Type: Langchain Agent  
   - Configure with OpenAI API key and link to:  
     - Deepseek (OpenAI Chat model) as language model  
     - Pinecone Vector Store as retrieval tool  
     - Google Docs as knowledge base tool  
     - Window Buffer Memory for short-term memory  
   - Connect Edit Fields_* nodes outputs to AI Agent input.  
   - Connect AI Agent output to Telegram node.

10. **Create Deepseek Node**  
    - Type: Langchain OpenAI Chat Model  
    - Configure with OpenAI API key, set parameters (temperature, max tokens).  
    - Connect AI Agent ai_languageModel input and output.

11. **Create Window Buffer Memory Node**  
    - Type: Langchain Memory Buffer  
    - Configure sliding window size for recent conversation context.  
    - Connect AI Agent ai_memory input and output.

12. **Create Pinecone Vector Store Node**  
    - Type: Langchain Vector Store (Pinecone)  
    - Configure with Pinecone API key, environment, index, and namespace.  
    - Connect Embeddings OpenAI output to Pinecone input.  
    - Connect Pinecone output to AI Agent ai_tool input.

13. **Create Google Docs Node (AI Tool)**  
    - Type: Google Docs Tool  
    - Configure with Google OAuth2 credentials and target Google Doc ID ("memory palace").  
    - Connect AI Agent ai_tool input and output.

14. **Create Telegram Node**  
    - Type: Telegram node  
    - Configure with Telegram bot credentials.  
    - Connect AI Agent output to Telegram input for sending responses.

15. **Create Schedule Trigger Node**  
    - Type: Schedule Trigger  
    - Configure to run daily at desired time (e.g., midnight).  
    - Connect output to Google Docs1.

16. **Create Google Docs1 Node**  
    - Type: Google Docs  
    - Configure with Google credentials and daily memory palace document ID.  
    - Connect Schedule Trigger output to Google Docs1 input.  
    - Connect output to Edit Fields.

17. **Create Edit Fields Node**  
    - Type: Set node  
    - Configure to prepare text content for embedding generation.  
    - Connect Google Docs1 output to Edit Fields input.  
    - Connect output to Pinecone Vector Store1.

18. **Create Pinecone Vector Store1 Node**  
    - Type: Langchain Vector Store (Pinecone)  
    - Configure with Pinecone API key, environment, index, and namespace.  
    - Connect Embeddings OpenAI1 output to Pinecone Vector Store1 input.  
    - Connect output to Google Docs2.

19. **Create Embeddings OpenAI1 Node**  
    - Type: Langchain Embeddings (OpenAI)  
    - Configure with OpenAI API key and embedding model.  
    - Connect Default Data Loader output to Embeddings OpenAI1 input.  
    - Connect output to Pinecone Vector Store1.

20. **Create Default Data Loader Node**  
    - Type: Langchain Document Default Data Loader  
    - Configure to load documents (Google Docs content).  
    - Connect Recursive Character Text Splitter output to Default Data Loader input.  
    - Connect output to Embeddings OpenAI1.

21. **Create Recursive Character Text Splitter Node**  
    - Type: Langchain Text Splitter  
    - Configure chunk size and overlap for splitting large text.  
    - Connect Google Docs2 output to Recursive Character Text Splitter input.  
    - Connect output to Default Data Loader.

22. **Create Google Docs2 Node**  
    - Type: Google Docs  
    - Configure with Google credentials and document ID for additional content.  
    - Connect Pinecone Vector Store1 output to Google Docs2 input.  
    - Connect output to Recursive Character Text Splitter.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                          |
|---------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Video guidance on Telegram integration setup is included in the workflow.                                      | Refer to the workflow documentation or attached video tutorial.                                         |
| This workflow leverages Retrieval-Augmented Generation (RAG) to combine vector search with language models.   | RAG concept: https://www.pinecone.io/learn/retrieval-augmented-generation/                              |
| Google Docs serves as a "memory palace" for permanent knowledge storage, enabling daily self-learning.        | Requires Google OAuth2 credentials and document setup.                                                  |
| OpenAI Whisper model is used for audio transcription, enabling voice note processing.                          | OpenAI Whisper info: https://openai.com/blog/whisper                                                     |
| Pinecone vector database is used for real-time vector storage and retrieval of knowledge embeddings.          | Pinecone docs: https://docs.pinecone.io/                                                                |
| The AI Agent node integrates multiple tools and memory buffers for advanced conversational capabilities.      | Langchain Agents documentation: https://js.langchain.com/docs/modules/agents/                            |

---

This structured documentation provides a comprehensive understanding of the workflow, enabling advanced users and AI agents to reproduce, modify, and troubleshoot the Self-Learning AI Assistant with Permanent Memory workflow effectively.