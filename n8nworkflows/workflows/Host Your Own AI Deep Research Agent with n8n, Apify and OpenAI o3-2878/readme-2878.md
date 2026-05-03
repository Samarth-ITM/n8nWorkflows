Host Your Own AI Deep Research Agent with n8n, Apify and OpenAI o3

https://n8nworkflows.xyz/workflows/host-your-own-ai-deep-research-agent-with-n8n--apify-and-openai-o3-2878


# Host Your Own AI Deep Research Agent with n8n, Apify and OpenAI o3

### 1. Workflow Overview

This workflow replicates OpenAI's DeepResearch feature, enabling users to perform multi-step, recursive research tasks by synthesizing large amounts of online information. It targets analysts, researchers, and teams who want to automate deep web research and generate detailed reports with AI assistance.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Initialization:** Captures user research queries and parameters via forms, sets initial variables, and creates a placeholder report page in Notion.
- **1.2 Clarifying Questions:** Uses AI to generate follow-up questions to refine the research direction and collects user answers via dynamic forms.
- **1.3 Deep Research Loop:** Recursively generates search queries, performs web searches and scraping via Apify, extracts learnings from results, and accumulates knowledge until the specified depth is reached.
- **1.4 Report Generation:** Compiles all learnings using a reasoning LLM model to generate a detailed markdown report.
- **1.5 Report Formatting and Upload:** Converts the markdown report into Notion blocks and uploads the content to the previously created Notion page, appending source URLs.
- **1.6 Workflow Control and Error Handling:** Manages asynchronous execution, routing, and error detection, especially for API authentication issues.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Initialization

**Overview:**  
This block captures the user's research query and configuration parameters (depth and breadth), initializes workflow variables, and creates an empty Notion page to hold the final report.

**Nodes Involved:**  
- On form submission  
- Research Request (form)  
- Set Variables  
- Report Page Generator (LLM)  
- Structured Output Parser4  
- Create Row (Notion)  
- Initiate DeepResearch (Execute Workflow)  
- Confirmation (form)  
- End Form  

**Node Details:**

- **On form submission**  
  - *Type:* Form Trigger  
  - *Role:* Entry point; triggers workflow on user form submission at path `/deep_research`.  
  - *Config:* Publicly accessible form with custom HTML and instructions.  
  - *Outputs:* Passes form data to "Research Request".

- **Research Request**  
  - *Type:* Form  
  - *Role:* Collects user input: research topic, depth (subquery levels), breadth (number of sources), and user acknowledgment of cost/time.  
  - *Config:* Uses textarea and range sliders (depth 1-5, breadth 1-5).  
  - *Edge Cases:* Invalid or missing inputs handled by default values in "Set Variables".

- **Set Variables**  
  - *Type:* Set  
  - *Role:* Extracts and normalizes user inputs into workflow variables: `request_id` (execution ID), `prompt` (research query), `depth`, and `breadth`.  
  - *Config:* Defaults depth and breadth to 1 if inputs are invalid or missing.  
  - *Expressions:* Uses `$json` from form input and `$execution.id` for request ID.

- **Report Page Generator**  
  - *Type:* Chain LLM (OpenAI o3-mini)  
  - *Role:* Generates a suitable title for the research report based on the user query.  
  - *Config:* Prompt instructs LLM to create a concise title.  
  - *Credentials:* OpenAI API.  
  - *Output Parser:* Structured Output Parser4.

- **Structured Output Parser4**  
  - *Type:* Output Parser Structured  
  - *Role:* Parses LLM output into JSON with `title` and `description` fields for Notion.  
  - *Schema:* Manual JSON schema defining title and description.

- **Create Row**  
  - *Type:* Notion (databasePage)  
  - *Role:* Creates a new page in the designated Notion database to hold the report.  
  - *Config:* Sets page properties: title, description, status ("Not started"), and request ID.  
  - *Credentials:* Notion API.  
  - *Edge Cases:* Notion API errors or invalid database ID.

- **Initiate DeepResearch**  
  - *Type:* Execute Workflow (self-call)  
  - *Role:* Starts the asynchronous deep research process with initial data (query, empty learnings, depth, breadth).  
  - *Config:* Runs the same workflow with `jobType` = `deepresearch_initiate`.  
  - *Execution:* Non-blocking (does not wait for completion).

- **Confirmation**  
  - *Type:* Form  
  - *Role:* Displays confirmation to user with link to the Notion page created.  
  - *Config:* Shows report page URL and name with branding image.  
  - *Webhook:* Separate webhook for form submission.

- **End Form**  
  - *Type:* Form (completion)  
  - *Role:* Finalizes user interaction with a thank-you message.  
  - *Config:* Simple completion message.

**Edge Cases:**  
- Invalid user inputs defaulted to safe values.  
- Notion API failures require credential validation.  
- OpenAI API quota or connectivity issues may delay title generation.

---

#### 2.2 Clarifying Questions

**Overview:**  
Generates follow-up questions to clarify the research direction and collects user answers via dynamic forms, enabling iterative refinement of the research query.

**Nodes Involved:**  
- Clarifying Questions (Chain LLM)  
- Structured Output Parser1  
- Feedback to Items (Split Out)  
- For Each Question... (Split In Batches)  
- Ask Clarity Questions (Form)  
- On form submission (loop back)  
- JobType Router (routes to this block)  
- Sticky Note (explanatory)

**Node Details:**

- **Clarifying Questions**  
  - *Type:* Chain LLM (OpenAI o3-mini)  
  - *Role:* Generates up to 3 follow-up questions based on the initial user prompt.  
  - *Prompt:* Expert researcher instructions with emphasis on detail and accuracy.  
  - *Output Parser:* Structured Output Parser1.

- **Structured Output Parser1**  
  - *Type:* Output Parser Structured  
  - *Role:* Parses LLM output into JSON array of questions.

- **Feedback to Items**  
  - *Type:* Split Out  
  - *Role:* Splits the array of questions into individual items for processing.

- **For Each Question...**  
  - *Type:* Split In Batches  
  - *Role:* Iterates over each question to present to the user.

- **Ask Clarity Questions**  
  - *Type:* Form  
  - *Role:* Presents each question to the user for input.  
  - *Config:* Textarea for user answers, required field.  
  - *Webhook:* Separate webhook for form submission.

- **On form submission** (loop back)  
  - *Role:* Collects user answers and feeds them back into the workflow to update the research query.

- **JobType Router**  
  - *Role:* Routes execution based on `jobType` parameter, directing to clarifying questions or other blocks.

- **Sticky Note**  
  - *Content:* Explains the use of dynamic forms for clarification, referencing the "AI Interviewer" template.

**Edge Cases:**  
- User may provide incomplete or ambiguous answers.  
- Looping forms require careful state management to avoid infinite loops.  
- Form webhook failures or timeouts.

---

#### 2.3 Deep Research Loop

**Overview:**  
Performs recursive web search and scraping to gather data on the research topic, generating subqueries and accumulating learnings until the specified depth and breadth are reached.

**Nodes Involved:**  
- JobType Router  
- Generate SERP Queries (Chain LLM)  
- Structured Output Parser2  
- SERP to Items (Split Out)  
- For Each Query... (Split In Batches)  
- DeepResearch Subworkflow (Execute Workflow Trigger)  
- Execution Data  
- Accumulate Results (Set)  
- Is Depth Reached? (If)  
- Get Research Results (Set)  
- Generate Learnings (Execute Workflow)  
- RAG Web Browser (HTTP Request to Apify)  
- Valid Results (Filter)  
- Has Content? (If)  
- Get Markdown + URL (Set)  
- DeepResearch Learnings (Chain LLM)  
- Research Goal + Learnings (Set)  
- Combine & Send back to Loop (Aggregate)  
- Item Ref (NoOp)  
- Sticky Notes (explanatory)

**Node Details:**

- **Generate SERP Queries**  
  - *Type:* Chain LLM (OpenAI o3-mini)  
  - *Role:* Generates a list of keyword-based search queries (SERP queries) based on the current research prompt and previous learnings.  
  - *Config:* Limits number of queries to `breadth` parameter.  
  - *Output Parser:* Structured Output Parser2.

- **Structured Output Parser2**  
  - *Role:* Parses LLM output into an array of query objects with `query` and `researchGoal`.

- **SERP to Items**  
  - *Type:* Split Out  
  - *Role:* Splits the array of queries into individual items for processing.

- **For Each Query...**  
  - *Type:* Split In Batches  
  - *Role:* Iterates over each query to perform web search and scraping.

- **DeepResearch Subworkflow**  
  - *Type:* Execute Workflow Trigger (self-call)  
  - *Role:* Runs a subworkflow for each query to perform search, scrape, and generate learnings.  
  - *Inputs:* `requestId`, `jobType` = `deepresearch_learnings`, and query data.  
  - *Advanced:* Enables recursive looping by spawning sub-executions.

- **Execution Data**  
  - *Type:* Execution Data  
  - *Role:* Saves execution metadata like `requestId` and `jobType`.

- **Accumulate Results**  
  - *Type:* Set  
  - *Role:* Aggregates all learnings and URLs collected so far across iterations.  
  - *Logic:* Uses `$runIndex` to track recursion depth and accumulates data from previous runs.

- **Is Depth Reached?**  
  - *Type:* If  
  - *Role:* Checks if the recursion depth limit is reached (`should_stop` boolean).  
  - *Branches:* If true, proceeds to report generation; else, continues loop.

- **Get Research Results**  
  - *Type:* Set  
  - *Role:* Prepares accumulated learnings and URLs for report generation.

- **Generate Learnings**  
  - *Type:* Execute Workflow  
  - *Role:* Runs subworkflow to generate learnings from scraped content for each query.  
  - *Waits:* Synchronous execution to gather results.

- **RAG Web Browser**  
  - *Type:* HTTP Request  
  - *Role:* Calls Apify's RAG Web Browser actor to perform web search and scrape results for the query.  
  - *Config:* Filters out certain sites and filetypes to improve relevance.  
  - *Credentials:* Apify API key.  
  - *Edge Cases:* Handles API errors, especially 401 auth errors.

- **Valid Results**  
  - *Type:* Filter  
  - *Role:* Filters results to only those with successful crawl status and non-empty markdown content.

- **Has Content?**  
  - *Type:* If  
  - *Role:* Checks if valid content exists; branches accordingly.

- **Get Markdown + URL**  
  - *Type:* Set  
  - *Role:* Extracts markdown content and source URL from valid results.

- **DeepResearch Learnings**  
  - *Type:* Chain LLM (OpenAI o3-mini)  
  - *Role:* Generates concise learnings from scraped content, including entities and metrics.  
  - *Output Parser:* Structured Output Parser.

- **Research Goal + Learnings**  
  - *Type:* Set  
  - *Role:* Combines research goal, learnings, follow-up questions, and URLs for next iteration.

- **Combine & Send back to Loop**  
  - *Type:* Aggregate  
  - *Role:* Aggregates all items to feed back into the loop for next depth iteration.

- **Item Ref**  
  - *Type:* NoOp  
  - *Role:* Placeholder node to reference current item data.

- **Sticky Notes**  
  - *Content:* Explain recursive looping, web scraping choice (Apify), and LLM usage for query generation and learnings.

**Edge Cases:**  
- API rate limits or auth failures with Apify.  
- Empty or irrelevant search results.  
- Recursive loop misconfiguration causing infinite loops or premature termination.  
- LLM output parsing errors.

---

#### 2.4 Report Generation

**Overview:**  
After completing the recursive research loop, this block compiles all learnings into a detailed markdown report using a reasoning LLM.

**Nodes Involved:**  
- JobType Router  
- DeepResearch Report (Chain LLM)  
- Structured Output Parser  
- Convert to HTML (Markdown node)  
- HTML to Array (Set)  
- Tags to Items (Split Out)  
- Notion Block Generator (Chain LLM)  
- Parse JSON blocks (Set)  
- Valid Blocks (Filter)  
- Sticky Notes (explanatory)

**Node Details:**

- **DeepResearch Report**  
  - *Type:* Chain LLM (OpenAI o3-mini)  
  - *Role:* Generates a detailed, multi-page markdown report based on the user query and all accumulated learnings.  
  - *Prompt:* Instructs LLM to be detailed, use headings, lists, and tables.  
  - *Output Parser:* Structured Output Parser.

- **Structured Output Parser**  
  - *Role:* Parses the LLM output into structured JSON.

- **Convert to HTML**  
  - *Type:* Markdown  
  - *Role:* Converts markdown report into HTML for easier semantic splitting.

- **HTML to Array**  
  - *Type:* Set  
  - *Role:* Extracts HTML blocks (tables, lists, headings) into an array for processing.

- **Tags to Items**  
  - *Type:* Split Out  
  - *Role:* Splits HTML blocks into individual items for conversion.

- **Notion Block Generator**  
  - *Type:* Chain LLM (Google Gemini 2.0)  
  - *Role:* Converts each HTML block into the equivalent Notion API block JSON schema.  
  - *Prompt:* Provides detailed instructions and examples for conversion.  
  - *Credentials:* Google Palm API (Gemini).

- **Parse JSON blocks**  
  - *Type:* Set  
  - *Role:* Parses the JSON string output from the LLM into actual JSON objects.  
  - *Error Handling:* Continues on error with empty blocks.

- **Valid Blocks**  
  - *Type:* Filter  
  - *Role:* Filters out any blocks with errors before uploading.

- **Sticky Notes**  
  - *Content:* Explain markdown to Notion block conversion approach and rationale.

**Edge Cases:**  
- LLM conversion errors or invalid JSON outputs.  
- HTML parsing edge cases with malformed HTML.  
- Google Gemini API quota or connectivity issues.

---

#### 2.5 Report Formatting and Upload

**Overview:**  
Uploads the generated Notion blocks to the previously created Notion page, appending the report content and a list of source URLs.

**Nodes Involved:**  
- Get Existing Row1 (Notion)  
- DeepResearch Report (triggered again)  
- Append Blocks (Merge)  
- URL Sources to Lists (Code)  
- For Each Block... (Split In Batches)  
- Upload to Notion Page (HTTP Request)  
- Set Done (Notion update)  
- Sticky Notes (explanatory)

**Node Details:**

- **Get Existing Row1**  
  - *Type:* Notion (getAll)  
  - *Role:* Retrieves the Notion page created earlier by matching `Request ID`.  
  - *Credentials:* Notion API.

- **Append Blocks**  
  - *Type:* Merge  
  - *Role:* Merges Notion blocks generated from report content and source URLs.

- **URL Sources to Lists**  
  - *Type:* Code  
  - *Role:* Converts all unique source URLs into bulleted list Notion blocks, chunked in groups of 50 to avoid API limits.

- **For Each Block...**  
  - *Type:* Split In Batches  
  - *Role:* Iterates over each Notion block batch for sequential upload.

- **Upload to Notion Page**  
  - *Type:* HTTP Request  
  - *Role:* PATCH request to Notion API to append blocks as children to the report page.  
  - *Config:* Uses Notion API version 2022-06-28, retries on failure with delay.  
  - *Credentials:* Notion API.

- **Set Done**  
  - *Type:* Notion (update)  
  - *Role:* Updates the report page status to "Done" and sets the last updated timestamp.

- **Sticky Notes**  
  - *Content:* Explain the unstable Notion API requiring batching and retries, and manual URL list appending.

**Edge Cases:**  
- Notion API rate limits or partial failures.  
- Large reports exceeding block limits per request.  
- Network timeouts during upload.

---

#### 2.6 Workflow Control and Error Handling

**Overview:**  
Manages routing of execution based on job type, handles API authentication errors, and controls asynchronous execution flow.

**Nodes Involved:**  
- JobType Router (Switch)  
- Is Apify Auth Error? (If)  
- Stop and Error  
- Execution Data  
- Sticky Notes (explanatory)

**Node Details:**

- **JobType Router**  
  - *Type:* Switch  
  - *Role:* Routes execution to appropriate logic blocks based on `jobType` field: `deepresearch_initiate`, `deepresearch_learnings`, or `deepresearch_report`.

- **Is Apify Auth Error?**  
  - *Type:* If  
  - *Role:* Detects HTTP 401 errors from Apify API calls indicating invalid or missing API key.

- **Stop and Error**  
  - *Type:* Stop and Error  
  - *Role:* Stops workflow execution and returns a descriptive error message about Apify authentication.

- **Execution Data**  
  - *Type:* Execution Data  
  - *Role:* Saves metadata for tracking and routing.

- **Sticky Notes**  
  - *Content:* Explain asynchronous job handling via subworkflows and recursive looping.

**Edge Cases:**  
- Missing or invalid API credentials causing workflow termination.  
- Unexpected jobType values causing routing failures.

---

### 3. Summary Table

| Node Name               | Node Type                          | Functional Role                                   | Input Node(s)                      | Output Node(s)                     | Sticky Note                                                                                     |
|-------------------------|----------------------------------|-------------------------------------------------|----------------------------------|----------------------------------|------------------------------------------------------------------------------------------------|
| On form submission      | Form Trigger                     | Entry point capturing user research request     | -                                | Research Request                  |                                                                                                |
| Research Request        | Form                             | Collects research query, depth, breadth          | On form submission               | Set Variables                    |                                                                                                |
| Set Variables           | Set                              | Normalizes inputs, sets request ID                | Research Request                 | Clarifying Questions             |                                                                                                |
| Clarifying Questions    | Chain LLM (OpenAI o3-mini)        | Generates follow-up clarification questions      | Set Variables                   | Structured Output Parser1        |                                                                                                |
| Structured Output Parser1| Output Parser Structured          | Parses follow-up questions                        | Clarifying Questions            | Feedback to Items                |                                                                                                |
| Feedback to Items       | Split Out                        | Splits questions into individual items            | Structured Output Parser1       | For Each Question...             |                                                                                                |
| For Each Question...    | Split In Batches                 | Iterates over each clarification question         | Feedback to Items               | Ask Clarity Questions            |                                                                                                |
| Ask Clarity Questions   | Form                             | Collects user answers to clarification questions | For Each Question...            | On form submission (loop back)  | ## 2. Ask Clarifying Questions: looping dynamic forms for user input collection                |
| JobType Router          | Switch                          | Routes execution by jobType                        | Execution Data                  | Get Existing Row, Generate SERP Queries, Get Existing Row1 |                                                                                                |
| Generate SERP Queries   | Chain LLM (OpenAI o3-mini)        | Generates search queries for web research         | JobType Router (learnings)      | Structured Output Parser2        | ## 7. Generate Search Queries: AI generates keyword-based queries for breadth                  |
| Structured Output Parser2| Output Parser Structured          | Parses SERP queries                               | Generate SERP Queries           | SERP to Items                   |                                                                                                |
| SERP to Items           | Split Out                        | Splits queries into individual items               | Structured Output Parser2       | For Each Query...                |                                                                                                |
| For Each Query...       | Split In Batches                 | Iterates over each search query                    | SERP to Items                  | DeepResearch Subworkflow, Combine & Send back to Loop | ## 6. Perform DeepSearch Loop: recursive web search and scraping loop                          |
| DeepResearch Subworkflow| Execute Workflow Trigger         | Runs subworkflow for each query                     | For Each Query...               | Execution Data                  | ## 14. Subworkflow Event Pattern: recursive looping via subworkflow calls                      |
| Execution Data          | Execution Data                  | Saves execution metadata                            | DeepResearch Subworkflow        | JobType Router                  |                                                                                                |
| Accumulate Results      | Set                              | Aggregates learnings and URLs across iterations    | Generate Learnings              | Is Depth Reached?               |                                                                                                |
| Is Depth Reached?       | If                               | Checks if recursion depth limit reached            | Accumulate Results             | Get Research Results, DeepResearch Results |                                                                                                |
| Get Research Results    | Set                              | Prepares accumulated learnings and URLs            | Is Depth Reached? (true)        | Generate Report                |                                                                                                |
| Generate Learnings      | Execute Workflow                | Generates learnings from scraped content           | Set Next Queries               | Accumulate Results             |                                                                                                |
| RAG Web Browser         | HTTP Request                   | Calls Apify to perform web search and scraping     | Item Ref                      | Is Apify Auth Error?            | ## 8. Web Search and Extracting Web Page Contents using APIFY.com                              |
| Valid Results           | Filter                          | Filters successful and non-empty crawl results     | Is Apify Auth Error? (false)    | Has Content?                   |                                                                                                |
| Has Content?            | If                               | Checks if valid content exists                      | Valid Results                 | Get Markdown + URL, Empty Response |                                                                                                |
| Get Markdown + URL      | Set                              | Extracts markdown and URL from results              | Has Content? (true)             | DeepResearch Learnings          |                                                                                                |
| DeepResearch Learnings  | Chain LLM (OpenAI o3-mini)        | Generates concise learnings from content            | Get Markdown + URL             | Research Goal + Learnings       |                                                                                                |
| Research Goal + Learnings| Set                              | Combines research goal, learnings, follow-ups, URLs| DeepResearch Learnings         | For Each Query...               |                                                                                                |
| Combine & Send back to Loop| Aggregate                      | Aggregates items to feed next iteration             | For Each Query...              | Accumulate Results             |                                                                                                |
| Item Ref                | NoOp                            | Placeholder for current item                         | For Each Query...              | RAG Web Browser                |                                                                                                |
| DeepResearch Report     | Chain LLM (OpenAI o3-mini)        | Generates final detailed markdown report            | Get Existing Row1             | Convert to HTML                | ## 10. Generate DeepSearch Report using Learnings: LLM compiles final report                   |
| Convert to HTML         | Markdown                        | Converts markdown report to HTML                     | DeepResearch Report           | HTML to Array                 | ## 11. Reformat Report as Notion Blocks: convert markdown to HTML for easier parsing          |
| HTML to Array           | Set                              | Extracts HTML blocks (tables, lists, headings)      | Convert to HTML               | Tags to Items                 |                                                                                                |
| Tags to Items           | Split Out                        | Splits HTML blocks into individual items             | HTML to Array                | Notion Block Generator        |                                                                                                |
| Notion Block Generator  | Chain LLM (Google Gemini 2.0)     | Converts HTML blocks to Notion API block JSON        | Tags to Items                | Parse JSON blocks             |                                                                                                |
| Parse JSON blocks       | Set                              | Parses JSON string output into JSON objects          | Notion Block Generator       | Valid Blocks                 |                                                                                                |
| Valid Blocks            | Filter                          | Filters out blocks with errors                        | Parse JSON blocks            | Append Blocks                |                                                                                                |
| Append Blocks           | Merge                           | Merges report blocks and source URL blocks           | Valid Blocks, URL Sources to Lists| For Each Block...          | ## 12. Append URL Sources List: manually compose Notion blocks for URLs                       |
| URL Sources to Lists    | Code                            | Converts source URLs into bulleted list Notion blocks| JobType Router               | Append Blocks                |                                                                                                |
| For Each Block...       | Split In Batches                 | Iterates over Notion blocks for sequential upload    | Append Blocks                | Upload to Notion Page, Set Done|                                                                                                |
| Upload to Notion Page   | HTTP Request                   | Uploads blocks to Notion page via API PATCH          | For Each Block...            | -                            | ## 13. Update Report in Notion: batch upload with retries due to unstable API                 |
| Set Done                | Notion (update)                | Marks report page status as "Done" and updates date  | For Each Block...            | -                            |                                                                                                |
| Is Apify Auth Error?    | If                               | Detects Apify API 401 authentication errors          | RAG Web Browser              | Stop and Error, Valid Results |                                                                                                |
| Stop and Error          | Stop and Error                 | Stops workflow with error message on auth failure    | Is Apify Auth Error? (true)   | -                            |                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node: "On form submission"**  
   - Type: Form Trigger  
   - Path: `/deep_research`  
   - Configure form with title "DeepResearcher" and descriptive HTML instructions.  
   - Set button label to "Next".  
   - Enable "Ignore Bots".  

2. **Create Form Node: "Research Request"**  
   - Collect fields:  
     - Textarea: "What would you like to research?" (required)  
     - Range slider: "depth" (1-5, default 1) with description  
     - Range slider: "breadth" (1-5, default 2) with description  
     - Multiselect dropdown: acknowledgment checkbox (required)  
   - Add branding image in description HTML.  

3. **Create Set Node: "Set Variables"**  
   - Assign variables:  
     - `request_id` = execution ID  
     - `prompt` = user research query  
     - `depth` = parsed number from input-depth or default 1  
     - `breadth` = parsed number from input-breadth or default 1  

4. **Create Chain LLM Node: "Report Page Generator"**  
   - Model: OpenAI o3-mini  
   - Prompt: Generate a suitable title for the report from the query.  
   - Enable structured output parser with schema for `title` and `description`.  

5. **Create Structured Output Parser Node: "Structured Output Parser4"**  
   - Define manual JSON schema for `title` and `description`.  

6. **Create Notion Node: "Create Row"**  
   - Resource: databasePage  
   - Database ID: your Notion DeepResearch database  
   - Set properties:  
     - Title = parsed title from LLM  
     - Description = parsed description from LLM  
     - Status = "Not started"  
     - Request ID = `request_id` variable  
   - Connect Notion API credentials.  

7. **Create Execute Workflow Node: "Initiate DeepResearch"**  
   - Workflow ID: this workflow's ID (self-call)  
   - Mode: each (for multiple items)  
   - Inputs:  
     - `requestId` = `request_id`  
     - `jobType` = "deepresearch_initiate"  
     - `data` = object with initial query, empty learnings, depth, breadth  

8. **Create Form Node: "Confirmation"**  
   - Show confirmation message with link to Notion page (URL from created row).  
   - Button label: "Done".  

9. **Create Form Node: "End Form"**  
   - Completion form with thank-you message.  

10. **Create Chain LLM Node: "Clarifying Questions"**  
    - Model: OpenAI o3-mini  
    - Prompt: Generate up to 3 follow-up questions to clarify research direction.  
    - Enable structured output parser with schema for `questions` array.  

11. **Create Structured Output Parser Node: "Structured Output Parser1"**  
    - Schema: array of strings for questions.  

12. **Create Split Out Node: "Feedback to Items"**  
    - Field to split out: `output.questions`.  

13. **Create Split In Batches Node: "For Each Question..."**  
    - Iterate over each question.  

14. **Create Form Node: "Ask Clarity Questions"**  
    - Show each question as textarea, required.  
    - Collect user answers.  

15. **Loop back answers to "Get Initial Query"**  
    - Create Set Node "Get Initial Query" to combine initial query and Q&A.  

16. **Create Switch Node: "JobType Router"**  
    - Route by `jobType`:  
      - `deepresearch_initiate` → start research loop  
      - `deepresearch_learnings` → generate learnings  
      - `deepresearch_report` → generate final report  

17. **Create Chain LLM Node: "Generate SERP Queries"**  
    - Model: OpenAI o3-mini  
    - Prompt: Generate keyword-based search queries limited by `breadth`.  
    - Output parser: Structured Output Parser2 with array of queries and research goals.  

18. **Create Structured Output Parser Node: "Structured Output Parser2"**  
    - Schema: array of objects with `query` and `researchGoal`.  

19. **Create Split Out Node: "SERP to Items"**  
    - Split queries into individual items.  

20. **Create Split In Batches Node: "For Each Query..."**  
    - Iterate over each query.  

21. **Create Execute Workflow Trigger Node: "DeepResearch Subworkflow"**  
    - Self-call with `jobType` = `deepresearch_learnings` and query data.  

22. **Create Execution Data Node: "Execution Data"**  
    - Save `requestId` and `jobType`.  

23. **Create Set Node: "Accumulate Results"**  
    - Aggregate all learnings and URLs from previous iterations.  
    - Calculate `should_stop` based on depth.  

24. **Create If Node: "Is Depth Reached?"**  
    - Branch based on `should_stop`.  

25. **Create Set Node: "Get Research Results"**  
    - Prepare accumulated learnings and URLs for report generation.  

26. **Create Execute Workflow Node: "Generate Learnings"**  
    - Run subworkflow to generate learnings from scraped content.  

27. **Create HTTP Request Node: "RAG Web Browser"**  
    - POST to Apify RAG Web Browser actor with query.  
    - Filter out unwanted sites and filetypes.  
    - Use Apify API credentials.  

28. **Create Filter Node: "Valid Results"**  
    - Filter results with status "handled" and non-empty markdown.  

29. **Create If Node: "Has Content?"**  
    - Branch based on content presence.  

30. **Create Set Node: "Get Markdown + URL"**  
    - Extract markdown and URL from results.  

31. **Create Chain LLM Node: "DeepResearch Learnings"**  
    - Generate concise learnings from content.  
    - Output parser: Structured Output Parser.  

32. **Create Set Node: "Research Goal + Learnings"**  
    - Combine research goal, learnings, follow-up questions, and URLs.  

33. **Create Aggregate Node: "Combine & Send back to Loop"**  
    - Aggregate items for next iteration.  

34. **Create NoOp Node: "Item Ref"**  
    - Placeholder for current item.  

35. **Create Chain LLM Node: "DeepResearch Report"**  
    - Generate final detailed markdown report from all learnings.  

36. **Create Markdown Node: "Convert to HTML"**  
    - Convert markdown report to HTML.  

37. **Create Set Node: "HTML to Array"**  
    - Extract HTML blocks (tables, lists, headings).  

38. **Create Split Out Node: "Tags to Items"**  
    - Split HTML blocks into items.  

39. **Create Chain LLM Node: "Notion Block Generator"**  
    - Convert HTML blocks to Notion API block JSON using Google Gemini 2.0.  

40. **Create Set Node: "Parse JSON blocks"**  
    - Parse JSON string output to JSON objects.  

41. **Create Filter Node: "Valid Blocks"**  
    - Filter out blocks with errors.  

42. **Create Code Node: "URL Sources to Lists"**  
    - Convert unique URLs into bulleted list Notion blocks, chunked by 50.  

43. **Create Merge Node: "Append Blocks"**  
    - Merge report blocks and URL blocks.  

44. **Create Split In Batches Node: "For Each Block..."**  
    - Iterate over blocks for upload.  

45. **Create HTTP Request Node: "Upload to Notion Page"**  
    - PATCH request to Notion API to append blocks to page.  
    - Use Notion API credentials.  
    - Retry on failure with delay.  

46. **Create Notion Node: "Set Done"**  
    - Update report page status to "Done" and set last updated date.  

47. **Create If Node: "Is Apify Auth Error?"**  
    - Detect HTTP 401 errors from Apify API.  

48. **Create Stop and Error Node: "Stop and Error"**  
    - Stop workflow with error message on Apify auth failure.  

49. **Add Sticky Notes**  
    - Add explanatory sticky notes at key points for documentation and user guidance.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                     | Context or Link                                                                                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| This template replicates OpenAI's DeepResearch feature, enabling multi-step recursive research using n8n, Apify, and OpenAI.                                                                                                   | Workflow description                                                                                             |
| Duplicate this Notion database to use with this workflow: https://jimleuk.notion.site/19486dd60c0c80da9cb7eb1468ea9afd?v=19486dd60c0c805c8e0c000ce8c87acf                                                                         | Notion database template                                                                                         |
| Sign up for Apify API key for web search and scraping: https://www.apify.com?fpr=414q6                                                                                                                                          | Apify API                                                                                                       |
| Use OpenAI's o3-mini model or switch to o1 series if unavailable.                                                                                                                                                               | OpenAI model                                                                                                    |
| The recursive looping technique uses subworkflows to enable asynchronous, multi-step research without blocking the main workflow.                                                                                              | Sticky Note14                                                                                                   |
| The workflow uses Google Gemini 2.0 for HTML to Notion block conversion, which requires Google Palm API credentials.                                                                                                            | Google Gemini API                                                                                                |
| For better markdown to Notion conversion, consider community nodes or external tools like Martian or notionmd.                                                                                                                 | Sticky Note19 with links: https://community.n8n.io/t/now-available-notion-markdown-conversion-community-node/59087, https://github.com/tryfabric/martian, https://github.com/brittonhayes/notionmd |
| Increasing depth and breadth increases runtime and cost exponentially; recommended defaults are depth=1 and breadth=2 for 10-15 minutes runtime.                                                                              | Workflow description                                                                                            |
| The workflow handles Apify authentication errors gracefully by stopping execution and displaying a clear error message.                                                                                                         | Error handling                                                                                                  |
| The Notion API is unstable for block uploads; batching and retry logic are implemented to improve reliability.                                                                                                                 | Sticky Note13 and Sticky Note10                                                                                 |
| Join the n8n Discord or Forum for help: https://discord.com/invite/XPKeKXeB7d, https://community.n8n.io/                                                                                                                        | Community support                                                                                               |

---

This documentation provides a comprehensive understanding of the DeepResearch workflow, enabling advanced users and AI agents to reproduce, modify, and troubleshoot it effectively.