Detect Team Burnout with Groq AI Analysis of GitHub Activity for Wellness Reports

https://n8nworkflows.xyz/workflows/detect-team-burnout-with-groq-ai-analysis-of-github-activity-for-wellness-reports-9517


# Detect Team Burnout with Groq AI Analysis of GitHub Activity for Wellness Reports

### 1. Workflow Overview

This n8n workflow titled **"Detect Team Burnout with Groq AI Analysis of GitHub Activity for Wellness Reports"** is designed to monitor and analyze software development team activity on GitHub to detect possible burnout risks and provide actionable wellness insights. It targets engineering managers, team leads, and organizational wellness officers aiming to maintain healthy developer workloads and prevent burnout through objective, data-driven analysis.

The workflow is structured into the following logical blocks:

- **1.1 Configuration & Scheduling**: Defines repository parameters and triggers periodic execution.
- **1.2 Data Acquisition from GitHub**: Fetches commits, pull requests, and workflow run data from GitHub repositories.
- **1.3 Data Analysis**: Processes raw data to identify activity patterns indicating workload intensity and signs of burnout.
- **1.4 AI-Powered Wellness Assessment**: Uses a Langchain AI agent with Groq’s large language model to generate a structured wellness report and recommendations.
- **1.5 Reporting & Notification**: Updates a GitHub issue with findings and sends an email report to stakeholders.
- **1.6 Control & Error Handling**: Includes no-operation nodes for flow control and potential expansion.

---

### 2. Block-by-Block Analysis

#### 2.1 Configuration & Scheduling

- **Overview:**  
  This block initializes workflow parameters such as the target GitHub repository and analysis period. It also sets the workflow to run automatically on a weekly schedule.

- **Nodes Involved:**  
  - Config  
  - Schedule Trigger

- **Node Details:**

  - **Config**  
    - Type: Set node  
    - Role: Stores static configuration data including the GitHub repo owner, repo name, analysis period (in days), and report email address.  
    - Configuration: JSON with keys `repoowner`, `reponame`, `period` (default 7 days), and `emailreport`.  
    - Inputs: None (starting node)  
    - Outputs: Connects to "Github Get Workflows" node.  
    - Edge Cases: Ensure valid repo owner and repo name; invalid values cause API failures downstream.

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Triggers workflow execution every 7 days to perform periodic analysis.  
    - Configuration: Interval set to 7 days.  
    - Inputs: None  
    - Outputs: Connects to "Config" node.  
    - Edge Cases: None specific, but workflow execution depends on schedule.

---

#### 2.2 Data Acquisition from GitHub

- **Overview:**  
  Fetches detailed GitHub activity data — workflow runs, commits, and pull requests — from the configured repository for the analysis period.

- **Nodes Involved:**  
  - Github Get Workflows  
  - Github Get Commits  
  - Get Prs

- **Node Details:**

  - **Github Get Workflows**  
    - Type: HTTP Request  
    - Role: Retrieves GitHub Actions workflow runs for the repo within the analysis period.  
    - Configuration: Calls GitHub API `/actions/runs` endpoint, filtered by creation date and limited to 50 results.  
    - Credentials: GitHub API OAuth token.  
    - Inputs: From "Config" node with repository and period parameters.  
    - Outputs: To "Github Get Commits".  
    - Edge Cases: API rate limits, repository permissions, zero workflows returned.

  - **Github Get Commits**  
    - Type: HTTP Request  
    - Role: Fetches commit history for the repository filtered by creation date over the analysis period.  
    - Configuration: Calls GitHub API `/commits` endpoint with `created` date range filter.  
    - Credentials: GitHub API OAuth token.  
    - Inputs: From "Github Get Workflows" outputs repository info.  
    - Outputs: To "Get Prs".  
    - Edge Cases: Large commit volumes may affect performance; API limits apply.

  - **Get Prs**  
    - Type: GitHub node (built-in)  
    - Role: Retrieves all pull requests (open, closed, merged) from the repository.  
    - Configuration: Uses GitHub node to list pull requests with `state=all` and descending order by creation date.  
    - Credentials: GitHub API OAuth token.  
    - Inputs: From "Github Get Commits".  
    - Outputs: To "Analyze Patterns Developer".  
    - Edge Cases: Pagination if many PRs; permissions.

---

#### 2.3 Data Analysis

- **Overview:**  
  Processes fetched commits, pull requests, and workflows to calculate team activity metrics and detect patterns such as late-night commits, weekend work, failed workflows, and developer-specific activity summaries.

- **Nodes Involved:**  
  - Analyze Patterns Developer

- **Node Details:**

  - **Analyze Patterns Developer**  
    - Type: Code (JavaScript)  
    - Role: Parses data arrays and computes metrics on commit timing, counts, and failures; aggregates developer activity statistics.  
    - Configuration: Custom JS code analyzing commits for late-night (10pm–6am) and weekend (Sat-Sun) commits, failed workflows count, and developer-wise activity breakdown. Calculates rates for failure, late-night, and weekend commits.  
    - Inputs: Receives arrays for commits, pull requests, and workflows from previous nodes.  
    - Outputs: JSON object summarizing all computed patterns and rates, passed to AI Agent node.  
    - Edge Cases: Handles missing commit author names; logs errors during commit processing; resilient to empty datasets.  
    - Version Requirements: Uses ES6+ JavaScript features supported by n8n code node.

---

#### 2.4 AI-Powered Wellness Assessment

- **Overview:**  
  Invokes a Langchain AI agent powered by Groq’s GPT-OSS 120B model to evaluate the analyzed data, generate an objective team health report in markdown/html format, and produce recommendations. The agent respects strict guidelines to ensure professionalism, privacy, and actionable output.

- **Nodes Involved:**  
  - AI Agent  
  - Groq Chat Model Report

- **Node Details:**

  - **AI Agent**  
    - Type: Langchain Agent node  
    - Role: Receives analyzed data JSON and instructions; generates a structured wellness report with confidence scores, observations, systemic risks, and recommendations.  
    - Configuration: Prompt includes explicit guardrails restricting personal judgments, emphasizing evidence-only analysis, and output formatting instructions with example markdown report. Provides tools integration hints like updating GitHub issues and sending emails for critical alerts.  
    - Inputs: From "Analyze Patterns Developer".  
    - Outputs: To "No Operation, do nothing" (end node) and triggers AI tool outputs for issue update and email.  
    - Edge Cases: Requires precise prompt adherence; failures possible if data is incomplete or malformed.

  - **Groq Chat Model Report**  
    - Type: Langchain Language Model (Groq GPT-OSS 120B)  
    - Role: Provides the language model backend for the AI Agent node to perform natural language generation.  
    - Configuration: Uses Groq API credentials; model is set to openai/gpt-oss-120b.  
    - Inputs: Accepts prompt from AI Agent node.  
    - Outputs: Feeds back into the AI Agent node’s outputs.  
    - Edge Cases: API rate limits, network failures, model latency.

---

#### 2.5 Reporting & Notification

- **Overview:**  
  Updates a GitHub issue with the generated wellness report and sends an email notification to the configured stakeholder email address when critical alerts are detected.

- **Nodes Involved:**  
  - Update Github Issue  
  - Send a message in Gmail

- **Node Details:**

  - **Update Github Issue**  
    - Type: GitHub Tool node  
    - Role: Creates or updates a GitHub issue in the repo with the AI-generated wellness report content.  
    - Configuration: Title and body are populated dynamically from AI Agent outputs; no labels or assignees specified.  
    - Credentials: GitHub API OAuth token.  
    - Inputs: AI Agent node’s AI tool output.  
    - Outputs: None (terminal node for issue update).  
    - Edge Cases: Permissions to create issues required; issue duplication if multiple runs not handled.

  - **Send a message in Gmail**  
    - Type: Gmail Tool node  
    - Role: Sends an email with the generated wellness report to the configured email address.  
    - Configuration: Recipient email is set from Config node; subject fixed as “📊 Team Health and Wellness Report”; message body populated from AI Agent output.  
    - Credentials: Gmail OAuth2 credentials.  
    - Inputs: AI Agent node’s AI tool output.  
    - Outputs: None (terminal node for email).  
    - Edge Cases: Gmail API quota, authentication expiry.

---

#### 2.6 Control & Error Handling

- **Overview:**  
  Contains a no-operation node to finalize the flow and placeholders for potential error handling or conditional branching.

- **Nodes Involved:**  
  - No Operation, do nothing

- **Node Details:**

  - **No Operation, do nothing**  
    - Type: NoOp node  
    - Role: Acts as a sink node to safely terminate workflow branches.  
    - Configuration: None required.  
    - Inputs: From AI Agent node’s main output.  
    - Outputs: None.  
    - Edge Cases: None.

---

### 3. Summary Table

| Node Name             | Node Type                      | Functional Role                        | Input Node(s)             | Output Node(s)               | Sticky Note                                                                                                                                                                                                                                                       |
|-----------------------|--------------------------------|-------------------------------------|---------------------------|-----------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger       | Schedule Trigger               | Triggers workflow weekly             | None                      | Config                      |                                                                                                                                                                                                                                                                  |
| Config                | Set                            | Holds configuration parameters       | Schedule Trigger          | Github Get Workflows         |                                                                                                                                                                                                                                                                  |
| Github Get Workflows   | HTTP Request                   | Fetches GitHub workflow runs         | Config                    | Github Get Commits           |                                                                                                                                                                                                                                                                  |
| Github Get Commits     | HTTP Request                   | Fetches commits from GitHub          | Github Get Workflows      | Get Prs                     |                                                                                                                                                                                                                                                                  |
| Get Prs                | GitHub                        | Fetches pull requests                 | Github Get Commits        | Analyze Patterns Developer   |                                                                                                                                                                                                                                                                  |
| Analyze Patterns Developer | Code (JavaScript)             | Analyzes activity patterns            | Get Prs                   | AI Agent                    |                                                                                                                                                                                                                                                                  |
| AI Agent               | Langchain Agent                | Generates wellness report via AI     | Analyze Patterns Developer | No Operation, do nothing; Update Github Issue; Send a message in Gmail |                                                                                                                                                                                                                                                                  |
| Groq Chat Model Report | Langchain Language Model (Groq GPT-OSS 120B) | AI language model backend            | AI Agent (ai_languageModel) | AI Agent (ai_languageModel) |                                                                                                                                                                                                                                                                  |
| Update Github Issue    | GitHub Tool                    | Creates/updates GitHub issue          | AI Agent (ai_tool)         | None                        |                                                                                                                                                                                                                                                                  |
| Send a message in Gmail | Gmail Tool                    | Sends email report                    | AI Agent (ai_tool)         | None                        |                                                                                                                                                                                                                                                                  |
| No Operation, do nothing | No Operation (NoOp)           | Workflow flow control sink            | AI Agent (main)            | None                        |                                                                                                                                                                                                                                                                  |
| Sticky Note            | Sticky Note                   | Visual comments                      | None                      | None                        |                                                                                                                                                                                                                                                                  |
| Sticky Note15          | Sticky Note                   | Contains project overview, demo links, and setup instructions | None                      | None                        | # Team-Wellness :  AI Burnout Detector Agent [devex]\n\n## Demo \n* [github action code alternative ]( https://github.com/suarifymy/adk-samples/blob/main/.github/workflows/devex-ai-burnout-detector.yml)\n* [sample report ](https://github.com/suarifymy/adk-samples/issues/1/)\n \n\n## How it works \nPeriodically, there will be a job to fetch GitHub Commits  , PRs, Active Flows. Then the llm ai agent analyzes total comits, late night commits, weekend commmits , failed workflow and developer's activity and work intensity patterns. Lastly, creates a github issues and sends an email\n\n \n## Setup\n   - Follow setup link [n8n-github-account-setup](\nhttps://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.github/) , [n8n-groq-setup](https://docs.n8n.io/integrations/builtin/credentials/groq/)  ,[n8n-gmail-setup](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail/) \n- Change the `config` node |

---

### 4. Reproducing the Workflow from Scratch

1. **Create “Schedule Trigger” node**  
   - Type: Schedule Trigger  
   - Set interval to run every 7 days (weekly).  
   - No credentials needed.

2. **Create “Config” node**  
   - Type: Set  
   - Mode: Raw JSON  
   - JSON content example:  
     ```json
     {
       "repoowner": "suarifymy",
       "reponame": "adk-samples",
       "period": 7,
       "emailreport": "aiix.space.noreply@gmail.com"
     }
     ```  
   - Connect Schedule Trigger output to Config input.

3. **Create “Github Get Workflows” node**  
   - Type: HTTP Request  
   - URL:  
     ```
     https://api.github.com/repos/{{ $json.repoowner }}/{{ $json.reponame }}/actions/runs
     ```  
   - Query parameters:  
     - `created`: from `{{$now.minus({ days: $json.period || 7 }).toISO()}}..*`  
     - `per_page`: 50  
   - Credentials: Use GitHub OAuth credentials with repo read access.  
   - Connect Config output to this node’s input.

4. **Create “Github Get Commits” node**  
   - Type: HTTP Request  
   - URL:  
     ```
     https://api.github.com/repos/{{ $json.repoowner }}/{{ $json.reponame }}/commits
     ```  
   - Query parameters:  
     - `created`: from `{{$now.minus({ days: $json.period || 7 }).toISO()}}..*`  
   - Credentials: Use same GitHub OAuth credentials.  
   - Connect “Github Get Workflows” output to this node.

5. **Create “Get Prs” node**  
   - Type: GitHub node (GitHub API)  
   - Operation: Get Pull Requests  
   - Repository owner: `{{$json.repoowner}}`  
   - Repository name: `{{$json.reponame}}`  
   - Filters: state = all, direction = desc  
   - Credentials: GitHub OAuth credentials.  
   - Connect “Github Get Commits” output to this node.

6. **Create “Analyze Patterns Developer” node**  
   - Type: Code (JavaScript)  
   - Paste the provided JS code to process commits, pull requests, and workflows to compute activity metrics (late-night commits, weekend commits, failure rate, and developer statistics).  
   - Inputs: From “Get Prs” node (and implicitly expects data from commits and workflows nodes as referenced in code).  
   - Connect “Get Prs” output to this node.

7. **Create “Groq Chat Model Report” node**  
   - Type: Langchain Language Model node  
   - Model: `openai/gpt-oss-120b` (Groq API)  
   - Credentials: Groq API credentials.  
   - Connect this node as ai_languageModel input to the AI Agent node.

8. **Create “AI Agent” node**  
   - Type: Langchain Agent  
   - Configure prompt with detailed instructions enforcing professional, evidence-based burnout analysis and output format, referencing “Analyze Patterns Developer” output JSON.  
   - Configure tools integration for GitHub issue update and email sending.  
   - Connect “Analyze Patterns Developer” output to AI Agent main input.  
   - Connect “Groq Chat Model Report” node as ai_languageModel input to AI Agent.

9. **Create “Update Github Issue” node**  
   - Type: GitHub Tool node  
   - Operation: Create or update an issue  
   - Owner and Repository: Use expressions from Config node values.  
   - Title and Body: Dynamically filled from AI Agent’s AI tool output.  
   - Credentials: GitHub OAuth credentials.  
   - Connect AI Agent ai_tool output to this node.

10. **Create “Send a message in Gmail” node**  
    - Type: Gmail Tool node  
    - Send To: Email address from Config node JSON.  
    - Subject: Fixed string “📊 Team Health and Wellness Report”  
    - Message Body: From AI Agent’s AI tool output.  
    - Credentials: Gmail OAuth2 credentials.  
    - Connect AI Agent ai_tool output to this node.

11. **Create “No Operation, do nothing” node**  
    - Type: No Operation node  
    - Connect AI Agent main output to this node to safely terminate the workflow.

12. **Connect all nodes as per above connection flow:**  
    Schedule Trigger → Config → Github Get Workflows → Github Get Commits → Get Prs → Analyze Patterns Developer → AI Agent → { Update Github Issue, Send a message in Gmail, No Operation }

13. **Test the workflow with valid GitHub and Gmail credentials and verify outputs (issue creation, email receipt, and logs).**

---

### 5. General Notes & Resources

| Note Content                                                                                                            | Context or Link                                                                                                         |
|-------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| Demo and sample report links: [GitHub Action alternative](https://github.com/suarifymy/adk-samples/blob/main/.github/workflows/devex-ai-burnout-detector.yml), [Sample report](https://github.com/suarifymy/adk-samples/issues/1/) | Sticky Note15 content in workflow provides these useful references.                                                     |
| Setup guides for integrations: [n8n GitHub node setup](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.github/), [Groq credential setup](https://docs.n8n.io/integrations/builtin/credentials/groq/), [Gmail node setup](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail/) | Sticky Note15 content includes these links to configure credentials and nodes correctly.                               |
| Analysis respects privacy by redacting developer names and avoids personal judgments or alarmist language.              | Enforced by AI Agent prompt instructions.                                                                              |
| The workflow depends on GitHub API rate limits and OAuth credentials with sufficient permissions.                       | Ensure OAuth tokens have repo and workflow read/write access.                                                          |
| Email notifications and GitHub issue updates are only triggered for critical alerts (health score below 90) as per prompt. | AI Agent enforces conditional notification rules.                                                                       |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.