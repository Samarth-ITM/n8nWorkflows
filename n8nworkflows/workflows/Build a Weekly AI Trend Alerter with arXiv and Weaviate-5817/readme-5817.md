Build a Weekly AI Trend Alerter with arXiv and Weaviate

https://n8nworkflows.xyz/workflows/build-a-weekly-ai-trend-alerter-with-arxiv-and-weaviate-5817


# Build a Weekly AI Trend Alerter with arXiv and Weaviate

### 1. Workflow Overview

This n8n workflow automates the creation and delivery of a weekly AI and Machine Learning (ML) research trend alert email. It is designed to fetch the latest research article abstracts from arXiv, enrich and classify them via AI, store them in a Weaviate vector database, and then use an AI agent to analyze trends and compile a summary email sent to users.

The workflow is logically divided into two main parts:

- **Part 1: Fetch, Clean, Enrich, and Insert arXiv Abstracts into Weaviate**
  - 1.1 Date Range Calculation: Determine the last week’s date range for querying.
  - 1.2 Data Retrieval: Query the arXiv API for recent AI/ML articles.
  - 1.3 Data Preprocessing: Convert XML to JSON, split entries, format and deduplicate data.
  - 1.4 Enrichment: Use an AI agent to classify articles by topic categories and predict potential impact.
  - 1.5 Data Insertion: Embed and insert enriched data into a Weaviate collection.
  - 1.6 Verification: Aggregate inserted article IDs and generate a static session ID for downstream use.

- **Part 2: Agentic Retrieval-Augmented Generation (RAG) for Trend Analysis and Email Notification**
  - 2.1 Setup AI Agent: Configure an AI agent to use Weaviate as a knowledge tool.
  - 2.2 Trend Analysis: Use the agent to identify key research trends from stored articles.
  - 2.3 Post-Processing: Parse and clean the AI agent’s output.
  - 2.4 Email Sending: Convert markdown to HTML and send the summarized trend report via email.

The workflow employs advanced AI capabilities including OpenAI and OpenRouter models, structured output parsing, vector embeddings, and Weaviate integration for scalable semantic search.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Date Range Calculation

**Overview:**  
Calculates today’s date and the start date from 7 days ago to define the weekly window for querying arXiv.

**Nodes Involved:**  
- Schedule Trigger  
- Get Current Date  
- Date & Time  
- Sticky Note7

**Node Details:**

- **Schedule Trigger**  
  - Type: Trigger node to run the workflow weekly (every 7 days).  
  - Configuration: Interval set to 7 days.  
  - Input/Output: No input; output triggers "Get Current Date" node.  
  - Failure Modes: Scheduler misconfiguration or time zone issues.

- **Get Current Date**  
  - Type: DateTime node  
  - Configuration: Returns current date/time for use in downstream queries.  
  - Output: Current date passed to "Date & Time" node.  
  - Edge Cases: System clock errors.

- **Date & Time**  
  - Type: DateTime  
  - Configuration: Subtracts 7 days from current date to get startDate.  
  - Key Expression: Uses expression referencing "Get Current Date" output to calculate `startDate` in format `yyyyMMdd`.  
  - Output: startDate used in HTTP request to arXiv.  
  - Failures: Expression errors or invalid date formats.

- **Sticky Note7**  
  - Contextual comment summarizing this block.

---

#### 1.2 Data Retrieval from arXiv

**Overview:**  
Queries the arXiv API for recent published articles in the categories cs.LG (Computer Science - Machine Learning) or stat.ML (Statistics - Machine Learning), limited to 200 results.

**Nodes Involved:**  
- Query arXiv (HTTP Request)  
- Convert XML to JSON  
- Sticky Note8

**Node Details:**

- **Query arXiv**  
  - Type: HTTP Request  
  - Configuration:  
    - URL dynamically built using `startDate` (7 days ago) to fetch latest papers.  
    - Query parameters include search categories, sorting by submission date descending, max 200 results.  
  - Output: XML response.  
  - Failures: HTTP errors, API rate limits, malformed URL.

- **Convert XML to JSON**  
  - Type: XML parser  
  - Configuration: Default parse options to convert arXiv XML feed to JSON.  
  - Output: JSON with article entries.  
  - Failures: Malformed XML, parsing errors.

- **Sticky Note8**  
  - Contextual explanation of fetching weekly articles.

---

#### 1.3 Data Preprocessing

**Overview:**  
Splits JSON feed into individual article entries, extracts and formats relevant metadata, and removes duplicates.

**Nodes Involved:**  
- Split Results  
- Prep Data for Weaviate (Set node)  
- Remove Duplicates  
- Sticky Note9

**Node Details:**

- **Split Results**  
  - Type: Split Out  
  - Configuration: Splits JSON array `feed.entry` into separate items for downstream processing.  
  - Output: Individual article JSON objects.  
  - Failures: Missing or malformed feed entries.

- **Prep Data for Weaviate**  
  - Type: Set node  
  - Configuration: Extracts and formats fields:  
    - `id` from article id  
    - `title`  
    - `summary` (abstract)  
    - `author` as array of author names  
    - `published` converted to ISO string  
    - `category` as array of category terms  
  - Expressions handle arrays or single objects to ensure consistent data types.  
  - Failures: Expression errors or missing fields.

- **Remove Duplicates**  
  - Type: Remove Duplicates  
  - Configuration: Compares based on `id` field to remove repeated articles.  
  - Failures: Incorrect comparison field could cause duplicates to persist.

- **Sticky Note9**  
  - Details the preprocessing steps and rationale.

---

#### 1.4 Enrichment via AI Agent (Topic Classification & Impact Prediction)

**Overview:**  
Uses an AI agent to classify each article into primary and secondary topic categories and estimate potential impact with a score from 1 to 5.

**Nodes Involved:**  
- Enrich Articles with Topic Classification (Agent node)  
- OpenRouter Chat Model1  
- Structured Output Parser1  
- Sticky Note (numbered 0 in JSON)  
- Sticky Note1  
- Sticky Note2

**Node Details:**

- **Enrich Articles with Topic Classification**  
  - Type: LangChain Agent  
  - Configuration:  
    - Prompt defines classification task with detailed instructions and predefined categories.  
    - System Message includes schema and scoring criteria.  
    - Source for prompt set to "Define below".  
    - Retry enabled on failure.  
  - Input: Article title and summary JSON.  
  - Output: JSON with `primary_category`, `secondary_categories`, and `potential_impact`.  
  - Failures: Model or API errors, malformed output, incorrect JSON format.

- **OpenRouter Chat Model1**  
  - Type: Language Model (OpenRouter)  
  - Configuration: Using Anthropic Claude 3.7 sonnet model.  
  - Role: Processes classification prompt.  
  - Failures: API errors, rate limits.

- **Structured Output Parser1**  
  - Type: Structured Output Parser  
  - Configuration: Validates and parses classification JSON output according to schema example.  
  - Failures: Invalid JSON output, parsing errors.

- **Sticky Note (0)**  
  - Instructions for AI classification node setup and category definitions.

- **Sticky Note1**  
  - Notes on model choice.

- **Sticky Note2**  
  - Describes merging enriched data with original article data and removing redundant fields.

---

#### 1.5 Merge Enriched Data and Insert into Weaviate

**Overview:**  
Merges enriched classification output with original article data, prepares it for embedding, splits text for embedding, generates embeddings, and inserts documents into the Weaviate vector store.

**Nodes Involved:**  
- Merge  
- Remove Fields (Set node)  
- Recursive Character Text Splitter1  
- Default Data Loader  
- Embeddings OpenAI  
- Weaviate Vector Store  
- Sticky Note10  
- Sticky Note11

**Node Details:**

- **Merge**  
  - Type: Merge  
  - Configuration: Combines original article data and classification output by position.  
  - Output: Enriched article JSON objects with metadata.

- **Remove Fields**  
  - Type: Set node  
  - Configuration: Removes redundant `output` fields and sets final data fields including `primary_topic`, `secondary_topics`, and `potential_impact` with correct types.  
  - Output: Clean JSON for uploading.

- **Recursive Character Text Splitter1**  
  - Type: Text Splitter  
  - Configuration: Splits article summaries into chunks of 2000 characters, typically one chunk per abstract, to optimize embedding.  
  - Failures: Long text truncation if chunk size too small.

- **Default Data Loader**  
  - Type: Document Data Loader  
  - Configuration: Loads JSON data with metadata fields to feed into embedding.  
  - Metadata fields: arxiv_id, published, author, title, category, primary_topic, secondary_topics, potential_impact.  
  - Failures: Missing metadata or wrong format.

- **Embeddings OpenAI**  
  - Type: Embeddings Node  
  - Configuration: Uses OpenAI embeddings model via credentials.  
  - Failures: API errors, rate limits.

- **Weaviate Vector Store**  
  - Type: Vector Store Insert  
  - Configuration:  
    - Mode: Insert documents  
    - Collection: `ArxivArticles` (existing or new)  
    - Text Key: `summary` (embedding target field)  
    - Credentials: Weaviate Cloud or local instance.  
  - Failures: Connection errors, authentication, schema mismatch.

- **Sticky Note10**  
  - Instructions for Weaviate Vector Store setup.

- **Sticky Note11**  
  - Details embedding node and data loader configuration.

---

#### 1.6 Verification of Upload & Session ID Generation

**Overview:**  
Aggregates the arXiv IDs of uploaded articles and adds a static session ID to trigger the downstream AI trend analysis agent.

**Nodes Involved:**  
- Aggregate Uploaded arXiv IDs  
- Add Static sessionId  
- Sticky Note12

**Node Details:**

- **Aggregate Uploaded arXiv IDs**  
  - Type: Aggregate  
  - Configuration: Collects list of all arXiv IDs from metadata after upload.  
  - Output: Array of arXiv IDs.

- **Add Static sessionId**  
  - Type: Set  
  - Configuration: Adds a fixed string `sessionId = "static_id"` to the data for session identification in memory and agent nodes.  
  - Output: Data with sessionId.

- **Sticky Note12**  
  - Explains verification step and static session ID purpose.

---

#### 2.1 AI Agent Setup for Trend Analysis

**Overview:**  
Configures an AI agent that uses Weaviate as a tool to retrieve and analyze the embedded article collection and generate a summarized email report of weekly AI/ML trends.

**Nodes Involved:**  
- Weaviate Vector Store1 (retrieval as tool)  
- Embeddings OpenAI1  
- Simple Memory  
- OpenRouter Chat Model  
- Structured Output Parser  
- Agentic RAG for Trend Analysis  
- Add Static sessionId (input to agent)  
- Sticky Note13  
- Sticky Note14  
- Sticky Note15

**Node Details:**

- **Weaviate Vector Store1**  
  - Type: Vector Store Retrieval as Tool for AI Agent  
  - Configuration:  
    - Mode: Retrieve documents as tool  
    - Tool Name: "ArxivPapers"  
    - Description: Instructs agent on how to query Weaviate and use metadata.  
    - Collection: `ArxivArticles` (same as upload)  
    - Include Metadata: Enabled  
  - Failures: Connection/auth issues, tool description clarity critical.

- **Embeddings OpenAI1**  
  - Type: Embeddings Node  
  - Configuration: OpenAI embeddings for query.  
  - Failures: API issues.

- **Simple Memory**  
  - Type: Memory Buffer Window  
  - Configuration: Short-term memory keyed by `sessionId` to maintain context during agent execution.  
  - Failures: Session ID mismatch.

- **OpenRouter Chat Model**  
  - Type: Language Model (OpenRouter)  
  - Configuration: Anthropic Claude 3.7 sonnet model, temperature set to 2 for creative summarization.  
  - Failures: API errors.

- **Structured Output Parser**  
  - Type: Output Parser  
  - Configuration: Schema expects JSON object with `subject` and `body` keys, defining email subject and content.  
  - Failures: Parsing errors or malformed agent output.

- **Agentic RAG for Trend Analysis**  
  - Type: LangChain Agent  
  - Configuration:  
    - System Message instructs agent to use Weaviate tool, perform vector searches, aggregate article counts by topic, identify trends, and output structured email content in JSON.  
    - Strict instructions prohibit hallucination or markdown code fences outside JSON.  
  - Failures: Model hallucination, API failures, retrieval errors.

- **Add Static sessionId**  
  - Provides the static session ID input to the agent for memory linkage.

- **Sticky Note13**  
  - Configuring AI Agent node with Weaviate as a tool.

- **Sticky Note14**  
  - Details on setting up the Weaviate vector store as a retrieval tool.

- **Sticky Note15**  
  - Instructions on adding model, memory, and output parser nodes.

---

#### 2.2 Post-Processing and Email Delivery

**Overview:**  
Cleans up the AI agent’s response, converts markdown email body to HTML, and sends the email using SMTP.

**Nodes Involved:**  
- Post Process Data (Set node)  
- Markdown (Markdown to HTML)  
- Send email  
- Sticky Note16  
- Sticky Note17

**Node Details:**

- **Post Process Data**  
  - Type: Set  
  - Configuration:  
    - Replaces escaped newline sequences (`\\n`) with real newlines in email body text for formatting.  
    - Sets `subject` and modified `output.body`.  
  - Failures: Text replacement errors.

- **Markdown**  
  - Type: Markdown node  
  - Configuration: Converts markdown email body to HTML with automatic URL linking.  
  - Output: HTML content.  
  - Failures: Markdown syntax issues.

- **Send email**  
  - Type: Email Send  
  - Configuration:  
    - Uses SMTP credentials.  
    - Subject set dynamically from agent output `subject`.  
    - Email body set to converted HTML content.  
  - Failures: SMTP connection/authentication failures, invalid email addresses.

- **Sticky Note16**  
  - Explains post-processing steps for agent response.

- **Sticky Note17**  
  - Instructions for email sending node configuration.

---

### 3. Summary Table

| Node Name                       | Node Type                                                 | Functional Role                                   | Input Node(s)                  | Output Node(s)                | Sticky Note                                                                                               |
|--------------------------------|-----------------------------------------------------------|--------------------------------------------------|-------------------------------|------------------------------|---------------------------------------------------------------------------------------------------------|
| Schedule Trigger               | Trigger                                                   | Trigger weekly execution                          |                               | Get Current Date              | ## 1. Specify date range for weekly automation 1. Calculate today's date 2. Calculate dates for the last week based on today |
| Get Current Date              | DateTime                                                  | Get current date/time                             | Schedule Trigger              | Date & Time                  |                                                                                                         |
| Date & Time                  | DateTime                                                  | Calculate start date (7 days ago)                 | Get Current Date              | Query arXiv                  |                                                                                                         |
| Query arXiv                  | HTTP Request                                             | Fetch latest AI/ML articles from arXiv API       | Date & Time                  | Convert XML to JSON          | ## 2. Fetch weekly articles from arXiv. Fetch ML article abstracts by querying the free arXiv API. The number of maximum papers returned is set by default to 200. You can change this by editing `max_results` in the query node. |
| Convert XML to JSON          | XML Parser                                               | Convert arXiv XML response to JSON                | Query arXiv                  | Split Results                | ## 3. Pre-process data 1. Convert XML response to JSON. 2. Split results by article ID. 3. Format data for Weaviate. 4. Remove any duplicates, if they exist. |
| Split Results                | Split Out                                                | Split JSON feed into individual articles          | Convert XML to JSON          | Prep Data for Weaviate       |                                                                                                         |
| Prep Data for Weaviate       | Set                                                     | Extract and format article metadata                | Split Results                | Remove Duplicates            |                                                                                                         |
| Remove Duplicates            | Remove Duplicates                                        | Remove duplicate articles based on ID              | Prep Data for Weaviate       | Enrich Articles with Topic Classification, Merge |                                                                                                         |
| Enrich Articles with Topic Classification | LangChain Agent                                         | Classify articles and predict impact               | Remove Duplicates            | Merge                       | ## 4. Enrich arXiv articles with topic classifications and potential impact predictions 1. Set source for prompt. 2. Provide classification instructions. 3. Define output format. |
| OpenRouter Chat Model1       | LangChain LM                                             | Chat model for classification task                 | Enrich Articles with Topic Classification | Structured Output Parser1    | Choose your models for the agent and structured output parser (we use `claude-3.7-sonnet`).              |
| Structured Output Parser1    | Output Parser                                            | Parse classification JSON output                    | OpenRouter Chat Model1       | Enrich Articles with Topic Classification |                                                                                                         |
| Merge                       | Merge                                                    | Combine original data with classification output     | Remove Duplicates, Enrich Articles with Topic Classification | Remove Fields              | ## 5. Post-process enriched data 1. Merge AI output with article data. 2. Remove redundant fields.          |
| Remove Fields               | Set                                                     | Clean final enriched article JSON                    | Merge                       | Weaviate Vector Store        |                                                                                                         |
| Recursive Character Text Splitter1 | Text Splitter                                          | Split abstracts into chunks for embedding           | Remove Fields               | Default Data Loader          | ## 7. Configure components for embeddings Embeddings Node, Data Loader, Text Splitter explained          |
| Default Data Loader          | Document Data Loader                                     | Load JSON data and metadata for embeddings          | Recursive Character Text Splitter1 | Embeddings OpenAI           |                                                                                                         |
| Embeddings OpenAI           | Embeddings Node                                          | Generate embeddings for abstracts                    | Default Data Loader          | Weaviate Vector Store        |                                                                                                         |
| Weaviate Vector Store       | Vector Store Insert                                     | Insert enriched, embedded documents into Weaviate   | Embeddings OpenAI           | Aggregate Uploaded arXiv IDs | ## 6. Create a new Weaviate collection or insert into existing with text key `summary`.                  |
| Aggregate Uploaded arXiv IDs | Aggregate                                               | Aggregate inserted arXiv IDs                         | Weaviate Vector Store        | Add Static sessionId         |                                                                                                         |
| Add Static sessionId        | Set                                                     | Add static session ID to track session               | Aggregate Uploaded arXiv IDs | Agentic RAG for Trend Analysis | ## 8. Confirm articles uploaded, generate static session ID for AI agent trigger                          |
| Weaviate Vector Store1      | Vector Store Retrieval as Tool                           | Provide Weaviate as a tool for AI agent retrieval    | Embeddings OpenAI1           | Agentic RAG for Trend Analysis | ## 3. Configure Weaviate Vector Store retrieval as tool for AI Agent                                    |
| Embeddings OpenAI1          | Embeddings Node                                          | Embeddings for queries to Weaviate                   |                             | Weaviate Vector Store1       |                                                                                                         |
| Simple Memory               | Memory Buffer Window                                    | Short-term memory for AI agent sessions              | Add Static sessionId         | Agentic RAG for Trend Analysis | ## 4. Add simple memory to AI agent                                                                    |
| OpenRouter Chat Model       | LangChain LM                                            | Chat model for AI agent summarization                 | Simple Memory                | Agentic RAG for Trend Analysis |                                                                                                         |
| Structured Output Parser    | Output Parser                                           | Parse AI agent output into structured email content  | OpenRouter Chat Model        | Agentic RAG for Trend Analysis |                                                                                                         |
| Agentic RAG for Trend Analysis | LangChain Agent                                       | Analyze trends using Weaviate tool and generate email summary | Weaviate Vector Store1, Simple Memory, OpenRouter Chat Model, Structured Output Parser, Add Static sessionId | Post Process Data         |                                                                                                         |
| Post Process Data           | Set                                                     | Clean agent output, replace escaped newlines          | Agentic RAG for Trend Analysis | Markdown                    | ## 2. Post-process agent response for markdown formatting                                              |
| Markdown                   | Markdown to HTML                                        | Convert markdown email body to HTML                    | Post Process Data            | Send email                  |                                                                                                         |
| Send email                 | Email Send                                              | Send the final email with subject and HTML content   | Markdown                    |                              | ## 5. Send the output as an email!                                                                      |

---

### 4. Reproducing the Workflow from Scratch

**Step 1:** Create a **Schedule Trigger** node  
- Set to run every 7 days.

**Step 2:** Add **Get Current Date** node (DateTime)  
- No special parameters; outputs current timestamp.

**Step 3:** Add **Date & Time** node  
- Operation: Subtract 7 days from current date.  
- Output field: `startDate`.  
- Expression: Reference output of Get Current Date.

**Step 4:** Add **HTTP Request** node named "Query arXiv"  
- URL:  
  `https://export.arxiv.org/api/query?search_query=cat:cs.LG+OR+cat:stat.ML&sortBy=submittedDate&sortOrder=descending&start=0&max_results=200&last_update_date_from={{ $json.startDate.toDateTime().toFormat("yyyyMMdd") }}`  
- Method: GET  
- Connect "Date & Time" output to this node.

**Step 5:** Add **XML** node named "Convert XML to JSON"  
- Default settings to parse XML response.  
- Connect from "Query arXiv".

**Step 6:** Add **Split Out** node named "Split Results"  
- Field to split: `feed.entry` (array of articles).  
- Connect from "Convert XML to JSON".

**Step 7:** Add **Set** node named "Prep Data for Weaviate"  
- Assign fields:  
  - `id`: `{{$json.id}}`  
  - `title`: `{{$json.title}}`  
  - `summary`: `{{$json.summary}}`  
  - `author`: Map author names to array  
  - `published`: Convert to ISO string  
  - `category`: Map category terms to array  
- Connect from "Split Results".

**Step 8:** Add **Remove Duplicates** node  
- Compare by field: `id`.  
- Connect from "Prep Data for Weaviate".

**Step 9:** Add **LangChain Agent** node named "Enrich Articles with Topic Classification"  
- Model: OpenRouter Chat Model (Anthropic Claude 3.7 sonnet).  
- Prompt: Define classification task with system prompt specifying categories and scoring.  
- Input: Article title and summary.  
- Enable retry on failure.  
- Connect from "Remove Duplicates".

**Step 10:** Add **OpenRouter Chat Model1** node  
- Model: Anthropic Claude 3.7 sonnet.  
- Connect to "Enrich Articles with Topic Classification".

**Step 11:** Add **Structured Output Parser1** node  
- JSON schema example for classification output including primary_category, secondary_categories, potential_impact.  
- Connect from OpenRouter Chat Model1.

**Step 12:** Add **Merge** node  
- Mode: Combine by position.  
- Connect "Remove Duplicates" (original data) and "Enrich Articles with Topic Classification" (classification output).

**Step 13:** Add **Set** node named "Remove Fields"  
- Remove redundant `output` field; assign enriched fields: `primary_topic`, `secondary_topics`, `potential_impact` with correct types.  
- Connect from "Merge".

**Step 14:** Add **Recursive Character Text Splitter1** node  
- Chunk size: 2000 characters.  
- Connect from "Remove Fields".

**Step 15:** Add **Default Data Loader** node  
- Type of data: JSON.  
- Mode: Load specific data with metadata fields for arxiv_id, published, author, title, category, primary_topic, secondary_topics, potential_impact.  
- Connect from "Recursive Character Text Splitter1".

**Step 16:** Add **Embeddings OpenAI** node  
- Provider: OpenAI (set credentials).  
- Connect from "Default Data Loader".

**Step 17:** Add **Weaviate Vector Store** node  
- Mode: Insert documents.  
- Collection: `ArxivArticles` (create or select existing).  
- Text Key: `summary`.  
- Credentials: Weaviate API credentials for cloud or local instance.  
- Connect from "Embeddings OpenAI".

**Step 18:** Add **Aggregate** node named "Aggregate Uploaded arXiv IDs"  
- Aggregate field: `metadata.arxiv_id`.  
- Connect from "Weaviate Vector Store".

**Step 19:** Add **Set** node named "Add Static sessionId"  
- Assign `sessionId` field with value `"static_id"`.  
- Connect from "Aggregate Uploaded arXiv IDs".

**Step 20:** Add **Weaviate Vector Store1** node (vector store retrieval as tool)  
- Mode: Retrieve documents as tool.  
- Tool Name: "ArxivPapers".  
- Description: Detailed instructions on tool usage by AI agent.  
- Collection: `ArxivArticles` (same as above).  
- Include Metadata: Enabled.  
- Connect from "Embeddings OpenAI1" (see next step).

**Step 21:** Add **Embeddings OpenAI1** node  
- Provider: OpenAI embeddings (set credentials).  
- Connect as input embedding provider for "Weaviate Vector Store1".

**Step 22:** Add **Simple Memory** node  
- Session key: `sessionId` (custom key).  
- Connect from "Add Static sessionId".

**Step 23:** Add **OpenRouter Chat Model** node for trend analysis  
- Model: Anthropic Claude 3.7 sonnet with temperature 2.  
- Connect memory and vector store tool inputs.

**Step 24:** Add **Structured Output Parser** node  
- Schema example: JSON object with `subject` and `body` keys for email content.  
- Connect from OpenRouter Chat Model.

**Step 25:** Add **LangChain Agent** node named "Agentic RAG for Trend Analysis"  
- Text: Prompt instructing agent to analyze articles using Weaviate tool, aggregate counts, identify trends, and output email summary JSON.  
- Connect inputs:  
  - AI memory (Simple Memory)  
  - AI language model (OpenRouter Chat Model)  
  - AI tool (Weaviate Vector Store1)  
  - AI output parser (Structured Output Parser)  
  - Main input: "Add Static sessionId".  

**Step 26:** Add **Set** node named "Post Process Data"  
- Replace escaped newlines (`\\n`) in agent output with real newlines in `output.body`.  
- Set `subject` field.  
- Connect from "Agentic RAG for Trend Analysis".

**Step 27:** Add **Markdown** node  
- Mode: markdownToHtml  
- Options: Simplified auto-link enabled.  
- Input: `output.body` from "Post Process Data".  
- Connect from "Post Process Data".

**Step 28:** Add **Send Email** node  
- SMTP credentials configured for sending email.  
- Subject: `{{ $json.subject }}`  
- HTML body: `{{ $json.data }}` (from Markdown node).  
- Connect from "Markdown".

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Context or Link                                                                                                                                                                                                                           |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow scrapes AI and machine learning article abstracts from [arXiv](https://arxiv.org), enriches them with topic categories using an LLM, and embeds them in a [Weaviate](https://weaviate.io) vector store. The vector store is used as a tool for agentic RAG to write a weekly summary of research trends.                                                                                                                                                                                    | Branding and general workflow description.                                                                                                                                                                                               |
| Prerequisites include: an existing Weaviate cluster ([local setup instructions](https://weaviate.io/developers/weaviate/installation/docker-compose#starter-docker-compose-file) or [Weaviate Cloud quickstart](https://weaviate.io/developers/wcs/quickstart)), API keys for OpenAI/OpenRouter, SMTP credentials for email sending, and a self-hosted n8n instance ([video setup guide](https://www.youtube.com/watch?v=kq5bmrjPPAY&t=108s)).                                                                                          | Setup requirements.                                                                                                                                                                                                                       |
| Sign up for a free 14-day trial of Weaviate Cloud [here](https://console.weaviate.cloud/?utm_source=recipe&utm_campaign=n8n&utm_content=n8n_arxiv_template).                                                                                                                                                                                                                                                                                                                                                      | Weaviate Cloud trial link.                                                                                                                                                                                                                 |
| The workflow uses Anthropic Claude 3.7 sonnet model through OpenRouter for both classification and trend summarization for better control and creativity. OpenAI embeddings power the vector store.                                                                                                                                                                                                                                                                                                             | Model choice rationale.                                                                                                                                                                                                                   |
| The workflow enforces strict JSON output from AI agents to ensure structured data parsing and reliable downstream processing. The agent is explicitly instructed to avoid hallucinations and to cite only valid arXiv IDs.                                                                                                                                                                                                                                                                                       | AI output constraints.                                                                                                                                                                                                                    |
| Email formatting uses markdown-to-HTML conversion with automatic link detection to produce readable, clickable email summaries.                                                                                                                                                                                                                                                                                                                                                                              | Email formatting details.                                                                                                                                                                                                                  |
| The workflow is designed to be modular: you can update the classification schema, models, or email templates independently.                                                                                                                                                                                                                                                                                                                                                                                  | Modularity and extensibility note.                                                                                                                                                                                                        |

---

**Disclaimer:** The text provided is derived exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data processed is legal and publicly available.