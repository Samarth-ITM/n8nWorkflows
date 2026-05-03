Generate Multi-Period Financial Reports from Google Sheets with AI Analysis

https://n8nworkflows.xyz/workflows/generate-multi-period-financial-reports-from-google-sheets-with-ai-analysis-6679


# Generate Multi-Period Financial Reports from Google Sheets with AI Analysis

---

## 1. Workflow Overview

This workflow automates the generation of multi-period financial reports by extracting revenue data from Google Sheets, processing it for different time periods, and utilizing AI for analysis. It is designed for company accountants or financial analysts who want to compare financial metrics across current, last month, and last year periods efficiently.

**Target Use Cases:**  
- Automated financial report generation from spreadsheet data  
- Multi-period revenue comparison and aggregation  
- AI-assisted summary and evaluation of financial data  

**Logical Blocks:**  
- **1.1 Input Reception:** Receive and parse date ranges and trigger events for report generation.  
- **1.2 Data Retrieval:** Fetch raw revenue data from Google Sheets.  
- **1.3 Data Processing:** Filter and aggregate revenue data for current period, last month, and last year.  
- **1.4 Data Formatting:** Rename and structure aggregated data for reporting.  
- **1.5 Data Consolidation:** Merge all period data into a single output.  
- **1.6 AI Analysis:** Use AI to interpret and summarize financial data and generate JSON formatted date parameters.  
- **1.7 Sub-Workflow Invocation:** Call a sub-workflow with date parameters for further detailed reporting.  

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception

**Overview:**  
This block listens for incoming chat messages or workflow executions that include date parameters for financial report generation.

**Nodes Involved:**  
- When chat message received  
- When Executed by Another Workflow  
- Format Date  

**Node Details:**  

- **When chat message received**  
  - Type: Chat Trigger (LangChain)  
  - Role: Entry trigger via chat interface.  
  - Config: Webhook ID setup to listen for incoming messages.  
  - Inputs: External chat messages.  
  - Outputs: Passes data to AI Agent.  
  - Failure cases: Webhook errors, connectivity issues.  

- **When Executed by Another Workflow**  
  - Type: Execute Workflow Trigger  
  - Role: Entry when called from another workflow with date inputs.  
  - Config: Accepts inputs for six date parameters (start/end for current, last month, last year).  
  - Inputs: Called workflow inputs.  
  - Outputs: Passes data to Format Date node.  
  - Failure cases: Missing parameters, invalid inputs.  

- **Format Date**  
  - Type: Set Node  
  - Role: Formats and standardizes incoming date strings for downstream nodes.  
  - Config: Assigns input date fields directly to JSON properties without transformation.  
  - Inputs: Data from When Executed by Another Workflow.  
  - Outputs: Passes formatted dates to Google Sheets data retrieval and formatting nodes.  
  - Failure cases: Missing date fields, incorrect formats.  

---

### 2.2 Data Retrieval

**Overview:**  
Fetches raw revenue data from a specific Google Sheets document and sheet.

**Nodes Involved:**  
- Get revenual from google sheet  

**Node Details:**  

- **Get revenual from google sheet**  
  - Type: Google Sheets  
  - Role: Retrieves revenue entries from a spreadsheet for further processing.  
  - Config:  
    - Document ID: "1UAMO24QtfkR50VGu1wpksuYiWffi28pHfm6bihd5IP8"  
    - Sheet ID: 1168390826 (named "Doanh thu theo ngày")  
  - Credentials: Google Sheets OAuth2 account required.  
  - Inputs: Receives date parameters from Format Date node.  
  - Outputs: Raw revenue data items passed to pivot nodes for filtering and aggregation.  
  - Failure cases: Authentication errors, API limit errors, missing document or sheet, network issues.  

---

### 2.3 Data Processing

**Overview:**  
Filters and aggregates the revenue data for three distinct periods: current period, last month, and last year, by parsing dates and summing amounts by category.

**Nodes Involved:**  
- Pivot current circle  
- Pivot last circle  
- Pivot last year cirle  

**Node Details:**  

- **Pivot current circle**  
  - Type: Code  
  - Role: Filters data for current period date range and aggregates sums by category ("Phân loại").  
  - Config:  
    - JavaScript code parses Start_Current and End_Current dates from Format Date node.  
    - Converts date strings (formats DD/MM/YYYY or YYYY-MM-DD) to Date objects.  
    - Iterates over all input data, filters by date range, normalizes category strings, and sums "Số tiền".  
  - Inputs: Raw data from Google Sheets node.  
  - Outputs: Aggregated array of objects with category and total sum.  
  - Failure cases: Invalid date formats, missing data fields, code exceptions.  
  - Error handling: Continues regular output on error.  

- **Pivot last circle**  
  - Type: Code  
  - Role: Same as Pivot current circle but for last month date range (Start_LastMonth, End_LastMonth).  
  - Config: Similar logic adapted for last month's date range.  
  - Inputs: Raw data from Google Sheets node.  
  - Outputs: Aggregated sums for last month.  
  - Failure cases: Same as above.  

- **Pivot last year cirle**  
  - Type: Code  
  - Role: Same aggregation logic for last year date range (Start_LastYear, End_LastYear).  
  - Config: Similar parsing and aggregation code.  
  - Inputs: Raw data from Google Sheets node.  
  - Outputs: Aggregated sums for last year.  
  - Failure cases: Same as above.  

---

### 2.4 Data Formatting

**Overview:**  
Formats the aggregated data arrays into structured JSON objects with descriptive field names and adds the corresponding date range metadata.

**Nodes Involved:**  
- Format data current circle  
- Format data last circle  
- Format data last year circle  
- Sum current circle  
- Sum last circle  
- Sum last year circle  
- Change title 1  
- Change title 2  
- Change title  

**Node Details:**  

- **Format data current circle / last circle / last year circle**  
  - Type: Set  
  - Role: Add date range metadata fields (e.g., currentCycle.startDate) from Format Date node to each data set.  
  - Config: Assigns startDate and endDate strings from formatted dates.  
  - Inputs: From Google Sheet pivot nodes.  
  - Outputs: Passes enriched data to summing nodes.  
  - Failure cases: Missing date fields.  

- **Sum current circle / last circle / last year circle**  
  - Type: Aggregate  
  - Role: Aggregates all incoming data items into a single object for summarization.  
  - Config: Uses 'aggregateAllItemData' operation to combine all JSON objects.  
  - Inputs: From respective format data nodes.  
  - Outputs: Summed data objects passed to merge nodes.  
  - Failure cases: Empty inputs, malformed JSON.  

- **Change title 1 / 2 / (Change title)**  
  - Type: Aggregate  
  - Role: Rename aggregated data fields to meaningful titles:  
    - Change title 1 → "Chu kỳ hiện tại" (Current cycle)  
    - Change title 2 → "Chu kỳ tháng trước" (Last month cycle)  
    - Change title → "Chu kỳ năm trước" (Last year cycle)  
  - Config: Uses 'aggregateAllItemData' with destinationFieldName to rename.  
  - Inputs: From respective sum nodes.  
  - Outputs: Passes renamed objects to merge.  
  - Failure cases: Missing data fields.  

---

### 2.5 Data Consolidation

**Overview:**  
Merges all renamed and aggregated period data into a single consolidated dataset and applies a final title for reporting.

**Nodes Involved:**  
- Sum data current transfer month  
- Sum data last transfer month  
- Add data last year  
- Collect all  
- Change title out come  

**Node Details:**  

- **Sum data current transfer month**  
  - Type: Merge (default mode)  
  - Role: Merges aggregated current cycle data.  
  - Inputs: From Sum current circle and Change title 1 nodes.  
  - Outputs: Passes to Change title 1 node.  

- **Sum data last transfer month**  
  - Type: Merge  
  - Role: Merges aggregated last month data.  
  - Inputs: From Sum last circle and Change title 2 nodes.  
  - Outputs: Passes to Change title 2 node.  

- **Add data last year**  
  - Type: Merge  
  - Role: Merges aggregated last year data with renamed fields.  
  - Inputs: From Sum last year circle and Format data last year circle.  
  - Outputs: Passes to Change title node.  

- **Collect all**  
  - Type: Merge (numberInputs: 3)  
  - Role: Combines all three period data objects into one collection.  
  - Inputs: From Change title 1, Change title 2, and Change title nodes.  
  - Outputs: Passes to final aggregate.  

- **Change title out come**  
  - Type: Aggregate  
  - Role: Aggregates all collected data and renames the output field to "data doanh thu" (revenue data).  
  - Inputs: From Collect all node.  
  - Outputs: Final consolidated output for reporting.  
  - Failure cases: Empty data, merge conflicts.  

---

### 2.6 AI Analysis

**Overview:**  
Uses LangChain AI components to interpret financial data, generate date parameters in JSON, and guide the report generation.

**Nodes Involved:**  
- When chat message received  
- DeepSeek Chat Model  
- Window Buffer Memory1  
- AI Agent  
- Call n8n Workflow Tool  

**Node Details:**  

- **When chat message received**  
  - As described in Input Reception, triggers AI processing on chat input.  

- **DeepSeek Chat Model**  
  - Type: LangChain DeepSeek Chat Model  
  - Role: Provides AI language model backend for the agent.  
  - Config: Uses DeepSeek API credentials.  
  - Inputs: AI Agent node.  
  - Outputs: AI Agent receives model responses.  
  - Failure cases: API key invalid, rate limits, network failure.  

- **Window Buffer Memory1**  
  - Type: LangChain Memory Buffer Window  
  - Role: Maintains chat memory context for conversation continuity.  
  - Inputs: AI Agent.  
  - Outputs: AI Agent for context-aware responses.  
  - Failure cases: Memory overflow, context limit.  

- **AI Agent**  
  - Type: LangChain Agent  
  - Role: Main AI logic processing node that:  
    - Acts as company accountant.  
    - Summarizes revenue and expense info.  
    - Compares periods.  
    - Returns JSON with date parameters in dd/MM/yyyy format.  
  - Config: System message defines role and output format.  
  - Inputs: Triggered from chat message received, uses DeepSeek Chat Model and Window Buffer Memory.  
  - Outputs: Passes JSON date parameters to Call n8n Workflow Tool.  
  - Failure cases: Parsing errors, malformed AI output.  

- **Call n8n Workflow Tool**  
  - Type: LangChain Tool Workflow  
  - Role: Calls a sub-workflow titled "Sub flowBao cao doanh thu" with date parameters generated by AI Agent.  
  - Config: Passes six date parameters (Start_Current, End_Current, etc.) as inputs.  
  - Inputs: AI Agent output.  
  - Outputs: Sub-workflow execution.  
  - Failure cases: Sub-workflow not found, parameter mismatch, execution errors.  

---

### 2.7 Sub-Workflow Invocation

**Overview:**  
This node calls another workflow (ID: 4ASVA3i1vaFwj20h) responsible for more detailed revenue reporting based on the AI-generated date parameters.

**Node Involved:**  
- Call n8n Workflow Tool (from AI Analysis block)  

**Sub-Workflow Details:**  
- Input parameters: Start_Current, End_Current, Start_LastMonth, End_LastMonth, Start_LastYear, End_LastYear (all strings, date format dd/MM/yyyy)  
- Expected output: Presumably detailed financial reports (not included in this workflow)  
- Note: Sticky Note "Sub-Workflow: Google Analytics Data" indicates the sub-workflow handles Google Analytics data for reporting.  

---

## 3. Summary Table

| Node Name                      | Node Type                          | Functional Role                              | Input Node(s)                         | Output Node(s)                         | Sticky Note                           |
|--------------------------------|----------------------------------|----------------------------------------------|-------------------------------------|--------------------------------------|-------------------------------------|
| When chat message received      | LangChain Chat Trigger           | Entry point for chat-based report requests   | -                                   | AI Agent                             |                                     |
| When Executed by Another Workflow| Execute Workflow Trigger         | Entry when called by another workflow        | -                                   | Format Date                         |                                     |
| Format Date                    | Set                              | Standardizes input date parameters            | When Executed by Another Workflow    | Get revenual from google sheet, Format data nodes |                                     |
| Get revenual from google sheet  | Google Sheets                    | Retrieves raw revenue data                      | Format Date                        | Pivot current circle, Pivot last circle, Pivot last year cirle |                                     |
| Pivot current circle           | Code                             | Filter and aggregate current period data      | Get revenual from google sheet       | Sum current circle                  |                                     |
| Pivot last circle              | Code                             | Filter and aggregate last month data          | Get revenual from google sheet       | Sum last circle                     |                                     |
| Pivot last year cirle          | Code                             | Filter and aggregate last year data            | Get revenual from google sheet       | Sum last year circle                |                                     |
| Format data current circle     | Set                              | Add current cycle date metadata                | Format Date                         | Sum data current transfer month     |                                     |
| Format data last circle        | Set                              | Add last month cycle date metadata             | Format Date                         | Sum data last transfer month        |                                     |
| Format data last year circle   | Set                              | Add last year cycle date metadata              | Format Date                         | Add data last year                  |                                     |
| Sum current circle             | Aggregate                       | Aggregate all current period data               | Pivot current circle                 | Sum data current transfer month     |                                     |
| Sum last circle                | Aggregate                       | Aggregate all last month data                    | Pivot last circle                   | Sum data last transfer month        |                                     |
| Sum last year circle           | Aggregate                       | Aggregate all last year data                     | Pivot last year cirle               | Add data last year                  |                                     |
| Sum data current transfer month| Merge                          | Merge current cycle aggregated data             | Sum current circle, Format data current circle | Change title 1                    |                                     |
| Sum data last transfer month   | Merge                          | Merge last month aggregated data                 | Sum last circle, Format data last circle | Change title 2                     |                                     |
| Add data last year             | Merge                          | Merge last year aggregated data                  | Sum last year circle, Format data last year circle | Change title                   |                                     |
| Change title 1                | Aggregate                       | Rename current cycle field to "Chu kỳ hiện tại" | Sum data current transfer month     | Collect all                        |                                     |
| Change title 2                | Aggregate                       | Rename last month field to "Chu kỳ tháng trước" | Sum data last transfer month        | Collect all                        |                                     |
| Change title                 | Aggregate                       | Rename last year field to "Chu kỳ năm trước"    | Add data last year                  | Collect all                        |                                     |
| Collect all                  | Merge (3 inputs)                | Merge all period data into one collection        | Change title 1, Change title 2, Change title | Change title out come             |                                     |
| Change title out come        | Aggregate                      | Final aggregation and title "data doanh thu"     | Collect all                        | -                                  |                                     |
| DeepSeek Chat Model          | LangChain DeepSeek Chat Model  | AI language model backend                        | AI Agent                          | AI Agent                           |                                     |
| Window Buffer Memory1        | LangChain Memory Buffer Window | Maintains AI chat context                        | AI Agent                          | AI Agent                           |                                     |
| AI Agent                    | LangChain Agent                | AI logic for financial summary and JSON date generation | When chat message received, DeepSeek Chat Model, Window Buffer Memory1 | Call n8n Workflow Tool            |                                     |
| Call n8n Workflow Tool       | LangChain Tool Workflow        | Calls sub-workflow with AI-generated dates       | AI Agent                          | Sub-workflow (ID: 4ASVA3i1vaFwj20h) | Sticky Note: "Sub-Workflow: Google Analytics Data" |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**  
   - Create a **LangChain Chat Trigger** node named "When chat message received" with a webhook to receive chat inputs.  
   - Create an **Execute Workflow Trigger** node named "When Executed by Another Workflow" configured to accept inputs: Start_Current, End_Current, Start_LastMonth, End_LastMonth, Start_LastYear, End_LastYear (all strings).  

2. **Create Date Formatting Node:**  
   - Add a **Set** node named "Format Date". Assign input parameters directly to JSON properties (Start_Current, End_Current, etc.). No transformations needed. Connect "When Executed by Another Workflow" to "Format Date".  

3. **Add Google Sheets Node:**  
   - Add a **Google Sheets** node named "Get revenual from google sheet". Configure with:  
     - Document ID: "1UAMO24QtfkR50VGu1wpksuYiWffi28pHfm6bihd5IP8"  
     - Sheet ID: 1168390826 (sheet name "Doanh thu theo ngày")  
     - Credentials: Google Sheets OAuth2 account with access to the document.  
   - Connect "Format Date" to this node.  

4. **Create Pivot Nodes for Each Period:**  
   For each period (current, last month, last year), add a **Code** node with the following:  
   - Parse respective start/end dates from "Format Date" node.  
   - Parse dates from input data (field "Date", formats DD/MM/YYYY or YYYY-MM-DD).  
   - Filter data in date range, normalize "Phân loại" field, sum "Số tiền" by category.  
   - Return an array of objects with fields "Phân loại" and "Tổng số tiền".  
   - Set error handling to "Continue Regular Output".  
   - Connect "Get revenual from google sheet" output to each pivot node.  
   - Name nodes:  
     - "Pivot current circle" (uses Start_Current, End_Current)  
     - "Pivot last circle" (uses Start_LastMonth, End_LastMonth)  
     - "Pivot last year cirle" (uses Start_LastYear, End_LastYear)  

5. **Add Set Nodes to Format Date Metadata:**  
   - For each period, create a **Set** node to assign startDate and endDate fields from "Format Date" node:  
     - "Format data current circle" → fields currentCycle.startDate, currentCycle.endDate  
     - "Format data last circle" → fields lastMonthCycle.startDate, lastMonthCycle.endDate  
     - "Format data last year circle" → fields lastYearCycle.startDate, lastYearCycle.endDate  
   - Connect "Format Date" node to each of these.  

6. **Add Aggregate Nodes to Sum Pivot Outputs:**  
   - Add **Aggregate** nodes to combine all data items:  
     - "Sum current circle" connected to "Pivot current circle"  
     - "Sum last circle" connected to "Pivot last circle"  
     - "Sum last year circle" connected to "Pivot last year cirle"  

7. **Add Merge Nodes to Combine Sum and Date Metadata:**  
   - Create **Merge** nodes to merge sum with date metadata:  
     - "Sum data current transfer month": merge "Sum current circle" and "Format data current circle"  
     - "Sum data last transfer month": merge "Sum last circle" and "Format data last circle"  
     - "Add data last year": merge "Sum last year circle" and "Format data last year circle"  

8. **Add Title Change Aggregate Nodes:**  
   - For each period, add **Aggregate** node to rename the field:  
     - "Change title 1": rename field to "Chu kỳ hiện tại" (input: "Sum data current transfer month")  
     - "Change title 2": rename field to "Chu kỳ tháng trước" (input: "Sum data last transfer month")  
     - "Change title": rename field to "Chu kỳ năm trước" (input: "Add data last year")  

9. **Merge All Period Data:**  
   - Add a **Merge** node named "Collect all" configured for 3 inputs.  
   - Connect outputs of "Change title 1", "Change title 2", and "Change title" to this node.  

10. **Final Aggregate Node:**  
    - Add **Aggregate** node named "Change title out come".  
    - Configure to aggregate all items and rename output field to "data doanh thu".  
    - Connect "Collect all" to this node.  

11. **Add AI Components:**  
    - Add a **LangChain DeepSeek Chat Model** node named "DeepSeek Chat Model" and configure with DeepSeek API credentials.  
    - Add **LangChain Window Buffer Memory** node named "Window Buffer Memory1".  
    - Add **LangChain Agent** node named "AI Agent" with system message:  
      ```
      You are the company accountant. Summarize revenue and expense info from the tool, evaluate, compare, and respond accordingly. Include all reporting info: Start_Current, End_Current, Start_LastMonth, End_LastMonth, Start_LastYear, End_LastYear in dd/MM/yyyy format. Output JSON only.
      ```  
      Set "Has Output Parser" to true.  
    - Connect:  
      - "When chat message received" → "AI Agent"  
      - "DeepSeek Chat Model" → "AI Agent" (ai_languageModel)  
      - "Window Buffer Memory1" → "AI Agent" (ai_memory)  

12. **Call Sub-Workflow Tool:**  
    - Add **LangChain Tool Workflow** node named "Call n8n Workflow Tool".  
    - Configure to call workflow ID "4ASVA3i1vaFwj20h" (Sub flowBao cao doanh thu).  
    - Map inputs to AI Agent outputs: Start_Current, End_Current, Start_LastMonth, End_LastMonth, Start_LastYear, End_LastYear.  
    - Connect output of "AI Agent" to "Call n8n Workflow Tool" (ai_tool).  

13. **Link Nodes for Full Flow:**  
    - For the final reporting logic, connect the "Format Date" node to Google Sheets and all pivots and format data nodes.  
    - Connect the pivot nodes to their sum nodes, then merges, then change title nodes, then collect all and final aggregate node.  
    - The AI path is triggered independently on chat message; it generates date parameters and calls the sub-workflow.  

14. **Credential Setup:**  
    - Google Sheets OAuth2 credentials with access to the target spreadsheet.  
    - DeepSeek API credentials for AI model usage.  

15. **Testing and Validation:**  
    - Test date input formats for all periods; ensure proper parsing.  
    - Check Google Sheets data access and correct sheet selection.  
    - Validate AI agent responses and JSON outputs.  
    - Verify sub-workflow execution with correct date parameters.  

---

## 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                         |
|----------------------------------------------------------------------------------------------------------|---------------------------------------|
| Sub-Workflow: Google Analytics Data                                                                       | Sticky Note in workflow, related to sub-workflow called by "Call n8n Workflow Tool" node. |
| Date formats accepted: dd/MM/yyyy or ISO yyyy-MM-dd                                                       | Important for date parsing in pivot code nodes. |
| AI Agent system message enforces JSON output for date ranges with strict formatting requirements.         | Ensures downstream nodes receive consistent input. |
| Google Sheets document: https://docs.google.com/spreadsheets/d/1UAMO24QtfkR50VGu1wpksuYiWffi28pHfm6bihd5IP8 | Source of financial data for aggregation. |
| DeepSeek API used as AI language model backend; requires API key and valid subscription.                  | Credential setup required for AI nodes. |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---