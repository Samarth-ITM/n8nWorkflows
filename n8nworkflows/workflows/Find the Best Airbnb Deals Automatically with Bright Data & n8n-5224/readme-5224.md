Find the Best Airbnb Deals Automatically with Bright Data & n8n

https://n8nworkflows.xyz/workflows/find-the-best-airbnb-deals-automatically-with-bright-data---n8n-5224


# Find the Best Airbnb Deals Automatically with Bright Data & n8n

### 1. Workflow Overview

This n8n workflow automates the process of finding and storing the best Airbnb deals daily, leveraging Bright Data’s web scraping capabilities to bypass bot detection and reliably extract rental listings data. It is designed for users who want to track Airbnb prices over time for specific locations and dates, such as travel bloggers, deal hunters, or market researchers.

The workflow consists of three logical blocks:

- **1.1 Define & Trigger the Search:** Sets the search parameters and triggers the workflow on a daily schedule.
- **1.2 Scrape & Extract Airbnb Data:** Uses Bright Data to scrape Airbnb listings and parses the HTML response to extract relevant information.
- **1.3 Store the Results:** Saves the extracted Airbnb listing data into a Google Sheets document for historical tracking and analysis.

---

### 2. Block-by-Block Analysis

#### 1.1 Define & Trigger the Search

**Overview:**  
This block initiates the workflow automatically every day at 9 AM and sets the search criteria for Airbnb listings including location, check-in/out dates, and number of guests.

**Nodes Involved:**  
- Run Daily (9 AM)  
- Set Search Criteria

**Node Details:**

- **Run Daily (9 AM)**  
  - Type: Schedule Trigger  
  - Role: Automatically triggers the workflow daily at 9:00 AM without manual intervention.  
  - Configuration: Triggers exactly at hour 9 daily.  
  - Inputs: None (trigger node)  
  - Outputs: Connects to "Set Search Criteria"  
  - Edge Cases: Workflow won't run if n8n server is offline at trigger time.  
  - Notes: Essential for full automation and hands-free operation.

- **Set Search Criteria**  
  - Type: Set  
  - Role: Defines fixed search parameters used to craft the Airbnb search URL.  
  - Configuration: Sets the following parameters:  
    - Location: "Los-Angeles"  
    - Checkin: "2025-09-15"  
    - Checkout: "2025-09-20"  
    - Adults: 2  
  - Key Expressions: Values are static strings/numbers assigned directly, referenced downstream via expressions (e.g., `{{ $json.Location }}`).  
  - Inputs: Receives trigger from "Run Daily (9 AM)"  
  - Outputs: Connects to "Scrape Airbnb via Bright Data"  
  - Edge Cases: Parameters are hardcoded; user must update manually for other searches or extend to dynamic input.

---

#### 1.2 Scrape & Extract Airbnb Data

**Overview:**  
This block uses Bright Data’s API to scrape Airbnb’s search results page for the specified criteria, then parses the returned HTML to extract listing titles and prices.

**Nodes Involved:**  
- Scrape Airbnb via Bright Data  
- Extract Price & Title

**Node Details:**

- **Scrape Airbnb via Bright Data**  
  - Type: HTTP Request  
  - Role: Sends a POST request to Bright Data's API to retrieve Airbnb search page content for the given parameters.  
  - Configuration:  
    - URL: `https://api.brightdata.com/request`  
    - Method: POST  
    - Body Parameters:  
      - zone: "n8n_unblocker" (Bright Data proxy zone)  
      - url: Constructed Airbnb search URL using expressions, e.g. `https://www.airbnb.com/s/{{ $json.Location }}--CA--United-States/homes?checkin={{ $json.checkin }}&checkout={{ $json.checkout }}&adults={{ $json.adults }}`  
      - country: "us"  
      - format: "raw" (returns raw HTML content)  
    - Headers: Authorization bearer token `"Bearer API KEY"` (user must replace with valid Bright Data API key)  
  - Inputs: From "Set Search Criteria"  
  - Outputs: Connects to "Extract Price & Title"  
  - Edge Cases:  
    - Authentication errors if API key invalid or missing.  
    - Rate limiting or quota exhaustion on Bright Data side.  
    - Changes in Airbnb page structure may result in missing or malformed data.  
    - Network timeouts or failures.  
  - Notes: Requires valid Bright Data credentials configured in n8n.

- **Extract Price & Title**  
  - Type: Code (JavaScript)  
  - Role: Parses the raw HTML response to extract listing titles and prices in a structured format.  
  - Configuration and Logic:  
    - Retrieves the HTML string from the previous node output (`items[0].json.data`).  
    - Uses regex to find the `<script>` tag with id `data-deferred-state-0` which contains embedded JSON data.  
    - Parses JSON content to locate the `searchResults` array with listing data.  
    - Maps each listing to an object with properties:  
      - `title`: The listing title.  
      - `price`: Extracted from `primaryLine.discountedPrice` or `primaryLine.price` plus any qualifier (e.g., currency).  
    - Returns an array of objects, each representing one listing for the next node.  
  - Inputs: From "Scrape Airbnb via Bright Data"  
  - Outputs: Connects to "Save to Google Sheets"  
  - Edge Cases:  
    - If regex does not find the script tag, returns error item with message "Data script tag not found...".  
    - If JSON parsing fails, returns error with message "Failed to parse JSON...".  
    - If expected keys are missing in JSON, returns error "Search results not found...".  
    - Airbnb page structure changes can break extraction logic.  
  - Notes: This node is critical for transforming raw data into usable structured data.

---

#### 1.3 Store the Results

**Overview:**  
This block takes the parsed Airbnb listings and appends them as new rows in a Google Sheets document, creating a historical log of deals.

**Nodes Involved:**  
- Save to Google Sheets

**Node Details:**

- **Save to Google Sheets**  
  - Type: Google Sheets  
  - Role: Appends rows to a Google Sheets spreadsheet with extracted Airbnb listing data plus search criteria metadata.  
  - Configuration:  
    - Operation: Append  
    - Document ID: Spreadsheet ID `"1nwNkfh8n1qr6ft5mF3PhhDTTaDXb1K_4H3gd3D-nADc"`  
    - Sheet: `gid=0` (first sheet)  
    - Columns: Mapping JSON keys to columns:  
      - Location, CheckIn, CheckOut, Adults (from "Set Search Criteria")  
      - Price, Title (from extracted listing)  
    - Credentials: Uses configured Google Sheets OAuth2 credentials.  
  - Inputs: From "Extract Price & Title"  
  - Outputs: None  
  - Edge Cases:  
    - Authentication or permission errors if Google credentials are invalid or lack access.  
    - If the sheet structure changes, columns may mismatch.  
    - Large volumes of data might hit Google Sheets API limits.  
  - Notes: Enables tracking of data over time and downstream analysis.

---

### 3. Summary Table

| Node Name                | Node Type           | Functional Role                          | Input Node(s)           | Output Node(s)              | Sticky Note                                                                                                  |
|--------------------------|---------------------|----------------------------------------|-------------------------|----------------------------|--------------------------------------------------------------------------------------------------------------|
| Run Daily (9 AM)         | Schedule Trigger    | Triggers workflow daily at 9 AM        | None                    | Set Search Criteria         | Section 1: Define & Trigger the Search — your daily alarm clock to start the workflow automatically.         |
| Set Search Criteria       | Set                 | Defines search filters (location, dates, guests) | Run Daily (9 AM)        | Scrape Airbnb via Bright Data | Section 1: Define & Trigger the Search — sets filters for the Airbnb search URL.                              |
| Scrape Airbnb via Bright Data | HTTP Request        | Scrapes Airbnb page via Bright Data API | Set Search Criteria      | Extract Price & Title       | Section 2: Scrape & Extract Airbnb Data — sends stealth request to Airbnb, bypassing bot detection.           |
| Extract Price & Title     | Code                 | Parses HTML to extract listing titles and prices | Scrape Airbnb via Bright Data | Save to Google Sheets        | Section 2: Scrape & Extract Airbnb Data — extracts and cleans listing data from page source.                  |
| Save to Google Sheets     | Google Sheets        | Appends extracted listings to Google Sheet  | Extract Price & Title    | None                       | Section 3: Store the Results — saves daily listings to a spreadsheet for historical tracking.                 |
| Sticky Note               | Sticky Note          | Informational notes                    | None                    | None                       | Section 1 explanation and overview.                                                                           |
| Sticky Note1              | Sticky Note          | Informational notes                    | None                    | None                       | Section 2 explanation and overview.                                                                           |
| Sticky Note2              | Sticky Note          | Informational notes                    | None                    | None                       | Section 3 explanation and overview.                                                                           |
| Sticky Note4              | Sticky Note          | Detailed workflow overview and benefits | None                    | None                       | Full automation overview and workflow summary.                                                               |
| Sticky Note5              | Sticky Note          | Affiliate link for Bright Data         | None                    | None                       | Bright Data affiliate link for supporting content creation.                                                   |
| Sticky Note9              | Sticky Note          | Contact and support information        | None                    | None                       | Workflow assistance contact and resource links.                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set to trigger daily at 9:00 AM (hour = 9)  
   - Name it "Run Daily (9 AM)"

2. **Add a Set node**  
   - Name it "Set Search Criteria"  
   - Add fields:  
     - Location (string): "Los-Angeles"  
     - checkin (string): "2025-09-15"  
     - checkout (string): "2025-09-20"  
     - adults (number): 2  
   - Connect "Run Daily (9 AM)" output to this node input

3. **Add an HTTP Request node**  
   - Name it "Scrape Airbnb via Bright Data"  
   - Set:  
     - HTTP Method: POST  
     - URL: `https://api.brightdata.com/request`  
     - Send Body: true  
     - Body Parameters (JSON):  
       ```json
       {
         "parameters": [
           { "name": "zone", "value": "n8n_unblocker" },
           { "name": "url", "value": "https://www.airbnb.com/s/{{ $json.Location }}--CA--United-States/homes?checkin={{ $json.checkin }}&checkout={{ $json.checkout }}&adults={{ $json.adults }}" },
           { "name": "country", "value": "us" },
           { "name": "format", "value": "raw" }
         ]
       }
       ```
     - Header Parameters:  
       - `Authorization`: `Bearer API KEY` (replace `API KEY` with your Bright Data API token)  
   - Connect "Set Search Criteria" output to this node input

4. **Add a Code node (JavaScript)**  
   - Name it "Extract Price & Title"  
   - Paste the provided JavaScript code that:  
     - Extracts the `<script id="data-deferred-state-0">` content from HTML  
     - Parses JSON inside  
     - Finds `searchResults` array  
     - Maps each to `{title, price}` objects  
     - Returns array of listings for downstream nodes  
   - Connect "Scrape Airbnb via Bright Data" output to this node input

5. **Add a Google Sheets node**  
   - Name it "Save to Google Sheets"  
   - Operation: Append  
   - Enter your Google Sheets OAuth2 credentials  
   - Document ID: Use the ID of your target Google Sheets document  
   - Sheet Name: `gid=0` or your desired sheet  
   - Map columns as follows:  
     - Location → `{{ $('Set Search Criteria').item.json.Location }}`  
     - CheckIn → `{{ $('Set Search Criteria').item.json.checkin }}`  
     - CheckOut → `{{ $('Set Search Criteria').item.json.checkout }}`  
     - Adults → `{{ $('Set Search Criteria').item.json.adults }}`  
     - Price → `{{ $json.price }}`  
     - Title → `{{ $json.title }}`  
   - Connect "Extract Price & Title" output to this node input

6. **Test the workflow** by manually executing each node step-by-step, making sure:  
   - Bright Data API key is valid and has quota  
   - Google Sheets credentials have write access  
   - The regex parsing works with the current Airbnb page structure

7. **Activate the workflow** to run automatically daily.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                                  |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| For questions or support, contact Yaron at Yaron@nofluff.online                                               | Contact email                                                                                    |
| Explore more tips and video tutorials on YouTube and LinkedIn                                                 | YouTube: https://www.youtube.com/@YaronBeen/videos<br>LinkedIn: https://www.linkedin.com/in/yaronbeen/ |
| Affiliate link to support Bright Data usage: https://get.brightdata.com/1tndi4600b25                           | Bright Data affiliate program                                                                    |
| This workflow is designed for automation simplicity and extendability, allowing future additions like alerts or analytics | General best practice                                                                             |

---

**Disclaimer:** The content provided is generated from an automated n8n workflow and complies with all applicable content policies. All data handled are legal and publicly available.