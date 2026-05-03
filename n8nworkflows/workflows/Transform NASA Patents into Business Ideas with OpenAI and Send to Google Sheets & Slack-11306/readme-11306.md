Transform NASA Patents into Business Ideas with OpenAI and Send to Google Sheets & Slack

https://n8nworkflows.xyz/workflows/transform-nasa-patents-into-business-ideas-with-openai-and-send-to-google-sheets---slack-11306


# Transform NASA Patents into Business Ideas with OpenAI and Send to Google Sheets & Slack

### 1. Workflow Overview

This workflow automates the transformation of NASA patents into actionable business ideas using AI, and distributes the results to Google Sheets and Slack on a weekly basis. It targets entrepreneurs, R&D teams, and content creators interested in leveraging NASA’s tech transfer patents for innovation and business development.

The workflow is logically structured into the following blocks:

- **1.1 Trigger & Settings Initialization:** Scheduled weekly trigger to start the workflow and load user-configured settings.
- **1.2 Data Retrieval & Filtering:** Fetches the latest NASA patents and filters out entries lacking abstracts.
- **1.3 Patent Processing Loop:** Iterates over each patent; translates abstracts, generates business ideas using OpenAI, saves results to Google Sheets, and formats messages.
- **1.4 Reporting & Notification:** Aggregates formatted messages and sends a weekly summary report to a Slack channel.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Settings Initialization

**Overview:**  
This block initiates the workflow on a weekly schedule and prepares necessary configuration parameters for downstream nodes.

**Nodes Involved:**  
- Weekly Schedule  
- Edit Settings  

**Node Details:**

- **Weekly Schedule**  
  - Type: Schedule Trigger  
  - Role: Starts workflow every Monday at 9 AM.  
  - Configuration: Weekly interval, trigger day = Monday (1), trigger hour = 9.  
  - Inputs: None (trigger node)  
  - Outputs: Triggers Edit Settings node  
  - Edge Cases: Possible time zone misconfiguration or node disablement prevents workflow start.

- **Edit Settings**  
  - Type: Set  
  - Role: Defines workflow parameters such as Google Sheet ID, Sheet Name, Slack Channel ID.  
  - Configuration: Static string assignments for `google_sheet_id`, `sheet_name`, `slack_channel_id`.  
  - Inputs: Trigger from Weekly Schedule  
  - Outputs: Passes settings to "Get NASA Patents" node  
  - Edge Cases: Missing or incorrect IDs cause failures in Google Sheets or Slack nodes downstream.

---

#### 1.2 Data Retrieval & Filtering

**Overview:**  
Fetches the latest tech transfer patents from NASA and filters out patents with empty abstracts to ensure meaningful data processing.

**Nodes Involved:**  
- Get NASA Patents  
- Filter Empty Abstracts  

**Node Details:**

- **Get NASA Patents**  
  - Type: NASA (n8n built-in integration)  
  - Role: Retrieves recent NASA tech transfer patents.  
  - Configuration: Resource set to "techTransfer" for patent data.  
  - Inputs: Receives settings from Edit Settings node  
  - Outputs: Patent data JSON array to Filter Empty Abstracts  
  - Edge Cases: API rate limits, network errors, or empty responses.

- **Filter Empty Abstracts**  
  - Type: If (Conditional)  
  - Role: Filters patents to exclude any with empty or missing abstracts.  
  - Configuration: Condition checks that `abstract` field is not empty (loose string check, case sensitive).  
  - Inputs: Patent array from Get NASA Patents  
  - Outputs: Passes only patents with abstracts to Loop Patents  
  - Edge Cases: Patents missing `abstract` field or with null abstract cause skip; expression errors if field missing.

---

#### 1.3 Patent Processing Loop

**Overview:**  
Processes each patent individually: translates abstracts into English, generates AI-powered business ideas, saves data to Google Sheets, and formats messages for reporting.

**Nodes Involved:**  
- Loop Patents  
- Translate Abstract  
- Generate Business Ideas  
- Save to Google Sheets  
- Format Item Message  

**Node Details:**

- **Loop Patents**  
  - Type: SplitInBatches  
  - Role: Splits patent list into individual items for sequential processing.  
  - Configuration: Default options, processes one item at a time.  
  - Inputs: Filtered patents from Filter Empty Abstracts  
  - Outputs: Sends each patent one-by-one to Translate Abstract and Aggregate Loop Results  
  - Edge Cases: Large patent sets may cause long execution times or timeouts.

- **Translate Abstract**  
  - Type: DeepL  
  - Role: Translates patent abstract text to American English (en-US).  
  - Configuration: Text expression set to current patent’s abstract; target language “EN-US”.  
  - Inputs: Single patent from Loop Patents  
  - Outputs: Translated abstract text to Generate Business Ideas  
  - Edge Cases: DeepL API key issues, rate limits, untranslated text if original is already English.

- **Generate Business Ideas**  
  - Type: OpenAI  
  - Role: Uses OpenAI chat completion API to generate business ideas from translated abstract.  
  - Configuration: Chat resource, default model settings; prompt implicitly uses translated abstract (configured in node).  
  - Inputs: Translated abstract from Translate Abstract  
  - Outputs: AI-generated business idea text to Save to Google Sheets  
  - Edge Cases: OpenAI API errors, token limits, prompt failures.

- **Save to Google Sheets**  
  - Type: Google Sheets  
  - Role: Appends or updates a row with patent data and AI-generated ideas in the specified sheet.  
  - Configuration:  
    - Operation: appendOrUpdate  
    - Spreadsheet ID and Sheet Name from Edit Settings  
    - Columns mapped: Date (current date), Link (patent URL), Title, Business Idea, Abstract Translated  
    - Matching on Title for update  
  - Inputs: AI-generated idea from Generate Business Ideas; patent data from Loop Patents and Translate Abstract  
  - Outputs: Confirmation to Format Item Message  
  - Edge Cases: Credential failures, sheet permission issues, duplicate titles causing unexpected updates.

- **Format Item Message**  
  - Type: Code  
  - Role: Creates a formatted Slack message snippet for the processed patent.  
  - Configuration: JavaScript code concatenates patent title and business idea into a markdown string.  
  - Inputs: Patent title from Loop Patents, business idea from Generate Business Ideas  
  - Outputs: Formatted message to Loop Patents for aggregation  
  - Edge Cases: Expression errors if referenced fields missing.

---

#### 1.4 Reporting & Notification

**Overview:**  
Aggregates all formatted patent messages into a single report and sends a weekly notification to a Slack channel.

**Nodes Involved:**  
- Aggregate Loop Results  
- Send Weekly Report  

**Node Details:**

- **Aggregate Loop Results**  
  - Type: Code  
  - Role: Collects all formatted messages from the loop, concatenates them into one report string separated by dividers.  
  - Configuration: JavaScript code accesses all items output by Format Item Message node, joins with separators.  
  - Inputs: All formatted messages from Format Item Message (all loop iterations)  
  - Outputs: Single aggregated report string to Send Weekly Report node  
  - Edge Cases: Empty loop results produce empty report; code errors if node references change.

- **Send Weekly Report**  
  - Type: Slack  
  - Role: Posts the weekly summary report to a specified Slack channel.  
  - Configuration:  
    - Text includes static header and dynamic report content from Aggregate Loop Results.  
    - Channel ID from Edit Settings.  
    - Uses Slack webhook or Bot User OAuth Token with `chat:write` permission.  
  - Inputs: Aggregated report string  
  - Outputs: None  
  - Edge Cases: Slack API errors, invalid channel ID, permission errors.

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role                         | Input Node(s)         | Output Node(s)               | Sticky Note                                                                                                    |
|---------------------|---------------------|---------------------------------------|-----------------------|-----------------------------|----------------------------------------------------------------------------------------------------------------|
| Weekly Schedule      | Schedule Trigger    | Initiates workflow weekly              | None                  | Edit Settings               |                                                                                                                |
| Edit Settings       | Set                 | Holds config params (Sheet ID, Slack) | Weekly Schedule       | Get NASA Patents            |                                                                                                                |
| Get NASA Patents    | NASA API            | Fetch latest NASA tech transfer patents| Edit Settings         | Filter Empty Abstracts      |                                                                                                                |
| Filter Empty Abstracts| If                  | Filters out patents with empty abstracts| Get NASA Patents      | Loop Patents                |                                                                                                                |
| Loop Patents        | SplitInBatches      | Processes patents one-by-one           | Filter Empty Abstracts| Translate Abstract, Aggregate Loop Results |                                                                                                                |
| Translate Abstract  | DeepL               | Translates abstract to English         | Loop Patents          | Generate Business Ideas     | Adjust Target Lang                                                                                              |
| Generate Business Ideas| OpenAI            | Generates AI business ideas             | Translate Abstract    | Save to Google Sheets       |                                                                                                                |
| Save to Google Sheets| Google Sheets       | Stores patent & idea data               | Generate Business Ideas| Format Item Message         |                                                                                                                |
| Format Item Message | Code                | Formats Slack message snippet           | Save to Google Sheets | Loop Patents                |                                                                                                                |
| Aggregate Loop Results| Code               | Combines all formatted messages         | Format Item Message   | Send Weekly Report          |                                                                                                                |
| Send Weekly Report  | Slack               | Sends the weekly summary to Slack       | Aggregate Loop Results| None                       |                                                                                                                |
| Sticky Note         | Sticky Note         | Documentation block                     | None                  | None                       | ## 🚀 Generate business ideas from NASA patents to Google Sheets and Slack... (Full content in node)          |
| Sticky Note1        | Sticky Note         | AI processing loop explanation          | None                  | None                       | ## 🔄 AI Processing Loop...                                                                                      |
| Sticky Note2        | Sticky Note         | Reporting block explanation              | None                  | None                       | ## 📊 Reporting...                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create `Weekly Schedule` node**:  
   - Type: Schedule Trigger  
   - Set to trigger weekly every Monday at 9:00 AM.

2. **Create `Edit Settings` node**:  
   - Type: Set  
   - Add three string fields:  
     - `google_sheet_id` (your Google Sheet document ID)  
     - `sheet_name` (e.g., "Sheet1")  
     - `slack_channel_id` (your Slack channel ID)  
   - Connect output of Weekly Schedule to this node.

3. **Create `Get NASA Patents` node**:  
   - Type: NASA integration  
   - Resource: `techTransfer`  
   - Connect output of Edit Settings to this node.

4. **Create `Filter Empty Abstracts` node**:  
   - Type: If  
   - Condition: Check if `abstract` field of patent JSON is not empty (string notEmpty).  
   - Connect output of Get NASA Patents to this node.

5. **Create `Loop Patents` node**:  
   - Type: SplitInBatches  
   - Default batch size (process patent one at a time).  
   - Connect True output of Filter Empty Abstracts to this node.

6. **Create `Translate Abstract` node**:  
   - Type: DeepL  
   - Text: Expression `{{$json.abstract}}` from current patent item.  
   - Target language: EN-US (American English).  
   - Connect output of Loop Patents to this node.

7. **Create `Generate Business Ideas` node**:  
   - Type: OpenAI (Chat completion)  
   - Use your OpenAI credentials.  
   - Prompt: Use translated abstract from previous node. Customize prompt as needed (e.g., ask for business idea suggestions).  
   - Connect output of Translate Abstract to this node.

8. **Create `Save to Google Sheets` node**:  
   - Type: Google Sheets  
   - Operation: appendOrUpdate  
   - Document ID: use expression to get from Edit Settings (`$('Edit Settings').item.json.google_sheet_id`)  
   - Sheet Name: expression from Edit Settings (`$('Edit Settings').item.json.sheet_name`)  
   - Map columns for Date (current date), Title, Link (build URL from patent id), Abstract_Translated, Business_Idea (from OpenAI output).  
   - Matching column: Title  
   - Connect output of Generate Business Ideas to this node.

9. **Create `Format Item Message` node**:  
   - Type: Code  
   - JavaScript code: format message combining title and business idea in markdown style.  
   - Connect output of Save to Google Sheets to this node.

10. **Connect output of Format Item Message back to Loop Patents**:  
    - This allows the aggregation after all patents processed.

11. **Create `Aggregate Loop Results` node**:  
    - Type: Code  
    - Combine all formatted messages into one report string separated by dividers.  
    - Input: all items from Format Item Message node after loop completes.

12. **Create `Send Weekly Report` node**:  
    - Type: Slack  
    - Configure Slack OAuth2 credentials with `chat:write` scope.  
    - Channel ID from Edit Settings (`$('Edit Settings').item.json.slack_channel_id`)  
    - Message text includes static header + aggregated report from previous node.  
    - Connect output of Aggregate Loop Results to this node.

13. **Add Sticky Notes** for documentation purposes, including instructions and summaries for each block.

14. **Configure credentials**:  
    - OpenAI API key (GPT-3.5-turbo or GPT-4)  
    - DeepL API key  
    - Google Sheets OAuth2 with Drive and Sheets access  
    - Slack Bot User OAuth Token with `chat:write` permission

15. **Test execution**, verify everything works, then activate the workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Context or Link                                                                                               |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| This workflow automates discovery of business ideas from NASA patents with AI assistance, delivering insights weekly to Slack and Google Sheets. It suits entrepreneurs, R&D teams, and content creators interested in aerospace tech innovation. Setup requires credentials for OpenAI, DeepL, Google Sheets, and Slack, plus a prepared Google Sheet with specific headers. Customize AI prompts to tailor ideas for niches like medical devices or others. Schedule can be adjusted as needed.                                                                                                                                                 | See Sticky Note content in workflow for detailed instructions and setup guidance.                           |
| Google Sheet preparation requires columns: Date, Title, Abstract_Translated, Business_Idea, Link in the first row for proper mapping.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Workflow’s "Save to Google Sheets" node expects these exact headers.                                         |
| OpenAI usage leverages chat completions; ensure your prompt in "Generate Business Ideas" node is adjusted to your domain or interests.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Customize prompt text inside “Generate Business Ideas” node properties.                                      |
| Slack notifications require a bot token with `chat:write` permission and the correct Slack channel ID.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Slack node uses these credentials and channel ID from settings.                                              |
| API rate limits and network errors can cause failures; monitor executions and consider error handling or retry mechanisms for production use.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Recommended monitoring and alerting outside this workflow.                                                  |
| This workflow does not contain raw patent data storage beyond Google Sheets; consider privacy and data compliance if extending.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |                                                                                                              |

---

**Disclaimer:** The provided content originates exclusively from an automated workflow created with n8n, adhering strictly to current content policies. It contains no illegal, offensive, or protected material. All data processed is legal and publicly available.