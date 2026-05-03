Automated Workflow Test Runner with Google Drive, Sheets & ClickUp Result Logging

https://n8nworkflows.xyz/workflows/automated-workflow-test-runner-with-google-drive--sheets---clickup-result-logging-9408


# Automated Workflow Test Runner with Google Drive, Sheets & ClickUp Result Logging

### 1. Workflow Overview

This workflow automates the execution and monitoring of a target n8n workflow ("Archive Payment Receipts with Stripe, Google Drive, and Google Sheets") for testing purposes. It is designed for automated regression testing and continuous integration/continuous deployment (CI/CD) pipelines, enabling teams to validate workflow runs, log results, and maintain audit trails.

The workflow logic is divided into the following functional blocks:

- **1.1 Input Reception:** Manual trigger to start the automated test run.
- **1.2 Execution of Target Workflow:** Executes a specified sub-workflow and captures its output including any errors.
- **1.3 Test Result Evaluation:** Checks whether the executed sub-workflow succeeded or failed by detecting error presence.
- **1.4 Success Handling:** Formats success results, generates human-readable reports, archives logs to Google Drive, and updates a ClickUp task.
- **1.5 Failure Handling:** Formats failure results, generates reports, archives failure logs to Google Drive, updates ClickUp, and logs error details to a Google Sheets error tracking sheet.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Initiates the entire automated test process via manual trigger.

**Nodes Involved:**  
- When clicking ‘Execute workflow’

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Starts the workflow manually on user demand.  
  - Configuration: No parameters configured; simple manual execution trigger.  
  - Inputs: None  
  - Outputs: Connects to "Execute Target Workflow Under Test" node.  
  - Edge Cases: User must manually trigger; no automation without user action.

---

#### 1.2 Execution of Target Workflow

**Overview:**  
Calls the target workflow ("Archive Payment Receipts") as a sub-process, waits for completion, and captures its output including errors.

**Nodes Involved:**  
- Execute Target Workflow Under Test  
- Sticky Note (description for this block)

**Node Details:**

- **Execute Target Workflow Under Test**  
  - Type: Execute Workflow (Sub-workflow execution)  
  - Role: Runs the workflow under test and collects output results synchronously.  
  - Configuration:  
    - WorkflowId set to "Archive Payment Receipts with Stripe, Google Drive, and Google Sheets" (ID: gr2rafzxOw1ks4Bd)  
    - Waits for sub-workflow completion (`waitForSubWorkflow: true`)  
    - Continues execution even if sub-workflow errors (`onError: continueRegularOutput`)  
    - No input parameters passed to sub-workflow.  
  - Inputs: Triggered by manual trigger node.  
  - Outputs: Passes data to "Test Result Evaluation" node.  
  - Edge Cases: Sub-workflow errors are handled gracefully; output may contain error objects for downstream evaluation.

- **Sticky Note** (positioned near this node)  
  - Explains the purpose and configuration of this execution step.

---

#### 1.3 Test Result Evaluation

**Overview:**  
Determines if the executed sub-workflow succeeded or failed based on presence of an error field in the JSON output, routing to appropriate success or failure handling branches.

**Nodes Involved:**  
- Test Result Evaluation (If node)  
- Sticky Note1 (explains test result evaluation logic)

**Node Details:**

- **Test Result Evaluation**  
  - Type: If node (Condition check)  
  - Role: Checks if `error` field exists in output JSON.  
  - Configuration:  
    - Condition: `error` field NOT EXISTS → success branch  
    - `error` field EXISTS → failure branch  
  - Inputs: Output of "Execute Target Workflow Under Test" node.  
  - Outputs: Two branches:  
    - True (no error) → "Format Success Test Result"  
    - False (error found) → "Format Failed Test Result" and "Log Error Details to Error Tracking Sheet"  
  - Edge Cases:  
    - If output JSON structure changes and error field is missing or differently named, evaluation may fail.  
    - Loose type validation used to avoid strict type conflicts.

- **Sticky Note1**  
  - Details the decision process and routing for pass/fail test results.

---

#### 1.4 Success Handling

**Overview:**  
Formats success test results, generates a text report, converts it to a file, archives it on Google Drive, and updates a ClickUp task to reflect success status.

**Nodes Involved:**  
- Format Success Test Result  
- Generate Success Report Text  
- Convert Success Report to Text File  
- Archive Success Report to Google Drive  
- Update ClickUp Task with Success Status  
- Sticky Notes (fcce2d0a, 90db750f, 18671bfd, 0df832de, 5622acb4)

**Node Details:**

- **Format Success Test Result**  
  - Type: Set node  
  - Role: Creates a structured JSON object for success, including status, workflow name, and timestamp.  
  - Configuration:  
    - status: "✅ Passed"  
    - tested_workflow: "Retention Tracking Post-Hire" (hardcoded)  
    - timestamp: current execution time (`{{$now}}`)  
  - Input: True output of "Test Result Evaluation" node.  
  - Output: Passes formatted data to "Generate Success Report Text".

- **Generate Success Report Text**  
  - Type: Code node (JavaScript)  
  - Role: Generates human-readable string report concatenating workflow name, status, and timestamp.  
  - Configuration: Simple template string using JSON fields from previous node.  
  - Input: Output from "Format Success Test Result".  
  - Output: Passes text string to "Convert Success Report to Text File".  
  - Edge Cases: If input JSON fields missing, output text may be incomplete.

- **Convert Success Report to Text File**  
  - Type: ConvertToFile node  
  - Role: Converts the text report string into a downloadable text file object.  
  - Configuration:  
    - Operation: toText  
    - Source property: `text` (from previous code node)  
  - Input: Text report JSON.  
  - Output: Passes file binary data to Google Drive upload node.

- **Archive Success Report to Google Drive**  
  - Type: Google Drive node  
  - Role: Uploads the success report text file to a Google Drive folder ("resume store").  
  - Configuration:  
    - File name dynamically built from tested workflow, status, and sanitized timestamp (colons and dots replaced by dashes).  
    - Drive: "My Drive"  
    - Folder ID: "16lOVXsq0xkvJ8sCM7hCFAghQvDOXann7" (resume store folder)  
    - Credentials: "Techdome Account" Google Drive OAuth2  
  - Input: File from Convert node.  
  - Output: Passes metadata to ClickUp update node.

- **Update ClickUp Task with Success Status**  
  - Type: ClickUp node  
  - Role: Updates a specific ClickUp task with the success report text content for team visibility.  
  - Configuration:  
    - Task ID: "86b700vbb" (configured)  
    - Operation: Update  
    - Content: Uses expression to set content to the text generated by "Generate Success Report Text" node.  
    - Credentials: ClickUp API credentials named "ClickUp account 3"  
  - Input: Output from Google Drive node.  
  - Output: End of success branch.

- **Sticky Notes** (several covering this block)  
  - Describe formatting, conversion, archiving, and ClickUp integration for success reporting.

- **Edge Cases:**  
  - Google Drive upload failure (auth or quota issues) can break archive step.  
  - ClickUp API failure or incorrect task ID may prevent update.  
  - Timestamp format sanitization critical to avoid file naming errors.  
  - Hardcoded tested workflow name may require update for different target workflows.

---

#### 1.5 Failure Handling

**Overview:**  
Formats failure test results, generates a text report, converts it to a file, archives it on Google Drive, updates ClickUp with failure status, and logs error details to Google Sheets for error tracking.

**Nodes Involved:**  
- Format Failed Test Result  
- Generate Failure Report Text  
- Convert Failure Report to Text File  
- Archive Failure Report to Google Drive  
- Update ClickUp Task with Failure Status  
- Log Error Details to Error Tracking Sheet  
- Sticky Notes (48482dbc, 54f6ba7c, 8e5de5e8, 2b1741e8, dcde5325-c1a3-4e33-acf9-661587287d9f, 5dff41d1-8ca3-49e2-a6cd-f181bb402cb0)

**Node Details:**

- **Format Failed Test Result**  
  - Type: Set node  
  - Role: Creates a structured JSON object for failure, including status, workflow name, and timestamp.  
  - Configuration:  
    - status: "❌ Failed"  
    - tested_workflow: "Retention Tracking Post-Hire" (hardcoded)  
    - timestamp: current execution time (`{{$now}}`)  
  - Input: False output of "Test Result Evaluation" node.  
  - Output: Passes data to "Generate Failure Report Text".

- **Generate Failure Report Text**  
  - Type: Code node (JavaScript)  
  - Role: Generates human-readable failure report string similar to success but indicating failure.  
  - Configuration: Template string using JSON fields.  
  - Input: Output from "Format Failed Test Result".  
  - Output: Passes to "Convert Failure Report to Text File".

- **Convert Failure Report to Text File**  
  - Type: ConvertToFile node  
  - Role: Converts failure report text to a file object.  
  - Configuration: Same as success conversion but for failure text.  
  - Input: Text from code node.  
  - Output: Passes file to Google Drive archive node.

- **Archive Failure Report to Google Drive**  
  - Type: Google Drive node  
  - Role: Uploads failure report file to the same Google Drive "resume store" folder.  
  - Configuration: Same dynamic file naming and folder as success archive node.  
  - Credentials: Same Google Drive OAuth2.  
  - Input: File from convert node.  
  - Output: Passes metadata to ClickUp update node.

- **Update ClickUp Task with Failure Status**  
  - Type: ClickUp node  
  - Role: Updates same ClickUp task with failure report text content.  
  - Configuration:  
    - Task ID: "86b700vbb"  
    - Content: Text from "Generate Failure Report Text" node.  
    - Credentials: Same ClickUp API.  
  - Input: Output from Google Drive failure archive node.

- **Log Error Details to Error Tracking Sheet**  
  - Type: Google Sheets node  
  - Role: Appends error details to a dedicated error log sheet for analysis and audit.  
  - Configuration:  
    - Spreadsheet ID: "1Uldk_4BxWbdZTDZxFUeohIfeBmGHHqVEl9Ogb0l6R8Y" (Interviewer Brief Pack)  
    - Sheet name: "error log sheet" (GID 1338537721)  
    - Operation: Append or update with error JSON string from output field `error`  
    - Credentials: Google Sheets OAuth2 for "automations@techdome.ai"  
  - Input: Error JSON from sub-workflow output (passed from "Test Result Evaluation").  
  - Output: None (end of failure branch).

- **Sticky Notes**  
  - Describe formatting, conversion, archiving, ClickUp update, and error logging for failure cases.

- **Edge Cases:**  
  - Google Sheets API limits or auth failure may prevent error log update.  
  - Google Drive upload or ClickUp update failures might cause incomplete reporting.  
  - Accurate error JSON capture critical for meaningful logs.  
  - Hardcoded workflow name requires update if testing different workflows.

---

### 3. Summary Table

| Node Name                          | Node Type            | Functional Role                              | Input Node(s)                       | Output Node(s)                                    | Sticky Note                                                                                                             |
|-----------------------------------|----------------------|----------------------------------------------|-----------------------------------|--------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’  | Manual Trigger       | Start workflow manually                       | None                              | Execute Target Workflow Under Test                |                                                                                                                         |
| Execute Target Workflow Under Test| Execute Workflow     | Run target sub-workflow and capture output   | When clicking ‘Execute workflow’  | Test Result Evaluation                            | 🔄 Execute Target Workflow Under Test: calls "Archive Payment Receipts" sub-workflow, captures output including errors.   |
| Test Result Evaluation            | If                   | Decide success or failure based on error field| Execute Target Workflow Under Test| Format Success Test Result / Format Failed Test Result | ✅ Test Result Evaluation (Pass/Fail Check): routes to success or failure branches based on error presence.               |
| Format Success Test Result         | Set                  | Format success result object                   | Test Result Evaluation (true)     | Generate Success Report Text                      | 🎯 Format Success Test Result: prepares standardized success data.                                                       |
| Generate Success Report Text       | Code                 | Create human-readable success report text     | Format Success Test Result        | Convert Success Report to Text File               | 📄 Generate Success Report Text: concatenates workflow name, status, timestamp.                                          |
| Convert Success Report to Text File| ConvertToFile        | Convert success report text to file            | Generate Success Report Text      | Archive Success Report to Google Drive            | 📦 Convert Success Report to Text File: prepares file for upload.                                                       |
| Archive Success Report to Google Drive| Google Drive      | Upload success report file to Drive            | Convert Success Report to Text File| Update ClickUp Task with Success Status           | ☁️ Archive Success Report to Google Drive: stores report in "resume store" folder.                                      |
| Update ClickUp Task with Success Status| ClickUp           | Update ClickUp task with success report        | Archive Success Report to Google Drive| End                                              | ✏️ Update ClickUp Task with Success Status: posts success test result to task.                                          |
| Format Failed Test Result          | Set                  | Format failure result object                    | Test Result Evaluation (false)    | Generate Failure Report Text                      | ❌ Format Failed Test Result: prepares standardized failure data.                                                       |
| Generate Failure Report Text       | Code                 | Create human-readable failure report text     | Format Failed Test Result         | Convert Failure Report to Text File                | 📄 Generate Failure Report Text: concatenates workflow name, fail status, timestamp.                                     |
| Convert Failure Report to Text File| ConvertToFile        | Convert failure report text to file             | Generate Failure Report Text      | Archive Failure Report to Google Drive             | 📦 Convert Failure Report to Text File: prepares file for upload.                                                       |
| Archive Failure Report to Google Drive| Google Drive      | Upload failure report file to Drive             | Convert Failure Report to Text File| Update ClickUp Task with Failure Status            | ☁️ Archive Failure Report to Google Drive: stores report in "resume store" folder.                                      |
| Update ClickUp Task with Failure Status| ClickUp           | Update ClickUp task with failure report         | Archive Failure Report to Google Drive| End                                              | ✏️ Update ClickUp Task with Failure Status: posts failure test result to task.                                          |
| Log Error Details to Error Tracking Sheet| Google Sheets    | Append error details to error log sheet         | Test Result Evaluation (false)    | None                                              | 📊 Log Error Details to Error Tracking Sheet: captures raw error info in Google Sheets for analysis.                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: "When clicking ‘Execute workflow’"  
   - Type: Manual Trigger  
   - No parameters needed.

2. **Create Execute Workflow Node for Sub-Workflow**  
   - Name: "Execute Target Workflow Under Test"  
   - Type: Execute Workflow  
   - Set Workflow ID to the target test workflow ("Archive Payment Receipts...")  
   - Enable "Wait for Sub-Workflow" to true  
   - Set "On Error" to "Continue Regular Output"  
   - No input parameters.  
   - Connect input from manual trigger.

3. **Create If Node for Result Evaluation**  
   - Name: "Test Result Evaluation"  
   - Type: If  
   - Condition: `error` field does NOT exist (`string operation: notExists`) in JSON output  
   - Connect input from Execute Workflow node.

4. **Create Success Branch Nodes:**  
   a. Set node:  
      - Name: "Format Success Test Result"  
      - Assign fields:  
        - status = "✅ Passed"  
        - tested_workflow = "Retention Tracking Post-Hire" (update as needed)  
        - timestamp = `{{$now}}`  
      - Input from If node (true branch).  
   
   b. Code node:  
      - Name: "Generate Success Report Text"  
      - JS code:  
        ```js
        return [{
          json: {
            text: `Workflow: ${$json.tested_workflow}\nStatus: ${$json.status}\nTimestamp: ${$json.timestamp}`
          }
        }];
        ```  
      - Input from Set node.  
   
   c. ConvertToFile node:  
      - Name: "Convert Success Report to Text File"  
      - Operation: toText  
      - Source Property: `text`  
      - Input from Code node.  
   
   d. Google Drive node:  
      - Name: "Archive Success Report to Google Drive"  
      - Operation: Upload file  
      - File Name: `={{ $json.tested_workflow }}_{{ $json.status }}_{{ $json.timestamp.replace(/[:.]/g, "-") }}.txt`  
      - Drive: "My Drive"  
      - Folder ID: Folder for "resume store" (use your folder ID)  
      - Credentials: Google Drive OAuth2 with appropriate access  
      - Input from ConvertToFile node.  
   
   e. ClickUp node:  
      - Name: "Update ClickUp Task with Success Status"  
      - Operation: Update Task  
      - Task ID: Set to your target task ID (e.g., "86b700vbb")  
      - Content: `={{ $('Generate Success Report Text').item.json.text }}`  
      - Credentials: ClickUp API credentials  
      - Input from Google Drive node.

5. **Create Failure Branch Nodes:**  
   a. Set node:  
      - Name: "Format Failed Test Result"  
      - Assign fields:  
        - status = "❌ Failed"  
        - tested_workflow = "Retention Tracking Post-Hire"  
        - timestamp = `{{$now}}`  
      - Input from If node (false branch).  
   
   b. Code node:  
      - Name: "Generate Failure Report Text"  
      - JS code: same as success but with failure fields.  
   
   c. ConvertToFile node:  
      - Name: "Convert Failure Report to Text File"  
      - Operation: toText  
      - Source Property: `text`  
   
   d. Google Drive node:  
      - Name: "Archive Failure Report to Google Drive"  
      - Same configuration as success archive node but for failure branch.  
   
   e. ClickUp node:  
      - Name: "Update ClickUp Task with Failure Status"  
      - Same as success update but content from failure report text.  
   
   f. Google Sheets node:  
      - Name: "Log Error Details to Error Tracking Sheet"  
      - Operation: Append or Update  
      - Spreadsheet ID: your sheet ID for error logging  
      - Sheet Name: your error log tab name  
      - Map column 'error' to `{{$json.error}}`  
      - Credentials: Google Sheets OAuth2 with write access  
      - Input from If node (false branch) — connect in parallel with Set node for failure.

6. **Connect Nodes:**  
   - Manual Trigger → Execute Workflow  
   - Execute Workflow → If Node (Test Result Evaluation)  
   - If Node true output → Success chain (Set → Code → ConvertToFile → Google Drive → ClickUp)  
   - If Node false output → Failure chain (Set → Code → ConvertToFile → Google Drive → ClickUp) and also to Google Sheets for error logging.

7. **Credential Setup:**  
   - Google Drive OAuth2 with access to target Drive and folder.  
   - ClickUp API credentials with permissions to update target task.  
   - Google Sheets OAuth2 with write access to the error logging spreadsheet.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                                                               |
|-------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Workflow integrates Google Drive, Google Sheets, and ClickUp for comprehensive test reporting.  | Core platforms used for archival, error logging, and project management integration.                         |
| Tested workflow name is hardcoded as "Retention Tracking Post-Hire" — update if needed.          | Be sure to synchronize tested workflow name with actual workflow executed for meaningful reports.            |
| Folder ID "16lOVXsq0xkvJ8sCM7hCFAghQvDOXann7" corresponds to "resume store" in Google Drive.   | Change folder ID if deploying in a different Google Drive environment.                                       |
| ClickUp task ID "86b700vbb" must be replaced with your project-specific task ID.                | Ensures updates go to the correct task in ClickUp for visibility.                                           |
| Timestamp strings are sanitized by replacing ":" and "." with "-" in file names for Drive upload| Prevents invalid characters in filenames on Google Drive.                                                   |
| Error logging sheet URL: https://docs.google.com/spreadsheets/d/1Uldk_4BxWbdZTDZxFUeohIfeBmGHHqVEl9Ogb0l6R8Y | Used for detailed error analysis and historical tracking.                                                    |

---

**Disclaimer:**  
This documentation is generated from an n8n workflow automation and complies strictly with content policies. All handled data is legal and public.