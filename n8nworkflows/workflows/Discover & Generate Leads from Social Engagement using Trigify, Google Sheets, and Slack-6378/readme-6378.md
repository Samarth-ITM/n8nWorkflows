Discover & Generate Leads from Social Engagement using Trigify, Google Sheets, and Slack

https://n8nworkflows.xyz/workflows/discover---generate-leads-from-social-engagement-using-trigify--google-sheets--and-slack-6378


# Discover & Generate Leads from Social Engagement using Trigify, Google Sheets, and Slack

### 1. Workflow Overview

This workflow automates the process of discovering and generating leads from social media engagement, specifically LinkedIn interactions tracked via Trigify.io. It captures new posts by thought leaders, logs these posts in Google Sheets, checks for duplicates, and alerts the team via Slack. It also identifies and records prospective engagers who match a target profile (e.g., marketing roles) into a separate Google Sheet for lead nurturing or outreach. The workflow is divided into key logical blocks:

- **1.1 Input Reception:** Receives webhook data from Trigify with LinkedIn engagement and prospect details.
- **1.2 Data Normalization:** Extracts and formats relevant fields from the webhook payload for posts and prospects.
- **1.3 Duplicate Check & Post Logging:** Checks whether a LinkedIn post is already logged in Google Sheets; if not, appends it and sends a Slack notification.
- **1.4 Prospect Filtering & Lead Logging:** Filters prospects based on job title (marketing-focused) and logs them into a dedicated Google Sheet.
- **1.5 Notifications:** Sends Slack alerts about new thought leader posts.
- **1.6 Safeguards:** Includes conditional checks and notes to prevent duplicates and ensure alignment with Ideal Customer Profile (ICP).

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
Receives incoming data from Trigify.io webhook POST requests containing LinkedIn engagement and prospect information.

- **Nodes Involved:**  
  - Webhook

- **Node Details:**  
  - **Webhook**  
    - Type: Webhook node  
    - Role: Entry point for the workflow; listens for POST requests at a specific path.  
    - Configuration: HTTP POST method, unique webhook path configured in Trigify integration.  
    - Input: External HTTP POST with JSON payload.  
    - Output: JSON data forwarded downstream.  
    - Potential Failures: Invalid payloads, webhook path misconfiguration, network issues.  
    - Notes: The webhook expects structured data with nested engagement and prospect details.

#### 2.2 Data Normalization

- **Overview:**  
Extracts and standardizes key fields from the incoming JSON payload to simplify downstream processing.

- **Nodes Involved:**  
  - Edit Fields  
  - Edit Fields1

- **Node Details:**  
  - **Edit Fields**  
    - Type: Set node  
    - Role: Extracts LinkedIn post URL, posted date, and post text from the nested engagement array inside the webhook payload.  
    - Configuration: Assigns specific JSON paths to simpler field names under `body.linkedin_engagement[0].linkedin_post`.  
    - Input: Webhook output JSON.  
    - Output: JSON with trimmed post-related fields.  
    - Edge Cases: Missing or malformed LinkedIn engagement data could cause empty or null assignments.  
  - **Edit Fields1**  
    - Type: Set node  
    - Role: Extracts prospect and company details from the webhook payload into flat fields for easier use.  
    - Configuration: Assigns first name, last name, job title, LinkedIn URLs, location, company name, size, industry, domain, etc.  
    - Input: Webhook output JSON (parallel to Edit Fields).  
    - Output: JSON with prospect and company data.  
    - Edge Cases: Missing fields or unexpected null values may require validation or default values.

#### 2.3 Duplicate Check & Post Logging

- **Overview:**  
Checks if the LinkedIn post URL already exists in the "Social Warming" Google Sheet; if not, appends the post data and triggers a Slack notification.

- **Nodes Involved:**  
  - Google Sheets1  
  - If  
  - Google Sheets  
  - Slack

- **Node Details:**  
  - **Google Sheets1**  
    - Type: Google Sheets node  
    - Role: Searches the "Social Warming" Google Sheet for existing rows matching the LinkedIn post URL to avoid duplicates.  
    - Configuration: Filter on "Post URL" column matching the extracted post URL.  
    - Input: Output from Edit Fields node.  
    - Output: Matching rows or empty if none found.  
    - Edge Cases: API rate limits, authentication errors, empty results.  
  - **If**  
    - Type: If node  
    - Role: Checks if the "Post URL" field returned by Google Sheets1 is not empty (i.e., duplicate exists).  
    - Configuration: Condition is "Post URL" not empty → means duplicate found.  
    - Input: Google Sheets1 output.  
    - Output:  
      - True branch: Duplicate exists → stops further processing (no connection from True branch).  
      - False branch: New post → continue to append and notify.  
  - **Google Sheets**  
    - Type: Google Sheets node  
    - Role: Appends new post data (URL, text, posted date) to the "Social Warming" sheet.  
    - Configuration: Defines columns with post details, sheet name "Sheet1", document ID for the specific Google Sheet.  
    - Input: False branch from If node (new post).  
    - Output: Confirmation of append operation.  
    - Edge Cases: Google Sheets API limits, authentication failures.  
  - **Slack**  
    - Type: Slack node  
    - Role: Sends a rich Slack block message alerting the team about the new LinkedIn post from a tracked thought leader.  
    - Configuration: Posts to a specific Slack channel, includes post URL, date, text, and action buttons ("View Post", "Engage Now", "Save for Later").  
    - Input: Output from Google Sheets append node.  
    - Edge Cases: Slack API rate limits, invalid channel ID, message formatting issues.

#### 2.4 Prospect Filtering & Lead Logging

- **Overview:**  
Filters prospects by job title containing "marketing" (customizable ICP condition) and appends qualifying prospects’ details to an "Engagers" Google Sheet.

- **Nodes Involved:**  
  - If1  
  - Google Sheets2

- **Node Details:**  
  - **If1**  
    - Type: If node  
    - Role: Checks if the prospect’s job title includes the string "marketing" (case sensitive).  
    - Configuration: Condition with string contains operator on `body.prospect.job_title`.  
    - Input: Output from Edit Fields1 node.  
    - Output:  
      - True branch: Matches ICP → continue.  
      - False branch: Does not match → stops here (no connection).  
    - Edge Cases: Case sensitivity may need adjustment; empty or missing job title fields.  
  - **Google Sheets2**  
    - Type: Google Sheets node  
    - Role: Appends filtered prospect and company data to the "Engagers" sheet in the same Google Sheet document.  
    - Configuration: Columns include first/last name, job title, LinkedIn URLs, location, company details, etc.  
    - Input: True branch from If1 node.  
    - Edge Cases: API limits, authentication errors.

#### 2.5 Notifications & Safeguards

- **Sticky Notes:**  
  - Provide contextual information and guidance for users to customize and understand workflow behavior.  
  - Highlight the purpose of alerting team members about new thought leader posts and new engagers.  
  - Remind users to edit the ICP filtering logic in the If1 node.  
  - Warn about the duplicate check preventing repeated posts in Google Sheets.  
  - Suggest alternative integrations such as CRM or outreach tools for the lead data.

---

### 3. Summary Table

| Node Name     | Node Type       | Functional Role                        | Input Node(s)         | Output Node(s)        | Sticky Note                                                                                     |
|---------------|-----------------|-------------------------------------|-----------------------|-----------------------|------------------------------------------------------------------------------------------------|
| Webhook       | Webhook         | Receives incoming webhook data       | —                     | Edit Fields, Edit Fields1 | ## Being alerted to new Thought Leader post from Trigify.io                                   |
| Edit Fields   | Set             | Extracts LinkedIn post data           | Webhook                | Google Sheets1          | This is stopping duplicate posts being added here.                                            |
| Edit Fields1  | Set             | Extracts prospect and company data    | Webhook                | If1                    | ## Being alerted to new engagers on that post from the Thought Leader from Trigify.io.        |
| Google Sheets1| Google Sheets   | Checks for duplicate LinkedIn posts   | Edit Fields             | If                     | This is stopping duplicate posts being added here.                                            |
| If            | If              | Filters out duplicate posts           | Google Sheets1          | Google Sheets (false branch) | This is stopping duplicate posts being added here.                                            |
| Google Sheets | Google Sheets   | Appends new LinkedIn post data        | If (false branch)       | Slack                   |                                                                                                |
| Slack         | Slack           | Sends Slack notification of new post | Google Sheets           | —                       |                                                                                                |
| If1           | If              | Filters prospects by job title (ICP)  | Edit Fields1            | Google Sheets2 (true branch) | Edit this IF for your ICP.                                                                     |
| Google Sheets2| Google Sheets   | Appends qualified prospects to sheet | If1 (true branch)       | —                       | You could use a CRM here instead or send this data through to an outreach tool.                |
| Sticky Note   | Sticky Note     | Informational notes                   | —                       | —                       | ## Being alerted to new Thought Leader post from Trigify.io                                   |
| Sticky Note1  | Sticky Note     | Informational notes                   | —                       | —                       | ## Being alerted to new engagers on that post from the Thought Leader from Trigify.io.        |
| Sticky Note2  | Sticky Note     | Informational notes                   | —                       | —                       | Edit this IF for your ICP.                                                                     |
| Sticky Note3  | Sticky Note     | Informational notes                   | —                       | —                       | This is stopping duplicate posts being added here.                                            |
| Sticky Note4  | Sticky Note     | Informational notes                   | —                       | —                       | You could use a CRM here instead or send this data through to an outreach tool.                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**
   - Type: Webhook  
   - Configure HTTP Method: POST  
   - Set unique webhook path (e.g., a UUID or meaningful string)  
   - No authentication required unless desired  
   - Purpose: Receive JSON payload from Trigify.io or similar source.

2. **Create Set Node "Edit Fields"**
   - Connect Webhook main output to this node.  
   - Configure assignments:  
     - `body.linkedin_engagement[0].linkedin_post.post_url` → assign to `body.linkedin_engagement[0].linkedin_post.post_url` (string)  
     - `body.linkedin_engagement[0].linkedin_post.posted_date` → same path (string)  
     - `body.linkedin_engagement[0].linkedin_post.text` → same path (string)  
   - Purpose: Extract LinkedIn post details for logging.

3. **Create Set Node "Edit Fields1"**
   - Also connect Webhook main output (parallel branch) to this node.  
   - Configure assignments for:  
     - Prospect details: first_name, last_name, job_title, linkedin_url, location_name  
     - Company details: name, size, industry, linkedin_url, domain  
   - Purpose: Flatten prospect and company info for filtering and logging.

4. **Create Google Sheets Node "Google Sheets1"**
   - Connect output of "Edit Fields" node.  
   - Operation: Read rows with filter  
   - Sheet name: "Sheet1" (or appropriate tab)  
   - Document ID: Use desired Google Sheet ID  
   - Filter: "Post URL" column equals `{{ $json.body.linkedin_engagement[0].linkedin_post.post_url }}`  
   - Purpose: Check if post URL already exists.

5. **Create If Node "If"**
   - Connect "Google Sheets1" output.  
   - Condition: Check if "Post URL" is not empty (meaning duplicate found).  
   - True branch: No further action (no connections).  
   - False branch: Continue to append new post.  
   - Purpose: Prevent duplicate post entries.

6. **Create Google Sheets Node "Google Sheets"**
   - Connect False branch from "If".  
   - Operation: Append  
   - Sheet name: "Sheet1"  
   - Document ID: Same as above  
   - Columns: Map post URL, post text, posted date from `Edit Fields` node output.  
   - Purpose: Log new LinkedIn post.

7. **Create Slack Node "Slack"**
   - Connect output from "Google Sheets" append operation.  
   - Configure Slack OAuth credentials.  
   - Channel: Set Slack channel ID for notifications.  
   - Message Type: Block  
   - Message content: Use Slack Block Kit JSON to include:  
     - Alert text about thought leader post  
     - Post URL as a clickable link  
     - Posted date  
     - Post content  
     - Buttons for "View Post", "Engage Now", "Save for Later" with appropriate URLs or actions  
   - Purpose: Notify team of new post.

8. **Create If Node "If1"**
   - Connect output from "Edit Fields1" (prospect data).  
   - Condition: Check if `body.prospect.job_title` contains "marketing" (case sensitive).  
   - True branch: Continue to lead logging.  
   - False branch: No connection (stop).  
   - Purpose: Filter prospects to target ICP.

9. **Create Google Sheets Node "Google Sheets2"**
   - Connect True branch from "If1".  
   - Operation: Append  
   - Sheet name: "Engagers" (or desired tab)  
   - Document ID: Same Google Sheet as other nodes  
   - Columns: Map first name, last name, job title, prospect LinkedIn URL, location, company name, domain, company LinkedIn URL, size, industry.  
   - Purpose: Log qualified prospect leads.

10. **Add Sticky Notes**
    - Add multiple Sticky Note nodes at logical points with content from original workflow to provide contextual guidance and reminders.

11. **Credential Setup**
    - Configure Google Sheets OAuth2 credentials with appropriate access to the Google Sheet document.  
    - Configure Slack OAuth credentials with permission to post messages to the target Slack channel.

12. **Testing & Validation**
    - Test webhook with sample payloads resembling Trigify output.  
    - Verify duplicate detection logic by re-sending identical posts.  
    - Validate Slack message formatting.  
    - Adjust ICP filtering conditions in If1 node as necessary.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                  |
|-------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| The workflow is designed around Trigify.io’s webhook data schema for LinkedIn engagement.       | Trigify.io documentation or support                             |
| Slack notification uses Block Kit formatting for rich interactive messages.                      | https://api.slack.com/block-kit                                 |
| Google Sheets API limits should be considered if scaling usage.                                  | https://developers.google.com/sheets/api/limits                 |
| Customize the ICP filter in the "If1" node to match your ideal customer profile keywords.        | Important for targeting correct leads                           |
| You may replace Google Sheets with a CRM or outreach tool integration for better lead management.| Suggested in sticky notes                                        |
| Sticky notes provide helpful context and should be preserved when modifying the workflow.        | Useful for maintaining workflow clarity                          |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and publicly available.