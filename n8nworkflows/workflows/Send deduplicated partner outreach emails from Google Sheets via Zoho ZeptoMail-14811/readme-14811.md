Send deduplicated partner outreach emails from Google Sheets via Zoho ZeptoMail

https://n8nworkflows.xyz/workflows/send-deduplicated-partner-outreach-emails-from-google-sheets-via-zoho-zeptomail-14811


# Send deduplicated partner outreach emails from Google Sheets via Zoho ZeptoMail

# Workflow Analysis: Send deduplicated partner outreach emails from Google Sheets via Zoho ZeptoMail

### 1. Workflow Overview

This workflow is designed to automate a personalized cold email outreach campaign. It extracts lead data from a Google Sheet, cleans it to ensure no duplicates or invalid entries are processed, checks against a tracking sheet to prevent emailing the same person twice, and routes leads to specific email templates based on their "partner type" (College, Influencer, Brand, or Other). Finally, it sends the email via Zoho ZeptoMail and logs the action back into Google Sheets for tracking.

The logic is organized into the following functional blocks:
- **1.1 Trigger & Data Extraction:** Scheduled initiation and retrieval of lead data.
- **1.2 Data Validation & Cleaning:** Removal of duplicates and validation of email/name fields.
- **1.3 Deduplication & Sequence Check:** A looping mechanism that cross-references leads with a tracking sheet to avoid redundant outreach.
- **1.4 Personalized Routing:** Logic to assign specific email content based on the lead's category.
- **1.5 Delivery & Tracking:** Email dispatch via ZeptoMail and updating the tracking record.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Data Extraction
**Overview:** Initiates the process on a schedule and pulls the raw list of potential partners.
- **Nodes Involved:** `Schedule Trigger`, `Get row(s) in sheet`.
- **Node Details:**
    - **Schedule Trigger:** (Trigger) Fires the workflow based on a defined interval.
    - **Get row(s) in sheet:** (Google Sheets) Fetches all rows from a specified Spreadsheet ID and Sheet Name.
        - *Failure Type:* Authentication error or incorrect Resource ID.

#### 2.2 Data Validation & Cleaning
**Overview:** Ensures that only high-quality, unique leads proceed to the outreach phase.
- **Nodes Involved:** `Remove Duplicate Emails`, `Validate Email and Name`.
- **Node Details:**
    - **Remove Duplicate Emails:** (Remove Duplicates) Filters the list to keep only one instance of each unique email address.
    - **Validate Email and Name:** (Filter) Uses regex and length checks to ensure the `Email` field is a valid email format and the `Name` field is not empty.
        - *Key Expression:* `^[^\\s@]+@[^\\s@]+\\.[^\\s@]+$` (Regex for email validation).

#### 2.3 Deduplication & Sequence Check
**Overview:** Iterates through the validated leads and checks if they have already been contacted to avoid spamming.
- **Nodes Involved:** `Loop Over Items`, `Read Tracking Sheet`, `Skip If Already In Sequence`.
- **Node Details:**
    - **Loop Over Items:** (Split In Batches) Processes leads one by one to allow per-item tracking checks.
    - **Read Tracking Sheet:** (Google Sheets) Fetches the current status of all contacted leads.
    - **Skip If Already In Sequence:** (Code) A JavaScript node that compares the current lead's email against the tracking sheet.
        - *Logic:* If the email exists in the tracking sheet with a status of `sent`, `followup_1`, `completed`, `replied`, or `unsubscribed`, the lead is discarded.
        - *Output:* Returns the lead if they are eligible for outreach; otherwise, returns an empty array.

#### 2.4 Personalized Routing
**Overview:** Selects the appropriate messaging based on the partner's classification.
- **Nodes Involved:** `Route by Partner Type`, `College Email`, `Influencer Email`, `Brand Email`, `Other Email`.
- **Node Details:**
    - **Route by Partner Type:** (Switch) Routes the lead to different paths based on the `partner_type` value (College, Influencer, Brand).
    - **Email Template Nodes (Set):** Four separate `Set` nodes that define the `recipientEmail`, `recipientName`, `emailSubject`, and `emailBody` (HTML).
        - *Logic:* Each node contains a specific HTML template tailored to the partner category.
        - *Variable used:* `{{ $json['Name'].split(' ')[0] }}` is used to address the lead by their first name.

#### 2.5 Delivery & Tracking
**Overview:** Executes the email send and records the event for future reference.
- **Nodes Involved:** `Wait`, `Initial Email`, `Add to Tracking Sheet`.
- **Node Details:**
    - **Wait:** (Wait) Introduces a 10-second delay between sends to avoid rate-limiting.
    - **Initial Email:** (Zoho ZeptoMail) Sends the HTML email using the credentials and mail agent ID provided.
        - *Requirement:* Requires the `n8n-nodes-zohozeptomail` community node.
    - **Add to Tracking Sheet:** (Google Sheets) Appends or updates the lead in the tracking sheet.
        - *Columns updated:* `status` (set to "sent"), `sent_date`, `follow_up_date` (current date + 5 days), and `message_id`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Schedule Trigger | Schedule Trigger | Workflow Initiation | None | Get row(s) in sheet | Trigger & Validation |
| Get row(s) in sheet | Google Sheets | Lead Extraction | Schedule Trigger | Remove Duplicate Emails | Trigger & Validation |
| Remove Duplicate Emails | Remove Duplicates | Data Cleaning | Get row(s) in sheet | Validate Email and Name | Trigger & Validation |
| Validate Email and Name | Filter | Quality Control | Remove Duplicate Emails | Loop Over Items | Trigger & Validation |
| Loop Over Items | Split In Batches | Iteration Control | Validate Email and Name / Add to Tracking Sheet | Read Tracking Sheet | Trigger & Validation |
| Read Tracking Sheet | Google Sheets | History Retrieval | Loop Over Items | Skip If Already In Sequence | Trigger & Validation |
| Skip If Already In Sequence | Code | Sequence Deduplication | Read Tracking Sheet | Route by Partner Type | Trigger & Validation |
| Route by Partner Type | Switch | Category Routing | Skip If Already In Sequence | College/Influencer/Brand/Other Email | Outreach Logic |
| College Email | Set | Content Definition | Route by Partner Type | Wait | Outreach Logic |
| Influencer Email | Set | Content Definition | Route by Partner Type | Wait | Outreach Logic |
| Brand Email | Set | Content Definition | Route by Partner Type | Wait | Outreach Logic |
| Other Email | Set | Content Definition | Route by Partner Type | Wait | Outreach Logic |
| Wait | Wait | Rate Limit Prevention | College/Influencer/Brand/Other Email | Initial Email | Outreach Logic |
| Initial Email | Zoho ZeptoMail | Email Delivery | Wait | Add to Tracking Sheet | Outreach Logic |
| Add to Tracking Sheet | Google Sheets | Activity Logging | Initial Email | Loop Over Items | Outreach Logic |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger & Extraction:**
    - Create a **Schedule Trigger** node.
    - Add a **Google Sheets** node (`Get Many` operation) connected to your lead source sheet. Configure the Document and Sheet IDs.

2.  **Validation:**
    - Add a **Remove Duplicates** node; set "Fields to Compare" to `Email`.
    - Add a **Filter** node. Create two conditions: 
        - `Email` matches regex `^[^\\s@]+@[^\\s@]+\\.[^\\s@]+$`
        - `Name` length is greater than 0.

3.  **Sequence Logic:**
    - Add a **Split In Batches** node to handle leads individually.
    - Connect this to a **Google Sheets** node (`Get Many`) that reads your "Tracking Sheet".
    - Add a **Code** node with the provided JavaScript to compare the current lead email with the tracking sheet data, filtering out those with statuses like `sent` or `replied`.

4.  **Routing & Templates:**
    - Add a **Switch** node. Define four output paths based on the `partner_type` field containing "College", "Influencer", "Brand", or a fallback "Other".
    - For each path, create a **Set** node. Define variables for `recipientEmail`, `recipientName`, `emailSubject`, and `emailBody` (paste the HTML templates provided in the workflow).

5.  **Delivery & Closing:**
    - Route all four Set nodes into a **Wait** node (set to 10 seconds).
    - Connect the Wait node to a **Zoho ZeptoMail** node. Configure the `To Address`, `Subject`, and `HTML Body` using the expressions from the Set nodes. Set your `Mail Agent ID` and `From Address`.
    - Connect the email node to a **Google Sheets** node (`Append or Update`). Use `Email` as the matching column and map the status to `sent`, and the `follow_up_date` using the expression `{{ $now.plus({days: 5}).toISODate() }}`.
    - Close the loop by connecting the Tracking Sheet node back to the **Split In Batches** node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Workflow requires a self-hosted n8n instance to install community nodes. | Technical Requirement |
| Community node required: `n8n-nodes-zohozeptomail.zohoZeptomail` | Dependency |
| Replace all `[YOUR COMPANY]`, `[YOUR NAME]`, and `REPLACE_WITH_CALENDAR_LINK` placeholders in Set nodes. | Customization |
| Ensure the Google Tracking Sheet has columns: `Email`, `Name`, `partner_type`, `status`, `sent_date`, `follow_up_date`, `thread_id`, `message_id`, `original_subject`. | Sheet Structure |