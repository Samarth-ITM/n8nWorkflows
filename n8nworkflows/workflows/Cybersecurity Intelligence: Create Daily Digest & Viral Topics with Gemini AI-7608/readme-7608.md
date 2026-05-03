Cybersecurity Intelligence: Create Daily Digest & Viral Topics with Gemini AI

https://n8nworkflows.xyz/workflows/cybersecurity-intelligence--create-daily-digest---viral-topics-with-gemini-ai-7608


# Cybersecurity Intelligence: Create Daily Digest & Viral Topics with Gemini AI

### 1. Workflow Overview

This n8n workflow automates the collection, analysis, and dissemination of cybersecurity intelligence from multiple RSS feeds. Its main purpose is to provide daily threat intelligence digests and identify recurring ("viral") cybersecurity topics over the past week, leveraging AI to summarize, deduplicate, and organize reports. The workflow is tailored for cybersecurity analysts who need to process large volumes of threat reports efficiently.

The workflow is logically divided into the following blocks:

- **1.1 Collection:** Automated ingestion of cybersecurity news articles from a curated set of RSS feeds, triggered daily at 7 AM.
- **1.2 Preprocessing:** Merging and filtering the collected articles to keep only recent (last 24 hours) and relevant content, preparing data for AI processing.
- **1.3 Daily Digest Generation:** Using a Google Gemini AI language model to summarize, deduplicate, and categorize the daily intelligence into a concise briefing in JSON format.
- **1.4 Daily Digest Dissemination:** Formatting the AI output into HTML and sending it via email; also storing it in a database for archival.
- **1.5 Historical Data Aggregation:** Retrieving daily digests from the last seven days from the database and flattening the data for AI analysis.
- **1.6 Viral Topics Identification:** Leveraging Google Gemini AI to identify and synthesize recurring cybersecurity topics over the past week, generating a viral topics report.
- **1.7 Viral Topics Dissemination:** Formatting the viral topics report into HTML, sending it via email, and storing it in the database.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Collection

**Overview:**  
This block collects cybersecurity news articles daily from 11 distinct RSS feeds. It triggers every day at 7 AM, fetching fresh content from well-known cybersecurity sources.

**Nodes Involved:**

- Daily trigger
- Bleepingcomputer (RSS)
- Securityweek (RSS)
- The Hacker News (RSS)
- Schneier on Security (RSS)
- Darkreading (RSS)
- Kaspersky Securelist (RSS)
- Google (Mandiant) (RSS)
- Microsoft Security (RSS)
- Trend Micro (RSS)
- Recorded Future (RSS)

**Node Details:**

- **Daily trigger**  
  - Type: Schedule Trigger  
  - Configuration: Fires daily at 7:00 AM local time.  
  - Role: Initiates the workflow.  
  - Edge Cases: Timezone mismatches might cause unexpected trigger times.  
- **RSS Feed Read Nodes (e.g., Bleepingcomputer)**  
  - Type: RSS Feed Read  
  - Configuration: URL set to the respective cybersecurity feed, SSL errors ignored.  
  - Role: Ingest news articles from each feed.  
  - Input: Trigger node  
  - Output: Array of articles with fields like title, link, pubDate, contentSnippet.  
  - Edge Cases: Feed unavailability, malformed RSS, or SSL issues (ignored here).  
  - On error: Continue regular output to avoid halting the workflow.

---

#### 1.2 Preprocessing

**Overview:**  
Combines all RSS feed outputs into a single dataset, filters articles published within the last 24 hours, and formats article data into a textual summary suitable for AI input.

**Nodes Involved:**

- Merge collected articles
- Precook data

**Node Details:**

- **Merge collected articles**  
  - Type: Merge  
  - Configuration: Waits for 10 inputs (all RSS feeds) to combine into one array.  
  - Role: Aggregates all articles into one collection for uniform processing.  
  - Input: All RSS Feed Read nodes  
  - Output: Merged array of articles  
  - Edge Cases: Missing inputs (feeds not responding) could reduce articles.

- **Precook data**  
  - Type: Code (JavaScript)  
  - Role:  
    - Filters merged articles to keep only those published in the last 24 hours.  
    - Formats articles into a string with Title, Link, Publish Date, and Text snippet separated by "---".  
  - Key Logic: Uses current date/time to calculate age of articles; excludes older ones.  
  - Input: Merged articles array  
  - Output: JSON with one property `articlesText` containing formatted string.  
  - Edge Cases: Articles without valid pubDate may be excluded or cause errors; time zone inconsistencies.

---

#### 1.3 Daily Digest Generation

**Overview:**  
Uses Google Gemini AI to summarize and organize the filtered cybersecurity articles into a structured daily digest JSON, grouped by predefined categories.

**Nodes Involved:**

- Write basic digest for today (AI Chain LLM)
- Structured Output Parser

**Node Details:**

- **Write basic digest for today**  
  - Type: Chain LLM (Google Gemini chat model)  
  - Configuration:  
    - Prompt instructs the AI to summarize threat intelligence articles into a concise briefing.  
    - The AI is asked to group info by topics: nation state actor activity, financially motivated actors, compromises and breaches, vulnerabilities, miscellaneous.  
    - Requires deduplication, source linking, and strict formatting in JSON.  
    - Temperature set to 0 for deterministic output.  
  - Input: `articlesText` from Precook data  
  - Output: AI-generated JSON with categorized threat summaries.  
  - Edge Cases: AI hallucination risk mitigated by prompt instructions; requires correct input format.

- **Structured Output Parser**  
  - Type: LangChain Structured Output Parser  
  - Configuration: Manual JSON schema defining expected output structure with arrays of news items and references.  
  - Role: Validates and parses AI output into structured JSON for downstream processing.  
  - Input: AI raw output  
  - Output: Parsed JSON object  
  - Edge Cases: Parsing failure if AI output doesn't conform to schema.

---

#### 1.4 Daily Digest Dissemination

**Overview:**  
Transforms AI-generated JSON digest into HTML for email, sends the daily digest email, and stores the digest in Baserow database for archival.

**Nodes Involved:**

- Process output into HTML
- Send daily digest email
- Baserow Push Data

**Node Details:**

- **Process output into HTML**  
  - Type: Code (JavaScript)  
  - Role: Converts JSON digest into well-structured HTML with headings per category and linked references.  
  - Input: Parsed JSON digest  
  - Output: JSON with `htmlOutput` property containing HTML string  
  - Edge Cases: Missing or empty categories handled gracefully; date formatting applied.

- **Send daily digest email**  
  - Type: Email Send  
  - Configuration:  
    - Uses SMTP credentials.  
    - Sends email to fixed recipient ("velden.tom@linkmet.me") with subject "CTI Digest - YYYY-MM-DD".  
    - Email body is the HTML from previous node.  
  - Input: HTML output from code node  
  - Edge Cases: SMTP auth failure, network issues.

- **Baserow Push Data**  
  - Type: Baserow (NoSQL database)  
  - Configuration:  
    - Creates new record in table 562928 with field 4515701 storing stringified JSON digest.  
    - Uses stored Baserow API credentials.  
  - Input: Parsed JSON digest  
  - Edge Cases: API rate limits, network errors.

---

#### 1.5 Historical Data Aggregation

**Overview:**  
Retrieves daily digests created within the last 7 days from Baserow, filters and flattens data for AI consumption.

**Nodes Involved:**

- Baserow Push Data (from daily digest)
- 7 days ago date
- Get data of previous days
- Filter summaries and created date
- Flatten

**Node Details:**

- **7 days ago date**  
  - Type: Code  
  - Role: Produces ISO date string for 7 days ago to use as filter parameter.  
  - Output: JSON with `previous_days` date string.

- **Get data of previous days**  
  - Type: Baserow (Read)  
  - Configuration:  
    - Queries table 562928 for records with "Created on" date after the `previous_days` date.  
    - Uses Baserow API credentials.  
  - Output: Array of daily digest records for the last week.

- **Filter summaries and created date**  
  - Type: Set  
  - Role: Extracts only the Description (digest content) and creation date from each record.  
  - Output: Simplified JSON with `daily_report` and `created_on`.

- **Flatten**  
  - Type: Code  
  - Role: Converts array of daily reports into a single flattened array for AI input.

---

#### 1.6 Viral Topics Identification

**Overview:**  
Using Google Gemini AI, this block analyzes the historical weekly data to identify viral cybersecurity topics mentioned on multiple days, merges them into unified entries with aggregated references, and generates a concise report.

**Nodes Involved:**

- Identify viral topics and write digest (AI Chain LLM)
- Structured Output Parser1

**Node Details:**

- **Identify viral topics and write digest**  
  - Type: Chain LLM (Google Gemini chat model)  
  - Configuration:  
    - Prompt instructs AI to identify same-topic items recurring across multiple days.  
    - Merges, synthesizes title, summary, and aggregates references.  
    - Sorts topics by newest source, descending.  
    - Produces JSON array of viral topics.  
    - On error: continue without stopping workflow.  
  - Input: Flattened historical daily digest array  
  - Output: AI-generated viral topics JSON.

- **Structured Output Parser1**  
  - Type: LangChain Structured Output Parser  
  - Configuration: Manual JSON schema expecting array of objects with title, content, references.  
  - Input: AI raw output  
  - Output: Parsed viral topics JSON array.

---

#### 1.7 Viral Topics Dissemination

**Overview:**  
Formats the viral topics JSON into HTML, sends the viral topics report email, and stores the report in Baserow for future reference.

**Nodes Involved:**

- Process output into HTML1
- Send viral topics email
- Baserow Push Data1

**Node Details:**

- **Process output into HTML1**  
  - Type: Code (JavaScript)  
  - Role: Converts viral topics JSON into HTML sections including title, summary, and linked references (only reports with ≥3 references).  
  - Input: Parsed viral topics JSON  
  - Output: JSON with `html` string property.

- **Send viral topics email**  
  - Type: Email Send  
  - Configuration:  
    - SMTP credentials used.  
    - Sends to same recipient as daily digest.  
    - Subject: "CTI Digest (Viral Topics) - YYYY-MM-DD".  
    - Email body: formatted viral topics HTML.  
  - Edge Cases: SMTP failures.

- **Baserow Push Data1**  
  - Type: Baserow (Create)  
  - Configuration:  
    - Stores viral topics JSON string in different table 628338 field 5118199.  
  - Edge Cases: API failures, rate limits.

---

### 3. Summary Table

| Node Name                     | Node Type                               | Functional Role                            | Input Node(s)                                                | Output Node(s)                                  | Sticky Note                                                                                                          |
|-------------------------------|---------------------------------------|--------------------------------------------|--------------------------------------------------------------|------------------------------------------------|----------------------------------------------------------------------------------------------------------------------|
| Daily trigger                 | Schedule Trigger                      | Workflow start trigger                      | -                                                            | Bleepingcomputer, Securityweek, The Hacker News, Schneier on Security, Darkreading, Kaspersky Securelist, Google (Mandiant), Microsoft Security, Trend Micro, Recorded Future | ## Collection Ingest intelligence from RSS feeds, every morning on 7am.                                              |
| Bleepingcomputer             | RSS Feed Read                        | Collect cybersecurity news                  | Daily trigger                                                | Merge collected articles                         | ## Collection Ingest intelligence from RSS feeds, every morning on 7am.                                              |
| Securityweek                 | RSS Feed Read                        | Collect cybersecurity news                  | Daily trigger                                                | Merge collected articles                         | ## Collection Ingest intelligence from RSS feeds, every morning on 7am.                                              |
| The Hacker News              | RSS Feed Read                        | Collect cybersecurity news                  | Daily trigger                                                | Merge collected articles                         | ## Collection Ingest intelligence from RSS feeds, every morning on 7am.                                              |
| Schneier on Security         | RSS Feed Read                        | Collect cybersecurity news                  | Daily trigger                                                | Merge collected articles                         | ## Collection Ingest intelligence from RSS feeds, every morning on 7am.                                              |
| Darkreading                  | RSS Feed Read                        | Collect cybersecurity news                  | Daily trigger                                                | Merge collected articles                         | ## Collection Ingest intelligence from RSS feeds, every morning on 7am.                                              |
| Kaspersky Securelist         | RSS Feed Read                        | Collect cybersecurity news                  | Daily trigger                                                | Merge collected articles                         | ## Collection Ingest intelligence from RSS feeds, every morning on 7am.                                              |
| Google (Mandiant)            | RSS Feed Read                        | Collect cybersecurity news                  | Daily trigger                                                | Merge collected articles                         | ## Collection Ingest intelligence from RSS feeds, every morning on 7am.                                              |
| Microsoft Security           | RSS Feed Read                        | Collect cybersecurity news                  | Daily trigger                                                | Merge collected articles                         | ## Collection Ingest intelligence from RSS feeds, every morning on 7am.                                              |
| Trend Micro                  | RSS Feed Read                        | Collect cybersecurity news                  | Daily trigger                                                | Merge collected articles                         | ## Collection Ingest intelligence from RSS feeds, every morning on 7am.                                              |
| Recorded Future              | RSS Feed Read                        | Collect cybersecurity news                  | Daily trigger                                                | Merge collected articles                         | ## Collection Ingest intelligence from RSS feeds, every morning on 7am.                                              |
| Merge collected articles     | Merge                                | Aggregate all collected articles            | All RSS Feed Read nodes                                     | Precook data                                   | ## Processing Process the ingested data for further analysis by merging it into a single data block. Filter articles older than 24h and irrelevant fields. |
| Precook data                | Code                                | Filter recent articles and format text      | Merge collected articles                                    | Write basic digest for today                    | ## Processing Process the ingested data for further analysis by merging it into a single data block. Filter articles older than 24h and irrelevant fields. |
| Write basic digest for today | Chain LLM (Google Gemini)             | Generate structured daily digest JSON       | Precook data                                               | Structured Output Parser, Process output into HTML, Baserow Push Data | ## Analysis Leverage AI to deduplicate, organize, and summarize the information for a daily digest email.             |
| Structured Output Parser     | LangChain Parser                    | Parse AI JSON output                         | Write basic digest for today                                 | Write basic digest for today (AI output)       | ## Analysis Leverage AI to deduplicate, organize, and summarize the information for a daily digest email.             |
| Process output into HTML      | Code                                | Convert JSON digest to HTML for email       | Structured Output Parser                                    | Send daily digest email                         | ## Dissemination Disseminate the report over email.                                                                    |
| Send daily digest email       | Email Send                         | Email the daily digest                        | Process output into HTML                                    | -                                              | ## Dissemination Disseminate the report over email.                                                                    |
| Baserow Push Data            | Baserow                             | Store daily digest JSON in database          | Structured Output Parser                                    | 7 days ago date                                | ## Processing Save the output in a database and collect the data of the last 7 days for viral topic identification.     |
| 7 days ago date              | Code                                | Generate ISO date string for 7 days ago      | Baserow Push Data                                          | Get data of previous days                       | ## Processing Save the output in a database and collect the data of the last 7 days for viral topic identification.     |
| Get data of previous days     | Baserow                             | Retrieve last 7 days of reports              | 7 days ago date                                            | Filter summaries and created date              | ## Processing Save the output in a database and collect the data of the last 7 days for viral topic identification.     |
| Filter summaries and created date | Set                              | Extract summaries and dates for AI input     | Get data of previous days                                   | Flatten                                        | ## Processing Save the output in a database and collect the data of the last 7 days for viral topic identification.     |
| Flatten                     | Code                                | Flatten array of daily reports for AI input  | Filter summaries and created date                           | Identify viral topics and write digest          | ## Processing Save the output in a database and collect the data of the last 7 days for viral topic identification.     |
| Identify viral topics and write digest | Chain LLM (Google Gemini)             | Identify and merge viral topics over 7 days  | Flatten                                                    | Structured Output Parser1, Process output into HTML1, Baserow Push Data1 | ## Analysis Leverage AI to identity viral topics in reporting from the previous week and organize it into a viral topics threat brief. |
| Structured Output Parser1    | LangChain Parser                    | Parse AI viral topics output                   | Identify viral topics and write digest                      | Identify viral topics and write digest          | ## Analysis Leverage AI to identity viral topics in reporting from the previous week and organize it into a viral topics threat brief. |
| Process output into HTML1     | Code                                | Convert viral topics JSON to HTML              | Structured Output Parser1                                   | Send viral topics email                         | ## Dissemination Disseminate the report over email and save it in a database for future usage.                          |
| Send viral topics email       | Email Send                         | Email the viral topics report                   | Process output into HTML1                                   | -                                              | ## Dissemination Disseminate the report over email and save it in a database for future usage.                          |
| Baserow Push Data1           | Baserow                             | Store viral topics report in database           | Identify viral topics and write digest                      | 7 days ago date                                | ## Dissemination Disseminate the report over email and save it in a database for future usage.                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node named "Daily trigger"**  
   - Set it to trigger daily at 7:00 AM.

2. **Create 11 RSS Feed Read nodes for cybersecurity sources:**  
   - Names: Bleepingcomputer, Securityweek, The Hacker News, Schneier on Security, Darkreading, Kaspersky Securelist, Google (Mandiant), Microsoft Security, Trend Micro, Recorded Future, Proofpoint Threat Insight (optional in this workflow)  
   - Configure each with the respective RSS feed URL.  
   - Set "Ignore SSL errors" to true.  
   - Connect the output of "Daily trigger" to all these nodes.

3. **Create a Merge node named "Merge collected articles"**  
   - Set "Mode" to "Merge By Index".  
   - Set "Number of Inputs" to 10 (for the 10 RSS feeds connected).  
   - Connect outputs of all RSS nodes into this Merge node.

4. **Create a Code node named "Precook data"**  
   - Paste JavaScript code to combine all articles, filter those older than 24 hours, and format them as a single string with Title, Link, Publish Date, and snippet.  
   - Connect output of "Merge collected articles" to this node.

5. **Create a Chain LLM node named "Write basic digest for today"**  
   - Use Google Gemini chat model credentials.  
   - Set prompt with instructions to summarize, deduplicate, and organize the articles text by categories, outputting well-formed JSON.  
   - Set temperature to 0 for deterministic outputs.  
   - Connect output of "Precook data" to this node.

6. **Create a Structured Output Parser node configured with the manual JSON schema** for the expected digest output format (five categories with news items and references).  
   - Connect output of "Write basic digest for today" to this parser.

7. **Create a Code node named "Process output into HTML"**  
   - Implement JavaScript to convert the parsed JSON digest into HTML with headings per category and linked references.  
   - Connect output of the Structured Output Parser to this node.

8. **Create an Email Send node named "Send daily digest email"**  
   - Configure SMTP credentials.  
   - Set recipient email (e.g., velden.tom@linkmet.me).  
   - Subject: "CTI Digest - {{ $now.format('yyyy-MM-dd') }}".  
   - HTML body: use output from "Process output into HTML".  
   - Connect output of "Process output into HTML" to this node.

9. **Create a Baserow node named "Baserow Push Data"**  
   - Configure with Baserow API credentials.  
   - Set operation to "Create" on the database and table where daily digests are stored.  
   - Store the JSON stringified digest in the appropriate field.  
   - Connect output of the Structured Output Parser to this node.

10. **Create a Code node named "7 days ago date"**  
    - Return ISO string for the date 7 days ago (to filter historical data).  
    - Connect output of "Baserow Push Data" to this node.

11. **Create a Baserow node named "Get data of previous days"**  
    - Use Baserow credentials.  
    - Configure to read records from daily digest table where "Created on" is after the date from "7 days ago date".  
    - Connect output of "7 days ago date" to this node.

12. **Create a Set node named "Filter summaries and created date"**  
    - Extract only the digest content (Description) and creation date fields.  
    - Connect output of "Get data of previous days" to this node.

13. **Create a Code node named "Flatten"**  
    - Flatten the array of daily digests into a single array.  
    - Connect output of "Filter summaries and created date" to this node.

14. **Create a Chain LLM node named "Identify viral topics and write digest"**  
    - Use Google Gemini Pro chat model credentials (higher capacity).  
    - Configure prompt to identify recurring topics appearing across multiple days, merge them, synthesize titles and summaries, and aggregate references sorted by newest source.  
    - Set to continue on error to avoid workflow interruption.  
    - Connect output of "Flatten" to this node.

15. **Create a Structured Output Parser node named "Structured Output Parser1"**  
    - Configure with manual JSON schema expecting an array of objects with title, content, and references.  
    - Connect output of "Identify viral topics and write digest" to this node.

16. **Create a Code node named "Process output into HTML1"**  
    - Convert viral topics JSON into HTML with sections, including only reports with 3 or more references.  
    - Connect output of "Structured Output Parser1" to this node.

17. **Create an Email Send node named "Send viral topics email"**  
    - Use same SMTP credentials and recipient as daily digest.  
    - Subject: "CTI Digest (Viral Topics) - {{ $now.format('yyyy-MM-dd') }}".  
    - HTML body from "Process output into HTML1".  
    - Connect output of "Process output into HTML1" to this node.

18. **Create a Baserow node named "Baserow Push Data1"**  
    - Configure to create records in a separate table for viral topics reports.  
    - Store JSON stringified viral topics output.  
    - Connect output of "Identify viral topics and write digest" to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Context or Link                                                                                               |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| This workflow generates two types of daily emails: a concise, categorized cybersecurity threat intelligence digest and a viral topics report highlighting recurring high-signal events over the past week. It supports analysts by reducing noise and tracking important topics efficiently. The workflow is highly customizable in terms of RSS feeds, AI prompts, and reporting parameters. Requires Google Gemini (PaLM API) and Baserow credentials, as well as an SMTP account for email delivery.                                                                                                                                                                                                                                                                                                                                                              | Workflow description and usage notes.                                                                         |
| The workflow triggers daily at 7am and uses a set of trusted cybersecurity RSS feeds such as Bleepingcomputer, The Hacker News, and Mandiant to ingest fresh intelligence.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Sticky Note #1                                                                                              |
| AI prompts are carefully designed to enforce strict JSON output formats and to avoid hallucinations or irrelevant content, prioritizing brevity, accuracy, and source linking.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Sticky Note #2 and #3                                                                                         |
| Baserow is used as a scalable storage backend for both daily digests and viral topic reports, enabling historical data analysis and archival.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Sticky Note #5                                                                                              |
| The viral topics identification process aggregates reports from the last 7 days, extracting only those topics mentioned multiple times and merging references, to highlight ongoing cybersecurity trends.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Sticky Note #4                                                                                              |
| Email sending requires an SMTP account, which can be Gmail or any other provider, properly configured with authentication.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Sticky Note #6                                                                                              |
| For more information on n8n workflows and Google Gemini API usage, consult official n8n documentation and Google PaLM API references.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | n8n Docs: https://docs.n8n.io/ <br> Google PaLM API: https://developers.generativeai.google/                   |
| This workflow was designed by cybersecurity intelligence analysts to automate and standardize daily reporting, improving situational awareness and analyst productivity.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Workflow project credits and context.                                                                         |

---

**Disclaimer:** The provided text is derived exclusively from an n8n automated workflow. It conforms strictly to content policies and contains no illegal, offensive, or protected material. All data processed is legal and publicly available.