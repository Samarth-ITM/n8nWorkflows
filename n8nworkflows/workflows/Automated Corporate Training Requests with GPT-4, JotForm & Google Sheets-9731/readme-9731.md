Automated Corporate Training Requests with GPT-4, JotForm & Google Sheets

https://n8nworkflows.xyz/workflows/automated-corporate-training-requests-with-gpt-4--jotform---google-sheets-9731


# Automated Corporate Training Requests with GPT-4, JotForm & Google Sheets

### 1. Workflow Overview

This workflow automates the processing of corporate training requests submitted via JotForm. It normalizes input data, checks budget availability per department, leverages GPT-4 AI analysis to evaluate training needs, recommends suitable courses, and routes approval requests to managers when necessary. It concludes by notifying relevant parties and logging the request details into Google Sheets.

The workflow is organized into the following logical blocks:

- **1.1 Input Reception & Normalization:** Captures and standardizes training request data from JotForm submissions.
- **1.2 Budget Validation:** Checks department-specific training budgets against requested training costs.
- **1.3 AI Training Needs Analysis:** Uses GPT-4 via Langchain to analyze training requests, recommend courses, and assess ROI and risks.
- **1.4 AI Output Extraction & Data Merging:** Parses AI JSON output and merges it with original data and budget info.
- **1.5 Approval Routing:** Determines if manager approval is needed and routes accordingly.
- **1.6 Notification Dispatch:** Sends emails to managers for approval, employees for confirmation, or rejection notices.
- **1.7 Logging:** Records training requests and AI insights in Google Sheets for future analytics.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Normalization

- **Overview:** Receives training request submissions via JotForm webhook and normalizes the data to a consistent format with default values and fallbacks.
- **Nodes Involved:**  
  - Jotform Trigger  
  - Parse Training Request (Code node)  
  - Sticky Note (for documentation)

- **Node Details:**

  - **Jotform Trigger**  
    - Type: Trigger node, listens to JotForm form submissions.  
    - Configured with Form ID linked to the corporate training request form.  
    - Outputs raw form submission data.  
    - Potential failures: webhook misconfiguration, authorization errors, form ID changes.

  - **Parse Training Request**  
    - Type: Code node (JavaScript).  
    - Function: Extracts, normalizes, and enriches form data into structured fields (e.g., employeeName, trainingTopic, urgency) with fallbacks and defaults.  
    - Uses expressions like `$input.first().json` to access data.  
    - Generates a unique `requestId` if not provided.  
    - Outputs a normalized JSON object representing the training request.  
    - Edge cases: missing fields, malformed inputs, type conversions for numeric fields (attendees, budget).  
    - No version-specific requirements.

  - **Sticky Note:**  
    - Content explains this block captures training requests via JotForm with skill gaps and business justification.  
    - Provides a link to JotForm for form creation.

#### 1.2 Budget Validation

- **Overview:** Checks the training request budget against department-specific annual training budgets and training catalog costs.
- **Nodes Involved:**  
  - Check Training Budget (Code node)  
  - Sticky Note

- **Node Details:**

  - **Check Training Budget**  
    - Type: Code node (JavaScript).  
    - Contains hardcoded department budgets and training catalog data with course details.  
    - Calculates if requested budget fits within remaining departmental budget.  
    - Outputs enriched data including budget totals, spent, remaining, budget availability boolean, and utilization percentage.  
    - Edge cases: department not found in budgets defaults to a fallback budget; estimatedBudget missing defaults to 1000; type coercion errors.  
    - No external API calls, so no network errors expected.

  - **Sticky Note:**  
    - Describes this block as retrieving department training budget and checking availability.

#### 1.3 AI Training Needs Analysis

- **Overview:** Uses GPT-4 via Langchain's agent node to analyze the training request, provide a detailed needs analysis, recommend courses, assess ROI and risks, and generate an approval recommendation in structured JSON format.
- **Nodes Involved:**  
  - AI Training Analysis (Langchain agent node)  
  - OpenAI Chat Model (LM Chat OpenAI - used as language model for Langchain)  
  - Sticky Note

- **Node Details:**

  - **AI Training Analysis**  
    - Type: Langchain agent node.  
    - Configured with a detailed prompt including employee info, training request details, budget context, and training catalog data.  
    - Prompt asks for a JSON response with multiple nested objects covering needs analysis, recommended courses, alternative options, budget analysis, approval recommendation, implementation plan, skill development path, team impact, risk assessment, and manager guidance.  
    - Uses GPT-4 via the linked OpenAI Chat Model node.  
    - Outputs AI-generated JSON text string.  
    - Edge cases: API rate limits, response parsing failures, incomplete JSON output, timeout errors.

  - **OpenAI Chat Model**  
    - Type: LM Chat OpenAI node.  
    - Configured to use GPT-4.1-mini model.  
    - Credential linked to OpenAI API key.  
    - Provides AI model backend for Langchain node.

  - **Sticky Note:**  
    - Describes this block as performing AI analysis of training need, recommending courses, and assessing ROI and risks.

#### 1.4 AI Output Extraction & Data Merging

- **Overview:** Extracts and parses the AI JSON output, merges it with the original training data and budget info to produce a consolidated dataset for routing and notifications.
- **Nodes Involved:**  
  - Extract AI Analysis (Set node)  
  - Merge Training Analysis (Code node)  
  - Sticky Notes

- **Node Details:**

  - **Extract AI Analysis**  
    - Type: Set node.  
    - Assigns the raw AI JSON string to a field `aiAnalysis` for downstream use.  
    - Simple assignment operation, no complex logic.

  - **Merge Training Analysis**  
    - Type: Code node (JavaScript).  
    - Parses the AI JSON string into an object; if parse fails, uses default fallback data.  
    - Extracts top recommended course and approval decision logic.  
    - Adds numerous properties from AI analysis directly into the main JSON output for easy access (e.g., skillGapSeverity, recommendedCourses, approval flags).  
    - Determines if approval is required based on budget and AI recommendation.  
    - Edge cases: JSON parse errors handled gracefully with fallback data.

  - **Sticky Notes:**  
    - Extract AI Analysis: marks extraction of structured AI recommendations.  
    - Merge Training Analysis: marks combination of AI analysis with training data and budget info.

#### 1.5 Approval Routing

- **Overview:** Checks if the training request requires managerial approval and routes the workflow accordingly.
- **Nodes Involved:**  
  - Requires Approval? (IF node)  
  - Sticky Note

- **Node Details:**

  - **Requires Approval?**  
    - Type: IF node.  
    - Condition checks boolean field `requiresApproval` from merged data.  
    - Routes to "true" branch if approval is needed, else to "false" branch for auto-rejection.  
    - Edge cases: missing or malformed boolean field may cause incorrect routing.

  - **Sticky Note:**  
    - Explains approval routing logic: sends to manager if needed or auto-approves based on AI.

#### 1.6 Notification Dispatch

- **Overview:** Sends email notifications to managers requesting approval, employees confirming receipt, or employees notifying rejection.
- **Nodes Involved:**  
  - Send Manager Approval (Gmail node)  
  - Send Rejection Email (Gmail node)  
  - Send Employee Confirmation (Gmail node)  
  - Sticky Note

- **Node Details:**

  - **Send Manager Approval**  
    - Type: Gmail node.  
    - Sends detailed approval request email to the manager's email address.  
    - Email includes full training request details, AI analysis results, budget overview, recommended courses, risk assessment, and approval instructions.  
    - Uses dynamic expressions to populate email body and subject.  
    - Credentials: Gmail OAuth2.  
    - Potential failures: invalid credentials, email quota limits, malformed email addresses.

  - **Send Rejection Email**  
    - Type: Gmail node.  
    - Sends rejection notification to the employee if the request is not approved.  
    - Includes AI analysis summary and alternative options.  
    - Uses dynamic expressions referencing data from earlier nodes.  
    - Credentials: Gmail OAuth2.

  - **Send Employee Confirmation**  
    - Type: Gmail node.  
    - Sends confirmation to the employee upon request submission or post-approval decision.  
    - Summarizes request details and AI preliminary analysis.  
    - Credentials: Gmail OAuth2.

  - **Sticky Note:**  
    - Describes comprehensive email notifications sent to manager, employee confirmation, and rejection notices.

#### 1.7 Logging

- **Overview:** Logs the complete training request data and AI insights into a Google Sheets spreadsheet for tracking and reporting.
- **Nodes Involved:**  
  - Log to Google Sheets (Google Sheets node)  
  - Sticky Note

- **Node Details:**

  - **Log to Google Sheets**  
    - Type: Google Sheets node.  
    - Configured to append or update rows in a specific Google Sheet identified by Document ID and Sheet name (gid=0).  
    - Uses auto-mapping on input data fields to spreadsheet columns (e.g., Employee Name, Email, Position, Department, Manager Email, etc.).  
    - Credentials: Google Sheets OAuth2.  
    - Edge cases: API quota limits, permission errors, schema mismatches.

  - **Sticky Note:**  
    - Marks logging of complete training request history with AI insights for analytics.

---

### 3. Summary Table

| Node Name              | Node Type                 | Functional Role                        | Input Node(s)          | Output Node(s)            | Sticky Note                                                                                              |
|------------------------|---------------------------|-------------------------------------|------------------------|---------------------------|---------------------------------------------------------------------------------------------------------|
| Jotform Trigger        | JotForm Trigger           | Capture training request submissions | -                      | Parse Training Request     | 📩 **TRIGGER** Captures training requests via Jotform with skill gaps and business justification. Create your form for free on [Jotform using this link](https://www.jotform.com/?partner=mediajade) |
| Parse Training Request | Code                      | Normalize and structure form data   | Jotform Trigger         | Check Training Budget      | 🧾 **PARSE** Normalizes training request data                                                           |
| Check Training Budget  | Code                      | Validate budget availability         | Parse Training Request  | AI Training Analysis       | 💰 **BUDGET CHECK** Retrieves department training budget and checks availability                         |
| OpenAI Chat Model      | LM Chat OpenAI            | GPT-4 language model backend         | -                      | AI Training Analysis       |                                                                                                         |
| AI Training Analysis   | Langchain Agent           | Analyze training, recommend courses  | Check Training Budget, OpenAI Chat Model | Extract AI Analysis         | 🤖 **AI ANALYSIS** Analyzes training need, recommends courses, assesses ROI and risks                    |
| Extract AI Analysis    | Set                       | Extract raw AI JSON output            | AI Training Analysis    | Merge Training Analysis    | 🔗 **EXTRACT** Extracts structured AI recommendations                                                    |
| Merge Training Analysis| Code                      | Parse AI JSON, merge with original data | Extract AI Analysis    | Requires Approval?         | 🧩 **MERGE** Combines AI analysis with training data and budget information                              |
| Requires Approval?     | IF                        | Route by approval necessity           | Merge Training Analysis | Send Manager Approval / Send Rejection Email | ✅ **APPROVAL ROUTING** Routes to manager if needed or auto-approves based on AI recommendation          |
| Send Manager Approval  | Gmail                     | Send approval request email to manager | Requires Approval? (true) | Send Employee Confirmation | 📧 **NOTIFICATIONS** Sends comprehensive emails: • Manager (approval request) • Employee (confirmation) • Rejection (if not approved) |
| Send Rejection Email   | Gmail                     | Send rejection email to employee      | Requires Approval? (false) | Send Employee Confirmation | (Same as above)                                                                                          |
| Send Employee Confirmation | Gmail                  | Confirm receipt or notify employee    | Send Manager Approval, Send Rejection Email | Log to Google Sheets        | (Same as above)                                                                                          |
| Log to Google Sheets   | Google Sheets             | Log request and AI insights           | Send Employee Confirmation | -                         | 📊 **LOGGING** Complete training request history with AI insights for analytics and reporting           |
| Sticky Note            | Sticky Note               | Documentation                        | -                      | -                         | See above for sticky note contents                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create JotForm Training Request Form**  
   - Use [JotForm](https://www.jotform.com/?partner=mediajade) to create a form capturing employee name, email, ID, department, position, manager details, training topic, category, skill gap, skill levels, urgency, justification, budget estimate, preferred format/dates, attendees, and notes.

2. **Add Jotform Trigger Node**  
   - Type: JotForm Trigger  
   - Configure with your form’s ID (e.g., "252852702090453")  
   - Set webhook to listen to submissions.

3. **Add Code Node: Parse Training Request**  
   - Name: Parse Training Request  
   - JS Code: Normalize form data fields with fallbacks and defaults.  
   - Output fields include requestId, employee info, training details, status='pending_review'.  
   - Connect input from Jotform Trigger.

4. **Add Code Node: Check Training Budget**  
   - Name: Check Training Budget  
   - JS Code: Implement hardcoded department budgets and training catalog.  
   - Compute budget availability and utilization.  
   - Connect input from Parse Training Request.

5. **Add OpenAI Chat Model Node**  
   - Name: OpenAI Chat Model  
   - Type: LM Chat OpenAI  
   - Configure with GPT-4.1-mini model or preferred GPT-4 variant.  
   - Set OpenAI API credentials.

6. **Add Langchain Agent Node: AI Training Analysis**  
   - Connect inputs from Check Training Budget and OpenAI Chat Model nodes.  
   - Configure prompt with detailed training request info, budget data, and catalog.  
   - Specify JSON output format requesting needs analysis, recommendations, budget, approval, and implementation details.

7. **Add Set Node: Extract AI Analysis**  
   - Assign field `aiAnalysis` to raw AI output JSON string from previous node.

8. **Add Code Node: Merge Training Analysis**  
   - Parse AI JSON string safely, fallback to defaults if parsing fails.  
   - Extract top recommended course and approval logic.  
   - Add detailed fields for downstream use.  
   - Connect input from Extract AI Analysis.

9. **Add IF Node: Requires Approval?**  
   - Condition: `$json.requiresApproval === true`  
   - Connect input from Merge Training Analysis.  
   - True branch: send approval email.  
   - False branch: send rejection email.

10. **Add Gmail Node: Send Manager Approval**  
    - Configure to send to manager’s email with detailed request and AI analysis.  
    - Use OAuth2 credentials.  
    - Connect from IF node True branch.

11. **Add Gmail Node: Send Rejection Email**  
    - Configure to send rejection notice to employee with AI reasoning and alternatives.  
    - Use OAuth2 credentials.  
    - Connect from IF node False branch.

12. **Add Gmail Node: Send Employee Confirmation**  
    - Configure to confirm receipt and provide preliminary AI analysis summary.  
    - Use OAuth2 credentials.  
    - Connect from both Send Manager Approval and Send Rejection Email nodes.

13. **Add Google Sheets Node: Log to Google Sheets**  
    - Configure with your Google Sheets document ID and sheet name.  
    - Map relevant fields (employee info, request details, AI insights) for append or update operation.  
    - Use Google Sheets OAuth2 credentials.  
    - Connect input from Send Employee Confirmation.

14. **Add Sticky Notes**  
    - Add descriptive sticky notes near each logical block for documentation.

15. **Test Workflow**  
    - Submit test training requests via JotForm.  
    - Verify parsing, budget checking, AI analysis completion, approval routing, email notifications, and logging.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| JotForm form creation link for training requests: Free form builder for capturing detailed training requests including skill gaps and business justification.                                                                                                                                                                                                                                                                                                                                          | https://www.jotform.com/?partner=mediajade                                                      |
| AI training analysis prompt designed for GPT-4 via Langchain, providing structured JSON output covering training needs, budget analysis, approval recommendations, implementation plans, risk assessments, and manager guidance. This enables automated, expert-level decision-making support.                                                                                                                                                                                                           | Internal prompt in AI Training Analysis node                                                   |
| Gmail OAuth2 credentials required for sending emails; ensure OAuth2 authentication is properly configured and tokens are valid to avoid send failures.                                                                                                                                                                                                                                                                                                                                                 | Gmail OAuth2 credentials in Send Manager Approval, Send Rejection Email, and Send Employee Confirmation nodes |
| Google Sheets OAuth2 credentials must have write access to the target spreadsheet for logging training requests and AI insights.                                                                                                                                                                                                                                                                                                                                                                        | Google Sheets OAuth2 credentials in Log to Google Sheets node                                  |
| Potential failure points include API rate limits (OpenAI, Gmail, Google Sheets), malformed or incomplete form data, budget data mismatches, and JSON parse errors from AI response. Implement error handling or retry strategies as needed.                                                                                                                                                                                                                                                                | Workflow-wide considerations                                                                   |

---

**Disclaimer:** The text provided comes exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All handled data are legal and public.