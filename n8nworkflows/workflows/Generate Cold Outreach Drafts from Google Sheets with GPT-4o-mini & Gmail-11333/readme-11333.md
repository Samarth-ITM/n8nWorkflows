Generate Cold Outreach Drafts from Google Sheets with GPT-4o-mini & Gmail

https://n8nworkflows.xyz/workflows/generate-cold-outreach-drafts-from-google-sheets-with-gpt-4o-mini---gmail-11333


# Generate Cold Outreach Drafts from Google Sheets with GPT-4o-mini & Gmail

### 1. Workflow Overview

This workflow automates the generation of personalized cold outreach email drafts using data from a Google Sheet, leveraging OpenAI's GPT-4o-mini model for content generation, and drafts emails in Gmail. It targets sales or marketing professionals aiming to streamline lead outreach by automatically creating customized email subjects and bodies based on business data.

The workflow is logically divided into four main blocks:

- **1.1 Input Reception & Filtering:** Manual trigger initiates the workflow; it reads business lead data from Google Sheets and filters out rows where emails have already been sent.

- **1.2 AI Content Generation:** For each unsent lead, it generates a personalized email body and a compelling subject line using OpenAI GPT-4o-mini API calls.

- **1.3 Email Draft Preparation & Creation:** Merges AI-generated content with original lead data, prepares the draft content, and creates a Gmail draft email.

- **1.4 Data Update & Rate Limiting:** Updates the original Google Sheet row to mark the email as sent and records the email content and date. It includes a wait node to manage API rate limits.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Filtering

- **Overview:**  
This block initiates the workflow manually, reads the lead data from a specified Google Sheet, and filters out leads where an email has already been sent or where email is missing.

- **Nodes Involved:**  
  - Manual Trigger  
  - Read Business Data (Google Sheets)  
  - Filter Unsent Emails (If node)  
  - Sticky Note1 (documentation)

- **Node Details:**

  - **Manual Trigger**  
    - Type: `manualTrigger`  
    - Role: Starts the workflow on demand.  
    - Configuration: Default, no parameters.  
    - Inputs: None  
    - Outputs: Connects to "Read Business Data"  
    - Failure Modes: None expected; manual initiation.  

  - **Read Business Data**  
    - Type: `googleSheets`  
    - Role: Reads rows from a Google Sheet tab containing lead data.  
    - Configuration: Reads from sheet named "Sheet (All info)" in a specific Google Sheet document. Uses OAuth2 credentials for Google Sheets.  
    - Key expressions: None dynamic here; all rows are fetched.  
    - Inputs: From "Manual Trigger"  
    - Outputs: Connects to "Filter Unsent Emails"  
    - Edge Cases: Google API auth failures, read timeouts, empty sheet.  

  - **Filter Unsent Emails**  
    - Type: `if`  
    - Role: Filters out records where `email_sent` equals "yes" or if `email` field is empty.  
    - Configuration: Condition checks:  
      - `$json["email_sent"] != "yes"`  
      - `$json["email"]` is not empty  
    - Inputs: From "Read Business Data"  
    - Outputs: To "Personalize Email (ChatGPT)" and "Generate Subject (ChatGPT)" for true condition.  
    - Edge Cases: Missing fields cause expression failures, incorrect column names cause filtering errors.

  - **Sticky Note1**  
    - Content: Describes Phase 1 - manual trigger, sheet read, and filtering unsent emails.

#### 2.2 AI Content Generation

- **Overview:**  
Generates personalized email body content and a subject line for each unsent lead using OpenAI GPT-4o-mini via HTTP Request nodes.

- **Nodes Involved:**  
  - Personalize Email (ChatGPT)  
  - Generate Subject (ChatGPT)  
  - Merge Subject and Body (Merge)  
  - Sticky Note2

- **Node Details:**

  - **Personalize Email (ChatGPT)**  
    - Type: `httpRequest`  
    - Role: Sends a POST request to OpenAI Chat Completions API to generate the personalized email body.  
    - Configuration:  
      - Model: `gpt-4o-mini`  
      - Temperature: 0.7  
      - Max tokens: 400  
      - Prompt dynamically inserts business details (`business_name`, `contact_name`, `city`, `business_type`) from the filtered row.  
      - Output expected: Email body text only.  
    - Credentials: OpenAI API key configured.  
    - Inputs: From "Filter Unsent Emails" (true branch)  
    - Outputs: Connects to the first input of "Merge Subject and Body"  
    - Failure modes: API auth errors, rate limits, empty or malformed responses, prompt errors.

  - **Generate Subject (ChatGPT)**  
    - Type: `httpRequest`  
    - Role: Sends a POST request to OpenAI Chat Completions API to generate a compelling email subject line.  
    - Configuration:  
      - Model: `gpt-4o-mini`  
      - Temperature: 0.6  
      - Max tokens: 50  
      - Prompt dynamically inserts business details to create a subject line under 50 characters, avoiding spammy words.  
      - Output expected: Subject line text only.  
    - Credentials: OpenAI API key configured.  
    - Inputs: From "Filter Unsent Emails" (true branch)  
    - Outputs: Connects to the second input of "Merge Subject and Body"  
    - Failure modes: Same as above.

  - **Merge Subject and Body**  
    - Type: `merge`  
    - Role: Combines the outputs of the two AI nodes by position so both subject and body can be processed together downstream.  
    - Configuration: Combines two inputs, no filtering or matching keys.  
    - Inputs: From "Personalize Email (ChatGPT)" and "Generate Subject (ChatGPT)"  
    - Outputs: Connects to "Prepare Email Draft"  
    - Failure modes: If one input is missing, merge may fail or produce incomplete data.

  - **Sticky Note2**  
    - Content: Describes Phase 2 - AI generation of subject and body.

#### 2.3 Email Draft Preparation & Creation

- **Overview:**  
Prepares the final email draft content by merging AI-generated text with lead details, creates a Gmail draft email, then prepares the Google Sheet update data.

- **Nodes Involved:**  
  - Prepare Email Draft (Set)  
  - Create Email Draft (Gmail)  
  - Merge Subject and Body (from previous block)  
  - Prepare Sheet Update (Set)  
  - Sticky Note3

- **Node Details:**

  - **Prepare Email Draft**  
    - Type: `set`  
    - Role: Assigns variables combining AI-generated email subject/body and original lead data for downstream use.  
    - Configuration:  
      - `email_subject` set to the subject generated by GPT from "Generate Subject (ChatGPT)"  
      - `email_body` set to the personalized email body from "Personalize Email (ChatGPT)"  
      - Copies of `business_name`, `email`, `contact_name`, `city`, `business_type` from "Filter Unsent Emails"  
    - Inputs: From "Merge Subject and Body"  
    - Outputs: Connects to "Create Email Draft"  
    - Failure modes: Expression errors if referenced nodes or properties are missing.

  - **Create Email Draft**  
    - Type: `gmail`  
    - Role: Creates an email draft in Gmail using the prepared subject and email body.  
    - Configuration:  
      - Email type: HTML (replaces newline `\n` with `<br>`)  
      - Subject: From prepared email subject  
      - Message body: From prepared email body  
      - Uses Gmail OAuth2 credentials.  
    - Inputs: From "Prepare Email Draft"  
    - Outputs: Connects to "Prepare Sheet Update"  
    - Failure modes: Gmail auth errors, API rate limits, draft creation errors.

  - **Prepare Sheet Update**  
    - Type: `set`  
    - Role: Prepares the data payload for updating the original Google Sheet row to mark the email as sent with content and date.  
    - Configuration: No explicit parameters shown but assigns updated fields from the email draft and subject.  
    - Inputs: From "Create Email Draft"  
    - Outputs: Connects to "Update Google Sheet"  
    - Failure modes: Expression errors if referencing missing data.

  - **Sticky Note3**  
    - Content: Describes Phase 3 - preparing fields and creating the draft email.

#### 2.4 Data Update & Rate Limiting

- **Overview:**  
Updates the Google Sheet row with email sent status, subject, body, and date; then waits 3 seconds to respect API usage limits before processing the next item.

- **Nodes Involved:**  
  - Update Google Sheet (Google Sheets)  
  - Rate Limit Wait (Wait)  
  - Sticky Note4

- **Node Details:**

  - **Update Google Sheet**  
    - Type: `googleSheets`  
    - Role: Updates the original Google Sheet row with email status (`email_sent`), date sent (`date_sent`), email body, subject used, and other business data.  
    - Configuration:  
      - Update operation matching on `business_name`.  
      - Columns updated: `city`, `email`, `date_sent`, `email_sent` (contains the email body), `contact_name`, `subject_used`, `business_name`, `business_type`, `row_number` (set to 0).  
      - Uses OAuth2 credentials for Google Sheets.  
    - Inputs: From "Prepare Sheet Update"  
    - Outputs: Connects to "Rate Limit Wait"  
    - Failure modes: API auth errors, matching failures if `business_name` is missing or duplicated, update conflicts.

  - **Rate Limit Wait**  
    - Type: `wait`  
    - Role: Pauses the workflow for 3 seconds to avoid hitting API rate limits.  
    - Configuration: Wait unit seconds, amount 3.  
    - Inputs: From "Update Google Sheet"  
    - Outputs: None (end of flow)  
    - Failure modes: None significant.

  - **Sticky Note4**  
    - Content: Describes Phase 4 - preparing fields for sheet update and managing API limits via wait.

---

### 3. Summary Table

| Node Name                 | Node Type         | Functional Role                          | Input Node(s)              | Output Node(s)                     | Sticky Note                                                                                         |
|---------------------------|-------------------|----------------------------------------|----------------------------|----------------------------------|---------------------------------------------------------------------------------------------------|
| Manual Trigger            | manualTrigger     | Initiate workflow manually             | -                          | Read Business Data                | Sticky Note1: Phase 1 - manual trigger, sheet read, filter unsent emails                          |
| Read Business Data        | googleSheets      | Read leads data from Google Sheet      | Manual Trigger             | Filter Unsent Emails             | Sticky Note1                                                                                      |
| Filter Unsent Emails      | if                | Filter rows where email not sent and email exists | Read Business Data          | Personalize Email (ChatGPT), Generate Subject (ChatGPT) | Sticky Note1                                                                                      |
| Personalize Email (ChatGPT)| httpRequest      | Generate personalized email body       | Filter Unsent Emails       | Merge Subject and Body (input 0) | Sticky Note2: Phase 2 - AI generation of subject and body                                        |
| Generate Subject (ChatGPT) | httpRequest      | Generate compelling email subject line | Filter Unsent Emails       | Merge Subject and Body (input 1) | Sticky Note2                                                                                      |
| Merge Subject and Body    | merge             | Combine generated subject and body     | Personalize Email, Generate Subject | Prepare Email Draft             | Sticky Note2                                                                                      |
| Prepare Email Draft       | set               | Prepare variables for email draft      | Merge Subject and Body     | Create Email Draft               | Sticky Note3: Phase 3 - prepare fields and create draft email                                    |
| Create Email Draft        | gmail             | Create Gmail draft email                | Prepare Email Draft        | Prepare Sheet Update             | Sticky Note3                                                                                      |
| Prepare Sheet Update      | set               | Prepare data for Google Sheet update   | Create Email Draft         | Update Google Sheet              | Sticky Note4: Phase 4 - prepare fields and update sheet                                          |
| Update Google Sheet       | googleSheets      | Update Google Sheet with status & content | Prepare Sheet Update        | Rate Limit Wait                 | Sticky Note4                                                                                      |
| Rate Limit Wait           | wait              | Wait 3 seconds to avoid API rate limits | Update Google Sheet        | -                              | Sticky Note4                                                                                      |
| Sticky Note               | stickyNote        | Documentation                          | -                          | -                              | See detailed notes on workflow phases, setup, use cases, troubleshooting                         |
| Sticky Note1              | stickyNote        | Documentation Phase 1                  | -                          | -                              | See above                                                                                        |
| Sticky Note2              | stickyNote        | Documentation Phase 2                  | -                          | -                              | See above                                                                                        |
| Sticky Note3              | stickyNote        | Documentation Phase 3                  | -                          | -                              | See above                                                                                        |
| Sticky Note4              | stickyNote        | Documentation Phase 4                  | -                          | -                              | See above                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - No parameters. This will start the workflow manually.

2. **Add a Google Sheets node (Read Business Data)**  
   - Operation: Read Rows  
   - Sheet Name: "Sheet (All info)" (or your target sheet tab)  
   - Document ID: Your Google Sheet document ID containing lead data  
   - Credentials: Configure Google Sheets OAuth2 credentials  
   - No filters set (read all rows).

3. **Add an If node (Filter Unsent Emails)**  
   - Condition (AND logic):  
     - `$json["email_sent"]` NOT EQUAL to "yes"  
     - `$json["email"]` IS NOT EMPTY  
   - Connect input from Google Sheets node output.

4. **Add two HTTP Request nodes for AI generation:**

   - **Personalize Email (ChatGPT)**  
     - HTTP Method: POST  
     - URL: `https://api.openai.com/v1/chat/completions`  
     - Authentication: Use OpenAI API credentials  
     - Headers: Content-Type: application/json  
     - Body (JSON):  
       ```json
       {
         "model": "gpt-4o-mini",
         "messages": [
           {
             "role": "user",
             "content": "You are writing a simple, direct cold email as an AI automation service provider ... Business Details: ... Write a personalized version of this template: ... (prompt text with variables)"
           }
         ],
         "temperature": 0.7,
         "max_tokens": 400,
         "top_p": 0.9
       }
       ```
     - Use expressions to inject business details from the filtered input.

   - **Generate Subject (ChatGPT)**  
     - Similar HTTP POST as above with:  
       - Model: `gpt-4o-mini`  
       - Prompt to generate a compelling subject line under 50 chars, avoiding spam words.  
       - Temperature: 0.6, Max tokens: 50  
     - Inject business details via expressions.

5. **Add Merge node (Merge Subject and Body)**  
   - Mode: Combine  
   - Combine By: Position  
   - Connect outputs of both AI HTTP Request nodes as inputs.

6. **Add Set node (Prepare Email Draft)**  
   - Assign variables:  
     - `email_subject` = from Generate Subject node result  
     - `email_body` = from Personalize Email node result  
     - Copy `business_name`, `email`, `contact_name`, `city`, `business_type` from filtered lead data  
   - Input: from Merge node output.

7. **Add Gmail node (Create Email Draft)**  
   - Resource: Draft  
   - Email Type: HTML  
   - Subject: `email_subject` from Set node  
   - Message: `email_body` from Set node, replace newlines with `<br>` for HTML formatting  
   - Credentials: Gmail OAuth2 credentials  
   - Input: from Set node.

8. **Add Set node (Prepare Sheet Update)**  
   - Prepare a JSON object with keys for updating the Google Sheet:  
     - `email_sent`: set to the email body content  
     - `date_sent`: current date formatted as "dd/MM/yyyy"  
     - `subject_used`: subject line from AI  
     - Include other business details as needed for matching.  
   - Input: from Gmail draft node.

9. **Add Google Sheets node (Update Google Sheet)**  
   - Operation: Update  
   - Sheet Name and Document ID: same as read node  
   - Matching Column: `business_name` (ensure unique or add unique ID column for better matching)  
   - Columns to update: `email_sent`, `date_sent`, `subject_used`, plus relevant business data fields  
   - Credentials: Google Sheets OAuth2 credentials  
   - Input: from Prepare Sheet Update node.

10. **Add Wait node (Rate Limit Wait)**  
    - Wait 3 seconds (unit: seconds)  
    - Input: from Update Google Sheet node  
    - Purpose: Prevent hitting API rate limits.

11. **Connect flow accordingly:**  
    - Manual Trigger → Read Business Data → Filter Unsent Emails (true branch) → Parallel to Personalize Email and Generate Subject → Merge → Prepare Email Draft → Create Email Draft → Prepare Sheet Update → Update Google Sheet → Rate Limit Wait.

12. **Credential Setup:**  
    - Configure Google Sheets OAuth2 credentials with read/write access to the target spreadsheet.  
    - Configure Gmail OAuth2 credentials with scope to create drafts.  
    - Configure OpenAI API key for Chat Completion calls.

13. **Testing & Validation:**  
    - Run manually with a test row where `email_sent` is empty.  
    - Verify draft creation in Gmail and sheet update.  
    - Adjust AI prompts or wait time if needed for reliability.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| This workflow automates cold email draft generation combining Google Sheets, OpenAI GPT-4o-mini, and Gmail drafts. It requires correct OAuth2 credentials for Google APIs and OpenAI. | Workflow Purpose |
| The workflow respects API rate limits by pausing 3 seconds between updates. Increase wait time if rate limits occur. | Rate Limiting Advice |
| AI prompts are customizable to adjust tone, style, and content of emails and subject lines. | Customization Hint |
| Troubleshooting tips include checking credential scopes for Gmail drafts and Google Sheets updates; ensure matching column exists and AI responses are valid. | Troubleshooting |
| Useful setup tip: add a unique ID column in your sheet for more reliable updates if multiple rows share business names. | Setup Recommendation |
| The sticky notes in the workflow offer detailed phase explanations and usage instructions; review them for context during setup. | Inline Documentation |

---

**Disclaimer:** The text provided is extracted exclusively from an automated workflow built with n8n, a no-code automation tool. It complies with current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and publicly available.