Automatically Enrich Salesforce Accounts with Web Crawling, LinkedIn Data and GPT-4o

https://n8nworkflows.xyz/workflows/automatically-enrich-salesforce-accounts-with-web-crawling--linkedin-data-and-gpt-4o-6586


# Automatically Enrich Salesforce Accounts with Web Crawling, LinkedIn Data and GPT-4o

### 1. Workflow Overview

This workflow, titled **"Automatically Enrich Salesforce Accounts with Web Crawling, LinkedIn Data and GPT-4o"**, is designed to automatically enhance Salesforce Account records by integrating web crawling, LinkedIn scraping, and AI-driven data enrichment. Upon creation of a new Salesforce Account, it initiates a process to gather additional insights from external web sources, primarily LinkedIn and related web pages, and then employs GPT-4o via OpenAI to generate enriched, actionable insights which are fed back into Salesforce.

The workflow is logically divided into the following blocks:

- **1.1 Salesforce Trigger and Data Initialization:** Detects new Salesforce Account creation and retrieves account details.
- **1.2 Crawl Initialization and Root Setup:** Prepares parameters and global variables for web crawling starting from the account’s domain.
- **1.3 LinkedIn URL Handling:** Determines if a LinkedIn URL is available for the account; if missing, performs a DuckDuckGo search to find it.
- **1.4 LinkedIn Page Fetching and Parsing:** Fetches LinkedIn company pages, extracts HTML content, and parses company information.
- **1.5 Web Crawling Logic:** Implements recursive web crawling to fetch pages, extract links, deduplicate URLs, and control crawl depth.
- **1.6 Data Aggregation and LLM Preparation:** Aggregates crawled pages and LinkedIn data, chunks content, and builds prompts for language model processing.
- **1.7 AI Insight Generation and Salesforce Update:** Sends the prompt to OpenAI GPT-4o to generate insights and updates Salesforce Account records with enriched data.

---

### 2. Block-by-Block Analysis

#### 2.1 Salesforce Trigger and Data Initialization

- **Overview:** This block listens for new Salesforce Account creation events and fetches the full account data to start enrichment.
- **Nodes Involved:**  
  - SF: On Account Created  
  - SF: Get Account  
  - Init Crawl Params  
  - Init Globals  
  - Prep LinkedIn Search Inputs

- **Node Details:**

  - **SF: On Account Created**  
    - Type: Salesforce Trigger  
    - Role: Starts workflow when a Salesforce Account is created.  
    - Configuration: Uses default Salesforce credentials and listens to Account object insertions.  
    - Input: Salesforce events  
    - Output: Account record ID for next node  
    - Failure: Auth errors or connectivity issues with Salesforce may occur.

  - **SF: Get Account**  
    - Type: Salesforce node  
    - Role: Retrieves detailed account data based on triggered event.  
    - Configuration: Queries Account object with ID from the trigger.  
    - Input: Output from trigger node  
    - Output: Full account data for crawl initialization and LinkedIn processing.

  - **Init Crawl Params**  
    - Type: Set node  
    - Role: Defines parameters for web crawling such as root URL, domain, max crawl depth.  
    - Configuration: Sets static or dynamic values extracted from Account data (e.g., website URL).  
    - Output: Crawl parameters for global initialization.

  - **Init Globals**  
    - Type: Code node  
    - Role: Initializes static data variable to track the number of pending crawl requests (for managing asynchronous crawl completion).  
    - Key Logic: Sets `pendingCount` to zero or initial value in static data.  
    - Output: Passes control to the root crawl seeding.

  - **Prep LinkedIn Search Inputs**  
    - Type: Set node  
    - Role: Prepares data required for LinkedIn URL lookup (company name, domain, existing URL).  
    - Output: Passes data to decision node checking LinkedIn URL presence.

#### 2.2 Crawl Initialization and Root Setup

- **Overview:** Seeds the crawl queue with the root URL and domain; sets up iterative crawling logic.
- **Nodes Involved:**  
  - Seed Root Crawl Item  
  - Fetch HTML Page  
  - Attach URL/Depth to HTML  
  - Extract Body & Links  
  - Queue & Dedup Links  
  - IF Crawl Depth OK?  
  - Requeue Link Item  
  - Loop Links (Batches)  
  - Store Page Data  
  - Collect Pages & Emit When Done

- **Node Details:**

  - **Seed Root Crawl Item**  
    - Type: Merge node  
    - Role: Starts the crawl queue with the initial URL and depth 0.  
    - Config: Merges crawl params and globals to create the first crawl item.

  - **Fetch HTML Page**  
    - Type: HTTP Request  
    - Role: Fetches the HTML content of the current URL to crawl.  
    - Config: Standard GET request, error handling continues on failure to avoid halting crawl.  
    - Output: Raw HTML page content.

  - **Attach URL/Depth to HTML**  
    - Type: Code node  
    - Role: Adds metadata (URL and depth) to the fetched HTML content for tracking.  

  - **Extract Body & Links**  
    - Type: HTML node  
    - Role: Parses the HTML content to extract visible body text and all anchor href links.  
    - Output: Body text + link array for crawl queueing.

  - **Queue & Dedup Links**  
    - Type: Code node  
    - Role: Cleans extracted URLs, removes duplicates, filters out visited URLs, and prepares new crawl queue entries.  
    - Error Handling: Continues on failure to avoid crawl disruption.

  - **IF Crawl Depth OK?**  
    - Type: If node  
    - Role: Checks if the current crawl depth is less than max allowed depth to decide further crawling or storage.  
    - Output: Branches to either requeue the link for crawling or store page data.

  - **Requeue Link Item**  
    - Type: Code node  
    - Role: Prepares the link item for re-enqueueing in the crawl queue, removing internal fields.  

  - **Loop Links (Batches)**  
    - Type: Split In Batches  
    - Role: Iterates over the queue of URLs to crawl one by one, enabling controlled crawling flow.  

  - **Store Page Data**  
    - Type: Set node  
    - Role: Stores the URL, HTML content, and depth for downstream aggregation.

  - **Collect Pages & Emit When Done**  
    - Type: Code node  
    - Role: Tracks crawl completion by counting pending requests and emits the aggregated pages once done.

#### 2.3 LinkedIn URL Handling

- **Overview:** Checks if a LinkedIn URL is provided; if not, performs a DuckDuckGo search to find the LinkedIn company page URL.
- **Nodes Involved:**  
  - IF LInkedin URL Missing?  
  - DDG Search (LinkedIn)  
  - Parse LinkedIn URL  
  - Use Provided URL (Fallback)  
  - Merge LinkedIn URL Branches

- **Node Details:**

  - **IF LInkedin URL Missing?**  
    - Type: If node  
    - Role: Checks if the Account has a LinkedIn URL field populated.  
    - Branch:  
      - True: Executes DDG Search node.  
      - False: Uses the provided URL fallback.

  - **DDG Search (LinkedIn)**  
    - Type: HTTP Request  
    - Role: Performs DuckDuckGo search to find LinkedIn company URL related to the account.  
    - Config: Constructs DDG query with company name and domain.

  - **Parse LinkedIn URL**  
    - Type: Code node  
    - Role: Extracts LinkedIn company URL from DDG search result JSON.  
    - Output: Passes extracted URL to merge node.

  - **Use Provided URL (Fallback)**  
    - Type: Code node  
    - Role: Uses the LinkedIn URL already available from Salesforce if present.

  - **Merge LinkedIn URL Branches**  
    - Type: Merge node  
    - Role: Combines outputs from DDG search and fallback URL nodes to continue with LinkedIn page fetching.

#### 2.4 LinkedIn Page Fetching and Parsing

- **Overview:** Fetches the LinkedIn company page, extracts the body HTML, and parses company info for enrichment.
- **Nodes Involved:**  
  - Fetch LinkedIn HTML1  
  - Extract LinkedIn Body  
  - Parse LinkedIn Company Info

- **Node Details:**

  - **Fetch LinkedIn HTML1**  
    - Type: HTTP Request  
    - Role: Fetches the LinkedIn company page HTML content.  
    - Config: GET request to LinkedIn URL.

  - **Extract LinkedIn Body**  
    - Type: HTML node  
    - Role: Extracts the main body or relevant section from LinkedIn HTML for parsing.

  - **Parse LinkedIn Company Info**  
    - Type: Code node  
    - Role: Parses extracted HTML to retrieve structured company info (e.g., size, industry, description).

#### 2.5 Data Aggregation and LLM Preparation

- **Overview:** Merges crawled web pages and LinkedIn JSON data, combines and chunks information, and builds a prompt for the language model.
- **Nodes Involved:**  
  - Merge Web Pages + LinkedIn JSON  
  - Combine & Chunk for LLM  
  - Build LLM Prompt

- **Node Details:**

  - **Merge Web Pages + LinkedIn JSON**  
    - Type: Merge node  
    - Role: Combines outputs of web crawl page data and LinkedIn company info JSON for unified processing.

  - **Combine & Chunk for LLM**  
    - Type: Code node  
    - Role: Aggregates textual data and splits it into manageable chunks to optimize language model input.

  - **Build LLM Prompt**  
    - Type: Code node  
    - Role: Constructs a structured prompt containing all enriched data, instructions, and context for GPT-4o.

#### 2.6 AI Insight Generation and Salesforce Update

- **Overview:** Sends the prepared prompt to OpenAI's GPT-4o model to generate insights and updates the Salesforce Account record with the enriched data.
- **Nodes Involved:**  
  - OpenAI: Generate Insights  
  - Salesforce

- **Node Details:**

  - **OpenAI: Generate Insights**  
    - Type: OpenAI node (via LangChain integration)  
    - Role: Calls GPT-4o with the prompt to generate insights and recommendations.  
    - Configuration: Uses OpenAI credentials, specifies GPT-4o model, input prompt from previous node.

  - **Salesforce**  
    - Type: Salesforce node  
    - Role: Updates the Salesforce Account record with the AI-generated enriched insights.  
    - Configuration: Uses Salesforce credentials, updates Account using ID and fields from OpenAI output.

---

### 3. Summary Table

| Node Name                     | Node Type                           | Functional Role                                | Input Node(s)              | Output Node(s)              | Sticky Note                                                                                   |
|-------------------------------|-----------------------------------|-----------------------------------------------|----------------------------|-----------------------------|----------------------------------------------------------------------------------------------|
| SF: On Account Created         | Salesforce Trigger                | Trigger on new Salesforce Account creation    |                            | SF: Get Account             |                                                                                              |
| SF: Get Account               | Salesforce                        | Fetch Account details                          | SF: On Account Created      | Init Crawl Params, Prep LinkedIn Search Inputs |                                                                                              |
| Init Crawl Params             | Set                              | Define crawl root URL, domain, max depth      | SF: Get Account            | Init Globals                |                                                                                              |
| Init Globals                 | Code                             | Initialize pending crawl count in static data | Init Crawl Params           | Seed Root Crawl Item        |                                                                                              |
| Seed Root Crawl Item          | Merge                            | Start crawl queue with root URL and depth 0  | Init Globals               | Fetch HTML Page             |                                                                                              |
| Fetch HTML Page               | HTTP Request                     | Fetch HTML content of current URL              | Seed Root Crawl Item        | Attach URL/Depth to HTML    | Makes HTTP request to fetch the content of the current URL.                                 |
| Attach URL/Depth to HTML      | Code                             | Attach URL and depth metadata to HTML content | Fetch HTML Page             | Extract Body & Links        |                                                                                              |
| Extract Body & Links          | HTML                             | Parse HTML to extract body text and links      | Attach URL/Depth to HTML    | Queue & Dedup Links         | Parses HTML content and extracts body text and anchor href links.                           |
| Queue & Dedup Links           | Code                             | Clean, deduplicate and queue links for crawl   | Extract Body & Links        | IF Crawl Depth OK?          |                                                                                              |
| IF Crawl Depth OK?            | If                               | Check if current depth < max allowed             | Queue & Dedup Links         | Requeue Link Item (true), Store Page Data (false) | Validates whether the current depth is below the maximum depth allowed.                      |
| Requeue Link Item             | Code                             | Prepare link for re-enqueueing                   | IF Crawl Depth OK?          | Loop Links (Batches)        | Removes internal 'type' field and re-enqueues the link for next crawl.                      |
| Loop Links (Batches)          | Split In Batches                 | Iterate crawl queue items one at a time          | Requeue Link Item           | Seed Root Crawl Item (index 1) | Iterates through the queue of links to be crawled one at a time.                             |
| Store Page Data               | Set                              | Store URL, content, depth for aggregation       | IF Crawl Depth OK?          | Collect Pages & Emit When Done | Captures the URL, page content, and depth for storage or export.                            |
| Collect Pages & Emit When Done| Code                             | Track crawl completion and emit aggregated data | Store Page Data             | Merge Web Pages + LinkedIn JSON |                                                                                              |
| Prep LinkedIn Search Inputs   | Set                              | Prepare LinkedIn search data inputs              | SF: Get Account            | IF LInkedin URL Missing?    |                                                                                              |
| IF LInkedin URL Missing?      | If                               | Decide if LinkedIn URL is missing                 | Prep LinkedIn Search Inputs | DDG Search (true), Use Provided URL (false) |                                                                                              |
| DDG Search (LinkedIn)         | HTTP Request                     | Search LinkedIn URL using DuckDuckGo             | IF LInkedin URL Missing?    | Parse LinkedIn URL          |                                                                                              |
| Parse LinkedIn URL            | Code                             | Extract LinkedIn URL from DDG results             | DDG Search (LinkedIn)        | Merge LinkedIn URL Branches |                                                                                              |
| Use Provided URL (Fallback)   | Code                             | Use existing LinkedIn URL if available            | IF LInkedin URL Missing?    | Merge LinkedIn URL Branches |                                                                                              |
| Merge LinkedIn URL Branches   | Merge                            | Merge LinkedIn URL branches to proceed            | Parse LinkedIn URL, Use Provided URL | Fetch LinkedIn HTML1   |                                                                                              |
| Fetch LinkedIn HTML1          | HTTP Request                     | Fetch LinkedIn company page HTML                   | Merge LinkedIn URL Branches | Extract LinkedIn Body       |                                                                                              |
| Extract LinkedIn Body         | HTML                             | Extract LinkedIn page body content                 | Fetch LinkedIn HTML1        | Parse LinkedIn Company Info |                                                                                              |
| Parse LinkedIn Company Info   | Code                             | Parse LinkedIn HTML to extract company info       | Extract LinkedIn Body       | Merge Web Pages + LinkedIn JSON |                                                                                              |
| Merge Web Pages + LinkedIn JSON| Merge                           | Merge web crawl and LinkedIn data                  | Collect Pages & Emit When Done, Parse LinkedIn Company Info | Combine & Chunk for LLM |                                                                                              |
| Combine & Chunk for LLM       | Code                             | Combine and chunk data for language model input   | Merge Web Pages + LinkedIn JSON | Build LLM Prompt          |                                                                                              |
| Build LLM Prompt             | Code                             | Build prompt for GPT-4o from aggregated data      | Combine & Chunk for LLM     | OpenAI: Generate Insights   |                                                                                              |
| OpenAI: Generate Insights     | OpenAI (LangChain)               | Generate insights using GPT-4o                      | Build LLM Prompt            | Salesforce                  |                                                                                              |
| Salesforce                   | Salesforce                       | Update Salesforce Account with AI-enriched insights| OpenAI: Generate Insights   |                             |                                                                                              |
| Sticky Note                  | Sticky Note                     | (Empty content)                                     |                            |                             |                                                                                              |
| Sticky Note1                 | Sticky Note                     | (Empty content)                                     |                            |                             |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Salesforce Trigger Node:**  
   - Type: Salesforce Trigger  
   - Configure to trigger on Account creation events using connected Salesforce OAuth2 credentials.

2. **Add Salesforce Node to Get Account:**  
   - Type: Salesforce  
   - Use Account ID from trigger to query full Account details.

3. **Add Set Node "Init Crawl Params":**  
   - Define parameters: root URL (e.g., Account Website), domain (extract domain from URL), max crawl depth (e.g., 3).  
   - Use expressions to extract domain from website URL.

4. **Add Code Node "Init Globals":**  
   - Initialize static data variable `pendingCount` to 0 for crawl tracking.

5. **Add Merge Node "Seed Root Crawl Item":**  
   - Merge crawl params and globals outputs to form initial crawl queue item with URL and depth 0.

6. **Add HTTP Request Node "Fetch HTML Page":**  
   - Configure GET request to URL from crawl item.  
   - Set error handling to continue on failure.

7. **Add Code Node "Attach URL/Depth to HTML":**  
   - Append URL and depth metadata to response for reference.

8. **Add HTML Node "Extract Body & Links":**  
   - Extract document body text and all `<a href>` links from HTML content.

9. **Add Code Node "Queue & Dedup Links":**  
   - Clean extracted links, remove duplicates, filter visited URLs using static data, prepare new crawl queue items.  
   - Continue on error to prevent crawl disruption.

10. **Add If Node "IF Crawl Depth OK?":**  
    - Check if current crawl depth < max depth parameter.  
    - If true, proceed to requeue link; else, store page data.

11. **Add Code Node "Requeue Link Item":**  
    - Remove internal fields, prepare link for re-enqueue.

12. **Add SplitInBatches Node "Loop Links (Batches)":**  
    - Loop over queued URLs one at a time for controlled crawling.

13. **Add Set Node "Store Page Data":**  
    - Store URL, HTML content, and depth for aggregation.

14. **Add Code Node "Collect Pages & Emit When Done":**  
    - Track `pendingCount` in static data, emit all collected pages when count reaches zero.

15. **Add Set Node "Prep LinkedIn Search Inputs":**  
    - Prepare LinkedIn search data such as company name, domain, and LinkedIn URL from Account data.

16. **Add If Node "IF LInkedin URL Missing?":**  
    - Check if LinkedIn URL field is empty or missing.

17. **Add HTTP Request Node "DDG Search (LinkedIn)":**  
    - Perform DuckDuckGo search with query combining company name and LinkedIn keywords.

18. **Add Code Node "Parse LinkedIn URL":**  
    - Extract LinkedIn company URL from DDG search JSON results.

19. **Add Code Node "Use Provided URL (Fallback)":**  
    - Use existing LinkedIn URL if present.

20. **Add Merge Node "Merge LinkedIn URL Branches":**  
    - Merge outputs of DDG search and provided URL branches.

21. **Add HTTP Request Node "Fetch LinkedIn HTML1":**  
    - Fetch LinkedIn company page HTML.

22. **Add HTML Node "Extract LinkedIn Body":**  
    - Extract main body content from LinkedIn HTML.

23. **Add Code Node "Parse LinkedIn Company Info":**  
    - Parse relevant company details (industry, size, description) from extracted HTML.

24. **Add Merge Node "Merge Web Pages + LinkedIn JSON":**  
    - Merge collected web crawl pages and LinkedIn data.

25. **Add Code Node "Combine & Chunk for LLM":**  
    - Aggregate and chunk textual data to fit language model input constraints.

26. **Add Code Node "Build LLM Prompt":**  
    - Build custom prompt for GPT-4o with structured data and instructions.

27. **Add OpenAI Node "OpenAI: Generate Insights":**  
    - Use OpenAI GPT-4o model credentials.  
    - Pass prompt from previous node.

28. **Add Salesforce Node "Salesforce":**  
    - Update Account record with AI-generated insights.  
    - Use Account ID and fields from OpenAI output.

29. **Connect nodes in order following above steps, ensuring proper data passing and error handling.**

30. **Configure all credentials: Salesforce OAuth2, OpenAI API key.**

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                                                                                   |
|-----------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| The workflow includes robust asynchronous crawl management using static data to track pending pages.      | Ensures the workflow knows when the entire crawl is complete before proceeding to aggregation and AI processing.                                                |
| DuckDuckGo is used as a fallback search engine to find LinkedIn URLs when missing in Salesforce data.     | Avoids reliance on LinkedIn API; uses public search results to discover company pages.                                                                           |
| GPT-4o via OpenAI LangChain integration is leveraged for advanced text understanding and insight creation.| Requires OpenAI API key with access to GPT-4o model.                                                                                                            |
| The workflow gracefully handles HTTP request errors during page fetch to avoid halting the crawl process.  | HTTP Request nodes are configured with "continue on error" to maximize crawl coverage.                                                                           |
| For more details on n8n Salesforce integration: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.salesforce/ | Official Salesforce node documentation                                                                                                                          |
| For OpenAI GPT-4o usage in n8n LangChain node: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-langchain.openAi/ | Official OpenAI LangChain node guide                                                                                                                            |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, a no-code automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.