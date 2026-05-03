Generate Daily Jira Summaries & Weekly Reports with Azure OpenAI and Gmail

https://n8nworkflows.xyz/workflows/generate-daily-jira-summaries---weekly-reports-with-azure-openai-and-gmail-9683


# Generate Daily Jira Summaries & Weekly Reports with Azure OpenAI and Gmail

### 1. Workflow Overview

This workflow automates the generation and distribution of daily and weekly Jira issue summaries for engineering teams using Azure OpenAI's GPT-4o-mini model and Gmail. It is designed to run on a schedule to:

- Retrieve Jira issues daily and produce a structured End-of-Day (EOD) summary.
- Store daily summaries in a Google Sheet for archival and aggregation.
- On Fridays, aggregate the week’s daily summaries into a comprehensive weekly report.
- Format the weekly report into a styled HTML email.
- Send the weekly report to stakeholders via Gmail.

The workflow is logically divided into two main pipelines:

- **1.1 Daily EOD Summary Generation**
  - Triggered every workday evening.
  - Retrieves all Jira issues for the day.
  - Normalizes and flattens the data.
  - Uses Azure OpenAI to generate a structured daily summary.
  - Appends the summary and metadata to a Google Sheet.

- **1.2 Weekly Summary Compilation and Distribution**
  - Triggered every Friday evening.
  - Reads all daily summaries from the Google Sheet for the week.
  - Normalizes the aggregated data.
  - Uses Azure OpenAI to generate a structured weekly summary.
  - Converts the summary into a formatted HTML email.
  - Sends the email to stakeholders via Gmail.

### 2. Block-by-Block Analysis

#### 2.1 Daily EOD Summary Generation

- **Overview:**  
  Automates the daily retrieval of Jira issues, processes them into a normalized format, generates an AI-driven EOD summary, and stores it into a spreadsheet for later aggregation.

- **Nodes Involved:**  
  - Daily Evening Trigger  
  - Get ALL issues (Jira)  
  - Flatten Input (Code)  
  - Azure OpenAI Chat Model  
  - Structured Output Parser1  
  - Create Summary (LangChain Agent)  
  - Append Summary (Google Sheets)  
  - Sticky Notes: Daily Evening Trigger, Get ALL issues (Jira), Flatten Input, Chat Model (EOD), Append Summary (Sheet)

- **Node Details:**

  1. **Daily Evening Trigger**  
     - Type: Schedule Trigger  
     - Role: Initiates the workflow every workday evening (default daily interval).  
     - Configuration: Runs once per day on workdays (default settings).  
     - Input/Output: No input; outputs trigger event to "Get ALL issues".  
     - Failure Modes: Missed trigger if n8n instance is down or schedule misconfigured.

  2. **Get ALL issues (Jira)**  
     - Type: Jira Node  
     - Role: Retrieves all Jira issues for the day including statuses, priorities, and assignees.  
     - Configuration: Operation "getAll" with "returnAll" enabled to fetch all matching issues.  
     - Credentials: Jira Software Cloud API configured with user "jyothi".  
     - Input: Trigger event from schedule.  
     - Output: Array of Jira issues in JSON.  
     - Failure Modes: Authentication errors if Jira token invalid; API rate limits; network timeouts.

  3. **Flatten Input (Code Node)**  
     - Type: Code (JavaScript)  
     - Role: Normalizes and flattens the Jira JSON data into a single combined array for consistent downstream processing.  
     - Key Logic: Parses stringified JSON if needed, merges all input arrays into one.  
     - Input: Jira issues array.  
     - Output: Single item with combined array and count.  
     - Edge Cases: Malformed or unexpected JSON structures could cause parsing errors.

  4. **Azure OpenAI Chat Model**  
     - Type: LangChain Azure OpenAI Chat Model (GPT-4o-mini)  
     - Role: Generates a structured EOD summary from the flattened issue data.  
     - Configuration: Model "gpt-4o-mini" with default options.  
     - Credentials: Azure OpenAI account linked.  
     - Input: Text input derived from combined Jira issues array.  
     - Output: Raw AI-generated chat response.  
     - Failure Modes: API quota limits, network errors, or malformed input causing generation failure.

  5. **Structured Output Parser1**  
     - Type: LangChain Structured Output Parser  
     - Role: Parses the AI-generated EOD summary into a JSON structure according to a detailed schema example.  
     - Input: Chat Model output.  
     - Output: Parsed structured summary text.  
     - Failure Modes: Parsing errors if AI output deviates from expected schema; incomplete or malformed AI responses.

  6. **Create Summary (LangChain Agent)**  
     - Type: LangChain Agent (text generation with output parser)  
     - Role: Orchestrates the prompt and output parser to ensure consistent EOD summary formatting and content.  
     - Configuration: System message defines strict input/output format rules and classification logic.  
     - Input: Flattened Jira issues array as text.  
     - Output: Structured EOD summary JSON with labeled sections.  
     - Failure Modes: Same as above; also prompt engineering errors if input data is incomplete.

  7. **Append Summary (Google Sheets)**  
     - Type: Google Sheets  
     - Role: Appends the current date, raw JSON data, and generated summary to a specified Google Sheet for archiving daily reports.  
     - Configuration: Appends to Sheet1 with columns Date, JSON, Summary; date formatted as dd-MMM-yyyy.  
     - Credentials: Google Sheets OAuth2 API with automations user.  
     - Input: Output from Create Summary.  
     - Output: Confirmation of append operation.  
     - Failure Modes: Credential expiration, API quota limits, Google Sheets connectivity issues.

#### 2.2 Weekly Summary Compilation and Distribution

- **Overview:**  
  Aggregates the stored daily summaries from the Google Sheet every Friday evening, generates a comprehensive weekly report using Azure OpenAI, formats it into an email, and sends it to stakeholders.

- **Nodes Involved:**  
  - Friday Evening Trigger  
  - Get all stored data (Google Sheets)  
  - Flatten the input (Code)  
  - Azure OpenAI Chat Model1  
  - Structured Output Parser  
  - Create Weekly Summary (LangChain Agent)  
  - Create Email (Code)  
  - Email to Stakeholders (Gmail)  
  - Sticky Notes: Friday Evening Trigger, Get all stored data (Sheet), Flatten the input (Weekly), Chat Model (Weekly), Create Email, Email to Stakeholders

- **Node Details:**

  1. **Friday Evening Trigger**  
     - Type: Schedule Trigger  
     - Role: Triggers the weekly summary workflow every Friday at 8 PM.  
     - Configuration: Weekly interval, triggers on Friday (weekday 5) at 20:00 hours.  
     - Input/Output: No input; outputs trigger event for "Get all stored data".  
     - Failure Modes: Same as daily trigger.

  2. **Get all stored data (Google Sheets)**  
     - Type: Google Sheets  
     - Role: Reads all daily EOD entries stored in the Google Sheet during the week.  
     - Configuration: Reads entire Sheet1 content from the specified document.  
     - Credentials: Google Sheets OAuth2 API with automations user.  
     - Input: Trigger event from Friday Evening Trigger.  
     - Output: All rows with daily summaries and metadata.  
     - Failure Modes: Credential or access issues, API rate limits.

  3. **Flatten the input (Code Node)**  
     - Type: Code (JavaScript)  
     - Role: Normalizes and flattens the spreadsheet rows into a unified array format for AI summarization.  
     - Key Logic: Parses JSON strings in sheet cells, merges arrays, handles malformed data gracefully.  
     - Input: Google Sheets data rows.  
     - Output: Single item with combined array and count for the week.  
     - Failure Modes: Malformed JSON in sheet cells, empty or incomplete rows.

  4. **Azure OpenAI Chat Model1**  
     - Type: LangChain Azure OpenAI Chat Model (GPT-4o-mini)  
     - Role: Generates a structured weekly summary from the combined daily EOD summaries.  
     - Configuration: Model "gpt-4o-mini" with default options.  
     - Credentials: Azure OpenAI account linked.  
     - Input: Flattened combined weekly data as text.  
     - Output: Raw AI-generated weekly summary text.  
     - Failure Modes: API limitations, malformed input, generation errors.

  5. **Structured Output Parser**  
     - Type: LangChain Structured Output Parser  
     - Role: Parses the AI-generated weekly summary into a structured JSON format based on a detailed schema example.  
     - Input: Chat Model1 output.  
     - Output: Parsed weekly summary JSON.  
     - Failure Modes: Parsing errors if AI output is inconsistent with schema.

  6. **Create Weekly Summary (LangChain Agent)**  
     - Type: LangChain Agent (text generation with output parser)  
     - Role: Ensures the weekly summary follows strict formatting and content rules, synthesizing daily inputs into a coherent report.  
     - Configuration: System message defines exact templates, classification, aggregation rules, error handling, and output structure.  
     - Input: Flattened weekly combined daily summaries.  
     - Output: Structured weekly summary JSON with labeled sections.  
     - Failure Modes: Missing or malformed daily inputs, incomplete data, prompt errors.

  7. **Create Email (Code Node)**  
     - Type: Code (JavaScript)  
     - Role: Converts the weekly summary text into a styled, mobile-friendly HTML email with subject and preview text.  
     - Key Logic: Parses markdown-like headings, converts bold text and Jira issue keys into styled HTML, formats bullet points and code blocks, includes timestamp in IST timezone.  
     - Input: Weekly summary output text.  
     - Output: Single item with `html`, `subject`, and `plainTextPreview` fields for email sending.  
     - Failure Modes: Malformed input text causing rendering issues.

  8. **Email to Stakeholders (Gmail)**  
     - Type: Gmail Node  
     - Role: Sends the formatted weekly summary email to configured stakeholders.  
     - Configuration: Sends to "jyothi.swarup@techdome.net.in" with subject and HTML body from the previous node.  
     - Credentials: Gmail OAuth2 with user "jyothi".  
     - Input: Email payload from Create Email.  
     - Output: Confirmation of email sent.  
     - Failure Modes: Authentication failures, Gmail API quota limits, network issues.

### 3. Summary Table

| Node Name              | Node Type                                | Functional Role                                   | Input Node(s)             | Output Node(s)              | Sticky Note                                                                                   |
|------------------------|-----------------------------------------|-------------------------------------------------|---------------------------|-----------------------------|----------------------------------------------------------------------------------------------|
| Daily Evening Trigger   | Schedule Trigger                        | Triggers daily EOD workflow every workday evening | —                         | Get ALL issues               | ## Daily Evening Trigger Schedules and initiates the EOD workflow each workday evening automatically. |
| Get ALL issues         | Jira Node                              | Retrieves all Jira issues for the day             | Daily Evening Trigger      | Flatten Input                | ## Get ALL issues (Jira) Retrieves all Jira issues for the day, including statuses, priorities, and assignees, to power the EOD summary. |
| Flatten Input          | Code Node                             | Normalizes and flattens Jira JSON into a consistent format | Get ALL issues             | Create Summary              | ## Flatten Input Normalizes and flattens the Jira JSON into a lighter structure so downstream nodes can parse consistently. |
| Azure OpenAI Chat Model | LangChain Azure OpenAI Chat Model     | Generates structured EOD summary from flattened data | Flatten Input              | Structured Output Parser1    | ## Chat Model (EOD) Generates a structured End‑of‑Day summary from the flattened issue data using the configured Azure OpenAI model. |
| Structured Output Parser1 | LangChain Output Parser Structured    | Parses AI-generated EOD summary into structured JSON | Azure OpenAI Chat Model    | Create Summary              |                                                                                              |
| Create Summary          | LangChain Agent                       | Orchestrates AI prompt and parsing for EOD summary | Structured Output Parser1  | Append Summary               |                                                                                              |
| Append Summary          | Google Sheets                         | Appends daily summary and metadata to Google Sheet | Create Summary             | —                           | ## Append Summary (Sheet) Writes the EOD summary and the day’s metadata into the Excel/Sheet store for later weekly aggregation. |
| Friday Evening Trigger  | Schedule Trigger                      | Triggers weekly summary workflow every Friday evening | —                         | Get all stored data          | ## Friday Evening Trigger Starts the weekly compilation workflow every Friday evening to build and send the weekly report. |
| Get all stored data     | Google Sheets                        | Reads all stored daily summaries from Google Sheet | Friday Evening Trigger     | Flatten the input            | ## Get all stored data (Sheet) Reads the week’s EOD entries from the sheet, providing the full set of daily summaries for the weekly report. |
| Flatten the input       | Code Node                           | Normalizes and flattens weekly sheet data for AI input | Get all stored data        | Create Weekly Summary        | ## Flatten the input (Weekly) Converts the read rows into a unified format for the weekly AI summarization step. |
| Azure OpenAI Chat Model1 | LangChain Azure OpenAI Chat Model   | Generates structured weekly summary from combined daily data | Flatten the input          | Structured Output Parser     | ## Chat Model (Weekly) Produces a consistent Weekly Summary by analyzing the five daily EOD entries (Monday–Friday). |
| Structured Output Parser | LangChain Output Parser Structured  | Parses AI-generated weekly summary into structured JSON | Azure OpenAI Chat Model1   | Create Weekly Summary        |                                                                                              |
| Create Weekly Summary   | LangChain Agent                     | Orchestrates AI prompt and parsing for weekly summary | Structured Output Parser   | Create Email                |                                                                                              |
| Create Email            | Code Node                         | Formats weekly summary text into styled HTML email | Create Weekly Summary      | Email to Stakeholders        | ## Create Email Transforms the weekly summary into styled, mobile‑friendly HTML with a clear subject and preview. |
| Email to Stakeholders   | Gmail Node                         | Sends weekly summary email to stakeholders        | Create Email               | —                           | ## Email to Stakeholders Sends the final weekly report email to the stakeholder list using the configured email provider. |

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node** named `Daily Evening Trigger`:  
   - Type: Schedule Trigger  
   - Set interval to trigger daily on workdays (default daily interval).

2. **Add a Jira node** named `Get ALL issues`:  
   - Operation: `getAll`  
   - Return all issues (`returnAll`=true)  
   - Set credentials with your Jira Software Cloud API (OAuth or API token).  
   - Connect `Daily Evening Trigger` output to this node.

3. **Add a Code node** named `Flatten Input`:  
   - Paste the provided JavaScript code to flatten and normalize any incoming JSON array or stringified JSON into a single combined array.  
   - Connect `Get ALL issues` output to this node.

4. **Add a LangChain Azure OpenAI Chat Model node** named `Azure OpenAI Chat Model`:  
   - Select model: `gpt-4o-mini`  
   - Configure Azure OpenAI API credentials.  
   - Connect `Flatten Input` output to this node's input for text generation.

5. **Add a LangChain Structured Output Parser node** named `Structured Output Parser1`:  
   - Paste the JSON schema example for daily EOD summary parsing (as per example).  
   - Connect `Azure OpenAI Chat Model` output to this node.

6. **Add a LangChain Agent node** named `Create Summary`:  
   - Configure with a system message specifying the EOD summary generation rules, input format, output template, classification rules, formatting, error handling, and consistency checks.  
   - Enable output parser and link it to `Structured Output Parser1`.  
   - Input is the output of `Flatten Input` node as text.  
   - Connect `Structured Output Parser1` output to this node.

7. **Add a Google Sheets node** named `Append Summary`:  
   - Operation: `append`  
   - Sheet: Sheet1 (or your chosen sheet)  
   - Document ID: your Google Sheet ID storing daily summaries.  
   - Columns: Date (formatted as dd-MMM-yyyy), JSON (raw combined issues), Summary (from `Create Summary` output).  
   - Configure Google Sheets OAuth2 credentials.  
   - Connect `Create Summary` output to this node.

---

8. **Create a Schedule Trigger node** named `Friday Evening Trigger`:  
   - Type: Schedule Trigger  
   - Set to trigger weekly on Fridays at 20:00 hours.

9. **Add a Google Sheets node** named `Get all stored data`:  
   - Operation: `read` (default)  
   - Sheet: Sheet1 (same sheet as above)  
   - Document ID: the same Google Sheet document ID.  
   - Configure Google Sheets OAuth2 credentials.  
   - Connect `Friday Evening Trigger` output to this node.

10. **Add a Code node** named `Flatten the input`:  
    - Use similar JavaScript code as the daily flatten node to normalize all rows into a combined array of daily summaries.  
    - Connect `Get all stored data` output to this node.

11. **Add a LangChain Azure OpenAI Chat Model node** named `Azure OpenAI Chat Model1`:  
    - Select model: `gpt-4o-mini`  
    - Configure Azure OpenAI API credentials (same as daily).  
    - Connect `Flatten the input` output to this node.

12. **Add a LangChain Structured Output Parser node** named `Structured Output Parser`:  
    - Paste JSON schema example for weekly summary parsing (as per example).  
    - Connect `Azure OpenAI Chat Model1` output to this node.

13. **Add a LangChain Agent node** named `Create Weekly Summary`:  
    - Configure with system message defining weekly summary rules, input format (five daily entries), output template, classification, aggregation, formatting and error handling.  
    - Enable output parser linked to `Structured Output Parser`.  
    - Connect `Structured Output Parser` output to this node.

14. **Add a Code node** named `Create Email`:  
    - Paste the provided JavaScript code that formats the weekly summary text into a styled, mobile-friendly HTML email with subject and plain text preview.  
    - Connect `Create Weekly Summary` output to this node.

15. **Add a Gmail node** named `Email to Stakeholders`:  
    - Configure to send email with:  
      - To: `jyothi.swarup@techdome.net.in` (or your recipients)  
      - Subject: from `Create Email` output field `subject`  
      - Message (HTML): from `Create Email` output field `html`  
    - Configure Gmail OAuth2 credentials.  
    - Connect `Create Email` output to this node.

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow uses the Azure OpenAI GPT-4o-mini model for text generation to balance cost and quality during summarization.        | Azure OpenAI model reference                                                                       |
| Google Sheets serves as a persistent store for daily summaries, enabling weekly aggregation without querying Jira repeatedly.       | Google Sheets usage for data archiving                                                             |
| The workflow handles timezone explicitly, using Asia/Kolkata (IST) for date and time formatting in summaries and email generation. | Timezone consistency is critical for correct timestamp interpretation.                              |
| The weekly email uses inline CSS and HTML formatting for mobile-friendly and accessible reports.                                    | Email formatting best practices                                                                     |
| Prompt templates enforce strict output structure and error handling to ensure reliable parsing of AI outputs.                     | Prompt engineering is key to avoid malformed AI responses.                                         |
| Sticky Notes in the workflow provide contextual explanations aiding maintainability and onboarding.                                | Use sticky notes for collaborative understanding.                                                  |

---

**Disclaimer:** The provided content is extracted and analyzed from a legitimate n8n automation workflow. It respects all content policies and contains no illegal or offensive material. All data used is assumed to be public or authorized for processing.