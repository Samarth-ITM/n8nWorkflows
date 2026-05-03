Monitor GitHub Releases with Gemini AI Chinese Translation & Slack Notifications

https://n8nworkflows.xyz/workflows/monitor-github-releases-with-gemini-ai-chinese-translation---slack-notifications-3452


# Monitor GitHub Releases with Gemini AI Chinese Translation & Slack Notifications

### 1. Workflow Overview

This workflow automates monitoring of selected GitHub repositories for new releases, leverages AI (Google Gemini by default) to summarize and translate release notes into Chinese, and notifies a Slack channel with a formatted message. It is designed for teams or communities who want timely, understandable updates on project releases without manually checking GitHub.

**Logical Blocks:**

- **1.1 Scheduled Trigger & Configuration Load:** Periodically triggers the workflow and loads the list of GitHub repositories to monitor.
- **1.2 Repository Iteration & RSS Fetching:** Iterates over each repository, fetching its releases feed and handling fetch errors gracefully.
- **1.3 New Release Detection:** Uses Redis caching to compare the latest release ID against previously processed ones to prevent duplicate notifications.
- **1.4 AI Processing & Translation:** Sends release content to Gemini AI to extract and translate key changes into Chinese.
- **1.5 Slack Notification & State Update:** Formats the AI output into Slack Block Kit messages, sends notifications, and updates Redis with the latest processed release ID.
- **1.6 Error Notification:** Sends Slack alerts on individual repository fetch failures without halting the workflow.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Configuration Load

- **Overview:** Initiates the workflow on a schedule and loads the repository list.
- **Nodes Involved:** `Cron Trigger`, `GitHub Config`, `Loop`

##### Node Details:

- **Cron Trigger**
  - *Type:* Schedule Trigger
  - *Role:* Starts the workflow every 10 minutes between 9 AM and 11 PM daily (configurable via cron expression).
  - *Key Config:* `0 */10 9-23 * * *`
  - *Input:* None (start node)
  - *Output:* Triggers `GitHub Config`
  - *Edge Cases:* Misconfigured cron expression could cause no executions or excessive runs.

- **GitHub Config**
  - *Type:* Code Node
  - *Role:* Provides an array of repository objects with `name` and `github` keys, defining which repos to monitor.
  - *Key Config:* JavaScript array of repos, e.g. `{ name: "n8n", github: "n8n-io/n8n" }`
  - *Input:* Trigger from `Cron Trigger`
  - *Output:* To `Loop`
  - *Edge Cases:* Malformed array or syntax errors could break iteration.

- **Loop**
  - *Type:* SplitInBatches
  - *Role:* Iterates over each repository item for downstream processing.
  - *Input:* Repository list from `GitHub Config`
  - *Output:* Splits and sends one repo per iteration to `If Not Empty` and `RSS for Release`
  - *Edge Cases:* Large lists might cause long runtime; batch size defaults to 1.

#### 2.2 Repository Iteration & RSS Fetching

- **Overview:** Fetches the GitHub releases RSS feed per repository and checks for errors.
- **Nodes Involved:** `If Not Empty`, `RSS for Release`, `If No Error`, `Send Error`, `Limit`

##### Node Details:

- **If Not Empty**
  - *Type:* If Node
  - *Role:* Checks if the repository object is non-empty before continuing.
  - *Condition:* Object is not empty.
  - *Input:* From `Loop`
  - *Output:* True branch to `Information Extractor`; False branch to `RSS for Release`
  - *Edge Cases:* Empty repository objects skipped.

- **RSS for Release**
  - *Type:* RSS Feed Read
  - *Role:* Fetches releases feed for the current repository using URL `https://github.com/{{github}}/releases.atom`
  - *Input:* From `Loop`
  - *Output:* To `If No Error`
  - *Error Handling:* `onError` set to continue, allowing workflow to handle errors downstream.
  - *Edge Cases:* Network errors, feed unavailability, or rate limits.

- **If No Error**
  - *Type:* If Node
  - *Role:* Checks if the RSS fetch resulted in an error (`$json.error` empty means success).
  - *Input:* From `RSS for Release`
  - *Output:* True to `Limit` (continue processing), False to `Send Error` (notify Slack)
  - *Edge Cases:* If error present, triggers Slack error message.

- **Send Error**
  - *Type:* Slack Node
  - *Role:* Sends an error notification to Slack for the repository that failed to fetch RSS.
  - *Text:* Includes repo name and error details.
  - *Credentials:* Slack API configured.
  - *Input:* From `If No Error` (false branch)
  - *Output:* To `Null` (ends error path)
  - *Edge Cases:* Slack API rate limits or credential misconfiguration.

- **Limit**
  - *Type:* Limit Node
  - *Role:* Limits number of releases processed downstream (default no parameter set).
  - *Input:* From `If No Error` (true branch)
  - *Output:* To `Redis Get`
  - *Edge Cases:* If improperly configured, might block processing.

#### 2.3 New Release Detection

- **Overview:** Uses Redis to check if the latest release ID is new compared to stored cache.
- **Nodes Involved:** `Redis Get`, `If New`, `Edit Fields`, `Null`

##### Node Details:

- **Redis Get**
  - *Type:* Redis Node
  - *Role:* Retrieves last processed release ID from Redis key `github_release:<repo_github>`
  - *Credentials:* Redis configured in n8n.
  - *Input:* From `Limit`
  - *Output:* To `If New`
  - *Edge Cases:* Redis connection failures, key missing (treated as new release).

- **If New**
  - *Type:* If Node
  - *Role:* Compares Redis cached release ID (`$json.cache`) with latest release ID from RSS (`$json.id`).
  - *Condition:* Proceed if IDs differ, indicating a new release.
  - *Input:* From `Redis Get`
  - *Output:* True to `Edit Fields`, False to `Null` (skip processing)
  - *Edge Cases:* False negatives if Redis cache corrupted.

- **Edit Fields**
  - *Type:* Set Node
  - *Role:* Assembles and normalizes release metadata for subsequent processing.
  - *Parameters:* Sets `name`, `title`, `link`, `pubDate`, `contentSnippet`, `id`, `github` from prior nodes.
  - *Input:* From `If New`
  - *Output:* To `Loop` (back to start of processing for refined data)
  - *Edge Cases:* Expression failures if referenced nodes missing or data malformed.

- **Null**
  - *Type:* Set Node (empty)
  - *Role:* Ends branch for already processed releases or error paths.
  - *Input:* From `If New` false branch or `If No Error` false branch (after error)
  - *Output:* To `Loop` or workflow end.
  - *Edge Cases:* None significant.

#### 2.4 AI Processing & Translation

- **Overview:** Uses Gemini AI via the LangChain `Information Extractor` to parse release notes, categorize changes, translate to Chinese, and simplify content.
- **Nodes Involved:** `Information Extractor`, `Date Format`, `Code for Slack Tpl`, `Gemini`, `If Not Empty`

##### Node Details:

- **Information Extractor**
  - *Type:* LangChain Information Extractor Node
  - *Role:* Takes `contentSnippet` text, extracts categorized change items (features, fixes, others), translates into Chinese, and simplifies descriptions.
  - *Configuration:* Uses a detailed system prompt specifying extraction rules, translation to Chinese, and formatting constraints (no markdown, no untranslated content).
  - *Input:* From `If Not Empty`
  - *Output:* To `Date Format`
  - *AI Model:* Connected to `Gemini` node for LLM completion.
  - *Edge Cases:* AI model errors, prompt misconfiguration, or output schema mismatch.

- **Gemini**
  - *Type:* LangChain Google Gemini Chat Node
  - *Role:* Provides AI language model backend for `Information Extractor`.
  - *Model:* `models/gemini-2.0-flash-001`
  - *Temperature:* 0.3 (low randomness for consistent output)
  - *Credentials:* Google Gemini configured.
  - *Input:* From `Information Extractor` (as AI model)
  - *Output:* To `Information Extractor`
  - *Edge Cases:* API quota exceeded, authentication failure.

- **Date Format**
  - *Type:* DateTime Node
  - *Role:* Formats the release `pubDate` into `yyyy-MM-dd HH:mm` with timezone awareness for display.
  - *Input:* From `Information Extractor`
  - *Output:* To `Code for Slack Tpl`
  - *Edge Cases:* Invalid date strings causing format failure.

- **Code for Slack Tpl**
  - *Type:* Code Node (JavaScript)
  - *Role:* Converts AI-extracted JSON content and metadata into Slack Block Kit message blocks with headers, context, dividers, and categorized bullet lists.
  - *Input:* From `Date Format`
  - *Output:* To `Send Message`
  - *Key Logic:* Generates rich_text block sections for Features, Fixes, Others categories.
  - *Edge Cases:* Unexpected AI output format or missing data causes malformed Slack blocks.

- **If Not Empty**
  - *Type:* If Node
  - *Role:* Ensures that the input JSON is not empty before starting AI extraction.
  - *Input:* From `Loop`
  - *Output:* True to `Information Extractor`
  - *Edge Cases:* Prevents AI processing on empty content.

#### 2.5 Slack Notification & State Update

- **Overview:** Sends the formatted Slack notification and updates Redis with the new release ID.
- **Nodes Involved:** `Send Message`, `Redis Set Id`

##### Node Details:

- **Send Message**
  - *Type:* Slack Node
  - *Role:* Sends a Slack message to a specified channel with the rich Block Kit layout derived from AI extraction.
  - *Text:* Simple fallback text with release name.
  - *Channel ID:* Configured (e.g., `C08ME7TDZ3J`)
  - *Credentials:* Slack API configured.
  - *Input:* From `Code for Slack Tpl`
  - *Output:* To `Redis Set Id`
  - *Edge Cases:* Slack API rate limits, invalid channel ID, or credential issues.

- **Redis Set Id**
  - *Type:* Redis Node
  - *Role:* Stores the current release ID into Redis with key `github_release:<repo_github>` to track processed releases.
  - *Credentials:* Redis account configured.
  - *Input:* From `Send Message`
  - *Output:* None (workflow end for this iteration)
  - *Edge Cases:* Redis connectivity or write failures.

---

### 3. Summary Table

| Node Name           | Node Type                       | Functional Role                          | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                                                         |
|---------------------|--------------------------------|----------------------------------------|-----------------------------|----------------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| Cron Trigger        | Schedule Trigger               | Starts workflow on schedule             | None                        | GitHub Config              | Adjust the `Rule` setting to change the update check frequency (default is `0 */10 9-23 * * *`, every 10 minutes 9 AM-11 PM)        |
| GitHub Config       | Code Node                     | Provides repository list config         | Cron Trigger                | Loop                       | Edit the JavaScript array to add/remove repositories to monitor. Example provided in sticky note.                                  |
| Loop                | SplitInBatches                | Iterates over each repository            | GitHub Config, Null         | If Not Empty, RSS for Release |                                                                                                                                     |
| If Not Empty        | If Node                      | Checks non-empty repository data        | Loop                        | Information Extractor       |                                                                                                                                     |
| RSS for Release     | RSS Feed Read                 | Fetches GitHub releases RSS feed        | Loop                        | If No Error                |                                                                                                                                     |
| If No Error         | If Node                      | Checks if RSS fetch succeeded            | RSS for Release             | Limit, Send Error           |                                                                                                                                     |
| Send Error          | Slack Node                   | Sends Slack error notification           | If No Error                 | Null                       | Select Slack credentials and set channel ID for error notifications.                                                                |
| Limit               | Limit Node                   | Limits number of releases processed      | If No Error                 | Redis Get                  |                                                                                                                                     |
| Redis Get           | Redis Node                   | Retrieves last processed release ID      | Limit                       | If New                     | Redis credentials required.                                                                                                         |
| If New              | If Node                      | Compares cached release ID with latest   | Redis Get                   | Edit Fields, Null           |                                                                                                                                     |
| Edit Fields         | Set Node                     | Normalizes release metadata               | If New                      | Loop                       |                                                                                                                                     |
| Null                | Set Node (empty)             | Ends branches for skips/errors            | If New, If No Error          | Loop                       |                                                                                                                                     |
| Information Extractor | LangChain Information Extractor | AI extraction and translation of release | If Not Empty                | Date Format                | AI configured to extract and translate release data to Chinese using Gemini by default. Prompt modifiable.                          |
| Gemini              | LangChain Google Gemini Node  | AI model backend                         | Information Extractor (ai_languageModel) | Information Extractor  | Use configured Google Gemini credentials; optional replacement possible.                                                            |
| Date Format         | DateTime Node                | Formats release publish date              | Information Extractor       | Code for Slack Tpl          |                                                                                                                                     |
| Code for Slack Tpl  | Code Node                   | Formats Slack Block Kit message           | Date Format                 | Send Message               |                                                                                                                                     |
| Send Message        | Slack Node                   | Sends release notification to Slack      | Code for Slack Tpl          | Redis Set Id               | Select Slack credentials and set the target channel ID.                                                                             |
| Redis Set Id        | Redis Node                   | Saves current release ID to Redis         | Send Message                | None                       | Redis credentials required.                                                                                                         |
| Sticky Note         | Sticky Note                  | Configuration and usage notes             | None                       | None                       | Multiple sticky notes exist for configuration instructions and usage tips, including Slack permissions and AI prompt details.      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create `Cron Trigger`:**
   - Type: Schedule Trigger
   - Parameters: Cron expression `0 */10 9-23 * * *` (runs every 10 min between 9 AM and 11 PM)
   - No credentials needed.

2. **Create `GitHub Config` Code Node:**
   - Type: Code
   - JavaScript code returns array of repository objects:
     ```javascript
     return [
       { name: "n8n", github: "n8n-io/n8n" },
       { name: "LobeChat", github: "lobehub/lobe-chat" },
       // add more as needed
     ];
     ```
   - Connect output from `Cron Trigger`.

3. **Create `Loop` Node:**
   - Type: SplitInBatches (default batch size 1)
   - Input: Connect from `GitHub Config` output.

4. **Create `If Not Empty` Node:**
   - Type: If
   - Condition: Check if current item is not empty (object not empty).
   - Input: Connect from `Loop` node.

5. **Create `RSS for Release` Node:**
   - Type: RSS Feed Read
   - URL: `https://github.com/{{ $json.github }}/releases.atom`
   - On error: Continue workflow.
   - Input: Connect from `Loop` node.

6. **Create `If No Error` Node:**
   - Type: If
   - Condition: `$json.error` is empty (no error).
   - Input: Connect from `RSS for Release`.

7. **Create `Send Error` Slack Node:**
   - Type: Slack
   - Text: `=:x: *{{ $json.name }}* Error\n> {{ $json.error }}`
   - Channel ID: Your Slack channel ID
   - Credentials: Configure Slack API credentials with `chat:write` and `chat:write.customize` scopes.
   - Input: Connect from `If No Error` false branch.

8. **Create `Limit` Node:**
   - Type: Limit Node (no parameters by default)
   - Input: Connect from `If No Error` true branch.

9. **Create `Redis Get` Node:**
   - Type: Redis
   - Operation: Get
   - Key: `github_release:{{ $json.github }}`
   - Credentials: Configure Redis connection.
   - Input: Connect from `Limit`.

10. **Create `If New` Node:**
    - Type: If
    - Condition: Compare Redis value (`$json.cache`) with current release ID (`$json.id`) — proceed if not equal.
    - Input: Connect from `Redis Get`.

11. **Create `Edit Fields` Set Node:**
    - Type: Set
    - Fields to assign from prior nodes: 
      - `name` from `Loop.item.json.name`
      - `title`, `link`, `pubDate`, `contentSnippet`, `id` from `Limit.item.json`
      - `github` from `Loop.item.json.github`
    - Input: Connect from `If New` true branch.

12. **Create `Null` Node:**
    - Type: Set (empty)
    - Input: Connect from `If New` false branch and from `Send Error` Slack node.

13. **Connect `Edit Fields` output back to `Loop` input:**
    - To iterate with augmented data.

14. **Create `If Not Empty` Node (second use for AI processing):**
    - Type: If
    - Condition: Input JSON is not empty.
    - Input: Connect from `Loop` node (second iteration).

15. **Create `Gemini` Node:**
    - Type: LangChain Google Gemini Chat
    - Model: `models/gemini-2.0-flash-001`
    - Temperature: 0.3
    - Credentials: Configure Google Gemini API credentials.

16. **Create `Information Extractor` Node:**
    - Type: LangChain Information Extractor
    - Parameters:
      - Text: `{{$json.contentSnippet}}`
      - System Prompt: Detailed prompt instructing extraction of categorized features, fixes, others, translation to Chinese, simplification, and formatting rules as in the description.
    - Input: Connect from `If Not Empty` true branch.
    - AI Model: Connect to `Gemini` node.

17. **Create `Date Format` Node:**
    - Type: DateTime
    - Operation: Format date from `pubDate`
    - Format: `yyyy-MM-dd HH:mm`
    - Timezone enabled.
    - Input: Connect from `Information Extractor`.

18. **Create `Code for Slack Tpl` Node:**
    - Type: Code
    - JavaScript code constructs Slack Block Kit messages with header, context, and categorized bullet lists from AI output and metadata.
    - Input: Connect from `Date Format`.

19. **Create `Send Message` Slack Node:**
    - Type: Slack
    - Message type: Block
    - Text fallback: `Release - {{ $json.name }}`
    - Channel ID: Your Slack channel ID
    - Credentials: Slack API credentials.
    - Input: Connect from `Code for Slack Tpl`.

20. **Create `Redis Set Id` Node:**
    - Type: Redis
    - Operation: Set
    - Key: `github_release:{{ $json.github }}`
    - Value: `{{ $json.id }}`
    - Credentials: Redis credentials.
    - Input: Connect from `Send Message`.

---

### 5. General Notes & Resources

| Note Content                                                                                                                | Context or Link                                                                                                         |
|-----------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| Slack permissions require adding `chat:write` and `chat:write.customize` scopes and reinstalling the app for tokens to work | Slack OAuth & Permissions configuration                                                                                 |
| AI prompt for `Information Extractor` node is highly customized to extract and translate GitHub release notes to Chinese    | Modify prompt to change language or extraction style                                                                    |
| Redis keys follow pattern `github_release:<repo_github>` to store last processed release ID                                  | Enables duplicate prevention                                                                                            |
| Cron expression defaults to every 10 minutes between 9 AM and 11 PM, adjust via `Cron Trigger` node                         | See node configuration                                                                                                |
| Slack messages use Block Kit rich_text sections for clear, categorized release notes                                          | Formatting done by `Code for Slack Tpl` node                                                                            |
| Workflow handles individual repository RSS fetch errors gracefully with Slack error notifications without stopping entire run | Robust for multi-repo monitoring                                                                                        |

---

This documentation fully describes the workflow structure, function, and configuration to empower users and automation agents to understand, replicate, or modify it effectively.