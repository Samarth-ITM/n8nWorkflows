Improving Title Clarity with Google Sheets and Email Notifications

https://n8nworkflows.xyz/workflows/improving-title-clarity-with-google-sheets-and-email-notifications-8827


# Improving Title Clarity with Google Sheets and Email Notifications

### 1. Workflow Overview

This workflow automates the process of running a YouTube video giveaway by scraping YouTube comments from a submitted video URL, randomly selecting a winner from the commenters, logging the winner’s information to a Google Sheet, and sending notification emails accordingly. It is designed for scenarios where an organizer wants to quickly and fairly pick a winner from YouTube video commenters, maintaining records and notifying stakeholders via email.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Triggered by a form submission capturing the YouTube video URL for the giveaway.
- **1.2 YouTube Comments Retrieval:** Sends a POST request to a RapidAPI YouTube comments scraper service to fetch comments.
- **1.3 API Response Validation:** Checks if the API call succeeded (HTTP 200); routes workflow accordingly.
- **1.4 Winner Selection:** Extracts commenter names and randomly selects one as the winner.
- **1.5 Error Notification:** Sends an email if the API response is invalid or the call fails.
- **1.6 Winner Logging:** Appends the winner’s details along with the URL and submission date to a Google Sheet.
- **1.7 Winner Notification:** Sends a congratulatory email announcing the winner.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Initiates the workflow when a user submits a form containing a YouTube video URL for the giveaway.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**  
  - **Node:** On form submission  
    - **Type:** Form Trigger  
    - **Role:** Entry point that listens for form submissions titled "YouTube Video giveaway"  
    - **Configuration:**  
      - Form Title: "YouTube Video giveaway"  
      - Form Field: Single required field named "url" to capture YouTube video URL  
      - Form Description: "YouTube Video giveaway"  
    - **Input/Output:** No input; outputs form submission data including the URL and submission timestamp  
    - **Failure Cases:** Form submission errors or missing required field will prevent workflow trigger  
    - **Sticky Note:** "Triggers the workflow when a form with a YouTube URL is submitted. Captures the URL input for further processing."

---

#### 1.2 YouTube Comments Retrieval

- **Overview:**  
  Sends a POST request with the submitted URL to a third-party API to scrape YouTube comments.

- **Nodes Involved:**  
  - Fetch YouTube Comments

- **Node Details:**  
  - **Node:** Fetch YouTube Comments  
    - **Type:** HTTP Request  
    - **Role:** Calls RapidAPI YouTube comments scraper endpoint using POST with multipart/form-data  
    - **Configuration:**  
      - URL: https://youtube-comments-scraper.p.rapidapi.com/comment.php  
      - Method: POST  
      - Body: Multipart form data with "url" parameter set from form submission (`{{$json.url}}`)  
      - Headers: Includes `x-rapidapi-host` and `x-rapidapi-key` for authentication (API key placeholder present, must be replaced)  
      - Response: Full HTTP response enabled to access status code and body  
    - **Input:** Receives `url` parameter from form submission node  
    - **Output:** API response JSON containing comment items and HTTP status code  
    - **Failure Cases:**  
      - HTTP errors (non-200)  
      - Invalid or malformed API key or quota exceeded  
      - Network timeouts or RapidAPI endpoint down  
    - **Sticky Note:** "Sends a POST request to RapidAPI to scrape YouTube comments from the given URL. Includes API key and necessary headers for authentication."

---

#### 1.3 API Response Validation

- **Overview:**  
  Validates the HTTP status code from the API response, routing the workflow based on success or failure.

- **Nodes Involved:**  
  - Check API Response Status

- **Node Details:**  
  - **Node:** Check API Response Status  
    - **Type:** If (Conditional)  
    - **Role:** Checks if `statusCode` equals 200 to confirm successful API call  
    - **Configuration:**  
      - Condition: Numeric equality — left value from response JSON `{{$json.statusCode}}` equals 200  
      - Loose type validation enabled for robustness  
    - **Input:** Receives full HTTP response from Fetch YouTube Comments  
    - **Output:**  
      - True branch: Proceed to winner selection  
      - False branch: Trigger failure notification  
    - **Failure Cases:** None internal; relies on previous node’s output presence  
    - **Sticky Note:** "Checks if the API response status code equals 200 (success). Routes workflow based on success or failure of the API call."

---

#### 1.4 Winner Selection

- **Overview:**  
  Extracts all top-level comment authors from the API response and randomly selects one as the giveaway winner.

- **Nodes Involved:**  
  - Select Random Commenter

- **Node Details:**  
  - **Node:** Select Random Commenter  
    - **Type:** Code (JavaScript)  
    - **Role:** Parses comments array from API response, extracts author display names, filters out invalid entries, and randomly picks one  
    - **Configuration:**  
      - JavaScript code safely accesses nested properties to avoid runtime errors  
      - Returns a JSON object with `randomAuthorDisplayName` field  
    - **Input:** Receives API response body with comments from Check API Response Status (true branch)  
    - **Output:** JSON with randomly selected winner’s display name or null if no authors found  
    - **Failure Cases:**  
      - Empty comment list resulting in null winner  
      - Unexpected API response structure causing exceptions (mitigated by safe property access)  
    - **Sticky Note:** "Extracts author names from the fetched comments. Randomly selects one commenter as the giveaway winner."

---

#### 1.5 Error Notification

- **Overview:**  
  Sends an email alerting the admin that the API request failed or returned an invalid response.

- **Nodes Involved:**  
  - Notify: Invalid API Response

- **Node Details:**  
  - **Node:** Notify: Invalid API Response  
    - **Type:** Email Send  
    - **Role:** Sends failure notification email to admin for manual intervention  
    - **Configuration:**  
      - Recipients: owner@gmail.com (example email)  
      - Subject: "YouTube Comments Scraper: API Request Failed"  
      - Body: Explains the failure and suggests verifying URL and API key  
      - From Email: admin@gmail.com  
      - SMTP credentials configured (secure credentials required)  
    - **Input:** Triggered if API response status code is not 200  
    - **Output:** None (terminal for failure branch)  
    - **Failure Cases:** SMTP authentication errors or misconfiguration could prevent email sending  
    - **Sticky Note:** "Sends an email notification if the API call fails or returns an invalid response. Alerts the admin to verify URL or API credentials."

---

#### 1.6 Winner Logging

- **Overview:**  
  Appends the winner’s details, video URL, and submission date to a Google Sheet for record-keeping.

- **Nodes Involved:**  
  - Log Winner to Google Sheet

- **Node Details:**  
  - **Node:** Log Winner to Google Sheet  
    - **Type:** Google Sheets  
    - **Role:** Appends a row containing the giveaway URL, submission date, and winner name  
    - **Configuration:**  
      - Operation: Append  
      - Sheet: "Sheet1" (gid=0)  
      - Document ID: Not provided in JSON, needs to be set during setup  
      - Columns mapped:  
        - "Url" from form submission URL  
        - "Date" from form submission timestamp (`submittedAt`)  
        - "Winner" from random author selected  
      - Authentication: Service account credentials (Google API)  
    - **Input:** Receives data from Select Random Commenter node  
    - **Output:** Passes data forward for notification  
    - **Failure Cases:**  
      - Authentication issues with Google API service account  
      - Invalid or missing document ID or sheet name  
      - API rate limits or quota exceeded  
    - **Sticky Note:** "Appends the winner’s name, URL, and submission date to a Google Sheet. Uses service account authentication for Google Sheets access."

---

#### 1.7 Winner Notification

- **Overview:**  
  Sends a congratulatory email announcing the giveaway winner to the owner/admin.

- **Nodes Involved:**  
  - Notify Winner Email

- **Node Details:**  
  - **Node:** Notify Winner Email  
    - **Type:** Email Send  
    - **Role:** Sends an email with winner’s name and giveaway details to notify the admin/owner  
    - **Configuration:**  
      - Recipient: owner@gmail.com (example)  
      - Subject: Dynamic with winner name: "Congratulations! [Winner] is the Winner of the Giveaway!"  
      - HTML Body: Includes winner name and notification message  
      - From Email: admin@gmail.com  
      - SMTP credentials configured (different SMTP account than error notification)  
    - **Input:** Receives winner data from Google Sheets logging node  
    - **Output:** Terminal node of success flow  
    - **Failure Cases:** SMTP misconfiguration or delivery failures  
    - **Sticky Note:** "Sends a congratulatory email announcing the giveaway winner. Includes the winner’s name and relevant giveaway details."

---

### 3. Summary Table

| Node Name                 | Node Type           | Functional Role                                  | Input Node(s)            | Output Node(s)                  | Sticky Note                                                                                           |
|---------------------------|---------------------|-------------------------------------------------|--------------------------|---------------------------------|-----------------------------------------------------------------------------------------------------|
| On form submission        | Form Trigger        | Entry point capturing YouTube URL from form     | —                        | Fetch YouTube Comments          | Triggers the workflow when a form with a YouTube URL is submitted. Captures the URL input for further processing. |
| Fetch YouTube Comments    | HTTP Request        | Calls API to scrape YouTube comments             | On form submission       | Check API Response Status       | Sends a POST request to RapidAPI to scrape YouTube comments from the given URL. Includes API key and necessary headers for authentication. |
| Check API Response Status | If (Conditional)    | Validates API response status code               | Fetch YouTube Comments   | Select Random Commenter, Notify: Invalid API Response | Checks if the API response status code equals 200 (success). Routes workflow based on success or failure of the API call. |
| Select Random Commenter   | Code (JavaScript)   | Extracts authors and selects random winner       | Check API Response Status (true branch) | Log Winner to Google Sheet          | Extracts author names from the fetched comments. Randomly selects one commenter as the giveaway winner. |
| Notify: Invalid API Response | Email Send       | Sends failure notification email to admin       | Check API Response Status (false branch) | —                               | Sends an email notification if the API call fails or returns an invalid response. Alerts the admin to verify URL or API credentials. |
| Log Winner to Google Sheet | Google Sheets      | Appends winner info to a Google Sheet             | Select Random Commenter   | Notify Winner Email             | Appends the winner’s name, URL, and submission date to a Google Sheet. Uses service account authentication for Google Sheets access. |
| Notify Winner Email       | Email Send          | Sends congratulatory email announcing winner     | Log Winner to Google Sheet | —                               | Sends a congratulatory email announcing the giveaway winner. Includes the winner’s name and relevant giveaway details. |
| Sticky Note               | Sticky Note         | Documentation                                      | —                        | —                               | —                                                                                                   |
| Sticky Note1              | Sticky Note         | Documentation                                      | —                        | —                               | —                                                                                                   |
| Sticky Note2              | Sticky Note         | Documentation                                      | —                        | —                               | —                                                                                                   |
| Sticky Note3              | Sticky Note         | Documentation                                      | —                        | —                               | —                                                                                                   |
| Sticky Note4              | Sticky Note         | Documentation                                      | —                        | —                               | —                                                                                                   |
| Sticky Note5              | Sticky Note         | Documentation                                      | —                        | —                               | —                                                                                                   |
| Sticky Note6              | Sticky Note         | Documentation                                      | —                        | —                               | —                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node: "On form submission"**  
   - Type: n8n-nodes-base.formTrigger  
   - Configure:  
     - Form Title: "YouTube Video giveaway"  
     - Description: "YouTube Video giveaway"  
     - Add one required form field:  
       - Field Label: "url"  
       - Required: true  
   - Position: Entry point  
   - No credentials required.

2. **Create HTTP Request Node: "Fetch YouTube Comments"**  
   - Type: n8n-nodes-base.httpRequest  
   - Connect input from: On form submission  
   - Set URL to: https://youtube-comments-scraper.p.rapidapi.com/comment.php  
   - Method: POST  
   - Content Type: multipart/form-data  
   - Body Parameters: Add parameter named "url" with value `={{ $json.url }}`  
   - Headers:  
     - x-rapidapi-host: youtube-comments-scraper.p.rapidapi.com  
     - x-rapidapi-key: [Your RapidAPI key here]  
   - Enable full response to access HTTP status code  
   - Position: after form trigger

3. **Create If Node: "Check API Response Status"**  
   - Type: n8n-nodes-base.if  
   - Connect input from: Fetch YouTube Comments  
   - Condition: Numeric equals  
     - Left value: `={{ $json.statusCode }}`  
     - Right value: `200`  
   - Loose type validation: enabled  
   - Position: after fetch comments

4. **Create Code Node: "Select Random Commenter"**  
   - Type: n8n-nodes-base.code  
   - Connect input from: Check API Response Status (true branch)  
   - Paste JavaScript code:  
     ```javascript
     const comments = $input.first().json.body.items || [];
     const authors = comments
       .map(item => {
         if (item.snippet && item.snippet.topLevelComment && item.snippet.topLevelComment.snippet) {
           return item.snippet.topLevelComment.snippet.authorDisplayName;
         }
         return null;
       })
       .filter(name => !!name);
     if (authors.length === 0) {
       return [{ json: { randomAuthorDisplayName: null } }];
     }
     const randomIndex = Math.floor(Math.random() * authors.length);
     return [{ json: { randomAuthorDisplayName: authors[randomIndex] } }];
     ```
   - Position: after API status check (true)

5. **Create Email Send Node: "Notify: Invalid API Response"**  
   - Type: n8n-nodes-base.emailSend  
   - Connect input from: Check API Response Status (false branch)  
   - Configure email:  
     - To: owner@gmail.com (replace with your admin email)  
     - From: admin@gmail.com (replace with your sender email)  
     - Subject: "YouTube Comments Scraper: API Request Failed"  
     - Body (HTML):  
       ```
       Hello,

       The attempt to fetch comments from the provided YouTube URL did not succeed.
       Please verify the URL and try again.

       If the problem persists, please check the API key or contact support.

       Best regards,
       YouTube Giveaway Bot
       ```
   - Credentials: Configure SMTP credentials with valid account  
   - Position: error notification branch

6. **Create Google Sheets Node: "Log Winner to Google Sheet"**  
   - Type: n8n-nodes-base.googleSheets  
   - Connect input from: Select Random Commenter  
   - Operation: Append  
   - Sheet Name: "Sheet1" (or your target sheet)  
   - Document ID: Set your Google Sheets document ID (must be shared with service account)  
   - Columns to append:  
     - Url: `={{ $('On form submission').item.json.url }}`  
     - Date: `={{ $('On form submission').item.json.submittedAt }}`  
     - Winner: `={{ $json.randomAuthorDisplayName }}`  
   - Authentication: Service Account credentials configured with appropriate scopes  
   - Position: after winner selection

7. **Create Email Send Node: "Notify Winner Email"**  
   - Type: n8n-nodes-base.emailSend  
   - Connect input from: Log Winner to Google Sheet  
   - Configure email:  
     - To: owner@gmail.com (replace with your admin email)  
     - From: admin@gmail.com (replace with your sender email)  
     - Subject: `=Congratulations! {{$json.randomAuthorDisplayName}} is the Winner of the Giveaway!`  
     - Body (HTML):  
       ```
       Hello,

       The YouTube giveaway has concluded, and the winner has been selected!

       Winner: {{$json.randomAuthorDisplayName}}

       The winner's details have been logged in the spreadsheet for your records.

       Best regards,
       Your Giveaway Automation Bot
       ```
   - Credentials: Configure SMTP credentials (can be different from error notification)  
   - Position: final node

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| The RapidAPI YouTube comments scraper requires an API key which must be obtained from RapidAPI platform.             | https://rapidapi.com/                                                                               |
| Google Sheets access requires a service account with delegated permissions and the target spreadsheet must be shared. | https://developers.google.com/identity/protocols/oauth2/service-account                              |
| SMTP credentials need to be configured properly in n8n for email sending nodes to work correctly.                    | n8n documentation on Email node and SMTP setup                                                     |
| This workflow assumes the input form provides a valid YouTube URL; no URL validation is performed internally.         | Consider adding URL validation for robustness                                                      |
| Random selection logic depends on presence of comments; no winner is selected if no valid commenters found.           | Could be improved by adding fallback or retry mechanisms                                           |

---

**Disclaimer:**  
The text provided is extracted exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly respects current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.