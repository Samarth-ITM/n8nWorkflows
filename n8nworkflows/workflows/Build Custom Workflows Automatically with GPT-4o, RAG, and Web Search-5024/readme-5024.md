Build Custom Workflows Automatically with GPT-4o, RAG, and Web Search

https://n8nworkflows.xyz/workflows/build-custom-workflows-automatically-with-gpt-4o--rag--and-web-search-5024


# Build Custom Workflows Automatically with GPT-4o, RAG, and Web Search

### 1. Workflow Overview

This workflow, titled **"Build Custom Workflows Automatically with GPT-4o, RAG, and Web Search"**, is designed to automatically generate custom n8n workflows based on user requests received via a chat interface. It leverages advanced AI models (OpenRouter GPT-4o), retrieval-augmented generation (RAG) with a Pinecone vector store for n8n documentation context, and web search capabilities via SerpAPI to build and validate comprehensive n8n workflow templates in JSON format.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Listens for user chat messages to trigger workflow generation.
- **1.2 AI Agent Processing:** Uses a LangChain AI agent integrating multiple AI tools (LLM, vector store, web search) to conceive and generate the workflow template.
- **1.3 Output Validation and Extraction:** Validates the generated JSON workflow, extracts it, and converts it into a downloadable file.
- **1.4 RAG Training (Supporting Sub-process):** Prepares and trains the Pinecone vector store with n8n documentation to enable relevant context retrieval.
- **1.5 Web Crawler (Supporting Sub-process):** Crawls and fetches n8n documentation from the web for use in RAG training.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception

**Overview:**  
This block triggers the workflow when a user sends a chat message to the system, initiating the workflow building process.

**Nodes Involved:**  
- When chat message received

**Node Details:**  

- **When chat message received**  
  - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
  - Role: Entry trigger that listens for incoming chat messages to start the workflow.  
  - Configuration: Uses default options; triggers on any chat message received.  
  - Input: External chat message via webhook.  
  - Output: Passes the message JSON to the AI Agent for processing.  
  - Edge Cases: Failure if webhook is unreachable; malformed message input.  
  - Version: 1.1  

---

#### 1.2 AI Agent Processing

**Overview:**  
This core block orchestrates AI-driven workflow generation by combining multiple AI models and tools, including GPT-4o (via OpenRouter), Pinecone vector store for documentation context, and SerpAPI for web search. It produces a valid n8n workflow JSON template.

**Nodes Involved:**  
- Set Preferences  
- AI Agent  
- OpenRouter Chat Model  
- SerpAPI  
- Pinecone Vector Store  
- Embeddings OpenAI  
- Embeddings OpenAI2  

**Node Details:**  

- **Set Preferences**  
  - Type: `n8n-nodes-base.set`  
  - Role: Initializes user preferences and configuration variables such as vector database type ("Pinecone"), chat model ("Open Router"), embedding model, and web search tool ("SerpAPI").  
  - Configuration: Sets specific strings for these preferences to be referenced in AI agent nodes.  
  - Input: Triggered from the chat message node.  
  - Output: Outputs preferences for downstream AI nodes.  
  - Edge Cases: None significant.  
  - Version: 3.4  

- **AI Agent**  
  - Type: `@n8n/n8n-nodes-langchain.agent`  
  - Role: Central orchestrator that receives user prompts and leverages AI models and tools to generate a full n8n workflow template in JSON.  
  - Configuration:  
    - System message instructs detailed stepwise reasoning, iterative tool usage, and ensures proper node connection.  
    - Uses LangChain to integrate multiple tools: Pinecone (vector store for RAG), SerpAPI (web search), OpenRouter GPT-4o model, and structured output parser.  
    - Prompt includes instructions to produce valid n8n JSON workflow templates based on user requests.  
    - Error handling set to continue with regular output on failure.  
  - Input: Receives chat messages and user preferences.  
  - Output: Produces workflow JSON as AI output.  
  - Edge Cases: LLM timeout, malformed JSON output, tool failures (e.g., web search API limits), connection issues with Pinecone.  
  - Version: 1.9  

- **OpenRouter Chat Model**  
  - Type: `@n8n/n8n-nodes-langchain.lmChatOpenRouter`  
  - Role: Provides the GPT-4o language model for the AI Agent to generate and reason.  
  - Configuration: Uses OpenRouter API with model `o4-mini-2025-04-16` by default.  
  - Input: Connected as AI language model to AI Agent.  
  - Output: Language model responses to AI Agent.  
  - Credentials: Requires OpenRouter API key.  
  - Edge Cases: API rate limits, invalid credentials, network failures.  
  - Version: 1  

- **SerpAPI**  
  - Type: `@n8n/n8n-nodes-langchain.toolSerpApi`  
  - Role: Enables AI Agent to perform web search queries to gather real-time information about n8n workflows and node documentation.  
  - Configuration: Default SerpAPI options.  
  - Input: Connected as AI tool to AI Agent.  
  - Output: Provides web search results to AI Agent.  
  - Credentials: Requires valid SerpAPI key.  
  - Edge Cases: API quota exceeded, invalid keys, network issues.  
  - Version: 1  

- **Pinecone Vector Store**  
  - Type: `@n8n/n8n-nodes-langchain.vectorStorePinecone`  
  - Role: Provides a retrieval-augmented generation capability by querying a Pinecone index containing n8n documentation to supply context to the AI Agent.  
  - Configuration:  
    - Mode: `retrieve-as-tool`  
    - Index: `agentbuilder` (contains n8n documentation vectors)  
    - Tool description explains its purpose as n8n documentation context.  
  - Input: Connected as AI tool to AI Agent.  
  - Output: Provides relevant vector search results to AI Agent.  
  - Credentials: Requires Pinecone API credentials (Environment, API Key, Index Name).  
  - Edge Cases: Index not available, API errors, empty search results.  
  - Version: 1.1  

- **Embeddings OpenAI & Embeddings OpenAI2**  
  - Type: `@n8n/n8n-nodes-langchain.embeddingsOpenAi`  
  - Role: Converts document chunks into embeddings for indexing or querying Pinecone.  
  - Configuration: Uses OpenAI embeddings model with default or specified embedding model (e.g., `text-embedding-3-large`).  
  - Input: Receives document chunks from data loaders or splitters.  
  - Output: Provides vector embeddings to Pinecone nodes.  
  - Credentials: Requires OpenAI API key.  
  - Edge Cases: API rate limits, credential issues.  
  - Version: 1.2  

---

#### 1.3 Output Validation and Extraction

**Overview:**  
This block validates the AI-generated JSON workflow, extracts the JSON content from the AI's textual response, converts it into a JSON file, and prepares it for download or further use.

**Nodes Involved:**  
- OpenAI (validator)  
- Code (JSON extraction)  
- Extract JSON (set node)  
- Convert to JSON File  
- Convert to File  
- Train Pinecone Vector Store  

**Node Details:**  

- **OpenAI (validator)**  
  - Type: `@n8n/n8n-nodes-langchain.openAi`  
  - Role: Validates the AI Agent's generated workflow JSON for correctness and pretty-prints errors if any.  
  - Configuration: Uses a prompt to transform input into valid n8n JSON template and ensure all nodes are properly connected.  
  - Input: AI Agent output JSON string.  
  - Output: Validated JSON text.  
  - Credentials: OpenAI API key.  
  - Edge Cases: API timeout, invalid JSON detection, malformed output.  
  - Version: 1.8  

- **Code (JSON extraction)**  
  - Type: `n8n-nodes-base.code`  
  - Role: Parses the AI textual response to extract the JSON workflow embedded inside triple backticks ```json ... ```.  
  - Configuration: JavaScript code that extracts JSON substring and parses it. Returns extracted JSON or null on failure.  
  - Input: OpenAI validator output text.  
  - Output: JSON object ready for conversion.  
  - Edge Cases: Parsing errors, missing or malformed JSON block.  
  - Version: 2  

- **Extract JSON (set node)**  
  - Type: `n8n-nodes-base.set`  
  - Role: Prepares the extracted JSON to be passed as proper JSON data in the workflow.  
  - Configuration: Sets output JSON to extracted JSON from previous node.  
  - Input: Code node output.  
  - Output: JSON object for file conversion.  
  - Version: 3.4  

- **Convert to JSON File**  
  - Type: `n8n-nodes-base.convertToFile`  
  - Role: Converts the extracted JSON into a JSON file format for download or sharing.  
  - Configuration: Converts each item to JSON text file with UTF-8 encoding.  
  - Input: Extracted JSON data.  
  - Output: Binary JSON file.  
  - Version: 1.1  

- **Convert to File**  
  - Type: `n8n-nodes-base.convertToFile`  
  - Role: Converts file to binary format if required, then passes it forward for storage or further processing.  
  - Configuration: Default options.  
  - Input: JSON file.  
  - Output: Binary file.  
  - Version: 1.1  

- **Train Pinecone Vector Store** (Repeated)  
  - Already described in RAG Trainer block, also triggered here to keep vector store updated after file conversion.  

---

#### 1.4 RAG Training (Supporting Sub-process)

**Overview:**  
This supporting sub-process prepares the vector store by loading crawled documentation, splitting it into chunks, embedding, and upserting into Pinecone. This ensures the AI agent has up-to-date knowledge base when generating workflows.

**Nodes Involved:**  
- Default Data Loader1  
- Recursive Character Text Splitter  
- Embeddings OpenAI2  
- Train Pinecone Vector Store  

**Node Details:**  

- **Default Data Loader1**  
  - Type: `@n8n/n8n-nodes-langchain.documentDefaultDataLoader`  
  - Role: Loads raw crawled documentation content.  
  - Configuration: Default options.  
  - Input: Raw markdown/HTML from Web Crawler.  
  - Output: Documents for splitting.  
  - Version: 1  

- **Recursive Character Text Splitter**  
  - Type: `@n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter`  
  - Role: Splits documents into chunks roughly 1-2k tokens each.  
  - Configuration: Default splitter options.  
  - Input: Document loader output.  
  - Output: Text chunks for embedding.  
  - Version: 1  

- **Embeddings OpenAI2**  
  - As above. Converts chunks to vectors.  

- **Train Pinecone Vector Store**  
  - Upserts embeddings into Pinecone index.  

---

#### 1.5 Web Crawler (Supporting Sub-process)

**Overview:**  
This sub-process fetches n8n documentation from the web using Firecrawl API, providing fresh content for use in RAG training.

**Nodes Involved:**  
- Set URL  
- Extract (HTTP Request)  
- 30 Secs (Wait)  
- Get Results (HTTP Request)  
- If (Conditional check)  
- 10 Seconds (Wait)  

**Node Details:**  

- **Set URL**  
  - Type: `n8n-nodes-base.set`  
  - Role: Sets the Firecrawl API URL for crawling n8n docs.  
  - Configuration: URL hardcoded to a Firecrawl crawl job.  
  - Output: URL to Extract node.  
  - Version: 3.4  

- **Extract**  
  - Type: `n8n-nodes-base.httpRequest`  
  - Role: Calls Firecrawl API to start crawling.  
  - Configuration: URL from Set URL node; uses Firecrawl API key via generic header auth.  
  - Output: Crawl job initiation response.  
  - Version: 4.2  

- **30 Secs**  
  - Type: `n8n-nodes-base.wait`  
  - Role: Waits 30 seconds before polling results.  
  - Version: 1.1  

- **Get Results**  
  - Type: `n8n-nodes-base.httpRequest`  
  - Role: Polls Firecrawl API to retrieve crawl results.  
  - Configuration: Uses the same crawl job ID.  
  - Output: Raw markdown/HTML content.  
  - Version: 4.2  

- **If**  
  - Type: `n8n-nodes-base.if`  
  - Role: Checks if crawl is complete or needs retry.  
  - Output: If incomplete, triggers 10 Seconds wait and retries Get Results; else proceeds.  
  - Version: 2.2  

- **10 Seconds**  
  - Type: `n8n-nodes-base.wait`  
  - Role: Retry interval before polling again.  
  - Version: 1.1  

---

### 3. Summary Table

| Node Name                     | Node Type                                      | Functional Role                      | Input Node(s)                  | Output Node(s)                        | Sticky Note                                                                                                        |
|-------------------------------|------------------------------------------------|------------------------------------|-------------------------------|-------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| When chat message received     | `@n8n/n8n-nodes-langchain.chatTrigger`         | Workflow trigger on chat input     | -                             | AI Agent                            | Part of AI Agent Builder logic                                                                                     |
| Set Preferences               | `n8n-nodes-base.set`                            | Set user preferences/configuration | When chat message received     | AI Agent                           | Part of AI Agent Builder logic                                                                                     |
| AI Agent                     | `@n8n/n8n-nodes-langchain.agent`               | Main AI orchestrator                | When chat message received, Set Preferences | OpenAI (validator)                | Integrates GPT-4o, SerpAPI, Pinecone, structured output parser                                                    |
| OpenRouter Chat Model        | `@n8n/n8n-nodes-langchain.lmChatOpenRouter`    | LLM GPT-4o model                   | AI Agent                      | AI Agent                          | Requires OpenRouter API key                                                                                        |
| SerpAPI                     | `@n8n/n8n-nodes-langchain.toolSerpApi`         | Web search tool                    | AI Agent                      | AI Agent                          | Requires SerpAPI key                                                                                               |
| Pinecone Vector Store        | `@n8n/n8n-nodes-langchain.vectorStorePinecone` | Vector DB for RAG context          | AI Agent                      | AI Agent                          | Requires Pinecone API key                                                                                          |
| Embeddings OpenAI            | `@n8n/n8n-nodes-langchain.embeddingsOpenAi`    | Create embeddings for Pinecone     | Default Data Loader1, Recursive Splitter | Pinecone Vector Store             | Requires OpenAI API key                                                                                            |
| Embeddings OpenAI2           | `@n8n/n8n-nodes-langchain.embeddingsOpenAi`    | Create embeddings for training     | Default Data Loader1, Recursive Splitter | Train Pinecone Vector Store       | Requires OpenAI API key                                                                                            |
| OpenAI (validator)           | `@n8n/n8n-nodes-langchain.openAi`               | Validate AI JSON output            | AI Agent                      | Code                             | Requires OpenAI API key                                                                                            |
| Code                        | `n8n-nodes-base.code`                            | Parse and extract JSON from text   | OpenAI (validator)            | Extract JSON                     | Custom JS code to parse JSON block                                                                                 |
| Extract JSON                | `n8n-nodes-base.set`                             | Set extracted JSON for conversion  | Code                         | Convert to JSON File             |                                                                                                                    |
| Convert to JSON File        | `n8n-nodes-base.convertToFile`                   | Convert JSON to file format         | Extract JSON                 | Convert to File                 |                                                                                                                    |
| Convert to File             | `n8n-nodes-base.convertToFile`                   | Final file conversion               | Convert to JSON File          | Train Pinecone Vector Store       |                                                                                                                    |
| Train Pinecone Vector Store | `@n8n/n8n-nodes-langchain.vectorStorePinecone`  | Update vector database              | Convert to File, Embeddings   | -                               | Requires Pinecone API key                                                                                          |
| Default Data Loader1        | `@n8n/n8n-nodes-langchain.documentDefaultDataLoader` | Load crawled docs                  | Recursive Character Text Splitter | Embeddings OpenAI2              |                                                                                                                    |
| Recursive Character Text Splitter | `@n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter` | Split documents into chunks         | Default Data Loader1          | Embeddings OpenAI, Embeddings OpenAI2 |                                                                                                                    |
| Set URL                     | `n8n-nodes-base.set`                             | Set Firecrawl API URL               | Manual Trigger (test)         | Extract                        | Part of Web Crawler sub-process                                                                                   |
| Extract                     | `n8n-nodes-base.httpRequest`                     | Call Firecrawl API to start crawl  | Set URL                      | 30 Secs                       | Requires Firecrawl API key                                                                                        |
| 30 Secs                     | `n8n-nodes-base.wait`                            | Wait 30 seconds                    | Extract                      | Get Results                   |                                                                                                                    |
| Get Results                 | `n8n-nodes-base.httpRequest`                     | Poll Firecrawl for crawl results   | 30 Secs, 10 Seconds           | If                           |                                                                                                                    |
| If                         | `n8n-nodes-base.if`                              | Check crawl status and retry logic | Get Results                  | 10 Seconds, Convert to File    |                                                                                                                    |
| 10 Seconds                  | `n8n-nodes-base.wait`                            | Retry wait                        | If                           | Get Results                   |                                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node**  
   - Add node: `When chat message received` (`@n8n/n8n-nodes-langchain.chatTrigger`)  
   - Purpose: Listen for incoming user chat messages to start workflow generation.  
   - Use default parameters; no credentials needed.

2. **Set User Preferences**  
   - Add node: `Set Preferences` (`n8n-nodes-base.set`)  
   - Configure to assign the following values:  
     - `vector database` = `"Pinecone"`  
     - `chat model` = `"Open Router"`  
     - `embedding` = `"text-embedding-3-large"`  
     - `web search tool` = `"SerpAPI"`  
   - Connect `When chat message received` → `Set Preferences`.

3. **Add AI Agent Node**  
   - Add node: `AI Agent` (`@n8n/n8n-nodes-langchain.agent`)  
   - Configure:  
     - Text: `=Respond to the user’s request given in the trigger item.`  
     - System message: Detailed instructions for generating valid n8n JSON workflows, iterative tool usage, and ensuring node connections.  
     - Enable tools:  
       - Pinecone Vector Store (retrieval)  
       - SerpAPI (web search)  
       - OpenRouter GPT-4o model  
       - Structured Output Parser for JSON validation  
     - Set `onError` to "continueRegularOutput".  
   - Connect `Set Preferences` → `AI Agent`.  

4. **Add OpenRouter Chat Model Node**  
   - Add node: `OpenRouter Chat Model` (`@n8n/n8n-nodes-langchain.lmChatOpenRouter`)  
   - Select model: `o4-mini-2025-04-16` or appropriate.  
   - Add OpenRouter API credential.  
   - Connect as AI languageModel input to `AI Agent`.

5. **Add SerpAPI Node**  
   - Add node: `SerpAPI` (`@n8n/n8n-nodes-langchain.toolSerpApi`)  
   - Add SerpAPI credential.  
   - Connect as AI tool input to `AI Agent`.

6. **Add Pinecone Vector Store Node**  
   - Add node: `Pinecone Vector Store` (`@n8n/n8n-nodes-langchain.vectorStorePinecone`)  
   - Mode: `retrieve-as-tool`  
   - Set Pinecone index name (e.g., `agentbuilder`).  
   - Add Pinecone API credential (environment, API key, index).  
   - Connect as AI tool input to `AI Agent`.

7. **Add OpenAI Embeddings Nodes**  
   - Add nodes: `Embeddings OpenAI` and `Embeddings OpenAI2` (`@n8n/n8n-nodes-langchain.embeddingsOpenAi`)  
   - Use embedding model `text-embedding-3-large`.  
   - Connect appropriately in training sub-process (see below).  
   - Add OpenAI API credentials.

8. **Add OpenAI Validator Node**  
   - Add node: `OpenAI` (`@n8n/n8n-nodes-langchain.openAi`)  
   - Use prompt to validate and pretty-print AI Agent JSON output.  
   - Connect `AI Agent` output → `OpenAI`.

9. **Add Code Node for JSON Extraction**  
   - Add node: `Code` (`n8n-nodes-base.code`)  
   - Paste JavaScript code to extract JSON block from AI response text.  
   - Connect `OpenAI` output → `Code`.

10. **Add Set Node to Extract JSON**  
    - Add node: `Extract JSON` (`n8n-nodes-base.set`)  
    - Set JSON property to extracted JSON from Code node output.  
    - Connect `Code` → `Extract JSON`.

11. **Add Convert to JSON File Node**  
    - Add node: `Convert to JSON File` (`n8n-nodes-base.convertToFile`)  
    - Convert extracted JSON to UTF-8 encoded JSON file.  
    - Connect `Extract JSON` → `Convert to JSON File`.

12. **Add Convert to File Node**  
    - Add node: `Convert to File` (`n8n-nodes-base.convertToFile`)  
    - Default parameters.  
    - Connect `Convert to JSON File` → `Convert to File`.

13. **Add Train Pinecone Vector Store Node**  
    - Add node: `Train Pinecone Vector Store` (`@n8n/n8n-nodes-langchain.vectorStorePinecone`)  
    - Mode: `insert` to upsert embeddings.  
    - Connect `Convert to File` → `Train Pinecone Vector Store`.  

14. **Sub-Workflow: Web Crawler Setup**  
    - Add nodes:  
      - `Set URL` (`n8n-nodes-base.set`): Set Firecrawl API URL for crawling n8n docs.  
      - `Extract` (`n8n-nodes-base.httpRequest`): Start Firecrawl job.  
      - `30 Secs` (`n8n-nodes-base.wait`): Wait before polling.  
      - `Get Results` (`n8n-nodes-base.httpRequest`): Poll Firecrawl results.  
      - `If` (`n8n-nodes-base.if`): Check completion, retry logic.  
      - `10 Seconds` (`n8n-nodes-base.wait`): Retry delay.  
    - Configure Firecrawl API credentials (Generic Header Auth with Authorization Bearer token).  
    - Connect nodes in order: Set URL → Extract → 30 Secs → Get Results → If → (If incomplete: 10 Seconds → Get Results) else proceed.

15. **Sub-Workflow: RAG Trainer Setup**  
    - Add nodes:  
      - `Default Data Loader1` (`@n8n/n8n-nodes-langchain.documentDefaultDataLoader`)  
      - `Recursive Character Text Splitter` (`@n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter`)  
      - `Embeddings OpenAI2` (`@n8n/n8n-nodes-langchain.embeddingsOpenAi`)  
      - `Train Pinecone Vector Store` (`@n8n/n8n-nodes-langchain.vectorStorePinecone`)  
    - Connect in order: Default Data Loader1 → Recursive Splitter → Embeddings OpenAI2 → Train Pinecone Vector Store.  
    - Use this sub-workflow to update vector index with fresh crawled docs.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                           | Context or Link                                                                                   |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| 🕸️ **Web Crawler (Sub-process #1)**: Triggered manually; uses Firecrawl API to scrape n8n docs. See configuration for Firecrawl API key setup.                                                                                                       | Sticky Note "Web Crawler Logic"                                                                   |
| 🧠 **RAG Trainer (Sub-process #2)**: Loads crawled docs, splits into chunks, creates embeddings with OpenAI, and inserts into Pinecone. Run once or on schedule when docs update.                                                                      | Sticky Note "RAG Trainer Logic"                                                                   |
| 🤖 **AI Agent Builder (Sub-process #3)**: Waits for chat input, uses OpenRouter GPT-4o, SerpAPI, Pinecone vector store, and structured output parser to generate fully wired n8n workflow JSON templates. Validated by OpenAI, then converted to file.    | Sticky Note "Agent Builder Logic"                                                                 |
| ⚙️ **Firecrawl API**: Requires generic credential with header auth "Authorization: Bearer YOUR_FIRECRAWL_KEY". Used in Extract and Get Results HTTP nodes.                                                                                                | Sticky Note "Configure Firecrawl"                                                                 |
| ⚙️ **Pinecone Vector Store**: Requires Pinecone API credential with Environment, API Key, and Index Name. Make sure same index used across nodes.                                                                                                     | Sticky Notes "Configure Pinecone" and "Configure Pinecone1"                                      |
| ⚙️ **OpenAI / Azure OpenAI**: Requires OpenAI API credential for embeddings and validation nodes. Ensure embedding model availability (e.g., `text-embedding-3-small`).                                                                                 | Sticky Notes "Configure OpenAI API" and "Configure OpenAI API1"                                  |
| ⚙️ **OpenRouter API**: Requires OpenRouter API Key credential. Used in OpenRouter Chat Model node with model `o4-mini-2025-04-16`.                                                                                                                    | Sticky Note "Configure OpenRouter GPT-4o"                                                        |
| ⚙️ **SerpAPI Key**: Requires SerpAPI credential for web search node.                                                                                                                                                                                  | Sticky Note "Configure SerpAPI"                                                                   |
| The workflow relies heavily on proper API key management and rate limits for all external services (OpenAI, OpenRouter, SerpAPI, Pinecone, Firecrawl).                                                                                               | -                                                                                               |
| The AI Agent prompt emphasizes iterative reasoning, tool usage, validation, and output formatting to ensure the generated workflows are valid and fully connected.                                                                                   | System message in AI Agent node                                                                   |
| Example user prompt includes detailed instructions on building workflows, consulting tools (vector DB and web search), and providing valid JSON output.                                                                                              | AI Agent system message                                                                           |
| The workflow is designed to be extensible and modifiable: replacing the AI model, vector store, or search tool can be done by changing node parameters and credentials accordingly.                                                                   | -                                                                                               |
| The workflow contains multiple sticky notes guiding setup and usage, including links to credential setup pages and external documentation for the integrated services.                                                                                | Multiple sticky notes in the workflow                                                            |

---

**Disclaimer:** The provided text and workflow originate exclusively from an n8n automated workflow. It strictly adheres to content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.