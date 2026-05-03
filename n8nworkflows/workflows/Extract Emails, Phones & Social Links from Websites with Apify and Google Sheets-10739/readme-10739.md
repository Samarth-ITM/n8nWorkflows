Extract Emails, Phones & Social Links from Websites with Apify and Google Sheets

https://n8nworkflows.xyz/workflows/extract-emails--phones---social-links-from-websites-with-apify-and-google-sheets-10739


# Extract Emails, Phones & Social Links from Websites with Apify and Google Sheets

### 1. Workflow Overview

This workflow automates the extraction of emails, phone numbers, and social media links from websites listed in a Google Sheet and writes the enriched contact data back into a newly created Google Sheet. It is designed primarily for lead enrichment, CRM updating, sales prospecting, recruiting, and marketing outreach.

The workflow logically divides into the following blocks:

- **1.1 Input Setup and Trigger**: Manual start and setting up Google Sheet details.
- **1.2 Google Sheets Data Handling**: Reading website URLs from the source sheet and creating a new sheet for results.
- **1.3 Data Formatting for Apify Actor**: Preparing the input data in the format required by the Apify Email & Phone Extractor actor.
- **1.4 Running the Apify Actor**: Executing the Apify actor to scrape contact details.
- **1.5 Retrieving and Saving Results**: Getting the results from Apify and writing them into the new Google Sheet.
- **1.6 Documentation and Notes**: Sticky notes provide explanations, usage instructions, and examples.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Setup and Trigger

- **Overview:**  
  This block initializes the workflow manually and sets the Google Sheet URL and source sheet name that contains the list of websites to process.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’  
  - Set google sheet URL & original sheet name

- **Node Details:**  

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow on manual execution.  
    - Configuration: Default trigger with no parameters.  
    - Inputs: None  
    - Outputs: Triggers "Set google sheet URL & original sheet name" node.  
    - Edge cases: None typical; user must manually start workflow.

  - **Set google sheet URL & original sheet name**  
    - Type: Set  
    - Role: Stores the Google Sheet URL and sheet name as workflow variables for downstream use.  
    - Configuration: Two string variables are set:  
      - `google_sheet_url` (e.g., "https://docs.google.com/spreadsheets/d/...")  
      - `google_sheet_name` (e.g., "websites")  
    - Key Expressions: Hardcoded values; expected to be replaced by user before execution.  
    - Inputs: From manual trigger  
    - Outputs: Flows to two nodes: "Create new sheet for founded emails and phones" and "Get website URLs from first sheet"  
    - Edge cases: If values are not set correctly or URL is invalid, later nodes will fail to access sheets.

---

#### 2.2 Google Sheets Data Handling

- **Overview:**  
  Reads the list of website URLs from the specified sheet and creates a new Google Sheet tab to store extraction results.

- **Nodes Involved:**  
  - Create new sheet for founded emails and phones  
  - Get website URLs from first sheet  
  - wait for previous nodes to finish (Merge node)

- **Node Details:**  

  - **Create new sheet for founded emails and phones**  
    - Type: Google Sheets (Create Operation)  
    - Role: Creates a new sheet tab inside the specified Google Sheet document where results will be stored.  
    - Configuration:  
      - Title: Dynamic with timestamp, e.g., "emails-phones-11/11-22:31"  
      - Document ID: from `google_sheet_url`  
    - Inputs: From "Set google sheet URL & original sheet name"  
    - Outputs: Passes new sheet ID downstream  
    - Edge cases: Requires valid Google Sheets OAuth2 credentials and permissions; errors if document is not accessible or quota exceeded.

  - **Get website URLs from first sheet**  
    - Type: Google Sheets (Read Operation)  
    - Role: Reads all rows from the sheet named in `google_sheet_name` to get the list of websites.  
    - Configuration:  
      - Sheet name: dynamic from input variables  
      - Document ID: from `google_sheet_url`  
    - Inputs: From "Set google sheet URL & original sheet name"  
    - Outputs: Passes website list to the merge node  
    - Edge cases: Empty or malformed sheet may produce empty or invalid URLs.

  - **wait for previous nodes to finish**  
    - Type: Merge (Wait for all inputs)  
    - Role: Synchronizes the flow to wait for both sheet creation and URLs retrieval before proceeding.  
    - Configuration: Default (wait for all inputs)  
    - Inputs: From "Create new sheet..." and "Get website URLs from first sheet"  
    - Outputs: To "format data for Apify INPUT type"  
    - Edge cases: If either input node fails or returns no data, workflow may hang or error.

---

#### 2.3 Data Formatting for Apify Actor

- **Overview:**  
  Formats the retrieved website URLs into the specific input schema expected by the Apify Email & Phone Extractor actor.

- **Nodes Involved:**  
  - format data for Apify INPUT type

- **Node Details:**  

  - **format data for Apify INPUT type**  
    - Type: Code (JavaScript)  
    - Role: Constructs the Apify actor input JSON object with crawl parameters and start URLs.  
    - Configuration:  
      - Sets parameters such as `maxRequests`, `sameDomain`, `onlyEmails`, `onlyOneEmailPerDomain`, `maxDepth`, `pseudoUrls`, and proxy usage.  
      - Extracts URLs from the input JSON (keyed as `url` or `website`) from the "Get website URLs from first sheet" node.  
      - Returns a single JSON object with all parameters and start URLs.  
    - Key Expressions:  
      - Uses jQuery-style selector `$('Get website URLs from first sheet')` to access upstream data.  
      - Constructs array of URLs as `startUrls` for actor input.  
    - Inputs: From "wait for previous nodes to finish" (merged data)  
    - Outputs: To "Run Actor on Apify"  
    - Edge cases: Missing or malformed URLs in input rows can cause actor input to be invalid; JSON structure must match actor requirements.  
    - Notes: Parameters are customizable as explained in Sticky Note3 for crawl depth, request limits, and filtering.

---

#### 2.4 Running the Apify Actor

- **Overview:**  
  Runs the Apify Email & Phone Extractor actor with the formatted input, initiating the scraping of contact data from the listed websites.

- **Nodes Involved:**  
  - Run Actor on Apify

- **Node Details:**  

  - **Run Actor on Apify**  
    - Type: Apify node (Execute Actor)  
    - Role: Executes the Apify actor `bxrabKhLv1c3fLmoj` (Email & Phone Extractor) with the JSON input from formatting node.  
    - Configuration:  
      - Actor ID specified by list mode referencing the official Apify actor.  
      - Input body set dynamically from previous node JSON.  
      - No timeout configured (default) — actor run time may vary based on crawl size.  
    - Credentials: Apify API credentials required and linked.  
    - Inputs: From "format data for Apify INPUT type"  
    - Outputs: To "Get Results from Apify"  
    - Edge cases:  
      - Actor run may take long time due to crawling multiple pages.  
      - Possible API errors, authentication failures, or rate limiting on Apify side.  
      - Actor logs must be checked manually for progress tracking.

---

#### 2.5 Retrieving and Saving Results

- **Overview:**  
  Retrieves the dataset results from Apify after actor execution and appends the extracted emails, phones, and social links into the newly created Google Sheet.

- **Nodes Involved:**  
  - Get Results from Apify  
  - Add emails, phones, socials... into the new Sheet

- **Node Details:**  

  - **Get Results from Apify**  
    - Type: Apify node (Get Dataset)  
    - Role: Fetches the dataset output from the completed Apify actor run using the dataset ID provided in the actor output.  
    - Configuration:  
      - Dataset ID dynamically read from actor run JSON (`defaultDatasetId`)  
      - Reads all records from the dataset.  
    - Credentials: Uses same Apify API credentials.  
    - Inputs: From "Run Actor on Apify"  
    - Outputs: To "Add emails, phones, socials... into the new Sheet"  
    - Edge cases: Dataset may be empty if actor failed or no contacts found.

  - **Add emails, phones, socials... into the new Sheet**  
    - Type: Google Sheets (Append Operation)  
    - Role: Appends the extracted contact data as rows into the newly created sheet for results.  
    - Configuration:  
      - Document ID from initial Google Sheet URL.  
      - Sheet name dynamically set from newly created sheet's ID.  
      - Columns are auto-mapped from input data fields including emails, phones, social links, and other profile data.  
      - Append operation to add new rows.  
    - Credentials: Google Sheets OAuth2 credentials required with edit rights.  
    - Inputs: From "Get Results from Apify"  
    - Outputs: End of workflow  
    - Edge cases: Mismatched or missing fields in data may cause append errors; sheet must exist and be accessible.

---

#### 2.6 Documentation and Notes

- **Overview:**  
  Provides user guidance, examples, and explanations via sticky notes attached throughout the workflow.

- **Nodes Involved:**  
  - Sticky Note7 (Main workflow overview and instructions)  
  - Sticky Note (Input example JSON)  
  - Sticky Note1 (Output example screenshot)  
  - Sticky Note2 (Google Sheet input example screenshot)  
  - Sticky Note3 (Parameter customization explanation)

- **Node Details:**  

  - Sticky notes contain detailed instructions on how to use and configure the workflow, including links to the Apify actor and expected inputs/outputs.  
  - Positioned strategically near relevant nodes to assist user understanding.  
  - No inputs or outputs; purely informational.  
  - Edge cases: None; only documentation.

---

### 3. Summary Table

| Node Name                                  | Node Type                | Functional Role                        | Input Node(s)                         | Output Node(s)                         | Sticky Note                                                                                                                                    |
|--------------------------------------------|--------------------------|-------------------------------------|-------------------------------------|---------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’            | Manual Trigger           | Workflow start trigger               | None                                | Set google sheet URL & original sheet name |                                                                                                                                                 |
| Set google sheet URL & original sheet name | Set                      | Define Google Sheet URL and sheet name | When clicking ‘Execute workflow’    | Create new sheet for founded emails and phones, Get website URLs from first sheet |                                                                                                                                                 |
| Create new sheet for founded emails and phones | Google Sheets (Create)   | Creates new sheet for output         | Set google sheet URL & original sheet name | wait for previous nodes to finish      |                                                                                                                                                 |
| Get website URLs from first sheet           | Google Sheets (Read)     | Reads list of websites from source sheet | Set google sheet URL & original sheet name | wait for previous nodes to finish      |                                                                                                                                                 |
| wait for previous nodes to finish            | Merge                    | Synchronizes sheet creation and data read | Create new sheet..., Get website URLs from first sheet | format data for Apify INPUT type        |                                                                                                                                                 |
| format data for Apify INPUT type             | Code                     | Formats input JSON for Apify actor  | wait for previous nodes to finish    | Run Actor on Apify                     | Sticky Note3: Explains crawl parameters like maxRequests, sameDomain, onlyEmails, maxDepth, pseudoUrls                                            |
| Run Actor on Apify                           | Apify                    | Runs Apify Email & Phone Extractor  | format data for Apify INPUT type     | Get Results from Apify                 | Sticky Note7: Main workflow explanation and Apify actor link                                                                                     |
| Get Results from Apify                       | Apify                    | Retrieves dataset results from actor | Run Actor on Apify                   | Add emails, phones, socials... into the new Sheet |                                                                                                                                                 |
| Add emails, phones, socials... into the new Sheet | Google Sheets (Append)  | Writes extracted contacts into new sheet | Get Results from Apify               | None                                  | Sticky Note1: Input example JSON; Sticky Note2: Google Sheet input example screenshot; Sticky Note7: Full usage and explanation                   |
| Sticky Note7                                 | Sticky Note              | Main usage notes and instructions   | None                                | None                                  | Full detailed instructions, use cases, links to Apify actor, speed note, requirements, and help information                                       |
| Sticky Note                                  | Sticky Note              | Input JSON example                  | None                                | None                                  | Shows example JSON input for Google Sheet URL and sheet name                                                                                    |
| Sticky Note1                                 | Sticky Note              | Output example screenshot           | None                                | None                                  | Shows example output screenshot of Google Sheet with enriched data                                                                              |
| Sticky Note2                                 | Sticky Note              | Google Sheet input example screenshot | None                                | None                                  | Shows screenshot of input Google Sheet with website URLs                                                                                        |
| Sticky Note3                                 | Sticky Note              | Explanation of Apify actor parameters | None                                | None                                  | Provides detailed explanation of customizable crawl parameters inside format data node                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow and add a manual trigger node:**  
   - Node: Manual Trigger (default settings)  
   - Purpose: To start workflow manually.

2. **Add a Set node to define Google Sheet URL and source sheet name:**  
   - Node: Set  
   - Add two string fields:  
     - `google_sheet_url`: Your Google Spreadsheet URL (e.g., "https://docs.google.com/spreadsheets/d/your-sheet-id/")  
     - `google_sheet_name`: Name of the sheet with website URLs (e.g., "websites")  
   - Connect from Manual Trigger.

3. **Add a Google Sheets node to create a new sheet for results:**  
   - Node: Google Sheets (Operation: Create)  
   - Document ID: Use expression referencing `google_sheet_url` from Set node.  
   - Title: Use expression with dynamic timestamp, e.g., `"emails-phones-" + new Date().toLocaleDateString() + "-" + new Date().toLocaleTimeString()`  
   - Connect from Set node.

4. **Add a Google Sheets node to read website URLs from source sheet:**  
   - Node: Google Sheets (Operation: Read)  
   - Document ID: From `google_sheet_url` (expression referencing the Set node).  
   - Sheet Name: From `google_sheet_name` (expression referencing the Set node).  
   - Connect from Set node.

5. **Add a Merge node (Wait for all inputs):**  
   - Node: Merge (Mode: Wait for all inputs)  
   - Connect inputs from both the "Create new sheet" and "Get website URLs" nodes.  
   - This ensures both sheet creation and website data retrieval complete before next step.

6. **Add a Code (JavaScript) node to format input for Apify actor:**  
   - Node: Code  
   - JavaScript snippet:  
     ```javascript
     let input = {
       maxRequests: 0,
       sameDomain: true,
       onlyEmails: false,
       onlyOneEmailPerDomain: false,
       maxDepth: 1,
       pseudoUrls: [".*"],
       proxyConfig: {
         useApifyProxy: true
       },
       considerChildFrames: true
     };

     const startUrls = [];
     for (const item of $('Get website URLs from first sheet').all()) {
       const newItem = {};
       newItem.url = item.json.url || item.json.website;
       startUrls.push(newItem);
     }

     return {
       json: {
         ...input,
         startUrls
       }
     };
     ```  
   - Connect from Merge node.

7. **Add Apify node to run the Email & Phone Extractor actor:**  
   - Node: Apify (Execute Actor)  
   - Actor ID: Use the official Email & Phone Extractor actor ID `"bxrabKhLv1c3fLmoj"` or select from list.  
   - Input: Use expression to pass JSON from Code node.  
   - Credentials: Set up Apify API credentials with access to this actor.  
   - Connect from Code node.

8. **Add Apify node to get results dataset:**  
   - Node: Apify (Get Dataset)  
   - Dataset ID: Expression referencing the output from the previous Apify node's `defaultDatasetId` field.  
   - Credentials: Same Apify credentials.  
   - Connect from Apify run node.

9. **Add Google Sheets node to append extracted data to the new sheet:**  
   - Node: Google Sheets (Operation: Append)  
   - Document ID: Use expression referencing the Google Sheet URL from Set node.  
   - Sheet Name: Use expression referencing the sheet ID from "Create new sheet" node output.  
   - Columns: Set to auto-map input data fields to sheet columns (emails, phones, socials, etc.).  
   - Credentials: Google Sheets OAuth2 credentials with read/write permission.  
   - Connect from Apify get dataset node.

10. **Add sticky notes as needed for documentation and user instructions.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Context or Link                                                                                         |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| This n8n template automates enriching a Google Sheet of websites with emails, phones, and social links using the Apify Email & Phone Extractor actor. Useful for sales, recruiting, marketing, research, CRM enrichment, and lead generation.                                                                                                                                                                                                                                                                                     | Sticky Note7 main workflow explanation with detailed use cases and instructions: [Apify actor](https://apify.com/anchor/email-phone-extractor) |
| Input JSON example for setting Google Sheet URL and sheet name: `[{ "google_sheet_url": "https://docs.google.com/spreadsheets/d/1ffxxxxxxxxxxvI/", "google_sheet_name": "profiles" }]`                                                                                                                                                                                                                                                                                                                                                 | Sticky Note near "Set google sheet URL & original sheet name" node                                   |
| Output example screenshot showing enriched Google Sheet with extracted emails and phones.                                                                                                                                                                                                                                                                                                                                                                                                                                           | Sticky Note near output Google Sheets node                                                           |
| Google Sheet input example screenshot showing column with website URLs.                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Sticky Note near input Google Sheets node                                                            |
| Customizable parameters for Apify actor input in the code node including maxRequests, sameDomain, onlyEmails, onlyOneEmailPerDomain, maxDepth, and pseudoUrls regex filters to control crawl behavior.                                                                                                                                                                                                                                                                                                                                | Sticky Note3 near code node "format data for Apify INPUT type"                                       |
| The "Run Actor on Apify" node can take a long time due to scraping many pages per website. Check Apify logs directly for progress.                                                                                                                                                                                                                                                                                                                                                                                                  | Sticky Note7 main instructions                                                                       |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created using n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.