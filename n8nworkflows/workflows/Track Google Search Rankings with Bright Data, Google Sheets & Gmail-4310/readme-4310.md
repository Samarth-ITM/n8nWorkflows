Track Google Search Rankings with Bright Data, Google Sheets & Gmail

https://n8nworkflows.xyz/workflows/track-google-search-rankings-with-bright-data--google-sheets---gmail-4310


# Track Google Search Rankings with Bright Data, Google Sheets & Gmail

### 1. Workflow Overview

This n8n workflow automates the process of tracking Google search rankings for a list of keywords and target domains, using Bright Data’s API to fetch Google search results, storing the results in Google Sheets, and emailing a summary report. It is designed for SEO professionals or marketers who want automated daily or on-demand rank tracking with email notifications.

The workflow consists of the following logical blocks:

- **1.1 Triggering Mechanism:** Manual trigger or scheduled trigger every 24 hours.
- **1.2 Input Retrieval:** Read keywords and target domains from a Google Sheets document.
- **1.3 Keyword Preparation:** Transform keywords for URL encoding.
- **1.4 Batch Processing:** Loop over each keyword for individual processing.
- **1.5 Data Retrieval:** Use Bright Data API to fetch raw Google search HTML for each keyword.
- **1.6 Rank Extraction:** Parse search results HTML to find the rank of the target domain.
- **1.7 Results Storage:** Append rank results to a "Results" Google Sheets tab.
- **1.8 Email Preparation:** Format the collected rank results into an HTML table.
- **1.9 Email Sending:** Send the HTML table via Gmail to a configured recipient.

---

### 2. Block-by-Block Analysis

#### 2.1 Triggering Mechanism

- **Overview:** Provides two ways to start the workflow — manually via a button or automatically every 24 hours.
- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Schedule Trigger

##### When clicking ‘Test workflow’

- **Type & Role:** Manual trigger node; starts workflow execution on demand.
- **Configuration:** Default manual trigger, no parameters.
- **Connections:** Output → Reading Keywords node.
- **Edge Cases:** None specific; manual start only.
- **Version:** 1

##### Schedule Trigger

- **Type & Role:** Time-based trigger; fires workflow every 24 hours.
- **Configuration:** Interval set to 24 hours.
- **Connections:** Output → Reading Keywords node.
- **Edge Cases:** Trigger may fail if n8n instance is down; no retries configured.
- **Version:** 1.2

---

#### 2.2 Input Retrieval

- **Overview:** Fetches keywords and associated target domains from a Google Sheets document to provide the input dataset.
- **Nodes Involved:**  
  - Reading Keywords

##### Reading Keywords

- **Type & Role:** Google Sheets node; reads rows from a specified sheet.
- **Configuration:**  
  - Document ID received via credentials (`googleSheetDocId` credential).  
  - Sheet name set to the sheet containing keywords (`Keyword` sheet).  
- **Credentials:** Google Sheets OAuth2.
- **Key Variables:** Outputs fields such as `Keyword` and `Domain`.
- **Connections:** Output → Transforming Keywords node.
- **Edge Cases:**  
  - Authentication failure if credentials expire.  
  - Empty or malformed sheet data.  
  - Sheet name or document ID misconfiguration.
- **Version:** 4.5

---

#### 2.3 Keyword Preparation

- **Overview:** Transforms the keywords into URL-encoded format by replacing spaces with plus signs, preparing them for search query URLs.
- **Nodes Involved:**  
  - Transforming Keywords

##### Transforming Keywords

- **Type & Role:** Code node; processes each keyword string.
- **Configuration:**  
  - JavaScript code replaces all spaces in `Keyword` with `+`.  
  - Preserves original keyword and appends `transformedKeyword` field.
- **Connections:** Output → Loop over Keywords node.
- **Edge Cases:**  
  - Keywords missing or empty strings handled by fallback to empty string.  
  - Expression errors if input data structure changes.
- **Version:** 2

---

#### 2.4 Batch Processing

- **Overview:** Splits the keyword list into batches for sequential processing. In this workflow, the default batch size is used (processes one by one).
- **Nodes Involved:**  
  - Loop over Keywords

##### Loop over Keywords

- **Type & Role:** SplitInBatches node; iterates over items one at a time.
- **Configuration:** Default options (no batch size override).
- **Connections:** Outputs two branches:  
  - → Making Email Template (for final email composition)  
  - → Getting Ranks (to fetch rank data per keyword)
- **Edge Cases:**  
  - Large keyword lists may cause long execution times.  
  - Batch settings could be adjusted for efficiency.
- **Version:** 3

---

#### 2.5 Data Retrieval

- **Overview:** Queries Bright Data’s API to get the raw HTML of Google search results for each transformed keyword.
- **Nodes Involved:**  
  - Getting Ranks

##### Getting Ranks

- **Type & Role:** HTTP Request node; sends POST requests to Bright Data API.
- **Configuration:**  
  - URL: `https://api.brightdata.com/request`  
  - Method: POST  
  - JSON body includes zone `"serp_n8n"` and URL constructed using transformed keyword and location parameter `gl=US`.  
  - Authorization header with Bearer token from `brightDataApiKey` credential.
- **Credentials:** Bright Data API key.
- **Connections:** Output → Rank Finder node.
- **Edge Cases:**  
  - API authentication failure or quota exceeded.  
  - Network timeouts or Bright Data service errors.  
  - Changes in Bright Data API specification.
- **Version:** 4.2

---

#### 2.6 Rank Extraction

- **Overview:** Parses the raw HTML search results to extract all outbound links, filters out Google URLs, and determines the rank position of the target domain.
- **Nodes Involved:**  
  - Rank Finder

##### Rank Finder

- **Type & Role:** Code node; parses HTML and finds the rank of the target domain.
- **Configuration:**  
  - Uses regular expressions to extract URLs from anchor tags.  
  - Filters out Google internal URLs (e.g., google.com, /search?, webcache).  
  - Finds the index (rank) where the target domain appears.  
  - Outputs rank, URL found, total results checked, boolean found flag, and current timestamp.
- **Key Variables:**  
  - `targetDomain` from the original Keywords sheet.  
  - Input HTML from Bright Data API response.
- **Connections:** Output → Post Rank Results node.
- **Edge Cases:**  
  - Target domain not found → rank set to "Not Ranked".  
  - HTML structure changes affecting regex parsing.  
  - Empty or malformed HTML input.
- **Version:** 2

---

#### 2.7 Results Storage

- **Overview:** Appends the rank results for each keyword into a "Results" sheet in the Google Sheets document.
- **Nodes Involved:**  
  - Post Rank Results

##### Post Rank Results

- **Type & Role:** Google Sheets node; appends rows.
- **Configuration:**  
  - Document ID: same as keywords sheet.  
  - Sheet name: "Results".  
  - Appends columns: `Keyword`, `Domain`, `rank`, `url`, `totalResultsChecked`, `found`, `checkedAt`.  
  - Uses values mapped from both current node’s output and the original keywords.
- **Credentials:** Google Sheets OAuth2.
- **Connections:** Output → Loop over Keywords node (to continue batches).
- **Edge Cases:**  
  - Authentication failure.  
  - Sheet missing or columns renamed.  
  - Data type mismatches.
- **Version:** 4.5

---

#### 2.8 Email Preparation

- **Overview:** Generates an HTML table summarizing the keywords and their ranks for email reporting.
- **Nodes Involved:**  
  - Making Email Template

##### Making Email Template

- **Type & Role:** Code node; constructs an HTML table from rank data.
- **Configuration:**  
  - Maps through all items to extract `Keyword` and `rank`.  
  - Builds an HTML table string with headers and rows.  
  - Returns JSON with `emailBody` containing the HTML.
- **Connections:** Output → Sending Email Message node.
- **Edge Cases:**  
  - Empty input array leads to empty table.  
  - Unexpected data shapes could break rendering.
- **Version:** 2

---

#### 2.9 Email Sending

- **Overview:** Sends the HTML email to a configured recipient using Gmail.
- **Nodes Involved:**  
  - Sending Email Message

##### Sending Email Message

- **Type & Role:** Gmail node; sends an email.
- **Configuration:**  
  - Recipient email address from credential `recipientEmail`.  
  - Email body from `emailBody` produced by the previous node.  
  - Subject set to "Ranked".  
- **Credentials:** Gmail OAuth2.
- **Connections:** None (terminal node).
- **Edge Cases:**  
  - Authentication failure or token expiry.  
  - Gmail API quota or limits.  
  - Recipient email misconfiguration.
- **Version:** 2.1

---

### 3. Summary Table

| Node Name                  | Node Type           | Functional Role                  | Input Node(s)          | Output Node(s)            | Sticky Note                                                                                                                     |
|----------------------------|---------------------|--------------------------------|------------------------|---------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger      | Manual start trigger            | —                      | Reading Keywords           |                                                                                                                               |
| Schedule Trigger           | Schedule Trigger    | Automated daily trigger         | —                      | Reading Keywords           |                                                                                                                               |
| Reading Keywords           | Google Sheets       | Reads keywords & domains        | When clicking ‘Test workflow’, Schedule Trigger | Transforming Keywords      |                                                                                                                               |
| Transforming Keywords      | Code                | Prepares keywords for URL       | Reading Keywords        | Loop over Keywords         |                                                                                                                               |
| Loop over Keywords         | SplitInBatches      | Processes keywords individually | Transforming Keywords   | Making Email Template, Getting Ranks |                                                                                                                               |
| Getting Ranks              | HTTP Request        | Calls Bright Data API           | Loop over Keywords      | Rank Finder                |                                                                                                                               |
| Rank Finder                | Code                | Parses search results & finds rank | Getting Ranks           | Post Rank Results          |                                                                                                                               |
| Post Rank Results          | Google Sheets       | Saves rank data to results sheet| Rank Finder             | Loop over Keywords         |                                                                                                                               |
| Making Email Template      | Code                | Generates HTML email content    | Loop over Keywords      | Sending Email Message      |                                                                                                                               |
| Sending Email Message      | Gmail               | Sends rank results via email   | Making Email Template   | —                         |                                                                                                                               |
| Sticky Note               | Sticky Note         | Contains workflow overview and setup guide | —                      | —                         | # n8n Workflow Explanation & Setup Guide ... (detailed content with setup instructions and location change info)              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes**  
   - Add a **Manual Trigger** node named `When clicking ‘Test workflow’` (default config).  
   - Add a **Schedule Trigger** node named `Schedule Trigger` with a 24-hour interval.

2. **Setup Google Sheets Input Node**  
   - Add a **Google Sheets** node named `Reading Keywords`.  
   - Configure to read from the Google Sheets document by specifying the `documentId` credential (create or use existing Google Sheets OAuth2 credentials).  
   - Set the sheet name to the one containing your keywords (e.g., "Keyword").  
   - Ensure the sheet has columns `Keyword` and `Domain`.

3. **Add Keyword Transformation Node**  
   - Add a **Code** node named `Transforming Keywords`.  
   - Paste the JavaScript code to replace spaces with `+` in the `Keyword` field and add a `transformedKeyword` property.  
   - Connect output of `Reading Keywords` → `Transforming Keywords`.

4. **Add Batch Processing Node**  
   - Add a **SplitInBatches** node named `Loop over Keywords`.  
   - Default batch size is 1.  
   - Connect output of `Transforming Keywords` → `Loop over Keywords`.

5. **Add Bright Data API Request Node**  
   - Add an **HTTP Request** node named `Getting Ranks`.  
   - Method: POST  
   - URL: `https://api.brightdata.com/request`  
   - Request Body (JSON):  
     ```json
     {
       "zone": "serp_n8n",
       "url": "https://www.google.com/search?q={{ $json.transformedKeyword }}&gl=US",
       "format": "raw"
     }
     ```  
   - Add header: `Authorization: Bearer <your_brightdata_api_key>` (configure credentials accordingly).  
   - Connect one output of `Loop over Keywords` → `Getting Ranks`.

6. **Add Rank Parsing Node**  
   - Add a **Code** node named `Rank Finder`.  
   - Paste JavaScript code that parses the HTML from Bright Data, extracts URLs, filters Google URLs, and finds the rank of the target domain.  
   - Inputs include raw HTML (`data` field) and `Domain` from the original keywords.  
   - Connect output of `Getting Ranks` → `Rank Finder`.

7. **Add Google Sheets Append Node**  
   - Add a **Google Sheets** node named `Post Rank Results`.  
   - Configure to append rows to the same Google Sheets document, sheet named "Results".  
   - Map columns: `Keyword`, `Domain`, `rank`, `url`, `totalResultsChecked`, `found`, `checkedAt`.  
   - Connect output of `Rank Finder` → `Post Rank Results`.  
   - Connect output of `Post Rank Results` → back to `Loop over Keywords` (to continue batch processing).

8. **Add Email Template Node**  
   - Add a **Code** node named `Making Email Template`.  
   - Paste JavaScript code that generates an HTML table from the batch results with columns `Keyword` and `rank`.  
   - Connect the second output of `Loop over Keywords` → `Making Email Template`.

9. **Add Gmail Node for Email Sending**  
   - Add a **Gmail** node named `Sending Email Message`.  
   - Configure Gmail OAuth2 credentials.  
   - Set `sendTo` to your recipient email (from credentials or manually).  
   - Set `message` to use `emailBody` from `Making Email Template`.  
   - Set subject to "Ranked".  
   - Connect output of `Making Email Template` → `Sending Email Message`.

10. **Connect Triggers**  
    - Connect both `When clicking ‘Test workflow’` and `Schedule Trigger` nodes to `Reading Keywords`.

11. **Verify and Test**  
    - Test manually and schedule to ensure data flows correctly, ranks are fetched, stored, and email sent.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                    | Context or Link                                                                                     |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| The workflow includes detailed instructions within a sticky note on how to set up the Google Sheets document, Bright Data API credentials, and Gmail OAuth2 credentials. It also explains how to change the Google Search location by modifying the `gl` parameter in the Bright Data API HTTP request.                                                                                | Embedded sticky note in workflow nodes.                                                           |
| The workflow requires valid and active API credentials for Google Sheets, Bright Data, and Gmail. Ensure APIs and OAuth tokens are properly configured to avoid authentication errors.                                                                                                                                                                                        | Credential setup prerequisite.                                                                    |
| For changing Google Search location, supported country codes include US, GB, CA, AU. Modify the URL parameter `gl` accordingly in the HTTP request node (`Getting Ranks`).                                                                                                                                                                                                      | Location customization note in sticky note.                                                      |
| Bright Data’s API may have usage quotas or costs; monitor usage to avoid unexpected charges.                                                                                                                                                                                                                                                                                    | Best practice reminder.                                                                            |
| The workflow assumes the Google Sheets input has at least two columns: `Keyword` and `Domain`. The "Results" sheet must exist with appropriate columns for appending data.                                                                                                                                                                                                      | Google Sheets structure requirement.                                                             |
| This workflow can be adapted to other search engines or data providers by adjusting the HTTP Request node and parsing logic accordingly.                                                                                                                                                                                                                                       | Extensibility comment.                                                                            |

---

**Disclaimer:** The text provided is extracted exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and does not contain any illegal, offensive, or protected elements. All data handled is legal and public.