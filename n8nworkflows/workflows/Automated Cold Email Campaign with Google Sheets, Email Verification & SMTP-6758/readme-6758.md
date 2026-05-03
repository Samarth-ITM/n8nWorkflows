Automated Cold Email Campaign with Google Sheets, Email Verification & SMTP

https://n8nworkflows.xyz/workflows/automated-cold-email-campaign-with-google-sheets--email-verification---smtp-6758


# Automated Cold Email Campaign with Google Sheets, Email Verification & SMTP

### 1. Workflow Overview

This workflow automates a cold/warm email outreach campaign by integrating Google Sheets for lead management, verifying emails via Hunter.io API, and sending emails via SMTP. It is designed for marketers or sales teams who want to automate personalized email outreach while ensuring email validity to reduce bounce rates.

Logical blocks:

- **1.1 Trigger & Lead Retrieval:** Initiates workflow manually or on schedule, retrieves lead data from Google Sheets.
- **1.2 Lead Filtering:** Checks if email was already sent or if the email field is empty to avoid duplicate or invalid sends.
- **1.3 Email Verification:** Validates email deliverability using Hunter.io to filter out undeliverable addresses.
- **1.4 Email Sending:** Sends the outreach email via SMTP if the email is verified.
- **1.5 Post-Send Update:** Marks the lead as "Sent" in a Google Sheet to track outreach status.
- **1.6 Informational Sticky Notes:** Embedded documentation and resource links for users.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Lead Retrieval

**Overview:**  
This block triggers the workflow execution either manually or via schedule and fetches lead data from a Google Sheet.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Schedule Trigger at 9 AM (Scheduled Trigger)  
- Lead Sheet - Get Rows from the sheet (Google Sheets Read)  

**Node Details:**  

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Allows manual initiation of the workflow for testing or one-time runs.  
  - Config: No parameters, simple trigger node.  
  - Inputs: None  
  - Outputs: Leads to "Lead Sheet - Get Rows from the sheet"  
  - Edge cases: None significant; manual trigger may be forgotten or misused.

- **Schedule Trigger at 9 AM**  
  - Type: Schedule Trigger  
  - Role: Automates daily execution at 9 AM.  
  - Config: Set to trigger exactly at hour 9 daily.  
  - Inputs: None  
  - Outputs: No direct connection shown in JSON but logically triggers the workflow similarly to manual trigger (likely connected off-JSON).  
  - Edge cases: Timezone mismatches or missed executions if n8n instance is down.

- **Lead Sheet - Get Rows from the sheet**  
  - Type: Google Sheets  
  - Role: Retrieves lead rows from a specific Google Sheet document and sheet (Sheet1).  
  - Config: Document ID and sheet GID specified pointing to leads data. Uses default options.  
  - Inputs: Manual trigger or schedule trigger output.  
  - Outputs: Data passed to "Checker - If Already 'Sent' or not".  
  - Edge cases: Google API errors, auth failures, empty sheet, or malformed sheet data.

---

#### 1.2 Lead Filtering

**Overview:**  
Filters leads to skip those who have already been emailed or have empty email fields.

**Nodes Involved:**  
- Checker - If Already "Sent" or not (If)  
- No Operation, Do Nothing (NoOp)  
- Checker - If Lead's "Email" Field is empty? (If)  
- No Operation, Do Nothing1 (NoOp)  

**Node Details:**  

- **Checker - If Already "Sent" or not**  
  - Type: If node  
  - Role: Checks if the "Email Sent" field in lead row contains "Sent" to avoid duplicates.  
  - Config: Condition checks if "Email Sent" includes "Sent".  
  - Inputs: Lead rows from Google Sheets.  
  - Outputs: Two branches:  
    - True: Leads to No Operation (skip sending)  
    - False: Leads to next check for empty email.  
  - Edge cases: Case sensitivity or partial matches might cause logic slips.

- **No Operation, Do Nothing**  
  - Type: NoOp  
  - Role: Placeholder to terminate processing for leads already emailed.  
  - Inputs: True branch from prior If node.  
  - Outputs: None  
  - Edge cases: None.

- **Checker - If Lead's "Email" Field is empty?**  
  - Type: If node  
  - Role: Checks if the Email field is empty to skip invalid entries.  
  - Config: Condition tests if "Email" is empty string.  
  - Inputs: False branch from previous If node.  
  - Outputs:  
    - True (empty): No Operation, Do Nothing1  
    - False (not empty): Proceeds to email verification.  
  - Edge cases: Blank spaces or malformed emails not caught here.

- **No Operation, Do Nothing1**  
  - Type: NoOp  
  - Role: Ends processing for leads with empty email.  
  - Inputs: True branch from prior If.  
  - Outputs: None  
  - Edge cases: None.

---

#### 1.3 Email Verification

**Overview:**  
Verifies the deliverability of the lead’s email using the Hunter.io email verification API.

**Nodes Involved:**  
- Email verification using Hunter (Hunter)  
- Checker - If emails are verified or no? (If)  
- No Operation, do nothing5 (NoOp)  

**Node Details:**  

- **Email verification using Hunter**  
  - Type: Hunter node (email verifier)  
  - Role: Calls Hunter.io API to verify if email is deliverable.  
  - Config: Uses email field from lead row as input.  
  - Inputs: Leads passing empty email filter.  
  - Outputs: Passes verification result to If node.  
  - Edge cases: API quota exceeded, network errors, invalid API key, false negatives.

- **Checker - If emails are verified or no?**  
  - Type: If node  
  - Role: Checks if Hunter API result contains "undeliverable".  
  - Config: Condition checks if result string contains "undeliverable".  
  - Inputs: Hunter verification output.  
  - Outputs:  
    - True (undeliverable): No Operation, do nothing5 (skip sending)  
    - False (deliverable): Proceeds to send email.  
  - Edge cases: Result parsing errors, API format changes.

- **No Operation, do nothing5**  
  - Type: NoOp  
  - Role: Ends processing for undeliverable emails.  
  - Inputs: True branch from If node.  
  - Outputs: None.

---

#### 1.4 Email Sending

**Overview:**  
Sends a personalized email via SMTP to verified leads.

**Nodes Involved:**  
- Send email using SMTP (Email Send)  
- Creating Field for Updating "Sent" in G Sheet1 (Set)  

**Node Details:**  

- **Send email using SMTP**  
  - Type: Email Send  
  - Role: Sends outreach email to lead via SMTP.  
  - Config:  
    - Subject: "Let's collaborate!"  
    - Body: Plain text with dynamic insertion of "Tool Name" from Google Sheet lead row.  
    - To: Lead's email field.  
    - From: support@dolphy.io (fixed)  
    - Email format: Text  
  - Inputs: Leads verified deliverable by Hunter.  
  - Outputs: Passes to next node to mark lead as emailed.  
  - Edge cases: SMTP auth errors, wrong credentials, blocked email, spam filters.

- **Creating Field for Updating "Sent" in G Sheet1**  
  - Type: Set  
  - Role: Prepares data to update Google Sheet to mark email as sent.  
  - Config: Sets field `accepted[0]` to "Sent".  
  - Inputs: Output of "Send email using SMTP".  
  - Outputs: Leads to "Updating GSheet if Email Is 'Sent'".  
  - Edge cases: None significant.

---

#### 1.5 Post-Send Update

**Overview:**  
Updates the Google Sheet to mark the lead’s outreach status as "Sent".

**Nodes Involved:**  
- Updating GSheet if Email Is "Sent" (Google Sheets Update)  

**Node Details:**  

- **Updating GSheet if Email Is "Sent"**  
  - Type: Google Sheets  
  - Role: Updates the "Email Sent" column in a separate Google Sheet to "Sent" for the lead.  
  - Config:  
    - Document ID and sheet ID point to "AI tool reachout" sheet.  
    - Matching by "Tool Name" column.  
    - Updates "Email Sent" field to value from previous set node (`accepted[0]` = "Sent").  
  - Inputs: From "Creating Field for Updating 'Sent' in G Sheet1".  
  - Outputs: None  
  - Edge cases: Google API errors, incorrect matching causing wrong updates.

---

#### 1.6 Informational Sticky Notes

**Overview:**  
Provides documentation, instructional videos, sample sheets, and resource links embedded as sticky notes for users maintaining or using this workflow.

**Nodes Involved:**  
- Sticky Note (Connect with Google Sheet or CRM)  
- Sticky Note1 (Send email using SMTP - Any Email Provider works)  
- Sticky Note2 (Find out SMTP info for your email service provider)  
- Sticky Note3 (Step-by-Step Tutorial YouTube Video Link)  
- Sticky Note4 (Sample Google Sheet for Leads I used)  
- Sticky Note5 (Add Email Verification API info)  

**Node Details:**  
- Each sticky note contains URLs or text instructions to help users configure credentials, understand the flow, or access example data.  
- They do not affect workflow logic but provide context.

---

### 3. Summary Table

| Node Name                                 | Node Type         | Functional Role                           | Input Node(s)                           | Output Node(s)                                    | Sticky Note                                                                                                                                              |
|-------------------------------------------|-------------------|-----------------------------------------|---------------------------------------|--------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’           | Manual Trigger    | Initiates manual run                     | None                                  | Lead Sheet - Get Rows from the sheet              |                                                                                                                                                          |
| Schedule Trigger at 9 AM                    | Schedule Trigger  | Initiates scheduled daily run at 9 AM  | None                                  | (Implicit trigger to Lead Sheet - Get Rows)       |                                                                                                                                                          |
| Lead Sheet - Get Rows from the sheet       | Google Sheets     | Retrieves lead data                      | When clicking ‘Execute workflow’, Schedule Trigger | Checker - If Already "Sent" or not                 | Sticky Note: Connect with Google Sheet or CRM                                                                                                           |
| Checker - If Already "Sent" or not          | If                | Filters leads already emailed            | Lead Sheet - Get Rows from the sheet  | No Operation, Do Nothing (if Sent), Checker - If Lead's "Email" Field is empty? (if not Sent) |                                                                                                                                                          |
| No Operation, Do Nothing                    | NoOp              | Ends processing for already emailed leads | Checker - If Already "Sent" or not    | None                                             |                                                                                                                                                          |
| Checker - If Lead's "Email" Field is empty? | If                | Filters leads with empty email           | Checker - If Already "Sent" or not    | No Operation, Do Nothing1 (if empty), Email verification using Hunter (if not empty) |                                                                                                                                                          |
| No Operation, Do Nothing1                   | NoOp              | Ends processing for leads missing email | Checker - If Lead's "Email" Field is empty? | None                                             |                                                                                                                                                          |
| Email verification using Hunter             | Hunter            | Verifies lead email deliverability       | Checker - If Lead's "Email" Field is empty? | Checker - If emails are verified or no?           | Sticky Note: Add Email Verification API (hunter.io)                                                                                                    |
| Checker - If emails are verified or no?    | If                | Filters undeliverable emails             | Email verification using Hunter       | No Operation, do nothing5 (if undeliverable), Send email using SMTP (if deliverable) |                                                                                                                                                          |
| No Operation, do nothing5                   | NoOp              | Ends processing for undeliverable emails | Checker - If emails are verified or no? | None                                             |                                                                                                                                                          |
| Send email using SMTP                       | Email Send        | Sends outreach email via SMTP            | Checker - If emails are verified or no? | Creating Field for Updating "Sent" in G Sheet1    | Sticky Note: Send email using SMTP - Any Email Provider works                                                                                            |
| Creating Field for Updating "Sent" in G Sheet1 | Set               | Prepares "Sent" status for sheet update  | Send email using SMTP                  | Updating GSheet if Email Is "Sent"                 |                                                                                                                                                          |
| Updating GSheet if Email Is "Sent"          | Google Sheets     | Updates outreach status in Google Sheet  | Creating Field for Updating "Sent" in G Sheet1 | None                                             | Sticky Note: Sample Google Sheet for Leads I used (https://docs.google.com/spreadsheets/d/1pWxsNGde3BHzIF7IMiRum4sGjVNRl9LfcJybmgkzuMM/edit)          |
| Sticky Note                                  | Sticky Note       | Documentation on Google Sheet connection | None                                  | None                                             | Connect with Google Sheet or CRM                                                                                                                        |
| Sticky Note1                                 | Sticky Note       | Documentation on SMTP email sending      | None                                  | None                                             | Send email using SMTP- Any Email Provider works                                                                                                         |
| Sticky Note2                                 | Sticky Note       | Documentation on finding SMTP info       | None                                  | None                                             | Find out SMTP info for your email service provider Link: https://docs.google.com/document/d/1UnAYprKmGWHQ7VDpzdMndV7-iPl1sVlIIke3GY5O8e0/edit?usp=sharing |
| Sticky Note3                                 | Sticky Note       | Link to step-by-step YouTube tutorial    | None                                  | None                                             | Step-by-Step Tutorial YouTube Video Link: https://youtu.be/Dg68OaYPhYs?si=LVu9pdxt9JUuCrzM                                                               |
| Sticky Note4                                 | Sticky Note       | Sample Google Sheet for leads             | None                                  | None                                             | Sample Google Sheet for Leads I used: https://docs.google.com/spreadsheets/d/1pWxsNGde3BHzIF7IMiRum4sGjVNRl9LfcJybmgkzuMM/edit?usp=sharing                 |
| Sticky Note5                                 | Sticky Note       | Email verification API instructions       | None                                  | None                                             | Add Email Verification API - hunter.io: https://hunter.io/api-keys                                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Triggers:**  
   - Add a **Manual Trigger** node named "When clicking ‘Execute workflow’" with default parameters.  
   - Add a **Schedule Trigger** node named "Schedule Trigger at 9 AM" configured to run daily at hour 9.

2. **Retrieve Leads from Google Sheets:**  
   - Add a **Google Sheets** node named "Lead Sheet - Get Rows from the sheet".  
   - Configure credentials for Google Sheets.  
   - Set Document ID to `1pWxsNGde3BHzIF7IMiRum4sGjVNRl9LfcJybmgkzuMM`.  
   - Set Sheet Name/GID to `gid=0` (Sheet1).  
   - Leave other options default.

3. **Filter Leads Already Emailed:**  
   - Add an **If** node named "Checker - If Already 'Sent' or not".  
   - Condition: Check if field `Email Sent` contains string "Sent".  
   - Connect "Lead Sheet - Get Rows" output to this node input.

4. **Skip Already Emailed Leads:**  
   - Add a **No Operation (NoOp)** node "No Operation, Do Nothing" connected to the True branch of the previous If node.

5. **Check for Empty Emails:**  
   - Add an **If** node "Checker - If Lead's 'Email' Field is empty?".  
   - Condition: Check if field `Email` is empty string.  
   - Connect False branch of previous If node to this node.

6. **Skip Empty Email Leads:**  
   - Add a **NoOp** node named "No Operation, Do Nothing1" connected to the True branch.

7. **Email Verification with Hunter.io:**  
   - Add a **Hunter** node named "Email verification using Hunter".  
   - Configure Hunter API credentials.  
   - Set email parameter to `{{$json.Email}}`.  
   - Connect False branch of previous If node to this node.

8. **Filter Undeliverable Emails:**  
   - Add an **If** node "Checker - If emails are verified or no?".  
   - Condition: Check if Hunter API `result` contains string "undeliverable".  
   - Connect Hunter node output to this node.

9. **Skip Undeliverable Emails:**  
   - Add a **NoOp** node "No Operation, do nothing5" connected to True branch.

10. **Send Email using SMTP:**  
    - Add an **Email Send** node "Send email using SMTP".  
    - Configure SMTP credentials (from your email provider).  
    - Set "To Email" to `{{$json.Email}}`.  
    - Set "From Email" to your sender address (e.g. support@dolphy.io).  
    - Set Subject to "Let's collaborate!".  
    - Set email body text to:  
      ```
      Hello {{$json['Tool Name']}} team, 

      We can collaborate on multiple features. Can we have a quick meet over the meet?

      Best regards,
      Janak
      ```  
    - Connect False branch of email verification If node to this node.

11. **Mark Lead as Sent:**  
    - Add a **Set** node "Creating Field for Updating 'Sent' in G Sheet1".  
    - Add assignment: Set field `accepted[0]` to string "Sent".  
    - Connect Email Send node output to this node.

12. **Update Google Sheet with Sent Status:**  
    - Add a **Google Sheets** node "Updating GSheet if Email Is 'Sent'".  
    - Configure Google Sheets credentials.  
    - Set Document ID to `1cMGVHzjsAKmjrNqS9Pi0HPAmq6uv8bl211LdzZNg1mY`.  
    - Set Sheet to `AI tool reachout` (GID 2051398108).  
    - Operation: Update.  
    - Matching Column: "Tool Name".  
    - Update Column: "Email Sent" with value `{{ $json.accepted[0] }}`.  
    - Connect Set node output to this node.

13. **Connect Trigger Outputs:**  
    - Connect both Manual Trigger and Schedule Trigger outputs to the Google Sheets Get Rows node to start the flow.

14. **Add Sticky Notes (Optional):**  
    - Add sticky notes with relevant documentation and resource links for easier maintenance.

---

### 5. General Notes & Resources

| Note Content                                                                                                           | Context or Link                                                                                                         |
|------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| Find out SMTP info for your email service provider                                                                     | https://docs.google.com/document/d/1UnAYprKmGWHQ7VDpzdMndV7-iPl1sVlIIke3GY5O8e0/edit?usp=sharing                        |
| Step-by-Step Tutorial YouTube Video Link                                                                               | https://youtu.be/Dg68OaYPhYs?si=LVu9pdxt9JUuCrzM                                                                       |
| Sample Google Sheet for Leads I used                                                                                    | https://docs.google.com/spreadsheets/d/1pWxsNGde3BHzIF7IMiRum4sGjVNRl9LfcJybmgkzuMM/edit?usp=sharing                     |
| Add Email Verification API - Hunter.io                                                                                  | https://hunter.io/api-keys                                                                                                |
| Send email using SMTP - Any Email Provider works                                                                        | General note explaining SMTP node compatibility                                                                         |
| Connect with Google Sheet or CRM                                                                                        | General note about integration with data source                                                                        |

---

**Disclaimer:** The text provided is exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.