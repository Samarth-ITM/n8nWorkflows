Backlink Monitoring Automation with Google Sheets + DataForSEO

https://n8nworkflows.xyz/workflows/backlink-monitoring-automation-with-google-sheets---dataforseo-3685


# Backlink Monitoring Automation with Google Sheets + DataForSEO

### 1. Workflow Overview

This workflow automates backlink monitoring by integrating Google Sheets with the DataForSEO On-Page API. It reads backlink URLs and corresponding landing pages from a Google Sheet, submits each backlink URL for crawling and analysis via DataForSEO, waits for the crawl to complete, fetches backlink status data, evaluates whether backlinks are live and dofollow or lost/nofollow, and finally updates the Google Sheet with the backlink status.

**Target Use Cases:**  
- SEO professionals and digital marketers who want to monitor the health and status of backlinks automatically.  
- Agencies managing multiple backlinks across client websites.  
- Anyone needing to track backlink existence and attributes without manual checking.

**Logical Blocks:**

- **1.1 Trigger and Input Reception:** Manual trigger to start the workflow and reading backlink data from Google Sheets.  
- **1.2 Data Preparation:** Extracting domain information from backlink URLs for API requests.  
- **1.3 DataForSEO API Interaction:** Sending backlink URLs to DataForSEO for crawling, waiting for crawl completion, and fetching backlink link data.  
- **1.4 Backlink Validation:** Analyzing the fetched data to determine backlink status (Live, Lost, Lost (Nofollow)).  
- **1.5 Output Update:** Writing backlink status results back to the Google Sheet.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Input Reception

**Overview:**  
This block initiates the workflow manually and reads backlink URLs and landing pages from a specified Google Sheet range.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Reads Google Sheets

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Starts the workflow on user command.  
  - Configuration: No parameters; triggers workflow execution manually.  
  - Inputs: None  
  - Outputs: Connected to "Reads Google Sheets" node.  
  - Edge Cases: None typical; user must manually trigger.

- **Reads Google Sheets**  
  - Type: Google Sheets node  
  - Role: Reads backlink data from a Google Sheet.  
  - Configuration:  
    - Document ID and Sheet Name set to a specific Google Sheet ("Sheet with lost links", sheet "Lost links").  
    - Data range specified as D1:E (columns expected to include "Backlink URL" and "Landing page").  
  - Inputs: Trigger node output  
  - Outputs: Passes data to "Cleans backlink url" node.  
  - Credentials: Google Sheets OAuth2 credentials configured.  
  - Edge Cases:  
    - Incorrect sheet name or document ID will cause read failure.  
    - Data range must include exact column headers "Backlink URL" and "Landing page" or data extraction will fail.  
    - Empty or malformed rows may cause downstream errors.

---

#### 1.2 Data Preparation

**Overview:**  
Extracts the domain from each backlink URL to prepare for DataForSEO API requests.

**Nodes Involved:**  
- Cleans backlink url  
- Loop Over Items

**Node Details:**

- **Cleans backlink url**  
  - Type: Code (JavaScript)  
  - Role: Parses each backlink URL to extract the domain.  
  - Configuration:  
    - Uses regex to extract domain from the full URL (e.g., from https://www.example.com/page to example.com).  
    - Returns JSON with `domain` and original `url`.  
  - Inputs: Data from "Reads Google Sheets"  
  - Outputs: Passes cleaned data to "Loop Over Items" node.  
  - Edge Cases:  
    - URLs not matching expected pattern may cause regex failure or undefined domain.  
    - Missing or empty URLs will cause errors or empty output.

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Processes backlinks one by one (batch size defaults to 1).  
  - Configuration: Default options, no batch size override.  
  - Inputs: Data from "Cleans backlink url"  
  - Outputs: Each item passed individually to the next node ("Sends HTTP POST Request to DataForSEO").  
  - Edge Cases: Large datasets may slow processing; batching helps manage API rate limits.

---

#### 1.3 DataForSEO API Interaction

**Overview:**  
Sends backlink URLs to DataForSEO for crawling, waits for crawl completion, then fetches backlink link data.

**Nodes Involved:**  
- Sends HTTP POST Request to DataForSEO  
- Waits 20 seconds  
- Sends HTTP links request to DataforSeo

**Node Details:**

- **Sends HTTP POST Request to DataForSEO**  
  - Type: HTTP Request  
  - Role: Posts a crawl task to DataForSEO On-Page API for each backlink domain and URL.  
  - Configuration:  
    - Method: POST  
    - URL: https://api.dataforseo.com/v3/on_page/task_post  
    - JSON Body: Contains `target` (domain), `start_url` (full backlink URL), and `max_crawl_pages` set to 1.  
    - Authentication: HTTP Basic Auth with DataForSEO API credentials.  
  - Inputs: Single backlink item from "Loop Over Items"  
  - Outputs: Passes task response to "Waits 20 seconds" node.  
  - Edge Cases:  
    - Authentication failure if credentials invalid.  
    - API rate limits or downtime may cause errors.  
    - Invalid domain or URL may cause API rejection.

- **Waits 20 seconds**  
  - Type: Wait  
  - Role: Pauses workflow to allow DataForSEO crawl to complete.  
  - Configuration: Wait time set to 20 seconds.  
  - Inputs: Output from POST request node.  
  - Outputs: Passes control to "Sends HTTP links request to DataforSeo".  
  - Edge Cases:  
    - Fixed wait time may be insufficient for slow crawls or excessive for fast responses.  
    - No dynamic polling implemented.

- **Sends HTTP links request to DataforSeo**  
  - Type: HTTP Request  
  - Role: Retrieves backlink link data from DataForSEO using the task ID.  
  - Configuration:  
    - Method: POST  
    - URL: https://api.dataforseo.com/v3/on_page/links  
    - JSON Body: Contains the task ID from previous response (`tasks[0].id`).  
    - Authentication: Same HTTP Basic Auth credentials as previous node.  
    - Batching enabled with batch size 1.  
    - On error: Continue regular output (does not stop workflow on failure).  
  - Inputs: Output from "Waits 20 seconds"  
  - Outputs: Passes backlink link data to "Checks which backlinks exists on the landing page".  
  - Edge Cases:  
    - API errors or empty results may cause missing data.  
    - Task ID missing or invalid causes failure.  
    - Network or timeout errors.

---

#### 1.4 Backlink Validation

**Overview:**  
Analyzes the fetched backlink data to determine if the backlink is live, lost, or lost due to nofollow attribute.

**Nodes Involved:**  
- Checks which backlinks exists on the landing page

**Node Details:**

- **Checks which backlinks exists on the landing page**  
  - Type: Code (JavaScript)  
  - Role: Parses DataForSEO response to find if the backlink exists on the landing page and its dofollow status.  
  - Configuration:  
    - Extracts `tasks[0].result[0].items` array containing links.  
    - Compares each link's `link_to` field to the expected landing page URL from the original Google Sheets data.  
    - Sets status:  
      - "Live" if backlink found and dofollow is true  
      - "Lost (Nofollow)" if backlink found but dofollow is false  
      - "Lost" if backlink not found  
  - Inputs: Data from "Sends HTTP links request to DataforSeo" and original Google Sheets data (via expression referencing "Reads Google Sheets").  
  - Outputs: JSON with backlink URL and status.  
  - Edge Cases:  
    - Missing or malformed API response leads to empty links array.  
    - Landing page URL mismatch causes false negatives.  
    - Expression failures if referenced nodes or fields missing.

---

#### 1.5 Output Update

**Overview:**  
Updates the Google Sheet with the backlink status results, matching rows by Backlink URL.

**Nodes Involved:**  
- Sends data to Google sheets

**Node Details:**

- **Sends data to Google sheets**  
  - Type: Google Sheets node  
  - Role: Appends or updates rows in the Google Sheet with backlink status.  
  - Configuration:  
    - Document ID and Sheet Name same as input node.  
    - Operation: appendOrUpdate  
    - Matching column: "Backlink URL" to find the correct row to update.  
    - Columns mapped:  
      - "Backlink URL" set from current item URL (`{{ $('Loop Over Items').item.json.url }}`)  
      - "Status" set from backlink status (`{{ $json.status }}`)  
  - Inputs: Output from "Checks which backlinks exists on the landing page"  
  - Outputs: Loops back to "Loop Over Items" to process next backlink.  
  - Credentials: Google Sheets OAuth2 credentials configured.  
  - Edge Cases:  
    - Missing or misspelled columns in Google Sheet cause update failure.  
    - Concurrent updates may cause race conditions.  
    - API quota limits or auth errors.

---

### 3. Summary Table

| Node Name                              | Node Type               | Functional Role                         | Input Node(s)                      | Output Node(s)                      | Sticky Note                                                                                                                        |
|--------------------------------------|-------------------------|---------------------------------------|----------------------------------|-----------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’         | Manual Trigger          | Starts the workflow manually           | None                             | Reads Google Sheets                |                                                                                                                                   |
| Reads Google Sheets                   | Google Sheets           | Reads backlink URLs and landing pages | When clicking ‘Test workflow’    | Cleans backlink url               | Connect your Google Sheets account. Ensure your Google Sheet has columns "Backlink URL" and "Landing page". Define data range D1:E. |
| Cleans backlink url                   | Code                    | Extracts domain from backlink URL      | Reads Google Sheets              | Loop Over Items                   |                                                                                                                                   |
| Loop Over Items                      | SplitInBatches          | Processes backlinks one by one         | Cleans backlink url              | Sends HTTP POST Request to DataForSEO, Sends data to Google sheets |                                                                                                                                   |
| Sends HTTP POST Request to DataForSEO | HTTP Request            | Sends crawl task to DataForSEO API    | Loop Over Items                  | Waits 20 seconds                 | Configure DataForSEO TASK POST node with HTTP Basic Auth, POST method, URL https://api.dataforseo.com/v3/on_page/task_post.       |
| Waits 20 seconds                     | Wait                    | Waits for crawl to complete            | Sends HTTP POST Request to DataForSEO | Sends HTTP links request to DataforSeo |                                                                                                                                   |
| Sends HTTP links request to DataforSeo | HTTP Request            | Fetches backlink link data from DataForSEO | Waits 20 seconds                | Checks which backlinks exists on the landing page | Configure DataForSEO ON-PAGE LINKS node with HTTP Basic Auth, POST method, URL https://api.dataforseo.com/v3/on_page/links.        |
| Checks which backlinks exists on the landing page | Code                    | Validates backlink existence and type | Sends HTTP links request to DataforSeo | Sends data to Google sheets      |                                                                                                                                   |
| Sends data to Google sheets          | Google Sheets           | Updates backlink status in Google Sheet | Checks which backlinks exists on the landing page | Loop Over Items                   | Send data to Google Sheets node updates "Status" and "Backlink URL" columns. Matching column: Backlink URL.                      |
| Sticky Note                          | Sticky Note             | Instruction for Google Sheets setup    | None                             | None                            | Connect your Google Sheets account. Ensure your Google Sheet has columns "Backlink URL" and "Landing page". Define data range D1:E. |
| Sticky Note1                        | Sticky Note             | Instruction for DataForSEO TASK POST node | None                             | None                            | Configure DataForSEO TASK POST node with HTTP Basic Auth, POST method, URL https://api.dataforseo.com/v3/on_page/task_post.       |
| Sticky Note2                        | Sticky Note             | Instruction for DataForSEO ON-PAGE LINKS node | None                             | None                            | Configure DataForSEO ON-PAGE LINKS node with HTTP Basic Auth, POST method, URL https://api.dataforseo.com/v3/on_page/links.        |
| Sticky Note3                        | Sticky Note             | Instruction for Google Sheets update node | None                             | None                            | Send data to Google Sheets node updates "Status" and "Backlink URL" columns. Matching column: Backlink URL.                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a "Manual Trigger" node named "When clicking ‘Test workflow’". No configuration needed.

2. **Add Google Sheets Node to Read Data**  
   - Add a "Google Sheets" node named "Reads Google Sheets".  
   - Set operation to "Read Rows".  
   - Configure Document ID to your Google Sheet containing backlinks.  
   - Set Sheet Name to the appropriate sheet (e.g., "Lost links").  
   - Specify data range as "D1:E" (or your columns containing "Backlink URL" and "Landing page").  
   - Use OAuth2 credentials for Google Sheets.  
   - Connect "When clicking ‘Test workflow’" output to this node.

3. **Add Code Node to Extract Domain**  
   - Add a "Code" node named "Cleans backlink url".  
   - Use JavaScript code to extract domain from "Backlink URL":  
     ```js
     return items.map(item => {
       let url = item.json['Backlink URL'];
       let domain = url.match(/https?:\/\/(?:www\.)?([^/]+)/)[1];
       return { json: { domain, url } };
     });
     ```  
   - Connect output of "Reads Google Sheets" to this node.

4. **Add SplitInBatches Node**  
   - Add a "SplitInBatches" node named "Loop Over Items".  
   - Default batch size (1) is fine.  
   - Connect output of "Cleans backlink url" to this node.

5. **Add HTTP Request Node to Post Crawl Task**  
   - Add an "HTTP Request" node named "Sends HTTP POST Request to DataForSEO".  
   - Set method to POST.  
   - URL: https://api.dataforseo.com/v3/on_page/task_post  
   - Authentication: HTTP Basic Auth with your DataForSEO API key and password.  
   - Body Content Type: JSON.  
   - JSON Body:  
     ```json
     [{
       "target": "{{ $json.domain }}",
       "start_url": "{{ $json.url }}",
       "max_crawl_pages": 1
     }]
     ```  
   - Connect output of "Loop Over Items" to this node.

6. **Add Wait Node**  
   - Add a "Wait" node named "Waits 20 seconds".  
   - Set wait time to 20 seconds.  
   - Connect output of "Sends HTTP POST Request to DataForSEO" to this node.

7. **Add HTTP Request Node to Fetch Links**  
   - Add an "HTTP Request" node named "Sends HTTP links request to DataforSeo".  
   - Set method to POST.  
   - URL: https://api.dataforseo.com/v3/on_page/links  
   - Authentication: Same HTTP Basic Auth credentials as previous DataForSEO node.  
   - Body Content Type: JSON.  
   - JSON Body:  
     ```json
     [
       {
         "id": "{{ $json.tasks[0].id }}"
       }
     ]
     ```  
   - Enable batching with batch size 1.  
   - Set "On Error" to "Continue" to avoid workflow stop on API errors.  
   - Connect output of "Waits 20 seconds" to this node.

8. **Add Code Node to Validate Backlink**  
   - Add a "Code" node named "Checks which backlinks exists on the landing page".  
   - Use JavaScript code:  
     ```js
     const result = $json.tasks?.[0]?.result?.[0];
     const links = result?.items || [];
     let backlink = $('Reads Google Sheets').item.json['Landing page'];
     let foundLink = links.find(link => link.link_to === backlink);
     let status = "Lost";
     if (foundLink) {
       status = foundLink.dofollow ? "Live" : "Lost (Nofollow)";
     }
     return { json: { backlink, status } };
     ```  
   - Connect output of "Sends HTTP links request to DataforSeo" to this node.

9. **Add Google Sheets Node to Update Status**  
   - Add a "Google Sheets" node named "Sends data to Google sheets".  
   - Set operation to "Append or Update".  
   - Use the same Document ID and Sheet Name as the reading node.  
   - Set Matching Column to "Backlink URL".  
   - Map columns:  
     - "Backlink URL": `{{ $('Loop Over Items').item.json.url }}`  
     - "Status": `{{ $json.status }}`  
   - Use OAuth2 credentials for Google Sheets.  
   - Connect output of "Checks which backlinks exists on the landing page" to this node.

10. **Loop Back to Process Next Item**  
    - Connect output of "Sends data to Google sheets" back to "Loop Over Items" to continue processing remaining backlinks.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                      | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Connect your Google Sheets account. Ensure your Google Sheet has clearly defined columns: "Backlink URL" and "Landing page". Define your data range explicitly (e.g., D1:E). The columns must be named exactly as specified.       | Sticky Note near "Reads Google Sheets" node                                                    |
| Configure your DataForSEO TASK POST node with HTTP Basic Authentication. Insert your API key and password into n8n's Credentials settings. This node sends each URL/domain pair to the DataForSEO On-Page API for analysis.          | Sticky Note near "Sends HTTP POST Request to DataForSEO" node                                  |
| Configure your DataForSEO ON-PAGE LINKS node with HTTP Basic Authentication. This node fetches backlink data, checking if the backlink exists and its status (dofollow/nofollow).                                                | Sticky Note near "Sends HTTP links request to DataforSeo" node                                 |
| Send data to Google Sheets node updates "Status" and "Backlink URL" columns. Matching column: Backlink URL. Make sure these columns exist and are spelled exactly to ensure correct updates.                                     | Sticky Note near "Sends data to Google sheets" node                                           |
| Your result will look like a Google Sheet with backlink URLs, landing pages, and a Status column showing "Live", "Lost", or "Lost (Nofollow)".                                                                                  | Workflow description and screenshot reference                                                 |
| Step-by-step setup instructions: Add DataForSEO and Google Sheets credentials, ensure Google Sheet columns exist, then run the workflow manually to test backlink batches.                                                        | Workflow description                                                                          |

---

This documentation provides a complete, structured understanding of the "Backlink Monitoring Automation with Google Sheets + DataForSEO" workflow, enabling reproduction, modification, and troubleshooting.