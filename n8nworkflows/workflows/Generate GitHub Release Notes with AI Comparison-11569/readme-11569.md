Generate GitHub Release Notes with AI Comparison

https://n8nworkflows.xyz/workflows/generate-github-release-notes-with-ai-comparison-11569


# Generate GitHub Release Notes with AI Comparison

### 1. Workflow Overview

This workflow automates the generation of GitHub release notes by comparing the latest two releases of a repository and using AI models to summarize and format the changes into user-friendly release notes. It is designed for software projects hosted on GitHub that want to automate changelog creation with the help of AI, improving accuracy and readability.

**Logical Blocks:**

- **1.1 GitHub Release Event Trigger**  
  Detects new releases on a specified GitHub repository.

- **1.2 Release Metadata Extraction and Tag Management**  
  Extracts details from the triggered release and fetches the last two release tags to determine comparison targets.

- **1.3 GitHub API Data Fetching**  
  Retrieves detailed release data and compares the two latest releases to gather commits and file changes.

- **1.4 Data Simplification and Diff Confirmation**  
  Processes and simplifies raw GitHub comparison data to extract meaningful change information, confirming whether differences exist.

- **1.5 AI-Powered Summary and Release Notes Generation**  
  Uses two AI chat models via OpenRouter to first summarize the release differences and then generate polished, structured release notes in Markdown.

---

### 2. Block-by-Block Analysis

#### 2.1 GitHub Release Event Trigger

- **Overview:**  
  Initiates the workflow when a new release event occurs in the configured GitHub repository.

- **Nodes Involved:**  
  - Github Trigger

- **Node Details:**  

  | Node Name     | Details                                                                                                                                        |
  |---------------|------------------------------------------------------------------------------------------------------------------------------------------------|
  | Github Trigger| *Type:* GitHub Trigger Node<br>*Role:* Webhook listener for GitHub release events<br>*Config:* Triggers on "release" event, OAuth2 auth used.<br>*Inputs:* None (trigger)<br>*Outputs:* Release event payload<br>*Edge cases:* Authentication failure, webhook misconfiguration, missing repository/owner values.<br>*Notes:* Requires OAuth2 GitHub credentials configured; triggers only on new releases.|

#### 2.2 Release Metadata Extraction and Tag Management

- **Overview:**  
  Extracts important metadata from the release event and identifies the two most recent release tags for comparison.

- **Nodes Involved:**  
  - Fetch the latest release body (Code node)  
  - Set Repo & Owner (Set node)  
  - Get the last two releases (GitHub node)  
  - Get the last two release tags (Code node)

- **Node Details:**  

  | Node Name                | Details                                                                                                                                                                                                                                                                                                                                                         |
  |--------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
  | Fetch the latest release body | *Type:* Code (JavaScript)<br>*Role:* Parses the incoming release event JSON to extract repository owner, repo name, current release tag, release ID, and URL.<br>*Config:* Uses expressions to extract nested fields from the webhook JSON.<br>*Inputs:* Output of GitHub Trigger<br>*Outputs:* JSON object with owner, repo, currentTag, currentReleaseId, currentReleaseUrl.<br>*Edge cases:* Missing or malformed webhook payload could cause runtime errors.|
  | Set Repo & Owner         | *Type:* Set Node<br>*Role:* Stores owner and repo as workflow variables for downstream use.<br>*Config:* Assigns owner and repo string values from the previous node.<br>*Inputs:* Output from Fetch the latest release body<br>*Outputs:* JSON with owner and repo fields.<br>*Edge cases:* If previous node fails or missing fields, could propagate empty values.                                                                                                                     |
  | Get the last two releases| *Type:* GitHub API Node<br>*Role:* Fetches the last two releases from GitHub using the owner and repo.<br>*Config:* Operation: getAll releases, limit 2, uses OAuth2 credentials.<br>*Inputs:* Workflow variables owner and repo<br>*Outputs:* List of release objects.<br>*Edge cases:* API rate limits, authentication failures, repository not found.                                                                                                                                |
  | Get the last two release tags | *Type:* Code (JavaScript)<br>*Role:* Sorts releases by published date, filters out draft and pre-release, selects current and previous tags.<br>*Config:* Logic to handle cases with only one or no published releases, throws errors if no previous tag found.<br>*Inputs:* Output from Get the last two releases<br>*Outputs:* JSON with currentTag, previousTag, owner, repo.<br>*Edge cases:* No releases found, no previous release to compare to, malformed date fields.                                         |

#### 2.3 GitHub API Data Fetching

- **Overview:**  
  Using the identified tags, fetches detailed comparison data including commits and file changes between the two releases.

- **Nodes Involved:**  
  - Fetch release details from last two releases (HTTP Request node)

- **Node Details:**  

  | Node Name                       | Details                                                                                                                                                                                                                      |
  |--------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
  | Fetch release details from last two releases | *Type:* HTTP Request<br>*Role:* Calls GitHub API to compare two tags and fetch commit and file change data.<br>*Config:* GET https://api.github.com/repos/{{owner}}/{{repo}}/compare/{{previousTag}}...{{currentTag}}, with Accept header 'application/vnd.github+json', OAuth2 authentication.<br>*Inputs:* JSON with owner, repo, previousTag, currentTag.<br>*Outputs:* JSON with full comparison data.<br>*Edge cases:* API rate limit, network errors, invalid tags, missing fields. |

#### 2.4 Data Simplification and Diff Confirmation

- **Overview:**  
  Processes raw comparison data to extract simplified commits and file change summaries; confirms if there are any changes.

- **Nodes Involved:**  
  - Confirm diff (Code node)

- **Node Details:**  

  | Node Name   | Details                                                                                                                                                                                                                                                                                                                                                                                                                                      |
  |-------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
  | Confirm diff| *Type:* Code (JavaScript)<br>*Role:* Simplifies commits and files arrays to essential fields for AI consumption; determines if there are any changes.<br>*Config:* Extracts commit SHA, message, author, date; file filename, status, additions, deletions, patch.<br>*Inputs:* Output from Fetch release details from last two releases.<br>*Outputs:* JSON with hasChanges boolean, simplified commits and files, total commit count, tags.<br>*Edge cases:* Missing expected fields, empty arrays, no changes.|

#### 2.5 AI-Powered Summary and Release Notes Generation

- **Overview:**  
  Uses two AI chat models sequentially to first summarize the differences and then generate polished Markdown release notes.

- **Nodes Involved:**  
  - Summarize release differences (LangChain Agent node)  
  - Generate latest release notes (LangChain Agent node)  
  - OpenRouter Chat Model (LangChain OpenRouter Chat Model node)  
  - OpenRouter Chat Model1 (LangChain OpenRouter Chat Model node)

- **Node Details:**  

  | Node Name                 | Details                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
  |---------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
  | OpenRouter Chat Model      | *Type:* LangChain OpenRouter Chat Model<br>*Role:* Provides AI model interface to summarize release differences.<br>*Config:* Uses "anthropic/claude-3.7-sonnet" AI model via OpenRouter API.<br>*Inputs:* Connected to Summarize release differences node as ai_languageModel input.<br>*Outputs:* Summary in Markdown.<br>*Credentials:* OpenRouter OAuth API key required.<br>*Edge cases:* API errors, rate limits, model unavailability, malformed input.                                                                 |
  | Summarize release differences | *Type:* LangChain Agent Node<br>*Role:* Defines prompt and logic to group changes into sections (New features, Improvements, Bug fixes, Other changes) with user-facing bullet points, returning Markdown only.<br>*Config:* Prompt uses JSON diff input from previous node; output is Markdown summary.<br>*Inputs:* ai_languageModel from OpenRouter Chat Model.<br>*Outputs:* Markdown summary.<br>*Edge cases:* AI prompt failures, unexpected output format.                                                                                                 |
  | OpenRouter Chat Model1     | *Type:* LangChain OpenRouter Chat Model<br>*Role:* Provides AI model interface for generating final release notes from summarized diff.<br>*Config:* Uses "google/gemini-2.5-flash-lite" model via OpenRouter.<br>*Inputs:* Connected as ai_languageModel for Generate latest release notes node.<br>*Outputs:* Final release notes in Markdown.<br>*Credentials:* OpenRouter API credentials.<br>*Edge cases:* API errors, network failures.                                                                                                            |
  | Generate latest release notes | *Type:* LangChain Agent Node<br>*Role:* Generates end-user friendly release notes based on diff data; applies rules for grouping and formatting.<br>*Config:* Prompt instructs Markdown output with sections only if relevant, user-focused bullet points, short notes if no changes.<br>*Inputs:* ai_languageModel from OpenRouter Chat Model1.<br>*Outputs:* Final Markdown release notes.<br>*Edge cases:* AI model failures, unexpected input data, empty changes.|

---

### 3. Summary Table

| Node Name                          | Node Type                                   | Functional Role                               | Input Node(s)                    | Output Node(s)                   | Sticky Note                                                                                              |
|-----------------------------------|---------------------------------------------|-----------------------------------------------|---------------------------------|---------------------------------|---------------------------------------------------------------------------------------------------------|
| Github Trigger                    | GitHub Trigger                              | Trigger workflow on GitHub release event      | None                            | Fetch the latest release body    | ## GitHub Trigger - This workflow triggers when a new release is created in your GitHub repository.   |
| Fetch the latest release body     | Code                                        | Extract key release metadata from webhook     | Github Trigger                  | Set Repo & Owner                | ## Confirm latest release tag and details - These nodes fetch the key release details for improved outputs at the release note generation stages |
| Set Repo & Owner                  | Set                                         | Store repo owner and name for downstream use  | Fetch the latest release body   | Get the last two releases       | Same as above                                                                                           |
| Get the last two releases         | GitHub                                      | Fetch last two releases for comparison        | Set Repo & Owner               | Get the last two release tags    | ## Fetch last two releases - These nodes focus on fetching details from the last two releases to compare and confirm differences |
| Get the last two release tags     | Code                                        | Determine tags of current and previous releases| Get the last two releases       | Fetch release details from last two releases | Same as above                                                                                           |
| Fetch release details from last two releases | HTTP Request                               | Retrieve detailed commit and file changes     | Get the last two release tags   | Confirm diff                    | Same as above                                                                                           |
| Confirm diff                     | Code                                        | Simplify and confirm if changes exist         | Fetch release details from last two releases | Summarize release differences   | ## Summarise and generate release notes - Analyze repo diff and prepare data for release note generation |
| Summarize release differences    | LangChain Agent                             | Summarize diff into categorized Markdown      | Confirm diff                   | Generate latest release notes    | Same as above                                                                                           |
| Generate latest release notes     | LangChain Agent                             | Generate final polished Markdown release notes| Summarize release differences  | None                           | Same as above                                                                                           |
| OpenRouter Chat Model             | LangChain OpenRouter Chat Model             | AI model for summarizing release differences  | Summarize release differences (ai_languageModel) | Summarize release differences   | ## How it works - AI Chat Model summarizes the diff and generates structured release notes. Setup steps include GitHub OAuth and OpenRouter API keys |
| OpenRouter Chat Model1            | LangChain OpenRouter Chat Model             | AI model for generating final release notes   | Generate latest release notes (ai_languageModel) | Generate latest release notes    | Same as above                                                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create GitHub Trigger Node**  
   - Type: GitHub Trigger  
   - Parameters: Event = "release"  
   - Authentication: OAuth2 with GitHub OAuth credentials  
   - This triggers the workflow when a new release is published.

2. **Add a Code Node: "Fetch the latest release body"**  
   - Type: Code (JavaScript)  
   - Input: GitHub Trigger output  
   - Code extracts owner, repo, currentTag, currentReleaseId, and currentReleaseUrl from webhook payload.  
   - Output: JSON with these details.

3. **Add a Set Node: "Set Repo & Owner"**  
   - Type: Set  
   - Assign variables: owner and repo based on the previous node's JSON fields.  
   - Output: JSON containing owner and repo strings.

4. **Add a GitHub Node: "Get the last two releases"**  
   - Type: GitHub  
   - Operation: getAll releases  
   - Limit: 2  
   - Owner and repo: Use expressions from "Set Repo & Owner" node.  
   - Authentication: OAuth2 GitHub credentials.

5. **Add a Code Node: "Get the last two release tags"**  
   - Type: Code (JavaScript)  
   - Input: Output from "Get the last two releases"  
   - Logic: Filter out drafts and prereleases, sort by published date, set currentTag and previousTag. Throw error if previous tag not found.  
   - Output: JSON with currentTag, previousTag, owner, repo.

6. **Add HTTP Request Node: "Fetch release details from last two releases"**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.github.com/repos/{{ $json.owner }}/{{ $json.repo }}/compare/{{ $json.previousTag }}...{{ $json.currentTag }}`  
   - Headers: Accept: application/vnd.github+json  
   - Authentication: GitHub OAuth2 credentials  
   - Input: Use currentTag, previousTag, owner, repo from previous node.

7. **Add Code Node: "Confirm diff"**  
   - Type: Code (JavaScript)  
   - Input: Output of HTTP request node  
   - Logic: Simplify commits and files arrays, create hasChanges flag, extract relevant info for AI.  
   - Output: JSON with hasChanges, commits, files, total_commits, tags.

8. **Add LangChain OpenRouter Chat Model Node: "OpenRouter Chat Model"**  
   - Type: LangChain OpenRouter Chat Model  
   - Model: "anthropic/claude-3.7-sonnet"  
   - Credentials: OpenRouter API key  
   - Connect this node as ai_languageModel input to next "Summarize release differences" node.

9. **Add LangChain Agent Node: "Summarize release differences"**  
   - Type: LangChain Agent  
   - Prompt: Provide JSON diff, instruct to group changes into sections (New features, Improvements, Bug fixes, Other changes) with bullet points, output Markdown only.  
   - Input: Output from "Confirm diff" node, AI model from previous node.  
   - Output: Markdown summary.

10. **Add second LangChain OpenRouter Chat Model Node: "OpenRouter Chat Model1"**  
    - Type: LangChain OpenRouter Chat Model  
    - Model: "google/gemini-2.5-flash-lite"  
    - Credentials: OpenRouter API key  
    - Connect this node as ai_languageModel input to next "Generate latest release notes" node.

11. **Add LangChain Agent Node: "Generate latest release notes"**  
    - Type: LangChain Agent  
    - Prompt: Use diff JSON input to generate final user-facing release notes, grouped by sections only if relevant, Markdown output with a title including currentTag.  
    - Input: Output from "Summarize release differences" node, AI model from previous node.  
    - Output: Final release notes Markdown.

12. **Connect nodes in sequence:**  
    Github Trigger → Fetch the latest release body → Set Repo & Owner → Get the last two releases → Get the last two release tags → Fetch release details from last two releases → Confirm diff → Summarize release differences (with OpenRouter Chat Model) → Generate latest release notes (with OpenRouter Chat Model1).

13. **Credential Setup:**  
    - GitHub OAuth2 API credentials must be configured and selected in all GitHub nodes and HTTP Request node.  
    - OpenRouter API credentials must be configured and selected in both LangChain OpenRouter Chat Model nodes.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Context or Link                                                                                                           |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| The workflow uses GitHub OAuth2 authentication and OpenRouter API keys for AI model access. Ensure proper credential setup for seamless execution.                                                                                                                                                                                                                                                                                                                   | Setup instructions in Sticky Note4.                                                                                       |
| AI prompt engineering is key: the LangChain Agent nodes use carefully crafted prompts to generate structured, user-friendly release notes in Markdown format, focusing on user impact rather than internal Git data.                                                                                                                                                                                                                                                 | Inline prompts in "Summarize release differences" and "Generate latest release notes" nodes.                              |
| The workflow is designed to be flexible to repositories with varying release cadences and handles edge cases such as missing previous releases or no code changes gracefully by throwing errors or generating appropriate minimal notes.                                                                                                                                                                                                                              | Error handling in "Get the last two release tags" and "Generate latest release notes" nodes.                              |
| The final output is a Markdown formatted changelog that can be pasted into GitHub Releases, documentation sites, or automated pipelines for publication.                                                                                                                                                                                                                                                                                                           | Described in Sticky Note2 and Sticky Note3.                                                                               |
| For more information about GitHub API limits and best practices, consult the official GitHub API documentation: https://docs.github.com/en/rest/reference/repos#compare-two-commits                                                                                                                                                                                                                                                                                 | GitHub API Docs                                                                                                           |

---

**Disclaimer:** The text provided is extracted exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected content. All data handled is legal and public.