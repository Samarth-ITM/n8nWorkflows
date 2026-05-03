Extract Instagram Profile Data with Apify and Store in Google Sheets

https://n8nworkflows.xyz/workflows/extract-instagram-profile-data-with-apify-and-store-in-google-sheets-4587


# Extract Instagram Profile Data with Apify and Store in Google Sheets

---
### 1. Workflow Overview

This workflow automates the process of extracting Instagram profile data by integrating n8n with the Apify Instagram Scraper and Google Sheets. It is designed for marketing teams, social analysts, and growth professionals who want to collect and store Instagram profile metrics efficiently without manual data entry or coding.

**Use Cases:**  
- Influencer outreach and campaign planning  
- Competitor and market analysis  
- CRM enrichment with up-to-date social media data  

**Logical Blocks:**

- **1.1 Input Reception:** Captures Instagram usernames via a form submission trigger.  
- **1.2 Data Extraction:** Sends the username to Apify’s Instagram Scraper via an HTTP POST request and receives structured profile data.  
- **1.3 Data Formatting:** Cleans and organizes the raw API response to match the Google Sheets schema.  
- **1.4 Data Storage:** Appends the formatted data as a new row into a specified Google Sheets document.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block initiates the workflow when a user submits an Instagram username through a form. It captures the input and forwards it downstream for processing.

- **Nodes Involved:**  
  - Provide Usernames

- **Node Details:**  

  **Provide Usernames**  
  - **Type:** Form Trigger  
  - **Role:** Entry point capturing Instagram usernames dynamically.  
  - **Configuration:**  
    - Form Title: "Instagram profile scraper"  
    - Single required field labeled "Username"  
    - No preset options for usernames; input is mandatory.  
  - **Input/Output:**  
    - Input: HTTP form submission payload, e.g., `{ "username": "influencer_1" }`  
    - Output: JSON object with key `Username` carrying submitted username string.  
  - **Failure Modes:**  
    - Missing required username leads to no trigger or form rejection.  
    - Network or webhook misconfiguration can prevent trigger activation.  
  - **Version Notes:** Uses n8n Form Trigger node version 2.2.

---

#### 1.2 Data Extraction

- **Overview:**  
  This block sends the submitted username to Apify’s Instagram Scraper API to retrieve public profile data synchronously.

- **Nodes Involved:**  
  - Scrape Instagram Profile via Apify

- **Node Details:**  

  **Scrape Instagram Profile via Apify**  
  - **Type:** HTTP Request  
  - **Role:** Executes a POST request to Apify to scrape Instagram data.  
  - **Configuration:**  
    - URL format: `https://api.apify.com/v2/actor-tasks/<TASK_ID>/run-sync-get-dataset-items?token=<YOUR_API_TOKEN>`  
    - Method: POST  
    - Headers: Content-Type automatically JSON  
    - Body: JSON including usernames array with the submitted username, e.g.:  
      ```json
      {
        "input": {
          "usernames": ["={{ $json.Username }}"],
          "resultsLimit": 1
        }
      }
      ```  
    - Send body as JSON.  
  - **Input/Output:**  
    - Input: JSON with `Username` from form trigger.  
    - Output: Array containing profile data objects with fields like:  
      `username`, `fullName`, `followersCount`, `followsCount`, `biography`, `profilePicUrl`, `externalUrl`, etc.  
  - **Failure Modes:**  
    - Invalid or missing Apify TASK_ID or API token results in 401 or 403 errors.  
    - Network timeouts or API limits may cause failures.  
    - If username does not exist or is private, response may be empty or incomplete.  
    - Expression errors if `$json.Username` is missing or malformed.  
  - **Version Notes:** HTTP Request node version 4.2.  
  - **Security:** API token must be securely stored and never exposed in plaintext.

---

#### 1.3 Data Formatting

- **Overview:**  
  Transforms the raw Apify response into a consistent schema aligned with the Google Sheets columns, ensuring clean and usable data.

- **Nodes Involved:**  
  - Format Instagram Profile Data

- **Node Details:**  

  **Format Instagram Profile Data**  
  - **Type:** Set node  
  - **Role:** Assigns and renames fields extracted from Apify response.  
  - **Configuration:**  
    - Sets the following fields explicitly from the input JSON:  
      - `username` = `{{$json.username}}`  
      - `fullName` = `{{$json.fullName}}`  
      - `followersCount` = `{{$json.followersCount}}`  
      - `followsCount` = `{{$json.followsCount}}`  
      - `biography` = `{{$json.biography}}`  
      - `profilePicUrl` = `{{$json.profilePicUrl}}`  
    - Note: The workflow does not map `externalUrl` although available.  
  - **Input/Output:**  
    - Input: Output array from Apify (single profile object).  
    - Output: Single JSON object with normalized fields.  
  - **Failure Modes:**  
    - Missing fields in Apify response could lead to empty or undefined values.  
    - Expression failures if source JSON structure changes.  
  - **Version Notes:** Set node version 3.4.

---

#### 1.4 Data Storage

- **Overview:**  
  Appends the formatted Instagram profile data into a Google Sheets spreadsheet, enabling persistent storage and further analysis.

- **Nodes Involved:**  
  - Append Profile to Google Sheet

- **Node Details:**  

  **Append Profile to Google Sheet**  
  - **Type:** Google Sheets node  
  - **Role:** Adds a new row to a specified sheet with profile details.  
  - **Configuration:**  
    - Operation: Append row  
    - Document ID: Specific Google Sheets document ID configured in parameters.  
    - Sheet Name: Specifies the sheet by GID (gid=0)  
    - Columns mapped explicitly:  
      - Username -> `{{$json.username}}`  
      - Biography -> `{{$json.biography}}`  
      - Full Name -> `{{$json.fullName}}`  
      - Followers Count -> `{{$json.followersCount}}`  
      - Following Count -> `{{$json.followsCount}}`  
      - Profile Pic URL -> `{{$json.profilePicUrl}}`  
    - Mapping mode: Define below schema; no automatic schema discovery or conversion.  
  - **Input/Output:**  
    - Input: Normalized JSON from the Set node.  
    - Output: Confirmation of append operation (row ID or status)  
  - **Failure Modes:**  
    - OAuth2 credential expiration or permission issues can cause authorization errors.  
    - Incorrect document ID or sheet name leads to append failure.  
    - API rate limits or network errors may interrupt operation.  
  - **Version Notes:** Google Sheets node version 4.5  
  - **Credentials:** Requires OAuth2 credentials configured in n8n with proper Google Sheets access.

---

### 3. Summary Table

| Node Name                      | Node Type         | Functional Role                         | Input Node(s)               | Output Node(s)                   | Sticky Note                                                                                                          |
|-------------------------------|-------------------|---------------------------------------|-----------------------------|---------------------------------|----------------------------------------------------------------------------------------------------------------------|
| Provide Usernames              | Form Trigger      | Captures Instagram username input     | —                           | Scrape Instagram Profile via Apify | ### 🔹 **1. `Trigger: On Form Submission`**<br>Starts workflow on username form submission.                          |
| Scrape Instagram Profile via Apify | HTTP Request     | Scrapes profile data using Apify API  | Provide Usernames            | Format Instagram Profile Data    | ### 🔹 **2. `Scrape Instagram Profile via Apify`**<br>Posts username to Apify, retrieves profile data.              |
| Format Instagram Profile Data  | Set               | Formats and normalizes scraped data   | Scrape Instagram Profile via Apify | Append Profile to Google Sheet | ### 🔹 **3. `Format Instagram Profile Data`**<br>Cleans and organizes data for Google Sheets.                        |
| Append Profile to Google Sheet | Google Sheets     | Appends formatted data to Google Sheet| Format Instagram Profile Data | —                               | ### 🔹 **4. `Append Profile to Google Sheet`**<br>Stores data in Google Sheets for persistent tracking.               |
| Sticky Note                   | Sticky Note       | Workflow step explanation              | —                           | —                               | Detailed explanation of step 1 & 2 with JSON and API examples                                                        |
| Sticky Note1                  | Sticky Note       | Workflow step explanation              | —                           | —                               | Detailed explanation of step 3 & 4 with data mapping and Google Sheets description                                    |
| Sticky Note4                  | Sticky Note       | Workflow overview and use cases        | —                           | —                               | Comprehensive workflow overview, purpose, use cases, and architecture                                               |
| Sticky Note9                  | Sticky Note       | Support contact and resource links     | —                           | —                               | Support contact info and external links for workflow assistance                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger Node ("Provide Usernames"):**  
   - Node Type: Form Trigger  
   - Configure form:  
     - Title: "Instagram profile scraper"  
     - Add one field labeled "Username" (string, required)  
   - This will be the workflow entry point.

2. **Add an HTTP Request Node ("Scrape Instagram Profile via Apify"):**  
   - Connect from "Provide Usernames" node output.  
   - Set method to POST.  
   - URL: `https://api.apify.com/v2/actor-tasks/<TASK_ID>/run-sync-get-dataset-items?token=<YOUR_API_TOKEN>` (replace `<TASK_ID>` and `<YOUR_API_TOKEN>` with your actual Apify credentials).  
   - Headers: Content-Type `application/json` (usually automatic).  
   - Body (JSON):  
     ```json
     {
       "input": {
         "usernames": ["={{ $json.Username }}"],
         "resultsLimit": 1
       }
     }
     ```  
   - Ensure body is sent as JSON.

3. **Add a Set Node ("Format Instagram Profile Data"):**  
   - Connect output from HTTP Request node.  
   - Configure fields to assign:  
     - `username` = `{{$json.username}}`  
     - `fullName` = `{{$json.fullName}}`  
     - `followersCount` = `{{$json.followersCount}}`  
     - `followsCount` = `{{$json.followsCount}}`  
     - `biography` = `{{$json.biography}}`  
     - `profilePicUrl` = `{{$json.profilePicUrl}}`  

4. **Add Google Sheets Node ("Append Profile to Google Sheet"):**  
   - Connect from Set node output.  
   - Operation: Append Row  
   - Configure authentication using OAuth2 credentials with Google Sheets access.  
   - Document ID: Use your target Google Sheets document’s ID.  
   - Sheet Name: Set to your target sheet (e.g., gid=0 or "Sheet1").  
   - Map columns explicitly:  
     - Username → `{{$json.username}}`  
     - Biography → `{{$json.biography}}`  
     - Full Name → `{{$json.fullName}}`  
     - Followers Count → `{{$json.followersCount}}`  
     - Following Count → `{{$json.followsCount}}`  
     - Profile Pic URL → `{{$json.profilePicUrl}}`  

5. **Save and Activate Workflow:**  
   - Test by submitting a username via the form trigger UI or HTTP request.  
   - Monitor the flow and data appending in your Google Sheet.

**Notes:**  
- Replace `<TASK_ID>` and `<YOUR_API_TOKEN>` with actual Apify task ID and API token, securely stored as environment variables or n8n credentials.  
- Google Sheets OAuth2 credentials must be created and authorized correctly for appending data.  
- Validate field mappings and sheet permissions to avoid write errors.  
- Optional: Add error handling nodes (e.g., IF, Error Trigger) for robustness.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                    | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow assistance contact: Yaron@nofluff.online. Explore tips and tutorials on YouTube and LinkedIn for n8n automation and integrations.                                                                                                                                                                                                                                                                                      | YouTube: https://www.youtube.com/@YaronBeen/videos<br>LinkedIn: https://www.linkedin.com/in/yaronbeen/ |
| For securing API tokens and credentials, store sensitive data as n8n environment variables to enhance security and avoid exposure in workflows.                                                                                                                                                                                                                                                                               | n8n Environment Variables Documentation: https://docs.n8n.io/hosting/environment-variables/       |
| The workflow is designed to be extendable with enhancements such as looping through batch usernames, avoiding duplicates, and triggering alerts upon data updates. These improvements can be added with additional n8n nodes based on specific requirements.                                                                                                                                                                      | General workflow extension ideas                                                                  |
| The Apify Instagram Scraper API returns a rich dataset including verification status, post counts, and external URLs, which can be included in custom workflow extensions.                                                                                                                                                                                                                                                       | Apify Instagram Scraper Documentation: https://apify.com/apify/instagram-scraper                  |

---

*Disclaimer: The text provided is exclusively extracted from an automated workflow created with n8n, adhering strictly to content policies and containing no illegal or protected elements. All processed data is legal and publicly accessible.*