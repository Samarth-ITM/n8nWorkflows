Convert GitHub commits into review-ready pull requests with Google Gemini

https://n8nworkflows.xyz/workflows/convert-github-commits-into-review-ready-pull-requests-with-google-gemini-13118


# Convert GitHub commits into review-ready pull requests with Google Gemini

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow turns GitHub issue context (received via an MCP trigger) into a **review-ready GitHub Pull Request** by:
1) extracting repository + branch targets from natural language using Google Gemini,  
2) validating the repo/branch via the GitHub API,  
3) summarizing commit messages,  
4) generating a PR title + description with an LLM,  
5) creating the PR through the GitHub REST API.

**Target use cases:**
- Auto-create PRs from issue discussions (or “please open a PR from branch X into main” requests).
- Standardize PR titles/descriptions from commit history.
- Provide AI-driven repo/branch extraction when users don’t format inputs consistently.

### Logical Blocks
**1.1 MCP Trigger & Tooling Exposure**  
Entry via MCP and exposure of tools (GitHub issue fetch, PR creation subworkflow tool).

**1.2 Context Extraction (AI → structured output)**  
Gemini + structured output parser extract `{owner, repo, branch_name, base_branch}` from incoming text.

**1.3 Repo/Branch Validation (GitHub API guardrail)**  
HTTP request checks that the repo exists and that the branch resolves to commits.

**1.4 Commit-to-PR Content Generation (summarize → LLM)**  
Commit messages are aggregated and passed to an LLM to produce “Title:” and “Description:” text.

**1.5 PR Creation (GitHub API)**  
POST to `/pulls` with parsed title/body and head/base branches.

**Note:** The workflow also includes an **Execute Workflow Trigger** entry point, implying it can be called as a sub-workflow (or tested) besides MCP.

---

## 2. Block-by-Block Analysis

### 2.1 MCP Trigger & Tooling Exposure
**Overview:**  
Starts the automation from an MCP webhook and exposes helper tools (GitHub issue fetch and a workflow tool) to the MCP environment.

**Nodes involved:**
- MCP Server Trigger for Github
- Fetch GitHub Issues
- create_github_pr

#### Node: MCP Server Trigger for Github
- **Type / role:** `@n8n/n8n-nodes-langchain.mcpTrigger` — MCP webhook trigger / workflow entry for MCP.
- **Configuration (interpreted):**
  - Webhook path/id: `a2c1b2dd-32ed-463a-97c0-0f7139719c3c`
  - Acts as the MCP “server trigger” endpoint.
- **Connections:**
  - Provides **AI tool connections** to:
    - `Fetch GitHub Issues` (as `ai_tool`)
    - `create_github_pr` (as `ai_tool`)
- **Edge cases / failures:**
  - MCP not enabled or not connected to GitHub → no triggers.
  - Invalid webhook path / MCP routing misconfiguration → requests never reach the workflow.
- **Version notes:** Node `typeVersion: 2`.

#### Node: Fetch GitHub Issues
- **Type / role:** `n8n-nodes-base.githubTool` — a LangChain-compatible “tool” that can fetch GitHub issues for an agent.
- **Configuration (interpreted):**
  - Resource: `repository`
  - Operation: “Get repository issues” (filters empty; `returnAll: true`)
  - Owner URL points to `https://github.com/AhmedSAAhmed`
  - Repository URL points to `https://github.com/AhmedSAAhmed/Dog-Classifier-CNN`
  - Auth: GitHub **OAuth2**
  - Tool description: fetch issue details + full comment thread for better context.
- **Connections:**
  - Exposed as an MCP tool via `MCP Server Trigger for Github` (`ai_tool` link).
- **Edge cases / failures:**
  - OAuth scope missing (issues read) → 403.
  - Rate limits → 429.
  - Private repo without access → 404.
- **Version notes:** `typeVersion: 1.1`.

#### Node: create_github_pr
- **Type / role:** `@n8n/n8n-nodes-langchain.toolWorkflow` — exposes a workflow as an agent tool.
- **Configuration (interpreted):**
  - Points to workflow ID `Y6oy5VN_oASw3lZR2RpMk` (cached name “My workflow”).
  - Description instructs to provide full `owner/repo` so backend can route correctly.
  - No explicit input schema defined (empty).
- **Connections:**
  - Exposed to MCP trigger as an `ai_tool`.
- **Important note / risk:**
  - It references the **same workflow ID** as the current workflow (`Y6oy5VN_oASw3lZR2RpMk`). If invoked, it can cause recursion or unintended self-calls unless n8n prevents it or you intend it as a callable interface. In practice, you usually point this node to a *separate* “Create PR” workflow.
- **Version notes:** `typeVersion: 2.2`.

---

### 2.2 Context Extraction (AI → structured output)
**Overview:**  
Transforms free-form issue text into structured repository targeting information used downstream for API calls and PR creation.

**Nodes involved:**
- Subworkflow Entry Point
- Extract Repo & Branch Context
- Google Gemini Chat Model
- Structured Output Parser

#### Node: Subworkflow Entry Point
- **Type / role:** `n8n-nodes-base.executeWorkflowTrigger` — allows this workflow to be invoked by another workflow (“Execute Workflow”).
- **Configuration (interpreted):**
  - Input source: `passthrough` (takes incoming JSON as-is).
- **Connections:**
  - `Subworkflow Entry Point` → `Extract Repo & Branch Context` (main)
- **Edge cases / failures:**
  - If callers don’t pass the expected text fields, the agent may have insufficient context.
- **Version notes:** `typeVersion: 1.1`.

#### Node: Extract Repo & Branch Context
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — LLM agent used here as an extraction step.
- **Configuration (interpreted):**
  - Prompt instructs extraction of **four required fields**: `owner`, `repo`, `branch_name`, `base_branch`.
  - “Use defaults provided when info not explicitly mentioned” (but the workflow does not actually define explicit defaults in variables; defaults would need to be embedded in the incoming text or prompt expanded).
  - Has output parser enabled.
- **Key expressions/variables:**
  - Downstream nodes access `{{ $json.output.owner }}`, etc., meaning this node must output an `output` object.
- **Connections:**
  - Uses `Google Gemini Chat Model` as `ai_languageModel`.
  - Uses `Structured Output Parser` as `ai_outputParser`.
  - Main output → `Repo or Branch Validation`.
- **Edge cases / failures:**
  - Model returns incomplete fields → parser errors or missing `output.*`.
  - Ambiguous repo/branch names (“main” vs “master”; forks) → wrong extraction.
- **Version notes:** `typeVersion: 3.1`.

#### Node: Google Gemini Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` — chat LLM used by the extraction agent.
- **Configuration (interpreted):**
  - Temperature: `0.5` (balanced creativity/consistency).
  - Credentials: Google Gemini (PaLM) API.
- **Connections:**
  - Connected to `Extract Repo & Branch Context` as its language model.
- **Edge cases / failures:**
  - Invalid/expired API key → auth failure.
  - Safety blocks or quota issues.
- **Version notes:** `typeVersion: 1`.

#### Node: Structured Output Parser
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — enforces structured JSON output.
- **Configuration (interpreted):**
  - Schema example requires:
    - `owner`
    - `repo`
    - `branch_name`
    - `base_branch`
- **Connections:**
  - Provides `ai_outputParser` to `Extract Repo & Branch Context`.
- **Edge cases / failures:**
  - If LLM output doesn’t match schema shape, parsing fails.
- **Version notes:** `typeVersion: 1.3`.

---

### 2.3 Repo/Branch Validation (GitHub API guardrail)
**Overview:**  
Checks that the extracted repository and branch resolve via GitHub’s commits endpoint before proceeding.

**Nodes involved:**
- Repo or Branch Validation
- Check for Repository error
- Build Error Response

#### Node: Repo or Branch Validation
- **Type / role:** `n8n-nodes-base.httpRequest` — calls GitHub REST API to fetch commits for a repo/branch.
- **Configuration (interpreted):**
  - URL: `https://api.github.com/repos/{owner}/{repo}/commits`
  - Query parameter: `sha={branch_name}`
  - Authentication: predefined credential type `githubApi`
  - **onError:** `continueErrorOutput` (important: errors still produce output)
- **Key expressions:**
  - URL uses `{{$json.output.owner}}` and `{{$json.output.repo}}` from the extraction output.
  - Query param uses `{{$json.output.branch_name}}`.
- **Connections:**
  - Main output → `Check for Repository error`.
- **Edge cases / failures:**
  - 404 for repo or branch.
  - 401/403 for auth/scope problems.
  - Rate limiting (429).
  - If error output shape differs, `Check for Repository error` may mis-detect.
- **Version notes:** `typeVersion: 4.3`.

#### Node: Check for Repository error
- **Type / role:** `n8n-nodes-base.if` — branches depending on presence of an error `message`.
- **Configuration (interpreted):**
  - Condition: `{{$json.message}}` **is not empty**
  - If true → considered “error” path (because GitHub error responses commonly include a `message` field).
- **Connections:**
  - Output 1 (true): → `Prepare PR Payload` (but see note below)
  - Output 2 (false): → `Build Error Response`
- **Important logic concern (likely inverted):**
  - A successful GitHub commits response generally does **not** have a top-level `message` field; an error often does (e.g., `"message": "Not Found"`).
  - As configured, **“message not empty” routes to the ‘success’ side** (`Prepare PR Payload`) and “message empty” routes to error response. This appears reversed.
- **Edge cases / failures:**
  - Some successful responses might contain `message` in nested objects; but top-level usually not.
- **Version notes:** `typeVersion: 2.3`.

#### Node: Build Error Response
- **Type / role:** `n8n-nodes-base.set` — constructs a user-facing error string.
- **Configuration (interpreted):**
  - Sets `result` to an error message referencing repo and branch.
- **Key expression issue:**
  - The expression references a node named **`Extract Repository Info`**, but the actual node name is **`Extract Repo & Branch Context`**. This will cause an expression resolution failure at runtime unless a node with that name exists.
- **Connections:**
  - Output → `Aggregate Commit Messages` (this is also logically inconsistent: error text is sent into commit aggregation).
- **Edge cases / failures:**
  - Expression errors due to wrong node name.
  - Even if fixed, routing error payload into commit summarization is likely unintended.
- **Version notes:** `typeVersion: 3.4`.

---

### 2.4 Commit-to-PR Content Generation (summarize → LLM)
**Overview:**  
Aggregates commit messages into a single text blob and asks Gemini to write a PR title + description in a strict plain-text format.

**Nodes involved:**
- Prepare PR Payload
- Aggregate Commit Messages
- Generate PR Title & Description
- LLM PR Writer Model

#### Node: Prepare PR Payload
- **Type / role:** `n8n-nodes-base.set` — maps fields for downstream summarization/LLM.
- **Configuration (interpreted):**
  - Sets:
    - `owner`, `repo`, `branch_name`, `base_branch` from extraction output
    - `message` from `{{$json.commit.message}}`
- **Critical data-shape concern:**
  - `Repo or Branch Validation` returns commit objects from GitHub with a structure like `{ commit: { message: "..." }, ... }`, but **only if the flow reaches this node**.
  - Because the IF logic appears inverted, this node may be receiving an error object instead, where `$json.commit.message` doesn’t exist.
  - Also, the node pulls extraction fields using `$('Extract Repository Info')...` which again does not match actual name (`Extract Repo & Branch Context`). In this node, it references `$('Extract Repository Info')` indirectly via expressions:
    - `={{ $('Extract Repository Info').item.json.output.owner }}` etc. (wrong node name → failure).
- **Connections:**
  - (Implicit) feeds into `Aggregate Commit Messages` through normal flow (via connections from IF).
- **Version notes:** `typeVersion: 3.4`.

#### Node: Aggregate Commit Messages
- **Type / role:** `n8n-nodes-base.summarize` — concatenates commit message fields.
- **Configuration (interpreted):**
  - Field summarized: `message`
  - Aggregation: `concatenate`
  - Separator: newline
- **Connections:**
  - Output → `Generate PR Title & Description`
- **Edge cases / failures:**
  - If `message` is missing, output can be empty.
  - If many commits, concatenated text can exceed LLM context limits.
- **Version notes:** `typeVersion: 1.1`.

#### Node: Generate PR Title & Description
- **Type / role:** `@n8n/n8n-nodes-langchain.chainLlm` — single-prompt LLM call to produce PR title+description.
- **Configuration (interpreted):**
  - Prompt injects: `{{ $('summarize_commits').first().json.concatenated_message }}`
  - Output format required:
    - `Title: ...`
    - `Description:\n...`
  - Explicitly requires **plain text only**, no markdown.
- **Key expression issue:**
  - It references a node called **`summarize_commits`**, but the actual node name is **`Aggregate Commit Messages`**. Unless the node was renamed previously and not reflected here, this will break.
- **Connections:**
  - Uses `LLM PR Writer Model` as `ai_languageModel`.
  - Main output → `Create Pull Request`
- **Edge cases / failures:**
  - Model deviates from required format → downstream string splitting fails.
  - Output may include extra “Title:” occurrences causing split parsing issues.
- **Version notes:** `typeVersion: 1.9`.

#### Node: LLM PR Writer Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` — Gemini model used for PR writing.
- **Configuration (interpreted):**
  - Default options (temperature not specified here).
  - Credentials: same Gemini account as earlier.
- **Connections:**
  - Connected to `Generate PR Title & Description` as language model.
- **Edge cases / failures:**
  - Quota/auth/safety restrictions.
- **Version notes:** `typeVersion: 1`.

---

### 2.5 PR Creation (GitHub API)
**Overview:**  
Creates the pull request by posting to GitHub’s pulls endpoint and parsing the LLM output into title/body.

**Nodes involved:**
- Create Pull Request

#### Node: Create Pull Request
- **Type / role:** `n8n-nodes-base.httpRequest` — GitHub REST API call to create a PR.
- **Configuration (interpreted):**
  - Method: `POST`
  - URL: `https://api.github.com/repos/{owner}/{repo}/pulls`
  - Auth: predefined credential type `githubApi`
  - Body parameters:
    - `title`: parsed from LLM output using string splits
    - `head`: `branch_name`
    - `base`: `base_branch`
    - `body`: parsed description part from LLM output
- **Key expressions (fragile parsing):**
  - Title: `text.split('Title:')[1].split('Description:')[0].trim()`
  - Body: `text.split('Description:')[1].trim()`
- **Connections:**
  - Terminal node (no outgoing connections).
- **Edge cases / failures:**
  - If LLM output doesn’t contain both markers exactly → runtime error (cannot read `[1]`).
  - GitHub errors:
    - 422 if head/base invalid or no commits between branches
    - 403 if missing scopes
    - 404 for repo access issues
- **Version notes:** `typeVersion: 4.3`.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| MCP Server Trigger for Github | `@n8n/n8n-nodes-langchain.mcpTrigger` | MCP entry point / webhook trigger | — | (AI tools to) Fetch GitHub Issues; create_github_pr | ## MCP Issue Trigger\n\nListens for new or updated GitHub issues via MCP and starts the automation flow. |
| Fetch GitHub Issues | `n8n-nodes-base.githubTool` | MCP tool: fetch issues + comments | MCP Server Trigger for Github (ai_tool) | — | ## How it works\n\nA GitHub issue triggers the MCP server.\n\nMCP forwards the issue context to this workflow.\n\nRepository details are extracted using AI.\n\nCommits from the target branch are fetched.\n\nCommit messages are summarized into intent.\n\nAn LLM generates a PR title and description.\n\nA pull request is created via GitHub API.\n\n## Setup steps\n\nEnable MCP and connect it to your GitHub repo.\n\nConfigure GitHub API credentials with PR access.\n\nSet default base branch, for example main.\n\nConnect an LLM, such as Google Gemini.\n\nEnsure the sub-workflow is callable and active.\n\n## Customization\n\nModify prompts to align with your issue template.\n\nEnforce branch or naming rules before PR creation.\n\nAdd guards to skip draft or WIP issues. |
| create_github_pr | `@n8n/n8n-nodes-langchain.toolWorkflow` | MCP tool: callable workflow to create PR | MCP Server Trigger for Github (ai_tool) | — | ## How it works\n\nA GitHub issue triggers the MCP server.\n\nMCP forwards the issue context to this workflow.\n\nRepository details are extracted using AI.\n\nCommits from the target branch are fetched.\n\nCommit messages are summarized into intent.\n\nAn LLM generates a PR title and description.\n\nA pull request is created via GitHub API.\n\n## Setup steps\n\nEnable MCP and connect it to your GitHub repo.\n\nConfigure GitHub API credentials with PR access.\n\nSet default base branch, for example main.\n\nConnect an LLM, such as Google Gemini.\n\nEnsure the sub-workflow is callable and active.\n\n## Customization\n\nModify prompts to align with your issue template.\n\nEnforce branch or naming rules before PR creation.\n\nAdd guards to skip draft or WIP issues. |
| Subworkflow Entry Point | `n8n-nodes-base.executeWorkflowTrigger` | Entry point for Execute Workflow calls | — | Extract Repo & Branch Context | ## Context & Intelligence\n\nReceives data from MCP, extracts repository and branch context, fetches commits, and prepares structured input for AI reasoning. |
| Extract Repo & Branch Context | `@n8n/n8n-nodes-langchain.agent` | Extract `{owner, repo, branch_name, base_branch}` from text | Subworkflow Entry Point | Repo or Branch Validation | ## Context & Intelligence\n\nReceives data from MCP, extracts repository and branch context, fetches commits, and prepares structured input for AI reasoning. |
| Google Gemini Chat Model | `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` | LLM for extraction agent | — | (to) Extract Repo & Branch Context (ai_languageModel) | ## Context & Intelligence\n\nReceives data from MCP, extracts repository and branch context, fetches commits, and prepares structured input for AI reasoning. |
| Structured Output Parser | `@n8n/n8n-nodes-langchain.outputParserStructured` | Enforce structured extraction output | — | (to) Extract Repo & Branch Context (ai_outputParser) | ## Context & Intelligence\n\nReceives data from MCP, extracts repository and branch context, fetches commits, and prepares structured input for AI reasoning. |
| Repo or Branch Validation | `n8n-nodes-base.httpRequest` | Validate repo/branch by listing commits | Extract Repo & Branch Context | Check for Repository error | ## Context & Intelligence\n\nReceives data from MCP, extracts repository and branch context, fetches commits, and prepares structured input for AI reasoning. |
| Check for Repository error | `n8n-nodes-base.if` | Branch on error presence (`message`) | Repo or Branch Validation | Prepare PR Payload; Build Error Response | ## PR Generation & Execution\n\nTransforms commit context into a PR title and description, validates output, and creates the pull request via GitHub API. |
| Build Error Response | `n8n-nodes-base.set` | Build error string | Check for Repository error | Aggregate Commit Messages | ## PR Generation & Execution\n\nTransforms commit context into a PR title and description, validates output, and creates the pull request via GitHub API. |
| Prepare PR Payload | `n8n-nodes-base.set` | Map commit message + repo context | Check for Repository error | Aggregate Commit Messages (implicit via flow) | ## PR Generation & Execution\n\nTransforms commit context into a PR title and description, validates output, and creates the pull request via GitHub API. |
| Aggregate Commit Messages | `n8n-nodes-base.summarize` | Concatenate commit messages | Build Error Response / Prepare PR Payload | Generate PR Title & Description | ## PR Generation & Execution\n\nTransforms commit context into a PR title and description, validates output, and creates the pull request via GitHub API. |
| Generate PR Title & Description | `@n8n/n8n-nodes-langchain.chainLlm` | Generate PR text from commit messages | Aggregate Commit Messages | Create Pull Request | ## PR Generation & Execution\n\nTransforms commit context into a PR title and description, validates output, and creates the pull request via GitHub API. |
| LLM PR Writer Model | `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` | LLM for PR writing | — | (to) Generate PR Title & Description (ai_languageModel) | ## PR Generation & Execution\n\nTransforms commit context into a PR title and description, validates output, and creates the pull request via GitHub API. |
| Create Pull Request | `n8n-nodes-base.httpRequest` | GitHub API call to create PR | Generate PR Title & Description | — | ## PR Generation & Execution\n\nTransforms commit context into a PR title and description, validates output, and creates the pull request via GitHub API. |
| Sticky Note | `n8n-nodes-base.stickyNote` | Documentation | — | — | ## How it works\n\nA GitHub issue triggers the MCP server.\n\nMCP forwards the issue context to this workflow.\n\nRepository details are extracted using AI.\n\nCommits from the target branch are fetched.\n\nCommit messages are summarized into intent.\n\nAn LLM generates a PR title and description.\n\nA pull request is created via GitHub API.\n\n## Setup steps\n\nEnable MCP and connect it to your GitHub repo.\n\nConfigure GitHub API credentials with PR access.\n\nSet default base branch, for example main.\n\nConnect an LLM, such as Google Gemini.\n\nEnsure the sub-workflow is callable and active.\n\n## Customization\n\nModify prompts to align with your issue template.\n\nEnforce branch or naming rules before PR creation.\n\nAdd guards to skip draft or WIP issues. |
| Sticky Note1 | `n8n-nodes-base.stickyNote` | Documentation | — | — | ## MCP Issue Trigger\n\nListens for new or updated GitHub issues via MCP and starts the automation flow. |
| Sticky Note2 | `n8n-nodes-base.stickyNote` | Documentation | — | — | ## Context & Intelligence\n\nReceives data from MCP, extracts repository and branch context, fetches commits, and prepares structured input for AI reasoning. |
| Sticky Note3 | `n8n-nodes-base.stickyNote` | Documentation | — | — | ## PR Generation & Execution\n\nTransforms commit context into a PR title and description, validates output, and creates the pull request via GitHub API. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
   - Name it (per your title): “Convert GitHub commits into review-ready pull requests with Google Gemini”.
   - (Optional) Keep it inactive until credentials and testing are complete.

2) **Add MCP trigger**
   - Add node: **MCP Server Trigger** (`mcpTrigger`).
   - Set **Path** to a unique value (n8n generates a UUID-like string).
   - In n8n MCP settings, ensure MCP is enabled and this trigger is reachable.

3) **(Optional) Add GitHub Issues tool for MCP**
   - Add node: **GitHub Tool**.
   - Configure:
     - Authentication: **OAuth2** GitHub credential (must have repo/issues read).
     - Resource: Repository → Get issues (include comments/thread if node supports it).
     - Set owner/repo or leave dynamic depending on your MCP design.
   - Connect it to the MCP trigger via the **ai_tool** connection.

4) **(Optional) Add a “Workflow Tool” for PR creation**
   - Add node: **Tool Workflow** (`toolWorkflow`).
   - Select a **separate workflow** dedicated to PR creation (recommended to avoid self-recursion).
   - Define input schema (recommended): `owner`, `repo`, `head`, `base`, `title`, `body`.
   - Connect it to MCP trigger via **ai_tool**.

5) **Add Execute Workflow Trigger (subworkflow entry)**
   - Add node: **Execute Workflow Trigger**.
   - Set **Input Source** = `passthrough`.
   - This lets other workflows call it directly.

6) **Add Gemini model for extraction**
   - Add node: **Google Gemini Chat Model**.
   - Set temperature ~ `0.5`.
   - Create/attach **Google Gemini (PaLM) API credential**.

7) **Add Structured Output Parser**
   - Add node: **Structured Output Parser**.
   - Provide a schema example:
     - `owner` (string)
     - `repo` (string)
     - `branch_name` (string)
     - `base_branch` (string)

8) **Add extraction agent**
   - Add node: **AI Agent** (`agent`).
   - Prompt: instruct it to extract the four fields and to apply defaults when missing (embed your defaults explicitly, e.g., “default base_branch = main”).
   - Enable **Output Parser**.
   - Connect:
     - Execute Workflow Trigger → Agent (main)
     - Gemini model → Agent (`ai_languageModel`)
     - Structured Output Parser → Agent (`ai_outputParser`)

9) **Add repo/branch validation request**
   - Add node: **HTTP Request**.
   - Method: GET
   - URL: `https://api.github.com/repos/{{$json.output.owner}}/{{$json.output.repo}}/commits`
   - Query param: `sha = {{$json.output.branch_name}}`
   - Auth: **GitHub API** predefined credential (token/OAuth with `repo` scope for private repos; `public_repo` may suffice for public PRs).
   - Set **On Error** = “Continue (error output)”.

10) **Add IF guard**
   - Add node: **IF**.
   - Recommended logic (fixing the inversion):
     - If `{{$json.message}}` **is empty** → success path
     - Else → error path
   - Connect:
     - Validation → IF

11) **Success path: prepare commit items**
   - Add node: **Set** (“Prepare PR Payload”).
   - Set fields:
     - `owner` = `{{$('Extract Repo & Branch Context').item.json.output.owner}}`
     - `repo` = `{{$('Extract Repo & Branch Context').item.json.output.repo}}`
     - `branch_name` = `{{$('Extract Repo & Branch Context').item.json.output.branch_name}}`
     - `base_branch` = `{{$('Extract Repo & Branch Context').item.json.output.base_branch}}`
     - `message` = `{{$json.commit.message}}` (assuming each input item is a commit object)
   - Connect IF (success) → Set.

12) **Error path: build error result**
   - Add node: **Set** (“Build Error Response”).
   - Set `result` to a clear message using the correct node name:
     - `{{$('Extract Repo & Branch Context').item.json.output.owner}}/...`
   - Connect IF (error) → Build Error Response.
   - Decide where to send this next (recommended: return/stop, not to summarization).

13) **Aggregate commit messages**
   - Add node: **Summarize**.
   - Field: `message`
   - Aggregation: concatenate with newline separator.
   - Connect Set (success) → Summarize.

14) **Add Gemini model for PR writing**
   - Add node: **Google Gemini Chat Model** (“LLM PR Writer Model”).
   - Attach same (or different) Gemini credential; set temperature as desired.

15) **Generate PR text**
   - Add node: **Chain LLM** (“Generate PR Title & Description”).
   - Prompt includes concatenated commit messages, for example:
     - `{{ $('Aggregate Commit Messages').first().json.concatenated_message }}`
   - Require strict output format:
     - `Title: ...`
     - `Description:\n...`
   - Connect:
     - Summarize → Chain LLM (main)
     - PR writer model → Chain LLM (`ai_languageModel`)

16) **Create PR via GitHub API**
   - Add node: **HTTP Request** (“Create Pull Request”).
   - Method: POST
   - URL: `https://api.github.com/repos/{{$('Extract Repo & Branch Context').item.json.output.owner}}/{{$('Extract Repo & Branch Context').item.json.output.repo}}/pulls`
   - Body params:
     - `title` parsed from LLM output
     - `head` = extracted `branch_name`
     - `base` = extracted `base_branch`
     - `body` parsed from LLM output
   - Credential: GitHub API credential with PR creation permissions.
   - Connect Chain LLM → Create PR.

17) **Add robust parsing guards (recommended)**
   - Before creating PR, add an IF or Code node to verify:
     - Output contains both `Title:` and `Description:`
     - Title length <= 72 chars
   - Fallback: regenerate with stricter prompt or return an error.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “How it works / Setup steps / Customization” guidance is embedded in sticky notes. | Internal workflow documentation (sticky notes). |
| MCP must be enabled and connected to GitHub for issue events to reach the workflow. | n8n MCP configuration + GitHub integration. |
| Several expressions reference node names that do not exist (`Extract Repository Info`, `summarize_commits`) and the IF logic appears inverted; these should be corrected for the workflow to run end-to-end. | Consistency + runtime reliability. |
| The `toolWorkflow` node points to the same workflow ID, which may cause recursion; typically you point it to a separate callable workflow. | Architecture / safety consideration. |