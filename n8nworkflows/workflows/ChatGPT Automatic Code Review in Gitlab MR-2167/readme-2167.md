ChatGPT Automatic Code Review in Gitlab MR

https://n8nworkflows.xyz/workflows/chatgpt-automatic-code-review-in-gitlab-mr-2167


# ChatGPT Automatic Code Review in Gitlab MR

### 1. Workflow Overview

This workflow automates code review on GitLab Merge Requests (MRs) by leveraging AI, specifically OpenAI's GPT models. It triggers when a specific comment (`+0`) is added to an MR discussion, fetches the code changes from the MR, processes each file's diff, and generates a detailed code review response posted back to the MR discussion. This assists engineers by providing a second opinion or automated review of their code changes.

The workflow's logic is divided into these main blocks:

- **1.1 Input Reception and Trigger Filtering:** Receives webhook events from GitLab, filters comments to detect the trigger comment `+0`.
- **1.2 Fetch and Split MR Changes:** Retrieves the MR changed files and splits them for individual processing.
- **1.3 Filter Relevant File Changes:** Filters out renamed, deleted files or diffs starting with certain patterns to focus on meaningful changes.
- **1.4 Analyze Diff and Prepare Review Input:** Parses diff data, separates original and new code sections, and prepares a prompt for AI review.
- **1.5 AI Review and Response Posting:** Sends the prompt to OpenAI GPT-4o-mini, receives the review text, and posts it as a discussion comment on the MR.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Trigger Filtering

- **Overview:** Listens for GitLab webhook events of type `note_events`, filtering for comments with content exactly `+0` to trigger the review process.
- **Nodes Involved:**  
  - Webhook  
  - Need Review (If node)  
  - Sticky Note2 (comment)  

- **Node Details:**  
  - **Webhook**  
    - Type: Webhook  
    - Role: Entry point receiving POST requests from GitLab webhook configured on `note_events`.  
    - Config: HTTP POST method, path set to a unique identifier for webhook URL.  
    - Input: External GitLab webhook POST payload.  
    - Output: JSON containing comment and MR data.  
    - Failures: Webhook misconfiguration or invalid payload may cause failure.  
  - **Need Review (If)**  
    - Type: If condition node  
    - Role: Filters comments by checking if the comment body equals `+0`.  
    - Config: String equality check on `{{$json.body.object_attributes.note}} == "+0"`, case-sensitive.  
    - Input: Webhook output.  
    - Output: Passes only when comment matches trigger.  
    - Failures: Expression errors if input data structure changes.  
  - **Sticky Note2**  
    - Type: Sticky Note (annotation)  
    - Content: "## Filter comments and customize your trigger words ⬇️"  
    - Role: Instruction to customize trigger condition.

#### 2.2 Fetch and Split MR Changes

- **Overview:** Retrieves the list of changed files and diffs for the MR, then splits each file's changes into separate items for individual processing.
- **Nodes Involved:**  
  - Get Changes (HTTP Request)  
  - Split Out (Split Out)  
  - Sticky Note3 (annotation)  

- **Node Details:**  
  - **Get Changes**  
    - Type: HTTP Request  
    - Role: Calls GitLab API to get MR changes (files and diffs) using project ID and MR IID from webhook payload.  
    - Config: GET request to `https://gitlab.com/api/v4/projects/{project_id}/merge_requests/{iid}/changes` with `PRIVATE-TOKEN` header for authentication.  
    - Input: Filtered webhook data with project and MR identifiers.  
    - Output: JSON containing MR changes array and diff refs.  
    - Failures: Auth errors if token invalid, network issues, or API rate limits.  
  - **Split Out**  
    - Type: Split Out  
    - Role: Splits the `changes` array from `Get Changes` node output into individual items for per-file processing.  
    - Config: Field to split: `changes`.  
    - Input: Output from `Get Changes`.  
    - Output: One item per changed file.  
    - Failures: Empty or missing `changes` field may cause error.  
  - **Sticky Note3**  
    - Content: "## Replace your gitlab URL and token ⬇️"  
    - Role: Instruction to configure correct GitLab URL and private token.

#### 2.3 Filter Relevant File Changes

- **Overview:** Filters out file changes that are renamed, deleted, or contain diffs starting with "@@", likely to exclude non-relevant or complex diffs.
- **Nodes Involved:**  
  - Skip File Changes (If)  
  - Sticky Note4 (annotation)  

- **Node Details:**  
  - **Skip File Changes**  
    - Type: If condition node  
    - Role: Filters out file changes where `renamed_file` or `deleted_file` flags are true, or diff starts with "@@".  
    - Config: Conditions:
      - `renamed_file == false`  
      - `deleted_file == false`  
      - `diff` does not start with "@@" (string startsWith negative).  
    - Input: Individual file change items from `Split Out`.  
    - Output: Passes only relevant diffs for review.  
    - Failures: Expression errors if expected fields missing.  
  - **Sticky Note4**  
    - Content: "## Replace your gitlab URL and token ⬇️"  
    - Role: Duplicate note for GitLab token instructions.

#### 2.4 Analyze Diff and Prepare Review Input

- **Overview:** Parses the diff to extract line information for context and splits original and new code snippets to construct the AI prompt.
- **Nodes Involved:**  
  - Parse Last Diff Line (Code)  
  - Code (Code)  
  - Basic LLM Chain (Langchain LLM Chain)  
  - Sticky Note (annotation)  

- **Node Details:**  
  - **Parse Last Diff Line**  
    - Type: Code (JavaScript)  
    - Role: Parses the diff text to find the last diff hunk header, calculates last old and new line numbers for positioning comments.  
    - Config: Custom JS that:
       - Removes trailing "No newline" notice.  
       - Finds last line starting with `@@ -{start},{count} +{start},{count} @@`.  
       - Computes line numbers for commenting.  
    - Input: Filtered diffs from `Skip File Changes`.  
    - Output: JSON with `lastOldLine`, `lastNewLine`, and original `gitDiff`.  
    - Failures: Parsing errors if diff format changes.  
  - **Code**  
    - Type: Code (JavaScript)  
    - Role: Separates diff lines into `originalCode` (lines starting with `-` or context) and `newCode` (lines starting with `+` or context) for AI prompt.  
    - Config: Loops over diff lines, aggregates lines based on prefix.  
    - Input: Output of `Parse Last Diff Line`, specifically `gitDiff`.  
    - Output: Two fields: `originalCode` and `newCode`.  
    - Failures: Unexpected diff format may cause incorrect code extraction.  
  - **Basic LLM Chain**  
    - Type: Langchain Chain LLM  
    - Role: Constructs a prompt for AI review using file path, original code, and new code sections.  
    - Config:  
      - Prompt template includes file path and formatted code blocks for original and changed code.  
      - Instructions to reject or accept changes with a score and detailed critique in Markdown.  
    - Input: Output from `Code` node combined with file info.  
    - Output: Text prompt to be sent to AI model.  
    - Failures: Expression issues if input JSON structure changes.  
  - **Sticky Note**  
    - Content: "## Edit your own prompt ⬇️"  
    - Role: Instruction to customize AI prompt text.

#### 2.5 AI Review and Response Posting

- **Overview:** Sends the constructed prompt to the OpenAI GPT model, then posts the review comment back to the GitLab MR discussion at the correct position.
- **Nodes Involved:**  
  - OpenAI Chat Model (Langchain LM Chat OpenAI)  
  - Post Discussions (HTTP Request)  

- **Node Details:**  
  - **OpenAI Chat Model**  
    - Type: Langchain OpenAI Chat Model  
    - Role: Sends prompt text to OpenAI GPT-4o-mini model to generate review response.  
    - Config: Model set to `gpt-4o-mini`.  
    - Credentials: OpenAI API key (configured externally).  
    - Input: Prompt from `Basic LLM Chain`.  
    - Output: AI-generated review text.  
    - Failures: API key invalid, quota exceeded, network failure, or model errors.  
  - **Post Discussions**  
    - Type: HTTP Request  
    - Role: Posts AI review as a discussion comment on the GitLab MR, with positioning data for inline comments.  
    - Config:  
      - POST to GitLab API endpoint `/projects/{project_id}/merge_requests/{iid}/discussions`.  
      - Body includes review text, position details (`old_path`, `new_path`, `old_line`, `new_line`), and commit SHAs for context.  
      - Auth via `PRIVATE-TOKEN` header.  
    - Input: Review text from AI and position info from previous nodes.  
    - Output: Confirmation of comment posted.  
    - Failures: Auth failures, invalid position data, API rate limits, or network errors.

---

### 3. Summary Table

| Node Name          | Node Type                      | Functional Role                               | Input Node(s)       | Output Node(s)        | Sticky Note                                                                                   |
|--------------------|--------------------------------|-----------------------------------------------|---------------------|-----------------------|----------------------------------------------------------------------------------------------|
| Webhook            | Webhook                        | Receives GitLab webhook events                 | (External trigger)  | Need Review           |                                                                                              |
| Need Review        | If                            | Filters comments for trigger `+0`              | Webhook             | Get Changes           | ## Filter comments and customize your trigger words ⬇️                                      |
| Get Changes        | HTTP Request                  | Retrieves MR changes from GitLab API           | Need Review          | Split Out             | ## Replace your gitlab URL and token ⬇️                                                     |
| Split Out          | Split Out                     | Splits MR changes array into individual items  | Get Changes          | Skip File Changes      |                                                                                              |
| Skip File Changes  | If                            | Filters out renamed, deleted files, or diff starting with "@@" | Split Out            | Parse Last Diff Line   | ## Replace your gitlab URL and token ⬇️                                                     |
| Parse Last Diff Line | Code (JavaScript)             | Parses diff to find last line numbers for comments | Skip File Changes     | Code                  |                                                                                              |
| Code               | Code (JavaScript)              | Separates diff into original and new code snippets | Parse Last Diff Line  | Basic LLM Chain       | ## Edit your own prompt ⬇️                                                                   |
| Basic LLM Chain     | Langchain Chain LLM            | Builds AI prompt for code review                | Code                 | Post Discussions      |                                                                                              |
| OpenAI Chat Model   | Langchain OpenAI Chat Model    | Sends prompt to OpenAI GPT-4o-mini model        | Basic LLM Chain      | Basic LLM Chain       |                                                                                              |
| Post Discussions    | HTTP Request                  | Posts AI review comment back to GitLab MR       | Basic LLM Chain      |                       |                                                                                              |
| Sticky Note        | Sticky Note                   | Instructions for editing prompt                  |                     |                       | ## Edit your own prompt ⬇️                                                                   |
| Sticky Note2       | Sticky Note                   | Instructions for customizing trigger             |                     |                       | ## Filter comments and customize your trigger words ⬇️                                      |
| Sticky Note3       | Sticky Note                   | Instructions for GitLab URL and token            |                     |                       | ## Replace your gitlab URL and token ⬇️                                                     |
| Sticky Note4       | Sticky Note                   | Duplicate GitLab URL and token instructions      |                     |                       | ## Replace your gitlab URL and token ⬇️                                                     |

---

### 4. Reproducing the Workflow from Scratch

**Step 1:** Create a **Webhook** node  
- Type: Webhook  
- HTTP Method: POST  
- Path: unique string (e.g., `e21095c0-1876-4cd9-9e92-a2eac737f03e`)  
- Purpose: Receive GitLab webhook `note_events` for MR comments.

**Step 2:** Create an **If** node named `Need Review`  
- Condition: Check if `{{$json.body.object_attributes.note}} == "+0"` (case-sensitive exact match)  
- Connect Webhook's main output to this node's input.

**Step 3:** Create an **HTTP Request** node named `Get Changes`  
- Method: GET  
- URL: `https://gitlab.com/api/v4/projects/{{ $json["body"]["project_id"] }}/merge_requests/{{ $json["body"]["merge_request"]["iid"] }}/changes`  
- Send Headers: true  
- Header Parameter: `PRIVATE-TOKEN` with your GitLab personal access token  
- Connect `Need Review` node's true output to this node.

**Step 4:** Create a **Split Out** node  
- Field to split out: `changes`  
- Connect output of `Get Changes` here.

**Step 5:** Create an **If** node named `Skip File Changes`  
- Conditions (all must be true):  
  - `renamed_file` is false  
  - `deleted_file` is false  
  - `diff` does not start with `"@@"` (string startsWith negative)  
- Connect output of `Split Out` here.

**Step 6:** Create a **Code** node named `Parse Last Diff Line`  
- Mode: Run once for each item  
- JS Code (to parse last diff hunk line numbers):

```js
const parseLastDiff = (gitDiff) => {
  gitDiff = gitDiff.replace(/\n\\ No newline at end of file/, '')

  const diffList = gitDiff.trimEnd().split('\n').reverse();
  const lastLineFirstChar = diffList?.[0]?.[0];
  const lastDiff =
    diffList.find((item) => {
      return /^@@ \-\d+,\d+ \+\d+,\d+ @@/g.test(item);
    }) || '';

  const [lastOldLineCount, lastNewLineCount] = lastDiff
    .replace(/@@ \-(\d+),(\d+) \+(\d+),(\d+) @@.*/g, ($0, $1, $2, $3, $4) => {
      return `${+ $1 + + $2},${+ $3 + + $4}`;
    })
    .split(',');

  if (!/^\d+$/.test(lastOldLineCount) || !/^\d+$/.test(lastNewLineCount)) {
    return {
      lastOldLine: -1,
      lastNewLine: -1,
      gitDiff,
    };
  }

  const lastOldLine = lastLineFirstChar === '+' ? null : (parseInt(lastOldLineCount) || 0) - 1;
  const lastNewLine = lastLineFirstChar === '-' ? null : (parseInt(lastNewLineCount) || 0) - 1;

  return {
    lastOldLine,
    lastNewLine,
    gitDiff,
  };
};

return parseLastDiff($input.item.json.diff);
```

- Connect `Skip File Changes` true output here.

**Step 7:** Create a **Code** node named `Code`  
- Mode: Run once for each item  
- JS Code (to separate original and new code):

```js
var diff = $input.item.json.gitDiff

let lines = diff.trimEnd().split('\n');

let originalCode = '';
let newCode = '';

lines.forEach(line => {
  if (line.startsWith('-')) {
    originalCode += line + "\n";
  } else if (line.startsWith('+')) {
    newCode += line + "\n";
  } else {
    originalCode += line + "\n";
    newCode += line + "\n";
  }
});

return {
  originalCode: originalCode,
  newCode: newCode
};
```

- Connect `Parse Last Diff Line` output here.

**Step 8:** Create a **Langchain Chain LLM** node named `Basic LLM Chain`  
- Parameters:  
  - Prompt text includes:  
    ```
    File path：{{ $('Skip File Changes').item.json.new_path }}

    ```Original code
    {{ $json.originalCode }}
    ```
    change to
    ```New code
    {{ $json.newCode }}
    ```
    Please review the code changes in this section:
    ```
  - Set prompt type to "define"  
  - Add initial system message instructing the AI to act as a senior programming expert, giving accept/reject, score 0-100, problems, and optionally corrected code in strict Markdown format.  
- Connect `Code` output here.

**Step 9:** Create a **Langchain OpenAI Chat Model** node named `OpenAI Chat Model`  
- Model: `gpt-4o-mini`  
- Credentials: Configure OpenAI API credentials with your key  
- Connect output from `Basic LLM Chain` to this node's input (ai_languageModel input).

**Step 10:** Connect OpenAI Chat Model output back to `Basic LLM Chain` node's ai_languageModel input (already configured internally).

**Step 11:** Create an **HTTP Request** node named `Post Discussions`  
- Method: POST  
- URL: `https://gitlab.com/api/v4/projects/{{ $('Webhook').item.json["body"]["project_id"] }}/merge_requests/{{ $('Webhook').item.json["body"]["merge_request"]["iid"] }}/discussions`  
- Content-Type: multipart-form-data  
- Headers: `PRIVATE-TOKEN` with your GitLab token  
- Body parameters:  
  - `body`: `={{ $('Basic LLM Chain').item.json["text"] }}` (AI review text)  
  - `position[position_type]`: "text"  
  - `position[old_path]`: `={{ $('Split Out').item.json.old_path }}`  
  - `position[new_path]`: `={{ $('Split Out').item.json.new_path }}`  
  - `position[start_sha]`: `={{ $('Get Changes').item.json.diff_refs.start_sha }}`  
  - `position[head_sha]`: `={{ $('Get Changes').item.json.diff_refs.head_sha }}`  
  - `position[base_sha]`: `={{ $('Get Changes').item.json.diff_refs.base_sha }}`  
  - `position[new_line]`: `={{ $('Parse Last Diff Line').item.json.lastNewLine || '' }}`  
  - `position[old_line]`: `={{ $('Parse Last Diff Line').item.json.lastOldLine || '' }}`  
- Connect `Basic LLM Chain` main output to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                               | Context or Link                                                                                                                      |
|--------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| This workflow triggers from GitLab `note_events` webhook comments, so ensure you configure your GitLab repo webhook accordingly.          | https://docs.gitlab.com/ee/user/project/integrations/webhooks.html#:~:text=Configure%20a%20webhook%20in%20GitLab                  |
| The AI prompt and scoring format can be customized in the `Basic LLM Chain` node to fit your team's review style.                         | Sticky Note in workflow: "## Edit your own prompt ⬇️"                                                                               |
| Replace all placeholders for GitLab URLs and tokens with your actual project URL and valid personal access tokens for API authentication.  | Sticky Notes: "## Replace your gitlab URL and token ⬇️"                                                                             |
| Uses OpenAI GPT-4o-mini model; ensure your OpenAI credentials are configured in n8n credentials with sufficient quota and permissions.       | Node: OpenAI Chat Model                                                                                                             |
| The workflow posts inline review comments referencing the exact changed lines using GitLab's position API, which requires accurate line info.| Nodes: Parse Last Diff Line and Post Discussions                                                                                     |

---

This document provides a detailed, stepwise understanding and reconstruction guide for the "ChatGPT Automatic Code Review in GitLab MR" workflow, suitable for advanced users, developers, and AI agents.