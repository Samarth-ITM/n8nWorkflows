University Application Evaluation & Scholarship Automation with GPT-4 & JotForm

https://n8nworkflows.xyz/workflows/university-application-evaluation---scholarship-automation-with-gpt-4---jotform-9574


# University Application Evaluation & Scholarship Automation with GPT-4 & JotForm

### 1. Workflow Overview

This workflow automates the evaluation and communication process for university applications submitted via JotForm, leveraging GPT-4 AI to generate detailed application assessments and scholarship recommendations. It targets university admissions offices aiming to streamline application review, decision-making, and applicant notification. The logical flow is divided into these blocks:

- **1.1 Input Reception:** Captures application data submitted through a JotForm online form.
- **1.2 Data Extraction:** Parses and structures the raw form data into standardized variables for processing.
- **1.3 AI Processing & Evaluation:** Sends the application data to a GPT-4 powered AI agent that scores and categorizes the applicant based on set criteria, returning structured evaluation results.
- **1.4 Evaluation Parsing & Fallback:** Parses AI JSON output; if parsing fails, applies a heuristic scoring fallback based on GPA and SAT scores.
- **1.5 Decision Routing:** Routes applications into “Auto Accept”, “Interview Required”, or “Reject” branches based on evaluation results.
- **1.6 Notification & Logging:** Sends tailored emails (acceptance, interview invitation, rejection) to applicants, alerts admissions staff of exceptional candidates, and logs application data and evaluation results to a Google Sheets database.

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception

- **Overview:**  
Receives and triggers workflow execution when a university application is submitted via a specific JotForm.

- **Nodes Involved:**  
  - JotForm Application

- **Node Details:**  
  - **JotForm Application**  
    - Type: JotForm Trigger  
    - Role: Entry point, listens for new submissions on JotForm form ID `252853345242456`.  
    - Configuration: Uses API credentials for JotForm; webhook ID set for this trigger.  
    - Expressions/Variables: Receives full JSON payload of form submission with applicant details.  
    - Input: External JotForm submission event.  
    - Output: Raw JSON with student data (name, email, phone, GPA, SAT score, essay, extracurriculars, intended major).  
    - Edge Cases: Form API connectivity issues, webhook failures, data schema changes in form inputs.  
    - Version-Specific: n8n JotForm node v1.

---

#### 2.2 Data Extraction

- **Overview:**  
Transforms raw form submission JSON into standardized workflow variables, assigning unique application IDs and timestamps.

- **Nodes Involved:**  
  - Extract Application Data

- **Node Details:**  
  - **Extract Application Data**  
    - Type: Set node  
    - Role: Maps and renames form fields into explicit variables for downstream processing.  
    - Configuration: Extracts fields such as `student_name`, `student_email`, `phone`, `gpa` (as number), `sat_score` (number), `intended_major`, `extracurriculars`, `essay`. Generates a unique application ID with timestamp and random string, captures submission date as ISO string.  
    - Expressions Used: Template expressions like `={{ $json.Name }}`, `={{ new Date().toISOString() }}`, and custom ID generation combining Date.now and random substring.  
    - Input: Raw JSON from JotForm Application node.  
    - Output: JSON with clean, typed fields and metadata for evaluation.  
    - Edge Cases: Missing or malformed form fields, type coercion errors.  
    - Version-Specific: n8n Set node v3.4.

---

#### 2.3 AI Processing & Evaluation

- **Overview:**  
Uses a GPT-4 powered AI agent to evaluate the applicant holistically, scoring academic merit, extracurriculars, essay quality, and fit, then categorizes the application.

- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model

- **Node Details:**  
  - **AI Agent**  
    - Type: Langchain Agent  
    - Role: Defines prompt and evaluation criteria for the AI model; acts as interface to GPT-4.  
    - Configuration: Contains detailed evaluation prompt specifying scoring weights, decision categories, scholarship tiers, ethical guidelines, and output requirements (strict JSON only, no markdown).  
    - Input: Cleaned application data from Extract Application Data node.  
    - Output: AI evaluation results JSON with scores (eligibility_score), decision_category, academic_strength, scholarship eligibility, estimated scholarship, admission likelihood, and interview focus suggestions.  
    - Edge Cases: Prompt misinterpretation, output formatting issues, API call failures or rate limiting.  
    - Version-Specific: Langchain Agent v2.2.  

  - **OpenAI Chat Model**  
    - Type: Langchain LM Chat OpenAI  
    - Role: Executes GPT-4.1-mini model call with prompt from AI Agent node.  
    - Configuration: Model set to GPT-4.1-mini, requires OpenAI API credentials.  
    - Input: Prompt from AI Agent node.  
    - Output: Raw AI response JSON with evaluation details.  
    - Edge Cases: API key invalidation, timeout, network errors.  
    - Version-Specific: Langchain OpenAI node v1.2.

---

#### 2.4 Evaluation Parsing & Fallback

- **Overview:**  
Parses the AI agent’s JSON evaluation output; if parsing fails, applies a heuristic fallback scoring based on GPA and SAT to ensure workflow continuity.

- **Nodes Involved:**  
  - Parse AI Evaluation

- **Node Details:**  
  - **Parse AI Evaluation**  
    - Type: Code node (JavaScript)  
    - Role: Extracts JSON content from AI response text, strips markdown code fences, parses JSON, merges with original application data, or applies fallback logic if parsing fails.  
    - Configuration: Code extracts AI message content from the first choice, cleans it, attempts JSON parse. On failure, computes a fallback score based on GPA and SAT thresholds, assigns decision categories (`auto_accept`, `interview_required`, `reject`), and estimates scholarship/likelihood.  
    - Input: AI Agent output JSON.  
    - Output: Fully merged application + evaluation JSON.  
    - Edge Cases: Malformed AI output, unexpected text format, JSON parse errors, missing fields.  
    - Version-Specific: Code node v2.

---

#### 2.5 Decision Routing

- **Overview:**  
Routes the application into one of three branches (accept, interview, reject) depending on the AI/fallback decision category.

- **Nodes Involved:**  
  - Auto Accept?  
  - Interview Required?

- **Node Details:**  
  - **Auto Accept?**  
    - Type: If node  
    - Role: Checks if `decision_category` equals `"auto_accept"`.  
    - Configuration: Case-sensitive string equality condition.  
    - Input: Parsed evaluation JSON.  
    - Output: True branch to acceptance email node, False branch to rejection email node.  
    - Edge Cases: Missing or unexpected `decision_category`.  
    - Version-Specific: If node v2.2.

  - **Interview Required?**  
    - Type: If node  
    - Role: Checks if `decision_category` equals `"interview_required"`.  
    - Configuration: Case-sensitive string equality condition.  
    - Input: Parsed evaluation JSON.  
    - Output: True branch to interview invitation email node.  
    - Edge Cases: Overlaps with other categories; must be mutually exclusive logic.  
    - Version-Specific: If node v2.2.

---

#### 2.6 Notification & Logging

- **Overview:**  
Sends customized emails to applicants based on decision, alerts admissions team for high-score applicants, and logs data to Google Sheets for recordkeeping.

- **Nodes Involved:**  
  - Send Acceptance Letter  
  - Send Interview Invitation  
  - Send Rejection Letter  
  - Alert Admissions Team  
  - Log to Database

- **Node Details:**  
  - **Send Acceptance Letter**  
    - Type: Gmail node (OAuth2)  
    - Role: Sends acceptance email including scholarship info and next steps.  
    - Configuration: Uses Gmail OAuth2 credentials; HTML email with dynamic placeholders for student name, major, scores, scholarship eligibility, and application ID. Subject line: "Congratulations! Acceptance to Our University".  
    - Input: True branch of Auto Accept? node.  
    - Output: Triggers admissions alert email node.  
    - Edge Cases: Email sending failures, invalid email addresses.  
    - Version-Specific: Gmail node v2.1.

  - **Send Interview Invitation**  
    - Type: Gmail node (OAuth2)  
    - Role: Sends interview invitation email with scheduling instructions and application ID.  
    - Configuration: Uses Gmail OAuth2 credentials; HTML formatted message with dynamic data. Subject: "Interview Invitation - University Admissions".  
    - Input: True branch of Interview Required? node.  
    - Output: Triggers admissions alert email node.  
    - Edge Cases: Email delivery issues.  
    - Version-Specific: Gmail node v2.1.

  - **Send Rejection Letter**  
    - Type: Gmail node (OAuth2)  
    - Role: Sends polite rejection email with encouragement to reapply or explore other options.  
    - Configuration: Gmail OAuth2; HTML email template with placeholders. Subject: "Update on Your University Application".  
    - Input: False branch of Auto Accept? node.  
    - Output: Triggers database logging node.  
    - Edge Cases: Email sending or formatting errors.  
    - Version-Specific: Gmail node v2.1.

  - **Alert Admissions Team**  
    - Type: Gmail node (OAuth2)  
    - Role: Notifies admissions team of exceptional applicants with detailed evaluation summary.  
    - Configuration: Sends to `admissions@university.edu`, HTML email summarizing student data and evaluation results, including scholarship status. Subject line includes student name.  
    - Input: After acceptance or interview invitation emails.  
    - Output: Triggers database logging node.  
    - Edge Cases: Internal email delivery issues.  
    - Version-Specific: Gmail node v2.1.

  - **Log to Database**  
    - Type: Google Sheets node (OAuth2)  
    - Role: Appends application and evaluation data to a Google Sheets document for persistent storage and reporting.  
    - Configuration: Maps fields such as GPA, Name, Essay, Major, Email, SAT Score, Phone Number, Extracurriculars; matches on `id` column; targets sheet `gid=0` in specified spreadsheet ID.  
    - Input: From Alert Admissions Team or Send Rejection Letter nodes.  
    - Output: Terminal node.  
    - Edge Cases: Sheet permission issues, rate limits, schema changes.  
    - Version-Specific: Google Sheets node v4.5.

---

### 3. Summary Table

| Node Name              | Node Type                   | Functional Role                       | Input Node(s)               | Output Node(s)                          | Sticky Note                                                                                  |
|------------------------|-----------------------------|-------------------------------------|-----------------------------|----------------------------------------|----------------------------------------------------------------------------------------------|
| JotForm Application    | JotForm Trigger             | Receives application submissions    | External JotForm webhook     | Extract Application Data                | ## 📝 Application Form - **JotForm** trigger receives application. Create your form for free on [JotForm using this link](https://www.jotform.com/?partner=mediajade) Captures: • Student info • GPA & test scores • Essay • Major choice |
| Extract Application Data | Set                        | Extracts and standardizes form data | JotForm Application         | AI Agent                               | ## 🤖 AI Evaluation Scores: • Academic merit • Scholarship • Decision Categories: • Auto-accept (95+) • Interview (70-94) • Reject (<70) |
| AI Agent               | Langchain Agent             | Constructs AI evaluation prompt      | Extract Application Data     | Parse AI Evaluation                     | ## 🤖 AI Evaluation (shared with Extract Application Data)                                   |
| OpenAI Chat Model      | Langchain LM Chat OpenAI    | Calls GPT-4 model                    | AI Agent                    | AI Agent                               |                                                                                              |
| Parse AI Evaluation    | Code                       | Parses AI JSON or applies fallback  | AI Agent                    | Auto Accept?, Interview Required?       | ## 📧 Smart Routing **Accept:** Letter + scholarship **Interview:** Invitation **Reject:** Polite letter **All:** Alert team + database |
| Auto Accept?           | If                         | Routes auto-accept decisions         | Parse AI Evaluation          | Send Acceptance Letter, Send Rejection Letter | ## 📧 Smart Routing (shared)                                                                |
| Interview Required?    | If                         | Routes interview-required decisions  | Parse AI Evaluation          | Send Interview Invitation               | ## 📧 Smart Routing (shared)                                                                |
| Send Acceptance Letter | Gmail                      | Sends acceptance email               | Auto Accept? (true branch)   | Alert Admissions Team                   | ## 📧 Smart Routing (shared)                                                                |
| Send Interview Invitation | Gmail                    | Sends interview invitation email     | Interview Required?          | Alert Admissions Team                   | ## 📧 Smart Routing (shared)                                                                |
| Send Rejection Letter  | Gmail                      | Sends rejection email                | Auto Accept? (false branch)  | Log to Database                        | ## 📧 Smart Routing (shared)                                                                |
| Alert Admissions Team  | Gmail                      | Alerts admissions staff              | Send Acceptance Letter, Send Interview Invitation | Log to Database                        | ## 📧 Smart Routing (shared)                                                                |
| Log to Database        | Google Sheets              | Logs application data                | Alert Admissions Team, Send Rejection Letter | None                                   | ## 📧 Smart Routing (shared)                                                                |
| Sticky Note1           | Sticky Note                | Notes on Application Form            | None                       | None                                   | ## 📝 Application Form - **JotForm** trigger receives application. Create your form for free on [JotForm using this link](https://www.jotform.com/?partner=mediajade) Captures: • Student info • GPA & test scores • Essay • Major choice |
| Sticky Note2           | Sticky Note                | Notes on AI Evaluation                | None                       | None                                   | ## 🤖 AI Evaluation Scores: • Academic merit • Scholarship • Decision Categories: • Auto-accept (95+) • Interview (70-94) • Reject (<70) |
| Sticky Note3           | Sticky Note                | Notes on Smart Routing & Notifications | None                       | None                                   | ## 📧 Smart Routing **Accept:** Letter + scholarship **Interview:** Invitation **Reject:** Polite letter **All:** Alert team + database |

---

### 4. Reproducing the Workflow from Scratch

1. **Create JotForm Application Trigger Node**  
   - Type: JotForm Trigger  
   - Configure with your JotForm API credentials.  
   - Select the university application form by its ID or URL (`252853345242456`).  
   - Ensure webhook is enabled to trigger on new submissions.

2. **Add Set Node: Extract Application Data**  
   - Map form fields to standardized variables:  
     - `student_name` = `{{$json.Name}}`  
     - `student_email` = `{{$json['E-mail']}}`  
     - `phone` = `{{$json['Phone Number'].full}}`  
     - `gpa` = numeric parse of `{{$json.GPA}}`  
     - `sat_score` = numeric parse of `{{$json['SAT Score']}}`  
     - `intended_major` = `{{$json.Major}}`  
     - `extracurriculars` = `{{$json.Extracurriculars}}`  
     - `essay` = `{{$json.Essay}}`  
     - `application_id` = `APP-{{Date.now()}}-{{Math.random().toString(36).substr(2,8).toUpperCase()}}`  
     - `submission_date` = `{{new Date().toISOString()}}`  
   - Connect output of JotForm node to this node.

3. **Add Langchain Agent Node: AI Agent**  
   - Configure prompt text specifying admissions evaluation criteria, scoring weights, decision categories, scholarship tiers, ethical guidelines, and output format.  
   - Use the extracted application data as input.  
   - Connect from Extract Application Data.

4. **Add Langchain LM Chat OpenAI Node: OpenAI Chat Model**  
   - Select GPT-4.1-mini model.  
   - Provide valid OpenAI API credentials.  
   - Connect AI Agent node to this node as language model.

5. **Connect OpenAI Chat Model output back to AI Agent input**  
   - This creates the AI agent loop for generating evaluation.

6. **Add Code Node: Parse AI Evaluation**  
   - Insert JavaScript code to:  
     - Extract AI JSON from text, clean markdown fences.  
     - Parse JSON safely.  
     - On failure, fallback to heuristic scoring based on GPA and SAT.  
     - Merge result with original application data.  
   - Connect output of AI Agent node to this node.

7. **Add If Node: Auto Accept?**  
   - Condition: Check if `{{$json.decision_category}}` equals `"auto_accept"` (case sensitive).  
   - Connect output of Parse AI Evaluation node.

8. **Add If Node: Interview Required?**  
   - Condition: Check if `{{$json.decision_category}}` equals `"interview_required"` (case sensitive).  
   - Connect output of Parse AI Evaluation node.

9. **Add Gmail Node: Send Acceptance Letter**  
   - Configure with Gmail OAuth2 credentials.  
   - Set recipient: `{{$json.student_email}}`.  
   - Compose HTML email including student name, major, eligibility score, academic strength, scholarship info (conditional), and next steps.  
   - Subject: "Congratulations! Acceptance to Our University".  
   - Connect true branch of Auto Accept? node to this node.

10. **Add Gmail Node: Send Interview Invitation**  
    - Configure with Gmail OAuth2.  
    - Recipient: `{{$json.student_email}}`.  
    - HTML email with interview invitation details, application ID, scheduling URL.  
    - Subject: "Interview Invitation - University Admissions".  
    - Connect true branch of Interview Required? node.

11. **Add Gmail Node: Send Rejection Letter**  
    - Configure with Gmail OAuth2.  
    - Recipient: `{{$json.student_email}}`.  
    - HTML email politely declining application, encouragement to reapply.  
    - Subject: "Update on Your University Application".  
    - Connect false branch of Auto Accept? node.

12. **Add Gmail Node: Alert Admissions Team**  
    - Configure with Gmail OAuth2.  
    - Recipient: admissions@university.edu  
    - HTML email summarizing student name, major, GPA, SAT, eligibility score, decision, scholarship status, application ID.  
    - Subject includes student name.  
    - Connect output of Send Acceptance Letter and Send Interview Invitation nodes to this node.

13. **Add Google Sheets Node: Log to Database**  
    - Configure with Google Sheets OAuth2 credentials.  
    - Document ID: your Google Sheet containing application records.  
    - Sheet name: `gid=0` (or your target sheet).  
    - Append operation with columns mapped from application data (GPA, Name, Essay, Major, E-mail, SAT Score, Phone Number, Extracurriculars, id).  
    - Connect output of Alert Admissions Team and Send Rejection Letter to this node.

14. **Add Sticky Notes (Optional)**  
    - Add notes describing application form, AI evaluation criteria, and smart routing logic for documentation clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                            | Context or Link                                                                                     |
|-------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Create your university application form free on JotForm using this link: https://www.jotform.com/?partner=mediajade     | JotForm signup link referenced in Sticky Note1                                                   |
| AI evaluation prompt designed to ensure fairness, ethical guidelines, and consistency in scoring across applications     | Embedded within AI Agent node configuration                                                     |
| Smart routing sends acceptance with scholarship info, interview invites, or rejection letters plus alerts and logging   | Sticky Note3 summarizes notification and logging logic                                           |
| Google Sheets used as a simple database for application recordkeeping; ensure proper OAuth2 credentials and sheet access | Google Sheets node configured for sheet ID and append operation                                  |
| Uses Gmail OAuth2 credentials for sending emails; ensure tokens are valid and have proper scopes                         | Gmail nodes require OAuth2 authentication with send email scope                                  |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated n8n workflow. It complies fully with content policies and contains no illegal, offensive, or protected content. All handled data is legal and publicly permissible.