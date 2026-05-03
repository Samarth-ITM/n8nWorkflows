Automated Employee Attendance & Salary Deduction with Google Sheets and GPT-4

https://n8nworkflows.xyz/workflows/automated-employee-attendance---salary-deduction-with-google-sheets-and-gpt-4-11625


# Automated Employee Attendance & Salary Deduction with Google Sheets and GPT-4

### 1. Workflow Overview

This workflow automates the monthly attendance analysis and salary deduction process for employees by integrating Google Sheets data with AI-powered report generation and email distribution. It targets HR departments or payroll teams who need to efficiently process attendance records, compute salary adjustments based on missed hours and absences, generate clear personalized reports using GPT-4, and send these reports via email while logging all email activities.

The workflow is logically divided into these blocks:

- **1.1 Trigger & Data Retrieval:** Monthly trigger initiates the workflow; attendance and salary data for employees are fetched from Google Sheets.
- **1.2 Attendance Calculation & Data Merging:** Daily working hours are calculated per employee; attendance statuses (present, absent, low hours) are derived; salary and personal info are merged.
- **1.3 Salary Deduction Calculation:** Salary deductions are computed based on missing hours and absent days.
- **1.4 Report Generation with AI:** AI (GPT-4) generates professional monthly summaries for each employee.
- **1.5 Report Saving & Email Dispatch:** Final reports are saved to a Google Sheet, then emailed to each employee individually; email sending is logged for auditing.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Data Retrieval

**Overview:**  
This block sets off the workflow on a monthly schedule and retrieves attendance data for all employees from a Google Sheet.

**Nodes Involved:**  
- Schedule Trigger  
- Employees Attendant Sheet

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger node  
  - Role: Initiates workflow monthly  
  - Configuration: Interval set to run every month  
  - Inputs: None (trigger node)  
  - Outputs: Connected to Employees Attendant Sheet node  
  - Edge Cases: Misconfigured schedule may cause missed or repeated runs

- **Employees Attendant Sheet**  
  - Type: Google Sheets node  
  - Role: Fetch attendance rows with Status “Present” or “Absent”  
  - Configuration: Reads data from first sheet (gid=0) of a specified Google Sheets document using service account credentials; filters rows where Status is “Present” OR “Absent”  
  - Inputs: From Schedule Trigger  
  - Outputs: To Calculate hours per day  
  - Edge Cases: Authentication failure, sheet structure changes, empty or partial data

---

#### 1.2 Attendance Calculation & Data Merging

**Overview:**  
Calculates daily hours worked per employee from attendance data; classifies each day as Present, LowHours, or Absent; aggregates monthly totals; fetches salary and employee details; merges data.

**Nodes Involved:**  
- Calculate hours per day  
- Employees Salary Sheet  
- Merge with Salary

**Node Details:**

- **Calculate hours per day**  
  - Type: Code node (JavaScript)  
  - Role: For each attendance record, calculates hours worked by subtracting InTime from OutTime; rounds hours; flags status; aggregates monthly totals per employee (total hours, missing hours, absent/low hours days)  
  - Key Expressions: Uses JS Date objects for time difference, rounds to 2 decimals, accumulates totals in an object grouped by EmpId  
  - Inputs: Attendance data from Employees Attendant Sheet  
  - Outputs: Monthly aggregated attendance summary per employee  
  - Edge Cases: Missing or malformed time entries, zero hours handled as absent, date/time format errors, time zones not explicitly handled

- **Employees Salary Sheet**  
  - Type: Google Sheets node  
  - Role: Retrieves employee salary and additional info (e.g., email) from another Google Sheet tab (gid=1839371816)  
  - Configuration: Uses service account credentials; reads entire sheet without filters  
  - Inputs: None directly, but connected downstream  
  - Outputs: Provides salary data to merge  
  - Edge Cases: Credential failure, missing employee records, data format changes

- **Merge with Salary**  
  - Type: Merge node  
  - Role: Combines attendance summary with salary data matching on EmpId; merges only the first match in case of duplicates  
  - Configuration: Mode “combine”, fieldsToMatchString “EmpId”  
  - Inputs: From Calculate hours per day (main input) and Employees Salary Sheet (additional input)  
  - Outputs: Combined data with attendance and salary info per employee  
  - Edge Cases: Missing EmpId in either dataset, multiple matches, data type mismatches

---

#### 1.3 Salary Deduction Calculation

**Overview:**  
Computes salary deductions based on total missing hours and absent days, calculates final salary after deductions.

**Nodes Involved:**  
- Calculate Salary Deduction

**Node Details:**

- **Calculate Salary Deduction**  
  - Type: Code node (JavaScript)  
  - Role: For each employee, calculates hourly rate (salary / (22 workdays * 8 hours)), computes deduction for missing hours and absent days, calculates final salary by subtracting deduction  
  - Key Expressions: Uses fixed workdays 22 and 8 hours/day; rounds deductions and final salary to 2 decimals  
  - Inputs: Merged attendance and salary data from Merge with Salary  
  - Outputs: JSON with salary breakdown and deduction info, including employee email  
  - Edge Cases: Salary or missing hours missing or zero, division by zero (should not occur), negative salary if deduction exceeds salary (no explicit guard)

---

#### 1.4 Report Generation with AI

**Overview:**  
Generates a professional monthly summary report per employee using OpenAI GPT-4, then converts AI response to JSON format for further processing.

**Nodes Involved:**  
- Adding the report to new sheet  
- OpenAI Chat Model  
- Generate AI Summary Report  
- Convert the result to JSON format

**Node Details:**

- **Adding the report to new sheet**  
  - Type: Google Sheets node  
  - Role: Appends salary and attendance report data to a dedicated “Employee Monthly Report” sheet (gid=735600063) for record-keeping  
  - Configuration: Auto-maps columns like EmpId, Name, OriginalSalary, TotalMissingHours, Deduction, FinalSalary, EmpEmail  
  - Inputs: From Calculate Salary Deduction  
  - Outputs: To Generate AI Summary Report  
  - Edge Cases: Sheet limits, authentication issues

- **OpenAI Chat Model**  
  - Type: Langchain OpenAI Chat node  
  - Role: Provides GPT-4 model interface for text generation  
  - Configuration: Uses GPT-4.1-mini model variant; linked to OpenAI API credentials  
  - Inputs: Connected as language model resource for Generate AI Summary Report  
  - Outputs: AI-generated text summaries  
  - Edge Cases: API quota, rate limits, network errors, malformed prompts

- **Generate AI Summary Report**  
  - Type: Langchain Agent node  
  - Role: Sends structured employee data as prompt to AI to generate a professional attendance and salary summary text; expects array of JSON objects as response  
  - Configuration: System message defines AI role as HR assistant; prompt template includes employee fields; output is an array of summary objects per employee  
  - Inputs: From Adding the report to new sheet  
  - Outputs: AI summary text  
  - Edge Cases: AI response parsing errors, invalid JSON returned, API failures

- **Convert the result to JSON format**  
  - Type: Code node (JavaScript)  
  - Role: Parses AI response text into JSON objects per employee; cleans escaped newlines and aggregates all employees’ summaries into one array  
  - Inputs: AI response from Generate AI Summary Report  
  - Outputs: JSON array of employee reports with SummaryText field  
  - Edge Cases: JSON parse errors, malformed AI output

---

#### 1.5 Report Saving & Email Dispatch

**Overview:**  
Sends personalized salary deduction reports via email to each employee; logs the email sending results to a Google Sheet.

**Nodes Involved:**  
- Send Salary Deduction Report Email  
- Save Email Log to Sheet

**Node Details:**

- **Send Salary Deduction Report Email**  
  - Type: Email Send node  
  - Role: Sends email with AI-generated summary as email body (text format) to the employee’s email address  
  - Configuration: From address is fixed (workspace.abu@gmail.com); subject includes employee name and ID; uses SMTP credentials for Google SMTP with OAuth2 or password  
  - Inputs: JSON with SummaryText and EmpEmail from Convert the result to JSON format  
  - Outputs: Email send response to logging node  
  - Edge Cases: SMTP errors, invalid email addresses, message size limits, network issues

- **Save Email Log to Sheet**  
  - Type: Google Sheets node  
  - Role: Appends email sending results (accepted, rejected, envelopeTime, messageId, etc.) to a dedicated “Email Send Log” sheet (gid=1224577084)  
  - Configuration: Service account authentication; auto-maps response data  
  - Inputs: From Send Salary Deduction Report Email  
  - Outputs: None (end node)  
  - Edge Cases: Sheet limits, auth errors

---

### 3. Summary Table

| Node Name                      | Node Type                      | Functional Role                           | Input Node(s)                | Output Node(s)                | Sticky Note                                                                                                     |
|--------------------------------|--------------------------------|-----------------------------------------|-----------------------------|------------------------------|----------------------------------------------------------------------------------------------------------------|
| Schedule Trigger               | n8n-nodes-base.scheduleTrigger | Monthly workflow trigger                 | None                        | Employees Attendant Sheet     | ## Trigger node and Employee List Runs monthly once before salary and adjust the time if needed. Fetch the attendance report of all employees from the google sheet. |
| Employees Attendant Sheet      | n8n-nodes-base.googleSheets    | Fetch attendance data                    | Schedule Trigger            | Calculate hours per day       | ## Trigger node and Employee List Runs monthly once before salary and adjust the time if needed. Fetch the attendance report of all employees from the google sheet. |
| Calculate hours per day        | n8n-nodes-base.code            | Calculate daily hours and attendance status | Employees Attendant Sheet   | Employees Salary Sheet, Merge with Salary | ## Calculate Daily Hours and Fetch Employees Details This function calculates employee's daily working hours and determines total monthly absences. It will compute total worked hours by subtracting the clock-out time from clock-in time. Fetch the information about all employees like (salary, email, name and etc) |
| Employees Salary Sheet         | n8n-nodes-base.googleSheets    | Fetch employee salary and details       | None                        | Merge with Salary             | ## Calculate Daily Hours and Fetch Employees Details This function calculates employee's daily working hours and determines total monthly absences. It will compute total worked hours by subtracting the clock-out time from clock-in time. Fetch the information about all employees like (salary, email, name and etc) |
| Merge with Salary              | n8n-nodes-base.merge           | Combine attendance and salary data      | Calculate hours per day, Employees Salary Sheet | Calculate Salary Deduction   | ## Calculate Daily Hours and Fetch Employees Details This function calculates employee's daily working hours and determines total monthly absences. It will compute total worked hours by subtracting the clock-out time from clock-in time. Fetch the information about all employees like (salary, email, name and etc) |
| Calculate Salary Deduction     | n8n-nodes-base.code            | Calculate salary deductions and final salary | Merge with Salary           | Adding the report to new sheet | ## Salary Deduction & Final Calculation Once received monthly attendance summary for each employees, it applies the deduction formula by calculating the hourly salary rate and absent days. Once all calculations are complete it formats the final report including salary breakdown, missing hours, absences, deductions and final salary Creates new google sheet for record-keeping and payroll processing. |
| Adding the report to new sheet | n8n-nodes-base.googleSheets    | Append monthly salary report to sheet   | Calculate Salary Deduction  | Generate AI Summary Report    | ## AI Summary & JSON Formatter AI Agent used to calculate the payroll report for each employee to generate a clean and readable summary for email communication. After response, formula node applies formatting rules to convert response summary to JSON objects for each employee. |
| OpenAI Chat Model             | @n8n/n8n-nodes-langchain.lmChatOpenAi | OpenAI GPT-4 model interface             | Used by Generate AI Summary Report (ai_languageModel input) | Generate AI Summary Report (ai_languageModel output) | ## AI Summary & JSON Formatter AI Agent used to calculate the payroll report for each employee to generate a clean and readable summary for email communication. After response, formula node applies formatting rules to convert response summary to JSON objects for each employee. |
| Generate AI Summary Report    | @n8n/n8n-nodes-langchain.agent | AI generates professional monthly reports | Adding the report to new sheet | Convert the result to JSON format | ## AI Summary & JSON Formatter AI Agent used to calculate the payroll report for each employee to generate a clean and readable summary for email communication. After response, formula node applies formatting rules to convert response summary to JSON objects for each employee. |
| Convert the result to JSON format | n8n-nodes-base.code          | Parse AI text to JSON array of summaries | Generate AI Summary Report  | Send Salary Deduction Report Email | ## AI Summary & JSON Formatter AI Agent used to calculate the payroll report for each employee to generate a clean and readable summary for email communication. After response, formula node applies formatting rules to convert response summary to JSON objects for each employee. |
| Send Salary Deduction Report Email | n8n-nodes-base.emailSend    | Email finalized salary reports to employees | Convert the result to JSON format | Save Email Log to Sheet       | ## Email Dispatch & Logging This stage sends each employee's finalized summary report to their respective email address using google SMTP. Once email delivered this workflow automatically logs the sent-mail details. |
| Save Email Log to Sheet       | n8n-nodes-base.googleSheets    | Log email sending results                | Send Salary Deduction Report Email | None                         | ## Email Dispatch & Logging This stage sends each employee's finalized summary report to their respective email address using google SMTP. Once email delivered this workflow automatically logs the sent-mail details. |
| Sticky Note                   | n8n-nodes-base.stickyNote      | Documentation note                       | None                        | None                         | ## How it works This workflow automates the generation and distribution of **monthly attendance and salary summarize** for all employees. it integrates Google Sheets, AI processing, JSON normalization and automated email delivery via google SMTP, ensuring accurate, consistent and personalized reporting to each employee. Setup steps and process steps detailed. |
| Sticky Note1                  | n8n-nodes-base.stickyNote      | Documentation note                       | None                        | None                         | ## Trigger node and Employee List Runs monthly once before salary and adjust the time if needed. Fetch the attendance report of all employees from the google sheet. |
| Sticky Note2                  | n8n-nodes-base.stickyNote      | Documentation note                       | None                        | None                         | ## Calculate Daily Hours and Fetch Employees Details This function calculates employee's daily working hours and determines total monthly absences. It will computes the total worked hours by subtracting the clock-out time from clock-in time. Fetch the information about all employees like (salary, email, name and etc) |
| Sticky Note3                  | n8n-nodes-base.stickyNote      | Documentation note                       | None                        | None                         | ## Salary Deduction & Final Calculation Once received monthly attendance summary for each employees. it applies the deduction formula by calculation the hours salary rate and absent days. Once all calculations are complete it formats the final report including salary breakdown, missing hours, absences, deductions and final salary Creates new google sheet for record-keeping and payroll processing. |
| Sticky Note4                  | n8n-nodes-base.stickyNote      | Documentation note                       | None                        | None                         | ## AI Summary & JSON Formatter AI Agent used for calculate the payroll report for each employee's to generate a clean and readable summary for email communication. After response formula node applies formatting rules to convert response summary to JSON objects for each employees. |
| Sticky Note5                  | n8n-nodes-base.stickyNote      | Documentation note                       | None                        | None                         | ## Email Dispatch & Logging This stage sends each employee's finalized summary report to their respective email address using google SMTP. Once email delivered this workflow automatically logs the sent-mail details. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set interval to run once per month (e.g., every 1 month)  
   - No input, connects to next node

2. **Create Google Sheets node “Employees Attendant Sheet”**  
   - Type: Google Sheets  
   - Credentials: Google Service Account with access to attendance sheet  
   - Document URL: Your attendance Google Sheet URL  
   - Sheet Name: First sheet (gid=0)  
   - Operation: Read rows  
   - Filters: Status column equals “Present” OR “Absent” (combineFilters set to OR)  
   - Connect Schedule Trigger output to this node

3. **Create Code node “Calculate hours per day”**  
   - Type: Code  
   - Language: JavaScript  
   - Paste provided JS code that calculates hours worked, missing hours, and attendance status per day; aggregates monthly data per employee  
   - Input: Connect from Employees Attendant Sheet  
   - Output: To Employees Salary Sheet and Merge with Salary

4. **Create Google Sheets node “Employees Salary Sheet”**  
   - Type: Google Sheets  
   - Credentials: Same Google Service Account  
   - Document URL: Your Google Sheet with employee salary info  
   - Sheet Name: Tab with salary info (e.g., gid=1839371816)  
   - Operation: Read rows  
   - Output: Connect to Merge with Salary node

5. **Create Merge node “Merge with Salary”**  
   - Type: Merge  
   - Mode: Combine  
   - Fields to Match: “EmpId” in both inputs  
   - Multiple Matches: Take first match only  
   - Input 1: From Calculate hours per day  
   - Input 2: From Employees Salary Sheet  
   - Output: To Calculate Salary Deduction

6. **Create Code node “Calculate Salary Deduction”**  
   - Type: Code  
   - Language: JavaScript  
   - Paste provided JS code to compute hourly rate, deductions for missing hours and absences, final salary  
   - Input: From Merge with Salary  
   - Output: To Adding the report to new sheet

7. **Create Google Sheets node “Adding the report to new sheet”**  
   - Type: Google Sheets  
   - Credentials: Google Service Account  
   - Document URL: Your main report Google Sheet  
   - Sheet Name: “Employee Monthly Report” (gid=735600063)  
   - Operation: Append rows  
   - Columns: Map fields EmpId, Name, OriginalSalary, TotalMissingHours, TotalAbsentDays, Deduction, FinalSalary, EmpEmail automatically  
   - Input: From Calculate Salary Deduction  
   - Output: To Generate AI Summary Report

8. **Create OpenAI Chat Model node**  
   - Type: @n8n/n8n-nodes-langchain.lmChatOpenAi  
   - Credentials: OpenAI API key with GPT-4 access  
   - Model: gpt-4.1-mini or equivalent GPT-4 model  
   - No direct input; this node is used as AI model resource for the next node

9. **Create Langchain Agent node “Generate AI Summary Report”**  
   - Type: @n8n/n8n-nodes-langchain.agent  
   - System Message: Define AI as HR assistant generating monthly attendance and salary summaries based on structured JSON input (as per workflow)  
   - Prompt: Template includes employee data fields (EmpId, Name, OriginalSalary, etc.)  
   - Input: From Adding the report to new sheet  
   - AI model: Link to OpenAI Chat Model node  
   - Output: To Convert the result to JSON format

10. **Create Code node “Convert the result to JSON format”**  
    - Type: Code  
    - Language: JavaScript  
    - Paste provided JS code that parses AI text response into JSON array of employee summaries  
    - Input: From Generate AI Summary Report  
    - Output: To Send Salary Deduction Report Email

11. **Create Email Send node “Send Salary Deduction Report Email”**  
    - Type: Email Send  
    - Credentials: Google SMTP (OAuth2 or app password)  
    - From Email: e.g., workspace.abu@gmail.com  
    - To Email: Use employee email field from JSON (EmpEmail)  
    - Subject: Include employee name and ID dynamically  
    - Text Body: Use AI-generated SummaryText from JSON  
    - Input: From Convert the result to JSON format  
    - Output: To Save Email Log to Sheet

12. **Create Google Sheets node “Save Email Log to Sheet”**  
    - Type: Google Sheets  
    - Credentials: Google Service Account  
    - Document URL: Your logging Google Sheet  
    - Sheet Name: “Email Send Log” (gid=1224577084)  
    - Operation: Append rows  
    - Columns: Map email response fields (accepted, rejected, messageId, etc.) automatically  
    - Input: From Send Salary Deduction Report Email  
    - No output (end of workflow)

13. **Connect all nodes according to dependencies described in the workflow.**

14. **Add sticky notes as documentation reminders for each block (optional).**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                              | Context or Link                                                                                                                                                                  |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow automates the generation and distribution of monthly attendance and salary summaries for employees, integrating Google Sheets, AI processing (GPT-4), JSON normalization, and automated email delivery via Google SMTP, ensuring accurate and personalized reporting. Setup requires credentials for Google Sheets, OpenAI, and Google SMTP. Adjust triggers, sheet URLs, and email templates as needed. | Original workflow description and setup instructions embedded in the main Sticky Note.                                                                                         |
| Official n8n Google Sheets node documentation: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets/                                                                                                                                                                                                                                                                                                          | For configuring Google Sheets nodes, filters, and authentication.                                                                                                              |
| OpenAI API documentation: https://platform.openai.com/docs/api-reference/chat/create                                                                                                                                                                                                                                                                                                                                                       | For understanding API limits, model options, and error handling.                                                                                                               |
| Google SMTP for sending emails securely requires OAuth2 or app passwords; refer to https://support.google.com/mail/answer/185833 for setup                                                                                                                                                                                                                                                                                                | Ensure credentials are properly configured for reliable email delivery.                                                                                                        |
| The calculation assumes 22 working days per month and 8 working hours per day for salary computation; adjust if your company policy differs.                                                                                                                                                                                                                                                                                               | Important for adapting salary deduction calculations.                                                                                                                          |

---

**Disclaimer:** The text provided comes exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.