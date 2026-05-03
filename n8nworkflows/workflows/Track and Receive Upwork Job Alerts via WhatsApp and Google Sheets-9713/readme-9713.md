Track and Receive Upwork Job Alerts via WhatsApp and Google Sheets

https://n8nworkflows.xyz/workflows/track-and-receive-upwork-job-alerts-via-whatsapp-and-google-sheets-9713


# Track and Receive Upwork Job Alerts via WhatsApp and Google Sheets

---

### 1. Workflow Overview

This workflow automates the process of searching for relevant Upwork job listings based on user-defined criteria, storing the results in a Google Sheet, filtering jobs by a quality score threshold, and sending alerts via WhatsApp. It is designed for freelancers or agencies who want to track Upwork job opportunities efficiently and receive instant notifications on promising jobs.

The workflow is logically divided into these blocks:

- **1.1 Triggering Block:** Receives input either via a webhook (API call) or manual trigger, and includes a scheduled trigger (though currently inactive in flow) to start the job search process.
- **1.2 Upwork Job Search Block:** Uses an HTTP POST request to a RapidAPI Upwork scraping API to retrieve job listings matching criteria.
- **1.3 Data Storage and Preparation Block:** Appends or updates job data into a Google Sheet, then reformats selected fields and prepares data for filtering.
- **1.4 Filtering Block:** Filters jobs based on a minimum score threshold to identify relevant opportunities.
- **1.5 Alert and Response Block:** Sends WhatsApp alerts for filtered jobs and responds to the webhook caller.

---

### 2. Block-by-Block Analysis

#### 2.1 Triggering Block

- **Overview:** This block initiates the workflow. It supports manual execution and receives job search parameters via an HTTP webhook POST request. There is also a scheduled trigger node present but not connected in the main flow.
- **Nodes Involved:**  
  - Schedule Trigger  
  - Manual Trigger ("When clicking ‘Execute workflow’")  
  - Webhook

- **Node Details:**

  - **Schedule Trigger**  
    - Type: `Schedule Trigger`  
    - Role: Intended to periodically trigger the workflow (e.g., hourly/daily)  
    - Configuration: Default interval (not further specified)  
    - Input: None  
    - Output: None connected in this workflow  
    - Notes: Currently not wired to downstream nodes, so inactive in main flow  
    - Potential Failures: Scheduling misconfiguration, node version compatibility

  - **Manual Trigger ("When clicking ‘Execute workflow’")**  
    - Type: `Manual Trigger`  
    - Role: Allows manual start of the workflow for testing or immediate execution  
    - Configuration: No parameters  
    - Input: None  
    - Output: None connected downstream  
    - Potential Failures: None significant; manual trigger

  - **Webhook**  
    - Type: `Webhook`  
    - Role: Receives external POST requests containing Upwork job search parameters  
    - Configuration:  
      - HTTP Method: POST  
      - Unique Path: `7f69e1de-e644-40a7-bb0b-288b096b2696`  
      - Response Mode: Uses a response node downstream  
    - Input: External HTTP POST with JSON body containing searchQuery, locations, minRate, maxRate  
    - Output: Triggers HTTP Request node  
    - Potential Failures: Invalid input data, missing fields, webhook authorization  
    - Notes: This is the main entry point for automated searches.

---

#### 2.2 Upwork Job Search Block

- **Overview:** Sends a POST request to a RapidAPI Upwork scraping API with search filters to retrieve job listings from Upwork.
- **Nodes Involved:**  
  - HTTP Request

- **Node Details:**

  - **HTTP Request**  
    - Type: `HTTP Request`  
    - Role: Query Upwork job listings via RapidAPI  
    - Configuration:  
      - URL: `https://upwork-scraping-api.p.rapidapi.com/upwork/search-jobs`  
      - Method: POST  
      - Headers:  
        - `x-rapidapi-host`: `upwork-scraping-api.p.rapidapi.com`  
        - `x-rapidapi-key`: Requires user API key (replace `"Your API key"`)  
      - Body Parameters (JSON): Includes query parameters such as  
        - `query` from webhook input `body.searchQuery`  
        - `type` fixed as hourly or fixed  
        - `difficulty` fixed as entry, intermediate, expert  
        - `hours_per_week` fixed as less_than_30 or more_than_30  
        - `client_hires` fixed as 0, 1-9, 10+  
        - `client_location` from webhook input `body.locations[0]`  
        - `min_hourly_rate` and `max_hourly_rate` from webhook inputs  
    - Input: JSON from Webhook node  
    - Output: Job listings JSON to Google Sheets node  
    - Potential Failures:  
      - API key missing or invalid (authorization errors)  
      - Network timeouts or rate limiting by RapidAPI  
      - Invalid input parameters causing API errors

- **Sticky Note:** "Rapid Api For Scraping" — indicates this node’s role as the job search engine.

---

#### 2.3 Data Storage and Preparation Block

- **Overview:** Saves or updates job listings data into a Google Sheet and restructures key fields for filtering and alerting.
- **Nodes Involved:**  
  - Append or update row in sheet  
  - Edit Fields  
  - If (Filter)  
  - Edit Fields2

- **Node Details:**

  - **Append or update row in sheet**  
    - Type: `Google Sheets`  
    - Role: Append new job records or update existing ones by matching on the "Title" column  
    - Configuration:  
      - Operation: Append or Update  
      - Document ID: Google Sheet ID `1_WNd8Pex9BkDj8HgyqceyBCyo5G6_a9_oscipZrWjWQ`  
      - Sheet Name: `Sheet1` (gid=0)  
      - Columns: Multiple fields including URL, Date, Found, Score, Title, Salary, Status, Message, Location, Experience, Description, Payment Type  
      - Matching Column: Title (to avoid duplicate job entries)  
      - Credentials: Google Sheets OAuth2 (user must configure with proper permissions)  
    - Input: Job list from HTTP Request node  
    - Output: Passes updated JSON to Edit Fields node  
    - Potential Failures:  
      - Google Sheets API errors (permission denied, quota exceeded)  
      - Mismatched schema or data type issues

  - **Edit Fields**  
    - Type: `Set`  
    - Role: Extracts and renames key job info fields for filtering  
    - Configuration:  
      - Creates new fields: "Job Tile" (from Title), "job posted" (from Date), "Payment type", "score", "budget" (from Salary), "url"  
    - Input: From Google Sheets node  
    - Output: To If node for filtering  
    - Potential Failures:  
      - Expression errors if input JSON fields missing or malformed

  - **If**  
    - Type: `If`  
    - Role: Filters jobs based on minimum score threshold of 70  
    - Configuration:  
      - Condition: `$json.score >= 70`  
    - Input: From Edit Fields node  
    - Output: True branch to Edit Fields2 node; false branch discarded  
    - Potential Failures:  
      - Type mismatch if score field missing or not numeric

  - **Edit Fields2**  
    - Type: `Set`  
    - Role: Prepares final fields for alert message  
    - Configuration:  
      - Sets fields: job, job posted, payment type, salary, score, url (some sourced from previous Edit Fields node)  
    - Input: True output from If node  
    - Output: To Respond to Webhook, Send Message, and HTTP Request1 nodes  
    - Potential Failures:  
      - Expression errors if referenced fields missing

- **Sticky Notes:**  
  - "Save Results in Sheet" near Google Sheets node  
  - "Edit Field and the If Filter to pass the specific Information" near Edit Fields and If nodes

---

#### 2.4 Alert and Response Block

- **Overview:** Sends notification messages with job details via WhatsApp and responds to the webhook caller.
- **Nodes Involved:**  
  - Respond to Webhook  
  - Send message (WhatsApp) [disabled]  
  - HTTP Request1 (WhatsApp alert via RapidAPI)

- **Node Details:**

  - **Respond to Webhook**  
    - Type: `Respond to Webhook`  
    - Role: Sends HTTP response back to webhook caller confirming job alert processing  
    - Configuration: Default response, no custom payload specified  
    - Input: From Edit Fields2 node  
    - Output: None  
    - Potential Failures: Network errors, webhook timeout

  - **Send message** (WhatsApp, disabled)  
    - Type: `WhatsApp`  
    - Role: Intended to send WhatsApp message directly via WhatsApp API node  
    - Configuration:  
      - Operation: send  
      - Text Body: "hy" (placeholder)  
      - Recipient Phone Number: empty (not configured)  
      - Credentials: WhatsApp API account  
    - Input: From Edit Fields2 node (but node disabled)  
    - Output: None  
    - Potential Failures: Not active; if enabled, requires valid phone number and WhatsApp credentials

  - **HTTP Request1**  
    - Type: `HTTP Request`  
    - Role: Sends WhatsApp alert via RapidAPI service (whin2.p.rapidapi.com)  
    - Configuration:  
      - URL: `https://whin2.p.rapidapi.com/send`  
      - Method: POST  
      - Headers:  
        - `x-rapidapi-host`: `whin2.p.rapidapi.com`  
        - `x-rapidapi-key`: User must supply API key (replace `"Your API Key"`)  
      - Body Parameters:  
        - `text`: Formatted message including job title, posted date, budget, score, and URL with template expressions  
    - Input: From Edit Fields2 node  
    - Output: None  
    - Potential Failures:  
      - API key missing or invalid  
      - Message formatting issues  
      - Rate limiting or network errors

- **Sticky Note:** "Send the Alert" covering this block

---

### 3. Summary Table

| Node Name                  | Node Type                 | Functional Role                          | Input Node(s)             | Output Node(s)                                          | Sticky Note                              |
|----------------------------|---------------------------|----------------------------------------|---------------------------|---------------------------------------------------------|----------------------------------------|
| Schedule Trigger           | Schedule Trigger          | Periodic workflow trigger (inactive)  | None                      | None                                                    | # Triggers to start the wokdlow        |
| When clicking ‘Execute workflow’ | Manual Trigger            | Manual start of workflow                | None                      | None                                                    | # Triggers to start the wokdlow        |
| Webhook                   | Webhook                   | Receives external job search requests | None                      | HTTP Request                                            | # Triggers to start the wokdlow        |
| HTTP Request              | HTTP Request              | Queries Upwork job listings API        | Webhook                   | Append or update row in sheet                           | ## Rapid Api For Scraping               |
| Append or update row in sheet | Google Sheets             | Saves/updates job data in Google Sheet | HTTP Request              | Edit Fields                                            | ## Save Results in Sheet                |
| Edit Fields               | Set                       | Extracts and renames key job fields    | Append or update row in sheet | If                                                    | ## Edit Field and the If Filter to pass the specific Information |
| If                        | If                        | Filters jobs by score >= 70             | Edit Fields               | Edit Fields2                                           | ## Edit Field and the If Filter to pass the specific Information |
| Edit Fields2              | Set                       | Prepares final fields for alert message | If                       | Respond to Webhook, Send message, HTTP Request1        | ## Edit Field and the If Filter to pass the specific Information |
| Respond to Webhook        | Respond to Webhook        | Sends response back to webhook caller  | Edit Fields2              | None                                                   | ## Send the Alert                      |
| Send message              | WhatsApp                  | Sends WhatsApp alert (disabled)         | Edit Fields2              | None                                                   | ## Send the Alert                      |
| HTTP Request1             | HTTP Request              | Sends WhatsApp alert via RapidAPI       | Edit Fields2              | None                                                   | ## Send the Alert                      |
| Sticky Note               | Sticky Note               | Title and section notes                  | None                      | None                                                   | # Upwork Job Scraper with Whatsapp Alert |
| Sticky Note1              | Sticky Note               | Notes on workflow triggers               | None                      | None                                                   | # Triggers to start the wokdlow        |
| Sticky Note2              | Sticky Note               | RapidAPI scraping info                   | None                      | None                                                   | ## Rapid Api For Scraping               |
| Sticky Note3              | Sticky Note               | Google Sheets storage note                | None                      | None                                                   | ## Save Results in Sheet                |
| Sticky Note4              | Sticky Note               | Edit fields and If filter note           | None                      | None                                                   | ## Edit Field and the If Filter to pass the specific Information |
| Sticky Note5              | Sticky Note               | Alert sending note                        | None                      | None                                                   | ## Send the Alert                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**
   - Add a **Webhook** node:
     - Set HTTP Method to POST
     - Set unique path (e.g., `7f69e1de-e644-40a7-bb0b-288b096b2696`)
     - Response Mode: `Response Node`
   - Add a **Manual Trigger** node named "When clicking ‘Execute workflow’"
   - (Optional) Add a **Schedule Trigger** node for periodic execution (configure interval as needed)

2. **Add Upwork Job Search HTTP Request:**
   - Add an **HTTP Request** node after the Webhook node
   - Configure Method: POST
   - URL: `https://upwork-scraping-api.p.rapidapi.com/upwork/search-jobs`
   - Headers:
     - `x-rapidapi-host`: `upwork-scraping-api.p.rapidapi.com`
     - `x-rapidapi-key`: Your RapidAPI key for Upwork scraping API
   - Body Parameters (JSON):
     - `query`: `={{ $json.body.searchQuery }}`
     - `type`: `hourly, fixed`
     - `difficulty`: `entry, intermediate, expert`
     - `hours_per_week`: `less_than_30, more_than_30`
     - `client_hires`: `0, 1-9, 10+`
     - `client_location`: `={{ $json.body.locations[0] }}`
     - `min_hourly_rate`: `={{ $json.body.minRate }}`
     - `max_hourly_rate`: `={{ $json.body.maxRate }}`
   - Connect Webhook output to this node

3. **Add Google Sheets Append or Update Node:**
   - Add **Google Sheets** node named "Append or update row in sheet"
   - Operation: Append or Update
   - Document ID: Your Google Sheet ID (e.g., `1_WNd8Pex9BkDj8HgyqceyBCyo5G6_a9_oscipZrWjWQ`)
   - Sheet name: `Sheet1`
   - Columns to map: Title, Description, Status, Payment Type, Experience, Salary, Location, Message, Score, URL, Date, Found
   - Matching Column: Title
   - Set up Google Sheets OAuth2 credentials with edit permissions
   - Connect HTTP Request output to this node

4. **Add "Edit Fields" Set Node:**
   - Add a **Set** node named "Edit Fields"
   - Assignments:
     - Job Tile: `={{ $json.Title }}`
     - job posted: `={{ $json.Date }}`
     - Payment type: `={{ $json['Payment Type'] }}`
     - score: `={{ $json.Score }}`
     - budget: `={{ $json.Salary }}`
     - url: `={{ $json.URL }}`
   - Connect Google Sheets node output to this node

5. **Add "If" Node for Filtering:**
   - Add an **If** node
   - Condition: Numeric comparison  
     - Left: `={{ $json.score }}`  
     - Operator: Greater than or equal (`>=`)  
     - Right: `70`
   - Connect "Edit Fields" node output to this node

6. **Add "Edit Fields2" Set Node:**
   - Add a **Set** node named "Edit Fields2"
   - Assignments:
     - job: `={{ $json['Job Tile'] }}`
     - job posted: `={{ $json['job posted'] }}`
     - payment type: `={{ $json['Payment type'] }}`
     - salary: `={{ $('Edit Fields').item.json.budget }}`
     - score: `={{ $json.score }}`
     - url: `={{ $json.url }}`
   - Connect "If" node’s True output to this node

7. **Add Respond to Webhook Node:**
   - Add a **Respond to Webhook** node
   - Connect "Edit Fields2" node output to this node

8. **Add WhatsApp Alert HTTP Request:**
   - Add an **HTTP Request** node named "HTTP Request1"
   - Method: POST
   - URL: `https://whin2.p.rapidapi.com/send`
   - Headers:
     - `x-rapidapi-host`: `whin2.p.rapidapi.com`
     - `x-rapidapi-key`: Your RapidAPI key for WhatsApp sending
   - Body Parameters:
     - `text`:  
       ```
       New Job Alert make the Proposal ASAP

       Job Title : {{ $json.job }}
       Posted On : {{ $json['job posted'] }}
       Budget : {{ $json.salary }}
       Job Score : {{ $json.score }}
       Job URL: {{ $json.url }}
       ```
   - Connect "Edit Fields2" node output to this node

9. **(Optional) Add WhatsApp Node:**
   - Add **WhatsApp** node named "Send message"
   - Operation: Send  
   - Text Body: Customize as desired  
   - Recipient Phone Number: Configure actual recipient  
   - Configure WhatsApp API credentials  
   - Currently disabled in original workflow; enable if needed  
   - Connect "Edit Fields2" node output to this node

10. **Connect and Test:**
    - Ensure all connections as above  
    - Test webhook by sending POST request with JSON body including:  
      - `searchQuery`: string  
      - `locations`: array of strings  
      - `minRate`: number  
      - `maxRate`: number  
    - Validate Google Sheets updates and WhatsApp alerts

---

### 5. General Notes & Resources

| Note Content                                                        | Context or Link                               |
|-------------------------------------------------------------------|-----------------------------------------------|
| # Upwork Job Scraper with Whatsapp Alert                          | Workflow title and main purpose                |
| # Triggers to start the wokdlow                                   | Used for Schedule Trigger, Manual Trigger, Webhook |
| ## Rapid Api For Scraping                                         | Reference to Upwork scraping API from RapidAPI |
| ## Save Results in Sheet                                         | Indicates usage of Google Sheets for persistence |
| ## Edit Field and the If Filter to pass the specific Information | Notes near field editing and filtering nodes    |
| ## Send the Alert                                                | Notes on sending WhatsApp alerts                |
| RapidAPI Upwork Scraping API documentation: https://rapidapi.com/user/upwork-scraping-api | For API usage and key registration               |
| WhatsApp API via RapidAPI (whin2): https://rapidapi.com/whin2-api | For WhatsApp messaging API                      |

---

**Disclaimer:**  
The text provided is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled are legal and public.

---