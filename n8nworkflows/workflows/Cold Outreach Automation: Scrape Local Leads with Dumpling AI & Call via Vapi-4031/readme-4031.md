Cold Outreach Automation: Scrape Local Leads with Dumpling AI & Call via Vapi

https://n8nworkflows.xyz/workflows/cold-outreach-automation--scrape-local-leads-with-dumpling-ai---call-via-vapi-4031


# Cold Outreach Automation: Scrape Local Leads with Dumpling AI & Call via Vapi

### 1. Workflow Overview

This workflow automates cold outreach to local businesses by scraping Google Maps listings using Dumpling AI and initiating AI-powered calls via Vapi. It is designed to help sales teams, agencies, and local service providers rapidly generate call lists and automate outbound calls without manual dialing or lead research.

The workflow is logically divided into two main blocks:

**1.1 Lead Generation and Preparation**  
- Input: Search queries from Google Sheets (e.g., “plumbers in Austin”)  
- Actions: Send queries to Dumpling AI to scrape Google Maps, split the results, extract business details, and filter for valid phone numbers  

**1.2 Automated Calling and Logging**  
- Input: Valid business records with properly formatted phone numbers  
- Actions: Format phone numbers for dialing, initiate AI calls via Vapi, and log call details to Google Sheets for tracking  

This structure ensures seamless data flow from lead sourcing to outreach and record-keeping.

---

### 2. Block-by-Block Analysis

#### 2.1 Lead Generation and Preparation

**Overview:**  
This block collects local business search queries from a Google Sheet, sends each query to Dumpling AI’s Google Maps scraping API, splits the returned business listings into individual leads, and extracts key details such as business name, phone number, and website. It then filters out any leads lacking a valid phone number.

**Nodes Involved:**  
- Start Workflow Manually  
- Get Search Keywords from Google Sheets  
- Scrape Google Map Businesses using Dumpling AI  
- Split Each Business Result  
- Extract Business Name, Phone and website  
- Filter Valid Phone Numbers Only  

**Node Details:**  

- **Start Workflow Manually**  
  - Type: Manual Trigger  
  - Role: Initiates the workflow manually to start the automation process.  
  - Config: No parameters; triggers execution on user action.  
  - Inputs: None  
  - Outputs: To `Get Search Keywords from Google Sheets`  
  - Potential Failures: None typical, unless execution environment issues occur.

- **Get Search Keywords from Google Sheets**  
  - Type: Google Sheets  
  - Role: Reads search query phrases from a configured Google Sheet tab.  
  - Config:  
    - Reads from a specified spreadsheet ID and sheet tab (e.g., "Sheet1").  
    - Requires Google Sheets OAuth2 credentials connected.  
  - Key Variables: Pulls queries typically in the first column, e.g., "best+restaurants+in+Chicago".  
  - Inputs: From `Start Workflow Manually`  
  - Outputs: To `Scrape Google Map Businesses using Dumpling AI`  
  - Edge Cases: Empty or malformed queries; API or auth errors with Google Sheets; rate limits.

- **Scrape Google Map Businesses using Dumpling AI**  
  - Type: HTTP Request  
  - Role: Sends each query to Dumpling AI’s `search-maps` API endpoint to scrape business data from Google Maps.  
  - Config:  
    - POST request with JSON body containing the query string (interpolated from sheet data) and language "en".  
    - Authentication via API key passed as HTTP header (configured in credentials).  
  - Key Expressions: `"query":"{{ $json.URLs }}"` dynamically inserts the search query from previous node.  
  - Inputs: From `Get Search Keywords from Google Sheets`  
  - Outputs: To `Split Each Business Result`  
  - Edge Cases: API authentication failure, network timeouts, empty or partial results, rate limits.

- **Split Each Business Result**  
  - Type: Split Out  
  - Role: Splits the array of businesses returned by Dumpling AI into individual items to process each lead separately.  
  - Config: Splits on the `places` field from the API response.  
  - Inputs: From `Scrape Google Map Businesses using Dumpling AI`  
  - Outputs: To `Extract Business Name, Phone and website`  
  - Edge Cases: Empty or missing `places` field leading to no outputs.

- **Extract Business Name, Phone and website**  
  - Type: Set  
  - Role: Extracts and standardizes key fields from each business entry: `title` (business name), `phoneNumber`, and `website`.  
  - Config: Assigns JSON fields to new, consistent variable names for downstream use.  
  - Key Expressions:  
    - `website` = `{{$json.website}}`  
    - `phoneNumber` = `{{$json.phoneNumber}}`  
    - `title` = `{{$json.title}}`  
  - Inputs: From `Split Each Business Result`  
  - Outputs: To `Filter Valid Phone Numbers Only`  
  - Edge Cases: Missing or malformed phone/website fields.

- **Filter Valid Phone Numbers Only**  
  - Type: Filter  
  - Role: Filters out any business records without a phone number to ensure only callable leads proceed.  
  - Config: Condition checks existence (`exists`) of `phoneNumber` field.  
  - Inputs: From `Extract Business Name, Phone and website`  
  - Outputs: To `Format Phone Number for Calling`  
  - Edge Cases: Businesses missing phone numbers are excluded; filter could exclude valid leads if phone data is incorrectly formatted.

---

#### 2.2 Automated Calling and Logging

**Overview:**  
This block formats phone numbers for US dialing, initiates AI calls using Vapi’s API with the business name passed as a variable for personalized scripting, and logs each successful call’s details into a Google Sheet tab for tracking outreach results and follow-up.

**Nodes Involved:**  
- Format Phone Number for Calling  
- Initiate Vapi AI Call to Business  
- Log Called Business Info to Sheet  

**Node Details:**  

- **Format Phone Number for Calling**  
  - Type: Set  
  - Role: Prepares phone numbers for dialing by formatting them to E.164 style with +1 country code for US numbers.  
  - Config: Uses JavaScript expression to remove all non-digit characters and prepend "+1".  
  - Key Expression: `'+1' + $json["phoneNumber"].replace(/\D/g, '')`  
  - Inputs: From `Filter Valid Phone Numbers Only`  
  - Outputs: To `Initiate Vapi AI Call to Business`  
  - Edge Cases: Phone numbers with country codes already present could be misformatted; non-US numbers require adjustment.

- **Initiate Vapi AI Call to Business**  
  - Type: HTTP Request  
  - Role: Sends a POST request to Vapi’s call API to initiate an outbound AI call to the formatted phone number.  
  - Config:  
    - URL: `https://api.vapi.ai/call`  
    - Payload includes:  
      - `customers`: array with `number` (formatted phone) and empty `name` field (business name passed in assistantOverrides)  
      - `assistantId` and `phoneNumberId`: must be filled with user’s Vapi assistant credentials  
      - `assistantOverrides.variableValues.Name`: dynamically set to business title for personalized call scripts  
    - Authenticated with Vapi API key via HTTP Header Auth credentials  
  - Inputs: From `Format Phone Number for Calling`  
  - Outputs: To `Log Called Business Info to Sheet`  
  - On Error: Continues execution even if call initiation fails (important for bulk processing)  
  - Edge Cases: API key invalid, network issues, incorrect assistant or phone number IDs, malformed payload, rate limiting.

- **Log Called Business Info to Sheet**  
  - Type: Google Sheets  
  - Role: Appends the called business details (company name, formatted phone number, website) into a configured Google Sheet tab for tracking outreach.  
  - Config:  
    - Uses append operation on a specified sheet tab (e.g., “leads”) within the same spreadsheet as keywords.  
    - Maps fields from previous nodes: `company name` from business title, `phone number` from formatted phone, `website` from extracted data.  
    - Requires Google Sheets OAuth2 credentials.  
  - Inputs: From `Initiate Vapi AI Call to Business`  
  - Outputs: None (end of workflow)  
  - Edge Cases: API auth failure, sheet access issues, rate limiting.

---

### 3. Summary Table

| Node Name                              | Node Type                 | Functional Role                                      | Input Node(s)                         | Output Node(s)                             | Sticky Note                                                                                  |
|--------------------------------------|---------------------------|-----------------------------------------------------|-------------------------------------|--------------------------------------------|----------------------------------------------------------------------------------------------|
| Start Workflow Manually               | Manual Trigger            | Initiates the workflow manually for controlled runs | None                                | Get Search Keywords from Google Sheets     |                                                                                              |
| Get Search Keywords from Google Sheets| Google Sheets             | Reads search queries from Google Sheet               | Start Workflow Manually             | Scrape Google Map Businesses using Dumpling AI |                                                                                              |
| Scrape Google Map Businesses using Dumpling AI | HTTP Request             | Scrapes business data from Google Maps via Dumpling AI | Get Search Keywords from Google Sheets | Split Each Business Result                   |                                                                                              |
| Split Each Business Result            | Split Out                 | Splits array of businesses into individual records   | Scrape Google Map Businesses using Dumpling AI | Extract Business Name, Phone and website    |                                                                                              |
| Extract Business Name, Phone and website | Set                       | Extracts and standardizes business info fields       | Split Each Business Result          | Filter Valid Phone Numbers Only             |                                                                                              |
| Filter Valid Phone Numbers Only       | Filter                    | Filters leads to keep only those with phone numbers  | Extract Business Name, Phone and website | Format Phone Number for Calling               |                                                                                              |
| Format Phone Number for Calling       | Set                       | Formats phone numbers for US dialing (+1 prefix)     | Filter Valid Phone Numbers Only     | Initiate Vapi AI Call to Business           |                                                                                              |
| Initiate Vapi AI Call to Business     | HTTP Request              | Initiates AI calls via Vapi using formatted numbers  | Format Phone Number for Calling     | Log Called Business Info to Sheet            |                                                                                              |
| Log Called Business Info to Sheet     | Google Sheets             | Logs call details to Google Sheet for tracking       | Initiate Vapi AI Call to Business   | None                                       |                                                                                              |
| Sticky Note                          | Sticky Note               | Describes lead scraping block                         | None                                | None                                       | "### 📍 Get Local Business Data from Google Maps with Dumpling AI..."                       |
| Sticky Note1                         | Sticky Note               | Describes calling and logging block                   | None                                | None                                       | "### 📞 Auto-Call Businesses and Track Results..."                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Manual Trigger node:**  
   - Name: `Start Workflow Manually`  
   - Purpose: To manually start the workflow.

3. **Add a Google Sheets node:**  
   - Name: `Get Search Keywords from Google Sheets`  
   - Operation: Read rows  
   - Configure credentials with your Google Sheets OAuth2 account.  
   - Set Document ID to your spreadsheet containing search queries.  
   - Set Sheet Name (tab) to the one with queries (e.g., "Sheet1").  
   - Ensure first column contains search queries like "plumbers in Austin".

4. **Connect** `Start Workflow Manually` → `Get Search Keywords from Google Sheets`.

5. **Add an HTTP Request node:**  
   - Name: `Scrape Google Map Businesses using Dumpling AI`  
   - Method: POST  
   - URL: `https://app.dumplingai.com/api/v1/search-maps`  
   - Authentication: HTTP Header Auth with your Dumpling AI API key.  
   - Body Content Type: JSON  
   - Body Parameters:  
     ```json
     {
       "query": "={{ $json.URLs }}",
       "language": "en"
     }
     ```  
   - Connect `Get Search Keywords from Google Sheets` → `Scrape Google Map Businesses using Dumpling AI`.

6. **Add a Split Out node:**  
   - Name: `Split Each Business Result`  
   - Field to split: `places` (the array of businesses in Dumpling AI response)  
   - Connect `Scrape Google Map Businesses using Dumpling AI` → `Split Each Business Result`.

7. **Add a Set node:**  
   - Name: `Extract Business Name, Phone and website`  
   - Assign:  
     - `title` = `{{$json.title}}`  
     - `phoneNumber` = `{{$json.phoneNumber}}`  
     - `website` = `{{$json.website}}`  
   - Connect `Split Each Business Result` → `Extract Business Name, Phone and website`.

8. **Add a Filter node:**  
   - Name: `Filter Valid Phone Numbers Only`  
   - Condition: Check that `phoneNumber` exists (non-empty)  
   - Connect `Extract Business Name, Phone and website` → `Filter Valid Phone Numbers Only`.

9. **Add a Set node for formatting phone number:**  
   - Name: `Format Phone Number for Calling`  
   - Assign a new field `formattedPhone` with expression:  
     ```js
     '+1' + $json["phoneNumber"].replace(/\D/g, '')
     ```  
   - Connect `Filter Valid Phone Numbers Only` → `Format Phone Number for Calling`.

10. **Add an HTTP Request node for Vapi call:**  
    - Name: `Initiate Vapi AI Call to Business`  
    - Method: POST  
    - URL: `https://api.vapi.ai/call`  
    - Authentication: HTTP Header Auth with your Vapi API Key  
    - Body Content Type: JSON  
    - JSON Body:  
      ```json
      {
        "customers": [
          {
            "number": "{{ $json.formattedPhone }}",
            "name": ""
          }
        ],
        "assistantId": "<YOUR_ASSISTANT_ID>",
        "phoneNumberId": "<YOUR_PHONE_NUMBER_ID>",
        "assistantOverrides": {
          "variableValues": {
            "Name": "{{ $('Filter Valid Phone Numbers Only').item.json.title }}"
          }
        }
      }
      ```  
    - Configure error handling to continue on failure (to avoid interrupting bulk calls).  
    - Connect `Format Phone Number for Calling` → `Initiate Vapi AI Call to Business`.

11. **Add a Google Sheets node to log calls:**  
    - Name: `Log Called Business Info to Sheet`  
    - Operation: Append  
    - Configure with your Google Sheets credentials.  
    - Document ID: same as initial sheet or another for logs.  
    - Sheet Name: Tab for logging calls (e.g., “leads”).  
    - Map columns:  
      - `company name` = `{{$node["Extract Business Name, Phone and website"].json.title}}`  
      - `phone number` = `{{$node["Format Phone Number for Calling"].json.formattedPhone}}`  
      - `website` = `{{$node["Extract Business Name, Phone and website"].json.website}}`  
    - Connect `Initiate Vapi AI Call to Business` → `Log Called Business Info to Sheet`.

12. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Dumpling AI: Use [https://www.dumplingai.com/](https://www.dumplingai.com/) to get your API key           | API Key needed for scraping Google Maps business data                                              |
| Vapi: Sign up and create an assistant at [https://vapi.ai](https://vapi.ai)                                | Retrieve assistantId, phoneNumberId, and API key for AI calling                                    |
| Google Sheets Setup: Prepare sheets with search keywords and a separate leads tab for logging calls       | Refer to workflow description for column headers and sheet naming conventions                      |
| Workflow optimized for US leads (+1 country code); adjust formatting logic for international use           | Modify `Format Phone Number for Calling` node's expression accordingly                              |
| Handle API rate limits from Dumpling AI, Google Sheets, and Vapi by scheduling or chunking large batches  | Important for scaling outreach without interruption                                                |

---

**Disclaimer:**  
The provided text is derived exclusively from an automated workflow created using n8n, adhering strictly to content policies and containing no illegal or protected material. All data processed is public and lawful.