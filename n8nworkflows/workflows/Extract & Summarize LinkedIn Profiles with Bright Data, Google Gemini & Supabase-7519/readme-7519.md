Extract & Summarize LinkedIn Profiles with Bright Data, Google Gemini & Supabase

https://n8nworkflows.xyz/workflows/extract---summarize-linkedin-profiles-with-bright-data--google-gemini---supabase-7519


# Extract & Summarize LinkedIn Profiles with Bright Data, Google Gemini & Supabase

### 1. Workflow Overview

This workflow automates the extraction, summarization, and storage of LinkedIn profile data using Bright Data for scraping, Google Gemini AI models for natural language processing, and Supabase as a backend database. It is designed to handle LinkedIn URLs submitted via webhook, extract detailed profile information, process it into structured summaries and categorized insights (such as skills, experiences, and emerging roles), and store or update the results in a Supabase database.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Data Extraction:** Receive input via webhook, set input fields, scrape LinkedIn profile data using Bright Data.
- **1.2 Data Validation & Error Handling:** Check HTTP status codes; respond immediately on errors.
- **1.3 AI Processing - Information Extraction:** Multiple Google Gemini AI models extract and summarize different profile sections: basic info, skills, experiences, summaries, emerging roles, and markdown content.
- **1.4 Data Aggregation & Merging:** Merge extracted data from various AI nodes.
- **1.5 Database Operations:** Conditionally create or update rows in Supabase, based on record existence and force-create flags.
- **1.6 Webhook Responses & Error Termination:** Provide appropriate webhook responses for success, errors, or missing records.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Data Extraction

**Overview:**  
Handles receiving the initial LinkedIn URL and scraping profile data from that URL using Bright Data's scraping service.

**Nodes Involved:**  
- Webhook  
- Set the Input Fields  
- Access and extract data from a specific URL (Bright Data)  
- Set the Scraped Response  

**Node Details:**

- **Webhook**  
  - Type: HTTP webhook receiver  
  - Role: Entry point receiving requests with LinkedIn profile URLs or parameters  
  - Configuration: Listens for incoming HTTP requests, triggers the workflow  
  - Inputs: External HTTP requests  
  - Outputs: Passes data to “Set the Input Fields”  
  - Edge Cases: Invalid or malformed requests, missing parameters  

- **Set the Input Fields**  
  - Type: Set node  
  - Role: Prepares and structures input data for scraping  
  - Configuration: May define defaults or extract specific fields from webhook data  
  - Inputs: Webhook data  
  - Outputs: To Bright Data node  
  - Edge Cases: Missing fields leading to incomplete scraping requests  

- **Access and extract data from a specific URL (Bright Data)**  
  - Type: Bright Data scraping node  
  - Role: Scrapes the LinkedIn profile page content using Bright Data infrastructure  
  - Configuration: Uses preconfigured Bright Data credentials and settings, retry enabled on failure  
  - Inputs: URL from “Set the Input Fields”  
  - Outputs: Raw scraped HTML or data to “Set the Scraped Response”  
  - Edge Cases: Scraping timeouts, IP blocks, CAPTCHA challenges, Bright Data auth errors  

- **Set the Scraped Response**  
  - Type: Set node  
  - Role: Prepares scraped data for validation and further processing  
  - Configuration: Structures data, adds status codes or flags  
  - Inputs: Bright Data scraped output  
  - Outputs: To status code check node  
  - Edge Cases: Empty or malformed scraped data  

---

#### 2.2 Data Validation & Error Handling

**Overview:**  
Checks the HTTP status code of the scraping result to decide whether to proceed or respond with an error.

**Nodes Involved:**  
- If status code <> 200  
- Respond to Webhook  
- Basic Profile Info  
- Emerging Roles  
- Markdown Content  
- Summarizer  
- Skills Extractor  

**Node Details:**

- **If status code <> 200**  
  - Type: If node  
  - Role: Conditional branch to separate error cases (non-200 HTTP status) from successful scrapes  
  - Configuration: Checks if status code differs from 200  
  - Inputs: “Set the Scraped Response” output  
  - Outputs:  
    - True branch: send error response  
    - False branch: trigger AI extraction nodes  
  - Edge Cases: Unexpected status codes, partial or delayed HTTP responses  

- **Respond to Webhook**  
  - Type: Respond to Webhook node  
  - Role: Sends an HTTP response back to the original webhook caller in error cases  
  - Configuration: Typically sends error messages or codes  
  - Inputs: From “If status code <> 200” true branch  
  - Outputs: Ends workflow branch  
  - Edge Cases: Network issues, response formatting errors  

- **Basic Profile Info, Emerging Roles, Markdown Content, Summarizer, Skills Extractor**  
  - Type: LangChain Information Extractor nodes  
  - Role: Prepare for AI processing on successful data extraction  
  - Configuration: Each node is triggered only if status code is 200 (successful scrape)  
  - Inputs: Scraped data  
  - Outputs: To respective AI models for deeper processing  
  - Edge Cases: None in this block; processing deferred to AI nodes  

---

#### 2.3 AI Processing - Information Extraction

**Overview:**  
Processes scraped profile data through specialized Google Gemini AI chat models to extract structured profile insights.

**Nodes Involved:**  
- Google Gemini Chat Model for Basic Profile Info  
- Basic Profile Info  
- Google Gemini Chat Model for Emerging Roles  
- Emerging Roles  
- Google Gemini Chat Model for Summarization  
- Summarizer  
- Google Gemini Chat Model for Skill Extraction  
- Skills Extractor  
- Google Gemini Chat Model for Experiences  
- Markdown Content  

**Node Details:**

- **Google Gemini Chat Model for Basic Profile Info**  
  - Type: LangChain Google Gemini LM Chat node  
  - Role: AI model configured to extract fundamental profile information (e.g., name, title, location)  
  - Inputs: Raw or preprocessed scraped data  
  - Outputs: To “Basic Profile Info” node for structured extraction  
  - Edge Cases: API auth failures, rate limits, unexpected data formats  

- **Basic Profile Info**  
  - Type: LangChain Information Extractor  
  - Role: Extracts structured data fields from AI model output  
  - Inputs: Output of Gemini chat model  
  - Outputs: To merge node  
  - Edge Cases: Parsing errors, incomplete extraction  

- **Google Gemini Chat Model for Emerging Roles**  
  - Role: Extracts emerging or future roles from profile text  
  - Connected to “Emerging Roles” node  

- **Emerging Roles**  
  - Extracts structured emerging role info from AI output  

- **Google Gemini Chat Model for Summarization**  
  - Role: Summarizes profile content into concise format  
  - Connected to “Summarizer” node  

- **Summarizer**  
  - Extracts summary text  

- **Google Gemini Chat Model for Skill Extraction**  
  - Role: Extracts skills and expertise listed or implied in profile  
  - Connected to “Skills Extractor” node  

- **Skills Extractor**  
  - Extracts structured skill data  

- **Google Gemini Chat Model for Experiences**  
  - Role: Extracts detailed professional experiences in markdown format  
  - Connected to “Markdown Content” node  

- **Markdown Content**  
  - Extracts markdown formatted experience details  

- **General Notes:**  
  - All AI nodes have retry enabled to handle transient failures.  
  - They rely on Google Gemini API credentials configured in n8n.  
  - Potential failures include API quota limits, network issues, or unexpected input data formats.  

---

#### 2.4 Data Aggregation & Merging

**Overview:**  
Combines all extracted information into a single unified dataset for database storage.

**Nodes Involved:**  
- Merge  
- Aggregate  

**Node Details:**

- **Merge**  
  - Type: Merge node  
  - Role: Collects and merges outputs from multiple AI extraction nodes (Basic Profile Info, Emerging Roles, Markdown Content, Summarizer, Skills Extractor)  
  - Inputs: Outputs from all extraction nodes  
  - Outputs: To “Aggregate” node  
  - Configuration: Typically merges based on index or key to ensure data alignment  
  - Edge Cases: Missing or partial data from some branches may cause incomplete merge  

- **Aggregate**  
  - Type: Aggregate node  
  - Role: Further consolidates merged data, possibly transforming or summarizing before DB storage  
  - Outputs: To decision node for database operations  

---

#### 2.5 Database Operations

**Overview:**  
Determines if a profile record already exists in Supabase, then either updates or creates a new record accordingly.

**Nodes Involved:**  
- If force create?  
- Get a row (Supabase)  
- If record exist  
- Create a row (Supabase)  
- Update Row (Supabase)  
- Respond to Webhook Create  
- Respond to Webhook Update  
- Respond to Webhook Not Found  

**Node Details:**

- **If force create?**  
  - Type: If node  
  - Role: Checks a flag to force creation of a new record even if one exists  
  - Inputs: Aggregated data  
  - Outputs: True branch leads directly to “Create a row”  
  - False branch leads to “Get a row” to check for existing data  

- **Get a row (Supabase)**  
  - Type: Supabase node  
  - Role: Queries Supabase DB for existing profile record by unique key (e.g., LinkedIn URL or user ID)  
  - Retry on failure enabled  
  - Outputs: To “If record exist” node  

- **If record exist**  
  - Type: If node  
  - Role: Checks if the query returned existing record(s)  
  - True branch: updates existing record  
  - False branch: responds with “Not Found” webhook response  

- **Create a row (Supabase)**  
  - Type: Supabase node  
  - Role: Inserts new profile record into the database  
  - Outputs: To “Respond to Webhook Create” node  

- **Update Row (Supabase)**  
  - Type: Supabase node  
  - Role: Updates existing profile record with new extracted data  
  - Outputs: To “Respond to Webhook Update” node  

- **Respond to Webhook Create / Update / Not Found**  
  - Type: Respond to Webhook nodes  
  - Role: Send appropriate HTTP response back to the API caller confirming success or failure  
  - Outputs: End workflow branches  

- **Edge Cases:**  
  - Database connectivity failures  
  - Data uniqueness conflicts  
  - Partial updates or insert errors  
  - Race conditions on record existence checks  

---

#### 2.6 Webhook Responses & Error Termination

**Overview:**  
Handles final responses to webhook calls and terminates workflow on errors.

**Nodes Involved:**  
- Respond to Webhook  
- Respond to Webhook Create  
- Respond to Webhook Update  
- Respond to Webhook Not Found  
- Stop and Error  

**Node Details:**

- **Respond to Webhook**  
  - Generic error response node for status code or scraping failures  

- **Respond to Webhook Create / Update / Not Found**  
  - Specific success or not found responses for database operations  

- **Stop and Error**  
  - Terminates the workflow execution with an error message after responding to webhook  

- **Edge Cases:**  
  - Response sending failures  
  - Unexpected termination without response  

---

### 3. Summary Table

| Node Name                             | Node Type                              | Functional Role                             | Input Node(s)                          | Output Node(s)                          | Sticky Note                      |
|-------------------------------------|--------------------------------------|---------------------------------------------|--------------------------------------|---------------------------------------|---------------------------------|
| Webhook                             | Webhook                              | Receive LinkedIn URL input                   | -                                    | Set the Input Fields                   |                                 |
| Set the Input Fields                | Set                                  | Prepare input fields for scraping           | Webhook                              | Access and extract data from a specific URL |                                 |
| Access and extract data from a specific URL | Bright Data Scraper Node              | Scrape LinkedIn profile data                 | Set the Input Fields                 | Set the Scraped Response              |                                 |
| Set the Scraped Response            | Set                                  | Prepare scraped data for validation          | Access and extract data from a specific URL | If status code <> 200                 |                                 |
| If status code <> 200               | If                                   | Check for HTTP errors in scraping            | Set the Scraped Response             | Respond to Webhook (error), AI extraction nodes |                                 |
| Respond to Webhook                  | Respond to Webhook                   | Send error response on scraping failure     | If status code <> 200                 | Stop and Error                        |                                 |
| Basic Profile Info                  | LangChain Information Extractor      | Extract basic profile info                    | If status code <> 200                 | Merge                                |                                 |
| Emerging Roles                     | LangChain Information Extractor      | Extract emerging roles info                   | If status code <> 200                 | Merge                                |                                 |
| Markdown Content                   | LangChain Information Extractor      | Extract experiences in markdown              | If status code <> 200                 | Merge                                |                                 |
| Summarizer                       | LangChain Information Extractor      | Summarize profile content                     | If status code <> 200                 | Merge                                |                                 |
| Skills Extractor                  | LangChain Information Extractor      | Extract skills and expertise                  | If status code <> 200                 | Merge                                |                                 |
| Google Gemini Chat Model for Basic Profile Info | LangChain LM Chat Google Gemini       | AI model for basic profile extraction        | Basic Profile Info                   | Basic Profile Info                   |                                 |
| Google Gemini Chat Model for Emerging Roles | LangChain LM Chat Google Gemini       | AI model for emerging roles extraction        | Emerging Roles                      | Emerging Roles                      |                                 |
| Google Gemini Chat Model for Summarization | LangChain LM Chat Google Gemini       | AI model for profile summarization            | Summarizer                         | Summarizer                         |                                 |
| Google Gemini Chat Model for Skill Extraction | LangChain LM Chat Google Gemini       | AI model for skill extraction                  | Skills Extractor                   | Skills Extractor                   |                                 |
| Google Gemini Chat Model for Experiences | LangChain LM Chat Google Gemini       | AI model for experience extraction             | Markdown Content                   | Markdown Content                   |                                 |
| Merge                             | Merge                               | Combine all extracted AI data                 | Basic Profile Info, Emerging Roles, Markdown Content, Summarizer, Skills Extractor | Aggregate                          |                                 |
| Aggregate                        | Aggregate                          | Consolidate merged data                        | Merge                              | If force create?                   |                                 |
| If force create?                  | If                                   | Decide to force create new DB record          | Aggregate                          | Get a row, Create a row           |                                 |
| Get a row                        | Supabase                           | Query for existing profile record             | If force create?                   | If record exist                   |                                 |
| If record exist                  | If                                   | Check if profile exists in DB                  | Get a row                         | Update Row, Respond to Webhook Not Found |                                 |
| Create a row                    | Supabase                           | Create new profile record                       | If force create?                   | Respond to Webhook Create          |                                 |
| Update Row                     | Supabase                           | Update existing profile record                  | If record exist                   | Respond to Webhook Update          |                                 |
| Respond to Webhook Create       | Respond to Webhook                   | Send success response after create             | Create a row                     | -                                 |                                 |
| Respond to Webhook Update       | Respond to Webhook                   | Send success response after update             | Update Row                      | -                                 |                                 |
| Respond to Webhook Not Found    | Respond to Webhook                   | Send not found response                         | If record exist                  | -                                 |                                 |
| Stop and Error                 | Stop and Error                     | Terminate workflow on error                      | Respond to Webhook               | -                                 |                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Purpose: Entry point to receive LinkedIn URLs or profile identifiers  
   - Configuration: Set HTTP Method and path as needed  

2. **Create Set Node “Set the Input Fields”**  
   - Type: Set  
   - Purpose: Structure and validate incoming webhook data before scraping  
   - Connect: Webhook → Set Input Fields  

3. **Create Bright Data Node “Access and extract data from a specific URL”**  
   - Type: Bright Data scraper node  
   - Purpose: Scrape LinkedIn profile page content  
   - Credentials: Configure Bright Data credentials  
   - Connection: Set Input Fields → Bright Data node  
   - Enable “Retry on Fail”  

4. **Create Set Node “Set the Scraped Response”**  
   - Type: Set  
   - Purpose: Prepare scraped data and extract HTTP status code for validation  
   - Connection: Bright Data → Set Scraped Response  

5. **Create If Node “If status code <> 200”**  
   - Type: If  
   - Condition: Check if scraped HTTP status is not 200  
   - Connection: Set Scraped Response → If status code check  

6. **Create Respond to Webhook Node for Errors**  
   - Type: Respond to Webhook  
   - Purpose: Return error response if scraping fails  
   - Connection: If status code true branch → Respond to Webhook  

7. **Create LangChain Information Extractors** for each data category:  
   - Basic Profile Info, Emerging Roles, Markdown Content, Summarizer, Skills Extractor  
   - Connect from If status code false branch to each extractor node  

8. **Create Google Gemini Chat Model Nodes** corresponding to each extractor:  
   - Basic Profile Info, Emerging Roles, Summarizer, Skill Extraction, Experiences  
   - Connect each Chat Model node → corresponding Information Extractor node  
   - Configure Google Gemini credentials  
   - Enable “Retry on Fail” where applicable  

9. **Create Merge Node**  
   - Type: Merge  
   - Purpose: Combine outputs from all extractor nodes  
   - Connect all extractor nodes → Merge  

10. **Create Aggregate Node**  
    - Type: Aggregate  
    - Purpose: Consolidate merged data for DB operations  
    - Connect Merge → Aggregate  

11. **Create If Node “If force create?”**  
    - Type: If  
    - Purpose: Check flag to force create new DB record  
    - Connect Aggregate → If force create?  

12. **Create Supabase Node “Get a row”**  
    - Type: Supabase  
    - Purpose: Query existing profile record by unique key  
    - Configure Supabase credentials and table details  
    - Connect If force create? false branch → Get a row  
    - Enable “Retry on Fail”  

13. **Create If Node “If record exist”**  
    - Type: If  
    - Purpose: Check if DB query found an existing record  
    - Connect Get a row → If record exist  

14. **Create Supabase Node “Create a row”**  
    - Type: Supabase  
    - Purpose: Insert new profile record  
    - Configure table and fields  
    - Connect If force create? true branch → Create a row  

15. **Create Supabase Node “Update Row”**  
    - Type: Supabase  
    - Purpose: Update existing profile record  
    - Configure table, primary key, and update fields  
    - Connect If record exist true branch → Update Row  

16. **Create Respond to Webhook Nodes for Create, Update, Not Found**  
    - Type: Respond to Webhook  
    - Purpose: Return appropriate HTTP responses after DB operations  
    - Connect Create a row → Respond to Webhook Create  
    - Connect Update Row → Respond to Webhook Update  
    - Connect If record exist false branch → Respond to Webhook Not Found  

17. **Create Stop and Error Node**  
    - Type: Stop and Error  
    - Purpose: End workflow on errors  
    - Connect Respond to Webhook error responses → Stop and Error  

18. **Validate all connections, credentials, and parameters**  
    - Ensure all API keys (Bright Data, Google Gemini, Supabase) are configured  
    - Set retry options on relevant nodes  
    - Test with sample LinkedIn URLs  

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                                               |
|-----------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------|
| Workflow leverages Bright Data for robust LinkedIn scraping bypassing LinkedIn blocks         | https://brightdata.com                                                        |
| Uses Google Gemini large language model with LangChain wrapper for specialized extraction tasks | Requires Google Gemini API credentials configured in n8n                      |
| Stores and updates profile data in Supabase, an open-source Firebase alternative              | https://supabase.com                                                          |
| Retry on failure enabled on critical nodes to handle transient errors and rate limits         | Configuration detail in node settings                                         |
| The workflow handles webhook inputs and provides clear HTTP responses for success and errors  | Facilitate integration with external applications or automation tools         |

---

**Disclaimer:**  
This document describes a workflow built exclusively with n8n automation, fully compliant with current content policies, containing no illegal or protected data, and operating only on publicly available or authorized information.