Google Maps Lead Scraper & Enrichment with AI-Powered Personalized Outreach

https://n8nworkflows.xyz/workflows/google-maps-lead-scraper---enrichment-with-ai-powered-personalized-outreach-6091


# Google Maps Lead Scraper & Enrichment with AI-Powered Personalized Outreach

### 1. Workflow Overview

This workflow automates the process of generating targeted business leads from Google Maps data, enriching those leads with detailed contact and social information, and crafting AI-powered personalized outreach messages. It is designed primarily for sales and marketing professionals seeking to build enriched lead lists and automate personalized communication at scale.

The workflow logically divides into the following blocks:

- **1.1 Trigger and Data Acquisition:** Starts the workflow manually or via a trigger, then fetches business lead data using an HTTP request to the Apify Google Maps Scraper API.
- **1.2 Initial Data Processing and Filtering:** Handles incoming data to separate enriched leads from those lacking enrichment and formats lead data.
- **1.3 Lead Enrichment and Outreach Message Generation:** Processes enriched leads, extracts individual contacts, and uses AI to generate personalized sales outreach paragraphs; results are appended to a Google Sheet.
- **1.4 Handling Un-Enriched Leads via Website Scraping:** For leads missing enrichment, a secondary process scrapes their websites to extract social media profiles and updates another Google Sheet with this new information.
- **1.5 Secondary Workflow Trigger for Continuous Enrichment:** Monitors the sheet of un-enriched leads, triggers website scraping and social media extraction in batches to augment lead data iteratively.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Data Acquisition

**Overview:**  
Initiates the workflow manually, then calls the Apify Google Maps Scraper API to fetch business leads data with optional enrichment.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- HTTP Request

**Node Details:**  

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Starts the workflow on manual execution.  
  - Configuration: Default manual trigger, no parameters.  
  - Inputs: None  
  - Outputs: Triggers HTTP Request node.  
  - Edge Cases: No input failure; manual start only.  

- **HTTP Request**  
  - Type: HTTP Request  
  - Role: Calls Apify API's "Run Actor synchronously and get dataset items" endpoint to retrieve Google Maps leads.  
  - Configuration:  
    - POST method to `https://api.apify.com/v2/acts/:actorId/run-sync-get-dataset-items`  
    - Headers include Accept: application/json and Authorization with Bearer token placeholder `<token>`.  
    - Body is JSON input for scraper parameters (configured externally).  
  - Inputs: Trigger from manual node  
  - Outputs: Data JSON array of business leads to filtering nodes.  
  - Edge Cases: API auth errors, rate limits, malformed JSON response.  
  - Sticky Note: Instructions on setting up Apify node with proper actor ID, JSON input, and API key.  

---

#### 1.2 Initial Data Processing and Filtering

**Overview:**  
Separates leads with enrichment data from those without, prepares data fields for downstream processing.

**Nodes Involved:**  
- Filter (for leads without enrichment)  
- Filter3 (for leads with enrichment)  
- Edit Fields1  
- Edit Fields2  
- Filter5  
- Code (contact extraction preparation)  
- Loop Over Items

**Node Details:**  

- **Filter**  
  - Type: Filter  
  - Role: Filters leads where `leadsEnrichment` array is empty (no enrichment).  
  - Configuration: Condition checks if `leadsEnrichment` is empty.  
  - Inputs: HTTP Request output  
  - Outputs: Leads without enrichment to Edit Fields1, leads with enrichment to Filter3.

- **Filter3**  
  - Type: Filter  
  - Role: Filters for leads where `leadsEnrichment` array is not empty (has enrichment).  
  - Configuration: Condition checks that `leadsEnrichment` is not empty.  
  - Inputs: HTTP Request output  
  - Outputs: Leads with enrichment to Edit Fields2.

- **Edit Fields1**  
  - Type: Set  
  - Role: Prepares fields for un-enriched leads for appending to Google Sheet (Sheet2).  
  - Configuration: Sets fields like title, street, city, state, website, phone from JSON.  
  - Inputs: Filter output (un-enriched leads)  
  - Outputs: Append or update row in sheet (un-enriched leads).

- **Edit Fields2**  
  - Type: Set  
  - Role: Prepares fields for enriched leads, flattening and selecting key properties including nested enrichment data.  
  - Configuration: Extracts title, categoryName, city, street, phone, website, plus up to three enriched lead contacts (fullName, linkedinProfile, jobTitle, email).  
  - Inputs: Filter3 output (enriched leads)  
  - Outputs: Filter5.

- **Filter5**  
  - Type: Filter  
  - Role: Filters enriched leads that have at least one lead contact email present.  
  - Configuration: Checks if any of `leadsEnrichment[0].email`, `[1].email`, or `[2].email` is non-empty.  
  - Inputs: Edit Fields2 output  
  - Outputs: Code node for contact extraction.

- **Code**  
  - Type: Code  
  - Role: Transforms flat enriched lead data into an array of individual contact objects, each combined with company data.  
  - Configuration: JavaScript groups leadsEnrichment fields by index, filters out empty contacts, and merges company info for each contact.  
  - Inputs: Filter5 output  
  - Outputs: Loop Over Items node.

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Iterates over individual enriched contacts for further processing.  
  - Configuration: Default batch options.  
  - Inputs: Code node output (array of contacts)  
  - Outputs: Edit Fields node for per-contact data preparation.

---

#### 1.3 Lead Enrichment and Outreach Message Generation

**Overview:**  
Generates AI-personalized outreach messages for each enriched contact and saves results to a Google Sheet.

**Nodes Involved:**  
- Edit Fields  
- AI Agent (Langchain Agent)  
- OpenAI Chat Model  
- Append or update row in sheet3 (Google Sheets)

**Node Details:**  

- **Edit Fields**  
  - Type: Set  
  - Role: Sets individual contact fields and company data for AI prompt.  
  - Configuration: Assigns title, categoryName, city, website, phone, fullName, linkedinProfile, jobTitle, email, street from current item.  
  - Inputs: Loop Over Items (contact)  
  - Outputs: AI Agent node.

- **AI Agent**  
  - Type: Langchain Agent (AI)  
  - Role: Uses a custom prompt to generate a personalized outbound email paragraph for the contact.  
  - Configuration:  
    - Prompt instructs to identify one pressing challenge relevant to the prospect's company/role.  
    - Returns a concise paragraph starting with “Hi {fullName},” mentioning the job title naturally, with empathy and insight but no product pitch.  
  - Inputs: Edit Fields output (contact info)  
  - Outputs: Append or update row in sheet3.

- **OpenAI Chat Model**  
  - Type: Langchain LM Chat OpenAI  
  - Role: Underlying language model for AI Agent (model: gpt-4.1-mini).  
  - Configuration: Uses OpenAI API credentials.  
  - Inputs: AI Agent language model request  
  - Outputs: AI Agent response (personalized message).

- **Append or update row in sheet3**  
  - Type: Google Sheets  
  - Role: Saves enriched lead contact details plus AI-generated personalized message to Google Sheet (Sheet3).  
  - Configuration: AppendOrUpdate operation matched by "Title" column, writing fields including fullName, city, email, phone, title, street, categoryName, jobTitle, linkedinProfile, personalized starter message.  
  - Inputs: AI Agent output  
  - Outputs: Loop Over Items (to continue batch processing).  
  - Credentials: Google Sheets OAuth2 account.

---

#### 1.4 Handling Un-Enriched Leads via Website Scraping

**Overview:**  
For leads missing enrichment, this block scrapes their websites to extract social media profiles and updates a Google Sheet with the results for further enrichment.

**Nodes Involved:**  
- Google Sheets Trigger  
- Filter6  
- Batch Processing Logic (SplitInBatches)  
- Edit Fields3  
- Scrape Website Content (HTTP Request to Firecrawl)  
- Extract Contact Information (Code)  
- Append or update row in sheet1 (Google Sheets)

**Node Details:**  

- **Google Sheets Trigger**  
  - Type: Google Sheets Trigger  
  - Role: Watches the Google Sheet (Sheet2) where un-enriched leads are saved.  
  - Configuration: Polls every minute for new rows.  
  - Inputs: External Google Sheet changes  
  - Outputs: Filter6.

- **Filter6**  
  - Type: Filter  
  - Role: Filters leads that have a non-empty Website field to proceed for scraping.  
  - Configuration: Checks if Website field is not empty.  
  - Inputs: Google Sheets Trigger data  
  - Outputs: Batch Processing Logic.

- **Batch Processing Logic**  
  - Type: SplitInBatches  
  - Role: Processes website scraping in batches for scalability.  
  - Inputs: Filter6 output  
  - Outputs: Edit Fields3.

- **Edit Fields3**  
  - Type: Set  
  - Role: Prepares fields for website scraping and Google Sheets update.  
  - Configuration: Sets Title, Website, City, Street, Phone fields from current item.  
  - Inputs: Batch Processing Logic output  
  - Outputs: Scrape Website Content.

- **Scrape Website Content**  
  - Type: HTTP Request  
  - Role: Calls Firecrawl API to scrape website content in markdown and HTML format.  
  - Configuration:  
    - POST to `https://api.firecrawl.dev/v1/scrape` with URL of the company's website in JSON body.  
    - Authorization header with Bearer token placeholder `<token>`.  
  - Inputs: Edit Fields3 output  
  - Outputs: Extract Contact Information.

- **Extract Contact Information**  
  - Type: Code  
  - Role: Parses scraped HTML/markdown for social media URLs using regex for Facebook, Instagram, LinkedIn, Twitter, Yelp, TikTok, YouTube.  
  - Configuration: Custom JavaScript that normalizes URLs, deduplicates, and returns object with social media arrays.  
  - Inputs: Scrape Website Content output  
  - Outputs: Append or update row in sheet1.

- **Append or update row in sheet1**  
  - Type: Google Sheets  
  - Role: Updates Google Sheet (Sheet3) with extracted social media profiles for each company.  
  - Configuration: AppendOrUpdate operation matched by "Title" column, sets social media URLs (LinkedIn, Twitter, Facebook, Instagram).  
  - Inputs: Extract Contact Information output  
  - Outputs: Batch Processing Logic (loop for next batch).  
  - Credentials: Google Sheets OAuth2 account.  
  - Sticky Note: Reminder to use Firecrawl API key in header parameters.

---

#### 1.5 Secondary Workflow Trigger for Continuous Enrichment

**Overview:**  
This block is effectively the ongoing batch processing loop driven by the Google Sheets Trigger and Batch Processing Logic nodes to continuously enrich leads missing data.

**Nodes Involved:**  
- Google Sheets Trigger (repeated)  
- Filter6  
- Batch Processing Logic  
- Edit Fields3  
- Scrape Website Content  
- Extract Contact Information  
- Append or update row in sheet1

**Node Details:**  
This block reuses the same nodes detailed in 1.4 above, enabling continuous monitoring and enrichment of leads without initial enrichment data.

---

### 3. Summary Table

| Node Name                  | Node Type                     | Functional Role                                          | Input Node(s)              | Output Node(s)                | Sticky Note                                                                                      |
|----------------------------|-------------------------------|----------------------------------------------------------|----------------------------|------------------------------|------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger               | Starts workflow manually                                  | None                       | HTTP Request                 |                                                                                                |
| HTTP Request               | HTTP Request                  | Calls Apify Google Maps Scraper API to fetch leads       | When clicking ‘Execute workflow’ | Filter3, Filter              | Instructions on API setup: actor ID, JSON body, API key                                        |
| Filter                    | Filter                       | Filters leads without enrichment                          | HTTP Request               | Edit Fields1                 |                                                                                                |
| Filter3                   | Filter                       | Filters leads with enrichment                             | HTTP Request               | Edit Fields2                 |                                                                                                |
| Edit Fields1              | Set                          | Prepares un-enriched lead fields                          | Filter                     | Append or update row in sheet |                                                                                                |
| Edit Fields2              | Set                          | Prepares enriched lead fields                             | Filter3                    | Filter5                     |                                                                                                |
| Filter5                   | Filter                       | Filters enriched leads with emails                        | Edit Fields2               | Code                        |                                                                                                |
| Code                      | Code                         | Converts enriched lead data into individual contacts      | Filter5                    | Loop Over Items             |                                                                                                |
| Loop Over Items           | SplitInBatches               | Iterates over contacts                                    | Code                       | Edit Fields                 |                                                                                                |
| Edit Fields               | Set                          | Prepares contact data for AI processing                   | Loop Over Items            | AI Agent                   |                                                                                                |
| AI Agent                  | Langchain Agent (AI)         | Generates personalized outreach paragraph                 | Edit Fields                | Append or update row in sheet3 |                                                                                                |
| OpenAI Chat Model         | Langchain LM Chat OpenAI     | Underlying LM for AI Agent                                | AI Agent (languageModel)   | AI Agent                   |                                                                                                |
| Append or update row in sheet3 | Google Sheets               | Saves enriched contacts with personalized messages       | AI Agent                   | Loop Over Items             |                                                                                                |
| Google Sheets Trigger     | Google Sheets Trigger        | Watches un-enriched leads sheet                           | External Google Sheet      | Filter6                    |                                                                                                |
| Filter6                   | Filter                       | Filters leads with non-empty Website field               | Google Sheets Trigger      | Batch Processing Logic      |                                                                                                |
| Batch Processing Logic    | SplitInBatches               | Processes website scraping in batches                     | Filter6                    | Edit Fields3                |                                                                                                |
| Edit Fields3              | Set                          | Prepares fields for website scraping                      | Batch Processing Logic     | Scrape Website Content      |                                                                                                |
| Scrape Website Content    | HTTP Request                 | Calls Firecrawl API to scrape website content            | Edit Fields3               | Extract Contact Information | Use Firecrawl API key in header parameters                                                     |
| Extract Contact Information | Code                         | Extracts social media URLs from scraped content           | Scrape Website Content     | Append or update row in sheet1 |                                                                                                |
| Append or update row in sheet1 | Google Sheets               | Updates sheet with social media profiles                  | Extract Contact Information | Batch Processing Logic      |                                                                                                |
| Sticky Note7              | Sticky Note                  | Full workflow overview, use cases, instructions          | None                       | None                       | Extensive description with usage instructions and contact info for support                    |
| Sticky Note               | Sticky Note                  | Apify node setup instructions                             | None                       | None                       | Explains setting actor ID, JSON input, and API key                                            |
| Sticky Note1              | Sticky Note                  | Firecrawl node API usage reminder                         | None                       | None                       | Reminder to replace <token> with Firecrawl API key                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Start the workflow manually.

2. **Add HTTP Request Node**  
   - Type: HTTP Request  
   - Configure:  
     - Method: POST  
     - URL: `https://api.apify.com/v2/acts/:actorId/run-sync-get-dataset-items` (replace `:actorId` with your Apify Google Maps Scraper actor ID)  
     - Headers:  
       - Accept: application/json  
       - Authorization: Bearer `<your Apify API token>` (configure credential or input securely)  
     - Body: JSON input specifying scraper parameters (search query, location, enrichment enabled).  
   - Connect Manual Trigger → HTTP Request.

3. **Add Filter Node to Separate Un-Enriched Leads**  
   - Type: Filter  
   - Condition: `$json.leadsEnrichment` is empty (array empty).  
   - Connect HTTP Request → Filter.

4. **Add Filter Node to Separate Enriched Leads**  
   - Type: Filter  
   - Condition: `$json.leadsEnrichment` is not empty.  
   - Connect HTTP Request → Filter3.

5. **Add Set Node (Edit Fields1) for Un-Enriched Leads**  
   - Type: Set  
   - Assign fields: title, street, city, state, website, phone from JSON.  
   - Connect Filter (un-enriched output) → Edit Fields1.

6. **Add Google Sheets Node to Append Un-Enriched Leads**  
   - Type: Google Sheets  
   - Operation: Append or Update  
   - Spreadsheet: Your Google Sheet ID  
   - Sheet: Sheet2 (or appropriate)  
   - Map columns: title, street, city, state, website, phone accordingly.  
   - Connect Edit Fields1 → Append or update row in sheet.

7. **Add Set Node (Edit Fields2) for Enriched Leads**  
   - Type: Set  
   - Assign fields: title, categoryName, city, street, phone, website plus up to 3 enriched contacts’ fullName, linkedinProfile, jobTitle, email.  
   - Connect Filter3 → Edit Fields2.

8. **Add Filter Node (Filter5) to Keep Enriched Leads with Emails**  
   - Type: Filter  
   - Condition: Any of `leadsEnrichment[0].email`, `[1].email`, or `[2].email` is not empty.  
   - Connect Edit Fields2 → Filter5.

9. **Add Code Node to Flatten Enriched Contacts**  
   - Type: Code  
   - JS script: Group leadsEnrichment by index, merge with company data, filter out empty contacts, output array of individual contact objects.  
   - Connect Filter5 → Code.

10. **Add SplitInBatches Node (Loop Over Items) to Iterate Contacts**  
    - Type: SplitInBatches  
    - Default batch size (e.g., 1 or more).  
    - Connect Code → Loop Over Items.

11. **Add Set Node (Edit Fields) for Contact Data**  
    - Type: Set  
    - Assign fields for AI prompt: title, categoryName, city, website, phone, fullName, linkedinProfile, jobTitle, email, street.  
    - Connect Loop Over Items → Edit Fields.

12. **Add Langchain Agent Node (AI Agent)**  
    - Type: @n8n/n8n-nodes-langchain.agent  
    - Configure prompt with instructions to generate personalized outreach paragraph using contact and company data, referencing fullName and jobTitle in greeting and message.  
    - Connect Edit Fields → AI Agent.

13. **Add OpenAI Chat Model Node**  
    - Type: @n8n/n8n-nodes-langchain.lmChatOpenAi  
    - Model: gpt-4.1-mini  
    - Credentials: OpenAI API key  
    - Connect AI Agent (ai_languageModel) → OpenAI Chat Model.

14. **Add Google Sheets Node to Append Enriched Contacts with AI Messages**  
    - Type: Google Sheets  
    - Operation: Append or Update  
    - Spreadsheet and Sheet: Your Google Sheet ID, Sheet3 (or appropriate)  
    - Map fields including contact info and AI-generated message.  
    - Connect AI Agent output → Append or update row in sheet3.

15. **Add Google Sheets Trigger Node**  
    - Type: Google Sheets Trigger  
    - Configure to watch Sheet2 (un-enriched leads sheet) with polling every minute.  
    - Credentials: Google Sheets OAuth2  
    - This node triggers the secondary enrichment process.

16. **Add Filter Node (Filter6) for Leads with Website URLs**  
    - Condition: Website field is not empty.  
    - Connect Google Sheets Trigger → Filter6.

17. **Add SplitInBatches Node (Batch Processing Logic)**  
    - For batch processing leads to crawl websites.  
    - Connect Filter6 → Batch Processing Logic.

18. **Add Set Node (Edit Fields3) to Prepare Website Scraping Data**  
    - Assign Title, Website, City, Street, Phone fields.  
    - Connect Batch Processing Logic → Edit Fields3.

19. **Add HTTP Request Node (Scrape Website Content)**  
    - POST to Firecrawl API: `https://api.firecrawl.dev/v1/scrape`  
    - Body JSON includes URL from Website field and formats markdown, html.  
    - Authorization header with Bearer `<Firecrawl API token>`.  
    - Connect Edit Fields3 → Scrape Website Content.

20. **Add Code Node (Extract Contact Information)**  
    - JavaScript to extract social media profiles (Facebook, Instagram, LinkedIn, Twitter, Yelp, TikTok, YouTube) from scraped HTML/markdown using regex.  
    - Connect Scrape Website Content → Extract Contact Information.

21. **Add Google Sheets Node to Append Social Media Profiles**  
    - AppendOrUpdate operation on Sheet3 (or designated sheet)  
    - Map fields: Title, Twitter, Facebook, LinkedIn, Instagram from extracted data.  
    - Connect Extract Contact Information → Append or update row in sheet1.

22. **Loop Back**  
    - Connect Append or update row in sheet1 → Batch Processing Logic to continue processing batches.

23. **Add Sticky Notes for Documentation**  
    - Add notes explaining API setup for Apify and Firecrawl, usage instructions, and workflow overview.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                                     |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| This n8n template automates business lead generation from Google Maps using Apify's Google Maps Scraper with built-in enrichment and personalized AI outreach. It includes a second workflow that scrapes websites with Firecrawl to extract social profiles for leads lacking initial enrichment. Use cases include building targeted lead lists, enriching CRM data, and automating personalized outreach. Requires Apify API key, Google Sheets account, Firecrawl API key, and optionally email/messaging credentials. Contact: designbyaze@gmail.com                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Full workflow description provided in the main sticky note node (Sticky Note7)                                   |
| Apify Node setup instructions: post the correct API endpoint URL under “Run Actor synchronously and get dataset items,” input JSON body with scraper parameters, and replace `<token>` with your Apify API key in header parameters.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Sticky Note near HTTP Request node.                                                                               |
| Firecrawl API usage reminder: Replace `<token>` with your Firecrawl API key in the header parameters for the HTTP Request node that scrapes website content.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Sticky Note near Scrape Website Content node.                                                                     |

---

# Disclaimer

The text provided derives exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.