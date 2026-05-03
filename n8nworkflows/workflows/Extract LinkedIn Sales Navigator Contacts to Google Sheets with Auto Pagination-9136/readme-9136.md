Extract LinkedIn Sales Navigator Contacts to Google Sheets with Auto Pagination

https://n8nworkflows.xyz/workflows/extract-linkedin-sales-navigator-contacts-to-google-sheets-with-auto-pagination-9136


# Extract LinkedIn Sales Navigator Contacts to Google Sheets with Auto Pagination

### 1. Workflow Overview

This workflow automates the extraction of LinkedIn Sales Navigator contacts and appends the data into a Google Sheets document, handling multiple pages of contacts through automatic pagination. It is designed for users with access to LinkedIn Sales Navigator and an API providing LinkedIn contact scraping capabilities. The workflow includes logical blocks for setting search parameters, scraping data via API, extracting and formatting contact data, saving contacts to Google Sheets, and managing pagination with rate limiting to prevent API blocks.

Logical blocks:

- **1.1 Initialization and Parameter Setup**: Starts the workflow manually and sets up search parameters including LinkedIn cookies, search URL, and pagination controls.
- **1.2 LinkedIn Contacts Scraping**: Calls an external API to fetch LinkedIn contacts data.
- **1.3 Data Extraction and Transformation**: Extracts nested contact arrays from API responses and formats data for storage.
- **1.4 Data Storage**: Appends extracted contact information into a Google Sheets spreadsheet.
- **1.5 Pagination Control and Rate Limiting**: Checks if more pages exist, increments pagination parameters, and manages delay between requests to avoid rate limiting or blocks.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization and Parameter Setup

- **Overview:**  
  This block triggers the workflow manually and sets initial parameters required for the LinkedIn scraping process, including authentication cookies, target URL, scraper type, pagination start index, and total number of pages to scrape.

- **Nodes Involved:**  
  - Start Workflow (Manual Trigger)  
  - Set Search Parameters (Set Node)  
  - Sticky Note (Configuration Settings)

- **Node Details:**  

  - **Start Workflow**  
    - Type: Manual Trigger  
    - Role: Entry point to start the workflow manually.  
    - Configuration: No parameters.  
    - Connections: Outputs to "Set Search Parameters".  
    - Edge Cases: None specific, but workflow depends on manual start.

  - **Set Search Parameters**  
    - Type: Set  
    - Role: Defines initial variables for scraping, including cookies, URL, scraper type, pagination start index, and total pages.  
    - Configuration:  
      - `cookies`: User must replace `[YOUR_COOKIES_HERE]` with LinkedIn session cookies.  
      - `url`: User must replace `[YOUR_SALES_NAVIGATOR_SEARCH_URL_HERE]` with the LinkedIn Sales Navigator search URL.  
      - `scraper_type`: Fixed as `"contacts"`.  
      - `start`: Pagination start index, default 0.  
      - `total_pages`: Number of pages to scrape, default 2.  
    - Connections: Outputs to "Scrape LinkedIn Contacts API".  
    - Expressions: Uses conditional expression for `start` to default to 0 if input is missing.  
    - Edge Cases: Missing or invalid cookies or URL will cause scraping to fail.

  - **Sticky Note (Configuration Settings)**  
    - Type: Sticky Note  
    - Role: Provides user instructions on required parameters and how to modify them before running.  
    - Content Summary: Explains cookies, URL, scraper_type, total_pages, and pagination size (~25 contacts per page).

---

#### 2.2 LinkedIn Contacts Scraping

- **Overview:**  
  This block sends a POST request to the LinkedIn scraping API with the configured parameters to retrieve contact data.

- **Nodes Involved:**  
  - Scrape LinkedIn Contacts API (HTTP Request)  
  - Sticky Note4 (API Access Instructions)

- **Node Details:**

  - **Scrape LinkedIn Contacts API**  
    - Type: HTTP Request  
    - Role: Sends POST requests to an external LinkedIn contacts scraping API endpoint.  
    - Configuration:  
      - URL: `https://api.nickautomations.com/linkedin/scrape`  
      - Method: POST  
      - Authentication: HTTP Header Auth with an API key credential named `[linkedin-scraper-api] Header Auth`.  
      - Body Parameters: Passes `cookies`, `url`, `scraper_type`, and `start` from incoming JSON.  
      - Sends body as form parameters.  
    - Connections: Outputs to "Extract Contact Data Array".  
    - Edge Cases:  
      - Authentication failures if API key invalid or missing.  
      - API timeouts or rate limits.  
      - Invalid or expired cookies causing scraping failure or empty data.  
      - Network errors.  
    - Sticky Note4 provides detailed instructions on obtaining API credentials via email and configuring the HTTP Header Auth credential.

---

#### 2.3 Data Extraction and Transformation

- **Overview:**  
  This block processes the raw API response, extracting the nested array of contact data for further processing.

- **Nodes Involved:**  
  - Extract Contact Data Array (Code Node)  
  - Sticky Note1 (Scraping Process Overview)

- **Node Details:**

  - **Extract Contact Data Array**  
    - Type: Code (JavaScript)  
    - Role: Extracts the 'data' array from the first item of the HTTP response JSON.  
    - Configuration:  
      - Code extracts `$input.all()` items, assumes first item’s `.json.data` contains the contacts array, and returns it as the output items.  
    - Connections: Outputs to "Save Contacts to Google Sheets".  
    - Edge Cases:  
      - If response does not contain `data` array, code may fail or output empty.  
      - JSON parsing errors if API response malformed.

  - **Sticky Note1**  
    - Type: Sticky Note  
    - Role: Explains the scraping process steps and data fields included (name, title, company, location, profile URL, etc.).

---

#### 2.4 Data Storage

- **Overview:**  
  This block appends the extracted LinkedIn contacts data into a Google Sheets spreadsheet with a defined schema and matching on contact ID to avoid duplicates.

- **Nodes Involved:**  
  - Save Contacts to Google Sheets (Google Sheets Node)  
  - Sticky Note (Google Sheets Authentication & Setup implicit in notes)

- **Node Details:**

  - **Save Contacts to Google Sheets**  
    - Type: Google Sheets  
    - Role: Appends data rows to a specified Google Sheet.  
    - Configuration:  
      - Operation: Append  
      - Document ID: Predefined Google Sheet ID (template or user’s sheet).  
      - Sheet Name: `gid=0` (first sheet).  
      - Columns: Mapped fields including `id`, `firstName`, `lastName`, `fullName`, `location`, `title`, `companyName`, `companyLocation`, `industry`, and constructed `profileUrl` (built from `entityUrn`).  
      - Matching Columns: `id` to prevent duplicate entries.  
      - Type Conversion: Disabled.  
    - Connections: Outputs to "Check If More Pages Available".  
    - Edge Cases:  
      - Google Sheets API errors (authentication, quota exceeded).  
      - Mismatched or missing data fields causing incorrect rows.  
      - Sheet access permissions.  

---

#### 2.5 Pagination Control and Rate Limiting

- **Overview:**  
  This block manages pagination by checking if more pages are available to scrape, increments the start parameter, and enforces a random wait delay between API requests to avoid rate limiting or blocking by LinkedIn.

- **Nodes Involved:**  
  - Check If More Pages Available (IF Node)  
  - Increment Page Start Parameter (Set Node)  
  - Rate Limit Delay Between Requests (Wait Node)  
  - Sticky Note2 (Pagination Logic)  
  - Sticky Note3 (Rate Limiting Protection)

- **Node Details:**

  - **Check If More Pages Available**  
    - Type: IF  
    - Role: Evaluates if the current page index is less than total pages to decide whether to continue pagination.  
    - Configuration:  
      - Condition compares calculated current page index (`paging.start` offset divided by 25) with `total_pages` from parameters.  
    - Connections:  
      - True: To "Increment Page Start Parameter" (continue scraping next page).  
      - False: Terminates workflow (no further connections).  
    - Edge Cases:  
      - Incorrect calculation if API response format changes.  
      - `total_pages` parameter not set or invalid.

  - **Increment Page Start Parameter**  
    - Type: Set  
    - Role: Increases the pagination start index by 25 (page size).  
    - Configuration:  
      - Sets `start` = previous `paging.start` + 25.  
    - Connections: Outputs to "Rate Limit Delay Between Requests".  
    - Edge Cases: None significant.

  - **Rate Limit Delay Between Requests**  
    - Type: Wait  
    - Role: Pauses the workflow for a random duration between 30 and 60 seconds to mimic human behavior and avoid API blocking.  
    - Configuration:  
      - Wait time: Random integer between 30 and 60 seconds (inclusive).  
    - Connections: Outputs back to "Set Search Parameters" to trigger next page scrape.  
    - Edge Cases: None significant, but lowering delay risks LinkedIn blocks.

  - **Sticky Note2**  
    - Explains the pagination logic and looping behavior.

  - **Sticky Note3**  
    - Emphasizes the importance of rate limiting and warns not to lower the delay.

---

### 3. Summary Table

| Node Name                      | Node Type           | Functional Role                             | Input Node(s)             | Output Node(s)                | Sticky Note                                                                                     |
|-------------------------------|---------------------|---------------------------------------------|---------------------------|------------------------------|------------------------------------------------------------------------------------------------|
| Start Workflow                | Manual Trigger      | Workflow entry point                         |                           | Set Search Parameters         |                                                                                                |
| Set Search Parameters         | Set                 | Defines initial scraping parameters          | Start Workflow            | Scrape LinkedIn Contacts API  | Configuration Settings: cookies, URL, scraper_type, total_pages, pagination size                 |
| Scrape LinkedIn Contacts API  | HTTP Request        | Calls LinkedIn scraping API                   | Set Search Parameters     | Extract Contact Data Array    | API Access instructions: how to get API key and configure HTTP Header Auth                      |
| Extract Contact Data Array    | Code                | Extracts contacts array from API response     | Scrape LinkedIn Contacts API | Save Contacts to Google Sheets | Scraping Process: steps and data fields included                                               |
| Save Contacts to Google Sheets| Google Sheets       | Appends contact data to Google Sheets         | Extract Contact Data Array | Check If More Pages Available |                                                                                                |
| Check If More Pages Available | IF                  | Determines if pagination should continue      | Save Contacts to Google Sheets | Increment Page Start Parameter | Pagination Logic: loop until all pages scraped                                                |
| Increment Page Start Parameter| Set                 | Increments pagination start index             | Check If More Pages Available | Rate Limit Delay Between Requests |                                                                                                |
| Rate Limit Delay Between Requests | Wait             | Delays execution between requests             | Increment Page Start Parameter | Set Search Parameters         | Rate Limiting Protection: 30-60s random delay to avoid API blocks                              |
| Sticky Note                  | Sticky Note          | Configuration instructions                     |                           |                              | Configuration Settings: cookies, URL, scraper_type, total_pages                                |
| Sticky Note1                 | Sticky Note          | Scraping process explanation                    |                           |                              | Scraping Process details                                                                      |
| Sticky Note2                 | Sticky Note          | Pagination loop explanation                      |                           |                              | Pagination Logic                                                                              |
| Sticky Note3                 | Sticky Note          | Rate limiting warning and explanation            |                           |                              | Rate Limiting Protection                                                                      |
| Sticky Note4                 | Sticky Note          | API credentials acquisition instructions          |                           |                              | Get 1 Month Free API Access instructions with email link                                    |
| Sticky Note5                 | Sticky Note          | How to extract LinkedIn cookies                    |                           |                              | Instructions for EditThisCookie extension and cookie export steps                            |
| Sticky Note6                 | Sticky Note          | Full workflow description and requirements         |                           |                              | Overview of workflow, requirements, quick setup, and notes                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: "Start Workflow"  
   - No parameters; this node manually triggers the workflow.

2. **Create Set Node for Search Parameters**  
   - Name: "Set Search Parameters"  
   - Add fields:  
     - `cookies` (type: array) → Set to your LinkedIn session cookies array (replace placeholder).  
     - `url` (string) → Set to your LinkedIn Sales Navigator search URL.  
     - `scraper_type` (string) → Set to `"contacts"`.  
     - `start` (number) → Set to `0` or expression `={{ $input.first()?.json?.start ?? 0 }}` to default to zero.  
     - `total_pages` (number) → Set how many pages to scrape (default `2`).  
   - Connect "Start Workflow" output to this node input.

3. **Create HTTP Request Node to Scrape API**  
   - Name: "Scrape LinkedIn Contacts API"  
   - Method: POST  
   - URL: `https://api.nickautomations.com/linkedin/scrape`  
   - Authentication: HTTP Header Auth  
     - Create new credential named `[linkedin-scraper-api] Header Auth`  
     - Add header `x-api-key` with your API key value obtained by emailing the creator.  
   - Body Parameters (form type):  
     - `cookies`: Expression `={{ $json.cookies }}`  
     - `url`: Expression `={{ $json.url }}`  
     - `scraper_type`: Expression `={{ $json.scraper_type }}`  
     - `start`: Expression `={{ $json.start }}`  
   - Connect "Set Search Parameters" output to this node input.

4. **Create Code Node to Extract Contact Array**  
   - Name: "Extract Contact Data Array"  
   - Language: JavaScript  
   - Code:  
     ```javascript
     const items = $input.all();
     const payload = items[0].json;
     return payload.data;
     ```  
   - Connect "Scrape LinkedIn Contacts API" output to this node input.

5. **Create Google Sheets Node to Save Contacts**  
   - Name: "Save Contacts to Google Sheets"  
   - Operation: Append  
   - Google Sheets Document: Use your Google Sheets document ID (copy from your sheet URL).  
   - Sheet Name: First sheet (usually `gid=0`).  
   - Columns mapping:  
     - id: `={{ $json.id }}`  
     - firstName: `={{ $json.firstName }}`  
     - lastName: `={{ $json.lastName }}`  
     - fullName: `={{ $json.fullName }}`  
     - location: `={{ $json.location }}`  
     - title: `={{ $json.title }}`  
     - companyName: `={{ $json.companyName }}`  
     - companyLocation: `={{ $json.companyLocation }}`  
     - industry: `={{ $json.industry }}`  
     - profileUrl: `={{ 'https://www.linkedin.com/in/' + $json.entityUrn.split('(')[1].split(',')[0] }}`  
   - Matching Columns: `id` (to prevent duplicates)  
   - Connect "Extract Contact Data Array" output to this node input.  
   - Configure Google Sheets OAuth2 credentials with your Google account.

6. **Create IF Node to Check More Pages**  
   - Name: "Check If More Pages Available"  
   - Condition: Number comparison (less than)  
     - Left Value Expression:  
       `={{ $('Scrape LinkedIn Contacts API').item.json.paging.start / 25 }}`  
     - Right Value Expression:  
       `={{ $('Set Search Parameters').item.json.total_pages }}`  
   - Connect "Save Contacts to Google Sheets" output to IF node input.

7. **Create Set Node to Increment Pagination Start**  
   - Name: "Increment Page Start Parameter"  
   - Set `start` to expression:  
     `={{ $('Scrape LinkedIn Contacts API').item.json.paging.start + 25 }}`  
   - Connect IF node's "true" output to this node.

8. **Create Wait Node for Rate Limiting**  
   - Name: "Rate Limit Delay Between Requests"  
   - Wait Time: Expression for random seconds between 30 and 60:  
     `={{ Math.floor(Math.random() * (60 - 30 + 1)) + 30 }}`  
   - Connect "Increment Page Start Parameter" output to this node.

9. **Loop Back to Set Search Parameters**  
   - Connect "Rate Limit Delay Between Requests" output to the input of "Set Search Parameters" node, restarting the scraping with updated start parameter for next page.

10. **Termination Condition**  
    - IF node's "false" output (no more pages) leads to workflow end (no further connections).

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                     | Context or Link                                                                                                       |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| Get 1 Month Free API Access: Email `nchoudhary110792@gmail.com` with subject "LinkedIn Scraper API - Free 1 Month Access Request" including your name, email, and use case. API keys are delivered within 24 hours.                                                                                                                                                                             | Email link: [mailto:nchoudhary110792@gmail.com?subject=LinkedIn%20Scraper%20API%20-%20Free%201%20Month%20Access%20Request&body=Hi%20Naveen%2C%0A%0AI%20would%20like%20to%20request%201%20month%20of%20free%20access%20to%20the%20LinkedIn%20Scraper%20API.%0A%0AName%3A%20%0AEmail%3A%20%0AUse%20Case%3A%20%0A%0AThank%20you!] |
| How to Get LinkedIn Cookies: Install the EditThisCookie browser extension, export cookies from LinkedIn Sales Navigator, and paste them into the workflow parameter. Cookies are used only to authenticate your session and are not stored elsewhere. Visual guide available at Google Drive link.                                                                                                   | Extension: https://chromewebstore.google.com/detail/editthiscookie-v3/ojfebgpkimhlhcblbalbfjblapadhbol<br>Guide: https://drive.google.com/file/d/1yY4xdXjrChAeKGWz3H6lqyp1lJ1ElKWU/view?usp=sharing |
| Workflow Requirements: Self-hosted n8n instance (does not work on n8n cloud), LinkedIn Sales Navigator account, API credentials, Google Sheets account.                                                                                                                                                                                                                                       | Sticky Note6 content                                                                                                  |
| Rate Limiting: Maintain random wait between 30-60 seconds to avoid LinkedIn blocking. Do not reduce delay.                                                                                                                                                                                                                                                                                       | Sticky Note3 content                                                                                                  |

---

This completes the detailed documentation and reference for the "Extract LinkedIn Sales Navigator Contacts to Google Sheets with Auto Pagination" workflow.