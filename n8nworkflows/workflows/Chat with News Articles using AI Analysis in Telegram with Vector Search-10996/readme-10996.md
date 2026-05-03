Chat with News Articles using AI Analysis in Telegram with Vector Search

https://n8nworkflows.xyz/workflows/chat-with-news-articles-using-ai-analysis-in-telegram-with-vector-search-10996


# Chat with News Articles using AI Analysis in Telegram with Vector Search

### 1. Workflow Overview

This workflow enables interactive chat-based summarization and exploration of news articles shared via Telegram links, leveraging AI analysis and vector search capabilities.  
It is designed to:  
- Receive Telegram messages containing URLs to news articles  
- Scrape and extract structured content (title, text, images, metadata) from the provided URLs  
- Use a Vision-Language Model (VLM) agent to generate highlights, summaries, and image extraction from the scraped content  
- Validate and download images linked in the summary  
- Send summarized reports and extracted documents back to the Telegram user  
- Store the summary embeddings in an in-memory vector store for semantic question-answering and follow-up chats  
- Provide an AI agent interface to interactively answer user queries based on the stored knowledge base

**Logical Blocks:**  
- 1.1 Telegram Input Reception and URL Extraction  
- 1.2 Web Page Scraping and Content Extraction  
- 1.3 Vision-Language Model Processing and Summary Generation  
- 1.4 Image Validation and Downloading  
- 1.5 Sending Summaries and Documents Back to Telegram  
- 1.6 Embedding Storage and Vector Search for Conversational AI  
- 1.7 AI Agent for Interactive Q&A in Telegram

---

### 2. Block-by-Block Analysis

#### Block 1.1: Telegram Input Reception and URL Extraction

**Overview:**  
This block listens for Telegram messages in a specified chat, detects if the message contains a valid URL, extracts it, and prepares it for scraping.

**Nodes Involved:**  
- Listen to Telegram for Link  
- Check Whether URL  
- Rename Link Field

**Node Details:**  

- **Listen to Telegram for Link**  
  - Type: Telegram Trigger  
  - Role: Listens to incoming Telegram messages filtered by chat ID  
  - Config: Subscribes to "message" updates only, restricted to chat ID 1872183963  
  - Output: Raw Telegram message JSON including potential link previews  
  - Failures: Network errors, invalid credentials, webhook misconfiguration

- **Check Whether URL**  
  - Type: If Condition  
  - Role: Checks if the incoming message contains a valid URL under `$json.message.link_preview_options.url` and that it is boolean true (exists)  
  - Logic: Boolean check for URL presence  
  - Outputs:  
    - True: URL exists → proceed to Rename Link Field  
    - False: No URL → fallback to AI Agent for direct chat  
  - Edge Cases: Messages without URL previews, malformed URLs

- **Rename Link Field**  
  - Type: Set  
  - Role: Extracts and renames the URL field from Telegram message to `URL_PARSE` for downstream processing  
  - Config: Assigns `URL_PARSE` = `{{$json.message.link_preview_options.url}}`  
  - Output: Sets up URL for scraping node  
  - Edge Cases: Missing or invalid URL string

---

#### Block 1.2: Web Page Scraping and Content Extraction

**Overview:**  
Fetches the full HTML content of the provided URL, extracts meaningful textual and media information using regex parsing, and compiles structured JSON data.

**Nodes Involved:**  
- Code (Function)  
- VLM Run Highlighter

**Node Details:**  

- **Code**  
  - Type: Function (JavaScript)  
  - Role:  
    - Executes HTTP GET request to fetch raw HTML of the URL (`URL_PARSE`)  
    - Extracts: `<title>`, meta description, all `<p>` paragraphs, all `<img>` URLs, Open Graph title/image metadata  
    - Creates a short summary from first 3 paragraphs  
    - Error handling: on fetch or parsing failure, returns nulls and empty arrays to downstream nodes  
  - Inputs: `URL_PARSE` string  
  - Outputs: JSON enriched with extracted fields: `title`, `description`, `text`, `images`, `summary`, `source_url`  
  - Edge Cases: HTTP errors, malformed HTML, no tags found, slow responses, timeouts

- **VLM Run Highlighter**  
  - Type: LangChain Agent (AI Agent)  
  - Role: Uses a Vision-Language Model (VLM) to analyze the scraped content and generate a structured highlight summary  
  - Input: Concatenated string of extracted fields (`source_url`, `title`, `description`, `text`, `images`, `summary`)  
  - Prompt: Instructed to highlight important news, provide correct complete URLs for images, and format output as per JSON schema  
  - Output: Parsed structured JSON summary with keys like `headline`, `key_points`, `summary`, `extracted_images_url`  
  - Edge Cases: AI timeout, incomplete input data, malformed output, overlength input

---

#### Block 1.3: Vision-Language Model Processing and Summary Generation

**Overview:**  
Processes the highlights from VLM, parses the structured JSON output, and prepares image URL data for validation and download.

**Nodes Involved:**  
- Structured Output Parser1  
- Check URLs Validity  
- Split Out  
- HTTP Request1

**Node Details:**  

- **Structured Output Parser1**  
  - Type: LangChain Structured Output Parser  
  - Role: Parses AI-generated JSON output to enforce schema correctness and auto-fixes minor errors  
  - Config: JSON schema example includes fields for headline, source_url, published_date, key_points, summary, extracted_images_url  
  - Input: Raw AI agent output from VLM Run Highlighter  
  - Output: Clean structured JSON with extracted news summary  
  - Edge Cases: Invalid JSON, parsing failures, schema mismatch

- **Check URLs Validity**  
  - Type: If Condition  
  - Role: Checks if `extracted_images_url` in the parsed summary is non-empty (string not equal to empty string)  
  - Outputs:  
    - True: Valid URLs exist → proceed to Split Out  
    - False: No URLs → No Operation (skip image downloading)  
  - Edge Cases: Empty strings, null fields, malformed URLs

- **Split Out**  
  - Type: Split Out  
  - Role: Splits the `extracted_images_url` array of URLs into individual items for separate processing  
  - Input: `output.research_paper_summary.extracted_images_url` (array)  
  - Output: Each URL as separate item for HTTP request  
  - Edge Cases: Empty arrays, non-array inputs

- **HTTP Request1**  
  - Type: HTTP Request  
  - Role: Downloads each image by URL to get binary data for sending to Telegram  
  - Config: URL taken dynamically from split-out URLs  
  - Output: Binary image data attached to item  
  - Failures: 404 errors, timeouts, invalid URLs, large files

---

#### Block 1.4: Image Validation and Downloading

**Overview:**  
Handles downloaded images, converts the final summary to a text file, and prepares documents for Telegram delivery.

**Nodes Involved:**  
- Covert to Text File  
- Send a document  
- Start Asking

**Node Details:**  

- **Covert to Text File**  
  - Type: Convert To File  
  - Role: Converts the JSON summary output into a text file format for sending as a document  
  - Input: Summary JSON under `output` field  
  - Output: Text file binary data  
  - Edge Cases: Large JSON causing file size issues

- **Send a document**  
  - Type: Telegram  
  - Role: Sends the converted text file and downloaded image documents back to Telegram chat  
  - Config: Uses chat ID 1872183963, sends binary document data  
  - Output: Confirmation message, triggers next node  
  - Failure Cases: Telegram API errors, file size limits, network errors

- **Start Asking**  
  - Type: Telegram  
  - Role: Sends a Telegram message prompting user to start asking questions about the provided link  
  - Config: Sends "Start asking about provided link now" with forced reply enabled  
  - Output: Message sent to user, awaits replies  
  - Edge Cases: Telegram message delivery failures

---

#### Block 1.5: Embedding Storage and Vector Search for Conversational AI

**Overview:**  
Converts the summary text file into embeddings and stores them in an in-memory vector store for semantic retrieval and Q&A.

**Nodes Involved:**  
- Default Data Loader  
- Embeddings OpenAI  
- Insert Data to Store  
- Query Data Tool

**Node Details:**  

- **Default Data Loader**  
  - Type: Document Default Data Loader (LangChain)  
  - Role: Prepares binary text file data for embedding operations  
  - Config: Data type set to binary  
  - Input: Text file from Convert to Text File node  
  - Output: Document object for embedding  
  - Edge Cases: Invalid binary data, empty files

- **Embeddings OpenAI**  
  - Type: LangChain OpenAI Embeddings  
  - Role: Generates embedding vectors from textual document data using OpenAI API  
  - Credentials: OpenAI API key configured  
  - Output: Embedding vectors for storage and retrieval  
  - Failures: API quota exceeded, network issues

- **Insert Data to Store**  
  - Type: Vector Store In-Memory  
  - Role: Inserts embeddings into an in-memory vector store keyed by `vector_store_key` for later retrieval  
  - Config: Mode set to "insert"  
  - Output: Confirmation of insertion  
  - Edge Cases: Memory overflow, duplicate keys

- **Query Data Tool**  
  - Type: Vector Store In-Memory (Retrieve as Tool)  
  - Role: Retrieves relevant embeddings for AI agent queries from the same vector store  
  - Config: Tool named "knowledge_base" with description for Q&A usage  
  - Output: Retrieved documents passed to AI Agent as a tool  
  - Edge Cases: Empty vector store, no relevant results

---

#### Block 1.6: AI Agent for Interactive Q&A in Telegram

**Overview:**  
Handles user queries in Telegram and uses the vector store retrieval and OpenAI chat model to answer questions based on stored news summaries.

**Nodes Involved:**  
- AI Agent  
- OpenAI Chat Model  
- Send a Reply

**Node Details:**  

- **AI Agent**  
  - Type: LangChain Agent  
  - Role: Main conversational AI node that:  
    - Receives user chat text from Telegram messages  
    - Uses vector store retrieval as a tool for knowledge base lookup  
    - Uses OpenAI Chat Model for generating responses  
  - Input: Telegram message text (fallback when no URL detected)  
  - Output: AI-generated answer JSON text  
  - Edge Cases: Missing vector data, API errors

- **OpenAI Chat Model**  
  - Type: LangChain OpenAI Chat Model  
  - Role: Provides conversational language model (GPT-4.1-nano) responses for AI Agent  
  - Credentials: OpenAI API key  
  - Config: Model set to GPT-4.1-nano  
  - Edge Cases: API rate limits, prompt size limits

- **Send a Reply**  
  - Type: Telegram  
  - Role: Sends the AI Agent’s response text back to the Telegram user  
  - Config: Uses chat ID 1872183963  
  - Edge Cases: Telegram delivery failures

---

### 3. Summary Table

| Node Name                  | Node Type                            | Functional Role                                   | Input Node(s)              | Output Node(s)              | Sticky Note                                                                                                         |
|----------------------------|------------------------------------|-------------------------------------------------|----------------------------|-----------------------------|---------------------------------------------------------------------------------------------------------------------|
| Listen to Telegram for Link | Telegram Trigger                   | Listens for Telegram messages in a chat         |                            | Check Whether URL            |                                                                                                                     |
| Check Whether URL           | If Condition                      | Checks if message contains valid URL             | Listen to Telegram for Link | Rename Link Field / AI Agent |                                                                                                                     |
| Rename Link Field           | Set                              | Extracts and renames URL field for scraping      | Check Whether URL           | Code                        |                                                                                                                     |
| Code                       | Function (JS)                    | Scrapes webpage HTML, extracts structured data   | Rename Link Field           | VLM Run Highlighter          | # Web Page Scraper: expects URL_PARSE, extracts title, description, paragraphs, images, Open Graph metadata, summary |
| VLM Run Highlighter         | LangChain Agent (VLM)            | Generates structured highlight summary           | Code                       | Covert to Text File / Check URLs Validity | # Daily Newspaper Summarizer: analyze text, extract images, output JSON schema                                     |
| Structured Output Parser1   | LangChain Output Parser          | Parses AI-generated JSON summary                  | VLM Run Highlighter         | VLM Run Highlighter          |                                                                                                                     |
| Check URLs Validity         | If Condition                    | Checks if extracted image URLs are non-empty      | Covert to Text File         | Split Out / No Operation     |                                                                                                                     |
| Split Out                  | Split Out                       | Splits image URLs array into individual URLs      | Check URLs Validity         | HTTP Request1                |                                                                                                                     |
| HTTP Request1              | HTTP Request                   | Downloads images from URLs                         | Split Out                   | Send a document              |                                                                                                                     |
| Covert to Text File         | Convert To File                 | Converts summary JSON to text file                 | VLM Run Highlighter         | Send a document, Insert Data to Store |                                                                                                                     |
| Send a document             | Telegram                       | Sends text file and images back to Telegram       | HTTP Request1, Covert to Text File | Start Asking                 |                                                                                                                     |
| Start Asking               | Telegram                       | Sends prompt for user to start asking questions   | Send a document             |                             |                                                                                                                     |
| Default Data Loader         | LangChain Document Loader       | Loads text file as document for embedding         | Covert to Text File         | Insert Data to Store         |                                                                                                                     |
| Embeddings OpenAI           | LangChain Embeddings OpenAI     | Creates vector embeddings from document            | Default Data Loader         | Insert Data to Store, Query Data Tool | # Embeddings: Insert and Retrieve use same node for consistency                                                   |
| Insert Data to Store        | Vector Store In-Memory          | Inserts embeddings into in-memory vector store    | Embeddings OpenAI, Default Data Loader | Query Data Tool             |                                                                                                                     |
| Query Data Tool             | Vector Store In-Memory (Retrieve as Tool) | Retrieves relevant embeddings for AI agent      | Insert Data to Store        | AI Agent                    |                                                                                                                     |
| AI Agent                   | LangChain Agent                | Handles chat queries using vector store and chat model | Check Whether URL (fallback), Query Data Tool | Send a Reply                 | # Chat with LLM: Ask anything and get answer in telegram about provided newspaper link                              |
| OpenAI Chat Model           | LangChain Chat Model OpenAI    | Provides GPT-4.1-nano responses for AI agent      | AI Agent                   | AI Agent                    |                                                                                                                     |
| Send a Reply               | Telegram                       | Sends AI-generated answers back to Telegram       | AI Agent                   |                             |                                                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure webhook ID, listen to "message" updates only  
   - Restrict to chat ID: 1872183963  
   - Connect output to "Check Whether URL"

2. **Create If Node (Check Whether URL)**  
   - Type: If  
   - Condition: Check if `{{$json.message.link_preview_options.url}}` exists and is boolean true  
   - True branch → "Rename Link Field" node  
   - False branch → direct to "AI Agent"

3. **Create Set Node (Rename Link Field)**  
   - Set field: `URL_PARSE` = `{{$json.message.link_preview_options.url}}`  
   - Connect output to "Code" node

4. **Create Function Node (Code for Web Scraping)**  
   - Write JavaScript to:  
     - HTTP GET request to `URL_PARSE` with User-Agent header  
     - Extract `<title>`, meta description, paragraphs (`<p>`), images (`<img src>`), Open Graph title/image  
     - Compose summary from first 3 paragraphs  
     - On error, set null or empty fields  
   - Connect output to "VLM Run Highlighter"

5. **Create LangChain Agent Node (VLM Run Highlighter)**  
   - Model: `vlm-agent-1`  
   - Input text: concatenation of extracted fields (source_url, title, description, text, images, summary)  
   - Prompt: instruct to highlight news, provide complete image URLs, obey JSON schema  
   - Enable output parser  
   - Connect output to "Covert to Text File" and "Check URLs Validity"

6. **Create LangChain Structured Output Parser Node**  
   - Configure JSON schema with fields: headline, source_url, published_date, key_points, summary, extracted_images_url  
   - Connect as AI output parser for "VLM Run Highlighter" output

7. **Create If Node (Check URLs Validity)**  
   - Condition: Check if `{{$json.output.news_summary.extracted_images_url}}` is not empty string  
   - True branch → "Split Out"  
   - False branch → "No Operation" (do nothing)

8. **Create Split Out Node**  
   - Field to split out: `output.research_paper_summary.extracted_images_url`  
   - Connect output to "HTTP Request1"

9. **Create HTTP Request Node (HTTP Request1)**  
   - URL: dynamic from split output items  
   - Operation: GET to download image binary data  
   - Connect output to "Send a document"

10. **Create Convert To File Node (Covert to Text File)**  
    - Convert `output` JSON to text file  
    - Connect output to "Send a document" and "Default Data Loader"

11. **Create Telegram Node (Send a document)**  
    - Operation: sendDocument  
    - Chat ID: 1872183963  
    - Attach binary data from image downloads and summary text file  
    - Connect output to "Start Asking"

12. **Create Telegram Node (Start Asking)**  
    - Text: "Start asking about provided link now"  
    - Chat ID: 1872183963  
    - Enable forced reply  
    - No output connection needed

13. **Create LangChain Document Default Data Loader**  
    - Data type: binary  
    - Input: text file from "Covert to Text File"  
    - Connect output to "Insert Data to Store"

14. **Create LangChain Embeddings Node (Embeddings OpenAI)**  
    - Credentials: OpenAI API key  
    - Input: document from Default Data Loader  
    - Connect output to both "Insert Data to Store" and "Query Data Tool"

15. **Create Vector Store In-Memory Node (Insert Data to Store)**  
    - Mode: Insert  
    - Memory key: "vector_store_key"  
    - Input: embedding vectors  
    - Connect output to "Query Data Tool"

16. **Create Vector Store In-Memory Node (Query Data Tool)**  
    - Mode: Retrieve-as-tool  
    - Tool name: "knowledge_base"  
    - Tool description: "Use this knowledge base to answer questions from the user"  
    - Memory key: "vector_store_key"  
    - Connect output to "AI Agent"

17. **Create LangChain Agent Node (AI Agent)**  
    - Text input: `{{$json.message.text}}` from Telegram messages without URLs  
    - Prompt type: define  
    - Enable output parser  
    - Connect output to "Send a Reply"  
    - Connect "Query Data Tool" as AI tool input

18. **Create LangChain OpenAI Chat Model Node**  
    - Model: GPT-4.1-nano  
    - Credentials: OpenAI API key  
    - Connect output as language model input for "AI Agent"

19. **Create Telegram Node (Send a Reply)**  
    - Text: `{{$json.output}}` (AI Agent response)  
    - Chat ID: 1872183963

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                      |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| 1. **Telegram Trigger Node** listens to incoming messages and captures text including possible URL previews. 2. **Link Detection** ensures only messages with URLs are processed for scraping; others are handled as chat queries. 3. **Scraping Node** uses custom JS to extract comprehensive article data from HTML. 4. **Output** stored as structured JSON for downstream AI summarization and Telegram sending.                                                                                                                                                                                                                                                                                                                                                                  | Sticky Note near Telegram Trigger and Code nodes                                                    |
| Embeddings Insert and Retrieve operations share the same OpenAI embeddings node to maintain embedding consistency and avoid errors caused by different embedding configurations.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Sticky Note near Embeddings OpenAI and Insert Data to Store nodes                                   |
| The workflow uses a Vision-Language Model (VLM) agent (vlm-agent-1) specialized for newspaper summarization, capable of extracting headlines, key points, and image URLs with schema validation.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Sticky Note near VLM Run Highlighter node                                                          |
| The workflow includes error handling such as fallback to AI chat agent when no URL is detected, and no-op when no image URLs are extracted. Telegram API credentials must be configured properly with chat permissions. OpenAI API keys require sufficient quota for embeddings and chat completions.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | General operational notes                                                                            |
| For advanced use, the vector store is in-memory, suitable for session or ephemeral storage. To persist embeddings longer term, consider integrating external vector DB.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Operational consideration                                                                            |

---

This document provides a detailed, modular understanding of the "Chat with News Articles using AI Analysis in Telegram with Vector Search" workflow, facilitating its reproduction, maintenance, and extension.