Generate & Email Personalized Certificates from Google Forms with Score Threshold

https://n8nworkflows.xyz/workflows/workflow-3771-1747220144669.png


# Generate & Email Personalized Certificates from Google Forms with Score Threshold

### 1. Workflow Overview

This workflow automates the process of generating personalized certificates for respondents of a Google Form quiz or survey, provided they meet a specified minimum score threshold. It listens for new responses, extracts essential information, verifies if the respondent passes the score criteria, creates a customized certificate in Google Slides, converts it to PDF, and emails it to the recipient.

**Target Use Cases:**  
- Online courses issuing completion or achievement certificates  
- Quizzes and workshops providing score-based certificates  
- Event participation acknowledgments based on performance  

**Logical Blocks:**  
- **1.1 Input Reception:** Triggered by new Google Form responses via linked Google Sheets.  
- **1.2 Data Extraction:** Extracts respondent’s name, email, and score from the form data.  
- **1.3 Score Validation:** Checks if the respondent’s score meets the passing threshold.  
- **1.4 Certificate Generation:** Copies a Google Slides template, replaces placeholders with respondent data.  
- **1.5 PDF Conversion:** Converts the personalized Google Slide into a PDF file.  
- **1.6 Email Delivery:** Sends the generated certificate PDF as an email attachment to the respondent.  
- **1.7 No-Op for Non-qualifiers:** Gracefully handles cases where respondents don’t meet the score threshold.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Listens for new rows added to a specific Google Sheet (linked to Google Form responses), triggering the workflow for each new submission.

- **Nodes Involved:**  
  - Google Sheets Trigger  
  - Sticky Note2 (instructional)

- **Node Details:**  
  - **Google Sheets Trigger**  
    - Type: Trigger node for Google Sheets  
    - Configured to trigger on “rowAdded” event on the sheet named "Form Responses 1" within the specified spreadsheet.  
    - Polls every minute for new rows.  
    - Credentials: OAuth2 connection to Google Sheets with appropriate permissions.  
    - Output: New row data including respondent’s answers.  
    - Edge Cases:  
      - Permissions errors if OAuth2 token expired or insufficient scopes.  
      - Delay if polling interval is too long or Google Sheets API limits hit.  
      - If sheet or column names change, data extraction downstream may fail.  

#### 2.2 Data Extraction

- **Overview:**  
  Selects and assigns the key fields (“Full Name”, “Email”, “Score”) from the form response for further processing.

- **Nodes Involved:**  
  - Extract essential data (Set node)  
  - Sticky Note3 (instructional)

- **Node Details:**  
  - **Extract essential data**  
    - Type: Set node  
    - Assigns variables:  
      - respondentName: extracted from column "ชื่อ (เป็นภาษาอังกฤษ)" (customized name column)  
      - respondentEmail: extracted from column "Email Address"  
      - respondentScore: extracted from column "Score"  
    - Uses expressions to map incoming JSON data fields to internal variables.  
    - Input: Output from Google Sheets Trigger.  
    - Output: JSON with respondentName, respondentEmail, respondentScore.  
    - Edge Cases:  
      - Column names must match exactly; otherwise, values will be undefined or errors will occur.  
      - Data type mismatch if “Score” is not a proper number.  
      - Missing or malformed email/name fields could cause email sending failures.

#### 2.3 Score Validation

- **Overview:**  
  Checks if the respondent’s score exceeds the passing threshold (currently set as > 3, which can be adjusted).

- **Nodes Involved:**  
  - Score Checker (If node)  
  - No Operation, do nothing (NoOp node)  
  - Sticky Note1 (passing score explanation)  
  - Sticky Note4 (score < passing note)

- **Node Details:**  
  - **Score Checker**  
    - Type: If node  
    - Condition: respondentScore > 3 (number comparison) — this threshold should be updated to desired passing score.  
    - Output:  
      - True branch: proceed with certificate generation.  
      - False branch: route to No Operation node (workflow ends gracefully).  
    - Input: Extract essential data node output.  
    - Output connections: True → Copy Google Slide template; False → No Operation node.  
    - Edge Cases:  
      - Score field missing or non-numeric could cause condition failure.  
      - Threshold must be manually updated to reflect actual passing criteria.

  - **No Operation, do nothing**  
    - Type: NoOp node  
    - Purpose: Ends the workflow silently when score threshold not met.  
    - Input: From Score Checker false output.  
    - No output connections.  
    - Edge Cases: None. It simply drops the flow.

#### 2.4 Certificate Generation

- **Overview:**  
  Creates a personalized Google Slides certificate by copying a template and replacing placeholder text with the respondent’s name.

- **Nodes Involved:**  
  - Copy from your template (Google Drive copy)  
  - Replace text (Google Slides replaceText)  
  - Sticky Note5 (template creation instructions)  
  - Sticky Note6 (text replacement explanation)

- **Node Details:**  
  - **Copy from your template**  
    - Type: Google Drive node  
    - Operation: Copy a Google Slides file (certificate template) to a specified folder with the filename set to respondent's name + "’s Certificate".  
    - Uses configured Google Drive OAuth2 credentials.  
    - Input: respondentName from previous nodes.  
    - Parameters:  
      - Source file ID (Google Slides template) must be replaced with user’s own template.  
      - Destination folder ID must be set to a valid Drive folder for storage.  
    - Output: Metadata of copied file including new presentation ID.  
    - Edge Cases:  
      - Invalid or revoked Drive OAuth2 credentials.  
      - Invalid file or folder IDs.  
      - Quota limits on Drive API.  

  - **Replace text**  
    - Type: Google Slides node  
    - Operation: replaceText  
    - Replaces placeholder text “[ NAME ]” in the copied presentation with respondentName.  
    - Page ID hardcoded as “p” (common first page ID, but may vary if template changes).  
    - Presentation ID taken dynamically from the copied file metadata.  
    - Uses Google Slides OAuth2 credentials.  
    - Output: Updates presentation with personalized name.  
    - Edge Cases:  
      - Placeholder text must exactly match “[ NAME ]” including case and spacing.  
      - PageObjectId must be accurate; otherwise, replacement may fail.  
      - Permissions or credential errors may block update.

#### 2.5 PDF Conversion

- **Overview:**  
  Converts the personalized Google Slides presentation into a downloadable PDF file for email attachment.

- **Nodes Involved:**  
  - Convert to PDF (Google Drive download with conversion)  
  - Sticky Note7 (PDF conversion note)

- **Node Details:**  
  - **Convert to PDF**  
    - Type: Google Drive node  
    - Operation: download with conversion from Google Slides to PDF (MIME type application/pdf).  
    - File ID input: dynamic presentationId from previous Replace text node.  
    - Output file name: respondentName + "’s Certificate.pdf".  
    - Credentials: Google Drive OAuth2.  
    - Output: Binary PDF data accessible for email attachment.  
    - Edge Cases:  
      - Conversion errors if file is corrupted or API quota exceeded.  
      - Filename formatting issues if respondentName contains invalid characters.

#### 2.6 Email Delivery

- **Overview:**  
  Sends the personalized certificate PDF as an email attachment to the respondent's email address.

- **Nodes Involved:**  
  - Send to user's email (Gmail node)  
  - Sticky Note8 (email sending instructions)

- **Node Details:**  
  - **Send to user's email**  
    - Type: Gmail node  
    - Parameters:  
      - Recipient: respondentEmail from Score Checker node.  
      - Subject: “Here's your certificate!!” (customizable).  
      - Message body: congratulatory text.  
      - Attachment: binary data from Convert to PDF node.  
      - Append Attribution: disabled (to avoid n8n branding in email footer).  
    - Credentials: Gmail OAuth2 with full send mail access.  
    - Edge Cases:  
      - Authentication errors if Gmail OAuth2 token expired.  
      - Invalid or missing email addresses cause send failures.  
      - Attachment size limits or network failures.

---

### 3. Summary Table

| Node Name               | Node Type                 | Functional Role                      | Input Node(s)           | Output Node(s)                   | Sticky Note                                        |
|-------------------------|---------------------------|------------------------------------|------------------------|---------------------------------|---------------------------------------------------|
| Google Sheets Trigger    | Google Sheets Trigger     | Listens for new form responses     | —                      | Extract essential data           | ### 2) Trigger Node * Replace your Google Sheet id's in this node. |
| Extract essential data   | Set                       | Extract name, email, score fields  | Google Sheets Trigger  | Score Checker                   | ### 3) Extract Node * Select data needed (Name, Email, Score)     |
| Score Checker           | If                        | Check if score meets threshold     | Extract essential data | Copy from your template, No Operation | ### 4) Passing Score * Adjust your passing score here           |
| No Operation, do nothing | No Operation (NoOp)       | Ends workflow for low scores       | Score Checker (false)  | —                               | ### 4.1) Score < passing criteria                          |
| Copy from your template  | Google Drive              | Copy Slides template for editing   | Score Checker (true)   | Replace text                   | ### 4.2) Score > passing criteria * Create Slides template, replace [ name ] placeholder |
| Replace text             | Google Slides             | Replace placeholder with name      | Copy from your template | Convert to PDF                 | ### 5) Replace text * Replace [ name ] with user's name         |
| Convert to PDF           | Google Drive              | Convert Slides to PDF              | Replace text           | Send to user's email           | ### 6) To PDF * Change file name as desired                   |
| Send to user's email     | Gmail                     | Email certificate to respondent    | Convert to PDF         | —                               | ### 7) Send email * Customize your message here                |
| Sticky Note              | Sticky Note               | Instructional notes                | —                      | —                               | ### 1) Start here * Create Google Form, enable quiz mode, link Sheet |
| Sticky Note1             | Sticky Note               | Passing score instructions         | —                      | —                               | ### 4) Passing Score * Adjust your passing score here          |
| Sticky Note2             | Sticky Note               | Trigger node notes                 | —                      | —                               | ### 2) Trigger Node * Replace your Google Sheet id's in this node |
| Sticky Note3             | Sticky Note               | Extraction notes                  | —                      | —                               | ### 3) Extract Node * Select data needed (Name, Email, Score)     |
| Sticky Note4             | Sticky Note               | Score < passing notes             | —                      | —                               | ### 4.1) Score < passing criteria                          |
| Sticky Note5             | Sticky Note               | Template creation notes           | —                      | —                               | ### 4.2) Score > passing criteria * Create Slides template, replace [ name ] placeholder |
| Sticky Note6             | Sticky Note               | Replace text notes                | —                      | —                               | ### 5) Replace text * Replace [ name ] with user's name         |
| Sticky Note7             | Sticky Note               | PDF conversion notes              | —                      | —                               | ### 6) To PDF * Change file name as desired                   |
| Sticky Note8             | Sticky Note               | Email sending notes               | —                      | —                               | ### 7) Send email * Customize your message here                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Form and Link to Google Sheets**  
   - Create a Google Form with quiz mode enabled.  
   - Add fields for Full Name (English), Email Address, and Score.  
   - Link responses to a new Google Sheet.

2. **Create Google Slides Certificate Template**  
   - Design a certificate in Google Slides.  
   - Insert placeholder text “[ NAME ]” where the respondent’s name should appear.  
   - Save and note the Slides file ID.

3. **Setup Google Drive Folder**  
   - Create or select a folder in Google Drive to store generated certificates.  
   - Note the folder ID for use in the workflow.

4. **Create n8n Workflow**

   - **Node 1: Google Sheets Trigger**  
     - Type: Google Sheets Trigger  
     - Set event: "rowAdded"  
     - Configure to watch the Form Responses Sheet (use your Sheet ID and tab name).  
     - Poll interval: every minute.  
     - Credential: Google Sheets OAuth2 with read access.

   - **Node 2: Extract essential data (Set node)**  
     - Type: Set  
     - Assign variables:  
       - respondentName = `{{$json["ชื่อ (เป็นภาษาอังกฤษ)"]}}`  
       - respondentEmail = `{{$json["Email Address"]}}`  
       - respondentScore = `{{$json.Score}}` (ensure Score column name matches)  
     - Input: from Google Sheets Trigger node.

   - **Node 3: Score Checker (If node)**  
     - Type: If  
     - Condition: respondentScore > desired passing score (e.g., 80).  
     - Input: from Extract essential data node.  
     - True output: proceed to copy template.  
     - False output: connect to No Operation node.

   - **Node 4: No Operation, do nothing (NoOp node)**  
     - Type: No Operation  
     - Input: from Score Checker (false branch).  
     - No outputs; ends workflow silently.

   - **Node 5: Copy from your template (Google Drive node)**  
     - Type: Google Drive  
     - Operation: copy  
     - File ID: your Google Slides certificate template ID.  
     - Folder ID: Google Drive folder for storing certificates.  
     - Name: `={{ $json.respondentName + "'s Certificate" }}`  
     - Credential: Google Drive OAuth2 with write access.  
     - Input: from Score Checker (true branch).

   - **Node 6: Replace text (Google Slides node)**  
     - Type: Google Slides  
     - Operation: replaceText  
     - Presentation ID: `={{ $json.id }}` (from copied file metadata)  
     - Text placeholder: “[ NAME ]”  
     - Replacement text: `={{ $json.respondentName }}`  
     - Page Object IDs: use “p” or verify your template’s page ID.  
     - Credential: Google Slides OAuth2.

   - **Node 7: Convert to PDF (Google Drive node)**  
     - Type: Google Drive  
     - Operation: download with conversion  
     - File ID: `={{ $json.presentationId }}` (from previous node)  
     - Conversion: Google Slides → PDF (`application/pdf`)  
     - File name: `={{ $json.respondentName + "'s Certificate" }}`  
     - Credential: Google Drive OAuth2.  
     - Input: from Replace text node.

   - **Node 8: Send to user's email (Gmail node)**  
     - Type: Gmail  
     - To: `={{ $json.respondentEmail }}`  
     - Subject: “Here's your certificate!!” (customizable)  
     - Message: “Congratulations on passing the quiz! Attached is your certificate.”  
     - Attachments: binary data from Convert to PDF node.  
     - Credential: Gmail OAuth2 with send mail permission.  
     - Input: from Convert to PDF node.

5. **Connect Nodes in Order**  
   Google Sheets Trigger → Extract essential data → Score Checker  
   - Score Checker (true) → Copy from template → Replace text → Convert to PDF → Send email  
   - Score Checker (false) → No Operation

6. **Test Workflow**  
   - Submit a test response to the Google Form.  
   - Verify that, if the score meets the threshold, a personalized certificate PDF is emailed.  
   - Check Google Drive folder for copied and updated Slides file.

7. **Adjust Parameters as Needed**  
   - Update passing score in Score Checker node.  
   - Replace IDs for Sheets, Slides, and Drive folder with your own.  
   - Customize email subject and body.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                             | Context or Link                                                     |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------|
| Ensure Google Sheets columns match exactly: "ชื่อ (เป็นภาษาอังกฤษ)" for name, "Email Address" for email, and "Score" for the score value.                                               | Column naming consistency is critical for data extraction.       |
| Placeholder text in Google Slides must exactly match "[ NAME ]" including brackets and spacing for replacement to work correctly.                                                      | Google Slides template design                                       |
| Google Drive and Gmail OAuth2 credentials must have appropriate scopes and be refreshed periodically to avoid authentication errors.                                                    | Credential management best practices                               |
| The workflow uses polling every minute on Google Sheets; consider API quota limits and adjust polling frequency accordingly.                                                             | Performance and quota considerations                               |
| If emails are not sent, verify Gmail OAuth2 credentials and check that email addresses are valid and properly formatted.                                                                  | Email delivery troubleshooting                                    |
| For advanced customization, placeholders other than name can be added and replaced by extending the Set and Replace text nodes accordingly.                                              | Extensibility note                                                |
| Video walkthrough and setup instructions are available from n8n community resources for Google Forms to certificate workflows.                                                            | https://community.n8n.io/                                          |

---

This reference document enables thorough understanding, reproduction, and modification of the “Generate & Email Personalized Certificates from Google Forms with Score Threshold” workflow, providing detailed insight into its structure, configuration, and operational considerations.