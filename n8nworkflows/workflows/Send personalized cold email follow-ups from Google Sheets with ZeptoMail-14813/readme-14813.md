Send personalized cold email follow-ups from Google Sheets with ZeptoMail

https://n8nworkflows.xyz/workflows/send-personalized-cold-email-follow-ups-from-google-sheets-with-zeptomail-14813


# Send personalized cold email follow-ups from Google Sheets with ZeptoMail

# Technical Documentation: Personalized Cold Email Follow-ups Workflow

## 1. Workflow Overview
This workflow automates a multi-stage cold email outreach sequence. It monitors a Google Sheet for leads, identifies those due for a follow-up based on their current status and date, generates a personalized HTML email based on the lead's "partner type" (e.g., College, Influencer, Brand), sends the email via ZeptoMail, and updates the tracking sheet to schedule the next action.

The logic is divided into three primary functional blocks:
- **1.1 Trigger & Data Sync:** Handles the daily schedule and retrieval of lead data from Google Sheets.
- **1.2 Processing & Routing:** Filters leads by status and date, then routes them to the correct email template (Follow-up 1, 2, or 3).
- **1.3 Delivery & Updates:** Executes the email send through ZeptoMail and logs the result back into the Google Sheet.

---

## 2. Block-by-Block Analysis

### 2.1 Trigger & Data Sync
**Overview:** This block initiates the process and ensures only valid, active leads are processed.

- **Nodes Involved:** `Run Daily`, `Read Tracking Sheet`, `Filter Active Leads`, `Filter Due Today`.
- **Node Details:**
    - **Run Daily** (Schedule Trigger): Triggers the workflow on a daily interval.
    - **Read Tracking Sheet** (Google Sheets): Retrieves all rows from a specified spreadsheet. Requires a valid Document ID and Sheet Name.
    - **Filter Active Leads** (Filter): Removes leads that are no longer eligible.
        - *Conditions:* Status $\neq$ "completed", $\neq$ "replied", $\neq$ "unsubscribed", and Email is not empty.
    - **Filter Due Today** (Filter): Checks if the `follow_up_date` is today or in the past.
        - *Key Expression:* `{{ $json.follow_up_date }} <= {{ $now.toISODate() }}`.

### 2.2 Processing & Routing
**Overview:** Manages the sequence flow and generates the personalized content for each specific stage of the follow-up.

- **Nodes Involved:** `Loop Over Leads`, `Route by Follow-up Stage`, `Build Follow-up 1 Email`, `Build Follow-up 2 Email`, `Build Follow-up 3 Email`.
- **Node Details:**
    - **Loop Over Leads** (Split In Batches): Processes leads one by one to prevent API rate-limiting and ensure individual tracking updates.
    - **Route by Follow-up Stage** (Switch): Routes the lead based on the `status` field.
        - *Route 1:* `sent` $\rightarrow$ Follow-up 1.
        - *Route 2:* `followup_1` $\rightarrow$ Follow-up 2.
        - *Route 3:* `followup_2` $\rightarrow$ Follow-up 3.
    - **Build Follow-up X Email** (Code): JavaScript nodes that select a template based on `partner_type` (College, Influencer, Brand, or Default).
        - *Logic:* Generates dynamic HTML body and subject lines.
        - *Threading:* Uses `message_id` and `thread_id` to ensure the email appears as a reply in the recipient's inbox.
        - *Customization:* Includes a Calendly link and sender signature.

### 2.3 Delivery & Updates
**Overview:** Sends the final email and updates the source of truth (Google Sheet) to move the lead to the next stage.

- **Nodes Involved:** `Wait`, `Wait1`, `Wait2`, `Send Follow-up 1/2/3`, `Update Status → followup_1/2/completed`.
- **Node Details:**
    - **Wait Nodes** (Wait): Introduces a 10-second delay before sending to simulate natural pacing.
    - **Send Follow-up X** (Zoho ZeptoMail): Sends the generated HTML email.
        - *Config:* Uses `emailSubject`, `emailBody`, and `Email` address. Requires a specific Mail Agent ID.
    - **Update Status Nodes** (Google Sheets): Updates the lead row.
        - *Key Updates:* Changes `status` to the next stage, sets the new `sent_date`, and calculates the next `follow_up_date` (e.g., +5 days for FU1, +8 days for FU2).
        - *Matching Column:* Uses `Email` as the unique identifier to locate the row.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Run Daily | Schedule Trigger | Workflow Trigger | - | Read Tracking Sheet | Trigger & Data Sync |
| Read Tracking Sheet | Google Sheets | Data Retrieval | Run Daily | Filter Active Leads | Trigger & Data Sync |
| Filter Active Leads | Filter | Eligibility Check | Read Tracking Sheet | Filter Due Today | Trigger & Data Sync |
| Filter Due Today | Filter | Date Validation | Filter Active Leads | Loop Over Leads | Trigger & Data Sync |
| Loop Over Leads | Split In Batches | Batch Processing | Filter Due Today, Update Status nodes | Route by Follow-up Stage | Processing & Routing |
| Route by Follow-up Stage | Switch | Sequence Logic | Loop Over Leads | Build Email nodes | Processing & Routing |
| Build Follow-up 1 Email | Code | Content Generation | Route by Follow-up Stage | Wait | Processing & Routing |
| Build Follow-up 2 Email | Code | Content Generation | Route by Follow-up Stage | Wait1 | Processing & Routing |
| Build Follow-up 3 Email | Code | Content Generation | Route by Follow-up Stage | Wait2 | Processing & Routing |
| Wait / Wait1 / Wait2 | Wait | Rate Pacing | Build Email nodes | Send Follow-up nodes | Delivery & Updates |
| Send Follow-up 1/2/3 | Zoho ZeptoMail | Email Dispatch | Wait nodes | Update Status nodes | Delivery & Updates |
| Update Status $\rightarrow$ FU1 | Google Sheets | Sheet Logging | Send Follow-up 1 | Loop Over Leads | Delivery & Updates |
| Update Status $\rightarrow$ FU2 | Google Sheets | Sheet Logging | Send Follow-up 2 | Loop Over Leads | Delivery & Updates |
| Update Status $\rightarrow$ Comp | Google Sheets | Sheet Logging | Send Follow-up 3 | Loop Over Leads | Delivery & Updates |

---

## 4. Reproducing the Workflow from Scratch

### Step 1: Trigger & Data Retrieval
1. Create a **Schedule Trigger** set to execute daily.
2. Add a **Google Sheets** node (Operation: *Get Many*). Configure the Document ID and Sheet Name where your leads are stored.
3. Add a **Filter** node to exclude `status` values "completed", "replied", and "unsubscribed". Ensure "Email" is not empty.
4. Add a second **Filter** node to compare `follow_up_date` with the current date (`{{ $now.toISODate() }}`).

### Step 2: Iteration & Logic
5. Add a **Split In Batches** node to handle leads individually.
6. Add a **Switch** node. Create four outputs:
    - Output 0: `status` equals `sent`
    - Output 1: `status` equals `followup_1`
    - Output 2: `status` equals `followup_2`
    - Output 3: (Fallback) Loop back to Split In Batches.

### Step 3: Content Generation (Code)
7. For each Switch output (0, 1, 2), create a **Code** node.
8. Implement JavaScript logic to:
    - Define templates based on `partner_type`.
    - Construct an HTML body including a Calendly link.
    - Map `message_id` to `inReplyTo` and `thread_id` to `references` for email threading.

### Step 4: Execution & Feedback Loop
9. Add a **Wait** node (10 seconds) after each Code node.
10. Add a **Zoho ZeptoMail** node. Configure the "Mail Agent ID" and set "From Address". Map the subject and body from the Code node.
11. Add a **Google Sheets** node (Operation: *Update*). 
    - Set "Matching Column" to `Email`.
    - Update `status` to the next stage.
    - Calculate the next `follow_up_date` using an expression (e.g., `{{ $now.plus({days: 5}).toISODate() }}`).
    - Update `message_id` with the response from ZeptoMail.
12. Connect the final Update nodes back to the **Split In Batches** node to process the next lead.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Requires a self-hosted n8n instance to use community nodes. | Infrastructure Requirement |
| Community node `n8n-nodes-zohozeptomail.zohoZeptomail` must be installed. | Dependencies |
| Update `[YOUR COMPANY]`, `[YOUR NAME]`, and Calendly links inside the Code nodes. | Customization |
| Ensure Google Sheet contains columns: Name, Email, status, sent_date, thread_id, email_type, message_id, partner_type, follow_up_date. | Data Schema |