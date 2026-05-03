Automated B2B Lead Generation: Google Places, Scrape.do & AI Enrichment

https://n8nworkflows.xyz/workflows/automated-b2b-lead-generation--google-places--scrape-do---ai-enrichment-8448


# Automated B2B Lead Generation: Google Places, Scrape.do & AI Enrichment

### 1. Workflow Overview

This automated workflow is designed for B2B lead generation by sourcing business data from Google Places, enriching it through website scraping and AI-powered contact information extraction, and finally saving the enriched leads into Google Sheets. It targets users who want to systematically discover, qualify, and enrich business leads within a specified category and location, leveraging APIs and AI to gather comprehensive contact details.

The workflow is logically divided into three main blocks:

- **1.1 Lead Discovery & Scoring:** Set search criteria, query Google Places API, parse responses, and compute a lead quality score.
- **1.2 Lead Enrichment Loop:** Iterate over qualified leads, scrape their websites for contact info, extract structured data using AI.
- **1.3 Data Persistence & Reporting:** Save enriched leads to Google Sheets and generate summary/error logs.

---

### 2. Block-by-Block Analysis

#### 2.1 Lead Discovery & Scoring

**Overview:**  
This block initializes search parameters, queries Google Places API to find businesses, processes the results into structured lead records, and filters leads based on a calculated quality score.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- 1. Set Search Parameters (Set)  
- 2. Find Businesses (Google Places) (HTTP Request)  
- 3. Parse & Score Leads (Function)  
- 4. Filter High-Quality Leads (IF)  
- Generate Report (Function)  
- Error Logging (Function)  

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Entry point to manually start the workflow.  
  - *Connections:* Outputs to “1. Set Search Parameters”.  
  - *Failure modes:* None expected; manual trigger.

- **1. Set Search Parameters**  
  - *Type:* Set  
  - *Role:* Defines search criteria such as category, location, coordinates, search radius, and max results.  
  - *Key Parameters:*  
    - `searchCategory`: e.g., "dentist"  
    - `locationName`: e.g., "Istanbul, Turkey"  
    - `latitude` & `longitude`: Geographic coordinates  
    - `radius`: Search radius in meters (e.g., 5000)  
    - `maxResults`: Max number of businesses to fetch (e.g., 20)  
  - *Connections:* Outputs to “2. Find Businesses (Google Places)”.  
  - *Edge Cases:* Incorrect or missing parameters may cause API errors or no results.

- **2. Find Businesses (Google Places)**  
  - *Type:* HTTP Request  
  - *Role:* Calls Google Places API's `places:searchNearby` endpoint to find businesses matching parameters.  
  - *Configuration:*  
    - POST request with JSON body containing `includedTypes`, `maxResultCount`, and circular location restriction.  
    - Headers include `X-Goog-FieldMask` specifying fields to return and `Content-Type: application/json`.  
    - Auth via Header Authentication with Google Places API key.  
  - *Connections:* Outputs to “3. Parse & Score Leads”.  
  - *Edge Cases:* API key invalid or quota exceeded errors, network timeouts.

- **3. Parse & Score Leads**  
  - *Type:* Function  
  - *Role:* Converts API response into consistent lead objects, calculates a `leadScore` based on rating, website presence, phone presence, and operational status.  
  - *Logic Highlights:*  
    - Rating ≥4.5 adds 30 points; ≥4.0 adds 20; ≥3.5 adds 10.  
    - Website presence adds 25, phone adds 20, operational status adds 25.  
  - *Connections:* Outputs to “4. Filter High-Quality Leads”.  
  - *Edge Cases:* Missing fields gracefully defaulted; leadScore can be zero.

- **4. Filter High-Quality Leads**  
  - *Type:* IF  
  - *Role:* Filters leads with `leadScore` > 50 and `businessName` not “N/A”.  
  - *Connections:*  
    - True branch to “5. Loop Through Each Lead” (next block)  
    - False branch to “Generate Report” and “Error Logging” nodes.  
  - *Edge Cases:* Threshold can be adjusted; leads below threshold logged.

- **Generate Report**  
  - *Type:* Function  
  - *Role:* Summarizes workflow run: total processed leads, count and list of low-quality leads, timestamp.  
  - *Connections:* Is an endpoint for low-quality leads path.  
  - *Edge Cases:* None expected.

- **Error Logging**  
  - *Type:* Function  
  - *Role:* Logs errors related to low-quality or invalid leads with time and workflow context.  
  - *Connections:* Runs alongside “Generate Report” on low-quality branch.  
  - *Edge Cases:* Console logging only; could be extended for persistent error tracking.

---

#### 2.2 Lead Enrichment Loop

**Overview:**  
Iterates over each high-quality lead to enrich its data by scraping the business website, extracting footer HTML, and applying AI to extract structured contact information.

**Nodes Involved:**  
- 5. Loop Through Each Lead (SplitInBatches)  
- 6a. Scrape Website with Scrape.do (HTTP Request)  
- 6b. Check if Scrape Was Successful (IF)  
- 6c. Extract Footer from HTML (HTML Extract)  
- 6d. Extract Contact Info with AI (Langchain Agent with OpenAI)  
- OpenAI Chat Model (Language Model)  
- Structured Output Parser (Output Parser)  

**Node Details:**

- **5. Loop Through Each Lead**  
  - *Type:* SplitInBatches  
  - *Role:* Processes leads one-by-one to allow sequential enrichment and avoid API rate limits.  
  - *Connections:*  
    - Main output (no data) back to itself or workflow end (not connected here).  
    - Second output (non-empty batch) to “6a. Scrape Website with Scrape.do”.  
  - *Edge Cases:* Large lead sets require batch size tuning.

- **6a. Scrape Website with Scrape.do**  
  - *Type:* HTTP Request  
  - *Role:* Calls Scrape.do API to fetch raw HTML of the business’s website URL.  
  - *Configuration:*  
    - GET request with query params `url` (lead website) and `token` from environment variable `SCRAPE_DO_API_KEY`.  
    - Continues workflow even if error occurs (`onError: continueRegularOutput`).  
  - *Connections:* Outputs to “6b. Check if Scrape Was Successful”.  
  - *Edge Cases:* Website down, invalid URLs, scraping service limits, HTTP errors.

- **6b. Check if Scrape Was Successful**  
  - *Type:* IF  
  - *Role:* Checks if scraping returned HTTP status different from 400 to proceed.  
  - *Connections:*  
    - True branch to “6c. Extract Footer from HTML”  
    - False branch ends or skips enrichment for this lead.  
  - *Edge Cases:* May need to handle other HTTP errors; currently only 400 checked.

- **6c. Extract Footer from HTML**  
  - *Type:* HTML Extract  
  - *Role:* Extracts the `<footer>` element from the scraped HTML content.  
  - *Configuration:* Uses CSS selector `footer`.  
  - *Connections:* Outputs footer HTML to “6d. Extract Contact Info with AI”.  
  - *Edge Cases:* Footer might be missing or malformed, resulting in empty data.

- **6d. Extract Contact Info with AI**  
  - *Type:* Langchain Agent (OpenAI)  
  - *Role:* Analyzes the extracted footer HTML snippet to parse detailed contact information into a structured JSON object.  
  - *Configuration:*  
    - System message instructs the AI to parse only and output structured JSON with contact emails, phones, social media URLs, location, and other contact methods.  
    - The prompt includes the footer HTML snippet dynamically.  
    - Uses “OpenAI Chat Model” (GPT-4.1-mini) as the language model.  
    - Uses “Structured Output Parser” node for schema validation and output parsing.  
  - *Connections:* Outputs enriched contact info to “7. Save Enriched Lead to Google Sheets”.  
  - *Edge Cases:* AI response errors, rate limits, malformed HTML input, incomplete or ambiguous contact info.

- **OpenAI Chat Model**  
  - *Type:* Language Model Node  
  - *Role:* Provides the GPT-4.1-mini AI model for language understanding and output generation.  
  - *Credentials:* OpenAI API key configured.  
  - *Connections:* Used as the AI engine by the agent node.  
  - *Edge Cases:* API quota, latency, or authentication errors.

- **Structured Output Parser**  
  - *Type:* Output Parser Node  
  - *Role:* Validates and structures the AI output according to a strict JSON schema describing contact, social, location, and other contact methods.  
  - *Connections:* Receives AI raw output and sends parsed data to the AI Agent node.  
  - *Edge Cases:* Parsing failures if AI output does not conform; can cause downstream errors.

---

#### 2.3 Data Persistence & Reporting

**Overview:**  
Stores the enriched lead data into a Google Sheets document and provides summary reporting on the workflow execution.

**Nodes Involved:**  
- 7. Save Enriched Lead to Google Sheets (Google Sheets)  
- Generate Report (Function)  
- Error Logging (Function)  

**Node Details:**

- **7. Save Enriched Lead to Google Sheets**  
  - *Type:* Google Sheets  
  - *Role:* Appends each enriched lead’s data row into a specified Google Sheet.  
  - *Configuration:*  
    - Appends to the sheet with `gid=0` in Spreadsheet ID `1I9uFYn0q-yBmaGwmN24w6L_1-qjqEVaQj0FwN5RfoFY`.  
    - Uses OAuth2 credentials for Google Sheets access.  
    - Maps many fields from the original lead and AI extracted data, including business name, address, phone, website, emails, social media URLs, ratings, location, and search parameters.  
  - *Connections:* Feeds back to "5. Loop Through Each Lead" (likely to continue processing).  
  - *Edge Cases:* Sheet not found, auth errors, column mismatches, rate limits.

- **Generate Report** and **Error Logging** nodes are detailed above in the Lead Discovery block.

---

### 3. Summary Table

| Node Name                       | Node Type                           | Functional Role                                     | Input Node(s)                  | Output Node(s)                         | Sticky Note                                                                                                                      |
|--------------------------------|-----------------------------------|----------------------------------------------------|-------------------------------|---------------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’| Manual Trigger                    | Starts workflow manually                           |                               | 1. Set Search Parameters               | # Lead Generation Bot: Google Places & AI Enrichment (covers entire workflow setup and usage)                                   |
| 1. Set Search Parameters        | Set                               | Defines search criteria                            | When clicking ‘Execute workflow’| 2. Find Businesses (Google Places)    | ### 1. Find, Parse & Score Leads (explains setup of search parameters and scoring threshold)                                    |
| 2. Find Businesses (Google Places)| HTTP Request                    | Queries Google Places API                          | 1. Set Search Parameters        | 3. Parse & Score Leads                 | ### 1. Find, Parse & Score Leads                                                                                               |
| 3. Parse & Score Leads          | Function                         | Parses API response and scores leads              | 2. Find Businesses              | 4. Filter High-Quality Leads           | ### 1. Find, Parse & Score Leads                                                                                               |
| 4. Filter High-Quality Leads    | IF                               | Filters leads by leadScore > 50 and valid name    | 3. Parse & Score Leads          | 5. Loop Through Each Lead; Generate Report; Error Logging | ### 1. Find, Parse & Score Leads                                                                                               |
| 5. Loop Through Each Lead       | SplitInBatches                   | Processes leads one by one                         | 4. Filter High-Quality Leads    | 6a. Scrape Website with Scrape.do      | ### 2. Data Enrichment Loop (explains scraping and AI enrichment per lead)                                                     |
| 6a. Scrape Website with Scrape.do| HTTP Request                    | Scrapes lead website HTML                          | 5. Loop Through Each Lead       | 6b. Check if Scrape Was Successful     | ### 2. Data Enrichment Loop                                                                                                    |
| 6b. Check if Scrape Was Successful| IF                             | Validates scraping response                        | 6a. Scrape Website with Scrape.do| 6c. Extract Footer from HTML           | ### 2. Data Enrichment Loop                                                                                                    |
| 6c. Extract Footer from HTML    | HTML Extract                     | Extracts footer HTML from scraped content         | 6b. Check if Scrape Was Successful| 6d. Extract Contact Info with AI       | ### 2. Data Enrichment Loop                                                                                                    |
| 6d. Extract Contact Info with AI| Langchain Agent (OpenAI)         | Uses AI to extract structured contact data        | 6c. Extract Footer from HTML; OpenAI Chat Model; Structured Output Parser | 7. Save Enriched Lead to Google Sheets | ### 2. Data Enrichment Loop                                                                                                    |
| OpenAI Chat Model               | Language Model                   | Provides GPT-4.1-mini model for AI node           |                               | 6d. Extract Contact Info with AI       |                                                                                                                                |
| Structured Output Parser        | Output Parser                   | Parses AI output into validated structured JSON   | OpenAI Chat Model             | 6d. Extract Contact Info with AI       |                                                                                                                                |
| 7. Save Enriched Lead to Google Sheets| Google Sheets               | Saves enriched lead data to Google Sheets          | 6d. Extract Contact Info with AI| 5. Loop Through Each Lead (loop continuation) | ### 3. Save to Google Sheets (explains saving enriched data to sheet)                                                          |
| Generate Report                | Function                         | Summarizes processed leads and low-quality counts | 4. Filter High-Quality Leads (false branch) |                                | ### 1. Find, Parse & Score Leads                                                                                               |
| Error Logging                 | Function                         | Logs errors for low-quality leads                   | 4. Filter High-Quality Leads (false branch) |                                | ### 1. Find, Parse & Score Leads                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: "When clicking ‘Execute workflow’"  
   - Type: Manual Trigger  
   - No parameters.

2. **Create Set Node for Search Parameters**  
   - Name: "1. Set Search Parameters"  
   - Type: Set  
   - Add string values:  
     - `searchCategory`: e.g., "dentist"  
     - `locationName`: e.g., "Istanbul, Turkey"  
     - `latitude`: e.g., "41.0082"  
     - `longitude`: e.g., "28.9784"  
     - `radius`: "5000" (meters)  
     - `maxResults`: "20"  
   - Connect from Manual Trigger node.

3. **Create HTTP Request Node for Google Places**  
   - Name: "2. Find Businesses (Google Places)"  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://places.googleapis.com/v1/places:searchNearby`  
   - Authentication: HTTP Header Auth (Google Places API key credential)  
   - Headers:  
     - `X-Goog-FieldMask`: `places.id,places.displayName,places.formattedAddress,places.nationalPhoneNumber,places.websiteUri,places.rating,places.businessStatus,places.primaryType,places.location`  
     - `Content-Type`: `application/json`  
   - Body (JSON):  
     ```json
     {
       "includedTypes": ["{{$json["searchCategory"]}}"],
       "maxResultCount": {{$json["maxResults"]}},
       "locationRestriction": {
         "circle": {
           "center": {
             "latitude": {{$json["latitude"]}},
             "longitude": {{$json["longitude"]}}
           },
           "radius": {{$json["radius"]}}
         }
       }
     }
     ```
   - Connect from "1. Set Search Parameters".

4. **Create Function Node to Parse & Score Leads**  
   - Name: "3. Parse & Score Leads"  
   - Type: Function  
   - Paste JavaScript code that parses the API response, normalizes fields, and calculates `leadScore` based on rating, website, phone, and status.  
   - Connect from "2. Find Businesses".

5. **Create IF Node to Filter High-Quality Leads**  
   - Name: "4. Filter High-Quality Leads"  
   - Type: IF  
   - Conditions:  
     - Number: `leadScore` > 50  
     - String: `businessName` != "N/A"  
   - Connect from "3. Parse & Score Leads".

6. **Create Function Node for Generate Report**  
   - Name: "Generate Report"  
   - Type: Function  
   - Include code to summarize total processed leads, count and list low-quality leads, timestamp.  
   - Connect from False branch of "4. Filter High-Quality Leads".

7. **Create Function Node for Error Logging**  
   - Name: "Error Logging"  
   - Type: Function  
   - Include code to log errors for low-quality leads with timestamp and workflow info.  
   - Connect from False branch of "4. Filter High-Quality Leads".

8. **Create SplitInBatches Node for Lead Loop**  
   - Name: "5. Loop Through Each Lead"  
   - Type: SplitInBatches  
   - Connect from True branch of "4. Filter High-Quality Leads".

9. **Create HTTP Request Node to Scrape Website**  
   - Name: "6a. Scrape Website with Scrape.do"  
   - Type: HTTP Request  
   - Method: GET (default)  
   - URL: `http://api.scrape.do/`  
   - Query Parameters (Send as Query):  
     - `url`: `={{ $json.website }}`  
     - `token`: `={{ $env["SCRAPE_DO_API_KEY"] }}`  
   - On error: Continue regular output  
   - Connect from "5. Loop Through Each Lead" second output.

10. **Create IF Node to Check Scrape Success**  
    - Name: "6b. Check if Scrape Was Successful"  
    - Type: IF  
    - Condition: HTTP error status not equal to 400  
    - Connect from "6a. Scrape Website with Scrape.do".

11. **Create HTML Extract Node to Extract Footer**  
    - Name: "6c. Extract Footer from HTML"  
    - Type: HTML Extract  
    - Operation: Extract HTML content  
    - CSS Selector: `footer`  
    - Connect from True branch of "6b. Check if Scrape Was Successful".

12. **Configure OpenAI Chat Model Node**  
    - Name: "OpenAI Chat Model"  
    - Type: Langchain LM Chat OpenAI  
    - Model: GPT-4.1-mini  
    - Credentials: OpenAI API key  
    - No direct input connections; used by agent node.

13. **Create Structured Output Parser Node**  
    - Name: "Structured Output Parser"  
    - Type: Langchain Output Parser Structured  
    - Define JSON schema for contact, social, location, other_contact_methods as per workflow.  
    - No direct input connections; used by agent node.

14. **Create Langchain Agent Node for AI Extraction**  
    - Name: "6d. Extract Contact Info with AI"  
    - Type: Langchain Agent  
    - Text prompt: instruct AI to analyze footer HTML snippet and extract structured contact info as JSON.  
    - System message: set to restrict output to structured data only, no explanations.  
    - Use "OpenAI Chat Model" as language model.  
    - Use "Structured Output Parser" for output parsing.  
    - Connect from "6c. Extract Footer from HTML" and link AI nodes as per n8n Langchain configuration.

15. **Create Google Sheets Node to Save Leads**  
    - Name: "7. Save Enriched Lead to Google Sheets"  
    - Type: Google Sheets  
    - Operation: Append  
    - Spreadsheet ID: `1I9uFYn0q-yBmaGwmN24w6L_1-qjqEVaQj0FwN5RfoFY` (replace with your own)  
    - Sheet Name / GID: `gid=0`  
    - Map fields from lead JSON and AI extracted JSON to corresponding columns (BusinessName, Address, Phone, emails, Facebook, Instagram, Youtube, Linkedin, Pinterest, Other, rating, businessStatus, primaryType, Latitude, Longitude, searchLocation, SearchCategory).  
    - Credentials: Google Sheets OAuth2  
    - Connect from "6d. Extract Contact Info with AI".

16. **Connect “7. Save Enriched Lead to Google Sheets” back to “5. Loop Through Each Lead”**  
    - To continue processing next lead batch.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                   | Context or Link                                                                                     |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| **Lead Generation Bot: Google Places & AI Enrichment**: Purpose and quick setup instructions including API credential configuration for Google Places, Scrape.do, OpenAI, and Google Sheets. | Sticky Note covering entire workflow setup.                                                      |
| **1. Find, Parse & Score Leads**: Explains importance of setting search parameters and lead score filtering threshold.                                                         | Sticky Note near “1. Set Search Parameters” to “4. Filter High-Quality Leads” nodes.              |
| **2. Data Enrichment Loop**: Describes website scraping and AI extraction logic that enriches lead data per entry.                                                              | Sticky Note near “5. Loop Through Each Lead” and scraping nodes.                                 |
| **3. Save to Google Sheets**: Instructions about configuring Google Sheets node and mapping enriched data fields properly.                                                     | Sticky Note near “7. Save Enriched Lead to Google Sheets”.                                      |
| Google Places API Documentation: https://developers.google.com/maps/documentation/places/web-service/search                                                                 | For understanding API parameters and usage.                                                      |
| Scrape.do API Documentation: https://scrape.do/docs                                                                                                                          | For configuring and troubleshooting scraping requests.                                           |
| OpenAI API Documentation: https://platform.openai.com/docs                                                                                                                    | For managing OpenAI credentials and prompt design.                                               |
| n8n Google Sheets Node Docs: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets/                                                                  | For configuring OAuth2 and field mapping.                                                        |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created using n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is lawful and publicly accessible.