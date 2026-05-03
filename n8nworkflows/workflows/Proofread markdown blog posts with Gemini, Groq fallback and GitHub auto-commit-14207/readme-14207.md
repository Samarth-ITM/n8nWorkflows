Proofread markdown blog posts with Gemini, Groq fallback and GitHub auto-commit

https://n8nworkflows.xyz/workflows/proofread-markdown-blog-posts-with-gemini--groq-fallback-and-github-auto-commit-14207


# Proofread markdown blog posts with Gemini, Groq fallback and GitHub auto-commit

# 1. Workflow Overview

This workflow performs automated quality review and correction of a Markdown blog post stored in GitHub. It fetches a target file, sends it through a two-stage AI process, applies safe line-based edits, then commits both the corrected file and a QA report back to the repository. It uses Google Gemini as the primary LLM and Groq as a fallback for both AI stages.

Primary use cases:
- Proofreading Markdown blog posts before publication
- Enforcing editorial quality checks on GitHub-hosted content
- Producing an auditable QA report alongside automated fixes
- Reducing manual review effort while preserving a safe fallback when edits cannot be confidently applied

## 1.1 Trigger and Repository Configuration

The workflow starts from a manual trigger, then defines the GitHub repository owner, repository name, file path, and report folder through a configuration node.

## 1.2 File Retrieval and Preparation

The Markdown file is downloaded from GitHub, decoded from base64 if needed, and transformed into a line-numbered representation. This numbered version is used to help the AI reference exact lines.

## 1.3 AI QA Analysis

A first AI agent reviews the numbered Markdown content and returns a JSON array of detected issues. The workflow then parses and filters the output, keeping only medium- and high-severity issues.

## 1.4 Decision: Issues Found or Not

If no actionable issues are found, the workflow generates a clean QA report and saves it to GitHub. If issues are found, it proceeds to a second AI stage.

## 1.5 AI Edit Generation and Local Edit Application

A second AI agent converts detected issues into executable line-based edit operations. These operations are parsed, sorted, and applied bottom-to-top to the original content to avoid line-number drift.

## 1.6 Decision: Safe to Commit or Not

If enough edits were successfully applied, the updated blog post is committed back to GitHub and a QA report is saved. If edits are too unreliable, no content change is committed; instead, a failure report is written to GitHub.

---

# 2. Block-by-Block Analysis

## 2.1 Trigger and Repository Configuration

**Overview:**  
This block initializes the workflow manually and provides repository-level parameters used by downstream GitHub nodes. It centralizes basic setup so the rest of the workflow can use expressions rather than hardcoded values.

**Nodes Involved:**  
- When clicking ‘Execute workflow’
- Github Config

### Node: When clicking ‘Execute workflow’
- **Type and role:** `n8n-nodes-base.manualTrigger`; entry point for manual execution.
- **Configuration choices:** No parameters configured; it simply starts the workflow when manually executed in the editor.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input: none  
  - Output: `Github Config`
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** No runtime logic; only unavailable in unattended operation because it requires manual start.
- **Sub-workflow reference:** None.

### Node: Github Config
- **Type and role:** `n8n-nodes-base.set`; stores repository settings for reuse.
- **Configuration choices:** Defines four string fields:
  - `repoOwner`: `YOUR_GITHUB_USERNAME`
  - `repoName`: `blog-n8n`
  - `filePath`: `blog-post.md`
  - `reportFolder`: `qa-reports`
- **Key expressions or variables used:** Downstream nodes reference values such as `{{$json.repoOwner}}` or `{{ $('Github Config').item.json.repoName }}`.
- **Input and output connections:**  
  - Input: `When clicking ‘Execute workflow’`
  - Output: `Fetch Blog Post from GitHub`
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:** Misconfigured owner/repo/file path will cause GitHub fetch or commit failures later. `reportFolder` is defined but not actually used by the GitHub report path expressions.
- **Sub-workflow reference:** None.

---

## 2.2 File Retrieval and Preparation

**Overview:**  
This block retrieves the target Markdown file from GitHub, decodes it, and creates a numbered text version for accurate QA referencing. It also preserves the original content and GitHub SHA for possible later commit.

**Nodes Involved:**  
- Fetch Blog Post from GitHub
- Decode Base64 & Add Line Numbers

### Node: Fetch Blog Post from GitHub
- **Type and role:** `n8n-nodes-base.github`; reads a repository file.
- **Configuration choices:**  
  - Resource: `file`
  - Operation: `get`
  - Authentication: `oAuth2`
  - Owner: `={{ $json.repoOwner }}`
  - Repository: `={{ $json.repoName }}`
  - File path: `={{ $json.filePath }}`
  - `asBinaryProperty`: `false`
  - Additional parameter `reference` left blank, so the default branch is used.
- **Key expressions or variables used:** Pulls repository configuration directly from `Github Config`.
- **Input and output connections:**  
  - Input: `Github Config`
  - Output: `Decode Base64 & Add Line Numbers`
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:** OAuth misconfiguration, missing file, repository access denial, branch/path mismatch, rate limiting, or network/API failures.
- **Sub-workflow reference:** None.

### Node: Decode Base64 & Add Line Numbers
- **Type and role:** `n8n-nodes-base.code`; prepares content for AI analysis.
- **Configuration choices:** JavaScript code:
  - Reads `content`, `encoding`, `sha`, and `path` from the GitHub file response
  - Decodes `content` if `encoding === 'base64'`
  - Splits the file into lines
  - Builds `numberedContent` in the form `1: first line`, `2: second line`, etc.
  - Returns:
    - `originalContent`
    - `numberedContent`
    - `lineCount`
    - `sha`
    - `path`
- **Key expressions or variables used:** Uses `$input.first().json`.
- **Input and output connections:**  
  - Input: `Fetch Blog Post from GitHub`
  - Output: `QA Agent - Analyze Content`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - Unexpected encoding values
  - Empty file content
  - Binary or non-UTF-8 content causing decode issues
  - Very large files potentially stressing token limits in later AI stages
- **Sub-workflow reference:** None.

---

## 2.3 AI QA Analysis

**Overview:**  
This block sends line-numbered Markdown to an AI QA agent that detects editorial issues, then parses the response into structured issue records. Gemini is primary; Groq is configured as fallback.

**Nodes Involved:**  
- QA Agent - Analyze Content
- QA Agent LLM
- Fallback Chat Model
- Parse QA Issues JSON

### Node: QA Agent - Analyze Content
- **Type and role:** `@n8n/n8n-nodes-langchain.agent`; orchestrates the first AI reasoning step.
- **Configuration choices:**  
  - Prompt text: asks the model to analyze the blog post and return issues as a JSON array
  - System message defines required fields:
    - `line_number`
    - `issue_type`
    - `severity`
    - `description`
    - `suggested_fix`
  - Strict output requirements:
    - valid JSON array only
    - specific line numbers
    - substantive issues only
    - complete corrected line in `suggested_fix`
  - `needsFallback: true`
  - `onError: continueErrorOutput`
  - `retryOnFail: true`
  - `maxTries: 3`
  - `waitBetweenTries: 5000`
- **Key expressions or variables used:**  
  - Prompt body includes `{{ $json.numberedContent }}`
- **Input and output connections:**  
  - Main input: `Decode Base64 & Add Line Numbers`
  - AI language model input 0: `QA Agent LLM`
  - AI language model input 1: `Fallback Chat Model`
  - Main output: `Parse QA Issues JSON`
- **Version-specific requirements:** Type version `3.1`; requires compatible LangChain agent support in n8n.
- **Edge cases or potential failure types:**  
  - LLM returns non-JSON despite instructions
  - Model hallucinates line numbers
  - Token/context overflow on large posts
  - Provider auth or quota errors
  - Agent fallback behavior depends on proper credential setup for both model nodes
- **Sub-workflow reference:** None.

### Node: QA Agent LLM
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`; primary LLM for QA analysis.
- **Configuration choices:** No special options set.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Output as AI language model to `QA Agent - Analyze Content`
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** Invalid Gemini credentials, quota exhaustion, API errors, regional/model availability issues.
- **Sub-workflow reference:** None.

### Node: Fallback Chat Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatGroq`; fallback LLM for QA analysis.
- **Configuration choices:**  
  - Model: `openai/gpt-oss-20b`
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Output as AI language model fallback to `QA Agent - Analyze Content`
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** Invalid Groq credentials, unavailable model, fallback not invoked if the primary path fails in unsupported ways.
- **Sub-workflow reference:** None.

### Node: Parse QA Issues JSON
- **Type and role:** `n8n-nodes-base.code`; validates and filters AI QA output.
- **Configuration choices:** JavaScript logic:
  - Reads `output` from the QA agent
  - Strips possible markdown code fences like ```json
  - Parses JSON
  - Validates that output is an array
  - Filters issues to those with:
    - numeric `line_number`
    - present `issue_type`
    - present `description`
    - severity `high` or `medium`
  - Pulls `originalContent`, `sha`, and `path` from `Decode Base64 & Add Line Numbers`
  - Returns either:
    - success object with `issues`, `issueCount`, `originalContent`, `sha`, `path`, `parseSuccess: true`
    - or fallback object with empty issues and parse diagnostics
- **Key expressions or variables used:**  
  - `$('Decode Base64 & Add Line Numbers').first().json.originalContent`
  - `$('Decode Base64 & Add Line Numbers').first().json.sha`
  - `$('Decode Base64 & Add Line Numbers').first().json.path`
- **Input and output connections:**  
  - Input: `QA Agent - Analyze Content`
  - Output: `Has Issues?`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - Invalid JSON from AI
  - Empty or malformed agent output
  - Missing upstream content if execution context is broken
  - Low-severity-only issue lists are intentionally discarded
- **Sub-workflow reference:** None.

---

## 2.4 Decision: Issues Found or Not

**Overview:**  
This block branches the workflow depending on whether valid medium/high issues were found. It either creates a clean report or continues into edit generation.

**Nodes Involved:**  
- Has Issues?
- Format Clean Report
- Save QA Report to GitHub Without Issues

### Node: Has Issues?
- **Type and role:** `n8n-nodes-base.if`; checks whether any actionable issues were detected.
- **Configuration choices:** Condition:
  - `{{$json.issueCount}} > 0`
- **Key expressions or variables used:** `={{ $json.issueCount }}`
- **Input and output connections:**  
  - Input: `Parse QA Issues JSON`
  - True output: `Editor Agent - Generate Edit Ops`
  - False output: `Format Clean Report`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:** If `issueCount` is missing or non-numeric, strict validation may affect evaluation behavior.
- **Sub-workflow reference:** None.

### Node: Format Clean Report
- **Type and role:** `n8n-nodes-base.code`; creates a report when no issues are found.
- **Configuration choices:** JavaScript builds a Markdown report containing:
  - date
  - summary section
  - “No issues detected”
  - statement that no changes are required
- **Key expressions or variables used:**  
  - `$('Parse QA Issues JSON').first().json.path`
- **Input and output connections:**  
  - Input: `Has Issues?` false branch
  - Output: `Save QA Report to GitHub Without Issues`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:** If upstream path is unavailable, report metadata may be incomplete.
- **Sub-workflow reference:** None.

### Node: Save QA Report to GitHub Without Issues
- **Type and role:** `n8n-nodes-base.github`; writes a “clean” QA report to the repository.
- **Configuration choices:**  
  - Resource: `file`
  - Operation: file creation
  - Authentication: `oAuth2`
  - Owner/repository from `Github Config`
  - File path: `qa-reports/{{ timestamp }}-CLEAN-report.md`
  - Commit message: `AI QA: Content improvements - YYYY-MM-DD`
- **Key expressions or variables used:**  
  - `={{ $('Github Config').item.json.repoOwner }}`
  - `={{ $('Github Config').item.json.repoName }}`
  - `={{ $json.report }}`
  - dynamic timestamp via `new Date().toISOString()`
- **Input and output connections:**  
  - Input: `Format Clean Report`
  - Output: none
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:** Missing `qa-reports` path support, repo permission issues, duplicate path collisions if timestamps match, OAuth problems.
- **Sub-workflow reference:** None.

---

## 2.5 AI Edit Generation and Local Edit Application

**Overview:**  
If issues exist, this block converts them into executable edit operations and applies them locally to the text. The workflow sorts edits descending by line number to reduce index shifts during replacements, insertions, and deletions.

**Nodes Involved:**  
- Editor Agent - Generate Edit Ops
- Editor Agent LLM
- Fallback Model
- Parse Edit Operations JSON
- Apply Line-by-Line Edits

### Node: Editor Agent - Generate Edit Ops
- **Type and role:** `@n8n/n8n-nodes-langchain.agent`; converts QA findings into structured edit operations.
- **Configuration choices:**  
  - Prompt provides serialized `issues`
  - System message requires JSON array of operations with:
    - `operation`: `replace`, `insert_after`, `delete`
    - `line_number`
    - `new_text` where applicable
  - Strict output rules:
    - JSON only
    - sorted descending by line number
    - skip items where `suggested_fix` is null
    - line number must be positive integer
  - `needsFallback: true`
  - `onError: continueErrorOutput`
  - `retryOnFail: true`
  - `maxTries: 3`
  - `waitBetweenTries: 5000`
- **Key expressions or variables used:**  
  - `{{ JSON.stringify($json.issues, null, 2) }}`
- **Input and output connections:**  
  - Main input: `Has Issues?` true branch
  - AI language model input 0: `Editor Agent LLM`
  - AI language model input 1: `Fallback Model`
  - Main output: `Parse Edit Operations JSON`
- **Version-specific requirements:** Type version `3.1`.
- **Edge cases or potential failure types:**  
  - Model emits malformed JSON
  - Operations are logically invalid even if syntactically correct
  - `delete` operations may be generated despite prompt saying omit `new_text`
  - Line references may drift if AI misunderstands numbering
- **Sub-workflow reference:** None.

### Node: Editor Agent LLM
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`; primary editor model.
- **Configuration choices:** No special options set.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Output as AI language model to `Editor Agent - Generate Edit Ops`
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** Credential, quota, or provider-side model issues.
- **Sub-workflow reference:** None.

### Node: Fallback Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatGroq`; fallback editor model.
- **Configuration choices:**  
  - Model: `openai/gpt-oss-20b`
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Output as AI language model fallback to `Editor Agent - Generate Edit Ops`
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** Same general Groq risks as above.
- **Sub-workflow reference:** None.

### Node: Parse Edit Operations JSON
- **Type and role:** `n8n-nodes-base.code`; validates editor output and normalizes operation order.
- **Configuration choices:** JavaScript logic:
  - Reads `output`
  - Removes possible markdown code fences
  - Parses JSON
  - Validates array structure
  - Sorts edits descending by `line_number`
  - Pulls forward:
    - `originalContent`
    - `sha`
    - `path`
    - `issues`
  - Returns parse diagnostics on failure
- **Key expressions or variables used:**  
  - `$('Parse QA Issues JSON').first().json.originalContent`
  - `$('Parse QA Issues JSON').first().json.sha`
  - `$('Parse QA Issues JSON').first().json.path`
  - `$('Parse QA Issues JSON').first().json.issues`
- **Input and output connections:**  
  - Input: `Editor Agent - Generate Edit Ops`
  - Output: `Apply Line-by-Line Edits`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - Malformed JSON
  - Missing `line_number`
  - Non-array response
  - Parse failure still leads to downstream node with empty edits
- **Sub-workflow reference:** None.

### Node: Apply Line-by-Line Edits
- **Type and role:** `n8n-nodes-base.code`; applies edit operations to the original text.
- **Configuration choices:** JavaScript logic:
  - Splits original content into lines
  - Re-sorts edits descending by `line_number`
  - For each edit:
    - validates line range
    - supports `replace`, `insert_after`, `delete`
    - tracks `appliedChanges` and `skippedChanges`
  - Computes:
    - `applied`
    - `skipped`
    - `successRate`
    - `success`
  - Success rule:
    - `successRate >= 0.5`
    - and `appliedCount > 0`
  - Returns `updatedContent` plus edit summary
- **Key expressions or variables used:** Uses `$input.first().json.*`.
- **Input and output connections:**  
  - Input: `Parse Edit Operations JSON`
  - Output: `Edits Applied?`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - Out-of-range line numbers
  - Unknown operation values
  - Edit collisions on same line
  - Insert/delete combinations may still produce unexpected content
  - Empty edit list results in `successRate = 1` but `success = false` because `appliedCount > 0` is required
- **Sub-workflow reference:** None.

---

## 2.6 Decision: Safe to Commit or Not

**Overview:**  
This block decides whether the applied edits are trustworthy enough to commit. Successful application leads to both content update and report creation; failure leads only to a failure report.

**Nodes Involved:**  
- Edits Applied?
- Commit Updated File to GitHub
- Format QA Report
- Save QA Report to GitHub
- Format Failure Report
- Create a file

### Node: Edits Applied?
- **Type and role:** `n8n-nodes-base.if`; checks whether local application succeeded.
- **Configuration choices:** Condition:
  - `{{$json.success}}` is `true`
- **Key expressions or variables used:** `={{ $json.success }}`
- **Input and output connections:**  
  - Input: `Apply Line-by-Line Edits`
  - True output: `Commit Updated File to GitHub`, `Format QA Report`
  - False output: `Format Failure Report`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:** If `success` is undefined because of upstream code issues, the false path is likely taken.
- **Sub-workflow reference:** None.

### Node: Commit Updated File to GitHub
- **Type and role:** `n8n-nodes-base.github`; updates the original Markdown file in the repository.
- **Configuration choices:**  
  - Resource: `file`
  - Operation: `edit`
  - Authentication: `oAuth2`
  - Owner/repository/path from `Github Config`
  - File content: `={{ $json.updatedContent }}`
  - Commit message: `AI QA: Applied X edit(s) - YYYY-MM-DD`
- **Key expressions or variables used:**  
  - `={{ $('Github Config').item.json.repoOwner }}`
  - `={{ $('Github Config').item.json.repoName }}`
  - `={{ $('Github Config').item.json.filePath }}`
  - `={{ $json.updatedContent }}`
- **Input and output connections:**  
  - Input: `Edits Applied?` true branch
  - Output: none
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**  
  - OAuth permission issues
  - SHA conflicts / concurrent file updates
  - Branch protections
  - Updating wrong path if config changed after fetch
- **Sub-workflow reference:** None.

### Node: Format QA Report
- **Type and role:** `n8n-nodes-base.code`; builds a Markdown report describing findings and applied changes.
- **Configuration choices:** JavaScript creates sections for:
  - report date
  - summary with applied/skipped/success rate
  - list of issues found
  - list of applied changes
  - optional skipped edits section
- **Key expressions or variables used:** Reads values from current input such as `issues`, `appliedChanges`, `skippedChanges`, `successRate`, `sha`, `path`, `updatedContent`, `success`.
- **Input and output connections:**  
  - Input: `Edits Applied?` true branch
  - Output: `Save QA Report to GitHub`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:** Missing issue/change arrays could result in incomplete report sections.
- **Sub-workflow reference:** None.

### Node: Save QA Report to GitHub
- **Type and role:** `n8n-nodes-base.github`; saves the success QA report to the repository.
- **Configuration choices:**  
  - Resource: `file`
  - Operation: file creation
  - File path: `qa-reports/{{ timestamp }}-report.md`
  - Commit message: `AI QA: Content improvements - YYYY-MM-DD`
  - Owner/repository from `Github Config`
- **Key expressions or variables used:**  
  - `={{ $('Github Config').item.json.repoOwner }}`
  - `={{ $('Github Config').item.json.repoName }}`
  - `={{ $json.report }}`
- **Input and output connections:**  
  - Input: `Format QA Report`
  - Output: none
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:** Same GitHub create-file risks as other report nodes.
- **Sub-workflow reference:** None.

### Node: Format Failure Report
- **Type and role:** `n8n-nodes-base.code`; generates a report when edits were not safely applicable.
- **Configuration choices:** JavaScript builds a Markdown report containing:
  - “Commit Skipped”
  - applied/skipped/success rate summary
  - skipped edit details if any
  - next-step recommendations
- **Key expressions or variables used:** Pulls current input values like `skippedChanges`, `applied`, `skipped`, `successRate`, and `path`.
- **Input and output connections:**  
  - Input: `Edits Applied?` false branch
  - Output: `Create a file`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:** Minimal risk; mostly depends on upstream structure.
- **Sub-workflow reference:** None.

### Node: Create a file
- **Type and role:** `n8n-nodes-base.github`; stores the failure report in GitHub.
- **Configuration choices:**  
  - Resource: `file`
  - Operation: file creation
  - File path: `qa-reports/{{ timestamp }}-FAILED-report.md`
  - Commit message: `AI QA: Edit application failed - YYYY-MM-DD`
  - Owner/repository from `Github Config`
- **Key expressions or variables used:**  
  - `={{ $('Github Config').item.json.repoOwner }}`
  - `={{ $('Github Config').item.json.repoName }}`
  - `={{ $json.report }}`
- **Input and output connections:**  
  - Input: `Format Failure Report`
  - Output: none
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:** Same GitHub create-file constraints and permissions issues.
- **Sub-workflow reference:** None.

---

## 2.7 Sticky Notes and Inline Documentation

**Overview:**  
These nodes provide visual documentation in the canvas. They do not affect runtime execution but are important for maintainability.

**Nodes Involved:**  
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5

### Node: Sticky Note
- **Type and role:** `n8n-nodes-base.stickyNote`; overall workflow documentation.
- **Configuration choices:** Large note explaining:
  - end-to-end logic
  - setup steps
  - customization ideas
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

### Node: Sticky Note1
- **Type and role:** sticky note; labels the fetch block as “Get blog post”.
- **Configuration choices:** Visual grouping only.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

### Node: Sticky Note2
- **Type and role:** sticky note; labels the analysis block.
- **Configuration choices:** Visual grouping only.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

### Node: Sticky Note3
- **Type and role:** sticky note; labels the generate/apply edits block.
- **Configuration choices:** Visual grouping only.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

### Node: Sticky Note4
- **Type and role:** sticky note; labels the repository update block.
- **Configuration choices:** Visual grouping only.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

### Node: Sticky Note5
- **Type and role:** sticky note; labels the “no issues found” path.
- **Configuration choices:** Visual grouping only.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Manual entry point |  | Github Config | ## Get blog post |
| Github Config | Set | Stores repo/file parameters | When clicking ‘Execute workflow’ | Fetch Blog Post from GitHub | ## Get blog post |
| Fetch Blog Post from GitHub | GitHub | Retrieves Markdown file content from repository | Github Config | Decode Base64 & Add Line Numbers | ## Get blog post |
| Decode Base64 & Add Line Numbers | Code | Decodes content and adds line numbering | Fetch Blog Post from GitHub | QA Agent - Analyze Content | ## Get blog post |
| QA Agent - Analyze Content | LangChain Agent | Finds content issues in numbered Markdown | Decode Base64 & Add Line Numbers; QA Agent LLM; Fallback Chat Model | Parse QA Issues JSON | ## Analyze content |
| QA Agent LLM | Google Gemini Chat Model | Primary LLM for QA analysis |  | QA Agent - Analyze Content | ## Analyze content |
| Fallback Chat Model | Groq Chat Model | Fallback LLM for QA analysis |  | QA Agent - Analyze Content | ## Analyze content |
| Parse QA Issues JSON | Code | Parses and filters AI issue list | QA Agent - Analyze Content | Has Issues? | ## Analyze content |
| Has Issues? | If | Routes based on whether actionable issues exist | Parse QA Issues JSON | Editor Agent - Generate Edit Ops; Format Clean Report |  |
| Editor Agent - Generate Edit Ops | LangChain Agent | Converts issues into edit operations | Has Issues?; Editor Agent LLM; Fallback Model | Parse Edit Operations JSON | ## Generate & apply edits |
| Editor Agent LLM | Google Gemini Chat Model | Primary LLM for edit generation |  | Editor Agent - Generate Edit Ops | ## Generate & apply edits |
| Fallback Model | Groq Chat Model | Fallback LLM for edit generation |  | Editor Agent - Generate Edit Ops | ## Generate & apply edits |
| Parse Edit Operations JSON | Code | Parses and sorts edit operations | Editor Agent - Generate Edit Ops | Apply Line-by-Line Edits | ## Generate & apply edits |
| Apply Line-by-Line Edits | Code | Applies line-based edits to original content | Parse Edit Operations JSON | Edits Applied? | ## Generate & apply edits |
| Edits Applied? | If | Decides whether edits are safe enough to commit | Apply Line-by-Line Edits | Commit Updated File to GitHub; Format QA Report; Format Failure Report | ## Generate & apply edits |
| Commit Updated File to GitHub | GitHub | Commits corrected blog post back to repository | Edits Applied? |  | ## Update repository |
| Format QA Report | Code | Builds success report for applied edits | Edits Applied? | Save QA Report to GitHub | ## Update repository |
| Save QA Report to GitHub | GitHub | Saves success QA report to repository | Format QA Report |  | ## Update repository |
| Format Failure Report | Code | Builds report when edits are not safely applied | Edits Applied? | Create a file | ## Update repository |
| Create a file | GitHub | Saves failure report to repository | Format Failure Report |  | ## Update repository |
| Format Clean Report | Code | Builds report for issue-free content | Has Issues? | Save QA Report to GitHub Without Issues | ## Path: No Issues Found |
| Save QA Report to GitHub Without Issues | GitHub | Saves clean QA report to repository | Format Clean Report |  | ## Path: No Issues Found |
| Sticky Note | Sticky Note | Canvas documentation and setup guidance |  |  | ## AI Automated Markdown Blog QA  \n### How it works  \n1.  Fetch: A manual trigger fetches your target markdown file from GitHub.  \n2. Prep: The file is decoded and numbered for the AI.  \n3. Agent 1 (Analysis): The QA Agent identifies tone, clarity, and grammar issues, outputting JSON.  \n4. Agent 2 (Editing): The Editor Agent translates those issues into precise line-by-line edits.  \n5. Execute: The workflow applies the edits, then commits the updated file and a Markdown report to GitHub.  \n### Setup steps  \n- [ ] Connect GitHub OAuth2 to all GitHub nodes.  \n- [ ] Add Google Gemini and Groq API keys.  \n- [ ] Update the Github Config node (repoOwner, repoName, filePath).  \n### Customization  \n- Customize  \n- Style: Edit the QA Agent's prompt to enforce brand rules.  \n- Models: Swap Gemini/Groq for OpenAI or Anthropic.  \n- Routing: Adjust the GitHub nodes to change report destinations. |
| Sticky Note1 | Sticky Note | Visual block label |  |  | ## Get blog post |
| Sticky Note2 | Sticky Note | Visual block label |  |  | ## Analyze content |
| Sticky Note3 | Sticky Note | Visual block label |  |  | ## Generate & apply edits |
| Sticky Note4 | Sticky Note | Visual block label |  |  | ## Update repository |
| Sticky Note5 | Sticky Note | Visual block label |  |  | ## Path: No Issues Found |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like **AI Markdown Proofreader with GitHub Auto-Commit**.

2. **Add a Manual Trigger node**.
   - Node type: **Manual Trigger**
   - Keep default settings.
   - This is the workflow entry point.

3. **Add a Set node** named **Github Config**.
   - Connect it after the Manual Trigger.
   - Add string fields:
     - `repoOwner` = your GitHub username or organization
     - `repoName` = target repository name
     - `filePath` = path to the Markdown file, for example `blog-post.md`
     - `reportFolder` = `qa-reports`
   - Even though `reportFolder` is stored, note that this workflow hardcodes `qa-reports` in later report nodes rather than referencing this field.

4. **Add a GitHub node** named **Fetch Blog Post from GitHub**.
   - Connect it after **Github Config**.
   - Configure:
     - **Authentication:** GitHub OAuth2
     - **Resource:** File
     - **Operation:** Get
     - **Owner:** expression `{{$json.repoOwner}}`
     - **Repository:** expression `{{$json.repoName}}`
     - **File Path:** expression `{{$json.filePath}}`
     - **As Binary Property:** false
     - Leave branch/reference blank to use the default branch
   - Configure a GitHub OAuth2 credential with repository read access.

5. **Add a Code node** named **Decode Base64 & Add Line Numbers**.
   - Connect it after the fetch node.
   - Paste logic equivalent to:
     - Read `content`, `encoding`, `sha`, `path`
     - If encoding is base64, decode to UTF-8
     - Split by newline
     - Prefix every line with its 1-based line number
     - Return:
       - `originalContent`
       - `numberedContent`
       - `lineCount`
       - `sha`
       - `path`

6. **Add a LangChain Agent node** named **QA Agent - Analyze Content**.
   - Connect it after the decode node.
   - Set:
     - **Prompt Type:** Define
     - **Text:** ask the model to analyze the blog post and return issues as JSON array, injecting `{{$json.numberedContent}}`
     - **System Message:** instruct the model to return only a JSON array with:
       - `line_number`
       - `issue_type`
       - `severity`
       - `description`
       - `suggested_fix`
     - Include strict rules:
       - JSON only
       - line numbers must be precise
       - include blank lines in numbering
       - focus on substantive issues
       - `suggested_fix` should be the full corrected line
   - Enable:
     - **Needs Fallback**
     - **Retry on Fail**
   - Set:
     - **Max Tries:** 3
     - **Wait Between Tries:** 5000 ms
   - Optional but recommended: set **On Error** to continue error output as in the source workflow.

7. **Add a Google Gemini Chat Model node** named **QA Agent LLM**.
   - Connect it to the QA agent’s **AI language model** input.
   - Use Google Gemini credentials.
   - Keep default model/options unless your n8n version requires explicit model selection.
   - Ensure the credential has access to the Gemini API.

8. **Add a Groq Chat Model node** named **Fallback Chat Model**.
   - Connect it to the QA agent’s fallback AI language model input.
   - Configure model: `openai/gpt-oss-20b`
   - Add Groq credentials.

9. **Add a Code node** named **Parse QA Issues JSON**.
   - Connect it after **QA Agent - Analyze Content**.
   - Implement logic to:
     - Read agent output from `output`
     - Strip markdown fences if present
     - Parse JSON
     - Verify it is an array
     - Filter to valid issues with:
       - numeric `line_number`
       - non-empty `issue_type`
       - non-empty `description`
       - severity only `medium` or `high`
     - Reattach:
       - `originalContent`
       - `sha`
       - `path`
     - Return `issueCount`
     - On parse failure, return empty issues and diagnostic fields

10. **Add an If node** named **Has Issues?**
    - Connect it after **Parse QA Issues JSON**.
    - Condition:
      - numeric comparison
      - `{{$json.issueCount}}` greater than `0`

11. **Add a Code node** named **Format Clean Report** for the false branch.
    - Connect the **false** output of **Has Issues?** to it.
    - Build a Markdown report stating:
      - current date
      - no issues detected
      - no changes required
    - Include the file path if desired.

12. **Add a GitHub node** named **Save QA Report to GitHub Without Issues**.
    - Connect it after **Format Clean Report**.
    - Configure:
      - **Authentication:** GitHub OAuth2
      - **Resource:** File
      - **Operation:** Create file
      - **Owner:** `{{ $('Github Config').item.json.repoOwner }}`
      - **Repository:** `{{ $('Github Config').item.json.repoName }}`
      - **File Path:** `qa-reports/{{ new Date().toISOString().slice(0,19).replace(/:/g, '-') }}-CLEAN-report.md`
      - **File Content:** `{{$json.report}}`
      - **Commit Message:** `AI QA: Content improvements - {{ new Date().toISOString().slice(0,10) }}`
    - Ensure the repository allows creation of files under `qa-reports/`.

13. **Add a second LangChain Agent node** named **Editor Agent - Generate Edit Ops** for the true branch.
    - Connect the **true** output of **Has Issues?** to it.
    - Configure:
      - **Prompt Type:** Define
      - **Text:** convert QA issues into edit operations using `{{ JSON.stringify($json.issues, null, 2) }}`
      - **System Message:** require JSON array only with fields:
        - `operation`
        - `line_number`
        - `new_text`
      - Constrain operations to:
        - `replace`
        - `insert_after`
        - `delete`
      - Require descending sort by line number
      - Only include operations when `suggested_fix` is not null
      - Require positive integer line numbers
    - Enable fallback and retries exactly as in the QA agent.

14. **Add a Google Gemini Chat Model node** named **Editor Agent LLM**.
    - Connect it to the editor agent’s primary AI language model input.
    - Use Gemini credentials.

15. **Add a Groq Chat Model node** named **Fallback Model**.
    - Connect it to the editor agent’s fallback AI language model input.
    - Set model to `openai/gpt-oss-20b`.
    - Use Groq credentials.

16. **Add a Code node** named **Parse Edit Operations JSON**.
    - Connect it after the editor agent.
    - Implement logic to:
      - read `output`
      - strip markdown code fences
      - parse JSON
      - ensure array format
      - sort edits descending by `line_number`
      - carry forward:
        - `originalContent`
        - `sha`
        - `path`
        - `issues`
      - if parsing fails, return empty `edits`, `editCount = 0`, and diagnostics

17. **Add a Code node** named **Apply Line-by-Line Edits**.
    - Connect it after **Parse Edit Operations JSON**.
    - Implement logic to:
      - split `originalContent` into lines
      - apply operations in descending line order
      - support:
        - `replace`
        - `insert_after`
        - `delete`
      - skip invalid operations or out-of-range lines
      - track:
        - `applied`
        - `skipped`
        - `appliedChanges`
        - `skippedChanges`
      - calculate:
        - `successRate = applied / (applied + skipped)` or `1` when no operations exist
        - `success = successRate >= 0.5 && applied > 0`
      - return:
        - `updatedContent`
        - `sha`
        - `path`
        - `issues`
        - `success`
        - summary arrays

18. **Add an If node** named **Edits Applied?**
    - Connect it after **Apply Line-by-Line Edits**.
    - Condition:
      - boolean true on `{{$json.success}}`

19. **Add a GitHub node** named **Commit Updated File to GitHub** on the true branch.
    - Connect the **true** output of **Edits Applied?** to it.
    - Configure:
      - **Authentication:** GitHub OAuth2
      - **Resource:** File
      - **Operation:** Edit
      - **Owner:** `{{ $('Github Config').item.json.repoOwner }}`
      - **Repository:** `{{ $('Github Config').item.json.repoName }}`
      - **File Path:** `{{ $('Github Config').item.json.filePath }}`
      - **File Content:** `{{$json.updatedContent}}`
      - **Commit Message:** `AI QA: Applied {{ $json.applied }} edit(s) - {{ new Date().toISOString().slice(0,10) }}`
    - Use a GitHub credential with write access.

20. **Add a Code node** named **Format QA Report** on the true branch.
    - Also connect the **true** output of **Edits Applied?** to this node.
    - Build a Markdown report including:
      - date
      - applied/skipped counts
      - success rate
      - issue list
      - applied change list
      - optional skipped edits section

21. **Add a GitHub node** named **Save QA Report to GitHub**.
    - Connect it after **Format QA Report**.
    - Configure:
      - **Authentication:** GitHub OAuth2
      - **Resource:** File
      - **Operation:** Create file
      - **Owner:** `{{ $('Github Config').item.json.repoOwner }}`
      - **Repository:** `{{ $('Github Config').item.json.repoName }}`
      - **File Path:** `qa-reports/{{ new Date().toISOString().slice(0,19).replace(/:/g, '-') }}-report.md`
      - **File Content:** `{{$json.report}}`
      - **Commit Message:** `AI QA: Content improvements - {{ new Date().toISOString().slice(0,10) }}`

22. **Add a Code node** named **Format Failure Report** on the false branch.
    - Connect the **false** output of **Edits Applied?** to it.
    - Build a Markdown report containing:
      - commit skipped notice
      - applied/skipped counts
      - success rate
      - skipped edit details
      - next steps for manual review

23. **Add a GitHub node** named **Create a file**.
    - Connect it after **Format Failure Report**.
    - Configure:
      - **Authentication:** GitHub OAuth2
      - **Resource:** File
      - **Operation:** Create file
      - **Owner:** `{{ $('Github Config').item.json.repoOwner }}`
      - **Repository:** `{{ $('Github Config').item.json.repoName }}`
      - **File Path:** `qa-reports/{{ new Date().toISOString().slice(0,19).replace(/:/g, '-') }}-FAILED-report.md`
      - **File Content:** `{{$json.report}}`
      - **Commit Message:** `AI QA: Edit application failed - {{ new Date().toISOString().slice(0,10) }}`

24. **Create credentials** required by the workflow:
    - **GitHub OAuth2**
      - Use a GitHub app or OAuth app configuration supported by n8n
      - Grant repository read/write access
    - **Google Gemini / Google AI credential**
      - Use an API key or the credential type supported by your n8n version
    - **Groq credential**
      - Add your Groq API key

25. **Optionally add sticky notes** to mirror the original canvas organization:
    - “Get blog post”
    - “Analyze content”
    - “Generate & apply edits”
    - “Update repository”
    - “Path: No Issues Found”
    - Plus one large note with setup steps and customization guidance

26. **Test with a small Markdown file first**.
    - Confirm GitHub fetch works
    - Confirm LLM nodes return valid JSON
    - Confirm reports are created in `qa-reports/`
    - Confirm the file update happens only when `success` is true

27. **Harden the workflow before production use**.
    - Consider adding:
      - branch targeting
      - PR creation instead of direct commit
      - stricter JSON schema validation
      - file-size guardrails
      - separate handling for parse failures vs zero-issue results

**Sub-workflow setup:**  
This workflow does not call any sub-workflows and is not itself documented as a callable child workflow. No Execute Workflow node is present.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| AI Automated Markdown Blog QA: fetch Markdown from GitHub, decode and number lines, run QA analysis, generate line-by-line edits, apply them, then commit both updated file and report to GitHub. | Overall workflow purpose |
| Setup steps: connect GitHub OAuth2 to all GitHub nodes; add Google Gemini and Groq API keys; update the Github Config node values (`repoOwner`, `repoName`, `filePath`). | Initial configuration |
| Customization: edit the QA Agent prompt for brand/style rules. | Prompt tuning |
| Customization: swap Gemini/Groq with OpenAI or Anthropic if preferred. | Model selection |
| Customization: adjust GitHub nodes to change report destinations. | Repository/report routing |

## Additional implementation notes
- The workflow has a single manual entry point.
- There are no sub-workflows.
- `reportFolder` is configured but not referenced by report-writing nodes; if desired, replace hardcoded `qa-reports/...` paths with expressions using that field.
- The workflow directly edits the source file in GitHub rather than creating a pull request.
- Parse failures in the QA stage currently behave like “no issues found” because `issueCount` becomes `0`; if that is undesirable, add a separate branch for `parseSuccess === false`.
- Edit parse failures flow into the edit-application stage with an empty edit list and usually end in the failure-report branch.