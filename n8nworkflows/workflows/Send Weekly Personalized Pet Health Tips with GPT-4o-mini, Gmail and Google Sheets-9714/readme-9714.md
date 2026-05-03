Send Weekly Personalized Pet Health Tips with GPT-4o-mini, Gmail and Google Sheets

https://n8nworkflows.xyz/workflows/send-weekly-personalized-pet-health-tips-with-gpt-4o-mini--gmail-and-google-sheets-9714


# Send Weekly Personalized Pet Health Tips with GPT-4o-mini, Gmail and Google Sheets

### 1. Workflow Overview

This workflow automates the weekly sending of personalized pet health tips via email, using AI-generated content tailored to each pet’s profile and environmental context. It targets pet care service providers or veterinary wellness teams aiming to maintain ongoing engagement with pet owners by delivering relevant, climate-specific, and age-appropriate advice.

The workflow is structured into the following logical blocks:

- **1.1 Weekly Scheduling & Data Loading**  
  Triggers the workflow every Monday at 9 AM and loads pet data from a Google Sheet, filtering for active pets.

- **1.2 Filtering Eligible Pets**  
  Filters pets to include only those who have not received an email in the past 7 days or never received one.

- **1.3 Per-Pet Processing**  
  Processes each pet individually in batches:
  - Calculates pet age from date of birth.
  - Generates a personalized pet health tip using the GPT-4o-mini model via OpenAI.
  - Formats the AI response and pet data for email composition.

- **1.4 Email Sending and Logging**  
  Sends the personalized health tip email via Gmail, updates the Google Sheet with the timestamp of the sent email, and logs the email activity in a separate sheet.

- **1.5 Auxiliary Controls**  
  Includes wait steps to manage pacing and a disabled SendGrid node for alternative email delivery options.

---

### 2. Block-by-Block Analysis

#### 1.1 Weekly Scheduling & Data Loading

**Overview:**  
Triggers the workflow regularly and loads active pet records from Google Sheets to initiate processing.

**Nodes Involved:**  
- Weekly Trigger (Mondays 9am)  
- Load Pet Info

**Node Details:**

- **Weekly Trigger (Mondays 9am)**  
  - Type: Schedule Trigger  
  - Role: Initiates workflow every Monday at 9 AM.  
  - Configuration: Weekly interval set to Monday 9 AM.  
  - Inputs: None  
  - Outputs: Connects to Load Pet Info  
  - Edge Cases: Misconfiguration can cause missed triggers or excessive runs.

- **Load Pet Info**  
  - Type: Google Sheets  
  - Role: Reads pet data from the “Pets” sheet, filtered to only include rows where Status = "Active".  
  - Configuration: Filter applied on Status column with value “Active”.  
  - Inputs: Trigger from Weekly Trigger  
  - Outputs: Connects to Filter: Active + 7-Day Check  
  - Edge Cases: Google Sheets API failures, invalid credentials, or empty sheets.  
  - Sticky Note: "**Loads all rows where Status = \"Active\"**"

---

#### 1.2 Filtering Eligible Pets

**Overview:**  
Filters pets to ensure emails are only sent to active pets who have not been emailed in the last 7 days or never emailed.

**Nodes Involved:**  
- Filter: Active + 7-Day Check

**Node Details:**

- **Filter: Active + 7-Day Check**  
  - Type: Code (JavaScript)  
  - Role: Filters input items based on: Status = "Active", Last_Email_Sent either empty or older than 7 days.  
  - Key Logic:  
    ```js
    if (item.json.Status !== 'Active') return false;
    if (!item.json.Last_Email_Sent) return true;
    const daysSince = (Date.now() - new Date(item.json.Last_Email_Sent)) / (1000*60*60*24);
    return daysSince > 7;
    ```  
  - Inputs: From Load Pet Info  
  - Outputs: Connects to Process Pets One-by-One  
  - Edge Cases: Date parsing errors, timezone issues, missing Last_Email_Sent fields.  
  - Sticky Note: "**Skips if:** Status ≠ \"Active\"; Last_Email_Sent < 7 days ago. Includes if never emailed or last email >7 days old."

---

#### 1.3 Per-Pet Processing

**Overview:**  
Processes each pet record individually, calculating pet age, generating personalized health tips with AI, and formatting data for email.

**Nodes Involved:**  
- Process Pets One-by-One  
- No Operation, do nothing  
- Calculate Pet Age  
- Generate Personalized Tip  
- Format Email Data

**Node Details:**

- **Process Pets One-by-One**  
  - Type: Split In Batches  
  - Role: Processes pets sequentially, one at a time, to avoid rate limits or API overload.  
  - Inputs: From Filter: Active + 7-Day Check  
  - Outputs: Two branches: one to No Operation (skipped), one to Calculate Pet Age  
  - Edge Cases: Large datasets might slow down processing; batch size defaults to 1.  

- **No Operation, do nothing**  
  - Type: NoOp (No Operation)  
  - Role: Placeholder or branch that performs no action.  
  - Inputs: From Process Pets One-by-One  
  - Outputs: None  
  - Edge Cases: None  

- **Calculate Pet Age**  
  - Type: Code (JavaScript)  
  - Role: Converts Date_of_Birth into pet age in months and human-readable format (e.g., "2 years and 3 months").  
  - Key Logic:  
    Calculates difference from current date, converts to months and years, returns pet_age_months and pet_age strings.  
  - Inputs: From Process Pets One-by-One  
  - Outputs: Connects to Generate Personalized Tip  
  - Edge Cases: Invalid or missing Date_of_Birth, timezone differences.  
  - Sticky Note: "Converts Date_of_Birth to readable age with pet_age and pet_age_months added."

- **Generate Personalized Tip**  
  - Type: OpenAI (LangChain)  
  - Role: Uses GPT-4o-mini to generate a personalized, veterinary-aligned, climate-specific pet health tip.  
  - Configuration:  
    - Model: gpt-4o-mini  
    - Temperature: 0.7 (for creativity)  
    - System message: Veterinary wellness expert tone, AAHA/WSAVA guidelines, seasonal considerations.  
    - User message: Includes pet data (name, type, age, location), current month/year, and detailed instructions for output format (JSON with topic, headline, content, category).  
  - Inputs: From Calculate Pet Age  
  - Outputs: Connects to Format Email Data  
  - Credentials: OpenAI API (configured with dummy account)  
  - Edge Cases: API rate limits, malformed responses, connection issues.  
  - Sticky Note: "Climate-specific advice based on pet type, age, country (hemisphere + season), date; returns JSON."

- **Format Email Data**  
  - Type: Code (JavaScript)  
  - Role: Parses AI JSON response, cleans formatting artifacts, merges with pet data, and preps email fields.  
  - Key Logic:  
    - Removes markdown code blocks from AI message.  
    - Attempts JSON.parse, falls back to a generic tip if parsing fails.  
    - Constructs JSON with email, owner_name, pet_name, tip content, and metadata.  
  - Inputs: From Generate Personalized Tip and Calculate Pet Age (accessed via $node reference)  
  - Outputs: Connects to Send Health Tip using Gmail and disabled SendGrid node.  
  - Edge Cases: Parsing errors, incomplete AI response, missing pet data.

---

#### 1.4 Email Sending and Logging

**Overview:**  
Sends the personalized tip email via Gmail, updates the pet record with the sending timestamp, and logs the email event.

**Nodes Involved:**  
- Send Health Tip using Gmail  
- Update Last_Email_Sent Date  
- Log to Email_Log Sheet  
- Wait 2 Seconds  
- Send via SendGrid (Disabled)

**Node Details:**

- **Send Health Tip using Gmail**  
  - Type: Gmail  
  - Role: Sends personalized HTML email to pet owner.  
  - Configuration:  
    - Recipient: pet owner’s email.  
    - Subject: AI-generated headline.  
    - Message body includes greeting, tip headline, content, and sign-off.  
    - Sender Name: "PetCare Team"  
    - Credentials: Gmail OAuth2 (dummy account)  
  - Inputs: From Format Email Data  
  - Outputs: Connects to Update Last_Email_Sent Date  
  - Edge Cases: Authentication errors, email delivery failures, invalid email addresses.

- **Update Last_Email_Sent Date**  
  - Type: Google Sheets  
  - Role: Updates the “Last_Email_Sent” column in the Pets sheet for the email recipient to current timestamp.  
  - Configuration:  
    - Matching column: Email  
    - Operation: Append or update  
    - Sheet: Pets  
  - Inputs: From Send Health Tip using Gmail  
  - Outputs: Connects to Log to Email_Log Sheet  
  - Edge Cases: Sheet write failures, concurrency issues.  
  - Sticky Note: "Updates: Last_Email_Sent = current timestamp; Matches on Email column."

- **Log to Email_Log Sheet**  
  - Type: Google Sheets  
  - Role: Logs each email sent with timestamp, pet name, owner email, tip category, and email status.  
  - Configuration:  
    - Matching column: Parent_Email  
    - Operation: Append or update  
    - Sheet: Email_Log  
  - Inputs: From Update Last_Email_Sent Date  
  - Outputs: Connects to Wait 2 Seconds  
  - Edge Cases: Sheet API errors, partial writes.

- **Wait 2 Seconds**  
  - Type: Wait  
  - Role: Introduces a 2-second delay to throttle processing pace.  
  - Inputs: From Log to Email_Log Sheet  
  - Outputs: Connects back to Process Pets One-by-One  
  - Edge Cases: None

- **Send via SendGrid (Disabled)**  
  - Type: SendGrid (Email)  
  - Role: Alternative email sending node, currently disabled.  
  - Configuration: Uses dynamic template with fields matching formatted email data.  
  - Inputs: From Format Email Data  
  - Outputs: Connects to Update Last_Email_Sent Date (same as Gmail node)  
  - Edge Cases: Disabled, inactive in current workflow.

---

#### 1.5 Auxiliary Controls and Metadata

**Nodes Involved:**  
- Sticky Note (multiple nodes)

**Sticky Note Summaries:**

- General workflow overview with instructions and requirements.  
- Descriptions of filtering logic, pet age calculation, AI prompt design, and sheet update mechanics.

---

### 3. Summary Table

| Node Name                      | Node Type                        | Functional Role                          | Input Node(s)               | Output Node(s)                      | Sticky Note                                                                                         |
|--------------------------------|---------------------------------|----------------------------------------|-----------------------------|------------------------------------|---------------------------------------------------------------------------------------------------|
| Weekly Trigger (Mondays 9am)    | Schedule Trigger                | Starts workflow weekly at Monday 9 AM | None                        | Load Pet Info                      |                                                                                                   |
| Load Pet Info                  | Google Sheets                  | Loads active pets from sheet           | Weekly Trigger              | Filter: Active + 7-Day Check       | **Loads all rows where Status = "Active"**                                                        |
| Filter: Active + 7-Day Check    | Code                          | Filters pets not emailed recently      | Load Pet Info               | Process Pets One-by-One             | **Skips if:** Status ≠ "Active"; Last_Email_Sent < 7 days ago. Includes if never emailed or old.  |
| Process Pets One-by-One         | Split In Batches               | Processes pets individually             | Filter: Active + 7-Day Check | No Operation, do nothing; Calculate Pet Age |                                                                                                   |
| No Operation, do nothing        | NoOp                          | Placeholder/no action                   | Process Pets One-by-One      | None                              |                                                                                                   |
| Calculate Pet Age              | Code                          | Calculates pet age from DOB             | Process Pets One-by-One      | Generate Personalized Tip          | Converts Date_of_Birth to readable age; adds pet_age and pet_age_months                            |
| Generate Personalized Tip       | OpenAI (LangChain)             | Generates AI pet health tip             | Calculate Pet Age           | Format Email Data                  | Climate-specific advice based on pet data and location; returns JSON                              |
| Format Email Data             | Code                          | Parses AI response & formats email data | Generate Personalized Tip   | Send via SendGrid (Disabled); Send Health Tip using Gmail |                                                                                                   |
| Send Health Tip using Gmail     | Gmail                         | Sends personalized health tip email    | Format Email Data           | Update Last_Email_Sent Date        |                                                                                                   |
| Send via SendGrid (Disabled)    | SendGrid                      | Alternative email sender (disabled)    | Format Email Data           | Update Last_Email_Sent Date        | Disabled node                                                                                      |
| Update Last_Email_Sent Date     | Google Sheets                 | Updates timestamp of last email sent   | Send Health Tip using Gmail; Send via SendGrid | Log to Email_Log Sheet             | Updates: Last_Email_Sent = current timestamp; matches on Email column                            |
| Log to Email_Log Sheet          | Google Sheets                 | Logs email sending activity             | Update Last_Email_Sent Date | Wait 2 Seconds                    |                                                                                                   |
| Wait 2 Seconds                 | Wait                          | Adds delay to pacing                    | Log to Email_Log Sheet      | Process Pets One-by-One             |                                                                                                   |
| Sticky Note                   | Sticky Note                   | Documentation and annotation            | None                        | None                              | Various notes on filtering, pet age calculation, AI prompt, scheduling, and updates               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**
   - Type: Schedule Trigger
   - Set interval: Weekly, trigger on Monday at 09:00.
   - Name it "Weekly Trigger (Mondays 9am)".

2. **Add a Google Sheets node to load pet info:**
   - Type: Google Sheets
   - Credentials: Configure Google Sheets OAuth2 with access to your spreadsheet.
   - Document ID: Set to your Google Sheet ID.
   - Sheet Name: Set to the sheet with pet data (e.g., “Pets”).
   - Filters: Add filter where Status equals "Active".
   - Name: "Load Pet Info"
   - Connect output of Schedule Trigger to this node.

3. **Add a Code node to filter pets:**
   - Type: Code
   - Paste JavaScript code filtering only:
     - Items with Status "Active"
     - Last_Email_Sent is empty or older than 7 days
   - Name: "Filter: Active + 7-Day Check"
   - Connect output of Load Pet Info to this node.

4. **Add a Split In Batches node:**
   - Type: SplitInBatches
   - Default batch size of 1 (process pets one at a time).
   - Name: "Process Pets One-by-One"
   - Connect output of Filter node to this node.

5. **Add a No Operation node (optional placeholder):**
   - Type: NoOp
   - Name: "No Operation, do nothing"
   - Connect one output of SplitInBatches here (optional, per original design).

6. **Add a Code node to calculate pet age:**
   - Type: Code
   - Implement JS code to compute age in months and human-readable format from Date_of_Birth.
   - Name: "Calculate Pet Age"
   - Connect other output of SplitInBatches here.

7. **Add an OpenAI node (LangChain) to generate pet health tip:**
   - Type: OpenAI (LangChain)
   - Credentials: Configure with your OpenAI API key.
   - Model: gpt-4o-mini
   - Temperature: 0.7
   - System message: Veterinary expert instructions with guidelines.
   - User message: Include pet name, type, age, country ISO, current month/year, and detailed instructions for output JSON format.
   - Name: "Generate Personalized Tip"
   - Connect output of Calculate Pet Age here.

8. **Add a Code node to format AI response and prepare email data:**
   - Type: Code
   - Parse AI JSON response, clean markdown, merge with pet info fields, build email subject, body, and metadata.
   - Name: "Format Email Data"
   - Connect output of Generate Personalized Tip here.

9. **Add a Gmail node to send email:**
   - Type: Gmail
   - Credentials: Gmail OAuth2 configured for sending emails.
   - Recipient: Set to email field from formatted data.
   - Subject: Use AI-generated headline.
   - Message: HTML formatted with greeting, headline, tip content, and sign-off.
   - Sender Name: "PetCare Team"
   - Name: "Send Health Tip using Gmail"
   - Connect output of Format Email Data here.

10. **(Optional) Add a SendGrid node (disabled):**
    - Type: SendGrid
    - Setup with your SendGrid credentials.
    - Use dynamic templates matching email fields.
    - Disable this node if not used.
    - Connect output of Format Email Data here (parallel with Gmail).

11. **Add a Google Sheets node to update "Last_Email_Sent":**
    - Type: Google Sheets
    - Credentials: Use same Google Sheets OAuth2.
    - Document and sheet as before.
    - Operation: Append or update matching on Email.
    - Update Last_Email_Sent with current timestamp ($now).
    - Name: "Update Last_Email_Sent Date"
    - Connect output of Gmail and SendGrid nodes here.

12. **Add a Google Sheets node to log email activity:**
    - Type: Google Sheets
    - Document: Same spreadsheet.
    - Sheet: "Email_Log"
    - Operation: Append or update.
    - Columns: Status, Pet_Name, Timestamp, Parent_Email, Tip_Category from previous nodes.
    - Name: "Log to Email_Log Sheet"
    - Connect output of Update Last_Email_Sent Date here.

13. **Add a Wait node to delay processing:**
    - Type: Wait
    - Duration: 2 seconds
    - Name: "Wait 2 Seconds"
    - Connect output of Log to Email_Log Sheet here.

14. **Loop back Wait node to Split In Batches "Process Pets One-by-One":**
    - Connect output of Wait node to input of Split In Batches to continue processing batch items.

15. **Add Sticky Note nodes for documentation:**
    - Add sticky notes describing filtering, age calculation, AI generation, email sending, and schedule.

16. **Test workflow end-to-end:**
    - Ensure all credentials are valid.
    - Validate Google Sheet columns required: Email, Owner_Name, Pet_Name, Pet_Type, Date_of_Birth, Country (ISO), Status, Last_Email_Sent.
    - Run manually or wait for trigger.
    - Monitor logs for API errors, email failures, or parsing issues.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                           | Context or Link                                                                                                        |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| The workflow requires two Google Sheets tabs: "Pets" (main data) and "Email_Log" (tracking sent emails).                                                                                                | Spreadsheet structure                                                                                                  |
| OpenAI GPT-4o-mini is used for cost-effective, high-quality AI text generation tailored to veterinary advice standards (AAHA/WSAVA).                                                                    | OpenAI model choice                                                                                                    |
| Gmail OAuth2 credentials must allow sending emails on behalf of the configured sender ("PetCare Team").                                                                                                | Gmail OAuth2 setup                                                                                                     |
| The AI prompt includes seasonal and geographic context by referencing country ISO codes and current month/year to tailor advice climatically.                                                           | AI prompt design                                                                                                       |
| The workflow includes a disabled SendGrid node for users preferring template-based transactional emails; can be enabled by configuring API keys and templates.                                        | Alternative email sending option                                                                                       |
| The batch processing with delays helps avoid API rate limits and Gmail sending restrictions when handling many pet records.                                                                             | Rate limiting and pacing                                                                                                |
| Recommended to monitor the "Email_Log" sheet for successful email sends and troubleshoot issues such as invalid email addresses or delivery failures.                                                  | Monitoring and troubleshooting                                                                                         |
| To customize scheduling, edit the Schedule Trigger node’s interval parameters.                                                                                                                          | Scheduling adjustments                                                                                                 |
| The code nodes use JavaScript and n8n expression syntax; familiarity with both helps to customize filtering or data formatting.                                                                         | Development notes                                                                                                      |
| For more on n8n Google Sheets integration, visit: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets/                                                                        | Official documentation                                                                                                |
| For OpenAI node details: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-langchain.openai/                                                                                                 | Official documentation                                                                                                |
| Gmail OAuth2 setup guide: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail/                                                                                                       | Official documentation                                                                                                |

---

This document provides a comprehensive understanding of the "Automate weekly pet health emails with AI, Gmail and Google Sheets" workflow, enabling reproduction, customization, and troubleshooting by advanced users and AI agents alike.