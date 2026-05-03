Monitor Authentication IPs from SaaS Alerts & Email Reports via SMTP2Go

https://n8nworkflows.xyz/workflows/monitor-authentication-ips-from-saas-alerts---email-reports-via-smtp2go-3126


# Monitor Authentication IPs from SaaS Alerts & Email Reports via SMTP2Go

### 1. Workflow Overview

This n8n workflow automates the collection and reporting of user authentication IP addresses from multiple SaaS alert event sources over the last 24 hours. It is designed to help security teams, IT administrators, and compliance officers monitor sign-in activity, detect suspicious IPs, and receive timely email alerts via SMTP2Go.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Captures user input (name, email, API key) via a web form trigger.
- **1.2 Variable Initialization:** Sets date/time and form variables used throughout the workflow.
- **1.3 Data Retrieval:** Queries three SaaS alert event APIs for different authentication event types within the last 24 hours.
- **1.4 Data Aggregation and Filtering:** Merges, filters, and deduplicates authentication event data to extract relevant IP information.
- **1.5 Data Formatting:** Converts filtered data into CSV format and encodes it for email attachment.
- **1.6 Email Notification:** Sends the compiled IP report as an email attachment using SMTP2Go.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Captures user input via a form trigger node, requesting the user’s name, email address, and SaaS Alerts API key to personalize and authenticate the workflow execution.

- **Nodes Involved:**  
  - Authentication Request Form

- **Node Details:**  
  - **Authentication Request Form**  
    - Type: Form Trigger (Webhook-based)  
    - Configuration:  
      - Form titled "Request Sign-In CSV" with three required fields: Name (text), Email (email), API key (text).  
      - Button labeled "Process".  
      - Description explains the purpose and expected wait time.  
    - Input: HTTP webhook triggered by form submission.  
    - Output: JSON containing user inputs for downstream use.  
    - Edge Cases: Missing required fields will prevent submission; webhook availability is critical.  
    - Notes: Webhook ID is fixed; ensure it is accessible externally.

#### 2.2 Variable Initialization

- **Overview:**  
  Sets workflow-wide variables including the timestamp for "last 24 hours," and stores user inputs from the form for use in API requests and email content.

- **Nodes Involved:**  
  - Set Date and Form Variables

- **Node Details:**  
  - **Set Date and Form Variables**  
    - Type: Set node  
    - Configuration:  
      - Defines "Last 24 Hours" as ISO timestamp 24 hours before current time.  
      - Passes through user inputs: API key, Name, and Email.  
    - Input: Data from Authentication Request Form node.  
    - Output: JSON with date and user variables.  
    - Edge Cases: Date calculation depends on system time; if system clock is incorrect, data range may be wrong.

#### 2.3 Data Retrieval

- **Overview:**  
  Queries three separate SaaS Alerts API endpoints to fetch authentication events of different types (login success, OAuth permission grants, Office365 shell logins) from the last 24 hours.

- **Nodes Involved:**  
  - GET Events - Login Successful  
  - GET Events - OAuth Authentication  
  - GET Events - Office365 Shell WCSS

- **Node Details:**  
  - **GET Events - Login Successful**  
    - Type: HTTP Request  
    - Configuration:  
      - GET request to SaaS Alerts API endpoint for `login.success` events.  
      - URL includes dynamic query parameters: eventType, start time (last 24 hours), sorting, size, and scroll timeout.  
      - API key passed as HTTP header.  
    - Input: Variables from Set Date and Form Variables node.  
    - Output: JSON array of login success events.  
    - Error Handling: Continues workflow on error (non-blocking).  
    - Edge Cases: API rate limits, invalid API key, network timeouts.

  - **GET Events - OAuth Authentication**  
    - Type: HTTP Request  
    - Configuration: Similar to above but queries `oauth.granted.permission` events.  
    - Same input/output and error handling.

  - **GET Events - Office365 Shell WCSS**  
    - Type: HTTP Request  
    - Configuration: Queries `ms.shell.login.success` events.  
    - Same input/output and error handling.

#### 2.4 Data Aggregation and Filtering

- **Overview:**  
  Merges the three event data streams, filters for relevant IP and user information, and removes duplicate IP entries to prepare a clean dataset.

- **Nodes Involved:**  
  - Combine all Authentication Events  
  - Filter IP Information  
  - Remove Duplicate IPs

- **Node Details:**  
  - **Combine all Authentication Events**  
    - Type: Merge  
    - Configuration: Merges three inputs (login success, OAuth, Office365 shell) into one dataset.  
    - Input: Outputs from the three HTTP Request nodes.  
    - Output: Combined event list.  
    - Edge Cases: If any input is empty or error, merge still proceeds.

  - **Filter IP Information**  
    - Type: Set  
    - Configuration:  
      - Selects specific fields: customer.name, user.fullName, ip, location.city, location.region, location.country.  
      - Includes other fields as well.  
    - Input: Combined events.  
    - Output: Filtered dataset with relevant authentication IP info.  
    - Edge Cases: Missing fields in some events may cause incomplete data.

  - **Remove Duplicate IPs**  
    - Type: Remove Duplicates  
    - Configuration: Compares on the "ip" field to remove duplicate IP entries.  
    - Input: Filtered IP info.  
    - Output: Deduplicated list of IPs.  
    - Edge Cases: IP field missing or malformed may cause duplicates to persist.

#### 2.5 Data Formatting

- **Overview:**  
  Converts the deduplicated IP data into CSV format and encodes it as base64 for email attachment compatibility.

- **Nodes Involved:**  
  - Convert to CSV  
  - Convert CSV to Base64

- **Node Details:**  
  - **Convert to CSV**  
    - Type: Convert To File  
    - Configuration: Includes header row in CSV output.  
    - Input: Deduplicated IP data.  
    - Output: CSV file binary data.  
    - Edge Cases: Large datasets may affect performance.

  - **Convert CSV to Base64**  
    - Type: Move Binary Data  
    - Configuration: Encodes CSV binary data to base64 string.  
    - Input: CSV binary data.  
    - Output: Base64 encoded CSV string for email attachment.  
    - Edge Cases: Encoding failures unlikely but possible with corrupted data.

#### 2.6 Email Notification

- **Overview:**  
  Sends the compiled authentication IP report as an email attachment using the SMTP2Go API, addressed to the user who submitted the form.

- **Nodes Involved:**  
  - Send Email Upon Completion (SMTP2Go)

- **Node Details:**  
  - **Send Email Upon Completion (SMTP2Go)**  
    - Type: HTTP Request  
    - Configuration:  
      - POST request to SMTP2Go email send API endpoint.  
      - Uses HTTP header authentication with stored SMTP2Go credentials.  
      - Email sender fixed as support@managedsaasalerts.com.  
      - Recipient email dynamically set from form input.  
      - Email subject: "Workflow Complete".  
      - Email body includes user name and notification text.  
      - Attachment: base64 encoded CSV file named "testfile.csv" with MIME type application/csv.  
    - Input: Base64 CSV data and user variables.  
    - Output: API response confirming email sent.  
    - Edge Cases: SMTP2Go API key invalid, network errors, attachment size limits.

---

### 3. Summary Table

| Node Name                          | Node Type           | Functional Role                         | Input Node(s)                             | Output Node(s)                          | Sticky Note                                                                                      |
|-----------------------------------|---------------------|---------------------------------------|------------------------------------------|---------------------------------------|------------------------------------------------------------------------------------------------|
| Authentication Request Form        | Form Trigger        | Capture user input (name, email, API) | (Webhook trigger)                        | Set Date and Form Variables            |                                                                                                |
| Set Date and Form Variables        | Set                 | Initialize date and form variables     | Authentication Request Form              | GET Events - Login Successful, GET Events - OAuth Authentication, GET Events - Office365 Shell WCSS |                                                                                                |
| GET Events - Login Successful      | HTTP Request        | Fetch login.success events             | Set Date and Form Variables              | Combine all Authentication Events     |                                                                                                |
| GET Events - OAuth Authentication  | HTTP Request        | Fetch oauth.granted.permission events | Set Date and Form Variables              | Combine all Authentication Events     |                                                                                                |
| GET Events - Office365 Shell WCSS  | HTTP Request        | Fetch ms.shell.login.success events    | Set Date and Form Variables              | Combine all Authentication Events     |                                                                                                |
| Combine all Authentication Events  | Merge               | Merge all event datasets                | GET Events - Login Successful, GET Events - OAuth Authentication, GET Events - Office365 Shell WCSS | Filter IP Information                 |                                                                                                |
| Filter IP Information              | Set                 | Filter relevant IP and user info       | Combine all Authentication Events       | Remove Duplicate IPs                   |                                                                                                |
| Remove Duplicate IPs               | Remove Duplicates   | Remove duplicate IP entries             | Filter IP Information                    | Convert to CSV                        |                                                                                                |
| Convert to CSV                    | Convert To File     | Convert filtered data to CSV            | Remove Duplicate IPs                     | Convert CSV to Base64                  |                                                                                                |
| Convert CSV to Base64              | Move Binary Data    | Encode CSV file to base64 for email    | Convert to CSV                          | Send Email Upon Completion (SMTP2Go)  |                                                                                                |
| Send Email Upon Completion (SMTP2Go) | HTTP Request        | Send email with CSV attachment          | Convert CSV to Base64                    | (End)                                | ## SMTP2Go API API Documentation [Link](https://developers.smtp2go.com/docs/send-an-email)      |
| Sticky Note                       | Sticky Note         | SaaS Alerts API Reference Guide         | None                                    | None                                 | ## Query the SaaS Alerts API SaaS Alerts API Reference Guide [Link](https://app.swaggerhub.com/apis/SaaS_Alerts/functions) |
| Sticky Note1                      | Sticky Note         | Data Processing and Deduplication       | None                                    | None                                 |                                                                                                |
| Sticky Note2                      | Sticky Note         | SMTP2Go API Documentation               | None                                    | None                                 | ## SMTP2Go API API Documentation [Link](https://developers.smtp2go.com/docs/send-an-email)      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Type: Form Trigger  
   - Configure webhook with external access.  
   - Form Title: "Request Sign-In CSV"  
   - Add fields:  
     - Name (text, required)  
     - Email (email, required)  
     - API key (text, required)  
   - Button label: "Process"  
   - Description: Explain purpose and expected wait time.

2. **Create Set Node for Variables**  
   - Name: Set Date and Form Variables  
   - Add fields:  
     - "Last 24 Hours": Expression `{{ new Date(Date.now() - 24 * 60 * 60 * 1000).toISOString() }}`  
     - "API": Expression `{{ $json["What is your API key?"] }}`  
     - "Name": Expression `{{ $json["What is your name?"] }}`  
     - "Email": Expression `{{ $json["What is your e-mail?"] }}`  
   - Connect output of Form Trigger to this node.

3. **Create HTTP Request Nodes to Fetch Events**  
   - For each event type, create one HTTP Request node:  
     - **GET Events - Login Successful**  
       - Method: GET  
       - URL: `https://us-central1-the-byway-248217.cloudfunctions.net/reportApi/api/v1/reports/events?eventType=login.success&start={{ $json['Last 24 Hours'] }}&timeSort=asc&size=10000&scroll=5s`  
       - Headers: `api_key` set to `{{ $json.API }}`  
       - On Error: Continue (do not stop workflow)  
     - **GET Events - OAuth Authentication**  
       - Same as above, eventType=`oauth.granted.permission`  
     - **GET Events - Office365 Shell WCSS**  
       - Same as above, eventType=`ms.shell.login.success`  
   - Connect output of Set Date and Form Variables node to all three HTTP Request nodes.

4. **Create Merge Node**  
   - Name: Combine all Authentication Events  
   - Type: Merge  
   - Number of Inputs: 3  
   - Mode: Append (default)  
   - Connect outputs of all three HTTP Request nodes to this node.

5. **Create Set Node to Filter Fields**  
   - Name: Filter IP Information  
   - Select fields to include: `customer.name, user.fullName, ip, location.city, location.region, location.country`  
   - Include other fields as needed.  
   - Connect output of Merge node to this node.

6. **Create Remove Duplicates Node**  
   - Name: Remove Duplicate IPs  
   - Compare by field: `ip`  
   - Connect output of Filter IP Information node to this node.

7. **Create Convert To File Node**  
   - Name: Convert to CSV  
   - Format: CSV  
   - Include header row: Yes  
   - Connect output of Remove Duplicate IPs node to this node.

8. **Create Move Binary Data Node**  
   - Name: Convert CSV to Base64  
   - Operation: Encode binary data to base64  
   - Connect output of Convert to CSV node to this node.

9. **Create HTTP Request Node to Send Email**  
   - Name: Send Email Upon Completion (SMTP2Go)  
   - Method: POST  
   - URL: `https://api.smtp2go.com/v3/email/send`  
   - Authentication: HTTP Header Auth with SMTP2Go API key credential  
   - Headers:  
     - Content-Type: application/json  
     - Accept: application/json  
   - Body (JSON):  
     ```json
     {
       "sender": "support@managedsaasalerts.com",
       "to": ["{{ $('Set Date and Form Variables').first().json.Email }}"],
       "attachments": [
         {
           "filename": "testfile.csv",
           "fileblob": "{{ $json.data }}",
           "mimetype": "application/csv"
         }
       ],
       "subject": "Workflow Complete",
       "text_body": "{{ $('Set Date and Form Variables').first().json.Name }}, attached is your IP information.\n\n\n\n"
     }
     ```  
   - Connect output of Convert CSV to Base64 node to this node.  
   - Set to execute once per workflow run.

10. **Activate and Test**  
    - Deploy the workflow.  
    - Test by submitting the form with valid inputs.  
    - Verify email receipt with attached CSV.

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                                                                              |
|---------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| SaaS Alerts API Reference Guide available at SwaggerHub for detailed API endpoint documentation.        | https://app.swaggerhub.com/apis/SaaS_Alerts/functions                                                        |
| SMTP2Go API documentation for sending emails with attachments.                                          | https://developers.smtp2go.com/docs/send-an-email                                                           |
| The workflow is designed to handle large data sets (up to 10,000 events per API call) with scroll support.| Ensure API keys have sufficient permissions and rate limits are respected.                                   |
| Email sender address (support@managedsaasalerts.com) must be verified in SMTP2Go account for successful delivery. | SMTP2Go account setup prerequisite.                                                                          |
| The workflow continues gracefully on API request errors to ensure partial data processing and reporting. | Useful for resilience but may result in incomplete reports if APIs fail.                                     |
| Customize filtering criteria in the "Filter IP Information" node to track specific users or IP ranges. | Adapt to organizational security policies and monitoring needs.                                             |

---

This documentation provides a complete understanding of the workflow’s structure, logic, and configuration, enabling reproduction, modification, and troubleshooting by advanced users or automation agents.