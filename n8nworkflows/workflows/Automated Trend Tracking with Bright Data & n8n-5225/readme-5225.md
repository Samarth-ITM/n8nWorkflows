Automated Trend Tracking with Bright Data & n8n

https://n8nworkflows.xyz/workflows/automated-trend-tracking-with-bright-data---n8n-5225


# Automated Trend Tracking with Bright Data & n8n

---

## 1. Workflow Overview

This workflow, titled **Automated Trend Tracking with Bright Data & n8n**, is designed to automate the monthly collection and archiving of viral trend data from Quora on specified topics (e.g., marketing). It leverages Bright Data’s scraping capabilities to bypass web restrictions and extract meaningful content without manual intervention. The workflow is structured into two main logical blocks:

- **1.1 Monthly Trigger and Data Scraping:** Automatically triggers the workflow monthly and fetches raw HTML data from Quora search results using Bright Data’s proxy-powered scraping API.

- **1.2 Data Extraction and Storage:** Parses the scraped HTML to extract relevant trend details such as question titles and answer counts, then appends this structured data into a Google Sheet for archival and further analysis.

This modular structure ensures reliable, scheduled data acquisition with subsequent transformation and persistence, enabling users to maintain an up-to-date database of viral Quora content for marketing insights or content strategy.

---

## 2. Block-by-Block Analysis

### 2.1 Monthly Trigger and Data Scraping

**Overview:**  
This block initiates the workflow automatically on a monthly basis and sends a POST request to Bright Data’s API to scrape the Quora search results page for a given topic. It handles dynamic content rendering and proxy bypassing.

**Nodes Involved:**  
- Monthly Trend Trigger1  
- Scrape Quora Trends (Bright Data)

#### Node: Monthly Trend Trigger1

- **Type & Role:** Schedule Trigger node; initiates workflow execution automatically based on a time schedule.
- **Configuration:** Set to trigger monthly, specifically once every month at 9:00 AM (hour 9), using n8n's interval-based schedule rule.
- **Expressions/Variables:** None; uses static schedule configuration.
- **Input Connections:** None (entry point).
- **Output Connections:** Outputs to `Scrape Quora Trends (Bright Data)`.
- **Version Requirements:** Uses version 1.2 of the scheduleTrigger node.
- **Potential Failures:**  
  - Cron misconfiguration could cause missed or multiple triggers.  
  - Timezone discrepancies if n8n instance is in a different timezone than expected.
- **Sub-workflows:** None.

#### Node: Scrape Quora Trends (Bright Data)

- **Type & Role:** HTTP Request node; performs a POST request to Bright Data’s scraping API to retrieve raw HTML of Quora search results.
- **Configuration:**  
  - Method: POST  
  - URL: `https://api.brightdata.com/request`  
  - Body Parameters:  
    - `zone`: "n8n_unblocker" (Bright Data proxy zone)  
    - `url`: `https://www.quora.com/search?q=marketing` (target Quora search URL; topic can be customized here)  
    - `country`: "us" (geolocation for scraping)  
    - `format`: "raw" (request raw HTML output)  
    - `render`: "true" (enable JavaScript rendering for dynamic content)  
  - Header Parameters: Authorization Bearer token for Bright Data API access.
- **Expressions/Variables:** Hardcoded URL and parameters but can be parameterized for different topics.
- **Input Connections:** Receives trigger from `Monthly Trend Trigger1`.
- **Output Connections:** Outputs raw HTML response to `Extract Post Titles & Stats`.
- **Version Requirements:** Uses version 4.2 of HTTP Request node.
- **Potential Failures:**  
  - Authentication failure if API token expires or is invalid.  
  - Network timeout or request throttling by Bright Data.  
  - API quota limits reached.  
  - Changes in Quora page structure may affect scraping efficacy.  
  - Incorrect proxy zone configuration.
- **Sub-workflows:** None.

---

### 2.2 Data Extraction and Storage

**Overview:**  
This block processes the raw HTML content received from Bright Data, extracts structured information such as post titles and answer counts, and appends this data into a Google Sheet for tracking and analysis.

**Nodes Involved:**  
- Extract Post Titles & Stats  
- Save Trends to Sheet

#### Node: Extract Post Titles & Stats

- **Type & Role:** HTML Extract node; parses raw HTML to extract specific content using CSS selectors.
- **Configuration:**  
  - Operation: Extract HTML content.  
  - Extraction Values:  
    - `Title`: Extracts question titles or headlines using a CSS selector referencing a specific `<span>` element with inline style.  
    - `Answers`: Extracts answer counts via anchor tags with certain class names and attributes.
- **Expressions/Variables:** Uses CSS selectors directly in configuration, e.g.,  
  ```css
  <span style="background: none;">What exactly is digital marketing</span>
  ```  
  and  
  ```css
  <a class="q-box qu-cursor--pointer qu-hover--textDecoration--underline b2c1r2a puppeteer_test_link" href="..." ...>3.2K answers</a>
  ```
- **Input Connections:** Receives raw HTML from `Scrape Quora Trends (Bright Data)`.
- **Output Connections:** Sends structured JSON with extracted fields to `Save Trends to Sheet`.
- **Version Requirements:** Version 1.2 of the HTML Extract node.
- **Potential Failures:**  
  - CSS selectors may fail if Quora changes its page DOM structure.  
  - Extraction may return empty or partial data if HTML is malformed or incomplete.  
  - Node may not handle multiple posts unless configured for multiple extraction.
- **Sub-workflows:** None.

#### Node: Save Trends to Sheet

- **Type & Role:** Google Sheets node; appends extracted trend data to a specified Google Sheet.
- **Configuration:**  
  - Operation: Append data to sheet.  
  - Document ID: Google Sheets document identified by ID (linked to a specific spreadsheet).  
  - Sheet Name: Uses sheet with `gid=0` (usually the first sheet).  
  - Columns Mapped:  
    - "Title" mapped to extracted Title field.  
    - "Answer" mapped to extracted Answers field.  
  - Mapping Mode: Defined explicitly with columns.  
  - Credential: Uses OAuth2 credentials for Google Sheets (named "Google Sheets account").
- **Expressions/Variables:** Uses expression to assign values from previous node JSON:  
  - Title: `={{ $json.Title }}`  
  - Answer: `={{ $json.Answers }}`
- **Input Connections:** Receives structured data from `Extract Post Titles & Stats`.
- **Output Connections:** None (end of workflow).
- **Version Requirements:** Version 4.6 of Google Sheets node.
- **Potential Failures:**  
  - Authentication errors if OAuth token expires or is revoked.  
  - Sheet ID or name issues if sheet is deleted or renamed.  
  - Permission errors if credentials lack required access.  
  - Data type mismatches if extracted data does not fit expected schema.
- **Sub-workflows:** None.

---

## 3. Summary Table

| Node Name                     | Node Type               | Functional Role                          | Input Node(s)             | Output Node(s)                | Sticky Note                                                                                                                               |
|-------------------------------|-------------------------|----------------------------------------|---------------------------|------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------|
| Monthly Trend Trigger1         | Schedule Trigger        | Monthly workflow trigger                |                           | Scrape Quora Trends (Bright Data) | Provides fully automated, monthly execution to ensure timely trend data scraping.                                                         |
| Scrape Quora Trends (Bright Data) | HTTP Request           | Scrapes Quora search results via Bright Data API | Monthly Trend Trigger1     | Extract Post Titles & Stats   | Sends POST request to Bright Data to bypass restrictions and get rendered HTML content for specified Quora topic.                         |
| Extract Post Titles & Stats    | HTML Extract            | Extracts question titles and answers from scraped HTML | Scrape Quora Trends (Bright Data) | Save Trends to Sheet          | Parses raw HTML to pull out meaningful data using CSS selectors, transforming messy HTML into structured trend info.                      |
| Save Trends to Sheet           | Google Sheets           | Appends extracted data to Google Sheet | Extract Post Titles & Stats |                              | Stores structured trend data monthly for archival, analysis, and reporting.                                                                |
| Sticky Note9                  | Sticky Note             | Workflow assistance/contact information |                           |                              | For questions/support contact Yaron@nofluff.online; links to YouTube and LinkedIn tutorials.                                              |
| Sticky Note4                  | Sticky Note             | Full workflow description and use case |                           |                              | Detailed description of the Monthly Viral Trend Tracker workflow, including sections and node explanations.                               |
| Sticky Note3                  | Sticky Note             | Section 1 explanation (Trigger & Scraper) |                           |                              | Explains purpose and role of Monthly Trend Trigger and Bright Data scraping nodes.                                                         |
| Sticky Note5                  | Sticky Note             | Section 2 explanation (Extract & Save)  |                           |                              | Describes extraction and storage nodes, highlighting their roles in transforming and saving data.                                          |
| Sticky Note                   | Sticky Note             | Affiliate link for Bright Data          |                           |                              | Contains referral link to Bright Data service: https://get.brightdata.com/1tndi4600b25                                                     |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it appropriately (e.g., "Trend_Tracker_via_Bright_Data").

2. **Add the "Schedule Trigger" node:**
   - Set node name: `Monthly Trend Trigger1`.
   - Configure the trigger rule to execute monthly:
     - Select "Interval" trigger.
     - Set interval to 1 month(s).
     - Set trigger hour to 9 (9:00 AM).
   - No credentials needed.

3. **Add an "HTTP Request" node:**
   - Name it `Scrape Quora Trends (Bright Data)`.
   - Set method to `POST`.
   - URL: `https://api.brightdata.com/request`.
   - In Body Parameters (as JSON or form fields depending on n8n version):
     - `zone`: `"n8n_unblocker"` (your Bright Data proxy zone).
     - `url`: `"https://www.quora.com/search?q=marketing"` (customize topic keyword as needed).
     - `country`: `"us"`.
     - `format`: `"raw"`.
     - `render`: `"true"` (enables JS rendering).
   - In Header Parameters:
     - `Authorization`: `Bearer YOUR_BRIGHT_DATA_API_TOKEN` (replace with your Bright Data API token).
   - Connect output of `Monthly Trend Trigger1` to this node's input.
   - Use HTTP Request node version >= 4.2.

4. **Add an "HTML Extract" node:**
   - Name it `Extract Post Titles & Stats`.
   - Set operation to "Extract HTML Content".
   - Define extraction values with at least two keys:
     - `Title`: Use CSS selector targeting question titles (e.g., specific `<span>` element or dynamic selector matching Quora's title structure).
     - `Answers`: Use CSS selector targeting answer count links (anchor tags with appropriate classes).
   - Connect output of `Scrape Quora Trends (Bright Data)` to this node's input.
   - Use HTML Extract node version >= 1.2.
   - Adjust CSS selectors as needed, depending on Quora's current DOM.

5. **Add a "Google Sheets" node:**
   - Name it `Save Trends to Sheet`.
   - Set operation to `Append`.
   - Select or enter your Google Sheets document ID (from your Google Sheet URL).
   - Choose sheet name or GID (e.g., `gid=0` for the first sheet).
   - Map columns:
     - Map `Title` column to `{{$json["Title"]}}`.
     - Map `Answer` column to `{{$json["Answers"]}}`.
   - Configure Google Sheets OAuth2 credentials:
     - Create or select existing OAuth2 credentials with write access to the target sheet.
   - Connect output of `Extract Post Titles & Stats` to this node's input.
   - Use Google Sheets node version >= 4.6.

6. **Save and activate your workflow.**

7. **Optional:**  
   - Add sticky notes for documentation and assistance as done in the original workflow.  
   - Customize scraping topic by changing the `url` parameter in the HTTP Request node.  
   - To extend for other platforms (Twitter, Reddit), duplicate and adjust the scraping nodes accordingly.

---

## 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                       |
|-----------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| For any questions or support, contact: Yaron@nofluff.online                                              | Workflow assistance and support contact                                                             |
| Explore more tips and tutorials by Yaron Been on YouTube and LinkedIn                                     | YouTube: https://www.youtube.com/@YaronBeen/videos<br>LinkedIn: https://www.linkedin.com/in/yaronbeen/ |
| Monthly Viral Trend Tracker automates scraping viral Quora posts monthly for trend research and content ideas | Workflow purpose and use case description                                                            |
| Referral affiliate link for Bright Data supporting free content creation                                   | https://get.brightdata.com/1tndi4600b25                                                             |

---

**Disclaimer:** The provided text is derived exclusively from an n8n automated workflow. It complies strictly with content policies, contains no illegal or protected material, and only processes legal and publicly available data.