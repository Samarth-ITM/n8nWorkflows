Scrape Google Places via Dumpling AI and Auto-Save to Google Sheets

https://n8nworkflows.xyz/workflows/scrape-google-places-via-dumpling-ai-and-auto-save-to-google-sheets-4632


# Scrape Google Places via Dumpling AI and Auto-Save to Google Sheets

### 1. Workflow Overview

This workflow automates the process of scraping Google Places data using Dumpling AI’s API and saving the results into a Google Sheet. It is designed to run daily at 1 PM, reading search queries from a Google Sheet, sending those queries to Dumpling AI to fetch place listings, splitting the returned list into individual place entries, and appending each place’s details into another Google Sheet. The workflow is ideal for users looking to build or update local business lead lists by leveraging Google Maps data efficiently.

Logical blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow daily at a specific time.
- **1.2 Input Reception:** Fetches search terms (queries) from a Google Sheet.
- **1.3 AI Processing:** Sends each search term to Dumpling AI’s Google Places scraping API.
- **1.4 Data Transformation:** Splits the returned list of places into individual items.
- **1.5 Data Persistence:** Appends each place’s detailed information into a target Google Sheet.
- **1.6 Documentation:** Provides an embedded sticky note explaining the workflow purpose.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  Initiates the entire workflow automatically every day at 1 PM to ensure timely data scraping and updating.

- **Nodes Involved:**  
  - Run Every Day at 1 PM

- **Node Details:**  

  - **Run Every Day at 1 PM**  
    - Type: Schedule Trigger  
    - Role: Triggers the workflow execution on a daily schedule.  
    - Configuration: Trigger set at 13:00 hours (1 PM) daily.  
    - Inputs: None (trigger node)  
    - Outputs: Connects to "Fetch Search Terms from Sheet" node.  
    - Potential Failures: Scheduling misconfiguration, time zone mismatches.  
    - Version: 1.2

#### 1.2 Input Reception

- **Overview:**  
  Reads search queries from the first Google Sheet ("Sheet1") which contains business-related keywords to be used for scraping.

- **Nodes Involved:**  
  - Fetch Search Terms from Sheet

- **Node Details:**  

  - **Fetch Search Terms from Sheet**  
    - Type: Google Sheets  
    - Role: Reads search terms from a predefined Google Sheet document and sheet tab.  
    - Configuration:  
      - Document ID linked to the target Google Sheet.  
      - Sheet name: "Sheet1" (assumed to contain search terms, e.g., "best dentist in Houston").  
      - Operation: Read (default).  
    - Credentials: Google Sheets OAuth2 account authorized for access.  
    - Inputs: Triggered by the schedule node.  
    - Outputs: Sends JSON containing the search terms to the next node.  
    - Potential Failures: Authentication errors, sheet access permissions, empty or malformed data.  
    - Version: 4.6

#### 1.3 AI Processing

- **Overview:**  
  Sends each fetched search term as a query to Dumpling AI’s "search-places" API to scrape Google Places data.

- **Nodes Involved:**  
  - Scrape Google Places with Dumpling AI

- **Node Details:**  

  - **Scrape Google Places with Dumpling AI**  
    - Type: HTTP Request  
    - Role: Interacts with Dumpling AI’s REST API to retrieve Google Places search results based on input queries.  
    - Configuration:  
      - Method: POST  
      - URL: https://app.dumplingai.com/api/v1/search-places  
      - Body format: JSON  
      - Body content uses expression: `{ "query": "{{ $json['places '] }}" }`  
        (Note: The key 'places ' has a trailing space which may be an error or intentional depending on the input data structure.)  
      - Authentication: HTTP Header Auth with stored Dumpling AI API key.  
    - Inputs: Receives search terms JSON from the previous node.  
    - Outputs: JSON response containing a list of places per query.  
    - Potential Failures: API key invalid or expired, rate limits, malformed body, network timeouts.  
    - Version: 4.2

#### 1.4 Data Transformation

- **Overview:**  
  Takes the list of places returned by Dumpling AI and splits each place entry into individual workflow items for granular processing.

- **Nodes Involved:**  
  - Split Resulting Places List

- **Node Details:**  

  - **Split Resulting Places List**  
    - Type: Split Out  
    - Role: Splits an array field ("places") in the JSON response into individual items for processing one by one.  
    - Configuration:  
      - Field to split out: "places" (array expected in the response).  
    - Inputs: Receives array of places from the Dumpling AI API node.  
    - Outputs: Multiple individual place JSON objects, each passed to the next node.  
    - Potential Failures: Missing or empty “places” field, malformed JSON.  
    - Version: 1

#### 1.5 Data Persistence

- **Overview:**  
  Appends each place’s details into a target Google Sheet ("Sheet2") as a new row, storing key fields such as title, rating, address, category, phone number, and website.

- **Nodes Involved:**  
  - Save Scraped Data to Sheet

- **Node Details:**  

  - **Save Scraped Data to Sheet**  
    - Type: Google Sheets  
    - Role: Inserts place data into a Google Sheet for persistent storage and later analysis.  
    - Configuration:  
      - Operation: Append (adds rows at the end).  
      - Document ID: Linked to the target Google Sheet.  
      - Sheet Name: "Sheet2"  
      - Columns mapped explicitly by expressions for each field:  
        - title, rating, address, website, category, phoneNumber  
      - Mapping Mode: Defines columns manually to ensure correct data placement.  
      - Type Conversion: Disabled (stores all as strings).  
    - Credentials: Google Sheets OAuth2 authorized account.  
    - Inputs: Receives individual place JSON objects from the Split node.  
    - Outputs: None (endpoint node).  
    - Potential Failures: Authentication errors, write permission issues, schema mismatch, empty fields.  
    - Version: 4.6

#### 1.6 Documentation

- **Overview:**  
  Contains a sticky note summarizing the workflow’s purpose and operation for user reference.

- **Nodes Involved:**  
  - Sticky Note

- **Node Details:**  

  - **Sticky Note**  
    - Type: Sticky Note  
    - Role: Provides explanatory text for users viewing the workflow.  
    - Content:  
      - Describes daily scraping at 1 PM, input/output sheets, API usage, and use case.  
      - Size configured for readability.  
    - Inputs/Outputs: None (informational only).  
    - Version: 1

---

### 3. Summary Table

| Node Name                          | Node Type        | Functional Role                          | Input Node(s)                | Output Node(s)                     | Sticky Note                                                                                                                                                 |
|-----------------------------------|------------------|----------------------------------------|-----------------------------|----------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Run Every Day at 1 PM              | Schedule Trigger | Initiates workflow daily at 1 PM       | None                        | Fetch Search Terms from Sheet    |                                                                                                                                                             |
| Fetch Search Terms from Sheet      | Google Sheets    | Reads search queries from input sheet  | Run Every Day at 1 PM        | Scrape Google Places with Dumpling AI |                                                                                                                                                             |
| Scrape Google Places with Dumpling AI | HTTP Request     | Sends queries to Dumpling AI API       | Fetch Search Terms from Sheet | Split Resulting Places List       |                                                                                                                                                             |
| Split Resulting Places List        | Split Out        | Splits API response into individual places | Scrape Google Places with Dumpling AI | Save Scraped Data to Sheet         |                                                                                                                                                             |
| Save Scraped Data to Sheet         | Google Sheets    | Appends place details into output sheet | Split Resulting Places List  | None                             |                                                                                                                                                             |
| Sticky Note                       | Sticky Note      | Explains workflow purpose               | None                        | None                             | ### 🗺️ Scrape and Save Google Places Listings\n\nThis workflow runs daily at 1 PM. It reads business-related search terms from a Google Sheet (Sheet1), such as "best dentist in Houston", and passes each term to Dumpling AI's `search-places` API. The returned list of places is split and parsed. Each entry is appended to another Google Sheet (Sheet2), capturing title, address, rating, category, phone number, and website. Perfect for building local lead lists from Google Maps results. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Name: "Run Every Day at 1 PM"  
   - Type: Schedule Trigger  
   - Set interval: Trigger at hour 13 (1 PM) daily.

2. **Create a Google Sheets node to fetch search terms:**  
   - Name: "Fetch Search Terms from Sheet"  
   - Operation: Read rows  
   - Document ID: Paste your Google Sheet’s ID where search terms are stored.  
   - Sheet Name: "Sheet1"  
   - Credentials: Configure OAuth2 credentials for Google Sheets with read access.  
   - Connect the output of the Schedule Trigger node to this node.

3. **Create an HTTP Request node to call Dumpling AI API:**  
   - Name: "Scrape Google Places with Dumpling AI"  
   - HTTP Method: POST  
   - URL: https://app.dumplingai.com/api/v1/search-places  
   - Authentication: HTTP Header Auth using Dumpling AI API key credential.  
   - Body Content (JSON):  
     ```json
     {
       "query": "{{ $json['places '] }}"
     }
     ```  
     Note: Confirm if 'places ' key with trailing space is correct, otherwise adjust to the actual field containing the search term.  
   - Connect the output of "Fetch Search Terms from Sheet" to this node.

4. **Create a Split Out node to split the places list:**  
   - Name: "Split Resulting Places List"  
   - Field to split out: "places" (the array field in Dumpling AI response)  
   - Connect the output of the HTTP Request node to this node.

5. **Create a Google Sheets node to save scraped data:**  
   - Name: "Save Scraped Data to Sheet"  
   - Operation: Append rows  
   - Document ID: Use your target Google Sheet’s ID for saving results.  
   - Sheet Name: "Sheet2"  
   - Define columns explicitly and map them with expressions:  
     - title: `={{ $json.title }}`  
     - rating: `={{ $json.rating }}`  
     - address: `={{ $json.address }}`  
     - website: `={{ $json.website }}`  
     - category: `={{ $json.category }}`  
     - phoneNumber: `={{ $json.phoneNumber }}`  
   - Credentials: Use Google Sheets OAuth2 credentials with write access.  
   - Connect the output of the Split node to this node.

6. **(Optional) Add a Sticky Note to document the workflow:**  
   - Add a sticky note with the content describing the workflow purpose and operation for clarity.

7. **Activate and test the workflow:**  
   - Ensure all credentials are valid and authorized.  
   - Test by manually triggering to verify data flows correctly.  
   - Monitor for any errors such as API failures, permission issues, or data mismatches.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                   | Context or Link                                                                                     |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow is designed to automate scraping of local business data from Google Places using Dumpling AI and Google Sheets integration.                                                                                                    | General workflow description                                                                       |
| Ensure Dumpling AI API key is valid and has appropriate quota to avoid request failures.                                                                                                                                                      | API usage best practices                                                                            |
| Google Sheets credentials must have both read and write permissions on the respective sheets involved ("Sheet1" for queries, "Sheet2" for results).                                                                                         | OAuth2 credential setup                                                                             |
| Verify that the key in the HTTP Request body expression matches the actual input data field; a trailing space in `'places '` could cause expression failures.                                                                                | Potential expression syntax issue                                                                  |
| For detailed API documentation, visit Dumpling AI’s official site: https://app.dumplingai.com/documentation                                                                                                                                    | Dumpling AI API documentation (external link)                                                     |
| The workflow can be adapted to other sources of input queries or extended with error handling nodes for robust production use.                                                                                                               | Suggested extensions                                                                               |

---

**Disclaimer:** The provided text is exclusively derived from an automated n8n workflow. It complies strictly with current content policies and contains no illegal, offensive, or protected content. All data processed is legal and public.