Curate Contributor-Friendly Issues with AI and Send GitHub Newsletter via Email

https://n8nworkflows.xyz/workflows/curate-contributor-friendly-issues-with-ai-and-send-github-newsletter-via-email-11549


# Curate Contributor-Friendly Issues with AI and Send GitHub Newsletter via Email

---
### 1. Workflow Overview

This n8n workflow automates the creation and delivery of a curated email newsletter featuring contributor-friendly issues from a specified GitHub repository. It is designed to help open-source enthusiasts discover and engage with high-potential issues suitable for new contributors, alongside summaries of recent releases. The workflow leverages GitHub's GraphQL API, multiple AI models for issue analysis and summarization, and Deepwiki's MCP tool for enriched insights, ultimately formatting the information into an HTML email and sending it.

The workflow is logically divided into the following functional blocks:

- **1.1 Input Reception and Repository Data Loading:** Entry point and repository identification.
- **1.2 GitHub Data Retrieval:** Fetching latest releases and open issues using GitHub GraphQL API.
- **1.3 AI-Based Issue Ranking and Selection:** Using AI to evaluate and rank issues based on contributor-friendliness.
- **1.4 Deepwiki MCP Analysis:** Deep semantic analysis of selected issues to generate detailed contribution guidance.
- **1.5 Newsletter Title Generation:** AI-based creation of an engaging newsletter title.
- **1.6 HTML Formatting and Email Dispatch:** Converting data into an HTML email and sending it via SMTP.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Repository Info Loading

- **Overview:**  
  This block starts the workflow upon manual trigger and defines the target GitHub repository by specifying the owner and repository name.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Load Repo Info (Code node)

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Initiates workflow execution manually.  
    - Config: No parameters; simply triggers on user command.  
    - Inputs: None  
    - Outputs: Starts flow to Load Repo Info node  
    - Edge cases: None specific; relies on manual execution.

  - **Load Repo Info**  
    - Type: Code node (JavaScript)  
    - Role: Provides repository owner and name as JSON for downstream queries.  
    - Config: Hardcoded values for owner (`vercel`) and repo name (`next.js`).  
    - Key variables: `{ owner: "vercel", name: "next.js" }`  
    - Input: Trigger from manual node  
    - Output: JSON with repo info for GitHub query  
    - Edge cases: None, but must be updated per target repository.

#### 2.2 GitHub Data Retrieval

- **Overview:**  
  Queries GitHub's GraphQL API to retrieve the latest release and the 10 most recent open issues, including metadata, labels, comments, and author information.

- **Nodes Involved:**  
  - Get Issue From Github (GraphQL node)

- **Node Details:**

  - **Get Issue From Github**  
    - Type: GraphQL node  
    - Role: Fetches repository data including issues and latest release info.  
    - Config:  
      - Query targets repository by owner and name from Load Repo Info.  
      - Retrieves latest release (1, ordered by creation date descending) with name, description, createdAt.  
      - Retrieves first 10 open issues ordered by creation date descending with detailed fields such as title, url, body, author login, labels, comments, and issue type.  
    - Authentication: Uses GitHub Personal Access Token via header authentication.  
    - Input: Repo info JSON  
    - Output: JSON containing repository, releases, and issues data.  
    - Edge cases:  
      - GitHub token expiration or insufficient scopes lead to auth errors.  
      - API rate limits may cause query failures or partial data.  
      - Repository not found or private repo without access.

#### 2.3 AI-Based Issue Ranking and Selection

- **Overview:**  
  Uses AI to analyze fetched issues and latest release information to classify issues by contributor-friendliness, selecting the top 3 most suitable issues for contribution.

- **Nodes Involved:**  
  - IssueRank Agent (LangChain Agent node)  
  - IssueRank Structured Output Parser (Output Parser)  
  - IssueRank Main Model (LM Chat Open Router)  
  - IssueRank Fallback Model (LM Chat Open Router)  
  - get Top Fit Issues (Code node)  
  - Split Out (SplitOut node)

- **Node Details:**

  - **IssueRank Agent**  
    - Type: LangChain Agent node  
    - Role: AI agent that receives issues and latest release data, outputs a YAML object classifying issues by suitability (high, medium, low) with reasons.  
    - Config:  
      - System message defines criteria for "good contribution opportunities" (e.g., detailed issues, official maintainer involvement, good-first-issue label, etc.)  
      - Input prompt includes GitHub issues and release notes from previous node.  
      - Output is structured YAML matching a TypeScript type describing release and issues.  
    - Models: Uses IssueRank Main Model with fallback IssueRank Fallback Model for robustness.  
    - Input: JSON data from GitHub query  
    - Output: Parsed structured data with suitability rankings  
    - Edge cases:  
      - AI model timeout or rate limits  
      - Parsing errors if AI output does not match expected YAML schema  
      - Insufficient or ambiguous issue data may reduce classification accuracy

  - **IssueRank Structured Output Parser**  
    - Type: Output Parser (JSON Schema validation)  
    - Role: Validates and auto-fixes AI output structure  
    - Input: Raw AI output from IssueRank Agent  
    - Output: Clean JSON with latestRelease and issues array

  - **get Top Fit Issues**  
    - Type: Code node  
    - Role: Sorts issues by suitability level (high > medium > low) and selects top 3 issues  
    - Config: Uses a custom order mapping for levels to sort and slices top 3  
    - Input: Output from IssueRank Agent parsed data  
    - Output: JSON with filtered top 3 issues  
    - Edge cases: If fewer than 3 suitable issues are present, it returns as many as available.

  - **Split Out**  
    - Type: SplitOut node  
    - Role: Separates the array of top issues into individual items for parallel processing  
    - Input: Top 3 issues JSON  
    - Output: Individual issue objects for deep analysis and merging later

#### 2.4 Deepwiki MCP Analysis

- **Overview:**  
  For each selected issue, calls Deepwiki's MCP tool to generate detailed contribution guidance including root cause, resolution approach, technical difficulty, keywords, analogy, and summary.

- **Nodes Involved:**  
  - Deepwiki Agent (LangChain Agent node)  
  - Deepwiki MCP (MCP Client Tool node)  
  - Deepwiki Structured Output Parser (Output Parser)  
  - Deepwiki Main Model (LM Chat Open Router)  
  - Deepwiki Fallback Model (LM Chat Open Router)  
  - Deepwiki Parser Model (LM Chat Open Router)

- **Node Details:**

  - **Deepwiki Agent**  
    - Type: LangChain Agent node  
    - Role: Acts as an AI assistant to analyze each issue in detail using Deepwiki MCP tool.  
    - Config:  
      - System message enforces tool usage for repository-related questions.  
      - Batch size of 3 issues with 5 seconds delay between batches to control API usage.  
      - Calls MCP tool `ask_question` with constructed question combining issue title and body, requesting root cause, resolution, difficulty, and analogy.  
      - Outputs a YAML matching a detailed Issue type with multiple fields for contribution guidance.  
    - Models: Deepwiki Main Model with fallback Deepwiki Fallback Model  
    - Input: Individual issues from Split Out node  
    - Output: Detailed issue analysis JSON  
    - Edge cases:  
      - MCP tool timeout or failure (configured 60s timeout)  
      - AI model or parser errors  
      - Missing or incomplete issue data affecting analysis quality

  - **Deepwiki MCP**  
    - Type: MCP Client Tool node  
    - Role: External API client to Deepwiki MCP service for semantic question answering  
    - Config: Endpoint URL set to `https://mcp.deepwiki.com/mcp`, timeout 60000ms  
    - Input: Calls from Deepwiki Agent  
    - Output: Tool response with detailed issue info

  - **Deepwiki Structured Output Parser**  
    - Type: Output Parser with strict JSON schema validation  
    - Role: Parses and validates Deepwiki Agent output to ensure required fields are present  
    - Input: Raw Deepwiki Agent output  
    - Output: Structured issue analysis data

#### 2.5 Newsletter Title Generation

- **Overview:**  
  Generates a catchy, emoji-prefixed newsletter title based on the top selected issue to increase email open rate.

- **Nodes Involved:**  
  - Title Generator Agent (LangChain Agent node)  
  - Title Generator Main Model (LM Chat Open Router)  
  - Title Generator Fallback Model (LM Chat Open Router)  
  - Merge1 (Merge node)

- **Node Details:**

  - **Title Generator Agent**  
    - Type: LangChain Agent node  
    - Role: Creates a short, fun, clickbait-style newsletter title using the first issue’s data.  
    - Config:  
      - System message instructs to include a relevant emoji at the start, keep title understandable for non-technical users, and translate to en-US.  
      - Output is a single text line (no markdown or extra formatting).  
    - Models: Uses Title Generator Main Model with fallback model.  
    - Input: The top fitted issues array from get Top Fit Issues node (first issue used)  
    - Output: Generated newsletter title text  
    - Edge cases: AI model fallback on failures; if no issues, title generation may fail.

  - **Merge1**  
    - Type: Merge node (combine by position)  
    - Role: Combines the generated title with issue data for final processing  
    - Input: Title Generator Agent output and other issue data flows  
    - Output: Combined data for HTML conversion

#### 2.6 HTML Formatting and Email Dispatch

- **Overview:**  
  Converts the enriched issue and release data into an HTML newsletter and sends it via email using SMTP credentials.

- **Nodes Involved:**  
  - convert HTML (Code node)  
  - Send email (Email Send node)

- **Node Details:**

  - **convert HTML**  
    - Type: Code node (JavaScript)  
    - Role: Generates full HTML email content embedding latest release info, quick summaries, and detailed issue sections.  
    - Config:  
      - Uses multiple helper functions to build bulleted lists and structured HTML tables.  
      - Embeds images and styles ensuring compatibility with email clients (including MS Outlook).  
      - Populates fields such as issue title, keywords, analogy, root cause, resolution steps, technical difficulty, and links to GitHub and Deepwiki.  
      - Includes example data structure internally for reference and formatting examples.  
    - Input: Merged data from previous node with title and issues analyses  
    - Output: JSON containing `html` property with full email HTML content  
    - Edge cases: Must have all required fields populated to avoid rendering issues.

  - **Send email**  
    - Type: Email Send node  
    - Role: Sends the generated HTML newsletter to the configured recipient via SMTP.  
    - Config:  
      - Subject line dynamically includes repository owner, name, and newsletter title.  
      - To and From emails must be set to the authenticated Google email account (using Google App Password).  
      - Supports default SMTP options without special overrides.  
    - Credentials: Requires Google App Password for SMTP authentication.  
    - Input: HTML content and subject line from convert HTML and Load Repo Info nodes.  
    - Output: Email sent confirmation or error  
    - Edge cases:  
      - Authentication failures with incorrect credentials or expired app passwords.  
      - Email delivery failures due to network issues or spam filtering.

---

### 3. Summary Table

| Node Name                   | Node Type                          | Functional Role                                      | Input Node(s)                   | Output Node(s)                        | Sticky Note                                                                                                                   |
|-----------------------------|----------------------------------|----------------------------------------------------|--------------------------------|-------------------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                   | Manual start of workflow                            | None                           | Load Repo Info                      |                                                                                                                               |
| Load Repo Info              | Code                             | Provides target GitHub repo owner and name         | When clicking ‘Execute workflow’ | Get Issue From Github               |                                                                                                                               |
| Get Issue From Github       | GraphQL                          | Fetch latest release and open issues from GitHub   | Load Repo Info                 | IssueRank Agent                    |                                                                                                                               |
| IssueRank Agent             | LangChain Agent                  | Classify issues by contributor-friendliness         | Get Issue From Github           | get Top Fit Issues                 | ## 1. Selecting Contributor-Friendly Issues: Retrieves issue data and identifies good contribution opportunities.              |
| IssueRank Structured Output Parser | Output Parser                   | Validates AI output structure                        | IssueRank Agent               | get Top Fit Issues                 |                                                                                                                               |
| IssueRank Main Model        | LM Chat Open Router              | Primary AI model for IssueRank Agent                | IssueRank Agent               | IssueRank Agent                   |                                                                                                                               |
| IssueRank Fallback Model    | LM Chat Open Router              | Fallback AI model for IssueRank Agent               | IssueRank Agent               | IssueRank Agent                   |                                                                                                                               |
| get Top Fit Issues          | Code                             | Sorts and selects top 3 contributor-friendly issues | IssueRank Structured Output Parser | Split Out, Title Generator Agent   |                                                                                                                               |
| Split Out                  | SplitOut                         | Splits top issues array to individual issues        | get Top Fit Issues             | Deepwiki Agent, Merge              | ## 2-2. Summarizing Contribution Details: Generates info on contribution methods and levels using Deepwiki MCP.               |
| Deepwiki Agent              | LangChain Agent                  | Detailed issue analysis using Deepwiki MCP           | Split Out                     | Merge                            |                                                                                                                               |
| Deepwiki MCP               | MCP Client Tool                  | Calls Deepwiki MCP API for semantic question answering | Deepwiki Agent               | Deepwiki Agent                   |                                                                                                                               |
| Deepwiki Structured Output Parser | Output Parser                   | Parses and validates Deepwiki AI output             | Deepwiki Agent               | Merge                            |                                                                                                                               |
| Deepwiki Main Model         | LM Chat Open Router              | Primary AI model for Deepwiki Agent                  | Deepwiki Agent               | Deepwiki Agent                   |                                                                                                                               |
| Deepwiki Fallback Model     | LM Chat Open Router              | Fallback AI model for Deepwiki Agent                 | Deepwiki Agent               | Deepwiki Agent                   |                                                                                                                               |
| Deepwiki Parser Model       | LM Chat Open Router              | Parses Deepwiki output                                | Deepwiki Structured Output Parser | Merge                            |                                                                                                                               |
| Title Generator Agent       | LangChain Agent                  | Creates catchy newsletter title                       | get Top Fit Issues            | Merge1                           | ## 2-1. Generating a Newsletter Title: Creates a newsletter title based on issue content.                                      |
| Title Generator Main Model  | LM Chat Open Router              | Primary AI model for Title Generator                  | Title Generator Agent         | Title Generator Agent            |                                                                                                                               |
| Title Generator Fallback Model | LM Chat Open Router              | Fallback AI model for Title Generator                 | Title Generator Agent         | Title Generator Agent            |                                                                                                                               |
| Merge1                     | Merge                           | Combines title with issue data for HTML conversion   | Title Generator Agent, Split Out | convert HTML                    |                                                                                                                               |
| Merge                      | Merge                           | Combines Deepwiki analysis outputs                    | Deepwiki Agent, Split Out      | convert HTML                    |                                                                                                                               |
| convert HTML               | Code                             | Builds HTML email content from issues and release    | Merge1, Merge                 | Send email                     | ## 3. Converting Issue Data to HTML and Sending via email: Converts issue data to HTML and sends email.                        |
| Send email                 | Email Send                      | Sends the generated newsletter email                  | convert HTML                  | None                           |                                                                                                                               |
| Sticky Note                | Sticky Note                     | Documentation and workflow usage instructions         | None                         | None                           | ## Usecases: Receive contributor-friendly newsletters, setup instructions, and customization tips. See https://deepwiki.com/  |
| Sticky Note6               | Sticky Note                     | Block 1 explanation                                   | None                         | None                           | ## 1. Selecting Contributor-Friendly Issues: Retrieves issue data and identifies good contribution opportunities.              |
| Sticky Note7               | Sticky Note                     | Block 2-1 explanation                                 | None                         | None                           | ## 2-1. Generating a Newsletter Title: Creates a newsletter title based on the issue content.                                   |
| Sticky Note8               | Sticky Note                     | Block 2-2 explanation                                 | None                         | None                           | ## 2-2. Summarizing Contribution Details: Generates info on contribution methods and levels using Deepwiki MCP.               |
| Sticky Note9               | Sticky Note                     | Block 3 explanation                                   | None                         | None                           | ## 3. Converting Issue Data to HTML and Sending via email: Converts issue data to HTML and sends email.                        |
| Sticky Note2               | Sticky Note                     | Newsletter visual example                             | None                         | None                           | ## Newsletter: ![](https://lh3.googleusercontent.com/d/1iHWRiMm5mrWkA2r-t81Uzykqcss2Pl8G)                                    |
| Sticky Note3               | Sticky Note                     | Newsletter visual example                             | None                         | None                           | ## Newsletter: ![](https://lh3.googleusercontent.com/d/1Pfdp4mttjAM9E7rV_y5qDbhgqmunECF_)                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Execute workflow’`  
   - Type: Manual Trigger  
   - No parameters required.

2. **Create Code Node to Load Repository Info**  
   - Name: `Load Repo Info`  
   - Type: Code  
   - Add JavaScript code to output:  
     ```js
     return {
       owner: "vercel",
       name: "next.js"
     };
     ```  
   - Connect output from Manual Trigger.

3. **Create GitHub GraphQL Node to Fetch Issues and Release**  
   - Name: `Get Issue From Github`  
   - Type: GraphQL  
   - Configure Endpoint URL: `https://api.github.com/graphql`  
   - Set Authentication: GitHub Personal Access Token (header auth)  
   - Query (replace variables with expressions):  
     ```graphql
     query {
       repository(owner: "{{ $json.owner }}", name: "{{ $json.name }}") {
         releases(first: 1, orderBy: { field: CREATED_AT, direction: DESC }) {
           nodes {
             name
             description
             createdAt
           }
         }
         issues(first: 10, states: OPEN, orderBy: { field: CREATED_AT, direction: DESC }) {
           nodes {
             title
             url
             body
             author {
               ... on User {
                 login
               }
             }
             issueType {
               name
             }
             labels(first: 5) {
               edges {
                 node {
                   name
                 }
               }
             }
             comments(first: 10) {
               nodes {
                 body
               }
             }
           }
         }
       }
     }
     ```  
   - Connect input from `Load Repo Info`.

4. **Create LangChain Agent Node for Issue Ranking**  
   - Name: `IssueRank Agent`  
   - Type: LangChain Agent  
   - Set main AI model to OpenRouter model (e.g., `x-ai/grok-4.1-fast:free`)  
   - Set fallback AI model (e.g., `qwen/qwen3-coder:free`)  
   - Configure system prompt with contribution criteria (as per workflow)  
   - Use expressions to pass issues and release data from GitHub query node output  
   - Connect input from `Get Issue From Github`.

5. **Create Output Parser Node for IssueRank**  
   - Name: `IssueRank Structured Output Parser`  
   - Type: Output Parser Structured  
   - Configure JSON schema as per workflow to validate AI output  
   - Connect input from `IssueRank Agent`.

6. **Create Code Node to Select Top 3 Issues**  
   - Name: `get Top Fit Issues`  
   - Type: Code  
   - JavaScript code to sort by suitability level (high > medium > low) and slice top 3  
   - Connect input from `IssueRank Structured Output Parser`.

7. **Create SplitOut Node to Split Issues Array**  
   - Name: `Split Out`  
   - Type: SplitOut  
   - Field to split: `issues`  
   - Connect input from `get Top Fit Issues`.

8. **Create LangChain Agent Node for Deepwiki MCP Analysis**  
   - Name: `Deepwiki Agent`  
   - Type: LangChain Agent  
   - Set main and fallback AI models as used (same model as `IssueRank` but separate config)  
   - Configure to call Deepwiki MCP tool with constructed question string for each issue  
   - Batch processing: batch size 3, 5 seconds delay between batches  
   - Connect input from `Split Out`.

9. **Create MCP Client Tool Node**  
   - Name: `Deepwiki MCP`  
   - Type: MCP Client Tool  
   - Configure endpoint URL: `https://mcp.deepwiki.com/mcp`  
   - Connect to `Deepwiki Agent` as tool client.

10. **Create Output Parser Node for Deepwiki**  
    - Name: `Deepwiki Structured Output Parser`  
    - Type: Output Parser Structured  
    - Configure JSON schema for detailed issue analysis  
    - Connect input from `Deepwiki Agent`.

11. **Create Merge Node to Aggregate Deepwiki Outputs**  
    - Name: `Merge`  
    - Type: Merge  
    - Mode: Combine  
    - Merge by field: `issueURL`  
    - Connect input 0 from `Deepwiki Agent`, input 1 from `Split Out`.

12. **Create LangChain Agent Node for Newsletter Title Generation**  
    - Name: `Title Generator Agent`  
    - Type: LangChain Agent  
    - Set main and fallback AI models (same family as others)  
    - Configure prompt to generate catchy newsletter title with emoji, translated to en-US  
    - Input: first issue from `get Top Fit Issues` output  
    - Connect input from `get Top Fit Issues`.

13. **Create Merge Node to Combine Title and Issue Data**  
    - Name: `Merge1`  
    - Type: Merge  
    - Mode: Combine by position  
    - Connect inputs from `Title Generator Agent` and `Split Out` or `Merge`.

14. **Create Code Node to Convert Data to HTML Email**  
    - Name: `convert HTML`  
    - Type: Code  
    - Implement JavaScript function that builds HTML email body with embedded release info, summaries, and detailed issues  
    - Use supplied email-compatible styling and formatting  
    - Connect input from `Merge1`.

15. **Create Email Send Node**  
    - Name: `Send email`  
    - Type: Email Send  
    - Configure SMTP credentials with Google App Password  
    - Set "To Email" and "From Email" to the authenticated Google account email  
    - Subject line: format with repo owner, name, and newsletter title from `Title Generator Agent`  
    - Connect input from `convert HTML`.

16. **Final Connections and Testing**  
    - Connect all nodes as per above  
    - Test workflow with manual trigger  
    - Adjust repo info, API keys, and credentials as needed  
    - Optionally replace manual trigger with a Schedule Trigger for automation.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| This workflow enables receiving periodic newsletters of curated, contributor-friendly issues from GitHub repositories, enhancing open source contribution habits. | Sticky Note content: Usecases section with detailed instructions and setup requirements. |
| Requires GitHub Personal Access Token with appropriate scopes, OpenRouter API key for AI models, and Google App Password for sending emails. | GitHub PAT doc: https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#creating-a-fine-grained-personal-access-token; OpenRouter: https://openrouter.ai/settings/keys; Google App Password: https://myaccount.google.com/apppasswords |
| Target repositories should be indexed on Deepwiki for full analysis: https://deepwiki.com/{owner}/{repo} | Example: https://deepwiki.com/vercel/next.js |
| To automate workflow execution, replace manual trigger node with a Schedule Trigger node or other scheduling-capable node. | Alternative cron setup for non-n8n Cloud users: https://github.com/dst03106/n8n-issue-collector?tab=readme-ov-file#%EF%B8%8F-how-to-use |
| Newsletter examples and branding images are embedded within Sticky Notes for visualization. | See Sticky Note2 and Sticky Note3 for newsletter preview images. |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created using n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.