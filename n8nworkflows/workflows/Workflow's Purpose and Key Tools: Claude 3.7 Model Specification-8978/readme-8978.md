Workflow's Purpose and Key Tools: Claude 3.7 Model Specification

https://n8nworkflows.xyz/workflows/workflow-s-purpose-and-key-tools--claude-3-7-model-specification-8978


# Workflow's Purpose and Key Tools: Claude 3.7 Model Specification

### 1. Workflow Overview

This n8n workflow automates the process of generating and sending personalized cold emails to leads using LinkedIn profile enrichment and AI-driven email composition. It targets sales or marketing teams aiming to scale outreach with tailored content that reflects deep understanding of prospects and their companies.

The workflow logically consists of four main blocks:

- **1.1 Lead Intake & Search**: Reads leads from Google Sheets and performs targeted web searches on LinkedIn profiles using Apify’s RAG web browser actor.
- **1.2 LinkedIn Profile Validation & Enrichment**: Filters and validates LinkedIn URLs from search results, fetches detailed LinkedIn profile data for persons, and enriches company data either from existing URLs or via additional LinkedIn company search.
- **1.3 Company Data Retrieval**: Based on the existence of a current company URL in the profile, either directly fetches company info from LinkedIn or performs a Google search to find company URLs, then fetches detailed company data.
- **1.4 Personalized Email Generation & Sending**: Uses a Claude 3.7 AI model via OpenRouter to generate highly personalized emails based on enriched person and company data, parses the AI output for subject and body, and sends the email via Gmail.

Each block is connected sequentially to ensure data enrichment before email generation, with error handling to avoid workflow interruption on partial failures.

---

### 2. Block-by-Block Analysis

#### 2.1 Lead Intake & Search

- **Overview:**  
  This block imports lead data from a Google Sheet and performs a batch-wise LinkedIn web search to find potential LinkedIn profiles for each lead based on their name and company.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Get Leads (Google Sheets)  
  - Loop Over Items (Split In Batches)  
  - Google Search for Person LinkedIn (HTTP Request - Apify RAG Web Browser)

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - *Type:* Manual Trigger  
    - *Role:* Entry point to start the workflow manually.  
    - *Config:* No parameters; triggers start of the flow.  
    - *Connections:* Outputs to Get Leads.  
    - *Failures:* None expected.

  - **Get Leads**  
    - *Type:* Google Sheets  
    - *Role:* Reads unprocessed leads (with fields such as First Name, Last Name, Company, Email) from a specified Google Sheet.  
    - *Config:* Uses OAuth2 credentials for Google Sheets; reads from a specific document and sheet gid.  
    - *Connections:* Outputs to Loop Over Items.  
    - *Failures:* Google auth errors, sheet not found.  

  - **Loop Over Items**  
    - *Type:* Split in Batches  
    - *Role:* Processes leads in batches to handle rate limits and stability.  
    - *Config:* Default batch settings (no specific batch size indicated).  
    - *Connections:* Main output to Google Search for Person LinkedIn.  
    - *Failures:* None expected.  

  - **Google Search for Person LinkedIn**  
    - *Type:* HTTP Request  
    - *Role:* Queries Apify RAG web browser to search LinkedIn person profiles with a query constructed as `{First Name} {Last Name} {Company} site:linkedin.com`.  
    - *Config:* POST request with query parameter, maxResults = 10, 4096MB memory, 180s timeout; uses HTTP Header Auth with a specific credential.  
    - *Key Expressions:* Query dynamically built from lead JSON data.  
    - *Connections:* Outputs to Only People Links filter node.  
    - *Failures:* API timeout, auth failure; set to continue on error to avoid stopping workflow.

---

#### 2.2 LinkedIn Profile Validation & Enrichment

- **Overview:**  
  Filters the search results to keep only valid personal LinkedIn profile URLs, aggregates these results, fetches detailed LinkedIn profile data for the first matched URL, and routes the flow to company data enrichment.

- **Nodes Involved:**  
  - Only People Links (Filter)  
  - Aggregate (Aggregate)  
  - Get LinkedIn Person (HTTP Request - Apify Profile Detail)  
  - Current Company Exists? (If)

- **Node Details:**

  - **Only People Links**  
    - *Type:* Filter  
    - *Role:* Keeps only URLs matching the regex pattern for personal LinkedIn profiles (`linkedin.com/in/...`).  
    - *Config:* Regex condition on `searchResult.url`.  
    - *Connections:* Outputs filtered results to Aggregate.  
    - *Failures:* Expression errors if `searchResult.url` missing; no hard stop.  

  - **Aggregate**  
    - *Type:* Aggregate  
    - *Role:* Aggregates all filtered items into a single JSON array under `results`.  
    - *Config:* Specifies inclusion of `searchResult` field only.  
    - *Connections:* Output feeds to Get LinkedIn Person.  
    - *Failures:* None expected.

  - **Get LinkedIn Person**  
    - *Type:* HTTP Request  
    - *Role:* Runs Apify actor `linkedin-profile-detail` synchronously to fetch detailed LinkedIn person profile data based on the first URL from aggregated results.  
    - *Config:* POST with username parameter extracted from first `searchResult.url`; uses same HTTP Header Auth; 4096MB memory, 180s timeout; on error continues run.  
    - *Connections:* Output to Current Company Exists? node.  
    - *Failures:* API errors, data missing; handled gracefully.  

  - **Current Company Exists?**  
    - *Type:* If  
    - *Role:* Checks if the fetched LinkedIn profile contains an existing current company URL.  
    - *Config:* Condition checks existence of `basic_info.current_company_url`.  
    - *Connections:*  
      - If true: to Set Company URL node.  
      - If false: to Google Search for Company LinkedIn node.  
    - *Failures:* None expected.

---

#### 2.3 Company Data Retrieval

- **Overview:**  
  Determines the company URL either from the LinkedIn profile or a fresh Google search, filters company URLs, aggregates results, fetches detailed company profile data from LinkedIn, and prepares data for email generation.

- **Nodes Involved:**  
  - Set Company URL (Set)  
  - Google Search for Company LinkedIn (HTTP Request - Apify RAG Web Browser)  
  - Only Company Links (Filter)  
  - Aggregate1 (Aggregate)  
  - Set Company URL1 (Set)  
  - Get LinkedIn Company (HTTP Request - Apify Company Detail)

- **Node Details:**

  - **Set Company URL**  
    - *Type:* Set  
    - *Role:* Sets `company_url` to the `current_company_url` from the LinkedIn person profile JSON.  
    - *Config:* Raw JSON mode output with expression.  
    - *Connections:* Outputs to Get LinkedIn Company.  
    - *Failures:* Expression issues if input missing.

  - **Google Search for Company LinkedIn**  
    - *Type:* HTTP Request  
    - *Role:* Uses Apify RAG web browser actor to perform a Google search with query `{Company} site:linkedin.com` to find company LinkedIn pages.  
    - *Config:* POST request with query parameter, 4096MB memory, 180s timeout; same HTTP Header Auth; continues on error.  
    - *Connections:* Output to Only Company Links filter.  
    - *Failures:* API or auth errors; continues gracefully.  

  - **Only Company Links**  
    - *Type:* Filter  
    - *Role:* Filters URLs matching the regex pattern for LinkedIn company pages (`linkedin.com/company/...`).  
    - *Config:* Regex on `searchResult.url`.  
    - *Connections:* Output to Aggregate1.  
    - *Failures:* Possible expression errors if input malformed.  

  - **Aggregate1**  
    - *Type:* Aggregate  
    - *Role:* Aggregates filtered company links into `results`.  
    - *Config:* Includes only `searchResult` field.  
    - *Connections:* Output to Set Company URL1.  
    - *Failures:* None expected.

  - **Set Company URL1**  
    - *Type:* Set  
    - *Role:* Sets `company_url` to the first company URL from aggregated results.  
    - *Config:* Raw JSON mode with expression targeting first `searchResult.url`.  
    - *Connections:* Output to Get LinkedIn Company.  
    - *Failures:* Expression failure if `results` empty.

  - **Get LinkedIn Company**  
    - *Type:* HTTP Request  
    - *Role:* Runs Apify actor `linkedin-company-detail` synchronously to fetch detailed company profile data based on `company_url`.  
    - *Config:* POST request with `identifier` containing company URL array; 4096MB memory, 180s timeout; HTTP Header Auth; continues on error.  
    - *Connections:* Outputs to Set Data node.  
    - *Failures:* API errors, timeout, missing `company_url`.

---

#### 2.4 Personalized Email Generation & Sending

- **Overview:**  
  Sets key company and sender data, sends enriched person and company JSON data to the Claude 3.7 model via OpenRouter for personalized email generation, parses the structured JSON output for subject and body, and sends the email through Gmail.

- **Nodes Involved:**  
  - Set Data (Set)  
  - OpenRouter Chat Model (AI Language Model - Claude 3.7)  
  - Structured Output Parser (AI Output Parser)  
  - Generate Personalized Email (LangChain Chain LLM)  
  - Gmail (Email Send)  

- **Node Details:**

  - **Set Data**  
    - *Type:* Set  
    - *Role:* Defines static data such as company name, sender name, email address, and detailed company info text for email context.  
    - *Config:* Assigns multiple string fields with company marketing content and sender info.  
    - *Connections:* Output to Generate Personalized Email.  
    - *Failures:* None expected.

  - **Generate Personalized Email**  
    - *Type:* LangChain Chain LLM  
    - *Role:* Composes a personalized cold email using rich JSON input of person and company data, applying specific email writing guidelines (concise, humorous, personalized).  
    - *Config:*  
      - Input includes JSON stringified LinkedIn person and company data.  
      - Prompt instructs AI to output structured JSON with `subject` and `body`.  
      - Batch processing enabled but no batch size specified.  
    - *Connections:* Output to Gmail node.  
    - *Failures:* AI model timeouts or malformed output; mitigated by output parser.  

  - **OpenRouter Chat Model**  
    - *Type:* AI Language Model (Claude 3.7 via OpenRouter)  
    - *Role:* Provides AI model backend for email generation chain.  
    - *Config:* Model set to `anthropic/claude-3.7-sonnet`; uses OpenRouter API credentials.  
    - *Connections:* AI languageModel input connected from Generate Personalized Email node.  
    - *Failures:* API rate limits, auth failures.

  - **Structured Output Parser**  
    - *Type:* AI Output Parser (Structured JSON)  
    - *Role:* Parses AI response to extract `subject` and `body` fields reliably.  
    - *Config:* Uses defined JSON schema example matching expected AI output format.  
    - *Connections:* Output feeds back into Generate Personalized Email node for final output consolidation.  
    - *Failures:* Parsing errors if AI output deviates from schema.  

  - **Gmail**  
    - *Type:* Gmail  
    - *Role:* Sends the composed email to the lead's email address.  
    - *Config:*  
      - `sendTo` is dynamically set to lead's email from Google Sheets.  
      - Subject and message body set from AI parsed output.  
      - Reply-to and sender name set from static data.  
      - Uses OAuth2 Gmail credentials.  
    - *Connections:* Outputs back to Loop Over Items for next lead.  
    - *Failures:* Gmail API auth issues, email delivery failures.

---

### 3. Summary Table

| Node Name                    | Node Type                                | Functional Role                            | Input Node(s)                   | Output Node(s)               | Sticky Note                                                                                               |
|------------------------------|-----------------------------------------|--------------------------------------------|--------------------------------|-----------------------------|----------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                          | Entry point to trigger workflow             | None                           | Get Leads                   |                                                                                                          |
| Get Leads                    | Google Sheets                           | Reads lead data from Google Sheets          | When clicking ‘Execute workflow’ | Loop Over Items              | ## 🧾 STEP 1: Lead Intake & Search<br>Input: Leads from Google Sheets<br>Search (Person): Apify RAG web browser queries<br>Loop processes leads in batches for stability |
| Loop Over Items              | Split In Batches                        | Processes leads in batches                    | Get Leads                     | Google Search for Person LinkedIn, Gmail (for looping)  |                                                                                                          |
| Google Search for Person LinkedIn | HTTP Request (Apify RAG Web Browser)  | Searches LinkedIn for person profiles        | Loop Over Items               | Only People Links            |                                                                                                          |
| Only People Links            | Filter                                 | Filters URLs to keep only personal LinkedIn profiles | Google Search for Person LinkedIn | Aggregate                  | ## 🔗 STEP 2: Validate LinkedIn URLs (Person)<br>Filter regex `linkedin.com/in/...`<br>Aggregate top search results<br>Profile fetch on first match<br>Continue on errors |
| Aggregate                   | Aggregate                              | Aggregates filtered person profiles          | Only People Links             | Get LinkedIn Person          |                                                                                                          |
| Get LinkedIn Person          | HTTP Request (Apify linkedin-profile-detail) | Fetches detailed LinkedIn person profile data | Aggregate                    | Current Company Exists?      |                                                                                                          |
| Current Company Exists?      | If                                     | Checks existence of current company URL      | Get LinkedIn Person           | Set Company URL (true), Google Search for Company LinkedIn (false) |                                                                                                          |
| Set Company URL              | Set                                    | Sets company URL from LinkedIn profile       | Current Company Exists? (true) | Get LinkedIn Company         | ## 🏢 STEP 3: Company Enrichment<br>If current company URL exists, use it<br>Else google search company LinkedIn<br>Aggregate first result<br>Profile fetch company detail |
| Google Search for Company LinkedIn | HTTP Request (Apify RAG Web Browser)  | Searches LinkedIn company pages via Google   | Current Company Exists? (false) | Only Company Links           |                                                                                                          |
| Only Company Links           | Filter                                 | Filters URLs to keep only LinkedIn company pages | Google Search for Company LinkedIn | Aggregate1                 |                                                                                                          |
| Aggregate1                  | Aggregate                              | Aggregates filtered company profiles          | Only Company Links            | Set Company URL1             |                                                                                                          |
| Set Company URL1             | Set                                    | Sets company URL from aggregated search       | Aggregate1                   | Get LinkedIn Company         |                                                                                                          |
| Get LinkedIn Company         | HTTP Request (Apify linkedin-company-detail) | Fetches detailed LinkedIn company profile     | Set Company URL, Set Company URL1 | Set Data                   |                                                                                                          |
| Set Data                    | Set                                    | Sets static company and sender info for email | Get LinkedIn Company          | Generate Personalized Email  | ## ✉️  STEP 4: Compose & Send<br>Set data: company name, sender, info<br>LLM via OpenRouter Claude 3.7<br>Structured output parser<br>Send email via Gmail |
| Generate Personalized Email  | LangChain Chain LLM                    | Generates personalized email text and subject | Set Data                     | Gmail                       |                                                                                                          |
| OpenRouter Chat Model        | AI Language Model                      | Provides Claude 3.7 model for email generation | Generate Personalized Email   | Generate Personalized Email  |                                                                                                          |
| Structured Output Parser     | AI Output Parser (Structured JSON)    | Parses AI output to extract email subject/body | Generate Personalized Email   | Generate Personalized Email  |                                                                                                          |
| Gmail                       | Gmail                                 | Sends the generated email to the lead         | Generate Personalized Email   | Loop Over Items             |                                                                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Execute workflow’`  
   - Purpose: Start the workflow manually.

2. **Create Google Sheets Node**  
   - Name: `Get Leads`  
   - Configuration:  
     - Set Google Sheets document ID and sheet (gid=0) for leads.  
     - Use OAuth2 credentials for Google Sheets.  
     - Reads lead rows including First Name, Last Name, Company, Email.

3. **Connect `When clicking ‘Execute workflow’` → `Get Leads`**

4. **Add Split In Batches Node**  
   - Name: `Loop Over Items`  
   - Purpose: Process leads one batch at a time (default batch size).  
   - Connect `Get Leads` → `Loop Over Items`.

5. **Add HTTP Request Node for Person LinkedIn Search**  
   - Name: `Google Search for Person LinkedIn`  
   - Method: POST  
   - URL: Apify RAG web browser actor endpoint for dataset items.  
   - Body Parameters:  
     - `query`: Expression using lead’s first name, last name, company + `site:linkedin.com`  
     - `maxResults`: 10  
   - Query Parameters: Memory 4096, Timeout 180  
   - Authentication: HTTP Header Auth with provided credential.  
   - On Error: Continue Regular Output.  
   - Connect `Loop Over Items` → `Google Search for Person LinkedIn`.

6. **Add Filter Node for Person LinkedIn URLs**  
   - Name: `Only People Links`  
   - Condition: Regex filter on `searchResult.url` to keep only URLs matching `^https:\/\/www\.linkedin\.com\/in\/[a-zA-Z0-9\-_%]+\/?$`  
   - Connect `Google Search for Person LinkedIn` → `Only People Links`.

7. **Add Aggregate Node**  
   - Name: `Aggregate`  
   - Aggregate all filtered items’ `searchResult` fields into a `results` array.  
   - Connect `Only People Links` → `Aggregate`.

8. **Add HTTP Request Node for LinkedIn Person Details**  
   - Name: `Get LinkedIn Person`  
   - Use Apify actor `linkedin-profile-detail` endpoint.  
   - POST with body parameter `username` set to the first element’s `searchResult.url` from `results`.  
   - Query parameters: Memory 4096, Timeout 180.  
   - HTTP Header Auth credentials.  
   - On Error: Continue Regular Output.  
   - Connect `Aggregate` → `Get LinkedIn Person`.

9. **Add If Node to Check Current Company URL**  
   - Name: `Current Company Exists?`  
   - Condition: Check if `basic_info.current_company_url` exists and is not empty in LinkedIn person JSON.  
   - Connect `Get LinkedIn Person` → `Current Company Exists?`.

10. **Add Set Node to Set Company URL from Profile**  
    - Name: `Set Company URL`  
    - Set JSON output field `company_url` to `basic_info.current_company_url`.  
    - Connect `Current Company Exists?` True output → `Set Company URL`.

11. **Add HTTP Request Node for Google Search Company LinkedIn**  
    - Name: `Google Search for Company LinkedIn`  
    - POST to Apify RAG web browser endpoint.  
    - Body parameter `query` set to `{Company} site:linkedin.com` (from lead/company data).  
    - Query parameters: Memory 4096, Timeout 180.  
    - HTTP Header Auth credentials.  
    - On Error: Continue Regular Output.  
    - Connect `Current Company Exists?` False output → `Google Search for Company LinkedIn`.

12. **Add Filter Node for Company LinkedIn URLs**  
    - Name: `Only Company Links`  
    - Regex filter for URLs matching `^https:\/\/www\.linkedin\.com\/company\/[a-zA-Z0-9\-_%]+\/?$`.  
    - Connect outputs from both `Google Search for Company LinkedIn` and later re-connect as needed.

13. **Add Aggregate Node for Company URLs**  
    - Name: `Aggregate1`  
    - Aggregate filtered company links’ `searchResult` fields into `results`.  
    - Connect `Only Company Links` → `Aggregate1`.

14. **Add Set Node to Set Company URL from Aggregated Results**  
    - Name: `Set Company URL1`  
    - Set JSON `company_url` to first `searchResult.url` from `results`.  
    - Connect `Aggregate1` → `Set Company URL1`.

15. **Add HTTP Request Node for LinkedIn Company Details**  
    - Name: `Get LinkedIn Company`  
    - Use Apify `linkedin-company-detail` actor endpoint.  
    - POST with body parameter `identifier` as array containing `company_url`.  
    - Query params: Memory 4096, Timeout 180.  
    - HTTP Header Auth credentials.  
    - On Error: Continue Regular Output.  
    - Connect both `Set Company URL` and `Set Company URL1` → `Get LinkedIn Company`.

16. **Add Set Node to Prepare Email Context Data**  
    - Name: `Set Data`  
    - Set static fields:  
      - `company_name`: "AAAutomations"  
      - `email_address`: "hello@aaautomations.com"  
      - `company_info`: Detailed marketing text describing company and services.  
      - `sender_name`: "Adam at AAAutomations"  
    - Connect `Get LinkedIn Company` → `Set Data`.

17. **Add LangChain Chain LLM Node to Generate Email**  
    - Name: `Generate Personalized Email`  
    - Input text includes stringified JSON of `Get LinkedIn Person` and `Get LinkedIn Company` outputs.  
    - Prompt instructs AI to produce JSON with `subject` and `body` fields for cold email.  
    - Connect `Set Data` → `Generate Personalized Email`.

18. **Add OpenRouter Chat Model Node (Claude 3.7)**  
    - Name: `OpenRouter Chat Model`  
    - Set model to `anthropic/claude-3.7-sonnet`.  
    - Use OpenRouter API credentials.  
    - Connect `Generate Personalized Email` AI language model input → `OpenRouter Chat Model`.

19. **Add Structured Output Parser Node**  
    - Name: `Structured Output Parser`  
    - Define JSON schema example matching AI output with `subject` and `body`.  
    - Connect AI output parser input from `Generate Personalized Email`.  
    - Connect parser output back to `Generate Personalized Email` node.

20. **Add Gmail Node to Send Email**  
    - Name: `Gmail`  
    - Recipient: Lead’s email from Google Sheets.  
    - Subject and message body: from AI parsed output.  
    - Reply-To and Sender Name: from `Set Data`.  
    - Use Gmail OAuth2 credentials.  
    - Connect `Generate Personalized Email` → `Gmail`.  
    - Connect `Gmail` → `Loop Over Items` (to process next lead).

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                       |
|-----------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| This workflow uses Apify actors for LinkedIn profile and company detail scraping with rate limits and timeouts configured for reliability. | Apify actors: `apimaestro~linkedin-profile-detail`, `apimaestro~linkedin-company-detail`, `apify~rag-web-browser` |
| AI model is Claude 3.7 accessed via OpenRouter API, configured with specialized prompt for personalized cold email generation. | OpenRouter Claude 3.7 model: `anthropic/claude-3.7-sonnet`                                          |
| Emails are composed with style guidelines emphasizing humor, conciseness, personalization, and Markdown link formatting. | Email style instructions embedded in LLM prompt.                                                    |
| Workflow handles API errors gracefully by continuing execution to avoid halting on partial failures. | HTTP request nodes have `onError: continueRegularOutput`.                                           |
| Google Sheets OAuth2 and Gmail OAuth2 credentials must be set up and authorized before running.     | Credential configuration steps required.                                                            |
| Sticky notes in the workflow describe each main step for easier maintenance and onboarding.         | Sticky notes visible in the workflow UI for context.                                                |

---

**Disclaimer:** The provided text is exclusively from an automated workflow created with n8n, respecting all content policies and legal use. All data processed is legal and publicly available.