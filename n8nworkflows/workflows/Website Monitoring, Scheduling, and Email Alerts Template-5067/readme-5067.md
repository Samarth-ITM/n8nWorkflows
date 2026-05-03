Website Monitoring, Scheduling, and Email Alerts Template

https://n8nworkflows.xyz/workflows/website-monitoring--scheduling--and-email-alerts-template-5067


# Website Monitoring, Scheduling, and Email Alerts Template

---

### 1. Workflow Overview

This workflow is designed for **automated website uptime monitoring** with email alert notifications when downtime is detected. It periodically checks the HTTP status of a specified website and sends an email alert if the website is not reachable or returns a non-200 HTTP status code. The primary use case is for IT teams, DevOps, or site administrators who want to be promptly informed about website outages.

The workflow logic is divided into three main blocks:

- **1.1 Scheduled Trigger:** Periodically initiates the website status check.
- **1.2 Website Status Check:** Sends an HTTP request to the target website to determine its availability.
- **1.3 Downtime Detection and Notification:** Evaluates the HTTP response and sends an email alert if the site is down.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  This block initiates the workflow execution every 5 minutes to ensure regular website monitoring.

- **Nodes Involved:**  
  - Schedule Website Check

- **Node Details:**

  - **Schedule Website Check**  
    - **Type:** Schedule Trigger  
    - **Role:** Starts the workflow at fixed time intervals.  
    - **Configuration:**  
      - Interval set to every 5 minutes.  
      - No additional filters or constraints.  
    - **Input:** None (trigger node).  
    - **Output:** Triggers the next node "Check Website Status".  
    - **Version Requirements:** n8n version supporting `scheduleTrigger` node (standard).  
    - **Potential Failures:**  
      - Workflow may not trigger if n8n instance is down or paused.  
      - Time zone discrepancies if n8n server is not configured properly.  
    - **Sub-workflows:** None.

#### 1.2 Website Status Check

- **Overview:**  
  This block performs an HTTP GET request to the target website to fetch its current status.

- **Nodes Involved:**  
  - Check Website Status

- **Node Details:**

  - **Check Website Status**  
    - **Type:** HTTP Request  
    - **Role:** Sends an HTTP GET request to check website availability.  
    - **Configuration:**  
      - URL: `https://yourdomain.com` (replace with the actual website URL).  
      - Method: GET.  
      - Response Format: JSON (expects JSON response, but primarily uses HTTP status code).  
      - No authentication or additional headers configured.  
    - **Input:** Triggered by "Schedule Website Check".  
    - **Output:** Passes HTTP response object, including `statusCode`, to the next node "Website Down?".  
    - **Version Requirements:** Standard HTTP Request node.  
    - **Potential Failures:**  
      - Network timeouts or connectivity issues.  
      - Non-JSON responses may cause parsing issues, but primarily status code is used.  
      - SSL certificate errors if the site uses HTTPS with invalid certs.  
      - If the site is slow, may hit timeout limits (not configured explicitly).  
    - **Sub-workflows:** None.

#### 1.3 Downtime Detection and Notification

- **Overview:**  
  This block analyzes the HTTP response status code to detect downtime and sends a formatted email alert if the website is down.

- **Nodes Involved:**  
  - Website Down?  
  - Send Downtime Email Alert

- **Node Details:**

  - **Website Down?**  
    - **Type:** If  
    - **Role:** Branches workflow based on whether the website is down.  
    - **Configuration:**  
      - Condition checks if HTTP `statusCode` is **not equal to 200**.  
      - Uses expression: `{{$json["statusCode"]}} != 200`  
    - **Input:** Receives HTTP response from "Check Website Status".  
    - **Output:**  
      - True branch (status code ≠ 200): proceeds to "Send Downtime Email Alert".  
      - False branch (status code = 200): workflow ends or could be extended.  
    - **Version Requirements:** Standard If node.  
    - **Potential Failures:**  
      - If `$json["statusCode"]` is undefined (e.g., HTTP request failed before response), expression evaluation may error.  
      - Misinterpretation if website returns other valid status codes (e.g., 3xx redirects).  
    - **Sub-workflows:** None.

  - **Send Downtime Email Alert**  
    - **Type:** Email Send  
    - **Role:** Sends an HTML-styled email to notify about website downtime.  
    - **Configuration:**  
      - From and To email addresses: `your@email.com` (to be replaced by actual addresses).  
      - Subject: "⚠️ Website Downtime Alert".  
      - HTML body contains styled notification with timestamp (`{{$now}}`) and link to the website.  
    - **Input:** Receives trigger from True branch of "Website Down?".  
    - **Output:** Ends workflow post sending email.  
    - **Version Requirements:** Requires configured email credentials in n8n (SMTP or other supported service).  
    - **Potential Failures:**  
      - Authentication errors if email credentials are invalid or not configured.  
      - Email sending limits or blocking by provider.  
      - Malformed HTML could affect email display.  
    - **Sub-workflows:** None.

---

### 3. Summary Table

| Node Name                 | Node Type          | Functional Role                      | Input Node(s)          | Output Node(s)               | Sticky Note                                                                                  |
|---------------------------|--------------------|------------------------------------|-----------------------|-----------------------------|----------------------------------------------------------------------------------------------|
| Schedule Website Check     | Schedule Trigger   | Initiates periodic workflow runs   | -                     | Check Website Status        |                                                                                              |
| Check Website Status       | HTTP Request      | Checks website HTTP status         | Schedule Website Check | Website Down?               |                                                                                              |
| Website Down?             | If                | Evaluates if website is down       | Check Website Status   | Send Downtime Email Alert   |                                                                                              |
| Send Downtime Email Alert | Email Send        | Sends alert email on downtime      | Website Down? (True)   | -                           | HTML email styled for clear alert; customize email addresses and website URL accordingly.    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n, name it e.g., "Website Downtime Monitoring with Email Alerts".

2. **Add the "Schedule Trigger" node**  
   - Type: Schedule Trigger  
   - Configure interval: Set to run every 5 minutes (minutesInterval = 5).  
   - Position: Start node.

3. **Add the "HTTP Request" node**  
   - Type: HTTP Request  
   - Connect input from "Schedule Trigger" node.  
   - Set URL to the website you want to monitor, e.g., `https://yourdomain.com`.  
   - Set method to GET.  
   - Set response format to JSON (default).  
   - Leave other options default unless you have specific needs (timeouts, headers).

4. **Add the "If" node**  
   - Type: If  
   - Connect input from "HTTP Request" node.  
   - Configure condition:  
     - Type: Number  
     - Operation: Not Equal  
     - Value 1: Expression - `{{$json["statusCode"]}}`  
     - Value 2: `200`  
   - This node will check if the website response status code is anything other than 200.

5. **Add the "Email Send" node**  
   - Type: Email Send  
   - Connect input from the True output of the If node.  
   - Configure email credentials in n8n settings (SMTP or any supported email provider).  
   - Set From Email and To Email fields to valid email addresses.  
   - Set Subject to "⚠️ Website Downtime Alert".  
   - Paste the provided HTML content into the Email body field. Adjust the link URL and email addresses as necessary.  
   - Use expression `{{$now}}` inside the HTML to insert current timestamp.

6. **Activate the workflow** after verification.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                          |
|----------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------|
| Replace placeholder URLs (`https://yourdomain.com`) and email addresses (`your@email.com`) with real values.         | Essential for proper functioning of the workflow.       |
| Ensure email credentials are configured properly in n8n for the Email Send node to work without errors.               | n8n documentation for Email node setup: https://docs.n8n.io/nodes/n8n-nodes-base.email-send/ |
| This workflow assumes the monitored website returns JSON or at least a valid HTTP status code accessible via `$json["statusCode"]`. | Modify HTTP Request node if your website returns non-JSON responses or requires authentication. |
| Customize the email HTML template to match branding or additional info if needed.                                    | The current template uses basic inline CSS styling.      |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---