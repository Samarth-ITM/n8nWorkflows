Build a company website RAG chatbot using Apify, Pinecone and Gemini

https://n8nworkflows.xyz/workflows/build-a-company-website-rag-chatbot-using-apify--pinecone-and-gemini-14157


# Build a company website RAG chatbot using Apify, Pinecone and Gemini

# 1. Workflow Overview

This workflow builds a retrieval-augmented chatbot for a company website using three main services:

- **Apify** to crawl and extract website content
- **Pinecone** to store vectorized document chunks
- **Google Gemini** for embeddings and conversational answering

It is divided into two main functional areas:

- **Training / indexing logic**: periodically crawl the website, split the content into chunks, embed it, and store it in Pinecone
- **Chatbot / retrieval logic**: accept chat messages, query Pinecone through a vector-store tool, and answer with Gemini using retrieved website content

## 1.1 Training Logic

This block starts with a schedule trigger, runs an Apify website crawler against the target site, converts the scraped content into documents, splits those documents into chunks, generates embeddings with Gemini, and inserts them into a Pinecone index named `company-website`.

## 1.2 Chat Bot Logic

This block starts when a chat message is received. An AI Agent uses a Gemini chat model, optional short-term conversation memory, and a vector-store retrieval tool backed by Pinecone and Gemini embeddings. The agent is instructed to answer only from company website data and to explicitly admit when the answer is not found.

## 1.3 Configuration and Guidance Layer

Several sticky notes document setup requirements:

- Google Cloud / Google AI API key setup
- Apify account and actor configuration
- Pinecone account and index creation
- Suggested trigger-frequency customization
- Target website replacement in the Apify input JSON

---

# 2. Block-by-Block Analysis

## 2.1 Block: Website Crawling Trigger and Source Extraction

### Overview

This block initiates periodic retraining of the knowledge base. It schedules a crawl job and runs the Apify Website Content Crawler actor to extract website content from the configured domain.

### Nodes Involved

- `Schedule Trigger`
- `Scrape website data`

### Node Details

#### Schedule Trigger

- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Time-based workflow entry point for the training pipeline.
- **Configuration choices:**  
  Configured with a recurring interval using `weeks`. No custom frequency value is visible beyond the interval field, so this should be reviewed in the UI.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - No input
  - Outputs to `Scrape website data`
- **Version-specific requirements:**  
  Type version `1.3`
- **Edge cases or potential failure types:**  
  - Misconfigured schedule frequency
  - Workflow left inactive, preventing execution
  - Unexpected retraining cadence if the schedule is not adjusted
- **Sub-workflow reference:**  
  None

#### Scrape website data

- **Type and technical role:** `@apify/n8n-nodes-apify.apify`  
  Executes an Apify actor and returns its dataset output.
- **Configuration choices:**  
  - Operation: **Run actor and get dataset**
  - Actor source: **store**
  - Actor: `Website Content Crawler (apify/website-content-crawler)`
  - Authentication: `apifyOAuth2Api`
  - Custom JSON body defines crawl behavior
- **Key expressions or variables used:**  
  No n8n expressions are used in the JSON body. The main configurable field is:
  - `startUrls[0].url = https://www.tetriz.io/`
- **Important crawl settings interpreted from configuration:**
  - Uses Playwright adaptive crawler
  - Saves Markdown output
  - Blocks media
  - Removes common noisy page elements such as nav, footer, scripts, dialogs, cookie banners
  - Uses sitemaps
  - Crawl depth set to `0`
  - `readableTextCharThreshold = 100`
  - `respectRobotsTxtFile = false`
- **Input and output connections:**  
  - Input from `Schedule Trigger`
  - Output to `Pinecone Vector Store`
- **Version-specific requirements:**  
  Type version `1`
- **Edge cases or potential failure types:**  
  - Apify authentication failure
  - Actor quota / billing / concurrency limitations
  - Target site blocking crawler traffic
  - Empty dataset if selectors remove too much content
  - Crawl depth `0` may restrict discovered pages depending on actor semantics
  - If target website remains `tetriz.io`, the chatbot will index the wrong company
  - `respectRobotsTxtFile: false` may be undesirable for some deployments
- **Sub-workflow reference:**  
  None

---

## 2.2 Block: Document Preparation, Splitting, Embedding, and Vector Storage

### Overview

This block transforms scraped website output into chunked documents suitable for semantic search. It uses a default data loader, a recursive text splitter, Gemini embeddings, and Pinecone insert mode to build the searchable knowledge base.

### Nodes Involved

- `Pinecone Vector Store`
- `Embeddings Google Gemini`
- `Default Data Loader`
- `Recursive Character Text Splitter`

### Node Details

#### Pinecone Vector Store

- **Type and technical role:** `@n8n/n8n-nodes-langchain.vectorStorePinecone`  
  Vector database write node used here in **insert** mode.
- **Configuration choices:**  
  - Mode: `insert`
  - Pinecone index: `company-website`
  - No extra options specified
- **Key expressions or variables used:**  
  None shown.
- **Input and output connections:**  
  - Main input from `Scrape website data`
  - AI document input from `Default Data Loader`
  - AI embedding input from `Embeddings Google Gemini`
  - No downstream output used
- **Version-specific requirements:**  
  Type version `1`
- **Edge cases or potential failure types:**  
  - Pinecone credential or API key failure
  - Missing index `company-website`
  - Dimension mismatch between embeddings model and Pinecone index configuration
  - Duplicate inserts if the workflow reruns without deduplication strategy
  - Large crawl output causing rate-limit or payload-size issues
- **Sub-workflow reference:**  
  None

#### Embeddings Google Gemini

- **Type and technical role:** `@n8n/n8n-nodes-langchain.embeddingsGoogleGemini`  
  Generates vector embeddings for document chunks before insertion into Pinecone.
- **Configuration choices:**  
  Default configuration; credentials are expected to supply access.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Outputs embedding connection to `Pinecone Vector Store`
- **Version-specific requirements:**  
  Type version `1`
- **Edge cases or potential failure types:**  
  - Invalid Google AI API key
  - Model availability or regional restrictions
  - Rate limits during large indexing runs
  - Possible compatibility issues if embedding dimensions do not match index dimensions
- **Sub-workflow reference:**  
  None

#### Default Data Loader

- **Type and technical role:** `@n8n/n8n-nodes-langchain.documentDefaultDataLoader`  
  Converts incoming scraped data into LangChain-style documents.
- **Configuration choices:**  
  - `dataType = binary`
  - `binaryMode = specificField`
  - No extra options specified
- **Key expressions or variables used:**  
  None visible in the JSON.
- **Input and output connections:**  
  - AI text splitter input from `Recursive Character Text Splitter`
  - AI document output to `Pinecone Vector Store`
- **Version-specific requirements:**  
  Type version `1`
- **Edge cases or potential failure types:**  
  - Incoming Apify output may not be in the expected binary field format
  - Binary field not defined or not populated
  - Loader may fail if markdown or saved content is missing from actor output
- **Sub-workflow reference:**  
  None

#### Recursive Character Text Splitter

- **Type and technical role:** `@n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter`  
  Splits long documents into overlapping chunks for retrieval.
- **Configuration choices:**  
  - Chunk overlap: `100`
  - Other options left default
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Outputs AI text splitter connection to `Default Data Loader`
- **Version-specific requirements:**  
  Type version `1`
- **Edge cases or potential failure types:**  
  - Default chunk size may not be optimal if not explicitly configured
  - Overlap too small or too large depending on content density
  - Poor chunk boundaries may reduce retrieval quality
- **Sub-workflow reference:**  
  None

---

## 2.3 Block: Chat Entry, Agent Reasoning, and Conversation Memory

### Overview

This block receives a user chat message and routes it to an AI Agent. The agent uses Gemini for generation, keeps a short conversation context using buffer memory, and is explicitly instructed to rely on the retrieval tool for company-site answers.

### Nodes Involved

- `When chat message received`
- `AI Agent`
- `Window Buffer Memory`
- `Google Gemini Chat Model`

### Node Details

#### When chat message received

- **Type and technical role:** `@n8n/n8n-nodes-langchain.chatTrigger`  
  Chat entry point for the question-answering flow.
- **Configuration choices:**  
  Default options.
- **Key expressions or variables used:**  
  None shown.
- **Input and output connections:**  
  - No input
  - Main output to `AI Agent`
- **Version-specific requirements:**  
  Type version `1.1`
  - Includes webhook-based chat entry behavior through a generated `webhookId`
- **Edge cases or potential failure types:**  
  - Chat interface not correctly exposed or connected
  - Message payload format issues
  - Trigger unreachable if deployment/webhook configuration is incorrect
- **Sub-workflow reference:**  
  None

#### AI Agent

- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  Central orchestration node that decides when to use tools and how to answer.
- **Configuration choices:**  
  Uses a custom system message that:
  - Defines the bot as answering based on company website data
  - Instructs it to use the tool named `company_documents_tool`
  - Requires concise and accurate answers
  - Forces fallback text:  
    `"I cannot find the answer in the available resources."`
- **Key expressions or variables used:**  
  No dynamic expressions shown; core logic is in the system prompt.
- **Input and output connections:**  
  - Main input from `When chat message received`
  - AI language model input from `Google Gemini Chat Model`
  - AI memory input from `Window Buffer Memory`
  - AI tool input from `Vector Store Tool`
- **Version-specific requirements:**  
  Type version `1.7`
- **Edge cases or potential failure types:**  
  - Hallucination risk if the model answers without retrieval despite prompt instruction
  - Tool-call failure if vector retrieval is unavailable
  - Memory may retain context that biases later answers
  - Prompt may not fully prevent unsupported answers unless tool use is enforced by node behavior
- **Sub-workflow reference:**  
  None

#### Window Buffer Memory

- **Type and technical role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
  Provides short-term conversational memory to the agent.
- **Configuration choices:**  
  Default settings; no explicit window size is visible.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Outputs AI memory connection to `AI Agent`
- **Version-specific requirements:**  
  Type version `1.3`
- **Edge cases or potential failure types:**  
  - Default memory window may be too small or too large
  - Session scoping behavior should be validated in deployment context
  - Cross-session memory contamination if session handling is not configured as expected
- **Sub-workflow reference:**  
  None

#### Google Gemini Chat Model

- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
  Main LLM used by the AI Agent to reason, plan tool use, and produce answers.
- **Configuration choices:**  
  - Model name: `models/gemini-2.0-flash-exp`
  - No extra options specified
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Outputs AI language model connection to `AI Agent`
- **Version-specific requirements:**  
  Type version `1`
  - Uses an experimental Gemini model identifier, which may change or be retired
- **Edge cases or potential failure types:**  
  - Model deprecation or naming changes
  - Google API authentication failure
  - Rate limits or quota exhaustion
  - Latency spikes affecting chat responsiveness
- **Sub-workflow reference:**  
  None

---

## 2.4 Block: Retrieval Tooling and Pinecone Search

### Overview

This block equips the agent with a searchable knowledge tool. It wraps Pinecone retrieval as an agent tool, powered by Gemini embeddings for vector search and a Gemini chat model for retrieval-oriented processing.

### Nodes Involved

- `Vector Store Tool`
- `Pinecone Vector Store (Retrieval)`
- `Embeddings Google Gemini (retrieval)`
- `Google Gemini Chat Model (retrieval)`

### Node Details

#### Vector Store Tool

- **Type and technical role:** `@n8n/n8n-nodes-langchain.toolVectorStore`  
  Exposes the vector store as an agent-usable tool.
- **Configuration choices:**  
  - Tool name: `company_documents_tool`
  - Description: `Retrieve information from any company webpage`
- **Key expressions or variables used:**  
  Tool name is critical because the agent prompt explicitly refers to it.
- **Input and output connections:**  
  - AI vector store input from `Pinecone Vector Store (Retrieval)`
  - AI language model input from `Google Gemini Chat Model (retrieval)`
  - AI tool output to `AI Agent`
- **Version-specific requirements:**  
  Type version `1`
- **Edge cases or potential failure types:**  
  - Renaming the tool without updating the agent prompt breaks prompt/tool alignment
  - Retrieval may surface weak matches if document quality is poor
  - Tool can return irrelevant chunks if embeddings are low quality or index is noisy
- **Sub-workflow reference:**  
  None

#### Pinecone Vector Store (Retrieval)

- **Type and technical role:** `@n8n/n8n-nodes-langchain.vectorStorePinecone`  
  Connects to the Pinecone index for semantic retrieval during chat.
- **Configuration choices:**  
  - Pinecone index: `company-website`
  - No extra options specified
  - Retrieval mode is implied by how it is connected to the tool node
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - AI embedding input from `Embeddings Google Gemini (retrieval)`
  - AI vector store output to `Vector Store Tool`
- **Version-specific requirements:**  
  Type version `1`
- **Edge cases or potential failure types:**  
  - Wrong index name
  - Empty index if training has not yet run
  - Retrieval quality degradation if the index contains stale or duplicate data
- **Sub-workflow reference:**  
  None

#### Embeddings Google Gemini (retrieval)

- **Type and technical role:** `@n8n/n8n-nodes-langchain.embeddingsGoogleGemini`  
  Embeds user queries for semantic search against Pinecone.
- **Configuration choices:**  
  Defaults only.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Outputs AI embedding connection to `Pinecone Vector Store (Retrieval)`
- **Version-specific requirements:**  
  Type version `1`
- **Edge cases or potential failure types:**  
  - Embedding model drift versus indexed vectors
  - If indexing and retrieval embeddings differ, search quality will degrade
  - Authentication or quota failures
- **Sub-workflow reference:**  
  None

#### Google Gemini Chat Model (retrieval)

- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
  Language model attached to the vector store tool layer.
- **Configuration choices:**  
  - Model name: `models/gemini-2.0-flash-exp`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Outputs AI language model connection to `Vector Store Tool`
- **Version-specific requirements:**  
  Type version `1`
- **Edge cases or potential failure types:**  
  - Experimental model may change
  - Credential or quota issues
  - Possible redundancy depending on your tool setup, but still valid in this configuration
- **Sub-workflow reference:**  
  None

---

## 2.5 Block: In-Canvas Guidance and Setup Notes

### Overview

This block contains all visual documentation embedded in the workflow canvas. These notes are not executable but are important for setup, maintenance, and reproducing the workflow correctly.

### Nodes Involved

- `Sticky Note`
- `Sticky Note1`
- `Sticky Note2`
- `Sticky Note3`
- `Sticky Note4`
- `Sticky Note5`
- `Sticky Note6`
- `Sticky Note7`
- `Sticky Note8`
- `Sticky Note9`
- `Sticky Note10`

### Node Details

#### Sticky Note10

- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual label for the training section.
- **Configuration choices:**  
  Content: `## Training logic`
- **Connections:** None
- **Edge cases:** None
- **Sub-workflow reference:** None

#### Sticky Note2

- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual label for the chatbot section.
- **Configuration choices:**  
  Content: `## Chat bot logic`
- **Connections:** None
- **Edge cases:** None
- **Sub-workflow reference:** None

#### Sticky Note1

- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Setup instructions for required accounts, credentials, trigger behavior, and Apify input.
- **Configuration choices:**  
  Contains guidance covering:
  - Google Cloud / Vertex AI API / Google AI Studio API key
  - Apify account creation
  - Pinecone account and index creation
  - Credentials required in n8n
  - Trigger-frequency adjustment
  - Option to replace schedule with click trigger
  - Apify authentication and website URL setup
- **Connections:** None
- **Edge cases:**  
  Some terminology is slightly inconsistent:
  - Refers to Vertex AI API and Google AI API key together
  - Mentions “Google Gemini(PaLM) Api”
  Developers should verify the exact credential type supported by their n8n version.
- **Sub-workflow reference:** None

#### Sticky Note

- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Content: `Choose the update frequency`
- **Connections:** None

#### Sticky Note3

- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Content: `Connect your Apify Actor Setup target website in JSON input, instead of tetriz.io`
- **Connections:** None

#### Sticky Note4

- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Content: `Environment setting: Connect Pinecone (using Pinecone API key)`
- **Connections:** None

#### Sticky Note5

- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Content: `Environment setting: Connect Google Gemini (using your Google AI API key)`
- **Connections:** None

#### Sticky Note6

- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Content: `Environment setting: Connect Google Gemini (using your Google AI API key)`
- **Connections:** None

#### Sticky Note7

- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Content: `Environment setting: Connect Google Gemini (using your Google AI API key)`
- **Connections:** None

#### Sticky Note8

- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Content: `Environment setting: Connect Google Gemini (using your Google AI API key)`
- **Connections:** None

#### Sticky Note9

- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Content: `Environment setting: Connect Pinecone (using Pinecone API key)`
- **Connections:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Starts periodic website re-indexing |  | Scrape website data | ## Training logic |
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Starts periodic website re-indexing |  | Scrape website data | Choose the update frequency |
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Starts periodic website re-indexing |  | Scrape website data | ## Set up guide  / All Nodes with an orange sticky note require setup. / **Get your tools set up:** / 1 Google Cloud Project and Vertex AI API: / * Create a Google Cloud project. / * Enable the Vertex AI API for your project. / * Obtain a Google AI API key from Google AI Studio /  / 2 Get an Apify account / * Create an [Apify account](https://apify.com/) /  / 3 Pinecone Account: / * Create a free account on the Pinecone website. / * Obtain your API key from your Pinecone dashboard. / * Create an index named company-website in your Pinecone project. /  /  / **Configure credentials in your n8n environment for:** / * Google Gemini(PaLM) Api (using your Google AI API key) / * Pinecone API (using your Pinecone API key) /  / **Setup trigger frequency** / * Edit the Schedule  Trigger to match the frequency at which you wish to update your RAG / * If you want to train your chatbot only once, you can replace it with a click trigger. /  / **Set up the Apify node** / * Authenticate (via Oath or APi) / * Set up your website URL in the JSON input /  / ** You should be all set** |
| Scrape website data | @apify/n8n-nodes-apify.apify | Crawls website content and returns dataset | Schedule Trigger | Pinecone Vector Store | ## Training logic |
| Scrape website data | @apify/n8n-nodes-apify.apify | Crawls website content and returns dataset | Schedule Trigger | Pinecone Vector Store | Connect your Apify Actor / Setup target website in JSON input, instead of tetriz.io |
| Scrape website data | @apify/n8n-nodes-apify.apify | Crawls website content and returns dataset | Schedule Trigger | Pinecone Vector Store | ## Set up guide  / All Nodes with an orange sticky note require setup. / **Get your tools set up:** / 1 Google Cloud Project and Vertex AI API: / * Create a Google Cloud project. / * Enable the Vertex AI API for your project. / * Obtain a Google AI API key from Google AI Studio /  / 2 Get an Apify account / * Create an [Apify account](https://apify.com/) /  / 3 Pinecone Account: / * Create a free account on the Pinecone website. / * Obtain your API key from your Pinecone dashboard. / * Create an index named company-website in your Pinecone project. /  /  / **Configure credentials in your n8n environment for:** / * Google Gemini(PaLM) Api (using your Google AI API key) / * Pinecone API (using your Pinecone API key) /  / **Setup trigger frequency** / * Edit the Schedule  Trigger to match the frequency at which you wish to update your RAG / * If you want to train your chatbot only once, you can replace it with a click trigger. /  / **Set up the Apify node** / * Authenticate (via Oath or APi) / * Set up your website URL in the JSON input /  / ** You should be all set** |
| Pinecone Vector Store | @n8n/n8n-nodes-langchain.vectorStorePinecone | Inserts embedded website chunks into Pinecone | Scrape website data; Default Data Loader; Embeddings Google Gemini |  | ## Training logic |
| Pinecone Vector Store | @n8n/n8n-nodes-langchain.vectorStorePinecone | Inserts embedded website chunks into Pinecone | Scrape website data; Default Data Loader; Embeddings Google Gemini |  | **Environment setting:** / Connect Pinecone (using Pinecone API key) |
| Embeddings Google Gemini | @n8n/n8n-nodes-langchain.embeddingsGoogleGemini | Creates embeddings for indexed documents |  | Pinecone Vector Store | ## Training logic |
| Embeddings Google Gemini | @n8n/n8n-nodes-langchain.embeddingsGoogleGemini | Creates embeddings for indexed documents |  | Pinecone Vector Store | **Environment setting:** / Connect Google Gemini (using your Google AI API key) |
| Default Data Loader | @n8n/n8n-nodes-langchain.documentDefaultDataLoader | Loads binary scraped content into documents | Recursive Character Text Splitter | Pinecone Vector Store | ## Training logic |
| Recursive Character Text Splitter | @n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter | Splits documents into overlapping chunks |  | Default Data Loader | ## Training logic |
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger | Entry point for chatbot conversations |  | AI Agent | ## Chat bot logic |
| AI Agent | @n8n/n8n-nodes-langchain.agent | Orchestrates answer generation with tool use | When chat message received; Window Buffer Memory; Google Gemini Chat Model; Vector Store Tool |  | ## Chat bot logic |
| Window Buffer Memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Supplies short-term conversation history |  | AI Agent | ## Chat bot logic |
| Google Gemini Chat Model | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Main LLM for agent reasoning and answers |  | AI Agent | ## Chat bot logic |
| Google Gemini Chat Model | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Main LLM for agent reasoning and answers |  | AI Agent | **Environment setting:** / Connect Google Gemini (using your Google AI API key) |
| Vector Store Tool | @n8n/n8n-nodes-langchain.toolVectorStore | Exposes Pinecone retrieval as an agent tool | Pinecone Vector Store (Retrieval); Google Gemini Chat Model (retrieval) | AI Agent | ## Chat bot logic |
| Pinecone Vector Store (Retrieval) | @n8n/n8n-nodes-langchain.vectorStorePinecone | Reads vectors from Pinecone for semantic search | Embeddings Google Gemini (retrieval) | Vector Store Tool | ## Chat bot logic |
| Pinecone Vector Store (Retrieval) | @n8n/n8n-nodes-langchain.vectorStorePinecone | Reads vectors from Pinecone for semantic search | Embeddings Google Gemini (retrieval) | Vector Store Tool | **Environment setting:** / Connect Pinecone (using Pinecone API key) |
| Embeddings Google Gemini (retrieval) | @n8n/n8n-nodes-langchain.embeddingsGoogleGemini | Embeds user queries for retrieval |  | Pinecone Vector Store (Retrieval) | ## Chat bot logic |
| Embeddings Google Gemini (retrieval) | @n8n/n8n-nodes-langchain.embeddingsGoogleGemini | Embeds user queries for retrieval |  | Pinecone Vector Store (Retrieval) | **Environment setting:** / Connect Google Gemini (using your Google AI API key) |
| Google Gemini Chat Model (retrieval) | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | LLM attached to vector-store tool behavior |  | Vector Store Tool | ## Chat bot logic |
| Google Gemini Chat Model (retrieval) | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | LLM attached to vector-store tool behavior |  | Vector Store Tool | **Environment setting:** / Connect Google Gemini (using your Google AI API key) |
| Sticky Note | n8n-nodes-base.stickyNote | Visual setup note |  |  |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Visual setup guide |  |  |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Visual label for chatbot logic |  |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Visual Apify setup note |  |  |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Visual Pinecone setup note |  |  |  |
| Sticky Note5 | n8n-nodes-base.stickyNote | Visual Gemini setup note |  |  |  |
| Sticky Note6 | n8n-nodes-base.stickyNote | Visual Gemini setup note |  |  |  |
| Sticky Note7 | n8n-nodes-base.stickyNote | Visual Gemini setup note |  |  |  |
| Sticky Note8 | n8n-nodes-base.stickyNote | Visual Gemini setup note |  |  |  |
| Sticky Note9 | n8n-nodes-base.stickyNote | Visual Pinecone setup note |  |  |  |
| Sticky Note10 | n8n-nodes-base.stickyNote | Visual label for training logic |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `RAG Workflow For Company Website Using Apify and Gemini`.

2. **Prepare external services**
   - Create or confirm access to:
     - **Apify**
     - **Pinecone**
     - **Google AI / Gemini**
   - In Pinecone, create an index named:
     - `company-website`
   - Ensure the index dimension matches the embedding model used by the Gemini embeddings node.

3. **Create credentials in n8n**
   - Add **Apify OAuth2/API** credentials for the Apify node.
   - Add **Pinecone API** credentials for both Pinecone vector store nodes.
   - Add **Google Gemini / Google AI API** credentials for:
     - `Embeddings Google Gemini`
     - `Embeddings Google Gemini (retrieval)`
     - `Google Gemini Chat Model`
     - `Google Gemini Chat Model (retrieval)`

4. **Add the training trigger**
   - Create a `Schedule Trigger` node.
   - Configure recurrence based on how often the website should be re-indexed.
   - In the original workflow, the interval is based on `weeks`.
   - If you only want a one-time indexing run, you can replace this with a manual trigger.

5. **Add the Apify crawler node**
   - Create an `Apify` node named `Scrape website data`.
   - Set:
     - **Operation**: `Run actor and get dataset`
     - **Actor source**: `Store`
     - **Actor**: `Website Content Crawler (apify/website-content-crawler)`
     - **Authentication**: your Apify credential
   - In the custom JSON body, configure the crawl target.
   - Replace:
     - `https://www.tetriz.io/`
     with your company website URL.
   - Keep or adapt these notable options:
     - `saveMarkdown: true`
     - `blockMedia: true`
     - `useSitemaps: true`
     - `removeCookieWarnings: true`
     - CSS removal rules for navigation/footer/dialog noise
   - Connect:
     - `Schedule Trigger` → `Scrape website data`

6. **Add the training Pinecone vector store**
   - Create a `Pinecone Vector Store` node named `Pinecone Vector Store`.
   - Set:
     - **Mode**: `Insert`
     - **Index**: `company-website`
   - This node will receive:
     - Main input from Apify crawl results
     - AI document input from the data loader
     - AI embedding input from the embedding node

7. **Add the document embeddings node for training**
   - Create an `Embeddings Google Gemini` node named `Embeddings Google Gemini`.
   - Attach your Google AI credential.
   - Connect:
     - `Embeddings Google Gemini` → `Pinecone Vector Store` using the **AI embedding** connection.

8. **Add the text splitter**
   - Create a `Recursive Character Text Splitter` node.
   - Set:
     - **Chunk overlap**: `100`
   - Leave other options at defaults unless you want a custom chunk size.
   - This splitter will feed the data loader through the AI text-splitter connection.

9. **Add the default document loader**
   - Create a `Default Data Loader` node.
   - Set:
     - **Data type**: `Binary`
     - **Binary mode**: `Specific field`
   - If required in your n8n version, explicitly choose the binary field containing the markdown/file data returned by Apify.
   - Connect:
     - `Recursive Character Text Splitter` → `Default Data Loader` using **AI textSplitter**
     - `Default Data Loader` → `Pinecone Vector Store` using **AI document**

10. **Connect Apify crawl output to Pinecone insertion**
    - Connect:
      - `Scrape website data` → `Pinecone Vector Store` via the main connection

11. **Add the chat trigger**
    - Create a `When chat message received` node.
    - Keep default options unless your deployment requires custom chat handling.
    - This node is the chatbot entry point.

12. **Add the main AI agent**
    - Create an `AI Agent` node.
    - In **System Message**, paste this instruction set:

      - You are a chatbot designed to answer questions based on company website.
      - Retrieve relevant information from the provided website pages and provide a concise, accurate, and informative answer to the question.
      - Use the tool called `company_documents_tool` to retrieve any information from the company's website.
      - If the answer cannot be found in the provided documents, respond with `I cannot find the answer in the available resources.`

    - Connect:
      - `When chat message received` → `AI Agent`

13. **Add the main chat model**
    - Create a `Google Gemini Chat Model` node.
    - Set:
      - **Model**: `models/gemini-2.0-flash-exp`
    - Attach your Google AI credential.
    - Connect:
      - `Google Gemini Chat Model` → `AI Agent` using **AI languageModel**

14. **Add memory**
    - Create a `Window Buffer Memory` node.
    - Leave default settings or configure the window size if needed.
    - Connect:
      - `Window Buffer Memory` → `AI Agent` using **AI memory**

15. **Add the vector retrieval tool**
    - Create a `Vector Store Tool` node.
    - Set:
      - **Name**: `company_documents_tool`
      - **Description**: `Retrieve information from any company webpage`
    - The tool name must match the name referenced in the AI Agent system message.

16. **Add the retrieval Pinecone vector store**
    - Create another `Pinecone Vector Store` node and rename it `Pinecone Vector Store (Retrieval)`.
    - Set it to use the same Pinecone index:
      - `company-website`
    - Connect it to the vector store tool through the **AI vectorStore** connection.

17. **Add retrieval embeddings**
    - Create another `Embeddings Google Gemini` node named `Embeddings Google Gemini (retrieval)`.
    - Attach the same Google AI credential.
    - Connect:
      - `Embeddings Google Gemini (retrieval)` → `Pinecone Vector Store (Retrieval)` using **AI embedding**

18. **Add retrieval chat model**
    - Create another `Google Gemini Chat Model` node named `Google Gemini Chat Model (retrieval)`.
    - Set:
      - **Model**: `models/gemini-2.0-flash-exp`
    - Connect:
      - `Google Gemini Chat Model (retrieval)` → `Vector Store Tool` using **AI languageModel**

19. **Connect retrieval tool to the agent**
    - Connect:
      - `Pinecone Vector Store (Retrieval)` → `Vector Store Tool` using **AI vectorStore**
      - `Vector Store Tool` → `AI Agent` using **AI tool**

20. **Validate the training path**
    - Confirm the indexing branch is:
      - `Schedule Trigger` → `Scrape website data` → `Pinecone Vector Store`
      - `Embeddings Google Gemini` → `Pinecone Vector Store`
      - `Recursive Character Text Splitter` → `Default Data Loader` → `Pinecone Vector Store`

21. **Validate the chat path**
    - Confirm the chatbot branch is:
      - `When chat message received` → `AI Agent`
      - `Google Gemini Chat Model` → `AI Agent`
      - `Window Buffer Memory` → `AI Agent`
      - `Embeddings Google Gemini (retrieval)` → `Pinecone Vector Store (Retrieval)` → `Vector Store Tool` → `AI Agent`
      - `Google Gemini Chat Model (retrieval)` → `Vector Store Tool`

22. **Run initial indexing**
    - Execute the training branch once.
    - Confirm that documents are inserted into Pinecone.
    - If no data appears, inspect:
      - Apify dataset output
      - Document loader binary-field assumptions
      - Pinecone credential/index setup
      - Embedding dimension compatibility

23. **Test the chatbot**
    - Trigger `When chat message received` with a question about the indexed website.
    - Verify the response comes from the indexed content.
    - Test a question that is not answered on the site and confirm the fallback response:
      - `I cannot find the answer in the available resources.`

24. **Optional hardening changes**
    - Add deduplication or delete-old-vectors logic before reinserting content.
    - Tune chunk size and overlap.
    - Change schedule frequency.
    - Replace the experimental Gemini model if needed.
    - Add metadata filters if you want page-specific retrieval.

### Sub-workflow setup

This workflow does **not** invoke any sub-workflows and does not contain any `Execute Workflow` nodes.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Apify account creation is explicitly referenced in the workflow notes. | [https://apify.com/](https://apify.com/) |
| Pinecone index must be created in advance with the name `company-website`. | Pinecone project setup |
| The Apify crawler input still targets `https://www.tetriz.io/` and must be replaced with the actual company website. | Apify node custom JSON input |
| The workflow notes state that orange sticky-note areas require setup. | Canvas guidance |
| The schedule trigger can be replaced with a manual trigger if only one indexing run is needed. | Training branch design |
| The model name used in both Gemini chat nodes is `models/gemini-2.0-flash-exp`, which may be experimental and subject to change. | Gemini chat model nodes |
| The system prompt forces a conservative fallback response when the answer is not found in indexed documents. | AI Agent configuration |