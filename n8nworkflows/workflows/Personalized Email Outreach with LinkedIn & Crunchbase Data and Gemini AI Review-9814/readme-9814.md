Personalized Email Outreach with LinkedIn & Crunchbase Data and Gemini AI Review

https://n8nworkflows.xyz/workflows/personalized-email-outreach-with-linkedin---crunchbase-data-and-gemini-ai-review-9814


# Personalized Email Outreach with LinkedIn & Crunchbase Data and Gemini AI Review

### 1. Workflow Overview

This workflow automates a personalized email outreach process by enriching contact and company data from LinkedIn and Crunchbase sources, then generating creative outreach emails using Google Gemini AI with a verification step to ensure quality before updating the data table. It targets sales, marketing, and business development teams seeking to send highly personalized, research-driven outreach messages.

The workflow is logically divided into these blocks:

- **1.1 Data Enrichment and Preparation:**  
  Fetches contact rows from a data table, filters for entries without email subjects, and enriches personal LinkedIn, company LinkedIn, and Crunchbase data using an external RapidAPI scraping service. Data updates are stored back in the table.

- **1.2 Creative Email Generation:**  
  For each enriched contact batch, uses a LangChain agent (Creative Outreach Agent) powered by Google Gemini AI to craft personalized outreach emails based on the enriched data and a predefined “About Me” profile.

- **1.3 Email Quality Judgment and Approval Routing:**  
  Evaluates each generated email with a LangChain Judge Agent to decide if it’s APPROVED or needs revision. Approved emails update the data table with subject and content. Non-approved emails loop back for reprocessing.

- **1.4 Workflow Execution and Triggering:**  
  Supports manual triggering and execution triggered by other workflows, initializing the enrichment and outreach process.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Enrichment and Preparation

**Overview:**  
Retrieves contact entries lacking email subjects, then enriches personal LinkedIn profiles, company LinkedIn pages, and Crunchbase company data via asynchronous HTTP requests to a RapidAPI cold outreach enrichment scraper. After each enrichment, results are saved back to the data table.

**Nodes Involved:**  
- RapidAPI-Key  
- Get row(s)1  
- Linkedin_URL  
- Wait  
- results  
- Update row(s)1  
- Linkedin_URL_COMPANY  
- Wait1  
- results1  
- Update row(s)2  
- Crunchbase_URL  
- Wait2  
- results2  
- Update row(s)3  
- Call 'My workflow' (sub-workflow invocation)  
- Sticky Note, Sticky Note1, Sticky Note2, Sticky Note3, Sticky Note4, Sticky Note5 (documentational nodes)

**Node Details:**

- **RapidAPI-Key**  
  *Type:* Set  
  *Role:* Provides the RapidAPI key credential to authenticate scraper API requests.  
  *Config:* Stores the RapidAPI key as a string variable for downstream HTTP requests.  
  *Input:* Manual trigger or upstream.  
  *Output:* Flows to Get row(s)1.  
  *Edge cases:* Invalid or expired API key causes HTTP 401 errors.

- **Get row(s)1**  
  *Type:* Data Table Get Rows  
  *Role:* Retrieves rows from the outreach data table where the email_subject field is empty (i.e., contacts not yet processed for email).  
  *Config:* Filter condition email_subject is empty; targets table "email_linkedin_list".  
  *Input:* From RapidAPI-Key.  
  *Output:* Feeds into LinkedIn and Crunchbase enrichment nodes and a sub-workflow call.  
  *Edge cases:* No rows found → downstream nodes receive empty inputs.

- **Linkedin_URL**  
  *Type:* HTTP Request  
  *Role:* Sends URL to cold outreach scraper API to initiate scraping of the personal LinkedIn profile URL.  
  *Config:* Passes LinkedIn URL from row data; uses RapidAPI key in headers; batching enabled with batch size 1.  
  *Input:* From Get row(s)1.  
  *Output:* Flows to Wait node.  
  *Edge cases:* API request failures, rate limiting, or invalid URLs.

- **Wait**  
  *Type:* Wait  
  *Role:* Delays flow to allow asynchronous scraping task completion.  
  *Config:* Waits for a specified time in minutes before querying results.  
  *Input:* From Linkedin_URL.  
  *Output:* Flows to results node.  
  *Edge cases:* Insufficient wait time may cause incomplete results.

- **results**  
  *Type:* HTTP Request  
  *Role:* Retrieves scraping results from the cold outreach API using task_id.  
  *Config:* Uses task_id from LinkedIn_URL response; includes RapidAPI credentials; batch size 1.  
  *Input:* From Wait.  
  *Output:* Flows to Update row(s)1.  
  *Edge cases:* Task ID not ready, API errors.

- **Update row(s)1**  
  *Type:* Data Table Update  
  *Role:* Updates the row’s linkedin_profile_scrape field with JSON stringified scraping results.  
  *Config:* Matches rows by email; updates linkedin_profile_scrape field.  
  *Input:* From results.  
  *Output:* No direct continuation; parallel flow continues for other enrichments.  
  *Edge cases:* Table row not found for update.

- **Linkedin_URL_COMPANY**  
  *Type:* HTTP Request  
  *Role:* Similar to Linkedin_URL but scrapes the company LinkedIn page URL.  
  *Config:* Uses company_linkedin URL from row; RapidAPI credentials; batching.  
  *Input:* From Get row(s)1.  
  *Output:* Flows to Wait1.  
  *Edge cases:* Same as Linkedin_URL node.

- **Wait1**  
  *Type:* Wait  
  *Role:* Delays to allow company LinkedIn scraping task completion.  
  *Config:* Wait duration in minutes.  
  *Input:* From Linkedin_URL_COMPANY.  
  *Output:* Flows to results1.  
  *Edge cases:* Same as Wait node.

- **results1**  
  *Type:* HTTP Request  
  *Role:* Fetches company LinkedIn scraping results by task_id.  
  *Config:* Uses task_id from Linkedin_URL_COMPANY node.  
  *Input:* From Wait1.  
  *Output:* Flows to Update row(s)2.  
  *Edge cases:* Same as results node.

- **Update row(s)2**  
  *Type:* Data Table Update  
  *Role:* Updates the linkedin_company_scrape field with the company LinkedIn data JSON.  
  *Config:* Matches rows by email.  
  *Input:* From results1.  
  *Output:* No direct continuation.  
  *Edge cases:* Same as Update row(s)1.

- **Crunchbase_URL**  
  *Type:* HTTP Request  
  *Role:* Initiates scraping of the company Crunchbase URL.  
  *Config:* Uses Crunchbase_URL from row data; RapidAPI credentials; batch size 1.  
  *Input:* From Get row(s)1.  
  *Output:* Flows to Wait2.  
  *Edge cases:* Same as other scraper HTTP nodes.

- **Wait2**  
  *Type:* Wait  
  *Role:* Delays for Crunchbase scraping task completion.  
  *Config:* Wait duration in minutes.  
  *Input:* From Crunchbase_URL.  
  *Output:* Flows to results2.  
  *Edge cases:* Same as Wait nodes.

- **results2**  
  *Type:* HTTP Request  
  *Role:* Retrieves Crunchbase scraping results by task_id.  
  *Config:* Uses task_id from Crunchbase_URL.  
  *Input:* From Wait2.  
  *Output:* Flows to Update row(s)3.  
  *Edge cases:* Same as results nodes.

- **Update row(s)3**  
  *Type:* Data Table Update  
  *Role:* Updates the crunchbase_company_scrape field with the Crunchbase data JSON string.  
  *Config:* Matches rows by email.  
  *Input:* From results2.  
  *Output:* No continuation.  
  *Edge cases:* Same as other Update nodes.

- **Call 'My workflow'**  
  *Type:* Execute Workflow Subprocess  
  *Role:* Invokes an external sub-workflow ("My workflow") for additional processing (details not included).  
  *Config:* Does not wait for sub-workflow to finish; no inputs passed.  
  *Input:* From Get row(s)1.  
  *Output:* No downstream connection.  
  *Edge cases:* Sub-workflow errors or unavailability.

- **Sticky Notes**  
  Provide contextual documentation on prerequisites, data enrichment steps, and API subscriptions.

---

#### 2.2 Creative Email Generation

**Overview:**  
Processes each contact in batches, leveraging the enriched data to generate a personalized outreach email via a LangChain agent powered by the Google Gemini language model.

**Nodes Involved:**  
- Get row(s)  
- Main Loop  
- Agent One  
- Structured Output Parser  
- Email Context

**Node Details:**

- **Get row(s)**  
  *Type:* Data Table Get Rows  
  *Role:* Fetches rows where email_subject is empty to process for email generation.  
  *Config:* Filters for empty email_subject in "email_linkedin_list".  
  *Input:* From "When Executed by Another Workflow" or Approval Route fallback.  
  *Output:* Feeds into Main Loop.  
  *Edge cases:* Empty result set stalls downstream processing.

- **Main Loop**  
  *Type:* Split In Batches  
  *Role:* Processes input rows individually or in small batches for sequential AI email generation.  
  *Config:* Default batching (1 or more).  
  *Input:* From Get row(s).  
  *Output:* On error or after loop, either stops or passes to Agent One.  
  *Edge cases:* Batch size too large can cause timeouts.

- **Agent One**  
  *Type:* LangChain Agent (AI)  
  *Role:* Generates a personalized email using the enriched data and a detailed system prompt defining creative outreach rules and constraints.  
  *Config:*  
    - Input text composes a structured data section with contact and company info.  
    - “About Me” section hardcoded as sender profile.  
    - Max iterations: 10 for refinement.  
    - System message defines tone, rules, and JSON output format (subject and body).  
    - Uses Google Gemini Chat model as backend.  
  *Input:* From Main Loop (one batch).  
  *Output:* Flows to Structured Output Parser.  
  *Edge cases:* AI service errors, output parsing failures, or incomplete JSON responses.

- **Structured Output Parser**  
  *Type:* LangChain Structured Output Parser  
  *Role:* Parses AI response to extract JSON fields "email_subject" and "email_content".  
  *Config:* JSON schema example defines expected keys.  
  *Input:* From Agent One.  
  *Output:* Flows to Email Context.  
  *Edge cases:* Parsing errors due to malformed AI output.

- **Email Context**  
  *Type:* Set  
  *Role:* Maps parsed output fields to workflow variables "email_subject" and "email_content", also passes recipient email.  
  *Config:* Assigns values from parser output and current item JSON email.  
  *Input:* From Structured Output Parser.  
  *Output:* Flows to Judge Agent.  
  *Edge cases:* Missing or null fields cause downstream failures.

---

#### 2.3 Email Quality Judgment and Approval Routing

**Overview:**  
Evaluates the AI-generated email's quality, personalization, tone, and relevance using a LangChain agent acting as a judge, then routes the result for approval or reprocessing.

**Nodes Involved:**  
- Judge Agent  
- Structured Output Parser1  
- Approval Route  
- Update row(s)  
- Get row(s)

**Node Details:**

- **Judge Agent**  
  *Type:* LangChain Agent (AI)  
  *Role:* Evaluates email subject and content to decide if the email is APPROVED or requires REVISION, providing optional feedback.  
  *Config:*  
    - Input text includes email subject and content fields.  
    - System message defines detailed evaluation criteria and expected outputs ("APPROVED" or "REVISE: feedback").  
    - Max iterations 20 for thorough evaluation.  
    - Uses Google Gemini Chat model backend.  
    - On error, continues workflow without failure.  
  *Input:* From Email Context.  
  *Output:* Flows to Structured Output Parser1.  
  *Edge cases:* AI evaluation errors, ambiguous results.

- **Structured Output Parser1**  
  *Type:* LangChain Structured Output Parser  
  *Role:* Parses Judge Agent output for approval status ("APPROVED" or "REVISE") and optional feedback.  
  *Config:* JSON schema with "approval" and "feedback" keys.  
  *Input:* From Judge Agent.  
  *Output:* Flows to Approval Route.  
  *Edge cases:* Parsing errors if AI output format deviates.

- **Approval Route**  
  *Type:* If Condition  
  *Role:* Checks if approval field contains "APPROVED".  
  *Config:* String contains "APPROVED" condition.  
  *Input:* From Structured Output Parser1.  
  *Output:*  
    - If true: to Update row(s) node to save approved email, then to Get row(s) for next batch.  
    - If false: loops back to Main Loop for reprocessing (email revision).  
  *Edge cases:* Logic errors if output format varies.

- **Update row(s)**  
  *Type:* Data Table Update  
  *Role:* Saves approved email subject and body to the data table row matching by email.  
  *Config:* Updates email_subject and email_body fields.  
  *Input:* From Approval Route (true path).  
  *Output:* Flows to Get row(s) to continue processing next contacts.  
  *Edge cases:* Missing rows or update conflicts.

- **Get row(s)**  
  *Type:* Data Table Get Rows  
  *Role:* Retrieves next batch of rows needing emails (email_subject empty).  
  *Config:* Same as previous Get row(s) node.  
  *Input:* From Approval Route (false path) and Update row(s) output.  
  *Output:* Feeds into Main Loop for continued processing.  
  *Edge cases:* No rows left to process ends workflow.

---

#### 2.4 Workflow Execution and Triggering

**Overview:**  
Provides manual and external workflow execution entry points to start the process.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- RapidAPI-Key (Set)  
- When Executed by Another Workflow (Execute Workflow Trigger)  
- Get row(s) (starting node after external trigger)

**Node Details:**

- **When clicking ‘Execute workflow’**  
  *Type:* Manual Trigger  
  *Role:* Allows manual start of the workflow from the n8n editor/interface.  
  *Config:* No parameters.  
  *Output:* Flows to RapidAPI-Key node.  
  *Edge cases:* None.

- **RapidAPI-Key**  
  *Described above.*

- **When Executed by Another Workflow**  
  *Type:* Execute Workflow Trigger  
  *Role:* Allows this workflow to be triggered and receive input from other workflows.  
  *Config:* Input source passthrough.  
  *Output:* Flows to Get row(s) node for processing.  
  *Edge cases:* Input validation needed if external workflow input differs.

- **Get row(s)**  
  *Described above.*

---

### 3. Summary Table

| Node Name                 | Node Type                               | Functional Role                                      | Input Node(s)                   | Output Node(s)                | Sticky Note                                                                                                    |
|---------------------------|---------------------------------------|-----------------------------------------------------|--------------------------------|------------------------------|---------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                       | Manual workflow starter                              | —                              | RapidAPI-Key                 |                                                                                                               |
| RapidAPI-Key              | Set                                   | Stores RapidAPI key for API authentication          | When clicking ‘Execute workflow’ | Get row(s)1                  |                                                                                                               |
| Get row(s)1               | Data Table Get Rows                    | Fetches contacts missing email_subject               | RapidAPI-Key                   | Linkedin_URL, Linkedin_URL_COMPANY, Crunchbase_URL, Call 'My workflow' |                                                                                                               |
| Linkedin_URL             | HTTP Request                          | Initiates personal LinkedIn scraping task            | Get row(s)1                   | Wait                         |                                                                                                               |
| Wait                     | Wait                                  | Delays for LinkedIn scraping task completion         | Linkedin_URL                  | results                      |                                                                                                               |
| results                  | HTTP Request                          | Fetches personal LinkedIn scraping results            | Wait                         | Update row(s)1               |                                                                                                               |
| Update row(s)1           | Data Table Update                     | Updates linkedin_profile_scrape field                 | results                      | —                            |                                                                                                               |
| Linkedin_URL_COMPANY      | HTTP Request                          | Initiates company LinkedIn scraping task              | Get row(s)1                   | Wait1                        |                                                                                                               |
| Wait1                    | Wait                                  | Delays for company LinkedIn scraping task             | Linkedin_URL_COMPANY          | results1                     |                                                                                                               |
| results1                 | HTTP Request                          | Fetches company LinkedIn scraping results             | Wait1                        | Update row(s)2               |                                                                                                               |
| Update row(s)2           | Data Table Update                     | Updates linkedin_company_scrape field                  | results1                     | —                            |                                                                                                               |
| Crunchbase_URL           | HTTP Request                          | Initiates Crunchbase company scraping task            | Get row(s)1                   | Wait2                        |                                                                                                               |
| Wait2                    | Wait                                  | Delays for Crunchbase scraping task                    | Crunchbase_URL               | results2                     |                                                                                                               |
| results2                 | HTTP Request                          | Fetches Crunchbase scraping results                    | Wait2                        | Update row(s)3               |                                                                                                               |
| Update row(s)3           | Data Table Update                     | Updates crunchbase_company_scrape field                | results2                     | —                            |                                                                                                               |
| Call 'My workflow'       | Execute Workflow                     | Invokes external sub-workflow                          | Get row(s)1                   | —                            |                                                                                                               |
| Get row(s)               | Data Table Get Rows                    | Fetches contacts missing email_subject for creative emails | When Executed by Another Workflow, Approval Route (false path), Update row(s) | Main Loop                   |                                                                                                               |
| Main Loop                | Split In Batches                      | Processes contacts in batches for email generation     | Get row(s)                   | Agent One                    |                                                                                                               |
| Agent One                | LangChain Agent                      | Generates personalized outreach email                  | Main Loop                    | Structured Output Parser     | ## Creative Outreach Agent - A dedicated agent designated for creativity and equipped with enriched data       |
| Structured Output Parser | LangChain Structured Output Parser   | Parses AI email generation output JSON                 | Agent One                   | Email Context                |                                                                                                               |
| Email Context            | Set                                   | Assigns AI output fields to workflow variables         | Structured Output Parser     | Judge Agent                 |                                                                                                               |
| Judge Agent              | LangChain Agent                      | Evaluates email quality, personalization, and tone     | Email Context                | Structured Output Parser1    |                                                                                                               |
| Structured Output Parser1| LangChain Structured Output Parser   | Parses Judge Agent approval output                      | Judge Agent                 | Approval Route              |                                                                                                               |
| Approval Route           | If Condition                        | Routes emails based on approval status                  | Structured Output Parser1    | Update row(s), Get row(s)    |                                                                                                               |
| Update row(s)            | Data Table Update                     | Updates data table with approved email subject & body | Approval Route (true path)   | Get row(s)                  |                                                                                                               |
| When Executed by Another Workflow | Execute Workflow Trigger          | External workflow trigger input                         | —                            | Get row(s)                  |                                                                                                               |
| Sticky Note              | Sticky Note                          | Document: Creative Outreach Agent description           | —                            | —                           | ## Creative Outreach Agent - A dedicated agent designated for creativity and equipped with enriched data       |
| Sticky Note1             | Sticky Note                          | Document: Data Enrichment step description               | —                            | —                           | ## Data Enrichment - Step 1; Fetch person details from local table, scrap data from multiple resources         |
| Sticky Note2             | Sticky Note                          | Document: Prerequisites and table headers                | —                            | —                           | ## Prerequisites... [includes link to RapidAPI subscription](https://rapidapi.com/ikemo-ikemo-default/api/cold-outreach-enrichment-scraper) |
| Sticky Note3             | Sticky Note                          | Document: Enrich personal LinkedIn data                   | —                            | —                           | ## Enrich personal LinkedIn data                                                                               |
| Sticky Note4             | Sticky Note                          | Document: Enrich company's LinkedIn data                  | —                            | —                           | ## Enrich Company's LinkedIn data                                                                              |
| Sticky Note5             | Sticky Note                          | Document: Enrich company's Crunchbase data                | —                            | —                           | ## Enrich Company's Crunchbase data                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - No special parameters.

2. **Create “RapidAPI-Key” Set Node**  
   - Type: Set  
   - Add field: `RapidAPI-Key` (string) with your RapidAPI key value for the cold outreach scraper API.  
   - Connect Manual Trigger → RapidAPI-Key.

3. **Create “Get row(s)1” Data Table Node**  
   - Operation: Get rows  
   - Table: `email_linkedin_list`  
   - Filter: `email_subject` is empty  
   - Connect RapidAPI-Key → Get row(s)1.

4. **Create HTTP Request Node “Linkedin_URL”**  
   - URL: `https://cold-outreach-enrichment-scraper.p.rapidapi.com/company_url`  
   - Query Parameter: `url` = `{{$json.Linkedin_URL}}`  
   - Header:  
     - `x-rapidapi-host`: `cold-outreach-enrichment-scraper.p.rapidapi.com`  
     - `x-rapidapi-key`: `{{$node["RapidAPI-Key"].json["RapidAPI-Key"]}}`  
   - Enable batching with batch size 1.  
   - Connect Get row(s)1 → Linkedin_URL.

5. **Create Wait Node “Wait”**  
   - Unit: minutes (duration depends on expected scraping delay)  
   - Connect Linkedin_URL → Wait.

6. **Create HTTP Request Node “results”**  
   - URL: `https://cold-outreach-enrichment-scraper.p.rapidapi.com/results`  
   - Query Parameter: `task_id` = `{{$json.task_id}}` (from Linkedin_URL response)  
   - Use same RapidAPI headers as above.  
   - Batching enabled.  
   - Connect Wait → results.

7. **Create Data Table Update Node “Update row(s)1”**  
   - Operation: Update rows  
   - Match column: `email` equals `{{$node["Get row(s)1"].json.email}}`  
   - Update field: `linkedin_profile_scrape` = `{{ $json.results.toJsonString() }}`  
   - Connect results → Update row(s)1.

8. **Repeat steps 4-7 for Company LinkedIn URL**  
   - Node names: Linkedin_URL_COMPANY, Wait1, results1, Update row(s)2  
   - Use `company_linkedin` as URL parameter.  
   - Connect Get row(s)1 → Linkedin_URL_COMPANY.

9. **Repeat steps 4-7 for Crunchbase URL**  
   - Node names: Crunchbase_URL, Wait2, results2, Update row(s)3  
   - Use `Crunchbase_URL` as URL parameter.  
   - Connect Get row(s)1 → Crunchbase_URL.

10. **Create “Call 'My workflow'” Execute Workflow Node**  
    - Select sub-workflow (ID: 8PMrSTmEahhkUV53)  
    - No inputs required, no wait for completion.  
    - Connect Get row(s)1 → Call 'My workflow'.

11. **Create “Get row(s)” Data Table Node for Email Generation**  
    - Same configuration as Get row(s)1 (filter email_subject empty).  
    - Connect external trigger “When Executed by Another Workflow” and Approval Route fallback → Get row(s).

12. **Create “Main Loop” Split In Batches Node**  
    - Default options.  
    - Connect Get row(s) → Main Loop.

13. **Create “Agent One” LangChain Agent Node**  
    - Text input includes templated contact and company info, plus an “About Me” section.  
    - Set system message with detailed instructions for creative outreach email generation (see original prompt).  
    - Max iterations: 10  
    - Use Google Gemini Chat model as AI backend with credentials configured.  
    - Enable output parser.  
    - Connect Main Loop (batch output) → Agent One.

14. **Create “Structured Output Parser” Node**  
    - JSON schema: keys `email_subject` and `email_content`.  
    - Connect Agent One → Structured Output Parser.

15. **Create “Email Context” Set Node**  
    - Assign `email_subject` and `email_content` from parser output.  
    - Assign `email` from Main Loop item data.  
    - Connect Structured Output Parser → Email Context.

16. **Create “Judge Agent” LangChain Agent Node**  
    - Input text template includes `email_subject` and `email_content`.  
    - System message defines evaluation criteria for email quality and personalization.  
    - Max iterations: 20  
    - Use Google Gemini Chat model as AI backend.  
    - Set onError to continue with error output.  
    - Enable output parser.  
    - Connect Email Context → Judge Agent.

17. **Create “Structured Output Parser1” Node**  
    - JSON schema keys: `approval` (APPROVED or REVISE), `feedback` (optional).  
    - Connect Judge Agent → Structured Output Parser1.

18. **Create “Approval Route” If Node**  
    - Condition: `approval` field contains string "APPROVED" (case sensitive).  
    - Connect Structured Output Parser1 → Approval Route.

19. **Create “Update row(s)” Data Table Update Node**  
    - On true path from Approval Route.  
    - Match by `email`.  
    - Update `email_subject` and `email_body` fields with approved email content.  
    - Connect Approval Route (true) → Update row(s).

20. **Connect Update row(s) → Get row(s)**  
    - Loop to process next contact.

21. **Connect Approval Route (false) → Get row(s)**  
    - For emails needing revision, reprocess via main loop.

22. **Create “When Executed by Another Workflow” Trigger Node**  
    - Input passthrough.  
    - Connect to Get row(s).

23. **Configure Credentials**  
    - Google Gemini (PaLM) API credentials for LangChain nodes.  
    - RapidAPI key configured in the Set node.  
    - Data table credentials and access configured in Data Table nodes.

24. **Add Sticky Notes**  
    - Add notes describing each block and prerequisites as per original content for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                           | Context or Link                                                                                                             |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| Workflow utilizes RapidAPI cold outreach enrichment scraper for LinkedIn and Crunchbase data enrichment. Subscription required.                                                                                                                        | [RapidAPI Scraper Subscription](https://rapidapi.com/ikemo-ikemo-default/api/cold-outreach-enrichment-scraper)               |
| Creative Outreach Agent designed to write warm, concise, research-driven outreach emails that emphasize personalization and value, avoiding sales pressure.                                                                                              | Described in Agent One system message.                                                                                      |
| Judge Agent ensures only high-quality, personalized emails are sent out, requesting revisions when necessary.                                                                                                                                           | Described in Judge Agent system message.                                                                                    |
| Data Table `email_linkedin_list` schema includes fields like First_name, Last_name, email, Title, Location, Company info, Linkedin and Crunchbase URLs, scraped data fields, and email content fields.                                                    | Referenced in sticky notes and node configurations.                                                                         |
| Workflow supports manual execution and triggering by other workflows, allowing flexible integration into larger automation pipelines.                                                                                                                  | Nodes: Manual Trigger and Execute Workflow Trigger.                                                                          |
| Google Gemini (PaLM) API credentials must be configured in n8n for LangChain nodes to operate correctly.                                                                                                                                                 | Credential setup is prerequisite.                                                                                             |
| Wait nodes implement asynchronous polling to allow external scraper tasks time to complete before fetching results, avoiding premature data requests.                                                                                                  | Wait duration should be tuned based on API response times and rate limits.                                                  |
| The workflow’s architecture ensures continuous processing by looping back on unapproved emails for refinement, preventing low-quality outreach.                                                                                                       | Loop implemented via Approval Route node.                                                                                     |

---

**Disclaimer:**  
The text provided derives entirely from an n8n automated workflow designed for legal, ethical, and public data processing. It complies fully with all content policies and contains no illegal, offensive, or protected elements. All data handled is lawful and publicly accessible.