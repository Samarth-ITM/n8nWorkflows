Transform Readwise Highlights into Weekly Content Ideas with Gemini AI

https://n8nworkflows.xyz/workflows/transform-readwise-highlights-into-weekly-content-ideas-with-gemini-ai-10217


# Transform Readwise Highlights into Weekly Content Ideas with Gemini AI

---
### 1. Workflow Overview

This workflow automates the process of transforming weekly Readwise highlights and articles into insightful content ideas using Gemini AI (Google’s advanced large language model). It targets professionals in tech consulting and AI engineering who want to leverage their reading habits into actionable insights, content creation, and professional development.

**Logical blocks:**

- **1.1 Input Reception & Scheduling:**  
  Workflow triggers either manually or automatically every Monday at 09:00 to start the data retrieval and processing.

- **1.2 Data Fetching from Readwise:**  
  Retrieves articles and highlights updated in the last 7 days from Readwise API with pagination support.

- **1.3 Data Filtering and Matching:**  
  Splits fetched data into individual items, filters articles with >10% reading progress, and matches highlights to their respective articles.

- **1.4 Data Merging and AI Prompt Preparation:**  
  Combines filtered articles with their highlights into structured data, then crafts a detailed AI prompt for generating insights.

- **1.5 AI Processing:**  
  Sends the prepared prompt to Google Gemini 2.5 Pro model via OpenRouter API to generate markdown-formatted weekly insights.

- **1.6 Postprocessing and Saving Results:**  
  Converts AI’s markdown output to HTML, formats it for Readwise’s save API, and posts it back to Readwise as a new article.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Scheduling

- **Overview:**  
  This block defines the workflow’s trigger mechanisms: manual execution or scheduled automatic runs every Monday at 09:00.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Monday - 09:00 (Schedule Trigger)  
  - Fetch Articles (first node after triggers)

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Allows ad-hoc manual start of the workflow.  
    - Outputs: Connects to Fetch Articles to initiate data fetching.  
    - Edge cases: Manual trigger requires user intervention; no auth issues here.

  - **Monday - 09:00**  
    - Type: Schedule Trigger  
    - Role: Automates weekly execution every Monday at 09:00 (cron expression: “0 9 * * 1”).  
    - Outputs: Connects to Fetch Articles node.  
    - Edge cases: Workflow depends on n8n scheduler uptime; cron misconfiguration can cause missed runs.

---

#### 2.2 Data Fetching from Readwise

- **Overview:**  
  Retrieves articles and highlights updated within the last 7 days from Readwise API, handling pagination via nextPageCursor.

- **Nodes Involved:**  
  - Fetch Articles (HTTP Request)  
  - Fetch Highlights (HTTP Request)  
  - Split Article Results (SplitOut)  
  - Split Highlights Results (SplitOut)

- **Node Details:**

  - **Fetch Articles**  
    - Type: HTTP Request  
    - Configuration: GET https://readwise.io/api/v3/list with query params: updatedAfter = 7 days ago, withHtmlContent=true, category=article  
    - Auth: HTTP Header Auth with Readwise credentials  
    - Pagination: Uses responseContainsNextURL mode with nextPageCursor to fetch all pages  
    - Outputs: JSON response containing articles list under “results” array  
    - Edge cases: API rate limits, auth failures, network timeouts.

  - **Split Article Results**  
    - Type: SplitOut  
    - Configuration: Splits input array field “results” into separate items for downstream filtering  
    - Inputs: Output of Fetch Articles  
    - Outputs: Individual article items  
    - Edge cases: Empty results array leads to zero downstream items.

  - **Fetch Highlights**  
    - Type: HTTP Request  
    - Configuration: GET https://readwise.io/api/v3/list with query params: updatedAfter=7 days ago, category=highlight  
    - Auth: Same as Fetch Articles  
    - Pagination: Identical pagination setup  
    - Outputs: JSON response with highlights list in “results”  
    - Edge cases: Same as Fetch Articles.

  - **Split Highlights Results**  
    - Type: SplitOut  
    - Configuration: Splits “results” array into individual highlight items  
    - Inputs: Output of Fetch Highlights  
    - Outputs: Individual highlight items  
    - Edge cases: Empty highlight list results in no items.

---

#### 2.3 Data Filtering and Matching

- **Overview:**  
  Filters articles to only those with reading progress >10%, then matches highlights belonging to those filtered articles.

- **Nodes Involved:**  
  - Filter Articles >10% Read (Filter)  
  - Find Highlights for Article (Filter)

- **Node Details:**

  - **Filter Articles >10% Read**  
    - Type: Filter  
    - Condition: reading_progress > 0.1 (strict numeric comparison)  
    - Inputs: Individual articles from Split Article Results  
    - Outputs: Articles passing the filter forwarded to Fetch Highlights  
    - Edge cases: Articles with missing or malformed reading_progress field will fail condition.

  - **Find Highlights for Article**  
    - Type: Filter  
    - Condition: highlight.parent_id == filtered article.id (matching highlights to their parent article)  
    - Inputs: Highlights from Split Highlights Results  
    - Outputs: Highlights filtered for each specific article  
    - Edge cases: Missing parent_id or mismatched IDs cause highlight exclusion.

---

#### 2.4 Data Merging and AI Prompt Preparation

- **Overview:**  
  Merges filtered articles and their matched highlights into a structured JSON object. Then builds a detailed prompt string for the AI model, including strict formatting rules.

- **Nodes Involved:**  
  - Merge Articles & Highlights (Code)  
  - Prepare Prompt (Code)

- **Node Details:**

  - **Merge Articles & Highlights**  
    - Type: Code (JavaScript)  
    - Function: Iterates filtered articles; for each article, collects highlights with matching parent_id, organizes data including title, URLs, summary, reading progress, highlight texts/notes.  
    - Inputs: Outputs of Filter Articles >10% Read and Find Highlights for Article  
    - Outputs: Array of merged article-highlight objects  
    - Edge cases: Handles missing highlights gracefully; mismatched or missing fields could cause undefined values.

  - **Prepare Prompt**  
    - Type: Code (JavaScript)  
    - Function: Converts merged data into a markdown-formatted string with explicit AI prompt instructions and formatting rules for markdown links.  
    - Key variables: `$input.all()` to gather all merged items  
    - Output: JSON with keys: prompt (string), articles_count (number), total_highlights (number)  
    - Edge cases: Large input data may cause prompt truncation; malformed content fields may break formatting.

---

#### 2.5 AI Processing

- **Overview:**  
  Sends the constructed prompt to Google Gemini 2.5 Pro model via OpenRouter API through LangChain integration to generate weekly reading insights in markdown format.

- **Nodes Involved:**  
  - Generate Weekly Insights (LangChain Chain LLM)  
  - AI Model (LangChain LM Chat OpenRouter)

- **Node Details:**

  - **Generate Weekly Insights**  
    - Type: LangChain Chain LLM node  
    - Input: Prompt text from Prepare Prompt node  
    - Configuration: Uses "define" prompt type, retries on failure enabled  
    - Outputs: AI-generated markdown text under `json.text`  
    - Edge cases: API rate limits, timeout, malformed prompt can cause errors; retry mitigates transient failures.

  - **AI Model**  
    - Type: LangChain LM Chat OpenRouter node  
    - Model: google/gemini-2.5-pro  
    - Credentials: OpenRouter API key configured  
    - Role: Backend LLM provider for Generate Weekly Insights  
    - Edge cases: Credential expiration, quota limits, network issues.

---

#### 2.6 Postprocessing and Saving Results

- **Overview:**  
  Converts AI markdown output to HTML, formats JSON payload for Readwise save API, and submits the new article with weekly insights back to Readwise.

- **Nodes Involved:**  
  - Convert Insights to HTML (Markdown)  
  - Format for Readwise Save API (Code)  
  - Save Insights to Readwise (HTTP Request)

- **Node Details:**

  - **Convert Insights to HTML**  
    - Type: Markdown node  
    - Function: Converts AI markdown text to HTML with options: tables enabled, simplified autolinks, links open in new window  
    - Inputs: AI-generated markdown text  
    - Outputs: HTML string  
    - Edge cases: Malformed markdown may produce invalid HTML.

  - **Format for Readwise Save API**  
    - Type: Code (JavaScript)  
    - Function: Packages HTML result into JSON formatted for Readwise /save endpoint: fields include url (with date), html content, title, author, summary, category, flags for HTML cleaning  
    - Inputs: HTML from previous node  
    - Outputs: JSON payload for saving  
    - Edge cases: Date formatting errors or missing content fields.

  - **Save Insights to Readwise**  
    - Type: HTTP Request (POST)  
    - URL: https://readwise.io/api/v3/save/  
    - Auth: HTTP Header Auth with Readwise credentials  
    - Sends: JSON body as formatted above  
    - Role: Creates a new article in Readwise with AI-generated insights  
    - Edge cases: API errors, auth failures, network issues.

---

### 3. Summary Table

| Node Name                    | Node Type                    | Functional Role                         | Input Node(s)                  | Output Node(s)                 | Sticky Note                                                                                     |
|------------------------------|------------------------------|---------------------------------------|-------------------------------|-------------------------------|------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger               | Manual start of workflow               |                               | Fetch Articles                 | ## 📌 START This workflow can be triggered in two ways: 1. Manually 2. Scheduled every Monday at 09:00 |
| Monday - 09:00               | Schedule Trigger             | Scheduled weekly trigger at Monday 09:00 |                               | Fetch Articles                 | ## 📌 START This workflow can be triggered in two ways: 1. Manually 2. Scheduled every Monday at 09:00 |
| Fetch Articles               | HTTP Request                | Fetch articles updated in last 7 days | When clicking ‘Execute workflow’, Monday - 09:00 | Split Article Results          | ## 📌 FETCH READWISE DATA These nodes connect to Readwise API and fetch articles updated in the last 7 days. Ensure credentials are set. |
| Split Article Results        | SplitOut                    | Split articles array into individual items | Fetch Articles                | Filter Articles >10% Read      | ## 📌 FILTER & MATCH DATA Splits articles for filtering.                                         |
| Filter Articles >10% Read    | Filter                      | Filter articles with reading progress > 10% | Split Article Results         | Fetch Highlights              | ## 📌 FILTER & MATCH DATA Filters articles to those read more than 10%.                         |
| Fetch Highlights             | HTTP Request                | Fetch highlights updated in last 7 days | Filter Articles >10% Read      | Split Highlights Results       | ## 📌 FETCH READWISE DATA These nodes connect to Readwise API and fetch highlights updated in the last 7 days. Ensure credentials are set. |
| Split Highlights Results     | SplitOut                    | Split highlights array into individual items | Fetch Highlights              | Find Highlights for Article    | ## 📌 FILTER & MATCH DATA Splits highlights for filtering.                                      |
| Find Highlights for Article  | Filter                      | Filter highlights belonging to filtered articles | Split Highlights Results      | Merge Articles & Highlights    | ## 📌 FILTER & MATCH DATA Matches highlights to articles based on parent_id.                    |
| Merge Articles & Highlights  | Code                        | Merge filtered articles and highlights into structured data | Find Highlights for Article    | Prepare Prompt                | ## 📌 PREPARE AI PROMPT Combines filtered data for AI prompt preparation.                       |
| Prepare Prompt              | Code                        | Build detailed AI prompt with formatting rules | Merge Articles & Highlights    | Generate Weekly Insights       | ## 📌 PREPARE AI PROMPT Customize this node to change AI persona, questions, and output format. |
| Generate Weekly Insights     | LangChain Chain LLM         | Send prompt to AI model for insight generation | Prepare Prompt                | Convert Insights to HTML       | ## 📌 GENERATE AI INSIGHTS Uses Gemini 2.5 Pro via OpenRouter to generate markdown insights.     |
| AI Model                    | LangChain LM Chat OpenRouter | Backend AI model node (Gemini 2.5 Pro) | Generate Weekly Insights       |                               | ## 📌 GENERATE AI INSIGHTS See above.                                                           |
| Convert Insights to HTML     | Markdown                    | Convert AI markdown response to HTML  | Generate Weekly Insights       | Format for Readwise Save API   | ## 📌 SAVE INSIGHTS BACK TO READWISE Converts AI markdown to HTML suitable for Readwise.         |
| Format for Readwise Save API | Code                        | Format HTML and metadata for Readwise save API | Convert Insights to HTML       | Save Insights to Readwise      | ## 📌 SAVE INSIGHTS BACK TO READWISE Package JSON payload for Readwise /save endpoint.           |
| Save Insights to Readwise    | HTTP Request                | POST the new insights article to Readwise | Format for Readwise Save API   |                               | ## 📌 SAVE INSIGHTS BACK TO READWISE Sends final payload to Readwise API to save new article.    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node**  
   - Name: When clicking ‘Execute workflow’  
   - Purpose: Manual start of workflow

2. **Create Schedule Trigger node**  
   - Name: Monday - 09:00  
   - Cron expression: `0 9 * * 1` (every Monday at 09:00)  
   - Connect this node’s output to Fetch Articles node (next step)

3. **Create HTTP Request node**  
   - Name: Fetch Articles  
   - Method: GET  
   - URL: `https://readwise.io/api/v3/list`  
   - Query parameters:  
     - updatedAfter = `={{ $now.minus(7, 'days') }}` (ISO date 7 days ago)  
     - withHtmlContent = `true`  
     - category = `article`  
   - Authentication: HTTP Header Auth with Readwise API token  
   - Pagination: responseContainsNextURL using `nextPageCursor` from response for next URL  
   - Connect outputs from Manual Trigger and Schedule Trigger here

4. **Create SplitOut node**  
   - Name: Split Article Results  
   - Field to split out: `results` (array of articles)  
   - Connect output of Fetch Articles here

5. **Create Filter node**  
   - Name: Filter Articles >10% Read  
   - Condition: `reading_progress` (number) greater than `0.1`  
   - Connect output of Split Article Results here

6. **Create HTTP Request node**  
   - Name: Fetch Highlights  
   - Method: GET  
   - URL: `https://readwise.io/api/v3/list`  
   - Query parameters:  
     - updatedAfter = `={{ $now.minus(7, 'days') }}`  
     - category = `highlight`  
   - Authentication: same as Fetch Articles  
   - Pagination setup same as Fetch Articles  
   - Connect output of Filter Articles >10% Read here

7. **Create SplitOut node**  
   - Name: Split Highlights Results  
   - Field to split out: `results` (array of highlights)  
   - Connect output of Fetch Highlights here

8. **Create Filter node**  
   - Name: Find Highlights for Article  
   - Condition: `parent_id` equals the filtered article’s `id` (reference Filter Articles >10% Read output item JSON id)  
   - Connect output of Split Highlights Results here

9. **Create Code node**  
   - Name: Merge Articles & Highlights  
   - JavaScript code to:  
     - Iterate filtered articles  
     - For each article, find matching highlights by parent_id  
     - Create combined JSON object with article fields and highlights array  
   - Inputs: outputs of Filter Articles >10% Read and Find Highlights for Article  
   - Connect output of Find Highlights for Article here

10. **Create Code node**  
    - Name: Prepare Prompt  
    - JavaScript code to:  
      - Aggregate merged data from Merge Articles & Highlights node  
      - Build formatted markdown prompt string with explicit formatting rules for AI  
      - Output JSON with prompt text, articles_count, total_highlights  
    - Connect output of Merge Articles & Highlights here

11. **Create LangChain Chain LLM node**  
    - Name: Generate Weekly Insights  
    - Input text: prompt from Prepare Prompt node  
    - Prompt type: define  
    - Enable retry on failure  
    - Connect output of Prepare Prompt here

12. **Create LangChain LM Chat OpenRouter node**  
    - Name: AI Model  
    - Model: google/gemini-2.5-pro  
    - Credentials: OpenRouter API key  
    - Connect output of Generate Weekly Insights here (as AI Model backend)

13. **Create Markdown node**  
    - Name: Convert Insights to HTML  
    - Mode: markdownToHtml  
    - Options: tables enabled, simplifiedAutoLink enabled, openLinksInNewWindow enabled  
    - Input: AI-generated markdown from Generate Weekly Insights  
    - Connect output of Generate Weekly Insights here

14. **Create Code node**  
    - Name: Format for Readwise Save API  
    - JavaScript code to:  
      - Extract AI insights HTML  
      - Create JSON payload with url (including current date), html, title, author, summary, category, and cleaning flag  
    - Connect output of Convert Insights to HTML here

15. **Create HTTP Request node**  
    - Name: Save Insights to Readwise  
    - Method: POST  
    - URL: https://readwise.io/api/v3/save/  
    - Authentication: HTTP Header Auth with Readwise credentials  
    - Body: JSON from Format for Readwise Save API node  
    - Connect output of Format for Readwise Save API here

16. **Wire connections** as per the sequence above, ensuring node outputs feed into the correct next node.

17. **Credential Setup:**  
    - Readwise: HTTP Header Auth with API token for all HTTP Request nodes interacting with Readwise  
    - OpenRouter: API key credential for AI Model node

18. **Optional:** Add sticky notes to document workflow sections for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                             | Context or Link                                                                                                                                             |
|------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Workflow supports two triggers: manual and scheduled (every Monday 09:00) to automate weekly insights generation.                       | Sticky Note "START"                                                                                                                                         |
| Readwise API usage requires API token configured as HTTP Header Auth on all relevant HTTP Request nodes.                                | Sticky Note "FETCH READWISE DATA"                                                                                                                          |
| Filter ensures only articles with meaningful engagement (>10% read) are processed to focus AI analysis.                                 | Sticky Note "FILTER & MATCH DATA"                                                                                                                          |
| AI prompt contains strict markdown link formatting rules to ensure output is clean, consistent, and usable in markdown environments.    | Sticky Note "PREPARE AI PROMPT"                                                                                                                            |
| Gemini 2.5 Pro model used via OpenRouter; can be replaced by other supported LLMs like OpenAI or Anthropic with credential updates.      | Sticky Note "GENERATE AI INSIGHTS"                                                                                                                         |
| Final AI insights converted to HTML and saved back into Readwise as a new article, enabling workflow-generated content to be tracked.   | Sticky Note "SAVE INSIGHTS BACK TO READWISE"                                                                                                               |
| Official Readwise API documentation: https://readwise.io/api/v3/docs                                                                          | Readwise API reference                                                                                                                                      |
| OpenRouter API info: https://openrouter.ai/docs                                                                                           | OpenRouter API for LLM integration                                                                                                                         |

---

**Disclaimer:**  
The provided text is exclusively sourced from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.