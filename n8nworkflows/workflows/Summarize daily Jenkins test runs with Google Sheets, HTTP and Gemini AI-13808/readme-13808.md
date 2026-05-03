Summarize daily Jenkins test runs with Google Sheets, HTTP and Gemini AI

https://n8nworkflows.xyz/workflows/summarize-daily-jenkins-test-runs-with-google-sheets--http-and-gemini-ai-13808


# Summarize daily Jenkins test runs with Google Sheets, HTTP and Gemini AI

# 1. Workflow Overview

This workflow automates a daily Jenkins test reporting process. It reads configuration rows from Google Sheets, builds Jenkins URLs dynamically, retrieves test report data for each configured feature/environment, aggregates the results, asks Gemini AI to transform the raw Jenkins data into a polished HTML email, and sends that report to the recipients listed in the sheet.

Typical use cases:
- Daily QA/test automation status reporting
- Jenkins-based regression monitoring across environments
- Automated stakeholder email updates without manually reviewing Jenkins

## 1.1 Scheduled Execution
The workflow starts automatically every day at a fixed time using a Schedule Trigger.

## 1.2 Configuration Retrieval from Google Sheets
It loads rows from a Google Sheet tab that define:
- Jenkins base URL
- environment
- feature class
- feature
- mailing list

These values drive all downstream HTTP requests and email delivery.

## 1.3 Jenkins Report Retrieval
For each row from the sheet, the workflow:
1. Requests the Jenkins HTML test report page
2. Extracts the build number from the page
3. Requests the Jenkins JSON API version of the same report with `depth=2` to obtain richer test details, including failures

## 1.4 Data Normalization and Aggregation
The workflow reshapes the Jenkins payload and environment/build metadata into AI-friendly fields, then aggregates all row-level records into a single dataset for one AI summarization pass.

## 1.5 AI Report Generation
A LangChain agent uses the Gemini chat model to convert the aggregated Jenkins results into a professional HTML email with consistent styling and grouped reporting by environment and feature.

## 1.6 Email Delivery
The generated HTML is sent by Gmail to the mailing list found in the first Google Sheets row.

---

# 2. Block-by-Block Analysis

## 2.1 Scheduled Trigger

### Overview
This block launches the workflow automatically once per day. It is the single entry point for the entire process.

### Nodes Involved
- Trigger workflow daily on set time

### Node Details

#### Trigger workflow daily on set time
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Starts the workflow on a calendar/time-based schedule.
- **Configuration choices:**  
  Configured to trigger daily at `09:05`.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: none
  - Output: `Retrieve Google Sheet data`
- **Version-specific requirements:**  
  Uses `typeVersion 1.3`; standard schedule configuration.
- **Edge cases or potential failure types:**  
  - Workflow must be active to run automatically
  - Server timezone affects execution time
  - If n8n is down at the scheduled moment, execution may be missed depending on deployment/runtime setup
- **Sub-workflow reference:**  
  None.

---

## 2.2 Configuration Retrieval from Google Sheets

### Overview
This block fetches the list of Jenkins targets and email recipients from Google Sheets. Each row acts as a configuration record for one Jenkins feature report.

### Nodes Involved
- Retrieve Google Sheet data

### Node Details

#### Retrieve Google Sheet data
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Reads rows from a specific Google Sheets tab.
- **Configuration choices:**  
  - Uses a Google Sheets document ID credential/configuration, though the JSON shows the document ID value blank and expected to be supplied during setup
  - Reads from the sheet/tab named `DataRequest`
  - No special options are enabled
- **Key expressions or variables used:**  
  Downstream nodes reference:
  - `BaseUrl`
  - `Environment`
  - `FeatureClass`
  - `Feature`
  - `MailingList`
- **Input and output connections:**  
  - Input: `Trigger workflow daily on set time`
  - Output: `Perform HTTP Request to Jenkins Server for build numbers`
- **Version-specific requirements:**  
  Uses `typeVersion 4.7`; ensure compatible Google Sheets credentials are available in your n8n version.
- **Edge cases or potential failure types:**  
  - Missing or invalid Google credentials
  - Google Sheets API not enabled
  - Empty document ID or inaccessible spreadsheet
  - Missing expected columns causes downstream expression failures
  - Multiple rows are supported, but later email logic uses only the first row’s `MailingList`
- **Sub-workflow reference:**  
  None.

---

## 2.3 Jenkins Report Retrieval

### Overview
This block builds Jenkins URLs from each Google Sheets row, retrieves the HTML test report page to discover the build number, then requests the JSON API version of the same report for detailed test content.

### Nodes Involved
- Perform HTTP Request to Jenkins Server for build numbers
- Extract the build number
- Perform HTTP Request with depth 2 to Jenkins Server

### Node Details

#### Perform HTTP Request to Jenkins Server for build numbers
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Downloads the Jenkins test report page as HTML.
- **Configuration choices:**  
  URL is dynamically constructed as:
  `BaseUrl/Environment/lastCompletedBuild/testReport/FeatureClass.Feature/`
- **Key expressions or variables used:**  
  - `{{ $node["Retrieve Google Sheet data"].json.BaseUrl }}`
  - `{{ $node["Retrieve Google Sheet data"].json.Environment }}`
  - `{{ $node["Retrieve Google Sheet data"].json.FeatureClass }}`
  - `{{ $node["Retrieve Google Sheet data"].json.Feature }}`
- **Input and output connections:**  
  - Input: `Retrieve Google Sheet data`
  - Output: `Extract the build number`
- **Version-specific requirements:**  
  Uses `typeVersion 4.4`.
- **Edge cases or potential failure types:**  
  - Jenkins server unreachable
  - Invalid base URL or path formatting
  - Authentication required by Jenkins but not configured here
  - `lastCompletedBuild` may not exist
  - HTML layout could differ, affecting downstream extraction
- **Sub-workflow reference:**  
  None.

#### Extract the build number
- **Type and technical role:** `n8n-nodes-base.html`  
  Parses the Jenkins HTML and extracts the build number text from a CSS selector.
- **Configuration choices:**  
  - Operation: extract HTML content
  - Extracts a field named `build_number`
  - CSS selector: `#breadcrumbs > li:nth-child(3) > a`
- **Key expressions or variables used:**  
  None directly; it parses prior node output.
- **Input and output connections:**  
  - Input: `Perform HTTP Request to Jenkins Server for build numbers`
  - Output: `Perform HTTP Request with depth 2 to Jenkins Server`
- **Version-specific requirements:**  
  Uses `typeVersion 1.2`.
- **Edge cases or potential failure types:**  
  - CSS selector may break if Jenkins theme/layout differs
  - Missing breadcrumb element returns empty extraction
  - If the upstream response is not valid HTML, extraction fails or returns blank
- **Sub-workflow reference:**  
  None.

#### Perform HTTP Request with depth 2 to Jenkins Server
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Fetches detailed Jenkins test report data in JSON form.
- **Configuration choices:**  
  - URL is dynamically constructed similarly to the prior request, with `/api/json?depth=2`
  - Response handling is customized:
    - full response enabled
    - response format as text
    - stored in output property `test_data`
- **Key expressions or variables used:**  
  - `{{ $node["Retrieve Google Sheet data"].json.BaseUrl }}`
  - `{{ $node["Retrieve Google Sheet data"].json.Environment }}`
  - `{{ $node["Retrieve Google Sheet data"].json.FeatureClass }}`
  - `{{ $node["Retrieve Google Sheet data"].json.Feature }}`
- **Input and output connections:**  
  - Input: `Extract the build number`
  - Output: `Set the fields correctly for AI`
- **Version-specific requirements:**  
  Uses `typeVersion 4.4`.
- **Edge cases or potential failure types:**  
  - Jenkins API endpoint may require auth
  - Response might be large for some reports
  - Because the response is captured as text, invalid or unexpected JSON text will later break parsing in the AI node
  - `depth=2` may still be insufficient if a custom Jenkins plugin nests data differently
- **Sub-workflow reference:**  
  None.

---

## 2.4 Data Normalization and Aggregation

### Overview
This block reshapes the Jenkins and metadata payload into simpler fields expected by the AI prompt, then combines all items into one aggregated structure so a single AI call can summarize the full report.

### Nodes Involved
- Set the fields correctly for AI
- Aggregate the data for AI processing

### Node Details

#### Set the fields correctly for AI
- **Type and technical role:** `n8n-nodes-base.set`  
  Adds normalized fields and preserves all other incoming fields.
- **Configuration choices:**  
  Assigns:
  - `environment` from Google Sheets `Environment`
  - `build_Number` as a stringified object from the HTML extraction output
  - `payload` from `test_data`
  - `includeOtherFields` enabled, so original fields remain available
- **Key expressions or variables used:**  
  - `{{ $node["Retrieve Google Sheet data"].json.Environment }}`
  - `{{ $node["Extract the build number"].json }}`
  - `{{ $json.test_data }}`
- **Input and output connections:**  
  - Input: `Perform HTTP Request with depth 2 to Jenkins Server`
  - Output: `Aggregate the data for AI processing`
- **Version-specific requirements:**  
  Uses `typeVersion 3.4`.
- **Edge cases or potential failure types:**  
  - `build_Number` is stored as a stringified object-like value and later parsed with `JSON.parse`; malformed content breaks the AI prompt
  - `payload` is set, but the AI prompt actually references `test_data`, so `payload` is not the main field used later
  - If upstream `test_data` is empty, aggregation still occurs but AI parsing will fail
- **Sub-workflow reference:**  
  None.

#### Aggregate the data for AI processing
- **Type and technical role:** `n8n-nodes-base.aggregate`  
  Combines all incoming items into a single item containing all item data.
- **Configuration choices:**  
  - Aggregate mode: `aggregateAllItemData`
- **Key expressions or variables used:**  
  Downstream AI prompt references aggregated data via:
  - `$json.data`
- **Input and output connections:**  
  - Input: `Set the fields correctly for AI`
  - Output: `Jenkins Report Analyzer`
- **Version-specific requirements:**  
  Uses `typeVersion 1`.
- **Edge cases or potential failure types:**  
  - If no items arrive, AI receives no meaningful dataset
  - Large numbers of rows can create a very large prompt and exceed model input/token limits
- **Sub-workflow reference:**  
  None.

---

## 2.5 AI Report Generation

### Overview
This block converts aggregated Jenkins data into a single HTML email body. It uses a LangChain agent with Gemini as the language model and a detailed system message containing parsing and formatting instructions.

### Nodes Involved
- Google Gemini Chat Model
- Jenkins Report Analyzer

### Node Details

#### Google Gemini Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
  Provides the Gemini LLM connection used by the agent node.
- **Configuration choices:**  
  Minimal/default options; credentials are expected to be configured in n8n.
- **Key expressions or variables used:**  
  None in this node itself.
- **Input and output connections:**  
  - AI language model output to: `Jenkins Report Analyzer`
- **Version-specific requirements:**  
  Uses `typeVersion 1`; requires compatible n8n LangChain integration and Google Gemini credentials/API access.
- **Edge cases or potential failure types:**  
  - Missing/invalid Gemini credentials
  - API quota/rate limits
  - Model availability or regional restrictions
  - Prompt too large due to aggregated Jenkins payload
- **Sub-workflow reference:**  
  None.

#### Jenkins Report Analyzer
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  Orchestrates LLM prompting and produces the final HTML report.
- **Configuration choices:**  
  - Prompt asks for overview of failed (`Gefaald`), skipped (`Overgeslagen`), passed (`Pass`), and total (`Totaal`) tests per feature
  - Requests failed tests by name
  - Requests summarization per environment and per feature combined into one email
  - System message:
    - Defines environment meaning: `develop = test`, `master = acceptance`
    - Parses aggregated data from `$json.data`
    - Uses `JSON.parse(item.build_Number)` and `JSON.parse(item.test_data)`
    - Builds a structured textual representation of package/story/test results
    - Instructs the model to return professional HTML only
    - Requires inline CSS suitable for email
    - Requires output to begin with `<html>` and end with `</html>`
- **Key expressions or variables used:**  
  Critical expression in the system message:
  - `{{ $json.data.map(item => { ... }).join(...) }}`
  Also uses:
  - `JSON.parse(item.build_Number)`
  - `JSON.parse(item.test_data)`
- **Input and output connections:**  
  - Main input: `Aggregate the data for AI processing`
  - AI model input: `Google Gemini Chat Model`
  - Main output: `Send the test results based on the MailingList`
- **Version-specific requirements:**  
  Uses `typeVersion 3.1`; requires n8n LangChain/AI nodes support.
- **Edge cases or potential failure types:**  
  - Invalid JSON in `build_Number` or `test_data` breaks prompt evaluation before model execution
  - LLM may produce malformed HTML despite instructions
  - Prompt references localized status labels (`Gefaald`, `Overgeslagen`, `Pass`, `Totaal`) while source JSON may expose statuses like `FAILED`; consistency depends on Jenkins payload/plugin language
  - If aggregated input is too large, request may fail or truncate
  - Email HTML may not render equally across clients
- **Sub-workflow reference:**  
  None.

---

## 2.6 Email Delivery

### Overview
This block sends the AI-generated HTML report by Gmail to the configured recipients. It uses the mailing list from the first Google Sheets row and inserts the AI output directly as the message body.

### Nodes Involved
- Send the test results based on the MailingList

### Node Details

#### Send the test results based on the MailingList
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends an email through Gmail.
- **Configuration choices:**  
  - Recipient:
    `{{ $('Retrieve Google Sheet data').first().json.MailingList }}`
  - Subject:
    `Jenkins - Test Results Report - {{new Date().toLocaleDateString()}}`
  - Message body:
    `{{ $json.output }}`
- **Key expressions or variables used:**  
  - `$('Retrieve Google Sheet data').first().json.MailingList`
  - `$json.output`
  - `new Date().toLocaleDateString()`
- **Input and output connections:**  
  - Input: `Jenkins Report Analyzer`
  - Output: none
- **Version-specific requirements:**  
  Uses `typeVersion 2.2`; requires Gmail OAuth2 credentials in n8n.
- **Edge cases or potential failure types:**  
  - Gmail auth failure
  - Sending limits or anti-spam restrictions
  - If multiple sheet rows contain different mailing lists, only the first row is used
  - If `$json.output` is empty or non-HTML, the email body may be blank or malformed
- **Sub-workflow reference:**  
  None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Trigger workflow daily on set time | n8n-nodes-base.scheduleTrigger | Daily workflow entry point |  | Retrieve Google Sheet data | ## Automate daily Jenkins test reports with AI and HTTP Requests<br>As a test automation engineer, staying on top of daily test runs in Jenkins is essential. This workflow automates that process: it pulls specific test details from a Google Sheet, retrieves data from your local Jenkins environment, and uses AI to generate a concise summary report to be sent via email.<br><br>**Who's it for**<br>* Test automation engineers using Jenkins.<br>* QA teams looking to streamline daily reporting.<br><br>**How it works**<br>* Scheduled Trigger: The workflow runs automatically at a pre-defined time.<br>* Dynamic Data Retrieval: It constructs an HTTP request based on the data in your Google Sheet to fetch specific Jenkins results.<br>* AI Optimization: Only relevant data is extracted to minimize AI token usage and focus on the most important metrics.<br>* Summarization: The AI groups the results and formats them into a clear, professional email.<br>* Distribution: The report is sent to every recipient listed in the MailingList column.<br><br>**How to set up**<br>* In the Google Sheet, set the BaseUrl, Environment, FeatureClass and Feature in order to build up the Jenkins url in their corresponding columns, for example:<br>BaseUrl \| Environment \| FeatureClass \| Feature \| MailingList<br>&lt;BaseUrl&gt; \| &lt;environment&gt; \| &lt;FeaturClassName&gt; \| &lt;Featurename&gt; \| &lt;mail&gt;<br>* Define Recipients: In the MailingList column, add the email addresses of the people who need to receive the report. If there are multiple recipients, ensure they are separated by commas.<br><br>**Requirements**<br>* Access to your Jenkins server.<br>* An AI API key (e.g., Gemini, OpenAI).<br>* A Google Cloud project with the Google Sheets API enabled. |
| Retrieve Google Sheet data | n8n-nodes-base.googleSheets | Load Jenkins request definitions and recipients from Google Sheets | Trigger workflow daily on set time | Perform HTTP Request to Jenkins Server for build numbers | ## Automate daily Jenkins test reports with AI and HTTP Requests<br>As a test automation engineer, staying on top of daily test runs in Jenkins is essential. This workflow automates that process: it pulls specific test details from a Google Sheet, retrieves data from your local Jenkins environment, and uses AI to generate a concise summary report to be sent via email.<br><br>**Who's it for**<br>* Test automation engineers using Jenkins.<br>* QA teams looking to streamline daily reporting.<br><br>**How it works**<br>* Scheduled Trigger: The workflow runs automatically at a pre-defined time.<br>* Dynamic Data Retrieval: It constructs an HTTP request based on the data in your Google Sheet to fetch specific Jenkins results.<br>* AI Optimization: Only relevant data is extracted to minimize AI token usage and focus on the most important metrics.<br>* Summarization: The AI groups the results and formats them into a clear, professional email.<br>* Distribution: The report is sent to every recipient listed in the MailingList column.<br><br>**How to set up**<br>* In the Google Sheet, set the BaseUrl, Environment, FeatureClass and Feature in order to build up the Jenkins url in their corresponding columns, for example:<br>BaseUrl \| Environment \| FeatureClass \| Feature \| MailingList<br>&lt;BaseUrl&gt; \| &lt;environment&gt; \| &lt;FeaturClassName&gt; \| &lt;Featurename&gt; \| &lt;mail&gt;<br>* Define Recipients: In the MailingList column, add the email addresses of the people who need to receive the report. If there are multiple recipients, ensure they are separated by commas.<br><br>**Requirements**<br>* Access to your Jenkins server.<br>* An AI API key (e.g., Gemini, OpenAI).<br>* A Google Cloud project with the Google Sheets API enabled. |
| Perform HTTP Request to Jenkins Server for build numbers | n8n-nodes-base.httpRequest | Fetch Jenkins HTML report page | Retrieve Google Sheet data | Extract the build number | ## Retrieve the Jenkins Test Data<br>* First we have to retrieve the report in order to get the build numbers, needed for the AI to know which report we are talking about.<br>* Then we extract the build numbers from the test reports using the HTML extract node.<br>* In order to have a complete test report, we perform an additional HTTP request but with an "/api/json?depth=2" to get all data, including the failed results. |
| Extract the build number | n8n-nodes-base.html | Parse Jenkins HTML to extract build number | Perform HTTP Request to Jenkins Server for build numbers | Perform HTTP Request with depth 2 to Jenkins Server | ## Retrieve the Jenkins Test Data<br>* First we have to retrieve the report in order to get the build numbers, needed for the AI to know which report we are talking about.<br>* Then we extract the build numbers from the test reports using the HTML extract node.<br>* In order to have a complete test report, we perform an additional HTTP request but with an "/api/json?depth=2" to get all data, including the failed results. |
| Perform HTTP Request with depth 2 to Jenkins Server | n8n-nodes-base.httpRequest | Fetch detailed Jenkins JSON report | Extract the build number | Set the fields correctly for AI | ## Retrieve the Jenkins Test Data<br>* First we have to retrieve the report in order to get the build numbers, needed for the AI to know which report we are talking about.<br>* Then we extract the build numbers from the test reports using the HTML extract node.<br>* In order to have a complete test report, we perform an additional HTTP request but with an "/api/json?depth=2" to get all data, including the failed results. |
| Set the fields correctly for AI | n8n-nodes-base.set | Normalize environment, build and payload fields | Perform HTTP Request with depth 2 to Jenkins Server | Aggregate the data for AI processing |  |
| Aggregate the data for AI processing | n8n-nodes-base.aggregate | Merge all Jenkins records into one AI input | Set the fields correctly for AI | Jenkins Report Analyzer |  |
| Google Gemini Chat Model | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Provide Gemini model connection for the AI agent |  | Jenkins Report Analyzer |  |
| Jenkins Report Analyzer | @n8n/n8n-nodes-langchain.agent | Generate HTML summary email from aggregated Jenkins data | Aggregate the data for AI processing; Google Gemini Chat Model | Send the test results based on the MailingList |  |
| Send the test results based on the MailingList | n8n-nodes-base.gmail | Send final HTML email via Gmail | Jenkins Report Analyzer |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Visual documentation note |  |  | ## Automate daily Jenkins test reports with AI and HTTP Requests<br>As a test automation engineer, staying on top of daily test runs in Jenkins is essential. This workflow automates that process: it pulls specific test details from a Google Sheet, retrieves data from your local Jenkins environment, and uses AI to generate a concise summary report to be sent via email.<br><br>**Who's it for**<br>* Test automation engineers using Jenkins.<br>* QA teams looking to streamline daily reporting.<br><br>**How it works**<br>* Scheduled Trigger: The workflow runs automatically at a pre-defined time.<br>* Dynamic Data Retrieval: It constructs an HTTP request based on the data in your Google Sheet to fetch specific Jenkins results.<br>* AI Optimization: Only relevant data is extracted to minimize AI token usage and focus on the most important metrics.<br>* Summarization: The AI groups the results and formats them into a clear, professional email.<br>* Distribution: The report is sent to every recipient listed in the MailingList column.<br><br>**How to set up**<br>* In the Google Sheet, set the BaseUrl, Environment, FeatureClass and Feature in order to build up the Jenkins url in their corresponding columns, for example:<br>BaseUrl \| Environment \| FeatureClass \| Feature \| MailingList<br>&lt;BaseUrl&gt; \| &lt;environment&gt; \| &lt;FeaturClassName&gt; \| &lt;Featurename&gt; \| &lt;mail&gt;<br>* Define Recipients: In the MailingList column, add the email addresses of the people who need to receive the report. If there are multiple recipients, ensure they are separated by commas.<br><br>**Requirements**<br>* Access to your Jenkins server.<br>* An AI API key (e.g., Gemini, OpenAI).<br>* A Google Cloud project with the Google Sheets API enabled. |
| Sticky Note | n8n-nodes-base.stickyNote | Visual documentation note |  |  | ## Retrieve the Jenkins Test Data<br>* First we have to retrieve the report in order to get the build numbers, needed for the AI to know which report we are talking about.<br>* Then we extract the build numbers from the test reports using the HTML extract node.<br>* In order to have a complete test report, we perform an additional HTTP request but with an "/api/json?depth=2" to get all data, including the failed results. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Automate daily Jenkins test reports with AI and HTTP Requests`.

2. **Add a Schedule Trigger node**
   - Node type: **Schedule Trigger**
   - Name: `Trigger workflow daily on set time`
   - Configure it to run every day at `09:05`
   - Keep in mind the n8n instance timezone determines actual run time

3. **Prepare the Google Sheet**
   - Create a Google Sheet with a tab named `DataRequest`
   - Add these columns exactly:
     - `BaseUrl`
     - `Environment`
     - `FeatureClass`
     - `Feature`
     - `MailingList`
   - Example row structure:
     - `BaseUrl`: Jenkins base URL
     - `Environment`: e.g. `develop` or `master`
     - `FeatureClass`: package/class group used by Jenkins report path
     - `Feature`: feature/test suite name
     - `MailingList`: one or more emails separated by commas

4. **Add a Google Sheets node**
   - Node type: **Google Sheets**
   - Name: `Retrieve Google Sheet data`
   - Connect it from the Schedule Trigger
   - Configure credentials using Google OAuth2 or Service Account supported by your n8n setup
   - Select the target spreadsheet
   - Set the sheet/tab to `DataRequest`
   - Configure the node to read rows from the sheet

5. **Add the first HTTP Request node**
   - Node type: **HTTP Request**
   - Name: `Perform HTTP Request to Jenkins Server for build numbers`
   - Connect it from `Retrieve Google Sheet data`
   - Set URL to this expression:
     ```text
     ={{ $node["Retrieve Google Sheet data"].json.BaseUrl }}/{{ $node["Retrieve Google Sheet data"].json.Environment }}/lastCompletedBuild/testReport/{{ $node["Retrieve Google Sheet data"].json.FeatureClass }}.{{ $node["Retrieve Google Sheet data"].json.Feature }}/
     ```
   - Leave response handling at defaults unless your Jenkins requires special headers/auth
   - If Jenkins requires authentication, add it here

6. **Add an HTML node**
   - Node type: **HTML**
   - Name: `Extract the build number`
   - Connect it from the first HTTP Request node
   - Operation: `Extract HTML Content`
   - Add one extraction value:
     - Key: `build_number`
     - CSS Selector: `#breadcrumbs > li:nth-child(3) > a`
   - This assumes your Jenkins HTML structure matches the selector

7. **Add the second HTTP Request node**
   - Node type: **HTTP Request**
   - Name: `Perform HTTP Request with depth 2 to Jenkins Server`
   - Connect it from `Extract the build number`
   - Set URL to:
     ```text
     ={{ $node["Retrieve Google Sheet data"].json.BaseUrl }}/{{ $node["Retrieve Google Sheet data"].json.Environment }}/lastCompletedBuild/testReport/{{ $node["Retrieve Google Sheet data"].json.FeatureClass }}.{{ $node["Retrieve Google Sheet data"].json.Feature }}/api/json?depth=2
     ```
   - In response options:
     - Enable full response
     - Set response format to `Text`
     - Set output property name to `test_data`
   - This is important because the downstream AI node expects `test_data`

8. **Add a Set node**
   - Node type: **Set**
   - Name: `Set the fields correctly for AI`
   - Connect it from the second HTTP Request node
   - Add these assignments:
     - `environment` as String:
       ```text
       ={{ $node["Retrieve Google Sheet data"].json.Environment }}
       ```
     - `build_Number` as String:
       ```text
       ={{ $node["Extract the build number"].json }}
       ```
     - `payload` as String:
       ```text
       ={{ $json.test_data }}
       ```
   - Enable **Include Other Input Fields**
   - Note: even though `payload` is created, the AI prompt still uses `test_data`

9. **Add an Aggregate node**
   - Node type: **Aggregate**
   - Name: `Aggregate the data for AI processing`
   - Connect it from `Set the fields correctly for AI`
   - Set aggregation mode to `Aggregate All Item Data`
   - This produces a single item with a `data` array containing all rows/results

10. **Add a Google Gemini Chat Model node**
    - Node type: **Google Gemini Chat Model**
    - Name: `Google Gemini Chat Model`
    - Configure Gemini credentials/API key
    - Leave options at defaults unless you want to control model behavior
    - Ensure your n8n instance supports LangChain AI nodes

11. **Add an AI Agent node**
    - Node type: **AI Agent** / `LangChain Agent`
    - Name: `Jenkins Report Analyzer`
    - Connect the main input from `Aggregate the data for AI processing`
    - Connect the AI language model input from `Google Gemini Chat Model`
    - Set prompt type to define/custom text
    - Set the user text/prompt to:
      ```text
      Use the provided data to create an overview of the failed ("Gefaald"), skipped ("Overgeslagen"), passed ("Pass") and total ("Totaal") tests for each feature.
      If it has failed ("Gefaald") tests, list them by name.

      Summize them per environment and per feature and combine them in one email.
      ```
    - Set the system message to:
      ```text
      =You are a Jenkins Report analyzer, and your job is to get the results of each environment (develop = test, master = acceptance), and report back the results of each environment and which tests passed and/or failed.
      If failed, show the specific failed test and the corresponding error.
      Use the data of {{ $json.data.map(item => {
          const buildInfo = JSON.parse(item.build_Number);
          const pkg = JSON.parse(item.test_data);
          
          const header = `### ENV: ${item.environment.toUpperCase()} | BUILD: ${buildInfo.build_number} ###\n`;
          
          const body = `PACKAGE: ${pkg.name} (Pass: ${pkg.passCount}, Fail: ${pkg.failCount})\n` + 
              pkg.child.map(story => 
                  `  STORY: ${story.name}\n` + 
                  story.child.map(test => 
                      `    [${test.status}] ${test.name}${test.status === 'FAILED' ? '\n    ERROR: ' + (test.errorDetails ? test.errorDetails.trim() : 'No details available') : ''}`
                  ).join('\n')
              ).join('\n');

          return header + body;
      }).join('\n\n====================\n\n') }}

      Beautify the HTML using inline CSS styling for an email. Format it like a modern newsletter. Output only the results without any header elements.
      Keep the formatting and layout each mail the same.
      Generate a professional HTML email report
      Output ONLY the raw HTML code. 
      DO NOT include introductory text (like 'Here is your report').
      DO NOT include markdown code blocks (no ```html).
      DO NOT include explanations, justifications, or Python code.
      Start your response with <html> and end it with </html>.
      ```
    - Keep default tool settings unless you intend to add tools

12. **Add a Gmail node**
    - Node type: **Gmail**
    - Name: `Send the test results based on the MailingList`
    - Connect it from `Jenkins Report Analyzer`
    - Configure Gmail OAuth2 credentials
    - Set **To**:
      ```text
      ={{ $('Retrieve Google Sheet data').first().json.MailingList }}
      ```
    - Set **Subject**:
      ```text
      =Jenkins - Test Results Report - {{new Date().toLocaleDateString()}}
      ```
    - Set **Message**:
      ```text
      ={{ $json.output }}
      ```
    - If available in your Gmail node version, configure the body as HTML or ensure the message field is treated as HTML content

13. **Add optional sticky notes for documentation**
    - Add one note describing the overall workflow and required sheet columns
    - Add another note describing the Jenkins retrieval sequence and purpose of the `depth=2` API request

14. **Validate credentials**
    - Google Sheets credentials must access the spreadsheet
    - Gemini credentials must be valid and have quota
    - Gmail credentials must allow sending mail
    - Jenkins must be reachable from the n8n host
    - If Jenkins is protected, add auth to both HTTP Request nodes

15. **Run a manual test**
    - Execute the workflow manually
    - Confirm the Google Sheets node returns rows
    - Confirm the first Jenkins HTTP request returns HTML
    - Confirm the HTML node extracts `build_number`
    - Confirm the second Jenkins HTTP request returns JSON text in `test_data`
    - Confirm the aggregate node outputs a `data` array
    - Confirm the AI node outputs valid HTML
    - Confirm Gmail sends to the expected recipients

16. **Activate the workflow**
    - Once validated, activate the workflow so the daily schedule starts running automatically

## Reproduction Notes and Constraints
- The workflow currently sends the report only to `MailingList` from the **first** sheet row, even if later rows contain different recipients.
- The workflow depends on the Jenkins HTML breadcrumb selector for build extraction. If your Jenkins theme differs, adjust the CSS selector.
- The AI prompt expects `item.test_data` to be JSON text. Do not convert it to a nested object unless you also update the prompt logic.
- For large Jenkins responses, consider preprocessing to reduce prompt size before the AI node.

## Sub-workflow Setup
- This workflow does **not** use any sub-workflows.
- There are no Execute Workflow or workflow-call nodes.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow is designed for test automation engineers and QA teams who need a daily Jenkins summary email. | Overall purpose |
| Google Sheet columns expected: `BaseUrl`, `Environment`, `FeatureClass`, `Feature`, `MailingList`. | Required input structure |
| Multiple recipients in `MailingList` should be comma-separated. | Email delivery setup |
| Jenkins access is required from the n8n runtime environment. | Infrastructure requirement |
| A Google Cloud project with Google Sheets API enabled is required. | Google Sheets integration |
| An AI provider credential such as Gemini is required. | AI integration |
| The workflow uses a two-step Jenkins retrieval process: HTML page for build number, then `/api/json?depth=2` for detailed test data. | Jenkins data acquisition logic |
| The current email recipient expression uses only the first Google Sheets row: `$('Retrieve Google Sheet data').first().json.MailingList`. | Important implementation behavior |