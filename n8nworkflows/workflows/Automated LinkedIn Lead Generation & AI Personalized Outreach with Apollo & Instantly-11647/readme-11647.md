Automated LinkedIn Lead Generation & AI Personalized Outreach with Apollo & Instantly

https://n8nworkflows.xyz/workflows/automated-linkedin-lead-generation---ai-personalized-outreach-with-apollo---instantly-11647


# Automated LinkedIn Lead Generation & AI Personalized Outreach with Apollo & Instantly

### 1. Workflow Overview

This n8n workflow automates LinkedIn lead generation and personalized outreach by integrating Apollo.io, Apify, OpenAI, Tavily, Google Sheets, and Instantly.ai. It targets sales and marketing teams seeking to efficiently identify relevant business leads, enrich lead and company data using AI, and execute personalized cold email campaigns.

The workflow is logically structured into three major blocks:

- **1.1 Lead Acquisition & Filtering:**  
  Starts from a web form submission describing lead criteria, generates an Apollo search URL, scrapes lead data via Apify, parses and filters leads.

- **1.2 Lead & Company Data Enrichment:**  
  Processes each lead to extract structured fields, enriches company data using Tavily-powered AI research, and crafts personalized outreach messages.

- **1.3 Data Storage & Campaign Integration:**  
  Stores leads and enriched data in Google Sheets, and uploads personalized leads to Instantly.ai campaigns for automated outreach.

---

### 2. Block-by-Block Analysis

#### 2.1 Lead Acquisition & Filtering

**Overview:**  
This block captures user input (lead criteria), generates a precise Apollo.io search URL, runs a scraping actor on Apify to retrieve lead raw data, parses the data using AI, filters valid leads, and appends them to Google Sheets.

**Nodes Involved:**  
- On form submission  
- OpenAI Chat Model (Apollo URL Generator)  
- Apollo URL Generator  
- Run Apify  
- Structured Output Parser  
- Parse Lead Data  
- If1  
- Filter  
- Add to Google Sheet  
- Limit1

**Node Details:**

- **On form submission**  
  - *Type:* Form Trigger  
  - *Role:* Entry point; captures lead criteria (Job Title, Company Size, Keywords, Location, Instantly Campaign ID) from user input form.  
  - *Config:* Custom form with dropdown and text inputs, webhook enabled.  
  - *Inputs:* External HTTP form submission  
  - *Outputs:* To Apollo URL Generator  
  - *Edge Cases:* Missing or invalid form data may produce malformed URLs or empty queries.

- **OpenAI Chat Model (Apollo URL Generator)**  
  - *Type:* Langchain OpenAI Chat Model (GPT-4.1)  
  - *Role:* Generates Apollo search URL from form data  
  - *Config:* Prompt instructs to parse multi-valued fields (comma-separated) into Apollo URL parameters, ensure URL encoding, and produce exact format URLs.  
  - *Inputs:* Form submission JSON  
  - *Outputs:* Text URL to Apollo URL Generator  
  - *Edge Cases:* Incorrect prompt input formatting may cause invalid URL output.

- **Apollo URL Generator**  
  - *Type:* Chain LLM node  
  - *Role:* Executes the OpenAI prompt to construct the Apollo.io search URL  
  - *Inputs:* Form data from On form submission  
  - *Outputs:* URL text to Run Apify  
  - *Edge Cases:* Same as above.

- **Run Apify**  
  - *Type:* HTTP Request  
  - *Role:* Calls Apify actor to scrape Apollo search results using the generated URL  
  - *Config:* POST request with JSON body containing URL and scraping flags; requires Apify API token in header.  
  - *Inputs:* Apollo URL  
  - *Outputs:* Raw scraped lead data JSON array to Parse Lead Data  
  - *Edge Cases:* API key invalid/expired, rate limits, network errors, malformed URL causing empty results.

- **Structured Output Parser**  
  - *Type:* Langchain Output Parser Structured  
  - *Role:* Parses AI output to structured JSON (used downstream)  
  - *Inputs:* Output of OpenAI nodes (Parse Lead Data)  
  - *Outputs:* Parsed JSON to Parse Lead Data node  
  - *Edge Cases:* Unexpected output format may cause parsing failures.

- **Parse Lead Data**  
  - *Type:* Chain LLM node  
  - *Role:* Extracts key lead fields (name, email, title, LinkedIn, company info) from raw Apify JSON, synthesizes a sales summary, formats as JSON.  
  - *Config:* Prompt includes instructions to handle missing fields as null, produce concise summary.  
  - *Inputs:* Raw lead JSON from Run Apify  
  - *Outputs:* Parsed lead JSON to If1  
  - *Edge Cases:* Partial or malformed lead data; output errors handled by "continueRegularOutput" mode.

- **If1**  
  - *Type:* If node  
  - *Role:* Checks if lead JSON contains an error field (filters out failed parses or errors).  
  - *Inputs:* Parsed lead JSON  
  - *Outputs:* Valid leads to Filter node; invalid leads discarded.  
  - *Edge Cases:* Missing error field or unexpected error format.

- **Filter**  
  - *Type:* Filter node  
  - *Role:* Ensures leads have non-null, non-empty email addresses before continuing.  
  - *Inputs:* Valid lead JSON  
  - *Outputs:* Leads with email to Add to Google Sheet  
  - *Edge Cases:* Leads without email will stop here.

- **Add to Google Sheet**  
  - *Type:* Google Sheets node  
  - *Role:* Appends or updates lead records in the Google Sheet with structured lead data fields.  
  - *Config:* Matches rows by Full Name; writes multiple lead and company-related columns.  
  - *Inputs:* Filtered leads  
  - *Outputs:* Triggers Limit1 node  
  - *Edge Cases:* Google Sheets API quota or authentication errors.

- **Limit1**  
  - *Type:* Limit node  
  - *Role:* Limits the number of leads processed downstream to 10 per workflow run (batch size control).  
  - *Inputs:* New/updated lead rows  
  - *Outputs:* To Loop Over Items node  
  - *Edge Cases:* None significant.

---

#### 2.2 Lead & Company Data Enrichment

**Overview:**  
Iterates over each lead to enrich company data using Tavily AI searches, parses company research results, generates personalized outreach messages with OpenAI, and prepares enriched data for storage and outreach.

**Nodes Involved:**  
- Loop Over Items  
- Company Research (Langchain Agent)  
- Tavily (Tavily API node)  
- Structured Output Parser1  
- OpenAI Chat Model2  
- OpenAI Chat Model3  
- Generate Outreach Message

**Node Details:**

- **Loop Over Items**  
  - *Type:* Split In Batches  
  - *Role:* Processes each lead individually in a batch.  
  - *Inputs:* Limited list of leads from Limit1  
  - *Outputs:* To Company Research and downstream nodes  
  - *Edge Cases:* Large batch sizes may cause timeouts.

- **Company Research**  
  - *Type:* Langchain Agent (with Tavily tool)  
  - *Role:* Uses Tavily search to extract company overview, recent news, key offerings, third-party sentiment, and synthesizes a summary.  
  - *Config:* System prompt instructs to prioritize official website info, then third-party sentiment, and recent news within last 12-18 months. Returns structured JSON.  
  - *Inputs:* Lead's company name, website, LinkedIn  
  - *Outputs:* Enriched company data JSON to Generate Outreach Message  
  - *Edge Cases:* No data found, Tavily API errors, partial results.

- **Tavily**  
  - *Type:* Tavily API node  
  - *Role:* Executes Tavily search queries triggered by Company Research agent.  
  - *Inputs:* Queries from Company Research  
  - *Outputs:* Search results back to Company Research  
  - *Edge Cases:* API key limits, network errors.

- **Structured Output Parser1**  
  - *Type:* Langchain Output Parser Structured  
  - *Role:* Parses the company research agent's JSON output into structured data.  
  - *Inputs:* AI response from Company Research  
  - *Outputs:* Parsed company data to Generate Outreach Message  
  - *Edge Cases:* Parsing errors if unexpected format.

- **OpenAI Chat Model2** and **OpenAI Chat Model3**  
  - *Type:* Langchain OpenAI Chat Models (GPT-4.1-mini)  
  - *Role:* Assist in company research parsing and outreach message generation.  
  - *Inputs:* Company Research data and lead data  
  - *Outputs:* To Structured Output Parser1 and Generate Outreach Message respectively  
  - *Edge Cases:* API rate limits or response failures.

- **Generate Outreach Message**  
  - *Type:* Chain LLM node  
  - *Role:* Drafts a concise, personalized cold outreach email body using lead personal background and enriched company background.  
  - *Config:* Detailed prompt to identify strongest hook, link AI consulting benefits, maintain professional tone, and strict 100-150 word limit.  
  - *Inputs:* Lead info, company summary, personal summary  
  - *Outputs:* Personalized outreach text to Add Company Info to Google Sheet  
  - *Edge Cases:* Lack of strong hooks, incomplete data; "continueRegularOutput" mode used.

---

#### 2.3 Data Storage & Campaign Integration

**Overview:**  
Stores enriched lead and company data in Google Sheets and adds leads with custom messages to Instantly.ai campaigns for automated outreach.

**Nodes Involved:**  
- Add Company Info to Google Sheet  
- Add Lead to Instantly AI

**Node Details:**

- **Add Company Info to Google Sheet**  
  - *Type:* Google Sheets node  
  - *Role:* Updates Google Sheet rows with enriched company info and outreach message, matching on Email.  
  - *Config:* Writes key offerings, company summary, background, recent news, sentiment, outreach text.  
  - *Inputs:* Generated outreach messages and company data  
  - *Outputs:* Triggers Add Lead to Instantly AI node  
  - *Edge Cases:* Google Sheets API limits or credential issues.

- **Add Lead to Instantly AI**  
  - *Type:* HTTP Request  
  - *Role:* Sends lead info and custom outreach message to Instantly.ai API to add lead to specified campaign.  
  - *Config:* POST JSON with campaign ID (from form), lead email, name, company, and custom email body; requires Instantly API token.  
  - *Inputs:* Lead enriched data and outreach message  
  - *Outputs:* None (end node)  
  - *Edge Cases:* API key invalid, request failures, invalid campaign ID.

---

### 3. Summary Table

| Node Name               | Node Type                          | Functional Role                          | Input Node(s)                | Output Node(s)                  | Sticky Note                                                                                                                                                                                                                                                 |
|-------------------------|----------------------------------|----------------------------------------|-----------------------------|-------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Sticky Note             | Sticky Note                      | Overview and documentation note        |                             |                               | LinkedIn Scraper + Cold Outreach overview, APIs needed, actor info, costs.                                                                                                                                                                                  |
| Sticky Note1            | Sticky Note                      | Section label                          |                             |                               | LinkedIn Scraper via Apollo                                                                                                                                                                                                                                |
| Sticky Note2            | Sticky Note                      | Section label                          |                             |                               | Company Research + Outreach                                                                                                                                                                                                                                |
| On form submission      | Form Trigger                    | Entry point: capture lead criteria     |                             | Apollo URL Generator           |                                                                                                                                                                                                                                                            |
| OpenAI Chat Model       | Langchain OpenAI Chat Model     | Generate Apollo search URL              | On form submission          | Apollo URL Generator           |                                                                                                                                                                                                                                                            |
| Apollo URL Generator    | Chain LLM                      | Construct Apollo search URL             | On form submission          | Run Apify                     |                                                                                                                                                                                                                                                            |
| Run Apify               | HTTP Request                   | Scrape Apollo search results            | Apollo URL Generator        | Parse Lead Data               |                                                                                                                                                                                                                                                            |
| Structured Output Parser| Langchain Output Parser Structured | Parse AI output to structured JSON     | OpenAI Chat Model1          | Parse Lead Data               |                                                                                                                                                                                                                                                            |
| Parse Lead Data         | Chain LLM                      | Extract lead fields and summary         | Run Apify, Structured Output Parser | If1                      |                                                                                                                                                                                                                                                            |
| If1                     | If                             | Filter out error leads                   | Parse Lead Data             | Filter                       |                                                                                                                                                                                                                                                            |
| Filter                  | Filter                         | Filter leads without email               | If1                        | Add to Google Sheet           |                                                                                                                                                                                                                                                            |
| Add to Google Sheet     | Google Sheets                  | Store lead data                         | Filter                      | Limit1                       |                                                                                                                                                                                                                                                            |
| Limit1                  | Limit                         | Limit number of leads processed          | Add to Google Sheet         | Loop Over Items               |                                                                                                                                                                                                                                                            |
| Loop Over Items         | Split In Batches               | Batch process leads                      | Limit1                      | Company Research, Company Research |                                                                                                                                                                                                                                                            |
| Tavily                  | Tavily Tool                   | Execute Tavily company info searches     | Company Research (AI Tool)  | Company Research              |                                                                                                                                                                                                                                                            |
| Company Research        | Langchain Agent               | Enrich company data with Tavily results | Loop Over Items             | Generate Outreach Message     |                                                                                                                                                                                                                                                            |
| Structured Output Parser1| Langchain Output Parser Structured | Parse company research JSON             | Company Research            | Generate Outreach Message     |                                                                                                                                                                                                                                                            |
| OpenAI Chat Model2      | Langchain OpenAI Chat Model     | Assist company research parsing          | Tavily                      | Company Research              |                                                                                                                                                                                                                                                            |
| OpenAI Chat Model3      | Langchain OpenAI Chat Model     | Assist outreach message drafting          | Company Research            | Generate Outreach Message     |                                                                                                                                                                                                                                                            |
| Generate Outreach Message| Chain LLM                    | Compose personalized outreach email     | Company Research            | Add Company Info to Google Sheet |                                                                                                                                                                                                                                                            |
| Add Company Info to Google Sheet | Google Sheets          | Store enriched company and outreach data| Generate Outreach Message   | Add Lead to Instantly AI      |                                                                                                                                                                                                                                                            |
| Add Lead to Instantly AI| HTTP Request                   | Upload lead and message to Instantly.ai | Add Company Info to Google Sheet |                             |                                                                                                                                                                                                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger Node**  
   - Type: `Form Trigger`  
   - Configure form fields:  
     - Job Title (text)  
     - Company Size (dropdown with specified ranges)  
     - Keywords (text)  
     - Location (text)  
     - Instantly Campaign ID (text)  
   - Enable webhook, set form title and description.

2. **Create OpenAI Chat Model Node (Apollo URL Generator)**  
   - Type: `Langchain OpenAI Chat Model` (GPT-4.1)  
   - Prompt: Generate Apollo.io search URL from form data, handle multiple comma-separated values, URL-encode parameters.  
   - Connect input from Form Trigger node.

3. **Create Apollo URL Generator Node (Chain LLM)**  
   - Type: `Chain LLM`  
   - Text input: Template that formats form data into prompt for Apollo URL generation.  
   - Connect input from Form Trigger node and OpenAI Chat Model.

4. **Create HTTP Request Node (Run Apify)**  
   - Type: `HTTP Request` POST  
   - URL: `https://api.apify.com/v2/acts/jljBwyyQakqrL1wae/run-sync-get-dataset-items`  
   - Headers: Accept: application/json, Authorization: Bearer `YOUR_APIFY_TOKEN`  
   - Body (JSON): Include `getPersonalEmails`, `getWorkEmails`, `totalRecords:500`, and the generated Apollo URL from previous node.  
   - Connect input from Apollo URL Generator.

5. **Create Structured Output Parser Node**  
   - Type: `Langchain Output Parser Structured`  
   - Provide JSON schema example for lead data extraction.  
   - Connect input from OpenAI Chat Model1 (used later).

6. **Create Parse Lead Data Node (Chain LLM)**  
   - Type: `Chain LLM`  
   - Prompt: Extract key lead fields from Apify JSON, synthesize professional summary, format as JSON.  
   - Connect input from Run Apify and Structured Output Parser.

7. **Create If Node (If1)**  
   - Type: `If`  
   - Condition: Check if `error` field does not exist in parsed lead JSON.  
   - Connect input from Parse Lead Data.

8. **Create Filter Node**  
   - Type: `Filter`  
   - Conditions: Lead’s `output.email` is not null or empty.  
   - Connect input from If1 (valid leads).

9. **Create Google Sheets Node (Add to Google Sheet)**  
   - Type: `Google Sheets`  
   - Operation: Append or update based on "Full Name" column.  
   - Map lead fields: Email, Title, Full Name, Last Name, First Name, Company Name, Company Website, Company LinkedIn, Personal Summary, Personal LinkedIn.  
   - Specify Google Sheet ID and sheet name.  
   - Connect input from Filter.

10. **Create Limit Node (Limit1)**  
    - Type: `Limit`  
    - Max items: 10  
    - Connect input from Add to Google Sheet.

11. **Create Split In Batches Node (Loop Over Items)**  
    - Type: `Split In Batches`  
    - Connect input from Limit node.

12. **Create Tavily Node**  
    - Type: `Tavily Tool`  
    - Requires Tavily API credentials.  
    - Connect input from Company Research node’s AI Tool input.

13. **Create Company Research Node (Langchain Agent)**  
    - Type: `Langchain Agent`  
    - System prompt instructs to use Tavily searches for company overview, news, offerings, sentiment.  
    - Connect input from Loop Over Items (main), and Tavily node (ai_tool).

14. **Create Structured Output Parser Node1**  
    - Type: `Langchain Output Parser Structured`  
    - JSON schema example for company research output.  
    - Connect input from Company Research.

15. **Create OpenAI Chat Model Node2**  
    - Type: `Langchain OpenAI Chat Model` (GPT-4.1-mini)  
    - Connect input from Tavily node (to assist Company Research).

16. **Create OpenAI Chat Model Node3**  
    - Type: `Langchain OpenAI Chat Model` (GPT-4.1-mini)  
    - Connect input from Company Research (for Generate Outreach Message).

17. **Create Generate Outreach Message Node (Chain LLM)**  
    - Type: `Chain LLM`  
    - Prompt: Draft personalized cold outreach message based on lead’s personal background and enriched company background; strict 100-150 word limit.  
    - Connect input from Company Research and Loop Over Items.

18. **Create Google Sheets Node (Add Company Info to Google Sheet)**  
    - Type: `Google Sheets`  
    - Operation: Append or update based on Email column.  
    - Map enriched company info fields and outreach message.  
    - Connect input from Generate Outreach Message.

19. **Create HTTP Request Node (Add Lead to Instantly AI)**  
    - Type: `HTTP Request` POST  
    - URL: `https://api.instantly.ai/api/v2/leads`  
    - Headers: Authorization Bearer `YOUR_INSTANTLY_TOKEN`, Content-Type application/json  
    - Body (JSON): Include campaign ID from form submission, lead email, first name, last name, company name, and custom outreach message as a variable.  
    - Connect input from Add Company Info to Google Sheet.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                  | Context or Link                                                                                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Workflow integrates Apollo.io lead search, Apify scraping actor (https://console.apify.com/actors/jljBwyyQakqrL1wae/input), Tavily company research, OpenAI GPT-4.1 models, Google Sheets, and Instantly.ai for outreach.                                                                                                                                           | Apify actor link and API info                                                                                   |
| Costs overview: Instantly.ai $38/month, Tavily free first 100 leads then $0.08/lead, Apify $39/month, Apollo $1.20 per 1000 leads (charged from Apify credit), GPT API < $0.10 per 100 leads.                                                                                                                                                                     | Cost estimation note                                                                                            |
| Requires API keys for Apify, OpenAI, Instantly.ai, and Tavily. Ensure proper credential setup in n8n credentials manager.                                                                                                                                                                                                                                   | API keys and credential setup reminder                                                                          |
| Form descriptions and input placeholders guide users to specify lead search parameters clearly (title, company size, keywords, location, Instantly campaign ID).                                                                                                                                                                                               | Form UX guidance                                                                                                |
| Prompts are carefully crafted to parse complex and nested JSON, handle missing fields gracefully, and produce structured JSON outputs for downstream use.                                                                                                                                                                                                    | AI prompt design best practices                                                                                 |
| Use of "continueRegularOutput" on error-prone nodes ensures workflow robustness by allowing partial data processing even on some errors.                                                                                                                                                                                                                      | Workflow error handling strategy                                                                                 |

---

**Disclaimer:**  
The text provided is sourced exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected content. All data handled is legal and publicly accessible.