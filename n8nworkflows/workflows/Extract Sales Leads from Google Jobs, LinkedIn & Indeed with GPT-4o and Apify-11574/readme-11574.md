Extract Sales Leads from Google Jobs, LinkedIn & Indeed with GPT-4o and Apify

https://n8nworkflows.xyz/workflows/extract-sales-leads-from-google-jobs--linkedin---indeed-with-gpt-4o-and-apify-11574


# Extract Sales Leads from Google Jobs, LinkedIn & Indeed with GPT-4o and Apify

---

### 1. Workflow Overview

This workflow is designed to automatically monitor job postings from three major job boards—Google Jobs, LinkedIn, and Indeed—to identify potential sales leads based on hiring signals. It leverages Apify actors for scraping, normalizes heterogeneous data into a uniform schema, filters opportunities by target keywords, and applies advanced AI (GPT-4o) for sales-oriented analysis and personalized email drafting. It also sends real-time Slack alerts for new leads and compiles weekly reports with trend analysis delivered via Slack and email.

**Target Use Cases:**  
- Sales teams seeking proactive lead generation from hiring trends  
- Automated market intelligence gathering for targeted outreach  
- Weekly strategic reporting on hiring signals and sales opportunities  

**Logical Blocks:**

- **1.1 Trigger and Configuration Setup**: Scheduling and parameter initialization  
- **1.2 Data Acquisition**: Scraping job postings from multiple sources using Apify  
- **1.3 Data Normalization and Aggregation**: Unifying data fields and deduplicating  
- **1.4 Filtering and Qualification**: Filtering by target keywords and recency  
- **1.5 AI-Driven Sales Lead Analysis**: Using GPT-4o to analyze jobs and draft emails  
- **1.6 Notifications and Data Persistence**: Slack alerts and saving qualified leads  
- **1.7 Weekly Reporting**: Aggregating weekly data, AI trend analysis, and report distribution  

---

### 2. Block-by-Block Analysis

---

#### 2.1 Trigger and Configuration Setup

**Overview:**  
This block defines the scheduled triggers for daily and weekly runs and sets key configuration parameters used throughout the workflow.

**Nodes Involved:**  
- Schedule Trigger - Daily 9AM  
- Schedule Trigger - Weekly Monday 8AM  
- Set Configuration  
- Set - Weekly Report Config  

**Node Details:**  

- **Schedule Trigger - Daily 9AM**  
  - Type: Schedule Trigger  
  - Configuration: Triggers workflow daily at 9 AM local time.  
  - Outputs: Initiates daily scraping flow.  
  - Potential failures: Timezone misconfigurations or missed triggers.  

- **Schedule Trigger - Weekly Monday 8AM**  
  - Type: Schedule Trigger  
  - Configuration: Triggers every Monday at 8 AM.  
  - Outputs: Initiates weekly reporting flow.  
  - Potential failures: Same as above.  

- **Set Configuration**  
  - Type: Set  
  - Role: Defines runtime variables such as:  
    - `daysToCheck`: Number of days back to filter jobs (default 7)  
    - `maxJobsPerSource`: Max jobs per scraping source (default 50)  
    - `targetIndustry`: Industry focus (e.g., "Technology")  
    - `runDate`: Current date string for timestamping  
  - Used in expressions across normalization and filtering nodes.  
  - Edge Cases: Missing or invalid parameter values could affect filtering or scraping limits.  

- **Set - Weekly Report Config**  
  - Type: Set  
  - Role: Defines the weekly report period boundaries using dynamic dates (7 days ago to today).  
  - Outputs: Used for filtering weekly jobs and leads for reporting.  
  - Edge Cases: Date formatting errors or timezone effects could cause inaccurate reporting periods.  

---

#### 2.2 Data Acquisition

**Overview:**  
This block scrapes job postings in parallel from Google Jobs, LinkedIn, and Indeed using Apify actors, based on target companies loaded from Google Sheets.

**Nodes Involved:**  
- Google Sheets - Get Target Companies  
- Apify - Scrape Google Jobs  
- Apify - Scrape LinkedIn Jobs  
- Apify - Scrape Indeed Jobs  

**Node Details:**  

- **Google Sheets - Get Target Companies**  
  - Type: Google Sheets  
  - Role: Reads the list of target companies and relevant metadata (keywords, solutions) from a sheet named "Target Companies".  
  - Config: Uses document ID from environment variable `GOOGLE_SHEETS_ID`.  
  - Output: Provides company data for scraping queries and normalization.  
  - Failure modes: Authentication issues, sheet not found, empty data.  

- **Apify - Scrape Google Jobs**  
  - Type: Apify Node  
  - Role: Runs an Apify actor to scrape Google Jobs data for each target company.  
  - Config:  
    - Actor ID from environment variable `APIFY_GOOGLE_JOBS_ACTOR_ID`.  
    - Query built as `"<Company Name> jobs"`.  
    - Max items limited by `maxJobsPerSource`.  
    - Country code set to 'us'.  
  - Output: Raw Google Jobs data set.  
  - Failure modes: Actor run failure, timeout, quota limits.  

- **Apify - Scrape LinkedIn Jobs**  
  - Type: Apify Node  
  - Role: Scrapes LinkedIn jobs for each target company.  
  - Config:  
    - Actor ID from `APIFY_LINKEDIN_JOBS_ACTOR_ID`.  
    - Search URL constructed with encoded company name keyword.  
    - Max items limited by `maxJobsPerSource`.  
  - Output: Raw LinkedIn jobs data.  
  - Failure modes: LinkedIn scraping restrictions, actor errors.  

- **Apify - Scrape Indeed Jobs**  
  - Type: Apify Node  
  - Role: Scrapes Indeed jobs for each company.  
  - Config:  
    - Actor ID from `APIFY_INDEED_ACTOR_ID`.  
    - Queries set as company name.  
    - Country set to 'US'.  
    - Max items limited by `maxJobsPerSource`.  
  - Output: Raw Indeed jobs data.  
  - Failure modes: Similar to LinkedIn.  

---

#### 2.3 Data Normalization and Aggregation

**Overview:**  
This block standardizes the heterogeneous job data from different sources into a consistent schema and merges them into a single dataset.

**Nodes Involved:**  
- Code - Normalize Google Jobs  
- Code - Normalize LinkedIn Jobs  
- Code - Normalize Indeed Jobs  
- Merge - Combine All Sources  
- Code - Deduplicate and Filter by Date  

**Node Details:**  

- **Code - Normalize Google Jobs**  
  - Type: Code (JavaScript)  
  - Role: Maps Google Jobs fields to unified schema with fields like source, title, company, location, description, postedAt, applyLink, salary, targetCompany, targetKeywords, mySolution, scrapedDate, jobId.  
  - Uses data from Google Sheets 'Target Companies' and runDate config.  
  - Normalizes missing fields with defaults (empty strings).  
  - Generates a unique `jobId` based on title and company.  
  - Edge cases: Missing or malformed fields, string truncation for jobId.  

- **Code - Normalize LinkedIn Jobs**  
  - Same role as above but for LinkedIn data.  
  - Fields mapped similarly with some source-specific field names.  

- **Code - Normalize Indeed Jobs**  
  - Same role for Indeed data.  
  - Handles variations like `positionName` for title.  

- **Merge - Combine All Sources**  
  - Type: Merge  
  - Role: Combines all normalized job arrays into a single array for further processing.  
  - Mode: Combine all inputs (union).  
  - Failure modes: Large datasets may cause performance issues.  

- **Code - Deduplicate and Filter by Date**  
  - Type: Code (JavaScript)  
  - Role: Removes duplicate jobs by `jobId` and filters jobs to those posted within `daysToCheck` days.  
  - Parses relative date strings like "3 days ago", "today", "just posted".  
  - Uses current date for filtering threshold.  
  - Edge cases: Unparseable date strings default to current date; could cause false inclusions.  

---

#### 2.4 Filtering and Qualification

**Overview:**  
Filters deduplicated jobs by whether their title or description contains any target keywords and flags jobs without matches.

**Nodes Involved:**  
- Google Sheets - Save Raw Jobs  
- Code - Filter by Target Keywords  
- IF - Check Matches Found  

**Node Details:**  

- **Google Sheets - Save Raw Jobs**  
  - Type: Google Sheets  
  - Role: Saves all deduplicated and filtered raw jobs into the "Raw Jobs" sheet for record-keeping.  
  - Uses the common Google Sheets document ID.  

- **Code - Filter by Target Keywords**  
  - Type: Code (JavaScript)  
  - Role: Checks if any target keyword (from normalized data field `targetKeywords`) appears in the combined job title and description (case-insensitive).  
  - Adds `matchedKeyword` field to jobs that pass filter.  
  - Returns filtered job list or a single object with `noMatches: true` if none found.  
  - Edge cases: Empty or malformed keyword lists; returns a no-match flag to control workflow path.  

- **IF - Check Matches Found**  
  - Type: If Condition  
  - Role: Routes workflow only if matches are found (`noMatches` is not true).  
  - Prevents AI analysis on empty input sets.  

---

#### 2.5 AI-Driven Sales Lead Analysis

**Overview:**  
For each qualified job, AI analyzes the posting to infer pain points, craft outreach angles, score urgency, and generate a personalized sales email draft.

**Nodes Involved:**  
- AI Agent - Analyze and Generate Email  
- OpenAI Chat Model (Daily)  
- Structured Output Parser (Daily)  
- Google Sheets - Save Qualified Leads  
- Slack - Send Lead Alert  

**Node Details:**  

- **AI Agent - Analyze and Generate Email**  
  - Type: Langchain Agent (AI Node)  
  - Role: Processes each job posting input, prompting GPT-4o to: infer pain points, hook angle, urgency score (1-10), and draft a cold sales email including subject line and personalized content.  
  - Input variables injected: company, title, description, location, source, matchedKeyword, and “mySolution.”  
  - System message guides AI as a B2B sales strategist.  
  - Output is JSON structured.  

- **OpenAI Chat Model (Daily)**  
  - Type: OpenAI Chat Model (GPT-4o)  
  - Role: Backend language model execution for the AI Agent node.  
  - No additional parameters beyond model selection.  

- **Structured Output Parser (Daily)**  
  - Type: Output Parser  
  - Role: Validates and parses AI output JSON against schema requiring pain_point, hook_angle, urgency_score (number), email_draft.  
  - Ensures structured data for downstream use.  

- **Google Sheets - Save Qualified Leads**  
  - Type: Google Sheets  
  - Role: Saves AI-qualified leads with enriched data into "Qualified Leads" sheet for tracking and reporting.  

- **Slack - Send Lead Alert**  
  - Type: Slack  
  - Role: Sends formatted Slack alert messages to a configured channel summarizing the hiring signal, pain points, approach angle, urgency score, and includes draft email and link to the job posting.  
  - Uses OAuth2 authentication.  
  - Failure modes: Slack API rate limits, auth token expiry.  

---

#### 2.6 Notifications and Data Persistence

**Overview:**  
This section encompasses saving data states and sending notifications for new leads detected during the daily flow.

**Nodes Involved:**  
Covered partially in prior blocks (Google Sheets and Slack nodes for raw jobs and qualified leads).

---

#### 2.7 Weekly Reporting

**Overview:**  
Aggregates weekly job and lead data, generates an AI-powered comprehensive hiring signal report, and distributes formatted output via Slack and Gmail.

**Nodes Involved:**  
- Google Sheets - Get Raw Jobs (Weekly)  
- Google Sheets - Get Qualified Leads (Weekly)  
- Merge - Weekly Data  
- Code - Aggregate Weekly Statistics  
- AI Agent - Weekly Trend Analysis  
- OpenAI Chat Model (Weekly)  
- Structured Output Parser (Weekly)  
- Google Sheets - Save Weekly Report  
- Slack - Send Weekly Report  
- Gmail - Send Weekly Report  

**Node Details:**  

- **Google Sheets - Get Raw Jobs (Weekly)** / **Google Sheets - Get Qualified Leads (Weekly)**  
  - Type: Google Sheets  
  - Role: Retrieve stored data from "Raw Jobs" and "Qualified Leads" for the week defined by the report period.  
  - Uses same document ID.  

- **Merge - Weekly Data**  
  - Type: Merge  
  - Role: Combines raw jobs and qualified leads datasets into one composite input for aggregation.  

- **Code - Aggregate Weekly Statistics**  
  - Type: Code (JavaScript)  
  - Role:  
    - Filters raw jobs and qualified leads by the report week period.  
    - Calculates totals, conversion rates, urgency distribution, counts by source, keywords, and top companies.  
    - Outputs structured statistics and samples for AI analysis.  
  - Edge cases: Empty data sets or date mismatches.  

- **AI Agent - Weekly Trend Analysis**  
  - Type: Langchain Agent (AI Node)  
  - Role: Generates a comprehensive weekly report in JSON with: executive summary, trend analysis, hot opportunities, recommended actions, forecast, and a styled HTML report.  
  - Inputs aggregated stats and samples from previous node.  
  - System prompt defines detailed sections and formatting requirements.  

- **OpenAI Chat Model (Weekly)**  
  - Type: GPT-4o model node  
  - Supports AI Agent execution.  

- **Structured Output Parser (Weekly)**  
  - Validates AI output JSON with required fields like executive_summary, trend_analysis, etc.  

- **Google Sheets - Save Weekly Report**  
  - Saves AI-generated weekly report into "Weekly Reports" sheet.  

- **Slack - Send Weekly Report**  
  - Posts a visually formatted weekly report summary to configured Slack channel with emojis, section headers, and key stats.  
  - Uses OAuth2 authentication.  

- **Gmail - Send Weekly Report**  
  - Sends the full HTML report via email to configured notification email address.  
  - Failure modes: Email deliverability, auth token issues.  

---

### 3. Summary Table

| Node Name                          | Node Type                          | Functional Role                              | Input Node(s)                           | Output Node(s)                          | Sticky Note                                                                                                                                                                                                                                                                                            |
|-----------------------------------|----------------------------------|----------------------------------------------|---------------------------------------|---------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Sticky Note - Main                | Sticky Note                      | Overview and instructions                     | -                                     | -                                     | 🎯 Hiring Signal Monitor & AI Sales Lead Generator: outlines overall workflow purpose and setup requirements.                                                                                                                                                                                        |
| Sticky Note - Daily Flow          | Sticky Note                      | Summary of daily scraping flow                 | -                                     | -                                     | 📅 Daily Scraping Flow: Daily trigger, load companies, scrape jobs, normalize, filter, AI analysis.                                                                                                                                                                                                  |
| Sticky Note - Normalize           | Sticky Note                      | Explains data normalization nodes             | -                                     | -                                     | 🔄 Data Normalization: Standardizes fields from all sources into a unified schema.                                                                                                                                                                                                                    |
| Sticky Note - AI Analysis         | Sticky Note                      | Explains AI analysis block                      | -                                     | -                                     | 🤖 AI Analysis: GPT-4o analyzes jobs to infer pain points, urgency, and draft emails.                                                                                                                                                                                                                 |
| Sticky Note - Weekly              | Sticky Note                      | Explains weekly report flow                     | -                                     | -                                     | 📊 Weekly Reports Flow: Aggregates stats, AI trend analysis, sends Slack and email reports.                                                                                                                                                                                                          |
| Sticky Note - Config              | Sticky Note                      | Highlights configuration parameters             | -                                     | -                                     | ⚙️ Configuration: daysToCheck, maxJobsPerSource, targetIndustry.                                                                                                                                                                                                                                     |
| Schedule Trigger - Daily 9AM      | Schedule Trigger                | Initiates daily scraping                        | -                                     | Set Configuration                     |                                                                                                                                                                                                                                                                                                      |
| Schedule Trigger - Weekly Monday 8AM | Schedule Trigger                | Initiates weekly reporting                      | -                                     | Set - Weekly Report Config            |                                                                                                                                                                                                                                                                                                      |
| Set Configuration                | Set                              | Defines runtime parameters                       | Schedule Trigger - Daily 9AM           | Google Sheets - Get Target Companies  |                                                                                                                                                                                                                                                                                                      |
| Set - Weekly Report Config        | Set                              | Defines weekly report period boundaries          | Schedule Trigger - Weekly Monday 8AM  | Google Sheets - Get Raw Jobs (Weekly), Google Sheets - Get Qualified Leads (Weekly) |                                                                                                                                                                                                                                                                                                      |
| Google Sheets - Get Target Companies | Google Sheets                   | Loads target companies and config for scraping  | Set Configuration                     | Apify - Scrape Google Jobs, LinkedIn Jobs, Indeed Jobs |                                                                                                                                                                                                                                                                                                      |
| Apify - Scrape Google Jobs        | Apify Node                      | Scrapes Google Jobs data                         | Google Sheets - Get Target Companies  | Code - Normalize Google Jobs          |                                                                                                                                                                                                                                                                                                      |
| Apify - Scrape LinkedIn Jobs      | Apify Node                      | Scrapes LinkedIn Jobs data                       | Google Sheets - Get Target Companies  | Code - Normalize LinkedIn Jobs        |                                                                                                                                                                                                                                                                                                      |
| Apify - Scrape Indeed Jobs        | Apify Node                      | Scrapes Indeed Jobs data                         | Google Sheets - Get Target Companies  | Code - Normalize Indeed Jobs          |                                                                                                                                                                                                                                                                                                      |
| Code - Normalize Google Jobs      | Code                            | Normalizes Google Jobs data fields               | Apify - Scrape Google Jobs            | Merge - Combine All Sources            |                                                                                                                                                                                                                                                                                                      |
| Code - Normalize LinkedIn Jobs    | Code                            | Normalizes LinkedIn Jobs data fields             | Apify - Scrape LinkedIn Jobs          | Merge - Combine All Sources            |                                                                                                                                                                                                                                                                                                      |
| Code - Normalize Indeed Jobs      | Code                            | Normalizes Indeed Jobs data fields               | Apify - Scrape Indeed Jobs            | Merge - Combine All Sources            |                                                                                                                                                                                                                                                                                                      |
| Merge - Combine All Sources       | Merge                           | Combines normalized jobs from all sources       | Code - Normalize Google/LinkedIn/Indeed Jobs | Code - Deduplicate and Filter by Date |                                                                                                                                                                                                                                                                                                      |
| Code - Deduplicate and Filter by Date | Code                        | Deduplicates jobs by jobId and filters by recency | Merge - Combine All Sources            | Google Sheets - Save Raw Jobs          |                                                                                                                                                                                                                                                                                                      |
| Google Sheets - Save Raw Jobs     | Google Sheets                   | Saves raw job data                               | Code - Deduplicate and Filter by Date | Code - Filter by Target Keywords       |                                                                                                                                                                                                                                                                                                      |
| Code - Filter by Target Keywords  | Code                            | Filters jobs by target keywords                   | Google Sheets - Save Raw Jobs          | IF - Check Matches Found               |                                                                                                                                                                                                                                                                                                      |
| IF - Check Matches Found          | If                              | Checks if any jobs matched keywords               | Code - Filter by Target Keywords       | AI Agent - Analyze and Generate Email  |                                                                                                                                                                                                                                                                                                      |
| AI Agent - Analyze and Generate Email | Langchain Agent             | AI analyzes job, infers pain points, drafts email | IF - Check Matches Found               | Google Sheets - Save Qualified Leads   |                                                                                                                                                                                                                                                                                                      |
| OpenAI Chat Model (Daily)         | OpenAI Model                   | Executes GPT-4o model for daily AI agent          | AI Agent - Analyze and Generate Email | Structured Output Parser (Daily)       |                                                                                                                                                                                                                                                                                                      |
| Structured Output Parser (Daily)  | Output Parser                  | Parses AI JSON output for daily analysis           | OpenAI Chat Model (Daily)              | AI Agent - Analyze and Generate Email  |                                                                                                                                                                                                                                                                                                      |
| Google Sheets - Save Qualified Leads | Google Sheets                | Stores qualified AI leads                          | AI Agent - Analyze and Generate Email | Slack - Send Lead Alert                |                                                                                                                                                                                                                                                                                                      |
| Slack - Send Lead Alert           | Slack                          | Sends Slack notification of qualified lead        | Google Sheets - Save Qualified Leads   | -                                     |                                                                                                                                                                                                                                                                                                      |
| Google Sheets - Get Raw Jobs (Weekly) | Google Sheets                | Retrieves raw jobs for the week                     | Set - Weekly Report Config             | Merge - Weekly Data                    |                                                                                                                                                                                                                                                                                                      |
| Google Sheets - Get Qualified Leads (Weekly) | Google Sheets            | Retrieves qualified leads for the week              | Set - Weekly Report Config             | Merge - Weekly Data                    |                                                                                                                                                                                                                                                                                                      |
| Merge - Weekly Data               | Merge                           | Combines raw jobs and qualified leads datasets      | Google Sheets - Get Raw Jobs (Weekly), Google Sheets - Get Qualified Leads (Weekly) | Code - Aggregate Weekly Statistics |                                                                                                                                                                                                                                                                                                      |
| Code - Aggregate Weekly Statistics | Code                          | Calculates weekly KPIs and stats                    | Merge - Weekly Data                    | AI Agent - Weekly Trend Analysis       |                                                                                                                                                                                                                                                                                                      |
| AI Agent - Weekly Trend Analysis  | Langchain Agent               | AI generates comprehensive weekly report            | Code - Aggregate Weekly Statistics     | Google Sheets - Save Weekly Report     |                                                                                                                                                                                                                                                                                                      |
| OpenAI Chat Model (Weekly)        | OpenAI Model                  | Executes GPT-4o for weekly AI agent                  | AI Agent - Weekly Trend Analysis       | Structured Output Parser (Weekly)      |                                                                                                                                                                                                                                                                                                      |
| Structured Output Parser (Weekly) | Output Parser                 | Parses AI weekly report JSON output                   | OpenAI Chat Model (Weekly)             | AI Agent - Weekly Trend Analysis       |                                                                                                                                                                                                                                                                                                      |
| Google Sheets - Save Weekly Report | Google Sheets                | Saves final weekly report                             | AI Agent - Weekly Trend Analysis       | Slack - Send Weekly Report, Gmail - Send Weekly Report |                                                                                                                                                                                                                                                                                                      |
| Slack - Send Weekly Report        | Slack                          | Posts formatted weekly report to Slack channel       | Google Sheets - Save Weekly Report     | -                                     |                                                                                                                                                                                                                                                                                                      |
| Gmail - Send Weekly Report        | Gmail                          | Emails weekly report as HTML to configured address   | Google Sheets - Save Weekly Report     | -                                     |                                                                                                                                                                                                                                                                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets Document and Tabs:**  
   - Create a Google Sheets document with tabs named:  
     - "Target Companies" (with columns: Company Name, Target Keywords, My Solution, etc.)  
     - "Raw Jobs"  
     - "Qualified Leads"  
     - "Weekly Reports"  

2. **Set Up Credentials:**  
   - Apify account with actors for Google Jobs, LinkedIn Jobs, and Indeed Jobs. Obtain actor IDs.  
   - OpenAI API key with GPT-4o access.  
   - Google Sheets API credentials for reading/writing sheets.  
   - Slack OAuth2 credentials with access to a Slack workspace and channel for notifications.  
   - Gmail OAuth2 credentials for sending emails.  

3. **Create Nodes in n8n:**

**Trigger and Config Nodes:**  
4. Add a **Schedule Trigger** node named "Schedule Trigger - Daily 9AM" configured to run daily at 9 AM.  
5. Add a **Schedule Trigger** node named "Schedule Trigger - Weekly Monday 8AM" configured for Mondays at 8 AM.  
6. Add a **Set** node named "Set Configuration" with variables:  
   - `daysToCheck` = 7  
   - `maxJobsPerSource` = 50  
   - `targetIndustry` = "Technology"  
   - `runDate` = current date (expression: `{{$now.format('yyyy-MM-dd')}}`)  
   Connect it to the daily schedule trigger.  
7. Add a **Set** node named "Set - Weekly Report Config" with:  
   - `reportWeekStart` = 7 days ago (`{{$now.minus({days:7}).format('yyyy-MM-dd')}}`)  
   - `reportWeekEnd` = today (`{{$now.format('yyyy-MM-dd')}}`)  
   Connect it to the weekly schedule trigger.  

**Data Acquisition Nodes:**  
8. Add a **Google Sheets** node "Google Sheets - Get Target Companies" to read the "Target Companies" tab. Use the document ID stored in a variable/env `GOOGLE_SHEETS_ID`. Connect from "Set Configuration".  
9. Add three **Apify** nodes for scraping jobs:  
   - "Apify - Scrape Google Jobs" using `APIFY_GOOGLE_JOBS_ACTOR_ID`, with JSON body:  
     `{"query": "={{ $json['Company Name'] + ' jobs' }}", "maxItems": {{$node["Set Configuration"].json["maxJobsPerSource"]}}, "countryCode": "us"}`  
   - "Apify - Scrape LinkedIn Jobs" using `APIFY_LINKEDIN_JOBS_ACTOR_ID`, with JSON body:  
     `{"searchUrl": "https://www.linkedin.com/jobs/search/?keywords=" + encodeURIComponent($json['Company Name']), "maxItems": {{$node["Set Configuration"].json["maxJobsPerSource"]}}}`  
   - "Apify - Scrape Indeed Jobs" using `APIFY_INDEED_ACTOR_ID`, with JSON body:  
     `{"queries": $json['Company Name'], "maxItems": {{$node["Set Configuration"].json["maxJobsPerSource"]}}, "country": "US"}`  
   Connect all three from "Google Sheets - Get Target Companies".  

**Normalization Nodes:**  
10. Add three **Code** nodes to normalize each source's data respectively ("Code - Normalize Google Jobs", "Code - Normalize LinkedIn Jobs", "Code - Normalize Indeed Jobs") using the JavaScript code to map source-specific fields to the unified schema, referencing the "Target Companies" data and `runDate`. Connect each from respective Apify nodes.  

11. Add a **Merge** node "Merge - Combine All Sources" with mode "Combine All" to merge normalized outputs. Connect all three normalization code nodes into this merge node.  

**Deduplication and Filtering Nodes:**  
12. Add a **Code** node "Code - Deduplicate and Filter by Date" containing logic to deduplicate jobs by jobId and filter by recency using `daysToCheck`. Connect from "Merge - Combine All Sources".  

13. Add a **Google Sheets** node "Google Sheets - Save Raw Jobs" to save the filtered jobs into the "Raw Jobs" tab. Connect from the deduplication node.  

14. Add a **Code** node "Code - Filter by Target Keywords" that filters jobs by presence of target keywords inside title or description and adds `matchedKeyword`. Connect from "Google Sheets - Save Raw Jobs".  

15. Add an **If** node "IF - Check Matches Found" that checks if filtered results contain matches (`noMatches` ≠ true). Connect from the filter node.  

**AI Lead Analysis Nodes:**  
16. Add a **Langchain AI Agent** node "AI Agent - Analyze and Generate Email" configured with the prompt to analyze job postings and produce pain_point, hook_angle, urgency_score, and email_draft in JSON. Connect from the true output of the IF node.  

17. Add an **OpenAI Chat Model** node "OpenAI Chat Model (Daily)" set to model `gpt-4o`. Connect as the AI language model to the AI Agent node.  

18. Add a **Structured Output Parser** node "Structured Output Parser (Daily)" with a JSON schema requiring pain_point, hook_angle, urgency_score (number), email_draft. Connect from OpenAI Chat Model (Daily) output parser.  

19. Connect the output parser back to "AI Agent - Analyze and Generate Email" as the output parser.  

20. Add a **Google Sheets** node "Google Sheets - Save Qualified Leads" saving AI-enriched leads to "Qualified Leads" tab. Connect from AI Agent node.  

21. Add a **Slack** node "Slack - Send Lead Alert" to send Slack messages on new leads. Configure OAuth2 credentials and select the appropriate Slack channel (variable `SLACK_CHANNEL`). Connect from "Google Sheets - Save Qualified Leads".  

**Weekly Reporting Nodes:**  
22. Add **Google Sheets** nodes "Google Sheets - Get Raw Jobs (Weekly)" and "Google Sheets - Get Qualified Leads (Weekly)" reading respective sheets. Connect both from "Set - Weekly Report Config".  

23. Add a **Merge** node "Merge - Weekly Data" in "Combine All" mode to combine weekly raw jobs and leads. Connect both Google Sheets nodes into this merge.  

24. Add a **Code** node "Code - Aggregate Weekly Statistics" that filters data by the report week, calculates totals, conversions, urgency distribution, and top keywords/companies. Connect from the merge node.  

25. Add a **Langchain AI Agent** node "AI Agent - Weekly Trend Analysis" with a system prompt to generate an executive summary, trend analysis, hot opportunities, recommended actions, forecast, and HTML formatted report based on aggregated stats (similar to daily agent but focused on trends). Connect from the aggregate statistics node.  

26. Add an **OpenAI Chat Model** node "OpenAI Chat Model (Weekly)" set to `gpt-4o`. Connect as AI language model for the weekly agent.  

27. Add a **Structured Output Parser** node "Structured Output Parser (Weekly)" with schema requiring executive_summary, trend_analysis, hot_opportunities, recommended_actions, forecast, report_html. Connect from weekly OpenAI node.  

28. Connect output parser back to weekly AI Agent node.  

29. Add a **Google Sheets** node "Google Sheets - Save Weekly Report" saving the weekly report data into "Weekly Reports" tab. Connect from weekly AI Agent node.  

30. Add a **Slack** node "Slack - Send Weekly Report" posting the formatted weekly report to Slack channel configured via OAuth2. Connect from Google Sheets - Save Weekly Report.  

31. Add a **Gmail** node "Gmail - Send Weekly Report" to email the HTML report to the configured notification address (`NOTIFICATION_EMAIL`). Connect from the same Google Sheets node.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                   | Context or Link                                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Workflow requires Apify actors for Google Jobs, LinkedIn Jobs, and Indeed Jobs scraping.                                                                                                        | Apify platform — https://apify.com                                                                                 |
| AI analysis uses GPT-4o with Langchain integration in n8n for structured JSON prompt/response handling.                                                                                         | OpenAI GPT-4o model, n8n Langchain nodes documentation                                                         |
| Slack notifications use OAuth2 authentication; ensure tokens have permissions to post in the target channel.                                                                                   | Slack OAuth2 setup guide — https://api.slack.com/authentication/oauth-v2                                        |
| Google Sheets API access requires enabling Sheets API and proper OAuth2 credentials with read/write scopes.                                                                                     | Google Sheets API docs — https://developers.google.com/sheets/api                                              |
| Gmail node sends weekly reports; configure OAuth2 with email-sending scope and verify deliverability.                                                                                           | Gmail API docs — https://developers.google.com/gmail/api                                                       |
| The workflow normalizes heterogeneous job data with custom JavaScript; any changes to source API output fields may require updates to normalization code blocks.                              | Source APIs: Google Jobs, LinkedIn, Indeed scraping actor outputs                                              |
| Weekly reports include visual HTML styled with inline CSS and emojis for clarity and engagement.                                                                                                | HTML email best practices — https://www.campaignmonitor.com/resources/guides/html-email-design-best-practices/ |
| Slack alert messages format includes markdown with sections, dividers, and code blocks for email drafts for readability.                                                                        | Slack Block Kit — https://api.slack.com/block-kit                                                              |
| To scale, consider limits on Apify actor usage, OpenAI tokens, Google Sheets data volume, and Slack rate limits.                                                                               | Monitor usage quotas and optimize scraping frequency or data retention policies.                                |

---

*Disclaimer: The provided text is exclusively the output from an automated n8n workflow. It strictly complies with content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.*