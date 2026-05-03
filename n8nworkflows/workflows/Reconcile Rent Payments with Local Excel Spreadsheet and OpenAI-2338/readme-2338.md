Reconcile Rent Payments with Local Excel Spreadsheet and OpenAI

https://n8nworkflows.xyz/workflows/reconcile-rent-payments-with-local-excel-spreadsheet-and-openai-2338


# Reconcile Rent Payments with Local Excel Spreadsheet and OpenAI

### 1. Workflow Overview

This workflow automates the reconciliation of rent payments by comparing bank statements against tenant records stored in a local Excel spreadsheet. It is designed for deployment on a self-hosted n8n instance with access to a local network drive and leverages OpenAI’s GPT-4o model to analyze payment data intelligently.

**Target Use Cases:**  
- Property managers or accounting teams needing to verify tenant rent payments against bank statements.  
- Detecting missed, late, or incorrect rent payments quickly.  
- Generating actionable reports for follow-up without exposing sensitive data externally.  

**Logical Blocks:**  
- **1.1 Input Reception:** Monitor a local network folder for new bank statement CSV files and read their content.  
- **1.2 Variable Initialization:** Define key environment variables such as the location of the tenant/property Excel file.  
- **1.3 AI Processing:** Send the parsed bank statement data to an AI agent which queries tenant and property details from the local spreadsheet, compares payments against expected amounts and due dates, and flags any discrepancies.  
- **1.4 Output Parsing and Reporting:** Parse the AI agent’s structured JSON output, split the alerts into individual items, and append them as rows to an alerts sheet in the Excel workbook for human review.  

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block watches a local folder for newly added bank statement CSV files and reads their contents for processing.

**Nodes Involved:**  
- Watch For Bank Statements  
- Set Variables  
- Get Bank Statement File  
- Get CSV Data  

**Node Details:**  

- **Watch For Bank Statements**  
  - Type: Local File Trigger  
  - Role: Watches a specified folder on the local network for new `.csv` files added.  
  - Configuration: Path set to `/home/node/host_mount/reconciliation_project`, watching for add events, ignoring all files except CSV using pattern `!**/*.csv`.  
  - Inputs: None (trigger node)  
  - Outputs: Emits new file path info when a CSV file is added.  
  - Edge cases: Permissions errors accessing local folders; file system event delays or missed triggers; non-CSV files ignored by filter.  
  - Version: Requires self-hosted n8n for local file system access.  

- **Set Variables**  
  - Type: Set  
  - Role: Defines a variable `spreadsheet_location` pointing to the local Excel file path `/home/node/host_mount/reconciliation_project/reconcilation-workbook.xlsx`.  
  - Inputs: Output from Watch For Bank Statements  
  - Outputs: Passes along file info plus this variable.  
  - Edge cases: Hardcoded path may cause issues if file moved or renamed.  

- **Get Bank Statement File**  
  - Type: Read/Write File  
  - Role: Reads the actual contents of the newly added CSV file using the file path from the trigger node.  
  - Configuration: File selected dynamically from the `path` property emitted by Watch For Bank Statements.  
  - Inputs: From Set Variables  
  - Outputs: File contents for parsing.  
  - Edge cases: File locked or deleted before read; encoding issues.  

- **Get CSV Data**  
  - Type: Extract From File  
  - Role: Parses the CSV content into structured JSON data for downstream processing.  
  - Inputs: File content from Get Bank Statement File  
  - Outputs: Array of bank statement rows with columns like date, reference, money in/out.  
  - Edge cases: Malformed CSV, missing columns, encoding errors.  

---

#### 1.2 AI Processing

**Overview:**  
This core block sends the bank statement data to an AI agent which uses tenant and property data from the Excel file to reconcile payments and identify any issues.

**Nodes Involved:**  
- Reconcile Rental Payments (LangChain Agent)  
- OpenAI Chat Model  
- Get Tenant Details (LangChain ToolCode)  
- Get Property Details (LangChain ToolCode)  
- Structured Output Parser  

**Node Details:**  

- **Reconcile Rental Payments**  
  - Type: LangChain Agent  
  - Role: Central AI agent node tasked with analyzing the bank statement data in the context of tenant and property information to flag payment issues.  
  - Configuration:  
    - Input text dynamically constructed as a markdown table of bank statement line items.  
    - System message instructs the AI to identify missed, late, incorrect payments, tenancy end dates, and remaining fees.  
    - Requires JSON-formatted output adhering to a specific schema for alerts.  
  - Inputs:  
    - Main input: CSV data rows from Get CSV Data  
    - AI model: OpenAI Chat Model  
    - AI tools: Get Tenant Details and Get Property Details nodes for querying spreadsheet data  
    - Output parser: Structured Output Parser to enforce JSON output format  
  - Outputs: Alert items as JSON objects.  
  - Edge cases: Model timeouts, incorrect or incomplete JSON output, mismatches in tenant/property data, handling partial month payments.  
  - Version: Requires n8n version supporting LangChain agent and AI tool nodes.  

- **OpenAI Chat Model**  
  - Type: LangChain LM Chat OpenAI  
  - Role: Provides the GPT-4o model for AI agent’s language model backend.  
  - Configuration: Model set to "gpt-4o" with default options. Credentials linked to an OpenAI account.  
  - Inputs: From Reconcile Rental Payments node’s AI language model input.  
  - Outputs: AI-generated responses.  
  - Edge cases: API key limits, network errors, model latency.  

- **Get Tenant Details**  
  - Type: LangChain ToolCode  
  - Role: Queries the "tenants" sheet in the local Excel file to retrieve tenant information by tenant ID or name.  
  - Configuration: JavaScript code using sheetJS library to load and parse tenants sheet, matching query parameters.  
  - Inputs: Query strings from AI agent.  
  - Outputs: JSON stringified tenant records or "no result" message.  
  - Edge cases: Missing tenant entries, Excel file access errors, malformed queries.  

- **Get Property Details**  
  - Type: LangChain ToolCode  
  - Role: Similar to Get Tenant Details but queries the "properties" sheet by Property ID to retrieve property address, postcode, and type.  
  - Configuration: JavaScript code as above adapted for properties sheet.  
  - Inputs: Property ID queries from AI agent.  
  - Outputs: JSON stringified property records or "no result" message.  
  - Edge cases: Missing property data, file read errors.  

- **Structured Output Parser**  
  - Type: LangChain Structured Output Parser  
  - Role: Enforces and parses the AI agent’s JSON response to match the expected schema for alerts.  
  - Configuration: JSON schema example specifying alert attributes such as tenant_id, property_postcode, action_required, etc.  
  - Inputs: AI agent response.  
  - Outputs: Parsed JSON alert objects for further processing.  
  - Edge cases: Parsing errors if AI output is malformed or deviates from schema.  

---

#### 1.3 Output Parsing and Reporting

**Overview:**  
This block processes the AI-generated alerts by splitting them into individual entries and appending them as rows into the alerts sheet of the local Excel file.

**Nodes Involved:**  
- Alert Actions To List  
- Append To Spreadsheet  

**Node Details:**  

- **Alert Actions To List**  
  - Type: Split Out  
  - Role: Takes the array of alert objects from the Structured Output Parser and splits them into individual outputs for row-wise processing.  
  - Inputs: Parsed alert JSON array.  
  - Outputs: Single alert objects one per output item.  
  - Edge cases: Empty alert arrays, malformed inputs.  

- **Append To Spreadsheet**  
  - Type: Code  
  - Role: Uses the sheetJS library to append each alert as a new row in the "alerts" worksheet of the Excel spreadsheet.  
  - Configuration:  
    - Reads spreadsheet_location variable to open the file.  
    - Backs up the original file before writing.  
    - Appends rows with alert date, property ID, postcode, tenant ID, tenant name, action required, and details.  
    - Updates the worksheet range to include new rows.  
    - Writes the file back preserving cell styles and date formats.  
  - Inputs: Individual alert objects from Alert Actions To List.  
  - Outputs: Confirmation message with count of rows added.  
  - Edge cases: File write permission errors, concurrent writes, backup failures, malformed alert data.  
  - Warning: Destructive operation risk (overwriting data); backups advised.  

---

### 3. Summary Table

| Node Name               | Node Type                          | Functional Role                              | Input Node(s)                  | Output Node(s)              | Sticky Note                                                                                                            |
|-------------------------|----------------------------------|----------------------------------------------|-------------------------------|-----------------------------|------------------------------------------------------------------------------------------------------------------------|
| Watch For Bank Statements | Local File Trigger               | Watches local folder for new CSV bank statements | None                          | Set Variables               | Step 1. Wait For Incoming Bank Statements. Read more about [local file triggers](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.localfiletrigger). Locally hosted XLSX used as datastore. |
| Set Variables           | Set                              | Defines path of local Excel spreadsheet       | Watch For Bank Statements      | Get Bank Statement File      |                                                                                                                        |
| Get Bank Statement File | Read/Write File                  | Reads CSV file content from local file path  | Set Variables                 | Get CSV Data                 |                                                                                                                        |
| Get CSV Data            | Extract From File                | Parses CSV content into structured JSON       | Get Bank Statement File       | Reconcile Rental Payments    |                                                                                                                        |
| Reconcile Rental Payments | LangChain Agent                 | AI agent analyzes bank statement vs tenants and flags issues | Get CSV Data, OpenAI Chat Model, Get Tenant Details, Get Property Details, Structured Output Parser | Alert Actions To List        | Step 2. Delegate to AI Agent to Quickly Identify Issues with Rental Payments. Read more about [AI Agents](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent/). |
| OpenAI Chat Model       | LangChain LM Chat OpenAI         | Provides GPT-4o model for AI agent            | Reconcile Rental Payments (ai_languageModel) | Reconcile Rental Payments (ai_languageModel) |                                                                                                                        |
| Get Tenant Details      | LangChain ToolCode               | Queries tenants sheet in Excel for tenant info | Reconcile Rental Payments (ai_tool) | Reconcile Rental Payments (ai_tool) |                                                                                                                        |
| Get Property Details    | LangChain ToolCode               | Queries properties sheet in Excel for property info | Reconcile Rental Payments (ai_tool) | Reconcile Rental Payments (ai_tool) |                                                                                                                        |
| Structured Output Parser | LangChain Structured Output Parser | Parses AI JSON output to alert objects         | Reconcile Rental Payments (ai_outputParser) | Alert Actions To List        |                                                                                                                        |
| Alert Actions To List   | Split Out                       | Splits alert array into individual alert items | Structured Output Parser      | Append To Spreadsheet        |                                                                                                                        |
| Append To Spreadsheet   | Code                            | Adds alert rows to "alerts" sheet in Excel file | Alert Actions To List         | None                        | Step 3. Generate a Report to Action any Issues. Read more about [Code Node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.code). 🚨Warning! Potentially destructive: always backup! |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Watch For Bank Statements" node:**  
   - Type: Local File Trigger  
   - Parameters:  
     - Path: `/home/node/host_mount/reconciliation_project`  
     - Events: Add  
     - Trigger On: Folder  
     - Ignored Files: `!**/*.csv` (to only trigger on CSV files)  
   - Position: (780, 400)  

2. **Create "Set Variables" node:**  
   - Type: Set  
   - Parameters:  
     - Assign variable `spreadsheet_location` with value `/home/node/host_mount/reconciliation_project/reconcilation-workbook.xlsx`  
   - Connect input from Watch For Bank Statements  
   - Position: (940, 400)  

3. **Create "Get Bank Statement File" node:**  
   - Type: Read/Write File  
   - Parameters:  
     - File Selector: Use expression `={{ $('Watch For Bank Statements').item.json.path }}` to dynamically select triggered file  
   - Connect input from Set Variables  
   - Position: (1100, 400)  

4. **Create "Get CSV Data" node:**  
   - Type: Extract From File  
   - Parameters: Use defaults for CSV extraction  
   - Connect input from Get Bank Statement File  
   - Position: (1260, 400)  

5. **Create "OpenAI Chat Model" node:**  
   - Type: LangChain LM Chat OpenAI  
   - Parameters:  
     - Model: `gpt-4o`  
   - Credentials: Link to your OpenAI account credentials  
   - Position: (1540, 540)  

6. **Create "Get Tenant Details" node:**  
   - Type: LangChain ToolCode  
   - Parameters:  
     - Name: `get_tenant_details`  
     - JavaScript Code: Uses `xlsx` library to read tenants sheet and find tenant records by ID or name (see workflow details for full code).  
   - Position: (1660, 540)  

7. **Create "Get Property Details" node:**  
   - Type: LangChain ToolCode  
   - Parameters:  
     - Name: `get_property_details`  
     - JavaScript Code: Uses `xlsx` to read properties sheet and find property records by Property ID.  
   - Position: (1800, 540)  

8. **Create "Structured Output Parser" node:**  
   - Type: LangChain Structured Output Parser  
   - Parameters:  
     - JSON Schema Example: Array of alert objects with fields tenant_id, tenant_name, property_id, property_postcode, action_required, details, date.  
   - Position: (1920, 540)  

9. **Create "Reconcile Rental Payments" node:**  
   - Type: LangChain Agent  
   - Parameters:  
     - Text input: Construct markdown table summarizing bank statement rows using expressions referencing CSV data.  
     - System Message: Instructions to flag missed, late, incorrect payments and tenancy endings, requiring JSON output (see workflow details).  
     - Prompt Type: Define  
     - Enable Output Parser and select Structured Output Parser node.  
   - Connections:  
     - Main input from Get CSV Data  
     - AI Language Model input from OpenAI Chat Model  
     - AI Tool inputs from Get Tenant Details and Get Property Details  
     - AI Output Parser connected to Structured Output Parser  
   - Position: (1640, 360)  

10. **Create "Alert Actions To List" node:**  
    - Type: Split Out  
    - Parameters:  
      - Field To Split Out: `output` (the array of alerts)  
    - Connect input from Reconcile Rental Payments main output  
    - Position: (2260, 400)  

11. **Create "Append To Spreadsheet" node:**  
    - Type: Code  
    - Parameters:  
      - JavaScript code that:  
        - Reads the Excel file path from `spreadsheet_location` variable  
        - Backs up original file by saving a `.bak.xlsx` copy  
        - Appends each alert as a new row to the "alerts" sheet with fields date, property_id, property_postcode, tenant_id, tenant_name, action_required, details  
        - Updates worksheet range accordingly  
        - Writes changes back to the original file preserving styles and dates  
    - Connect input from Alert Actions To List  
    - Position: (2480, 400)  

12. **Connect nodes in order:**  
    - Watch For Bank Statements → Set Variables → Get Bank Statement File → Get CSV Data → Reconcile Rental Payments → Alert Actions To List → Append To Spreadsheet  
    - Connect AI Language Model, AI tools, and output parser nodes to Reconcile Rental Payments as detailed.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                    | Context or Link                                                                                               |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| This workflow requires a self-hosted version of n8n for local filesystem access and to preserve data privacy.                                                                                                                                 | See [n8n docs on local file trigger](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.localfiletrigger) |
| AI Agent uses OpenAI GPT-4o model but can be swapped for self-hosted LLMs supporting function calls.                                                                                                                                            | Ollama version available for fully local setup: https://drive.google.com/file/d/1YRKjfakpInm23F_g8AHupKPBN-fphWgK/view?usp=sharing |
| Backup your Excel spreadsheet before running the workflow to prevent data loss. The Append To Spreadsheet node overwrites the file.                                                                                                           | Warning in Sticky Note with destructive operation caution                                                    |
| Consider integrating with Slack, Teams, or Email to send alerts automatically for improved productivity.                                                                                                                                         | Customization suggestions in workflow description                                                           |
| Join the n8n community on Discord or Forum for support.                                                                                                                                                                                         | Discord: https://discord.com/invite/XPKeKXeB7d; Forum: https://community.n8n.io/                              |
| Uses SheetJS (`xlsx` library) inside code nodes for reading and writing Excel files locally.                                                                                                                                                     | Documentation: https://sheetjs.com/                                                                          |

---

This comprehensive documentation enables replication, adaptation, and troubleshooting of the provided reconciliation workflow for rent payments within n8n.