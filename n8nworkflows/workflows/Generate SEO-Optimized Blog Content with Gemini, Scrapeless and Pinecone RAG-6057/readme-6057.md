Generate SEO-Optimized Blog Content with Gemini, Scrapeless and Pinecone RAG

https://n8nworkflows.xyz/workflows/generate-seo-optimized-blog-content-with-gemini--scrapeless-and-pinecone-rag-6057


# Generate SEO-Optimized Blog Content with Gemini, Scrapeless and Pinecone RAG

### 1. Workflow Overview

This workflow automates the generation of SEO-optimized blog content by integrating web scraping, knowledge base creation, semantic vector storage, and AI-driven content generation. It targets content creators, marketers, or SEO specialists who want to automate blog writing with data-driven insights and AI language models.

The workflow is logically divided into four main blocks:

- **1.1 Scrape and Crawl Website for Knowledge Base**  
  Crawls and scrapes blog content from a target website, extracting markdown content and links, then further scraping detailed pages.

- **1.2 Store Data on Pinecone Vector Store**  
  Processes scraped content by embedding it via Google Gemini embeddings and storing it in the Pinecone vector database for semantic search.

- **1.3 SERP Analysis Using AI**  
  Takes user-provided keywords and search intent, scrapes Google SERP data using Scrapeless, and uses an LLM chain to analyze and suggest relevant keywords aligned with search intent.

- **1.4 Use the Knowledge Base to Create Blogs**  
  Receives chat input queries, loads relevant documents from Pinecone vector store, uses memory buffering, and an AI agent (Google Gemini Chat model) to generate SEO-optimized blog content dynamically.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Scrape and Crawl Website for Knowledge Base

**Overview:**  
This block scrapes a target blog website to build a knowledge base. It extracts markdown content, parses titles, main content, and embedded links, then scrapes linked pages for detailed content.

**Nodes Involved:**  
- When clicking ‘Execute workflow’  
- Crawl all Blogs  
- Parse content and extract information (Code node)  
- Split Out the url and text  
- Scrape detailed contents  
- Loop Over Items  
- Aggregate  
- Convert to File  
- Pinecone Vector Store (insert)  
- Recursive Character Text Splitter  
- Default Data Loader  
- Embeddings Google Gemini  

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Starts the crawling and scraping process on manual execution.  
  - No parameters.  
  - Outputs to Crawl all Blogs and Edit Fields1.  
  - Potential failure: User forgetting to trigger.

- **Crawl all Blogs**  
  - Type: Scrapeless crawler  
  - Role: Crawls the blog website at https://www.scrapeless.com/en/blog, limited to 20 pages.  
  - Outputs markdown content of blog pages.  
  - Credential: Scrapeless API.  
  - Failure modes: API rate limits, site structure changes, network issues.

- **Parse content and extract information**  
  - Type: Code node (JavaScript)  
  - Role: Parses markdown to extract the blog title (first-level heading), main content (rest of markdown), and any embedded links.  
  - Inputs: Markdown content from Crawl all Blogs.  
  - Outputs: JSON with title, mainContent, extractedLinks array.  
  - Edge cases: Missing or malformed markdown, no title found, empty links.

- **Split Out the url and text**  
  - Type: Split Out (array splitter)  
  - Role: Splits extractedLinks array into separate items for further scraping.  
  - Output connected to Scrape detailed contents.

- **Scrape detailed contents**  
  - Type: Scrapeless crawler  
  - Role: Crawls each extracted URL from the links to get detailed content.  
  - Credentials: Scrapeless API.  
  - Outputs to Loop Over Items.

- **Loop Over Items**  
  - Type: Split In Batches  
  - Role: Processes scraped detailed contents in batches for aggregation and further processing.  
  - Outputs: Two branches - Aggregate and Scrape detailed contents (re-iterative? The connection to Scrape detailed contents is unusual; possibly a misconfiguration or intended for deep crawling).  
  - Edge case: Large batch size causing memory issues.

- **Aggregate**  
  - Type: Aggregate node  
  - Role: Collects all markdown fields from batch items into a single array.  
  - Outputs to Convert to File.

- **Convert to File**  
  - Type: Convert to File  
  - Role: Converts aggregated markdown data to text file format for embedding.  
  - Outputs to Pinecone Vector Store (insert).

- **Pinecone Vector Store (insert)**  
  - Type: Vector Store Pinecone  
  - Role: Inserts embedded documents into Pinecone under namespace "DataPlace" and index "seo-writer".  
  - Credential: Pinecone API.  
  - Input: Text data to embed.  
  - Failure modes: Pinecone connection issues, API quota, malformed data.

- **Recursive Character Text Splitter**  
  - Type: Text Splitter  
  - Role: Splits long text documents into chunks of 2000 characters with 200 overlap for better embedding.  
  - Connected to Default Data Loader.

- **Default Data Loader**  
  - Type: Document loader  
  - Role: Loads the split text chunks for embedding and storage.  
  - Outputs ai_document to Pinecone Vector Store (insert).

- **Embeddings Google Gemini**  
  - Type: Embeddings node (Google Gemini)  
  - Role: Creates embeddings from text chunks using model "models/embedding-001".  
  - Credential: Google Gemini (PaLM) API.  
  - Outputs to Pinecone Vector Store insert.

---

#### 2.2 Store Data on Pinecone Vector Store

**Overview:**  
This block overlaps with the above but focuses on embedding the processed text content and storing it in Pinecone vector database for semantic search.

**Nodes Involved:**  
- Recursive Character Text Splitter  
- Default Data Loader  
- Embeddings Google Gemini  
- Pinecone Vector Store (insert)

**Details:**  
See above, these nodes work together to chunk, embed, and store documents.

---

#### 2.3 SERP Analysis Using AI

**Overview:**  
This block analyzes target keywords by scraping Google SERP results via Scrapeless, then uses an AI language model chain to suggest more relevant keywords aligned with search intent.

**Nodes Involved:**  
- Edit Fields1  
- Analyze target keywords on Google SERP  
- Basic LLM Chain1  
- Google Gemini Chat Model1  
- Markdown  
- HTML  

**Node Details:**

- **Edit Fields1**  
  - Type: Set node  
  - Role: Sets static fields "Keywords" and "Search Intent" (e.g., "Scraping", "Google trends" and "People searching to get tips on Scraping").  
  - Outputs to Analyze target keywords on Google SERP.

- **Analyze target keywords on Google SERP**  
  - Type: Scrapeless scraper  
  - Role: Scrapes Google SERP results for the keywords defined in Edit Fields1.  
  - Credential: Scrapeless API.  
  - Outputs SERP data to Basic LLM Chain1.

- **Basic LLM Chain1**  
  - Type: Langchain LLM Chain  
  - Role: Takes keywords, search intent, and SERP data as input, and prompts an AI to suggest more relevant keywords aligned with the intent.  
  - Prompt includes JSON stringified SERP organic_results.  
  - Uses Google Gemini Chat Model1 as language model.

- **Google Gemini Chat Model1**  
  - Type: Langchain Google Gemini Chat model  
  - Role: Language model "models/gemini-1.5-flash" used by Basic LLM Chain1.  
  - Credential: Google Gemini API.

- **Markdown**  
  - Type: Markdown to HTML converter  
  - Role: Converts markdown text output from Basic LLM Chain1 into HTML.  
  - Output connected to HTML node.

- **HTML**  
  - Type: HTML node  
  - Role: Wraps HTML content into a styled webpage with embedded CSS and JavaScript console log.  
  - Provides a formatted report summary for viewing keyword analysis results.

**Edge cases:**  
- SERP scraping failures due to Google blocking or API limits.  
- AI prompt failures if organic_results data is malformed or missing.  
- Model API quota or latency issues.

---

#### 2.4 Use the Knowledge Base to Create Blogs

**Overview:**  
This block interfaces with chat input to generate SEO-optimized blog content by querying the Pinecone knowledge base, maintaining session memory, and leveraging an AI agent with Google Gemini chat model.

**Nodes Involved:**  
- When chat message received  
- Pinecone Vector Store3 (load)  
- Embeddings Google Gemini3  
- AI Agent1  
- Window Buffer Memory  
- Google Gemini Chat Model3  

**Node Details:**

- **When chat message received**  
  - Type: Langchain chat trigger  
  - Role: Listens for chat input webhook requests, triggers content generation.  
  - Outputs to Pinecone Vector Store3.

- **Pinecone Vector Store3 (load)**  
  - Type: Vector Store Pinecone (load mode)  
  - Role: Loads relevant documents from Pinecone vector store by embedding chat input query and performing semantic search in "seo-writer" index and "DataPlace" namespace.  
  - Credential: Pinecone API.  
  - Outputs documents to AI Agent1.

- **Embeddings Google Gemini3**  
  - Type: Embeddings node  
  - Role: Generates embeddings for the chat input query to enable vector search in Pinecone.  
  - Credential: Google Gemini API.

- **AI Agent1**  
  - Type: Langchain agent  
  - Role: Uses context documents from Pinecone and chat input to generate SEO-optimized blog content.  
  - System message instructs to use provided context for answering.  
  - PromptType: define.  
  - Uses Google Gemini Chat Model3 as language model.  
  - Inputs memory from Window Buffer Memory.

- **Window Buffer Memory**  
  - Type: Memory Buffer Window  
  - Role: Maintains conversation state/session memory keyed by chat sessionId for context continuity.

- **Google Gemini Chat Model3**  
  - Type: Langchain Google Gemini Chat model  
  - Role: Language model “models/gemini-1.5-flash” powering AI Agent1.  
  - Credential: Google Gemini API.

**Edge cases:**  
- Missing or invalid chat sessionId breaking memory context.  
- Pinecone load failures or empty results.  
- AI model latency or API quota errors.

---

### 3. Summary Table

| Node Name                      | Node Type                              | Functional Role                                   | Input Node(s)                     | Output Node(s)                | Sticky Note                                                                                 |
|--------------------------------|--------------------------------------|-------------------------------------------------|----------------------------------|------------------------------|---------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’| Manual Trigger                       | Starts crawling and scraping workflow           | -                                | Crawl all Blogs, Edit Fields1 | ## Scrape and Crawl Website for Knowledge Base                                              |
| Crawl all Blogs                | Scrapeless crawler                   | Crawls blog website for raw markdown content    | When clicking ‘Execute workflow’ | Parse content and extract information |                                                                                             |
| Parse content and extract information | Code node (JS)                     | Parses markdown, extracts title, content, links | Crawl all Blogs                  | Split Out the url and text    |                                                                                             |
| Split Out the url and text     | Split Out node                      | Splits extracted links for detailed scraping    | Parse content and extract information | Scrape detailed contents     |                                                                                             |
| Scrape detailed contents       | Scrapeless crawler                   | Scrapes each extracted link for detailed content| Split Out the url and text       | Loop Over Items               |                                                                                             |
| Loop Over Items                | Split In Batches                    | Processes scraped details in batches             | Scrape detailed contents         | Aggregate, Scrape detailed contents |                                                                                             |
| Aggregate                     | Aggregate                          | Aggregates markdown fields from batch items     | Loop Over Items                  | Convert to File               |                                                                                             |
| Convert to File               | Convert to File                    | Converts aggregated markdown to text file       | Aggregate                       | Pinecone Vector Store         |                                                                                             |
| Pinecone Vector Store          | Vector Store Pinecone (insert)     | Inserts embedded documents to Pinecone           | Convert to File, Default Data Loader, Embeddings Google Gemini | -                            | ## Store data on Pinecone                                                                   |
| Recursive Character Text Splitter | Text Splitter                     | Splits text into chunks for embedding            | -                              | Default Data Loader           |                                                                                             |
| Default Data Loader            | Document Loader                   | Loads text chunks for embedding                   | Recursive Character Text Splitter | Pinecone Vector Store         |                                                                                             |
| Embeddings Google Gemini       | Embeddings node                   | Creates embeddings for text                        | Default Data Loader             | Pinecone Vector Store         |                                                                                             |
| Edit Fields1                  | Set node                         | Defines keywords and search intent for SERP analysis | When clicking ‘Execute workflow’ | Analyze target keywords on Google SERP | ## SERP Analysis using AI                                                                   |
| Analyze target keywords on Google SERP | Scrapeless scraper               | Scrapes Google SERP for keywords                   | Edit Fields1                   | Basic LLM Chain1             |                                                                                             |
| Basic LLM Chain1              | Langchain LLM Chain               | Suggests relevant keywords based on SERP and intent | Analyze target keywords on Google SERP | Markdown                    |                                                                                             |
| Google Gemini Chat Model1     | Langchain Google Gemini Chat model | Language model for Basic LLM Chain1                | -                              | Basic LLM Chain1             |                                                                                             |
| Markdown                     | Markdown to HTML converter         | Converts markdown output to HTML                   | Basic LLM Chain1               | HTML                        |                                                                                             |
| HTML                        | HTML generator                   | Styles and formats HTML content                     | Markdown                      | -                           |                                                                                             |
| When chat message received    | Langchain chat trigger            | Receives chat input to trigger blog generation    | -                              | Pinecone Vector Store3       | ## Use the Knowledge Base to Create Blogs                                                   |
| Pinecone Vector Store3        | Vector Store Pinecone (load)      | Loads relevant documents from Pinecone             | When chat message received     | AI Agent1                   |                                                                                             |
| Embeddings Google Gemini3     | Embeddings node                   | Embeds chat input query for vector search          | -                              | Pinecone Vector Store3       |                                                                                             |
| AI Agent1                   | Langchain agent                  | Generates SEO-optimized blog content using context | Pinecone Vector Store3, Window Buffer Memory | -                           |                                                                                             |
| Window Buffer Memory          | Memory Buffer Window              | Maintains session memory for chat                   | -                              | AI Agent1                   |                                                                                             |
| Google Gemini Chat Model3     | Langchain Google Gemini Chat model | Language model for AI Agent1                         | -                              | AI Agent1                   |                                                                                             |
| Sticky Note                  | Sticky Note                      | Visual annotation                                   | -                              | -                           | ## Scrape and Crawl Website for Knowledge Base                                              |
| Sticky Note1                 | Sticky Note                      | Visual annotation                                   | -                              | -                           | ## Store data on Pinecone                                                                   |
| Sticky Note2                 | Sticky Note                      | Visual annotation                                   | -                              | -                           | ## SERP Analysis using AI                                                                   |
| Sticky Note3                 | Sticky Note                      | Visual annotation                                   | -                              | -                           | ## Use the Knowledge Base to Create Blogs                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the manual trigger node:**  
   - Type: Manual Trigger  
   - Purpose: Start crawling and keyword analysis.

2. **Scrape and Crawl Website:**  
   - Add Scrapeless node "Crawl all Blogs"  
     - Operation: Crawl  
     - URL: https://www.scrapeless.com/en/blog  
     - Limit crawl pages: 20  
     - Credential: Scrapeless API  
   - Connect manual trigger → Crawl all Blogs.

3. **Parse markdown content:**  
   - Add Code node "Parse content and extract information"  
   - JavaScript code to extract title, mainContent, and links from markdown input.  
   - Connect Crawl all Blogs → Parse content and extract information.

4. **Split extracted links:**  
   - Add Split Out node "Split Out the url and text"  
   - Field to split out: extractedLinks  
   - Connect Parse content and extract information → Split Out the url and text.

5. **Scrape detailed pages:**  
   - Add Scrapeless node "Scrape detailed contents"  
   - Operation: Crawl  
   - URL: expression to use current item's URL  
   - Credential: Scrapeless API  
   - Connect Split Out the url and text → Scrape detailed contents.

6. **Batch process detailed contents:**  
   - Add Split In Batches node "Loop Over Items"  
   - Connect Scrape detailed contents → Loop Over Items.

7. **Aggregate markdown fields:**  
   - Add Aggregate node "Aggregate"  
   - Aggregate all markdown fields in batch  
   - Connect Loop Over Items → Aggregate.

8. **Convert aggregated markdown into text file:**  
   - Add Convert to File node "Convert to File"  
   - Operation: toText  
   - Source property: data  
   - Connect Aggregate → Convert to File.

9. **Setup Pinecone vector store insert:**  
   - Add Pinecone Vector Store node  
   - Mode: insert  
   - Pinecone Index: "seo-writer"  
   - Namespace: "DataPlace"  
   - Credential: Pinecone API  
   - Connect Convert to File → Pinecone Vector Store.

10. **Add Recursive Character Text Splitter node:**  
    - Chunk size: 2000  
    - Chunk overlap: 200  
    - Connect upstream sources as needed (usually before embedding).

11. **Add Default Data Loader node:**  
    - Data type: binary  
    - Connect Recursive Character Text Splitter → Default Data Loader.

12. **Add Google Gemini Embeddings node:**  
    - Model: "models/embedding-001"  
    - Credential: Google Gemini (PaLM) API  
    - Connect Default Data Loader → Embeddings Google Gemini → Pinecone Vector Store.

13. **Setup keyword and search intent fields:**  
    - Add Set node "Edit Fields1"  
    - Add fields:  
      - Keywords: "Scraping", "Google trends"  
      - Search Intent: "People searching to get tips on Scraping"  
    - Connect manual trigger → Edit Fields1.

14. **Scrape Google SERP for keywords:**  
    - Add Scrapeless node "Analyze target keywords on Google SERP"  
    - Query: Expression to use Keywords field from Edit Fields1  
    - Credential: Scrapeless API  
    - Connect Edit Fields1 → Analyze target keywords on Google SERP.

15. **Create AI chain for keyword suggestions:**  
    - Add Langchain LLM Chain node "Basic LLM Chain1"  
    - Prompt: Provide keywords, search intent, SERP organic_results to suggest more relevant keywords.  
    - Connect Analyze target keywords on Google SERP → Basic LLM Chain1.

16. **Add Google Gemini Chat Model node:**  
    - Model: "models/gemini-1.5-flash"  
    - Credential: Google Gemini API  
    - Connect Google Gemini Chat Model1 → Basic LLM Chain1.

17. **Convert AI markdown output to HTML:**  
    - Add Markdown node to convert markdown to HTML  
    - Connect Basic LLM Chain1 → Markdown.  
    - Add HTML node with custom styling and formatting.  
    - Connect Markdown → HTML.

18. **Setup chat input trigger:**  
    - Add Langchain Chat Trigger node "When chat message received"  
    - Webhook enabled for chat input.  

19. **Load relevant docs from Pinecone:**  
    - Add Pinecone Vector Store node (mode: load)  
    - Index: "seo-writer"  
    - Namespace: "DataPlace"  
    - Credential: Pinecone API  
    - Connect When chat message received → Pinecone Vector Store (load).

20. **Embed chat input for retrieval:**  
    - Add Google Gemini Embeddings node  
    - Model: "models/embedding-001"  
    - Credential: Google Gemini API  
    - Connect chat input → Embeddings → Pinecone Vector Store (load).

21. **Setup memory buffer for session:**  
    - Add Memory Buffer Window node  
    - SessionKey: Extract sessionId from chat input JSON  
    - Connect to AI Agent1.

22. **Setup AI Agent for blog generation:**  
    - Add Langchain Agent node "AI Agent1"  
    - Use system message to instruct usage of context documents for answering.  
    - Connect Pinecone Vector Store3 and Window Buffer Memory → AI Agent1.

23. **Add Google Gemini Chat Model for AI Agent:**  
    - Model: "models/gemini-1.5-flash"  
    - Credential: Google Gemini API  
    - Connect Google Gemini Chat Model3 → AI Agent1.

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                     |
|------------------------------------------------------------------------------|----------------------------------------------------|
| Workflow uses Google Gemini (PaLM) API for embeddings and chat models.       | Google Gemini API credentials required              |
| Scrapeless API used for crawling blogs and scraping SERP data.               | Scrapeless API credentials required                  |
| Pinecone vector DB used for semantic storage and retrieval of blog content.  | Pinecone API credentials required                     |
| Sticky notes visually separate workflow blocks: Crawling, Vector Store, SERP, Blog Generation. | Sticky Note nodes in workflow                        |
| HTML node includes custom styling with embedded Inter font and responsive design. | For generating user-friendly report summaries      |
| JavaScript code carefully extracts markdown headers and links, handling edge cases. | Parsing blog content robustly for knowledge base    |
| The AI Agent uses session memory keyed on chat sessionId for context continuity. | Important for chat-based blog generation             |

---

**Disclaimer:** The text provided is derived exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.