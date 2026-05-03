Automated PR Code Reviews with GitHub, GPT-4, and Google Sheets Best Practices

https://n8nworkflows.xyz/workflows/automated-pr-code-reviews-with-github--gpt-4--and-google-sheets-best-practices-3804


# Automated PR Code Reviews with GitHub, GPT-4, and Google Sheets Best Practices

### 1. Workflow Overview

This n8n workflow automates AI-assisted code reviews for GitHub pull requests (PRs) by leveraging GPT-4-based OpenAI models and optionally enriching the review with internal coding best practices stored in a Google Sheet. Its primary use case is to streamline and standardize code review processes for iOS projects (or other codebases) by analyzing PR diffs, generating contextual review comments, posting them back to GitHub, and visually marking reviewed PRs with labels.

The workflow is logically divided into the following blocks:

- **1.1 GitHub PR Event Trigger**: Listens for new or updated pull requests on specified repositories to start the workflow.
- **1.2 PR Diff Extraction**: Fetches the list of changed files and their diffs from the triggered PR via GitHub API.
- **1.3 Prompt Construction**: Parses and formats the code diffs into a structured natural language prompt instructing the AI to perform a code review.
- **1.4 AI-Powered Code Review Agent**: Uses OpenAI’s GPT-4o-mini model to generate a detailed code review based on the prompt, optionally enriched by referencing best practices from a Google Sheet.
- **1.5 GitHub Comment Poster**: Posts the AI-generated code review as an inline comment on the pull request.
- **1.6 Pull Request Labeler (Optional)**: Adds a customizable label, e.g. "ReviewedByAI", to the PR to indicate it has been reviewed by the AI.

---

### 2. Block-by-Block Analysis

#### Block 1.1: GitHub PR Event Trigger

- **Overview**:  
  This block initiates the workflow by listening to pull request events on configured GitHub repositories. It ensures real-time automation by triggering on PR creation or updates.

- **Nodes Involved**:  
  - PR Trigger

- **Node Details**:

  - **PR Trigger**  
    - Type: GitHub Trigger node  
    - Role: Event listener for GitHub pull request events  
    - Configuration:  
      - Monitors specific GitHub repositories (selected from a list)  
      - Triggered on the "pull_request" event  
      - Authentication via GitHub OAuth2 credentials for secure access  
    - Inputs: None (webhook-based trigger)  
    - Outputs: JSON payload containing PR metadata, including repository info, PR number, and diffs URL  
    - Edge Cases / Failures:  
      - Authentication errors if OAuth token is invalid or expired  
      - Network issues causing webhook reception failures  
      - Incorrect repository configuration leading to missed events  
    - Notes: This node must be publicly accessible or properly configured in n8n to receive GitHub webhooks.

---

#### Block 1.2: PR Diff Extraction

- **Overview**:  
  Fetches the changed files and their diffs for the triggered PR using GitHub’s REST API. This data is essential for generating an accurate code review.

- **Nodes Involved**:  
  - Get file's Diffs from PR

- **Node Details**:

  - **Get file's Diffs from PR**  
    - Type: HTTP Request node  
    - Role: Calls GitHub REST API to retrieve the list of changed files and their patch diffs for the PR  
    - Configuration:  
      - URL dynamically constructed from PR trigger data:  
        `https://api.github.com/repos/{{$json.body.sender.login}}/{{$json.body.repository.name}}/pulls/{{$json.body.number}}/files`  
      - No explicit authentication (relies on public API or OAuth token from trigger context)  
      - HTTP GET method (default)  
    - Inputs: Receives PR event JSON from PR Trigger node  
    - Outputs: Array of file objects, each containing filename and patch diff text  
    - Edge Cases / Failures:  
      - API rate limiting by GitHub  
      - No patch data for binary files  
      - Invalid PR number causing 404 errors  
      - Network timeouts or request failures  
    - Notes: This node only fetches file diffs, not the entire PR content.

---

#### Block 1.3: Prompt Construction

- **Overview**:  
  Transforms the raw PR diffs into a well-structured prompt for the AI. It formats each file’s diff with markdown syntax, cleans triple backticks to prevent formatting issues, and creates an instructional message guiding the AI reviewer.

- **Nodes Involved**:  
  - Create target Prompt from PR Diffs

- **Node Details**:

  - **Create target Prompt from PR Diffs**  
    - Type: Code node (JavaScript)  
    - Role: Parses the array of diffs, formats them with filenames and markdown diff blocks, and generates a natural language review prompt  
    - Configuration:  
      - JavaScript code loops over each file object, replacing triple backticks in patches with double single-quotes to avoid markdown conflicts  
      - Constructs a prompt starting with a role description ("You are a senior iOS developer...") and instructions for reviewing changes file by file  
      - Ignores files without patches (e.g., binaries)  
      - Output JSON object with a `user_message` field containing the final prompt string  
    - Inputs: Receives file diffs array from HTTP Request node  
    - Outputs: JSON object with AI prompt string for downstream use  
    - Edge Cases / Failures:  
      - Empty diffs array producing an empty or invalid prompt  
      - Unexpected patch formatting causing code errors  
      - Non-text diffs causing improper parsing  
    - Notes: The prompt is crafted to maximize code review quality by the AI.

---

#### Block 1.4: AI-Powered Code Review Agent

- **Overview**:  
  Uses an OpenAI GPT-4o-mini model to analyze the formatted prompt and generate a detailed code review. It optionally enriches the review by referencing best practices from a Google Sheet.

- **Nodes Involved**:  
  - OpenAI Chat Model  
  - Code Best Practices (optional)  
  - Code Review Agent

- **Node Details**:

  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI Chat Model node  
    - Role: Provides the language model backend (GPT-4o-mini) for the AI agent  
    - Configuration:  
      - Model set to "gpt-4o-mini" (a GPT-4 variant optimized for cost and speed)  
      - No additional options configured  
      - Requires OpenAI API key credentials  
    - Inputs: None directly; linked as AI language model for the agent node  
    - Outputs: Language model interface for the agent node  
    - Edge Cases / Failures:  
      - API key invalid or quota exceeded  
      - Network latency or downtime on OpenAI side  

  - **Code Best Practices**  
    - Type: Google Sheets Tool node  
    - Role: Provides a lookup tool to supply coding guideline references from a configured Google Sheet  
    - Configuration:  
      - Connected to a Google Sheet document by URL and sheet name containing best practices  
      - OAuth2 for Google Sheets API authentication  
      - Optional; if not configured, the agent works without best practice enrichment  
    - Inputs: None directly; linked as AI tool for the agent node  
    - Outputs: Provides best practice data to the agent at runtime  
    - Edge Cases / Failures:  
      - Invalid or expired Google OAuth token  
      - Incorrect sheet URL or name  
      - Sheet data format mismatches  
    - Notes: This node enhances review accuracy and opinionation.

  - **Code Review Agent**  
    - Type: LangChain Agent node  
    - Role: Central AI agent that combines the prompt, language model, and tools (Google Sheet) to output the final code review text  
    - Configuration:  
      - Accepts the `user_message` prompt from the prompt creation node as input text  
      - Connected to OpenAI Chat Model as language model  
      - Connected optionally to Code Best Practices node as a reference tool  
      - Set to "define" prompt type for controlled generation  
    - Inputs:  
      - Prompt string from prompt creation node  
      - Language model and best practices tools  
    - Outputs: JSON with AI-generated review text under `output`  
    - Edge Cases / Failures:  
      - Failure to connect to OpenAI or Google Sheets nodes  
      - Unexpected input formats causing generation errors  
      - Excessively long prompts exceeding token limits  
    - Notes: This is the core AI processing node.

---

#### Block 1.5: GitHub Comment Poster

- **Overview**:  
  Posts the AI-generated review comment back to the pull request on GitHub via API, enabling reviewers and developers to see the automated feedback inline.

- **Nodes Involved**:  
  - GitHub Robot

- **Node Details**:

  - **GitHub Robot**  
    - Type: GitHub node  
    - Role: Posts a review comment on the PR using the GitHub API  
    - Configuration:  
      - Operation: Comment on "review" resource (pull request reviews)  
      - Body: Uses the AI-generated review text from the Code Review Agent node output (`{{$json.output}}`)  
      - Repository and owner dynamically selected from the workflow context  
      - Pull request number from the PR Trigger node data  
      - Authentication via GitHub API credentials (OAuth or PAT)  
    - Inputs: AI review text from Code Review Agent  
    - Outputs: API response confirming comment creation  
    - Edge Cases / Failures:  
      - Authentication failure or insufficient permissions to post comments  
      - API rate limits or network errors  
      - Invalid PR number or repository causing 404 errors  
    - Notes: This node ensures the AI review is visible on GitHub.

---

#### Block 1.6: Pull Request Labeler (Optional)

- **Overview**:  
  Adds a label such as "ReviewedByAI" to the PR after successfully posting the review comment, providing a visual indicator for team members.

- **Nodes Involved**:  
  - Add Label to PR

- **Node Details**:

  - **Add Label to PR**  
    - Type: GitHub node  
    - Role: Edits the PR’s labels by adding a custom label  
    - Configuration:  
      - Operation: Edit issue (PRs are issues in GitHub API terms)  
      - Labels: Adds "ReviewedByAI" label (customizable)  
      - Repository, owner, and issue number dynamically derived from PR Trigger data  
      - Authentication via GitHub OAuth2 credentials  
    - Inputs: Triggered after GitHub Robot node confirms comment posting  
    - Outputs: API response confirming label addition  
    - Edge Cases / Failures:  
      - Label does not exist in the repo (GitHub may auto-create or fail depending on settings)  
      - Authentication or permission errors  
      - API rate limits or network issues  
    - Notes: This node is optional but improves workflow visibility.

---

### 3. Summary Table

| Node Name                  | Node Type                          | Functional Role                          | Input Node(s)             | Output Node(s)           | Sticky Note                                                                                 |
|----------------------------|----------------------------------|----------------------------------------|---------------------------|--------------------------|---------------------------------------------------------------------------------------------|
| PR Trigger                 | GitHub Trigger                   | Starts workflow on PR events           | -                         | Get file's Diffs from PR  | **1-The GitHub Trigger** node initiates the workflow whenever a pull request event occurs on a specified repository. It enables real-time automation based on GitHub activity. |
| Get file's Diffs from PR    | HTTP Request                    | Fetches changed files and diffs from PR| PR Trigger                | Create target Prompt from PR Diffs | **2-Get PR Diffs** The HTTP Request node fetches the list of changed files and their diffs from the pull request that triggered the workflow. It uses the GitHub REST API to retrieve this data dynamically based on the trigger payload. https://api.github.com/repos/{{$json.body.sender.login}}/{{$json.body.repository.name}}/pulls/{{$json.body.number}}/files |
| Create target Prompt from PR Diffs | Code                       | Formats diffs into AI review prompt    | Get file's Diffs from PR  | Code Review Agent         | **3-Create Prompt from diffs** This Code node runs a JavaScript snippet to: -Parse file diffs from the previous HTTP Request node -Format each diff with its file name -Build a structured natural language prompt for the AI agent The final output is a clear, contextual instruction like: *"You are a senior iOS developer. Please review the following code changes in these files..."* |
| OpenAI Chat Model          | LangChain OpenAI Chat Model      | Provides GPT-4o-mini AI model           | - (used by Code Review Agent) | Code Review Agent         |                                                                                             |
| Code Best Practices        | Google Sheets Tool               | Supplies coding best practices data     | - (used by Code Review Agent as tool) | Code Review Agent         | **Google Sheet Best Practices** Enables the AI agent to reference to your team coding guidelines stored in a Google Sheet for more accurate and opinionated reviews. You can replace Google Sheets with any other database or tool. |
| Code Review Agent          | LangChain Agent                  | Generates AI code review comment        | Create target Prompt from PR Diffs, OpenAI Chat Model, Code Best Practices | GitHub Robot             |                                                                                             |
| GitHub Robot               | GitHub Node                     | Posts AI code review comment on PR      | Code Review Agent         | Add Label to PR           | **Github Comment Poster** Posts the AI-generated review as a comment on the pull request using GitHub API. |
| Add Label to PR            | GitHub Node                     | Adds label to PR to mark it as reviewed | GitHub Robot             | -                        | **PR Labeler (optional)** Automatically adds a label like *ReviewedByAI* to the pull request once the AI comment is posted. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "PR Trigger" Node**  
   - Type: GitHub Trigger  
   - Configure:  
     - Select your GitHub repository(ies) to monitor  
     - Set event to "pull_request"  
     - Authenticate using GitHub OAuth2 credentials with permissions for PR read and comment write  
   - Position: Left-top area (for logical flow)  

2. **Create "Get file's Diffs from PR" Node**  
   - Type: HTTP Request  
   - Configure:  
     - Method: GET (default)  
     - URL:  
       ```
       https://api.github.com/repos/{{$json.body.sender.login}}/{{$json.body.repository.name}}/pulls/{{$json.body.number}}/files
       ```  
     - No authentication needed here (relies on trigger context)  
   - Connect "PR Trigger" output to this node’s input  

3. **Create "Create target Prompt from PR Diffs" Node**  
   - Type: Code (JavaScript)  
   - Configure code to:  
     - Loop over input array of file diffs  
     - For each file, append filename and patch (replacing triple backticks with double single-quotes)  
     - Format patches inside markdown diff code blocks  
     - Build a user prompt string instructing the AI to review changes file by file  
     - Output JSON: `{ user_message: <prompt string> }`  
   - Connect "Get file's Diffs from PR" output to this node  

4. **Create "OpenAI Chat Model" Node**  
   - Type: LangChain OpenAI Chat Model  
   - Configure:  
     - Model: Select "gpt-4o-mini" (or "gpt-4o", "gpt-4-turbo", "gpt-3.5-turbo")  
     - Provide your OpenAI API key credentials  
   - No direct connection needed here, the node is linked as AI language model for the agent  

5. **Create "Code Best Practices" Node (Optional)**  
   - Type: Google Sheets Tool  
   - Configure:  
     - Provide Google Sheets OAuth2 credentials  
     - Set the Google Sheet URL containing your internal coding guidelines  
     - Specify the sheet name where best practices are stored  
   - No direct connection needed here, linked as AI tool for the agent  

6. **Create "Code Review Agent" Node**  
   - Type: LangChain Agent  
   - Configure:  
     - Text input: Bind to `user_message` output from "Create target Prompt from PR Diffs" node  
     - AI Language Model: Select the "OpenAI Chat Model" node created earlier  
     - AI Tool: Add "Code Best Practices" node if using best practices reference  
     - Prompt Type: Set to "define"  
   - Connect "Create target Prompt from PR Diffs" output to this node’s input  

7. **Create "GitHub Robot" Node**  
   - Type: GitHub  
   - Configure:  
     - Operation: Comment on pull request review  
     - Body: Bind to AI review output text from "Code Review Agent" (`{{$json.output}}`)  
     - Owner and Repository: Select same GitHub repo as trigger  
     - Pull Request Number: Use expression from "PR Trigger" node’s JSON path `body.number`  
     - Credentials: Use GitHub API credentials with permission to post PR comments  
   - Connect "Code Review Agent" output to this node  

8. **Create "Add Label to PR" Node (Optional)**  
   - Type: GitHub  
   - Configure:  
     - Operation: Edit issue (pull request)  
     - Labels: Add label "ReviewedByAI" or custom label text  
     - Owner, Repository: Same as trigger  
     - Issue Number: Use expression from "PR Trigger" node’s JSON path `body.number`  
     - Credentials: GitHub OAuth2 with permissions to edit issues/PR metadata  
   - Connect "GitHub Robot" output to this node  

9. **Arrange nodes in logical left-to-right flow**:  
   PR Trigger → Get file's Diffs from PR → Create target Prompt from PR Diffs → Code Review Agent (with linked OpenAI Chat Model and optionally Code Best Practices) → GitHub Robot → Add Label to PR

10. **Enable the workflow** and ensure webhook URLs are registered with GitHub for the PR trigger.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                       | Context or Link                                                                                                               |
|----------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Use a GitHub OAuth App or Personal Access Token with repo scope for posting comments and fetching PR diffs.                                       | GitHub API Authentication                                                                                                    |
| Google Sheets integration is optional; replace with another database or lookup service if desired.                                                | Flexible AI Tool Integration                                                                                                 |
| The AI prompt construction code replaces triple backticks to avoid markdown parsing issues in GitHub comments.                                   | Prompt Formatting Best Practice                                                                                              |
| Consider GitHub API rate limits and implement retries or error handling for production use.                                                       | GitHub API Limits: https://docs.github.com/en/rest/overview/resources-in-the-rest-api#rate-limiting                            |
| OpenAI GPT-4o-mini is used here for cost-effective, fast code reviews; switch to more powerful models for deeper analysis if needed.             | OpenAI Models: https://platform.openai.com/docs/models                                                                    |
| Labeling PRs after review helps visibility and supports automated workflows or status checks.                                                     | GitHub Labels Usage                                                                                                          |
| Sticky notes in the workflow provide contextual explanations and useful URLs, improving maintainability.                                         | Workflow Documentation Practice                                                                                             |

---

This comprehensive reference document provides a detailed understanding of the automated PR code review workflow, enabling advanced users or AI agents to replicate, modify, and troubleshoot it effectively.