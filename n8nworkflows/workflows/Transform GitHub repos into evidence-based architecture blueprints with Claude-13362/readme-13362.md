Transform GitHub repos into evidence-based architecture blueprints with Claude

https://n8nworkflows.xyz/workflows/transform-github-repos-into-evidence-based-architecture-blueprints-with-claude-13362


# Transform GitHub repos into evidence-based architecture blueprints with Claude

## 1. Workflow Overview

**Purpose:**  
This workflow takes a **public GitHub repository URL** and generates an **evidence-based system architecture blueprint** (Markdown + Mermaid diagrams + risks) using **Anthropic Claude**, then **pushes it back to the repo** as `README_ARCH.md`, optionally **notifies Slack**, and returns a **styled success page** to the caller.

**Target use cases:**
- Automated architecture documentation for repos
- Consistent “forensic” codebase summaries for audits, onboarding, or portfolio reviews
- AI-generated diagrams with strict anti-hallucination constraints

### 1.1 Input Reception (two entry points)
- Accepts a repo URL either via a **Webhook POST** (API usage) or an **n8n Form** (browser usage), both feeding into URL parsing.

### 1.2 Evidence Gathering (GitHub API)
- Fetches repository metadata (description, language, default branch, topics).
- Fetches recursive repo tree.
- Identifies “key files” and downloads their contents (limited set).
- Builds a compact “evidence package” (tree + truncated file contents) to feed the LLM.

### 1.3 AI Processing (Claude via LangChain nodes)
- Sends evidence to Claude with strict instructions: **no speculation** and **only claim what is present**.
- Produces a Markdown blueprint with Mermaid flowchart + sequence diagram + evidence-based risks.
- Applies Markdown sanitation fixes.

### 1.4 Delivery (GitHub commit + Slack + HTTP response)
- Wraps the analysis into a Markdown “blueprint document”.
- Fetches existing `README_ARCH.md` to decide create vs update (using SHA).
- Pushes `README_ARCH.md` to GitHub.
- Posts a Slack message (optional; node continues on failure).
- Returns an HTML success page to the webhook/form client.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception

**Overview:**  
Provides two entry points (Webhook and Form). Both deliver a GitHub repo URL into a shared parsing node to normalize the inputs.

**Nodes involved:**
- Receive GitHub URL
- Blueprint Form
- Parse Owner and Repo

#### Node: Receive GitHub URL
- **Type / role:** `Webhook` — API entry point for POST requests.
- **Key configuration:**
  - **Path:** `repo-blueprint`
  - **Method:** `POST`
  - **Response mode:** `responseNode` (workflow must end with a Respond to Webhook node)
  - **onError:** `continueRegularOutput` (errors won’t hard-stop; can pass to downstream nodes if they don’t throw)
  - **Options:** `ignoreBots: false`
- **Expected input:** JSON body containing `github_url` or `repository_url`.
- **Outputs:** To **Parse Owner and Repo**.
- **Potential failures / edge cases:**
  - Missing `body.github_url` / `body.repository_url` leads to error thrown in parser node.
  - If callers send non-JSON or unexpected content-type, the body may not parse as expected.

#### Node: Blueprint Form
- **Type / role:** `Form Trigger` — browser-based entry point.
- **Key configuration:**
  - **Path:** `repo-blueprint-form`
  - **Title:** “Repo Blueprint Architect”
  - **Field:** “GitHub Repository URL” (required)
  - **Button label:** “Generate Blueprint”
  - **Response mode:** `responseNode`
- **Outputs:** To **Parse Owner and Repo**.
- **Potential failures / edge cases:**
  - User submits a URL that is not a GitHub repo URL → parser throws.
  - Private repo URLs will later fail during GitHub API calls unless token has access.

#### Node: Parse Owner and Repo
- **Type / role:** `Code` — validates and normalizes inputs; extracts `{owner, repo}`.
- **Key configuration choices (interpreted):**
  - Reads URL from multiple possible locations:
    - `input.body.github_url` (webhook)
    - `input.body.repository_url` (webhook)
    - `input['GitHub Repository URL']` (form)
  - Regex: `github\.com/([^/]+)/([^/\.]+)` extracting `owner` and `repo` (repo stops before a dot).
  - Throws explicit errors for missing/invalid URL.
- **Outputs:** `{ github_url, owner, repo }` to **Fetch Repo Metadata**.
- **Edge cases / failure types:**
  - URLs with extra path segments (e.g., `/tree/main`) still match, but repo capture may not if format differs.
  - Repo names containing dots may be truncated due to `([^/\.]+)` (dot ends the match). This is a functional limitation.
  - If the webhook body structure differs (e.g., nested fields), URL won’t be found.

---

### 2.2 Evidence Gathering (GitHub API)

**Overview:**  
Builds the “evidence set” for analysis: repo metadata, recursive file tree, and a small subset of key file contents. This limits token usage and reduces hallucination by constraining context.

**Nodes involved:**
- Fetch Repo Metadata
- Fetch Repo Tree
- Identify Key Files
- Fetch Key File Contents
- Prepare LLM Input

#### Node: Fetch Repo Metadata
- **Type / role:** `HTTP Request` — GitHub REST API call to repo metadata.
- **Key configuration:**
  - `GET https://api.github.com/repos/{owner}/{repo}`
  - Auth: `predefinedCredentialType` → **GitHub API** credential
  - Headers:
    - `Accept: application/vnd.github.v3+json`
    - `User-Agent: n8n-repo-blueprint`
  - `neverError: true` (don’t throw on non-2xx; returns response anyway)
- **Inputs:** from **Parse Owner and Repo**.
- **Outputs:** to **Fetch Repo Tree**.
- **Edge cases / failures:**
  - 404/403 returned as data (not thrown) due to `neverError`; downstream nodes may fail when expected fields are missing (`default_branch`, etc.).
  - GitHub rate limiting (403) if token missing/low quota.
  - Topics field can require the correct preview header in older APIs; here it assumes `topics` exists—often true, but may be empty/missing.

#### Node: Fetch Repo Tree
- **Type / role:** `HTTP Request` — fetches recursive tree.
- **Key configuration:**
  - `GET /repos/{owner}/{repo}/git/trees/{default_branch}?recursive=1`
  - Uses `Fetch Repo Metadata` output: `{{ $json.default_branch || 'main' }}`
  - Pulls owner/repo specifically from `Parse Owner and Repo` node (not from current item).
  - Auth + headers same as above.
- **Inputs:** from **Fetch Repo Metadata**.
- **Outputs:** to **Identify Key Files**.
- **Edge cases / failures:**
  - Git Trees API expects a **tree SHA**; using a **branch name** frequently works because GitHub resolves it, but it can fail in some scenarios. Safer approach is: fetch branch ref → get commit SHA → tree SHA.
  - Large repos: tree can be huge; GitHub may truncate or respond slowly.
  - If metadata call returned error payload, `default_branch` may be undefined (falls back to `main`, which may not exist).

#### Node: Identify Key Files
- **Type / role:** `Code` — filters tree, builds tree string, selects up to 12 key files.
- **Key configuration choices:**
  - Filters out common noise paths: `node_modules`, `.git/`, lock files, build artifacts, Python cache/egg-info.
  - Creates `fileTreeStr` of blob paths separated by newline; truncates to **40,000 chars**.
  - Selects key files using regex patterns including (examples):
    - Entry points: `main.py`, `app.py`, `index.js/ts/tsx`, `server.js/ts`
    - Dependency manifests: `package.json`, `requirements.txt`, `pyproject.toml`, `go.mod`, `Cargo.toml`, etc.
    - Containers/config: `Dockerfile`, `docker-compose.*`, `config/**/*.yml|json|toml`
    - GitHub Actions: `.github/workflows/*.yml`
  - Emits multiple items: one per key file (max 12). If none found, emits a single item with `filePath: null`.
  - Also outputs `totalFiles` and `totalDirs`.
- **Inputs:** from **Fetch Repo Tree**.
- **Outputs:** to **Fetch Key File Contents**.
- **Edge cases / failures:**
  - If tree response is not in expected shape (`tree` missing), outputs empty evidence and counts may be zero.
  - Key-file selection is heuristic and might miss important files (e.g., monorepo entry points not matching patterns).
  - If no key files: next node will request `/contents/null` unless handled—currently it will, because the workflow still maps into Fetch Key File Contents.

#### Node: Fetch Key File Contents
- **Type / role:** `HTTP Request` — GitHub Contents API for each key file.
- **Key configuration:**
  - `GET /repos/{owner}/{repo}/contents/{filePath}`
  - `neverError: true`
  - Auth + headers same as above
- **Inputs:** one item per key file from **Identify Key Files**.
- **Outputs:** to **Prepare LLM Input**.
- **Edge cases / failures:**
  - If `filePath` is `null`, request becomes `/contents/null` → likely 404 payload returned. It won’t throw due to `neverError`, but content will be missing.
  - Contents API returns base64 only for files; if path is a directory or too large/binary, content may be absent or encoded unexpectedly.
  - Git LFS files may not contain real content.

#### Node: Prepare LLM Input
- **Type / role:** `Code` — builds a single consolidated prompt string.
- **Key configuration choices:**
  - Pulls:
    - `owner`, `repo` from **Parse Owner and Repo**
    - `metadata` from **Fetch Repo Metadata** (description/topics/language)
    - `fileTreeStr`, `totalFiles`, `totalDirs` from **Identify Key Files**
  - Decodes base64 `content` from each item and truncates each file to **3000 chars**.
  - If decode fails, marks file as `[BINARY/UNREADABLE]`.
  - Produces `llmInput` as:
    - Repo header + stats
    - Full file tree
    - Key file contents section
- **Inputs:** from **Fetch Key File Contents**.
- **Outputs:** single item to **Analyze Architecture**.
- **Edge cases / failures:**
  - If GitHub API errors were returned (403/404 payload), there will be no `content`, resulting in sparse evidence but still a prompt.
  - If tree is very large, tree is truncated; AI conclusions may be incomplete.

---

### 2.3 AI Processing

**Overview:**  
Runs Claude with a strict “evidence-only” system instruction set to produce a structured blueprint and Mermaid diagrams, then sanitizes Markdown formatting.

**Nodes involved:**
- Claude 3.5 Sonnet (Anthropic chat model node)
- Analyze Architecture (LLM chain)
- Fix Markdown

#### Node: Claude 3.5 Sonnet
- **Type / role:** `lmChatAnthropic` — provides the LLM backend to LangChain.
- **Key configuration:**
  - **Model ID:** `claude-sonnet-4-5-20250929` (labelled “Claude 3.5 Sonnet” in node name, but configured to Sonnet 4.5)
  - `temperature: 0.1` (low randomness)
  - `maxTokensToSample: 3000`
  - Credential: **Anthropic API**
- **Connections:** supplies `ai_languageModel` to **Analyze Architecture**.
- **Edge cases / failures:**
  - Model ID may not exist in an account/region or may change; would cause provider error.
  - Token limits: large repos may still exceed context depending on n8n/langchain overhead.

#### Node: Analyze Architecture
- **Type / role:** `chainLlm` — sends the evidence to Claude with a strict rubric and output schema.
- **Key configuration:**
  - **Input text:** `{{$json.llmInput}}`
  - **Prompt type:** “define” with a long system-style instruction enforcing:
    - No speculation; only tech found in file tree/contents
    - Explicit prohibitions (e.g., don’t mention Docker if no Dockerfile)
    - Mermaid syntax constraints (IDs, quoting, flowchart TD, etc.)
    - Required sections:
      - Project Purpose
      - Technical Stack
      - Architecture Blueprint (Mermaid flowchart)
      - Request Flow (Mermaid sequenceDiagram)
      - Evidence-Based Risks
- **Inputs:** main input from **Prepare LLM Input**; model from **Claude 3.5 Sonnet**.
- **Outputs:** to **Fix Markdown**.
- **Edge cases / failures:**
  - Claude output might violate the “strict section-only” requirement or Mermaid rules; downstream doesn’t validate Mermaid, only massages Markdown.
  - If evidence is too thin (e.g., API failures), Claude may produce minimal content but should still comply.

#### Node: Fix Markdown
- **Type / role:** `Code` — normalizes Markdown formatting to reduce rendering issues.
- **Key transformations:**
  - Ensures space after `#` headers
  - Adds blank line before headers and before code fences
  - Ensures blank line after opening code fence (attempted)
  - Trims trailing whitespace
- **Input expectation:** looks for `input.text`; keeps rest of fields.
- **Outputs:** to **Generate Dashboard HTML**.
- **Edge cases / failures:**
  - If Analyze node output field isn’t `text` (varies by node/version), md becomes empty; the workflow would push an empty or near-empty README. (This is a version-coupling risk.)

---

### 2.4 Delivery (Commit + Slack + Web Response)

**Overview:**  
Wraps analysis into a repo-friendly Markdown document, detects whether `README_ARCH.md` already exists, pushes create/update to GitHub, notifies Slack, and returns an HTML confirmation page.

**Nodes involved:**
- Generate Dashboard HTML
- Fetch Existing README
- Prepare Push Data
- Push README_ARCH.md
- Notify on Update
- Send Success Page

#### Node: Generate Dashboard HTML
- **Type / role:** `Code` — constructs final Markdown payload and extracts Mermaid blocks for preview.
- **Key configuration choices:**
  - Reads:
    - `analysisOutput` from `$input.first().json.text`
    - repo metadata fields from earlier nodes (owner/repo/description/default branch)
    - totals from **Prepare LLM Input**
  - Extracts Mermaid blocks via regex: ```mermaid ... ```
  - Builds `markdownContent` with:
    - Title, description, date
    - Full AI output
    - Repo stats table
    - Source link
  - Sets:
    - `mermaidCode`: first diagram snippet (first 1500 chars) used in Slack preview
    - `diagramCount`
    - `commitMessage`: `docs: update system blueprint (YYYY-MM-DD)`
    - `defaultBranch` for pushing
- **Outputs:** to **Fetch Existing README**.
- **Edge cases / failures:**
  - If AI output contains malformed code fences, regex may miss diagrams → diagramCount inaccurate.
  - The commit message always says “update” even on create; Slack message uses isUpdate to distinguish.

#### Node: Fetch Existing README
- **Type / role:** `HTTP Request` — checks if `README_ARCH.md` exists and gets its SHA.
- **Key configuration:**
  - `GET /repos/{owner}/{repo}/contents/README_ARCH.md?ref={defaultBranch}`
  - `neverError: true`
- **Inputs:** from **Generate Dashboard HTML**.
- **Outputs:** to **Prepare Push Data**.
- **Edge cases / failures:**
  - If file doesn’t exist, response is likely an error JSON without `sha`; this is expected and handled.
  - If branch name wrong, returns error payload; push may then fail.

#### Node: Prepare Push Data
- **Type / role:** `Code` — prepares GitHub Contents API PUT body.
- **Key configuration choices:**
  - Base64 encodes `markdownContent`.
  - Creates body `{ message, content, branch }`.
  - If existing `sha` found, adds `sha` (update), else creates new file.
  - Outputs:
    - `pushBodyStr` as a JSON string
    - `isUpdate` boolean
    - `diagramCount`, `mermaidCode` for Slack
- **Outputs:** to **Push README_ARCH.md**.
- **Edge cases / failures:**
  - Uses `pushBodyStr` string passed into HTTP node as JSON; if string is invalid JSON, PUT fails.
  - Large Markdown can exceed GitHub API limits (contents API has practical size limits).

#### Node: Push README_ARCH.md
- **Type / role:** `HTTP Request` — writes the Markdown file to the repo.
- **Key configuration:**
  - `PUT /repos/{owner}/{repo}/contents/README_ARCH.md`
  - Sends body as JSON: `jsonBody = {{$json.pushBodyStr}}` with `specifyBody: json`
  - Auth: GitHub API credential
- **Inputs:** from **Prepare Push Data**.
- **Outputs:** to **Notify on Update**.
- **Edge cases / failures:**
  - 409 conflict if SHA mismatched (file changed since read).
  - 403 if token lacks `repo` scope or repo is not accessible.
  - Branch protections can block commits.

#### Node: Notify on Update
- **Type / role:** `Slack` — posts a message summarizing results.
- **Key configuration:**
  - Operation: `chat.postMessage` (resource message → post)
  - Channel: selectable, but `channelId.value` is empty and must be set.
  - Message includes:
    - Create vs update based on `isUpdate`
    - Links to repo and `README_ARCH.md`
    - diagram count + repo stats
    - Mermaid preview snippet in a code block
  - `continueOnFail: true` (Slack errors won’t stop the workflow)
- **Outputs:** to **Send Success Page**.
- **Edge cases / failures:**
  - Missing channel ID will fail; flow continues anyway.
  - Bot token missing `chat:write` or not in channel → failure.
  - Message includes strings like `:blueprint:` / `:arrows_counterclockwise:` which assume Slack emoji names exist in the workspace.

#### Node: Send Success Page
- **Type / role:** `Respond to Webhook` — returns HTML to the original caller.
- **Key configuration:**
  - Response code: `200`
  - Header: `Content-Type: text/html; charset=utf-8`
  - Body: full HTML page with dynamic interpolations (owner/repo, diagramCount, link to README_ARCH.md)
- **Inputs:** from **Notify on Update** (so it only responds after push + Slack attempt).
- **Edge cases / failures:**
  - Because both entry points are configured with `responseNode`, failing to reach this node can leave requests hanging/time out.
  - If earlier nodes throw (e.g., Parse Owner/Repo), no response node may be reached.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Receive GitHub URL | Webhook | API entry point (POST GitHub URL) | — | Parse Owner and Repo | ## Input<br>Two entry points: web form for browser users, webhook POST for API calls. Both feed into the URL parser. |
| Blueprint Form | Form Trigger | Browser entry point (GitHub URL form) | — | Parse Owner and Repo | ## Input<br>Two entry points: web form for browser users, webhook POST for API calls. Both feed into the URL parser. |
| Parse Owner and Repo | Code | Validate URL and extract owner/repo | Receive GitHub URL; Blueprint Form | Fetch Repo Metadata | ## Input<br>Two entry points: web form for browser users, webhook POST for API calls. Both feed into the URL parser. |
| Fetch Repo Metadata | HTTP Request | Get repo description/language/default branch/topics | Parse Owner and Repo | Fetch Repo Tree | ## Evidence gathering<br>Fetches repo metadata, file tree, and key file contents. This evidence is the only input to the AI. |
| Fetch Repo Tree | HTTP Request | Get recursive git tree for repo | Fetch Repo Metadata | Identify Key Files | ## Evidence gathering<br>Fetches repo metadata, file tree, and key file contents. This evidence is the only input to the AI. |
| Identify Key Files | Code | Filter tree, select key files, compute stats | Fetch Repo Tree | Fetch Key File Contents | ## Evidence gathering<br>Fetches repo metadata, file tree, and key file contents. This evidence is the only input to the AI. |
| Fetch Key File Contents | HTTP Request | Download key file contents from GitHub | Identify Key Files | Prepare LLM Input | ## Evidence gathering<br>Fetches repo metadata, file tree, and key file contents. This evidence is the only input to the AI. |
| Prepare LLM Input | Code | Assemble evidence prompt (tree + decoded file snippets) | Fetch Key File Contents | Analyze Architecture | ## Evidence gathering<br>Fetches repo metadata, file tree, and key file contents. This evidence is the only input to the AI. |
| Claude 3.5 Sonnet | Anthropic Chat Model (LangChain) | LLM provider/model configuration | — | Analyze Architecture (ai_languageModel) | ## AI analysis<br>Claude Sonnet 4.5 analyzes only what exists in the code. Anti-hallucination rules enforced. Markdown auto-fixed. |
| Analyze Architecture | LLM Chain (LangChain) | Evidence-only architecture analysis + Mermaid output | Prepare LLM Input | Fix Markdown | ## AI analysis<br>Claude Sonnet 4.5 analyzes only what exists in the code. Anti-hallucination rules enforced. Markdown auto-fixed. |
| Fix Markdown | Code | Normalize Markdown formatting | Analyze Architecture | Generate Dashboard HTML | ## AI analysis<br>Claude Sonnet 4.5 analyzes only what exists in the code. Anti-hallucination rules enforced. Markdown auto-fixed. |
| Generate Dashboard HTML | Code | Build final Markdown + extract Mermaid preview + commit metadata | Fix Markdown | Fetch Existing README | ## Delivery<br>Assembles blueprint, pushes to repo, notifies Slack, and returns a success page. |
| Fetch Existing README | HTTP Request | Check for existing README_ARCH.md + SHA | Generate Dashboard HTML | Prepare Push Data | ## Delivery<br>Assembles blueprint, pushes to repo, notifies Slack, and returns a success page. |
| Prepare Push Data | Code | Prepare GitHub Contents API PUT body (create/update) | Fetch Existing README | Push README_ARCH.md | ## Delivery<br>Assembles blueprint, pushes to repo, notifies Slack, and returns a success page. |
| Push README_ARCH.md | HTTP Request | Commit README_ARCH.md to repo | Prepare Push Data | Notify on Update | ## Delivery<br>Assembles blueprint, pushes to repo, notifies Slack, and returns a success page. |
| Notify on Update | Slack | Post Slack notification (optional) | Push README_ARCH.md | Send Success Page | ## Delivery<br>Assembles blueprint, pushes to repo, notifies Slack, and returns a success page. |
| Send Success Page | Respond to Webhook | Return HTML success response | Notify on Update | — | ## Delivery<br>Assembles blueprint, pushes to repo, notifies Slack, and returns a success page. |
| Sticky Note - Setup | Sticky Note | Documentation / setup guidance | — | — | ## How it works<br><br>This workflow analyzes any public GitHub repository and generates an evidence-based architecture blueprint with Mermaid.js diagrams.<br><br>1. A GitHub URL is submitted via the web form or webhook API<br>2. The workflow fetches repository metadata, the full file tree, and contents of key files (package.json, main.py, Dockerfiles, etc.)<br>3. Claude AI (Sonnet 4.5, temperature 0.1) analyzes ONLY the evidence — strict anti-hallucination rules prevent it from inventing technologies not found in the code<br>4. A Markdown blueprint with architecture diagrams and risk analysis is assembled, validated, and pushed as `README_ARCH.md` to the repository<br>5. A Slack notification is sent and the form user receives a styled success page<br><br>## Setup steps<br><br>1. **GitHub API** — Create a Personal Access Token with `repo` scope. Add as "GitHub API" credential<br>2. **Anthropic API** — Get an API key from console.anthropic.com. Add as "Anthropic API" credential<br>3. **Slack** *(optional)* — Create a Slack Bot Token with `chat:write` scope. Add as "Slack API" credential and update the channel ID in the Notify node<br>4. **Activate** — Toggle the workflow active. Use the Form URL in your browser or POST `{"github_url": "https://github.com/owner/repo"}` to the webhook |
| Sticky Note - Input | Sticky Note | Comment | — | — | ## Input<br>Two entry points: web form for browser users, webhook POST for API calls. Both feed into the URL parser. |
| Sticky Note - Evidence | Sticky Note | Comment | — | — | ## Evidence gathering<br>Fetches repo metadata, file tree, and key file contents. This evidence is the only input to the AI. |
| Sticky Note - AI | Sticky Note | Comment | — | — | ## AI analysis<br>Claude Sonnet 4.5 analyzes only what exists in the code. Anti-hallucination rules enforced. Markdown auto-fixed. |
| Sticky Note - Delivery | Sticky Note | Comment | — | — | ## Delivery<br>Assembles blueprint, pushes to repo, notifies Slack, and returns a success page. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n  
   - Set workflow settings:
     - Execution Order: `v1`
     - Save execution progress: enabled
     - Save data on success + error: enabled (or match your governance)

2. **Add entry point 1: Webhook**
   - Node: **Webhook**
   - Name: `Receive GitHub URL`
   - Method: `POST`
   - Path: `repo-blueprint`
   - Response mode: `Using 'Respond to Webhook' node` (responseNode)
   - Error behavior: continue regular output

3. **Add entry point 2: Form Trigger**
   - Node: **Form Trigger**
   - Name: `Blueprint Form`
   - Path: `repo-blueprint-form`
   - Title: `Repo Blueprint Architect`
   - Description: “Enter a public GitHub repository URL…”
   - Field:
     - Label: `GitHub Repository URL`
     - Required: true
     - Placeholder: `https://github.com/owner/repo`
   - Button label: `Generate Blueprint`
   - Response mode: responseNode

4. **Add URL parsing node**
   - Node: **Code**
   - Name: `Parse Owner and Repo`
   - Paste logic:
     - Read URL from `body.github_url`, `body.repository_url`, or `GitHub Repository URL`
     - Regex extract owner/repo
     - Output: `{github_url, owner, repo}`
   - **Connect**:
     - `Receive GitHub URL` → `Parse Owner and Repo`
     - `Blueprint Form` → `Parse Owner and Repo`

5. **Configure GitHub credential**
   - Create credential: **GitHub API**
   - Use a **Personal Access Token** with at least:
     - Public repos: typically no scope needed for reads, but **writes require `repo`** scope for private; for public repos, classic PAT often still needs appropriate scopes to write contents.
   - Ensure the token user has permission to push to the target repos.

6. **Fetch repository metadata**
   - Node: **HTTP Request**
   - Name: `Fetch Repo Metadata`
   - Authentication: predefined credential type → **GitHub API**
   - URL: `https://api.github.com/repos/{{$json.owner}}/{{$json.repo}}`
   - Headers:
     - `Accept: application/vnd.github.v3+json`
     - `User-Agent: n8n-repo-blueprint`
   - Response handling: enable “never error” behavior on non-2xx (if available in your node version).
   - **Connect:** `Parse Owner and Repo` → `Fetch Repo Metadata`

7. **Fetch recursive repo tree**
   - Node: **HTTP Request**
   - Name: `Fetch Repo Tree`
   - Authentication: GitHub API credential
   - URL: `https://api.github.com/repos/{{owner}}/{{repo}}/git/trees/{{default_branch}}?recursive=1`
     - Use expressions referencing:
       - owner/repo from `Parse Owner and Repo`
       - default branch from the metadata node, fallback `main`
   - Headers as above.
   - **Connect:** `Fetch Repo Metadata` → `Fetch Repo Tree`

8. **Identify key files**
   - Node: **Code**
   - Name: `Identify Key Files`
   - Implement:
     - Filter tree entries to exclude build/vendor noise
     - Build `fileTreeStr` (truncate around 40k chars)
     - Select up to 12 key paths by regex patterns (entry points + dependency manifests + config/CI)
     - Output multiple items each with `{owner, repo, filePath, fileTreeStr, totalFiles, totalDirs}`
   - **Connect:** `Fetch Repo Tree` → `Identify Key Files`

9. **Fetch key file contents**
   - Node: **HTTP Request**
   - Name: `Fetch Key File Contents`
   - Authentication: GitHub API credential
   - URL: `https://api.github.com/repos/{{$json.owner}}/{{$json.repo}}/contents/{{$json.filePath}}`
   - “Never error” enabled.
   - Headers as above.
   - **Connect:** `Identify Key Files` → `Fetch Key File Contents`

10. **Prepare LLM evidence input**
    - Node: **Code**
    - Name: `Prepare LLM Input`
    - Implement:
      - Decode base64 `content` for each file response
      - Truncate each file to ~3000 chars
      - Combine repo metadata + file tree + file excerpts into `llmInput`
      - Output one item with `{owner, repo, llmInput, fileTreeStr, totalFiles, totalDirs}`
    - **Connect:** `Fetch Key File Contents` → `Prepare LLM Input`

11. **Configure Anthropic credential**
    - Create credential: **Anthropic API**
    - API key from `console.anthropic.com`

12. **Add Anthropic chat model node**
    - Node: **Anthropic Chat Model (LangChain)**
    - Name: `Claude 3.5 Sonnet` (name is arbitrary)
    - Model: set to your available Sonnet model (the workflow uses `claude-sonnet-4-5-20250929`)
    - Temperature: `0.1`
    - Max tokens: `3000`

13. **Add LLM chain node**
    - Node: **LLM Chain** (`chainLlm`)
    - Name: `Analyze Architecture`
    - Text/input: `{{$json.llmInput}}`
    - Prompt: paste the strict “evidence-only” instruction and required output sections (including Mermaid rules)
    - Attach the model:
      - Connect `Claude 3.5 Sonnet` → `Analyze Architecture` via **ai_languageModel**
    - **Connect:** `Prepare LLM Input` → `Analyze Architecture`

14. **Fix Markdown formatting**
    - Node: **Code**
    - Name: `Fix Markdown`
    - Implement header spacing and code fence spacing cleanup; output `text` updated.
    - **Connect:** `Analyze Architecture` → `Fix Markdown`

15. **Generate final Markdown payload**
    - Node: **Code**
    - Name: `Generate Dashboard HTML` (despite name, it builds Markdown + stats)
    - Implement:
      - Extract Mermaid blocks with regex
      - Build a top-level title + stats table + links
      - Output:
        - `markdownContent`, `commitMessage`, `defaultBranch`, `diagramCount`, `mermaidCode`, `owner`, `repo`
    - **Connect:** `Fix Markdown` → `Generate Dashboard HTML`

16. **Check if README exists**
    - Node: **HTTP Request**
    - Name: `Fetch Existing README`
    - Auth: GitHub API
    - URL: `https://api.github.com/repos/{{$json.owner}}/{{$json.repo}}/contents/README_ARCH.md?ref={{$json.defaultBranch}}`
    - “Never error” enabled.
    - **Connect:** `Generate Dashboard HTML` → `Fetch Existing README`

17. **Prepare PUT body**
    - Node: **Code**
    - Name: `Prepare Push Data`
    - Implement:
      - Base64 encode `markdownContent`
      - Build `{message, content, branch}` and add `sha` if present
      - Output `pushBodyStr` (stringified JSON), `isUpdate`, `diagramCount`, `mermaidCode`
    - **Connect:** `Fetch Existing README` → `Prepare Push Data`

18. **Push README_ARCH.md**
    - Node: **HTTP Request**
    - Name: `Push README_ARCH.md`
    - Method: `PUT`
    - URL: `https://api.github.com/repos/{{$json.owner}}/{{$json.repo}}/contents/README_ARCH.md`
    - Auth: GitHub API
    - Body: JSON from the prepared push data (`pushBodyStr`)
    - **Connect:** `Prepare Push Data` → `Push README_ARCH.md`

19. **(Optional) Slack notification**
    - Create credential: **Slack Bot Token** with `chat:write`
    - Node: **Slack**
    - Name: `Notify on Update`
    - Operation: post message to a channel
    - Set **Channel ID**
    - Use expressions for links/stats/isUpdate/diagram preview
    - Enable **Continue on Fail**
    - **Connect:** `Push README_ARCH.md` → `Notify on Update`

20. **Respond to caller with HTML**
    - Node: **Respond to Webhook**
    - Name: `Send Success Page`
    - Response code: 200
    - Header: `Content-Type: text/html; charset=utf-8`
    - Body: HTML template with expressions for owner/repo/diagramCount/link
    - **Connect:** `Notify on Update` → `Send Success Page`
    - (If you want Slack optional without affecting response timing, you can also branch: push → respond, and push → slack in parallel.)

21. **Activate workflow**
    - Use:
      - Form URL: `/form/repo-blueprint-form`
      - Webhook endpoint: POST to `/webhook/repo-blueprint` with body like `{"github_url":"https://github.com/owner/repo"}`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| GitHub API credential: create PAT with `repo` scope (needed for writing `README_ARCH.md`). | Setup guidance from sticky note |
| Anthropic API: get key from console.anthropic.com and configure Anthropic credential in n8n. | Setup guidance from sticky note |
| Slack is optional; requires Bot Token with `chat:write`, and the channel ID must be set in the Slack node. | Setup guidance from sticky note |
| Workflow behavior summary: submit URL → gather evidence → Claude evidence-only analysis → push `README_ARCH.md` → Slack notify → success page. | “How it works” sticky note |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.