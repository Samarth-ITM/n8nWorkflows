Monitor AI Research Papers with Gemini-powered Filtering and Email Summaries

https://n8nworkflows.xyz/workflows/monitor-ai-research-papers-with-gemini-powered-filtering-and-email-summaries-9859


# Monitor AI Research Papers with Gemini-powered Filtering and Email Summaries

### 1. Workflow Overview

This workflow, titled **"Monitor AI Research Papers with Gemini-powered Filtering and Email Summaries"**, automates the daily monitoring of new AI research papers submitted to ArXiv, specifically in the cs.AI category. Its purpose is to identify, analyze, and summarize the most relevant research papers with actionable insights for AI practitioners, then deliver these summaries via email.

**Target Use Cases:**
- Researchers, AI practitioners, and content strategists who want daily curated updates on AI research with practical takeaways.
- Teams or individuals seeking to stay informed about recent papers with a focus on applied AI research relevant to tool users.
- Automating content curation workflows for newsletters or internal knowledge sharing.

**Logical Blocks:**

- **1.1 Paper Retrieval and Validation**  
  Scheduled daily (Tues-Fri), the workflow queries ArXiv API for new AI papers, validates availability, and stops if no papers are found (e.g., weekends).

- **1.2 Data Extraction and Initial Structuring**  
  Parses the raw XML feed from ArXiv, extracts key metadata (title, abstract, PDF links), and structures the data for AI analysis.

- **1.3 AI-Based Relevance Filtering**  
  Uses Google Gemini-powered LLM to rank and filter papers by relevance based on a detailed prompt targeting actionable research insights for AI tool users.

- **1.4 Detailed Paper Summarization**  
  For the top-ranked papers, the workflow obtains detailed structured summaries including methodology, findings, practical value, and limitations, again using Google Gemini LLM.

- **1.5 Data Merging and Formatting**  
  Merges relevance scores and summaries, aggregates the data, and generates a polished HTML email report that highlights key insights, impact, and links.

- **1.6 Email Dispatch**  
  Sends the curated report to recipients via SMTP email with a custom subject and formatted content.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Paper Retrieval and Validation

**Overview:**  
This block fetches newly submitted AI papers from ArXiv for the previous day (Monday to Friday), ensuring data availability before continuing or stopping with an error message.

**Nodes Involved:**  
- Schedule Trigger  
- Construct API query  
- Get ArXiv Paper Data  
- Convert XML Feed  
- Check Paper Availability  
- Stop and Error  
- Sticky Note (explanatory)

**Node Details:**

- **Schedule Trigger**  
  - Type: Cron-based trigger  
  - Configuration: Runs at 00:06 AM on Tuesday to Friday  
  - Input: None (trigger event)  
  - Output: Triggers next node daily on specified days  
  - Potential Failures: Misconfigured cron expression may prevent triggering

- **Construct API query**  
  - Type: Code (JavaScript)  
  - Role: Builds ArXiv API URL querying AI category (cs.AI) papers submitted between yesterday and today, max 1000 results  
  - Key Expression: Formats dates as YYYYMMDD0400 for query  
  - Input: Trigger data  
  - Output: JSON with API URL  
  - Edge Cases: Date calculation errors, API URL malformed

- **Get ArXiv Paper Data**  
  - Type: HTTP Request  
  - Role: Fetches XML feed from constructed ArXiv API URL  
  - Input: URL from previous node  
  - Output: Raw XML feed of paper metadata  
  - Failures: Network errors, API downtime, rate limiting

- **Convert XML Feed**  
  - Type: XML Parser  
  - Role: Converts XML feed into JSON format  
  - Input: Raw XML from HTTP Request  
  - Output: Structured JSON (ArXiv entries)  
  - Failures: Malformed XML, parsing errors

- **Check Paper Availability**  
  - Type: Switch  
  - Role: Checks if the feed contains any paper entries  
  - Configuration: Branch 1 if no entries (empty), branch 2 if entries exist  
  - Output:  
    - If empty: routes to Stop and Error node  
    - If present: continues to extraction  
  - Failures: Expression errors, unexpected feed structure

- **Stop and Error**  
  - Type: Stop node with error  
  - Role: Stops workflow with a message if no papers found (e.g., weekend)  
  - Parameters: Error message explaining likely weekend or issue  
  - Input: From Switch node if no papers  
  - Output: Terminates workflow gracefully

- **Sticky Note**  
  - Content: Explains this block retrieves paper titles and abstracts, only runs Tues-Fri, stops if no data.

---

#### 2.2 Data Extraction and Initial Structuring

**Overview:**  
This block extracts relevant paper metadata (title, abstract, ArXiv link, PDF link) from the JSON feed to prepare for AI relevance analysis.

**Nodes Involved:**  
- Extract Relevant Data (Python code)  
- Sticky Note (explanatory)

**Node Details:**

- **Extract Relevant Data**  
  - Type: Code (Python)  
  - Role: Iterates over feed entries, extracts title, abstract, arXiv ID, and direct PDF link  
  - Variables: Iterates over `feed.entry`, looks for `link` with pdf MIME type or title "pdf"  
  - Input: JSON from XML parser  
  - Output: JSON containing array of cleaned paper entries under `entries`  
  - Edge Cases: Missing PDF links, unexpected feed formats, large number of papers causing performance issues

- **Sticky Note**  
  - Content: Marks this block as responsible for extracting paper data and preparing for LLM relevance analysis.

---

#### 2.3 AI-Based Relevance Filtering

**Overview:**  
Uses Google Gemini language model with a sophisticated prompt to rank papers by practical relevance, selecting exactly 3 papers with highest actionable value.

**Nodes Involved:**  
- Google Gemini Chat Model  
- Structured Output Parser  
- Paper Relevance Analyzer (LangChain Agent)  
- Split Data Fragments  
- Sticky Note (explanatory)

**Node Details:**

- **Google Gemini Chat Model**  
  - Type: LLM Chat (Google Gemini)  
  - Role: Receives prompt with paper list, outputs JSON array with 3 ranked papers and detailed relevance data  
  - Model: `models/gemini-2.5-flash-lite`  
  - Credentials: Google PaLM API key  
  - Input: JSON stringified paper entries from extraction node  
  - Output: Raw LLM response with JSON array of 3 papers  
  - Edge Cases: API quota limits, malformed prompt input, LLM output parsing errors, rate limits

- **Structured Output Parser**  
  - Type: LangChain Output Parser  
  - Role: Parses LLM JSON output according to schema with fields like title, abstract, relevance_score, key_insight, etc.  
  - Input: Raw text from Gemini node  
  - Output: Parsed structured JSON array  
  - Edge Cases: Parsing failures if LLM returns invalid JSON or incomplete data

- **Paper Relevance Analyzer**  
  - Type: LangChain Agent  
  - Role: Wrapper connecting Gemini LLM and output parser, handles retry on failure and prompt logic  
  - Input: Extracted paper entries  
  - Output: Parsed and ranked paper list  
  - Edge Cases: Failures in LLM call, output parser, or empty results

- **Split Data Fragments**  
  - Type: Split Out  
  - Role: Splits array of 3 papers into individual items for downstream processing (summarization and merging)  
  - Input: Parsed array from relevance analyzer  
  - Output: One item per paper  
  - Edge Cases: Empty input array, splitting errors

- **Sticky Note**  
  - Content: Marks this block as responsible for LLM-powered relevance ranking and filtering.

---

#### 2.4 Detailed Paper Summarization

**Overview:**  
For each top-ranked paper, this block fetches the paper PDF and performs a detailed summary extraction via Google Gemini LLM, then formats the result into a strict JSON schema.

**Nodes Involved:**  
- Paper Summarizer (Google Gemini LLM)  
- Structured Output Parser1  
- Google Gemini Chat Model1  
- Summary Formatter (LangChain Agent)  
- Sticky Note (explanatory)

**Node Details:**

- **Paper Summarizer**  
  - Type: Google Gemini LLM (LangChain Google Gemini node variant)  
  - Role: Summarizes paper by analyzing PDF content URL with a prompt focusing on approach, key findings, value, limitations, and bottom line  
  - Model: `models/gemini-flash-lite-latest`  
  - Input: Paper PDF URL and ArXiv ID from previous split node  
  - Output: Raw text summary structured by numbered sections  
  - Edge Cases: PDF fetch failures, LLM timeouts, incomplete summaries

- **Google Gemini Chat Model1**  
  - Type: Google Gemini Chat Model  
  - Role: Used by Summary Formatter for parsing summary text into structured JSON  
  - Input: Raw text from Paper Summarizer  
  - Output: Parsed text for next node  
  - Edge Cases: Parsing failures, API issues

- **Structured Output Parser1**  
  - Type: LangChain Output Parser  
  - Role: Parses detailed summary text into JSON with fields: arxiv_id, approach, key_findings, value, limitations, bottom_line  
  - Input: Raw summary text  
  - Output: Structured JSON summary  
  - Edge Cases: Parsing errors if LLM output is malformed

- **Summary Formatter**  
  - Type: LangChain Agent  
  - Role: Converts raw text summary into final JSON format strictly adhering to defined schema  
  - Input: Text summary from Paper Summarizer  
  - Output: JSON object array containing structured summary for each paper  
  - Edge Cases: Formatting mistakes, output schema mismatches

- **Sticky Note**  
  - Content: Marks this block as focused on summarizing and structuring paper summaries via LLM calls.

---

#### 2.5 Data Merging and Formatting

**Overview:**  
This block merges the ranked papers with their detailed summaries, aggregates all data, and generates a polished HTML email report with key insights, scores, and links.

**Nodes Involved:**  
- Merge Paper Data with Summaries  
- Create Aggregate  
- Create HTML Template  
- Sticky Note (explanatory)

**Node Details:**

- **Merge Paper Data with Summaries**  
  - Type: Merge (enrich input1)  
  - Role: Joins original ranked paper data with detailed summary JSON using `arxiv_id` as key  
  - Input: Paper ranking items and summary items (split outputs)  
  - Output: Combined paper objects including scores, insights, and summaries  
  - Edge Cases: Missing matching IDs, data inconsistencies

- **Create Aggregate**  
  - Type: Aggregate  
  - Role: Combines array of merged paper objects into a single aggregated item for email generation  
  - Output: Single aggregated JSON object with all paper data  
  - Edge Cases: Empty inputs

- **Create HTML Template**  
  - Type: Code (JavaScript)  
  - Role: Generates full HTML email body with styling, iterating over aggregated papers to render titles, scores, insights, and summary sections; also handles papers with processing errors  
  - Input: Aggregated paper data  
  - Output: JSON with `html` field containing the full email HTML content  
  - Edge Cases: XSS risks mitigated by escaping, formatting errors, empty data arrays

- **Sticky Note**  
  - Content: Notes this block creates the output format and sends the email.

---

#### 2.6 Email Dispatch

**Overview:**  
Sends the generated daily ArXiv paper report email with the curated content to recipients.

**Nodes Involved:**  
- Send email

**Node Details:**

- **Send email**  
  - Type: SMTP Email Send  
  - Role: Sends email with subject "Your daily ArXiv Paper report by @OfficeOptOut" and HTML body generated in previous node  
  - Credentials: SMTP account configured  
  - Input: Email content JSON with `html` field  
  - Output: Email sent status  
  - Edge Cases: SMTP authentication failures, email delivery issues, malformed HTML causing rendering problems

---

### 3. Summary Table

| Node Name                   | Node Type                                   | Functional Role                                  | Input Node(s)                | Output Node(s)                       | Sticky Note                                                                                     |
|-----------------------------|---------------------------------------------|-------------------------------------------------|------------------------------|------------------------------------|------------------------------------------------------------------------------------------------|
| Schedule Trigger            | n8n-nodes-base.scheduleTrigger              | Triggers workflow on Tue-Fri mornings            | None                         | Construct API query                 |                                                                                                |
| Construct API query         | n8n-nodes-base.code                         | Builds ArXiv API URL for previous day's papers   | Schedule Trigger             | Get ArXiv Paper Data               |                                                                                                |
| Get ArXiv Paper Data        | n8n-nodes-base.httpRequest                   | Fetches ArXiv XML feed                            | Construct API query          | Convert XML Feed                   |                                                                                                |
| Convert XML Feed            | n8n-nodes-base.xml                          | Parses XML to JSON                                | Get ArXiv Paper Data         | Check Paper Availability           |                                                                                                |
| Check Paper Availability    | n8n-nodes-base.switch                       | Branches if papers exist or not                   | Convert XML Feed             | Stop and Error / Extract Relevant Data |                                                                                                |
| Stop and Error              | n8n-nodes-base.stopAndError                 | Stops workflow if no papers found                 | Check Paper Availability     | None                             |                                                                                                |
| Extract Relevant Data       | n8n-nodes-base.code (Python)                 | Extracts title, abstract, PDF link from feed     | Check Paper Availability     | Paper Relevance Analyzer           |                                                                                                |
| Google Gemini Chat Model    | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | LLM ranks papers by relevance                      | Extract Relevant Data        | Paper Relevance Analyzer (ai_languageModel) |                                                                                                |
| Structured Output Parser    | @n8n/n8n-nodes-langchain.outputParserStructured | Parses LLM JSON output for ranked papers          | Google Gemini Chat Model     | Paper Relevance Analyzer (ai_outputParser) |                                                                                                |
| Paper Relevance Analyzer    | @n8n/n8n-nodes-langchain.agent              | Combines LLM and parser to produce ranked papers | Extract Relevant Data, Gemini nodes | Split Data Fragments              |                                                                                                |
| Split Data Fragments        | n8n-nodes-base.splitOut                      | Splits array of 3 papers into individual items   | Paper Relevance Analyzer     | Merge Paper Data with Summaries, Paper Summarizer |                                                                                                |
| Paper Summarizer            | @n8n/n8n-nodes-langchain.googleGemini       | Summarizes each paper's PDF into structured text | Split Data Fragments         | Summary Formatter                 |                                                                                                |
| Google Gemini Chat Model1   | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Supports parsing summaries                         | Paper Summarizer             | Summary Formatter (ai_languageModel) |                                                                                                |
| Structured Output Parser1   | @n8n/n8n-nodes-langchain.outputParserStructured | Parses summary text into structured JSON          | Google Gemini Chat Model1    | Summary Formatter (ai_outputParser) |                                                                                                |
| Summary Formatter           | @n8n/n8n-nodes-langchain.agent              | Formats summary text strictly to JSON schema     | Paper Summarizer, Gemini nodes | Merge Paper Data with Summaries   |                                                                                                |
| Merge Paper Data with Summaries | n8n-nodes-base.merge                      | Merges paper ranking and detailed summaries      | Split Data Fragments, Summary Formatter | Create Aggregate               |                                                                                                |
| Create Aggregate            | n8n-nodes-base.aggregate                     | Aggregates merged paper data for email generation | Merge Paper Data with Summaries | Create HTML Template             |                                                                                                |
| Create HTML Template        | n8n-nodes-base.code                          | Generates full HTML email content                  | Create Aggregate            | Send email                       |                                                                                                |
| Send email                 | n8n-nodes-base.emailSend                     | Sends the curated ArXiv report email               | Create HTML Template         | None                             |                                                                                                |
| Sticky Note                | n8n-nodes-base.stickyNote                     | Provides explanations for paper retrieval block    | None                        | None                             | "## Get Paper Data from ArXiv - Retrieve Paper Titles and Abstracts for the current day - Only schedules from Tuesday to Friday, as no uploads during the weekend - Workflow will stop if no paper data is fetched (i.e. for weekends)" |
| Sticky Note1               | n8n-nodes-base.stickyNote                     | Explains extraction and relevance analysis block  | None                        | None                             | "## Extract paper data and analyze paper relevance via LLM Call"                               |
| Sticky Note2               | n8n-nodes-base.stickyNote                     | Explains summarization and formatting block       | None                        | None                             | "## Summarize & structure summary output via LLM calls"                                       |
| Sticky Note3               | n8n-nodes-base.stickyNote                     | Explains output formatting and email sending block | None                        | None                             | "## Create output format and send via Email"                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Configure: Cron expression `6 0 * * 2-5` (6 minutes after midnight, Tuesday to Friday)  
   - This triggers the workflow only on working days to avoid empty data on weekends.

2. **Create Construct API query Node (Code, JavaScript)**  
   - Connect from Schedule Trigger  
   - Code: Calculate `yesterday` and `today` dates, format as `YYYYMMDD0400`, construct ArXiv API URL:  
     ```
     https://export.arxiv.org/api/query?search_query=cat:cs.AI+AND+submittedDate:[<yesterday>+TO+<today>]&max_results=1000
     ```  
   - Output JSON with key `url`.

3. **Create Get ArXiv Paper Data Node (HTTP Request)**  
   - Connect from Construct API query  
   - Set URL: `={{ $json.url }}`  
   - Method: GET  
   - No authentication  
   - Retrieve XML feed of daily AI papers.

4. **Create Convert XML Feed Node (XML Parser)**  
   - Connect from Get ArXiv Paper Data  
   - Default parsing options.

5. **Create Check Paper Availability Node (Switch)**  
   - Connect from Convert XML Feed  
   - Condition: Check if `feed.entry` exists and is not empty array  
   - If empty: route to Stop and Error node  
   - If not empty: route to Extract Relevant Data node

6. **Create Stop and Error Node**  
   - Connect from Check Paper Availability (empty branch)  
   - Set error message: "No Paper were retrieved from ArXiv. It's probably the weekend. Otherwise have a look what went wrong!"  
   - Stops workflow if no papers.

7. **Create Extract Relevant Data Node (Code, Python)**  
   - Connect from Check Paper Availability (non-empty branch)  
   - Python code to loop over `feed.entry`, extract `title`, `summary` (abstract), ArXiv id, and PDF link from links array  
   - Return JSON with array `entries`.

8. **Create Google Gemini Chat Model Node**  
   - Connect from Extract Relevant Data  
   - Use LangChain Google Gemini node type  
   - Model: `models/gemini-2.5-flash-lite`  
   - Credentials: Google PaLM API key  
   - Prompt: Provide JSON stringified paper entries and detailed instructions to rank and select exactly 3 papers with relevance scores and actionable insights.

9. **Create Structured Output Parser Node**  
   - Connect from Google Gemini Chat Model (ai_outputParser)  
   - Schema: Manual JSON schema with fields: title, abstract, arxiv_id, pdf_link, relevance_score, key_insight, expected_impact  
   - Parses LLM JSON response into structured array.

10. **Create Paper Relevance Analyzer Node (LangChain Agent)**  
    - Connect inputs from Extract Relevant Data (text) and Google Gemini Chat Model + Structured Output Parser (ai_languageModel + ai_outputParser)  
    - Handles retry logic and output parsing.

11. **Create Split Data Fragments Node**  
    - Connect from Paper Relevance Analyzer  
    - Field to split: `output` (array of 3 papers)  
    - Splits array into individual paper items.

12. **Create Paper Summarizer Node (Google Gemini LLM)**  
    - Connect from Split Data Fragments  
    - Model: `models/gemini-flash-lite-latest`  
    - Input: Paper PDF link and ArXiv ID  
    - Prompt: Summarize paper with sections: Approach, Key Findings, Practical Value, Limitations, Bottom Line.

13. **Create Google Gemini Chat Model1 Node**  
    - Connect from Paper Summarizer (ai_languageModel)  
    - Used for formatting summary.

14. **Create Structured Output Parser1 Node**  
    - Connect from Google Gemini Chat Model1 (ai_outputParser)  
    - Schema: Manual JSON with fields for detailed summary (arxiv_id, approach, key_findings, value, limitations, bottom_line).

15. **Create Summary Formatter Node (LangChain Agent)**  
    - Connect inputs from Paper Summarizer, Google Gemini Chat Model1, Structured Output Parser1  
    - Formats raw summary into strict JSON array.

16. **Create Merge Paper Data with Summaries Node (Merge)**  
    - Connect from Split Data Fragments (input 1) and Summary Formatter (input 2)  
    - Merge mode: Enrich Input1  
    - Merge by field: `arxiv_id` (node1) = `output[0].arxiv_id` (node2)  
    - Combines ranked paper data with detailed summaries.

17. **Create Create Aggregate Node (Aggregate)**  
    - Connect from Merge Paper Data with Summaries  
    - Aggregate all items into single data object for email.

18. **Create Create HTML Template Node (Code, JavaScript)**  
    - Connect from Create Aggregate  
    - Generate HTML email body: includes date, paper titles, scores, key insights, expected impact, video ideas (if any), and summary sections with styling and error handling  
    - Escapes HTML to avoid injection  
    - Produces JSON with `html` field.

19. **Create Send Email Node (SMTP)**  
    - Connect from Create HTML Template  
    - Configure SMTP credentials  
    - Subject: "Your daily ArXiv Paper report by @OfficeOptOut"  
    - HTML content: `={{ $json.html }}`  
    - Sends curated email.

20. **Add Sticky Notes**  
    - Add explanatory sticky notes above each logical block for documentation clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                             |
|----------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------|
| Workflow only runs Tue-Fri due to no new ArXiv submissions on weekends, with graceful stop on empty data             | Sticky Note near Schedule Trigger and Stop and Error nodes                 |
| Uses Google Gemini (PaLM) LLM models for both relevance ranking and summarization with detailed prompting            | Google PaLM API credentials required; see Google Gemini Chat Model nodes   |
| Output email is richly formatted HTML designed for clarity and ease of reading with sections for insights and impact | Code node "Create HTML Template" contains detailed HTML/CSS email markup   |
| Workflow designed to select exactly 3 papers per run based on actionable relevance for AI tool users                  | Paper Relevance Analyzer prompt enforces selection count and criteria      |
| This workflow can be adapted for other ArXiv categories by modifying the API query construction code                  | See Construct API query node for category filter                           |

---

**Disclaimer:** The provided content originates exclusively from an automated workflow created with n8n, respecting all applicable content policies and containing only legal, public data.