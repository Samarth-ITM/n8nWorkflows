Automatic monitoring of multiple URLs with downtime alerts

https://n8nworkflows.xyz/workflows/automatic-monitoring-of-multiple-urls-with-downtime-alerts-5298


# Automatic monitoring of multiple URLs with downtime alerts

### 1. Workflow Overview

This workflow automates the monitoring of multiple URLs by periodically checking their availability and recording the status in a Google Sheet. It identifies URLs that are down ("caída") and those that are functional, compiles a summary, and sends an email alert listing the URLs currently down along with statistics.

Logical blocks and their roles:

- **1.1 Input Reception & Scheduling**: Defines the list of URLs to monitor and triggers the workflow on schedule or manually.
- **1.2 URL Loop & Status Check**: Iterates over each URL, performs HTTP requests to check availability, and categorizes results as Success or Error.
- **1.3 Data Logging**: Appends the status of each URL (functional or down) to a Google Sheet for record-keeping.
- **1.4 Summary & Alert Preparation**: Summarizes the results, extracts down URLs, and prepares a notification message.
- **1.5 Notification Sending**: Sends an email alert with the list of down URLs and summary statistics.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Scheduling

- **Overview:**  
  This block initializes the workflow with a predefined set of URLs and triggers the monitoring process either manually or on a schedule.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Run trigger  
  - URLs  
  - Sticky Note5 (comment)  
  - Sticky Note4 (comment)  
  - Sticky Note (comment)  
  - Sticky Note1 (comment)  

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger node  
    - Role: Automatically triggers the workflow at regular intervals (default is every minute, configured with empty interval object meaning default).  
    - Configuration: Uses default schedule settings without specific interval customization.  
    - Input: None (trigger)  
    - Output: Starts the workflow, output connected to "URLs" node.  
    - Edge cases: Incorrect schedule settings could cause unexpected execution frequency.  
    - Comments: Sticky Note5 suggests alternative triggers like webhook or MCP could be used.

  - **Run trigger**  
    - Type: Manual Trigger node  
    - Role: Allows manual initiation for testing or immediate execution.  
    - Configuration: Default manual trigger.  
    - Input: None (trigger)  
    - Output: Starts the workflow, output connected to "URLs" node.  
    - Edge cases: None significant.

  - **URLs**  
    - Type: Set node  
    - Role: Defines the URLs to monitor as a JSON object with named keys.  
    - Configuration: Raw JSON mode with a dictionary of URLs keyed as "URL 1", "URL 2", etc.  
    - Key expressions: None dynamic; static URL list is hardcoded.  
    - Input: Trigger nodes ("Schedule Trigger" or "Run trigger").  
    - Output: JSON object with "urls" field passed to "Split Out" node.  
    - Edge cases: If URLs are malformed or unreachable, later steps handle errors.

  - **Sticky Notes (5, 4, and others)**  
    - Type: Sticky Note nodes  
    - Role: Provide user instructions and context.  
    - Content:  
      - Sticky Note5: Suggests alternative triggers such as webhook or MCP for workflow initiation.  
      - Sticky Note4: Marks where URLs should be entered.  
      - Sticky Note & Sticky Note1: Explain that URLs are entered before looping begins and the workflow is triggered manually or by schedule.

---

#### 2.2 URL Loop & Status Check

- **Overview:**  
  This block splits the URLs set into individual entries, iterates over each URL, performs HTTP requests to check each URL’s availability, and branches based on success or failure.

- **Nodes Involved:**  
  - Split Out  
  - Bucle URLs (SplitInBatches)  
  - Request (HTTP Request)  
  - Success (Google Sheets append)  
  - Error (Google Sheets append)  
  - Sticky Note2 (comment)  
  - Sticky Note3 (comment)  

- **Node Details:**

  - **Split Out**  
    - Type: Split Out node  
    - Role: Splits the "urls" object from "URLs" node into individual items (key-value pairs).  
    - Configuration: Splits on field "urls".  
    - Input: JSON with "urls" object.  
    - Output: One item per URL with key and URL.  
    - Edge cases: Empty or malformed "urls" input results in zero outputs.

  - **Bucle URLs**  
    - Type: SplitInBatches node  
    - Role: Processes URLs in batches (default batch size since none specified).  
    - Configuration: Default options, no batch size defined (default is 1).  
    - Input: Items from "Split Out".  
    - Output: Each URL is processed in turn, connected to "Total" and "Request".  
    - Edge cases: Large URL lists may slow processing; batch size could be tuned.

  - **Request**  
    - Type: HTTP Request node  
    - Role: Performs an HTTP GET request on the URL to check availability.  
    - Configuration: URL set dynamically from the input item’s "urls" property (`={{ $json.urls }}`). No additional options set.  
    - Input: Single URL item from "Bucle URLs".  
    - Output: Two outputs: main (success), error (failure), controlled by `onError: continueErrorOutput`.  
    - Edge cases: Network timeouts, DNS failures, server errors. Errors do not stop workflow but route to error output.

  - **Success**  
    - Type: Google Sheets node (append operation)  
    - Role: Logs successful URL checks into the spreadsheet, marking them as functional.  
    - Configuration: Appends rows with columns "Funcional" (text showing "Funciona: [URL]") and "Caida" (empty).  
    - Input: Success output of "Request".  
    - Output: Connected back to "Bucle URLs" to continue processing.  
    - Credentials: Uses Google Sheets OAuth2 credentials.  
    - Edge cases: Google Sheets API errors, quota limits.

  - **Error**  
    - Type: Google Sheets node (append operation)  
    - Role: Logs failed URL checks into the spreadsheet, marking them as down.  
    - Configuration: Appends rows with column "Caida" containing the URL and empty "Funcional".  
    - Input: Error output of "Request".  
    - Output: Connected back to "Bucle URLs".  
    - Credentials: Same as "Success".  
    - Edge cases: Same as "Success".

  - **Sticky Note2 & Sticky Note3**  
    - Provide user guidance on the loop process and logging to Google Sheets, in English and Spanish respectively.

---

#### 2.3 Summary & Alert Preparation

- **Overview:**  
  After processing all URLs, this block summarizes the results, extracts the URLs marked as down, and prepares data for the notification email.

- **Nodes Involved:**  
  - Total  
  - Code  
  - Split Out2  

- **Node Details:**

  - **Total**  
    - Type: Summarize node  
    - Role: Aggregates counts of "Caida" and "Funcional" fields across all processed URLs.  
    - Configuration: Summarizes fields "Caida" and "Funcional".  
    - Input: From "Bucle URLs" after all URLs processed.  
    - Output: Summary object with counts.  
    - Edge cases: If no URLs processed, summary will be empty or zero.

  - **Code**  
    - Type: Code node (JavaScript)  
    - Role: Extracts all URLs marked as down ("Caida") from the batch of URL results generated by "Bucle URLs".  
    - Configuration: Script reads all items from "Bucle URLs", collects non-empty "Caida" values, throws error if none found, returns array of down URLs in individual items.  
    - Input: Items from "Bucle URLs".  
    - Output: Items with individual down URLs keyed as "url".  
    - Edge cases: Throws error if no down URLs found, which could interrupt downstream nodes if not handled.

  - **Split Out2**  
    - Type: Split Out node  
    - Role: Splits the array of down URLs from "Code" into individual items for sending in email.  
    - Configuration: Splits on field "url".  
    - Input: Output of "Code".  
    - Output: Each down URL as separate item, connected to "Send a message".  
    - Edge cases: Empty input if "Code" errored or no down URLs.

---

#### 2.4 Notification Sending

- **Overview:**  
  Sends an email notification listing URLs currently down and includes counts of down and functional URLs.

- **Nodes Involved:**  
  - Send a message  

- **Node Details:**

  - **Send a message**  
    - Type: Gmail node  
    - Role: Sends an email with details of down URLs and summary statistics.  
    - Configuration:  
      - Recipient: `email@youemail.com` (placeholder to be replaced).  
      - Subject: "Webs caídas" (Websites down).  
      - Message body: Includes list of down URLs and counts of down and functional URLs dynamically inserted using expressions:  
        ```
        Te informamos que actualmente estos URLs se encuentran caídos: 

        {{ $json.url }}

        Caídos: {{ $('Total').item.json.count_Caida }}

        Funcionales: {{ $('Total').item.json.count_Funcional }}
        ```
    - Input: Items from "Split Out2".  
    - Credentials: Gmail OAuth2 credentials configured.  
    - Edge cases: Email send failures due to authentication, quota limits, invalid recipient address.

---

### 3. Summary Table

| Node Name       | Node Type               | Functional Role                     | Input Node(s)       | Output Node(s)   | Sticky Note                                                                                                  |
|-----------------|-------------------------|-----------------------------------|---------------------|------------------|--------------------------------------------------------------------------------------------------------------|
| Schedule Trigger| Schedule Trigger         | Periodic workflow start            | None                | URLs             | "### As a trigger you could also implement webhook or MCP"                                                  |
| Run trigger     | Manual Trigger           | Manual workflow start              | None                | URLs             |                                                                                                              |
| URLs            | Set                      | Define URLs to monitor             | Schedule Trigger, Run trigger | Split Out       | "### Enter URLs to scan here"                                                                               |
| Split Out       | Split Out                | Split URLs object into items       | URLs                | Bucle URLs       | "## How it works (P1) Before the loop, you enter the URLs to scan in the \"URLs\" stream..."                |
| Bucle URLs      | SplitInBatches           | Iterate over each URL in batches   | Split Out, Success, Error | Total, Request | "## How it works (P2) Start a loop for each URL entered, adding the status to a Google Sheet..."            |
| Request         | HTTP Request             | Check URL availability             | Bucle URLs          | Success, Error   |                                                                                                              |
| Success         | Google Sheets Append     | Log functional URLs                | Request (success)   | Bucle URLs       |                                                                                                              |
| Error           | Google Sheets Append     | Log down URLs                     | Request (error)     | Bucle URLs       |                                                                                                              |
| Total           | Summarize                | Count functional and down URLs    | Bucle URLs          | Code             |                                                                                                              |
| Code            | Code                     | Extract down URLs for notification | Bucle URLs          | Split Out2       |                                                                                                              |
| Split Out2      | Split Out                | Split down URLs for email items   | Code                | Send a message   |                                                                                                              |
| Send a message  | Gmail                    | Send email alert with down URLs   | Split Out2          | None             |                                                                                                              |
| Sticky Note     | Sticky Note              | Instructional notes                | None                | None             | Various notes on working steps and context (see nodes 4,5 and others).                                       |
| Sticky Note1    | Sticky Note              | Instructional notes                | None                | None             |                                                                                                              |
| Sticky Note2    | Sticky Note              | Instructional notes                | None                | None             |                                                                                                              |
| Sticky Note3    | Sticky Note              | Instructional notes                | None                | None             |                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes**  
   - Add a **Schedule Trigger** node. Use default interval for periodic execution (e.g., every minute).  
   - Add a **Manual Trigger** node to allow manual runs.

2. **Define URLs Node**  
   - Add a **Set** node named "URLs".  
   - Configure in "Raw JSON" mode with a single field `urls` containing an object of key-value pairs, where keys are labels ("URL 1", "URL 2", etc.) and values are URL strings, e.g.:  
     ```json
     {
       "URL 1": "https://google.es",
       "URL 2": "https://example.com",
       "URL 3": "https://n8n.io",
       "URL 4": "https://github.com",
       "URL 5": "https://n8niouou.io",
       "URL 6": "https://n8niopuou.io"
     }
     ```
3. **Connect Triggers to URLs node**  
   - Connect both "Schedule Trigger" and "Manual Trigger" nodes to the "URLs" node.

4. **Split URLs**  
   - Add a **Split Out** node named "Split Out".  
   - Configure to split the `urls` field from incoming JSON into individual items.  
   - Connect "URLs" to "Split Out".

5. **Batch Processing of URLs**  
   - Add a **SplitInBatches** node named "Bucle URLs".  
   - Use default batch size (1) for sequential processing.  
   - Connect "Split Out" to "Bucle URLs".

6. **HTTP Request for Each URL**  
   - Add an **HTTP Request** node named "Request".  
   - Set the URL field dynamically to `={{ $json.urls }}` to get URL from the current item.  
   - Set "On Error" to "Continue on Error Output" to handle failures gracefully.  
   - Connect "Bucle URLs" to "Request".

7. **Google Sheets Setup**  
   - Create a Google Sheets spreadsheet with columns "Funcional" and "Caida".  
   - Configure OAuth2 credentials in n8n for Google Sheets access.

8. **Success Logging**  
   - Add a **Google Sheets** node named "Success".  
   - Set operation to "Append". Select the spreadsheet and sheet to write to.  
   - Map columns:  
     - "Funcional" = `Funciona: {{ $('Split Out').item.json.urls }}`  
     - "Caida" = `=` (empty)  
   - Connect the success output of "Request" to "Success".  
   - Connect "Success" back to "Bucle URLs" to continue.

9. **Error Logging**  
   - Add a **Google Sheets** node named "Error".  
   - Set operation to "Append". Same spreadsheet and sheet as "Success".  
   - Map columns:  
     - "Caida" = `Caida: {{ $json.urls }}`  
     - "Funcional" = `=` (empty)  
   - Connect the error output of "Request" to "Error".  
   - Connect "Error" back to "Bucle URLs".

10. **Summarize Results**  
    - Add a **Summarize** node named "Total".  
    - Configure to summarize counts of fields "Caida" and "Funcional".  
    - Connect "Bucle URLs" main output to "Total".

11. **Extract Down URLs**  
    - Add a **Code** node named "Code".  
    - Use the following JavaScript code to extract all URLs marked as down:  
      ```javascript
      const items = $items('Bucle URLs');
      const caidaUrls = items
        .map(it => it.json.Caida)
        .filter(url => typeof url === 'string' && url.trim().length);
      if (caidaUrls.length === 0) {
        throw new Error('No se encontraron URLs en la propiedad "Caida".');
      }
      return caidaUrls.map(url => ({ json: { url } }));
      ```
    - Connect "Total" to "Code".

12. **Split Down URLs for Email**  
    - Add a **Split Out** node named "Split Out2".  
    - Configure to split field "url" from Code node output.  
    - Connect "Code" to "Split Out2".

13. **Send Email Notification**  
    - Add a **Gmail** node named "Send a message".  
    - Configure OAuth2 credentials for Gmail.  
    - Set recipient email address (replace `email@youemail.com` with real address).  
    - Subject: "Webs caídas".  
    - Message:  
      ```
      Te informamos que actualmente estos URLs se encuentran caídos: 

      {{ $json.url }}

      Caídos: {{ $('Total').item.json.count_Caida }}

      Funcionales: {{ $('Total').item.json.count_Funcional }}
      ```
    - Connect "Split Out2" to "Send a message".

14. **Optional**: Add Sticky Notes to document workflow steps and instructions.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                                                         |
|-------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Alternative triggers such as Webhook or MCP can be used instead of schedule or manual triggers. | Sticky Note5                                                                                           |
| URLs must be correctly formatted and reachable to ensure accurate monitoring.                    | General workflow requirement                                                                            |
| Google Sheets quota limits and API usage restrictions might affect logging in large-scale usage. | General operational consideration                                                                       |
| Email sending requires valid Gmail OAuth2 credentials and email recipient address configuration. | Node "Send a message" configuration                                                                     |
| Workflow is bilingual with instructional notes in English and Spanish to aid understanding.      | Sticky Notes 2 & 3                                                                                      |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.