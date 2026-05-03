Create SEO-Optimized Blogs for WordPress with Perplexity & GPT, Keywords & Media

https://n8nworkflows.xyz/workflows/create-seo-optimized-blogs-for-wordpress-with-perplexity---gpt--keywords---media-7259


# Create SEO-Optimized Blogs for WordPress with Perplexity & GPT, Keywords & Media

### 1. Workflow Overview

This n8n workflow automates the creation of SEO-optimized blog posts for WordPress by leveraging AI models (OpenAI GPT, Perplexity, OpenRouter) and integrates keyword selection, content planning, media handling, and publishing. It targets content creators, SEO specialists, and marketing teams who want to generate well-structured, keyword-rich blogs with automated media cover images and publish directly on WordPress while maintaining internal link logs and sitemap updates.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Keyword Fetching & Scoring**: Periodic trigger to fetch keywords from a Postgres database, score them, and prepare the best candidates for blog generation.

- **1.2 Blog Planning & AI Content Generation**: Using AI models to create a detailed blog plan, including headers, introductions, development sections, FAQs, and conclusions. This includes keyword and intent selection.

- **1.3 Media Handling**: Fetching or generating cover images relevant to the blog content via HTTP requests.

- **1.4 Blog Creation & Publishing**: Compiling the generated content and media, creating or editing files in GitHub, and finally publishing the post on WordPress.

- **1.5 Post-Publishing Actions**: Logging published blog links for internal SEO, sitemap updating, and notifying for approval via Slack.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Keyword Fetching & Scoring

- **Overview:**  
  This block triggers at scheduled intervals to fetch keywords from a Postgres database, score them for relevance or quality, and prepare the best keywords for further blog planning.

- **Nodes Involved:**  
  - Schedule Trigger (`Schedule Trigger`)  
  - Fetch Keywords (`Fetch Keywords1`)  
  - Limit Keywords (`Limit2`)  
  - Select Rows from Table (`Select rows from a table`)  
  - Loop Over Items (`Loop Over Items`)  
  - Score Keywords (Code) (`Score Keywords`)  
  - Update Score (`Update score`)  
  - All scored (NoOp)  

- **Node Details:**  
  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Initiates workflow periodically.  
    - Parameters: Defines schedule timing (not explicitly shown).  
    - Edge cases: Trigger failures, missed triggers, timezone issues.  

  - **Fetch Keywords1**  
    - Type: Postgres  
    - Role: Queries keywords from Postgres DB.  
    - Configuration: SQL query to fetch keywords for scoring.  
    - Edge cases: DB connection errors, empty results.  

  - **Limit2**  
    - Type: Limit  
    - Role: Restricts number of keywords processed (e.g., top N).  
    - Configuration: Limit parameter (e.g., 10 or 20).  
    - Edge cases: Limit too low or high, empty inputs.  

  - **Select rows from a table**  
    - Type: Postgres  
    - Role: Selects related data for each keyword batch.  
    - Edge cases: DB errors, invalid queries.  

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Processes keywords in manageable batches.  
    - Parameters: Batch size (not shown).  

  - **Score Keywords (Code)**  
    - Type: Code  
    - Role: Custom JavaScript logic to compute keyword scores.  
    - Expressions: Uses input data fields to calculate scores.  
    - Edge cases: Expression errors, missing fields.  

  - **Update Score**  
    - Type: Postgres  
    - Role: Updates keyword scores back into the database.  
    - Edge cases: DB write errors, concurrency issues.  

  - **All scored (NoOp)**  
    - Type: NoOp  
    - Role: Flow synchronization point.  

---

#### 1.2 Blog Planning & AI Content Generation

- **Overview:**  
  Based on selected keywords, this block uses AI models (OpenAI GPT, Perplexity, OpenRouter) to generate a detailed blog plan, including keyword intent, headings, introduction, content development segments, conclusion, and FAQ. The plan is parsed, refined, and merged progressively.

- **Nodes Involved:**  
  - Preliminary Plan (OpenAI)  
  - Message a model (Perplexity)  
  - Structured Output Parser (Langchain)  
  - Choose keywords, Intent, etc (Langchain Agent)  
  - Create plan for dev (Langchain Chain)  
  - Intro, dev 1, dev 2, conclusion, FAQ section (Langchain Chains)  
  - Various OpenAI and OpenRouter Chat Model nodes  
  - Merge and Aggregate nodes  
  - Edit Fields (Set nodes) and Code nodes for data transformation  

- **Node Details:**  

  - **Preliminary Plan (OpenAI Chat Model)**  
    - Role: Creates initial blog content plan based on keyword input.  
    - Configuration: Uses GPT with prompt templates (not raw JSON).  
    - Edge cases: API rate limits, malformed prompts.  

  - **Message a model (Perplexity)**  
    - Role: Queries Perplexity AI for supplementary content or validation.  
    - Configuration: API call with keyword context.  
    - Edge cases: API failures, timeouts.  

  - **Structured Output Parser**  
    - Role: Parses AI outputs into structured JSON for further processing.  
    - Edge cases: Parsing errors due to unexpected output formats.  

  - **Choose keywords, Intent, etc (Langchain Agent)**  
    - Role: Analyzes and decides final keyword usage and intent for the blog.  
    - Edge cases: Ambiguous intents or missing keyword data.  

  - **Create plan for dev (Langchain Chain)**  
    - Role: Constructs detailed content development plan blocks.  
    - Connected to multiple sub-chains (Intro, dev 1, dev 2, conclusion, FAQ).  

  - **Intro, dev 1, dev 2, conclusion, FAQ section**  
    - Role: Each creates corresponding blog content sections using AI models.  
    - Edge cases: Partial content generation, API errors.  

  - **Merge and Aggregate nodes**  
    - Role: Combine partial AI outputs into a cohesive blog content structure.  

  - **Edit Fields (Set nodes) and Code nodes**  
    - Role: Data cleaning, formatting, and adding metadata at various steps.  

---

#### 1.3 Media Handling

- **Overview:**  
  This block manages retrieval or generation of blog cover images via HTTP requests, then processes and routes these images into the blog creation flow.

- **Nodes Involved:**  
  - Image Covers2 (HTTP Request)  
  - Split Out (Splitter)  
  - Code nodes for processing  
  - Edit Fields nodes for mapping media data  

- **Node Details:**  

  - **Image Covers2**  
    - Type: HTTP Request  
    - Role: Fetches or generates cover images based on blog content or keywords.  
    - Configuration: API endpoint for image service (e.g., Unsplash or custom).  
    - Edge cases: Network errors, invalid API keys, empty image responses.  

  - **Split Out**  
    - Role: Splits image data array into individual items for processing.  

  - **Code and Edit Fields**  
    - Role: Extract URLs, filenames, metadata, or resize images as needed.  

---

#### 1.4 Blog Creation & Publishing

- **Overview:**  
  This block compiles the generated textual and media content, creates or updates blog files in GitHub, and finally publishes the post on WordPress.

- **Nodes Involved:**  
  - Create blog (GitHub)  
  - Edit a file, Edit a file1 (GitHub)  
  - Create a post1 (WordPress)  
  - Merge nodes and Edit Fields nodes for data preparation  

- **Node Details:**  

  - **Create blog**  
    - Type: GitHub  
    - Role: Commits new blog markdown or content files to GitHub repository.  
    - Configuration: Repository, branch, file path, commit message.  
    - Edge cases: Auth errors, commit conflicts.  

  - **Edit a file, Edit a file1**  
    - Role: Update existing files with new content or metadata.  

  - **Create a post1**  
    - Type: WordPress  
    - Role: Publishes the compiled blog post on WordPress via API.  
    - Configuration: Post title, body, categories, featured image.  
    - Edge cases: API authentication, permission errors, invalid post data.  

---

#### 1.5 Post-Publishing Actions

- **Overview:**  
  After publishing, this block logs the blog link for internal linking, updates sitemap variables, submits sitemap updates, and sends a Slack notification for approval or review.

- **Nodes Involved:**  
  - Log blog link for future internal links (Postgres)  
  - set sitemap variables (Set)  
  - submit sitemap (HTTP Request)  
  - wait approval (Slack)  
  - Edit Fields nodes for data handling  

- **Node Details:**  

  - **Log blog link for future internal links**  
    - Type: Postgres  
    - Role: Inserts or updates the blog URL and metadata for internal SEO tracking.  
    - Edge cases: DB errors, duplicate inserts.  

  - **set sitemap variables**  
    - Type: Set  
    - Role: Prepares data for sitemap submission (e.g., URLs, timestamps).  

  - **submit sitemap**  
    - Type: HTTP Request  
    - Role: Notifies search engines or services to update sitemap.  
    - Edge cases: Network, API errors.  

  - **wait approval**  
    - Type: Slack  
    - Role: Sends a message or request for human approval or review before final steps.  
    - Edge cases: Slack API rate limits, missing channels or permissions.  

---

### 3. Summary Table

| Node Name                       | Node Type                      | Functional Role                          | Input Node(s)                                    | Output Node(s)                                   | Sticky Note                  |
|--------------------------------|--------------------------------|-----------------------------------------|-------------------------------------------------|-------------------------------------------------|------------------------------|
| Schedule Trigger                | Schedule Trigger               | Initiate workflow periodically          |                                                 | Fetch Keywords1                                  |                              |
| Fetch Keywords1                 | Postgres                      | Fetch keywords from DB                   | Schedule Trigger                                 | Limit2                                           |                              |
| Limit2                         | Limit                         | Limits number of keywords processed     | Fetch Keywords1                                  | Select rows from a table                          |                              |
| Select rows from a table        | Postgres                      | Select data related to keywords         | Limit2                                           | Loop Over Items                                  |                              |
| Loop Over Items                 | SplitInBatches                | Batch keyword processing                 | Select rows from a table                         | All scored, Score Keywords                        |                              |
| Score Keywords                 | Code                          | Compute keyword scores                   | Loop Over Items                                  | Update score                                     |                              |
| Update score                   | Postgres                      | Update keyword scores in DB              | Score Keywords                                   | Loop Over Items (next batch)                      |                              |
| All scored                    | NoOp                          | Synchronize after scoring                | Loop Over Items                                  |                                                  |                              |
| select blog table             | Postgres                      | Select blog data                         | Schedule Trigger1                                | limit to 1                                       |                              |
| limit to 1                   | Limit                         | Limits blogs to one for planning        | select blog table                                | Preliminary Plan                                 |                              |
| Preliminary Plan             | OpenAI Chat Model             | Generate initial blog plan               | limit to 1                                       | Message a model                                  |                              |
| Message a model              | Perplexity                    | Query Perplexity AI for content          | Preliminary Plan                                 | Split Out1, Split Out3, Edit Fields7, etc.       |                              |
| Split Out                   | SplitOut                     | Split image data                         | Image Covers2                                    | Code1                                            |                              |
| Image Covers2               | HTTP Request                  | Fetch/generate blog cover images         | Edit Fields19                                    | Split Out                                        |                              |
| Code1                       | Code                          | Process image data                       | Split Out                                        | Code                                             |                              |
| Code                        | Code                          | Further image or data processing        | Code1                                            | Edit Fields4                                     |                              |
| Edit Fields4                | Set                           | Data formatting                         | Code                                              | Filter                                           |                              |
| Filter                      | Filter                        | Filter data according to criteria       | Edit Fields4                                     | Limit                                            |                              |
| Limit                      | Limit                         | Restrict number of filtered items       | Filter                                           | Edit Fields14                                    |                              |
| Edit Fields14               | Set                           | Prepare data for merging                 | Limit                                            | Merge                                            |                              |
| Merge                      | Merge                         | Combine multiple data streams            | Edit Fields14, Aggregate, Aggregate3, etc.      | Aggregate4                                        |                              |
| Aggregate4                 | Aggregate                     | Aggregate content parts                   | Merge                                             | Intro, dev 1, dev 2, conclusion, FAQ section     |                              |
| Intro                      | Langchain ChainLlm            | Generate introduction content            | Aggregate4                                         | Edit Fields9                                     |                              |
| dev 1                      | Langchain ChainLlm            | Generate first development section       | Aggregate4                                         | Edit Fields10                                    |                              |
| dev 2                      | Langchain ChainLlm            | Generate second development section      | Aggregate4                                         | Edit Fields11                                    |                              |
| conclusion                 | Langchain ChainLlm            | Generate conclusion content              | Aggregate4                                         | Edit Fields12                                    |                              |
| FAQ section                | Langchain ChainLlm            | Generate FAQ section                     | Aggregate4                                         | Edit Fields20                                    |                              |
| Edit Fields9               | Set                           | Prepare intro for merging                 | Intro                                             | Merge2                                           |                              |
| Edit Fields10              | Set                           | Prepare dev 1 for merging                 | dev 1                                             | Merge2                                           |                              |
| Edit Fields11              | Set                           | Prepare dev 2 for merging                 | dev 2                                             | Merge2                                           |                              |
| Edit Fields12              | Set                           | Prepare conclusion for merging            | conclusion                                        | Merge2                                           |                              |
| Edit Fields20              | Set                           | Prepare FAQ for merging                   | FAQ section                                       | Merge2                                           |                              |
| Merge2                    | Merge                         | Combine final blog content parts          | Edit Fields9, Edit Fields10, Edit Fields11, Edit Fields12, Edit Fields20 | Aggregate5                                        |                              |
| Aggregate5                 | Aggregate                     | Aggregate full blog content               | Merge2                                            | Edit Fields                                       |                              |
| Edit Fields                | Set                           | Final preparation before GitHub commit   | Aggregate5                                         | Code6                                            |                              |
| Code6                      | Code                          | Create blog data for GitHub               | Edit Fields                                        | get articles.json1                               |                              |
| get articles.json1         | GitHub                       | Retrieve articles JSON file                | Code6                                              | Extract from File3                               |                              |
| Extract from File3         | Extract from File             | Parse articles.json content                | get articles.json1                                 | Edit Fields1                                     |                              |
| Edit Fields1               | Set                           | Prepare data for blog creation             | Extract from File3                                 | Code7                                            |                              |
| Code7                      | Code                          | Transform blog content                      | Edit Fields1                                       | Edit Fields15                                    |                              |
| Edit Fields15              | Set                           | Prepare content for GitHub commit           | Code7                                              | Edit a file                                     |                              |
| Edit a file                | GitHub                       | Commit blog content file                    | Edit Fields15                                      | Merge3                                           |                              |
| Merge3                     | Merge                         | Combine GitHub commit outputs               | Edit a file, Edit a file1, Create blog            | Limit3                                           |                              |
| Limit3                     | Limit                         | Limit output posts to process               | Merge3                                             | Edit Fields16, Edit Fields17                      |                              |
| Edit Fields16              | Set                           | Prepare Slack approval notification         | Limit3                                             | wait approval                                    |                              |
| wait approval              | Slack                        | Send Slack message for approval             | Edit Fields16                                      | set sitemap variables                            |                              |
| set sitemap variables      | Set                           | Prepare sitemap data                         | wait approval                                      | submit sitemap                                   |                              |
| submit sitemap             | HTTP Request                 | Submit sitemap update                        | set sitemap variables                              |                                                  |                              |
| Create a post1             | WordPress                   | Publish blog post                            | Edit Fields2                                       |                                                  |                              |
| Log blog link for future internal links | Postgres             | Log new blog URLs for SEO                      | Edit Fields17                                      |                                                  |                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a scheduled trigger node:**  
   - Type: Schedule Trigger  
   - Configure schedule (e.g., daily or weekly) to kick off keyword fetching.

2. **Add Postgres node to fetch keywords:**  
   - Query relevant keywords from your Postgres database.

3. **Add Limit node:**  
   - Limit keyword count (e.g., top 10-20) to control processing volume.

4. **Add Postgres node to select additional data:**  
   - Use keyword data to fetch associated records for scoring.

5. **Add SplitInBatches node:**  
   - Process keywords in batches for scalability.

6. **Add a Code node to score keywords:**  
   - Implement JavaScript to calculate keyword relevance or quality.

7. **Add Postgres node to update scores:**  
   - Write back scores into the database.

8. **Add NoOp node for synchronization.**

9. **Set up a second scheduled trigger:**  
   - For selecting blogs to plan and generate content.

10. **Add Postgres node to select blog table data.**

11. **Add Limit node (limit to 1):**  
    - To process one blog at a time for planning.

12. **Add OpenAI Chat Model node:**  
    - Configure with your OpenAI credentials and prompt templates to create a preliminary blog plan.

13. **Add Perplexity node:**  
    - Configure API credentials to supplement content or validate.

14. **Add SplitOut nodes:**  
    - To split multi-item responses (e.g., images, content parts).

15. **Add HTTP Request node for Image Covers:**  
    - Configure API call to fetch or generate relevant blog cover images.

16. **Add Code and Set nodes:**  
    - To format and prepare image data for blog integration.

17. **Add Langchain Structured Output Parser nodes:**  
    - To parse AI-generated structured content into usable JSON.

18. **Add Langchain Agent and ChainLlm nodes:**  
    - For keyword intent selection, detailed content sections (intro, dev 1, dev 2, conclusion, FAQ).

19. **Add Merge and Aggregate nodes:**  
    - Combine partial AI outputs into a single blog content object.

20. **Add GitHub nodes:**  
    - Create or edit blog files in your repository (configure OAuth or Personal Access Token credentials).

21. **Add WordPress node:**  
    - Configure with WordPress API credentials (OAuth2 or Application Passwords) to publish the blog.

22. **Add Postgres node to log published blog URLs:**  
    - For internal link tracking.

23. **Add Set and HTTP Request nodes:**  
    - To prepare and submit sitemap updates to search engines.

24. **Add Slack node:**  
    - Send approval requests or notifications (configure Slack webhook or OAuth).

25. **Ensure all nodes are connected according to logical flow:**  
    - Schedule Trigger → Fetch Keywords → Score → Blog Planning → AI Content → Media Handling → File Commit → WordPress Publish → Post Actions.

26. **Test each step with valid credentials and sample data.**

27. **Implement error handling where possible:**  
    - For API failures, DB errors, empty data, or rate limits.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                                           |
|------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------|
| Workflow leverages OpenAI GPT models and Perplexity AI for content generation and validation. | Requires API keys and credentials for OpenAI and Perplexity.              |
| GitHub integration commits blog content as markdown or JSON files into a repository.           | Use Personal Access Tokens with repo write permissions.                   |
| WordPress node requires API credentials with publishing rights (OAuth2 or Application Password). | Ensure user roles allow post creation and media uploads.                  |
| Slack node configured for approval notifications to enable manual review before sitemap submission. | Slack app with chat:write permission and webhook URL needed.             |
| Sitemap submission via HTTP request targets search engines or indexing services.               | Ensure correct endpoint URLs and API keys if required.                    |
| For AI prompt engineering, customize prompt templates inside Langchain/OpenAI nodes per content needs. | Enables improved SEO-optimized structure and keyword usage.              |
| The workflow uses multiple data transformation nodes (Code, Set) for fine control on data format. | Adapt these nodes carefully when modifying the workflow.                  |

---

**Disclaimer:** The provided text is exclusively from an automated workflow created with n8n, respecting all current content policies and containing no illegal, offensive, or protected elements. All data processed are legal and public.