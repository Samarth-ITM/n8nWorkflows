Use skills In n8n agent node

https://n8nworkflows.xyz/workflows/use-skills-in-n8n-agent-node-13270


# Use skills In n8n agent node

## 1. Workflow Overview

**Purpose:**  
This workflow implements an n8n **AI Agent** that can “use skills” stored as markdown files in public GitHub repositories. When a chat message arrives, the workflow fetches and summarizes the directory structure of the configured skills repositories, injects that structure into the agent’s **system message**, and gives the agent two tools to (1) browse directories and (2) fetch raw file content on demand.

**Target use cases:**
- Build an agent that follows instructions from external “skill files” rather than relying on general model knowledge.
- Mimic “skills” behavior similar to Claude Code / Open Code: browse repo directories, open relevant markdown files, then execute instructions.

### 1.1 Chat Input Reception
Receives a message from n8n chat and starts the workflow.

### 1.2 Skills Repo Configuration
Defines which GitHub repos contain skills, then fans out processing per repo.

### 1.3 Skills Directory Discovery & Cleaning
Lists root and `/skills` directories (if present), filters out irrelevant entries, and removes errors.

### 1.4 Merge & Publish Directory Structure to Agent
Merges the cleaned listings into a single stream used by the AI Agent’s system prompt to show “Available Skills”.

### 1.5 AI Agent + Tools (Directory Browse + File Fetch)
Runs an agent using an OpenRouter chat model and a memory buffer, with tools to list GitHub paths and retrieve raw file text via GitHub API.

---

## 2. Block-by-Block Analysis

### Block 1 — Chat Input Reception

**Overview:**  
Starts the workflow when a chat message is received, providing the user’s text as the agent input.

**Nodes Involved:**  
- When chat message received

**Node Details**

1) **When chat message received**  
- **Type / role:** `@n8n/n8n-nodes-langchain.chatTrigger` — entry trigger for n8n chat interactions.  
- **Key configuration:** No special options set.  
- **Key variables/fields:** Provides `item.json.chatInput` (used later by the AI Agent).  
- **Outputs:** Connected to **Set GitHub Repo URLs**.  
- **Potential failures / edge cases:**  
  - If used outside Chat or Chat Hub context, it won’t trigger.
  - If `chatInput` is empty, the agent will receive an empty prompt.

**Sticky note context (covers this area):**  
- “Interact with Chat… enable `Make Available in n8n Chat Hub`… [chat hub](https://www.youtube.com/watch?v=s_GHteEnHc4)”

---

### Block 2 — Skills Repo Configuration (Repo list → fan-out)

**Overview:**  
Creates an array of GitHub repository URLs that contain skills, then splits the array into one item per repo for downstream GitHub listing.

**Nodes Involved:**  
- Set GitHub Repo URLs  
- Split Out

**Node Details**

1) **Set GitHub Repo URLs**  
- **Type / role:** `n8n-nodes-base.set` — defines `SKILLS_REPOS` as an array.  
- **Configuration choices:** Assigns a single field:
  - `SKILLS_REPOS` (array) set to:
    - `https://github.com/anthropics/skills`
    - `https://github.com/anthropics/knowledge-work-plugins`
- **Outputs:** To **Split Out**.  
- **Edge cases / failures:**  
  - If a URL is malformed or not a GitHub repo URL, downstream GitHub listing nodes may error.
  - Private repos will fail unless credentials have access.

**Sticky note context:**  
- “TODO Add Skills… Preselected is Anthropic’s example Skills Repo and Knowledge Work Skills Repo… ensure to match the same array syntax” with links:
  - https://github.com/anthropics/skills  
  - https://github.com/anthropics/knowledge-work-plugins

2) **Split Out**  
- **Type / role:** `n8n-nodes-base.splitOut` — turns the `SKILLS_REPOS` array into multiple items (one per repo).  
- **Configuration choices:** Splits by field `SKILLS_REPOS`.  
- **Outputs:** Sends each repo item to both:
  - **List Root Dirs**
  - **List Skills Dirs**
- **Edge cases / failures:**  
  - If `SKILLS_REPOS` is missing or not an array, split will produce no items or error depending on runtime conditions.

---

### Block 3 — Skills Directory Discovery & Cleaning

**Overview:**  
For each repo, the workflow lists the repository root and also tries `/skills`. It filters out dotfiles and excludes the `skills` directory from the root listing to reduce noise. Errors from the `/skills` lookup are tolerated and filtered out.

**Nodes Involved:**  
- List Root Dirs  
- Remove Skills Dirs & Dot Files  
- List Skills Dirs  
- Remove Errors and Dot Files

**Node Details**

1) **List Root Dirs**  
- **Type / role:** `n8n-nodes-base.github` (File → List) — lists contents of repo root `/`.  
- **Configuration choices (interpreted):**
  - **Owner** and **Repository** are both set from the current item’s `SKILLS_REPOS` using “URL mode” resource locator.
  - `filePath: "/"`, `operation: list`, `resource: file`.
- **Credentials:** GitHub API credential `liamdmcgarrigle read only`.  
- **Outputs:** To **Remove Skills Dirs & Dot Files**.  
- **Potential failures:**  
  - Auth/permission errors (401/403), rate limiting (403), repo not found (404).
  - If the URL parsing/resolution fails, owner/repo may be incorrect.

2) **Remove Skills Dirs & Dot Files**  
- **Type / role:** `n8n-nodes-base.filter` — cleans the root listing.  
- **Configuration choices:** Keeps entries where:
  - `name != "skills"`
  - `name` does **not** start with `.`
- **Outputs:** To **Merge Directory Structures** (input index 0).  
- **Edge cases:**  
  - If GitHub returns entries without a `name` field (unexpected), filter expressions may behave unexpectedly.
  - This intentionally removes the `skills` folder from root because `/skills` is listed separately.

3) **List Skills Dirs**  
- **Type / role:** `n8n-nodes-base.github` (File → List) — attempts to list `/skills`.  
- **Configuration choices:**
  - Same owner/repo resolution from `SKILLS_REPOS`.
  - `filePath: "/skills"`.
  - `onError: continueRegularOutput` — important: errors are returned as normal output items rather than stopping execution.
- **Credentials:** Same GitHub API credential.  
- **Outputs:** To **Remove Errors and Dot Files**.  
- **Potential failures:**  
  - `/skills` may not exist in some repos (404). This is expected and handled via `continueRegularOutput`.

4) **Remove Errors and Dot Files**  
- **Type / role:** `n8n-nodes-base.filter` — removes error items and dotfiles from the `/skills` listing path.  
- **Configuration choices:** Keeps entries where:
  - `error` does **not** exist
  - `name` does **not** start with `.`
- **Outputs:** To **Merge Directory Structures** (input index 1).  
- **Edge cases:**  
  - If GitHub changes error payload shape, the `error` existence check might not catch all failures.

**Sticky note context (covers this phase):**  
- “Fetch & Prepare Skills for Agent… gets directory… removes duplicates & irrelevant files… gets nested `/skills` directory if it exists… merge all data…”

*(Note: although the sticky note mentions removing duplicates, there is no explicit de-duplication node in this JSON. Any de-duplication would need to be added manually if required.)*

---

### Block 4 — Merge & Publish Directory Structure to Agent

**Overview:**  
Combines the cleaned root listings and `/skills` listings into a single merged stream that the AI Agent uses to build its “Available Skills” directory overview.

**Nodes Involved:**  
- Merge Directory Structures

**Node Details**

1) **Merge Directory Structures**  
- **Type / role:** `n8n-nodes-base.merge` — merges two inputs into one output stream.  
- **Configuration choices:** Default merge behavior (no explicit mode shown in parameters). In n8n v3.x, default is typically “Append” behavior when used as two-input combiner unless changed in UI.  
- **Inputs:**  
  - Input 0 from **Remove Skills Dirs & Dot Files** (root cleaned listing)  
  - Input 1 from **Remove Errors and Dot Files** (`/skills` cleaned listing)  
- **Outputs:** To **AI Agent** (main input).  
- **Edge cases / failures:**  
  - If one branch produces no items, merge behavior depends on mode; verify it still emits items as expected.
  - If you later add additional repo listing sources, you may need additional merges or a different aggregation strategy.

---

### Block 5 — AI Agent + Model + Memory + Tools

**Overview:**  
Runs an AI Agent with a system message that instructs it to use skill files. The system message includes a dynamically generated list of available files/paths from the merged GitHub directory listings. The agent is equipped with two tools: list files by path (GitHub) and fetch raw file content (HTTP request). A memory buffer is attached for conversational continuity.

**Nodes Involved:**  
- AI Agent  
- Chat Model  
- Simple Memory  
- List Files by Path Name (tool)  
- Get a File From GitHub (tool)

**Node Details**

1) **AI Agent**  
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — orchestration agent that can call tools.  
- **Key configuration choices:**
  - **User input text:** `{{ $('When chat message received').item.json.chatInput }}`
  - **Max iterations:** `150` (very high; allows extensive tool exploration but increases cost/time and risk of looping)
  - **Prompt type:** `define` (custom system prompt used)
  - **executeOnce:** `true` (node runs once for the workflow execution)
  - **System message (core behavior):**
    - Tells the agent it **must** use skill files rather than general knowledge.
    - Provides step process: identify skill(s) → traverse directories using tool → fetch files → follow instructions.
    - Injects **Available Skills Files and Directories** generated from `Merge Directory Structures`:
      - Maps each merged item to `{ type, path, orgName, repoName }` derived partly from `item.json.url.split('/')[4]` and `[5]`.
      - Filters out entries where the last path segment is `readme.md`, `template`, or `license` (case-insensitive).
- **Inputs / connections:**
  - **Main input:** from **Merge Directory Structures**
  - **Language model input:** from **Chat Model** (`ai_languageModel`)
  - **Memory input:** from **Simple Memory** (`ai_memory`)
  - **Tools input:** from **List Files by Path Name** and **Get a File From GitHub** (`ai_tool`)
- **Edge cases / failures:**
  - If GitHub listing items do not have `url` in the expected format, `split('/')[4]` and `[5]` can produce wrong org/repo names or undefined.
  - The expression uses `i.path.toLowerCase().split('/').last()` — `last()` is not standard JS; it works only if n8n’s expression runtime provides it. If not available in your n8n version, this will fail. Safer alternative: `split('/').slice(-1)[0]`.
  - High `maxIterations` can lead to long runs/timeouts depending on your n8n execution limits.

**Sticky note context:**  
- “Just a Regular Ol Agent… build up from here…”
- “How the Agent Uses the Skills… uses `List Files by Path Name`… `Get a File From GitHub` returns full text… mimics Claude Code or Open Code”

2) **Chat Model**  
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenRouter` — provides the LLM for the agent via OpenRouter.  
- **Configuration choices:**
  - Model: `google/gemini-3-flash-preview`
  - Temperature: `0.1` (low variance, more deterministic)
- **Credentials:** OpenRouter API credential `n8n-general-use-mcgarrigle`.  
- **Outputs:** Connects to **AI Agent** via `ai_languageModel`.  
- **Potential failures:**  
  - Invalid/expired OpenRouter key, model name not available, rate limits, provider outages.

**Sticky note context:**  
- “Replace Me… replace with provider and model you prefer… [AI Benchmark](https://n8n.io/ai-benchmark)”

3) **Simple Memory**  
- **Type / role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow` — conversation memory window.  
- **Configuration choices:** Defaults (window size not specified here).  
- **Outputs:** To **AI Agent** via `ai_memory`.  
- **Edge cases:**  
  - If the memory window is too small/large for your needs, the agent may forget context or spend tokens repeating.

4) **List Files by Path Name** (Tool)  
- **Type / role:** `n8n-nodes-base.githubTool` — exposes GitHub “list directory contents” as an agent tool.  
- **Configuration choices:**
  - Owner: from AI variable `$fromAI('repoUsername')`
  - Repo: `$fromAI('repoName')`
  - Path: `$fromAI('Path', "...", 'string')` (AI-provided path; `/` for root)
  - Tool description explicitly instructs: “Use this tool on any DIR…”
- **Credentials:** GitHub API credential (read-only).  
- **Outputs:** Connected to **AI Agent** as `ai_tool`.  
- **Edge cases / failures:**  
  - The agent may pass invalid paths; GitHub returns 404.
  - Rate limiting if the agent explores many directories (especially with maxIterations=150).

**Sticky note context:**  
- “This lets the agent look through the directories to find relevant files”

5) **Get a File From GitHub** (Tool)  
- **Type / role:** `n8n-nodes-base.httpRequestTool` — exposes a raw HTTP GET to GitHub contents API as an agent tool, returning **text** rather than base64.  
- **Configuration choices:**
  - URL constructed from AI variables:
    - `repoOwnerUsername`, `repoName`, `File_Path`
    - Endpoint pattern: `https://api.github.com/repos/{owner}/{repo}/contents/{path}`
  - Headers:
    - `Accept: application/vnd.github.raw+json` (requests raw content)
    - `X-GitHub-Api-Version: 2022-11-28`
  - Authentication: predefined credential type `githubApi`
- **Outputs:** Connected to **AI Agent** as `ai_tool`.  
- **Edge cases / failures:**  
  - If file is binary or large, response may be unusable or truncated depending on limits.
  - Wrong Accept header could return JSON metadata/base64 instead of raw text.
  - GitHub API rate limits and 404s for wrong paths.

**Sticky note context:**  
- “HTTP request is used here instead of… GitHub node because the node returns base 64…”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger | Chat entry trigger | — | Set GitHub Repo URLs | Interact with Chat… enable `Make Available in n8n Chat Hub`… https://www.youtube.com/watch?v=s_GHteEnHc4 |
| Set GitHub Repo URLs | n8n-nodes-base.set | Define skills repo URL list | When chat message received | Split Out | TODO Add Skills… https://github.com/anthropics/skills and https://github.com/anthropics/knowledge-work-plugins …match array syntax |
| Split Out | n8n-nodes-base.splitOut | Fan-out per repo | Set GitHub Repo URLs | List Root Dirs; List Skills Dirs | Fetch & Prepare Skills for Agent… gets directory… removes irrelevant… gets `/skills`… merge data |
| List Root Dirs | n8n-nodes-base.github | List root directory contents | Split Out | Remove Skills Dirs & Dot Files | Fetch & Prepare Skills for Agent… gets directory… removes irrelevant… gets `/skills`… merge data |
| Remove Skills Dirs & Dot Files | n8n-nodes-base.filter | Filter root listing | List Root Dirs | Merge Directory Structures | Fetch & Prepare Skills for Agent… gets directory… removes irrelevant… gets `/skills`… merge data |
| List Skills Dirs | n8n-nodes-base.github | List `/skills` contents (if exists) | Split Out | Remove Errors and Dot Files | Fetch & Prepare Skills for Agent… gets directory… removes irrelevant… gets `/skills`… merge data |
| Remove Errors and Dot Files | n8n-nodes-base.filter | Drop GitHub errors and dotfiles | List Skills Dirs | Merge Directory Structures | Fetch & Prepare Skills for Agent… gets directory… removes irrelevant… gets `/skills`… merge data |
| Merge Directory Structures | n8n-nodes-base.merge | Merge root + `/skills` listings | Remove Skills Dirs & Dot Files; Remove Errors and Dot Files | AI Agent | Fetch & Prepare Skills for Agent… gets directory… removes irrelevant… gets `/skills`… merge data |
| AI Agent | @n8n/n8n-nodes-langchain.agent | Tool-using agent; uses merged directory structure in system prompt | Merge Directory Structures (+ ai_languageModel + ai_memory + ai_tool) | — | Just a Regular Ol Agent… build up from here… / How the Agent Uses the Skills… mimics Claude Code or Open Code |
| Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenRouter | LLM provider (OpenRouter) | — | AI Agent (ai_languageModel) | Replace Me… pick provider/model… https://n8n.io/ai-benchmark |
| Simple Memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Conversation memory | — | AI Agent (ai_memory) |  |
| List Files by Path Name | n8n-nodes-base.githubTool | Agent tool: list dir by path | — (tool invoked by agent) | AI Agent (ai_tool) | This lets the agent look through the directories to find relevant files |
| Get a File From GitHub | n8n-nodes-base.httpRequestTool | Agent tool: fetch raw file text | — (tool invoked by agent) | AI Agent (ai_tool) | HTTP request used instead of GitHub node to get text (avoid base64) |
| Sticky Note | n8n-nodes-base.stickyNote | Comment | — | — | Interact with Chat… enable `Make Available in n8n Chat Hub`… https://www.youtube.com/watch?v=s_GHteEnHc4 |
| Sticky Note1 | n8n-nodes-base.stickyNote | Comment | — | — | HTTP request used instead of GitHub node to get text (avoid base64) |
| Sticky Note2 | n8n-nodes-base.stickyNote | Comment | — | — | This lets the agent look through the directories to find relevant files |
| Sticky Note3 | n8n-nodes-base.stickyNote | Comment | — | — | TODO Add Skills… https://github.com/anthropics/skills and https://github.com/anthropics/knowledge-work-plugins …match array syntax |
| Sticky Note4 | n8n-nodes-base.stickyNote | Comment | — | — | Just a Regular Ol Agent… build up from here… |
| Sticky Note5 | n8n-nodes-base.stickyNote | Comment | — | — | How the Agent Uses the Skills… uses `List Files by Path Name` + `Get a File From GitHub`… mimics Claude Code or Open Code |
| Sticky Note6 | n8n-nodes-base.stickyNote | Comment | — | — | Fetch & Prepare Skills for Agent… gets directory… removes irrelevant… gets `/skills`… merge data |
| Sticky Note7 | n8n-nodes-base.stickyNote | Comment | — | — | Replace Me… pick provider/model… https://n8n.io/ai-benchmark |

---

## 4. Reproducing the Workflow from Scratch

1) **Create the trigger**
   1. Add node: **When chat message received** (`Chat Trigger`).
   2. Keep default options.
   3. (Optional) Enable “Make Available in n8n Chat Hub” in node/UI if you want it accessible in Chat Hub.

2) **Define skills repositories**
   1. Add node: **Set GitHub Repo URLs** (`Set`).
   2. Add field:
      - Name: `SKILLS_REPOS`
      - Type: `Array`
      - Value:
        - `https://github.com/anthropics/skills`
        - `https://github.com/anthropics/knowledge-work-plugins`
   3. Connect: **When chat message received → Set GitHub Repo URLs**.

3) **Split repos into items**
   1. Add node: **Split Out** (`Split Out`).
   2. Set “Field to split out” = `SKILLS_REPOS`.
   3. Connect: **Set GitHub Repo URLs → Split Out**.

4) **List repository root**
   1. Add node: **List Root Dirs** (`GitHub`).
   2. Configure:
      - Resource: **File**
      - Operation: **List**
      - File Path: `/`
      - Owner: use **Resource Locator (URL mode)** referencing `{{$json.SKILLS_REPOS}}`
      - Repository: **URL mode** referencing `{{$json.SKILLS_REPOS}}`
   3. Set GitHub credentials (read-only is enough for listing).
   4. Connect: **Split Out → List Root Dirs**.

5) **Filter root listing**
   1. Add node: **Remove Skills Dirs & Dot Files** (`Filter`).
   2. Conditions (AND):
      - `{{$json.name}}` **not equals** `skills`
      - `{{$json.name}}` **not starts with** `.`
   3. Connect: **List Root Dirs → Remove Skills Dirs & Dot Files**.

6) **List `/skills` directory (tolerate missing)**
   1. Add node: **List Skills Dirs** (`GitHub`).
   2. Configure like List Root Dirs, but File Path = `/skills`.
   3. Set node setting **On Error** = “Continue (regular output)” (so missing `/skills` doesn’t fail the workflow).
   4. Connect: **Split Out → List Skills Dirs**.

7) **Filter `/skills` listing**
   1. Add node: **Remove Errors and Dot Files** (`Filter`).
   2. Conditions (AND):
      - `{{$json.error}}` **does not exist**
      - `{{$json.name}}` **not starts with** `.`
   3. Connect: **List Skills Dirs → Remove Errors and Dot Files**.

8) **Merge both directory streams**
   1. Add node: **Merge Directory Structures** (`Merge`).
   2. Leave default merge mode (verify it behaves as append/combined stream in your n8n version).
   3. Connect:
      - **Remove Skills Dirs & Dot Files → Merge Directory Structures** (Input 0)
      - **Remove Errors and Dot Files → Merge Directory Structures** (Input 1)

9) **Add the chat model**
   1. Add node: **Chat Model** (`OpenRouter Chat Model`).
   2. Select model: `google/gemini-3-flash-preview` (or replace with your preferred model).
   3. Set temperature: `0.1`.
   4. Configure OpenRouter credentials.

10) **Add memory**
   1. Add node: **Simple Memory** (`Memory Buffer Window`).
   2. Keep defaults (or adjust window size in node options if desired).

11) **Add agent tools**
   - **Tool A: List Files by Path Name**
     1. Add node: **List Files by Path Name** (`GitHub Tool`).
     2. Configure:
        - Resource: File; Operation: List
        - Owner: `{{$fromAI('repoUsername')}}`
        - Repository: `{{$fromAI('repoName')}}`
        - File Path: `{{$fromAI('Path', "`/` would be root, `/folder` would list files in the folder named folder", 'string')}}`
     3. Add tool description: “List the files and directories inside of dirs…”
     4. Attach GitHub API credentials.

   - **Tool B: Get a File From GitHub**
     1. Add node: **Get a File From GitHub** (`HTTP Request Tool`).
     2. Configure:
        - Authentication: **Predefined credentials** → GitHub API credential
        - URL:
          `https://api.github.com/repos/{{ $fromAI('repoOwnerUsername', '', 'string') }}/{{ $fromAI('repoName', '', 'string') }}/contents/{{ $fromAI('File_Path', '', 'string') }}`
        - Headers:
          - `Accept: application/vnd.github.raw+json`
          - `X-GitHub-Api-Version: 2022-11-28`
     3. Tool description: “Get text content from a file on github…”

12) **Create the agent**
   1. Add node: **AI Agent** (`Agent`).
   2. Set input text:
      - `{{ $('When chat message received').item.json.chatInput }}`
   3. Set options:
      - Max iterations: `150`
      - Prompt type: `Define`
      - System message: include instructions to (a) browse skills repos via tool, (b) fetch skill files, (c) follow skill file instructions, and inject available skills list.
   4. In the system message, dynamically inject directory structure from merge:
      - Use an expression that reads all items from **Merge Directory Structures**, maps to `{type, path, orgName, repoName}`, and filters out README/template/license.
   5. Connect:
      - **Merge Directory Structures → AI Agent** (main)
      - **Chat Model → AI Agent** (ai_languageModel)
      - **Simple Memory → AI Agent** (ai_memory)
      - **List Files by Path Name → AI Agent** (ai_tool)
      - **Get a File From GitHub → AI Agent** (ai_tool)

13) **Credentials checklist**
   - GitHub API credential with permission to read the target repos (public read is fine; consider rate limits).
   - OpenRouter API credential for the chosen model.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Interact with Chat… enable `Make Available in n8n Chat Hub`…” | https://www.youtube.com/watch?v=s_GHteEnHc4 |
| Preselected public skills repos | https://github.com/anthropics/skills ; https://github.com/anthropics/knowledge-work-plugins |
| Model/provider selection reference | https://n8n.io/ai-benchmark |
| Rationale for HTTP tool for file fetch (raw text vs base64) | Uses GitHub Contents API with `Accept: application/vnd.github.raw+json` |
| Disclaimer (provided by user) | Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n… (kept as context for compliance) |