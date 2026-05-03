Compare Multi-Period Financial Data from Google Sheets with DeepSeek AI Analysis

https://n8nworkflows.xyz/workflows/compare-multi-period-financial-data-from-google-sheets-with-deepseek-ai-analysis-6023


# Compare Multi-Period Financial Data from Google Sheets with DeepSeek AI Analysis

### 1. Workflow Overview

This workflow titled **"Compare Multi-Period Financial Data from Google Sheets with DeepSeek AI Analysis"** is designed for an accounting or financial reporting context. It automates the process of retrieving revenue and expense data from Google Sheets, aggregates this data over multiple time periods (current cycle, last month cycle, and last year cycle), and uses AI to analyze, compare, and summarize the financial results.

**Target Use Cases:**  
- Financial analysts or accountants needing automated multi-period financial summaries.  
- Real-time chat-triggered financial insights based on up-to-date Google Sheets data.  
- Integration of AI-driven reporting and comparison of financial metrics over time.

**Logical Blocks:**

- **1.1 Input Reception & AI Processing:**  
  Receives chat messages to trigger AI-based analysis, manages conversational context, and invokes an AI agent integrated with DeepSeek for summarization and analysis.

- **1.2 Date Formatting & Workflow Trigger:**  
  Handles input dates for various periods and triggers the data aggregation workflow with these parameters.

- **1.3 Data Retrieval from Google Sheets:**  
  Pulls raw financial data (revenue and expenses) from a specific Google Sheet and sheet tab.

- **1.4 Data Aggregation & Pivoting:**  
  Processes raw data into aggregated sums grouped by "Phân loại" (Category), split across the current cycle, last month, and last year.

- **1.5 Data Merging and Final Formatting:**  
  Combines aggregated data from all periods and formats it for final output to the AI agent.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception & AI Processing

**Overview:**  
Handles chat messages as triggers, manages conversation memory with a window buffer, calls a sub-workflow tool with date parameters, and sends data to an AI Agent powered by DeepSeek.

**Nodes Involved:**  
- When chat message received  
- Window Buffer Memory1  
- Call n8n Workflow Tool  
- DeepSeek Chat Model  
- AI Agent

**Node Details:**

- **When chat message received**  
  - *Type:* Chat trigger node  
  - *Role:* Entry point for chat-based input that initiates the workflow.  
  - *Config:* Listens for incoming chat messages via webhook.  
  - *Connections:* Outputs to AI Agent.  
  - *Failure cases:* Webhook failure, invalid payload.  

- **Window Buffer Memory1**  
  - *Type:* Memory buffer node (Langchain)  
  - *Role:* Maintains conversation context within a window for the AI Agent.  
  - *Config:* Uses default window size; no special parameters.  
  - *Connections:* Connected as AI memory input to AI Agent.  
  - *Failure cases:* Memory overflow or corrupted state.

- **Call n8n Workflow Tool**  
  - *Type:* Tool invocation node (Langchain)  
  - *Role:* Calls a sub-workflow (ID: 4ASVA3i1vaFwj20h) with date parameters extracted from AI inputs.  
  - *Config:* Passes "Start_Current", "End_Current", "Start_LastMonth", "End_LastMonth", "Start_LastYear", "End_LastYear" as strings.  
  - *Connections:* Tool output feeds into AI Agent.  
  - *Failure cases:* Sub-workflow not found, parameter mismatch, execution errors.

- **DeepSeek Chat Model**  
  - *Type:* DeepSeek language model node  
  - *Role:* Provides AI language model services via DeepSeek API.  
  - *Config:* Uses stored DeepSeek API credentials; default model options.  
  - *Connections:* Feeds AI Agent node.  
  - *Failure cases:* API auth failure, rate limits, network issues.

- **AI Agent**  
  - *Type:* Langchain AI agent node  
  - *Role:* Core AI node that processes input, runs the system prompt, calls tools, and returns formatted JSON output.  
  - *Config:* System message instructs the AI to act as an accountant summarizing revenues and expenses for given periods, outputting JSON with specific date keys. Uses output parser to enforce JSON response.  
  - *Connections:* Input from chat trigger, memory, and tool nodes; outputs final AI response.  
  - *Failure cases:* Parsing errors, AI response format errors, timeout.

---

#### 2.2 Date Formatting & Workflow Trigger

**Overview:**  
Receives execution triggers from other workflows with date inputs, formats date parameters, and passes them downstream for data retrieval and aggregation.

**Nodes Involved:**  
- When Executed by Another Workflow  
- Format Date

**Node Details:**

- **When Executed by Another Workflow**  
  - *Type:* Execute Workflow Trigger node  
  - *Role:* Receives external workflow execution calls with date inputs as parameters.  
  - *Config:* Expects parameters: Start_Current, End_Current, Start_LastMonth, End_LastMonth, Start_LastYear, End_LastYear.  
  - *Connections:* Outputs to Format Date node.  
  - *Failure cases:* Missing parameters, invalid input data.

- **Format Date**  
  - *Type:* Set node  
  - *Role:* Maps incoming date inputs directly into workflow JSON fields without transformation.  
  - *Config:* Assigns all received date parameters directly as strings preserving format.  
  - *Connections:* Outputs to Google Sheets data retrieval and subsequent formatting nodes.  
  - *Failure cases:* Null or improperly formatted input data.

---

#### 2.3 Data Retrieval from Google Sheets

**Overview:**  
Retrieves raw financial data from a specified Google Sheets document and sheet tab, serving as the source for all financial aggregation.

**Nodes Involved:**  
- Get revenual from google sheet

**Node Details:**

- **Get revenual from google sheet**  
  - *Type:* Google Sheets node  
  - *Role:* Extracts financial data rows from a Google Sheets document identified by document ID and sheet tab (sheet ID 1168390826).  
  - *Config:* Reads all rows without filters or transformations.  
  - *Connections:* Outputs data to pivoting nodes for each of the three periods.  
  - *Credentials:* Uses Google Sheets OAuth2 credentials.  
  - *Failure cases:* API auth failure, sheet permission issues, network errors.

---

#### 2.4 Data Aggregation & Pivoting

**Overview:**  
Aggregates financial data by "Phân loại" (Category) and sums "Số tiền" (Amount) for each of the three time periods: current cycle, last month, and last year. Uses custom JavaScript code nodes for filtering and grouping.

**Nodes Involved:**  
- Pivot current circle  
- Sum current circle  
- Format data current circle  
- Sum data current transfer month  
- Pivot last circle  
- Sum last circle  
- Format data last circle  
- Sum data last transfer month  
- Pivot last year cirle  
- Sum last year circle  
- Format data last year circle

**Node Details:**

- **Pivot current circle**  
  - *Type:* Code node (JavaScript)  
  - *Role:* Filters and sums financial data for the current cycle dates (Start_Current to End_Current).  
  - *Config:* Parses dates, validates format, normalizes category strings, aggregates sums by category.  
  - *Connections:* Outputs to Sum current circle.  
  - *Failure cases:* Date parsing errors, missing data, invalid date formats.

- **Sum current circle**  
  - *Type:* Aggregate node  
  - *Role:* Aggregates all item data (effectively passes through aggregated data).  
  - *Connections:* Outputs to Sum data current transfer month.  
  - *Failure cases:* Internal aggregation errors.

- **Format data current circle**  
  - *Type:* Set node  
  - *Role:* Sets JSON fields for current cycle start and end dates for downstream merging.  
  - *Connections:* Outputs to Sum data current transfer month.  
  - *Failure cases:* Missing input data.

- **Sum data current transfer month**  
  - *Type:* Merge node  
  - *Role:* Prepares data for merging by collecting current cycle aggregation results.  
  - *Connections:* Outputs to Change title 1.  
  - *Failure cases:* Merge conflicts.

- **Pivot last circle**  
  - *Type:* Code node (JavaScript)  
  - *Role:* Same logic as Pivot current circle but filters for last month cycle (Start_LastMonth to End_LastMonth).  
  - *Connections:* Outputs to Sum last circle.  
  - *Failure cases:* Same as current circle pivot.

- **Sum last circle**  
  - *Type:* Aggregate node  
  - *Role:* Aggregates last month data.  
  - *Connections:* Outputs to Sum data last transfer month.  
  - *Failure cases:* Internal errors.

- **Format data last circle**  
  - *Type:* Set node  
  - *Role:* Sets last month cycle start and end dates in JSON for downstream use.  
  - *Connections:* Outputs to Sum data last transfer month.  
  - *Failure cases:* Missing input data.

- **Sum data last transfer month**  
  - *Type:* Merge node  
  - *Role:* Prepares last month aggregated data for later merging.  
  - *Connections:* Outputs to Change title 2.  
  - *Failure cases:* Merge conflicts.

- **Pivot last year cirle**  
  - *Type:* Code node (JavaScript)  
  - *Role:* Filters and aggregates data for last year cycle (Start_LastYear to End_LastYear) with same logic as other pivots.  
  - *Connections:* Outputs to Sum last year circle.  
  - *Failure cases:* Date format errors, missing data.

- **Sum last year circle**  
  - *Type:* Aggregate node  
  - *Role:* Aggregates last year data.  
  - *Connections:* Outputs to Add data last year.  
  - *Failure cases:* Internal errors.

- **Format data last year circle**  
  - *Type:* Set node  
  - *Role:* Sets last year cycle start and end dates in JSON for downstream use.  
  - *Connections:* Outputs to Add data last year.  
  - *Failure cases:* Missing data.

---

#### 2.5 Data Merging and Final Formatting

**Overview:**  
Combines aggregated data from all three periods (current, last month, last year), renames aggregated fields for clarity, and prepares the final dataset for output or further AI processing.

**Nodes Involved:**  
- Change title 1  
- Change title 2  
- Change title  
- Collect all  
- Change title out come  
- Add data last year

**Node Details:**

- **Change title 1**  
  - *Type:* Aggregate node  
  - *Role:* Renames aggregated data from current cycle as "Chu kỳ hiện tại" (Current Cycle).  
  - *Connections:* Outputs to Collect all.  
  - *Failure cases:* Aggregation errors.

- **Change title 2**  
  - *Type:* Aggregate node  
  - *Role:* Renames last month cycle aggregated data as "Chu kỳ tháng trước" (Last Month Cycle).  
  - *Connections:* Outputs to Collect all.  
  - *Failure cases:* Aggregation errors.

- **Change title**  
  - *Type:* Aggregate node  
  - *Role:* Renames last year cycle aggregated data as "Chu kỳ năm trước" (Last Year Cycle).  
  - *Connections:* Outputs to Collect all.  
  - *Failure cases:* Aggregation errors.

- **Collect all**  
  - *Type:* Merge node (3 inputs)  
  - *Role:* Merges the three renamed datasets into a single combined dataset.  
  - *Connections:* Outputs to Change title out come.  
  - *Failure cases:* Merge conflicts.

- **Change title out come**  
  - *Type:* Aggregate node  
  - *Role:* Final aggregation and renaming, sets destination field as "data doanh thu" (revenue data).  
  - *Connections:* Final node in data processing chain.  
  - *Failure cases:* Aggregation errors.

- **Add data last year**  
  - *Type:* Merge node  
  - *Role:* Merges last year aggregation with previous steps before renaming.  
  - *Connections:* Outputs to Change title.  
  - *Failure cases:* Merge conflicts.

---

### 3. Summary Table

| Node Name                     | Node Type                         | Functional Role                                | Input Node(s)                     | Output Node(s)                     | Sticky Note                     |
|-------------------------------|----------------------------------|------------------------------------------------|----------------------------------|----------------------------------|--------------------------------|
| When chat message received     | Chat Trigger                     | Entry point for chat-triggered AI analysis     | -                                | AI Agent                         |                                |
| Window Buffer Memory1          | Memory Buffer (Langchain)        | Maintains conversation context                  | -                                | AI Agent                         |                                |
| Call n8n Workflow Tool         | Tool Workflow Invocation         | Calls sub-workflow with date parameters         | -                                | AI Agent                         |                                |
| DeepSeek Chat Model            | AI Language Model (DeepSeek)     | Provides AI language model computation           | -                                | AI Agent                         |                                |
| AI Agent                      | Langchain AI Agent              | Processes chat inputs, runs AI logic, outputs JSON | When chat message received, Window Buffer Memory1, Call n8n Workflow Tool | -                                |                                |
| When Executed by Another Workflow | Execute Workflow Trigger         | Receives external workflow execution with dates | -                                | Format Date                      |                                |
| Format Date                   | Set Node                        | Formats input date parameters into JSON fields  | When Executed by Another Workflow | Get revenual from google sheet, Format data current circle, Format data last circle, Format data last year circle |                                |
| Get revenual from google sheet | Google Sheets                   | Retrieves raw financial data rows                 | Format Date                      | Pivot current circle, Pivot last circle, Pivot last year cirle |                                |
| Pivot current circle          | Code (JavaScript)               | Filters and aggregates current cycle data       | Get revenual from google sheet   | Sum current circle               |                                |
| Sum current circle            | Aggregate                      | Aggregates current cycle data                     | Pivot current circle             | Sum data current transfer month  |                                |
| Format data current circle    | Set                           | Sets current cycle start/end dates                | Format Date                      | Sum data current transfer month  |                                |
| Sum data current transfer month | Merge                         | Prepares current cycle aggregated data for merging | Sum current circle, Format data current circle | Change title 1                 |                                |
| Pivot last circle             | Code (JavaScript)               | Filters and aggregates last month cycle data     | Get revenual from google sheet   | Sum last circle                 |                                |
| Sum last circle               | Aggregate                      | Aggregates last month cycle data                  | Pivot last circle                | Sum data last transfer month    |                                |
| Format data last circle       | Set                           | Sets last month cycle start/end dates              | Format Date                      | Sum data last transfer month    |                                |
| Sum data last transfer month  | Merge                         | Prepares last month aggregated data for merging   | Sum last circle, Format data last circle | Change title 2                 |                                |
| Pivot last year cirle         | Code (JavaScript)               | Filters and aggregates last year cycle data      | Get revenual from google sheet   | Sum last year circle            |                                |
| Sum last year circle          | Aggregate                      | Aggregates last year cycle data                    | Pivot last year cirle            | Add data last year              |                                |
| Format data last year circle  | Set                           | Sets last year cycle start/end dates               | Format Date                      | Add data last year              |                                |
| Add data last year            | Merge                         | Merges last year data with other aggregated results | Sum last year circle, Format data last year circle | Change title                 |                                |
| Change title 1                | Aggregate                      | Renames current cycle data field                   | Sum data current transfer month  | Collect all                    |                                |
| Change title 2                | Aggregate                      | Renames last month cycle data field                | Sum data last transfer month     | Collect all                    |                                |
| Change title                 | Aggregate                      | Renames last year cycle data field                 | Add data last year               | Collect all                    |                                |
| Collect all                  | Merge                         | Merges current, last month, and last year data    | Change title 1, Change title 2, Change title | Change title out come         |                                |
| Change title out come        | Aggregate                      | Final aggregation and renaming of combined data   | Collect all                     | -                              |                                |
| Sticky Note                  | Sticky Note                   | Comment: "## Sub-Workflow: Google Analytics Data" | -                                | -                              | Covers sub-workflow context    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Chat Trigger Node: "When chat message received"**  
   - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Configure webhook to receive chat messages.

2. **Create Conversation Memory Node: "Window Buffer Memory1"**  
   - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
   - Use default settings.

3. **Create Tool Invocation Node: "Call n8n Workflow Tool"**  
   - Type: `@n8n/n8n-nodes-langchain.toolWorkflow`  
   - Set the workflow ID to the sub-workflow that handles data aggregation (`4ASVA3i1vaFwj20h`).  
   - Define inputs: Start_Current, End_Current, Start_LastMonth, End_LastMonth, Start_LastYear, End_LastYear as strings.  
   - Map these inputs to values extracted from AI output variables.

4. **Create AI Language Model Node: "DeepSeek Chat Model"**  
   - Type: `@n8n/n8n-nodes-langchain.lmChatDeepSeek`  
   - Provide DeepSeek API credentials.  
   - Use default model options.

5. **Create AI Agent Node: "AI Agent"**  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - System message: instruct AI to summarize revenue and expense data for all periods, output JSON with date fields in `dd/MM/yyyy` format.  
   - Enable output parser for JSON.  
   - Input connections from chat trigger, memory, and tool nodes.

6. **Link "When chat message received" → "AI Agent"**  
   - Also connect "Window Buffer Memory1" and "Call n8n Workflow Tool" as memory and tool inputs to "AI Agent".  
   - Connect "DeepSeek Chat Model" as the language model for "AI Agent".

7. **Create "When Executed by Another Workflow" Node**  
   - Type: `@n8n/n8n-nodes-base.executeWorkflowTrigger`  
   - Define inputs for six date parameters (Start_Current, End_Current, Start_LastMonth, End_LastMonth, Start_LastYear, End_LastYear).

8. **Create "Format Date" Node**  
   - Type: `Set` node  
   - Assign all six date inputs directly to JSON fields with the same names.

9. **Create "Get revenual from google sheet" Node**  
   - Type: `Google Sheets` node  
   - Credentials: Google Sheets OAuth2.  
   - Document ID: `1UAMO24QtfkR50VGu1wpksuYiWffi28pHfm6bihd5IP8`  
   - Sheet ID: `1168390826` (tab "Doanh thu theo ngày")  
   - Read all rows.

10. **Create Pivot Nodes for Each Period** (three code nodes):  
    - "Pivot current circle" (dates: Start_Current to End_Current)  
    - "Pivot last circle" (dates: Start_LastMonth to End_LastMonth)  
    - "Pivot last year cirle" (dates: Start_LastYear to End_LastYear)  
    - Each node parses dates, validates, normalizes category strings, sums "Số tiền" by category.

11. **Create Aggregate Nodes to Sum Each Pivot Output:**  
    - "Sum current circle", "Sum last circle", "Sum last year circle"  
    - Aggregate all items.

12. **Create Set Nodes to Format Period Dates:**  
    - "Format data current circle" → sets currentCycle.startDate and endDate  
    - "Format data last circle" → sets lastMonthCycle.startDate and endDate  
    - "Format data last year circle" → sets lastYearCycle.startDate and endDate

13. **Create Merge Nodes for Each Period's Data:**  
    - "Sum data current transfer month" merges "Sum current circle" and "Format data current circle"  
    - "Sum data last transfer month" merges "Sum last circle" and "Format data last circle"  
    - "Add data last year" merges "Sum last year circle" and "Format data last year circle"

14. **Create "Change title" Aggregate Nodes:**  
    - "Change title 1" renames current cycle as "Chu kỳ hiện tại"  
    - "Change title 2" renames last month cycle as "Chu kỳ tháng trước"  
    - "Change title" renames last year cycle as "Chu kỳ năm trước"

15. **Create "Collect all" Merge Node**  
    - Inputs: "Change title 1", "Change title 2", "Change title"  
    - Merges all aggregated data.

16. **Create "Change title out come" Aggregate Node**  
    - Renames final merged data as "data doanh thu" (final revenue data).

17. **Connect Nodes According to Dependencies:**  
    - Format Date → Get revenual from google sheet → All three Pivot nodes → respective Sum & Format data nodes → respective Merge nodes → Change title nodes → Collect all → Change title out come.

18. **Optionally Add Sticky Note Node**  
    - Content: "## Sub-Workflow: Google Analytics Data" to indicate sub-workflow context.

19. **Credentials Setup:**  
    - DeepSeek API credentials for DeepSeek Chat Model.  
    - Google Sheets OAuth2 API credentials for Google Sheets node.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                  |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| The date format used across all nodes is primarily `dd/MM/yyyy`. Consistency in date formats is critical.     | Important for date parsing and filtering logic. |
| The workflow integrates a sub-workflow identified by ID `4ASVA3i1vaFwj20h` for detailed data processing.      | Sub-workflow handles actual multi-period aggregation logic. |
| The DeepSeek AI API is used; ensure API keys are valid and rate limits are respected.                          | DeepSeek documentation: https://www.deepseek.ai/ |
| The custom JavaScript pivot nodes include detailed console logging for debugging date parsing and aggregation. | Useful for troubleshooting date format issues.  |
| The workflow expects Google Sheets data to have columns named exactly "Date", "Phân loại" (Category), and "Số tiền" (Amount). | Data structure must be consistent for accurate aggregation. |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to prevailing content policies and contains no illegal, offensive, or copyrighted elements. All handled data is legal and publicly available.