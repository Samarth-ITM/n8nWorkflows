Optimize Amazon Ads with GPT-4o for Bid, Budget & Keyword Recommendations

https://n8nworkflows.xyz/workflows/optimize-amazon-ads-with-gpt-4o-for-bid--budget---keyword-recommendations-3793


# Optimize Amazon Ads with GPT-4o for Bid, Budget & Keyword Recommendations

### 1. Workflow Overview

This workflow automates the optimization of Amazon Sponsored Products advertising campaigns by analyzing performance reports and generating actionable recommendations using OpenAI’s GPT-4o model. It targets Amazon sellers and advertisers seeking to streamline campaign analysis, bidding, budgeting, and keyword management without manual report review.

The workflow is logically divided into these blocks:

- **1.1 Input Reception & File Acquisition**  
  Automatically lists and downloads Amazon Ads reports stored in a specified Google Drive folder.

- **1.2 File Type Detection & Data Extraction**  
  Differentiates between XLSX and CSV report files, extracting structured data accordingly.

- **1.3 Data Normalization & Aggregation**  
  Normalizes filenames, classifies reports into categories (search terms, campaigns, targeting, placement, budgets), and merges data for AI processing.

- **1.4 AI Analysis & Recommendation Generation**  
  Sends cleaned, structured data to GPT-4o with detailed instructions to produce bid, budget, and keyword recommendations.

- **1.5 Email Notification**  
  Formats the AI output into a human-readable email and sends it via Gmail to configured recipients.

- **1.6 Workflow Trigger & Configuration**  
  Manual trigger and email option setup nodes facilitate testing and configuration.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & File Acquisition

**Overview:**  
This block lists files from a user-specified Google Drive folder containing Amazon Ads reports and downloads each file for processing.

**Nodes Involved:**  
- List Files  
- Get File  
- Set fileName

**Node Details:**

- **List Files**  
  - Type: Google Drive (fileFolder resource, query method)  
  - Role: Lists files in a specified Drive folder (configured via folderId filter)  
  - Key Config: Folder ID must be set to the folder containing Amazon Ads reports  
  - Input: Trigger from "Email Options" node  
  - Output: List of files metadata (id, name)  
  - Failure Modes: Invalid folder ID, permission errors, API rate limits

- **Get File**  
  - Type: Google Drive (download operation)  
  - Role: Downloads each listed file’s binary content for further processing  
  - Key Config: Uses fileId from previous node, downloads file as binary property "data"  
  - Input: Files from "List Files"  
  - Output: Binary file content with metadata  
  - Failure Modes: File not found, download errors, permission issues

- **Set fileName**  
  - Type: Set  
  - Role: Extracts and assigns the filename from the downloaded file metadata to a JSON property `fileName` for downstream use  
  - Input: Downloaded file data from "Get File"  
  - Output: Adds `fileName` property to JSON  
  - Failure Modes: Missing or malformed filename

---

#### 2.2 File Type Detection & Data Extraction

**Overview:**  
Determines whether each file is XLSX or CSV and extracts data accordingly.

**Nodes Involved:**  
- is XLSX (If node)  
- Extract XLSX Data  
- Extract CSV Data  
- Preserve File Name (for XLSX)  
- Preserve CSV File Name

**Node Details:**

- **is XLSX**  
  - Type: If  
  - Role: Checks if filename contains ".xlsx" (case-sensitive) to branch extraction logic  
  - Input: JSON with `fileName`  
  - Output: Two branches: XLSX or CSV  
  - Failure Modes: Filename missing or unexpected format

- **Extract XLSX Data**  
  - Type: Extract From File (xlsx operation)  
  - Role: Parses XLSX binary data into JSON rows  
  - Input: Binary data from "Get File" when XLSX branch is true  
  - Output: JSON array of rows from the spreadsheet  
  - Failure Modes: Corrupt XLSX, unsupported formats

- **Extract CSV Data**  
  - Type: Extract From File (default CSV)  
  - Role: Parses CSV binary data into JSON rows  
  - Input: Binary data from "Get File" when XLSX branch is false  
  - Output: JSON array of rows from CSV  
  - Failure Modes: Malformed CSV, encoding issues

- **Preserve File Name (XLSX & CSV)**  
  - Type: Set  
  - Role: Copies the filename from the XLSX branch to CSV branch or vice versa to maintain consistent metadata for downstream merging  
  - Input: Extracted data from respective extraction nodes  
  - Output: Adds `fileName` property to JSON  
  - Failure Modes: Missing filename propagation

---

#### 2.3 Data Normalization & Aggregation

**Overview:**  
Merges extracted data from both XLSX and CSV files and normalizes filenames to classify reports into predefined categories for AI input.

**Nodes Involved:**  
- Merge XLSX and CSV  
- Format Data

**Node Details:**

- **Merge XLSX and CSV**  
  - Type: Merge  
  - Role: Combines data arrays from XLSX and CSV extraction paths into a single stream  
  - Input: Outputs from "Preserve File Name" nodes for XLSX and CSV  
  - Output: Combined dataset for processing  
  - Failure Modes: Data format mismatches

- **Format Data**  
  - Type: Code  
  - Role: Normalizes filenames by removing spaces, suffixes, and converting to lowercase; maps files to keys: search_terms, campaigns, targeting, placement, budgets  
  - Key Logic: Uses regex matching on normalized base filename to assign category keys  
  - Input: Merged data with filenames  
  - Output: JSON object with arrays keyed by report type  
  - Failure Modes: Unrecognized filenames cause workflow errors; malformed data

---

#### 2.4 AI Analysis & Recommendation Generation

**Overview:**  
Sends the structured input data to an AI model (GPT-4o) with detailed instructions to generate bid, budget, keyword, and targeting recommendations.

**Nodes Involved:**  
- AI Analyze (Chain LLM)  
- OpenAI Chat Model (GPT-4o)

**Node Details:**

- **AI Analyze**  
  - Type: Chain LLM (LangChain)  
  - Role: Wraps the OpenAI Chat Model node, sends JSON stringified structured data with system prompt defining expected JSON output format  
  - Key Config: System prompt instructs AI to return campaign adjustments, keyword recommendations, and targeting recommendations in strict JSON format without explanations  
  - Input: JSON from "Format Data"  
  - Output: AI-generated JSON recommendations  
  - Failure Modes: API errors, malformed AI output, rate limits, timeout, JSON parse errors downstream

- **OpenAI Chat Model**  
  - Type: OpenAI Chat Completion  
  - Role: Executes GPT-4o model call with prompt from "AI Analyze"  
  - Input: Prompt and messages from "AI Analyze"  
  - Output: Raw AI response text  
  - Failure Modes: Authentication errors, quota exceeded, network issues

---

#### 2.5 Email Notification

**Overview:**  
Formats AI-generated JSON recommendations into a structured HTML email and sends it to configured recipients via Gmail.

**Nodes Involved:**  
- Email Optimizations  
- Email Options

**Node Details:**

- **Email Options**  
  - Type: Set  
  - Role: Defines email recipient address and subject line for outgoing optimization emails  
  - Input: Manual trigger or workflow start  
  - Output: JSON with `send_to` and `subject` fields  
  - Failure Modes: Missing or invalid email addresses

- **Email Optimizations**  
  - Type: Gmail (OAuth2)  
  - Role: Parses AI JSON output, formats it into a detailed HTML email including campaign adjustments, keyword and targeting recommendations, and sends it via Gmail  
  - Key Logic:  
    - Removes markdown code blocks from AI output  
    - Parses JSON safely with error handling  
    - Summarizes total budget increase, estimated spend and sales  
    - Lists campaign bid and budget changes, keyword additions and negatives, targeting actions  
  - Input: AI Analyze output and Email Options  
  - Output: Sent email confirmation  
  - Failure Modes: Email sending errors, invalid JSON, missing credentials

---

#### 2.6 Workflow Trigger & Configuration

**Overview:**  
Facilitates manual testing and initial configuration of email parameters.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Sticky Notes (multiple)

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Starts the workflow manually for testing  
  - Output: Triggers "Email Options" node to start the chain  
  - Failure Modes: None

- **Sticky Notes**  
  - Type: Sticky Note  
  - Role: Provide detailed instructions and guidance for users on folder selection, report scheduling, report delivery methods, upgrading to API automation, and AI analysis explanation  
  - Content Highlights:  
    - Folder selection for reports  
    - Amazon Ads report scheduling instructions (report types, frequency, format)  
    - Report delivery options (manual upload, Gmail automation, Drive sync)  
    - Upgrade path to Amazon Ads API for full automation  
    - AI analysis explanation and expected output format  
  - Failure Modes: None (informational only)

---

### 3. Summary Table

| Node Name               | Node Type                      | Functional Role                                | Input Node(s)           | Output Node(s)           | Sticky Note                                                                                   |
|-------------------------|--------------------------------|-----------------------------------------------|-------------------------|--------------------------|----------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                | Manual workflow start trigger                  |                         | Email Options            | ## Trigger You may replace this with a scheduled event or poll the folder for changes.       |
| Email Options           | Set                            | Configures email recipient and subject        | When clicking ‘Test workflow’ | List Files               | ## Change! Edit these email options.                                                         |
| List Files              | Google Drive                   | Lists files in Amazon Ads reports folder       | Email Options            | Get File                 | ## Change Choose the "folder" in the filter options to the folder containing your Ad reports |
| Get File                | Google Drive                   | Downloads each file listed                      | List Files               | Set fileName              |                                                                                              |
| Set fileName            | Set                            | Assigns filename property for downstream use   | Get File                 | is XLSX                  |                                                                                              |
| is XLSX                 | If                             | Checks if file is XLSX or CSV                   | Set fileName             | Extract XLSX Data, Extract CSV Data |                                                                                              |
| Extract XLSX Data       | Extract From File (XLSX)       | Extracts data from XLSX files                    | is XLSX (true)           | Preserve File Name        |                                                                                              |
| Extract CSV Data        | Extract From File (CSV)        | Extracts data from CSV files                     | is XLSX (false)          | Preserve CSV File Name    |                                                                                              |
| Preserve File Name      | Set                            | Preserves filename metadata for XLSX branch    | Extract XLSX Data        | Merge XLSX and CSV        |                                                                                              |
| Preserve CSV File Name  | Set                            | Preserves filename metadata for CSV branch     | Extract CSV Data         | Merge XLSX and CSV        |                                                                                              |
| Merge XLSX and CSV      | Merge                          | Combines XLSX and CSV extracted data            | Preserve File Name, Preserve CSV File Name | Format Data              |                                                                                              |
| Format Data             | Code                           | Normalizes filenames and maps reports to keys  | Merge XLSX and CSV       | AI Analyze                |                                                                                              |
| AI Analyze              | Chain LLM (LangChain)          | Sends structured data to GPT-4o for analysis   | Format Data              | Email Optimizations       | ## AI Analysis Uses GPT-4o to process the bundled reports and generate optimization instructions. Passes system instructions and cleaned data as input. |
| OpenAI Chat Model       | OpenAI Chat Completion         | Executes GPT-4o model call                       | AI Analyze (ai_languageModel) | AI Analyze               |                                                                                              |
| Email Optimizations     | Gmail (OAuth2)                 | Formats AI output and sends optimization email | AI Analyze, Email Options |                          |                                                                                              |
| Sticky Note             | Sticky Note                    | Instructions: Trigger info                      |                         |                          | ## Trigger You may replace this with a scheduled event or poll the folder for changes.       |
| Sticky Note1            | Sticky Note                    | Instructions: Email options                      |                         |                          | ## Change! Edit these email options.                                                         |
| Sticky Note2            | Sticky Note                    | Instructions: Folder selection                   |                         |                          | ## Change Choose the "folder" in the filter options to the folder containing your Ad reports |
| Sticky Note3            | Sticky Note                    | Instructions: AI analysis explanation            |                         |                          | ## AI Analysis Uses GPT-4o to process the bundled reports and generate optimization instructions. Passes system instructions and cleaned data as input. |
| Sticky Note5            | Sticky Note                    | Instructions: Amazon Ads report scheduling       |                         |                          | ## Amazon Ads Report Scheduling Instructions To run this workflow, schedule the following Sponsored Products reports in the Amazon Ads Console: Use "Detailed" for Search Term and Targeting reports, "Summary" for Campaign, Placement, and Budget reports. Use Last 30 Days, Daily frequency, and .xlsx or .csv format. |
| Sticky Note6            | Sticky Note                    | Instructions: Report delivery options             |                         |                          | ## Report Delivery How to get reports into Google Drive: manual upload, Gmail automation, or Drive sync folder. Reports must match expected filenames. |
| Sticky Note7            | Sticky Note                    | Instructions: Upgrade to Amazon Ads API           |                         |                          | ## Upgrade! 🚀 Apply for an Amazon Advertising API developer account to unlock full automation: https://advertising.amazon.com/API/docs/en-us/ |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: When clicking ‘Test workflow’  
   - Type: Manual Trigger  
   - Purpose: To manually start the workflow for testing.

2. **Create Set Node for Email Options**  
   - Name: Email Options  
   - Type: Set  
   - Parameters:  
     - `send_to`: Set to recipient email address (e.g., your email)  
     - `subject`: Set to desired email subject for optimization reports  
   - Connect "When clicking ‘Test workflow’" → "Email Options"

3. **Create Google Drive List Files Node**  
   - Name: List Files  
   - Type: Google Drive (fileFolder resource, query method)  
   - Credentials: Connect your Google Drive OAuth2 credentials  
   - Parameters:  
     - Filter: Set folderId to the Google Drive folder containing Amazon Ads reports  
   - Connect "Email Options" → "List Files"

4. **Create Google Drive Get File Node**  
   - Name: Get File  
   - Type: Google Drive (download operation)  
   - Credentials: Same Google Drive OAuth2 credentials  
   - Parameters:  
     - fileId: Set to `={{ $json.id }}` from "List Files" output  
     - Binary Property Name: `data`  
   - Connect "List Files" → "Get File"

5. **Create Set Node to Assign Filename**  
   - Name: Set fileName  
   - Type: Set  
   - Parameters:  
     - Assign `fileName` to `={{ $json.name }}`  
     - Include other fields  
   - Connect "Get File" → "Set fileName"

6. **Create If Node to Check XLSX**  
   - Name: is XLSX  
   - Type: If  
   - Parameters:  
     - Condition: Check if `fileName` contains `.xlsx` (case-sensitive)  
   - Connect "Set fileName" → "is XLSX"

7. **Create Extract From File Node for XLSX**  
   - Name: Extract XLSX Data  
   - Type: Extract From File  
   - Parameters:  
     - Operation: `xlsx`  
   - Connect "is XLSX" (true branch) → "Extract XLSX Data"

8. **Create Extract From File Node for CSV**  
   - Name: Extract CSV Data  
   - Type: Extract From File  
   - Parameters:  
     - Binary Property Name: `data`  
   - Connect "is XLSX" (false branch) → "Extract CSV Data"

9. **Create Set Node to Preserve Filename for XLSX**  
   - Name: Preserve File Name  
   - Type: Set  
   - Parameters:  
     - Assign `fileName` from `={{ $('is XLSX').item.json.fileName }}`  
     - Include other fields  
   - Connect "Extract XLSX Data" → "Preserve File Name"

10. **Create Set Node to Preserve Filename for CSV**  
    - Name: Preserve CSV File Name  
    - Type: Set  
    - Parameters:  
      - Assign `fileName` from `={{ $('is XLSX').item.json.fileName }}`  
      - Include other fields  
    - Connect "Extract CSV Data" → "Preserve CSV File Name"

11. **Create Merge Node to Combine XLSX and CSV Data**  
    - Name: Merge XLSX and CSV  
    - Type: Merge  
    - Parameters: Default (merge by append)  
    - Connect "Preserve File Name" → "Merge XLSX and CSV" (input 1)  
    - Connect "Preserve CSV File Name" → "Merge XLSX and CSV" (input 2)

12. **Create Code Node to Format Data**  
    - Name: Format Data  
    - Type: Code  
    - Parameters: Use the provided JavaScript code to:  
      - Normalize filenames (remove spaces, suffixes, lowercase)  
      - Map files to keys: search_terms, campaigns, targeting, placement, budgets  
      - Throw error if filename is unrecognized  
    - Connect "Merge XLSX and CSV" → "Format Data"

13. **Create Chain LLM Node for AI Analysis**  
    - Name: AI Analyze  
    - Type: Chain LLM (LangChain)  
    - Credentials: OpenAI API key with GPT-4o access  
    - Parameters:  
      - Text: `={{JSON.stringify($json)}}`  
      - Prompt: Provide system instructions specifying expected JSON output format including campaign adjustments, keyword and targeting recommendations  
    - Connect "Format Data" → "AI Analyze"

14. **Create OpenAI Chat Model Node**  
    - Name: OpenAI Chat Model  
    - Type: OpenAI Chat Completion  
    - Credentials: Same OpenAI API key  
    - Parameters:  
      - Model: gpt-4o  
    - Connect "AI Analyze" (ai_languageModel output) → "OpenAI Chat Model"

15. **Create Gmail Node to Send Email**  
    - Name: Email Optimizations  
    - Type: Gmail (OAuth2)  
    - Credentials: Gmail OAuth2 credentials  
    - Parameters:  
      - Send To: `={{ $('Email Options').first().json.send_to }}`  
      - Subject: `={{ $('Email Options').first().json.subject }}`  
      - Message: JavaScript expression that:  
        - Parses AI JSON output safely  
        - Formats HTML email with campaign adjustments, keyword recommendations, targeting recommendations, and summary totals  
    - Connect "AI Analyze" → "Email Optimizations"

16. **Add Sticky Notes**  
    - Add sticky notes at appropriate places with instructions on:  
      - Folder selection for Google Drive  
      - Amazon Ads report scheduling details  
      - Report delivery options  
      - AI analysis explanation  
      - Upgrade path to Amazon Ads API automation

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                   |
|------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| To run this workflow, schedule Sponsored Products reports in Amazon Ads Console with specific report types, date ranges, and formats. | Amazon Ads Report Scheduling Instructions sticky note content                                   |
| Reports must be delivered to a Google Drive folder monitored by this workflow, either manually or via Gmail automation.           | Report Delivery sticky note content                                                             |
| Upgrade to Amazon Advertising API developer account for full automation of report generation and retrieval.                       | Upgrade sticky note with link: https://advertising.amazon.com/API/docs/en-us/                    |
| AI analysis uses GPT-4o to generate structured JSON recommendations for bids, budgets, keywords, and targeting adjustments.       | AI Analysis sticky note content                                                                 |
| Workflow trigger can be replaced with scheduled trigger or folder polling for automation.                                          | Trigger sticky note content                                                                      |
| Detailed instructions and field-specific guidance are embedded in sticky notes within the workflow for user reference.            | Workflow description and sticky notes                                                           |

---

This documentation enables advanced users and AI agents to understand, reproduce, and extend the Amazon Ads AI Optimization workflow effectively, anticipating potential errors and integration points.