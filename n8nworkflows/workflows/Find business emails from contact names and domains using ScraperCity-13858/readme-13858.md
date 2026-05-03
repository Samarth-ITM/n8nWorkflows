Find business emails from contact names and domains using ScraperCity

https://n8nworkflows.xyz/workflows/find-business-emails-from-contact-names-and-domains-using-scrapercity-13858


# Find business emails from contact names and domains using ScraperCity

# 1. Workflow Overview

This workflow automates the process of finding professional email addresses using the ScraperCity API. It is designed for sales development representatives, recruiters, and growth marketers who have a list of contact names and company domains and need to enrich them with verified email addresses at scale.

The process operates asynchronously: it submits a batch of contacts, polls the ScraperCity server until the background job is finished, downloads the enriched data, and finally appends the successful results into a Google Sheet.

The workflow is organized into four functional blocks:
1.  **Contact Configuration:** Defining the input list of names and domains.
2.  **Job Submission:** Sending the data to ScraperCity and capturing the tracking ID.
3.  **Async Polling Loop:** Checking the job status repeatedly until the processing is complete.
4.  **Data Processing & Export:** Normalizing the API response and writing found emails to a spreadsheet.

---

# 2. Block-by-Block Analysis

### 2.1 Contact Configuration
This block initializes the workflow with the data intended for enrichment.

*   **Nodes Involved:** `When clicking 'Execute workflow'`, `Set Contact List`.
*   **Node Details:**
    *   **When clicking 'Execute workflow' (Manual Trigger):** Starts the process manually.
    *   **Set Contact List (Set):** 
        *   **Role:** Defines the input data.
        *   **Configuration:** Assigns a string variable `contacts` containing a JSON array of objects (e.g., `[{"first_name":"Jane", "last_name":"Smith", "domain":"example.com"}]`).
        *   **Edge Cases:** Malformed JSON strings here will cause the subsequent HTTP request to fail.

### 2.2 Job Submission
This block handles the initial communication with the ScraperCity API.

*   **Nodes Involved:** `Submit Email Finder Job`, `Store Run ID`.
*   **Node Details:**
    *   **Submit Email Finder Job (HTTP Request):** 
        *   **Role:** Submits the contact list to the `/api/v1/scrape/email-finder` endpoint via POST.
        *   **Configuration:** Uses Header Authentication. The body sends the `contacts` variable from the previous node.
        *   **Failure Types:** 401 Unauthorized (invalid API key) or 400 Bad Request (invalid contact format).
    *   **Store Run ID (Set):**
        *   **Role:** Extracts the `runId` from the API response.
        *   **Configuration:** Maps `{{ $json.runId }}` to a local variable for easy reference in polling nodes.

### 2.3 Async Polling Loop
Because email finding is an intensive process, the API requires a polling mechanism to determine when results are ready.

*   **Nodes Involved:** `Wait Before First Poll`, `Polling Loop`, `Check Job Status`, `Is Job Complete?`, `Wait 60 Seconds Before Retry`.
*   **Node Details:**
    *   **Wait Before First Poll (Wait):** Pauses for 30 seconds to give the API time to start processing.
    *   **Polling Loop (Split In Batches):** Used as a control gate for the loop. Set to batch size 1 to process the single `runId`.
    *   **Check Job Status (HTTP Request):** Performs a GET request to `/api/v1/scrape/status/{{runId}}`.
    *   **Is Job Complete? (If):** Checks if the `status` field in the response equals `SUCCEEDED`.
    *   **Wait 60 Seconds Before Retry (Wait):** If the status is not `SUCCEEDED`, it waits one minute before routing back to the `Polling Loop` node.

### 2.4 Data Processing & Export
Once the job is finished, this block retrieves, cleans, and saves the data.

*   **Nodes Involved:** `Download Email Results`, `Parse and Format Results`, `Filter Emails Found`, `Write Results to Google Sheets`.
*   **Node Details:**
    *   **Download Email Results (HTTP Request):** Fetches the final JSON payload from `/api/downloads/{{runId}}`.
    *   **Parse and Format Results (Code):**
        *   **Role:** Normalizes the data structure.
        *   **Logic:** It iterates through the results and ensures fields like `email`, `email_status`, and `confidence` are present and flat, making them compatible with spreadsheet rows.
    *   **Filter Emails Found (Filter):** Removes any rows where the `email` field is empty to ensure only successful matches are saved.
    *   **Write Results to Google Sheets (Google Sheets):** 
        *   **Role:** Appends the cleaned data to a specific spreadsheet.
        *   **Configuration:** Maps internal JSON keys to Sheet column headers (e.g., `first_name`, `last_name`, `email`).

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| When clicking 'Execute workflow' | manualTrigger | Workflow Entry | (None) | Set Contact List | (None) |
| Set Contact List | set | Data Input | When clicking 'Execute workflow' | Submit Email Finder Job | The `Set Contact List` node is where you define the contacts to look up. |
| Submit Email Finder Job | httpRequest | API Submission | Set Contact List | Store Run ID | Add your ScraperCity API key credential here |
| Store Run ID | set | ID Extraction | Submit Email Finder Job | Wait Before First Poll | `Store Run ID` captures this ID and makes it available throughout the rest of the workflow. |
| Wait Before First Poll | wait | Initial Delay | Store Run ID | Polling Loop | Wait 30 seconds on the first pass, then enters a loop. |
| Polling Loop | splitInBatches | Loop Controller | Wait Before First Poll, Wait 60 Seconds Before Retry | Check Job Status | ScraperCity email-finder jobs run in the background. |
| Check Job Status | httpRequest | Status Check | Polling Loop | Is Job Complete? | Polls the status endpoint via GET. |
| Is Job Complete? | if | Logic Switch | Check Job Status | Download Email Results (True), Wait 60 Seconds Before Retry (False) | Checks whether status equals `SUCCEEDED`. |
| Wait 60 Seconds Before Retry | wait | Retry Delay | Is Job Complete? | Polling Loop | If not done, `Wait 60 Seconds Before Retry` pauses before looping back. |
| Download Email Results | httpRequest | Data Retrieval | Is Job Complete? | Parse and Format Results | `Download Email Results` fetches the completed results from ScraperCity. |
| Parse and Format Results | code | Data Cleaning | Download Email Results | Filter Emails Found | Normalizes each contact row into flat fields. |
| Filter Emails Found | filter | Data Filtering | Parse and Format Results | Write Results to Google Sheets | Removes contacts where no email was returned. |
| Write Results to Google Sheets | googleSheets | Data Export | Filter Emails Found | (None) | Set your Google Sheets document ID and sheet name here |

---

# 4. Reproducing the Workflow from Scratch

1.  **Setup Credentials:**
    *   Create a **Header Auth** credential: Name it `ScraperCity API Key`, Header: `Authorization`, Value: `Bearer [YOUR_API_KEY]`.
    *   Create a **Google Sheets OAuth2** credential for your Google account.

2.  **Initial Logic:**
    *   Add a **Manual Trigger** node.
    *   Connect a **Set** node (`Set Contact List`). Create a string variable `contacts` and paste a JSON array of contacts with `first_name`, `last_name`, and `domain`.

3.  **API Submission:**
    *   Add an **HTTP Request** node (`Submit Email Finder Job`).
        *   Method: `POST`.
        *   URL: `https://scrapercity.com/api/v1/scrape/email-finder`.
        *   Authentication: `Header Auth`.
        *   Body: `JSON`, `contacts` field linked to the previous node.
    *   Add a **Set** node (`Store Run ID`). Assign `runId` from the expression `{{ $json.runId }}`.

4.  **Polling Mechanism:**
    *   Add a **Wait** node (`Wait Before First Poll`) set to 30 seconds.
    *   Add a **Split In Batches** node (`Polling Loop`) with batch size 1.
    *   Add an **HTTP Request** node (`Check Job Status`).
        *   Method: `GET`.
        *   URL: `https://scrapercity.com/api/v1/scrape/status/{{ $node["Store Run ID"].json["runId"] }}`.
    *   Add an **If** node (`Is Job Complete?`). Condition: String `{{ $json.status }}` equals `SUCCEEDED`.
    *   Add a **Wait** node (`Wait 60 Seconds Before Retry`) set to 60 seconds. Connect it from the **False** output of the If node and loop it back to the **Polling Loop** node.

5.  **Data Extraction:**
    *   Add an **HTTP Request** node (`Download Email Results`) to the **True** output of the If node.
        *   Method: `GET`.
        *   URL: `https://scrapercity.com/api/downloads/{{ $node["Store Run ID"].json["runId"] }}`.
    *   Add a **Code** node (`Parse and Format Results`). Use JavaScript to map the array items into flat JSON objects containing `first_name`, `last_name`, `domain`, `email`, `email_status`, and `confidence`.
    *   Add a **Filter** node (`Filter Emails Found`). Rule: `email` is not empty.

6.  **Final Export:**
    *   Add a **Google Sheets** node. 
        *   Operation: `Append`.
        *   Document/Sheet: Select your target sheet.
        *   Mapping: Map the fields from the Filter node to the sheet columns.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Typical Completion Time** | Jobs usually take 1-10 minutes depending on list size. |
| **API Documentation** | Refer to ScraperCity API docs for rate limits or plan details. |
| **Data Normalization** | The Code node ensures that even if no results are found, the workflow doesn't crash but returns a "no_results" status. |
| **Sheet Preparation** | Ensure your Google Sheet has headers: first_name, last_name, domain, email, email_status, confidence. |