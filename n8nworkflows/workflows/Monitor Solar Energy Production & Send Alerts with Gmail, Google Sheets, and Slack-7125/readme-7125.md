Monitor Solar Energy Production & Send Alerts with Gmail, Google Sheets, and Slack

https://n8nworkflows.xyz/workflows/monitor-solar-energy-production---send-alerts-with-gmail--google-sheets--and-slack-7125


# Monitor Solar Energy Production & Send Alerts with Gmail, Google Sheets, and Slack

### 1. Workflow Overview

This workflow automates the monitoring of solar energy production by periodically fetching production data, analyzing it for low output conditions, and sending alerts or logging data accordingly. It targets energy management teams who need timely notifications on suboptimal solar power generation along with an ongoing record of production levels. The workflow is logically divided into the following blocks:

- **1.1 Scheduled Data Fetch**: Periodic trigger and retrieval of solar production data from an external API.
- **1.2 Data Filtering and Evaluation**: Filtering out data entries that indicate low production and evaluating if alerts are required.
- **1.3 Alerting and Logging**: Sending email alerts when production is critically low, logging data to Google Sheets otherwise, and posting summaries to Slack for team visibility.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Data Fetch

- **Overview:** This block initiates the workflow every two hours and retrieves the latest solar energy production data from a designated HTTP endpoint.

- **Nodes Involved:**  
  - Trigger: Every 2 Hours  
  - Fetch Solar Production Data

- **Node Details:**

  - **Trigger: Every 2 Hours**  
    - *Type & Role:* Schedule Trigger node that runs the workflow on a time interval.  
    - *Configuration:* Set to trigger every 2 hours.  
    - *Inputs/Outputs:* No input; outputs trigger signal to "Fetch Solar Production Data".  
    - *Potential Failures:* Workflow may not trigger if n8n instance is down; no authentication needed here.

  - **Fetch Solar Production Data**  
    - *Type & Role:* HTTP Request node fetching external solar production data.  
    - *Configuration:* URL is empty in the JSON but in practice should be set to the solar data API endpoint (e.g., Energidataservice).  
    - *Inputs/Outputs:* Receives trigger from schedule; outputs raw JSON data with records array.  
    - *Potential Failures:* HTTP errors (timeout, 4xx/5xx), invalid or missing URL, API authentication errors if applicable.

#### 1.2 Data Filtering and Evaluation

- **Overview:** Filters the fetched data to identify entries where total solar production is below 1000 MW, then evaluates whether any such low-production entries exist.

- **Nodes Involved:**  
  - Filter Low Production Entries  
  - Check for Low Production

- **Node Details:**

  - **Filter Low Production Entries**  
    - *Type & Role:* Code node running JavaScript to parse and filter data entries.  
    - *Configuration:* Iterates over `records` array from API data, selects entries with `TotalCon` < 1000, adds an alert message field.  
    - *Key Expressions:* Uses `$input.first().json.records` to access data, outputs filtered array.  
    - *Inputs/Outputs:* Input from HTTP Request node; outputs filtered low production entries.  
    - *Edge Cases:* If API data format changes or `records` is missing, code may error. If no entries are below threshold, outputs empty array.

  - **Check for Low Production**  
    - *Type & Role:* If node evaluating if any production entries are below the threshold.  
    - *Configuration:* Checks if `TotalCon` field is less than 1000.  
    - *Inputs/Outputs:* Input from Code node; True branch triggers email alert, False branch logs data.  
    - *Potential Failures:* Expression errors if data fields missing or not numeric.

#### 1.3 Alerting and Logging

- **Overview:** Based on the evaluation, sends email alerts for low production or logs valid data to Google Sheets, then posts summary messages to Slack.

- **Nodes Involved:**  
  - Send Email Alert (Low Production)  
  - Log Valid Production Data  
  - Post Summary to Slack

- **Node Details:**

  - **Send Email Alert (Low Production)**  
    - *Type & Role:* Gmail node to send alert emails.  
    - *Configuration:* Sends to specified email addresses (empty in JSON, should be set), with subject and message body containing production details and timestamp.  
    - *Key Expressions:* Uses `{{ $json.HourUTC }}` and `{{ $json.TotalCon }}` to inject dynamic content.  
    - *Credentials:* Uses OAuth2 Gmail credentials.  
    - *Inputs/Outputs:* Input from If node True branch; outputs none (end node).  
    - *Edge Cases:* Email sending failure due to auth issues, invalid recipient, quota limits.

  - **Log Valid Production Data**  
    - *Type & Role:* Google Sheets node to append or update production data.  
    - *Configuration:* Maps several data fields (`alert`, `HourDK`, `HourUTC`, `TotalCon`, etc.) into a Google Sheet named "energy_production" within a specified document.  
    - *Credentials:* Google Sheets OAuth2 credentials required.  
    - *Inputs/Outputs:* Input from If node False branch; outputs to Slack node.  
    - *Edge Cases:* Sheet access permissions, rate limits, schema mismatch.

  - **Post Summary to Slack**  
    - *Type & Role:* Slack node posting messages to a Slack user/channel.  
    - *Configuration:* Sends message with latest `TotalCon` value.  
    - *Credentials:* Slack API credentials required.  
    - *Inputs/Outputs:* Input from Google Sheets node; no outputs.  
    - *Edge Cases:* Slack API permission errors, invalid channel/user.

---

### 3. Summary Table

| Node Name                   | Node Type               | Functional Role                          | Input Node(s)                 | Output Node(s)                        | Sticky Note                                                                                                                                            |
|-----------------------------|-------------------------|----------------------------------------|------------------------------|-------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------|
| Sticky Note                 | Sticky Note             | Workflow title display                  |                              |                                     | ## Solar Energy Production Monitoring Alert Workflow                                                                                                   |
| Trigger: Every 2 Hours      | Schedule Trigger        | Triggers workflow every 2 hours        |                              | Fetch Solar Production Data          | The workflow starts with a schedule trigger that runs every 2 hours to automate monitoring.                                                           |
| Fetch Solar Production Data | HTTP Request            | Retrieves solar production data via API| Trigger: Every 2 Hours        | Filter Low Production Entries        | Fetches the latest solar energy production data from Energidataservice API.                                                                           |
| Filter Low Production Entries| Code                    | Filters for low production entries     | Fetch Solar Production Data   | Check for Low Production             | Filters entries where production is below the threshold for alerting.                                                                                 |
| Check for Low Production    | If                      | Decides if low production alerts needed| Filter Low Production Entries| Send Email Alert (Low Production), Log Valid Production Data | Checks if any production data is below threshold; branches accordingly.                                                                                |
| Send Email Alert (Low Production)| Gmail              | Sends email alert for low production   | Check for Low Production (True)|                                     | Sends email notification with production details if below threshold.                                                                                 |
| Log Valid Production Data   | Google Sheets           | Logs valid production data to sheet    | Check for Low Production (False)| Post Summary to Slack                | Records normal production data for tracking in Google Sheets.                                                                                        |
| Post Summary to Slack       | Slack                   | Sends summary message to Slack         | Log Valid Production Data     |                                     | Posts latest production summary to Slack channel for team visibility.                                                                                |
| Sticky Note1                | Sticky Note             | Node breakdown and description         |                              |                                     | Detailed node breakdown & descriptions provided, explaining workflow steps and logic.                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it "Solar Energy Production Monitoring Alert Workflow".

2. **Add a Schedule Trigger node**:
   - Name: "Trigger: Every 2 Hours"
   - Set the trigger interval to every 2 hours.

3. **Add an HTTP Request node**:
   - Name: "Fetch Solar Production Data"
   - Set the HTTP method to GET.
   - Configure the URL to the solar energy production API endpoint (e.g., Energidataservice).
   - Leave authentication empty unless the API requires it.
   - Connect "Trigger: Every 2 Hours" output to this node input.

4. **Add a Code node**:
   - Name: "Filter Low Production Entries"
   - Paste the following JavaScript code:
     ```javascript
     const inputArray = $input.first().json.records;

     const alerts = [];

     for (const data of inputArray) {
       if (data.TotalCon < 1000) {
         alerts.push({
           json: {
             ...data,
             alert: `⚠️ TotalCon too low: ${data.TotalCon}`
           }
         });
       }
     }

     return alerts;
     ```
   - Connect output of "Fetch Solar Production Data" to this Code node.

5. **Add an If node**:
   - Name: "Check for Low Production"
   - Set condition to check if `TotalCon` field is less than 1000.
     - Use expression: `{{$json.TotalCon}} < 1000`
   - Connect output of "Filter Low Production Entries" to this node.

6. **Add a Gmail node**:
   - Name: "Send Email Alert (Low Production)"
   - Configure OAuth2 credentials for Gmail.
   - Set recipient email(s) in "Send To" field.
   - Set subject: "Critical Alert: Emergency Power Generation Below Threshold"
   - Set message body with dynamic expressions, e.g.:
     ```
     Dear Team,

     This is to inform you that the emergency power production has dropped below the acceptable threshold as of {{ $json.HourUTC }}.

     Details:
     Production Level: {{ $json.TotalCon }} MW
     Threshold Level: 1000 MW

     This could potentially impact dependent systems or operations. Please take immediate action to investigate and restore power production to safe levels.

     If you require assistance or further diagnostics, kindly escalate to the relevant maintenance or energy management team.

     Stay safe,
     [Your System Name / Monitoring Service]
     ```
   - Connect the True output of "Check for Low Production" to this node.

7. **Add a Google Sheets node**:
   - Name: "Log Valid Production Data"
   - Configure OAuth2 credentials for Google Sheets.
   - Select or create a Google Sheet document to store data.
   - Use sheet named "energy_production" or create it.
   - Set operation to "Append or Update".
   - Map columns:
     - alert: `={{ $json.alert }}`
     - HourDK: `={{ $json.HourDK }}`
     - HourUTC: `={{ $json.HourUTC }}`
     - TotalCon: `={{ $json.TotalCon }}`
     - PriceArea: `={{ $json.PriceArea }}`
     - ConsumerType_DE35: `={{ $json.ConsumerType_DE35 }}`
   - Connect the False output of "Check for Low Production" to this node.

8. **Add a Slack node**:
   - Name: "Post Summary to Slack"
   - Configure Slack API credentials.
   - Set message text to: `={{ $json.TotalCon }}`
   - Choose the user or channel to post the message.
   - Connect output of "Log Valid Production Data" to this node.

9. **Add Sticky Note nodes** for documentation or visual separation as desired.

10. **Save and activate** the workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                                     |
|------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Workflow title and description are documented in Sticky Notes nodes for clarity within the n8n editor.        | Visual organization inside n8n workflow editor.                                                   |
| Gmail node requires OAuth2 credentials setup with sufficient permissions to send emails on behalf of user.   | Gmail OAuth2 setup instructions: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail/ |
| Google Sheets integration requires OAuth2 credentials with read/write access to the target spreadsheet.      | Google Sheets OAuth2 setup: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googleSheets/ |
| Slack node requires API token with permission to post messages to the chosen channel or user.                 | Slack API token generation guide: https://api.slack.com/authentication/token-types                      |
| The HTTP Request node needs to be configured with the actual URL for solar production data (e.g. Energidataservice API). | Energidataservice API documentation: https://www.energidataservice.dk/                            |
| The code node assumes the API returns an array called `records` with a `TotalCon` numeric field for production.| Confirm API response format matches this assumption to avoid runtime errors.                        |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created using n8n, an integration and automation tool. This process adheres strictly to applicable content policies and contains no illegal, offensive, or protected elements. All processed data are legal and public.