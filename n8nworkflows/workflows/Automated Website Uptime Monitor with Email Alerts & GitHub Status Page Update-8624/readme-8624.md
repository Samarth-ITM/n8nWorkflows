Automated Website Uptime Monitor with Email Alerts & GitHub Status Page Update

https://n8nworkflows.xyz/workflows/automated-website-uptime-monitor-with-email-alerts---github-status-page-update-8624


# Automated Website Uptime Monitor with Email Alerts & GitHub Status Page Update

### 1. Workflow Overview

This workflow, titled **Automated Website Uptime Monitor with Email Alerts & GitHub Status Page Update**, is designed to continuously monitor the uptime status of a specified website by pinging its health endpoint every 2 minutes. It performs three main functions:

- **Monitoring:** Periodic HTTP requests check the website’s health status.
- **Alerting:** If the website is down (non-200 status or error), an HTML-formatted email alert is sent.
- **Status Page Update:** It dynamically generates and updates a GitHub-hosted status page (`index.html`) reflecting the website’s current status (Up/Down).

The workflow is logically divided into these blocks:

- **1.1 Schedule Trigger & HTTP Request:** Periodic initiation and website status check.
- **1.2 Status Evaluation & Notification:** Decision making based on HTTP response; sending email alerts if down.
- **1.3 Dynamic HTML Generation:** Creating the status page HTML content based on site status.
- **1.4 GitHub Status Page Management:** Comparing the generated status page with the existing one in GitHub, and updating it only if changes are detected.

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule Trigger & HTTP Request

- **Overview:** This block triggers the workflow every 2 minutes and sends an HTTP request to the target website’s health endpoint to check uptime.
- **Nodes Involved:** 
  - Schedule Trigger
  - HTTP Request
  - Switch - status code

- **Node Details:**

  - **Schedule Trigger**
    - Type: Schedule Trigger
    - Role: Initiates the workflow automatically at fixed intervals.
    - Configuration: Set to run every 2 minutes.
    - Inputs: None (trigger node).
    - Outputs: Connected to the HTTP Request node.
    - Potential Failures: None (internal scheduling).
  
  - **HTTP Request**
    - Type: HTTP Request
    - Role: Pings the website health URL to fetch the status.
    - Configuration: URL set to `https://app.yourdomain.com/health` (user must replace with their URL); configured to return full HTTP response including status code.
    - Inputs: Triggered by Schedule Trigger.
    - Outputs: Connected to Switch node.
    - Error handling: On HTTP errors or bad responses, continues workflow without failure.
    - Edge Cases: Network errors, DNS failures, timeouts, non-200 responses.
  
  - **Switch - status code**
    - Type: Switch
    - Role: Evaluates HTTP response status to branch workflow.
    - Configuration: Checks for status code 200 (Up) and error 503 (Down). Outputs two paths:
      - Output 1: status code 503 or error → considered Down.
      - Output 2: status code 200 → considered Up.
    - Inputs: From HTTP Request node.
    - Outputs: Two branches; one branch triggers email alert + HTML generation, the other only triggers HTML generation.
    - Edge Cases: Unexpected status codes not explicitly handled (will fall through to default or be ignored).

#### 1.2 Status Evaluation & Notification

- **Overview:** Based on the switch decision, sends an alert email if the site is down and proceeds to generate the status page HTML.
- **Nodes Involved:**
  - Send a notification mail
  - Template HTML Code (common node for both up/down paths)

- **Node Details:**

  - **Send a notification mail**
    - Type: Gmail node
    - Role: Sends an HTML-formatted email alert when the site is down.
    - Configuration:
      - Recipient: `example@gmail.com` (user must replace).
      - Subject: "Server Down".
      - Message: Pre-styled HTML email showing status "Down", status code (503), and error message (`ERR_BAD_RESPONSE`).
    - Inputs: Connected from Switch node's "Down" output.
    - Outputs: None (terminal for alert branch).
    - Edge Cases: Gmail authentication errors, sending quota limits, network issues.

  - **Template HTML Code**
    - Type: Code node (JavaScript)
    - Role: Dynamically creates the HTML content for the status page based on site status.
    - Configuration:
      - Generates green “Server Up” page if HTTP 200.
      - Generates red “Server Down” page with error details if HTTP error.
      - Converts HTML string to Base64 binary data for GitHub upload.
    - Inputs: Both outputs from Switch node feed here.
    - Outputs: Passes generated HTML binary data to next block.
    - Edge Cases: Expression errors if input data structure changes.

#### 1.3 GitHub Status Page Management - Fetch & Compare

- **Overview:** Retrieves the existing `index.html` from GitHub, extracts its content, compares it with the newly generated HTML, and decides whether to update the file.
- **Nodes Involved:**
  - Extract from File - Generated HTML
  - Get existing index.html file
  - Extract from existing File (github)
  - If - Compare existing HTML file with generated HTML
  - Update Index.html file

- **Node Details:**

  - **Extract from File - Generated HTML**
    - Type: Extract from File
    - Role: Extracts text content from the binary HTML generated by the Code node.
    - Inputs: From Template HTML Code.
    - Outputs: Sends extracted text to GitHub file retrieval node.
    - Edge Cases: Extraction failure if input binary is malformed.

  - **Get existing index.html file**
    - Type: GitHub node
    - Role: Retrieves current `index.html` file from GitHub repository.
    - Configuration:
      - Owner URL: `https://github.com/<OWNER_NAME>` (user must replace).
      - Repository: `status` (default).
      - File path: `index.html`.
      - Outputs file binary data under `github_data`.
    - Inputs: From extracted generated HTML text.
    - Outputs: To extraction node.
    - Edge Cases: GitHub authentication failures, repository/file missing, rate limits.

  - **Extract from existing File (github)**
    - Type: Extract from File
    - Role: Extracts text content from the binary data of existing `index.html`.
    - Inputs: From GitHub file retrieval.
    - Outputs: Sends extracted text for comparison.
    - Edge Cases: Extraction failure if file corrupted.

  - **If - Compare existing HTML file with generated HTML**
    - Type: If
    - Role: Compares the extracted existing HTML content with the newly generated content.
    - Configuration: Compares strings for exact equality.
    - Inputs: From extracted GitHub file content and generated HTML content.
    - Outputs:
      - True: If contents are identical → no update needed; workflow ends.
      - False: If contents differ → triggers file update.
    - Edge Cases: Minor differences (whitespace, encoding) may trigger unnecessary updates.

  - **Update Index.html file**
    - Type: GitHub node
    - Role: Commits the newly generated `index.html` file to the GitHub repository.
    - Configuration:
      - Owner URL and repo same as above.
      - File path: `index.html`.
      - Uses binary data from Code node (Base64).
      - Commit message: "test" (user configurable).
    - Inputs: From If node when content differs.
    - Outputs: None (terminal node).
    - Edge Cases: GitHub API rate limits, permission issues, commit conflicts.

---

### 3. Summary Table

| Node Name                            | Node Type              | Functional Role                            | Input Node(s)                     | Output Node(s)                          | Sticky Note                                                                                          |
|------------------------------------|------------------------|------------------------------------------|----------------------------------|----------------------------------------|----------------------------------------------------------------------------------------------------|
| Schedule Trigger                   | Schedule Trigger        | Initiates workflow every 2 minutes       | None                             | HTTP Request                          | Runs automatically every 2 minutes; acts as heartbeat of the workflow; configurable interval.       |
| HTTP Request                      | HTTP Request           | Pings website health endpoint             | Schedule Trigger                 | Switch - status code                   | Sends HTTP request to `https://app.yourdomain.com/health`; returns full response; error-tolerant.  |
| Switch - status code              | Switch                 | Decides path based on HTTP status code   | HTTP Request                    | Template HTML Code, Send a notification mail | Routes based on status 200 (Up) or 503/error (Down).                                                |
| Send a notification mail          | Gmail                  | Sends HTML email alert on downtime       | Switch - status code             | None                                  | Sends alert to `example@gmail.com`; customizable; includes HTML alert page.                         |
| Template HTML Code                | Code                   | Generates dynamic HTML status page        | Switch - status code             | Extract from File - Generated HTML    | Creates green (Up) or red (Down) status page HTML; encodes as Base64 binary.                         |
| Extract from File - Generated HTML | Extract from File       | Extracts text from generated HTML binary | Template HTML Code               | Get existing index.html file           | Prepares HTML text for comparison/update.                                                           |
| Get existing index.html file      | GitHub                  | Fetches current status page from GitHub  | Extract from File - Generated HTML | Extract from existing File (github)   | Retrieves `index.html` from GitHub repo `<OWNER_NAME>/status`.                                     |
| Extract from existing File (github) | Extract from File       | Extracts text from GitHub’s binary file  | Get existing index.html file     | If - Compare existing HTML file with generated HTML | Prepares existing file content for comparison.                                                      |
| If - Compare existing HTML file with generated HTML | If                      | Compares existing and generated HTML     | Extract from existing File (github), Extract from File - Generated HTML | Update Index.html file (if different)      | Updates GitHub file only if content differs.                                                        |
| Update Index.html file            | GitHub                  | Commits new status page to GitHub repo   | If - Compare existing HTML file with generated HTML | None                                  | Commits updated `index.html` to GitHub repo; commit message "test".                                 |
| Sticky Note                      | Sticky Note             | Informational notes for user              | None                             | None                                  | Various sticky notes provide documentation, instructions, and tips for configuration and usage.    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**
   - Type: Schedule Trigger
   - Set interval to every 2 minutes (minutesInterval = 2).
   - Position early in the flow.

2. **Add an HTTP Request node:**
   - Connect input from Schedule Trigger.
   - Set HTTP method to GET.
   - Enter URL of your website health endpoint (e.g., `https://app.yourdomain.com/health`).
   - Enable returning full response data including status code.
   - Set error handling to continue workflow on error.
   
3. **Add a Switch node named "Switch - status code":**
   - Connect input from HTTP Request node.
   - Create two rules:
     - Rule 1: Check if `$json.error.status` equals 503 → output key "503".
     - Rule 2: Check if `$json.statusCode` equals 200 → output key "200".
   - Configure outputs accordingly.

4. **Add a Gmail node "Send a notification mail":**
   - Connect input from Switch node's "503" output (Down path).
   - Configure Gmail credentials (OAuth2 or API keys).
   - Set recipient email to your alert address.
   - Subject: "Server Down".
   - Message: Paste the provided HTML alert template showing red status and error details.
   - Optional: Customize message content.

5. **Add a Code node "Template HTML Code":**
   - Connect inputs from both Switch node outputs (Down and Up).
   - Paste the JavaScript code that dynamically generates the HTML status page as Base64 binary.
   - This code should:
     - Check input for status code or error.
     - Generate green "Up" page or red "Down" page accordingly.
     - Encode HTML as Base64 for GitHub upload.

6. **Add Extract from File node "Extract from File - Generated HTML":**
   - Connect input from Code node.
   - Configure to extract text from the binary HTML data.
   
7. **Add GitHub node "Get existing index.html file":**
   - Connect input from the Extract from File node.
   - Configure GitHub credentials (Personal Access Token with repo permissions).
   - Set:
     - Owner: `https://github.com/<OWNER_NAME>` (replace with your GitHub username).
     - Repository: `status` (or your repo name).
     - File Path: `index.html`.
     - Binary property name: `github_data`.
   - Operation: Get file.

8. **Add Extract from File node "Extract from existing File (github)":**
   - Connect input from GitHub get node.
   - Extract text from binary property `github_data`.

9. **Add If node "If - Compare existing HTML file with generated HTML":**
   - Connect input from the Extract from existing File node.
   - Set condition: Compare extracted existing HTML (`$json.github_data`) with generated HTML (`$('Extract from File - Generated HTML').item.json.data`).
   - Use string equality.
   - If true (same content), do nothing further.
   - If false (content differs), proceed to update file.

10. **Add GitHub node "Update Index.html file":**
    - Connect input from If node's false output.
    - Set operation to edit file.
    - Use same GitHub credentials and repo details as before.
    - File path: `index.html`.
    - Use binary data from Code node (Base64 encoded HTML).
    - Commit message: Set to "test" or customize.
    - This node commits the updated status page to GitHub.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Context or Link                                                                                                                         |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------|
| This workflow runs automatically every 2 minutes but the interval is configurable to seconds, minutes, hours, or days, depending on monitoring needs.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Sticky Note near Schedule Trigger                                                                                                     |
| The HTTP Request node is configured to always output data even on error, which ensures the workflow does not break if the website is down or unreachable.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Sticky Note near HTTP Request                                                                                                         |
| Email alerts are sent using Gmail node; replace the example email with your own or a team distribution list. The email contains an HTML page styled with red color indicating downtime status.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Sticky Note near Send a notification mail                                                                                             |
| The GitHub repository should be pre-created with a blank `index.html` file to allow updates. The repo owner, name, and file path need to be configured in the GitHub nodes accordingly.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Sticky Note near GitHub nodes                                                                                                         |
| The Code node generates the status page HTML dynamically with two modes: “Up” (green) and “Down” (red). You can customize the HTML and CSS inside this node for branding or additional details.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Sticky Note near Template HTML Code                                                                                                  |
| The workflow only commits the new status page to GitHub if there is an actual change detected in content compared to the existing file, preventing unnecessary commits.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Sticky Note near If node comparing HTML files                                                                                        |
| GitHub Pages can be used to host the `index.html` status page publicly. After enabling Pages on your repository, the status page will be accessible via a URL like `https://<USERNAME>.github.io/status`.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Additional instructions in main sticky note (Sticky Note3)                                                                           |
| Required credentials include GitHub personal access token with repo permissions and Gmail credentials for sending emails.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Main sticky note (Sticky Note3)                                                                                                      |
| The workflow can be extended to monitor multiple websites by duplicating the HTTP Request and Switch nodes accordingly and managing separate status pages or combined reporting.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Main sticky note (Sticky Note3)                                                                                                      |

---

_Disclaimer: The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public._