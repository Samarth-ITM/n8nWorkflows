Track SEO Keyword Rankings in Google Search with ScrapingBee API

https://n8nworkflows.xyz/workflows/track-seo-keyword-rankings-in-google-search-with-scrapingbee-api-4089


# Track SEO Keyword Rankings in Google Search with ScrapingBee API

### 1. Workflow Overview

This workflow automates the process of scraping Google Search Engine Results Pages (SERPs) for a given keyword, formatting the data into useful reports, and delivering the output for SEO monitoring and analysis. It targets digital marketers, SEO consultants, and content strategists who need to efficiently track keyword rankings without manual data collection.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** A form trigger node accepts a user-submitted keyword to initiate the workflow.
- **1.2 Data Preparation:** Prepares and validates input parameters for the API call.
- **1.3 Google SERP Scraping:** Calls the ScrapingBee API to retrieve live Google search results for the keyword.
- **1.4 Data Formatting:** Converts the raw API response into two distinct table formats: an HTML table for email reports and a CSV-compatible Markdown table for downloads.
- **1.5 Report Delivery:** Generates downloadable files from the CSV data and optionally sends an email with the formatted HTML report.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Captures the search keyword input from the user via a web form trigger, starting the workflow execution.

- **Nodes Involved:**  
  - Start (Form Trigger)

- **Node Details:**  
  - **Start (Form Trigger):**  
    - *Type & Role:* Trigger node that provides a webhook with a form interface for user input.  
    - *Configuration:* Accepts a keyword as user input; no complex parameters are set by default.  
    - *Inputs:* None (trigger node).  
    - *Outputs:* Passes form data to "Assign Valid Values".  
    - *Edge Cases:* Missing or empty keyword input could cause downstream API calls to fail or return no data. Validation or default handling is needed.  
    - *Version:* Compatible with n8n 0.151+.

#### 2.2 Data Preparation

- **Overview:**  
  Validates and assigns default or corrected values for the search keyword and any other parameters before making the API call.

- **Nodes Involved:**  
  - Assign Valid Values (Code)

- **Node Details:**  
  - *Type & Role:* Function code node that processes incoming data to ensure valid inputs for the API request.  
  - *Configuration:* Contains JavaScript code that checks the keyword input, possibly sets default search parameters or sanitizes the input.  
  - *Inputs:* Receives raw user input from "Start".  
  - *Outputs:* Validated and structured parameter data for the HTTP request node.  
  - *Expressions:* Likely uses expressions to read incoming JSON and conditionally assign values.  
  - *Edge Cases:* Improper input formats or missing keywords; code should handle or throw errors.  
  - *Version:* Requires n8n supporting Code node version 2.

#### 2.3 Google SERP Scraping

- **Overview:**  
  Sends a request to the ScrapingBee API to scrape Google search results for the validated keyword.

- **Nodes Involved:**  
  - Scrape Google SERPs (HTTP Request)

- **Node Details:**  
  - *Type & Role:* HTTP Request node configured to call the ScrapingBee API endpoint, passing the keyword and API key.  
  - *Configuration:*  
    - Method: GET or POST depending on API spec.  
    - URL: ScrapingBee Google SERP endpoint.  
    - Authentication: API key in headers or query parameters.  
    - Additional parameters: Search keyword, possibly search location, language, or number of results.  
  - *Inputs:* Receives validated parameters from "Assign Valid Values".  
  - *Outputs:* Raw JSON response with Google SERP data.  
  - *Edge Cases:*  
    - API key invalid or expired → authentication error.  
    - Rate limiting by ScrapingBee → HTTP 429 errors.  
    - Network timeouts or JSON parse errors.  
  - *Version:* Requires HTTP Request node v4.2+ for advanced features.

#### 2.4 Data Formatting

- **Overview:**  
  Parses and transforms the raw SERP JSON data into two user-friendly formats: an HTML table for embedding in emails and a CSV-compatible Markdown table for downloads.

- **Nodes Involved:**  
  - Build HTML Data Tables (Code)  
  - Build CSV Data Tables (Code)

- **Node Details:**  
  - *Build HTML Data Tables:*  
    - *Type & Role:* Code node generating an HTML table representation of the SERP data.  
    - *Configuration:* Uses JavaScript to iterate over result items, building semantic HTML with headings, links, and rankings.  
    - *Inputs:* JSON output from "Scrape Google SERPs".  
    - *Outputs:* HTML string for email body.  
    - *Edge Cases:* Malformed or incomplete SERP data; should handle missing fields gracefully.  
  - *Build CSV Data Tables:*  
    - *Type & Role:* Code node generating Markdown-formatted CSV data for file creation.  
    - *Configuration:* Converts JSON into pipe-separated Markdown table rows to support spreadsheet import.  
    - *Inputs:* Same as HTML builder.  
    - *Outputs:* Markdown string representing CSV.  
    - *Edge Cases:* Similar to HTML node, plus escaping special characters for CSV compatibility.  
  - *Version:* Requires Code node v2.

#### 2.5 Report Delivery

- **Overview:**  
  Converts the Markdown CSV data into a downloadable file and sends the HTML report via email.

- **Nodes Involved:**  
  - Download SEO Report (Convert to File)  
  - Mail SEO Report (Mailjet)

- **Node Details:**  
  - *Download SEO Report:*  
    - *Type & Role:* Converts Markdown data into a .csv file for download or further processing.  
    - *Configuration:* Input from "Build CSV Data Tables". Sets file type as CSV, file name, and encoding.  
    - *Outputs:* File binary data ready for download or storage.  
    - *Edge Cases:* Large data sizes may affect performance; encoding issues could corrupt CSV.  
    - *Version:* Convert to File node v1.1 or higher recommended.  
  - *Mail SEO Report:*  
    - *Type & Role:* Sends the HTML report via email using Mailjet SMTP or API.  
    - *Configuration:*  
      - Recipient email address configured in node parameters.  
      - HTML content passed from "Build HTML Data Tables".  
      - Subject line, sender email setup required.  
    - *Inputs:* HTML formatted report.  
    - *Edge Cases:* Invalid email credentials, bouncebacks, or network issues.  
    - *Version:* Mailjet node v1.

---

### 3. Summary Table

| Node Name             | Node Type             | Functional Role                 | Input Node(s)          | Output Node(s)                   | Sticky Note                   |
|-----------------------|-----------------------|--------------------------------|------------------------|---------------------------------|-------------------------------|
| Start                 | Form Trigger          | User keyword input trigger     | None                   | Assign Valid Values             |                               |
| Assign Valid Values    | Code                  | Validates/prepares input       | Start                  | Scrape Google SERPs             |                               |
| Scrape Google SERPs   | HTTP Request          | Calls ScrapingBee API          | Assign Valid Values     | Build HTML Data Tables, Build CSV Data Tables |                               |
| Build HTML Data Tables | Code                  | Creates HTML table report      | Scrape Google SERPs     | Mail SEO Report                 |                               |
| Build CSV Data Tables  | Code                  | Creates Markdown/CSV data      | Scrape Google SERPs     | Download SEO Report             |                               |
| Download SEO Report    | Convert to File       | Converts CSV data to file      | Build CSV Data Tables   | None                           |                               |
| Mail SEO Report        | Mailjet               | Sends email with HTML report   | Build HTML Data Tables  | None                           |                               |
| Sticky Note1           | Sticky Note           | (No content)                   | None                   | None                           |                               |
| Sticky Note            | Sticky Note           | (No content)                   | None                   | None                           |                               |
| Sticky Note2           | Sticky Note           | (No content)                   | None                   | None                           |                               |
| Sticky Note3           | Sticky Note           | (No content)                   | None                   | None                           |                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node named "Start":**  
   - Type: Form Trigger  
   - Configure to accept a single input field named `keyword` (text).  
   - This node will act as the entry point for users to submit a search term.

2. **Add a Code node named "Assign Valid Values":**  
   - Connect from "Start".  
   - Purpose: Validate and sanitize the keyword input.  
   - Sample logic:  
     - Check if `keyword` exists and is non-empty.  
     - Set default search parameters if needed (e.g., language, location).  
     - Output JSON with properties ready for API call.  

3. **Add an HTTP Request node named "Scrape Google SERPs":**  
   - Connect from "Assign Valid Values".  
   - Configure:  
     - Method: GET (or POST per API spec).  
     - URL: ScrapingBee API endpoint for Google SERP scraping.  
     - Query Parameters: Include the keyword and any other required parameters.  
     - Authentication: Add your ScrapingBee API key in headers or query parameters.  
     - Response format: JSON.  
   - Test the API call with a sample keyword.

4. **Add a Code node named "Build HTML Data Tables":**  
   - Connect from "Scrape Google SERPs".  
   - Purpose: Parse the JSON response and generate an HTML table string.  
   - Logic details:  
     - Loop over organic search results.  
     - For each item, create a table row with rank, title (as hyperlink), and URL.  
     - Return the full HTML table string.  

5. **Add a Code node named "Build CSV Data Tables":**  
   - Connect from "Scrape Google SERPs".  
   - Purpose: Create a Markdown-formatted CSV string from the same data.  
   - Logic details:  
     - Loop over the results.  
     - Format rows with pipe `|` separators.  
     - Escape special characters as needed.  

6. **Add a Convert To File node named "Download SEO Report":**  
   - Connect from "Build CSV Data Tables".  
   - Configure:  
     - Set file format to CSV.  
     - Set filename, e.g., `seo-report.csv`.  
     - Encoding: UTF-8.  

7. **Add a Mailjet node named "Mail SEO Report":**  
   - Connect from "Build HTML Data Tables".  
   - Configure:  
     - Set recipient email address.  
     - Set sender email address and SMTP/API credentials.  
     - Subject line, e.g., "SEO Keyword Ranking Report".  
     - Body: Use the HTML table output as the email content.  

8. **Activate the workflow:**  
   - Test by triggering the form with a keyword.  
   - Confirm email arrival and downloadable CSV creation.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                | Context or Link                                                                                     |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow template is ideal for SEO professionals needing quick, automated SERP data collection and reporting.                                                         | Workflow description and use case.                                                                |
| ScrapingBee API requires an API key and is subject to rate limits; ensure your subscription supports your usage volume.                                                   | ScrapingBee API documentation: https://www.scrapingbee.com/documentation/                         |
| You can extend this workflow by integrating Google Sheets or Airtable nodes to automate data logging and historical tracking.                                             | n8n integrations: https://docs.n8n.io/integrations/                                               |
| For email delivery, Mailjet credentials must be configured correctly; alternatively, other SMTP or email nodes can be used.                                               | Mailjet node docs: https://docs.n8n.io/nodes/n8n-nodes-base.mailjet/                              |
| Consider adding error handling nodes (e.g., "Error Trigger") to catch API failures or email sending issues for robust automation.                                         | n8n error handling best practices: https://docs.n8n.io/workflows/error-handling/                   |

---

**Disclaimer:** The provided text is derived exclusively from an n8n automated workflow. It strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.