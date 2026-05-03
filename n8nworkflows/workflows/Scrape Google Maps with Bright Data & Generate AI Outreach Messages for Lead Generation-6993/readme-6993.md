Scrape Google Maps with Bright Data & Generate AI Outreach Messages for Lead Generation

https://n8nworkflows.xyz/workflows/scrape-google-maps-with-bright-data---generate-ai-outreach-messages-for-lead-generation-6993


# Scrape Google Maps with Bright Data & Generate AI Outreach Messages for Lead Generation

---

## 1. Workflow Overview

This workflow automates lead generation by scraping business information from Google Maps using Bright Data's API, enriching that data with AI-generated outreach messages, and storing the results in a Postgres database (Supabase). It is designed for marketing agencies, sales teams, or business developers who want to quickly collect targeted leads and prepare personalized cold call scripts.

The workflow is logically divided into these blocks:

- **1.1 Input Reception & Validation**  
  Receives user input from a web form with Google Maps URL, keyword, country, and company segment. Extracts geographic coordinates from the URL for scraping.

- **1.2 Data Scraping with Bright Data**  
  Initiates a scraping job on Bright Data’s platform, monitors the job’s progress with retry logic, and downloads the results once ready.

- **1.3 Data Processing & Limiting**  
  Organizes and limits the scraped data to a manageable number of leads for further processing.

- **1.4 AI Enrichment**  
  For each lead, uses AI tools (Bright Data MCP and OpenAI/Gemini LLMs) to scrape the company website, summarize company info, and generate personalized cold call scripts and talking points.

- **1.5 Data Storage**  
  Upserts enriched lead data, including AI-generated messages, into a Postgres table hosted on Supabase.

- **1.6 Workflow Initialization & Setup**  
  Includes nodes for manual trigger to create the database table and sticky notes for instructions and references.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception & Validation

**Overview:**  
Receives user inputs via a form trigger and extracts latitude and longitude from the provided Google Maps URL for use in scraping.

**Nodes Involved:**  
- On form submission  
- Extract latitude and logitude from URL

**Node Details:**

- **On form submission**  
  - *Type:* Form Trigger  
  - *Role:* Entry point for user input: Location URL, Keyword, Country, and Company segment.  
  - *Config:* Form fields are mandatory. The form responds immediately with "Scraping started 🔎".  
  - *Connections:* Outputs to "Extract latitude and logitude from URL".  
  - *Edge cases:* Invalid or malformed URLs could fail regex extraction downstream.

- **Extract latitude and logitude from URL**  
  - *Type:* Set  
  - *Role:* Parses latitude and longitude from the Location URL using regex on the '@' pattern in the URL string.  
  - *Config:* Extracts two string fields `latitude` and `longitude` from captured regex groups. Passes through all other fields.  
  - *Connections:* Outputs to "Bright Data | Request data".  
  - *Edge cases:* URL not matching expected Google Maps format will cause errors or empty values.

---

### 2.2 Data Scraping with Bright Data

**Overview:**  
Triggers a Bright Data scraping job for Google Maps, monitors its progress with retries, and downloads snapshot results when ready.

**Nodes Involved:**  
- Bright Data | Request data  
- Count1  
- No Operation, do nothing4  
- Count Increment1  
- Request finished  
- Reached retry limit  
- Wait5  
- Check status of data extraction  
- Download the snapshot content  
- Snapshot finished building

**Node Details:**

- **Bright Data | Request data**  
  - *Type:* HTTP Request (Bright Data API)  
  - *Role:* Starts a scraping job with parameters from form input (country, keyword, lat, long) and Bright Data dataset ID for Google Maps.  
  - *Config:* Uses Bright Data API credentials; sends JSON body with search parameters and query parameters specifying dataset and scrape type.  
  - *Connections:* Outputs to "Count1".  
  - *Edge cases:* API auth failure, invalid parameters, network errors.

- **Count1**  
  - *Type:* Set  
  - *Role:* Initializes a retry counter `count` to 0 for monitoring scraping job status.  
  - *Connections:* Outputs to "No Operation, do nothing4".

- **No Operation, do nothing4**  
  - *Type:* NoOp (pass-through)  
  - *Role:* Placeholder to carry data forward without modification.  
  - *Connections:* Outputs to "Check status of data extraction".

- **Check status of data extraction**  
  - *Type:* Bright Data node (monitorProgressSnapshot)  
  - *Role:* Checks the current status of the scraping job snapshot by snapshot_id.  
  - *Connections:* Outputs to "Request finished".

- **Request finished**  
  - *Type:* If  
  - *Role:* Checks if scraping status is no longer "running". If finished, continue to download snapshot; else check if retry limit reached.  
  - *Connections:* True branch to "Download the snapshot content", False branch to "Reached retry limit".

- **Reached retry limit**  
  - *Type:* If  
  - *Role:* Checks if retry count has reached 10 attempts.  
  - *Connections:* If yes, no further retry (ends); if no, triggers a wait before retry.  
  - *Edge cases:* Infinite loop prevented by retry count limit.

- **Wait5**  
  - *Type:* Wait  
  - *Role:* Waits 60 seconds before incrementing retry count and rechecking status.  
  - *Connections:* Outputs to "Count Increment1".

- **Count Increment1**  
  - *Type:* Set  
  - *Role:* Increments retry counter `count` by 1.  
  - *Connections:* Outputs to "No Operation, do nothing4" (loop back).

- **Download the snapshot content**  
  - *Type:* Bright Data node (downloadSnapshot)  
  - *Role:* Downloads the scraped data snapshot once ready.  
  - *Connections:* Outputs to "Snapshot finished building".

- **Snapshot finished building**  
  - *Type:* If  
  - *Role:* Confirms snapshot status is not "building".  
  - *Connections:* True branch to "Organize data", False branch to "Wait" (10 seconds wait before retry).

---

### 2.3 Data Processing & Limiting

**Overview:**  
Organizes the downloaded data, extracts relevant fields, and limits the number of items to 15 for further processing.

**Nodes Involved:**  
- Organize data  
- Set Limit  
- Loop Over Items  
- Company website exists

**Node Details:**

- **Organize data**  
  - *Type:* Set  
  - *Role:* Prepares structured fields from raw scraped JSON, mapping place ID, company details, reviews, and URLs.  
  - *Connections:* Outputs to "Set Limit".

- **Set Limit**  
  - *Type:* Limit  
  - *Role:* Restricts the number of leads processed downstream to a maximum of 15.  
  - *Connections:* Outputs to "Loop Over Items".

- **Loop Over Items**  
  - *Type:* Split In Batches  
  - *Role:* Processes each lead individually in a loop for AI enrichment and database upsert.  
  - *Connections:* Outputs to "Company website exists" on second output, meanwhile first output is for empty branch.

- **Company website exists**  
  - *Type:* If  
  - *Role:* Checks if the company website URL exists and is non-empty to decide on further scraping.  
  - *Connections:* True branch to "Scrape & Summarize", False branch to "Merge" (skip scraping).

---

### 2.4 AI Enrichment

**Overview:**  
Uses AI to scrape each company's website homepage, summarize business info, and generate a personalized cold call message and talking points.

**Nodes Involved:**  
- Scrape & Summarize  
- scrape_as_markdown  
- OpenAI Chat Model  
- Google Gemini Chat Model  
- Message Generator  
- Set Fields  
- Merge

**Node Details:**

- **Scrape & Summarize**  
  - *Type:* LangChain Agent (AI agent)  
  - *Role:* Uses the Bright Data MCP scrape_as_markdown tool to scrape the company's homepage and generate a 300-500 word structured summary of the business, including mission, products, audience, and key messaging.  
  - *Config:* System message instructs focused scraping and summary output format.  
  - *Connections:* Outputs to "Set Fields".  
  - *Edge cases:* Website inaccessible, scraping failures, AI timeouts.

- **scrape_as_markdown**  
  - *Type:* LangChain MCP Client Tool  
  - *Role:* Provides the scraping tool via Bright Data MCP, used by "Scrape & Summarize" node.  
  - *Connections:* Used as AI tool by "Scrape & Summarize".

- **OpenAI Chat Model**  
  - *Type:* LangChain OpenAI LLM  
  - *Role:* Provides GPT-4.1-mini model for natural language generation.  
  - *Connections:* Feeds output to "Scrape & Summarize" for AI processing.

- **Google Gemini Chat Model**  
  - *Type:* LangChain Google Gemini LLM  
  - *Role:* Alternative LLM model (Gemini 2.5 Pro) for message generation.  
  - *Connections:* Feeds output to "Message Generator".

- **Message Generator**  
  - *Type:* LangChain Chain LLM  
  - *Role:* Generates personalized cold call opening messages and talking points based on structured lead data and company segment from form input.  
  - *Config:* Uses a detailed prompt with instructions and output format for cold call script and call support notes.  
  - *Connections:* Outputs to "Supabase | Upsert row".  
  - *Edge cases:* AI generation errors, token limits.

- **Set Fields**  
  - *Type:* Set  
  - *Role:* Combines AI-generated summary and company data into a structured object for database insertion.  
  - *Connections:* Outputs to "Merge".

- **Merge**  
  - *Type:* Merge  
  - *Role:* Merges data from "Set Fields" and "Message Generator" to consolidate enriched lead data.  
  - *Connections:* Outputs to "Supabase | Upsert row".

---

### 2.5 Data Storage

**Overview:**  
Upserts enriched lead data into a Supabase Postgres database table for later use in outreach campaigns.

**Nodes Involved:**  
- Supabase | Upsert row  
- Create Table

**Node Details:**

- **Supabase | Upsert row**  
  - *Type:* Postgres node  
  - *Role:* Inserts or updates a row in the `business_scraping_result` table with enriched lead data and AI-generated sales helper text.  
  - *Config:* Matches on `place_id` primary key; maps all relevant columns like name, rating, address, reviews, summaries, and generated messages.  
  - *Connections:* Outputs back to "Loop Over Items" for next batch item.  
  - *Edge cases:* Database connection errors, constraint violations.

- **Create Table**  
  - *Type:* Postgres node (manual trigger)  
  - *Role:* Defines the database schema for storing scraped and enriched data.  
  - *Config:* SQL CREATE TABLE statement defining columns and types, including JSONB for reviews and text arrays for services.  
  - *Connections:* Triggered from manual trigger node.  
  - *Edge cases:* Table already exists errors.

---

### 2.6 Workflow Initialization & Setup

**Overview:**  
Nodes to facilitate workflow initialization, manual triggers, and user instructions.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (manual trigger)  
- Sticky Notes (multiple)

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Allows manual execution to create the database table before running the full workflow.  
  - *Connections:* Outputs to "Create Table".  
  - *Edge cases:* Must be run once before normal operation.

- **Sticky Notes**  
  - *Type:* Sticky Note  
  - *Role:* Provide guidance, instructions, references to official docs, and project credits throughout the workflow canvas.  
  - *Content Highlights:*  
    - Retry logic explanation  
    - Bright Data MCP token placement  
    - Database connection instructions  
    - Workflow purpose and usage instructions  
    - Author info and external links

---

## 3. Summary Table

| Node Name                     | Node Type                         | Functional Role                                | Input Node(s)                | Output Node(s)                  | Sticky Note                                                                                         |
|-------------------------------|----------------------------------|-----------------------------------------------|------------------------------|---------------------------------|---------------------------------------------------------------------------------------------------|
| On form submission             | Form Trigger                     | Entry point to receive user inputs            |                              | Extract latitude and logitude from URL |                                                                                                   |
| Extract latitude and logitude from URL | Set                            | Parses coordinates from Google Maps URL       | On form submission            | Bright Data | Request data            |                                                                                                   |
| Bright Data | Request data        | HTTP Request (Bright Data API)    | Starts scraping job on Bright Data             | Extract latitude and logitude from URL | Count1                       | Contains Bright Data API token note                                                               |
| Count1                        | Set                              | Initializes retry counter                       | Bright Data | Request data            | No Operation, do nothing4         | Retry logic explanation                                                                            |
| No Operation, do nothing4     | No Operation                     | Pass-through node                              | Count1                       | Check status of data extraction  | Retry logic explanation                                                                            |
| Check status of data extraction | Bright Data node (monitorProgressSnapshot) | Monitors scraping job status                   | No Operation, do nothing4     | Request finished                | Retry logic explanation                                                                            |
| Request finished              | If                               | Checks if scraping is finished or still running| Check status of data extraction | Download the snapshot content, Reached retry limit | Retry logic explanation                                                                            |
| Reached retry limit           | If                               | Checks if retry count reached max (10)        | Request finished             | Wait5 (if not reached)           | Retry logic explanation                                                                            |
| Wait5                        | Wait                             | Waits 60 seconds before retry                  | Reached retry limit          | Count Increment1                 | Retry logic explanation                                                                            |
| Count Increment1             | Set                              | Increments retry count                         | Wait5                        | No Operation, do nothing4        | Retry logic explanation                                                                            |
| Download the snapshot content | Bright Data node (downloadSnapshot) | Downloads scraped data snapshot                 | Request finished             | Snapshot finished building       | Retry logic explanation                                                                            |
| Snapshot finished building    | If                               | Confirms snapshot is ready                      | Download the snapshot content | Organize data, Wait              | Retry logic explanation                                                                            |
| Wait                         | Wait                             | Waits 10 seconds before rechecking snapshot    | Snapshot finished building    | Check status of data extraction  | Retry logic explanation                                                                            |
| Organize data                | Set                              | Maps and structures scraped data                | Snapshot finished building    | Set Limit                       | Limit the amount of items to save note                                                            |
| Set Limit                   | Limit                            | Restricts number of leads to 15                 | Organize data                | Loop Over Items                 | Limit the amount of items to save note                                                            |
| Loop Over Items             | Split In Batches                 | Processes each lead individually                | Set Limit                   | Company website exists (conditional) | Loop usage explanation                                                                             |
| Company website exists       | If                               | Checks if company website URL exists            | Loop Over Items              | Scrape & Summarize (if true), Merge (if false) |                                                                                                   |
| Scrape & Summarize           | LangChain Agent                  | Scrapes website and summarizes company info   | Company website exists       | Set Fields                     |                                                                                                   |
| scrape_as_markdown           | LangChain MCP Client Tool        | Tool for scraping websites                      |                             | Scrape & Summarize (as AI tool) | Bright Data MCP token note                                                                         |
| OpenAI Chat Model            | LangChain LLM                   | Provides GPT-4.1-mini model for AI tasks        |                             | Scrape & Summarize             |                                                                                                   |
| Google Gemini Chat Model      | LangChain LLM                   | Alternative LLM (Gemini) for message generation|                             | Message Generator              |                                                                                                   |
| Message Generator            | LangChain Chain LLM              | Generates cold call script and talking points | Merge                       | Supabase | Upsert row             |                                                                                                   |
| Set Fields                  | Set                              | Combines AI summary and company data           | Scrape & Summarize          | Merge                         |                                                                                                   |
| Merge                       | Merge                           | Combines data from Set Fields and Message Generator | Company website exists, Set Fields | Supabase | Upsert row            |                                                                                                   |
| Supabase | Upsert row          | Postgres Node                   | Inserts or updates enriched lead data           | Merge                       | Loop Over Items               | Supabase connection instructions note                                                            |
| Create Table                | Postgres Node                   | Creates database table schema                    | When clicking ‘Execute workflow’ |                             | Run this to create your Supabase table note                                                       |
| When clicking ‘Execute workflow’ | Manual Trigger                 | Manual start to create table                     |                             | Create Table                  |                                                                                                   |
| Sticky Note (various)        | Sticky Note                     | Provides explanations, instructions, credits   |                             |                             | See individual notes in the workflow for details                                                  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Name: "When clicking ‘Execute workflow’"  
   - Purpose: To run initial setup for database.  
   - Connect it to a Postgres Node.

2. **Create Postgres Node for Table Creation**  
   - Name: "Create Table"  
   - Operation: Execute Query  
   - Query: SQL to create `business_scraping_result` table with appropriate columns as per schema in overview.  
   - Connect "When clicking ‘Execute workflow’" → "Create Table".  
   - Configure with Supabase Postgres credentials (Transaction Pooler parameters).

3. **Create Form Trigger Node**  
   - Name: "On form submission"  
   - Configure form fields: Location URL (string, required), Keyword Search (string, required), Country (string, required), My company’s segment (string, required).  
   - Configure form response: Button label "Scrape", form submitted text "Scraping started 🔎".  
   - This is the entry point for user input.

4. **Add Set Node to Extract Latitude and Longitude**  
   - Name: "Extract latitude and logitude from URL"  
   - Use regex on the Location URL to extract latitude and longitude into variables `latitude` and `longitude`.  
   - Pass through all other form variables.  
   - Connect "On form submission" → "Extract latitude and logitude from URL".

5. **Add HTTP Request Node to Trigger Bright Data Scrape**  
   - Name: "Bright Data | Request data"  
   - Method: POST  
   - URL: `https://api.brightdata.com/datasets/v3/trigger`  
   - Body (JSON): include country, keyword, latitude, longitude, zoom level 14.  
   - Query parameters: dataset_id for Google Maps scraper, include_errors=true, type=discover_new, discover_by=location.  
   - Authentication: Use Bright Data API credential.  
   - Connect "Extract latitude and logitude from URL" → "Bright Data | Request data".

6. **Add Set Node to Initialize Retry Counter**  
   - Name: "Count1"  
   - Set variable `count` = 0.  
   - Connect "Bright Data | Request data" → "Count1".

7. **Add No Operation Node**  
   - Name: "No Operation, do nothing4"  
   - Connect "Count1" → "No Operation, do nothing4".

8. **Add Bright Data Node to Monitor Scrape Status**  
   - Name: "Check status of data extraction"  
   - Operation: monitorProgressSnapshot  
   - Use snapshot_id from "Bright Data | Request data".  
   - Connect "No Operation, do nothing4" → "Check status of data extraction".

9. **Add If Node to Check If Scrape Finished**  
   - Name: "Request finished"  
   - Condition: status != "running"  
   - True → "Download the snapshot content"  
   - False → "Reached retry limit"  
   - Connect "Check status of data extraction" → "Request finished".

10. **Add If Node to Check Retry Limit**  
    - Name: "Reached retry limit"  
    - Condition: count == 10  
    - True → End (no output)  
    - False → "Wait5" (wait 60 seconds)  
    - Connect "Request finished" False → "Reached retry limit".

11. **Add Wait Node**  
    - Name: "Wait5"  
    - Wait 60 seconds  
    - Connect "Reached retry limit" False → "Wait5".

12. **Add Set Node to Increment Retry Counter**  
    - Name: "Count Increment1"  
    - Set count = previous count + 1 using expression.  
    - Connect "Wait5" → "Count Increment1".

13. **Connect Count Increment1 back to No Operation Node**  
    - Connect "Count Increment1" → "No Operation, do nothing4" (loop).

14. **Add Bright Data Node to Download Snapshot**  
    - Name: "Download the snapshot content"  
    - Operation: downloadSnapshot  
    - Use snapshot_id from "Bright Data | Request data".  
    - Connect "Request finished" True → "Download the snapshot content".

15. **Add If Node to Confirm Snapshot Ready**  
    - Name: "Snapshot finished building"  
    - Condition: status != "building"  
    - True → "Organize data"  
    - False → "Wait" (10 seconds)  
    - Connect "Download the snapshot content" → "Snapshot finished building".

16. **Add Wait Node**  
    - Name: "Wait"  
    - Wait 10 seconds  
    - Connect "Snapshot finished building" False → "Wait".  
    - Connect "Wait" → "Check status of data extraction".

17. **Add Set Node to Organize Data**  
    - Name: "Organize data"  
    - Map raw snapshot data fields to structured format (place_id, company details, reviews, URLs).  
    - Connect "Snapshot finished building" True → "Organize data".

18. **Add Limit Node**  
    - Name: "Set Limit"  
    - Max items: 15 (limit leads processed).  
    - Connect "Organize data" → "Set Limit".

19. **Add Split In Batches Node**  
    - Name: "Loop Over Items"  
    - Loop over limited leads individually.  
    - Connect "Set Limit" → "Loop Over Items".

20. **Add If Node to Check Company Website Exists**  
    - Name: "Company website exists"  
    - Condition: company.open_website exists and non-empty.  
    - True → "Scrape & Summarize"  
    - False → "Merge" (skip scraping, merge data).  
    - Connect "Loop Over Items" → "Company website exists".

21. **Add LangChain MCP Client Tool Node**  
    - Name: "scrape_as_markdown"  
    - Configure to use Bright Data MCP endpoint with your token.  
    - Include tool "scrape_as_markdown".  
    - No direct input connections; used by "Scrape & Summarize".

22. **Add LangChain Agent Node**  
    - Name: "Scrape & Summarize"  
    - Use "scrape_as_markdown" tool to scrape company website homepage and generate structured summary (300-500 words).  
    - Configure system message with detailed instructions on scraping and summary format.  
    - Connect "Company website exists" True → "Scrape & Summarize".

23. **Add Set Node to Prepare Data Fields**  
    - Name: "Set Fields"  
    - Combine scraped summary and company data into structured object.  
    - Connect "Scrape & Summarize" → "Set Fields".

24. **Add Merge Node**  
    - Name: "Merge"  
    - Merge output from "Set Fields" (True branch from scraping) and False branch from "Company website exists" (no scraping).  
    - Connect "Company website exists" False → "Merge".  
    - Connect "Set Fields" → "Merge".

25. **Add LangChain Chain LLM Node**  
    - Name: "Message Generator"  
    - Input: JSON stringified merged company data and form input segment.  
    - Prompt: Generate cold call opening message + talking points with detailed instructions.  
    - Model Credentials: Choose LLM provider (OpenAI or Gemini).  
    - Connect "Merge" → "Message Generator".

26. **Add Postgres Node to Upsert Data**  
    - Name: "Supabase | Upsert row"  
    - Operation: Upsert into `business_scraping_result` table using place_id as primary key.  
    - Map all relevant fields including AI-generated sales helpers.  
    - Connect "Message Generator" → "Supabase | Upsert row".

27. **Connect "Supabase | Upsert row" back to "Loop Over Items"**  
    - This completes the loop to process all leads.

28. **Add and Configure Credentials**  
    - Bright Data API credentials for scraping and MCP.  
    - Supabase Postgres credentials using Transaction Pooler parameters.  
    - OpenAI or Google Gemini credentials for AI nodes.

29. **Add Sticky Notes** at strategic points describing:  
    - Retry logic and loop explanation.  
    - Bright Data MCP token placement.  
    - Supabase connection guide.  
    - Overall workflow purpose and author credits.

---

## 5. General Notes & Resources

| Note Content                                                                                                           | Context or Link                                                                                   |
|------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This n8n workflow automates lead extraction from Google Maps, enriches data with AI, and stores results for cold outreach. It uses Bright Data MCP and community nodes for scraping and AI message generation. | Sticky Note5 content on the canvas                                                              |
| To connect to Supabase using Postgres credentials, use the Transaction Pooler parameters from Supabase dashboard. Official documentation: https://supabase.com/docs/guides/database/connecting-to-postgres | Sticky Note8 and Sticky Note7                                                                   |
| The `zoom_level` parameter controls the search radius in Google Maps scraping; lower zoom = larger radius. The `dataset_id` corresponds to Bright Data’s Google Maps scraper. | Sticky Note3                                                                                     |
| Retry logic implemented: The workflow polls the scraping job status every 60 seconds, retrying up to 10 times until data is ready. | Sticky Note explaining retry logic (Sticky Note near Count1 and No Operation nodes)               |
| Bright Data MCP token needs to be set in the "scrape_as_markdown" node to enable AI scraping tools.                    | Sticky Note1                                                                                     |
| Author: Solomon - AI & Automation Educator. Follow on LinkedIn https://www.linkedin.com/in/guisalomao/ and other social links. | Sticky Note10                                                                                   |
| For advanced n8n skills and workflow monetization, see Scrapes Academy: https://www.skool.com/scrapes/about?ref=21f10ad99f4d46ba9b8aaea8c9f58c34 | Sticky Note16                                                                                   |

---

**Disclaimer:** The text provided is exclusively from an automated workflow created with n8n, respecting current content policies, containing no illegal, offensive, or protected elements. All data is legal and public.

---