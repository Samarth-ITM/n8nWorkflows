Review GitLab merge requests with parallel Azure OpenAI reviewers

https://n8nworkflows.xyz/workflows/review-gitlab-merge-requests-with-parallel-azure-openai-reviewers-14338


# Review GitLab merge requests with parallel Azure OpenAI reviewers

# 1. Workflow Overview

This workflow performs AI-assisted code review for GitLab merge requests. It is triggered when someone posts a specific comment in a merge request discussion thread, then collects the merge request diff, prepares per-file review context, sends each file through three parallel Azure OpenAI reviewers, verifies and filters the findings, posts actionable comments back to GitLab, and finally adds a summary reply to the original discussion.

Typical use cases:
- Automated first-pass review of GitLab merge requests
- Parallel triage of bugs, security risks, and maintainability issues
- Structured AI review with confidence gating before publishing comments
- GitLab discussion-driven review initiation instead of automatic review on every MR event

## 1.1 Trigger and workflow configuration
The workflow starts from a GitLab discussion webhook and immediately loads a central configuration object containing the GitLab API base URL, trigger phrase, confidence threshold, and standard reply messages.

## 1.2 Trigger validation and initial acknowledgment
It checks whether the posted note exactly matches the configured review trigger phrase. If it matches, the workflow both:
- posts a “review started” reply to the triggering discussion
- fetches the merge request changes from GitLab

## 1.3 Diff extraction and filtering
The merge request changes are split into one item per changed file. Unsupported diffs are filtered out, specifically renamed files, deleted files, and diffs without standard hunk markers.

## 1.4 Review context preparation
For each supported file diff, the workflow derives normalized context:
- file paths
- raw diff
- extracted old/new code fragments
- a logical-to-real new-line mapping
- review-ready code annotated with GitLab line numbers
- MR metadata and diff SHAs needed later for inline GitLab comments

## 1.5 Parallel AI review
Each eligible file is reviewed in parallel by three AI reviewers:
- Bug reviewer
- Security reviewer
- Maintainability reviewer

Each reviewer uses an Azure OpenAI chat model plus a structured output parser so the result is machine-readable.

## 1.6 Findings merge and verification
The reviewer outputs are merged, obvious duplicates are removed, and a verifier model re-evaluates all findings against the diff. The verifier normalizes severity, confidence, line targeting, and whether a finding should be kept, dropped, or treated as duplicate/pre-existing.

## 1.7 Publishing decisions
Verified findings are normalized into GitLab-postable metadata, then filtered so only findings with:
- `status = keep`
- `final_confidence >= minConfidenceToPost`

are eligible for publication.

## 1.8 Comment posting and summary reply
For each publishable finding, the workflow attempts to resolve a valid inline diff position. If successful, it posts an inline MR discussion comment. Otherwise, it posts a fallback reply in the original discussion. Independently, it always posts a final summary reply indicating either completion or that no issues were found.

---

# 2. Block-by-Block Analysis

## 2.1 Trigger and Configuration

### Overview
This block receives GitLab discussion events and injects all configurable workflow settings into the execution. It defines the review trigger phrase, confidence threshold, GitLab base URL, and standard status messages.

### Nodes Involved
- GitLab Discussion Webhook
- Workflow Configuration

### Node Details

#### GitLab Discussion Webhook
- **Type and role:** `n8n-nodes-base.webhook`; entry point for GitLab webhook events
- **Configuration choices:**  
  - HTTP method: `POST`
  - Webhook path: fixed generated path
- **Key expressions or variables used:** none internally; downstream nodes read `body.*`
- **Input and output connections:**  
  - Input: none
  - Output: `Workflow Configuration`
- **Version-specific requirements:** Type version `2`
- **Edge cases / failures:**  
  - GitLab webhook not configured or incorrect endpoint
  - Wrong HTTP method
  - Missing expected JSON payload fields such as `body.object_attributes.note`
- **Sub-workflow reference:** none

#### Workflow Configuration
- **Type and role:** `n8n-nodes-base.set`; central configuration store
- **Configuration choices:**  
  - Adds these fields:
    - `gitlabBaseUrl = https://gitlab.example.com/api/v4`
    - `reviewTriggerPhrase = +0`
    - `minConfidenceToPost = 75`
    - `reviewStartedMessage = 🔍 I'm reviewing the code right now. Please hang tight for a moment.`
    - `noIssuesMessage = 🟢 The changes look good this time!`
    - `summaryMessage = ✅ Review complete!`
  - `includeOtherFields = true`, so the original webhook payload is preserved
- **Key expressions or variables used:** values are later referenced through `$('Workflow Configuration').first().json.*`
- **Input and output connections:**  
  - Input: `GitLab Discussion Webhook`
  - Output: `Check Review Trigger Comment`
- **Version-specific requirements:** Type version `3.4`
- **Edge cases / failures:**  
  - Misconfigured GitLab base URL breaks all later HTTP requests
  - Non-numeric threshold or malformed messages can break expressions or post unexpected content
- **Sub-workflow reference:** none

---

## 2.2 Trigger Validation and Review Start Reply

### Overview
This block ensures the workflow only runs when the posted discussion comment exactly matches the configured trigger phrase. If matched, it posts a “review started” reply and fetches merge request changes in parallel.

### Nodes Involved
- Check Review Trigger Comment
- Post Review Started Reply
- Fetch Merge Request Changes

### Node Details

#### Check Review Trigger Comment
- **Type and role:** `n8n-nodes-base.if`; gatekeeper for workflow activation
- **Configuration choices:**  
  - Compares the discussion note text with the configured trigger phrase using strict equality
- **Key expressions or variables used:**  
  - Left: `{{$json.body.object_attributes.note}}`
  - Right: `{{$('Workflow Configuration').first().json.reviewTriggerPhrase}}`
- **Input and output connections:**  
  - Input: `Workflow Configuration`
  - True output: `Fetch Merge Request Changes`, `Post Review Started Reply`
  - False output: none
- **Version-specific requirements:** Type version `2.2`
- **Edge cases / failures:**  
  - Exact match means whitespace, casing, or formatting differences will prevent triggering
  - Missing `body.object_attributes.note` field causes expression issues or false condition results
- **Sub-workflow reference:** none

#### Post Review Started Reply
- **Type and role:** `n8n-nodes-base.httpRequest`; posts a status acknowledgment in the same GitLab discussion
- **Configuration choices:**  
  - Method: `POST`
  - Authentication: predefined `gitlabApi`
  - Content type: multipart form-data
  - Endpoint targets MR discussion notes
  - Sends configured review-start message as `body`
- **Key expressions or variables used:**  
  - URL built from:
    - `$('Workflow Configuration').first().json.gitlabBaseUrl`
    - `$('GitLab Discussion Webhook').item.json.body.project.id`
    - `$('GitLab Discussion Webhook').item.json.body.merge_request.iid`
    - `$('GitLab Discussion Webhook').item.json.body.object_attributes.discussion_id`
  - Body:
    - `$('Workflow Configuration').first().json.reviewStartedMessage`
- **Input and output connections:**  
  - Input: `Check Review Trigger Comment` true branch
  - Output: none
- **Version-specific requirements:** Type version `4.2`
- **Edge cases / failures:**  
  - GitLab auth failure
  - Wrong project/MR/discussion IDs in webhook payload
  - GitLab instance URL mismatch
- **Sub-workflow reference:** none

#### Fetch Merge Request Changes
- **Type and role:** `n8n-nodes-base.httpRequest`; retrieves the MR diff and diff refs
- **Configuration choices:**  
  - Method: GET (default)
  - Authentication: predefined `gitlabApi`
  - Calls `/projects/{project_id}/merge_requests/{iid}/changes`
- **Key expressions or variables used:**  
  - `{{$json["body"]["project_id"]}}`
  - `{{$json["body"]["merge_request"]["iid"]}}`
  - Base URL from configuration
- **Input and output connections:**  
  - Input: `Check Review Trigger Comment` true branch
  - Output: `Split Changed Files`
- **Version-specific requirements:** Type version `4.2`
- **Edge cases / failures:**  
  - GitLab API permissions insufficient to access MR changes
  - API timeout on very large merge requests
  - Unexpected payload shape if GitLab version differs
- **Sub-workflow reference:** none

---

## 2.3 Diff Extraction and Filtering

### Overview
This block converts the merge request changes payload into one item per changed file and discards unsupported diff types. It protects the review pipeline from files that cannot be commented on reliably.

### Nodes Involved
- Split Changed Files
- Filter Supported Diffs

### Node Details

#### Split Changed Files
- **Type and role:** `n8n-nodes-base.splitOut`; expands the `changes` array into one item per file
- **Configuration choices:**  
  - `fieldToSplitOut = changes`
- **Key expressions or variables used:** none
- **Input and output connections:**  
  - Input: `Fetch Merge Request Changes`
  - Output: `Filter Supported Diffs`
- **Version-specific requirements:** Type version `1`
- **Edge cases / failures:**  
  - Missing or empty `changes` array results in no downstream items
- **Sub-workflow reference:** none

#### Filter Supported Diffs
- **Type and role:** `n8n-nodes-base.if`; excludes files not suitable for review/comment posting
- **Configuration choices:**  
  - Requires:
    - `renamed_file = false`
    - `deleted_file = false`
    - diff contains a hunk header matching `@@ -x,y +a,b @@`
- **Key expressions or variables used:**  
  - `{{$json.renamed_file}}`
  - `{{$json.deleted_file}}`
  - Regex test:
    `{{ /(^|\n)@@ -\d+(?:,\d+)? \+\d+(?:,\d+)? @@/m.test($json.diff ?? '') }}`
- **Input and output connections:**  
  - Input: `Split Changed Files`
  - True output: `Prepare Review Context`
  - False output: none
- **Version-specific requirements:** Type version `2.2`
- **Edge cases / failures:**  
  - Binary diffs or unusual patches are excluded
  - Files with no commentable hunk markers are skipped
  - Renamed files with useful content changes are intentionally ignored
- **Sub-workflow reference:** none

---

## 2.4 Review Context Preparation

### Overview
This block converts each file diff into a review-ready payload for the AI reviewers and later GitLab posting logic. It builds both code excerpts and line mapping metadata needed to resolve inline comment positions.

### Nodes Involved
- Prepare Review Context

### Node Details

#### Prepare Review Context
- **Type and role:** `n8n-nodes-base.code`; parses Git diff and constructs normalized context
- **Configuration choices:**  
  - Runs once per item
  - Parses hunk headers and diff lines
  - Builds:
    - `path`, `oldPath`, `newPath`
    - `gitDiff`
    - `originalCode`
    - `newCode`
    - `newCodeLineMap`
    - `reviewableNewCode`
    - MR metadata
    - `startSha`, `headSha`, `baseSha`
- **Key expressions or variables used:**  
  Reads from:
  - current diff item: `$input.item.json.*`
  - webhook payload:
    - `$('GitLab Discussion Webhook').item.json.body.merge_request.title`
    - `...description`
    - `...project.id`
    - `...merge_request.iid`
    - `...object_attributes.discussion_id`
  - merge request changes:
    - `$('Fetch Merge Request Changes').item.json.diff_refs.*`
- **Input and output connections:**  
  - Input: `Filter Supported Diffs`
  - Outputs in parallel to:
    - `Analyze Security Risks`
    - `Analyze Bugs`
    - `Analyze Maintainability Risks`
- **Version-specific requirements:** Type version `2`
- **Edge cases / failures:**  
  - Large diffs may produce large prompts
  - Non-standard patch text may cause incomplete line mapping
  - If `new_path` and `old_path` are both absent, path becomes empty
  - Incorrect line mapping can lead to failed inline comment resolution later
- **Sub-workflow reference:** none

---

## 2.5 Parallel AI Review

### Overview
This block sends the prepared file context through three independent reviewer agents. Each reviewer specializes in one class of concern and returns structured JSON findings parsed by an output parser.

### Nodes Involved
- Analyze Bugs
- Bug Reviewer Model
- Parse Bug Review Output
- Analyze Security Risks
- Security Reviewer Model
- Parse Security Review Output
- Analyze Maintainability Risks
- Maintainability Reviewer Model
- Parse Maintainability Review Output

### Node Details

#### Analyze Bugs
- **Type and role:** `@n8n/n8n-nodes-langchain.agent`; LLM-based bug reviewer
- **Configuration choices:**  
  - Prompt type: defined in node
  - Input text: stringified current JSON payload
  - `hasOutputParser = true`
  - `onError = continueRegularOutput`
  - retries enabled
  - always outputs data
- **Key expressions or variables used:**  
  - `={{ JSON.stringify($json, null, 2) }}`
  - System message instructs the model to produce only bug findings and return strict JSON
- **Input and output connections:**  
  - Main input: `Prepare Review Context`
  - AI language model input: `Bug Reviewer Model`
  - AI output parser input: `Parse Bug Review Output`
  - Main output: `Merge Reviewer Results`
- **Version-specific requirements:** Type version `3.1`; requires LangChain-compatible n8n version
- **Edge cases / failures:**  
  - Model may still produce malformed JSON despite parser
  - False positives if prompt grounding is weak
  - Large diffs may exceed token budget
- **Sub-workflow reference:** none

#### Bug Reviewer Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatAzureOpenAi`; Azure OpenAI chat model for bug review
- **Configuration choices:**  
  - Model: `gpt-5.4-mini`
  - `topP = 1`
- **Key expressions or variables used:** none
- **Input and output connections:**  
  - Output to `Analyze Bugs` as AI language model
- **Version-specific requirements:** Type version `1`; requires Azure OpenAI credentials configured in n8n
- **Edge cases / failures:**  
  - Bad Azure credentials
  - Unsupported deployment/model name
  - Rate limiting or API quota exhaustion
- **Sub-workflow reference:** none

#### Parse Bug Review Output
- **Type and role:** `@n8n/n8n-nodes-langchain.outputParserStructured`; validates bug-review JSON shape
- **Configuration choices:**  
  - JSON schema example expects:
    - `reviewer`
    - `findings[]` with title, severity, confidence, path, line, category, why, suggestion
- **Key expressions or variables used:** none
- **Input and output connections:**  
  - Output to `Analyze Bugs` as AI output parser
- **Version-specific requirements:** Type version `1.3`
- **Edge cases / failures:**  
  - Model output that deviates from expected structure may fail parsing
- **Sub-workflow reference:** none

#### Analyze Security Risks
- **Type and role:** `@n8n/n8n-nodes-langchain.agent`; LLM-based security reviewer
- **Configuration choices:**  
  - Same structural pattern as bug reviewer
  - Specialized security-oriented system prompt
  - `hasOutputParser = true`
  - retries and always-output enabled
- **Key expressions or variables used:**  
  - `={{ JSON.stringify($json, null, 2) }}`
- **Input and output connections:**  
  - Main input: `Prepare Review Context`
  - AI language model input: `Security Reviewer Model`
  - AI output parser input: `Parse Security Review Output`
  - Main output: `Merge Reviewer Results`
- **Version-specific requirements:** Type version `3.1`
- **Edge cases / failures:**  
  - Security over-reporting
  - Parser mismatch
  - Azure OpenAI latency/rate limits
- **Sub-workflow reference:** none

#### Security Reviewer Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatAzureOpenAi`; Azure model for security review
- **Configuration choices:**  
  - Model: `gpt-5.4-mini`
  - `topP = 1`
- **Key expressions or variables used:** none
- **Input and output connections:**  
  - Output to `Analyze Security Risks`
- **Version-specific requirements:** Type version `1`
- **Edge cases / failures:** same as other Azure model nodes
- **Sub-workflow reference:** none

#### Parse Security Review Output
- **Type and role:** `@n8n/n8n-nodes-langchain.outputParserStructured`; structured parser for security findings
- **Configuration choices:**  
  - Schema example expects `reviewer = security` and `findings[]`
- **Key expressions or variables used:** none
- **Input and output connections:**  
  - Output to `Analyze Security Risks`
- **Version-specific requirements:** Type version `1.3`
- **Edge cases / failures:** malformed model output
- **Sub-workflow reference:** none

#### Analyze Maintainability Risks
- **Type and role:** `@n8n/n8n-nodes-langchain.agent`; LLM-based maintainability reviewer
- **Configuration choices:**  
  - Same pattern as the other reviewers
  - Restricts severity to medium/low
  - `hasOutputParser = true`
  - retries and always-output enabled
- **Key expressions or variables used:**  
  - `={{ JSON.stringify($json, null, 2) }}`
- **Input and output connections:**  
  - Main input: `Prepare Review Context`
  - AI language model input: `Maintainability Reviewer Model`
  - AI output parser input: `Parse Maintainability Review Output`
  - Main output: `Merge Reviewer Results`
- **Version-specific requirements:** Type version `3.1`
- **Edge cases / failures:**  
  - Subjective maintainability findings may vary across runs
  - Parser or token-limit issues
- **Sub-workflow reference:** none

#### Maintainability Reviewer Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatAzureOpenAi`; Azure model for maintainability review
- **Configuration choices:**  
  - Model: `gpt-5.4-mini`
  - `topP = 1`
- **Key expressions or variables used:** none
- **Input and output connections:**  
  - Output to `Analyze Maintainability Risks`
- **Version-specific requirements:** Type version `1`
- **Edge cases / failures:** auth, quota, deployment-name mismatch
- **Sub-workflow reference:** none

#### Parse Maintainability Review Output
- **Type and role:** `@n8n/n8n-nodes-langchain.outputParserStructured`; structured parser for maintainability findings
- **Configuration choices:**  
  - Schema example expects `reviewer = maintainability` and `findings[]`
- **Key expressions or variables used:** none
- **Input and output connections:**  
  - Output to `Analyze Maintainability Risks`
- **Version-specific requirements:** Type version `1.3`
- **Edge cases / failures:** malformed or incomplete JSON
- **Sub-workflow reference:** none

---

## 2.6 Reviewer Result Merge

### Overview
This block merges the outputs from the three specialized reviewers and performs a first-pass deduplication. It also reattaches all file and MR context to the combined findings payload.

### Nodes Involved
- Merge Reviewer Results
- Combine Findings

### Node Details

#### Merge Reviewer Results
- **Type and role:** `n8n-nodes-base.merge`; merges three parallel reviewer streams
- **Configuration choices:**  
  - `numberInputs = 3`
- **Key expressions or variables used:** none
- **Input and output connections:**  
  - Input 0: `Analyze Bugs`
  - Input 1: `Analyze Security Risks`
  - Input 2: `Analyze Maintainability Risks`
  - Output: `Combine Findings`
- **Version-specific requirements:** Type version `3.2`
- **Edge cases / failures:**  
  - If one reviewer outputs empty or malformed content, combined results may still proceed because agents are configured to continue output
- **Sub-workflow reference:** none

#### Combine Findings
- **Type and role:** `n8n-nodes-base.code`; consolidates reviewer findings into one array and removes obvious duplicates
- **Configuration choices:**  
  - Collects all input items
  - Reads from `item.json.output` if present, otherwise `item.json`
  - Concatenates all `findings`
  - De-duplicates using key `${title}_${path}_${line}`
  - Rebuilds a single JSON object with original review context from `Prepare Review Context`
- **Key expressions or variables used:**  
  - `$input.all()`
  - `$('Prepare Review Context').first().json.*`
- **Input and output connections:**  
  - Input: `Merge Reviewer Results`
  - Output: `Verify Findings`
- **Version-specific requirements:** Type version `2`
- **Edge cases / failures:**  
  - Deduplication is simplistic; semantically duplicate findings with different titles survive
  - Uses `.first()` from `Prepare Review Context`, which assumes context alignment per current execution item; if workflow shape changes, this could become unsafe
- **Sub-workflow reference:** none

---

## 2.7 Verification and Normalization

### Overview
This block rechecks all merged findings against the diff and produces final publication-ready decisions. It is the main quality-control stage for reducing hallucinations, duplicates, and weak claims.

### Nodes Involved
- Verify Findings
- Verifier Model
- Parse Verification Output
- Normalize Verified Findings
- Check Findings Exist

### Node Details

#### Verify Findings
- **Type and role:** `@n8n/n8n-nodes-langchain.agent`; verification agent
- **Configuration choices:**  
  - Receives merged findings plus full file context
  - Strict instructions for:
    - grounding in diff
    - line selection
    - severity normalization
    - confidence normalization to `{0,25,50,75,100}`
    - status normalization to `{keep, drop, duplicate, pre_existing}`
    - inline posting decision
  - `hasOutputParser = true`
  - retries enabled
- **Key expressions or variables used:**  
  - `={{ JSON.stringify($json, null, 2) }}`
  - System prompt includes `{{ JSON.stringify($json.findings, null, 2) }}`
- **Input and output connections:**  
  - Main input: `Combine Findings`
  - AI language model input: `Verifier Model`
  - AI output parser input: `Parse Verification Output`
  - Main output: `Normalize Verified Findings`
- **Version-specific requirements:** Type version `3.1`
- **Edge cases / failures:**  
  - Verifier may still produce inconsistent line numbers
  - Strong prompt dependence for reliable filtering
  - Large finding sets increase latency and token usage
- **Sub-workflow reference:** none

#### Verifier Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatAzureOpenAi`; Azure model used by verifier
- **Configuration choices:**  
  - Model: `gpt-5.4-mini`
  - `topP = 1`
- **Key expressions or variables used:** none
- **Input and output connections:**  
  - Output to `Verify Findings`
- **Version-specific requirements:** Type version `1`
- **Edge cases / failures:** Azure auth, rate limit, deployment issues
- **Sub-workflow reference:** none

#### Parse Verification Output
- **Type and role:** `@n8n/n8n-nodes-langchain.outputParserStructured`; parser for final verified findings schema
- **Configuration choices:**  
  - Expects `validated_findings[]` with fields:
    - `title`
    - `path`
    - `line`
    - `final_severity`
    - `final_confidence`
    - `status`
    - `post_inline`
    - `comment`
- **Key expressions or variables used:** none
- **Input and output connections:**  
  - Output to `Verify Findings`
- **Version-specific requirements:** Type version `1.3`
- **Edge cases / failures:** parser failure on invalid JSON or missing fields
- **Sub-workflow reference:** none

#### Normalize Verified Findings
- **Type and role:** `n8n-nodes-base.code`; converts verifier line references to actual GitLab new-line numbers and reattaches metadata
- **Configuration choices:**  
  - Reads verifier output from `$json.output ?? $json`
  - Uses `newCodeLineMap` from `Combine Findings`
  - Converts verifier logical line values into actual diff line numbers
  - Forces `post_inline = true` only if:
    - status is `keep`
    - resolved line is non-null
    - verifier requested inline posting
  - Adds MR/project/discussion/SHA metadata to each finding
  - Calculates `keep_count`
- **Key expressions or variables used:**  
  - `$('Combine Findings').first().json`
- **Input and output connections:**  
  - Input: `Verify Findings`
  - Output: `Check Findings Exist`
- **Version-specific requirements:** Type version `2`
- **Edge cases / failures:**  
  - If verifier returns invalid `line`, it becomes `null`
  - If `newCodeLineMap` is incomplete, inline posting may be disabled
- **Sub-workflow reference:** none

#### Check Findings Exist
- **Type and role:** `n8n-nodes-base.if`; branches based on whether any kept findings remain
- **Configuration choices:**  
  - Condition: `keep_count >= 1`
- **Key expressions or variables used:**  
  - `={{$json.keep_count}}`
- **Input and output connections:**  
  - Input: `Normalize Verified Findings`
  - True output:
    - `Build Summary Comment`
    - `Split Verified Findings`
  - False output:
    - `Build No-Issues Comment`
- **Version-specific requirements:** Type version `2.3`
- **Edge cases / failures:**  
  - If normalization fails and `keep_count` is absent, branching may behave unexpectedly
- **Sub-workflow reference:** none

---

## 2.8 Publishable Finding Filtering and Inline Position Resolution

### Overview
This block selects only findings that meet the configured publication threshold, then tries to compute valid GitLab diff coordinates for inline comment posting.

### Nodes Involved
- Split Verified Findings
- Filter Publishable Findings
- Resolve GitLab Diff Position
- Build Inline Comment
- Check Inline Position

### Node Details

#### Split Verified Findings
- **Type and role:** `n8n-nodes-base.splitOut`; expands `validated_findings` into one finding per item
- **Configuration choices:**  
  - `fieldToSplitOut = validated_findings`
- **Key expressions or variables used:** none
- **Input and output connections:**  
  - Input: `Check Findings Exist` true branch
  - Output: `Filter Publishable Findings`
- **Version-specific requirements:** Type version `1`
- **Edge cases / failures:** empty array results in no comment posting
- **Sub-workflow reference:** none

#### Filter Publishable Findings
- **Type and role:** `n8n-nodes-base.if`; publication gate
- **Configuration choices:**  
  - Requires:
    - `status = keep`
    - `final_confidence >= minConfidenceToPost`
- **Key expressions or variables used:**  
  - `={{$json.status}}`
  - `={{$json.final_confidence}}`
  - `={{ $('Workflow Configuration').first().json.minConfidenceToPost }}`
- **Input and output connections:**  
  - Input: `Split Verified Findings`
  - True output: `Resolve GitLab Diff Position`
  - False output: none
- **Version-specific requirements:** Type version `2.3`
- **Edge cases / failures:**  
  - High threshold may suppress borderline but useful findings
  - Non-standard confidence value could interact poorly if verifier prompt is changed
- **Sub-workflow reference:** none

#### Resolve GitLab Diff Position
- **Type and role:** `n8n-nodes-base.code`; maps final line number to GitLab discussion position fields
- **Configuration choices:**  
  - Re-parses the original git diff
  - Builds a map of new-side positions:
    - added lines -> `newLine`, `oldLine = null`
    - context lines -> both `newLine` and `oldLine`
  - Adds:
    - `positionKind`
    - `positionNewLine`
    - `positionOldLine`
    - `canPostInline`
- **Key expressions or variables used:**  
  - `$('Combine Findings').first().json.gitDiff`
  - `Number($json.line)`
- **Input and output connections:**  
  - Input: `Filter Publishable Findings`
  - Output: `Build Inline Comment`
- **Version-specific requirements:** Type version `2`
- **Edge cases / failures:**  
  - If target line cannot be found in diff map, inline posting becomes impossible
  - Complex diffs may still produce mismatches with GitLab’s accepted position model
- **Sub-workflow reference:** none

#### Build Inline Comment
- **Type and role:** `n8n-nodes-base.code`; formats the final GitLab comment text with severity icon
- **Configuration choices:**  
  - Runs once per item
  - Builds:
    - `🔴` for high
    - `🟡` for medium
    - `🔵` for low
  - Final text format:
    - bold title
    - blank line
    - comment body
- **Key expressions or variables used:**  
  - `$json.final_severity`
  - `$json.title`
  - `$json.comment`
- **Input and output connections:**  
  - Input: `Resolve GitLab Diff Position`
  - Output: `Check Inline Position`
- **Version-specific requirements:** Type version `2`
- **Edge cases / failures:**  
  - Missing title/comment causes low-quality published comment
- **Sub-workflow reference:** none

#### Check Inline Position
- **Type and role:** `n8n-nodes-base.if`; chooses between inline posting and fallback reply posting
- **Configuration choices:**  
  - Condition: `canPostInline = true`
- **Key expressions or variables used:**  
  - `={{$json.canPostInline}}`
- **Input and output connections:**  
  - Input: `Build Inline Comment`
  - True output: `Post Inline Review Comment`
  - False output: `Build Fallback Comment`
- **Version-specific requirements:** Type version `2.3`
- **Edge cases / failures:**  
  - A technically mapped position may still be rejected by GitLab if diff refs are stale
- **Sub-workflow reference:** none

---

## 2.9 Comment Publication

### Overview
This block posts either inline MR review comments or fallback general replies in the original trigger discussion. It ensures that findings are still published even when an exact inline position cannot be resolved.

### Nodes Involved
- Post Inline Review Comment
- Build Fallback Comment
- Post Fallback Reply

### Node Details

#### Post Inline Review Comment
- **Type and role:** `n8n-nodes-base.httpRequest`; creates a new inline MR discussion
- **Configuration choices:**  
  - Method: `POST`
  - Auth: predefined `gitlabApi`
  - Endpoint: `/projects/{projectId}/merge_requests/{mrIid}/discussions`
  - Sends multipart form-data including:
    - `body`
    - `position[position_type] = text`
    - `position[old_path]`
    - `position[new_path]`
    - `position[start_sha]`
    - `position[head_sha]`
    - `position[base_sha]`
    - `position[new_line]`
    - `position[old_line]`
- **Key expressions or variables used:**  
  - values from current normalized finding item
  - base URL from configuration
- **Input and output connections:**  
  - Input: `Check Inline Position` true branch
  - Output: none
- **Version-specific requirements:** Type version `4.2`
- **Edge cases / failures:**  
  - GitLab rejects invalid or stale diff positions
  - Missing SHA values prevent inline discussion creation
  - Path mismatch against GitLab diff representation
- **Sub-workflow reference:** none

#### Build Fallback Comment
- **Type and role:** `n8n-nodes-base.code`; formats a non-inline discussion note when exact diff placement is unavailable
- **Configuration choices:**  
  - Runs once per item
  - Builds a message containing:
    - severity icon
    - “General review comment” label
    - finding title
    - comment text
    - file path
- **Key expressions or variables used:**  
  - `$json.final_severity`
  - `$json.title`
  - `$json.comment`
  - `$json.path`, `$json.newPath`, `$json.oldPath`
- **Input and output connections:**  
  - Input: `Check Inline Position` false branch
  - Output: `Post Fallback Reply`
- **Version-specific requirements:** Type version `2`
- **Edge cases / failures:**  
  - Missing path fields yield `unknown`
- **Sub-workflow reference:** none

#### Post Fallback Reply
- **Type and role:** `n8n-nodes-base.httpRequest`; posts fallback note into the original trigger discussion
- **Configuration choices:**  
  - Method: `POST`
  - Auth: predefined `gitlabApi`
  - Endpoint: discussion notes endpoint
  - Body field is `fallbackBody`
- **Key expressions or variables used:**  
  - `projectId`, `mrIid`, `discussionId`
  - `fallbackBody`
  - base URL from configuration
- **Input and output connections:**  
  - Input: `Build Fallback Comment`
  - Output: none
- **Version-specific requirements:** Type version `4.2`
- **Edge cases / failures:**  
  - If original discussion was deleted or inaccessible, posting fails
- **Sub-workflow reference:** none

---

## 2.10 Summary Reply

### Overview
This block always posts a final status reply to the original discussion. It either says the review is complete or explicitly says no issues were found.

### Nodes Involved
- Build Summary Comment
- Build No-Issues Comment
- Post Summary Reply

### Node Details

#### Build Summary Comment
- **Type and role:** `n8n-nodes-base.code`; prepares the normal completion reply
- **Configuration choices:**  
  - Returns `summaryMessage` if configured, else default fallback
- **Key expressions or variables used:**  
  - `$('Workflow Configuration').first().json.summaryMessage`
- **Input and output connections:**  
  - Input: `Check Findings Exist` true branch
  - Output: `Post Summary Reply`
- **Version-specific requirements:** Type version `2`
- **Edge cases / failures:** minimal
- **Sub-workflow reference:** none

#### Build No-Issues Comment
- **Type and role:** `n8n-nodes-base.code`; prepares the no-issues reply
- **Configuration choices:**  
  - Returns configured `noIssuesMessage`
- **Key expressions or variables used:**  
  - `$('Workflow Configuration').first().json.noIssuesMessage`
- **Input and output connections:**  
  - Input: `Check Findings Exist` false branch
  - Output: `Post Summary Reply`
- **Version-specific requirements:** Type version `2`
- **Edge cases / failures:** if message blank, an empty reply may be posted
- **Sub-workflow reference:** none

#### Post Summary Reply
- **Type and role:** `n8n-nodes-base.httpRequest`; posts final summary note to trigger discussion
- **Configuration choices:**  
  - Method: `POST`
  - Auth: predefined `gitlabApi`
  - Endpoint: discussion notes endpoint
  - Body set to generated `text`
- **Key expressions or variables used:**  
  - base URL from configuration
  - project/MR/discussion IDs from `Combine Findings`
  - `{{$json.text}}`
- **Input and output connections:**  
  - Inputs:
    - `Build Summary Comment`
    - `Build No-Issues Comment`
  - Output: none
- **Version-specific requirements:** Type version `4.2`
- **Edge cases / failures:**  
  - If no supported files ever reached `Combine Findings`, this node may not execute because it depends on metadata from that path
  - GitLab permissions/auth issues
- **Sub-workflow reference:** none

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| GitLab Discussion Webhook | n8n-nodes-base.webhook | Receives GitLab discussion webhook event |  | Workflow Configuration | ## Trigger the review\nPost the configured trigger comment in a GitLab merge request discussion to start the workflow.\nDefault trigger: `+0`\nChange it in **Workflow Configuration**. |
| Workflow Configuration | n8n-nodes-base.set | Stores environment-specific settings and reusable messages | GitLab Discussion Webhook | Check Review Trigger Comment | ## Trigger the review\nPost the configured trigger comment in a GitLab merge request discussion to start the workflow.\nDefault trigger: `+0`\nChange it in **Workflow Configuration**. |
| Check Review Trigger Comment | n8n-nodes-base.if | Validates that the posted comment matches the configured trigger phrase | Workflow Configuration | Fetch Merge Request Changes; Post Review Started Reply | ## Trigger the review\nPost the configured trigger comment in a GitLab merge request discussion to start the workflow.\nDefault trigger: `+0`\nChange it in **Workflow Configuration**. |
| Post Review Started Reply | n8n-nodes-base.httpRequest | Posts an acknowledgment reply to the triggering discussion | Check Review Trigger Comment |  | ## Fetch and filter changed files\nLoad the merge request changes, split them into one file per item, and skip unsupported diffs such as renamed, deleted, or non-commentable files. |
| Fetch Merge Request Changes | n8n-nodes-base.httpRequest | Retrieves MR changes and diff refs from GitLab | Check Review Trigger Comment | Split Changed Files | ## Fetch and filter changed files\nLoad the merge request changes, split them into one file per item, and skip unsupported diffs such as renamed, deleted, or non-commentable files. |
| Split Changed Files | n8n-nodes-base.splitOut | Splits the changes array into one item per changed file | Fetch Merge Request Changes | Filter Supported Diffs | ## Fetch and filter changed files\nLoad the merge request changes, split them into one file per item, and skip unsupported diffs such as renamed, deleted, or non-commentable files. |
| Filter Supported Diffs | n8n-nodes-base.if | Keeps only commentable, non-renamed, non-deleted diffs with hunk markers | Split Changed Files | Prepare Review Context | ## Fetch and filter changed files\nLoad the merge request changes, split them into one file per item, and skip unsupported diffs such as renamed, deleted, or non-commentable files. |
| Prepare Review Context | n8n-nodes-base.code | Builds normalized review payload and line mapping from each diff | Filter Supported Diffs | Analyze Security Risks; Analyze Bugs; Analyze Maintainability Risks | ## Prepare review context\nConvert each diff into a clean review payload with file paths, code snippets, merge request metadata, and diff SHAs needed later for GitLab comments. |
| Analyze Bugs | @n8n/n8n-nodes-langchain.agent | AI agent reviewing newly introduced functional or logical bugs | Prepare Review Context; Bug Reviewer Model; Parse Bug Review Output | Merge Reviewer Results | ## Run parallel AI reviewers\nAnalyze each changed file with bug, security, and maintainability reviewers.\nEach reviewer returns structured findings for final verification. |
| Bug Reviewer Model | @n8n/n8n-nodes-langchain.lmChatAzureOpenAi | Azure OpenAI chat model for bug reviewer |  | Analyze Bugs | ## Run parallel AI reviewers\nAnalyze each changed file with bug, security, and maintainability reviewers.\nEach reviewer returns structured findings for final verification. |
| Parse Bug Review Output | @n8n/n8n-nodes-langchain.outputParserStructured | Structured parser enforcing bug reviewer JSON output |  | Analyze Bugs | ## Run parallel AI reviewers\nAnalyze each changed file with bug, security, and maintainability reviewers.\nEach reviewer returns structured findings for final verification. |
| Analyze Security Risks | @n8n/n8n-nodes-langchain.agent | AI agent reviewing newly introduced security risks | Prepare Review Context; Security Reviewer Model; Parse Security Review Output | Merge Reviewer Results | ## Run parallel AI reviewers\nAnalyze each changed file with bug, security, and maintainability reviewers.\nEach reviewer returns structured findings for final verification. |
| Security Reviewer Model | @n8n/n8n-nodes-langchain.lmChatAzureOpenAi | Azure OpenAI chat model for security reviewer |  | Analyze Security Risks | ## Run parallel AI reviewers\nAnalyze each changed file with bug, security, and maintainability reviewers.\nEach reviewer returns structured findings for final verification. |
| Parse Security Review Output | @n8n/n8n-nodes-langchain.outputParserStructured | Structured parser enforcing security reviewer JSON output |  | Analyze Security Risks | ## Run parallel AI reviewers\nAnalyze each changed file with bug, security, and maintainability reviewers.\nEach reviewer returns structured findings for final verification. |
| Analyze Maintainability Risks | @n8n/n8n-nodes-langchain.agent | AI agent reviewing maintainability risks introduced by the diff | Prepare Review Context; Maintainability Reviewer Model; Parse Maintainability Review Output | Merge Reviewer Results | ## Run parallel AI reviewers\nAnalyze each changed file with bug, security, and maintainability reviewers.\nEach reviewer returns structured findings for final verification. |
| Maintainability Reviewer Model | @n8n/n8n-nodes-langchain.lmChatAzureOpenAi | Azure OpenAI chat model for maintainability reviewer |  | Analyze Maintainability Risks | ## Run parallel AI reviewers\nAnalyze each changed file with bug, security, and maintainability reviewers.\nEach reviewer returns structured findings for final verification. |
| Parse Maintainability Review Output | @n8n/n8n-nodes-langchain.outputParserStructured | Structured parser enforcing maintainability reviewer JSON output |  | Analyze Maintainability Risks | ## Run parallel AI reviewers\nAnalyze each changed file with bug, security, and maintainability reviewers.\nEach reviewer returns structured findings for final verification. |
| Merge Reviewer Results | n8n-nodes-base.merge | Collects outputs from the three reviewer branches | Analyze Bugs; Analyze Security Risks; Analyze Maintainability Risks | Combine Findings | ## Merge reviewer outputs\nCombine findings from all reviewers, remove obvious duplicates, and keep the file and merge request context for final verification. |
| Combine Findings | n8n-nodes-base.code | Merges and deduplicates reviewer findings while preserving file context | Merge Reviewer Results | Verify Findings | ## Merge reviewer outputs\nCombine findings from all reviewers, remove obvious duplicates, and keep the file and merge request context for final verification. |
| Verify Findings | @n8n/n8n-nodes-langchain.agent | Re-evaluates and normalizes findings before publication | Combine Findings; Verifier Model; Parse Verification Output | Normalize Verified Findings | ## Verify and normalize findings\nRe-check each finding against the diff, remove weak or duplicate issues, and normalize the final severity and confidence. |
| Verifier Model | @n8n/n8n-nodes-langchain.lmChatAzureOpenAi | Azure OpenAI chat model for final finding verification |  | Verify Findings | ## Verify and normalize findings\nRe-check each finding against the diff, remove weak or duplicate issues, and normalize the final severity and confidence. |
| Parse Verification Output | @n8n/n8n-nodes-langchain.outputParserStructured | Structured parser enforcing verifier output schema |  | Verify Findings | ## Verify and normalize findings\nRe-check each finding against the diff, remove weak or duplicate issues, and normalize the final severity and confidence. |
| Normalize Verified Findings | n8n-nodes-base.code | Maps logical lines to real diff lines and enriches findings with posting metadata | Verify Findings | Check Findings Exist | ## Verify and normalize findings\nRe-check each finding against the diff, remove weak or duplicate issues, and normalize the final severity and confidence. |
| Check Findings Exist | n8n-nodes-base.if | Branches between summary/no-issues flow and detailed comment publication | Normalize Verified Findings | Build Summary Comment; Split Verified Findings; Build No-Issues Comment | ## Decide what to publish\nPost only findings with `status = keep` and confidence at or above the configured threshold.\nIf a valid diff position is found, post an inline comment.\nOtherwise, post a reply comment in the trigger discussion. |
| Split Verified Findings | n8n-nodes-base.splitOut | Splits verified findings into one item per finding | Check Findings Exist | Filter Publishable Findings | ## Decide what to publish\nPost only findings with `status = keep` and confidence at or above the configured threshold.\nIf a valid diff position is found, post an inline comment.\nOtherwise, post a reply comment in the trigger discussion. |
| Filter Publishable Findings | n8n-nodes-base.if | Keeps only findings eligible for publication based on status and confidence | Split Verified Findings | Resolve GitLab Diff Position | ## Decide what to publish\nPost only findings with `status = keep` and confidence at or above the configured threshold.\nIf a valid diff position is found, post an inline comment.\nOtherwise, post a reply comment in the trigger discussion. |
| Resolve GitLab Diff Position | n8n-nodes-base.code | Resolves GitLab inline diff coordinates from verified line numbers | Filter Publishable Findings | Build Inline Comment | ## Publish review comments\nResolve the GitLab diff position for each verified finding and post the final review comment.\nIf a valid inline position is available, create an inline diff discussion.\nOtherwise, post a reply comment in the trigger discussion. |
| Build Inline Comment | n8n-nodes-base.code | Formats final review comment text with severity icon | Resolve GitLab Diff Position | Check Inline Position | ## Publish review comments\nResolve the GitLab diff position for each verified finding and post the final review comment.\nIf a valid inline position is available, create an inline diff discussion.\nOtherwise, post a reply comment in the trigger discussion. |
| Check Inline Position | n8n-nodes-base.if | Chooses inline posting versus fallback discussion reply | Build Inline Comment | Post Inline Review Comment; Build Fallback Comment | ## Publish review comments\nResolve the GitLab diff position for each verified finding and post the final review comment.\nIf a valid inline position is available, create an inline diff discussion.\nOtherwise, post a reply comment in the trigger discussion. |
| Post Inline Review Comment | n8n-nodes-base.httpRequest | Posts an inline MR discussion comment in GitLab | Check Inline Position |  | ## Publish review comments\nResolve the GitLab diff position for each verified finding and post the final review comment.\nIf a valid inline position is available, create an inline diff discussion.\nOtherwise, post a reply comment in the trigger discussion. |
| Build Fallback Comment | n8n-nodes-base.code | Builds a general discussion comment when inline position cannot be resolved | Check Inline Position | Post Fallback Reply | ## Publish review comments\nResolve the GitLab diff position for each verified finding and post the final review comment.\nIf a valid inline position is available, create an inline diff discussion.\nOtherwise, post a reply comment in the trigger discussion. |
| Post Fallback Reply | n8n-nodes-base.httpRequest | Posts non-inline finding to the original trigger discussion | Build Fallback Comment |  | ## Publish review comments\nResolve the GitLab diff position for each verified finding and post the final review comment.\nIf a valid inline position is available, create an inline diff discussion.\nOtherwise, post a reply comment in the trigger discussion. |
| Build Summary Comment | n8n-nodes-base.code | Creates the standard completion reply text | Check Findings Exist | Post Summary Reply | ## Post a summary reply\nAlways reply to the trigger discussion with either a completion message or a no-issues message.\nInline and fallback review comments are posted separately. |
| Build No-Issues Comment | n8n-nodes-base.code | Creates the no-issues reply text | Check Findings Exist | Post Summary Reply | ## Post a summary reply\nAlways reply to the trigger discussion with either a completion message or a no-issues message.\nInline and fallback review comments are posted separately. |
| Post Summary Reply | n8n-nodes-base.httpRequest | Posts final summary note to the original discussion | Build Summary Comment; Build No-Issues Comment |  | ## Post a summary reply\nAlways reply to the trigger discussion with either a completion message or a no-issues message.\nInline and fallback review comments are posted separately. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **Review GitLab merge requests with parallel AI reviewers**.
   - Keep it inactive until fully tested.

2. **Add a Webhook node**
   - Type: **Webhook**
   - Name: **GitLab Discussion Webhook**
   - Method: `POST`
   - Path: choose any fixed path, for example a UUID-like value
   - This will be the endpoint GitLab calls for discussion events.

3. **Configure GitLab to send discussion webhooks**
   - In GitLab, create a webhook pointing to the n8n webhook URL.
   - Ensure discussion/note-related MR events are included.
   - Confirm the payload contains:
     - `body.object_attributes.note`
     - `body.object_attributes.discussion_id`
     - `body.project.id`
     - `body.project_id`
     - `body.merge_request.iid`
     - `body.merge_request.title`
     - `body.merge_request.description`

4. **Add a Set node**
   - Type: **Set**
   - Name: **Workflow Configuration**
   - Connect: `GitLab Discussion Webhook -> Workflow Configuration`
   - Enable “Include Other Fields”
   - Add fields:
     1. `gitlabBaseUrl` (string), e.g. `https://gitlab.example.com/api/v4`
     2. `reviewTriggerPhrase` (string), default `+0`
     3. `minConfidenceToPost` (number), default `75`
     4. `reviewStartedMessage` (string), default `🔍 I'm reviewing the code right now. Please hang tight for a moment.`
     5. `noIssuesMessage` (string), default `🟢 The changes look good this time!`
     6. `summaryMessage` (string), default `✅ Review complete!`

5. **Add an IF node to validate the trigger**
   - Type: **If**
   - Name: **Check Review Trigger Comment**
   - Connect: `Workflow Configuration -> Check Review Trigger Comment`
   - Condition:
     - Left value: `{{$json.body.object_attributes.note}}`
     - Operator: `equals`
     - Right value: `{{$('Workflow Configuration').first().json.reviewTriggerPhrase}}`
   - Use strict comparison.

6. **Create GitLab credentials**
   - Add an n8n credential of type **GitLab API**
   - It must have permission to:
     - read merge request changes
     - post notes/discussions on merge requests

7. **Add the “review started” HTTP Request node**
   - Type: **HTTP Request**
   - Name: **Post Review Started Reply**
   - Connect from the **true** output of `Check Review Trigger Comment`
   - Method: `POST`
   - Authentication: predefined credential type = **GitLab API**
   - Content type: `multipart-form-data`
   - URL:
     - `{{$('Workflow Configuration').first().json.gitlabBaseUrl}}/projects/{{$('GitLab Discussion Webhook').item.json.body.project.id}}/merge_requests/{{$('GitLab Discussion Webhook').item.json.body.merge_request.iid}}/discussions/{{$('GitLab Discussion Webhook').item.json.body.object_attributes.discussion_id}}/notes`
   - Body parameter:
     - `body = {{$('Workflow Configuration').first().json.reviewStartedMessage}}`

8. **Add the MR changes fetch HTTP Request node**
   - Type: **HTTP Request**
   - Name: **Fetch Merge Request Changes**
   - Connect from the **true** output of `Check Review Trigger Comment`
   - Method: `GET`
   - Authentication: predefined credential type = **GitLab API**
   - URL:
     - `{{$('Workflow Configuration').first().json.gitlabBaseUrl}}/projects/{{$json["body"]["project_id"]}}/merge_requests/{{$json["body"]["merge_request"]["iid"]}}/changes`

9. **Split the changed files**
   - Type: **Split Out**
   - Name: **Split Changed Files**
   - Connect: `Fetch Merge Request Changes -> Split Changed Files`
   - Field to split out: `changes`

10. **Filter supported diffs**
    - Type: **If**
    - Name: **Filter Supported Diffs**
    - Connect: `Split Changed Files -> Filter Supported Diffs`
    - Add three AND conditions:
      1. `{{$json.renamed_file}}` is `false`
      2. `{{$json.deleted_file}}` is `false`
      3. `{{ /(^|\n)@@ -\d+(?:,\d+)? \+\d+(?:,\d+)? @@/m.test($json.diff ?? '') }}` is `true`
    - Only the true branch continues.

11. **Add the diff-preparation Code node**
    - Type: **Code**
    - Name: **Prepare Review Context**
    - Connect: `Filter Supported Diffs -> Prepare Review Context`
    - Mode: **Run once for each item**
    - Paste logic that:
      - parses the diff
      - extracts old/new code
      - builds `reviewableNewCode` with `[L<number>]` prefixes
      - builds `newCodeLineMap`
      - outputs:
        - `path`, `oldPath`, `newPath`
        - `gitDiff`
        - `originalCode`
        - `newCode`
        - `newCodeLineMap`
        - `reviewableNewCode`
        - `mrTitle`, `mrDescription`
        - `projectId`, `mrIid`, `discussionId`
        - `startSha`, `headSha`, `baseSha`
    - Use data from:
      - current diff item
      - `GitLab Discussion Webhook`
      - `Fetch Merge Request Changes.diff_refs`

12. **Create Azure OpenAI credentials**
    - Add an n8n credential of type **Azure OpenAI API**
    - Configure endpoint, API key, API version, and deployment/model availability.
    - Ensure the deployment name used by n8n supports chat completions.

13. **Add the Bug reviewer model node**
    - Type: **Azure OpenAI Chat Model**
    - Name: **Bug Reviewer Model**
    - Model: `gpt-5.4-mini`
    - `topP = 1`

14. **Add the Bug output parser**
    - Type: **Structured Output Parser**
    - Name: **Parse Bug Review Output**
    - Configure a schema example containing:
      - `reviewer: "bug"`
      - `findings[]` with `title`, `severity`, `confidence`, `path`, `line`, `category`, `why`, `suggestion`

15. **Add the Bug agent**
    - Type: **AI Agent**
    - Name: **Analyze Bugs**
    - Connect main input from `Prepare Review Context`
    - Connect AI model input from `Bug Reviewer Model`
    - Connect AI output parser input from `Parse Bug Review Output`
    - Set text input to:
      - `={{ JSON.stringify($json, null, 2) }}`
    - Enable structured output parser
    - Configure retry on fail
    - Set `onError` to continue regular output
    - Use a system prompt that:
      - focuses only on newly introduced functional/logical bugs
      - forbids style and maintainability-only issues
      - requires JSON-only response
      - requires line selection from `reviewableNewCode`

16. **Add the Security reviewer model**
    - Type: **Azure OpenAI Chat Model**
    - Name: **Security Reviewer Model**
    - Model: `gpt-5.4-mini`
    - `topP = 1`

17. **Add the Security output parser**
    - Type: **Structured Output Parser**
    - Name: **Parse Security Review Output**
    - Configure schema example with:
      - `reviewer: "security"`
      - `findings[]` in the same shape as the bug reviewer

18. **Add the Security agent**
    - Type: **AI Agent**
    - Name: **Analyze Security Risks**
    - Connect main input from `Prepare Review Context`
    - Connect AI model input from `Security Reviewer Model`
    - Connect parser input from `Parse Security Review Output`
    - Input text:
      - `={{ JSON.stringify($json, null, 2) }}`
    - System prompt should focus only on real newly introduced security risks.

19. **Add the Maintainability reviewer model**
    - Type: **Azure OpenAI Chat Model**
    - Name: **Maintainability Reviewer Model**
    - Model: `gpt-5.4-mini`
    - `topP = 1`

20. **Add the Maintainability output parser**
    - Type: **Structured Output Parser**
    - Name: **Parse Maintainability Review Output**
    - Configure schema example with:
      - `reviewer: "maintainability"`
      - `findings[]`
      - severity constrained to medium/low in prompt expectations

21. **Add the Maintainability agent**
    - Type: **AI Agent**
    - Name: **Analyze Maintainability Risks**
    - Connect main input from `Prepare Review Context`
    - Connect model input from `Maintainability Reviewer Model`
    - Connect parser input from `Parse Maintainability Review Output`
    - Input text:
      - `={{ JSON.stringify($json, null, 2) }}`
    - System prompt should focus only on meaningful maintainability risks introduced by the change.

22. **Add a Merge node for reviewer outputs**
    - Type: **Merge**
    - Name: **Merge Reviewer Results**
    - Number of inputs: `3`
    - Connect:
      - `Analyze Bugs -> input 1`
      - `Analyze Security Risks -> input 2`
      - `Analyze Maintainability Risks -> input 3`

23. **Add the findings combination Code node**
    - Type: **Code**
    - Name: **Combine Findings**
    - Connect: `Merge Reviewer Results -> Combine Findings`
    - Logic should:
      - read all incoming reviewer outputs
      - extract `findings`
      - merge arrays
      - remove duplicates using a key like `title + path + line`
      - return a single item containing:
        - all original context from `Prepare Review Context`
        - merged `findings`

24. **Add the verifier model**
    - Type: **Azure OpenAI Chat Model**
    - Name: **Verifier Model**
    - Model: `gpt-5.4-mini`
    - `topP = 1`

25. **Add the verification output parser**
    - Type: **Structured Output Parser**
    - Name: **Parse Verification Output**
    - Schema example should include:
      - `validated_findings[]`
      - each finding has:
        - `title`
        - `path`
        - `line`
        - `final_severity`
        - `final_confidence`
        - `status`
        - `post_inline`
        - `comment`

26. **Add the verifier agent**
    - Type: **AI Agent**
    - Name: **Verify Findings**
    - Connect main input from `Combine Findings`
    - Connect AI model from `Verifier Model`
    - Connect parser from `Parse Verification Output`
    - Input text:
      - `={{ JSON.stringify($json, null, 2) }}`
    - System prompt must instruct the model to:
      - re-check grounding in the diff
      - remove speculative/duplicate findings
      - normalize severity to high/medium/low
      - normalize confidence to 0/25/50/75/100
      - output `status` in `keep/drop/duplicate/pre_existing`
      - decide `post_inline`
      - return JSON only

27. **Add normalization Code node**
    - Type: **Code**
    - Name: **Normalize Verified Findings**
    - Connect: `Verify Findings -> Normalize Verified Findings`
    - Logic should:
      - read verifier output
      - map verifier logical line numbers through `newCodeLineMap`
      - force line to `null` when unresolved
      - attach `oldPath`, `newPath`, `projectId`, `mrIid`, `discussionId`, `startSha`, `headSha`, `baseSha`
      - compute `keep_count`

28. **Add findings-existence IF node**
    - Type: **If**
    - Name: **Check Findings Exist**
    - Connect: `Normalize Verified Findings -> Check Findings Exist`
    - Condition:
      - `{{$json.keep_count}} >= 1`

29. **Add the summary comment Code node**
    - Type: **Code**
    - Name: **Build Summary Comment**
    - Connect from the true branch of `Check Findings Exist`
    - Return:
      - `text = summaryMessage` or a default completion string

30. **Add the no-issues comment Code node**
    - Type: **Code**
    - Name: **Build No-Issues Comment**
    - Connect from the false branch of `Check Findings Exist`
    - Return:
      - `text = noIssuesMessage`

31. **Add the summary posting HTTP Request node**
    - Type: **HTTP Request**
    - Name: **Post Summary Reply**
    - Connect both:
      - `Build Summary Comment -> Post Summary Reply`
      - `Build No-Issues Comment -> Post Summary Reply`
    - Method: `POST`
    - Auth: GitLab API credential
    - Content type: multipart form-data
    - URL:
      - `{{$('Workflow Configuration').first().json.gitlabBaseUrl}}/projects/{{$('Combine Findings').item.json.projectId}}/merge_requests/{{$('Combine Findings').item.json.mrIid}}/discussions/{{$('Combine Findings').item.json.discussionId}}/notes`
    - Body:
      - `body = {{$json.text}}`

32. **Add Split Out for verified findings**
    - Type: **Split Out**
    - Name: **Split Verified Findings**
    - Connect from the true branch of `Check Findings Exist`
    - Field to split out: `validated_findings`

33. **Add publish filter IF node**
    - Type: **If**
    - Name: **Filter Publishable Findings**
    - Connect: `Split Verified Findings -> Filter Publishable Findings`
    - Conditions:
      1. `{{$json.status}}` equals `keep`
      2. `{{$json.final_confidence}} >= {{$('Workflow Configuration').first().json.minConfidenceToPost}}`

34. **Add the diff-position resolver Code node**
    - Type: **Code**
    - Name: **Resolve GitLab Diff Position**
    - Connect: `Filter Publishable Findings -> Resolve GitLab Diff Position`
    - Logic should:
      - reparse the original git diff
      - map actual new-side lines to GitLab-compatible `new_line` and `old_line`
      - output:
        - `positionKind`
        - `positionNewLine`
        - `positionOldLine`
        - `canPostInline`

35. **Add the inline comment formatter**
    - Type: **Code**
    - Name: **Build Inline Comment**
    - Connect: `Resolve GitLab Diff Position -> Build Inline Comment`
    - Mode: run once per item
    - Build:
      - severity icon (`🔴`, `🟡`, `🔵`)
      - `text = {icon} **{title}**\n\n{comment}`

36. **Add inline/fallback IF node**
    - Type: **If**
    - Name: **Check Inline Position**
    - Connect: `Build Inline Comment -> Check Inline Position`
    - Condition:
      - `{{$json.canPostInline}}` is `true`

37. **Add inline posting HTTP Request**
    - Type: **HTTP Request**
    - Name: **Post Inline Review Comment**
    - Connect from the true branch of `Check Inline Position`
    - Method: `POST`
    - Auth: GitLab API credential
    - Content type: multipart form-data
    - URL:
      - `{{$('Workflow Configuration').first().json.gitlabBaseUrl}}/projects/{{$json.projectId}}/merge_requests/{{$json.mrIid}}/discussions`
    - Body params:
      - `body = {{$json.text}}`
      - `position[position_type] = text`
      - `position[old_path] = {{$json.oldPath}}`
      - `position[new_path] = {{$json.newPath}}`
      - `position[start_sha] = {{$json.startSha}}`
      - `position[head_sha] = {{$json.headSha}}`
      - `position[base_sha] = {{$json.baseSha}}`
      - `position[new_line] = {{$json.positionNewLine ?? ''}}`
      - `position[old_line] = {{$json.positionOldLine ?? ''}}`

38. **Add fallback formatter Code node**
    - Type: **Code**
    - Name: **Build Fallback Comment**
    - Connect from the false branch of `Check Inline Position`
    - Build a fallback body like:
      - severity icon
      - “General review comment (exact diff line could not be resolved)”
      - title
      - comment
      - file path

39. **Add fallback posting HTTP Request**
    - Type: **HTTP Request**
    - Name: **Post Fallback Reply**
    - Connect: `Build Fallback Comment -> Post Fallback Reply`
    - Method: `POST`
    - Auth: GitLab API credential
    - Content type: multipart form-data
    - URL:
      - `{{$('Workflow Configuration').first().json.gitlabBaseUrl}}/projects/{{$json.projectId}}/merge_requests/{{$json.mrIid}}/discussions/{{$json.discussionId}}/notes`
    - Body:
      - `body = {{$json.fallbackBody}}`

40. **Connect all parallel AI branches carefully**
    - `Prepare Review Context` must connect to all three reviewer agents.
    - Each agent must also be connected to its own model and output parser via the AI-specific connectors, not main connectors.

41. **Set retry and resilience options**
    - For all AI Agent nodes:
      - enable retries
      - keep `always output data`
      - set `onError` to continue output if you want the workflow to survive partial reviewer failures
    - This matches the original workflow’s fault-tolerance approach.

42. **Test with a sample merge request**
    - Post the trigger phrase in an MR discussion.
    - Confirm:
      - the “review started” reply appears
      - only supported diffs are analyzed
      - findings are verified and filtered
      - inline comments appear where positions are valid
      - fallback comments appear where inline resolution fails
      - final summary comment is always posted

43. **Adjust prompts and thresholds**
    - Tune `minConfidenceToPost` depending on desired precision/recall.
    - Review reviewer prompts and verifier prompt before production use.
    - Lower threshold increases recall but also false positives.
    - Higher threshold is stricter and may suppress useful borderline findings.

44. **Known implementation caution**
    - `Post Summary Reply` depends on metadata from `Combine Findings`.
    - If no supported files reach that stage, the summary path may not have enough context to post.
    - A robust enhancement is to preserve `projectId`, `mrIid`, and `discussionId` earlier in a globally available branch.

45. **No sub-workflows are required**
    - This workflow does not call any external n8n sub-workflow.
    - Everything is implemented in one workflow.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow runs an AI-assisted review for GitLab merge requests. When a reviewer posts the trigger comment in a merge request discussion, the workflow fetches the changed files, filters unsupported diffs, and prepares review context for each file. Each changed file is analyzed in parallel by three reviewers: Bug reviewer, Security reviewer, Maintainability reviewer. Their findings are merged and then checked by a verifier, which removes weak, duplicate, or pre-existing issues. Verified findings that pass the posting checks can be posted as inline GitLab comments. A separate summary reply is posted back to the original trigger discussion. | Workflow description note |
| Before using this workflow: connect your GitLab credential, connect your LLM credential, update values in Workflow Configuration for your environment, review prompts and model settings, and test on a sample merge request discussion before enabling it. | Setup guidance |
| The verifier assigns a confidence score to each finding. Only findings with `status=keep` and `final_confidence` at or above the configured threshold are posted. Findings with a resolved inline position are posted as inline comments; otherwise they are posted as reply comments. | Publication policy |
| Lower threshold values increase recall but may allow more false positives. Higher threshold values are stricter and may suppress borderline findings. | Threshold tuning guidance |