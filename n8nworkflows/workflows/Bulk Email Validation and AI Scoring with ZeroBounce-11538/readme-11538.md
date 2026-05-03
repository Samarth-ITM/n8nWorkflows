Bulk Email Validation and AI Scoring with ZeroBounce

https://n8nworkflows.xyz/workflows/bulk-email-validation-and-ai-scoring-with-zerobounce-11538


# Bulk Email Validation and AI Scoring with ZeroBounce

### 1. Workflow Overview

This workflow performs bulk email validation and AI-based scoring using the ZeroBounce service. It targets use cases where users need to verify large lists of email addresses for deliverability and quality before sending marketing or transactional emails. The workflow is structured into three logical blocks:

- **1.1 Input Preparation:** Receives or defines bulk email addresses for validation. It uses sandbox test emails here for demonstration but can accept real input data.
- **1.2 ZeroBounce Bulk Email Validation:** Sends the bulk email list to ZeroBounce, waits for processing, retrieves validation results, and branches emails by validity status.
- **1.3 ZeroBounce AI Scoring and Filtering:** For emails validated as "valid," sends them to ZeroBounce AI Scoring API, waits for scoring results, merges these with validation data, and finally categorizes email addresses into confidence tiers based on their AI scores.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Preparation

**Overview:**  
Prepares input data in the form of a list of email addresses. This example uses ZeroBounce sandbox emails which do not consume API credits and are intended for testing the workflow. In production, this block would be replaced or extended to accept real input data via triggers or file uploads.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Sandbox emails (Set)  
- Sticky Note (Input data explanation)

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Starts the workflow upon manual execution.  
  - Config: No parameters.  
  - Connections: Outputs to "Sandbox emails".  
  - Failure Modes: None expected, but workflow won’t proceed unless triggered.  

- **Sandbox emails**  
  - Type: Set  
  - Role: Defines an array of sandbox email addresses in JSON format.  
  - Config: Raw JSON with a property `emails` containing multiple test email addresses.  
  - Key Expressions: The emails array is referenced downstream for file sending.  
  - Connections: Outputs to "Send file for validation".  
  - Failure Modes: None if JSON is valid.  

- **Sticky Note (Input data)**  
  - Type: Sticky Note  
  - Role: Documentation for users describing sandbox emails usage and input source.  
  - Content: Explains sandbox mode usage and typical production input scenario.  

---

#### 2.2 ZeroBounce Bulk Email Validation

**Overview:**  
This block sends the batch of emails to ZeroBounce's Bulk Email Validator API, waits for the processing to complete via webhook or timeout, checks the file status, retrieves the results, and filters the email addresses by their validation status.

**Nodes Involved:**  
- Send file for validation (ZeroBounce node)  
- Wait for validation file to be processed (Wait node with webhook resume)  
- Get validation file status (ZeroBounce node)  
- Validation file status (Switch)  
- Get validation results file (ZeroBounce node)  
- Is valid? (Switch)  
- Not valid (NoOp)  
- Validation results (NoOp)  
- Bulk validation failed (Stop and Error)  
- Sticky Note (ZeroBounce Validation explanation)

**Node Details:**

- **Send file for validation**  
  - Type: ZeroBounce node with operation "sendFile"  
  - Role: Uploads bulk email list for validation.  
  - Config: Sends emails from the "Sandbox emails" node; sends array of emails; sets returnUrl for webhook callback to resume workflow.  
  - Credentials: ZeroBounce API key required.  
  - Connections: Outputs to "Wait for validation file to be processed".  
  - Failure Modes: API errors, invalid API key, network issues.  

- **Wait for validation file to be processed**  
  - Type: Wait node with webhook resume  
  - Role: Pauses workflow and waits up to 5 minutes for ZeroBounce webhook callback signaling processing completion.  
  - Config: Webhook suffix "zb-validation", HTTP POST method, resume with webhook or timeout.  
  - Connections: Outputs to "Get validation file status".  
  - Failure Modes: Webhook not received, timeout triggers status check.  

- **Get validation file status**  
  - Type: ZeroBounce node, operation "fileStatus"  
  - Role: Queries status of the validation file processing using file_id from "Send file for validation".  
  - Config: Uses file_id from previous node output.  
  - Connections: Outputs to "Validation file status" switch.  
  - Failure Modes: API errors, invalid file_id.  

- **Validation file status**  
  - Type: Switch  
  - Role: Routes workflow based on file_status: "Complete", "Processing", or "Failed".  
  - Config: Checks string equality on file_status field.  
  - Connections:  
    - "Complete" → "Get validation results file"  
    - "Processing" → loops back to "Wait for validation file to be processed"  
    - "Failed" → "Bulk validation failed" (stop with error)  
  - Failure Modes: Unexpected status values, processing delays.  

- **Get validation results file**  
  - Type: ZeroBounce node, operation "getFile"  
  - Role: Retrieves processed validation results with detailed fields.  
  - Config: Uses file_id from status node.  
  - Connections: Outputs to "Is valid?" switch.  
  - Failure Modes: API errors, file not found.  

- **Is valid?**  
  - Type: Switch  
  - Role: Checks each email's validation status from ZeroBounce JSON field "ZB Status".  
  - Config: Routes "valid" status to next scoring step; all other statuses to "Not valid".  
  - Connections:  
    - "valid" → "Send file for scoring" and "Validation results" (NoOp)  
    - "other" → "Not valid" (NoOp)  
  - Failure Modes: Missing or unexpected status values.  

- **Not valid**  
  - Type: NoOp  
  - Role: Placeholder for emails that failed validation.  
  - Connections: None further.  

- **Validation results**  
  - Type: NoOp  
  - Role: Placeholder node to continue with valid emails.  
  - Connections: Outputs to "Merge validation and scoring results" (merge input 2).  

- **Bulk validation failed**  
  - Type: Stop and Error  
  - Role: Terminates workflow with error message if bulk validation fails.  

- **Sticky Note (ZeroBounce Validation)**  
  - Type: Sticky Note  
  - Role: Describes the bulk validation process, including wait logic and input file creation.  

---

#### 2.3 ZeroBounce AI Scoring and Filtering

**Overview:**  
For emails validated as "valid," this block sends them to ZeroBounce's Bulk AI Scoring API, waits for scoring completion, retrieves scores, merges with validation results, and finally filters emails by confidence tier: high, medium, or low score.

**Nodes Involved:**  
- Send file for scoring (ZeroBounce node)  
- Wait for scoring file to process (Wait node with webhook resume)  
- Get scoring file status (ZeroBounce node)  
- Scoring file status (Switch)  
- Get scoring results file (ZeroBounce node)  
- Merge validation and scoring results (Merge node)  
- Filter by score (Switch)  
- High score (NoOp)  
- Medium score (NoOp)  
- Low score (NoOp)  
- Bulk scoring failed (Stop and Error)  
- Sticky Note (ZeroBounce AI Scoring explanation)  
- Sticky Note (Filtered outputs explanation)  
- Sticky Note (Split results by score explanation)  

**Node Details:**

- **Send file for scoring**  
  - Type: ZeroBounce node, resource "scoring", operation "scoringSendFile"  
  - Role: Sends validated emails for AI scoring.  
  - Config: Maps input emails, includes file in output, sets returnUrl webhook for resume.  
  - Credentials: ZeroBounce API key required.  
  - Connections: Outputs to "Wait for scoring file to process".  
  - Failure Modes: API errors, invalid mapping, network issues.  

- **Wait for scoring file to process**  
  - Type: Wait node with webhook resume  
  - Role: Waits up to 5 minutes for scoring processing webhook callback.  
  - Config: Webhook suffix "zb-scoring", HTTP POST method, resume with webhook or timeout.  
  - Connections: Outputs to "Get scoring file status".  
  - Failure Modes: Webhook not received, timeout triggers status check.  

- **Get scoring file status**  
  - Type: ZeroBounce node, resource "scoring", operation "scoringFileStatus"  
  - Role: Checks scoring file processing status using file_id from "Send file for scoring".  
  - Connections: Outputs to "Scoring file status" switch.  
  - Failure Modes: API errors, invalid file_id.  

- **Scoring file status**  
  - Type: Switch  
  - Role: Routes workflow based on scoring file_status: "Complete", "Processing", or "Failed".  
  - Connections:  
    - "Complete" → "Get scoring results file"  
    - "Processing" → loops back to "Wait for scoring file to process"  
    - "Failed" → "Bulk scoring failed" (stop with error)  
  - Failure Modes: Unexpected status values.  

- **Get scoring results file**  
  - Type: ZeroBounce node, resource "scoring", operation "scoringGetFile"  
  - Role: Retrieves scoring results with detailed fields.  
  - Connections: Outputs to "Merge validation and scoring results" (merge input 1).  
  - Failure Modes: API errors, file missing.  

- **Merge validation and scoring results**  
  - Type: Merge  
  - Role: Enriches validation data with scoring data, matching on "email_address" field.  
  - Config: Mode "combine", join mode "enrichInput2", field to match "email_address".  
  - Connections: Outputs to "Filter by score".  
  - Failure Modes: Missing matching keys, mismatch in email formats.  

- **Filter by score**  
  - Type: Switch  
  - Role: Categorizes emails into confidence tiers based on "ZeroBounceQualityScore" numeric field.  
  - Config:  
    - "high": score >= 9  
    - "medium": score >= 3 and < 9  
    - "low": score < 3  
  - Connections:  
    - "high" → "High score" (NoOp)  
    - "medium" → "Medium score" (NoOp)  
    - "low" → "Low score" (NoOp)  
  - Failure Modes: Missing or invalid score values, type conversion errors.  

- **High score, Medium score, Low score**  
  - Type: NoOp  
  - Role: Placeholders for further processing or export of filtered email groups.  

- **Bulk scoring failed**  
  - Type: Stop and Error  
  - Role: Terminates workflow with error message if scoring fails.  

- **Sticky Notes**  
  - Several sticky notes document the AI scoring process, filtering rationale, and possible next steps with filtered outputs.  

---

### 3. Summary Table

| Node Name                         | Node Type                             | Functional Role                             | Input Node(s)                            | Output Node(s)                            | Sticky Note                                                  |
|----------------------------------|-------------------------------------|---------------------------------------------|-----------------------------------------|-------------------------------------------|--------------------------------------------------------------|
| When clicking ‘Execute workflow’  | Manual Trigger                      | Initiates workflow manually                  | -                                       | Sandbox emails                            |                                                              |
| Sandbox emails                   | Set                                 | Defines test email list                       | When clicking ‘Execute workflow’         | Send file for validation                  | See note on Sandbox emails usage and input data              |
| Send file for validation          | ZeroBounce                          | Sends bulk emails for validation             | Sandbox emails                          | Wait for validation file to be processed  | ZeroBounce Validation explanation note                        |
| Wait for validation file to be processed | Wait (with webhook resume)    | Waits for ZeroBounce validation webhook      | Send file for validation                | Get validation file status                 | ZeroBounce Validation explanation note                        |
| Get validation file status        | ZeroBounce                          | Checks validation file processing status     | Wait for validation file to be processed | Validation file status                     | ZeroBounce Validation explanation note                        |
| Validation file status            | Switch                             | Routes based on validation file status       | Get validation file status               | Get validation results file / Wait / Error | ZeroBounce Validation explanation note                        |
| Get validation results file       | ZeroBounce                          | Downloads validation results                  | Validation file status                   | Is valid?                                 | ZeroBounce Validation explanation note                        |
| Is valid?                       | Switch                             | Splits emails by "valid" status               | Get validation results file              | Send file for scoring / Not valid         | ZeroBounce Validation explanation note                        |
| Not valid                      | NoOp                               | Handles invalid emails                         | Is valid? (other)                        | -                                         | ZeroBounce Validation explanation note                        |
| Validation results                | NoOp                               | Passes valid emails forward                    | Is valid? (valid)                       | Merge validation and scoring results      | ZeroBounce Validation explanation note                        |
| Send file for scoring             | ZeroBounce                          | Sends valid emails for AI scoring              | Is valid? (valid)                       | Wait for scoring file to process           | ZeroBounce AI Scoring explanation note                        |
| Wait for scoring file to process  | Wait (with webhook resume)          | Waits for ZeroBounce scoring webhook          | Send file for scoring                   | Get scoring file status                    | ZeroBounce AI Scoring explanation note                        |
| Get scoring file status           | ZeroBounce                          | Checks scoring file processing status          | Wait for scoring file to process        | Scoring file status                        | ZeroBounce AI Scoring explanation note                        |
| Scoring file status              | Switch                             | Routes based on scoring file status            | Get scoring file status                  | Get scoring results file / Wait / Error   | ZeroBounce AI Scoring explanation note                        |
| Get scoring results file          | ZeroBounce                          | Downloads scoring results                       | Scoring file status                     | Merge validation and scoring results      | ZeroBounce AI Scoring explanation note                        |
| Merge validation and scoring results | Merge                           | Combines validation and scoring data          | Validation results, Get scoring results file | Filter by score                          | ZeroBounce AI Scoring explanation note                        |
| Filter by score                  | Switch                             | Categorizes emails by AI quality score         | Merge validation and scoring results    | High score / Medium score / Low score     | Filtered outputs and Split results by score notes            |
| High score                      | NoOp                               | Placeholder for high confidence emails         | Filter by score                         | -                                         | Filtered outputs and Split results by score notes            |
| Medium score                    | NoOp                               | Placeholder for medium confidence emails       | Filter by score                         | -                                         | Filtered outputs and Split results by score notes            |
| Low score                      | NoOp                               | Placeholder for low confidence emails          | Filter by score                         | -                                         | Filtered outputs and Split results by score notes            |
| Bulk validation failed          | Stop and Error                    | Stops workflow on validation failure           | Validation file status (failed)         | -                                         | ZeroBounce Validation explanation note                        |
| Bulk scoring failed             | Stop and Error                    | Stops workflow on scoring failure              | Scoring file status (failed)             | -                                         | ZeroBounce AI Scoring explanation note                        |
| Sticky Note                     | Sticky Note                      | Documentation notes                             | -                                       | -                                         | Multiple notes explaining each major block and process steps |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node** named `When clicking ‘Execute workflow’` to start the workflow manually.

2. **Create a Set node** named `Sandbox emails`:
   - Set mode to "Raw".
   - Define JSON output with an `emails` array containing sample sandbox email addresses as given in the workflow.
   - Connect `When clicking ‘Execute workflow’` → `Sandbox emails`.

3. **Add ZeroBounce node** named `Send file for validation`:
   - Operation: `sendFile`.
   - Input type: `items`.
   - Map `emails` field from `Sandbox emails` node JSON path: `{{$node["Sandbox emails"].json["emails"]}}`.
   - Set returnUrl to `{{$execution.resumeUrl}}/zb-validation`.
   - Configure ZeroBounce API credentials.
   - Connect `Sandbox emails` → `Send file for validation`.

4. **Add Wait node** named `Wait for validation file to be processed`:
   - Set to resume on webhook POST with suffix `zb-validation`.
   - Timeout after 5 minutes.
   - Connect `Send file for validation` → `Wait for validation file to be processed`.

5. **Add ZeroBounce node** named `Get validation file status`:
   - Operation: `fileStatus`.
   - Use `file_id` from `Send file for validation` output.
   - Connect `Wait for validation file to be processed` → `Get validation file status`.

6. **Add Switch node** named `Validation file status`:
   - Rules for `file_status` field:
     - `Complete`: continue.
     - `Processing`: loop back to waiting node.
     - `Failed`: stop with error.
   - Connect `Get validation file status` → `Validation file status`.

7. **Add ZeroBounce node** named `Get validation results file`:
   - Operation: `getFile`.
   - Use `file_id` from previous steps.
   - Include file content in fields.
   - Connect `Validation file status` (Complete) → `Get validation results file`.

8. **Add Switch node** named `Is valid?`:
   - Check field `ZB Status`.
   - Route `"valid"` emails to next step.
   - Other statuses to a NoOp node `Not valid`.
   - Connect `Get validation results file` → `Is valid?`.

9. **Add NoOp node** named `Not valid` for invalid emails handling.

10. **Add NoOp node** named `Validation results` to hold valid emails for merging.

11. **Connect `Is valid?` (valid output) to both `Send file for scoring` and `Validation results`.
   
12. **Add ZeroBounce node** named `Send file for scoring`:
    - Resource: `scoring`.
    - Operation: `scoringSendFile`.
    - Map emails from input.
    - Set returnUrl to `{{$execution.resumeUrl}}/zb-scoring`.
    - Configure credentials.
    - Connect `Is valid?` (valid output) → `Send file for scoring`.

13. **Add Wait node** named `Wait for scoring file to process`:
    - Resume on webhook POST with suffix `zb-scoring`.
    - Timeout after 5 minutes.
    - Connect `Send file for scoring` → `Wait for scoring file to process`.

14. **Add ZeroBounce node** named `Get scoring file status`:
    - Resource: `scoring`.
    - Operation: `scoringFileStatus`.
    - Use `file_id` from `Send file for scoring`.
    - Connect `Wait for scoring file to process` → `Get scoring file status`.

15. **Add Switch node** named `Scoring file status`:
    - Similar to validation file status switch.
    - Route `Complete` to `Get scoring results file`.
    - Route `Processing` back to wait.
    - Route `Failed` to stop error.
    - Connect `Get scoring file status` → `Scoring file status`.

16. **Add ZeroBounce node** named `Get scoring results file`:
    - Resource: `scoring`.
    - Operation: `scoringGetFile`.
    - Use `file_id` from scoring file status.
    - Include file content in fields.
    - Connect `Scoring file status` (Complete) → `Get scoring results file`.

17. **Add Merge node** named `Merge validation and scoring results`:
    - Mode: Combine.
    - Join Mode: Enrich Input 2.
    - Field to match: `email_address`.
    - Connect `Validation results` → Merge (input 2).
    - Connect `Get scoring results file` → Merge (input 1).

18. **Add Switch node** named `Filter by score`:
    - Field: `ZeroBounceQualityScore`.
    - Rules:
      - High: score >= 9.
      - Medium: score >= 3 and < 9.
      - Low: score < 3.
    - Connect `Merge validation and scoring results` → `Filter by score`.

19. **Add three NoOp nodes** named `High score`, `Medium score`, and `Low score` for downstream processing of filtered lists.

20. **Add Stop and Error nodes** named `Bulk validation failed` and `Bulk scoring failed` for error handling on respective failures.

21. **Add Sticky Notes** at relevant points to document workflow blocks and logic for user clarity (optional but recommended).

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                              | Context or Link                                                                                                  |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| ZeroBounce Bulk Email Validator API documentation and quickstart guide.                                                                                                                                                                                                                   | https://www.zerobounce.net/docs/email-validation-api-quickstart/v2-send-file                                     |
| ZeroBounce Bulk AI Scoring API documentation.                                                                                                                                                                                                                                            | https://www.zerobounce.net/docs/ai-scoring-api/ai-scoring-api__send-file                                        |
| Sandbox email addresses for testing do not consume API credits and can be used for workflow testing and development.                                                                                                                                                                     | https://www.zerobounce.net/docs/email-validation-api-quickstart/v2-sandbox-mode                                  |
| ZeroBounce API key required to use nodes; can be generated at ZeroBounce members area.                                                                                                                                                                                                    | https://www.zerobounce.net/members/API                                                                          |
| Filtering emails by ZeroBounce “ZB Status” and “ZeroBounceQualityScore” allows segmentation into valid/invalid and confidence tiers, improving campaign targeting and reducing bounce rates.                                                                                               | Workflow logic explanation                                                                                        |
| ZeroBounce n8n node source and logo available on GitHub for branding and integration details.                                                                                                                                                                                             | https://github.com/zerobounce/n8n-nodes-zerobounce                                                              |
| Examples of next steps with filtered results include using ZeroBounce Email Finder or Domain Search APIs integrated with n8n.                                                                                                                                                              | https://www.zerobounce.net/docs/email-finder-api and https://www.zerobounce.net/docs/domain-search-api           |

---

**Disclaimer:** The text provided is exclusively based on an n8n automated workflow using ZeroBounce API integration. The workflow complies fully with content policies and does not include any illegal or offensive elements. All data handled are legal and publicly available.