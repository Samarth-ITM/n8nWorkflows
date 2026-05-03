Review GitHub pull requests with Gemini and post feedback automatically

https://n8nworkflows.xyz/workflows/review-github-pull-requests-with-gemini-and-post-feedback-automatically-13563


# Review GitHub pull requests with Gemini and post feedback automatically

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Review GitHub pull requests with Gemini AI and post feedback automatically  
**Purpose:** Automatically react to a GitHub Pull Request event, fetch the PR diff, analyze it, ask **Google Gemini** to generate review feedback, post that feedback back to GitHub as a PR review, then log the action to Google Sheets and notify a team in Slack.

### 1.1 Input Reception (GitHub event ingestion)
Receives an incoming webhook (intended for GitHub PR events) and passes the payload forward for extraction.

### 1.2 PR Data Preparation + Diff Retrieval
Extracts essential PR identifiers/URLs from the webhook payload and downloads the PR diff content.

### 1.3 Diff Pre-processing
Transforms/summarizes/cleans the diff (likely to reduce size, remove noise, or format for LLM input).

### 1.4 AI Code Review (Gemini via LangChain node)
Runs an LLM chain using Google Gemini Chat as the language model to generate structured review feedback.

### 1.5 Results Delivery + Logging
Formats the AI output into GitHub’s review API shape, posts the review, logs the run to Sheets, and sends a Slack notification.

---

## 2. Block-by-Block Analysis

### Block 1.1 — Input Reception
**Overview:** Entry point. Accepts an HTTP webhook request (expected from GitHub), producing the raw event JSON for downstream nodes.  
**Nodes involved:** `GitHub PR webhook`

#### Node: GitHub PR webhook
- **Type / Role:** `n8n-nodes-base.webhook` — inbound trigger to start the workflow.
- **Configuration (interpreted):**
  - Parameters are empty in the JSON, meaning defaults are used (method/path/auth not visible here).
  - Has a fixed `webhookId`, which n8n uses internally to identify the endpoint.
- **Key expressions/variables:** None shown (no parameters configured).
- **Connections:**
  - **Output →** `Parse PR data`
- **Version-specific requirements:** TypeVersion **1**.
- **Edge cases / failure types:**
  - GitHub not configured to call the webhook URL (no events).
  - Missing signature verification / authentication (if not configured): endpoint could accept unwanted calls.
  - Payload not matching expected GitHub PR event schema, causing downstream parsing failures.

---

### Block 1.2 — PR Data Preparation + Diff Retrieval
**Overview:** Extracts PR metadata from the webhook payload and downloads the PR diff using HTTP.  
**Nodes involved:** `Parse PR data`, `Get PR diff`

#### Node: Parse PR data
- **Type / Role:** `n8n-nodes-base.code` — custom JavaScript to normalize/shape incoming PR event data (e.g., repo owner/name, PR number, diff URL, API URLs).
- **Configuration (interpreted):**
  - Parameters are empty in the JSON, so the actual script is not included here.
  - In a typical implementation, it would read `items[0].json` and output fields such as:
    - `owner`, `repo`, `pull_number`
    - `diff_url` or `url` to fetch diff
    - `review_url` or `pull_request.url`
- **Key expressions/variables:** Not visible (code not provided).
- **Connections:**
  - **Input ←** `GitHub PR webhook`
  - **Output →** `Get PR diff`
- **Version-specific requirements:** TypeVersion **2** (newer Code node runtime).
- **Edge cases / failure types:**
  - Webhook payload differs across GitHub events (e.g., `pull_request` missing if wrong event).
  - PR fields null for draft PRs or certain actions (opened vs synchronize vs reopened).
  - Code node throwing due to missing keys.

#### Node: Get PR diff
- **Type / Role:** `n8n-nodes-base.httpRequest` — retrieves the PR diff text from GitHub.
- **Configuration (interpreted):**
  - Parameters are empty in JSON; normally requires:
    - URL set from parsed data (diff endpoint)
    - Headers such as `Accept: application/vnd.github.v3.diff` (or GitHub diff media type)
    - Authentication (token) to avoid rate limiting / access private repos
- **Key expressions/variables:** Not visible (would typically reference parsed fields like `{{$json.diff_url}}`).
- **Connections:**
  - **Input ←** `Parse PR data`
  - **Output →** `Analyze diff`
- **Version-specific requirements:** TypeVersion **4.1**.
- **Edge cases / failure types:**
  - 401/403 if token missing/insufficient scopes.
  - 404 if diff URL wrong or PR not accessible.
  - Large diffs causing memory/timeout issues.
  - Rate limiting (429) on GitHub API.

---

### Block 1.3 — Diff Pre-processing
**Overview:** Converts raw diff into a format appropriate for the AI model (e.g., trimming, chunking, extracting changed files, or summarizing).  
**Nodes involved:** `Analyze diff`

#### Node: Analyze diff
- **Type / Role:** `n8n-nodes-base.code` — custom logic to analyze/prepare diff content.
- **Configuration (interpreted):**
  - Parameters empty; code not present.
  - Typical outputs might include:
    - `diff_summary`
    - `files_changed`
    - `risks` / `areas_to_check`
    - `prepared_prompt_input`
- **Key expressions/variables:** Not visible.
- **Connections:**
  - **Input ←** `Get PR diff`
  - **Output →** `Code review`
- **Version-specific requirements:** TypeVersion **2**.
- **Edge cases / failure types:**
  - Diff too large for single prompt → needs truncation/chunking.
  - Binary files or minified files producing noisy diffs.
  - Code errors if input is not plain text (depending on HTTP node response settings).

---

### Block 1.4 — AI Code Review (Gemini via LangChain)
**Overview:** Sends the prepared diff context into a LangChain “chain” node using **Google Gemini Chat** as the language model, producing review comments/feedback.  
**Nodes involved:** `Code review`, `Google Gemini AI`

#### Node: Google Gemini AI
- **Type / Role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` — provides the chat model used by the chain.
- **Configuration (interpreted):**
  - Parameters empty; normally includes:
    - Model selection (e.g., Gemini 1.5 Pro/Flash depending on node options)
    - API key / credentials (Google AI / Gemini)
    - Temperature / max tokens
- **Key expressions/variables:** Not visible.
- **Connections:**
  - **AI Language Model output →** `Code review` (connection type `ai_languageModel`)
- **Version-specific requirements:** TypeVersion **1**; requires n8n LangChain package availability and appropriate credentials set.
- **Edge cases / failure types:**
  - Invalid/absent Gemini API key.
  - Safety filters / refusal depending on content.
  - Token/context limit exceeded for large diffs.

#### Node: Code review
- **Type / Role:** `@n8n/n8n-nodes-langchain.chainLlm` — orchestrates prompt + LLM call to generate review.
- **Configuration (interpreted):**
  - Parameters empty; typically includes:
    - System/user prompt template
    - Possibly structured output instructions (JSON, Markdown, bullet points)
    - Input mapping from `Analyze diff` output
- **Key expressions/variables:** Not visible (would commonly reference fields created by `Analyze diff`).
- **Connections:**
  - **Main input ←** `Analyze diff`
  - **AI model input ←** `Google Gemini AI`
  - **Main output →** `Format review`
- **Version-specific requirements:** TypeVersion **1.4**.
- **Edge cases / failure types:**
  - Prompt not robust → inconsistent output format.
  - Chain expects fields that the previous node didn’t output.
  - Model latency/timeouts.

---

### Block 1.5 — Results Delivery + Logging
**Overview:** Converts the LLM response into GitHub review API payload, posts it to GitHub, logs details to Google Sheets, and notifies Slack.  
**Nodes involved:** `Format review`, `Post GitHub review`, `Log to Sheets`, `Notify team`

#### Node: Format review
- **Type / Role:** `n8n-nodes-base.code` — transforms AI output into the request body required by GitHub’s “create a review” endpoint (and/or into Markdown).
- **Configuration (interpreted):**
  - Parameters empty; code not included.
  - Typical outputs:
    - `event` (e.g., `COMMENT` or `REQUEST_CHANGES`)
    - `body` (review text)
    - Optional `comments[]` with file/line-level notes
- **Connections:**
  - **Input ←** `Code review`
  - **Output →** `Post GitHub review`
- **Version-specific requirements:** TypeVersion **2**.
- **Edge cases / failure types:**
  - AI output not parseable (if expecting JSON).
  - GitHub review API requires specific fields; malformed payload returns 422.
  - Markdown formatting issues (very large body).

#### Node: Post GitHub review
- **Type / Role:** `n8n-nodes-base.httpRequest` — calls GitHub API to submit the review on the PR.
- **Configuration (interpreted):**
  - Parameters empty; typically includes:
    - POST to `https://api.github.com/repos/{owner}/{repo}/pulls/{pull_number}/reviews`
    - Headers: `Authorization: Bearer <token>`, `Accept: application/vnd.github+json`
    - Body from `Format review`
- **Connections:**
  - **Input ←** `Format review`
  - **Output →** `Log to Sheets`
- **Version-specific requirements:** TypeVersion **4.1**.
- **Edge cases / failure types:**
  - 401/403 for auth/scopes (needs `pull_requests:write` or repo scope depending on token type).
  - 422 if review body/event invalid.
  - Posting reviews on draft PRs or from restricted accounts may fail depending on repo settings.

#### Node: Log to Sheets
- **Type / Role:** `n8n-nodes-base.googleSheets` — writes a log entry (PR link, timestamp, summary, status, reviewer output, etc.) to Google Sheets.
- **Configuration (interpreted):**
  - Parameters empty; normally includes:
    - Document + sheet selection
    - “Append row” operation
    - Field mapping from previous node outputs
- **Connections:**
  - **Input ←** `Post GitHub review`
  - **Output →** `Notify team`
- **Version-specific requirements:** TypeVersion **4.1**; requires Google Sheets OAuth2 credentials.
- **Edge cases / failure types:**
  - OAuth token expired / insufficient permissions.
  - Sheet/tab not found, wrong header mapping.
  - Quota limits for Sheets API.

#### Node: Notify team
- **Type / Role:** `n8n-nodes-base.slack` — sends a Slack message to notify that a review was posted (and possibly include highlights).
- **Configuration (interpreted):**
  - Node includes a `webhookId` (internal). Parameters empty; could be Slack API (OAuth) or webhook-based messaging depending on setup.
- **Connections:**
  - **Input ←** `Log to Sheets`
  - **Output:** None (end node)
- **Version-specific requirements:** TypeVersion **2.1**.
- **Edge cases / failure types:**
  - Slack auth/webhook misconfigured.
  - Posting to a channel the bot isn’t in.
  - Message length limits.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| GitHub PR webhook | n8n-nodes-base.webhook | Entry trigger receiving GitHub PR event payload | — | Parse PR data |  |
| Parse PR data | n8n-nodes-base.code | Extract PR metadata needed for API calls | GitHub PR webhook | Get PR diff |  |
| Get PR diff | n8n-nodes-base.httpRequest | Download PR diff from GitHub | Parse PR data | Analyze diff |  |
| Analyze diff | n8n-nodes-base.code | Prepare/summarize diff for LLM consumption | Get PR diff | Code review |  |
| Code review | @n8n/n8n-nodes-langchain.chainLlm | Run LLM chain to produce review feedback | Analyze diff; (AI model) Google Gemini AI | Format review |  |
| Google Gemini AI | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Chat LLM provider (Gemini) for the chain | — | Code review (ai_languageModel) |  |
| Format review | n8n-nodes-base.code | Convert LLM output into GitHub review payload | Code review | Post GitHub review |  |
| Post GitHub review | n8n-nodes-base.httpRequest | Call GitHub API to post PR review | Format review | Log to Sheets |  |
| Log to Sheets | n8n-nodes-base.googleSheets | Append log row to Google Sheet | Post GitHub review | Notify team |  |
| Notify team | n8n-nodes-base.slack | Send Slack notification | Log to Sheets | — |  |
| Overview Sticky | n8n-nodes-base.stickyNote | Canvas annotation | — | — |  |
| Webhook Setup Section | n8n-nodes-base.stickyNote | Canvas annotation | — | — |  |
| Data Analysis Section | n8n-nodes-base.stickyNote | Canvas annotation | — | — |  |
| AI Review Section | n8n-nodes-base.stickyNote | Canvas annotation | — | — |  |
| Results Section | n8n-nodes-base.stickyNote | Canvas annotation | — | — |  |

> Note: All sticky notes have **empty content** in the provided JSON, so no comment text can be associated to nodes.

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named:  
   `Review GitHub pull requests with Gemini AI and post feedback automatically`

2. **Add Trigger: Webhook**
   - Node: **Webhook** → rename to `GitHub PR webhook`
   - Set an HTTP method (commonly **POST**).
   - Set a path (e.g., `/github-pr-review`).
   - (Recommended) Enable authentication or implement GitHub signature verification (HMAC) via a Code node.
   - Copy the Production URL.

3. **Configure GitHub Webhook (in GitHub repo settings)**
   - Payload URL: the n8n webhook Production URL
   - Content type: `application/json`
   - Events: select **Pull requests** (at minimum).  
   - Save.

4. **Add Code node: Parse PR data**
   - Node: **Code** → rename `Parse PR data`
   - Implement logic to read the GitHub event and output essential fields, typically:
     - `owner`, `repo`, `pull_number`
     - `diff_url` (or construct it)
     - `api_review_url` (`/pulls/{pull_number}/reviews`)
     - `pr_html_url` for logging/Slack
   - Connect: `GitHub PR webhook` → `Parse PR data`

5. **Add HTTP Request: Get PR diff**
   - Node: **HTTP Request** → rename `Get PR diff`
   - Method: **GET**
   - URL: expression referencing your parsed diff URL (e.g., `{{$json.diff_url}}`)
   - Authentication:
     - Use GitHub token (Header Auth): `Authorization: Bearer <token>`
   - Headers:
     - `Accept: application/vnd.github.v3.diff` (or GitHub diff media type)
   - Response:
     - Ensure you keep the response as **text** (diff is not JSON).
   - Connect: `Parse PR data` → `Get PR diff`

6. **Add Code node: Analyze diff**
   - Node: **Code** → rename `Analyze diff`
   - Prepare LLM input:
     - Optionally truncate large diffs
     - Extract per-file sections
     - Output fields such as `preparedDiff`, `fileList`, `stats`
   - Connect: `Get PR diff` → `Analyze diff`

7. **Add Gemini model node**
   - Node: **Google Gemini Chat Model** (LangChain) → rename `Google Gemini AI`
   - Credentials:
     - Configure Gemini/Google AI API key in n8n credentials.
   - Choose model + parameters (temperature, max tokens) appropriate to diff size.

8. **Add LangChain “Chain LLM” node**
   - Node: **Chain LLM** → rename `Code review`
   - Configure prompt(s):
     - Provide instructions to produce actionable review feedback.
     - Prefer a predictable format (e.g., Markdown sections or JSON).
   - Map inputs from `Analyze diff` (e.g., include `preparedDiff`).
   - Connect model:
     - Connect `Google Gemini AI` to `Code review` via the **AI Language Model** connection.
   - Connect: `Analyze diff` → `Code review`

9. **Add Code node: Format review**
   - Node: **Code** → rename `Format review`
   - Convert LLM output into GitHub review payload, typically:
     - `event`: `COMMENT` (safe default) or `REQUEST_CHANGES`
     - `body`: final review text (Markdown)
     - Optional inline `comments` if you generate line-level notes
   - Connect: `Code review` → `Format review`

10. **Add HTTP Request: Post GitHub review**
    - Node: **HTTP Request** → rename `Post GitHub review`
    - Method: **POST**
    - URL: `https://api.github.com/repos/{{$json.owner}}/{{$json.repo}}/pulls/{{$json.pull_number}}/reviews` (or your computed endpoint)
    - Auth header: `Authorization: Bearer <token>`
    - Headers: `Accept: application/vnd.github+json`
    - Body: JSON from `Format review` output
    - Connect: `Format review` → `Post GitHub review`

11. **Add Google Sheets node: Log to Sheets**
    - Node: **Google Sheets** → rename `Log to Sheets`
    - Credentials: Google OAuth2 with access to the target spreadsheet
    - Operation: typically **Append** row
    - Map fields such as:
      - Timestamp, repo, PR number, PR URL
      - Result status (success/failure), review summary
    - Connect: `Post GitHub review` → `Log to Sheets`

12. **Add Slack node: Notify team**
    - Node: **Slack** → rename `Notify team`
    - Credentials:
      - Slack OAuth2 bot token or Slack Incoming Webhook (depending on chosen Slack node operation)
    - Operation: “Post message” to channel
    - Message: include PR link + review status
    - Connect: `Log to Sheets` → `Notify team`

13. **Activate workflow**
    - Switch workflow to **Active**
    - Send a test PR event (open/update PR) and verify:
      - Diff fetch works
      - Gemini returns expected format
      - GitHub review is posted
      - Sheets row appended
      - Slack notified

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sticky notes exist for sections (“Overview Sticky”, “Webhook Setup Section”, “Data Analysis Section”, “AI Review Section”, “Results Section”) but their content is empty in the provided workflow JSON. | n8n canvas annotations (empty) |

--- 

If you paste the **Code node contents** (Parse PR data / Analyze diff / Format review) and the **HTTP Request settings** (URLs/headers/body), I can document the exact variables, prompt structure, and GitHub/Slack/Sheets field mappings precisely, including the expected input/output schemas for each node.