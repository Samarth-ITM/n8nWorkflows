Creating SEO-Optimized Blogs for WordPress Using Specific Tools

https://n8nworkflows.xyz/workflows/creating-seo-optimized-blogs-for-wordpress-using-specific-tools-8394


# Creating SEO-Optimized Blogs for WordPress Using Specific Tools

### 1. Workflow Overview

This workflow automates the creation of SEO-optimized blog posts for WordPress using a variety of AI models, data extraction tools, and database interactions. It is designed to fetch keyword data from a Postgres database, generate structured blog content through AI language models, manage blog post metadata, create posts on WordPress, and submit updated sitemaps for SEO purposes.

The workflow is logically divided into the following blocks:

- **1.1 Scheduling and Input Data Acquisition:** Initiates the workflow on a schedule, fetches keyword and blog metadata from databases and GitHub repositories.
- **1.2 AI Planning and Content Generation:** Uses multiple AI models (OpenAI, OpenRouter, Perplexity) for planning, drafting, and structuring blog content including titles, introductions, development sections, conclusions, and FAQs.
- **1.3 Data Processing and Keyword Handling:** Processes and scores keywords, filters and limits data sets, aggregates and prepares data for AI input.
- **1.4 Blog Post Creation and Publishing:** Generates blog posts on WordPress, updates internal databases for link management, and manages approval notifications via Slack.
- **1.5 Sitemap Management and SEO Submission:** Extracts and updates sitemap files from GitHub, submits the sitemap to search engines to notify updated content.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduling and Input Data Acquisition

**Overview:**  
This block triggers the workflow periodically, fetches keywords and blog metadata from a Postgres database and GitHub files, and prepares data for AI processing.

**Nodes Involved:**  
- Schedule Trigger  
- Schedule Trigger1  
- Fetch Keywords1 (Postgres)  
- select blog table (Postgres)  
- limit to 1 (Limit)  
- get articles.json1 (GitHub)  
- get sitemap1 (GitHub)  
- Extract from File2, Extract from File3 (File Extraction)  
- Edit Fields (multiple Set nodes for data formatting)

**Node Details:**  

- **Schedule Trigger & Schedule Trigger1**  
  - Type: Trigger, schedules workflow execution (e.g., daily/weekly)  
  - Configured to initiate keyword fetch and blog metadata retrieval  
  - Possible failure: scheduling misconfiguration, trigger not firing

- **Fetch Keywords1**  
  - Type: Postgres node to fetch keywords from database  
  - Key SQL query likely fetching primary and secondary keywords for blog topics  
  - Requires configured Postgres credentials  
  - Edge cases: DB connection failure, empty keyword sets

- **select blog table & limit to 1**  
  - Postgres node fetching latest blog metadata with limit to one record  
  - Ensures selection of a single blog entry for processing  
  - Edge cases: empty blog table or stale data

- **get articles.json1 & get sitemap1**  
  - GitHub nodes fetching JSON files for articles and sitemap  
  - Authenticated requests using GitHub credentials  
  - Potential failures: missing files, auth errors, API rate limits

- **Extract from File2 & Extract from File3**  
  - Extract contents from retrieved GitHub files for further processing  
  - Input: raw file data from GitHub nodes  
  - Output: parsed JSON or structured data  
  - Edge cases: file format errors, extraction failures

- **Edit Fields Nodes**  
  - Multiple Set nodes to clean, transform and prepare data for downstream nodes  
  - Used extensively for renaming fields, setting defaults, or formatting  
  - Possible expression errors if fields missing or malformed

---

#### 1.2 AI Planning and Content Generation

**Overview:**  
This block leverages multiple AI language models and agents to create a structured blog post plan, generate content sections, and output a structured format ready for publishing.

**Nodes Involved:**  
- Preliminary Plan (OpenAI)  
- Message a model (Perplexity AI)  
- Structured Output Parser, Structured Output Parser1 (LangChain Parsers)  
- Choose keywords, Intent, etc (LangChain Agent)  
- create plan for dev (LangChain Chain LLM)  
- Intro, dev 1, dev 2, conclusion, FAQ section, header post (LangChain Chain LLM nodes)  
- Multiple OpenAI Chat Model and OpenRouter Chat Model nodes  
- Merge, Merge2, Merge3 (Merge nodes to aggregate AI outputs)  
- Edit Fields nodes post AI generation for formatting

**Node Details:**  

- **Preliminary Plan (OpenAI)**  
  - Generates initial blog post outline or plan based on inputs  
  - Uses OpenAI credentials and model (likely GPT-4 or similar)  
  - Output feeds into Perplexity and subsequent AI nodes  
  - Failures: API quota, rate limits, malformed prompts

- **Message a model (Perplexity)**  
  - Queries Perplexity AI for additional content or validation  
  - Parallel path to OpenAI for content enrichment  
  - Must handle API limits and connection errors

- **Structured Output Parser(s)**  
  - Parse AI-generated structured text or JSON back into usable data format for n8n  
  - Critical for downstream nodes expecting structured input  
  - Failures: parsing errors from unexpected AI output

- **Choose keywords, Intent, etc (LangChain Agent)**  
  - AI agent selects and scores keywords, defines content intent and focus  
  - Integrates with Postgres to insert and select keyword data  
  - Handles branching logic in content creation

- **create plan for dev & content section nodes (Intro, dev 1, dev 2, conclusion, FAQ section, header post)**  
  - Generate specific blog content sections using chained LLM calls  
  - Sequential and parallel AI calls to build full blog narrative  
  - Merge nodes combine outputs for final assembly  
  - Failures: incomplete outputs, API errors, rate limits

- **OpenAI Chat Model and OpenRouter Chat Model nodes**  
  - Different AI model integrations providing variety and fallback options  
  - Configured with appropriate credentials and parameters  
  - Must be monitored for API quota, latency, and output consistency

---

#### 1.3 Data Processing and Keyword Handling

**Overview:**  
This block processes keyword data fetched from databases, scores keywords, filters by quality and size, aggregates results, and prepares data for AI input and database updates.

**Nodes Involved:**  
- Score Keywords (Code node)  
- Update score (Postgres)  
- Loop Over Items (SplitInBatches)  
- Filter, Filter1  
- Limit, Limit1, Limit2, Limit3 (Limit nodes)  
- Aggregate, Aggregate2, Aggregate3, Aggregate4, Aggregate5 (Aggregate nodes)  
- Split Out, Split Out1, Split Out2, Split Out3 (Split nodes)  
- Edit Fields nodes for data mutation

**Node Details:**  

- **Score Keywords (Code node)**  
  - Custom JavaScript logic to evaluate keyword relevance or quality  
  - May calculate keyword density, search volume, or other SEO metrics  
  - Edge cases: invalid input data, division by zero, empty arrays

- **Update score (Postgres)**  
  - Updates database with new keyword scores  
  - Requires transactional handling to avoid data corruption  
  - Failures: DB write errors, concurrency conflicts

- **Loop Over Items (SplitInBatches)**  
  - Processes large keyword sets in manageable batches for scoring and updates  
  - Prevents API or DB timeouts  
  - Edge case: batch size too large, incomplete batch processing

- **Filter & Filter1**  
  - Conditions to exclude low-quality or irrelevant keywords/posts  
  - Uses expressions and field checks  
  - Possible expression evaluation errors

- **Limit nodes**  
  - Restrict number of items passed downstream to avoid overload or API limits  
  - Critical in controlling workflow scale

- **Aggregate nodes**  
  - Combine data items from multiple branches or outputs into single datasets  
  - Used to assemble final inputs for AI or DB insertion

- **Split Out nodes**  
  - Split arrays or grouped data into individual items for granular processing

- **Edit Fields nodes**  
  - Prepare and format data between processing steps

---

#### 1.4 Blog Post Creation and Publishing

**Overview:**  
This block manages the creation of WordPress blog posts, updates internal logging databases, and sends Slack notifications for editorial approval.

**Nodes Involved:**  
- Create blog (GitHub node)  
- Create a post1 (WordPress node)  
- Log blog link for future internal links (Postgres)  
- wait approval (Slack node)  
- set sitemap variables (Set node)  
- Sticky Notes (likely for documentation)

**Node Details:**  

- **Create blog (GitHub)**  
  - Commits or updates blog content files in a GitHub repository  
  - Uses GitHub credentials and webhook triggers  
  - Failure points: API rate limits, commit conflicts

- **Create a post1 (WordPress)**  
  - Creates or updates a WordPress blog post with generated content  
  - Requires WordPress credentials (OAuth2 or API key)  
  - Handles post metadata such as SEO tags, featured images, categories  
  - Failures: auth errors, WordPress API errors, content validation

- **Log blog link for future internal links (Postgres)**  
  - Inserts or updates blog URL and metadata in a Postgres database for internal linking strategies  
  - Ensures future blog posts can reference this content

- **wait approval (Slack)**  
  - Sends message to Slack channel to request editorial approval  
  - Uses Slack webhook or OAuth credentials  
  - Possible failures: webhook misconfiguration, network issues

- **set sitemap variables (Set node)**  
  - Prepares variables needed for sitemap submission

---

#### 1.5 Sitemap Management and SEO Submission

**Overview:**  
This block extracts sitemap files from GitHub, updates them as needed, and submits the sitemap URL to search engines for SEO indexing.

**Nodes Involved:**  
- get sitemap1 (GitHub)  
- Extract from File2 (File Extraction)  
- Edit Fields8 (Set)  
- Edit a file1 (GitHub)  
- submit sitemap (HTTP Request)  
- Sticky Notes

**Node Details:**  

- **get sitemap1 & Extract from File2**  
  - Retrieve the sitemap XML or JSON file from GitHub and extract content  
  - Requires GitHub API credentials and proper file path

- **Edit Fields8**  
  - Prepares and formats sitemap data for updating or submission

- **Edit a file1 (GitHub)**  
  - Commits updated sitemap back to GitHub repository  
  - Handles version control and potential merge conflicts

- **submit sitemap (HTTP Request)**  
  - Sends HTTP POST or GET request to search engine API endpoints (e.g., Google Search Console) to submit sitemap URL  
  - Parameters include sitemap URL and API keys if necessary  
  - Possible failures: API auth errors, invalid sitemap URL, rate limits

---

### 3. Summary Table

| Node Name                     | Node Type                              | Functional Role                               | Input Node(s)                      | Output Node(s)                  | Sticky Note                                                                                          |
|-------------------------------|--------------------------------------|-----------------------------------------------|----------------------------------|--------------------------------|----------------------------------------------------------------------------------------------------|
| Schedule Trigger              | scheduleTrigger                      | Workflow start trigger for keyword fetch      | –                                | Fetch Keywords1                 |                                                                                                    |
| Fetch Keywords1               | postgres                            | Fetch keywords from Postgres                   | Schedule Trigger                 | Limit2                         |                                                                                                    |
| Limit2                       | limit                              | Limit keyword rows                             | Fetch Keywords1                  | Edit Fields3                   |                                                                                                    |
| Edit Fields3                 | set                                | Prepare keyword data                           | Limit2                         | Aggregate2                    |                                                                                                    |
| Aggregate2                   | aggregate                          | Aggregate keywords                            | Edit Fields3                   | Choose keywords, Intent, etc   |                                                                                                    |
| Choose keywords, Intent, etc | @n8n/n8n-nodes-langchain.agent    | AI agent to select keywords and intent         | Aggregate2                    | Insert rows in a table, Select rows from a table1, Split Out2 |                                                                                                    |
| Insert rows in a table       | postgres                           | Insert keywords into DB                          | Choose keywords, Intent, etc    | Select rows from a table1      |                                                                                                    |
| Select rows from a table1    | postgres                           | Select keywords for scoring                     | Insert rows in a table           | Code5                         |                                                                                                    |
| Code5                       | code                              | Custom code for keyword processing             | Select rows from a table1        | Primary keywords               |                                                                                                    |
| Primary keywords             | postgres                           | Fetch primary keywords                         | Code5                         | Select rows from a table2      |                                                                                                    |
| Select rows from a table2    | postgres                           | Fetch secondary keywords                       | Split Out2                    | Code4                         |                                                                                                    |
| Code4                       | code                              | Process secondary keywords                      | Select rows from a table2        | Used secondary                |                                                                                                    |
| Used secondary              | postgres                           | Fetch used secondary keywords                   | Code4                         | –                             |                                                                                                    |
| Score Keywords              | code                              | Score keywords                                  | Loop Over Items                | Update score                  |                                                                                                    |
| Update score                | postgres                           | Update keyword scores in DB                      | Score Keywords                 | Loop Over Items               |                                                                                                    |
| Loop Over Items             | splitInBatches                    | Batch process keywords                          | Select rows from a table        | All scored, Score Keywords     |                                                                                                    |
| All scored                  | noOp                              | Sync point after scoring                        | Loop Over Items                | –                             |                                                                                                    |
| Preliminary Plan            | @n8n/n8n-nodes-langchain.openAi   | Generate blog post plan                         | limit to 1                    | Message a model               |                                                                                                    |
| limit to 1                  | limit                              | Limit blog table entries to 1                   | select blog table              | Preliminary Plan              |                                                                                                    |
| select blog table           | postgres                           | Fetch blog metadata                             | Schedule Trigger1              | limit to 1                   |                                                                                                    |
| Message a model             | perplexity                        | Query Perplexity AI for content suggestions     | Preliminary Plan              | Split Out1, Split Out3, Edit Fields7, Edit Fields19, blog table |                                                                                                    |
| Split Out1                  | splitOut                          | Split AI response for processing                | Message a model               | Quality?                     |                                                                                                    |
| Quality?                   | code                              | Assess content quality                           | Split Out1                   | size?                        |                                                                                                    |
| size?                      | code                              | Assess content size                              | Quality?                     | Filter1                      |                                                                                                    |
| Filter1                    | filter                            | Filter based on conditions                       | size?                        | Limit1                       |                                                                                                    |
| Limit1                     | limit                              | Limit filtered items                             | Filter1                      | Edit Fields5                 |                                                                                                    |
| Edit Fields5               | set                                | Format data for merging                          | Limit1                       | Merge                        |                                                                                                    |
| Merge                      | merge                             | Merge multiple AI content branches               | Edit Fields5, Aggregate3, Aggregate | Aggregate4                  |                                                                                                    |
| Aggregate3                 | aggregate                          | Aggregate data                                  | Edit Fields6                 | Merge                        |                                                                                                    |
| Edit Fields6               | set                                | Prepare aggregated data                          | Split Out3                   | Aggregate3                   |                                                                                                    |
| Aggregate                 | aggregate                          | Aggregate filtered data                          | Edit Fields18                | Merge                        |                                                                                                    |
| Edit Fields18              | set                                | Prepare data after blog table                    | blog table                   | Aggregate                    |                                                                                                    |
| Aggregate4                 | aggregate                          | Aggregate content sections                       | Merge                        | Intro, dev 1, dev 2, conclusion, header post, FAQ section |                                                                                                    |
| Intro                      | @n8n/n8n-nodes-langchain.chainLlm | Generate introduction section                    | Aggregate4                   | Edit Fields9                 |                                                                                                    |
| dev 1                      | @n8n/n8n-nodes-langchain.chainLlm | Generate development section 1                   | Aggregate4                   | Edit Fields10                |                                                                                                    |
| dev 2                      | @n8n/n8n-nodes-langchain.chainLlm | Generate development section 2                   | Aggregate4                   | Edit Fields11                |                                                                                                    |
| conclusion                 | @n8n/n8n-nodes-langchain.chainLlm | Generate conclusion section                       | Aggregate4                   | Edit Fields12                |                                                                                                    |
| header post                | @n8n/n8n-nodes-langchain.chainLlm | Generate header post section                      | Aggregate4                   | Merge2                      |                                                                                                    |
| FAQ section                | @n8n/n8n-nodes-langchain.chainLlm | Generate FAQ section                              | Aggregate4                   | Edit Fields20                |                                                                                                    |
| Merge2                     | merge                             | Merge all content sections                        | Edit Fields9, Edit Fields10, Edit Fields11, Edit Fields12, Edit Fields20, Edit Fields13 | Aggregate5                  |                                                                                                    |
| Aggregate5                 | aggregate                          | Aggregate merged sections                         | Merge2                      | Edit Fields                  |                                                                                                    |
| Edit Fields                | set                                | Prepare final content                             | Aggregate5                  | Code6                       |                                                                                                    |
| Code6                      | code                              | Final content processing                          | Edit Fields                 | get articles.json1, Edit Fields13, Edit Fields2 |                                                                                                    |
| get articles.json1          | github                            | Fetch articles JSON from GitHub                   | Code6                       | Extract from File3           |                                                                                                    |
| Extract from File3          | extractFromFile                   | Extract content from articles JSON                 | get articles.json1           | Edit Fields1                 |                                                                                                    |
| Edit Fields1               | set                                | Prepare article data                              | Extract from File3           | Create blog, Edit Fields2    |                                                                                                    |
| Create blog                | github                            | Commit blog content to GitHub                      | Edit Fields2                | Merge3                      |                                                                                                    |
| Merge3                     | merge                             | Merge outputs from blog creation                   | Create blog, Edit a file, Edit a file1 | Limit3                      |                                                                                                    |
| Edit a file                | github                            | Update file in GitHub                              | Edit Fields15               | Merge3                      |                                                                                                    |
| Edit a file1               | github                            | Update another file in GitHub                      | Code9                       | Merge3                      |                                                                                                    |
| Code9                      | code                              | Custom logic before final commit                   | Edit Fields8               | Edit a file1                |                                                                                                    |
| Limit3                     | limit                              | Limit number of blog creation outputs              | Merge3                      | Edit Fields16, Edit Fields17 |                                                                                                    |
| Edit Fields16              | set                                | Prepare Slack approval notification                 | Limit3                      | wait approval               |                                                                                                    |
| wait approval              | slack                             | Send Slack message for editorial approval           | Edit Fields16               | set sitemap variables       |                                                                                                    |
| set sitemap variables      | set                                | Prepare variables for sitemap submission             | wait approval               | submit sitemap              |                                                                                                    |
| submit sitemap             | httpRequest                      | Submit sitemap to search engines                     | set sitemap variables       | –                           |                                                                                                    |
| Create a post1             | wordpress                        | Create WordPress blog post                           | Edit Fields2                | set sitemap variables, Edit Fields17 |                                                                                                    |
| Edit Fields17              | set                                | Update DB after post creation                         | Create a post1              | Log blog link for future internal links |                                                                                                    |
| Log blog link for future internal links | postgres                          | Insert blog link metadata for internal linking      | Edit Fields17              | –                           |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Triggers:**  
   - Add two `Schedule Trigger` nodes to initiate workflow periodically for keyword fetching and blog metadata fetching.

2. **Add Postgres Nodes to Fetch Data:**  
   - Configure `Fetch Keywords1` to query keywords from the Postgres database.  
   - Configure `select blog table` to fetch blog metadata; add a `Limit` node (limit to 1) to restrict to single entry.

3. **Add GitHub Nodes for Data Fetch:**  
   - Add `get articles.json1` and `get sitemap1` to fetch blog articles and sitemap files from GitHub. Authenticate with GitHub OAuth.

4. **Extract Data from Files:**  
   - Add `Extract from File2` and `Extract from File3` to parse content from GitHub files.

5. **Add Edit Fields Nodes:**  
   - Insert multiple `Set` nodes to rename, format, and prepare data for AI and DB nodes.

6. **Add AI Nodes for Planning and Content Generation:**  
   - Add `Preliminary Plan` (OpenAI) node to generate blog outline.  
   - Connect to `Message a model` (Perplexity) for enrichment.  
   - Add `Structured Output Parser` nodes to parse AI responses.  
   - Add `Choose keywords, Intent, etc` (LangChain Agent) node for keyword selection and intent definition.  
   - Add `create plan for dev` and content generation nodes (`Intro`, `dev 1`, `dev 2`, `conclusion`, `FAQ section`, `header post`) configured with OpenAI or OpenRouter Chat models as appropriate.  
   - Use `Merge` nodes to combine content sections.

7. **Add Data Processing Nodes:**  
   - Insert `Score Keywords` (Code node) to score keywords.  
   - Add `Update score` (Postgres) to update DB scores.  
   - Use `SplitInBatches` (`Loop Over Items`) to batch process keywords.  
   - Add filtering (`Filter`, `Filter1`) and limiting (`Limit`, `Limit1`, `Limit2`, `Limit3`) nodes to control data flow.

8. **Add Blog Post Creation Nodes:**  
   - Insert `Create blog` (GitHub) to commit blog content files.  
   - Add `Create a post1` (WordPress) node to publish blog posts on WordPress platform. Configure with WordPress credentials.  
   - Add `Log blog link for future internal links` (Postgres) to store blog URLs.  
   - Add `wait approval` (Slack) node to notify editorial team.

9. **Add Sitemap Management Nodes:**  
   - Add `Edit a file1` (GitHub) to update sitemap files.  
   - Add `submit sitemap` (HTTP Request) to notify search engines of sitemap changes.  
   - Connect with a `Set` node (`set sitemap variables`) for URL preparation.

10. **Connect Workflow:**  
    - Connect all nodes respecting the data flow described in the analysis, ensuring outputs feed into appropriate inputs as per block dependencies.

11. **Configure Credentials:**  
    - Configure Postgres credentials with database access rights.  
    - Set up GitHub OAuth credentials with repo read/write scopes.  
    - Set up OpenAI API credentials for LLM usage.  
    - Configure Perplexity AI credentials if applicable.  
    - Add WordPress API credentials (OAuth2 or basic auth).  
    - Configure Slack webhook or OAuth for notifications.

12. **Set Default Values and Constraints:**  
    - In Limit nodes, set batch sizes appropriate for API limits (e.g., 10-50).  
    - In Code nodes, ensure error handling and input validation.  
    - In AI nodes, configure prompt templates and model parameters (temperature, max tokens).  
    - In HTTP Request for sitemap submission, set correct API endpoint and headers.

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                      |
|------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| Workflow automates SEO-optimized blog creation integrating AI content generation, database management, WordPress publishing, and SEO sitemap submission. | Overall workflow purpose                             |
| Uses LangChain nodes for advanced AI orchestration with OpenAI and OpenRouter models.                                               | AI content generation block                          |
| Slack integration for editorial approval workflow.                                                                                 | Editorial process notification                       |
| GitHub nodes handle blog content and sitemap file versioning.                                                                       | Content storage and version control                  |
| Postgres DB used for keyword management, scoring, and internal linking strategies.                                                  | Data persistence and SEO keyword management          |
| Sitemap submission via HTTP request to search engines to ensure SEO indexing of new content.                                        | SEO optimization                                     |
| Recommended to monitor API quotas for OpenAI, Perplexity, and GitHub to avoid rate limit errors.                                    | Operational considerations                            |
| Consider adding error handling and retry mechanisms in Code and HTTP Request nodes for robustness.                                  | Reliability and fault tolerance                       |

---

This document provides a complete, detailed description of the workflow, node-by-node analysis, and instructions to reproduce it fully in n8n, enabling advanced users and AI agents to understand, modify, and extend it.