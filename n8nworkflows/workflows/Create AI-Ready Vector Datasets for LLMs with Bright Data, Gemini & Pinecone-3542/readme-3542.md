Create AI-Ready Vector Datasets for LLMs with Bright Data, Gemini & Pinecone

https://n8nworkflows.xyz/workflows/create-ai-ready-vector-datasets-for-llms-with-bright-data--gemini---pinecone-3542


# Create AI-Ready Vector Datasets for LLMs with Bright Data, Gemini & Pinecone

### 1. Workflow Overview

This workflow automates the end-to-end process of collecting, cleaning, formatting, and vectorizing web data to create AI-ready datasets for training or fine-tuning Large Language Models (LLMs). It is designed for ML engineers, AI startups, data teams, and LLM-as-a-Service providers who need scalable, high-quality, structured web content.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Initialization**: Manual trigger and setting of target URLs and webhook URLs.
- **1.2 Web Crawling with Bright Data Web Unlocker**: Sending HTTP requests to Bright Data’s Web Unlocker API to scrape raw web content while bypassing anti-bot measures.
- **1.3 Data Extraction and Formatting with AI Agents**: Using Google Gemini chat models and LangChain AI agents to extract structured information from raw HTML/text and format it according to predefined JSON schemas.
- **1.4 Vectorization and Persistence in Pinecone**: Splitting text, embedding it with Google Gemini embeddings, and storing vectors in Pinecone vector database for semantic search.
- **1.5 Webhook Notification Handling**: Sending structured data and AI agent responses to specified webhook URLs for downstream processing or monitoring.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Initialization

- **Overview:**  
  This block starts the workflow manually and sets the target URL for web scraping and the webhook URL for notifications.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’  
  - Set Fields - URL and Webhook URL  
  - Sticky Note (general note about URL setting)

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point to manually start the workflow for testing or execution.  
    - Configuration: No parameters; triggers downstream nodes on manual activation.  
    - Inputs: None  
    - Outputs: Connected to "Set Fields - URL and Webhook URL"  
    - Edge cases: None, but manual trigger requires user interaction.

  - **Set Fields - URL and Webhook URL**  
    - Type: Set  
    - Role: Defines the target URL to scrape and the webhook URL for notifications.  
    - Configuration:  
      - `url` set to `"https://news.ycombinator.com?product=unlocker&method=api"` (default example)  
      - `webhook_url` set to `"https://webhook.site/bc804ce5-4a45-4177-a68a-99c80e5c86e6"` (example webhook)  
    - Inputs: Manual Trigger node  
    - Outputs:  
      - To "Make a web request" (web scraping)  
      - To webhook notification nodes (for initial notification or fallback)  
    - Edge cases: URLs must be updated by user to target desired sites and webhook endpoints.

  - **Sticky Note**  
    - Type: Sticky Note  
    - Role: Provides user instructions to set the URL and explains usage of Web Unlocker and AI Agents.  
    - Content: Reminder to set URL, usage of Bright Data Web Unlocker, and AI Agents for formatting and persistence.

---

#### 2.2 Web Crawling with Bright Data Web Unlocker

- **Overview:**  
  This block sends a POST request to Bright Data’s Web Unlocker API to retrieve raw HTML or data from the target URL, bypassing anti-bot protections.

- **Nodes Involved:**  
  - Make a web request

- **Node Details:**

  - **Make a web request**  
    - Type: HTTP Request  
    - Role: Calls Bright Data Web Unlocker API to scrape the target URL.  
    - Configuration:  
      - URL: `https://api.brightdata.com/request`  
      - Method: POST  
      - Body parameters:  
        - `zone`: `"web_unlocker1"` (configured zone for Web Unlocker)  
        - `url`: Expression referencing `url` from previous node (`={{ $json.url }}`)  
        - `format`: `"raw"` (request raw data)  
      - Authentication: Generic Header Auth (configured with Bright Data API key)  
    - Inputs: From "Set Fields - URL and Webhook URL"  
    - Outputs: To "Structured JSON Data Formatter" and "Information Extractor with Data Formatter"  
    - Edge cases:  
      - API key or zone misconfiguration leads to auth errors.  
      - Network timeouts or rate limits from Bright Data.  
      - Invalid URLs or blocked content may return empty or error responses.

---

#### 2.3 Data Extraction and Formatting with AI Agents

- **Overview:**  
  This block uses Google Gemini chat models and LangChain AI agents to parse raw scraped data, extract relevant structured information, and format it into JSON schemas suitable for downstream processing.

- **Nodes Involved:**  
  - Structured JSON Data Formatter  
  - Information Extractor with Data Formatter  
  - AI Agent  
  - Google Gemini Chat Model  
  - Google Gemini Chat Model1  
  - Google Gemini Chat Model2  
  - Structured Output Parser  
  - Sticky Notes (AI Data Formatter, Data Extraction/Formatting with AI Agent)

- **Node Details:**

  - **Structured JSON Data Formatter**  
    - Type: Chain LLM  
    - Role: Formats raw web response into a structured JSON array with fields like rank, title, site, points, user, age, comments.  
    - Configuration:  
      - Prompt includes example JSON schema for formatting.  
      - Uses Google Gemini Chat Model2 as language model.  
    - Inputs: From "Make a web request"  
    - Outputs: To "Webhook for structured data" and "Information Extractor with Data Formatter"  
    - Edge cases:  
      - Formatting errors if input data is malformed.  
      - Model API limits or latency.

  - **Information Extractor with Data Formatter**  
    - Type: Information Extractor (LangChain)  
    - Role: Extracts key content from the search result using AI, producing a collection of items.  
    - Configuration:  
      - System prompt: Expert HTML extractor analyzing search results.  
      - Attribute: `search_result` containing the raw data.  
      - Uses Google Gemini Chat Model as language model.  
    - Inputs: From "Structured JSON Data Formatter" and "Make a web request"  
    - Outputs: To "AI Agent"  
    - Edge cases:  
      - Extraction failures if HTML is malformed or unexpected.  
      - Model API errors.

  - **AI Agent**  
    - Type: LangChain Agent  
    - Role: Further formats the extracted search result text into a clean, textual data output.  
    - Configuration:  
      - Prompt: "Format the below search result {{ $json.output.search_result }}"  
      - Uses Google Gemini Chat Model1 as language model.  
      - Output parser: Structured Output Parser.  
    - Inputs: From "Information Extractor with Data Formatter"  
    - Outputs: To "Pinecone Vector Store" and "Webhook for structured AI agent response"  
    - Edge cases:  
      - Parsing errors if output does not match expected schema.  
      - Model API errors or timeouts.

  - **Google Gemini Chat Models (Google Gemini Chat Model, Chat Model1, Chat Model2)**  
    - Type: Language Model (Google Gemini via PaLM API)  
    - Role: Provide LLM capabilities for extraction, formatting, and agent processing.  
    - Configuration:  
      - Model name: `"models/gemini-2.0-flash-exp"`  
      - Credentials: Google Gemini (PaLM) API key  
    - Inputs/Outputs: Connected to respective AI nodes above.  
    - Edge cases: API key limits, latency, or quota issues.

  - **Structured Output Parser**  
    - Type: Output Parser  
    - Role: Parses AI Agent output into structured JSON format based on example schema.  
    - Configuration: JSON schema example with fields id, title, summary, keywords, topics.  
    - Inputs: From AI Agent  
    - Outputs: To AI Agent node (feedback loop)  
    - Edge cases: Parsing failures if AI output is malformed.

  - **Sticky Notes (AI Data Formatter, Data Extraction/Formatting with AI Agent)**  
    - Provide contextual information about the AI data formatting and extraction process.

---

#### 2.4 Vectorization and Persistence in Pinecone

- **Overview:**  
  This block splits the formatted textual data into chunks, generates embeddings using Google Gemini embeddings model, and inserts the vectors into a Pinecone vector database index for semantic search and retrieval.

- **Nodes Involved:**  
  - Recursive Character Text Splitter  
  - Default Data Loader  
  - Embeddings Google Gemini  
  - Pinecone Vector Store  
  - Sticky Note (Vector Database Persistence)

- **Node Details:**

  - **Recursive Character Text Splitter**  
    - Type: Text Splitter  
    - Role: Splits large text documents into smaller chunks recursively for better embedding.  
    - Configuration: Default options (recursive character splitting).  
    - Inputs: From "Default Data Loader"  
    - Outputs: To "Default Data Loader"  
    - Edge cases: Improper splitting may cause context loss or overlap.

  - **Default Data Loader**  
    - Type: Document Data Loader  
    - Role: Loads JSON data (from AI extraction) into document format for embedding.  
    - Configuration:  
      - JSON data expression: `={{ $('Information Extractor with Data Formatter').item.json.output.search_result }}`  
      - JSON mode: expressionData  
    - Inputs: From "Recursive Character Text Splitter"  
    - Outputs: To "Pinecone Vector Store"  
    - Edge cases: Invalid JSON or empty data causes failure.

  - **Embeddings Google Gemini**  
    - Type: Embeddings Generator  
    - Role: Generates vector embeddings for text chunks using Google Gemini embedding model.  
    - Configuration:  
      - Model name: `"models/text-embedding-004"`  
      - Credentials: Google Gemini (PaLM) API key  
    - Inputs: From "Default Data Loader"  
    - Outputs: To "Pinecone Vector Store"  
    - Edge cases: API limits, embedding failures.

  - **Pinecone Vector Store**  
    - Type: Vector Store (Pinecone)  
    - Role: Inserts generated embeddings into Pinecone index named `"hacker-news"`.  
    - Configuration:  
      - Mode: Insert  
      - Pinecone index: `"hacker-news"`  
      - Credentials: Pinecone API key  
    - Inputs: From "AI Agent" (main), "Default Data Loader" (ai_document), and "Embeddings Google Gemini" (ai_embedding)  
    - Outputs: None (terminal for vector persistence)  
    - Edge cases: API key errors, index misconfiguration, network issues.

  - **Sticky Note (Vector Database Persistence)**  
    - Provides context about vector storage and persistence.

---

#### 2.5 Webhook Notification Handling

- **Overview:**  
  This block sends the structured data and AI agent responses to configured webhook URLs for external notification, logging, or further processing.

- **Nodes Involved:**  
  - Webhook for structured data  
  - Webhook for structured AI agent response  
  - Sticky Note (Webhook Notification Handler)

- **Node Details:**

  - **Webhook for structured data**  
    - Type: HTTP Request  
    - Role: Sends the formatted JSON data response to the webhook URL.  
    - Configuration:  
      - URL: Expression referencing `webhook_url` from earlier node (`={{ $json.webhook_url }}`)  
      - Method: POST  
      - Body parameter: `response` set to `={{ $json.text }}` (formatted JSON text)  
    - Inputs: From "Structured JSON Data Formatter" and "Set Fields - URL and Webhook URL"  
    - Outputs: None  
    - Edge cases: Webhook URL invalid or unreachable, network errors.

  - **Webhook for structured AI agent response**  
    - Type: HTTP Request  
    - Role: Sends AI agent’s formatted output to the webhook URL.  
    - Configuration:  
      - URL: Expression referencing `webhook_url`  
      - Method: POST  
      - Body parameter: `response` set to `={{ $json.output }}` (AI agent output)  
    - Inputs: From "Pinecone Vector Store" (after AI Agent processing) and "Set Fields - URL and Webhook URL"  
    - Outputs: None  
    - Edge cases: Same as above.

  - **Sticky Note (Webhook Notification Handler)**  
    - Describes the webhook notification functionality.

---

### 3. Summary Table

| Node Name                          | Node Type                                   | Functional Role                                   | Input Node(s)                          | Output Node(s)                                  | Sticky Note                                                                                   |
|-----------------------------------|---------------------------------------------|--------------------------------------------------|--------------------------------------|------------------------------------------------|-----------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’      | Manual Trigger                              | Manual start of workflow                          | None                                 | Set Fields - URL and Webhook URL                |                                                                                               |
| Set Fields - URL and Webhook URL   | Set                                         | Sets target URL and webhook URL                   | When clicking ‘Test workflow’         | Make a web request, Webhook for structured data, Webhook for structured AI agent response      | Set the URL which you are interested to scrap the data                                        |
| Make a web request                 | HTTP Request                                | Calls Bright Data Web Unlocker API                | Set Fields - URL and Webhook URL      | Structured JSON Data Formatter, Information Extractor with Data Formatter                      |                                                                                               |
| Structured JSON Data Formatter     | Chain LLM                                   | Formats raw response into structured JSON         | Make a web request                   | Webhook for structured data, Information Extractor with Data Formatter                         | AI Data Formatter                                                                            |
| Information Extractor with Data Formatter | Information Extractor (LangChain)          | Extracts key content from search results          | Structured JSON Data Formatter, Make a web request | AI Agent                                         | Data Extraction/Formatting with the AI Agent                                                |
| AI Agent                          | LangChain Agent                             | Formats extracted data into clean textual output  | Information Extractor with Data Formatter | Pinecone Vector Store, Webhook for structured AI agent response                               |                                                                                               |
| Pinecone Vector Store             | Vector Store (Pinecone)                     | Inserts embeddings into Pinecone index             | AI Agent, Default Data Loader, Embeddings Google Gemini | None                                           | Vector Database Persistence                                                                 |
| Embeddings Google Gemini          | Embeddings Generator                        | Generates vector embeddings using Google Gemini   | Default Data Loader                  | Pinecone Vector Store                            |                                                                                               |
| Default Data Loader               | Document Data Loader                        | Loads JSON data for embedding                       | Recursive Character Text Splitter    | Pinecone Vector Store                            |                                                                                               |
| Recursive Character Text Splitter | Text Splitter                              | Splits text into chunks for embedding              | Default Data Loader                  | Default Data Loader                              |                                                                                               |
| Google Gemini Chat Model          | Language Model (Google Gemini)              | Provides LLM capabilities for extraction           | None (used by Information Extractor) | Information Extractor with Data Formatter        |                                                                                               |
| Google Gemini Chat Model1         | Language Model (Google Gemini)              | Provides LLM capabilities for AI Agent             | None (used by AI Agent)              | AI Agent                                         |                                                                                               |
| Google Gemini Chat Model2         | Language Model (Google Gemini)              | Provides LLM capabilities for JSON formatting      | None (used by Structured JSON Data Formatter) | Structured JSON Data Formatter                   |                                                                                               |
| Structured Output Parser          | Output Parser                              | Parses AI Agent output into structured JSON        | AI Agent                           | AI Agent                                         |                                                                                               |
| Webhook for structured data       | HTTP Request                              | Sends formatted JSON data to webhook URL           | Structured JSON Data Formatter, Set Fields - URL and Webhook URL | None                                           | Webhook Notification Handler                                                               |
| Webhook for structured AI agent response | HTTP Request                              | Sends AI agent response to webhook URL             | Pinecone Vector Store, Set Fields - URL and Webhook URL | None                                           |                                                                                               |
| Sticky Note                      | Sticky Note                                | Instructional note about URL and Web Unlocker usage | None                                 | None                                           | Please make sure to set the URL for web crawling. Web-Unlocker Product is being utilized for performing the web scrapping. This workflow is utilizing the Basic LLM Chain, Information Extraction with the AI Agents for formatting, extracting and persisting the response in PineCone Vector Database |
| Sticky Note1                     | Sticky Note                                | AI Data Formatter note                              | None                                 | None                                           | AI Data Formatter                                                                           |
| Sticky Note2                     | Sticky Note                                | Vector Database Persistence note                    | None                                 | None                                           | Vector Database Persistence                                                                |
| Sticky Note3                     | Sticky Note                                | Webhook Notification Handler note                   | None                                 | None                                           | Webhook Notification Handler                                                              |
| Sticky Note4                     | Sticky Note                                | Data Extraction/Formatting with AI Agent note      | None                                 | None                                           | Data Extraction/Formatting with the AI Agent                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To start the workflow manually.

2. **Create Set Node: "Set Fields - URL and Webhook URL"**  
   - Type: Set  
   - Parameters:  
     - `url`: Set to your target scraping URL (e.g., `https://news.ycombinator.com?product=unlocker&method=api`)  
     - `webhook_url`: Set to your webhook endpoint URL (e.g., `https://webhook.site/...`)  
   - Connect Manual Trigger → Set Node.

3. **Create HTTP Request Node: "Make a web request"**  
   - Type: HTTP Request  
   - Parameters:  
     - URL: `https://api.brightdata.com/request`  
     - Method: POST  
     - Body Parameters:  
       - `zone`: `"web_unlocker1"` (your Bright Data Web Unlocker zone)  
       - `url`: Expression referencing `url` from Set Node (`={{ $json.url }}`)  
       - `format`: `"raw"`  
     - Authentication: Generic Header Auth with Bright Data API key configured in credentials.  
   - Connect Set Node → HTTP Request Node.

4. **Create Chain LLM Node: "Structured JSON Data Formatter"**  
   - Type: Chain LLM  
   - Parameters:  
     - Text prompt: Format the input data into JSON schema with fields rank, title, site, points, user, age, comments.  
     - Use Google Gemini Chat Model2 as the language model.  
   - Connect HTTP Request → Chain LLM Node.

5. **Create Information Extractor Node: "Information Extractor with Data Formatter"**  
   - Type: Information Extractor (LangChain)  
   - Parameters:  
     - Input text: Expression referencing raw data (`={{ $json.data }}`)  
     - System prompt: Expert HTML extractor analyzing search results.  
     - Attribute: `search_result`  
     - Use Google Gemini Chat Model as language model.  
   - Connect Chain LLM Node → Information Extractor Node.  
   - Also connect HTTP Request → Information Extractor Node (parallel input).

6. **Create LangChain Agent Node: "AI Agent"**  
   - Type: LangChain Agent  
   - Parameters:  
     - Text: `"Format the below search result\n\n{{ $json.output.search_result }}"`  
     - Use Google Gemini Chat Model1 as language model.  
     - Output parser: Structured Output Parser (create next).  
   - Connect Information Extractor → AI Agent.

7. **Create Structured Output Parser Node**  
   - Type: Output Parser  
   - Parameters: JSON schema example with fields id, title, summary, keywords, topics.  
   - Connect Structured Output Parser → AI Agent (ai_outputParser input).

8. **Create Recursive Character Text Splitter Node**  
   - Type: Text Splitter  
   - Parameters: Default recursive character splitting.  
   - Connect to Default Data Loader (next step).

9. **Create Default Data Loader Node**  
   - Type: Document Data Loader  
   - Parameters:  
     - JSON data: Expression referencing `search_result` from Information Extractor (`={{ $('Information Extractor with Data Formatter').item.json.output.search_result }}`)  
     - JSON mode: expressionData  
   - Connect Recursive Character Text Splitter → Default Data Loader.

10. **Create Embeddings Node: "Embeddings Google Gemini"**  
    - Type: Embeddings Generator  
    - Parameters:  
      - Model name: `"models/text-embedding-004"`  
      - Credentials: Google Gemini API key.  
    - Connect Default Data Loader → Embeddings Node.

11. **Create Pinecone Vector Store Node**  
    - Type: Vector Store (Pinecone)  
    - Parameters:  
      - Mode: Insert  
      - Pinecone index: `"hacker-news"` (or your own index)  
      - Credentials: Pinecone API key.  
    - Connect AI Agent → Pinecone Vector Store (main input).  
    - Connect Default Data Loader → Pinecone Vector Store (ai_document input).  
    - Connect Embeddings Google Gemini → Pinecone Vector Store (ai_embedding input).

12. **Create HTTP Request Node: "Webhook for structured data"**  
    - Type: HTTP Request  
    - Parameters:  
      - URL: Expression referencing `webhook_url` (`={{ $json.webhook_url }}`)  
      - Method: POST  
      - Body parameter: `response` set to `={{ $json.text }}` (formatted JSON text)  
    - Connect Structured JSON Data Formatter → Webhook node.  
    - Connect Set Fields node → Webhook node (for initial data).

13. **Create HTTP Request Node: "Webhook for structured AI agent response"**  
    - Type: HTTP Request  
    - Parameters:  
      - URL: Expression referencing `webhook_url`  
      - Method: POST  
      - Body parameter: `response` set to `={{ $json.output }}` (AI agent output)  
    - Connect Pinecone Vector Store → Webhook node.  
    - Connect Set Fields node → Webhook node.

14. **Add Sticky Notes**  
    - Add notes at relevant positions with content as per original workflow for user guidance.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                  | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Sign up at [Bright Data](https://brightdata.com/) and create a Web Unlocker zone under Proxies & Scraping → Web Unlocker API.                                                                                               | Setup instructions for Bright Data Web Unlocker.                                                |
| Configure Header Auth credentials in n8n with your Bright Data API key using Generic Auth Type: Header Authentication.                                                                                                        | Credential setup for Bright Data API.                                                           |
| Obtain Google Gemini API key (or access via Vertex AI or proxy) and configure credentials in n8n for Google Gemini nodes.                                                                                                     | Credential setup for Google Gemini (PaLM) API.                                                  |
| Update the LinkedIn URL and target URLs in the "Set Fields - URL and Webhook URL" node to customize scraping targets.                                                                                                         | Customization instructions.                                                                     |
| This workflow uses LangChain nodes and AI Agents for structured information extraction and formatting, leveraging Google Gemini models for both chat and embeddings.                                                           | AI model usage details.                                                                          |
| Pinecone vector database is used for semantic vector storage; ensure your Pinecone API key and index name are correctly configured.                                                                                           | Vector database setup.                                                                           |
| Webhook URLs are used for notification and downstream integration; update them to your own endpoints for monitoring or further processing.                                                                                   | Webhook integration notes.                                                                      |
| For customization: adjust prompts, embedding models, Pinecone metadata, and add validation or deduplication as needed.                                                                                                        | Workflow extensibility and customization advice.                                               |
| Video and blog resources for similar workflows and LangChain usage can be found on n8n community and LangChain documentation sites.                                                                                         | External learning resources (not linked here).                                                 |

---

This documentation provides a comprehensive, stepwise understanding of the workflow, enabling reproduction, modification, and troubleshooting by advanced users and AI agents alike.