Crawl Website Blog Content and Save to Google Sheets with Dumpling AI

https://n8nworkflows.xyz/workflows/crawl-website-blog-content-and-save-to-google-sheets-with-dumpling-ai-8195


# Crawl Website Blog Content and Save to Google Sheets with Dumpling AI

### 1. Workflow Overview

This workflow automates the process of crawling a client’s website blog content, extracting relevant blog URLs, scraping their content, and saving structured data into a newly created Google Sheets document. It leverages Dumpling AI for crawling and scraping tasks, and Google Sheets for data storage and audit purposes.

The workflow is logically organized into the following blocks:

- **1.1 Input Reception:** Captures client input via a form trigger.
- **1.2 Google Sheets Setup:** Creates a new spreadsheet, sets headers, and prepares it to receive data.
- **1.3 Website Crawl and URL Extraction:** Uses Dumpling AI to crawl the submitted website URL, then extracts and filters blog-related URLs.
- **1.4 Blog Content Scraping:** Scrapes content from each filtered blog URL using Dumpling AI.
- **1.5 Data Preparation and Storage:** Formats the scraped data and appends it into the Google Sheet for audit and review.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block triggers the workflow when a client submits a URL through a form. It initiates the entire process by capturing the target website URL.

- **Nodes Involved:**  
  - Form Submission

- **Node Details:**

  - **Form Submission**  
    - Type: Form Trigger  
    - Role: Starts the workflow by receiving a client-submitted URL.  
    - Configuration:  
      - Form titled "blog content strategy"  
      - Single required field labeled "Client URL"  
    - Expressions: None beyond form field capture.  
    - Input: External user form submission  
    - Output: JSON object containing `{ Client URL: <submitted_url> }`  
    - Version: 2.2  
    - Potential failures: Form submission timeout, invalid or empty URL input, webhook connectivity issues.

---

#### 2.2 Google Sheets Setup

- **Overview:**  
  Creates a new Google Sheet named after the client URL domain, initializes a single sheet tab titled "Blog content audit," defines the header row, formats it, and inserts it into the sheet.

- **Nodes Involved:**  
  - Create Blog Audit Sheet  
  - Set Sheet Headers  
  - Format Header Row  
  - Insert Headers into Sheet

- **Node Details:**

  - **Create Blog Audit Sheet**  
    - Type: Google Sheets  
    - Role: Creates a new Google Sheets document.  
    - Configuration:  
      - Title derived by trimming the submitted URL and extracting the domain name portion before the first dot. For example, "https://crane-baby.com" → "https://crane-baby" (trimmed and split logic).  
      - Creates one sheet tab titled "Blog content audit"  
    - Credentials: OAuth2 with Google Sheets account  
    - Input: Client URL from Form Submission  
    - Output: JSON with spreadsheet ID, URL, and sheet properties  
    - Version: 4.6  
    - Potential failures: Google OAuth permission issues, API rate limits, invalid spreadsheet creation parameters.

  - **Set Sheet Headers**  
    - Type: Set Node  
    - Role: Defines the header row as a CSV string "Url,Crawled_pages,website_content" in a variable named `rows`.  
    - Input: Output of "Create Blog Audit Sheet"  
    - Output: JSON with header string  
    - Version: 1  
    - Potential failures: None significant (simple data setting).

  - **Format Header Row**  
    - Type: Code (JavaScript)  
    - Role: Converts the CSV string into a 2D array format suitable for Google Sheets API (an array of arrays).  
    - Key code: Splits the string by commas and wraps it inside an array:  
      ```js
      return [
        {
          json: {
            data: [ $json.rows.split(',') ]
          }
        }
      ];
      ```  
    - Input: From Set Sheet Headers  
    - Output: JSON with a `data` property containing the array of headers  
    - Version: 2  
    - Potential failures: If `rows` is missing or malformed.

  - **Insert Headers into Sheet**  
    - Type: HTTP Request  
    - Role: Calls Google Sheets API v4 to insert the header row into the created sheet.  
    - Configuration:  
      - HTTP Method: PUT  
      - URL constructed dynamically using the spreadsheet ID and sheet title from "Create Blog Audit Sheet" output.  
      - Request body includes `range` (sheet name + columns), and `values` (header array).  
      - Query parameter `valueInputOption=RAW` to insert values as-is.  
      - Uses Google Sheets OAuth2 credential.  
    - Input: Formatted headers from "Format Header Row", spreadsheet info from "Create Blog Audit Sheet"  
    - Output: API response confirming update  
    - Version: 4.1  
    - Potential failures: OAuth token expiration, API quota limits, invalid spreadsheet ID or range.

---

#### 2.3 Website Crawl and URL Extraction

- **Overview:**  
  Crawls the submitted client website URL using Dumpling AI API to discover URLs, then extracts and filters URLs related to blog content using pattern matching.

- **Nodes Involved:**  
  - Dumpling AI: Crawl Website  
  - Extract Blog URLs

- **Node Details:**

  - **Dumpling AI: Crawl Website**  
    - Type: HTTP Request  
    - Role: Sends a POST request to Dumpling AI’s crawl API endpoint.  
    - Configuration:  
      - URL: `https://app.dumplingai.com/api/v1/crawl`  
      - Body parameters:  
        - `url` set to the client URL from form input  
        - `limit` set to 10 (max pages to crawl)  
      - Authentication: HTTP Header Auth using stored Dumpling AI API key  
    - Input: Client URL from Form Submission  
    - Output: JSON containing crawled page data and URLs  
    - Version: 4.2  
    - Potential failures: API key invalid/expired, Dumpling AI service downtime, network timeouts.

  - **Extract Blog URLs**  
    - Type: Code (JavaScript)  
    - Role: Parses the JSON crawl results to extract all URLs, then filters them to identify blog-related URLs based on inclusion and exclusion regex patterns.  
    - Logic highlights:  
      - Extracts all URLs from the JSON as strings using regex.  
      - Deduplicates URLs.  
      - Filters URLs matching typical blog patterns (`/blog/`, `/post/`, `/article/`, date-based URLs, guides, tips, etc.)  
      - Excludes URLs with file extensions (images, css, js, pdf, zip) or ecommerce/cart/account related paths.  
      - Returns a sorted list of unique blog-related URLs.  
    - Input: Dumpling AI crawl output  
    - Output: Array of JSON objects each with a `blogUrl` property or a message if none found  
    - Version: 2  
    - Edge cases: No blog URLs found, malformed URLs, encoding issues.

---

#### 2.4 Blog Content Scraping

- **Overview:**  
  For each extracted blog URL, scrapes the webpage content using Dumpling AI’s scraping API.

- **Nodes Involved:**  
  - Dumpling AI: Scrape Blog Pages

- **Node Details:**

  - **Dumpling AI: Scrape Blog Pages**  
    - Type: HTTP Request  
    - Role: Sends POST requests to Dumpling AI’s scrape API for each blog URL.  
    - Configuration:  
      - URL: `https://app.dumplingai.com/api/v1/scrape`  
      - Body parameter `url` set dynamically per each blog URL from the previous node  
      - Authentication: HTTP Header Auth with Dumpling AI API key  
    - Input: Blog URLs from Extract Blog URLs  
    - Output: JSON containing scraped content from each blog page  
    - Version: 4.2  
    - Potential failures: API errors for invalid URLs, rate limiting, network issues, content not found.

---

#### 2.5 Data Preparation and Storage

- **Overview:**  
  Structures the data into a row format with URL, crawled page URL, and website content, then appends it to the Google Sheet.

- **Nodes Involved:**  
  - Prepare Row Data  
  - Save Blog Data to Google Sheets

- **Node Details:**

  - **Prepare Row Data**  
    - Type: Set Node  
    - Role: Maps and constructs objects with three fields:  
      - `Url`: original client URL from form input  
      - `Crawled_pages`: blog URL currently processed  
      - `website_content`: scraped content from Dumpling AI scrape response  
    - Input: Output from Dumpling AI scrape node and original form submission  
    - Output: JSON object with the three fields for each blog page  
    - Version: 3.4  
    - Potential failures: Missing fields in scraped content, null content.

  - **Save Blog Data to Google Sheets**  
    - Type: Google Sheets  
    - Role: Appends rows of structured blog data to the audit sheet created earlier.  
    - Configuration:  
      - Operation: Append  
      - Columns automatically mapped based on input fields (Url, Crawled_pages, website_content)  
      - Target sheet specified by sheet ID from "Create Blog Audit Sheet" output  
      - Document specified by URL from "Create Blog Audit Sheet"  
      - Credentials: OAuth2 Google Sheets account  
    - Input: Structured row data from "Prepare Row Data"  
    - Output: Confirmation of appended rows  
    - Version: 4.7  
    - Potential failures: API limits, invalid sheet ID, OAuth failures.

---

### 3. Summary Table

| Node Name                  | Node Type          | Functional Role                              | Input Node(s)             | Output Node(s)                   | Sticky Note                                                                                              |
|----------------------------|--------------------|----------------------------------------------|---------------------------|---------------------------------|---------------------------------------------------------------------------------------------------------|
| Form Submission            | Form Trigger       | Entry point to receive client URL             | -                         | Create Blog Audit Sheet          | ## Workflow Overview 1. Trigger: Form Submission (Client URL) — Starts the workflow when a client URL is entered. |
| Create Blog Audit Sheet    | Google Sheets      | Creates a new Google Sheets document          | Form Submission           | Set Sheet Headers               | 2. Create Blog Audit Sheet — Creates a new Google Sheet for the audit.                                  |
| Set Sheet Headers          | Set Node           | Defines header row as CSV string               | Create Blog Audit Sheet   | Format Header Row               | 3. Set Sheet Headers — Defines the columns (URL, Crawled Pages, Website Content).                        |
| Format Header Row          | Code Node          | Converts CSV headers into array for Sheets API | Set Sheet Headers         | Insert Headers into Sheet       | 4. Format Header Row — Prepares the headers into the right format for Google Sheets.                    |
| Insert Headers into Sheet  | HTTP Request       | Inserts header row into Google Sheet           | Format Header Row         | Dumpling AI: Crawl Website      | 5. Insert Headers into Sheet — Updates the sheet with the headers.                                      |
| Dumpling AI: Crawl Website | HTTP Request       | Crawls submitted website to discover URLs      | Insert Headers into Sheet | Extract Blog URLs               | 6. Dumpling AI: Crawl Website — Crawls the submitted URL to discover pages.                             |
| Extract Blog URLs          | Code Node          | Extracts and filters blog-related URLs         | Dumpling AI: Crawl Website | Dumpling AI: Scrape Blog Pages | 7. Extract Blog URLs — Filters the crawl results to keep only blog-related links.                      |
| Dumpling AI: Scrape Blog Pages | HTTP Request  | Scrapes content from each blog page             | Extract Blog URLs         | Prepare Row Data               | 8. Dumpling AI: Scrape Blog Pages — Scrapes the content from each blog page.                            |
| Prepare Row Data           | Set Node           | Maps URL, crawled page, and content into fields | Dumpling AI: Scrape Blog Pages | Save Blog Data to Google Sheets | 9. Prepare Row Data — Maps the URL, crawled page, and content into structured fields.                   |
| Save Blog Data to Google Sheets | Google Sheets | Appends blog data rows to the audit sheet       | Prepare Row Data          | -                             | 10. Save Blog Data to Google Sheets — Appends the results into the audit sheet for review.              |
| Sticky Note                | Sticky Note        | Provides workflow overview and step summary    | -                         | -                             | ## Workflow Overview (Describes all main steps from Form Submission to Saving Data in Sheets)           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node**  
   - Type: Form Trigger (version 2.2)  
   - Configure with a form titled "blog content strategy"  
   - Add one required field labeled "Client URL"  
   - This node will trigger the workflow on form submission with the client URL.

2. **Add Google Sheets Node to Create Spreadsheet**  
   - Type: Google Sheets (version 4.6)  
   - Operation: Create spreadsheet  
   - Title expression:  
     ```= {{ $json["Client URL"].trim().split(/›|>|»/)[0].trim().split(".")[0] }}```  
   - Add one sheet named "Blog content audit"  
   - Use Google Sheets OAuth2 credentials.

3. **Add Set Node to Define Header Row**  
   - Type: Set (version 1)  
   - Set field `rows` to string: `"Url,Crawled_pages,website_content"`

4. **Add Code Node to Format Header Row**  
   - Type: Code (version 2)  
   - JS code:  
     ```js
     return [
       {
         json: {
           data: [ $json.rows.split(',') ]
         }
       }
     ];
     ```

5. **Add HTTP Request Node to Insert Headers into Sheet**  
   - Type: HTTP Request (version 4.1)  
   - Method: PUT  
   - URL expression:  
     ```= https://sheets.googleapis.com/v4/spreadsheets/{{ $('Create Blog Audit Sheet').first().json.spreadsheetId }}/values/{{ $('Create Blog Audit Sheet').first().json.sheets[0].properties.title }}!A:Z```  
   - Body parameters:  
     - `range`: sheet title + `!A:Z`  
     - `values`: from `data` property of previous node  
   - Query parameter: `valueInputOption=RAW`  
   - Authentication: Google Sheets OAuth2 credentials.

6. **Add HTTP Request Node for Dumpling AI Crawl**  
   - Type: HTTP Request (version 4.2)  
   - Method: POST  
   - URL: `https://app.dumplingai.com/api/v1/crawl`  
   - Body parameters:  
     - `url` from form submission: `={{ $('Form Submission ').item.json["Client URL"] }}`  
     - `limit`: `10`  
   - Authentication: HTTP Header Auth with Dumpling AI API key.

7. **Add Code Node to Extract Blog URLs**  
   - Type: Code (version 2)  
   - Paste the provided JavaScript code that extracts and filters blog-related URLs using regex patterns and returns the list.

8. **Add HTTP Request Node for Dumpling AI Scrape**  
   - Type: HTTP Request (version 4.2)  
   - Method: POST  
   - URL: `https://app.dumplingai.com/api/v1/scrape`  
   - Body parameter: `url` set dynamically for each `blogUrl` from previous node  
   - Authentication: HTTP Header Auth with Dumpling AI API key.

9. **Add Set Node to Prepare Row Data**  
   - Type: Set (version 3.4)  
   - Assign fields:  
     - `Url`: `={{ $('Form Submission ').item.json["Client URL"] }}`  
     - `Crawled_pages`: `={{ $('Extract Blog URLs').item.json.blogUrl }}`  
     - `website_content`: `={{ $json.content }}`

10. **Add Google Sheets Node to Append Data**  
    - Type: Google Sheets (version 4.7)  
    - Operation: Append  
    - Document Id: `={{ $('Create Blog Audit Sheet').item.json.spreadsheetUrl }}` (mode URL)  
    - Sheet Name: `={{ $('Create Blog Audit Sheet').item.json.sheets[0].properties.sheetId }}` (mode ID)  
    - Columns: Auto-mapped for `Url`, `Crawled_pages`, `website_content`  
    - Use Google Sheets OAuth2 credentials.

11. **Connect the Nodes in the Following order:**  
    - Form Submission → Create Blog Audit Sheet → Set Sheet Headers → Format Header Row → Insert Headers into Sheet → Dumpling AI: Crawl Website → Extract Blog URLs → Dumpling AI: Scrape Blog Pages → Prepare Row Data → Save Blog Data to Google Sheets

12. **Credential Setup:**  
    - Google Sheets OAuth2 API credential with permissions to create and edit spreadsheets.  
    - Dumpling AI HTTP Header Auth credential containing the API key.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                              |
|---------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| This workflow uses Dumpling AI APIs for crawling and scraping web content. You must have a valid API key set up in n8n credentials. | Dumpling AI API documentation (https://app.dumplingai.com/)  |
| The Google Sheets operations require OAuth2 credentials with scopes to create and edit spreadsheets.          | Google Sheets API documentation (https://developers.google.com/sheets/api) |
| The blog URL extraction step uses custom regex patterns tailored to common blog URL structures and exclusions. | Patterns can be adjusted in the Extract Blog URLs node code. |
| The workflow is designed for blog content auditing and can be adapted to other content types with regex tuning. |                                                              |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. All data handled is legal and public, respecting content policies strictly.