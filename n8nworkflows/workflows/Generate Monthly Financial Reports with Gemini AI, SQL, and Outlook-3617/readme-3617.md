Generate Monthly Financial Reports with Gemini AI, SQL, and Outlook

https://n8nworkflows.xyz/workflows/generate-monthly-financial-reports-with-gemini-ai--sql--and-outlook-3617


# Generate Monthly Financial Reports with Gemini AI, SQL, and Outlook

### 1. Workflow Overview

This workflow automates the generation and delivery of monthly financial performance reports for business units (cost centers) using SQL data extraction, AI-driven analysis with Google Gemini 2.5, and email distribution via Microsoft Outlook. It is designed for CFOs, finance analysts, and BI teams to gain executive-level insights without manual effort.

The workflow is logically divided into the following blocks:

- **1.1 Scheduling & Date Calculation:** Triggers the workflow monthly and calculates the previous month and year for dynamic querying.
- **1.2 Cost Center Retrieval & Filtering:** Fetches active cost centers with budget and GL data, then filters for a specific cost center (for testing or targeted reporting).
- **1.3 Iteration Over Cost Centers:** Loops through each cost center to generate individual reports.
- **1.4 Financial Data Extraction:** Executes multiple SQL queries to retrieve Year-To-Date (YTD) vs Previous Month financials, departmental vertical P&L, project WIP, and employee statistics.
- **1.5 Data Transformation to HTML:** Converts raw SQL data into formatted HTML tables for report presentation.
- **1.6 AI-Powered Financial Analysis:** Uses Google Gemini AI to analyze the combined HTML financial data and generate a comprehensive business performance report with executive summary and recommendations.
- **1.7 Email Preparation & Delivery:** Packages the AI-generated HTML report and sends it via Microsoft Outlook.
- **1.8 Workflow Control:** Includes wait and batch control nodes to manage processing load and email frequency.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduling & Date Calculation

- **Overview:** This block triggers the workflow monthly on the 5th day and calculates the previous month and year for dynamic SQL filtering.
- **Nodes Involved:** Schedule Trigger, Date & Time, PreviousMonth

**Schedule Trigger**  
- Type: Schedule Trigger  
- Role: Initiates workflow execution on the 5th day of every month.  
- Configuration: Interval set to monthly, trigger at day 5.  
- Inputs: None  
- Outputs: Current date/time to Date & Time node.  
- Edge Cases: Workflow will not trigger if n8n instance is down or paused.

**Date & Time**  
- Type: Date & Time  
- Role: Captures current date/time for use in subsequent nodes.  
- Configuration: Default current date/time output.  
- Inputs: From Schedule Trigger  
- Outputs: Current date/time JSON to PreviousMonth node.  
- Edge Cases: Timezone misconfiguration could affect date calculations.

**PreviousMonth**  
- Type: Code (JavaScript)  
- Role: Calculates previous month and year based on current date.  
- Configuration: Custom JS code that sets date to first of current month, then steps back one day to get previous month and year. Outputs padded month string and year string.  
- Inputs: Current date from Date & Time node.  
- Outputs: JSON with `previousMonth` (e.g., "04") and `year` (e.g., "2024").  
- Edge Cases: Handles year boundary correctly (e.g., January to December previous year).  
- Version: Requires n8n supporting JavaScript code node v2.

---

#### 2.2 Cost Center Retrieval & Filtering

- **Overview:** Retrieves all cost centers with budget and GL data for the selected period, then filters to a specific cost center for focused reporting or testing.  
- **Nodes Involved:** Get Cost Centers with Budgets, CostCentrs (Set), Filter

**Get Cost Centers with Budgets**  
- Type: MySQL  
- Role: Executes SQL query to fetch distinct cost centers having budget and GL entries up to the previous month and year.  
- Configuration: SQL query uses dynamic parameters `year` and `previousMonth` from PreviousMonth node.  
- Inputs: PreviousMonth JSON  
- Outputs: List of cost centers with budgets.  
- Edge Cases: Query failure due to DB connectivity or schema mismatch.  
- Notes: Query joins budget group details and GL entries to ensure cost centers are active and relevant.

**CostCentrs**  
- Type: Set  
- Role: Passes through or formats cost center data for filtering.  
- Configuration: Assigns `Cost Center` field from SQL query results.  
- Inputs: From Get Cost Centers with Budgets  
- Outputs: JSON with cost center names.  
- Edge Cases: Empty input if no cost centers found.

**Filter**  
- Type: Filter  
- Role: Filters cost centers to only process the "AI DEPARTMENT" cost center (example).  
- Configuration: Condition equals `Cost Center` to "AI DEPARTMENT".  
- Inputs: Cost center list from CostCentrs  
- Outputs: Filtered cost center(s) to Loop Over Items node.  
- Edge Cases: No matching cost center results in no further processing.

---

#### 2.3 Iteration Over Cost Centers

- **Overview:** Loops through each filtered cost center to generate reports individually, enabling multi-division automation.  
- **Nodes Involved:** Loop Over Items, Selected Cost Center

**Loop Over Items**  
- Type: SplitInBatches  
- Role: Iterates over cost centers one at a time (batch size 1 by default).  
- Configuration: Default batch options.  
- Inputs: Filtered cost centers from Filter node.  
- Outputs: Single cost center JSON per iteration to downstream nodes.  
- Edge Cases: Large number of cost centers may require batch size tuning.

**Selected Cost Center**  
- Type: Set  
- Role: Sets variables for current cost center, previous month, and year for use in SQL queries.  
- Configuration: Assigns `CostCenter` from current item, `previousMonth` and `year` from PreviousMonth node.  
- Inputs: From Loop Over Items  
- Outputs: JSON with these three fields for query parameterization.  
- Edge Cases: Missing or malformed cost center data.

---

#### 2.4 Financial Data Extraction

- **Overview:** Executes multiple SQL queries to retrieve detailed financial data for the selected cost center, including YTD vs previous month budget/actuals, departmental vertical P&L, project WIP, and employee statistics.  
- **Nodes Involved:** YTD vs Prevoius Month1, Departments, Projects, Employees

**YTD vs Prevoius Month1**  
- Type: MySQL  
- Role: Retrieves budget and actual financial figures grouped by budget group for YTD and previous month, including variances.  
- Configuration: Complex SQL query with dynamic parameters for cost center, year, and previous month.  
- Inputs: Selected Cost Center JSON  
- Outputs: Financial data rows with budget, actuals, and variance.  
- Edge Cases: SQL errors, missing data, or parameter mismatch.

**Departments**  
- Type: MySQL  
- Role: Fetches income, expenses, and profit/loss by vertical (sub-division) within the cost center for the previous month.  
- Configuration: SQL query filtered by cost center, month, and year from Selected Cost Center.  
- Inputs: Selected Cost Center JSON  
- Outputs: Departmental financial summary rows.  
- Edge Cases: Empty results if no vertical data.

**Projects**  
- Type: MySQL  
- Role: Retrieves project counts, contract values, revenue, costs, invoice %, cost %, and WIP for open projects in the cost center.  
- Configuration: SQL query with dynamic cost center parameter, includes subqueries for GL entries.  
- Inputs: Selected Cost Center JSON  
- Outputs: Project financial metrics.  
- Edge Cases: Complex query may timeout or fail on large datasets.

**Employees**  
- Type: MySQL  
- Role: Counts total active employees, those joined this year and month, grouped by payroll cost center.  
- Configuration: SQL query filtered by cost center from Selected Cost Center.  
- Inputs: Selected Cost Center JSON  
- Outputs: Employee statistics.  
- Edge Cases: No employees found or data inconsistency.

---

#### 2.5 Data Transformation to HTML

- **Overview:** Converts raw SQL data rows into formatted HTML tables for inclusion in the final report.  
- **Nodes Involved:** CostCenter, verticalPL, WIP1, Employees1, Merge, Code, HTML

**CostCenter**  
- Type: Code  
- Role: Converts YTD vs Previous Month financial data into an HTML table.  
- Configuration: JS code builds table headers and rows dynamically from input JSON.  
- Inputs: YTD vs Prevoius Month1 SQL output  
- Outputs: JSON with `table` field containing HTML string.  
- Edge Cases: Empty input causes errors; expects at least one row.

**verticalPL**  
- Type: Code  
- Role: Converts Departments SQL data into an HTML table.  
- Configuration: Similar dynamic table generation JS code.  
- Inputs: Departments SQL output  
- Outputs: HTML table string.  
- Edge Cases: Same as CostCenter.

**WIP1**  
- Type: Code  
- Role: Converts Projects SQL data into an HTML table.  
- Configuration: Same dynamic HTML table generation.  
- Inputs: Projects SQL output  
- Outputs: HTML table string.  
- Edge Cases: Same as above.

**Employees1**  
- Type: Code  
- Role: Converts Employees SQL data into an HTML table.  
- Configuration: Same as above.  
- Inputs: Employees SQL output  
- Outputs: HTML table string.  
- Edge Cases: Same as above.

**Merge**  
- Type: Merge  
- Role: Combines the four HTML tables from CostCenter, verticalPL, WIP1, and Employees1 into one data stream.  
- Configuration: Number of inputs set to 4.  
- Inputs: Outputs from CostCenter, verticalPL, WIP1, Employees1  
- Outputs: Combined data to Code node.  
- Edge Cases: Missing input from any node breaks merge.

**Code**  
- Type: Code  
- Role: Builds the full HTML report by embedding all four tables into a styled HTML document with headings and CSS.  
- Configuration: JS code concatenates HTML strings with CSS styles for formatting and colors.  
- Inputs: Merged HTML tables  
- Outputs: JSON with full HTML report string under `html`.  
- Edge Cases: Requires all four tables present; malformed HTML if inputs missing.

**HTML**  
- Type: HTML  
- Role: Passes the final HTML report downstream.  
- Configuration: Uses expression `{{ $json.html }}` to output HTML content.  
- Inputs: From Code node  
- Outputs: HTML content for AI analysis.  
- Edge Cases: None.

---

#### 2.6 AI-Powered Financial Analysis

- **Overview:** Uses Google Gemini 2.5 AI model and LangChain agent to analyze the financial HTML report and generate a detailed business performance analysis with calculations and recommendations.  
- **Nodes Involved:** Business Performance AI Agent (Analyst), Google Gemini Chat Model, Calculator, Think, Financial Performance

**Business Performance AI Agent (Analyst)**  
- Type: LangChain Agent  
- Role: Receives the HTML financial report and instructions to analyze and produce a comprehensive financial performance report, including executive summary, P&L calculations, vertical analysis, project progress, employee KPIs, and recommendations.  
- Configuration: Prompt defines detailed instructions for analysis, including use of Calculator tool for precise computations and formatting output as responsive HTML with color-coded remarks.  
- Inputs: HTML report from HTML node, AI language model and calculator tools linked.  
- Outputs: AI-generated HTML report string with financial insights.  
- Edge Cases: API key or quota issues, AI model errors, incomplete input data.

**Google Gemini Chat Model**  
- Type: LangChain Language Model  
- Role: Provides the AI language model backend (Google Gemini 2.5 Pro) for the agent.  
- Configuration: Model name set to `models/gemini-2.5-pro-exp-03-25`.  
- Inputs: Text prompts from AI Agent node.  
- Outputs: AI-generated text responses.  
- Edge Cases: API authentication or rate limits.

**Calculator**  
- Type: LangChain Tool Calculator  
- Role: Performs precise mathematical calculations requested by the AI agent during analysis.  
- Configuration: Default calculator tool.  
- Inputs: Calculation requests from AI Agent.  
- Outputs: Computed numeric results.  
- Edge Cases: Complex calculations may timeout.

**Think**  
- Type: LangChain Tool Think  
- Role: Allows the AI agent to perform reasoning or intermediate thought processes.  
- Configuration: Default.  
- Inputs: From AI Agent.  
- Outputs: Reasoning outputs.  
- Edge Cases: None.

**Financial Performance**  
- Type: Code  
- Role: Cleans the AI-generated HTML output by removing markdown code block markers (```html ... ```).  
- Configuration: JS code trims and removes markdown syntax.  
- Inputs: AI Agent output.  
- Outputs: Cleaned HTML string for email.  
- Edge Cases: Missing or malformed AI output.

---

#### 2.7 Email Preparation & Delivery

- **Overview:** Prepares the email content with the cleaned AI-generated HTML report and sends it via Microsoft Outlook.  
- **Nodes Involved:** Email Data, Microsoft Outlook2, Wait

**Email Data**  
- Type: Set  
- Role: Sets email fields including recipient, subject, and HTML body content from cleaned AI output.  
- Configuration: Assigns `Email Output` from Financial Performance node, sets subject and recipient email address.  
- Inputs: Cleaned HTML from Financial Performance node.  
- Outputs: JSON with email parameters.  
- Edge Cases: Missing email content or recipient.

**Microsoft Outlook2**  
- Type: Microsoft Outlook  
- Role: Sends the email with the HTML report to the specified recipient.  
- Configuration: Uses Outlook OAuth2 credentials, sends to `amjid@amjidali.com`, subject "Business Performance Syncbricks", body content type HTML.  
- Inputs: Email parameters from Email Data node.  
- Outputs: Email send confirmation.  
- Edge Cases: Authentication errors, network issues, invalid recipient.

**Wait**  
- Type: Wait  
- Role: Pauses workflow for a specified duration (minutes) before looping back to process next cost center.  
- Configuration: Wait unit set to minutes (default duration unspecified).  
- Inputs: From Microsoft Outlook2 node.  
- Outputs: Loops back to Loop Over Items node to process next batch.  
- Edge Cases: Long wait times may delay report delivery.

---

### 3. Summary Table

| Node Name                         | Node Type                     | Functional Role                                    | Input Node(s)                      | Output Node(s)                      | Sticky Note                                                                                              |
|----------------------------------|-------------------------------|--------------------------------------------------|----------------------------------|-----------------------------------|--------------------------------------------------------------------------------------------------------|
| Schedule Trigger                 | Schedule Trigger              | Monthly trigger on 5th day                        | None                             | Date & Time                       | Key Sections of n8n Workflow (covers scheduling and date calculation)                                  |
| Date & Time                     | Date & Time                  | Provides current date/time                         | Schedule Trigger                 | PreviousMonth                    | Key Sections of n8n Workflow                                                                            |
| PreviousMonth                   | Code                         | Calculates previous month and year                | Date & Time                     | Get Cost Centers with Budgets     | Key Sections of n8n Workflow                                                                            |
| Get Cost Centers with Budgets   | MySQL                        | Fetches cost centers with budgets and GL data    | PreviousMonth                   | CostCentrs                      | SQL Query Nodes: Fetches structured financial data; can be replaced with other data sources             |
| CostCentrs                     | Set                          | Passes cost center data                           | Get Cost Centers with Budgets   | Filter                         | Key Sections of n8n Workflow                                                                            |
| Filter                         | Filter                       | Filters cost centers to "AI DEPARTMENT"          | CostCentrs                     | Loop Over Items                 | Key Sections of n8n Workflow                                                                            |
| Loop Over Items                | SplitInBatches               | Iterates over cost centers                        | Filter                         | Selected Cost Center (main output), empty (second output) | Key Sections of n8n Workflow                                                                            |
| Selected Cost Center           | Set                          | Sets variables for current cost center and date | Loop Over Items                | YTD vs Prevoius Month1, Departments, Projects, Employees | Key Sections of n8n Workflow                                                                            |
| YTD vs Prevoius Month1         | MySQL                        | Retrieves YTD vs previous month budget/actuals   | Selected Cost Center            | CostCenter                     | SQL Query Nodes                                                                                         |
| Departments                   | MySQL                        | Fetches vertical P&L data                         | Selected Cost Center            | verticalPL                    | SQL Query Nodes                                                                                         |
| Projects                      | MySQL                        | Retrieves project and WIP data                    | Selected Cost Center            | WIP1                         | SQL Query Nodes                                                                                         |
| Employees                    | MySQL                        | Retrieves employee statistics                     | Selected Cost Center            | Employees1                   | SQL Query Nodes                                                                                         |
| CostCenter                   | Code                         | Converts YTD vs Previous Month data to HTML      | YTD vs Prevoius Month1          | Merge                        | Key Sections of n8n Workflow                                                                            |
| verticalPL                   | Code                         | Converts Departments data to HTML                 | Departments                    | Merge                        | Key Sections of n8n Workflow                                                                            |
| WIP1                        | Code                         | Converts Projects data to HTML                     | Projects                      | Merge                        | Key Sections of n8n Workflow                                                                            |
| Employees1                  | Code                         | Converts Employees data to HTML                    | Employees                     | Merge                        | Key Sections of n8n Workflow                                                                            |
| Merge                       | Merge                        | Combines all HTML tables                           | CostCenter, verticalPL, WIP1, Employees1 | Code                         | Key Sections of n8n Workflow                                                                            |
| Code                        | Code                         | Builds full HTML report with CSS styling          | Merge                         | HTML                         | Key Sections of n8n Workflow                                                                            |
| HTML                        | HTML                         | Passes final HTML report                           | Code                         | Business Performance AI Agent (Analyst) | Key Sections of n8n Workflow                                                                            |
| Business Performance AI Agent (Analyst) | LangChain Agent             | Analyzes financial data and generates report     | HTML                         | Financial Performance          | Key Sections of n8n Workflow                                                                            |
| Google Gemini Chat Model     | LangChain Language Model     | Provides AI language model backend                 | Business Performance AI Agent  | Business Performance AI Agent   | Key Sections of n8n Workflow                                                                            |
| Calculator                  | LangChain Tool Calculator    | Performs precise calculations                      | Business Performance AI Agent  | Business Performance AI Agent   | Key Sections of n8n Workflow                                                                            |
| Think                       | LangChain Tool Think         | Assists AI agent with reasoning                    | Business Performance AI Agent  | Business Performance AI Agent   | Key Sections of n8n Workflow                                                                            |
| Financial Performance       | Code                         | Cleans AI output HTML                              | Business Performance AI Agent  | Email Data                   | Key Sections of n8n Workflow                                                                            |
| Email Data                  | Set                          | Prepares email content                             | Financial Performance          | Microsoft Outlook2            | Key Sections of n8n Workflow                                                                            |
| Microsoft Outlook2          | Microsoft Outlook            | Sends email with report                            | Email Data                   | Wait                         | Key Sections of n8n Workflow                                                                            |
| Wait                       | Wait                         | Controls batch processing and pacing               | Microsoft Outlook2            | Loop Over Items              | Key Sections of n8n Workflow                                                                            |
| Sticky Note                | Sticky Note                  | Workflow overview and key section explanations    | None                         | None                         | Contains detailed workflow explanation and node grouping                                              |
| Sticky Note1               | Sticky Note                  | Explanation of SQL query nodes and data sources   | None                         | None                         | Contains guidance on SQL nodes and alternative data sources                                           |
| Sticky Note11              | Sticky Note                  | Author credits and support links                   | None                         | None                         | Contains author info, support links, and credits                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set to trigger monthly on the 5th day.

2. **Create Date & Time Node**  
   - Type: Date & Time  
   - Connect Schedule Trigger → Date & Time  
   - Use default settings to output current date/time.

3. **Create PreviousMonth Code Node**  
   - Type: Code  
   - Connect Date & Time → PreviousMonth  
   - Paste JS code to calculate previous month and year from current date:  
     ```js
     const inputDateStr = $input.first().json.currentDate;
     const inputDate = new Date(inputDateStr);
     inputDate.setDate(1);
     inputDate.setDate(0);
     const previousMonth = inputDate.getMonth() + 1;
     const year = inputDate.getFullYear();
     return [{ json: { previousMonth: previousMonth.toString().padStart(2, '0'), year: year.toString() } }];
     ```

4. **Create Get Cost Centers with Budgets MySQL Node**  
   - Type: MySQL  
   - Connect PreviousMonth → Get Cost Centers with Budgets  
   - Use provided SQL query with parameters `{{ $json.year }}` and `{{ $json.previousMonth }}`.  
   - Configure database credentials for your SQL server.

5. **Create CostCentrs Set Node**  
   - Type: Set  
   - Connect Get Cost Centers with Budgets → CostCentrs  
   - Assign field `Cost Center` from SQL output.

6. **Create Filter Node**  
   - Type: Filter  
   - Connect CostCentrs → Filter  
   - Set condition: `Cost Center` equals `"AI DEPARTMENT"` (or your target cost center).

7. **Create Loop Over Items Node**  
   - Type: SplitInBatches  
   - Connect Filter → Loop Over Items  
   - Use default batch size (1).

8. **Create Selected Cost Center Set Node**  
   - Type: Set  
   - Connect Loop Over Items → Selected Cost Center  
   - Assign:  
     - `CostCenter` = `{{$json["Cost Center"]}}`  
     - `previousMonth` = `{{ $('PreviousMonth').item.json.previousMonth }}`  
     - `year` = `{{ $('PreviousMonth').item.json.year }}`

9. **Create YTD vs Prevoius Month1 MySQL Node**  
   - Type: MySQL  
   - Connect Selected Cost Center → YTD vs Prevoius Month1  
   - Paste provided complex SQL query with dynamic parameters for cost center, year, previous month.  
   - Configure DB credentials.

10. **Create Departments MySQL Node**  
    - Type: MySQL  
    - Connect Selected Cost Center → Departments  
    - Paste SQL query for vertical P&L with dynamic parameters.  
    - Configure DB credentials.

11. **Create Projects MySQL Node**  
    - Type: MySQL  
    - Connect Selected Cost Center → Projects  
    - Paste SQL query for project and WIP data.  
    - Configure DB credentials.

12. **Create Employees MySQL Node**  
    - Type: MySQL  
    - Connect Selected Cost Center → Employees  
    - Paste SQL query for employee stats.  
    - Configure DB credentials.

13. **Create CostCenter Code Node**  
    - Type: Code  
    - Connect YTD vs Prevoius Month1 → CostCenter  
    - Paste JS code to convert JSON rows to HTML table.

14. **Create verticalPL Code Node**  
    - Type: Code  
    - Connect Departments → verticalPL  
    - Paste same JS code for HTML table conversion.

15. **Create WIP1 Code Node**  
    - Type: Code  
    - Connect Projects → WIP1  
    - Paste same JS code for HTML table conversion.

16. **Create Employees1 Code Node**  
    - Type: Code  
    - Connect Employees → Employees1  
    - Paste same JS code for HTML table conversion.

17. **Create Merge Node**  
    - Type: Merge  
    - Set number of inputs to 4  
    - Connect CostCenter, verticalPL, WIP1, Employees1 → Merge

18. **Create Code Node (HTML Builder)**  
    - Type: Code  
    - Connect Merge → Code  
    - Paste JS code that builds full HTML report with CSS styling and embeds all four tables.

19. **Create HTML Node**  
    - Type: HTML  
    - Connect Code → HTML  
    - Set HTML content to `{{ $json.html }}`

20. **Create Business Performance AI Agent (Analyst) Node**  
    - Type: LangChain Agent  
    - Connect HTML → Business Performance AI Agent (Analyst)  
    - Configure prompt with detailed instructions for financial analysis and report generation (as provided).  
    - Link Google Gemini Chat Model, Calculator, and Think tools.

21. **Create Google Gemini Chat Model Node**  
    - Type: LangChain Language Model  
    - Connect to AI Agent node as language model.  
    - Configure with Gemini API key and model name `models/gemini-2.5-pro-exp-03-25`.

22. **Create Calculator Node**  
    - Type: LangChain Tool Calculator  
    - Connect to AI Agent node as tool.

23. **Create Think Node**  
    - Type: LangChain Tool Think  
    - Connect to AI Agent node as tool.

24. **Create Financial Performance Code Node**  
    - Type: Code  
    - Connect Business Performance AI Agent → Financial Performance  
    - Paste JS code to clean markdown code block markers from AI output.

25. **Create Email Data Set Node**  
    - Type: Set  
    - Connect Financial Performance → Email Data  
    - Assign:  
      - `CostCenter` from Selected Cost Center node  
      - `Email Output` from cleaned HTML output  
      - `For the Month` string with previous month and year.

26. **Create Microsoft Outlook2 Node**  
    - Type: Microsoft Outlook  
    - Connect Email Data → Microsoft Outlook2  
    - Configure with Outlook OAuth2 credentials.  
    - Set recipient email, subject, and HTML body content.

27. **Create Wait Node**  
    - Type: Wait  
    - Connect Microsoft Outlook2 → Wait  
    - Set wait duration in minutes (e.g., 1-5 minutes).

28. **Connect Wait Node back to Loop Over Items**  
    - Connect Wait → Loop Over Items to process next cost center.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                    | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow triggers monthly on the 5th day to automate financial reporting with zero manual work.                                                 | Workflow Overview                                                                                |
| SQL queries are designed for ERPNext MySQL schema but can be adapted to other SQL databases or replaced with Excel, Google Sheets, or APIs.    | Sticky Note1                                                                                    |
| AI analysis uses Google Gemini 2.5 Pro model via LangChain agent with calculator and reasoning tools for precise financial insights.            | Block 2.6 AI-Powered Financial Analysis                                                        |
| Final report is a responsive HTML email with color-coded financial tables and expert commentary, ready for distribution to business managers.  | Block 2.5 Data Transformation and Block 2.7 Email Delivery                                     |
| Author: Amjid Ali, SyncBricks™ – Automation for Everyone. Support and credits requested when sharing.                                           | Sticky Note11                                                                                   |
| Video tutorial available: https://youtu.be/yatQpQZLqg4                                                                                          | Workflow Description                                                                            |
| n8n Cloud recommended for ease of use: https://n8n.syncbricks.com                                                                              | Workflow Description                                                                            |
| Guidebook and full course links: https://lms.syncbricks.com/books/n8n and https://lms.syncbricks.com/courses/n8n                               | Workflow Description                                                                            |
| Replace SQL nodes with other data sources as needed; swap AI model to OpenAI GPT, Claude, or DeepSeek; customize report structure and styling. | Workflow Description                                                                            |

---

This documentation provides a detailed, structured reference for understanding, reproducing, and modifying the "Generate Monthly Financial Reports with Gemini AI, SQL, and Outlook" n8n workflow. It covers all nodes, their roles, configurations, and interconnections, enabling advanced users and AI agents to fully grasp the workflow logic and anticipate integration challenges.