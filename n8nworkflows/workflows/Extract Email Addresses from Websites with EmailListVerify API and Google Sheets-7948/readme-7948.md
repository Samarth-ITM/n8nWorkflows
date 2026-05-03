Extract Email Addresses from Websites with EmailListVerify API and Google Sheets

https://n8nworkflows.xyz/workflows/extract-email-addresses-from-websites-with-emaillistverify-api-and-google-sheets-7948


# Extract Email Addresses from Websites with EmailListVerify API and Google Sheets

### 1. Workflow Overview

This workflow automates the extraction of email addresses from a list of websites, using a two-step approach: first scraping the website HTML for visible email addresses, and if none are found, querying the EmailListVerify API to guess generic emails related to the domain. The final results, including the website, email, and confidence level, are appended or updated in a Google Sheet.

The workflow is logically divided into these blocks:

- **1.1 Input Reception and Preparation:** Retrieves website URLs from a Google Sheet, ensures URLs have the correct HTTP prefix.
- **1.2 Website Scraping for Emails:** Requests website HTML, extracts email addresses using regex.
- **1.3 Email Existence Check & Branching:** Checks if emails were found; if yes, processes them, if no, invokes EmailListVerify API.
- **1.4 EmailListVerify API Query:** Uses the domain extracted from the website URL to call EmailListVerify API to find generic emails.
- **1.5 Data Transformation and Output:** Transforms data formats as needed and appends or updates the output Google Sheet with found emails.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Preparation

- **Overview:**  
  Retrieves website URLs from a Google Sheets document and ensures each URL starts with "http" for valid HTTP requests.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Get input Data (Google Sheets)  
  - Add http to url if missing (Code)  
  - Merge (Merge)

- **Node Details:**  
  1. **When clicking ‘Execute workflow’**  
     - Type: Manual Trigger  
     - Role: Entry point for manual execution  
     - Connections: Outputs to Get input Data  
     - Edge Cases: None (manual trigger)  

  2. **Get input Data**  
     - Type: Google Sheets  
     - Role: Reads website URLs from the "Input" sheet of a specified Google Sheets document  
     - Config: OAuth2 credentials required, uses URL of the Google Sheet, sheet name "Input"  
     - Output: JSON objects with a "website" field  
     - Edge Cases: Authentication failures, sheet not found, empty data  

  3. **Add http to url if missing**  
     - Type: Code (JavaScript)  
     - Role: Checks each URL, prepends "http://" if missing to ensure valid HTTP requests  
     - Key logic: Uses string indexOf to detect "http" prefix and modifies URL accordingly  
     - Input: JSON with "website" field  
     - Output: JSON with "url" field (normalized URL)  
     - Edge Cases: Malformed URLs, missing website field, code exceptions handled by try/catch  

  4. **Merge**  
     - Type: Merge  
     - Role: Combines outputs from multiple nodes into one dataset  
     - Mode: Combine by position  
     - Input: Merges outputs from Add http to url if missing and Extract the emails found (later in flow)  
     - Edge Cases: Mismatched input lengths could cause misalignment  

---

#### 2.2 Website Scraping for Emails

- **Overview:**  
  Sends HTTP requests to each website URL, scrapes HTML content to extract email addresses using regex.

- **Nodes Involved:**  
  - Get the website data (HTTP Request)  
  - Extract the emails found (Set)

- **Node Details:**  
  1. **Get the website data**  
     - Type: HTTP Request  
     - Role: Retrieves raw HTML content from each URL  
     - Config: URL set dynamically from input JSON `{{$json.url}}`, retries enabled on failure, continues on error  
     - Output: Raw website HTML under `data` field  
     - Edge Cases: Connection timeouts, invalid URLs, HTTP errors (404, 500 etc.), handled by continuing workflow  

  2. **Extract the emails found**  
     - Type: Set  
     - Role: Extracts email addresses from the raw HTML using a regex pattern for emails  
     - Key expression: `{{$json.data.match(/(?:[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,})/g)}}`  
     - Output: Array of found emails under "Email" field  
     - Edge Cases: No matches result in null or empty arrays, regex might miss some valid but unusual emails  

---

#### 2.3 Email Existence Check & Branching

- **Overview:**  
  Checks if the scraping step found any emails. If yes, the workflow proceeds to process these emails; if no, it falls back to querying the EmailListVerify API.

- **Nodes Involved:**  
  - Email found? (If)  
  - Split Out (Split Out)

- **Node Details:**  
  1. **Email found?**  
     - Type: If  
     - Role: Conditional check on whether the "Email" array length is greater than 0  
     - Condition: `lengthGt 0` on the array `{{$json.Email}}`  
     - Outputs:  
       - True: Emails found, proceed to Split Out node  
       - False: No emails found, proceed to Transform website into domain name1 node (to prepare for API call)  
     - Edge Cases: Null or missing "Email" field causes false branch  
     
  2. **Split Out**  
     - Type: Split Out  
     - Role: Splits the array of emails into individual items for further processing  
     - Config:  
       - Field to split: "Email"  
       - Include other fields: "website"  
       - Output field name: "email"  
     - Output: One item per email with associated website  
     - Edge Cases: Empty input leads to no output items  

---

#### 2.4 EmailListVerify API Query

- **Overview:**  
  For domains where no emails were found by scraping, this block extracts the domain name from the URL and calls the EmailListVerify API to find generic email addresses.

- **Nodes Involved:**  
  - Transform website into domain name1 (Code)  
  - Use EmailListVerify API to find generic emails (HTTP Request)  
  - Transform website into domain name (Code)

- **Node Details:**  
  1. **Transform website into domain name1**  
     - Type: Code (JavaScript)  
     - Role: Parses the input URLs to extract the domain name (e.g., "example.com") for API use  
     - Logic:  
       - Extracts substring between "//" and next "/" in URL; removes "www." prefix if present  
       - Handles URLs without slashes by using the whole URL  
     - Input: JSON with "website" field  
     - Output: JSON with added "domain" field  
     - Edge Cases: URLs without expected format, exceptions caught and logged  

  2. **Use EmailListVerify API to find generic emails**  
     - Type: HTTP Request  
     - Role: Calls EmailListVerify's `findContact` API endpoint with the domain to get generic emails  
     - Config:  
       - POST method  
       - JSON body: `{"domain": "domain_from_previous_node"}`  
       - Authentication: HTTP Header Auth with API key (credential required)  
     - Output: API response with possible generic emails and confidence levels  
     - Edge Cases: API key invalid or missing, rate limits, network errors  

  3. **Transform website into domain name**  
     - Type: Code (JavaScript)  
     - Role: Parses the email addresses returned by EmailListVerify to extract the domain part and sets it as "website" field for output  
     - Logic: Extract substring after "@" in email address  
     - Output: JSON with fields "website", "email", and likely "confidence"  
     - Edge Cases: Malformed emails, exceptions handled  

---

#### 2.5 Data Transformation and Output

- **Overview:**  
  Splits arrays of emails into individual entries and appends or updates the results into an output Google Sheet.

- **Nodes Involved:**  
  - Split Out (already described above)  
  - Add result to google (Google Sheets)

- **Node Details:**  
  1. **Add result to google**  
     - Type: Google Sheets  
     - Role: Appends or updates rows in the "Output" sheet of the specified Google Sheets document  
     - Config:  
       - Document ID and sheet name "Output" provided  
       - Columns mapped automatically: "website", "email", "confidence"  
       - Operation: Append or update existing rows based on matching columns  
       - OAuth2 credentials required  
     - Edge Cases: Authentication errors, sheet access issues, schema mismatch  

---

### 3. Summary Table

| Node Name                         | Node Type            | Functional Role                             | Input Node(s)             | Output Node(s)                      | Sticky Note                                                                                                            |
|----------------------------------|----------------------|---------------------------------------------|---------------------------|-----------------------------------|------------------------------------------------------------------------------------------------------------------------|
| Sticky Note                      | Sticky Note          | Comment: Scraping emails from websites      |                           |                                   | * Scraping emails from websites using an html                                                                        |
| Sticky Note2                     | Sticky Note          | Instructions and setup notes                 |                           |                                   | # How to get emails from websites; workflow explanation, setup instructions, links to Google Sheet and EmailListVerify |
| When clicking ‘Execute workflow’ | Manual Trigger       | Workflow entry point                         |                           | Get input Data                    |                                                                                                                        |
| Get input Data                   | Google Sheets        | Reads input websites from Google Sheet       | When clicking ‘Execute workflow’ | Add http to url if missing        | ## copy template; https://docs.google.com/spreadsheets/d/1VOTFM8UeWHhJbtBM7SRca6vsVJlRUXzX71kjJ8n2jUY/edit             |
| Add http to url if missing       | Code                 | Adds http prefix to URLs if missing          | Get input Data             | Get the website data              |                                                                                                                        |
| Get the website data             | HTTP Request         | Fetches website HTML content                  | Add http to url if missing | Extract the emails found          |                                                                                                                        |
| Extract the emails found         | Set                  | Extracts emails from HTML using regex        | Get the website data       | Merge                           |                                                                                                                        |
| Merge                          | Merge                | Combines outputs from email extraction and input | Add http to url if missing, Extract the emails found | Email found?                     |                                                                                                                        |
| Email found?                    | If                   | Checks if emails were found                    | Merge                     | Split Out (true), Transform website into domain name1 (false) | * Check if email was found                                                                                                |
| Split Out                      | Split Out            | Splits email array into individual items       | Email found? (true branch) | Add result to google             | * Add result to google sheet                                                                                            |
| Transform website into domain name1 | Code                 | Extracts domain from website URL               | Email found? (false branch) | Use EmailListVerify API to find generic emails |                                                                                                                        |
| Use EmailListVerify API to find generic emails | HTTP Request         | Calls EmailListVerify API to find generic emails | Transform website into domain name1 | Transform website into domain name | ## Add your ELV API Key; Start a free trial at EmailListVerify.com and get your API key                                  |
| Transform website into domain name | Code                 | Extracts domain from email returned by API     | Use EmailListVerify API to find generic emails | Add result to google             | ## Find email address with EmailListVerify; Find generic email address with EmailListVerify API if the website is known |
| Add result to google            | Google Sheets        | Appends or updates results in output Google Sheet | Split Out, Transform website into domain name |                                   | ## copy template; https://docs.google.com/spreadsheets/d/1VOTFM8UeWHhJbtBM7SRca6vsVJlRUXzX71kjJ8n2jUY/edit             |
| Sticky Note1                   | Sticky Note          | Comment for email checking step                |                           |                                   | * Check if email was found                                                                                            |
| Sticky Note3                   | Sticky Note          | Comment for adding results to Google Sheet     |                           |                                   | * Add result to google sheet                                                                                          |
| Sticky Note5                   | Sticky Note          | Reminder to add EmailListVerify API key        |                           |                                   | ## Add your ELV API Key                                                                                                |
| Sticky Note6                   | Sticky Note          | Explanation of EmailListVerify API usage       |                           |                                   | ## Find email address with EmailListVerify                                                                           |
| Sticky Note7                   | Sticky Note          | Reminder to copy Google Sheet template          |                           |                                   | ## copy template; File need to include column email, website é confidence                                             |
| Sticky Note8                   | Sticky Note          | Reminder to add target URLs to Input sheet      |                           |                                   | ## copy template; Add target url in Input sheet                                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: When clicking ‘Execute workflow’  
   - Purpose: Start workflow manually  

2. **Create Google Sheets Node to Get Input Data**  
   - Name: Get input Data  
   - Operation: Read data  
   - Sheet Name: "Input"  
   - Google Sheet Document: Use URL https://docs.google.com/spreadsheets/d/1VOTFM8UeWHhJbtBM7SRca6vsVJlRUXzX71kjJ8n2jUY/edit#gid=0  
   - Credential: Set up Google Sheets OAuth2 credentials  

3. **Create Code Node to Add HTTP Prefix if Missing**  
   - Name: Add http to url if missing  
   - Code: JavaScript code to check if "website" field starts with "http" and prepend it if missing; output field renamed to "url"  
   - Input: Data from Get input Data node  

4. **Create HTTP Request Node to Get Website Data**  
   - Name: Get the website data  
   - HTTP Method: GET  
   - URL: Set dynamically to `{{$json.url}}`  
   - Retry on Fail: Enabled  
   - On Error: Continue Regular Output (to avoid stopping workflow if website unreachable)  

5. **Create Set Node to Extract Emails from Website HTML**  
   - Name: Extract the emails found  
   - Set Field: "Email"  
   - Value: Use regex expression `{{$json.data.match(/(?:[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,})/g)}}` to extract emails array  

6. **Create Merge Node**  
   - Name: Merge  
   - Mode: Combine by position  
   - Inputs: Connect outputs from Add http to url if missing and Extract the emails found  

7. **Create If Node to Check if Emails Found**  
   - Name: Email found?  
   - Condition: Check if length of "Email" array > 0  

8. **Create Split Out Node to Split Emails Array**  
   - Name: Split Out  
   - Field to Split Out: "Email"  
   - Include Other Fields: "website"  
   - Destination Field Name: "email"  

9. **Create Google Sheets Node to Add Result to Output Sheet**  
   - Name: Add result to google  
   - Operation: Append or Update  
   - Sheet Name: "Output"  
   - Google Sheet Document: Use URL https://docs.google.com/spreadsheets/d/1VOTFM8UeWHhJbtBM7SRca6vsVJlRUXzX71kjJ8n2jUY/edit#gid=1538095319  
   - Columns: website, email, confidence  
   - Credential: Google Sheets OAuth2  

10. **Create Code Node to Transform Website into Domain Name (Fallback)**  
    - Name: Transform website into domain name1  
    - Code: Extract domain from URL, removing "www." if present, handle exceptions  

11. **Create HTTP Request Node to Use EmailListVerify API**  
    - Name: Use EmailListVerify API to find generic emails  
    - HTTP Method: POST  
    - URL: https://api.emaillistverify.com/api/findContact  
    - Body: JSON `{ "domain": "{{ $json.domain }}" }`  
    - Send Body as JSON: true  
    - Authentication: Use HTTP Header Auth with your EmailListVerify API key credential  
    - Handle possible errors and rate limits accordingly  

12. **Create Code Node to Transform Emails from EmailListVerify into Output Format**  
    - Name: Transform website into domain name  
    - Code: Extract domain part from email and set as "website" field for output  

13. **Connect Nodes**  
    - Manual Trigger → Get input Data → Add http to url if missing → Get the website data → Extract the emails found → Merge → Email found?  
    - Email found? True → Split Out → Add result to google  
    - Email found? False → Transform website into domain name1 → Use EmailListVerify API → Transform website into domain name → Add result to google  

14. **Add Sticky Notes at Key Points**  
    - Add instructional sticky notes with provided content for user guidance and API key reminders  

15. **Credentials Setup**  
    - Configure Google Sheets OAuth2 for Google Sheets nodes  
    - Configure HTTP Header Auth credential with your EmailListVerify API key for the HTTP Request node  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                              | Context or Link                                                                                                        |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| Workflow retrieves emails first by scraping website HTML, then falls back to EmailListVerify API for generic emails if none found.                                       | Workflow Logic                                                                                                         |
| Google Sheet template required for input and output: https://docs.google.com/spreadsheets/d/1VOTFM8UeWHhJbtBM7SRca6vsVJlRUXzX71kjJ8n2jUY/edit?gid=1538095319#gid=1538095319    | Input and Output Data Management                                                                                        |
| EmailListVerify API key is mandatory. Start a free trial at https://emaillistverify.com/                                                                                   | API Authentication                                                                                                     |
| This workflow is more suited for small businesses, as it typically returns generic emails like "contact@" which are monitored by owners, less suitable for enterprises. | Use Case Advisory                                                                                                      |
| Ensure retry and error continuation settings are enabled on HTTP request nodes to handle transient network or website errors gracefully.                                  | Resilience and Error Handling                                                                                          |

---

This documentation provides a full understanding of the workflow’s structure, node roles, configuration, and execution flow, enabling reproduction, customization, and troubleshooting.