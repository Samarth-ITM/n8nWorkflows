Track Website Traffic & Backlinks with Ahref API and Google Sheets

https://n8nworkflows.xyz/workflows/track-website-traffic---backlinks-with-ahref-api-and-google-sheets-8757


# Track Website Traffic & Backlinks with Ahref API and Google Sheets

### 1. Workflow Overview

This n8n workflow automates the retrieval and logging of website traffic and backlink data by integrating a user-submitted domain with the RapidAPI "Website Traffic Checker" API and Google Sheets. It is designed for SEO professionals, digital marketers, and agencies to monitor website analytics efficiently.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures domain input from a web form submission.
- **1.2 Domain Preparation:** Stores the submitted domain for consistent reuse.
- **1.3 API Interaction:** Sends the domain to the RapidAPI endpoint and receives traffic and backlink data.
- **1.4 API Response Validation:** Checks the API response status to determine success or failure.
- **1.5 Failure Handling:** Sends an email alert if the API call fails.
- **1.6 Data Extraction & Logging:** Extracts backlink and traffic data from the API response and appends it into respective Google Sheets.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Triggers the workflow upon user submission of a domain through a web form, ensuring the primary input is captured correctly.

- **Nodes Involved:**  
  - On form submission  
  - Sticky Note (explaining the trigger)

- **Node Details:**  

  - **On form submission**  
    - *Type & Role:* Form Trigger; initiates workflow when a form is submitted.  
    - *Configuration:* Form titled "Website Traffic Checker" with a single required field "Domain".  
    - *Key Variables:* `$json.Domain` captures the submitted domain string.  
    - *Connections:* Outputs to "Set Domain Value".  
    - *Failure Cases:* Missing domain input (form validation), webhook connectivity issues.

  - **Sticky Note** (Positioned near the trigger)  
    - Describes that this node triggers workflow on domain submission and enforces the domain field as required.

#### 2.2 Domain Preparation

- **Overview:**  
  Stores the submitted domain into a consistent variable for reuse in subsequent calls and nodes.

- **Nodes Involved:**  
  - Set Domain Value  
  - Sticky Note (explaining the node)

- **Node Details:**  

  - **Set Domain Value**  
    - *Type & Role:* Set node; assigns the domain from the form submission to a variable `Domain`.  
    - *Configuration:* Sets `Domain` = `={{ $json.Domain }}` from the form trigger output.  
    - *Connections:* Outputs to "Check Website Traffic API".  
    - *Failure Cases:* Expression failure if `$json.Domain` is undefined or malformed.

  - **Sticky Note**  
    - Notes the purpose of storing the domain for consistent referencing.

#### 2.3 API Interaction

- **Overview:**  
  Sends a POST request to the RapidAPI "Website Traffic Checker" endpoint with the domain as multipart form data, including required headers for API access.

- **Nodes Involved:**  
  - Check Website Traffic API  
  - Sticky Note (explaining the HTTP Request node)

- **Node Details:**  

  - **Check Website Traffic API**  
    - *Type & Role:* HTTP Request node; performs POST to RapidAPI endpoint.  
    - *Configuration:*  
      - URL: `https://website-traffic-checker-ahref.p.rapidapi.com/check-traffic.php`  
      - Method: POST  
      - Content-Type: multipart/form-data  
      - Body Parameter: domain = `={{ $json.Domain }}`  
      - Headers:  
        - `x-rapidapi-host`: `website-traffic-checker-ahref.p.rapidapi.com`  
        - `x-rapidapi-key`: `your key` (must be replaced with a valid API key)  
      - Response: Full HTTP response captured including status code.  
    - *Connections:* Outputs to "Check if API Call Succeeded".  
    - *Failure Cases:* Invalid API key, network errors, invalid domain format, API rate limits, timeouts.

  - **Sticky Note**  
    - Clarifies the setup of the POST request with headers and form data.

#### 2.4 API Response Validation

- **Overview:**  
  Verifies if the API call succeeded by checking for HTTP status code 200, branching workflow accordingly.

- **Nodes Involved:**  
  - Check if API Call Succeeded  
  - Sticky Note (explaining the conditional check)

- **Node Details:**  

  - **Check if API Call Succeeded**  
    - *Type & Role:* If node; conditionally routes workflow based on status code.  
    - *Configuration:* Checks if `$json.statusCode == 200`.  
    - *Connections:*  
      - True branch: to "Extract Backlink Info" and "Extract Traffic Info" nodes.  
      - False branch: to "Send Failure Email Alert" node.  
    - *Failure Cases:* Unexpected response formats, missing statusCode field.

  - **Sticky Note**  
    - Describes the logic for branching on success or failure of API call.

#### 2.5 Failure Handling

- **Overview:**  
  Sends an email alert to notify administrators if the API call fails, including the domain and HTTP status code.

- **Nodes Involved:**  
  - Send Failure Email Alert  
  - Sticky Note (explaining the email alert)

- **Node Details:**  

  - **Send Failure Email Alert**  
    - *Type & Role:* Email Send node; notifies admin of failure.  
    - *Configuration:*  
      - To: `user@test.com` (replace with actual recipient)  
      - From: `admin@test.com`  
      - Subject: `Failed to Retrieve Data for - {{ domain }}`  
      - HTML Body: Includes domain and status code dynamically from workflow data.  
    - *Credentials:* Uses SMTP credentials named "SMTP account".  
    - *Connections:* Ends workflow path.  
    - *Failure Cases:* SMTP authentication errors, invalid email addresses, network issues.

  - **Sticky Note**  
    - Notes the purpose of sending failure alerts with relevant data.

#### 2.6 Data Extraction & Logging

- **Overview:**  
  Extracts backlink and traffic data from API response JSON body and logs them into two separate Google Sheets for ongoing monitoring.

- **Nodes Involved:**  
  - Extract Backlink Info  
  - Log Backlinks to Sheet  
  - Extract Traffic Info  
  - Log Traffic to Sheet  
  - Sticky Notes (explaining extraction and logging)

- **Node Details:**  

  - **Extract Backlink Info**  
    - *Type & Role:* Code node; extracts `backlinks_info` object from API response body.  
    - *Code:* `return $input.first().json.body.backlinks_info;`  
    - *Connections:* Outputs to "Log Backlinks to Sheet".  
    - *Failure Cases:* Missing or malformed `backlinks_info` field.

  - **Log Backlinks to Sheet**  
    - *Type & Role:* Google Sheets node; appends backlink data to sheet "Backlink Info".  
    - *Configuration:*  
      - Document ID: (set to target Google Sheet)  
      - Sheet: `gid=0` (sheet named "Backlink Info")  
      - Operation: Append  
      - Columns mapped explicitly: `ascore`, `website` (domain), `total backlinks`, `referring domain`.  
      - Authentication: Google Service Account credentials.  
    - *Connections:* Ends backlink logging path.  
    - *Failure Cases:* Google API auth errors, quota exceeded, invalid data types.

  - **Extract Traffic Info**  
    - *Type & Role:* Code node; extracts `traffic_analysis` object from API response body.  
    - *Code:* `return $input.first().json.body.traffic_analysis;`  
    - *Connections:* Outputs to "Log Traffic to Sheet".  
    - *Failure Cases:* Missing or malformed `traffic_analysis` data.

  - **Log Traffic to Sheet**  
    - *Type & Role:* Google Sheets node; appends traffic data to sheet "Traffic Data".  
    - *Configuration:*  
      - Document ID: (set to target Google Sheet)  
      - Sheet: identified by numeric ID 1530109257 (named "Traffic Data")  
      - Operation: Append  
      - Columns: Automatically mapped from input data with extensive traffic-related fields (e.g., bounce_rate, users, visits, etc.).  
      - Authentication: Google Service Account credentials.  
    - *Connections:* Ends traffic logging path.  
    - *Failure Cases:* Google API auth errors, mapping errors, rate limits.

  - **Sticky Notes**  
    - For backlink extraction and logging: explains the extraction of `backlinks_info` and logging key backlink metrics to the sheet.  
    - For traffic extraction and logging: explains extraction of `traffic_analysis` and automatic mapping of traffic metrics to the sheet.

---

### 3. Summary Table

| Node Name                | Node Type           | Functional Role                                  | Input Node(s)            | Output Node(s)                         | Sticky Note                                                                                                              |
|--------------------------|---------------------|-------------------------------------------------|--------------------------|---------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| On form submission       | Form Trigger        | Trigger workflow on domain submission            | —                        | Set Domain Value                      | Triggers the workflow when a user submits a form with a domain name. Captures the input field Domain as required.         |
| Set Domain Value         | Set                 | Stores domain variable for reuse                  | On form submission       | Check Website Traffic API             | Stores the submitted domain value into a variable Domain for reuse. Ensures consistent referencing throughout the workflow. |
| Check Website Traffic API| HTTP Request        | Sends POST request to RapidAPI with domain       | Set Domain Value         | Check if API Call Succeeded           | Sends a POST request to the RapidAPI endpoint with the submitted domain. Includes required headers and multipart form data.|
| Check if API Call Succeeded | If               | Checks if API response status is 200              | Check Website Traffic API| Extract Backlink Info; Extract Traffic Info; Send Failure Email Alert | Checks if the API response returned HTTP status code 200. Branches based on success or failure.                          |
| Send Failure Email Alert | Email Send          | Sends email alert on API failure                   | Check if API Call Succeeded (false branch) | —                              | Sends an email alert if the API call fails. Includes the failed domain and HTTP status code in the message.                |
| Extract Backlink Info    | Code                | Extracts backlinks_info from API response         | Check if API Call Succeeded (true branch) | Log Backlinks to Sheet            | Extracts the backlinks_info object from the API response body. This data will be logged to the backlinks sheet.             |
| Log Backlinks to Sheet   | Google Sheets       | Appends backlink metrics to "Backlink Info" sheet| Extract Backlink Info    | —                                   | Appends backlink-related data to the "Backlink Info" sheet. Logs values like ascore, referring domains, and total backlinks.|
| Extract Traffic Info     | Code                | Extracts traffic_analysis from API response       | Check if API Call Succeeded (true branch) | Log Traffic to Sheet             | Extracts traffic_analysis data from the API response body. Prepares traffic metrics for logging.                           |
| Log Traffic to Sheet     | Google Sheets       | Appends traffic data to "Traffic Data" sheet      | Extract Traffic Info     | —                                   | Appends website traffic data to the "Traffic Data" sheet. Automatically maps fields from the API response to sheet columns.|
| Sticky Note              | Sticky Note         | Explanatory notes for trigger node                 | —                        | —                                   | Triggers the workflow when a user submits a form with a domain name. Captures the input field Domain as required.          |
| Sticky Note1             | Sticky Note         | Explanatory notes for domain variable setter       | —                        | —                                   | Stores the submitted domain value into a variable Domain for reuse. Ensures consistent referencing throughout the workflow.|
| Sticky Note2             | Sticky Note         | Explanatory notes for API request setup            | —                        | —                                   | Sends a POST request to the RapidAPI endpoint with the submitted domain. Includes required headers and multipart form data.|
| Sticky Note3             | Sticky Note         | Explains API response status check                  | —                        | —                                   | Checks if the API response returned HTTP status code 200. Branches based on success or failure.                           |
| Sticky Note4             | Sticky Note         | Explains failure email alert                         | —                        | —                                   | Sends an email alert if the API call fails. Includes the failed domain and HTTP status code in the message.               |
| Sticky Note5             | Sticky Note         | Explains backlink info extraction                    | —                        | —                                   | Extracts the backlinks_info object from the API response body. This data will be logged to the backlinks sheet.            |
| Sticky Note6             | Sticky Note         | Explains backlink logging to sheet                   | —                        | —                                   | Appends backlink-related data to the "Backlink Info" sheet. Logs values like ascore, referring domains, and total backlinks.|
| Sticky Note7             | Sticky Note         | Explains traffic info extraction                      | —                        | —                                   | Extracts traffic_analysis data from the API response body. Prepares traffic metrics for logging.                          |
| Sticky Note8             | Sticky Note         | Explains traffic logging to sheet                     | —                        | —                                   | Appends website traffic data to the "Traffic Data" sheet. Automatically maps fields from the API response to sheet columns.|
| Sticky Note9             | Sticky Note         | Workflow overview and use case summary                | —                        | —                                   | Automated Website Traffic & Backlink Tracker in Google Sheets. Summary and use case description included.                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node**  
   - Node Type: Form Trigger  
   - Name: "On form submission"  
   - Configure form with title: "Website Traffic Checker"  
   - Add one required field: Label = "Domain" (type: string)  
   - Save and activate webhook.

2. **Create a Set Node**  
   - Node Type: Set  
   - Name: "Set Domain Value"  
   - Add one field: Name = "Domain", Type = string  
   - Set value: `={{ $json.Domain }}` (copy from form trigger output)  
   - Connect "On form submission" output to this node's input.

3. **Create an HTTP Request Node**  
   - Node Type: HTTP Request  
   - Name: "Check Website Traffic API"  
   - HTTP Method: POST  
   - URL: `https://website-traffic-checker-ahref.p.rapidapi.com/check-traffic.php`  
   - Content-Type: multipart/form-data  
   - Add Body Parameter: name = "domain", value = `={{ $json.Domain }}`  
   - Add Header Parameters:  
     - `x-rapidapi-host` = `website-traffic-checker-ahref.p.rapidapi.com`  
     - `x-rapidapi-key` = *(enter your valid RapidAPI key)*  
   - Enable "Always Output Data" to capture full response  
   - Connect "Set Domain Value" output to this node's input.

4. **Create an If Node**  
   - Node Type: If  
   - Name: "Check if API Call Succeeded"  
   - Condition: Check if `$json.statusCode == 200` (strict number equality)  
   - Connect HTTP Request node output to this if node.

5. **Create an Email Send Node**  
   - Node Type: Email Send  
   - Name: "Send Failure Email Alert"  
   - To Email: `user@test.com` (replace with actual recipient)  
   - From Email: `admin@test.com`  
   - Subject: `Failed to Retrieve Data for - {{ $('Set Domain Value').item.json.Domain }}`  
   - Email Body (HTML):  
     ```
     Dear Team,<br><br>
     This is to inform you that the system failed to retrieve data for: {{ $('Set Domain Value').item.json.Domain }}.<br>
     Status Code: {{ $json.statusCode }}<br><br>
     Kindly take note of this issue and do the needful at the earliest.<br><br>
     Regards,<br>
     n8n Automation
     ```  
   - Credentials: Configure SMTP credentials (e.g., Gmail, Outlook SMTP)  
   - Connect If node's "false" (failure) branch to this node.

6. **Create a Code Node to Extract Backlink Info**  
   - Node Type: Code  
   - Name: "Extract Backlink Info"  
   - JavaScript Code: `return $input.first().json.body.backlinks_info;`  
   - Connect If node's "true" (success) branch to this node.

7. **Create Google Sheets Node for Backlinks**  
   - Node Type: Google Sheets  
   - Name: "Log Backlinks to Sheet"  
   - Operation: Append  
   - Sheet Name: "Backlink Info" (gid=0 or by name)  
   - Document ID: Enter your Google Sheets document ID or URL  
   - Authentication: Google Service Account (configure credentials)  
   - Mapping Mode: Define below  
   - Columns to map:  
     - website: `={{ $('Set Domain Value').item.json.Domain }}`  
     - ascore: `={{ $json.data.ascore }}`  
     - total backlinks: `={{ $json.data.total }}`  
     - referring domain: `={{ $json.data.domains }}`  
   - Connect "Extract Backlink Info" output to this node.

8. **Create Code Node to Extract Traffic Info**  
   - Node Type: Code  
   - Name: "Extract Traffic Info"  
   - JavaScript Code: `return $input.first().json.body.traffic_analysis;`  
   - Connect If node's "true" branch also to this node (parallel to backlink extraction).

9. **Create Google Sheets Node for Traffic Data**  
   - Node Type: Google Sheets  
   - Name: "Log Traffic to Sheet"  
   - Operation: Append  
   - Sheet Name: "Traffic Data" (using gid or name)  
   - Document ID: Same or different sheet document ID as appropriate  
   - Authentication: Google Service Account  
   - Mapping Mode: Auto map input data (handles multiple traffic fields)  
   - Connect "Extract Traffic Info" output to this node.

**Additional Notes:**  
- Replace placeholder API keys and Google Sheets document IDs with live credentials.  
- Ensure all credentials (SMTP, Google API) are configured in n8n.  
- Test webhook URL to confirm form submission triggers correctly.  
- Validate Google Sheets column headers match mapped fields.  
- Handle API rate limits by implementing delays or error handling as required.

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                              |
|------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Automated Website Traffic & Backlink Tracker in Google Sheets. This workflow is ideal for SEO and digital marketing monitoring.    | Summary included in Sticky Note near workflow start (Sticky Note9).                                         |
| Use of RapidAPI "Website Traffic Checker" endpoint requires valid API key and adherence to API usage policies.                     | API Documentation: https://rapidapi.com/                                                                           |
| Google Sheets nodes use Service Account authentication; ensure proper permissions on target sheets.                                | Google Sheets API Setup Guide: https://developers.google.com/sheets/api/quickstart/nodejs                     |
| SMTP credentials must be valid and tested to enable email failure notifications.                                                   | SMTP Setup varies by provider; common examples include Gmail, Outlook SMTP credentials.                       |

---

This completes the comprehensive reference for the "Track Website Traffic & Backlinks with Ahref API and Google Sheets" n8n workflow.