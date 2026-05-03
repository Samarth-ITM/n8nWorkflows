Generate AI-Powered Strava Performance Evolution Reports with Google Sheets & GPT-5-nano

https://n8nworkflows.xyz/workflows/generate-ai-powered-strava-performance-evolution-reports-with-google-sheets---gpt-5-nano-11214


# Generate AI-Powered Strava Performance Evolution Reports with Google Sheets & GPT-5-nano

### 1. Workflow Overview

This n8n workflow automates the generation of AI-powered performance evolution reports for Strava activities, integrating data extraction from Strava, data transformation, AI analysis using GPT-5-nano, and storing results in Google Sheets. It targets athletes and coaches who want automated, detailed performance insights based on Strava activity data, enhanced by AI-generated commentary and analysis.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Manual and scheduled triggers to start the workflow, including form input for specific activity IDs.
- **1.2 Data Acquisition:** Retrieving Strava activities data, either all activities or a single manual activity, with error handling.
- **1.3 Data Processing & Transformation:** Splitting activities into batches, mapping Strava data fields into AI-friendly formats.
- **1.4 AI Analysis:** Sending processed data to GPT-5-nano models for intensity and performance report generation.
- **1.5 Google Sheets Integration:** Clearing, appending, and reading rows in Google Sheets to store inputs and AI-generated reports.
- **1.6 Messaging & Notification:** Optional email sending and markdown formatting for reports (some disabled by default).
- **1.7 Control & Error Handling:** Limit nodes, stop and error nodes, and retry mechanisms to ensure workflow robustness.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception

- **Overview:** This block provides entry points to trigger the workflow either manually, on a schedule, or via a form input specifying an activity ID.
- **Nodes Involved:** 
  - When clicking ‘Execute workflow’ (Manual Trigger)
  - Schedule Trigger
  - Activity ID (Form Trigger)
- **Node Details:**

| Node Name                  | Details                                                                                                   |
|----------------------------|-----------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual trigger node to start the workflow on demand. No parameters configured. Output triggers clearing of the sheet. |
| Schedule Trigger           | Runs the workflow on a defined schedule (schedule details not specified, likely cron or interval). Triggers the OPTIONS node. |
| Activity ID                | Form trigger to accept an activity ID input from a user; triggers retrieval of a single manual Strava activity. |

- **Edge Cases:**  
  - If no activity ID is provided in the form, downstream nodes may not find data.  
  - Scheduled runs must handle rate limits and API timeouts gracefully.

---

#### Block 1.2: Data Acquisition

- **Overview:** Retrieves Strava activities from the API, either all activities or a manually specified single activity.
- **Nodes Involved:** 
  - Get all activities (Strava node)
  - Get manual activity (Strava node)
- **Node Details:**

| Node Name           | Details                                                                                          |
|---------------------|------------------------------------------------------------------------------------------------|
| Get all activities   | Strava API node configured to fetch all user activities; retry enabled on failure, with wait between tries (5s). On error, continues with error output. |
| Get manual activity  | Strava API node fetching a specific activity based on form input. Same retry and error handling as above. |

- **Edge Cases:**  
  - API authentication failures or expired tokens.  
  - API rate limiting or downtime.  
  - Manual activity ID not found or invalid.  
  - Partial data retrieval due to pagination not handled explicitly.

---

#### Block 1.3: Data Processing & Transformation

- **Overview:** Processes raw Strava data by splitting into manageable batches, transforming fields into AI-compatible formats, and preparing options for AI queries.
- **Nodes Involved:** 
  - Loop Activities (SplitInBatches node)
  - Strava to AI Fields (Set node)
  - OPTIONS (Set node)
  - Limit (Limit node)
  - Get row(s) in sheet (Google Sheets node)
  - Code in JavaScript nodes (various for data manipulation)
- **Node Details:**

| Node Name          | Details                                                                                                         |
|--------------------|-----------------------------------------------------------------------------------------------------------------|
| Loop Activities    | Splits the array of activities into batches for sequential processing, improving stability and rate limit handling. |
| Strava to AI Fields | Extracts and maps relevant activity data fields into a structure suitable as input to AI models.                  |
| OPTIONS            | Sets parameters/options for the AI prompt, e.g., “total,” “last year,” “last month,” “last week”.                |
| Limit              | Limits the number of rows processed from Google Sheets for performance and API quota control.                    |
| Get row(s) in sheet | Reads data from Google Sheets, likely to retrieve historical or reference data for AI analysis.                 |
| Code in JavaScript (multiple) | Performs custom data transformations, concatenations, or formatting to prepare data for AI input or output handling. |

- **Expressions/Variables:**  
  - Dynamic references to batch items and Google Sheets rows.  
  - Mapping between Strava JSON fields and AI prompt variables.

- **Edge Cases:**  
  - Empty or malformed activity data batches.  
  - Google Sheets API failures or empty sheet scenarios.  
  - JavaScript code errors if data shape changes.

---

#### Block 1.4: AI Analysis

- **Overview:** Sends prepared activity data and options to GPT-5-nano AI models via OpenAI integration to generate intensity estimates and performance reports.
- **Nodes Involved:** 
  - IA Intensity (OpenAI node)
  - Message a model2 (OpenAI node)
- **Node Details:**

| Node Name        | Details                                                                                                       |
|------------------|-----------------------------------------------------------------------------------------------------------------|
| IA Intensity     | Sends activity data transformed by “Strava to AI Fields” to GPT-5-nano to analyze intensity-related metrics. Configured with retries and wait between tries. |
| Message a model2 | Sends the final AI prompt constructed from processed data and options for generating detailed performance evolution reports. Also retries on failure. |

- **Expressions/Variables:**  
  - AI prompt templates integrating dynamic data from prior nodes.  
  - Model parameters (temperature, max tokens, model name) configured internally.

- **Edge Cases:**  
  - API authentication failures or rate limit errors.  
  - AI model response errors or unexpected output format.  
  - Network timeouts.

---

#### Block 1.5: Google Sheets Integration

- **Overview:** Manages clearing existing data, appending new rows of raw or AI-processed data, and reading sheets for reference.
- **Nodes Involved:** 
  - Clear sheet (Google Sheets node)
  - Append row in sheet (Google Sheets node)
  - Append row in sheet1 (Google Sheets node)
  - Informe de seguimiento (Google Sheets node)
  - Get row(s) in sheet (Google Sheets node)
- **Node Details:**

| Node Name          | Details                                                                                                             |
|--------------------|---------------------------------------------------------------------------------------------------------------------|
| Clear sheet        | Clears data in a specific Google Sheet range before new data ingestion. Includes retry logic and wait time.           |
| Append row in sheet | Adds a new row with processed data or AI output to the designated Google Sheet.                                      |
| Append row in sheet1 | Similar to the above but likely targets a different sheet or data format.                                            |
| Informe de seguimiento | Reads data from a Google Sheet to inform report generation, possibly historical tracking data.                       |
| Get row(s) in sheet | Retrieves rows from Google Sheets for use in prompts or validation.                                                  |

- **Edge Cases:**  
  - Google Sheets API errors due to permissions or quota exceeded.  
  - Race conditions if multiple appends happen simultaneously.  
  - Data format mismatches leading to incorrect appends or reads.

---

#### Block 1.6: Messaging & Notification

- **Overview:** Prepares markdown-formatted reports and optionally sends emails with generated performance reports.
- **Nodes Involved:** 
  - Markdown (Markdown node)
  - Send a message1 (Gmail node)
  - Send a message (Gmail node, disabled)
- **Node Details:**

| Node Name        | Details                                                                                                   |
|------------------|-------------------------------------------------------------------------------------------------------------|
| Markdown         | Converts AI-generated text into markdown format for readable reports or emails.                             |
| Send a message1  | Sends email with the markdown report attached or in the body. Configured with retries and wait times.       |
| Send a message   | Disabled Gmail node that could be used for additional notifications or error messages.                       |

- **Edge Cases:**  
  - Gmail OAuth authentication failures.  
  - Email sending limits or quota exceeded.  
  - Markdown formatting errors if AI output is malformed.

---

#### Block 1.7: Control & Error Handling

- **Overview:** Ensures clean workflow termination on errors or limits processing to manageable sizes.
- **Nodes Involved:** 
  - Stop and Error (two nodes)
- **Node Details:**

| Node Name        | Details                                                                                                |
|------------------|--------------------------------------------------------------------------------------------------------|
| Stop and Error   | Halts the workflow and throws an error when critical failures occur, preventing further faulty processing. |

- **Edge Cases:**  
  - Misfired stop nodes could halt workflow prematurely.  
  - Lack of detailed error messages might complicate debugging.

---

### 3. Summary Table

| Node Name               | Node Type                    | Functional Role                                  | Input Node(s)                           | Output Node(s)                     | Sticky Note                                      |
|-------------------------|------------------------------|-------------------------------------------------|---------------------------------------|----------------------------------|-------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger             | Start workflow manually                          | -                                     | Clear sheet                      |                                                 |
| Schedule Trigger         | Schedule Trigger             | Start workflow on schedule                        | -                                     | OPTIONS                         |                                                 |
| Activity ID             | Form Trigger                 | Receive manual activity ID input                  | -                                     | Get manual activity             |                                                 |
| Clear sheet             | Google Sheets                | Clear existing data before new ingestion         | When clicking ‘Execute workflow’      | Get all activities              |                                                 |
| Get all activities      | Strava                      | Fetch all Strava activities                        | Clear sheet                          | Loop Activities, Stop and Error  |                                                 |
| Get manual activity     | Strava                      | Fetch activity by manual ID input                  | Activity ID                         | Loop Activities, Stop and Error1 |                                                 |
| Loop Activities         | SplitInBatches              | Process activities in batches                      | Get all activities, Get manual activity | Send a message, Strava to AI Fields |                                                 |
| Send a message          | Gmail                       | Send email notifications (disabled by default)  | Loop Activities                     | -                                |                                                 |
| Strava to AI Fields     | Set                         | Map Strava data to AI-friendly fields             | Loop Activities                     | IA Intensity                    |                                                 |
| IA Intensity            | OpenAI                      | Analyze intensity metrics with AI                 | Strava to AI Fields                 | Code in JavaScript               |                                                 |
| Code in JavaScript      | Code                        | Transform AI output for sheet appending           | IA Intensity                       | Append row in sheet              |                                                 |
| Append row in sheet     | Google Sheets                | Append AI data to Google Sheets                    | Code in JavaScript                 | Loop Activities                 |                                                 |
| OPTIONS                 | Set                         | Set AI prompt options                              | Schedule Trigger                   | Informe de seguimiento          |                                                 |
| Informe de seguimiento  | Google Sheets                | Read report data from sheet                        | OPTIONS                           | Limit                          |                                                 |
| Limit                   | Limit                       | Limit number of rows for processing                | Informe de seguimiento            | Get row(s) in sheet             |                                                 |
| Get row(s) in sheet     | Google Sheets                | Read rows for AI prompt                            | Limit                            | Code in JavaScript1             |                                                 |
| Code in JavaScript1     | Code                        | Prepare AI prompt for model                         | Get row(s) in sheet               | Message a model2                |                                                 |
| Message a model2        | OpenAI                      | Generate AI report                                 | Code in JavaScript1               | Append row in sheet1            |                                                 |
| Append row in sheet1    | Google Sheets                | Append AI-generated report                          | Message a model2                 | Code in JavaScript2             |                                                 |
| Code in JavaScript2     | Code                        | Format report for markdown                          | Append row in sheet1             | Markdown                       |                                                 |
| Markdown                | Markdown                    | Convert report to markdown format                   | Code in JavaScript2              | Send a message1                |                                                 |
| Send a message1         | Gmail                       | Send markdown report via email                       | Markdown                       | -                              |                                                 |
| Stop and Error          | Stop and Error              | Halt workflow on critical failures                  | Get all activities               | -                              |                                                 |
| Stop and Error1         | Stop and Error              | Halt workflow on critical failures                  | Get manual activity              | -                              |                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Input Triggers:**
   - Add a **Manual Trigger** node named “When clicking ‘Execute workflow’”.
   - Add a **Schedule Trigger** node named “Schedule Trigger” with desired cron or interval settings.
   - Add a **Form Trigger** node named “Activity ID” with a field to input activity ID.

2. **Set up Data Clearing:**
   - Add a **Google Sheets** node “Clear sheet” configured to clear the target sheet range before new data insertion.
   - Connect “When clicking ‘Execute workflow’” to “Clear sheet”.

3. **Fetch Strava Activities:**
   - Add a **Strava** node “Get all activities” configured with OAuth credentials for Strava.
   - Connect “Clear sheet” to “Get all activities”.
   - Add a **Strava** node “Get manual activity” configured to retrieve activity by ID from “Activity ID” form input.
   - Connect “Activity ID” to “Get manual activity”.

4. **Handle API Results:**
   - Add two **Stop and Error** nodes named “Stop and Error” and “Stop and Error1” respectively.
   - Connect “Get all activities” error output to “Stop and Error”.
   - Connect “Get manual activity” error output to “Stop and Error1”.

5. **Process Activities in Batches:**
   - Add a **SplitInBatches** node named “Loop Activities”.
   - Connect main outputs of “Get all activities” and “Get manual activity” to “Loop Activities”.

6. **Transform Data for AI:**
   - Add a **Set** node “Strava to AI Fields” mapping Strava activity fields to AI prompt variables.
   - Connect “Loop Activities” to “Strava to AI Fields”.

7. **AI Intensity Analysis:**
   - Add an **OpenAI** node “IA Intensity” configured with OpenAI credentials and GPT-5-nano model.
   - Connect “Strava to AI Fields” to “IA Intensity”.

8. **Process AI Output and Append to Sheet:**
   - Add a **Code** node “Code in JavaScript” to transform AI output into structured data for Google Sheets.
   - Connect “IA Intensity” to “Code in JavaScript”.
   - Add a **Google Sheets** node “Append row in sheet” configured to append transformed data.
   - Connect “Code in JavaScript” to “Append row in sheet”.
   - Connect “Append row in sheet” back to “Loop Activities” to process next batch.

9. **Set AI Prompt Options on Schedule:**
   - Connect “Schedule Trigger” to a **Set** node “OPTIONS” that defines options like “total”, “last year”, etc.

10. **Read Report Data:**
    - Connect “OPTIONS” to a **Google Sheets** node “Informe de seguimiento” to read sheet data.
    - Connect “Informe de seguimiento” to a **Limit** node “Limit” to control row processing.
    - Connect “Limit” to “Get row(s) in sheet” Google Sheets node to fetch rows.

11. **Prepare AI Report Prompt:**
    - Add a **Code** node “Code in JavaScript1” to create AI prompt from sheet data.
    - Connect “Get row(s) in sheet” to “Code in JavaScript1”.

12. **Generate AI Report:**
    - Add an **OpenAI** node “Message a model2” configured with GPT-5-nano for report generation.
    - Connect “Code in JavaScript1” to “Message a model2”.

13. **Append AI Report and Format:**
    - Add a **Google Sheets** node “Append row in sheet1” to store AI-generated report.
    - Connect “Message a model2” to “Append row in sheet1”.
    - Add a **Code** node “Code in JavaScript2” to format report for markdown.
    - Connect “Append row in sheet1” to “Code in JavaScript2”.
    - Add a **Markdown** node “Markdown” to convert text.
    - Connect “Code in JavaScript2” to “Markdown”.

14. **Send Report via Email:**
    - Add a **Gmail** node “Send a message1” configured with OAuth2 credentials to send emails.
    - Connect “Markdown” to “Send a message1”.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                           |
|-------------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| The workflow uses GPT-5-nano via the LangChain OpenAI node integration for AI-powered analysis. | AI integration details: https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-langchain.openai/ |
| Google Sheets nodes require OAuth2 credentials with access to the target spreadsheet.            | Set up Google API credentials in n8n credentials settings. |
| Strava nodes require OAuth2 credentials with access to the Strava API for user activity data.    | Strava API documentation: https://developers.strava.com/  |
| The workflow includes retry logic and wait intervals (5 seconds) to handle rate limits and transient errors with Strava and AI API calls. | Ensures robustness under API quota constraints.           |

---

This structured documentation enables understanding, modification, and re-creation of the workflow for generating AI-enhanced Strava performance reports with Google Sheets integration.