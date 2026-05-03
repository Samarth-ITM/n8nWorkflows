Generate & Email Professional Release Notes with GitHub, JIRA & Google Gemini

https://n8nworkflows.xyz/workflows/generate---email-professional-release-notes-with-github--jira---google-gemini-6129


# Generate & Email Professional Release Notes with GitHub, JIRA & Google Gemini

---

### 1. Workflow Overview

This workflow automates the generation and emailing of professional production release notes by integrating GitHub, JIRA, and Google Gemini AI services. It is designed for technical release managers who need to produce polished, business-friendly release notes based on commit messages and JIRA issue details, then distribute these notes via email to subscribed stakeholders, such as CxOs and clients.

**Target Use Cases:**  
- Automatically trigger release note generation on GitHub push events  
- Extract relevant JIRA issue keys from commit messages  
- Retrieve detailed JIRA issue information  
- Use AI (Google Gemini) to generate well-structured, HTML-formatted release notes  
- Email the release notes to predefined recipients  

**Logical Blocks:**  
- **1.1 Input Reception:** GitHub webhook listens for push events and extracts commit data  
- **1.2 JIRA Issue Retrieval:** Extract JIRA IDs from commits, query JIRA for issue details  
- **1.3 Data Aggregation & Transformation:** Clean and combine JIRA issue data for AI input  
- **1.4 AI Processing:** Use Google Gemini and LangChain to generate formatted release notes  
- **1.5 Email Dispatch:** Send the formatted release notes via SMTP email  

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block listens for GitHub push events, extracts commit messages and timestamps, and identifies associated JIRA issue IDs referenced in the commit messages.

**Nodes Involved:**  
- Github Trigger  
- Code  

**Node Details:**  

- **Github Trigger**  
  - Type: GitHub Trigger (Webhook)  
  - Configuration: Listens to push events on a specified owner and repository; uses secured GitHub API credentials  
  - Inputs: External webhook call from GitHub on push  
  - Outputs: JSON containing push payload including commits array  
  - Failure Modes: Webhook connectivity issues, GitHub API rate limits, repository permission errors  
  - Version: 1  
  - Notes: Secure SSL enabled (insecureSSL: false)  

- **Code**  
  - Type: Code (JavaScript)  
  - Configuration: Extracts commits array from webhook data; uses regex to extract JIRA IDs matching pattern like "ABC-123" (case-insensitive) from commit messages; outputs an array of objects with commit message, timestamp, and extracted jira_id (uppercased) or null if none found  
  - Expressions: Uses $input.all() to fetch all input data; regex `/([A-Z]+-\d+)/i` for JIRA ID extraction  
  - Inputs: Output of Github Trigger  
  - Outputs: Array of simplified commit objects with jira_id  
  - Failure Modes: No commits found, commit message missing expected pattern, regex failures  

#### 1.2 JIRA Issue Retrieval

**Overview:**  
For each extracted JIRA ID, this block queries JIRA Software Cloud to obtain detailed issue data including summary and description.

**Nodes Involved:**  
- Get an issue  
- Code2  

**Node Details:**  

- **Get an issue**  
  - Type: JIRA Node  
  - Configuration: Uses dynamic issueKey from `jira_id` field; operation is "get" to retrieve issue details for each JIRA ID  
  - Inputs: Output from Code node (commit objects with jira_id)  
  - Outputs: Raw JIRA issue JSON objects  
  - Credentials: JIRA Software Cloud API OAuth2 or API token  
  - Failure Modes: Invalid or missing jira_id, API authentication failure, rate limits, issue not found errors  
  - Version: 1  

- **Code2**  
  - Type: Code (JavaScript)  
  - Configuration: Maps all input JIRA issues; extracts key fields: `jira_id` (issue key), `jira_summary`, and `jira_description` (with optional chaining to handle missing fields)  
  - Inputs: Output of Get an issue (multiple JIRA issues)  
  - Outputs: Cleaned array of simplified JIRA issue objects with essential fields  
  - Failure Modes: Missing or malformed fields, empty responses  

#### 1.3 Data Aggregation & Transformation

**Overview:**  
Combines the cleaned JIRA issue data into a single collection suitable for input to the AI model.

**Nodes Involved:**  
- Merge  
- Code3  

**Node Details:**  

- **Merge**  
  - Type: Merge  
  - Configuration: Combines multiple input streams by position into a single output array  
  - Inputs: Outputs from Get an issue and Code2 nodes  
  - Outputs: Combined array of JIRA issue objects  
  - Failure Modes: Mismatched input array lengths, timing issues  

- **Code3**  
  - Type: Code (JavaScript)  
  - Configuration: Consolidates all incoming items into a single JSON object containing an `items` array, which holds all JIRA issues  
  - Inputs: Output from Merge node  
  - Outputs: Single JSON object with `items` array  
  - Failure Modes: Empty input, incorrect data structure  

#### 1.4 AI Processing

**Overview:**  
This block generates the formal release notes in HTML format based on the JIRA issue data using LangChain with Google Gemini AI model.

**Nodes Involved:**  
- Google Gemini Chat Model  
- Structured Output Parser1  
- Basic LLM Chain  

**Node Details:**  

- **Google Gemini Chat Model**  
  - Type: LangChain Language Model (Google Gemini)  
  - Configuration: Uses Google Gemini model "models/gemini-2.5-pro" for natural language generation  
  - Inputs: Receives prompt and data from Basic LLM Chain (as AI language model backend)  
  - Credentials: Google PaLM API key (Google Gemini)  
  - Failure Modes: API quota limits, latency, malformed prompts  
  - Version: 1  

- **Structured Output Parser1**  
  - Type: LangChain Structured Output Parser  
  - Configuration: Defines expected JSON schema with a string property "releasenote"  
  - Inputs: AI model raw output  
  - Outputs: Parsed JSON with release note HTML content  
  - Failure Modes: Parsing errors if AI output deviates from schema  

- **Basic LLM Chain**  
  - Type: LangChain Chain Node  
  - Configuration:  
    - Uses a human message prompt template instructing the AI to generate HTML-formatted release notes with a structured format (title, overview, key changes) based on JIRA items data  
    - Inputs data as JSON stringified `items` array with jira_id, jira_summary, jira_description, and commit message  
    - Outputs parsed by Structured Output Parser1  
  - Inputs: Output from Code3 (aggregated JIRA items)  
  - Outputs: Parsed release note HTML as `releasenote` field  
  - Failure Modes: Prompt misinterpretation, AI hallucination, empty input data  
  - Version: 1.7  

#### 1.5 Email Dispatch

**Overview:**  
Sends the generated release notes as an HTML email to a predefined recipient list.

**Nodes Involved:**  
- Send email  

**Node Details:**  

- **Send email**  
  - Type: Email Send  
  - Configuration:  
    - Sends HTML content from `releasenote` field as email body  
    - Subject includes company name and location  
    - Recipient and sender emails are hardcoded or set via parameters  
  - Inputs: Output from Basic LLM Chain (release note HTML)  
  - Credentials: SMTP account with authentication  
  - Failure Modes: SMTP connection failures, invalid email addresses, email formatting issues  
  - Version: 2.1  

---

### 3. Summary Table

| Node Name               | Node Type                      | Functional Role                          | Input Node(s)         | Output Node(s)            | Sticky Note                                       |
|-------------------------|--------------------------------|----------------------------------------|-----------------------|---------------------------|--------------------------------------------------|
| Github Trigger          | GitHub Trigger (Webhook)        | Listens for GitHub push events         | -                     | Code                      |                                                  |
| Code                    | Code (JavaScript)               | Extracts commit data and JIRA IDs      | Github Trigger        | Get an issue              |                                                  |
| Get an issue            | JIRA                           | Retrieves JIRA issue details            | Code                  | Code2                     |                                                  |
| Code2                   | Code (JavaScript)               | Extracts key JIRA fields                | Get an issue          | Merge                     |                                                  |
| Merge                   | Merge                          | Combines multiple inputs into one      | Get an issue, Code2   | Code3                     |                                                  |
| Code3                   | Code (JavaScript)               | Aggregates all issues into single array| Merge                 | Basic LLM Chain           |                                                  |
| Basic LLM Chain          | LangChain Chain Node            | Generates HTML release notes using AI  | Code3                 | Send email                |                                                  |
| Structured Output Parser1| LangChain Output Parser         | Parses AI output to JSON                | Google Gemini Chat Model| Basic LLM Chain           |                                                  |
| Google Gemini Chat Model | LangChain Language Model (AI)  | Provides AI text generation             | Basic LLM Chain       | Structured Output Parser1 |                                                  |
| Send email              | Email Send                     | Emails generated release notes          | Basic LLM Chain       | -                         |                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create GitHub Trigger node:**  
   - Type: GitHub Trigger  
   - Set event to "push" for specific repository and owner  
   - Configure GitHub OAuth credentials  
   - Ensure SSL enabled (insecureSSL: false)  

2. **Add Code node (Extract JIRA IDs):**  
   - Connect GitHub Trigger output to this node  
   - Use JavaScript code to extract commits and regex match JIRA IDs (`/([A-Z]+-\d+)/i`) from commit messages  
   - Output array with `message`, `timestamp`, and `jira_id` (uppercased or null)  

3. **Add JIRA node (Get an issue):**  
   - Connect Code node output  
   - Set operation to "get"  
   - Use dynamic expression `={{ $json.jira_id }}` for issueKey parameter  
   - Configure JIRA Software Cloud credentials (OAuth or API token)  

4. **Add Code node (Clean JIRA data):**  
   - Connect Get an issue output  
   - Extract and output `jira_id`, `jira_summary`, `jira_description` with optional chaining  
   - Use JavaScript code for mapping input items  

5. **Add Merge node:**  
   - Connect outputs of Get an issue and Code2 nodes to Merge inputs  
   - Set mode to "combine" and combineBy to "position"  

6. **Add Code node (Aggregate issues):**  
   - Connect Merge output  
   - Consolidate all input items into one object with `items` array containing all JIRA issue objects  

7. **Add Basic LLM Chain node:**  
   - Connect Code3 output  
   - Configure prompt as:  
     - Instruct AI to generate professional, HTML-formatted release notes for CxOs and clients  
     - Include title, metadata, overview, and bullet list of key changes based on JIRA items  
     - Provide input data as JSON stringified `items` array  
   - Use LangChain v1.7+  

8. **Add Google Gemini Chat Model node:**  
   - Connect Basic LLM Chain as AI language model backend  
   - Set modelName to "models/gemini-2.5-pro"  
   - Configure Google PaLM API credentials  

9. **Add Structured Output Parser node:**  
   - Connect Google Gemini Chat Model output  
   - Define manual JSON schema with a string property `releasenote` to parse AI output  

10. **Add Send email node:**  
    - Connect Basic LLM Chain output (with parsed release note HTML)  
    - Configure SMTP credentials  
    - Set email parameters:  
      - Subject with company/location info  
      - From and To email addresses  
      - HTML body from `={{ $json.output.releasenote }}`  

11. **Set node connections as per the described flow:**  
    - Github Trigger → Code → Get an issue → Code2 → Merge → Code3 → Basic LLM Chain → Send email  
    - Basic LLM Chain uses Google Gemini Chat Model and Structured Output Parser in LangChain AI chain  

12. **Configure workflow settings:**  
    - Enable manual and error execution data saving  
    - Set execution order to v1  

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                      |
|------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| The release notes output is in clean HTML with simple inline styles for readability.           | Ensures compatibility with email clients           |
| Use regex `/([A-Z]+-\d+)/i` to match JIRA issue keys in commit messages.                        | Critical for correct JIRA issue extraction          |
| Google Gemini AI model "models/gemini-2.5-pro" is used for high-quality natural language output.| Requires Google PaLM API credentials                 |
| For SMTP, ensure the server supports HTML emails and credentials are up-to-date.               | Avoid email delivery failures                         |
| Workflow saves execution progress and data for both successful and error executions.            | Useful for debugging and audits                       |

---

**Disclaimer:** The text provided is generated exclusively from an automated workflow created with n8n, a workflow automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.