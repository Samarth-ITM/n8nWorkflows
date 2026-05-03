Automated SEC Form D Filings Tracker with Google Sheets Integration

https://n8nworkflows.xyz/workflows/automated-sec-form-d-filings-tracker-with-google-sheets-integration-6755


# Automated SEC Form D Filings Tracker with Google Sheets Integration

### 1. Workflow Overview

This n8n workflow automates the tracking of SEC Form D filings, which are filings related to private placements and fundraising activities. It fetches the latest filings from the SEC EDGAR RSS feed every 10 minutes during U.S. business hours, extracts and formats key data fields, filters out duplicates, and appends new filings to a Google Sheets spreadsheet for ongoing monitoring. The workflow is aimed at venture capitalists, investment analysts, and anyone interested in tracking private placement filings in near real-time.

Logical blocks in the workflow:

- **1.1 Scheduled Trigger:** Runs the workflow every 10 minutes during business hours (6 AM to 9 PM EST, Monday to Friday).
- **1.2 SEC Data Fetching:** Retrieves the latest 40 Form D filings from the SEC EDGAR RSS feed, with proper request headers.
- **1.3 Data Parsing and Extraction:** Parses the XML RSS feed, extracts relevant filing data fields (CIK, title, form type, links, updated date), and formats them.
- **1.4 Duplicate Filtering:** Removes filings already processed in previous executions to avoid duplicate entries.
- **1.5 Data Storage:** Appends the filtered, new filings to a Google Sheets document via OAuth2 authentication.
- **1.6 Documentation (Sticky Notes):** Provides contextual information and instructions about workflow operation, configuration, and usage.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  Initiates the workflow automatically every 10 minutes during U.S. business hours (6 AM - 9 PM EST), Monday to Friday.

- **Nodes Involved:**  
  - Schedule: Every 10 min (Business Hours)

- **Node Details:**  
  - **Node Name:** Schedule: Every 10 min (Business Hours)  
  - **Type:** Schedule Trigger  
  - **Configuration:**  
    - Cron expression: `*/10 6-21 * * 1-5`  
    - Meaning: Run every 10 minutes (`*/10`) during the 6 AM to 9 PM hours (`6-21`) on Monday to Friday (`1-5`)  
  - **Inputs:** None (trigger node)  
  - **Outputs:** Triggers "Fetch SEC Form D Filings"  
  - **Version:** 1.2  
  - **Potential Failures:**  
    - Cron misconfiguration might cause unexpected scheduling  
    - Workflow will not run outside configured hours/days  
  - **Sticky Notes:**  
    - Covers schedule details, customization advice, and SEC server load respect (Sticky Note1)

#### 1.2 SEC Data Fetching

- **Overview:**  
  Makes an HTTP GET request to the SEC EDGAR RSS feed endpoint to retrieve the 40 most recent Form D filings in RSS/XML format. Includes mandatory headers to comply with SEC server policies.

- **Nodes Involved:**  
  - Fetch SEC Form D Filings

- **Node Details:**  
  - **Node Name:** Fetch SEC Form D Filings  
  - **Type:** HTTP Request  
  - **Configuration:**  
    - URL: `https://www.sec.gov/cgi-bin/browse-edgar?action=getcurrent&CIK=&type=D&company=&dateb=&owner=include&start=0&count=40&output=atom`  
    - Headers:  
      - `User-Agent`: "iRocket VC matthew@irocket.vc" (required by SEC for identification)  
      - `Accept-Encoding`: "gzip, deflate" to accept compressed responses  
    - Method: GET (default)  
    - Options: No special options enabled  
  - **Inputs:** Trigger from Schedule  
  - **Outputs:** Sends response data to "Parse SEC RSS Feed"  
  - **Version:** 4.2  
  - **Potential Failures:**  
    - HTTP errors (e.g., 429 Too Many Requests if SEC blocks due to too frequent requests)  
    - Missing or improper User-Agent header may cause request failure or denial  
    - Network issues or SEC server downtime  
  - **Sticky Notes:**  
    - Highlights the need for User-Agent header and RSS/XML response format (Sticky Note2)

#### 1.3 Data Parsing and Extraction

- **Overview:**  
  Parses the XML RSS feed response and extracts an array of filing entries. Each entry is converted into a flat object containing key properties relevant for tracking and storage.

- **Nodes Involved:**  
  - Parse SEC RSS Feed  
  - Extract & Format Filing Data

- **Node Details:**  
  - **Node Name:** Parse SEC RSS Feed  
  - **Type:** XML Parser  
  - **Configuration:** Default XML parsing options to convert RSS XML to JSON  
  - **Inputs:** HTTP response from "Fetch SEC Form D Filings"  
  - **Outputs:** JSON feed object to "Extract & Format Filing Data"  
  - **Version:** 1  
  - **Potential Failures:**  
    - Malformed XML response could cause parsing errors  
    - Empty or unexpected feed structure  
  - **Sticky Notes:** None directly, but part of the data extraction block

  - **Node Name:** Extract & Format Filing Data  
  - **Type:** Code (JavaScript)  
  - **Configuration:**  
    - Custom JavaScript code to:  
      - Access `feed.entry` array from parsed feed  
      - Extract CIK number from filing title via regex (removes leading zeros)  
      - Extract filing type, title, updated date  
      - Construct both HTML and TXT filing links by manipulating the provided URL  
      - Return array of simplified objects with fields: `cikNumber`, `title`, `formType`, `filingLinkHtml`, `filingLinkTxt`, `updated`  
  - **Key Expressions/Variables:**  
    - Regex: `/\((\d{10})\)/` to find CIK in title  
    - Link transformation: replace `-index.htm` suffix with `.txt` for TXT link  
  - **Inputs:** JSON from "Parse SEC RSS Feed"  
  - **Outputs:** Flat JSON array to "Filter New Filings Only"  
  - **Version:** 2  
  - **Potential Failures:**  
    - Regex may fail if titles have unexpected format, resulting in empty CIK  
    - Missing or changed feed structure can break the code  
  - **Sticky Notes:**  
    - Notes on data extraction, CIK parsing, link formatting, and output fields (Sticky Note3)

#### 1.4 Duplicate Filtering

- **Overview:**  
  Removes filings that have been processed in previous workflow runs based on the TXT filing link, ensuring only new filings proceed.

- **Nodes Involved:**  
  - Filter New Filings Only

- **Node Details:**  
  - **Node Name:** Filter New Filings Only  
  - **Type:** Remove Duplicates  
  - **Configuration:**  
    - Operation: Remove items seen in previous executions  
    - Field for deduplication: `filingLinkTxt` (TXT filing link)  
  - **Inputs:** JSON array of filings from "Extract & Format Filing Data"  
  - **Outputs:** Filtered new filings to "Save to SEC Data Sheet"  
  - **Version:** 2  
  - **Potential Failures:**  
    - Data without `filingLinkTxt` or with inconsistent links could cause duplicate tracking errors  
    - If workflow state is lost or reset, duplicates may reappear  
  - **Sticky Notes:** None

#### 1.5 Data Storage

- **Overview:**  
  Appends new SEC Form D filings to a Google Sheets document with specific columns and OAuth2 authentication.

- **Nodes Involved:**  
  - Save to SEC Data Sheet

- **Node Details:**  
  - **Node Name:** Save to SEC Data Sheet  
  - **Type:** Google Sheets  
  - **Configuration:**  
    - Operation: Append rows  
    - Sheet Name: `gid=0` (default first sheet)  
    - Document ID: `1VoGfVpk1mMrqKIc5hsO7peYuLx0SwhsbW7uUeYJCmrU` (template sheet)  
    - Columns mapped: `title`, `updated`, `formType`, `cikNumber`, `filingLinkTxt`, `filingLinkHtml`  
    - Mapping mode: Define below (explicit column mapping)  
  - **Credentials:** Google Sheets OAuth2 (named "[Naveen]Google Sheets account")  
  - **Inputs:** Filtered filings from "Filter New Filings Only"  
  - **Outputs:** None (end node)  
  - **Version:** 4.6  
  - **Potential Failures:**  
    - OAuth token expiration or permission issues  
    - Incorrect sheet ID or name may cause write failures  
    - Rate limits or quota restrictions on Google Sheets API  
  - **Sticky Notes:**  
    - Instructions for using the Google Sheets template, making a copy, and updating the node with the copied sheet ID (Sticky Note4)  

#### 1.6 Documentation (Sticky Notes)

- **Overview:**  
  Provides detailed notes on workflow purpose, scheduling, data fetching, extraction, and output configuration for users.

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1  
  - Sticky Note2  
  - Sticky Note3  
  - Sticky Note4

- **Node Details:**  
  Each sticky note covers a specific topic:  
  - Sticky Note: Overall workflow summary and target audience  
  - Sticky Note1: Scheduling details and customization tips  
  - Sticky Note2: SEC feed fetching and header requirements  
  - Sticky Note3: Data extraction details and output fields  
  - Sticky Note4: Google Sheets output instructions and template link

- **Position:** Placed visually near relevant nodes for easy reference.

---

### 3. Summary Table

| Node Name                      | Node Type           | Functional Role                 | Input Node(s)                  | Output Node(s)                 | Sticky Note                                                                                           |
|-------------------------------|---------------------|--------------------------------|-------------------------------|-------------------------------|-----------------------------------------------------------------------------------------------------|
| Schedule: Every 10 min (Business Hours) | Schedule Trigger     | Triggers workflow on schedule  | None                          | Fetch SEC Form D Filings       | ⏰ AUTOMATED MONITORING: Schedule every 10 min, 6 AM - 9 PM EST, Mon-Fri; customize timing; respects SEC server load |
| Fetch SEC Form D Filings       | HTTP Request        | Fetches latest SEC Form D RSS  | Schedule: Every 10 min         | Parse SEC RSS Feed             | 🌐 SEC EDGAR FEED: Fetches 40 most recent filings with User-Agent header required                     |
| Parse SEC RSS Feed             | XML Parser          | Parses SEC RSS XML feed        | Fetch SEC Form D Filings       | Extract & Format Filing Data   |                                                                                                     |
| Extract & Format Filing Data   | Code (JavaScript)   | Extracts and formats filing data | Parse SEC RSS Feed             | Filter New Filings Only        | ⚙️ DATA EXTRACTION: Extracts CIK, generates HTML & TXT links, formats data for Google Sheets         |
| Filter New Filings Only        | Remove Duplicates   | Filters out already processed filings | Extract & Format Filing Data   | Save to SEC Data Sheet         |                                                                                                     |
| Save to SEC Data Sheet         | Google Sheets       | Appends new filings to Google Sheet | Filter New Filings Only        | None                          | 📋 GOOGLE SHEETS OUTPUT: Appends new filings, requires OAuth, uses template sheet link               |
| Sticky Note                   | Sticky Note         | Workflow overview and purpose  | None                          | None                          | 🏛️ SEC FORM D FILING TRACKER: Monitors filings, runs every 10 min, filters duplicates, saves to Sheets |
| Sticky Note1                  | Sticky Note         | Scheduling details and customization | None                          | None                          | ⏰ AUTOMATED MONITORING: Schedule details and customization tips                                     |
| Sticky Note2                  | Sticky Note         | SEC feed HTTP request notes    | None                          | None                          | 🌐 SEC EDGAR FEED: User-Agent header required                                                       |
| Sticky Note3                  | Sticky Note         | Data extraction notes          | None                          | None                          | ⚙️ DATA EXTRACTION: CIK extraction, link formatting                                                |
| Sticky Note4                  | Sticky Note         | Google Sheets output instructions | None                          | None                          | 📋 GOOGLE SHEETS OUTPUT: Template link and usage instructions                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Name: "Schedule: Every 10 min (Business Hours)"  
   - Configure cron expression: `*/10 6-21 * * 1-5` (Every 10 minutes, 6AM-9PM, Monday-Friday)  

2. **Create HTTP Request Node**  
   - Type: HTTP Request  
   - Name: "Fetch SEC Form D Filings"  
   - URL: `https://www.sec.gov/cgi-bin/browse-edgar?action=getcurrent&CIK=&type=D&company=&dateb=&owner=include&start=0&count=40&output=atom`  
   - Method: GET (default)  
   - Headers:  
     - User-Agent: `<your email or identifier>` (e.g., "iRocket VC matthew@irocket.vc")  
     - Accept-Encoding: gzip, deflate  
   - Connect output of Schedule node to this node's input  

3. **Create XML Parse Node**  
   - Type: XML  
   - Name: "Parse SEC RSS Feed"  
   - Use default XML parsing settings  
   - Connect output of HTTP Request node to this node's input  

4. **Create Code Node**  
   - Type: Code (JavaScript)  
   - Name: "Extract & Format Filing Data"  
   - Code:  
     ```javascript
     const entries = $('Parse SEC RSS Feed').first().json.feed.entry;

     return entries.map(entry => {
       const titleMatch = entry.title.match(/\((\d{10})\)/);
       const cikNumber = titleMatch ? titleMatch[1].replace(/^0+/, '') : '';
       const htmlLink = entry.link.href;
       const txtLink = htmlLink.replace('-index.htm', '.txt');

       return {
         cikNumber: cikNumber || '',
         title: entry.title,
         formType: entry.category.term,
         filingLinkHtml: htmlLink,
         filingLinkTxt: txtLink,
         updated: entry.updated
       };
     });
     ```  
   - Connect output of XML Parse node to this code node  

5. **Create Remove Duplicates Node**  
   - Type: Remove Duplicates  
   - Name: "Filter New Filings Only"  
   - Operation: Remove items seen in previous executions  
   - Deduplication field: `filingLinkTxt`  
   - Connect output of Code node to this node  

6. **Create Google Sheets Node**  
   - Type: Google Sheets  
   - Name: "Save to SEC Data Sheet"  
   - Operation: Append  
   - Document ID: Use your Google Sheets document ID (copy of template recommended)  
   - Sheet Name: Use `gid=0` or appropriate sheet tab  
   - Columns mapping (define below):  
     - `title` → title  
     - `updated` → updated  
     - `formType` → formType  
     - `cikNumber` → cikNumber  
     - `filingLinkTxt` → filingLinkTxt  
     - `filingLinkHtml` → filingLinkHtml  
   - Credentials: Set up Google Sheets OAuth2 with appropriate permissions  
   - Connect output of Remove Duplicates node to this node  

7. **Add Sticky Notes (Optional)**  
   - Add sticky notes at relevant positions for documentation and instructions, including:  
     - Workflow overview and purpose  
     - Scheduling details and customization  
     - SEC feed fetch notes (User-Agent header)  
     - Data extraction notes  
     - Google Sheets output instructions and template link  

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                                  |
|----------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| The workflow runs every 10 minutes during business hours to respect SEC server load and prevent blocking.      | Scheduling rationale                                                                                             |
| User-Agent header is mandatory for SEC EDGAR HTTP requests to prevent request denial.                          | https://www.sec.gov/os/accessing-edgar-data                                                                        |
| Google Sheets template used: https://docs.google.com/spreadsheets/d/1VoGfVpk1mMrqKIc5hsO7peYuLx0SwhsbW7uUeYJCmrU | Used for storing SEC Form D filing data                                                                          |
| To customize the schedule, modify the cron expression in the Schedule Trigger node accordingly.                | Cron expression format reference: https://crontab.guru/                                                          |
| Ensure Google Sheets OAuth credentials have write access to the target spreadsheet, or appending will fail.   | Google Cloud Console and OAuth2 setup instructions                                                                |
| The workflow uses deduplication based on the TXT filing link; changes in SEC feed URL format may require updates to the deduplication field or extraction logic. | Important for maintaining data integrity                                                                        |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected material. All handled data is legal and public.