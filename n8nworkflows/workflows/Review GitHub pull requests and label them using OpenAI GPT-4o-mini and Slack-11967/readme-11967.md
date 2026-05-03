Review GitHub pull requests and label them using OpenAI GPT-4o-mini and Slack

https://n8nworkflows.xyz/workflows/review-github-pull-requests-and-label-them-using-openai-gpt-4o-mini-and-slack-11967


# Review GitHub pull requests and label them using OpenAI GPT-4o-mini and Slack

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
Automatically review GitHub Pull Requests (PRs) when they are opened or updated, generate an AI-based code review using **OpenAI GPT-4o-mini**, apply labels to the PR, post a review comment, notify a Slack channel, and log the run to Google Sheets, then respond back to GitHub via webhook response.

**Target use cases:**
- Fast, consistent first-pass PR review (quality/security/complexity signals)
- Automatic sizing labels (e.g., size/S, size/L)
- Team notifications and audit logging of PR reviews

### 1.1 Input Reception & Filtering
Receives GitHub webhook events and filters to only process relevant PR actions.

### 1.2 PR Data Extraction & Gathering
Extracts key PR metadata, fetches the PR file list and raw diff, then merges them.

### 1.3 Change Analysis & AI Review
Computes basic heuristics (size label, diff preview) and calls the OpenAI-powered agent for a structured review.

### 1.4 Execution & Alerts
Parses AI output (currently stubbed) and performs GitHub labeling, PR commenting, and Slack notification.

### 1.5 Reporting & Response
Aggregates outputs, appends a row to Google Sheets, and responds to the webhook call with success JSON.

---

## 2. Block-by-Block Analysis

### Block 1 — Intake & Filter
**Overview:**  
Accepts incoming PR events from GitHub and ensures only “opened” and “synchronize” actions trigger downstream processing.

**Nodes involved:**
- Main Overview (Sticky Note)
- Intake Group (Sticky Note)
- GitHub PR Webhook (Webhook)
- Filter PR Events (Filter)

#### Node: Main Overview (Sticky Note)
- **Type/role:** Sticky Note; documentation.
- **Configuration:** Explains end-to-end behavior and setup steps (GitHub webhook, OpenAI key, Slack/Sheets IDs, credentials).
- **Connections:** None (informational only).
- **Failure/edge cases:** None.

#### Node: Intake Group (Sticky Note)
- **Type/role:** Sticky Note; labels the “Intake & Filter” block.
- **Connections:** None.
- **Failure/edge cases:** None.

#### Node: GitHub PR Webhook
- **Type/role:** `Webhook` node; workflow entry point.
- **Configuration choices:**
  - **HTTP Method:** POST
  - **Path:** `github-pr-review`
  - **Response mode:** `responseNode` (response is sent by “Respond to Webhook” later)
- **Input/Output:**
  - **Outputs to:** Filter PR Events
- **Version-specific requirements:** Webhook node v2.
- **Edge cases / failures:**
  - GitHub webhook misconfiguration (wrong URL/path, content type, secret).
  - Payload structure differences (depends on GitHub event type).
  - If downstream fails and no response is sent, GitHub may retry delivery.

#### Node: Filter PR Events
- **Type/role:** `Filter` node; gates execution.
- **Configuration choices:**
  - OR condition:
    - `{{$json.body.action}} == "opened"`
    - `{{$json.body.action}} == "synchronize"`
- **Input/Output:**
  - **Input:** GitHub PR Webhook
  - **Output:** Extract PR Data (only when condition passes)
- **Edge cases / failures:**
  - If GitHub payload isn’t nested under `body` (some webhook configurations or proxies), filter will evaluate to false.
  - Additional PR actions (e.g., `reopened`, `edited`, `ready_for_review`) will be ignored.

---

### Block 2 — Data Gathering
**Overview:**  
Normalizes PR metadata, fetches changed files and PR diff content, and merges results for downstream analysis.

**Nodes involved:**
- Data Group (Sticky Note)
- Extract PR Data (Code)
- Get PR Files (GitHub)
- Fetch PR Diff (HTTP Request)
- Merge PR Info (Merge)

#### Node: Data Group (Sticky Note)
- **Type/role:** Sticky Note; labels the “Data Gathering” block.
- **Connections:** None.
- **Failure/edge cases:** None.

#### Node: Extract PR Data
- **Type/role:** `Code` node; transforms webhook payload into a compact PR object.
- **Configuration choices (interpreted):**
  - Reads `body` from incoming data:  
    `const body = $input.first().json.body || $input.first().json;`
  - Extracts `pull_request` and repository owner/name and produces:
    - `prNumber`, `prTitle`, `prBody`, `author`
    - `repoOwner`, `repoName`
    - `additions`, `deletions`, `changedFiles`
    - `diffUrl`, `htmlUrl`
- **Input/Output:**
  - **Input:** Filter PR Events
  - **Outputs to:** Get PR Files and Fetch PR Diff (parallel branches)
- **Edge cases / failures:**
  - If webhook is not a PR event, `body.pull_request` may be undefined → missing fields downstream.
  - `diff_url` may require authentication depending on repository visibility and GitHub headers.

#### Node: Get PR Files
- **Type/role:** `GitHub` node; fetches file list (intended).
- **Configuration choices:**
  - **Resource:** file
  - **Operation:** list
  - **Owner/Repo:** from expressions `{{$json.repoOwner}}`, `{{$json.repoName}}`
- **Input/Output:**
  - **Input:** Extract PR Data
  - **Output:** Merge PR Info (index 0)
- **Important integration note (potential issue):**
  - The configuration does **not** pass the PR number to the “list files in a PR” endpoint. In many GitHub APIs, listing PR files requires a PR identifier. If this node is actually “List repository files”, it won’t reflect PR changes. Validate in your n8n GitHub node UI which “file/list” operation this maps to.
- **Edge cases / failures:**
  - GitHub OAuth token missing scopes (repo/private repo access, issues/PR permissions).
  - Pagination: large PRs may have many files; node may paginate or truncate depending on n8n’s GitHub node behavior.

#### Node: Fetch PR Diff
- **Type/role:** `HTTP Request` node; downloads raw diff.
- **Configuration choices:**
  - **URL:** `{{ $('Extract PR Data').first().json.diffUrl }}`
  - Response left as default (stored under `.json.data` in later code).
- **Input/Output:**
  - **Input:** Extract PR Data
  - **Output:** Merge PR Info (index 1)
- **Edge cases / failures:**
  - GitHub may require `Authorization` header and/or `Accept: application/vnd.github.v3.diff`.
  - If unauthenticated, private repos will return 404/401.
  - Large diffs can exceed memory/time limits.
  - Output format may not land in `.json.data` depending on node settings; later code assumes `.json.data`.

#### Node: Merge PR Info
- **Type/role:** `Merge` node; combines data from the “files” and “diff” branches.
- **Configuration choices:**
  - **Mode:** combine
  - **Combine by:** combineAll
- **Input/Output:**
  - **Inputs:** Get PR Files (input 0) and Fetch PR Diff (input 1)
  - **Output:** Analyze File Changes
- **Edge cases / failures:**
  - If one branch errors or returns no item, merge may output nothing.
  - `combineAll` can create unexpected structures when each side returns arrays of different lengths.

---

### Block 3 — AI Code Analysis
**Overview:**  
Creates a diff preview, generates a basic size label, then asks an OpenAI-backed agent for a structured review.

**Nodes involved:**
- AI Group (Sticky Note)
- Analyze File Changes (Code)
- OpenAI Chat Model (OpenAI / LangChain)
- AI Code Reviewer (LangChain Agent)

#### Node: AI Group (Sticky Note)
- **Type/role:** Sticky Note; labels the “AI Code Analysis” block.
- **Connections:** None.
- **Failure/edge cases:** None.

#### Node: Analyze File Changes
- **Type/role:** `Code` node; computes heuristics and packages prompt input.
- **Configuration choices (interpreted):**
  - Pulls PR metadata from `Extract PR Data`
  - Reads diff from `Fetch PR Diff`. Assumes diff is in `$('Fetch PR Diff').first().json.data`
  - Creates size label:
    - if `additions + deletions > 200` → `['size/L']` else `['size/S']`
  - Creates `diffPreview` truncated to first 3000 chars
  - Outputs merged structure: PR fields + `diffPreview` + `suggestedLabels`
- **Input/Output:**
  - **Input:** Merge PR Info
  - **Output:** AI Code Reviewer
- **Edge cases / failures:**
  - If `additions`/`deletions` are null/undefined, label logic may produce `NaN` comparisons.
  - Diff may be binary or empty; preview may be misleading.
  - If HTTP Request output isn’t `.json.data`, diffPreview becomes empty.

#### Node: OpenAI Chat Model
- **Type/role:** LangChain `lmChatOpenAi`; provides the model to the agent.
- **Configuration choices:**
  - **Model:** `gpt-4o-mini`
  - Uses OpenAI credentials (API key) in node credentials settings.
- **Input/Output connections:**
  - Connected via **ai_languageModel** port into “AI Code Reviewer”.
- **Version-specific requirements:** node v1.2.
- **Edge cases / failures:**
  - Invalid API key / insufficient quota.
  - Model name not available in the configured OpenAI account/region.
  - Rate limits/timeouts.

#### Node: AI Code Reviewer
- **Type/role:** LangChain `agent`; sends PR title + diff preview to AI and requests JSON output.
- **Configuration choices:**
  - **Prompt text:** `PR Review for: {{ $json.prTitle }}. Diff: {{ $json.diffPreview }}`
  - **System message:** “Format response as JSON with: qualityScore, summary, strengths, concerns, suggestions.”
  - Uses “OpenAI Chat Model” as its language model input.
- **Input/Output:**
  - **Input:** Analyze File Changes
  - **Output:** Parse AI Review
- **Edge cases / failures:**
  - The agent may output non-JSON or malformed JSON despite instruction.
  - Diff truncation can omit context; AI conclusions may be incomplete.
  - Token limits: large diffs truncated; still might be too big depending on other prompt parts.

---

### Block 4 — Execution & Alerts
**Overview:**  
Transforms AI output into actionable labels and comments, applies labels and posts comment on GitHub, and sends a Slack alert.

**Nodes involved:**
- Action Group (Sticky Note)
- Parse AI Review (Code)
- Add PR Labels (GitHub)
- Post Review Comment (GitHub)
- Notify Slack (Slack)

#### Node: Action Group (Sticky Note)
- **Type/role:** Sticky Note; labels the “Execution & Alerts” block.
- **Connections:** None.

#### Node: Parse AI Review
- **Type/role:** `Code` node; intended to parse AI JSON and prepare final actions.
- **Configuration choices (interpreted):**
  - Currently **does not parse** actual AI output; it sets a hard-coded `review` object:
    - `qualityScore: 85`, `summary: "AI Analysis Complete"`, `approvalRecommendation: "approve"`
  - Sets:
    - `finalLabels` = `suggestedLabels`
    - `reviewComment` = `"AI reviewed your PR."`
  - Merges into PR payload: `{...prData, aiReview: review, finalLabels, reviewComment}`
- **Input/Output:**
  - **Input:** AI Code Reviewer
  - **Outputs (fan-out):** Add PR Labels, Post Review Comment, Notify Slack
- **Edge cases / failures:**
  - As implemented, it ignores real AI results, so labels/comments won’t reflect analysis.
  - If `Analyze File Changes` didn’t produce expected fields, labels/comments may be empty.

#### Node: Add PR Labels
- **Type/role:** `GitHub` node; edits issue/PR labels.
- **Configuration choices:**
  - **Operation:** edit
  - **Owner/Repo:** expressions from payload
  - **Issue Number:** `{{$json.prNumber}}` (PRs are issues in GitHub API labeling)
  - **Labels:** `{{$json.finalLabels}}` (array)
- **Input/Output:**
  - **Input:** Parse AI Review
  - **Output:** Aggregate Results
- **Edge cases / failures:**
  - Permissions: token needs rights to modify issues/labels.
  - Label names must exist in repo unless GitHub auto-creates (typically labels must exist).
  - Replacing vs adding labels: many APIs “set labels” replacing existing; confirm node behavior.

#### Node: Post Review Comment
- **Type/role:** `GitHub` node; posts an issue comment on PR.
- **Configuration choices:**
  - **Operation:** createComment
  - **Body:** `{{$json.reviewComment}}`
  - **Issue Number:** `{{$json.prNumber}}`
- **Input/Output:**
  - **Input:** Parse AI Review
  - **Output:** Aggregate Results
- **Edge cases / failures:**
  - Rate limits on commenting.
  - Comment body too large (if expanded later with AI details).

#### Node: Notify Slack
- **Type/role:** `Slack` node; sends channel message.
- **Configuration choices:**
  - **Channel selection:** by name
  - **Channel value:** `#code-reviews`
  - **Text:** `New PR Review for #{{ $json.prNumber }}`
- **Input/Output:**
  - **Input:** Parse AI Review
  - **Output:** Aggregate Results
- **Edge cases / failures:**
  - Slack credential scope missing (chat:write).
  - Channel name mismatch or private channel requires channel ID / bot membership.

---

### Block 5 — Reporting
**Overview:**  
Collects results of parallel action nodes, logs to Google Sheets, and returns a JSON success response.

**Nodes involved:**
- Report Group (Sticky Note)
- Aggregate Results (Aggregate)
- Log to Sheets (Google Sheets)
- Respond to Webhook (Respond to Webhook)

#### Node: Report Group (Sticky Note)
- **Type/role:** Sticky Note; labels “Reporting” block.
- **Connections:** None.

#### Node: Aggregate Results
- **Type/role:** `Aggregate` node; merges results from Slack + GitHub actions.
- **Configuration choices:**
  - **Aggregate:** aggregateAllItemData (collect everything)
- **Input/Output:**
  - **Inputs:** Add PR Labels, Post Review Comment, Notify Slack
  - **Output:** Log to Sheets
- **Edge cases / failures:**
  - If any branch fails and stops execution, aggregation may never run.
  - Aggregated structure can be nested and harder to map to sheets columns.

#### Node: Log to Sheets
- **Type/role:** `Google Sheets` node; appends a row (operation: append).
- **Configuration choices:**
  - Spreadsheet and sheet/tab are not specified in the JSON excerpt (must be configured in UI).
  - Appends whatever fields are mapped (currently unspecified).
- **Input/Output:**
  - **Input:** Aggregate Results
  - **Output:** Respond to Webhook
- **Edge cases / failures:**
  - Missing spreadsheet ID / sheet name.
  - Header mismatch vs appended fields.
  - Google OAuth token expiration, permission denied.

#### Node: Respond to Webhook
- **Type/role:** `Respond to Webhook`; sends HTTP response back to webhook caller.
- **Configuration choices:**
  - Respond with JSON: `{{ {success: true} }}`
- **Input/Output:**
  - **Input:** Log to Sheets
  - **Output:** (terminus)
- **Edge cases / failures:**
  - If workflow errors before this node, webhook request may time out and GitHub will retry.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Main Overview | Sticky Note | Documentation / overview |  |  | ## How it works… (setup steps for GitHub/OpenAI/Slack/Sheets/credentials) |
| Intake Group | Sticky Note | Block label: Intake & Filter |  |  | ## 1. Intake & Filter… |
| Data Group | Sticky Note | Block label: Data Gathering |  |  | ## 2. Data Gathering… |
| AI Group | Sticky Note | Block label: AI Code Analysis |  |  | ## 3. AI Code Analysis… |
| Action Group | Sticky Note | Block label: Execution & Alerts |  |  | ## 4. Execution & Alerts… |
| Report Group | Sticky Note | Block label: Reporting |  |  | ## 5. Reporting… |
| GitHub PR Webhook | Webhook | Entry point for GitHub PR events | — | Filter PR Events | ## 1. Intake & Filter… |
| Filter PR Events | Filter | Allow only opened/synchronize | GitHub PR Webhook | Extract PR Data | ## 1. Intake & Filter… |
| Extract PR Data | Code | Normalize webhook payload into PR fields | Filter PR Events | Get PR Files; Fetch PR Diff | ## 2. Data Gathering… |
| Get PR Files | GitHub | Fetch file list (intended PR changed files) | Extract PR Data | Merge PR Info | ## 2. Data Gathering… |
| Fetch PR Diff | HTTP Request | Download raw PR diff via diff_url | Extract PR Data | Merge PR Info | ## 2. Data Gathering… |
| Merge PR Info | Merge | Combine files + diff branches | Get PR Files; Fetch PR Diff | Analyze File Changes | ## 2. Data Gathering… |
| Analyze File Changes | Code | Size labeling + diffPreview | Merge PR Info | AI Code Reviewer | ## 3. AI Code Analysis… |
| OpenAI Chat Model | OpenAI Chat Model (LangChain) | Provides GPT-4o-mini model | — | AI Code Reviewer (ai_languageModel) | ## 3. AI Code Analysis… |
| AI Code Reviewer | Agent (LangChain) | AI review of title+diffPreview; requests JSON | Analyze File Changes + OpenAI Chat Model | Parse AI Review | ## 3. AI Code Analysis… |
| Parse AI Review | Code | Prepare finalLabels + reviewComment (currently stubbed) | AI Code Reviewer | Add PR Labels; Post Review Comment; Notify Slack | ## 4. Execution & Alerts… |
| Add PR Labels | GitHub | Apply labels to PR (issue labels) | Parse AI Review | Aggregate Results | ## 4. Execution & Alerts… |
| Post Review Comment | GitHub | Comment on PR | Parse AI Review | Aggregate Results | ## 4. Execution & Alerts… |
| Notify Slack | Slack | Send Slack message to #code-reviews | Parse AI Review | Aggregate Results | ## 4. Execution & Alerts… |
| Aggregate Results | Aggregate | Join results from parallel actions | Add PR Labels; Post Review Comment; Notify Slack | Log to Sheets | ## 5. Reporting… |
| Log to Sheets | Google Sheets | Append log row | Aggregate Results | Respond to Webhook | ## 5. Reporting… |
| Respond to Webhook | Respond to Webhook | Return success JSON | Log to Sheets | — | ## 5. Reporting… |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Sticky Notes (optional but matching workflow)**
   - Add 6 Sticky Note nodes named:
     - `Main Overview`, `Intake Group`, `Data Group`, `AI Group`, `Action Group`, `Report Group`
   - Paste their respective block descriptions (from the workflow) to document intent.

2. **Webhook entry**
   - Add **Webhook** node named `GitHub PR Webhook`
     - Method: **POST**
     - Path: `github-pr-review`
     - Response mode: **Using “Respond to Webhook” node**
   - In GitHub repo settings → Webhooks:
     - Payload URL = your n8n webhook URL + `/github-pr-review`
     - Content type: `application/json`
     - Events: Pull requests
     - (Optional) Secret: configure and validate if you add verification logic.

3. **Filter PR actions**
   - Add **Filter** node `Filter PR Events`
     - Conditions → OR:
       - `{{$json.body.action}}` equals `opened`
       - `{{$json.body.action}}` equals `synchronize`
   - Connect: `GitHub PR Webhook` → `Filter PR Events`.

4. **Extract PR metadata**
   - Add **Code** node `Extract PR Data` with JS:
     - Extract `pull_request`, repo owner/name, additions/deletions, diff_url, etc.
   - Connect: `Filter PR Events` → `Extract PR Data`.

5. **Fetch PR changed files**
   - Add **GitHub** node `Get PR Files`
     - Credentials: GitHub OAuth/PAT with repo access
     - Owner: `{{$json.repoOwner}}`
     - Repository: `{{$json.repoName}}`
     - Resource/Operation: set to the operation that lists **PR files** if available in your n8n version (ensure it uses PR number).
   - Connect: `Extract PR Data` → `Get PR Files`.

6. **Fetch PR diff**
   - Add **HTTP Request** node `Fetch PR Diff`
     - URL: `{{ $('Extract PR Data').first().json.diffUrl }}`
     - Add headers if needed:
       - `Authorization: Bearer <GitHub token>` (or use OAuth2 in HTTP node)
       - `Accept: application/vnd.github.v3.diff`
   - Connect: `Extract PR Data` → `Fetch PR Diff`.

7. **Merge**
   - Add **Merge** node `Merge PR Info`
     - Mode: **Combine**
     - Combine by: **Combine All**
   - Connect:
     - `Get PR Files` → `Merge PR Info` (Input 1)
     - `Fetch PR Diff` → `Merge PR Info` (Input 2)

8. **Analyze change size + prepare diff preview**
   - Add **Code** node `Analyze File Changes`
     - Compute `suggestedLabels` using additions+deletions threshold (200)
     - Create `diffPreview` by truncating diff to 3000 chars
   - Connect: `Merge PR Info` → `Analyze File Changes`.

9. **OpenAI model**
   - Add **OpenAI Chat Model** node (LangChain) named `OpenAI Chat Model`
     - Model: `gpt-4o-mini`
     - Credentials: OpenAI API Key
   - No “main” connection; it connects to the agent via the AI language model port.

10. **AI agent**
    - Add **AI Agent** node (LangChain) named `AI Code Reviewer`
      - Prompt text: `PR Review for: {{ $json.prTitle }}. Diff: {{ $json.diffPreview }}`
      - System message: “Format response as JSON with: qualityScore, summary, strengths, concerns, suggestions.”
    - Connect:
      - `Analyze File Changes` → `AI Code Reviewer` (main)
      - `OpenAI Chat Model` → `AI Code Reviewer` (ai_languageModel port)

11. **Parse AI output and prepare actions**
    - Add **Code** node `Parse AI Review`
      - In the provided workflow it is stubbed (hard-coded review). To match exactly, keep the stub.
      - If improving: parse agent output JSON safely and build `reviewComment` from it.
    - Connect: `AI Code Reviewer` → `Parse AI Review`.

12. **Apply labels**
    - Add **GitHub** node `Add PR Labels`
      - Operation: **Edit**
      - Owner/Repo: expressions from input
      - Issue number: `{{$json.prNumber}}`
      - Labels: `{{$json.finalLabels}}`
    - Connect: `Parse AI Review` → `Add PR Labels`.

13. **Post comment**
    - Add **GitHub** node `Post Review Comment`
      - Operation: **Create Comment**
      - Issue number: `{{$json.prNumber}}`
      - Body: `{{$json.reviewComment}}`
    - Connect: `Parse AI Review` → `Post Review Comment`.

14. **Slack notification**
    - Add **Slack** node `Notify Slack`
      - Credentials: Slack OAuth with `chat:write`
      - Channel: by name → `#code-reviews` (or use channel ID)
      - Text: `New PR Review for #{{ $json.prNumber }}`
    - Connect: `Parse AI Review` → `Notify Slack`.

15. **Aggregate**
    - Add **Aggregate** node `Aggregate Results`
      - Mode: aggregate all item data
    - Connect:
      - `Add PR Labels` → `Aggregate Results`
      - `Post Review Comment` → `Aggregate Results`
      - `Notify Slack` → `Aggregate Results`

16. **Log to Google Sheets**
    - Add **Google Sheets** node `Log to Sheets`
      - Credentials: Google OAuth2
      - Operation: **Append**
      - Configure Spreadsheet ID + Sheet name, and map columns (e.g., PR number, URL, author, labels, qualityScore, timestamp).
    - Connect: `Aggregate Results` → `Log to Sheets`.

17. **Respond to GitHub webhook**
    - Add **Respond to Webhook** node `Respond to Webhook`
      - Respond with: JSON
      - Body: `{{ { success: true } }}`
    - Connect: `Log to Sheets` → `Respond to Webhook`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Workflow triggers on PR creation/update, retrieves metadata + diff, uses OpenAI for analysis, labels PR, comments, Slack notify, logs to Sheets. | From sticky note “Main Overview” |
| Setup steps: GitHub webhook + OAuth, OpenAI API key in model node, Slack channel ID and Spreadsheet ID, credentials for GitHub/OpenAI/Slack/Google Sheets. | From sticky note “Main Overview” |