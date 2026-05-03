Extract Google My Business Leads by Service or Location with Bright Data

https://n8nworkflows.xyz/workflows/extract-google-my-business-leads-by-service-or-location-with-bright-data-4555


# Extract Google My Business Leads by Service or Location with Bright Data

---

### 1. Workflow Overview

This workflow automates the extraction of Google My Business leads filtered by service type and geographic location, leveraging Bright Data's Google Maps dataset. It is designed for users who want to collect comprehensive business listings by specifying a service, state, and country, then dynamically generating city-level queries, scraping data, removing duplicates, and saving the clean data into Google Sheets.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Validation**: Receives user input via a form for service, state, and country.
- **1.2 AI-Assisted Location and Category Processing**: Uses AI models to generate a list of cities within the specified region and categorize the service type.
- **1.3 City-wise Data Preparation**: Splits the city list and prepares structured data items for each city.
- **1.4 Bright Data Scraping and Monitoring**: Submits scraping requests to Bright Data, monitors scraping progress, handles rate limiting, and fetches results.
- **1.5 Duplicate Detection and Removal**: Retrieves existing data from Google Sheets, identifies duplicates, and removes them to maintain clean records.
- **1.6 Data Persistence**: Appends newly scraped, deduplicated business data into Google Sheets.
- **1.7 Notes and Monitoring**: Includes sticky notes for documentation and operational guidance.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception and Validation

**Overview:**  
This block captures the user inputs via a form, requiring service name, state, and country to initiate data extraction.

**Nodes Involved:**  
- Form Submission Trigger  
- Workflow Description (Sticky Note)

**Node Details:**

- **Form Submission Trigger**  
  - *Type:* Form Trigger node  
  - *Role:* Entry point that triggers the workflow when a user submits the form.  
  - *Configuration:*  
    - Form titled "Extract Business Data Using Service Name and Location" with fields:  
      - Service (text, required, e.g., "Laptop Store")  
      - State (text, required, e.g., "Texas")  
      - Country (dropdown, required, options: US, India)  
    - Form description: "Fill details to extract business leads"  
  - *Inputs:* None (trigger node)  
  - *Outputs:* JSON containing user input fields  
  - *Edge Cases:*  
    - Missing required fields triggers validation errors.  
    - Malformed or unexpected input values might cause downstream errors.

- **Workflow Description (Sticky Note)**  
  - Provides a high-level textual description of the workflow purpose and functionality for user reference.

---

#### 2.2 AI-Assisted Location and Category Processing

**Overview:**  
Uses AI models to generate a list of cities within the specified state and country and categorize the service to an appropriate business category for targeted scraping.

**Nodes Involved:**  
- Claude AI Model for Cities  
- Generate City List (Langchain agent)  
- Claude AI Model for Categories  
- Categorize Service Type (Langchain agent)

**Node Details:**

- **Claude AI Model for Cities**  
  - *Type:* Langchain Anthropic LM Chat node  
  - *Role:* Generates AI-based suggestions or assists in city list generation (used as an AI model reference by the next node)  
  - *Configuration:* Model "claude-sonnet-4-20250514"  
  - *Inputs:* Output from form trigger (service/state/country)  
  - *Outputs:* AI model response for city-related queries  
  - *Edge Cases:* API authentication issues, rate limits, or unexpected AI response format.

- **Generate City List**  
  - *Type:* Langchain agent node  
  - *Role:* Requests a plain text list of cities or sub-areas within the specified state and country to be used for Google Maps searches.  
  - *Configuration:*  
    - Prompt uses dynamic expressions for state and country from form input.  
    - Output is plain text, no formatting, just city names separated by new lines.  
  - *Inputs:* Form submission data, AI model assistance  
  - *Outputs:* Plain text city list string  
  - *Edge Cases:* AI returning formatted text, empty output, or irrelevant city names.

- **Claude AI Model for Categories**  
  - *Type:* Langchain Anthropic LM Chat node  
  - *Role:* Provides AI capabilities for service categorization.  
  - *Configuration:* Same model as above, "claude-sonnet-4-20250514".  
  - *Inputs:* Service name from form  
  - *Outputs:* Category suggestions  
  - *Edge Cases:* Invalid or ambiguous service names may cause uncertain categories.

- **Categorize Service Type**  
  - *Type:* Langchain agent node  
  - *Role:* Determines the best fitting category name for the given service input.  
  - *Configuration:* Prompt instructs output to be a single category name from a suggested set (e.g., Electronics, Healthcare).  
  - *Inputs:* Service name from form submission  
  - *Outputs:* Single category string  
  - *Edge Cases:* Service name not matching expected categories leading to incorrect or generic categories.

---

#### 2.3 City-wise Data Preparation

**Overview:**  
This block splits the AI-generated city list into individual city entries and structures data per city for scraping.

**Nodes Involved:**  
- Separate Data by City (Code node)  
- City Processing Note (Sticky Note)  
- Process Cities in Batches (SplitInBatches node)

**Node Details:**

- **Separate Data by City**  
  - *Type:* Function Code node (JavaScript)  
  - *Role:* Parses the city list string into an array of city names, then outputs an item per city with structured JSON including service name, state, country, city, and category.  
  - *Configuration:*  
    - Reads variables from form input and generated city list.  
    - Throws error if any required field missing.  
    - Trims and filters empty city entries.  
  - *Inputs:* Output from Generate City List and Categorize Service Type  
  - *Outputs:* Multiple JSON items, each representing one city search context  
  - *Edge Cases:* Missing or empty city list, malformed text input, or runtime errors in JS code.

- **City Processing Note (Sticky Note)**  
  - Instructional note: "Separate each search by city name for comprehensive coverage."

- **Process Cities in Batches**  
  - *Type:* SplitInBatches node  
  - *Role:* Processes city search items in manageable batches to avoid API overload or rate limits.  
  - *Inputs:* Output from Separate Data by City  
  - *Outputs:* Batches of city items for downstream processing  
  - *Edge Cases:* Batch size defaults to 1 (default), potential performance issues if batch size not tuned.

---

#### 2.4 Bright Data Scraping and Monitoring

**Overview:**  
Submits scraping tasks to Bright Data, monitors scraping progress through polling, handles rate limiting, and fetches the scraped Google Maps business data.

**Nodes Involved:**  
- Get Existing Business Data (Google Sheets)  
- Scrape Business Data from Google Maps (Bright Data node)  
- Check Scraping Status (If node)  
- Check Data Collection Status (HTTP Request node)  
- Wait for Rate Limiting (Wait node)  
- Check if Data Ready (If node)  
- Fetch Scraped Data (HTTP Request node)  
- Data Collection Notes (Sticky Note)

**Node Details:**

- **Get Existing Business Data**  
  - *Type:* Google Sheets node  
  - *Role:* Reads existing business leads from a specified Google Sheet to compare against new data and identify duplicates.  
  - *Configuration:*  
    - Reads from sheet "gid=0" in the specified Google Sheet document.  
    - Uses OAuth2 credentials.  
  - *Inputs:* From Process Cities in Batches (batch trigger)  
  - *Outputs:* Existing business data items  
  - *Edge Cases:* Google Sheets API quota, invalid credentials, sheet access issues.

- **Scrape Business Data from Google Maps**  
  - *Type:* Bright Data scraping node  
  - *Role:* Sends scraping requests to Bright Data's Google Maps dataset API with URLs dynamically constructed from city, state, and service category.  
  - *Configuration:*  
    - URL format: `https://www.google.com/maps/search/{{ serviceName }}+in+{{ city }} {{ state }}`  
    - Dataset ID: "gd_lh0tnzlo2bie4uhdhr" (Google Maps businesses)  
    - Proxy: empty (optional)  
    - Credentials: Bright Data API key required  
  - *Inputs:* Batch city data from Process Cities in Batches  
  - *Outputs:* Scraping job initiation response  
  - *Edge Cases:* API key invalid, rate limiting, network errors, malformed URLs.

- **Check Scraping Status**  
  - *Type:* If node  
  - *Role:* Checks if the scraping response contains a "message" property to determine if scraping request was accepted or if errors occurred.  
  - *Inputs:* Output from Scrape Business Data from Google Maps  
  - *Outputs:* Branches: If message exists, continue monitoring; else, save data.

- **Check Data Collection Status**  
  - *Type:* HTTP Request node  
  - *Role:* Polls Bright Data API for scraping progress using the snapshot ID returned by the scraping job.  
  - *Configuration:*  
    - GET request to `https://api.brightdata.com/datasets/v3/progress/{{ snapshot_id }}`  
    - Uses Bright Data HTTP Header Auth credentials  
  - *Inputs:* From Check Scraping Status node or Check if Data Ready node  
  - *Outputs:* Scraping progress JSON  
  - *Edge Cases:* API rate limits, snapshot ID invalid, network issues.

- **Wait for Rate Limiting**  
  - *Type:* Wait node  
  - *Role:* Introduces a delay (25 seconds) between successive polling attempts to avoid API rate limits.  
  - *Inputs:* From Check Data Collection Status  
  - *Outputs:* Delayed trigger to Check if Data Ready node  
  - *Edge Cases:* Workflow pause duration too short or too long affecting responsiveness.

- **Check if Data Ready**  
  - *Type:* If node  
  - *Role:* Determines if scraping status is "ready" to proceed fetching data or continue polling.  
  - *Inputs:* Output from Wait for Rate Limiting  
  - *Outputs:* If ready, proceeds to Fetch Scraped Data; else, loops back to Check Data Collection Status  
  - *Edge Cases:* Status field missing or API returning unexpected status.

- **Fetch Scraped Data**  
  - *Type:* HTTP Request node  
  - *Role:* Retrieves the final scraped dataset in JSON format from Bright Data using snapshot ID.  
  - *Configuration:*  
    - GET request to `https://api.brightdata.com/datasets/v3/snapshot/{{ snapshot_id }}` with query parameter `format=json`  
    - Uses Bright Data HTTP Header Auth credentials  
  - *Inputs:* From Check if Data Ready node when data is ready  
  - *Outputs:* Final scraped business listings JSON data  
  - *Edge Cases:* Snapshot expired, network issues, malformed response.

- **Data Collection Notes (Sticky Note)**  
  - Documents the polling and data retrieval logic for maintainers.

---

#### 2.5 Duplicate Detection and Removal

**Overview:**  
This block compares newly scraped data against existing Google Sheets entries to detect and delete duplicates to maintain data quality.

**Nodes Involved:**  
- Check for Duplicate Entries (Code node)  
- Process Duplicates in Batches (SplitInBatches node)  
- Find Duplicate Row Number (Google Sheets node)  
- Delete Duplicate Row (Google Sheets node)  
- Duplicate Removal Notes (Sticky Note)

**Node Details:**

- **Check for Duplicate Entries**  
  - *Type:* Function Code node  
  - *Role:* Iterates over combined existing and new data, identifies duplicates by business name (case-insensitive) and phone number matches, and outputs duplicates for removal.  
  - *Logic:*  
    - Uses a Map to track seen business names and phone numbers.  
    - Marks items as duplicate if business name already seen and phone number matches or is missing.  
  - *Inputs:* Output from Get Existing Business Data and new scraped data  
  - *Outputs:* Items identified as duplicates (containing Address and Google Maps URL)  
  - *Edge Cases:* Missing business name or phone number can affect duplicate detection accuracy.

- **Process Duplicates in Batches**  
  - *Type:* SplitInBatches node  
  - *Role:* Processes duplicates in batches to safely query and delete duplicates from Google Sheets.  
  - *Inputs:* Output from Check for Duplicate Entries  
  - *Outputs:* Batches for downstream lookup and deletion

- **Find Duplicate Row Number**  
  - *Type:* Google Sheets node  
  - *Role:* Looks up the row number of the duplicate entry in Google Sheets by matching Address and Google Maps URL.  
  - *Configuration:*  
    - Uses filters on Address and Google Maps URL columns.  
    - Returns the first matching row number.  
  - *Inputs:* From Process Duplicates in Batches  
  - *Outputs:* Row number for deletion  
  - *Edge Cases:* No matching row found, multiple matches, or API errors.

- **Delete Duplicate Row**  
  - *Type:* Google Sheets node  
  - *Role:* Deletes the identified duplicate row from the Google Sheet using the row number.  
  - *Configuration:*  
    - Deletes row at index provided by Find Duplicate Row Number.  
  - *Inputs:* From Find Duplicate Row Number  
  - *Outputs:* Confirmation of deletion  
  - *Edge Cases:* Attempting to delete non-existent rows, permission errors.

- **Duplicate Removal Notes (Sticky Note)**  
  - Explains the logic of duplicate identification and removal.

---

#### 2.6 Data Persistence

**Overview:**  
Appends the final cleaned business data into the Google Sheets document for storage and further use.

**Nodes Involved:**  
- Save Business Data to Sheet (Google Sheets node)

**Node Details:**

- **Save Business Data to Sheet**  
  - *Type:* Google Sheets node  
  - *Role:* Appends new business data rows into the specified Google Sheet.  
  - *Configuration:*  
    - Columns mapped explicitly: Business Name, Website URL, Category, Country, City, Phone Number, Google Maps URL, Address, Operating Hours, Google Rating, Total Reviews, Reviews URL  
    - Uses OAuth2 credentials  
    - Appends data to sheet "gid=0" in the target document  
  - *Inputs:* Final JSON business data from Fetch Scraped Data or Check Scraping Status (if no error)  
  - *Outputs:* Confirmation of data append  
  - *Edge Cases:* API limits, invalid data causing append failures.

---

#### 2.7 Notes and Monitoring

**Overview:**  
Sticky notes provide contextual information and usage instructions to operators and maintainers.

**Nodes Involved:**  
- Workflow Description  
- City Processing Note  
- Data Collection Notes  
- Duplicate Removal Notes

**Node Details:**  
- These are documentation nodes with no inputs or outputs, used purely for annotation.

---

### 3. Summary Table

| Node Name                        | Node Type                                 | Functional Role                           | Input Node(s)                    | Output Node(s)                       | Sticky Note                                                                                     |
|---------------------------------|-------------------------------------------|-----------------------------------------|---------------------------------|------------------------------------|------------------------------------------------------------------------------------------------|
| Form Submission Trigger          | Form Trigger                              | Entry form input collection              | None                            | Generate City List                  |                                                                                                |
| Workflow Description             | Sticky Note                              | Workflow purpose and overview            | None                            | None                              | ## Extract Google My Business Leads by Service and Location<br>Extract Google business listings by service type and geographic location using Bright Data's Google Maps dataset. Automatically processes cities, removes duplicates, and saves results to Google Sheets. |
| Generate City List               | Langchain Agent                          | Generate list of cities in region        | Form Submission Trigger, Claude AI Model for Cities | Categorize Service Type            |                                                                                                |
| Claude AI Model for Cities       | Langchain LM Chat Anthropic              | AI model assisting city generation       | Form Submission Trigger          | Generate City List                 |                                                                                                |
| Categorize Service Type          | Langchain Agent                          | Determine category for service           | Generate City List, Claude AI Model for Categories | Separate Data by City              |                                                                                                |
| Claude AI Model for Categories   | Langchain LM Chat Anthropic              | AI model assisting category classification | Form Submission Trigger          | Categorize Service Type           |                                                                                                |
| Separate Data by City            | Code                                    | Parse city list and structure data       | Categorize Service Type, Generate City List | Process Cities in Batches          | Separate each search by city name for comprehensive coverage                                   |
| City Processing Note             | Sticky Note                              | Instructional note for city-based search | None                            | None                              | Separate each search by city name for comprehensive coverage                                   |
| Process Cities in Batches        | SplitInBatches                          | Batch processing per city                 | Separate Data by City            | Get Existing Business Data, Scrape Business Data from Google Maps |                                                                                                |
| Get Existing Business Data       | Google Sheets                           | Fetch existing data for duplicate checking | Process Cities in Batches        | Check for Duplicate Entries        |                                                                                                |
| Scrape Business Data from Google Maps | Bright Data Scraper Node                | Submit Google Maps scraping queries      | Process Cities in Batches        | Check Scraping Status              |                                                                                                |
| Check Scraping Status            | If                                      | Check if scraping request returned message | Scrape Business Data from Google Maps | Check Data Collection Status, Save Business Data to Sheet |                                                                                                |
| Check Data Collection Status     | HTTP Request                            | Poll scraping progress from Bright Data | Check Scraping Status, Check if Data Ready | Wait for Rate Limiting            |                                                                                                |
| Wait for Rate Limiting           | Wait                                    | Delay to avoid rate limiting              | Check Data Collection Status     | Check if Data Ready                |                                                                                                |
| Check if Data Ready              | If                                      | Determine if data scraping is complete   | Wait for Rate Limiting           | Fetch Scraped Data, Check Data Collection Status |                                                                                                |
| Fetch Scraped Data               | HTTP Request                            | Retrieve final scraped dataset            | Check if Data Ready              | Save Business Data to Sheet        |                                                                                                |
| Save Business Data to Sheet      | Google Sheets                           | Append cleaned business data to sheet    | Fetch Scraped Data, Check Scraping Status | Process Cities in Batches          |                                                                                                |
| Check for Duplicate Entries      | Code                                    | Identify duplicates by business name and phone number | Get Existing Business Data       | Process Duplicates in Batches      |                                                                                                |
| Process Duplicates in Batches    | SplitInBatches                          | Batch processing of duplicates            | Check for Duplicate Entries      | Find Duplicate Row Number, (empty batch output) |                                                                                                |
| Find Duplicate Row Number        | Google Sheets                           | Lookup duplicate row number by address and URL | Process Duplicates in Batches    | Delete Duplicate Row               |                                                                                                |
| Delete Duplicate Row             | Google Sheets                           | Delete identified duplicate row           | Find Duplicate Row Number        | Process Duplicates in Batches      |                                                                                                |
| City Processing Note             | Sticky Note                              | Instruction on city data separation       | None                            | None                              | Separate each search by city name for comprehensive coverage                                   |
| Data Collection Notes            | Sticky Note                              | Explains scraping progress monitoring     | None                            | None                              | ## Data Collection Monitoring<br>Monitors Bright Data scraping progress and fetches results when ready<br>Polls the API until scraping completes, then retrieves the final dataset |
| Duplicate Removal Notes          | Sticky Note                              | Explains duplicate removal process        | None                            | None                              | ## Duplicate Removal Process<br>Identifies and removes duplicate business entries<br>Compares business names and phone numbers to eliminate redundant data |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Submission Trigger node:**  
   - Type: Form Trigger  
   - Title: "Extract Business Data Using Service Name and Location"  
   - Fields:  
     - Service (text, required, placeholder "Laptop Store")  
     - State (text, required, placeholder "Texas")  
     - Country (dropdown, required) with options: US, India  
   - Description: "Fill details to extract business leads"  
   - Save and activate webhook.

2. **Add a Sticky Note for Workflow Description:**  
   - Content:  
     "## Extract Google My Business Leads by Service and Location  
      Extract Google business listings by service type and geographic location using Bright Data's Google Maps dataset. Automatically processes cities, removes duplicates, and saves results to Google Sheets."

3. **Add Claude AI Model for Cities node:**  
   - Type: Langchain LM Chat (Anthropic)  
   - Model: "claude-sonnet-4-20250514"  
   - Set API credentials for Anthropic API.

4. **Add Generate City List node:**  
   - Type: Langchain Agent  
   - Prompt:  
     "Provide a list of cities or sub areas located inside the Location of {{ $json.State }}, within the country {{ $json.Country }}. This data will be used to search on Google maps for businesses in that region. The output should be a plain text list, without bullets, numbering, or any special characters. Do not include any introduction, explanation, or concluding text—only the list of city names."  
   - Connect Form Submission Trigger to this node.  
   - Use Claude AI Model for Cities node as AI language model.

5. **Add Claude AI Model for Categories node:**  
   - Type: Langchain LM Chat (Anthropic)  
   - Model: "claude-sonnet-4-20250514"  
   - Set API credentials.

6. **Add Categorize Service Type node:**  
   - Type: Langchain Agent  
   - Prompt:  
     "Determine the appropriate category based on the following input: \"{{ $('Form Submission Trigger').item.json.Service }}\"  
      The output must contain only the category name that best describes the given service. Examples of categories include but are not limited to: Electronics, Healthcare, Education, Food & Beverage, Automotive, Finance, Real Estate, etc."  
   - Connect Generate City List to this node.  
   - Use Claude AI Model for Categories as AI language model.

7. **Add Separate Data by City node:**  
   - Type: Code node (JavaScript)  
   - Code:  
     ```js
     const serviceName = $('Form Submission Trigger').first().json.Service;
     const state = $('Form Submission Trigger').first().json.State;
     const country = $('Form Submission Trigger').first().json.Country;
     const citiesString = $('Generate City List').first().json.output;

     if (!serviceName || !state || !country || !citiesString) {
       throw new Error("Missing required input data (Service, State, Country, or City list)");
     }

     const cities = citiesString.split('\n').map(city => city.trim()).filter(city => city.length > 0);

     const output = cities.map(city => ({
       json: {
         name: serviceName,
         country: country,
         state: state,
         city: city,
         category: serviceName
       }
     }));

     return output;
     ```  
   - Connect Categorize Service Type to this node.

8. **Add Sticky Note for City Processing:**  
   - Content: "Separate each search by city name for comprehensive coverage."

9. **Add Process Cities in Batches node:**  
   - Type: SplitInBatches  
   - Connect Separate Data by City to this node.

10. **Add Get Existing Business Data node:**  
    - Type: Google Sheets (Read)  
    - Operation: Read rows from the sheet named "gid=0" in your Google Sheet document.  
    - Configure OAuth2 credentials for Google Sheets.  
    - Connect Process Cities in Batches to this node.

11. **Add Scrape Business Data from Google Maps node:**  
    - Type: Bright Data Scraper node  
    - Dataset ID: "gd_lh0tnzlo2bie4uhdhr" (Google Maps businesses)  
    - URL template:  
      `https://www.google.com/maps/search/{{ $json.name }}+in+{{ $json.city }} {{ $json.state }}`  
    - Category: `{{ $json.category }}`  
    - Country Name: `{{ $json.country }}`  
    - Proxy: leave empty or configure as needed  
    - Set Bright Data API credentials  
    - Connect Process Cities in Batches to this node.

12. **Add Check Scraping Status node:**  
    - Type: If node  
    - Condition: Check if `$json.message` exists (string exists operator).  
    - Connect Scrape Business Data from Google Maps to this node.

13. **Add Check Data Collection Status node:**  
    - Type: HTTP Request  
    - Method: GET  
    - URL: `https://api.brightdata.com/datasets/v3/progress/{{ $json.snapshot_id }}`  
    - Send headers with API credentials (Bright Data HTTP Header Auth)  
    - Connect "true" branch from Check Scraping Status to this node.

14. **Add Wait for Rate Limiting node:**  
    - Type: Wait node  
    - Delay: 25 seconds  
    - Connect Check Data Collection Status to this node.

15. **Add Check if Data Ready node:**  
    - Type: If node  
    - Condition: `$json.status` equals "ready"  
    - Connect Wait for Rate Limiting to this node.

16. **Connect Check if Data Ready:**  
    - "true" branch to Fetch Scraped Data node.  
    - "false" branch back to Check Data Collection Status node (loop).

17. **Add Fetch Scraped Data node:**  
    - Type: HTTP Request  
    - Method: GET  
    - URL: `https://api.brightdata.com/datasets/v3/snapshot/{{ $json.snapshot_id }}`  
    - Query parameter: format=json  
    - Use Bright Data HTTP Header Auth credentials  
    - Connect "true" branch of Check if Data Ready to this node.

18. **Add Save Business Data to Sheet node:**  
    - Type: Google Sheets (Append)  
    - Map columns explicitly: Business Name, Website URL, Category, Country, City, Phone Number, Google Maps URL, Address, Operating Hours, Google Rating, Total Reviews, Reviews URL (mapping from fetched data fields and city data)  
    - Use OAuth2 credentials  
    - Connect Fetch Scraped Data node output to this node.  
    - Also connect "false" branch of Check Scraping Status directly to this node to handle immediate data.

19. **Add Check for Duplicate Entries node:**  
    - Type: Code node  
    - Logic: Identify duplicates by comparing business name (case-insensitive) and phone numbers between existing data and new data.  
    - Connect Get Existing Business Data to this node.

20. **Add Process Duplicates in Batches node:**  
    - Type: SplitInBatches  
    - Connect Check for Duplicate Entries to this node.

21. **Add Find Duplicate Row Number node:**  
    - Type: Google Sheets (Lookup)  
    - Filters: Match Address and Google Maps URL columns from duplicates.  
    - Return first matching row number.  
    - Use OAuth2 credentials.  
    - Connect second output (duplicates batch) of Process Duplicates in Batches to this node.

22. **Add Delete Duplicate Row node:**  
    - Type: Google Sheets (Delete)  
    - Delete row at index provided by Find Duplicate Row Number.  
    - Connect Find Duplicate Row Number to this node.

23. **Connect Delete Duplicate Row back to Process Duplicates in Batches (first output) to continue processing duplicates.**

24. **Add Sticky Notes for Data Collection and Duplicate Removal:**  
    - Provide explanations and instructions per the original notes.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                | Context or Link                                                                                     |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Workflow extracts Google My Business leads filtered by service and location using Bright Data's Google Maps dataset, automates city-level scraping, de-duplicates, and saves data. | Main workflow description (Sticky Note)                                                           |
| Separate each search by city name for comprehensive coverage in scraping.                                                                                                   | City Processing Note                                                                                |
| Data Collection Monitoring: Monitors Bright Data scraping progress and fetches results when ready; polls API until scraping completes.                                       | Data Collection Notes                                                                              |
| Duplicate Removal Process: Identifies and removes duplicate business entries by comparing business names and phone numbers.                                                 | Duplicate Removal Notes                                                                            |
| Bright Data Dataset ID used: "gd_lh0tnzlo2bie4uhdhr" is specific to Google Maps business listings.                                                                           | Bright Data node configuration                                                                     |
| Requires credentials for: Bright Data API, Google Sheets OAuth2, Anthropic API for AI models.                                                                                | Credential setup                                                                                   |

---

**Disclaimer:** The provided content is extracted exclusively from an automated n8n workflow. It respects all applicable content policies and contains no illegal, offensive, or protected elements. All data manipulated is legal and publicly accessible.

---