Daily Pull Request Summaries from GitHub to Telegram using GPT-4o-mini

https://n8nworkflows.xyz/workflows/daily-pull-request-summaries-from-github-to-telegram-using-gpt-4o-mini-10585


# Daily Pull Request Summaries from GitHub to Telegram using GPT-4o-mini

### 1. Workflow Overview

This workflow automates daily monitoring of the n8n GitHub repository’s latest pull requests, generates AI-powered summaries of these updates using GPT-4o-mini, and sends concise notifications to a Telegram channel. It is designed for n8n users, development teams, or workflow managers who want to stay informed about recent platform changes without manually checking GitHub.

**Logical Blocks:**

- **1.1 Scheduled Trigger:** Automatically initiates the workflow daily at 10 AM.
- **1.2 GitHub Data Retrieval:** Fetches the latest pull request from the n8n repository.
- **1.3 Filtering:** Filters pull requests to only include those created on the current day.
- **1.4 Data Preparation:** Extracts the pull request summary text for AI processing.
- **1.5 AI Summary Generation:** Uses GPT-4o-mini to generate a clear, technical, and scannable English summary.
- **1.6 Telegram Notification:** Sends the generated summary as a formatted message to a Telegram channel.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  Triggers the workflow once daily at 10 AM to start the monitoring process automatically.

- **Nodes Involved:**  
  - Daily Check at 10 AM

- **Node Details:**

  - **Daily Check at 10 AM**  
    - Type: Schedule Trigger  
    - Role: Initiates workflow execution daily at a fixed hour.  
    - Configuration: Trigger time set to 10:00 AM daily.  
    - Expressions: None  
    - Input: None (trigger node)  
    - Output: Passes trigger signal to "Fetch Latest Pull Request" node.  
    - Version: 1.2  
    - Edge Cases: Workflow will not run if n8n instance is down or if the schedule is misconfigured. Timezone considerations may cause timing issues.  
    - Sticky Note: Explains schedule trigger usage and customization [Step 1 Details].

#### 1.2 GitHub Data Retrieval

- **Overview:**  
  Connects to GitHub API to fetch the most recent pull request from the n8n-io/n8n repository.

- **Nodes Involved:**  
  - Fetch Latest Pull Request

- **Node Details:**

  - **Fetch Latest Pull Request**  
    - Type: GitHub node  
    - Role: Retrieves the latest pull request data from the specified repository.  
    - Configuration:  
      - Owner: n8n-io  
      - Repository: n8n  
      - Operation: Get pull requests  
      - Limit: 1 (only the most recent PR)  
    - Expressions: Owner and repository are linked via URLs and list references for dynamic resolution.  
    - Input: Trigger from Schedule node  
    - Output: Pull request JSON including metadata such as body, created_at, etc.  
    - Credentials: Requires GitHub API credentials.  
    - Version: 1.1  
    - Edge Cases: Possible failures include API rate limits, invalid credentials, or repository access issues.  
    - Sticky Note: Details configuration and credential importance [Step 2 Details].

#### 1.3 Filtering

- **Overview:**  
  Filters the fetched pull request to ensure only those created on the current day proceed further, preventing duplicate or outdated notifications.

- **Nodes Involved:**  
  - Filter Today's Updates Only

- **Node Details:**

  - **Filter Today's Updates Only**  
    - Type: Filter node  
    - Role: Compares PR creation date to today’s date, allowing only today's updates.  
    - Configuration:  
      - Condition: Checks if `created_at` > today's date (using `$today` expression).  
      - Case Sensitive: true  
      - Type Validation: loose to handle date formats flexibly.  
    - Input: Pull request data from GitHub node  
    - Output: Passes forward only if condition is true  
    - Version: 2.2  
    - Edge Cases: Date parsing errors, timezone mismatches, or PRs created close to midnight may cause missed or duplicate notifications.  
    - Sticky Note: Explanation of filtering logic and its importance [Step 3 Details].

#### 1.4 Data Preparation

- **Overview:**  
  Extracts the body (summary) text from the pull request JSON and assigns it to a clean field for AI processing.

- **Nodes Involved:**  
  - Extract PR Summary

- **Node Details:**

  - **Extract PR Summary**  
    - Type: Set node (Edit Fields)  
    - Role: Creates a new field `Summary` containing the PR body text.  
    - Configuration: Assigns `Summary` = `{{$json.body}}`  
    - Input: Filtered PR data  
    - Output: JSON with added `Summary` field  
    - Version: 3.4  
    - Edge Cases: If `body` is empty or missing, AI summary input will be incomplete or fail.  
    - Sticky Note: Describes field extraction purpose [Step 4 Details].

#### 1.5 AI Summary Generation

- **Overview:**  
  Uses OpenAI GPT-4o-mini model to generate a concise, clear, and technically accurate summary of the pull request changes, formatted for Telegram delivery.

- **Nodes Involved:**  
  - Generate AI Summary  
  - OpenAI Chat Model

- **Node Details:**

  - **Generate AI Summary**  
    - Type: LangChain LLM Chain  
    - Role: Prepares prompt and invokes AI language model to generate the summary.  
    - Configuration:  
      - Text prompt includes PR summary and creation date.  
      - Instructions emphasize bullet points, technical accuracy, English language, date formatting, and readability.  
    - Input: JSON with `Summary` field and PR date from previous node.  
    - Output: AI-generated summary text.  
    - Version: 1.7  
    - Edge Cases: API key invalid or quota exceeded; input text too long or malformed; network timeouts.  
    - Sub-workflow: Invokes OpenAI Chat Model node.

  - **OpenAI Chat Model**  
    - Type: LangChain LLM Chat Node  
    - Role: Executes GPT-4o-mini chat completion request.  
    - Configuration:  
      - Model: gpt-4o-mini  
      - No additional options set.  
    - Credentials: Requires valid OpenAI API key.  
    - Input: Prompt from Generate AI Summary node.  
    - Output: AI's text response as a summary.  
    - Version: 1.2  
    - Edge Cases: API errors, rate limits, malformed prompts.

  - Sticky Notes: Provide detailed instructions on AI prompt setup, cost estimate, and usage [Step 5 Details].

#### 1.6 Telegram Notification

- **Overview:**  
  Sends the AI-generated summary text as an HTML-formatted message to a specified Telegram chat or channel.

- **Nodes Involved:**  
  - Send to Telegram Channel

- **Node Details:**

  - **Send to Telegram Channel**  
    - Type: Telegram node  
    - Role: Delivers the summary message to Telegram users.  
    - Configuration:  
      - Text: Uses AI summary text (`{{$json.text}}`).  
      - Chat ID: Placeholder `YOUR_TELEGRAM_CHAT_ID` to be replaced by user.  
      - Parse Mode: HTML enabled for rich text formatting.  
      - Append Attribution: Disabled to avoid extra text.  
    - Credentials: Telegram Bot API token required.  
    - Input: AI-generated summary text.  
    - Output: Telegram message sent confirmation.  
    - Version: 1.2  
    - Edge Cases: Invalid chat ID, bot permissions missing, network issues, message length limits.  
    - Sticky Note: Includes instructions for bot creation, chat ID retrieval, and formatting [Step 6 Details].

---

### 3. Summary Table

| Node Name                  | Node Type                      | Functional Role                    | Input Node(s)              | Output Node(s)              | Sticky Note                                                                                                    |
|----------------------------|--------------------------------|----------------------------------|----------------------------|-----------------------------|----------------------------------------------------------------------------------------------------------------|
| Daily Check at 10 AM        | Schedule Trigger               | Initiate daily workflow trigger  | None                       | Fetch Latest Pull Request    | Explains schedule trigger usage and customization [Step 1 Details]                                             |
| Fetch Latest Pull Request   | GitHub node                   | Fetch latest PR from GitHub repo | Daily Check at 10 AM        | Filter Today's Updates Only  | Details configuration and credential importance [Step 2 Details]                                              |
| Filter Today's Updates Only | Filter node                   | Filter PRs created today only    | Fetch Latest Pull Request   | Extract PR Summary           | Explains filtering logic and its importance [Step 3 Details]                                                  |
| Extract PR Summary          | Set node (Edit Fields)         | Extract PR body text for AI      | Filter Today's Updates Only | Generate AI Summary          | Describes field extraction purpose [Step 4 Details]                                                           |
| Generate AI Summary         | LangChain LLM Chain            | Prepare prompt and invoke AI     | Extract PR Summary          | Send to Telegram Channel     | Details AI prompt setup, cost, and usage instructions [Step 5 Details]                                         |
| OpenAI Chat Model           | LangChain LLM Chat Node        | Execute GPT-4o-mini completion   | Generate AI Summary         | Generate AI Summary (response) | Connected as sub-step inside Generate AI Summary node                                                        |
| Send to Telegram Channel    | Telegram node                  | Send AI summary to Telegram chat | Generate AI Summary         | None                        | Instructions for bot creation, chat ID setup, formatting [Step 6 Details]                                      |
| Main Workflow Explanation   | Sticky Note                   | Overview and instructions        | None                       | None                        | Overview and detailed instructions for entire workflow                                                        |
| Step 1 Details             | Sticky Note                   | Explains Schedule Trigger node   | None                       | None                        | Details about schedule trigger and customization                                                              |
| Step 2 Details             | Sticky Note                   | Explains GitHub node             | None                       | None                        | Details about GitHub node configuration and credentials                                                       |
| Step 3 Details             | Sticky Note                   | Explains Filter node             | None                       | None                        | Details about filtering today's updates                                                                        |
| Step 4 Details             | Sticky Note                   | Explains Set node for summary    | None                       | None                        | Details about extracting PR summary                                                                            |
| Step 5 Details             | Sticky Note                   | Explains AI summary generation   | None                       | None                        | Details about AI prompt and cost                                                                                |
| Step 6 Details             | Sticky Note                   | Explains Telegram node           | None                       | None                        | Details about Telegram bot setup and message formatting                                                        |
| Configuration Checklist     | Sticky Note                   | Pre-activation checklist         | None                       | None                        | Checklist for credentials and testing                                                                          |
| Customization Ideas         | Sticky Note                   | Suggestions for extending workflow | None                      | None                        | Ideas for multi-repo, email, Slack, digest, and filtering customization                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Name: Daily Check at 10 AM  
   - Configuration: Set to trigger at 10:00 AM every day  
   - No credentials needed  
   - Connect output to GitHub node

2. **Create GitHub Node**  
   - Type: GitHub  
   - Name: Fetch Latest Pull Request  
   - Credentials: Connect GitHub API credentials  
   - Operation: Get Pull Requests  
   - Owner: n8n-io  
   - Repository: n8n  
   - Limit: 1 (fetch only latest)  
   - Connect input from Schedule Trigger output  
   - Connect output to Filter node

3. **Create Filter Node**  
   - Type: Filter  
   - Name: Filter Today's Updates Only  
   - Condition: Check if `created_at` field of PR is after today's date (use expression `{{$today}}`)  
   - Connect input from GitHub node output  
   - Connect output (true branch) to Set node  
   - False branch left unconnected (discard old PRs)

4. **Create Set Node**  
   - Type: Set (Edit Fields)  
   - Name: Extract PR Summary  
   - Add new field named `Summary`  
   - Assign value: `{{$json["body"]}}` (pull request body)  
   - Connect input from Filter node (true branch)  
   - Connect output to LangChain LLM Chain node

5. **Create LangChain LLM Chain Node**  
   - Type: LangChain Chain LLM  
   - Name: Generate AI Summary  
   - Configure prompt with detailed instructions:  
     - Include PR summary (`{{$json.Summary}}`)  
     - Include PR creation date (`{{$node["Fetch Latest Pull Request"].json["created_at"]}}`)  
     - Instructions for bullet points, technical accuracy, English language, etc.  
   - Connect input from Set node output  
   - Connect output to Telegram node

6. **Create OpenAI Chat Model Node**  
   - Type: LangChain LLM Chat OpenAI  
   - Name: OpenAI Chat Model  
   - Model: Select `gpt-4o-mini`  
   - Credentials: Add OpenAI API key  
   - Connect input from LangChain Chain LLM node (Generate AI Summary)  
   - Output back to Generate AI Summary node for message retrieval

7. **Create Telegram Node**  
   - Type: Telegram  
   - Name: Send to Telegram Channel  
   - Credentials: Add Telegram Bot token  
   - Chat ID: Replace placeholder with actual Telegram chat ID  
   - Message Text: Use expression `{{$json.text}}` from AI summary output  
   - Enable HTML parse mode  
   - Connect input from Generate AI Summary output

8. **Activate Workflow**  
   - Test manually to verify each step completes successfully  
   - Ensure credentials are valid and permissions set properly  
   - Replace placeholder chat ID in Telegram node with the real one  
   - Activate the workflow for automatic daily execution

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                                 |
|-----------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------|
| Workflow monitors n8n GitHub repo and sends AI-generated summaries to Telegram daily at 10 AM       | Overview in Main Workflow Explanation node                                                                     |
| Schedule Trigger documentation                                                                       | https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.scheduletrigger/                             |
| GitHub node documentation                                                                            | https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.github/                                      |
| Filter node documentation                                                                            | https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.filter/                                     |
| Set node (Edit Fields) documentation                                                                 | https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.set/                                        |
| LangChain Chain LLM node documentation                                                               | https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.chainllm/                 |
| Telegram node documentation                                                                          | https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.telegram/                                    |
| Telegram Bot creation and chat ID retrieval instructions                                            | Via Telegram @BotFather and user chat with bot                                                                 |
| AI model used: GPT-4o-mini (cost estimate ~$0.0001 per summary)                                      | OpenAI pricing and model details                                                                                 |
| Customization ideas: monitoring multiple repos, email notifications, Slack integration, weekly digest | Included as sticky notes within the workflow                                                                    |
| Configuration checklist: Ensure GitHub, OpenAI, and Telegram credentials are set before activation    | Included as sticky notes within the workflow                                                                    |

---

This documentation should provide a complete and clear understanding of the workflow structure, node details, and configuration necessary for reproduction, modification, or troubleshooting.