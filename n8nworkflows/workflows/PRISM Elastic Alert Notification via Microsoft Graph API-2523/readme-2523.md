PRISM Elastic Alert Notification via Microsoft Graph API

https://n8nworkflows.xyz/workflows/prism-elastic-alert-notification-via-microsoft-graph-api-2523


# PRISM Elastic Alert Notification via Microsoft Graph API

### 1. Workflow Overview

This workflow automates the monitoring of PRISM Elastic alerts and sends email notifications via Microsoft Graph API when alerts are detected. It is designed for teams and administrators who want real-time alert email notifications directly in their inbox.

The workflow is logically divided into the following blocks:

- **1.1 Trigger & Data Retrieval:** Periodically triggers the workflow and fetches alert data from the PRISM Elastic API.
- **1.2 Response Validation:** Checks if the API response contains alert data.
- **1.3 Alert Processing Loop:** Iterates over each alert item retrieved.
- **1.4 Notification Dispatch:** Sends an email notification for each alert via Microsoft Graph API.
- **1.5 End Flow / No Action:** Handles cases where no alerts are present or marks the end of processing.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Data Retrieval

- **Overview:**  
  This block initiates the workflow on a schedule and retrieves alert data from the PRISM Elastic API endpoint.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Get Elastic Alert

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Periodically initiates the workflow based on a set interval.  
    - Configuration: Uses default interval settings (not specifically detailed, defaults to every minute or as configured).  
    - Inputs: None (trigger node)  
    - Outputs: Connects to "Get Elastic Alert"  
    - Edge Cases: Misconfiguration of interval could cause too frequent or infrequent triggers. Network downtime during trigger has no direct impact but limits workflow execution.  
    - Version: 1.2  

  - **Get Elastic Alert**  
    - Type: HTTP Request  
    - Role: Fetches alert data from the PRISM Elastic API.  
    - Configuration:  
      - URL set to `https://your-prism-elastic-api-endpoint.com/alerts` (to be customized).  
      - Method: GET (default in HTTP Request node).  
      - No authentication specified, implying either public API or credentials managed elsewhere.  
    - Inputs: Receives trigger from "Schedule Trigger"  
    - Outputs: Connects to "Response is not empty" node  
    - Edge Cases:  
      - API endpoint unreachable or returns error (timeout, 4xx/5xx).  
      - Empty or malformed response.  
      - Possible rate limits or authentication failures if applied.  
    - Version: 2  

#### 1.2 Response Validation

- **Overview:**  
  Validates whether the response from the PRISM Elastic API contains alert data to proceed.

- **Nodes Involved:**  
  - Response is not empty  

- **Node Details:**

  - **Response is not empty**  
    - Type: If  
    - Role: Checks if the API response contains alerts; routes workflow accordingly.  
    - Configuration: Condition not explicitly shown, but implied to check for non-empty response data.  
    - Inputs: From "Get Elastic Alert"  
    - Outputs:  
      - True path: To "Loop Over Each Alert Items"  
      - False path: To "No Operation, do nothing"  
    - Edge Cases:  
      - Incorrect condition expression could lead to false positives or negatives.  
      - Response format changes may break condition checks.  
    - Version: 2.1  

#### 1.3 Alert Processing Loop

- **Overview:**  
  Iterates over each alert item received to process them individually.

- **Nodes Involved:**  
  - Loop Over Each Alert Items  
  - No Operation, end of loop  

- **Node Details:**

  - **Loop Over Each Alert Items**  
    - Type: SplitInBatches  
    - Role: Processes alert items one-by-one (or in batches) for individual handling.  
    - Configuration: Default batching options (batch size unspecified, defaults to 1).  
    - Inputs: From "Response is not empty" (True path)  
    - Outputs:  
      - First output to "No Operation, end of loop" (likely used as a pass-through or loop management)  
      - Second output to "Send Email Notification"  
    - Edge Cases:  
      - Large number of alerts could increase runtime.  
      - Handling of empty or undefined items.  
    - Version: 3  

  - **No Operation, end of loop**  
    - Type: NoOp  
    - Role: Acts as a placeholder or end marker for loop iterations.  
    - Inputs: From "Loop Over Each Alert Items"  
    - Outputs: None  
    - Edge Cases: None  
    - Version: 1  

#### 1.4 Notification Dispatch

- **Overview:**  
  Sends an email notification for each alert item using Microsoft Graph API.

- **Nodes Involved:**  
  - Send Email Notification

- **Node Details:**

  - **Send Email Notification**  
    - Type: HTTP Request  
    - Role: Sends the alert notification email via Microsoft Graph API.  
    - Configuration:  
      - URL: `https://graph.microsoft.com/v1.0/me/sendMail`  
      - Method: POST  
      - Authentication: OAuth2, using Microsoft Graph credentials with Mail.Send permission.  
      - Body content type: JSON  
      - Email content dynamically populated using expressions referencing alert properties:  
        - Subject includes alert name.  
        - Body includes alert name, severity, timestamp, and alert message formatted in HTML.  
        - Recipient email address set to `user@example.com` (customizable).  
      - Saves sent email to Sent Items.  
    - Inputs: From second output of "Loop Over Each Alert Items"  
    - Outputs: To "Loop Over Each Alert Items" (implying feedback or continuation in loop)  
    - Edge Cases:  
      - OAuth2 token expiry or authentication failure.  
      - API rate limits or Microsoft Graph service issues.  
      - Malformed email data causing API rejection.  
    - Version: 2  
    - Credentials Required: Microsoft Graph OAuth2 with Mail.Send scope  

#### 1.5 End Flow / No Action

- **Overview:**  
  Handles cases when no alerts are received or to mark the workflow end after processing.

- **Nodes Involved:**  
  - No Operation, do nothing  

- **Node Details:**

  - **No Operation, do nothing**  
    - Type: NoOp  
    - Role: Terminates the workflow silently when no alerts are present.  
    - Inputs: From "Response is not empty" (False path)  
    - Outputs: None  
    - Edge Cases: None  
    - Version: 1  

---

### 3. Summary Table

| Node Name                 | Node Type          | Functional Role                    | Input Node(s)              | Output Node(s)                  | Sticky Note                                          |
|---------------------------|--------------------|----------------------------------|----------------------------|--------------------------------|-----------------------------------------------------|
| Schedule Trigger          | Schedule Trigger   | Periodic workflow initiation     | -                          | Get Elastic Alert              |                                                     |
| Get Elastic Alert          | HTTP Request       | Fetch alerts from PRISM Elastic  | Schedule Trigger           | Response is not empty          |                                                     |
| Response is not empty      | If                 | Check if alert data exists       | Get Elastic Alert          | Loop Over Each Alert Items (T)<br>No Operation, do nothing (F) |                                                     |
| Loop Over Each Alert Items | SplitInBatches     | Iterate over each alert          | Response is not empty (T)  | No Operation, end of loop<br>Send Email Notification |                                                     |
| No Operation, end of loop  | NoOp               | End of batch loop                | Loop Over Each Alert Items | -                              |                                                     |
| Send Email Notification    | HTTP Request       | Send alert email via MS Graph API | Loop Over Each Alert Items | Loop Over Each Alert Items     | Setup OAuth2 Credentials for Microsoft Graph API with Mail.Send permission |
| No Operation, do nothing   | NoOp               | End workflow if no alerts        | Response is not empty (F)  | -                              |                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node:**  
   - Type: Schedule Trigger  
   - Configuration: Set desired interval (e.g., every 5 minutes) to poll alerts.  
   - No credentials needed.  

2. **Create Get Elastic Alert Node:**  
   - Type: HTTP Request  
   - Connect input from Schedule Trigger.  
   - Set method to GET.  
   - Set URL to your PRISM Elastic API endpoint (e.g., `https://your-prism-elastic-api-endpoint.com/alerts`).  
   - Configure authentication if required (not specified here, but add if needed).  

3. **Create Response is not empty Node:**  
   - Type: If  
   - Connect input from Get Elastic Alert.  
   - Add condition to check if the response body contains alert data. For example, check if `{{$json.length}} > 0` or appropriate path depending on response structure.  

4. **Create No Operation, do nothing Node:**  
   - Type: NoOp  
   - Connect input from the False output of Response is not empty node.  
   - No configuration needed.  

5. **Create Loop Over Each Alert Items Node:**  
   - Type: SplitInBatches  
   - Connect input from True output of Response is not empty node.  
   - Set batch size to 1 (default) to process alerts one by one.  

6. **Create No Operation, end of loop Node:**  
   - Type: NoOp  
   - Connect input from first output of Loop Over Each Alert Items node (usually for loop management).  

7. **Create Send Email Notification Node:**  
   - Type: HTTP Request  
   - Connect input from second output of Loop Over Each Alert Items node.  
   - Set method to POST.  
   - Set URL to `https://graph.microsoft.com/v1.0/me/sendMail`.  
   - Under Authentication, select OAuth2 and configure credentials with Microsoft Graph API scope `Mail.Send`.  
   - Set `Body Content Type` to JSON.  
   - Enable `JSON/RAW Parameters`.  
   - Use the following JSON body, utilizing expressions to map alert data fields:

```json
{
  "message": {
    "subject": "PRISM Elastic Alert: {{$json[\"alert_name\"]}}",
    "body": {
      "contentType": "HTML",
      "content": "Hello,<br><br>An alert has been triggered:<br><strong>Alert Name:</strong> {{$json[\"alert_name\"]}}<br><strong>Severity:</strong> {{$json[\"severity\"]}}<br><strong>Timestamp:</strong> {{$json[\"timestamp\"]}}<br><br>Details:<br>{{$json[\"alert_message\"]}}<br><br>Regards,<br>PRISM Alert System"
    },
    "toRecipients": [
      {
        "emailAddress": {
          "address": "user@example.com"
        }
      }
    ]
  },
  "saveToSentItems": "true"
}
```

   - Replace `"user@example.com"` with the intended recipient email address(es).  
   - Credentials setup requires registering an Azure AD app with Mail.Send permission and configuring OAuth2 credentials in n8n accordingly.  

8. **Connect Send Email Notification output back to Loop Over Each Alert Items node:**  
   - This allows continued processing of batched alerts until complete.  

---

### 5. General Notes & Resources

| Note Content                                                                                                      | Context or Link                                                                                                  |
|------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Setup OAuth2 Credentials in n8n for Microsoft Graph API with Mail.Send permission is mandatory for email sending. | See Microsoft documentation for registering Azure AD app and OAuth2 setup: https://docs.microsoft.com/en-us/azure/active-directory/develop/quickstart-register-app |
| Customize the PRISM Elastic API endpoint URL to your environment in the HTTP Request node "Get Elastic Alert".  | Ensure API credentials and endpoint are securely managed.                                                       |
| Modify the email recipient(s) in the "Send Email Notification" node to route alerts to the correct users.        | Can be changed to multiple recipients by adding more objects to the "toRecipients" array.                       |
| Email content is HTML-formatted allowing rich text and formatting.                                               | Adjust as needed for corporate branding or additional alert details.                                            |
| Potential error sources include API downtime, OAuth token expiry, and malformed alert data.                      | Implement error handling or retry mechanisms externally if required.                                            |

---

This reference document provides a complete understanding of the PRISM Elastic Alert Notification workflow in n8n, enabling users to reproduce, customize, or troubleshoot the automation effectively.