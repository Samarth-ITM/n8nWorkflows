Generate Employee Retention Analytics Reports with GPT-4o and Gmail Digests

https://n8nworkflows.xyz/workflows/generate-employee-retention-analytics-reports-with-gpt-4o-and-gmail-digests-9236


# Generate Employee Retention Analytics Reports with GPT-4o and Gmail Digests

### 1. Workflow Overview

This workflow, titled **"Generate Employee Retention Analytics Reports with GPT-4o and Gmail Digests"**, automates the process of generating insightful retention analytics reports for employee post-hire data. It targets HR teams and hiring managers seeking to understand retention patterns, traits influencing retention, and actionable recommendations to improve hiring decisions.

The workflow is logically grouped into these main blocks:

- **1.1 Input Reception:** Trigger and data retrieval from Google Sheets for candidate and trait data.
- **1.2 Data Consolidation and Scoring:** Merging datasets, cleaning, normalizing, and scoring candidates based on traits.
- **1.3 Data Validation and Error Handling:** Ensuring data integrity before processing and logging failures.
- **1.4 AI Processing and Digest Generation:** Leveraging Azure OpenAI GPT-4o to generate an HTML retention digest.
- **1.5 Email Delivery:** Sending the formatted retention digest via Gmail.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block triggers the workflow execution and fetches candidate and trait data from designated Google Sheets, which serve as the data sources.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Candidate Data Fetch (Google Sheets)  
- Trait Summary Fetch (Google Sheets)

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Entry point for manual execution in n8n.  
  - *Config:* No parameters needed; triggers workflow on user click.  
  - *Connections:* Outputs to both Candidate Data Fetch and Trait Summary Fetch.  
  - *Edge Cases:* None; manual trigger is straightforward.

- **Candidate Data Fetch**  
  - *Type:* Google Sheets (Read)  
  - *Role:* Retrieves detailed candidate post-hire data, including names, roles, traits, start dates, and retention status.  
  - *Config:* Reads from the "Retention Summary" sheet within a specific Google Sheet document. OAuth2 credentials linked to `automations@techdome.ai`.  
  - *Key Expressions:* None; static sheet and document IDs.  
  - *Inputs:* From Manual Trigger  
  - *Outputs:* To Merge Candidate + Trait Data node (via second input).  
  - *Failure Types:* Authentication errors, sheet access issues, empty or malformed data.

- **Trait Summary Fetch**  
  - *Type:* Google Sheets (Read)  
  - *Role:* Retrieves aggregated trait-level retention data, such as retention percentages and weight adjustments.  
  - *Config:* Reads from "Hires Tracking" sheet in the same Google Sheet document as above. Same OAuth2 credentials.  
  - *Inputs:* From Manual Trigger  
  - *Outputs:* To Merge Candidate + Trait Data node (via first input).  
  - *Failure Types:* Same as Candidate Data Fetch.

---

#### 1.2 Data Consolidation and Scoring

**Overview:**  
Merges candidate and trait datasets into a unified stream, then normalizes, cleans, and enriches data by calculating candidate scores based on weighted traits.

**Nodes Involved:**  
- Merge Candidate + Trait Data (Merge)  
- Candidate Scoring & Data Normalization (Code)

**Node Details:**

- **Merge Candidate + Trait Data**  
  - *Type:* Merge  
  - *Role:* Combines trait summary and candidate data streams into one consolidated dataset.  
  - *Config:* Default merge settings; merges two input streams intact.  
  - *Inputs:* Trait Summary Fetch (first input), Candidate Data Fetch (second input)  
  - *Outputs:* Candidate Scoring & Data Normalization  
  - *Failure Types:* Mismatch in data structure or empty inputs could cause downstream issues.

- **Candidate Scoring & Data Normalization**  
  - *Type:* Code (JavaScript)  
  - *Role:*  
    - Splits combined dataset into candidate and trait arrays.  
    - Normalizes headers and trims whitespace for consistency.  
    - Builds a lookup map from traits to weight adjustments.  
    - Calculates each candidate’s score by summing weights of their associated traits.  
    - Outputs JSON with arrays: `candidates[]` and `traits[]`.  
  - *Config Highlights:*  
    - Accesses input via `$input.all()`.  
    - Handles missing data gracefully (e.g., defaulting to empty strings or zeros).  
    - Uses `split()` and `map()` for trait parsing and scoring.  
  - *Inputs:* Merged data from previous node  
  - *Outputs:* Data Validation node  
  - *Edge Cases:*  
    - Missing or malformed fields in input JSON.  
    - Candidates with traits not listed in the trait weight map (defaults to zero weight).  
    - Potential JavaScript runtime errors if input structure radically deviates.

---

#### 1.3 Data Validation and Error Handling

**Overview:**  
Validates that both candidate and trait datasets exist and contain records before proceeding. If validation fails, logs errors into a Google Sheets error log to ensure graceful failure.

**Nodes Involved:**  
- Data Validation (If)  
- Error Handling Logic (Google Sheets Append)

**Node Details:**

- **Data Validation**  
  - *Type:* If (Conditional)  
  - *Role:* Checks if both `candidates` and `traits` arrays have length > 0.  
  - *Config:*  
    - Condition: `candidates.length > 0 AND traits.length > 0`  
    - Routes TRUE to Retention Digest Generator, FALSE to Error Handling Logic.  
  - *Inputs:* Candidate Scoring & Data Normalization output  
  - *Outputs:*  
    - TRUE → Retention Digest Generator  
    - FALSE → Error Handling Logic  
  - *Edge Cases:* Empty input arrays, unexpected JSON structure.

- **Error Handling Logic**  
  - *Type:* Google Sheets (Append)  
  - *Role:* Logs error details such as error_id and error message into a dedicated error log sheet.  
  - *Config:*  
    - Appends rows to "error log sheet" within the same Google Sheet document.  
    - Maps `error_id` and `error` fields.  
    - Same OAuth2 credentials as other Google Sheets nodes.  
  - *Inputs:* From Data Validation (FALSE branch)  
  - *Outputs:* None (end of error path)  
  - *Edge Cases:* Sheet permission or connectivity issues, malformed error data.

---

#### 1.4 AI Processing and Digest Generation

**Overview:**  
Uses Azure OpenAI GPT-4o to generate an insightful, styled HTML retention digest email based on the validated candidate and trait data.

**Nodes Involved:**  
- AI Processing Backend (Azure OpenAI LLM Chat)  
- Retention Digest Generator (LangChain LLM Chain)

**Node Details:**

- **AI Processing Backend**  
  - *Type:* LangChain LLM Chat Azure OpenAI  
  - *Role:* Executes GPT-4o-mini model for generating AI content.  
  - *Config:*  
    - Model: `gpt-4o-mini`  
    - Credentials: Azure OpenAI API linked via OAuth2.  
    - No additional options configured.  
  - *Inputs:* From Data Validation (TRUE branch) via Retention Digest Generator node (ai_languageModel input)  
  - *Outputs:* To Retention Digest Generator node (ai_languageModel output)  
  - *Edge Cases:* API quota exceeded, authentication failures, response latency.

- **Retention Digest Generator**  
  - *Type:* LangChain Chain LLM  
  - *Role:*  
    - Defines a strict prompt to generate an HTML retention digest email with sections such as TL;DR summary, top/weak traits, candidate highlights, and actionable tips.  
    - Enforces strict rules: no hallucination, dataset-only content, valid inline CSS HTML output.  
  - *Config:*  
    - Prompt includes dataset JSON stringification and detailed formatting instructions (colors, headers, button styles).  
    - Output: HTML email body (no markdown or code fences).  
  - *Inputs:* From AI Processing Backend (language model output) and Data Validation (main input)  
  - *Outputs:* Email Delivery node  
  - *Edge Cases:* Malformed AI output, invalid HTML, prompt processing errors.

---

#### 1.5 Email Delivery

**Overview:**  
Sends the generated retention digest email to designated recipients via Gmail.

**Nodes Involved:**  
- Email Delivery (Gmail)

**Node Details:**

- **Email Delivery**  
  - *Type:* Gmail (Send Email)  
  - *Role:* Sends the retention digest email with HTML body generated by the AI.  
  - *Config:*  
    - To: `newscctv22@gmail.com` (example email)  
    - Subject: "Retention Analysis Digest - Weekly Update"  
    - Body: HTML content from Retention Digest Generator  
    - Credentials: Gmail OAuth2 linked account  
    - CC: None by default  
  - *Inputs:* From Retention Digest Generator  
  - *Outputs:* None (end of workflow)  
  - *Edge Cases:* Authentication failure, recipient address issues, email sending limits.

---

### 3. Summary Table

| Node Name                     | Node Type                       | Functional Role                            | Input Node(s)                      | Output Node(s)                   | Sticky Note                                                                                                         |
|-------------------------------|--------------------------------|-------------------------------------------|----------------------------------|---------------------------------|---------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                 | Workflow start trigger                      | None                             | Candidate Data Fetch, Trait Summary Fetch |                                                                                                                     |
| Candidate Data Fetch           | Google Sheets                  | Fetch candidate post-hire data              | When clicking ‘Execute workflow’ | Merge Candidate + Trait Data     | 📑 Candidate Data Fetch (Google Sheets – Hires Tracking): Retrieves detailed candidate info for scoring and reporting. |
| Trait Summary Fetch            | Google Sheets                  | Fetch trait-level retention summary         | When clicking ‘Execute workflow’ | Merge Candidate + Trait Data     | 📑 Trait Summary Fetch (Google Sheets – Retention Summary): Fetches aggregated trait stats for scoring.              |
| Merge Candidate + Trait Data   | Merge                         | Combine candidate and trait datasets        | Candidate Data Fetch, Trait Summary Fetch | Candidate Scoring & Data Normalization | 🔀 Merge Candidate + Trait Data: Combines granular and aggregated data for scoring.                                |
| Candidate Scoring & Data Normalization | Code                     | Normalize data, calculate candidate scores  | Merge Candidate + Trait Data      | Data Validation                 | 🧮 Candidate Scoring & Data Normalization (Code Node): Cleans, normalizes, enriches data, calculates scores.          |
| Data Validation               | If                            | Validate candidate and trait data presence  | Candidate Scoring & Data Normalization | Retention Digest Generator (TRUE), Error Handling Logic (FALSE) | ✅ Data Validation: Checks for non-empty datasets before AI/email stages.                                           |
| Error Handling Logic          | Google Sheets (Append)         | Log errors for failed runs                   | Data Validation (FALSE)           | None                           | ⚠️ Error Handling Logic: Logs error details to error log sheet for visibility and graceful failure handling.          |
| AI Processing Backend         | LangChain LLM Chat Azure OpenAI | Execute GPT-4o model for AI content         | Retention Digest Generator (ai_languageModel input) | Retention Digest Generator (ai_languageModel output) | 🧠 AI Processing Backend: Uses Azure OpenAI GPT-4o for processing candidate + trait data.                             |
| Retention Digest Generator    | LangChain Chain LLM            | Generate HTML retention digest email         | Data Validation (TRUE), AI Processing Backend | Email Delivery                | 🤖 Retention Digest Generator: Creates styled retention insights HTML email with strict prompting rules.             |
| Email Delivery               | Gmail                         | Send retention digest email                   | Retention Digest Generator        | None                           | 📧 Email Delivery: Sends digest email to hiring managers/stakeholders.                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Name: "When clicking ‘Execute workflow’"  
   - No configuration needed.

2. **Create Google Sheets Node for Candidate Data**  
   - Type: Google Sheets (Read)  
   - Name: "Candidate Data Fetch"  
   - Set Document ID to the source Google Sheet (e.g., `1Uldk_4BxWbdZTDZxFUeohIfeBmGHHqVEl9Ogb0l6R8Y`)  
   - Set Sheet Name to "Retention Summary" (by sheet GID or name)  
   - Authenticate with OAuth2 credentials linked to the relevant Google account.  
   - Connect from Manual Trigger.

3. **Create Google Sheets Node for Trait Data**  
   - Type: Google Sheets (Read)  
   - Name: "Trait Summary Fetch"  
   - Use the same Document ID as above.  
   - Set Sheet Name to "Hires Tracking"  
   - Use same OAuth2 credentials.  
   - Connect from Manual Trigger.

4. **Create Merge Node**  
   - Type: Merge  
   - Name: "Merge Candidate + Trait Data"  
   - Connect Trait Summary Fetch output to Merge node’s first input.  
   - Connect Candidate Data Fetch output to Merge node’s second input.

5. **Create Code Node for Scoring and Normalization**  
   - Type: Code (JavaScript)  
   - Name: "Candidate Scoring & Data Normalization"  
   - Copy and paste the provided JavaScript code that:  
     - Splits input into candidates and traits arrays.  
     - Normalizes data fields and trims spaces.  
     - Builds weight lookup from traits.  
     - Calculates candidate scores by summing trait weights.  
     - Returns JSON containing both arrays.  
   - Connect output of Merge node to this Code node.

6. **Create If Node for Data Validation**  
   - Type: If  
   - Name: "Data Validation"  
   - Set condition:  
     - `$json.candidates.length > 0` (number greater than 0)  
     - AND  
     - `$json.traits.length > 0` (number greater than 0)  
   - Connect output of Code node to If node.

7. **Create Google Sheets Node for Error Logging**  
   - Type: Google Sheets (Append)  
   - Name: "Error Handling Logic"  
   - Set Document ID to the same Google Sheet.  
   - Set Sheet Name to "error log sheet" (by GID or name).  
   - Map columns `error_id` and `error` for error details.  
   - Use same OAuth2 credentials.  
   - Connect FALSE output of If node to this node.

8. **Create Azure OpenAI Node**  
   - Type: LangChain LLM Chat Azure OpenAI  
   - Name: "AI Processing Backend"  
   - Set model to `gpt-4o-mini`.  
   - Configure Azure OpenAI credentials (API key, endpoint).  
   - Connect ai_languageModel input from Retention Digest Generator node (next step).

9. **Create LangChain Chain LLM Node for Digest Generation**  
   - Type: LangChain Chain LLM  
   - Name: "Retention Digest Generator"  
   - Insert prompt text with embedded `{{ JSON.stringify($json, null, 2) }}` for dataset input.  
   - Include strict rules to avoid hallucination, enforce HTML email formatting with inline CSS.  
   - Connect Data Validation (TRUE) output to this node.  
   - Connect AI Processing Backend node’s output to this node’s ai_languageModel input.

10. **Create Gmail Node for Email Delivery**  
    - Type: Gmail (Send Email)  
    - Name: "Email Delivery"  
    - Configure recipient email (e.g., `newscctv22@gmail.com`)  
    - Subject: "Retention Analysis Digest - Weekly Update"  
    - Message body: Use HTML content from Retention Digest Generator node (`{{$json.text}}`).  
    - Authenticate using Gmail OAuth2 credentials.  
    - Connect output of Retention Digest Generator.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                     | Context or Link                                                      |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------|
| The workflow enforces strict AI prompt rules to prevent hallucinations and ensure all insights are drawn solely from the provided dataset. The HTML output is styled with inline CSS and designed for email compatibility.          | Prompt details in Retention Digest Generator node.                 |
| OAuth2 credentials for Google Sheets and Gmail must have appropriate permissions to read/write sheets and send emails, respectively.                                                                                           | Credential setup instructions in n8n documentation.               |
| The error log sheet provides visibility into data or execution issues without breaking the entire workflow, enabling easier debugging and operational monitoring.                                                               | Error Handling Logic sticky note.                                  |
| Candidate Scores are dynamically calculated by summing weights assigned to traits, which are fetched from the trait summary sheet, allowing for adaptable retention analysis based on real data.                               | Candidate Scoring & Data Normalization sticky note.                |
| The workflow is designed for manual execution but can be scheduled or triggered via webhook with minor adjustments to the trigger node.                                                                                        | Manual Trigger node can be replaced with other trigger nodes.     |
| For further understanding of LangChain nodes and Azure OpenAI integration, refer to official n8n docs and Azure AI service guides.                                                                                            | https://docs.n8n.io/nodes/ | https://learn.microsoft.com/en-us/azure/cognitive-services/openai/ |

---

**Disclaimer:**  
The text provided originates exclusively from an n8n automated workflow. It strictly complies with current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and publicly accessible.