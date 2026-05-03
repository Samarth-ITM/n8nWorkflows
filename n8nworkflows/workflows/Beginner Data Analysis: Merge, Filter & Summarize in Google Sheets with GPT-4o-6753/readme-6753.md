Beginner Data Analysis: Merge, Filter & Summarize in Google Sheets with GPT-4o

https://n8nworkflows.xyz/workflows/beginner-data-analysis--merge--filter---summarize-in-google-sheets-with-gpt-4o-6753


# Beginner Data Analysis: Merge, Filter & Summarize in Google Sheets with GPT-4o

### 1. Workflow Overview

This workflow automates beginner-level data analysis by merging, filtering, summarizing, and interpreting marketing data stored in Google Sheets. It is designed to combine two datasets—marketing performance metrics and leader assignments—filter records based on performance criteria, branch data according to spend thresholds, summarize outcomes per leader, and finally generate an AI-powered summary identifying best and worst performers.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Retrieve raw data from two Google Sheets documents.
- **1.2 Data Merging:** Combine datasets on a common key (Channel).
- **1.3 Filtering and Branching:** Apply filters based on clicks and classify data by spend threshold.
- **1.4 Summarization:** Aggregate results per leader for different outcome branches.
- **1.5 Data Preparation for AI:** Format merged data into text for AI consumption.
- **1.6 AI Processing:** Use an AI agent (GPT-4o) to summarize leader performance.
- **1.7 Output Handling:** Parse structured AI output and conclude workflow.

Supporting nodes include sticky notes for documentation and instructions.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block retrieves two distinct datasets from Google Sheets: marketing performance data and leader assignment data.

- **Nodes Involved:**  
  - Start Workflow  
  - Sample Marketing Data (Google Sheets)  
  - Google Sheets1 (Google Sheets)

- **Node Details:**

  - **Start Workflow**  
    - Type: Manual Trigger  
    - Role: Initiates the workflow manually.  
    - Connections: Outputs to both Google Sheets nodes simultaneously.  
    - Edge cases: None (manual trigger).  

  - **Sample Marketing Data**  
    - Type: Google Sheets  
    - Role: Reads raw marketing data from the sheet "Sheet1" in the specified Google Spreadsheet.  
    - Configuration: Document ID and sheet GID correspond to a publicly sharable marketing dataset.  
    - Credentials: Google Sheets OAuth2 with authorized access.  
    - Output: JSON rows representing marketing data.  
    - Edge cases: Google Sheets API authorization errors, network timeout, empty sheet.  

  - **Google Sheets1**  
    - Type: Google Sheets  
    - Role: Reads leader assignment data from "Sheet2" in the same spreadsheet.  
    - Configuration: Document ID and sheet ID specified.  
    - Credentials: Same as above.  
    - Output: JSON rows representing leader assignment information.  
    - Edge cases: Same as above.

---

#### 1.2 Data Merging

- **Overview:**  
  This block merges the two datasets based on the "Channel" column, combining marketing metrics with leader information.

- **Nodes Involved:**  
  - Merge on Channel Field

- **Node Details:**

  - **Merge on Channel Field**  
    - Type: Merge Node  
    - Role: Combines the two incoming data streams on the "Channel" field using the "combine" mode, which matches rows based on this key.  
    - Inputs: Two inputs from Sample Marketing Data and Google Sheets1.  
    - Outputs: Combined dataset used for filtering next.  
    - Edge cases: Mismatched or missing "Channel" fields, no matching rows leading to empty output.

---

#### 1.3 Filtering and Branching

- **Overview:**  
  Applies filtering to keep only records with at least 5 clicks, then branches data based on whether spend is above or below $350, marking outcomes accordingly.

- **Nodes Involved:**  
  - Filter  
  - Check if spend over $350 (If Node)

- **Node Details:**

  - **Filter**  
    - Type: Filter Node  
    - Role: Filters incoming merged data to keep only rows where "Clicks" field is greater than or equal to 5.  
    - Expression: `{{ $json.Clicks }} >= 5`  
    - Input: Output from Merge node.  
    - Output: Filtered data forwarded to the If node.  
    - Edge cases: Missing or non-numeric "Clicks" values causing expression failure.  

  - **Check if spend over $350**  
    - Type: If Node  
    - Role: Branches data into two paths based on "Spend ($)" field being ≥ 350 (Great outcome) or less (Poor outcome).  
    - Expression: `{{ $json['Spend ($)'] }} >= 350`  
    - Input: Filtered rows.  
    - Output: Two branches for downstream summarization.  
    - Edge cases: Missing or invalid "Spend ($)" data.

---

#### 1.4 Summarization

- **Overview:**  
  Counts the number of rows (days) per leader in each branch (Great or Poor), then sets outcome labels and organizes fields into a tabular structure.

- **Nodes Involved:**  
  - Count bad days (Summarize)  
  - Count good days (Summarize)  
  - Organize fields into table (Set)  
  - Organize second data into table (Set)

- **Node Details:**

  - **Count bad days**  
    - Type: Summarize  
    - Role: Counts rows grouped by "Leader" for the 'Poor' outcome branch.  
    - Field: Counts by "row_number" (assuming auto-generated or existing).  
    - Input: If node "False" branch (Spend < $350).  
    - Output: Summarized counts of bad days per leader.  
    - Edge cases: Missing "Leader" field or empty input.

  - **Count good days**  
    - Type: Summarize  
    - Role: Counts rows grouped by "Leader" for the 'Great' outcome branch.  
    - Field: Counts by "row_number".  
    - Input: If node "True" branch (Spend ≥ $350).  
    - Output: Summarized counts of good days per leader.  
    - Edge cases: Same as Count bad days.

  - **Organize fields into table**  
    - Type: Set  
    - Role: Creates structured output for "Poor" outcome by setting fields `Outcome` = "Poor", `# of Days` from count, and `Leader`.  
    - Input: Output of Count bad days.  
    - Output: Table-formatted JSON ready for merging.

  - **Organize second data into table**  
    - Type: Set  
    - Role: Same as above but for "Great" outcome with respective labels.  
    - Input: Output of Count good days.  
    - Output: Table-formatted JSON.

---

#### 1.5 Data Preparation for AI

- **Overview:**  
  Merges the "Good" and "Poor" outcome tables, converts the data into a single textual string, and prepares it to be fed into the AI agent.

- **Nodes Involved:**  
  - Append datasources together (Merge)  
  - Convert table names and columns into single text for agent (Code)

- **Node Details:**

  - **Append datasources together**  
    - Type: Merge  
    - Role: Combines the two summarized tables (Great and Poor outcomes) into a single dataset for AI input.  
    - Input: Outputs of both Set nodes.  
    - Output: Combined array of leader performance data.

  - **Convert table names and columns into single text for agent**  
    - Type: Code (JavaScript)  
    - Role: Converts each JSON item into a JSON string, joins all lines with newline characters, producing a textual representation suitable for AI input.  
    - Code snippet:  
      ```js
      return [
        {
          json: {
            text: items.map(item => JSON.stringify(item.json)).join('\n'),
          },
        },
      ];
      ```
    - Input: Merged dataset.  
    - Output: Single JSON object with a `text` property for AI consumption.  
    - Edge cases: Large datasets may exceed token limits for the AI model.

---

#### 1.6 AI Processing

- **Overview:**  
  Uses a GPT-4o-based AI agent to analyze leader performance summaries and generate a concise paragraph highlighting the best and worst performers and reasons.

- **Nodes Involved:**  
  - AI Agent - Summarize Leader Performance  
  - OpenAI Chat Model  
  - Structured Output Parser

- **Node Details:**

  - **AI Agent - Summarize Leader Performance**  
    - Type: Langchain Agent Node  
    - Role: Receives text input and system prompt, runs the AI model, and outputs structured JSON containing a summary paragraph.  
    - Parameters:  
      - Text input from previous code node.  
      - System Message prompt instructs AI to output one paragraph identifying performance outcomes with a specific JSON format:  
        ```json
        {
          "paragraph": "full paragraph"
        }
        ```  
    - Input: Text string.  
    - Output: AI generated JSON paragraph.  
    - Edge cases: API rate limits, model unavailability, malformed output.

  - **OpenAI Chat Model**  
    - Type: Langchain OpenAI Chat Model  
    - Role: Underlying language model node used by the agent with GPT-4o-mini selected.  
    - Credentials: OpenAI API key needed.  
    - Edge cases: Authentication failure, quota exceeded.

  - **Structured Output Parser**  
    - Type: Langchain Output Parser  
    - Role: Parses AI output to enforce expected JSON schema.  
    - Input: Raw AI output.  
    - Output: Validated structured JSON.  
    - Edge cases: Parsing errors if AI response deviates from schema.

---

#### 1.7 Supporting Documentation and Instructions

- **Overview:**  
  Several sticky notes provide user guidance, workflow explanation, and contact information.

- **Nodes Involved:**  
  - Sticky Note (Merge Multiple Datasets)  
  - Sticky Note1 (Filter and If Steps)  
  - Sticky Note2 (Summarize and Set Steps)  
  - Sticky Note3 (Analyze Results with an AI Agent)  
  - Sticky Note4 (Setup Instructions)  
  - Sticky Note5 (Contact and Feedback)

- **Details:**  
  These nodes have no inputs or outputs; they serve as inline documentation and setup instructions, including links such as the sample Google Sheet copy URL and contact info for support.

---

### 3. Summary Table

| Node Name                             | Node Type                         | Functional Role                                  | Input Node(s)                  | Output Node(s)                          | Sticky Note                                                                                                         |
|-------------------------------------|----------------------------------|-------------------------------------------------|-------------------------------|----------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| Start Workflow                      | Manual Trigger                   | Starts workflow manually                         |                               | Sample Marketing Data, Google Sheets1  |                                                                                                                     |
| Sample Marketing Data               | Google Sheets                   | Reads marketing data sheet                       | Start Workflow                | Merge on Channel Field                 |                                                                                                                     |
| Google Sheets1                     | Google Sheets                   | Reads leader assignment sheet                    | Start Workflow                | Merge on Channel Field                 |                                                                                                                     |
| Merge on Channel Field             | Merge                          | Combines datasets on "Channel"                   | Sample Marketing Data, Google Sheets1 | Filter                                  |                                                                                                                     |
| Filter                            | Filter                         | Filter rows where Clicks ≥ 5                      | Merge on Channel Field        | Check if spend over $350               |                                                                                                                     |
| Check if spend over $350           | If                            | Branch data by Spend ≥ 350                        | Filter                       | Count bad days (False), Count good days (True) |                                                                                                                     |
| Count bad days                    | Summarize                     | Count rows per Leader in "Poor" branch           | Check if spend over $350      | Organize fields into table             |                                                                                                                     |
| Count good days                   | Summarize                     | Count rows per Leader in "Great" branch          | Check if spend over $350      | Organize second data into table        |                                                                                                                     |
| Organize fields into table        | Set                           | Format "Poor" outcome data into table             | Count bad days               | Append datasources together            |                                                                                                                     |
| Organize second data into table   | Set                           | Format "Great" outcome data into table            | Count good days              | Append datasources together            |                                                                                                                     |
| Append datasources together       | Merge                         | Combine "Poor" and "Great" tables                 | Organize fields into table, Organize second data into table | Convert table names and columns into single text for agent |                                                                                                                     |
| Convert table names and columns into single text for agent | Code | Convert JSON to text string for AI input          | Append datasources together  | AI Agent - Summarize Leader Performance |                                                                                                                     |
| AI Agent - Summarize Leader Performance | Langchain Agent             | Generate summary paragraph about leader performance | Convert table names and columns into single text for agent, OpenAI Chat Model, Structured Output Parser | (End)                                |                                                                                                                     |
| OpenAI Chat Model                | Langchain OpenAI Chat Model | Language model interface using GPT-4o-mini        | AI Agent - Summarize Leader Performance | AI Agent - Summarize Leader Performance (ai_languageModel) |                                                                                                                     |
| Structured Output Parser         | Langchain Output Parser       | Parse AI output into structured JSON              | AI Agent - Summarize Leader Performance | AI Agent - Summarize Leader Performance (ai_outputParser) |                                                                                                                     |
| Sticky Note                     | Sticky Note                   | Document "Merge Multiple Datasets"                  |                               |                                        | ### 📌 Merge Multiple Datasets Pull two Google Sheets: Marketing data and Leader assignment, then merge on Channel. |
| Sticky Note1                    | Sticky Note                   | Document "Filter and If Steps"                       |                               |                                        | ### 📌 Filter and If Steps Step 1: Filter Clicks ≥ 5; Step 2: Branch on Spend ≥ 350 ("Great") or else ("Poor").       |
| Sticky Note2                    | Sticky Note                   | Document "Summarize and Set Steps"                   |                               |                                        | ### 📌 Summarize and Set Steps Count rows per Leader in each branch, assign labels Outcome, # of Days, Leader.         |
| Sticky Note3                    | Sticky Note                   | Document "Analyze Results with AI Agent"             |                               |                                        | ### 📌 Analyze Results with an AI Agent Merge outcomes, convert to text, use AI to summarize and identify performers.|
| Sticky Note4                    | Sticky Note                   | Setup Instructions                                   |                               |                                        | ### 📌 Setup Instructions 1. Copy sample Sheet 2. Setup Google Sheets OAuth2 credentials 3. Add OpenAI API key.       |
| Sticky Note5                    | Sticky Note                   | Contact and Feedback                                 |                               |                                        | ### 👋 Questions or Feedback? Contact Robert Breen, Ynteractive, with links and email provided.                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Manual Trigger node** named `Start Workflow`. No configuration needed.

3. **Add two Google Sheets nodes:**

   - **Sample Marketing Data**  
     - Type: Google Sheets  
     - Credentials: Link your Google Sheets OAuth2 account authorized for the target spreadsheet.  
     - Document ID: `19aUQYZq02qHsCelO4eeV4sx_MTJJupC5qe0gDLQBtRA`  
     - Sheet Name: `Sheet1` (or GID=0)  
     - Operation: Read rows (default).  

   - **Google Sheets1**  
     - Same credentials and Document ID as above.  
     - Sheet Name: `Sheet2` (GID=173686600)  
     - Operation: Read rows.

4. **Connect `Start Workflow` outputs to both Google Sheets nodes' inputs.**

5. **Add a Merge node named `Merge on Channel Field`:**

   - Mode: Combine  
   - Fields to match on: `Channel` (string)  
   - Connect `Sample Marketing Data` to input 1, `Google Sheets1` to input 2.

6. **Add a Filter node named `Filter`:**

   - Condition: Keep records with `Clicks >= 5`  
   - Expression: `{{ $json.Clicks }} >= 5`  
   - Connect output of `Merge on Channel Field` to `Filter`.

7. **Add an If node named `Check if spend over $350`:**

   - Condition: `Spend ($) >= 350`  
   - Expression: `{{ $json['Spend ($)'] }} >= 350`  
   - Connect output of `Filter` to this node.

8. **Add two Summarize nodes:**

   - **Count bad days** (for False branch):  
     - Group by: `Leader`  
     - Summarize: Count rows based on `row_number` (or any unique field).  
     - Connect False output from If node here.

   - **Count good days** (for True branch):  
     - Same configuration as above.  
     - Connect True output from If node here.

9. **Add two Set nodes to organize data:**

   - **Organize fields into table** (for bad days):  
     - Set fields:  
       - `Outcome`: `"Poor"` (string)  
       - `# of Days`: `={{ $json.count_row_number }}` (number)  
       - `Leader`: `={{ $json.Leader }}` (string)  
     - Connect output of Count bad days here.

   - **Organize second data into table** (for good days):  
     - Same fields but `Outcome` as `"Great"`.  
     - Connect output of Count good days here.

10. **Add a Merge node named `Append datasources together`:**

    - Mode: Append  
    - Connect outputs of both Set nodes here.

11. **Add a Code node named `Convert table names and columns into single text for agent`:**

    - Code (JavaScript):  
      ```js
      return [
        {
          json: {
            text: items.map(item => JSON.stringify(item.json)).join('\n'),
          },
        },
      ];
      ```
    - Connect output of the previous Merge node here.

12. **Add an AI Agent node named `AI Agent - Summarize Leader Performance`:**

    - Text input: `={{ $json.text }}`  
    - System Message prompt:  
      ```
      You are given a list of outcomes by leader. write a summary of the outcomes and identify who is doing the best and worst, and why. 

      output as one paragraph. 

      Output like this. 

      {
          "paragraph": "full paragraph"
      }
      ```
    - Prompt type: Define  
    - Enable output parser.

13. **Add OpenAI Chat Model node:**

    - Model: `gpt-4o-mini`  
    - Connect this node as the language model for the AI Agent node.  
    - Set OpenAI API credentials.

14. **Add Structured Output Parser node:**

    - JSON schema example:  
      ```json
      {
        "paragraph": "full paragraph"
      }
      ```
    - Connect as output parser for AI Agent node.

15. **Connect the nodes accordingly as described in the connections section.**

16. **Add Sticky Note nodes anywhere desired for inline documentation and setup instructions:**

    - Copy the content from the sticky notes in the original workflow for user guidance.

17. **Configure credentials:**

    - Google Sheets OAuth2: Create credentials with access to the sample spreadsheet.  
    - OpenAI API: Add your OpenAI API key.

18. **Run the workflow manually to test.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                           | Context or Link                                                                                      |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Before running, copy the sample Google Sheet: [Copy the Google Sheet](https://docs.google.com/spreadsheets/d/19aUQYZq02qHsCelO4eeV4sx_MTJJupC5qe0gDLQBtRA/copy)                                                                                      | Setup Instructions sticky note                                                                       |
| For Google Sheets API authorization, create Google Sheets OAuth2 credentials in n8n and authorize access to the copied spreadsheet.                                                                                                                 | Setup Instructions sticky note                                                                       |
| Add your OpenAI API key in the AI Agent step credentials to enable GPT-4o model usage.                                                                                                                                                                  | Setup Instructions sticky note                                                                       |
| Workflow author and support contact: Robert Breen, Founder at Ynteractive. Contact via email: robert@ynteractive.com or LinkedIn: [linkedin.com/in/robertbreen](https://www.linkedin.com/in/robertbreen)                                                  | Sticky Note5 (Contact and Feedback)                                                                  |
| This workflow demonstrates a practical example of integrating Google Sheets data with AI summarization using n8n and Langchain nodes, suited for marketers or analysts wanting automated performance summaries.                                      | General understanding                                                                                |

---

**Disclaimer:** The text analyzed and documented is generated from an automated n8n workflow. It complies with content policies and handles only legal and public data sources.