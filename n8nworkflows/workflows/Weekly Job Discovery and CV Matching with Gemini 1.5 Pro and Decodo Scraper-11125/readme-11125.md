Weekly Job Discovery and CV Matching with Gemini 1.5 Pro and Decodo Scraper

https://n8nworkflows.xyz/workflows/weekly-job-discovery-and-cv-matching-with-gemini-1-5-pro-and-decodo-scraper-11125


# Weekly Job Discovery and CV Matching with Gemini 1.5 Pro and Decodo Scraper

---
### 1. Workflow Overview

This workflow automates weekly job discovery and candidate-job matching by scraping job listings from specified platforms (LinkedIn or Indeed) for a given topic and region, summarizing job descriptions with AI, and matching them against a candidate’s CV using Google Gemini AI models. It produces a weekly email report highlighting relevant job opportunities and insights.

The logic is divided into three main blocks:

- **1.1 Job Discovery & Search Configuration**  
  Defines search parameters, triggers scheduled weekly, and uses the Decodo API to perform Google job searches filtered by region and platform.

- **1.2 Job Page Scraping & Text Cleaning**  
  Iterates over each discovered job URL, scrapes the full HTML page using Decodo, and cleans the HTML content to extract plain text suitable for AI summarization.

- **1.3 AI Summarization, Matching & Reporting**  
  Uses AI to extract key job details from the cleaned text, aggregates all summaries, matches jobs to the candidate’s CV using an AI agent, and emails a structured weekly report.

---

### 2. Block-by-Block Analysis

#### 2.1 Job Discovery & Search Configuration

- **Overview:**  
  This block runs once per week (Saturday) to configure the job search query, perform the search via Decodo’s Google Search API, and parse the raw results into a structured list of job URLs and metadata.

- **Nodes Involved:**  
  - Schedule Trigger1  
  - Set Search Config1  
  - Google Search Jobs1  
  - Parse Search Results1  
  - Loop Job URLs1 (entry point for next block)  
  - Sticky Note10 (documentation)

- **Node Details:**  

  - **Schedule Trigger1**  
    - Type: scheduleTrigger  
    - Role: Triggers the workflow weekly on Saturdays.  
    - Configuration: Interval set to every 1 week, trigger at day 6 (Saturday).  
    - Inputs: None (trigger node).  
    - Outputs: Triggers "Set Search Config1".  
    - Edge Cases: Workflow only triggers weekly; misconfiguration could cause no runs.

  - **Set Search Config1**  
    - Type: set  
    - Role: Defines the search parameters as JSON including topic, regions, platforms, and search terms.  
    - Configuration: Raw JSON input with placeholders for topic and region, default platform "linkedin.com", and common job search terms like "hiring", "vacancy".  
    - Inputs: Trigger from Schedule Trigger1.  
    - Outputs: Provides config data to "Google Search Jobs1".  
    - Edge Cases: If placeholders are not replaced, search may be ineffective.

  - **Google Search Jobs1**  
    - Type: Decodo Google Search node  
    - Role: Performs a Google search using Decodo API with configured parameters.  
    - Configuration: Queries Google with topic + search terms + site filter + region + date filter (after 2025-10-01), uses Decodo credentials.  
    - Inputs: Receives search config JSON.  
    - Outputs: Raw search results JSON to "Parse Search Results1".  
    - Edge Cases: API quota exceeded, invalid credentials, network timeouts.

  - **Parse Search Results1**  
    - Type: code  
    - Role: Extracts organic search results from Decodo JSON response into individual items with job URL, title, description, and metadata.  
    - Configuration: JavaScript code handles different Decodo JSON structures for organic results.  
    - Inputs: Raw search results.  
    - Outputs: List of job URL items to "Loop Job URLs1".  
    - Edge Cases: Empty or malformed results, missing fields.

  - **Loop Job URLs1**  
    - Type: splitInBatches  
    - Role: Processes job URLs one at a time downstream for scraping.  
    - Configuration: Batch size = 1.  
    - Inputs: Parsed job URLs.  
    - Outputs: Each job URL item to scraping block.  
    - Edge Cases: Large number of URLs could cause long processing times.

  - **Sticky Note10**  
    - Content:  
      "Group A:  
       defines target search parameters and uses the Decodo API (requires credentials) to search for job listings. It then filters out invalid entries to generate a clean list of URLs for processing."

---

#### 2.2 Job Page Scraping & Text Cleaning

- **Overview:**  
  Iterates through each job URL, scrapes the full job posting page HTML using Decodo (handles proxying), and cleans the HTML to remove scripts, styles, and tags to produce clean plain text for AI summarization.

- **Nodes Involved:**  
  - Loop Job URLs1 (input)  
  - Scrape Page HTML1  
  - Clean HTML Text1  
  - AI Summarizer1  
  - Sticky Note (no number)

- **Node Details:**

  - **Scrape Page HTML1**  
    - Type: Decodo web scrape node  
    - Role: Downloads the full HTML content of the job listing page from the URL.  
    - Configuration: Uses the URL from the current item, default Decodo credentials for scraping.  
    - Inputs: Single job URL item from Loop Job URLs1.  
    - Outputs: Scraped HTML content JSON to Clean HTML Text1.  
    - Edge Cases: Site blocks, captchas, proxy failures, page not found errors.

  - **Clean HTML Text1**  
    - Type: code  
    - Role: Cleans the raw HTML string by removing <script>, <style>, <noscript>, and all HTML tags, replaces HTML entities, and normalizes whitespace.  
    - Configuration: JavaScript runs once per item, outputs a cleaned plain text string under `text_clean`.  
    - Inputs: Scraped HTML JSON.  
    - Outputs: Cleaned text JSON to AI Summarizer1.  
    - Edge Cases: Unexpected HTML structure, missing content.

  - **AI Summarizer1**  
    - Type: langchain chainSummarization  
    - Role: Uses Google Gemini 1.5 Pro to generate a concise, structured summary of key job listing details from the cleaned text.  
    - Configuration: Custom prompt guiding the model to extract company, job title, location, employment type, key responsibilities, skills, qualifications, posted date, salary/benefits. Output is short human-readable text (not JSON).  
    - Inputs: Cleaned job text.  
    - Outputs: Single summary text per job.  
    - Edge Cases: Model failures, rate limits, irrelevant or incomplete summaries.

  - **Sticky Note**  
    - Content:  
      "Group B:  
       goes through each job URL to scrape the full page HTML using Decodo, ensuring proxies handle any blocking issues. It then strips out all tags and scripts to produce clean, plain text for analysis."

---

#### 2.3 AI Summarization, Matching & Reporting

- **Overview:**  
  Collects all individual job summaries, aggregates them into a single JSON array, then uses an AI agent to deeply analyze the candidate’s CV and match jobs to the candidate profile with detailed reasoning, finally sending a plain text email report.

- **Nodes Involved:**  
  - Aggregate Summaries1  
  - Job Matcher Agent1  
  - Send Email Report1  
  - Sticky Note1

- **Node Details:**  

  - **Aggregate Summaries1**  
    - Type: code  
    - Role: Collects all job summaries from the AI summarizer into a single JSON array for batch processing.  
    - Configuration: Extracts the summary text from each item, returns one item containing all summaries as an array under `summaries`.  
    - Inputs: Multiple summary items from AI Summarizer1.  
    - Outputs: Aggregated summaries to Job Matcher Agent1.  
    - Edge Cases: Empty input array, malformed summaries.

  - **Job Matcher Agent1**  
    - Type: langchain agent (Google Gemini Chat Model)  
    - Role: AI personal job-matching analyst that receives:  
      1. The full CV text (injected via expression in the node).  
      2. The aggregated job listings from one region and platform.  
      It analyzes the CV for skills, role fit, seniority, then matches jobs with a computed match percentage, filters out low matches, and generates a structured weekly report including candidate overview, matched jobs list, and key insights.  
    - Configuration:  
      - Model: Google Gemini 1.5 Pro (linked as ai_languageModel for this node).  
      - Prompt: Detailed multi-step instructions for analysis, match scoring, and report formatting.  
      - Input CV text is dynamically injected via expression `{{ $json.cv_text }}`.  
      - Output is a plain text report.  
    - Inputs: Aggregated job summaries.  
    - Outputs: Final report text to Send Email Report1.  
    - Edge Cases: Model rate limits, incomplete CV input, inconsistent job data, prompt failures.

  - **Send Email Report1**  
    - Type: Gmail node  
    - Role: Sends the final weekly job matching report to the configured email address.  
    - Configuration:  
      - Recipient email must be replaced by user.  
      - Subject includes current date dynamically.  
      - Email body is plain text from AI agent output.  
      - Gmail OAuth2 credentials required.  
    - Inputs: Text report from Job Matcher Agent1.  
    - Outputs: None (email sent).  
    - Edge Cases: Email sending errors, invalid credentials, rate limits.

  - **Sticky Note1**  
    - Content:  
      "Group C:  
       uses AI and takes all the raw job postings converts them into key details. It then collects all the individual summaries and aggregates them into a single list for the final analysis and prepare them to be emailed."

---

### 3. Summary Table

| Node Name                   | Node Type                             | Functional Role                                         | Input Node(s)             | Output Node(s)            | Sticky Note                                            |
|-----------------------------|-------------------------------------|---------------------------------------------------------|---------------------------|---------------------------|--------------------------------------------------------|
| Schedule Trigger1            | scheduleTrigger                     | Weekly trigger to start the workflow                    | None                      | Set Search Config1         |                                                        |
| Set Search Config1           | set                                 | Defines job search parameters (topic, region, platforms) | Schedule Trigger1          | Google Search Jobs1        | Group A: defines target search parameters and uses Decodo API |
| Google Search Jobs1          | Decodo Google Search node           | Performs Google job search using Decodo API             | Set Search Config1         | Parse Search Results1      | Group A: defines target search parameters and uses Decodo API |
| Parse Search Results1        | code                               | Parses raw search results JSON into job URL items       | Google Search Jobs1        | Loop Job URLs1             | Group A: defines target search parameters and uses Decodo API |
| Loop Job URLs1               | splitInBatches                     | Processes job URLs one by one                            | Parse Search Results1      | Scrape Page HTML1, AI Summarizer1 | Group B: scrapes full pages and cleans text          |
| Scrape Page HTML1            | Decodo web scraper node             | Scrapes full HTML from job listing URLs                  | Loop Job URLs1             | Clean HTML Text1           | Group B: scrapes full pages and cleans text            |
| Clean HTML Text1             | code                               | Cleans HTML content removing tags, scripts, styles      | Scrape Page HTML1          | Loop Job URLs1             | Group B: scrapes full pages and cleans text            |
| AI Summarizer1               | langchain chainSummarization        | Summarizes cleaned job text into key job details         | Clean HTML Text1           | Aggregate Summaries1       | Group C: AI extracts key details and prepares data     |
| Aggregate Summaries1         | code                               | Aggregates all job summaries into a single array         | AI Summarizer1             | Job Matcher Agent1         | Group C: AI extracts key details and prepares data     |
| Job Matcher Agent1           | langchain agent (Google Gemini)     | Matches candidate CV to jobs, generates weekly report    | Aggregate Summaries1       | Send Email Report1         | Group C: AI extracts key details and prepares data     |
| Send Email Report1           | Gmail                              | Sends the final email report to user                      | Job Matcher Agent1         | None                      |                                                        |
| Sticky Note10                | stickyNote                         | Documentation on Group A                                  | None                      | None                      | Group A: defines target search parameters and uses Decodo API |
| Sticky Note                  | stickyNote                         | Documentation on Group B                                  | None                      | None                      | Group B: scrapes full pages and cleans text            |
| Sticky Note1                 | stickyNote                         | Documentation on Group C                                  | None                      | None                      | Group C: AI extracts key details and prepares data     |
| Sticky Note15                | stickyNote                         | Instructions for setting up Decodo credentials           | None                      | None                      | See section 5 for details                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node**  
   - Type: `scheduleTrigger`  
   - Configure to run every week on Saturdays (interval: 1 week, triggerAtDay: 6).

2. **Create a Set Node for Search Configuration**  
   - Type: `set`  
   - Mode: Raw JSON  
   - Define JSON with keys:  
     ```json
     {
       "topic": "INSERT_TOPIC_HERE",
       "regions": ["INSERT_REGION_HERE"],
       "platforms": ["linkedin.com"],
       "search_terms": ["hiring", "vacancy", "job opening", "career"]
     }
     ```  
   - Connect output of Schedule Trigger node here.

3. **Add Decodo Google Search Node**  
   - Type: `@decodo/n8n-nodes-decodo.decodo`  
   - Operation: google_search  
   - Geo: `={{ $json.regions[0] }}`  
   - Query: `={{$json.topic}} {{$json.search_terms}} site:{{ $json.platforms[0] }} {{ $json.regions[0] }} after:2025-10-01`  
   - Use Decodo API credentials (setup per instructions in section 5).  
   - Connect output of Set Search Config node here.

4. **Add Code Node to Parse Search Results**  
   - Type: `code`  
   - Paste JavaScript to extract organic results safely, mapping to items with fields: url, title, desc, pos, url_shown, favicon_text, region, platform.  
   - Connect output of Decodo Google Search node here.

5. **Add SplitInBatches Node for Job URLs**  
   - Type: `splitInBatches`  
   - Batch Size: 1  
   - Connect output of Parse Search Results node here.

6. **Add Decodo Scrape Node to Scrape Job Pages**  
   - Type: `@decodo/n8n-nodes-decodo.decodo`  
   - Operation: default scrape (or configure to fetch full HTML)  
   - URL: `={{ $json.url }}`  
   - Connect output of SplitInBatches node here.

7. **Add Code Node to Clean HTML Text**  
   - Type: `code`  
   - JavaScript to strip scripts, styles, noscript, all HTML tags, decode entities, and normalize whitespace, outputting cleaned text under `text_clean`.  
   - Connect output of Scrape Page HTML node here.

8. **Add Langchain Chain Summarization Node**  
   - Type: `@n8n/n8n-nodes-langchain.chainSummarization`  
   - Use Google Gemini 1.5 Pro model (configure Langchain credentials accordingly).  
   - Prompt: Custom prompt instructing to extract key fields (company, job title, location, employment type, responsibilities, skills, qualifications, posted date, salary) and produce a short human-readable summary under 8-10 lines (see node details for exact prompt).  
   - Connect output of Clean HTML Text node here.

9. **Add Code Node to Aggregate Summaries**  
   - Type: `code`  
   - JavaScript that collects all incoming items and produces a single JSON object with `summaries` array.  
   - Connect output of AI Summarizer node here.

10. **Add Langchain Agent Node for Job Matching**  
    - Type: `@n8n/n8n-nodes-langchain.agent`  
    - Model: Google Gemini 1.5 Pro (linked in Workflow as Google Gemini Chat Model (Agent))  
    - Input:  
      - Candidate CV text via expression: `{{ $json.cv_text }}` (must be injected from input or another prior node with CV text).  
      - Aggregated job summaries JSON.  
    - Prompt: Detailed instructions to analyze CV, match jobs with a computed match percentage, exclude below 50%, and generate a plain text weekly report with candidate overview, job matches list, and insights.  
    - Connect output of Aggregate Summaries node here.

11. **Add Gmail Node to Send Email**  
    - Type: `gmail`  
    - Authentication: OAuth2 configured with Gmail account.  
    - Send To: Replace with your email address.  
    - Subject: `Weekly Job Listing Report – LinkedIn/Indeed Hiring Trends ({{ new Date().toLocaleDateString() }})`  
    - Message: `={{ $json.output }}` (output of Job Matcher Agent node)  
    - Email type: text/plain  
    - Connect output of Job Matcher Agent node here.

12. **Configure Credentials**  
    - Decodo API Credentials: Create new credential of type "Decodo Credentials API" in n8n and paste your Decodo token (see section 5).  
    - Google Gemini / Langchain: Configure OpenAI or Google Gemini credentials as needed, ensuring model `models/gemini-1.5-pro` is accessible.  
    - Gmail OAuth2: Create credentials in n8n for Gmail with required scopes for sending email.

13. **Test Workflow**  
    - Replace placeholders for topic, region, and CV text with real data.  
    - Run manually or wait for scheduled trigger.  
    - Monitor logs for errors (API limits, parsing issues).

---

### 5. General Notes & Resources

| Note Content                                                                                                                   | Context or Link                                                                                                                              |
|-------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------|
| How to Set Up **Decodo** Credentials in n8n: Activate plan, copy token from Decodo dashboard, create Decodo Credentials API in n8n, paste token. | Detailed instructions and credentials setup: github.com/Decodo/n8n-nodes-decodo/tree/main                                                    |
| Workflow uses Google Gemini 1.5 Pro model via Langchain nodes; ensure API access and quota is available.                        | Requires Google Gemini API access and Langchain integration within n8n                                                                        |
| Gmail node requires OAuth2 credentials with send email scope. Use your Gmail account or Google Workspace.                      | Gmail OAuth2 setup in n8n credential manager                                                                                                |
| The workflow expects a CV text input to be injected into the Job Matcher Agent node (`{{ $json.cv_text }}`); ensure this is provided. | CV input can be from webhook or manual entry before running the workflow                                                                     |
| The search query includes a date filter "after:2025-10-01" which is a future date placeholder—update to current date to get valid results. | Modify the date in "Google Search Jobs1" query parameter to reflect realistic recent job postings                                            |
| The overall process respects API rate limits and proxy handling as performed internally by Decodo nodes.                       | Failure to handle proxies or exceeding API quotas can cause node failures                                                                     |

---

**Disclaimer:** The provided text is generated exclusively from an automated n8n workflow. It strictly adheres to content policies and contains no illegal or offensive elements. All data handled is legal and public.