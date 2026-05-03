Generate Professional Changelogs from Git Commits with GPT-4 and GitHub

https://n8nworkflows.xyz/workflows/generate-professional-changelogs-from-git-commits-with-gpt-4-and-github-8137


# Generate Professional Changelogs from Git Commits with GPT-4 and GitHub

### 1. Workflow Overview

This n8n workflow automates the generation of professional changelogs from Git commit data using GPT-4 and GitHub APIs. The core purpose is to listen for GitHub repository events, fetch the related commits, process them through an AI agent powered by GPT-4 to create a refined changelog, and finally publish a GitHub Release incorporating that changelog.

The workflow is logically divided into the following blocks:

- **1.1 Event Reception:** Listens to GitHub repository events via a webhook trigger.
- **1.2 Commit Retrieval:** Fetches commit details associated with the event.
- **1.3 AI Processing:** Uses a LangChain AI agent with OpenAI GPT-4 to generate a professional changelog from raw commit data.
- **1.4 Release Creation:** Creates a new GitHub Release with the generated changelog.

---

### 2. Block-by-Block Analysis

#### 2.1 Event Reception

- **Overview:**  
This block initiates the workflow when a GitHub event occurs (e.g., push or pull request merged). It uses the GitHub Trigger node to listen for these events.

- **Nodes Involved:**  
  - GitHub Trigger  
  - Note - GitHub Trigger

- **Node Details:**  

  - **GitHub Trigger**  
    - Type: `n8n-nodes-base.githubTrigger`  
    - Role: Listens for GitHub webhook events to start the workflow automatically.  
    - Configuration: Default parameters, expects configured GitHub webhook credentials and repository setup.  
    - Input: External GitHub webhook calls.  
    - Output: Emits event data to the next node ("Get Commits").  
    - Edge cases:  
      - Webhook misconfiguration or missing permissions can cause non-triggering.  
      - Network issues may delay or fail event reception.  
    - Version: Compatible with n8n GitHub Trigger v1.  

  - **Note - GitHub Trigger**  
    - Type: Sticky Note  
    - Content: Empty in this export but serves as a placeholder or for documentation in UI.

#### 2.2 Commit Retrieval

- **Overview:**  
Fetches the detailed list of commits related to the triggering GitHub event to provide raw data for changelog generation.

- **Nodes Involved:**  
  - Get Commits  
  - Note - Get Commits

- **Node Details:**  

  - **Get Commits**  
    - Type: `n8n-nodes-base.github`  
    - Role: Retrieves commit information from GitHub repository based on the event context.  
    - Configuration: Likely configured to request commits from the repository and branch associated with the trigger event.  
    - Input: Trigger event data from "GitHub Trigger".  
    - Output: Sends commit data to the "AI Agent" node.  
    - Edge cases:  
      - API rate limits or authentication errors from GitHub.  
      - Event data might not contain expected commit references.  
      - Empty commit sets for certain event types.  
    - Version: Compatible with n8n GitHub node v1.  

  - **Note - Get Commits**  
    - Type: Sticky Note  
    - Content: Empty placeholder.

#### 2.3 AI Processing

- **Overview:**  
Processes the raw commits through an AI-powered LangChain agent using OpenAI's GPT-4 model to generate a professional and human-readable changelog summary.

- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model  
  - Simple Memory  
  - Note - LLM Agent

- **Node Details:**  

  - **AI Agent**  
    - Type: `@n8n/n8n-nodes-langchain.agent`  
    - Role: Central orchestrator that runs LangChain AI logic, leveraging a language model and memory.  
    - Configuration: Connected to the "OpenAI Chat Model" as its language model and "Simple Memory" for context retention.  
    - Input: Receives commit data from "Get Commits".  
    - Output: Sends processed changelog text to "Create GitHub Release".  
    - Edge cases:  
      - AI model API key or quota issues.  
      - Unexpected commit data format causing prompt or parsing failures.  
      - Memory overflow or context window limits.  
    - Version: LangChain Agent v2.2.  

  - **OpenAI Chat Model**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
    - Role: Provides GPT-4 based conversational language model functionality.  
    - Configuration: Uses OpenAI credentials; defaults likely set for GPT-4 or similar.  
    - Input: Connected as language model for "AI Agent".  
    - Output: N/A (used internally by AI Agent).  
    - Edge cases:  
      - API limits, network timeouts, or invalid credentials.  
      - Model version mismatches or parameter misconfiguration.  
    - Version: v1.2.  

  - **Simple Memory**  
    - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
    - Role: Maintains a short-term memory buffer window to support context in AI processing.  
    - Configuration: Likely default buffer size; helps retain conversation or processing history.  
    - Input: Connected as memory for "AI Agent".  
    - Output: N/A (used internally by AI Agent).  
    - Edge cases:  
      - Memory buffer overflow or loss of relevant context if buffer too small.  
    - Version: v1.3.  

  - **Note - LLM Agent**  
    - Type: Sticky Note  
    - Content: Empty placeholder.

#### 2.4 Release Creation

- **Overview:**  
Creates a new GitHub Release entry using the changelog generated by the AI agent.

- **Nodes Involved:**  
  - Create GitHub Release  
  - Note - GitHub Release

- **Node Details:**  

  - **Create GitHub Release**  
    - Type: `n8n-nodes-base.github`  
    - Role: Uses GitHub API to create a release in the repository with the AI-generated changelog as release notes.  
    - Configuration: Set to create releases, requires repository and authentication credentials.  
    - Input: Receives changelog text from "AI Agent".  
    - Output: Final output, can be routed for notifications or logging if extended.  
    - Edge cases:  
      - Permission errors if the token lacks release creation rights.  
      - Conflicts if release tag already exists.  
      - API rate limits or network failures.  
    - Version: Compatible with n8n GitHub node v1.  

  - **Note - GitHub Release**  
    - Type: Sticky Note  
    - Content: Empty placeholder.

---

### 3. Summary Table

| Node Name            | Node Type                         | Functional Role                         | Input Node(s)   | Output Node(s)         | Sticky Note          |
|----------------------|----------------------------------|---------------------------------------|-----------------|------------------------|----------------------|
| GitHub Trigger       | n8n-nodes-base.githubTrigger     | Event listener for GitHub repository  | (external)      | Get Commits            |                      |
| Get Commits          | n8n-nodes-base.github            | Fetch commits from GitHub              | GitHub Trigger  | AI Agent               |                      |
| AI Agent             | @n8n/n8n-nodes-langchain.agent  | AI processing of commit data to changelog | Get Commits     | Create GitHub Release  |                      |
| OpenAI Chat Model    | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides GPT-4 language model       | (internal to AI Agent) | AI Agent           |                      |
| Simple Memory        | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintains AI context memory       | (internal to AI Agent) | AI Agent           |                      |
| Create GitHub Release| n8n-nodes-base.github            | Creates GitHub release with changelog | AI Agent        | (end)                  |                      |
| Note - GitHub Trigger| n8n-nodes-base.stickyNote        | Documentation / UI note                |                 |                        |                      |
| Note - Get Commits   | n8n-nodes-base.stickyNote        | Documentation / UI note                |                 |                        |                      |
| Note - LLM Agent     | n8n-nodes-base.stickyNote        | Documentation / UI note                |                 |                        |                      |
| Note - GitHub Release| n8n-nodes-base.stickyNote        | Documentation / UI note                |                 |                        |                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the GitHub Trigger node:**  
   - Type: `GitHub Trigger`  
   - Configure credentials with a GitHub OAuth token or personal access token with webhook and repo access.  
   - Set to listen for repository events (e.g., push or pull request merged).  
   - Position it as the start node.

2. **Add the Get Commits node:**  
   - Type: `GitHub` (API node)  
   - Connect input from `GitHub Trigger`.  
   - Configure to fetch commits from the repository and branch associated with the incoming event.  
   - Use expressions to extract the repository owner, repo name, and commit range from the trigger payload.  
   - Ensure credentials are set and have read access to the repository.

3. **Add the AI Agent node:**  
   - Type: `LangChain Agent`  
   - Connect input from `Get Commits`.  
   - Set up to use an OpenAI Chat Model node as the language model and a Simple Memory node for context.  
   - Define prompt templates or instructions to transform raw commit messages into a professional changelog format.  
   - Provide OpenAI API credentials with GPT-4 access.

4. **Add the OpenAI Chat Model node:**  
   - Type: `LangChain OpenAI Chat Model`  
   - Connect this node to the AI Agent’s language model input.  
   - Configure with your OpenAI API key and specify GPT-4 or equivalent model.  
   - Set temperature and max tokens per your requirements (e.g., temperature ~0.3 for consistency).

5. **Add the Simple Memory node:**  
   - Type: `LangChain Memory Buffer Window`  
   - Connect this node to the AI Agent’s memory input.  
   - Configure buffer length as needed (default is typically sufficient).

6. **Add the Create GitHub Release node:**  
   - Type: `GitHub` (API node)  
   - Connect input from the AI Agent node.  
   - Configure to create a new release in the repository.  
   - Use expressions to set the release tag/version and include the AI-generated changelog text as release notes.  
   - Ensure credentials have permission to create releases.

7. **Sticky Notes (optional):**  
   - Add sticky notes above each block for documentation or explanations if desired.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                   |
|-----------------------------------------------------------------------------------------------|--------------------------------------------------|
| This workflow leverages GPT-4 via LangChain integration to enhance GitHub commit data handling.| n8n LangChain Nodes documentation                 |
| Ensure GitHub personal access tokens have appropriate scopes: `repo`, `admin:repo_hook`.      | GitHub Token scopes documentation                  |
| OpenAI GPT-4 API usage requires a valid API key with sufficient quota and permissions.        | https://platform.openai.com/docs/api-reference    |
| Using memory buffer helps maintain conversational context across AI prompts for better output.| LangChain memory concepts overview                 |

---

**Disclaimer:**  
The text provided here is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled are legal and public.