Build an AI-Powered Employee Recognition System with GPT-4, Jotform and Google Sheets

https://n8nworkflows.xyz/workflows/build-an-ai-powered-employee-recognition-system-with-gpt-4--jotform-and-google-sheets-9817


# Build an AI-Powered Employee Recognition System with GPT-4, Jotform and Google Sheets

### 1. Workflow Overview

This workflow automates an AI-powered employee recognition and rewards system designed to enhance peer-to-peer recognition, streamline award processes, and provide insightful analytics. It integrates Jotform for collecting recognition submissions, GPT-4 via Langchain for intelligent analysis and categorization, Google Sheets for logging and tracking, and Gmail for automated notification emails.

Logical blocks:

- **1.1 Input Reception**: Captures peer recognition submissions from Jotform and extracts relevant data fields.
- **1.2 AI Processing**: Uses GPT-4 to analyze and categorize recognition entries, generating detailed insights and scoring.
- **1.3 Recognition Tracking & Logging**: Stores all recognition data along with AI analysis results in a Google Sheet for record-keeping and further processing.
- **1.4 Automated Notifications**: Sends personalized acknowledgment emails to nominators, notifications to nominees, and summary emails to HR leadership.
- **1.5 Monthly Award Automation**: Scheduled trigger that reads eligible nominations, filters by date and eligibility, generates an Employee of the Month report using GPT-4, and mails it to leadership.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block captures recognition submissions via Jotform webhook and extracts designated fields into structured variables for downstream processing.

- **Nodes Involved:**  
  - Jotform Trigger  
  - Extract Recognition Data

- **Node Details:**  

  - **Jotform Trigger**  
    - Type: Trigger node for webhook from Jotform.  
    - Configuration: Listens for new recognition submissions via webhook.  
    - Inputs: External submission from Jotform form.  
    - Outputs: Raw JSON data containing all form fields.  
    - Edge Cases: Possible webhook failures or malformed data.  
    - Notes: Uses webhook ID “recognition-submission-webhook”.

  - **Extract Recognition Data**  
    - Type: Set node to map raw submission data to well-named variables.  
    - Configuration: Extracts fields such as nominator name/email, nominee name/email, department, recognition title, description, specific example, impact, submission ID, and timestamp.  
    - Expressions: Uses `$json.rawRequest['field_name']` to access Jotform fields; `$now.toISO()` for current timestamp.  
    - Inputs: Output from Jotform Trigger.  
    - Outputs: Structured JSON with cleanly named keys for further use.  
    - Edge Cases: Missing or empty form fields; incorrect field names could cause expression failures.

#### 2.2 AI Processing

- **Overview:**  
  This block performs intelligent analysis of recognition submissions, providing detailed categorization, scoring, and award recommendations using GPT-4 via Langchain agents.

- **Nodes Involved:**  
  - AI Recognition Analysis (Langchain Agent)  
  - Parse AI Analysis  
  - OpenAI Chat Model1 (GPT-4 Chat Model used by AI Recognition Analysis)

- **Node Details:**  

  - **AI Recognition Analysis**  
    - Type: Langchain conversational agent node calling GPT-4.  
    - Configuration: Receives structured recognition data and prompts GPT-4 to analyze the nomination in detail.  
    - Key Prompt: Requests JSON-formatted analysis including primary/secondary categories, recognition strength, key qualities, impact level and score, behavioral examples, core values, award recommendations, nomination quality, standout elements, leadership summary, public recognition text, suggested reward, eligibility for monthly award, and reasoning notes.  
    - Inputs: Structured data from “Extract Recognition Data”.  
    - Outputs: JSON analysis results.  
    - Edge Cases: API errors, rate limits, incomplete or ambiguous data leading to incorrect or partial analysis.  
    - System Message: Expert HR analyst persona ensures objective and actionable output.

  - **Parse AI Analysis**  
    - Type: Set node configured to convert Langchain output into accessible JSON fields.  
    - Inputs: Output of AI Recognition Analysis.  
    - Outputs: Parsed JSON fields for use in logging and notifications.

  - **OpenAI Chat Model1**  
    - Type: GPT-4 Chat Model node used internally by AI Recognition Analysis agent.  
    - Configuration: Model set to “gpt-4.1-mini”.  
    - Credentials: OpenAI API key.  
    - Inputs/Outputs: Connected internally to AI Recognition Analysis node.

#### 2.3 Recognition Tracking & Logging

- **Overview:**  
  Logs all recognition submissions and corresponding AI analysis data into a Google Sheet for tracking, analytics, and award processing.

- **Nodes Involved:**  
  - Log to Recognition Sheet

- **Node Details:**  

  - **Log to Recognition Sheet**  
    - Type: Google Sheets node.  
    - Configuration: Appends or updates a row in the “Recognition_Log” sheet of a specified Google Sheets document.  
    - Column Mappings: Includes nomination details, AI-generated categories, scores, award recommendations, timestamps, and status (“Pending Review”).  
    - Inputs: Parsed AI analysis and extracted recognition data.  
    - Outputs: Confirmation of successful logging.  
    - Credentials: OAuth2 for Google Sheets.  
    - Edge Cases: API quota limits, permission errors, invalid Sheet ID or sheet name, data mapping errors.

#### 2.4 Automated Notifications

- **Overview:**  
  Sends tailored acknowledgment and notification emails to nominators, nominees, and HR leadership summarizing the recognition and AI analysis.

- **Nodes Involved:**  
  - Send Nominator Acknowledgment  
  - Send Nominee Notification  
  - Notify HR Leadership  
  - OpenAI Chat Model (GPT-4 Chat Model used by Generate Award Report)

- **Node Details:**  

  - **Send Nominator Acknowledgment**  
    - Type: Gmail node.  
    - Configuration: Sends a thank-you email to the nominator with AI analysis highlights (category, strength, impact).  
    - Inputs: Extracted recognition data and AI analysis.  
    - Outputs: Email sent confirmation.  
    - Credentials: Gmail OAuth2.  
    - Edge Cases: Email delivery failures, invalid email addresses.

  - **Send Nominee Notification**  
    - Type: Gmail node.  
    - Configuration: Sends a congratulatory email to the nominee including nominator name, recognition title, public recognition text, and recognition highlights.  
    - Inputs: Extracted recognition data and AI analysis.  
    - Outputs: Email sent confirmation.  
    - Credentials: Gmail OAuth2.  
    - Edge Cases: Same as above.

  - **Notify HR Leadership**  
    - Type: Gmail node.  
    - Configuration: Sends a detailed summary email to HR leadership (fixed address hr@yourcompany.com) with nomination details, AI scores, award recommendations, and eligibility.  
    - Inputs: Extracted recognition data and AI analysis.  
    - Outputs: Email sent confirmation.  
    - Credentials: Gmail OAuth2.  
    - Edge Cases: Same as above.

  - **OpenAI Chat Model**  
    - Type: GPT-4 Chat Model node used in Monthly Award Automation (see below).  
    - Credentials: OpenAI API key.

#### 2.5 Monthly Award Automation

- **Overview:**  
  Scheduled process that runs monthly to select Employee of the Month by reading eligible nominations from the recognition log, filtering recent and eligible entries, generating a report with GPT-4, and emailing it to leadership.

- **Nodes Involved:**  
  - Monthly Award Trigger  
  - Read Recognition Log  
  - Filter Monthly Eligible  
  - Check Has Nominations  
  - Generate Award Report (Langchain Agent)  
  - Parse Award Report  
  - Send Award Report  
  - OpenAI Chat Model (used by Generate Award Report)

- **Node Details:**  

  - **Monthly Award Trigger**  
    - Type: Schedule Trigger node.  
    - Configuration: Cron expression set to trigger at 9:00 AM on the 1st day of every month.  
    - Outputs: Activation signal for monthly award process.  
    - Edge Cases: Clock synchronization, missed triggers if n8n is down.

  - **Read Recognition Log**  
    - Type: Google Sheets node.  
    - Configuration: Reads entire “Recognition_Log” sheet data from Google Sheet document.  
    - Inputs: Trigger from Monthly Award Trigger.  
    - Outputs: Array of all nominations.  
    - Credentials: Google Sheets OAuth2.

  - **Filter Monthly Eligible**  
    - Type: Code node (JavaScript).  
    - Configuration: Filters nominations to those eligible for monthly award (`eligibleForMonthlyAward` = “Yes”) and submitted within the previous calendar month.  
    - Logic:  
      - Calculates last month date range.  
      - Sorts nominations by average of recognition strength and impact score, descending.  
    - Inputs: Data from Google Sheets read node.  
    - Outputs: Filtered and sorted eligible nominations.  
    - Edge Cases: Empty data, date parsing errors.

  - **Check Has Nominations**  
    - Type: If node.  
    - Configuration: Checks if filtered nominations list is not empty by verifying presence of nomineeName.  
    - Outputs: Conditional path to continue report generation or halt.  

  - **Generate Award Report**  
    - Type: Langchain conversational agent node calling GPT-4.  
    - Configuration: Receives JSON array of eligible nominations and prompts GPT-4 to generate an Employee of the Month report in JSON format with month/year, total nominations, winner recommendation and justification, runner-up, and executive summary.  
    - Inputs: Filtered nominations from Check Has Nominations node.  
    - Outputs: Award report JSON.  
    - System Message: HR analytics expert persona.  
    - Credentials: OpenAI API key.

  - **Parse Award Report**  
    - Type: Set node.  
    - Configuration: Parses the JSON award report for downstream use.  
    - Inputs: Output of Generate Award Report node.  
    - Outputs: Structured award report fields.

  - **Send Award Report**  
    - Type: Gmail node.  
    - Configuration: Sends the Employee of the Month report email to leadership (leadership@yourcompany.com) and CCs HR (hr@yourcompany.com).  
    - Inputs: Parsed award report.  
    - Outputs: Email sent confirmation.  
    - Credentials: Gmail OAuth2.  
    - Edge Cases: Email delivery failures.

---

### 3. Summary Table

| Node Name                   | Node Type                          | Functional Role                      | Input Node(s)                | Output Node(s)                           | Sticky Note                                                                                               |
|-----------------------------|----------------------------------|------------------------------------|-----------------------------|-----------------------------------------|-----------------------------------------------------------------------------------------------------------|
| Sticky Note                 | Sticky Note                      | Overview & Purpose Description     |                             |                                         | 🏆 Employee Recognition & Rewards system overview, key features, and impact                                 |
| Sticky Note1                | Sticky Note                      | Jotform Form Fields Description    |                             |                                         | 📝 Jotform Recognition Fields and form creation link                                                     |
| Jotform Trigger             | JotForm Trigger                 | Input Reception from Jotform       |                             | Extract Recognition Data                 |                                                                                                           |
| Extract Recognition Data    | Set                             | Extracts and Structures Input Data | Jotform Trigger             | AI Recognition Analysis                  |                                                                                                           |
| Sticky Note2                | Sticky Note                      | AI Recognition Analysis Overview   |                             |                                         | 🤖 AI Recognition Analysis description                                                                    |
| AI Recognition Analysis     | Langchain Agent                 | AI Analysis and Categorization     | Extract Recognition Data    | Parse AI Analysis                        |                                                                                                           |
| OpenAI Chat Model1          | Langchain Chat Model            | GPT-4 Model for AI Analysis        |                             | AI Recognition Analysis                  |                                                                                                           |
| Parse AI Analysis           | Set                             | Parses AI Analysis Output           | AI Recognition Analysis     | Log to Recognition Sheet, Send Nominator Acknowledgment, Send Nominee Notification, Notify HR Leadership |                                                                                                           |
| Sticky Note3                | Sticky Note                      | Recognition Tracking Description   |                             |                                         | 📊 Recognition Tracking with Google Sheets                                                               |
| Log to Recognition Sheet    | Google Sheets                   | Logs recognition data and AI output| Parse AI Analysis           | Send Nominator Acknowledgment            |                                                                                                           |
| Send Nominator Acknowledgment| Gmail                          | Sends acknowledgment to nominator | Parse AI Analysis           |                                         |                                                                                                           |
| Sticky Note5                | Sticky Note                      | Nominee Notification Description   |                             |                                         | 🎉 Nominee Notification and morale boost                                                                  |
| Send Nominee Notification   | Gmail                          | Sends notification to nominee      | Parse AI Analysis           |                                         |                                                                                                           |
| Sticky Note6                | Sticky Note                      | HR Leadership Notification Overview|                             |                                         | 👔 HR Leadership Notification summary                                                                     |
| Notify HR Leadership        | Gmail                          | Sends summary email to HR          | Parse AI Analysis           |                                         |                                                                                                           |
| Sticky Note7                | Sticky Note                      | Monthly Award Process Description  |                             |                                         | 📅 Monthly Award Selection process and automation                                                         |
| Monthly Award Trigger       | Schedule Trigger               | Monthly job trigger                 |                             | Read Recognition Log                     |                                                                                                           |
| Read Recognition Log        | Google Sheets                   | Reads all recognition logs         | Monthly Award Trigger       | Filter Monthly Eligible                   |                                                                                                           |
| Filter Monthly Eligible     | Code                           | Filters eligible nominations        | Read Recognition Log        | Check Has Nominations                    |                                                                                                           |
| Check Has Nominations       | If                             | Checks for presence of nominations | Filter Monthly Eligible     | Generate Award Report                     |                                                                                                           |
| Generate Award Report       | Langchain Agent                | Generates monthly award report      | Check Has Nominations       | Parse Award Report                        |                                                                                                           |
| Parse Award Report          | Set                            | Parses report output                | Generate Award Report       | Send Award Report                        |                                                                                                           |
| Send Award Report           | Gmail                          | Sends Employee of the Month report | Parse Award Report          |                                         |                                                                                                           |
| OpenAI Chat Model           | Langchain Chat Model           | GPT-4 Model for Award Report        |                             | Generate Award Report                     |                                                                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Jotform Trigger Node**  
   - Type: Jotform Trigger  
   - Configure webhook to listen for submissions from your Jotform recognition form.  
   - Note: Use webhook ID or configure in Jotform form integration settings.

2. **Create Set Node "Extract Recognition Data"**  
   - Map Jotform raw submission fields to named variables:  
     - nominatorName ← `rawRequest['q3_nominatorName']`  
     - nominatorEmail ← `rawRequest['q4_nominatorEmail']`  
     - nomineeName ← `rawRequest['q5_nomineeName']`  
     - nomineeEmail ← `rawRequest['q6_nomineeEmail']`  
     - department ← `rawRequest['q7_department']`  
     - recognitionTitle ← `rawRequest['q8_recognitionTitle']`  
     - description ← `rawRequest['q9_description']`  
     - example ← `rawRequest['q10_example']`  
     - impact ← `rawRequest['q11_impact']`  
     - submissionId ← submissionID from trigger  
     - submittedAt ← current timestamp ISO string (`$now.toISO()`)  
   - Connect output from Jotform Trigger.

3. **Create Langchain Agent Node "AI Recognition Analysis"**  
   - Model: GPT-4 (gpt-4.1-mini) via Langchain agent.  
   - Prompt: Provide comprehensive JSON analysis with categories, scores, qualities, award recommendation, etc.  
   - System Message: HR analyst expert persona.  
   - Connect input from "Extract Recognition Data".  
   - Credentials: Configure OpenAI API key.

4. **Create Set Node "Parse AI Analysis"**  
   - Configure to parse Langchain JSON output into usable fields.  
   - Connect input from "AI Recognition Analysis".

5. **Create Google Sheets Node "Log to Recognition Sheet"**  
   - Operation: Append or update row.  
   - Document ID: Your Google Sheet ID containing "Recognition_Log" sheet.  
   - Map all extracted and AI analysis fields to corresponding columns.  
   - Status column default to “Pending Review”.  
   - Credentials: Google Sheets OAuth2.  
   - Connect input from "Parse AI Analysis".

6. **Create Gmail Node "Send Nominator Acknowledgment"**  
   - Send To: nominatorEmail from extracted data.  
   - Subject and HTML body: Thank you message including nominee name, AI analysis category, strength, and impact.  
   - Credentials: Gmail OAuth2.  
   - Connect input from "Parse AI Analysis".

7. **Create Gmail Node "Send Nominee Notification"**  
   - Send To: nomineeEmail from extracted data.  
   - Subject and HTML body: Congratulations message including nominator name, recognition title, public recognition text, and highlights.  
   - Credentials: Gmail OAuth2.  
   - Connect input from "Parse AI Analysis".

8. **Create Gmail Node "Notify HR Leadership"**  
   - Send To: hr@yourcompany.com  
   - Subject and body: Summary of nomination details, AI scores, award recommendations, and monthly award eligibility.  
   - Credentials: Gmail OAuth2.  
   - Connect input from "Parse AI Analysis".

9. **Create Schedule Trigger Node "Monthly Award Trigger"**  
   - Cron expression: `0 9 1 * *` (9am on 1st of each month).  
   - Connect output to next node.

10. **Create Google Sheets Node "Read Recognition Log"**  
    - Operation: Read entire sheet.  
    - Document ID and Sheet Name: Same as logging node.  
    - Credentials: Google Sheets OAuth2.  
    - Connect input from "Monthly Award Trigger".

11. **Create Code Node "Filter Monthly Eligible"**  
    - JavaScript code: Filter nominations where `eligibleForMonthlyAward == 'Yes'` and nomination date is within last calendar month.  
    - Sort results by average of recognition strength and impact score descending.  
    - Connect input from "Read Recognition Log".

12. **Create If Node "Check Has Nominations"**  
    - Condition: Check if filtered nomination list is not empty (nomineeName not empty).  
    - Connect input from "Filter Monthly Eligible".

13. **Create Langchain Agent Node "Generate Award Report"**  
    - Model: GPT-4 via Langchain agent.  
    - Input: JSON array of eligible nominations.  
    - Prompt: Generate Employee of the Month report JSON with monthYear, totalNominations, winnerRecommendation, etc.  
    - System Message: HR analytics expert.  
    - Credentials: OpenAI API key.  
    - Connect input from "Check Has Nominations" (true branch).

14. **Create Set Node "Parse Award Report"**  
    - Parse Langchain JSON output for use in email.  
    - Connect input from "Generate Award Report".

15. **Create Gmail Node "Send Award Report"**  
    - Send To: leadership@yourcompany.com  
    - CC: hr@yourcompany.com  
    - Subject and body: Employee of the Month report including executive summary, winner, and runner-up.  
    - Credentials: Gmail OAuth2.  
    - Connect input from "Parse Award Report".

---

### 5. General Notes & Resources

| Note Content                                                                                                                       | Context or Link                                                                                 |
|-----------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Jotform form fields correspond to specific questions (e.g., q3_nominatorName, q4_nominatorEmail, etc.)                             | See Sticky Note1 for detailed form field mapping                                               |
| Jotform form creation and integration can be done for free at [Jotform](https://www.jotform.com/?partner=mediajade)               | Sticky Note1                                                                                   |
| AI Analysis prompt expects GPT-4 with HR analyst persona for objective and actionable insights                                     | Sticky Note2                                                                                   |
| Recognition tracking relies on Google Sheets document with sheet named "Recognition_Log"                                          | Sticky Note3                                                                                   |
| Nominee and Nominator notifications are designed to boost morale and reinforce positive culture                                   | Sticky Note5                                                                                   |
| HR leadership notifications provide quick review summaries for decision makers                                                    | Sticky Note6                                                                                   |
| Monthly Award selection automated process runs on 1st of month, streamlining Employee of the Month selection                       | Sticky Note7                                                                                   |
| Emails use Gmail OAuth2 credentials and require proper OAuth consent and scopes                                                    | Credentials must be configured in n8n                                                         |
| OpenAI API keys used for GPT-4 must have sufficient quota and permissions                                                         | Ensure API key is properly scoped for chat completions and Langchain agent usage                |
| Timestamp format is ISO 8601; timezone considerations should be consistent across environment                                      | Important for accurate filtering in monthly award process                                     |

---

_Disclaimer: The text provided is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with content policies and contains no illegal, offensive, or protected elements. All processed data is legal and publicly available._