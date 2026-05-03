Match students to scholarships with Claude AI, Airtable, and Google Sheets

https://n8nworkflows.xyz/workflows/match-students-to-scholarships-with-claude-ai--airtable--and-google-sheets-13690


# Match students to scholarships with Claude AI, Airtable, and Google Sheets

### 1. Workflow Overview

The **Automated Scholarship Eligibility Matcher** is a sophisticated n8n workflow designed to automate the complex process of matching student profiles against a database of scholarships. It replaces manual review with an AI-driven scoring system that evaluates both quantitative data (GPA, residency) and qualitative attributes (career goals, extracurriculars).

The workflow operates in two modes:
1.  **On-Demand:** Triggered via a webhook (e.g., from a form submission).
2.  **Batch Processing:** Triggered nightly to evaluate all active student records.

**Logical Blocks:**
*   **1.1 Intake & Data Loading:** Captures incoming student data and retrieves the current scholarship catalogue from Airtable.
*   **1.2 Data Normalisation & Pre-Filtering:** Standardizes data formats and applies "hard" exclusion rules (e.g., GPA cutoffs) to reduce AI token costs and improve accuracy.
*   **1.3 AI Eligibility Scoring:** Employs Claude AI (Anthropic) via LangChain to perform a nuanced assessment, ranking, and advice generation for each match.
*   **1.4 Multi-Channel Distribution & Logging:** Dispatches personalized emails to students, alerts advisors via Slack, updates the CRM (Airtable), and maintains an audit trail in Google Sheets.

---

### 2. Block-by-Block Analysis

#### 2.1 Intake & Data Loading
This block handles the entry points and the retrieval of external data.
*   **Nodes Involved:** `Receive Student Profile Submission`, `Nightly Batch Schedule`, `Load Student Profiles`, `Load Scholarship Catalogue`.
*   **Node Details:**
    *   **Webhook (Receive Student Profile Submission):** Accepts POST requests with student JSON.
    *   **Schedule Trigger:** Runs a CRON job at 22:00 nightly.
    *   **Airtable (Load Student Profiles):** Searches for "Active" students where a matching run has not occurred today.
    *   **Airtable (Load Scholarship Catalogue):** Fetches scholarships that are "Active" and whose deadline is after the current date.
*   **Potential Failures:** Authentication errors with Airtable; empty catalogue resulting in workflow termination.

#### 2.2 Profile Normalisation & Scholarship Pairing
This block prepares the data for the AI model by cleaning inputs and pairing students with the scholarship list.
*   **Nodes Involved:** `Normalise Profiles & Build Pairs`, `Pre-Filter Hard Ineligibility`.
*   **Node Details:**
    *   **Code Node (Normalise):** Merges Webhook or Airtable data into a unified schema. It includes fallback "demo" data logic if the sources are empty, ensuring the workflow is testable.
    *   **Code Node (Pre-Filter):** Executes deterministic logic. It removes scholarships where the student fails a "hard" requirement:
        *   GPA < Minimum GPA.
        *   Residency Status mismatch.
        *   Gender mismatch.
        *   Financial Need missing when required.
*   **Key Expressions:** Uses `student.gpa < sch.minGPA` and `sch.yearLevels.includes('ANY')`.

#### 2.3 Claude AI Eligibility Scoring & Ranking
The core intelligence block where qualitative matching occurs.
*   **Nodes Involved:** `AI Eligibility Scorer`, `Claude AI Model`.
*   **Node Details:**
    *   **AI Agent (Scorer):** Uses a custom prompt that includes a scoring rubric (0–100). It instructs the AI to return a specific JSON structure containing match scores, "selling points," and application tips.
    *   **Anthropic Chat Model:** Configured with `claude-sonnet-4-20250514` and a temperature of `0.2` for high consistency and low creativity (to ensure factual accuracy).
*   **Edge Cases:** AI might occasionally wrap JSON in markdown blocks (handled by the subsequent parsing node) or hallucinate a score if the prompt is too vague.

#### 2.4 Filtering & Output Processing
Post-processing the AI output and delivering results.
*   **Nodes Involved:** `Parse & Rank Match Results`, `Filter Qualified Matches`, `Build Personalised Student Email`.
*   **Node Details:**
    *   **Code (Parse & Rank):** Uses Regex to clean AI output and `JSON.parse` to turn text into objects. It sorts matches by `matchScore` descending and calculates total potential award value.
    *   **Filter:** Only passes items where `recommendedCount >= 1` (Match Score >= 60).
    *   **Code (Build Email):** Generates a dynamic HTML template with conditional "Urgent" banners for deadlines within 14 days and color-coded match badges.

#### 2.5 Distribution, CRM & Audit
Final execution of external actions.
*   **Nodes Involved:** `Send Scholarship Email to Student`, `Notify Academic Advisor on Slack`, `Update Student CRM Record`, `Write Match Audit Log`, `Return Match Results to Caller`.
*   **Node Details:**
    *   **Email (SMTP):** Sends the generated HTML to the student.
    *   **HTTP Request (Slack):** Posts a block-formatted message to `#scholarship-advisors` with student stats and top match.
    *   **Airtable (Update):** Writes the match results back to the student record.
    *   **Google Sheets:** Appends a log for historical auditing.
    *   **Respond to Webhook:** Returns the top 5 matches as a JSON response to the initial caller.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Receive Student Profile Submission | Webhook | Trigger (Manual) | None | Load Scholarship Catalogue | 1. Intake & Data Loading |
| Nightly Batch Schedule | Schedule | Trigger (Batch) | None | Load Student Profiles | 1. Intake & Data Loading |
| Load Student Profiles | Airtable | Data Retrieval | Nightly Batch Schedule | Normalise Profiles & Build Pairs | 1. Intake & Data Loading |
| Load Scholarship Catalogue | Airtable | Data Retrieval | Receive Student Profile Submission | Normalise Profiles & Build Pairs | 1. Intake & Data Loading |
| Normalise Profiles & Build Pairs | Code | Data Transformation | Load Student Profiles, Load Scholarship Catalogue | Pre-Filter Hard Ineligibility | 2. Profile Normalisation & Scholarship Pairing |
| Pre-Filter Hard Ineligibility | Code | Business Logic | Normalise Profiles & Build Pairs | AI Eligibility Scorer | 2. Profile Normalisation & Scholarship Pairing |
| AI Eligibility Scorer | AI Agent | Analysis | Pre-Filter Hard Ineligibility | Parse & Rank Match Results | 3. Claude AI Eligibility Scoring & Ranking |
| Claude AI Model | Anthropic Chat | AI Brain | AI Eligibility Scorer | AI Eligibility Scorer | 3. Claude AI Eligibility Scoring & Ranking |
| Parse & Rank Match Results | Code | Data Cleaning | AI Eligibility Scorer | Filter Qualified Matches | 4. Filtering · Personalised Notifications · CRM Update · Audit Log |
| Filter Qualified Matches | Filter | Logic Gate | Parse & Rank Match Results | Build Personalised Student Email | 4. Filtering · Personalised Notifications · CRM Update · Audit Log |
| Build Personalised Student Email | Code | Content Generation | Filter Qualified Matches | Email Send, Slack, Airtable Update | 4. Filtering · Personalised Notifications · CRM Update · Audit Log |
| Send Scholarship Email to Student | Email Send | Distribution | Build Personalised Student Email | Write Match Audit Log | 4. Filtering · Personalised Notifications · CRM Update · Audit Log |
| Notify Academic Advisor on Slack | HTTP Request | Notification | Build Personalised Student Email | Write Match Audit Log | 4. Filtering · Personalised Notifications · CRM Update · Audit Log |
| Update Student CRM Record | Airtable | Data Storage | Build Personalised Student Email | Write Match Audit Log | 4. Filtering · Personalised Notifications · CRM Update · Audit Log |
| Write Match Audit Log | Google Sheets | Logging | Send Email, Slack, Airtable Update | Return Results | 4. Filtering · Personalised Notifications · CRM Update · Audit Log |
| Return Match Results to Caller | Respond to Webhook | API Response | Write Match Audit Log | None | 4. Filtering · Personalised Notifications · CRM Update · Audit Log |

---

### 4. Reproducing the Workflow from Scratch

1.  **Environment Setup:** Ensure you have credentials for Anthropic, Airtable, Google Sheets, Slack, and an SMTP server.
2.  **Trigger Setup:** Create a **Webhook** node (POST, path: `match-scholarships`) and a **Schedule** node (set to your preferred batch time).
3.  **Airtable Integration:** 
    *   Create a "Scholarships" table with fields for Name, GPA requirements, residency requirements, and deadline.
    *   Create a "Students" table with profile details.
    *   Configure the **Airtable Search** nodes to fetch these records.
4.  **Data Processing (Normalisation):** Add a **Code** node. Use logic to detect if the input is from the webhook (single object) or Airtable (array). Map both to a standard internal JSON student object.
5.  **Hard Filters:** Add a **Code** node to iterate through the scholarship array for each student. Apply `if` statements for GPA and residency. This drastically reduces the data sent to the AI.
6.  **AI Integration:**
    *   Add an **AI Agent** node. Set the prompt to "Evaluate this student [JSON] against these scholarships [JSON]". 
    *   Strictly define a JSON-only response schema in the prompt.
    *   Connect the **Anthropic Chat Model** node; select `claude-3-5-sonnet` (or equivalent) and set temperature to 0.2.
7.  **Parsing Logic:** Use a **Code** node with `JSON.parse(aiText.replace(/```json/g, ''))` to handle the AI response safely.
8.  **Filtering:** Use a **Filter** node to discard students with no matches above a 60% score.
9.  **Communication Setup:**
    *   Add a **Code** node to generate an HTML string. Use `${student.firstName}` and loop through `${matches}` to build a table/card view.
    *   Add an **Email Send** node (SMTP) and map the HTML string.
    *   Add an **HTTP Request** node for Slack; use the `chat.postMessage` API with a block-kit JSON payload.
10. **Data Persistence:** 
    *   Add an **Airtable Update** node to mark the student record as "Processed".
    *   Add a **Google Sheets Append** node to log the student ID and match count.
11. **API Response:** Add a **Respond to Webhook** node as the final step to return the top results to the initiating system.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Full Project Source & Credits | Developed by OneClick IT Solution |
| Custom AI Solutions Contact | [Contact OneClick IT Solution](https://www.oneclickitsolution.com/contact-us/) |
| Recommended AI Scoring Rubric | 0–100 scale: 90+ Exceptional, 75-89 Strong, 60-74 Good. |
| Supported Criteria | GPA, Merit, Major, Financial Need, Nationality, Gender, Career Focus. |