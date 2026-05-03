Daily GitHub Release notification by Email

https://n8nworkflows.xyz/workflows/daily-github-release-notification-by-email-2590


# Daily GitHub Release notification by Email

### 1. Workflow Overview

This workflow automates daily notifications of the latest releases from a specified GitHub repository via email. It is designed for developers, project managers, or teams who want to stay informed of software updates without manually checking GitHub.

The workflow is logically divided into these blocks:

- **1.1 Daily Trigger:** Initiates the workflow once per day.
- **1.2 Fetch Repository Data:** Retrieves the latest release information from GitHub's API.
- **1.3 Release Date Check:** Determines if the latest release occurred within the last 24 hours.
- **1.4 Content Processing:** Extracts and processes the release notes from the API response, converting Markdown to HTML.
- **1.5 Email Notification:** Sends an email containing the formatted release notes.

This modular design allows easy customization and integration with other notification channels such as Slack or Gmail.

---

### 2. Block-by-Block Analysis

#### 1.1 Daily Trigger

- **Overview:**  
Initiates the workflow once every day to check for new GitHub releases.

- **Nodes Involved:**  
  - Daily Trigger

- **Node Details:**  
  - **Daily Trigger**  
    - *Type:* Schedule Trigger  
    - *Role:* Starts the workflow daily based on a fixed schedule.  
    - *Configuration:* Uses the default daily interval trigger.  
    - *Expressions/Variables:* None.  
    - *Input Connections:* None (start node).  
    - *Output Connections:* Connected to "Fetch Github Repo Releases" node.  
    - *Edge Cases:* Workflow won’t run if n8n is offline or paused at trigger time.  
    - *Version Requirements:* n8n version supporting scheduleTrigger v1.2.

---

#### 1.2 Fetch Repository Data

- **Overview:**  
Fetches the latest release details from the specified GitHub repository via GitHub API.

- **Nodes Involved:**  
  - Fetch Github Repo Releases  
  - Sticky Note (for user guidance on URL modification)

- **Node Details:**  
  - **Fetch Github Repo Releases**  
    - *Type:* HTTP Request  
    - *Role:* Calls GitHub API to get latest release info JSON from the repository.  
    - *Configuration:*  
      - URL set to `https://api.github.com/repos/n8n-io/n8n/releases/latest` (modifiable).  
      - No authentication configured (public API endpoint).  
    - *Expressions/Variables:* URL is static but intended for user modification (see Sticky Note).  
    - *Input Connections:* From Daily Trigger.  
    - *Output Connections:* To "If new release in the last day" node.  
    - *Edge Cases:*  
      - API rate limiting by GitHub (may cause 403 errors).  
      - Network timeouts or GitHub API downtime.  
      - Repository URL misconfiguration leading to 404 errors.  
    - *Version Requirements:* HTTP Request node v4.2 or higher.

  - **Sticky Note**  
    - *Type:* Sticky Note  
    - *Role:* Instructs user to change the GitHub repository URL in the HTTP Request node.  
    - *Content:* "Change **url** for Github Repo here"  
    - *Placement:* Near the "Fetch Github Repo Releases" node for visibility.

---

#### 1.3 Release Date Check

- **Overview:**  
Checks if the latest release was published within the last 24 hours to avoid redundant notifications.

- **Nodes Involved:**  
  - If new release in the last day

- **Node Details:**  
  - **If new release in the last day**  
    - *Type:* IF node  
    - *Role:* Conditional filter that passes data only if the release date is newer than 24 hours ago.  
    - *Configuration:*  
      - Condition compares the `.published_at` field from the GitHub API JSON response (converted to datetime) against current UTC time minus 1 day.  
      - Uses strict type validation and case sensitivity is true (although irrelevant here).  
    - *Expressions:*  
      - Left value: `{{$json.published_at.toDateTime()}}`  
      - Right value: `{{DateTime.utc().minus(1, 'days')}}`  
    - *Input Connections:* From "Fetch Github Repo Releases".  
    - *Output Connections:*  
      - If true: To "Split Out Content" node.  
      - If false: Workflow ends (no output connection).  
    - *Edge Cases:*  
      - If `.published_at` is missing or malformed, expression may fail.  
      - Timezone discrepancies handled by UTC usage.  
    - *Version Requirements:* IF node v2.2 supports advanced datetime operations.

---

#### 1.4 Content Processing

- **Overview:**  
Extracts the JSON body content, splits it out for processing, and converts release notes from Markdown to HTML for email formatting.

- **Nodes Involved:**  
  - Split Out Content  
  - Convert Markdown to HTML

- **Node Details:**  
  - **Split Out Content**  
    - *Type:* Split Out  
    - *Role:* Extracts the `body` field (release notes in Markdown) from the API response JSON to isolate content.  
    - *Configuration:*  
      - Field to split out: `body` (release notes in Markdown format).  
    - *Input Connections:* From "If new release in the last day" (true branch).  
    - *Output Connections:* To "Convert Markdown to HTML".  
    - *Edge Cases:*  
      - If `body` is missing or empty, downstream nodes may receive empty input.  
      - No error handling for non-string content.  
    - *Version Requirements:* Split Out node v1.

  - **Convert Markdown to HTML**  
    - *Type:* Markdown  
    - *Role:* Converts Markdown-formatted release notes into HTML for richer email display.  
    - *Configuration:*  
      - Mode: markdownToHtml  
      - Input Markdown: expression referencing `{{$json.body}}` from previous node.  
      - Output key: `html` (stores generated HTML).  
    - *Input Connections:* From "Split Out Content".  
    - *Output Connections:* To "Send Email".  
    - *Edge Cases:*  
      - If Markdown is malformed, HTML output may be incorrect but node usually handles gracefully.  
      - Empty or null Markdown results in empty HTML.  
    - *Version Requirements:* Markdown node v1.

---

#### 1.5 Email Notification

- **Overview:**  
Sends an email containing the formatted release notes HTML to a configured recipient.

- **Nodes Involved:**  
  - Send Email  
  - Sticky Note1 (for user guidance on modifying recipient email)

- **Node Details:**  
  - **Send Email**  
    - *Type:* Email Send  
    - *Role:* Sends an email with the latest release notes in HTML format.  
    - *Configuration:*  
      - Subject: "New n8n release" (static, can be customized).  
      - HTML content: Expression referencing `{{$json.html}}` from Markdown node.  
      - To Email: set to `email@example.com` by default (user must update).  
      - From Email: set to `email@example.com` by default (user must update).  
    - *Credentials:* Uses SMTP credential named "SMTP account - internal use only".  
    - *Input Connections:* From "Convert Markdown to HTML".  
    - *Output Connections:* None (end node).  
    - *Edge Cases:*  
      - SMTP misconfiguration causing authentication errors.  
      - Invalid email addresses causing send failure.  
      - HTML content injection or formatting issues (rare).  
    - *Version Requirements:* Email Send node v2.1 or higher.

  - **Sticky Note1**  
    - *Type:* Sticky Note  
    - *Role:* Instructs user to update the "To Email" address in the Send Email node.  
    - *Content:* "Change **to Email** here"  
    - *Placement:* Near "Send Email" node for user visibility.

---

### 3. Summary Table

| Node Name                   | Node Type             | Functional Role                             | Input Node(s)               | Output Node(s)                | Sticky Note                        |
|-----------------------------|-----------------------|---------------------------------------------|-----------------------------|------------------------------|----------------------------------|
| Daily Trigger               | Schedule Trigger      | Initiates workflow daily                     | None                        | Fetch Github Repo Releases    |                                  |
| Fetch Github Repo Releases  | HTTP Request          | Gets latest release data from GitHub API    | Daily Trigger               | If new release in the last day| Change **url** for Github Repo here |
| If new release in the last day | IF                   | Checks if release is within last 24 hours   | Fetch Github Repo Releases  | Split Out Content             |                                  |
| Split Out Content           | Split Out             | Extracts release notes from JSON body        | If new release in the last day | Convert Markdown to HTML      |                                  |
| Convert Markdown to HTML    | Markdown              | Converts release notes from Markdown to HTML| Split Out Content           | Send Email                   |                                  |
| Send Email                 | Email Send            | Sends release notes email                     | Convert Markdown to HTML    | None                        | Change **to Email** here          |
| Sticky Note                | Sticky Note           | User guidance: update GitHub URL             | None                        | None                        | Change **url** for Github Repo here |
| Sticky Note1               | Sticky Note           | User guidance: update recipient email        | None                        | None                        | Change **to Email** here           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Daily Trigger" node**  
   - Type: Schedule Trigger  
   - Set to trigger once per day (default daily interval).  
   - Position it as the starting node.

2. **Create "Fetch Github Repo Releases" node**  
   - Type: HTTP Request  
   - Configure URL: `https://api.github.com/repos/n8n-io/n8n/releases/latest` (replace with target repo URL).  
   - Leave Authentication blank (public endpoint).  
   - Connect output of "Daily Trigger" to this node's input.

3. **Add Sticky Note near HTTP Request node**  
   - Content: "Change **url** for Github Repo here"  
   - Purpose: Remind user to update repository URL.

4. **Create "If new release in the last day" node**  
   - Type: IF node  
   - Add condition:  
     - Left value: `{{$json.published_at.toDateTime()}}`  
     - Operator: After  
     - Right value: `{{DateTime.utc().minus(1, 'days')}}`  
   - Connect output of "Fetch Github Repo Releases" to this IF node.

5. **Create "Split Out Content" node**  
   - Type: Split Out  
   - Field to split out: `body`  
   - Connect IF node's "true" output (new release detected) to this node.

6. **Create "Convert Markdown to HTML" node**  
   - Type: Markdown  
   - Mode: markdownToHtml  
   - Markdown field: Expression `{{$json.body}}`  
   - Destination key: `html`  
   - Connect "Split Out Content" output to this node.

7. **Create "Send Email" node**  
   - Type: Email Send  
   - Subject: "New n8n release" (customize as needed).  
   - HTML content: Expression `{{$json.html}}`  
   - To Email: placeholder like `email@example.com` (update to actual recipient).  
   - From Email: placeholder like `email@example.com` (update as needed).  
   - Configure SMTP credentials (valid SMTP server with authentication).  
   - Connect "Convert Markdown to HTML" output to this node.

8. **Add Sticky Note near "Send Email" node**  
   - Content: "Change **to Email** here"  
   - Purpose: Remind user to update recipient email address.

9. **Validate and Activate Workflow**  
   - Test trigger and nodes stepwise.  
   - Confirm email delivery and proper formatting.

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                          |
|------------------------------------------------------------------------------|----------------------------------------------------------|
| This workflow template is modular and can be extended to send notifications via Slack or Gmail by replacing the final email node. | Workflow design best practice                              |
| GitHub API has rate limits for unauthenticated requests (60 per hour). For higher limits, consider using OAuth credentials. | https://docs.github.com/en/rest/overview/resources-in-the-rest-api#rate-limiting |
| Markdown to HTML conversion enhances readability in email clients supporting HTML. Plain text fallback is not implemented here. | Markdown node documentation in n8n                        |
| Ensure SMTP credentials are securely stored in n8n for email sending to avoid authentication failures. | Email node and credentials configuration                   |

---

This completes the exhaustive documentation of the "Daily GitHub Release notification by Email" n8n workflow.