Scrape Business Leads from Google Maps and Extract Decision-Maker Info with Olostep

https://n8nworkflows.xyz/workflows/scrape-business-leads-from-google-maps-and-extract-decision-maker-info-with-olostep-11086


# Scrape Business Leads from Google Maps and Extract Decision-Maker Info with Olostep

### 1. Workflow Overview

This workflow automates the generation of business leads by scraping Google Maps search results and then enriching this data by extracting decision-maker information from each business’s website. It is designed primarily for sales or marketing teams seeking targeted outreach lists with enriched contact details.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures user input specifying the target city and business type from a web form.
- **1.2 Google Maps Data Scraping:** Uses the Olostep API to scrape Google Maps for business listings matching the query.
- **1.3 Data Cleaning and Preparation:** Parses the raw JSON response, splits it into individual business entries, and removes duplicates.
- **1.4 Website Scraping for Decision-Maker Info:** For each business, scrapes the official website to extract CEO/founder name and contact email.
- **1.5 Data Storage:** Appends the cleaned and enriched lead information into a Google Sheet.
- **1.6 Control Flow:** Implements batching and wait steps to manage processing rate and avoid API limits or timeouts.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Captures the user’s input for city and business type via a form submission, triggering the workflow.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**

  - **On form submission**  
    - Type: Form Trigger  
    - Role: Entry point triggered by user submitting city and business type.  
    - Configuration:  
      - Form titled "Olostep lead generation"  
      - Fields: City (required, placeholder "London"), Business (required, placeholder "Dentist")  
      - Form description prompts user to enter required fields.  
    - Inputs: None (trigger node)  
    - Outputs: To "scrape information" HTTP Request node  
    - Edge cases: Missing required fields will prevent trigger activation; malformed inputs could affect downstream scraping URLs.

#### 2.2 Google Maps Data Scraping

- **Overview:**  
  Sends a dynamically constructed search URL to Olostep’s API to scrape Google Maps business listings based on user inputs. Extracts key business information such as name, website, location, and phone number.

- **Nodes Involved:**  
  - scrape information

- **Node Details:**

  - **scrape information**  
    - Type: HTTP Request  
    - Role: Calls Olostep API to scrape Google Maps results for the query "City + Business".  
    - Configuration:  
      - POST to `https://api.olostep.com/v1/scrapes`  
      - JSON body includes:  
        - `url_to_scrape` dynamically constructed as `https://www.google.com/maps/search/{{ $json.City }}+{{ $json.business }}`  
        - Wait and scroll actions to allow page to load and reveal listings  
        - Extraction schema for business details (name, website, location, phone number) using Olostep’s `llm_extract` with specified JSON schema  
        - Screen size set to desktop resolution  
        - Authorization header with Bearer token (user must insert their Olostep API key)  
      - Retries up to 5 times on failure, continues on error if it occurs  
    - Inputs: Output from "On form submission" node  
    - Outputs: Raw JSON response to "parsedInfo" node  
    - Edge cases:  
      - API rate limits or network timeouts  
      - 504 Gateway Timeout errors related to the `llm_extract` field (see Sticky Note7)  
      - Authorization errors if API key is missing or invalid  

#### 2.3 Data Cleaning and Preparation

- **Overview:**  
  Processes the raw JSON string by removing escape characters, splits the JSON array into individual business objects, and removes duplicates before proceeding.

- **Nodes Involved:**  
  - parsedInfo  
  - Split Out  
  - Remove Duplicates

- **Node Details:**

  - **parsedInfo**  
    - Type: Set  
    - Role: Cleans raw JSON string by removing backslash escape characters to produce valid JSON array.  
    - Configuration: Assigns a new field `parsedJson` to cleaned JSON string from `result.json_content` in the previous response.  
    - Inputs: From "scrape information" node  
    - Outputs: To "Split Out" node  

  - **Split Out**  
    - Type: Split Out  
    - Role: Splits the array field `parsedJson` into separate items, one per business.  
    - Configuration: Field to split out is `parsedJson`.  
    - Inputs: From "parsedInfo" node  
    - Outputs: To "Remove Duplicates" node  

  - **Remove Duplicates**  
    - Type: Remove Duplicates  
    - Role: Ensures uniqueness of business entries to avoid redundant processing.  
    - Configuration: Default options (likely deduplication based on whole item or key fields)  
    - Inputs: From "Split Out" node  
    - Outputs: To "Loop Over Items" node  
    - Edge cases: May fail if items lack consistent keys or fields for deduplication.  

#### 2.4 Website Scraping for Decision-Maker Info

- **Overview:**  
  For each unique business, scrapes the business’s official website to extract the first name, last name of the CEO or decision-maker, and optionally an email.

- **Nodes Involved:**  
  - Loop Over Items  
  - scrape the name  
  - name

- **Node Details:**

  - **Loop Over Items**  
    - Type: Split In Batches  
    - Role: Iterates over each business item one by one (or in batches) to process individually.  
    - Configuration: Default batch parameters (likely batch size 1)  
    - Inputs: From "Remove Duplicates" node  
    - Outputs:  
      - Main output (empty, possibly for batch control)  
      - Second output to "scrape the name" node for processing each batch item  

  - **scrape the name**  
    - Type: HTTP Request  
    - Role: Calls Olostep API to scrape the business website URL for decision-maker info.  
    - Configuration:  
      - POST to `https://api.olostep.com/v1/scrapes`  
      - `url_to_scrape` is the business website from the current item JSON.  
      - Wait time before scraping: 10 seconds  
      - `llm_extract` schema specifying required fields: `firstName`, `lastName`, optional `email`  
      - Screen size set to default small resolution  
      - Authorization header with Bearer token (same token as first API call)  
      - Retries 5 times on failure, continues on error  
    - Inputs: From "Loop Over Items" node  
    - Outputs: To "name" node  
    - Edge cases:  
      - Websites without clear CEO/contact info  
      - API timeouts or rate limits  
      - Missing or malformed website URLs  

  - **name**  
    - Type: Set  
    - Role: Cleans JSON string returned from "scrape the name" by removing escape characters.  
    - Configuration: Assigns cleaned JSON string to `parsedJson` field.  
    - Inputs: From "scrape the name" node  
    - Outputs: To "Append row in sheet" node  

#### 2.5 Data Storage

- **Overview:**  
  Combines the business info and scraped decision-maker details, then appends a new row to a Google Sheet for record keeping.

- **Nodes Involved:**  
  - Append row in sheet

- **Node Details:**

  - **Append row in sheet**  
    - Type: Google Sheets  
    - Role: Appends each enriched lead as a new row in a specified Google Sheet tab.  
    - Configuration:  
      - Operation: Append  
      - Document ID and Sheet Name specified for target spreadsheet and tab (Sheet ID: `1IoiN3ISmNrxOOWcfw1bQQb6Ru2GV0aN4wbXxGA73LCY`, tab ID 900434725 named "leads")  
      - Columns mapped:  
        - name, website, location, phoneNumber from Loop Over Items current item JSON  
        - contactEmail from `parsedJson.email` (website scrape)  
        - decisionMaker name concatenated from `parsedJson.firstName` and `parsedJson.lastName`  
      - Retries up to 5 times on failure  
    - Inputs: From "name" node  
    - Outputs: To "Wait" node  
    - Edge cases:  
      - Google API limits or authorization errors  
      - Schema mismatches if fields are missing or wrongly formatted  

#### 2.6 Control Flow and Rate Limiting

- **Overview:**  
  Implements a wait step to respect API rate limits and pacing between processing batches of business records.

- **Nodes Involved:**  
  - Wait

- **Node Details:**

  - **Wait**  
    - Type: Wait  
    - Role: Adds delay after appending each lead to avoid rate limiting or excessive scraping speed.  
    - Configuration: Default wait parameters (duration not explicitly set in given data, implying default or manual configuration)  
    - Inputs: From "Append row in sheet" node  
    - Outputs: Back to "Loop Over Items" node to continue processing next batch  
    - Edge cases: Excessive waiting can slow down the workflow; insufficient waiting risks rate limits or API blocking.

---

### 3. Summary Table

| Node Name          | Node Type           | Functional Role                         | Input Node(s)        | Output Node(s)         | Sticky Note                                                                                                   |
|--------------------|---------------------|---------------------------------------|----------------------|------------------------|--------------------------------------------------------------------------------------------------------------|
| On form submission  | Form Trigger        | Entry point: capture city and business| None                 | scrape information     | ## Form Trigger  Enter city + business type to begin scraping.                                              |
| scrape information  | HTTP Request        | Scrapes Google Maps via Olostep API   | On form submission   | parsedInfo             | ## Google Maps Scrape  Extracts business name, location, website, phone number.                              |
| parsedInfo          | Set                 | Cleans raw JSON string                 | scrape information   | Split Out              | ## Clean Data  Split JSON → Remove duplicates → Loop through businesses.                                     |
| Split Out          | Split Out            | Splits business array into items      | parsedInfo           | Remove Duplicates       | ## Clean Data  Split JSON → Remove duplicates → Loop through businesses.                                     |
| Remove Duplicates   | Remove Duplicates   | Removes duplicate business entries    | Split Out            | Loop Over Items         | ## Clean Data  Split JSON → Remove duplicates → Loop through businesses.                                     |
| Loop Over Items     | Split In Batches    | Iterates over each business for scraping | Remove Duplicates   | scrape the name (2nd output), (1st output empty) | ## Website Scrape  Scrapes each business's website to find CEO / Founder info.                               |
| scrape the name     | HTTP Request        | Scrapes business website for decision-maker info | Loop Over Items | name                   | ## Website Scrape  Scrapes each business's website to find CEO / Founder info.                               |
| name               | Set                 | Cleans JSON string from website scrape| scrape the name      | Append row in sheet     | ## Clean Data  set first name, last name, and email each in its proper field.                                |
| Append row in sheet | Google Sheets       | Appends enriched lead data to sheet   | name                 | Wait                   | ## Save Lead  Appends results to Google Sheets with all captured fields.                                     |
| Wait               | Wait                | Adds delay to control scraping rate   | Append row in sheet  | Loop Over Items         |                                                                                                              |
| Sticky Note         | Sticky Note         | Documentation notes                   | None                 | None                   | # Olostep Google Maps Lead Generation Automation  Explains workflow steps and setup instructions.            |
| Sticky Note1        | Sticky Note         | Documentation notes                   | None                 | None                   | ## Google Maps Scrape  Extracts business name, location, website, phone number.                              |
| Sticky Note2        | Sticky Note         | Documentation notes                   | None                 | None                   | ## Clean Data  Split JSON → Remove duplicates → Loop through businesses.                                     |
| Sticky Note3        | Sticky Note         | Documentation notes                   | None                 | None                   | ## Website Scrape  Scrapes each business's website to find CEO / Founder info.                               |
| Sticky Note4        | Sticky Note         | Documentation notes                   | None                 | None                   | ## Save Lead  Appends results to Google Sheets with all captured fields.                                     |
| Sticky Note5        | Sticky Note         | Documentation notes                   | None                 | None                   | ## Clean Data  set first name, last name, and email each in it's proper field.                               |
| Sticky Note6        | Sticky Note         | Documentation notes                   | None                 | None                   | # Olostep Google Maps Lead Generation Automation  Full overview with setup and usage instructions.           |
| Sticky Note7        | Sticky Note         | Warning about 504 Gateway Timeout    | None                 | None                   | ## WARNING  If http request runs into 504 timeout, consider removing `llm_extract` schema or use alternative extraction. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Type: Form Trigger  
   - Name: "On form submission"  
   - Settings:  
     - Form Title: "Olostep lead generation"  
     - Fields:  
       - City (required, placeholder "London")  
       - business (required, placeholder "Dentist")  
     - Form Description: "enter the required fields to start!"  
   - This node is the entry point triggered by form submissions.

2. **Create HTTP Request Node for Google Maps Scrape**  
   - Type: HTTP Request  
   - Name: "scrape information"  
   - Method: POST  
   - URL: https://api.olostep.com/v1/scrapes  
   - Headers: Authorization: Bearer `<your_Olostep_API_token>`  
   - Body Type: JSON  
   - JSON Body:  
     - `url_to_scrape`: `https://www.google.com/maps/search/{{ $json.City }}+{{ $json.business }}` (use expression to insert form inputs)  
     - `wait_before_scraping`: 100000 ms  
     - `formats`: ["json"]  
     - `actions`: series of wait, click, and scroll actions to fully load the map listings  
     - `country`: "US"  
     - `llm_extract`: JSON schema specifying array of businesses with name, website, location, phone number (see schema in original)  
     - Screen size: desktop (1920x1080)  
     - Screenshot: full page  
   - Retries: 5 times on failure  
   - On error: continue  
   - Connect output of "On form submission" to this node.

3. **Create Set Node to Clean JSON**  
   - Type: Set  
   - Name: "parsedInfo"  
   - Assignments:  
     - Field: `parsedJson`  
     - Value: `={{ $json.result.json_content.replace(/\\\\/g, '') }}` (removes escape characters)  
   - Connect output of "scrape information" to this node.

4. **Create Split Out Node**  
   - Type: Split Out  
   - Name: "Split Out"  
   - Field to split out: `parsedJson`  
   - Connect output of "parsedInfo" to this node.

5. **Create Remove Duplicates Node**  
   - Type: Remove Duplicates  
   - Name: "Remove Duplicates"  
   - Default settings  
   - Connect output of "Split Out" to this node.

6. **Create Split In Batches Node**  
   - Type: Split In Batches  
   - Name: "Loop Over Items"  
   - Default batch size: 1 (or as desired)  
   - Connect output of "Remove Duplicates" to this node.

7. **Create HTTP Request Node to Scrape Website**  
   - Type: HTTP Request  
   - Name: "scrape the name"  
   - Method: POST  
   - URL: https://api.olostep.com/v1/scrapes  
   - Headers: Authorization: Bearer `<your_Olostep_API_token>`  
   - Body Type: JSON  
   - JSON Body:  
     - `url_to_scrape`: `{{ $json.website }}` (from current batch item)  
     - `wait_before_scraping`: 10000 ms  
     - `formats`: ["json"]  
     - `llm_extract`: JSON schema for object with `firstName`, `lastName`, and optional `email` fields  
     - Screen size: default (small resolution)  
     - Screenshot: full page  
   - Retries: 5 times on failure  
   - On error: continue  
   - Connect second output of "Loop Over Items" node to this node.

8. **Create Set Node to Clean Website Scrape JSON**  
   - Type: Set  
   - Name: "name"  
   - Assignments:  
     - Field: `parsedJson`  
     - Value: `={{ $json.result.json_content.replace(/\\\\/g, '') }}`  
   - Connect output of "scrape the name" to this node.

9. **Create Google Sheets Node to Append Data**  
   - Type: Google Sheets  
   - Name: "Append row in sheet"  
   - Operation: Append  
   - Document ID: `1IoiN3ISmNrxOOWcfw1bQQb6Ru2GV0aN4wbXxGA73LCY` (replace with your Google Sheet ID)  
   - Sheet Name/ID: Use sheet/tab ID 900434725 named "leads"  
   - Columns Mapping:  
     - name: `={{ $('Loop Over Items').item.json.name || "not found" }}`  
     - website: `={{ $('Loop Over Items').item.json.website || "not found" }}`  
     - location: `={{ $('Loop Over Items').item.json.location || "location" }}`  
     - phoneNumber: `="{{ $('Loop Over Items').item.json.phone_number || "not found" }}"`  
     - contactEmail: `={{ $json.parsedJson.email || "not found" }}`  
     - decisionMaker name: `={{ $('name').item.json.parsedJson.firstName || "not found" }} {{ $('name').item.json.parsedJson.lastName || "" }}`  
   - Connect output of "name" node to this node.

10. **Create Wait Node**  
    - Type: Wait  
    - Name: "Wait"  
    - Use default wait settings or configure delay as necessary to avoid rate limits (e.g., a few seconds)  
    - Connect output of "Append row in sheet" to this node.

11. **Connect Wait Node to Loop Over Items**  
    - This closes the loop by connecting "Wait" output back to "Loop Over Items" input, enabling batch processing.

12. **Add Sticky Notes for Documentation**  
    - Add notes describing each functional block and any warnings, such as the 504 timeout warning related to `llm_extract` schema in the HTTP Request nodes.

13. **Credentials Setup**  
    - Olostep API Key: Required for both HTTP Request nodes (Authorization Bearer token header)  
    - Google Sheets OAuth2 Credentials: Required for Google Sheets node access.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                     |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| # Olostep Google Maps Lead Generation Automation  This n8n template automates lead generation by scraping Google Maps using the Olostep API. It extracts business names, locations, websites, phone numbers, and decision-maker names, then saves everything into a Google Sheet. Detailed explanation and setup instructions are included in sticky notes. | See Sticky Note6 in the workflow.                                                                  |
| WARNING: If the HTTP request returns a 504 Gateway Timeout error, it is likely due to the `llm_extract` field in the API request. To resolve, remove the schema fields inside `llm_extract` and consider using a separate information extraction node or LLM node to process the extracted data afterward.                 | See Sticky Note7 in the workflow.                                                                  |
| Google Sheets Integration requires appropriate OAuth2 credentials with write access to the target spreadsheet.                                                                                                                                                                                                       | Google Sheets node setup.                                                                          |
| Olostep API usage requires a valid API key. API rate limits and quotas should be monitored. The workflow includes retries and wait steps to mitigate this.                                                                                                                                                           | Olostep API documentation and user account.                                                       |
| The form trigger facilitates dynamic lead generation queries without modifying workflow code, enabling non-technical users to initiate targeted scrapes by specifying city and business type.                                                                                                                       | Form Trigger node configuration.                                                                  |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.