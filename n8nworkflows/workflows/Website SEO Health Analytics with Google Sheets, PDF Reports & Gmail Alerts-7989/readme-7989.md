Website SEO Health Analytics with Google Sheets, PDF Reports & Gmail Alerts

https://n8nworkflows.xyz/workflows/website-seo-health-analytics-with-google-sheets--pdf-reports---gmail-alerts-7989


# Website SEO Health Analytics with Google Sheets, PDF Reports & Gmail Alerts

### 1. Workflow Overview

This automated workflow performs daily SEO health monitoring on a list of websites stored in a Google Sheet. It analyzes each website’s SEO fundamentals by fetching and parsing their HTML content, runs a PageSpeed Insights performance check, and evaluates the overall SEO health score. Based on the results, it either sends alert emails for low-performing sites or generates detailed SEO reports saved as PDFs in Google Drive. The workflow also updates the performance log in the Google Sheet daily.

Logical blocks:

- **1.1 Daily Trigger & Input Fetching**: Trigger workflow daily at 9 AM and read website URLs from Google Sheets.
- **1.2 Configuration Attachment & Website HTML Retrieval**: Attach request config (timeout), then fetch raw HTML content for each site.
- **1.3 SEO HTML Analysis**: Parse the HTML to extract SEO-related metadata and perform internal link validation.
- **1.4 SEO Health Scoring & Performance Check**: Run PageSpeed Insights API with retries and backoff, producing a performance score.
- **1.5 Performance Threshold Branching**: Branch depending on whether performance score is below 50.
- **1.6 Alerting & Logging for Low Performance**: Send alert email if score < 50, then merge results for logging.
- **1.7 Report Generation for Good Performance**: Generate a detailed HTML report, convert it to PDF, download, and save to Google Drive.
- **1.8 Performance Log Update**: Update Google Sheet with latest performance scores and timestamps.

---

### 2. Block-by-Block Analysis

#### 2.1 Daily Trigger & Input Fetching

- **Overview**: Starts the workflow every day at 9 AM, then reads the list of websites to monitor from a Google Sheet.
- **Nodes Involved**: `Daily 9 AM Trigger`, `Fetch Website Links`
  
##### Node: Daily 9 AM Trigger
- Type: Cron Trigger
- Role: Initiates the workflow daily at 9 AM server time.
- Configuration: Trigger set at 9:00 hours daily.
- Input: None
- Output: Trigger signal to next node.
- Edge cases: Cron time zone mismatches or n8n downtime could delay execution.

##### Node: Fetch Website Links
- Type: Google Sheets (Read)
- Role: Reads website URLs from a specific sheet tab.
- Configuration: Reads from sheet "Website_Seo_Testing" in Google Sheet document ID `1cQ-TBf3-dqo7njDYzYpxpASYFvEp8lIzH7vpIqTLcwc`.
- Credentials: Uses Google Sheets OAuth2.
- Input: Trigger from previous node.
- Output: List of website URLs with their metadata.
- Edge cases: Google Sheets API rate limits, invalid/missing spreadsheet, or no URLs.

---

#### 2.2 Configuration Attachment & Website HTML Retrieval

- **Overview**: Attaches additional parameters to each site record (such as timeout), then fetches the raw HTML for each URL.
- **Nodes Involved**: `Attach Config & Timeout`, `Fetch Website HTML`

##### Node: Attach Config & Timeout
- Type: Set Node
- Role: Adds fields `siteUrl` (from "Website Link") and `timeoutMs` (fixed 2000 ms) to each item.
- Configuration: Extracts "Website Link" from Google Sheets and sets timeout to 2000 milliseconds.
- Input: Website URLs array.
- Output: Items with `siteUrl` and `timeoutMs`.
- Edge cases: Missing "Website Link" field, malformed URLs.

##### Node: Fetch Website HTML
- Type: HTTP Request
- Role: Performs HTTP GET request to fetch website HTML content.
- Configuration: URL from `siteUrl` field; timeout set from `timeoutMs`.
- Input: Items with site URL and timeout.
- Output: HTTP response body containing HTML.
- Edge cases: Request timeouts, network errors, invalid URLs, HTTP errors (404, 500), SSL issues.

---

#### 2.3 SEO HTML Analysis

- **Overview**: Parses the fetched HTML content to extract SEO metadata such as title, meta description, H1 tags, canonical link, robots meta, image alt attributes, and identifies broken internal links.
- **Nodes Involved**: `Analyze HTML (SEO Basics)`

##### Node: Analyze HTML (SEO Basics)
- Type: Code (JavaScript)
- Role: Custom analyzer extracting SEO fields and performing link validation.
- Configuration: 
  - Accepts HTML from HTTP response or binary data.
  - Infers site URL using canonical, og:url, base href, JSON-LD, or dominant domain heuristics.
  - Extracts title, meta description (with fallback), robots tag, canonical link.
  - Counts H1 tags, images missing alt attributes.
  - Checks internal links (HEAD requests preferred) up to a max of 30 links for broken ones.
  - Records issues with severity levels ("warning"/"critical").
- Key expressions: Uses regex extensively; asynchronous HTTP HEAD/GET requests for link validation.
- Input: Raw HTML content with `siteUrl` and config.
- Output: JSON object with SEO metadata, summary, brokenLinks array, issues array, and critical flag.
- Edge cases: Invalid HTML, slow responses on link checks, request timeouts, malformed links, concurrency issues with HTTP requests.
- Version-specific: Requires n8n supporting asynchronous code execution in Code node (v2+).
- Potential failure: HTTP failures during link checking, regex failures on malformed HTML.

---

#### 2.4 SEO Health Scoring & Performance Check

- **Overview**: Runs Google PageSpeed Insights API with exponential backoff retries to fetch performance scores for each site.
- **Nodes Involved**: `SEO Health Check & Score`

##### Node: SEO Health Check & Score
- Type: Code (JavaScript)
- Role: Calls PSI API, handling transient errors with retry and backoff, extracts performance score.
- Configuration:
  - Uses hardcoded API key (placeholder: `AIzaSyDb-9zvazRVz2w105NPec89NSTKN9-Ratg`).
  - Calls `https://pagespeedonline.googleapis.com/pagespeedonline/v5/runPagespeed` with parameters for URL, strategy (default "mobile"), category "performance".
  - Retries up to 6 times with exponential backoff + jitter on rate limits or blocking errors.
  - Extracts performance score scaled 0-100.
- Input: Site URL from previous node’s JSON.
- Output: Adds `psi` object with fields: `ok`, `performance`, `rawError`, `testedUrl`, `fetchedAt`, `strategy`.
- Edge cases: API key missing or invalid, quota exceeded, Google blocking automated queries, network errors.
- Version-specific: Requires n8n environment variable or hardcoded API key.
- Failures: Throws error if API key missing or after retries fail.

---

#### 2.5 Performance Threshold Branching

- **Overview**: Branches the workflow based on whether the PSI performance score is below 50.
- **Nodes Involved**: `Check Performance < 50`

##### Node: Check Performance < 50
- Type: If Node
- Role: Evaluates if `psi.performance` field is less than 50.
- Configuration: Numeric comparison `< 50`.
- Input: Items with PSI results.
- Output: 
  - True branch: performance < 50.
  - False branch: performance >= 50.
- Edge cases: Missing or null `psi.performance` leads to false-negative or error.

---

#### 2.6 Alerting & Logging for Low Performance

- **Overview**: If performance is poor (<50), sends alert email and merges alert and normal results for logging.
- **Nodes Involved**: `Send Alert Email (Low SEO Score)`, `Merge Alert & Normal Results`

##### Node: Send Alert Email (Low SEO Score)
- Type: Gmail Node (Send Email)
- Role: Sends detailed alert email with SEO and PSI results for low-performing sites.
- Configuration:
  - Recipient: `mobile1.wli@gmail.com`
  - Subject: "Website Health & SEO Audit"
  - HTML message constructed with site details, SEO summary, PSI score, critical flags.
- Credentials: Gmail OAuth2
- Input: Items filtered for performance < 50.
- Output: Email sent confirmation.
- Edge cases: Gmail auth failures, quota limits, malformed email content.

##### Node: Merge Alert & Normal Results
- Type: Merge Node
- Role: Combines items from both alert and normal branches for unified logging.
- Input: Two inputs: alert branch and non-alert branch.
- Output: Merged item stream.
- Edge cases: Mismatched data schemas could cause merge issues.

---

#### 2.7 Report Generation for Good Performance

- **Overview**: For sites with acceptable performance, generates rich HTML SEO report, converts to PDF, downloads and saves to Google Drive.
- **Nodes Involved**: `Generate SEO Report (HTML)`, `Convert HTML to PDF`, `Download PDF File`, `Save SEO Report to Drive`

##### Node: Generate SEO Report (HTML)
- Type: HTML Node
- Role: Creates a styled HTML report from SEO data.
- Configuration: HTML template with placeholders for SEO metadata, PSI scores, issue summaries.
- Input: Items from performance >= 50 branch.
- Output: `html` field containing full HTML report.
- Edge cases: Missing data fields could cause placeholders to be empty.

##### Node: Convert HTML to PDF
- Type: HTTP Request
- Role: Sends HTML to PDF.co API to convert to PDF.
- Configuration:
  - POST to `https://api.pdf.co/v1/pdf/convert/from/html`
  - Body includes HTML content, PDF options (A4, margins, orientation).
  - Header includes `x-api-key` (not set in JSON, must be configured).
- Input: HTML report.
- Output: JSON response with PDF URL.
- Edge cases: API key missing, API limit exceeded, conversion errors.

##### Node: Download PDF File
- Type: HTTP Request
- Role: Downloads generated PDF from URL returned by PDF.co.
- Configuration: GET request to PDF download URL.
- Input: URL from previous node.
- Output: Binary PDF file data.
- Edge cases: URL expired, network errors.

##### Node: Save SEO Report to Drive
- Type: Google Drive (Upload)
- Role: Uploads PDF report to Google Drive folder.
- Configuration:
  - Filename: derived from domain second-level name.
  - Folder ID: `1lpAV7XUx2pByMu58WYbiShuxEiwwLDQK`
- Credentials: Google Drive OAuth2
- Input: Binary PDF file.
- Output: Confirmation of upload.
- Edge cases: Drive permission issues, quota limits.

---

#### 2.8 Performance Log Update

- **Overview**: Updates Google Sheet with the latest performance score, timestamp, and domain info for all analyzed sites.
- **Nodes Involved**: `Update Performance Log`

##### Node: Update Performance Log
- Type: Google Sheets (Update)
- Role: Updates existing rows matching on "Domain" with new values.
- Configuration:
  - Sheet: "Website_Seo_Testing" in the same Google Sheet document.
  - Columns updated: Date (from PSI fetch time), Domain (extracted from URL), Performance, Website Link.
  - Matching on "Domain" to update the correct row.
- Credentials: Google Sheets OAuth2
- Input: Merged results of all site analyses.
- Output: Updated Google Sheet.
- Edge cases: Matching failures if domain format differs, API rate limits.

---

### 3. Summary Table

| Node Name                  | Node Type              | Functional Role                        | Input Node(s)               | Output Node(s)                    | Sticky Note                                                                                          |
|----------------------------|------------------------|-------------------------------------|----------------------------|---------------------------------|----------------------------------------------------------------------------------------------------|
| Sticky Note                | Sticky Note            | Documentation Header                 |                            |                                 | # **Automated SEO Health Monitoring & Reporting** This workflow runs daily to analyze websites...  |
| Sticky Note1               | Sticky Note            | Documentation Node Breakdown         |                            |                                 | # **Node Breakdown & Descriptions:** Detailed node roles and order provided.                        |
| Daily 9 AM Trigger         | Cron Trigger           | Starts workflow daily at 9 AM        |                            | Fetch Website Links              |                                                                                                    |
| Fetch Website Links        | Google Sheets (Read)   | Reads URLs from Google Sheet         | Daily 9 AM Trigger          | Attach Config & Timeout          |                                                                                                    |
| Attach Config & Timeout    | Set Node               | Add site URL and timeout config      | Fetch Website Links         | Fetch Website HTML              |                                                                                                    |
| Fetch Website HTML         | HTTP Request           | Fetch website HTML content           | Attach Config & Timeout     | Analyze HTML (SEO Basics)       |                                                                                                    |
| Analyze HTML (SEO Basics)  | Code Node              | Parse HTML for SEO metadata          | Fetch Website HTML          | SEO Health Check & Score        |                                                                                                    |
| SEO Health Check & Score   | Code Node              | Run PSI API, get performance score   | Analyze HTML (SEO Basics)   | Check Performance < 50          |                                                                                                    |
| Check Performance < 50     | If Node                | Branch based on performance threshold| SEO Health Check & Score    | Send Alert Email, Generate SEO Report |                                                                                                    |
| Send Alert Email (Low SEO Score) | Gmail Node        | Send alert for low SEO scores        | Check Performance < 50 (True) | Merge Alert & Normal Results    |                                                                                                    |
| Merge Alert & Normal Results | Merge Node            | Merge alert and normal branch results| Send Alert Email, Generate SEO Report | Update Performance Log          |                                                                                                    |
| Generate SEO Report (HTML) | HTML Node              | Create HTML SEO report                | Check Performance < 50 (False) | Convert HTML to PDF            |                                                                                                    |
| Convert HTML to PDF        | HTTP Request           | Convert HTML report to PDF            | Generate SEO Report (HTML)  | Download PDF File               |                                                                                                    |
| Download PDF File          | HTTP Request           | Download PDF from PDF.co              | Convert HTML to PDF         | Save SEO Report to Drive        |                                                                                                    |
| Save SEO Report to Drive   | Google Drive (Upload)  | Upload PDF report to Google Drive    | Download PDF File           |                                 |                                                                                                    |
| Update Performance Log     | Google Sheets (Update) | Update Google Sheet with scores      | Merge Alert & Normal Results|                                 |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Cron Node: "Daily 9 AM Trigger"**  
   - Type: Cron  
   - Set trigger time to 9 AM daily.

2. **Create Google Sheets Node: "Fetch Website Links"**  
   - Type: Google Sheets (Read)  
   - Configure with OAuth2 credentials.  
   - Set Document ID to `1cQ-TBf3-dqo7njDYzYpxpASYFvEp8lIzH7vpIqTLcwc`.  
   - Set Sheet name to `Website_Seo_Testing`.  
   - Connect from "Daily 9 AM Trigger".

3. **Create Set Node: "Attach Config & Timeout"**  
   - Type: Set  
   - Add field `siteUrl` with value expression `={{ $json["Website Link"] }}`.  
   - Add field `timeoutMs` with fixed string value `"2000"`.  
   - Connect from "Fetch Website Links".

4. **Create HTTP Request Node: "Fetch Website HTML"**  
   - Type: HTTP Request  
   - URL: Expression `={{$json["siteUrl"]}}`.  
   - Timeout: Expression `={{$json["timeoutMs"]}}` in milliseconds.  
   - Method: GET (default).  
   - Connect from "Attach Config & Timeout".

5. **Create Code Node: "Analyze HTML (SEO Basics)"**  
   - Type: Code  
   - Paste the provided JavaScript SEO analyzer code (as in node details).  
   - This code extracts SEO metadata, validates internal links, and produces a summary with issues.  
   - Connect from "Fetch Website HTML".

6. **Create Code Node: "SEO Health Check & Score"**  
   - Type: Code  
   - Paste the provided PSI API call code with backoff and retries.  
   - Important: Replace placeholder API key with a valid Google PSI API key or set environment variable `PSI_API_KEY`.  
   - Connect from "Analyze HTML (SEO Basics)".

7. **Create If Node: "Check Performance < 50"**  
   - Type: If  
   - Condition: Numeric comparison, check if value `{{$json.psi.performance}}` is less than 50.  
   - Connect from "SEO Health Check & Score".

8. **Create Gmail Node: "Send Alert Email (Low SEO Score)"**  
   - Type: Gmail (Send Email)  
   - Configure with OAuth2 credentials.  
   - Set recipient to monitoring team email (e.g., `mobile1.wli@gmail.com`).  
   - Subject: "Website Health & SEO Audit".  
   - Body: Use provided HTML template with placeholders for SEO data and PSI results.  
   - Connect from "Check Performance < 50" true branch.

9. **Create HTML Node: "Generate SEO Report (HTML)"**  
   - Type: HTML  
   - Paste provided HTML report template with placeholders for SEO and PSI data.  
   - Connect from "Check Performance < 50" false branch.

10. **Create HTTP Request Node: "Convert HTML to PDF"**  
    - Type: HTTP Request (POST)  
    - URL: `https://api.pdf.co/v1/pdf/convert/from/html`  
    - Headers: `x-api-key` with your PDF.co API key.  
    - Body parameters: include HTML content (`{{$json.html}}`), paper size, margins, orientation, etc.  
    - Connect from "Generate SEO Report (HTML)".

11. **Create HTTP Request Node: "Download PDF File"**  
    - Type: HTTP Request (GET)  
    - URL: Expression `={{ $json.url }}` (URL returned by PDF.co).  
    - Connect from "Convert HTML to PDF".

12. **Create Google Drive Node: "Save SEO Report to Drive"**  
    - Type: Google Drive (Upload)  
    - Configure with Google Drive OAuth2 credentials.  
    - Folder ID: `1lpAV7XUx2pByMu58WYbiShuxEiwwLDQK` (configured target folder).  
    - Filename: Extract domain second-level name from URL expression.  
    - Connect from "Download PDF File".

13. **Create Merge Node: "Merge Alert & Normal Results"**  
    - Type: Merge  
    - Connect two inputs:  
      - From "Send Alert Email (Low SEO Score)".  
      - From "Generate SEO Report (HTML)".  
    - This unifies both alert and normal branches.

14. **Create Google Sheets Node: "Update Performance Log"**  
    - Type: Google Sheets (Update)  
    - Configure with same Sheet and Document as "Fetch Website Links".  
    - Columns to update: Date (from `psi.fetchedAt`), Domain (extracted from URL), Performance (from `psi.performance`), Website Link (full URL).  
    - Set matching on "Domain" column for updating existing rows.  
    - Connect from "Merge Alert & Normal Results".

15. **Connect all nodes following the described flow**:  
    - Daily 9 AM Trigger → Fetch Website Links → Attach Config & Timeout → Fetch Website HTML → Analyze HTML (SEO Basics) → SEO Health Check & Score → Check Performance < 50  
    - True branch (score < 50) → Send Alert Email → Merge Alert & Normal Results  
    - False branch (score ≥ 50) → Generate SEO Report (HTML) → Convert HTML to PDF → Download PDF File → Save SEO Report to Drive → Merge Alert & Normal Results  
    - Merge Alert & Normal Results → Update Performance Log

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                  | Context or Link                                                                                                   |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| This workflow requires valid API keys for Google PageSpeed Insights (PSI) and PDF.co API. The PSI key is critical for performance scoring and should be set as an environment variable or hardcoded securely. PDF.co API key is required for HTML-to-PDF conversion.                                                           | API keys for PSI: https://developers.google.com/speed/docs/insights/v5/get-started<br>PDF.co: https://pdf.co/  |
| The Google Sheets and Google Drive integrations require OAuth2 credentials authorized with appropriate scopes to read/write sheets and upload files to drive folders.                                                                                                                                                         | n8n Google credentials setup docs: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.google/    |
| The workflow uses custom code nodes for HTML parsing and PSI calls. It requires n8n version supporting asynchronous JavaScript in Code nodes (typically v0.151+).                                                                                                                                                             | n8n Code Node documentation: https://docs.n8n.io/nodes/n8n-nodes-base.code/                                      |
| The email alert template is styled HTML suitable for Gmail clients and includes detailed SEO and PSI data. Adjust recipient emails and message content as needed per monitoring team preferences.                                                                                                                             | Gmail Node docs: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail/                        |
| The workflow assumes the Google Sheet "Website_Seo_Testing" has a column named "Website Link" containing the URLs to monitor. It also expects the sheet to have rows identified by domain for update operations.                                                                                                                  |                                                                                                                  |
| The workflow handles up to 30 internal links per site for broken link checking to avoid performance bottlenecks and API overloads. Adjust this limit in the code node if required, considering API quotas and runtime.                                                                                                         |                                                                                                                  |
| For enhanced monitoring, users can extend the workflow to handle multiple URLs per email or generate trend charts over time using historical data in the Google Sheet.                                                                                                                                                        |                                                                                                                  |

---

**Disclaimer**: The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All handled data are legal and public.