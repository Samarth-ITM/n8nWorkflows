Find & Score LinkedIn Leads with GPT-4 AI and Export to Google Sheets CRM

https://n8nworkflows.xyz/workflows/find---score-linkedin-leads-with-gpt-4-ai-and-export-to-google-sheets-crm-11189


# Find & Score LinkedIn Leads with GPT-4 AI and Export to Google Sheets CRM

### 1. Workflow Overview

This workflow automates the process of finding, enriching, scoring, and exporting LinkedIn company leads to a Google Sheets CRM using AI. It targets sales and marketing professionals who want to identify potential business clients based on LinkedIn company data, enriched and scored by GPT-4 AI, then stored for further action.

The workflow is logically divided into three main blocks:

- **1.1 Input Initialization & Search**: Setting parameters for target company profiles and fetching company lists via the Ghost Genius LinkedIn API.
- **1.2 Company Data Enrichment & Validation**: Retrieving detailed company info, filtering for quality, and checking for duplicates in the CRM.
- **1.3 AI Scoring & CRM Export**: Using GPT-4 to score company relevance, then adding scored companies to a Google Sheets CRM with throttling to respect API limits.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization & Search

- **Overview:**  
Defines search criteria variables and queries the Ghost Genius API for LinkedIn companies matching those criteria. It handles pagination and limits results to manageable batches.

- **Nodes Involved:**  
  - Start  
  - Set Variables  
  - Search Companies  
  - Extract Company Data  
  - Process Each Company Batch Control  
  - Sticky Note (LinkedIn Company Search)

- **Node Details:**  

  - **Start**  
    - Type: Manual Trigger  
    - Role: Initiates workflow manually.  
    - Connections: Outputs to Set Variables.  
    - Edge cases: None.

  - **Set Variables**  
    - Type: Set  
    - Role: Defines key search parameters such as target keywords, company size, location ID (from Ghost Genius location finder), product/service description, and positive/negative indicators for AI scoring.  
    - Configuration: Multiple string assignments setting filters and AI prompt variables.  
    - Connections: Outputs to Search Companies.  
    - Edge cases: User must correctly input variables; invalid location or size codes cause poor search results.

  - **Search Companies**  
    - Type: HTTP Request  
    - Role: Calls Ghost Genius API endpoint `/v2/search/companies` with search parameters: keywords, location, company size.  
    - Configuration: Uses generic HTTP header authentication with Bearer token, paginates with max 3 pages, 2-second intervals, stops when data is empty.  
    - Connections: Outputs to Extract Company Data.  
    - Edge cases: API rate limits, network timeouts, incorrect credentials, empty search results.

  - **Extract Company Data**  
    - Type: Split Out  
    - Role: Splits the batch response array (`data` field) into individual company items for processing.  
    - Connections: Outputs to Process Each Company Batch Control.  
    - Edge cases: Empty arrays cause no output; malformed responses may fail splitting.

  - **Process Each Company Batch Control**  
    - Type: Split In Batches  
    - Role: Controls per-company processing by batching items one at a time (batch size 1).  
    - Connections: For empty batch, loops back to no node; else feeds into Get Company Info (next block).  
    - Edge cases: Batch processing halts if input is empty; must handle batch flow correctly.

  - **Sticky Note (LinkedIn Company Search)**  
    - Provides detailed instructions on how to set filters for LinkedIn company search, API limits, and tips for handling large result sets.

#### 2.2 Company Data Enrichment & Validation

- **Overview:**  
For each company, retrieves detailed information (via LinkedIn URL), filters companies based on website presence and follower count (>200), and checks if the company already exists in the Google Sheets CRM by LinkedIn ID to avoid duplicates.

- **Nodes Involved:**  
  - Get Company Info  
  - Filter Valid Companies  
  - Check If Company Exists  
  - Is New Company?  
  - Sticky Note (Company Data Processing/Validation)

- **Node Details:**

  - **Get Company Info**  
    - Type: HTTP Request  
    - Role: Fetches full company details from Ghost Genius API using company LinkedIn URL as a query parameter.  
    - Configuration: Uses generic HTTP header auth, batch size 1 with 2-second interval delay to respect API limits, retries on failure.  
    - Input: LinkedIn URL from batch item.  
    - Output: Detailed company JSON.  
    - Edge cases: API timeouts, invalid URLs, rate limits.

  - **Filter Valid Companies**  
    - Type: If  
    - Role: Filters companies that have a non-empty website and more than 200 followers to ensure quality.  
    - Configuration: Conditions check `website` not empty and `followers_count` > 200.  
    - Input: Company details from Get Company Info.  
    - Outputs: True branch to Check If Company Exists; false branch loops back to Process Each Company Batch Control (skips invalid).  
    - Edge cases: Missing fields in response JSON may cause condition evaluation errors.

  - **Check If Company Exists**  
    - Type: Google Sheets  
    - Role: Looks up the company LinkedIn ID in the CRM Google Sheet to prevent duplicate entries.  
    - Configuration: Filters by company ID column in the "Companies" sheet of the linked Google Sheet document.  
    - Input: Company ID from previous node.  
    - Output: Passes to Is New Company? node.  
    - Edge cases: Google Sheets API quota limits, authentication errors.

  - **Is New Company?**  
    - Type: If  
    - Role: Determines if the company is new by checking if the lookup result is empty.  
    - Configuration: Condition tests if first item of lookup result JSON is empty (meaning company not found).  
    - Outputs: True branch to AI Company Scoring (new company); false branch loops back to Process Each Company Batch Control (skip existing).  
    - Edge cases: Lookup failures or malformed data cause false negatives.

  - **Sticky Note (Company Data Processing/Validation)**  
    - Describes the enrichment and filtering process, batching for API rate limit compliance, and duplicate prevention.

#### 2.3 AI Scoring & CRM Export

- **Overview:**  
Applies GPT-4 AI scoring to evaluate company fit based on detailed data and user-defined positive/negative indicators, then writes the scored companies to the Google Sheets CRM with appropriate rate limiting.

- **Nodes Involved:**  
  - AI Company Scoring  
  - Wait 2s  
  - Add Company to CRM  
  - Sticky Note (AI Scoring and Storage)

- **Node Details:**

  - **AI Company Scoring**  
    - Type: OpenAI (LangChain)  
    - Role: Uses GPT-4 to score company interest likelihood on scale 0-10, based on company profile and custom positive/negative indicators.  
    - Configuration: Model GPT-4.1, temperature 0.2 for deterministic output, system prompt dynamically includes product/service and indicators from Set Variables node, company info from Filter Valid Companies node.  
    - Input: Company details from Is New Company? true branch.  
    - Output: JSON with score field.  
    - Edge cases: API key invalid, rate limits, parsing errors if AI response deviates from expected JSON format.

  - **Wait 2s**  
    - Type: Wait  
    - Role: Adds 3 seconds delay to avoid Google Sheets API rate limiting when adding rows.  
    - Configuration: Wait 3 seconds.  
    - Input: AI Company Scoring output.  
    - Output: To Add Company to CRM.  
    - Edge cases: None.

  - **Add Company to CRM**  
    - Type: Google Sheets  
    - Role: Appends company data and AI score to the "Companies" sheet in the Google Sheets CRM.  
    - Configuration: Maps company ID, name, score, state ("Qualified"), summary, website, LinkedIn URL into columns. Uses append operation.  
    - Input: Wait 2s node output.  
    - Output: Loops back to Process Each Company Batch Control for next company.  
    - Edge cases: Google Sheets API quota limits, authentication errors, schema mismatch.

  - **Sticky Note (AI Scoring and Storage)**  
    - Notes importance of customizing variables and prompt, always storing companies regardless of score, and Google Sheets rate limiting with wait module.

---

### 3. Summary Table

| Node Name                    | Node Type                 | Functional Role                                | Input Node(s)                   | Output Node(s)                      | Sticky Note                                                                                                             |
|------------------------------|---------------------------|------------------------------------------------|---------------------------------|------------------------------------|-------------------------------------------------------------------------------------------------------------------------|
| Start                        | Manual Trigger            | Initiates workflow                             | -                               | Set Variables                      |                                                                                                                         |
| Set Variables                | Set                       | Defines search filters and AI prompt variables | Start                           | Search Companies                  |                                                                                                                         |
| Search Companies             | HTTP Request              | Queries LinkedIn companies via Ghost Genius API | Set Variables                  | Extract Company Data              | LinkedIn Company Search: explains search filters, API limits, paging, and tips.                                         |
| Extract Company Data         | Split Out                 | Splits company list JSON to individual items   | Search Companies               | Process Each Company Batch Control |                                                                                                                         |
| Process Each Company Batch Control | Split In Batches          | Controls per-company batch processing           | Extract Company Data, Filter Valid Companies, Add Company to CRM | Get Company Info (main flow), or no node (empty batch) |                                                                                                                         |
| Get Company Info             | HTTP Request              | Retrieves detailed company info by LinkedIn URL | Process Each Company Batch Control | Filter Valid Companies            | Company Data Processing/Validation: describes enrichment, filtering, batching for rate limits, duplicate checking.       |
| Filter Valid Companies       | If                        | Filters companies by website presence and followers count | Get Company Info               | Check If Company Exists, Process Each Company Batch Control |                                                                                                                         |
| Check If Company Exists      | Google Sheets             | Checks CRM sheet for existing company by ID    | Filter Valid Companies          | Is New Company?                   |                                                                                                                         |
| Is New Company?              | If                        | Determines if company is new or duplicate       | Check If Company Exists         | AI Company Scoring (new), Process Each Company Batch Control (duplicate) |                                                                                                                         |
| AI Company Scoring           | OpenAI (LangChain)        | Scores company interest likelihood using GPT-4 | Is New Company? (true branch)  | Wait 2s                         | AI Scoring and Storage: explains prompt customization, always storing companies, Google Sheets rate limits.             |
| Wait 2s                     | Wait                      | Adds delay to prevent Google Sheets API throttling | AI Company Scoring             | Add Company to CRM               |                                                                                                                         |
| Add Company to CRM           | Google Sheets             | Appends scored company data to Google Sheets CRM | Wait 2s                       | Process Each Company Batch Control |                                                                                                                         |
| Sticky Note                 | Sticky Note               | Instructional content for LinkedIn Company Search | -                             | -                              | LinkedIn Company Search: details on API usage and search refinement.                                                    |
| Sticky Note1                | Sticky Note               | Instructional content for Company Data Processing | -                             | -                              | Company Data Processing/Validation: filtering criteria, batching, duplicate avoidance.                                   |
| Sticky Note2                | Sticky Note               | Instructional content for AI scoring and storage | -                             | -                              | AI Scoring and Storage: importance of variable setup and scoring quality, link to Google Sheet template.                 |
| Sticky Note4                | Sticky Note               | Introduction and setup instructions             | -                             | -                              | Workflow introduction, setup links for Ghost Genius API, Google Sheets, OpenAI credentials, and helpful resources.       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow and add a Manual Trigger node named `Start`.**

2. **Add a `Set` node named `Set Variables`:**  
   - Create string fields for:  
     - "Your target" (e.g., "Growth Marketing Agency")  
     - "B: 1-10 employees, C: 11-50 employees, D: 51-200 employees, E: 201-500 employees, F: 501-1000 employees, G: 1001-5000 employees, H: 5001-10,000 employees, I: 10,001+ employees" (e.g., "C")  
     - "Location (find it on : https://www.ghostgenius.fr/tools/search-sales-navigator-locations-id)" (e.g., "103644278")  
     - "Your product or service" (e.g., "our CRM implementation services")  
     - "Positive indicators" (list 3-5 positive factors as multiline string)  
     - "Negative indicators" (list 3-5 negative factors as multiline string)

3. **Add an HTTP Request node named `Search Companies`:**  
   - Method: GET  
   - URL: `https://api.ghostgenius.fr/v2/search/companies`  
   - Query parameters:  
     - keywords = `={{ $json['Your target'] }}`  
     - locations = `={{ $json['Location (find it on : https://www.ghostgenius.fr/tools/search-sales-navigator-locations-id)'] }}`  
     - company_size = `={{ $json['B: 1-10 employees, C: 11-50 employees, D: 51-200 employees, E: 201-500 employees, F: 501-1000 employees, G: 1001-5000 employees, H: 5001-10,000 employees, I: 10,001+ employees'] }}`  
   - Authentication: Create a Generic Credential with HTTP Header Auth named e.g. "Ghost Genius Auth" with header `"Authorization": "Bearer YOUR_TOKEN_HERE"`  
   - Enable pagination: max 3 pages, 2-second interval, stop when response data empty.

4. **Add a Split Out node named `Extract Company Data`:**  
   - Field to split out: `data`

5. **Add a Split In Batches node named `Process Each Company Batch Control`:**  
   - Batch size: 1

6. **Add an HTTP Request node named `Get Company Info`:**  
   - Method: GET  
   - URL: `https://api.ghostgenius.fr/v2/company`  
   - Query parameter: `url = ={{ $json.url }}` (the LinkedIn URL from the batch)  
   - Authentication: Use the same Generic Credential as Search Companies  
   - Enable batching with batch size 1 and 2-second interval to avoid API limits  
   - Enable retry on failure

7. **Add an If node named `Filter Valid Companies`:**  
   - Condition 1: `website` field is not empty  
   - Condition 2: `followers_count` > 200  
   - Both conditions combined with AND

8. **Add a Google Sheets node named `Check If Company Exists`:**  
   - Operation: Lookup  
   - Document: Use your Google Sheet for CRM  
   - Sheet: "Companies" (gid=0)  
   - Filter: lookup value = `={{ $json.id }}` in column "ID"  
   - Credential: Google Sheets OAuth2 credential configured per n8n docs

9. **Add an If node named `Is New Company?`:**  
   - Condition: Check if the first item from `Check If Company Exists` is empty (object empty)  
   - True: proceed to AI Company Scoring  
   - False: loop to next batch company (skip duplicate)

10. **Add an OpenAI (LangChain) node named `AI Company Scoring`:**  
    - Model: GPT-4 (modelId: "gpt-4.1")  
    - Temperature: 0.2  
    - System message: Include a prompt that defines scoring logic based on product/service and positive/negative indicators from `Set Variables` node.  
    - User message: Include company data fields (name, description, size, industry, etc.) from `Filter Valid Companies` node.  
    - Output: JSON with a numeric `score` field  
    - Credentials: OpenAI API key credential configured in n8n

11. **Add a Wait node named `Wait 2s`:**  
    - Wait time: 3 seconds

12. **Add a Google Sheets node named `Add Company to CRM`:**  
    - Operation: Append  
    - Document and sheet: Same as `Check If Company Exists`  
    - Map columns:  
      - ID = `={{ $('Get Company Info').item.json.id }}`  
      - Name = `={{ $('Get Company Info').item.json.name }}`  
      - Score = `={{ $json.message.content.score }}`  
      - State = "Qualified" (static)  
      - Summary = `={{ $('Get Company Info').item.json.description }}`  
      - Website = `={{ $('Get Company Info').item.json.website }}`  
      - LinkedIn = `={{ $('Get Company Info').item.json.url }}`

13. **Connect nodes as follows:**  
    - Start → Set Variables → Search Companies → Extract Company Data → Process Each Company Batch Control → Get Company Info → Filter Valid Companies → Check If Company Exists → Is New Company?  
    - Is New Company? true → AI Company Scoring → Wait 2s → Add Company to CRM → Process Each Company Batch Control (loop)  
    - Is New Company? false → Process Each Company Batch Control (loop)  
    - Filter Valid Companies false → Process Each Company Batch Control (loop)  
    - Process Each Company Batch Control empty batch → end

14. **Create required credentials:**  
    - Ghost Genius API: HTTP Header Auth with Bearer token  
    - Google Sheets OAuth2 credentials following n8n documentation  
    - OpenAI API key credential

15. **Test the workflow with small batches and adjust variables in `Set Variables` for target audience and scoring criteria.**

---

### 5. General Notes & Resources

| Note Content                                                                                                               | Context or Link                                                                                          |
|----------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Workflow enables LinkedIn company search, enrichment, AI scoring, and export to Google Sheets CRM for lead generation.      | General workflow purpose                                                                                 |
| Use Ghost Genius API for LinkedIn data: [Ghost Genius API](https://ghostgenius.fr)                                          | LinkedIn company data API                                                                                |
| Ghost Genius API docs: [API Documentation](https://ghostgenius.fr/docs)                                                    | API reference                                                                                           |
| Location ID finder tool: [Ghost Genius Locations ID Finder](https://ghostgenius.fr/tools/search-sales-navigator-locations-id) | Helps get location codes for filtering                                                                  |
| Google Sheets CRM template: [Make a copy](https://docs.google.com/spreadsheets/d/1LfhqpyjimLjyQcmWY8mUr6YtNBcifiOVLIhAJGV9jiM/edit?usp=sharing) | Example CRM sheet used in workflow                                                                       |
| OpenAI platform overview: [OpenAI docs](https://platform.openai.com/docs/overview)                                         | AI scoring setup                                                                                        |
| Workflow author contact: Upwork profile for questions [Upwork Freelancer](https://www.upwork.com/freelancers/~01cb2136b6898df6c0?s=1110580764771602432) | Support and consulting                                                                                   |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.