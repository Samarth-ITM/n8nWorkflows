Build an OpenAI HR assistant for Telegram attendance and salary reports

https://n8nworkflows.xyz/workflows/build-an-openai-hr-assistant-for-telegram-attendance-and-salary-reports-13441


# Build an OpenAI HR assistant for Telegram attendance and salary reports

This technical document provides a detailed analysis and reconstruction guide for the **HR Assistant for Telegram** n8n workflow. This system integrates Telegram, OpenAI, and Google Sheets to manage employee onboarding, attendance tracking, and automated financial reporting.

---

### 1. Workflow Overview

The workflow serves as an automated HR department via a Telegram bot. It handles real-time interactions with employees and scheduled administrative reporting.

**Logical Blocks:**
*   **1.1 Input Reception & Classification:** Captures Telegram messages and uses an AI Agent to determine if the user is attempting to register, log an activity (attendance/advance), or request a financial report.
*   **1.2 New Employee Onboarding:** A conversational flow that collects personal details (ID, Name, Salary, etc.) and saves them to a master Google Sheet.
*   **1.3 Transaction Processing:** Logs daily actions such as "Check-in" and "Check-out." It includes logic to calculate penalties for early departures (working less than 8 hours).
*   **1.4 Financial Reporting:** An AI-powered analyst fetches historical logs for a specific employee, applies company-specific financial rules, and generates a formatted net salary report.
*   **1.5 Daily Administrative Job:** A scheduled trigger that compares daily attendance logs against the master employee list to identify and report absentees to management.

---

### 2. Block-by-Block Analysis

#### 2.1 Input & Intent Classification
Determines the nature of the user's request using LLM-based intent recognition.
*   **Nodes Involved:** `Telegram_Trigger`, `Route_New_vs_Existing`, `Agent_Classifier`, `Model_Classifier`, `Memory_Chat`, `Parse_AI_Response`, `Route_Intent`.
*   **Node Details:**
    *   **Agent_Classifier:** Uses GPT-4o-mini to extract structured JSON containing `intent`, `action`, `identifier`, and `timestamp`. It handles both Arabic and English inputs.
    *   **Parse_AI_Response (Code Node):** A JavaScript snippet that strips Markdown formatting (e.g., \`\`\`json) from the AI's response to ensure valid JSON parsing.
    *   **Route_Intent:** Directs the flow to either "Transaction Recording" or "Financial Reporting" based on the AI's classification.

#### 2.2 New Employee Onboarding
Handles the registration of staff not yet in the system.
*   **Nodes Involved:** `Msg_Ask_New_Emp_Data`, `Agent_Onboarding`, `Model_Onboarding`, `Memory_Onboarding`, `Validate_New_Emp_JSON`, `Filter_Valid_Data`, `Register_New_Emp`, `Msg_Registration_Success`.
*   **Node Details:**
    *   **Agent_Onboarding:** Manages the conversation. It is programmed to provide a specific template if the user confirms interest but hasn't provided data yet.
    *   **Validate_New_Emp_JSON (Code Node):** Uses Regex to extract employee data from the AI's conversational text.
    *   **Register_New_Emp (Google Sheets):** Appends the validated `Name`, `ID`, `Position`, `Branch`, and `Salary` to the "data" tab.

#### 2.3 Transaction Logic (Attendance & Advances)
Records specific workplace movements and calculates time-based metrics.
*   **Nodes Involved:** `Lookup_Employee`, `Check_Employee_Found`, `Route_Action_Type`, `Log_Transaction`, `Get_Arrival_Time`, `Calc_Departure_Stats`, `Log_Departure`, `Log_Advance`.
*   **Node Details:**
    *   **Lookup_Employee:** Performs an `OR` filter search in Google Sheets using either the Employee ID or Name.
    *   **Calc_Departure_Stats (Code Node):** Compares the `timestamp` of the current "Check-out" with the "Check-in" found by `Get_Arrival_Time`. It calculates hours worked and flags "Early Departure" if the duration is under 8 hours.
    *   **Log_Departure:** Writes the check-out time and any calculated penalties to the logs.

#### 2.4 Financial Reporting
Synthesizes logs into a human-readable financial summary.
*   **Nodes Involved:** `Fetch_Salary`, `Fetch_Movement_History`, `Agent_Financial`, `Model_Analyst`, `Memory_Analyst`, `Msg_Analysis_Report`.
*   **Node Details:**
    *   **Agent_Financial:** Act as a "Professional HR Financial Analyst." It receives the Basic Salary and all movement logs. It applies specific business rules: Absence = -1 day, Overtime = +1 day, Early Departure = -0.25 day.

#### 2.5 Daily Scheduled Jobs
Automated administrative oversight.
*   **Nodes Involved:** `Daily_Trigger`, `Fetch_Today_Logs`, `Filter_Today_Attendance`, `Fetch_All_Employees`, `Merge_Lists`, `Calc_Absentees`, `Msg_Absentee_Report`.
*   **Node Details:**
    *   **Daily_Trigger:** Set to run daily at 1:00 PM (13:00).
    *   **Calc_Absentees (Code Node):** Uses `$items()` to compare the list of all employees against the list of those who checked in today, returning only the missing records.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Telegram_Trigger | Telegram Trigger | Entry point for users | None | Route_New_vs_Existing | |
| Route_New_vs_Existing | Switch | Initial triage | Telegram_Trigger | Msg_Ask_New_Emp_Data, Agent_Onboarding, Agent_Classifier | |
| Agent_Classifier | AI Agent | Intent extraction | Route_New_vs_Existing | Parse_AI_Response | 1. Input & Intent Classification... |
| Parse_AI_Response | Code | JSON Sanitization | Agent_Classifier | Route_Intent | 1. Input & Intent Classification... |
| Route_Intent | Switch | Logical Routing | Parse_AI_Response | Lookup_Employee, Fetch_Salary | |
| Lookup_Employee | Google Sheets | Database Lookup | Route_Intent | Check_Employee_Found | |
| Agent_Financial | AI Agent | Payroll Calculation | Fetch_Movement_History | Msg_Analysis_Report | 4. Financial Reporting... |
| Agent_Onboarding | AI Agent | Registration Flow | Route_New_vs_Existing | Validate_New_Emp_JSON | 2. New Employee Onboarding... |
| Register_New_Emp | Google Sheets | Data Persistence | Filter_Valid_Data | Msg_Registration_Success | 2. New Employee Onboarding... |
| Calc_Departure_Stats | Code | Penalty Logic | Get_Arrival_Time | Log_Departure | 3. Transaction Logic... |
| Log_Transaction | Google Sheets | Attendance Logging | Route_Action_Type | Msg_Attendance_Success | 3. Transaction Logic... |
| Daily_Trigger | Schedule Trigger | Periodic reporting | None | Fetch_Today_Logs, Fetch_All_Employees | 5. Daily Scheduled Jobs... |
| Calc_Absentees | Code | Gap Analysis | Merge_Lists | Msg_Absentee_Report | 5. Daily Scheduled Jobs... |

---

### 4. Reproducing the Workflow from Scratch

1.  **Prerequisites:**
    *   Create a Telegram Bot via @BotFather and obtain the API Token.
    *   Set up a Google Sheet with two tabs: `data` (Columns: Name, ID, Role, Branch, Salary) and `logs` (Columns: Timestamp, ID, Name, Action, Value, Notes).
    *   Obtain an OpenAI API Key.

2.  **Step 1: Input Handling:**
    *   Add a **Telegram Trigger** node.
    *   Add a **Switch** node (`Route_New_vs_Existing`) to detect keywords like "Yes", "Salary", or a dash "-" (typical of the registration format).

3.  **Step 2: AI Intent Logic:**
    *   Connect an **AI Agent** (`Agent_Classifier`). Set the System Prompt to require JSON output with keys: `intent`, `action`, `identifier`, `value`, `notes`.
    *   Attach a **Window Buffer Memory** and **Chat OpenAI** model (GPT-4o-mini).
    *   Add a **Code Node** to parse the output: `JSON.parse(rawText.replace(/```json|```/g, ""))`.

4.  **Step 3: Registration Logic:**
    *   Create an **AI Agent** (`Agent_Onboarding`) with a system prompt defining the registration format: `(Code - Name - Position - Branch - Salary)`.
    *   Add a **Google Sheets** node set to `Append` for the "data" tab.

5.  **Step 4: Transaction & Attendance Logic:**
    *   Add a **Google Sheets** node (`Lookup_Employee`) to verify the user exists in the "data" tab.
    *   Add a **Switch** node (`Route_Action_Type`) to branch based on whether the action is "Check-out" (انصراف) or "Advance" (سلف).
    *   For "Check-out", add a **Google Sheets** node to find the last "Check-in" for that ID. Use a **Code Node** to subtract timestamps and verify if > 8 hours.

6.  **Step 5: Reporting Logic:**
    *   Add a **Google Sheets** node to fetch the Employee's salary.
    *   Add a second **Google Sheets** node to fetch all rows from the "logs" tab matching that Employee ID.
    *   Add an **AI Agent** (`Agent_Financial`) with the company's deduction rules (e.g., Early departure = -0.25 day).

7.  **Step 6: Admin Automation:**
    *   Add a **Schedule Trigger** set to 13:00.
    *   Fetch all rows from the `data` tab and all "Check-in" rows from `logs` for the current date.
    *   Use a **Code Node** to filter the `data` list for IDs not present in the `logs` list.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Replace all `[YOUR_SPREADSHEET_ID]` placeholders | Essential for Google Sheets node functionality. |
| Replace `[ADMIN_CHAT_ID]` in Msg_Absentee_Report | Directs the daily report to the correct manager. |
| Timezone Adjustment | AI prompts use `Africa/Cairo`; change to your local TZ in expressions. |
| Arabic Match Terms | Ensure Google Sheet column headers match: `كود الموظف`, `نوع الحركه`. |