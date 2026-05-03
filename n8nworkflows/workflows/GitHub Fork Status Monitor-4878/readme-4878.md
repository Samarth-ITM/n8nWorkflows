GitHub Fork Status Monitor

https://n8nworkflows.xyz/workflows/github-fork-status-monitor-4878


# GitHub Fork Status Monitor

### 1. Workflow Overview

**Purpose:**  
The "GitHub Fork Status Monitor" workflow automates the process of monitoring a user's GitHub forked repositories. It tracks whether each fork is ahead, behind, or up-to-date relative to its upstream (original) repository. The workflow summarizes these statuses and sends a formatted report via Telegram. It supports both manual execution and Telegram bot command triggers.

**Target Use Cases:**  
- Developers or teams wanting to monitor synchronization status of their forked repositories regularly.  
- Automated notifications to stay informed about whether forks need updating or have new commits ahead of upstream.  
- Integration with Telegram for real-time alerts and easy status review.

**Logical Blocks:**  
- **1.1 Input Reception & Command Parsing:** Accepts triggers manually or via Telegram, extracts the number of repositories to check.  
- **1.2 GitHub Repository Retrieval & Filtering:** Fetches user repositories from GitHub and filters only forked repositories.  
- **1.3 Fork and Upstream Relationship Processing:** Builds comparison URLs for fork vs upstream branch differences.  
- **1.4 Status Comparison & Processing:** Calls GitHub API to compare branches, processes results to classify repo status.  
- **1.5 Result Aggregation & Formatting:** Combines results, formats a markdown table message for Telegram.  
- **1.6 Notification Delivery:** Sends the formatted status report via Telegram message.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Command Parsing

- **Overview:**  
  Accepts user input triggers either manually or through a Telegram bot command `/forkcheck`. Parses the input to determine how many repositories to check, with a default of 1000.

- **Nodes Involved:**  
  - Telegram Trigger  
  - Code (Command Processing)  
  - If (Check Valid Number)  
  - When clicking ‘Execute workflow’ (Manual trigger)

- **Node Details:**

  - **Telegram Trigger**  
    - *Type:* Telegram Trigger  
    - *Role:* Listens for incoming Telegram messages to trigger workflow execution.  
    - *Configuration:* Listens for 'message' updates only; authenticated with a Telegram bot credential.  
    - *Inputs/Outputs:* Outputs message object to the Code node.  
    - *Edge Cases:* Invalid or unsupported commands; messages without text.

  - **Code (Command Processing)**  
    - *Type:* Code (JavaScript)  
    - *Role:* Parses Telegram message text to extract a number for repositories to check; defaults to 1000 if none provided or invalid.  
    - *Key Logic:* Splits message text by space; checks if first word is `/forkcheck`; parses second word as integer; sets default if invalid.  
    - *Input:* Telegram message JSON.  
    - *Output:* JSON with field `repos` indicating number of repos to check.  
    - *Edge Cases:* Non-`/forkcheck` commands result in repos=0; malformed inputs logged.

  - **If (Check Valid Number)**  
    - *Type:* If  
    - *Role:* Determines if parsed `repos` number is greater than 0 to proceed.  
    - *Condition:* Number of repos > 0.  
    - *Input:* JSON containing `repos` field.  
    - *Output:* If true, continues to GitHub repo fetching; else stops.  
    - *Edge Cases:* Zero or negative numbers block workflow progression.

  - **When clicking ‘Execute workflow’**  
    - *Type:* Manual Trigger  
    - *Role:* Allows manual start of the workflow for testing or on-demand runs.  
    - *Input/Output:* Starts the flow directly to repository fetching.

---

#### 2.2 GitHub Repository Retrieval & Filtering

- **Overview:**  
  Fetches a limited number of repositories from the configured GitHub user account, then filters to only those marked as forks.

- **Nodes Involved:**  
  - Get repos (GitHub node)  
  - Filter forked repositories only (Filter node)

- **Node Details:**

  - **Get repos**  
    - *Type:* GitHub node (User resource)  
    - *Role:* Retrieves repositories owned by a specified user up to the limit specified by input `repos`.  
    - *Configuration:* Owner set to a fixed username ("mk-in"); limit dynamically set from input JSON `repos`.  
    - *Credentials:* Uses GitHub API personal access token.  
    - *Output:* List of repositories with detailed JSON metadata.

  - **Filter forked repositories only**  
    - *Type:* Filter  
    - *Role:* Filters the list to only include repositories where `fork` = true.  
    - *Condition:* JSON field `fork` equals `true` (strict string check).  
    - *Input:* Output of Get repos node.  
    - *Output:* Forked repositories only.

---

#### 2.3 Fork and Upstream Relationship Processing

- **Overview:**  
  For each forked repository, constructs a GitHub API URL to compare the fork's default branch with its upstream repository's default branch. Embeds original repository metadata for downstream use.

- **Nodes Involved:**  
  - Loop Over Items (splitInBatches)  
  - get-repo-details (GitHub node)  
  - Prepare Upstream URL (Code node)

- **Node Details:**

  - **Loop Over Items**  
    - *Type:* SplitInBatches  
    - *Role:* Processes repositories in batches (default batch size).  
    - *Input:* Filtered forked repositories.  
    - *Output:* Single repo per batch for detailed processing.

  - **get-repo-details**  
    - *Type:* GitHub node (Repository resource)  
    - *Role:* Retrieves detailed info for each repository, ensuring complete data including fork and parent info.  
    - *Parameters:* Owner and repo name extracted dynamically from input item JSON.  
    - *Credentials:* GitHub API token.  
    - *Output:* Complete repo metadata including parent repository details.

  - **Prepare Upstream URL**  
    - *Type:* Code (JavaScript)  
    - *Role:* Validates if repository is a fork with a parent; extracts upstream and fork owner/name/branches; constructs a compare API URL in format:  
      `https://api.github.com/repos/{upstreamOwner}/{upstreamName}/compare/{upstreamDefaultBranch}...{forkOwner}:{forkDefaultBranch}`  
    - *Includes:* Embeds original repo metadata nested in output JSON under `original_repo_data` for later correlation.  
    - *Edge Cases:* Skips repos not marked as forks or lacking parent info with console logging.

---

#### 2.4 Status Comparison & Processing

- **Overview:**  
  Calls GitHub’s branch compare API for each prepared URL, then processes the API response to determine if the fork is ahead, behind, or up-to-date relative to upstream.

- **Nodes Involved:**  
  - Compare Branches API Call (HTTP Request)  
  - Process Comparison Result (Code node)  
  - Combine Results (Merge node)

- **Node Details:**

  - **Compare Branches API Call**  
    - *Type:* HTTP Request  
    - *Role:* Calls the GitHub compare API endpoint using the URL constructed previously.  
    - *Headers:* Sets `Accept: application/vnd.github.v3+json` for GitHub API versioning.  
    - *Authentication:* Uses HTTP header authentication with a GitHub personal access token (PAT).  
    - *Input:* JSON with `compareApiUrl`.  
    - *Output:* API response with ahead/behind commit counts.

  - **Process Comparison Result**  
    - *Type:* Code (JavaScript)  
    - *Role:* Reads API response; extracts `behind_by` and `ahead_by` commit counts; classifies status as:  
      - `BEHIND` if behind_by > 0  
      - `AHEAD` if ahead_by > 0 (and behind_by = 0)  
      - `UP-TO-DATE` if neither ahead nor behind  
    - *Includes:* Original repo info to annotate output.  
    - *Edge Cases:* Handles missing or malformed API data by returning error status and logging.

  - **Combine Results**  
    - *Type:* Merge  
    - *Role:* Aggregates all processed comparison results back into a single data stream for final formatting.  
    - *Input:* Multiple items from batch processing.  
    - *Output:* Combined list for final reporting.

---

#### 2.5 Result Aggregation & Formatting

- **Overview:**  
  Formats the combined repository status data into a markdown table message suitable for Telegram with counts of ahead and behind commits and repository links.

- **Nodes Involved:**  
  - Format for Telegram (Code node)

- **Node Details:**

  - **Format for Telegram**  
    - *Type:* Code (JavaScript)  
    - *Role:*  
      - Builds a message header with date and title.  
      - Iterates through each repository status item, ignoring up-to-date repos in the table but counting them.  
      - Constructs a MarkdownV2-compatible table with columns: Ahead, Behind, Repository (with escaped repository names for Telegram markdown).  
      - Formats numeric columns with zero-padded width for alignment.  
    - *Output:* JSON field `telegramMessage` containing the fully formatted markdown message.  
    - *Edge Cases:* Proper escaping of special characters in repo names to avoid Telegram markdown parsing errors.

---

#### 2.6 Notification Delivery

- **Overview:**  
  Sends the formatted GitHub fork status report to a predefined Telegram chat.

- **Nodes Involved:**  
  - Telegram (Send message)

- **Node Details:**

  - **Telegram**  
    - *Type:* Telegram node (Send message)  
    - *Role:* Sends a markdown message (`telegramMessage`) to a configured chat ID.  
    - *Parameters:* Uses chat ID `6083752603`; disables attribution footer for message.  
    - *Credentials:* Authenticated with Telegram bot API credentials.  
    - *Input:* Receives formatted message JSON.  
    - *Edge Cases:* Telegram API errors, chat ID invalid, message formatting issues.

---

### 3. Summary Table

| Node Name                    | Node Type              | Functional Role                                  | Input Node(s)                  | Output Node(s)                | Sticky Note                                                                                       |
|------------------------------|------------------------|-------------------------------------------------|-------------------------------|------------------------------|-------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger         | Starts workflow manually                         |                               | Get repos                    |                                                                                                 |
| Telegram Trigger             | Telegram Trigger       | Listens for Telegram commands                    |                               | Code                         | ## Triggers - *Set your triggers here*<br>## Default<br>- Manual<br>- Telegram bot with command /forkcheck<br>## Usage<br>- /forkcheck = 1000 repositories max<br>- /forkcheck n = n repositories<br>- any other command is rejected |
| Code                        | Code                   | Parses Telegram command to extract repo count   | Telegram Trigger              | If                           |                                                                                                 |
| If                          | If                     | Checks if repo count > 0                         | Code                         | Get repos                    |                                                                                                 |
| Get repos                   | GitHub                  | Fetches user repositories                        | If, When clicking ‘Execute workflow’ | Filter forked repositories only | ## Filter repositories<br>Select only forked repositories<br>Required argument: Count to select repositories |
| Filter forked repositories only | Filter                 | Filters to forked repos only                     | Get repos                    | Loop Over Items              |                                                                                                 |
| Loop Over Items             | SplitInBatches          | Processes repos in batches for detail processing | Filter forked repositories only | Format for Telegram, get-repo-details | ## Processing logic                                                                           |
| get-repo-details            | GitHub                  | Gets detailed info for each repo                 | Loop Over Items              | Prepare Upstream URL         |                                                                                                 |
| Prepare Upstream URL        | Code                   | Constructs GitHub compare API URLs               | get-repo-details             | Compare Branches API Call    |                                                                                                 |
| Compare Branches API Call   | HTTP Request            | Calls GitHub API to compare branches             | Prepare Upstream URL         | Process Comparison Result    |                                                                                                 |
| Process Comparison Result   | Code                   | Parses API response to determine fork status     | Compare Branches API Call    | Combine Results              |                                                                                                 |
| Combine Results             | Merge                   | Aggregates processed repo status results         | Process Comparison Result    | Loop Over Items              |                                                                                                 |
| Format for Telegram         | Code                   | Formats repo status as Telegram markdown message | Loop Over Items              | Telegram                    | ## Summarize and send                                                                           |
| Telegram                   | Telegram                | Sends status report message                       | Format for Telegram          |                              |                                                                                                 |
| Sticky Note                 | Sticky Note             | Documentation and usage notes                     |                               |                              | ## Triggers - *Set your triggers here*<br>## Default<br>- Manual<br>- Telegram bot with command /forkcheck<br>## Usage<br>- /forkcheck = 1000 repositories max<br>- /forkcheck n = n repositories<br>- any other command is rejected |
| Sticky Note1                | Sticky Note             | Notes on filtering repositories                   |                               |                              | ## Filter repositories<br>Select only forked repositories<br>Required argument: Count to select repositories |
| Sticky Note2                | Sticky Note             | Processing logic overview                          |                               |                              | ## Processing logic                                                                             |
| Sticky Note3                | Sticky Note             | Summarize and send block note                      |                               |                              | ## Summarize and send                                                                           |
| Sticky Note4                | Sticky Note             | General workflow description                       |                               |                              | ## GitHub Fork Status Monitor<br>This workflow helps you keep an eye on your GitHub forks, notifying you when they fall behind or pull ahead of their upstream repositories. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Allow manual execution for testing or ad-hoc runs.

2. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure to listen to "message" updates only.  
   - Attach Telegram Bot credentials (OAuth2 or Bot token).  
   - Connect output to the next node (Code node).

3. **Create Code Node (Command Parsing)**  
   - Type: Code  
   - Paste JavaScript code to parse Telegram text commands:  
     - Split text by space.  
     - Check if first word is `/forkcheck`.  
     - Parse second word as integer for repo count; default 1000 if missing or zero.  
     - Output JSON with numeric field `repos`.  
   - Connect input from Telegram Trigger node.

4. **Create If Node (Check Valid Number)**  
   - Condition: `repos > 0` (number greater than zero).  
   - Connect input from Code node.  
   - True branch continues to GitHub repo fetching; false branch ends workflow.

5. **Create GitHub Node (Get Repos)**  
   - Resource: User repositories.  
   - Parameters:  
     - Owner: fixed username (e.g., "mk-in").  
     - Limit: set dynamically from input `repos` field.  
   - Credentials: GitHub personal access token (PAT).  
   - Connect input from If node.

6. **Create Filter Node (Filter Forked Repositories Only)**  
   - Condition: JSON field `fork` equals `true`.  
   - Connect input from Get repos node.

7. **Create SplitInBatches Node (Loop Over Items)**  
   - Process repos in batches (default batch size).  
   - Connect input from Filter node.

8. **Create GitHub Node (get-repo-details)**  
   - Resource: Repository (get detailed info).  
   - Parameters:  
     - Owner: set dynamically from input item's `owner.login`.  
     - Repository: set dynamically from input item's `html_url` (parsed for repo name).  
   - Credentials: GitHub PAT.  
   - Connect input from SplitInBatches node.

9. **Create Code Node (Prepare Upstream URL)**  
   - Paste JavaScript code to:  
     - Validate repo is a fork with a parent.  
     - Extract upstream owner, repo name, branches.  
     - Construct GitHub compare API URL for branch comparison.  
     - Include original repo data nested for correlation.  
   - Connect input from get-repo-details node.

10. **Create HTTP Request Node (Compare Branches API Call)**  
    - HTTP Method: GET.  
    - URL: set dynamically from input JSON field `compareApiUrl`.  
    - Headers: `Accept: application/vnd.github.v3+json`.  
    - Authentication: HTTP Header Auth with GitHub PAT credential.  
    - Connect input from Prepare Upstream URL node.

11. **Create Code Node (Process Comparison Result)**  
    - Paste JavaScript code to:  
      - Extract `ahead_by` and `behind_by` from API response.  
      - Determine status: BEHIND, AHEAD, or UP-TO-DATE.  
      - Handle errors on missing data with error status.  
      - Output normalized JSON with status and counts.  
    - Connect input from HTTP Request node.

12. **Create Merge Node (Combine Results)**  
    - Merge all processed results into a single stream.  
    - Connect input from Process Comparison Result node.

13. **Connect Merge Node output back to SplitInBatches node**  
    - This creates a batch processing loop.

14. **Create Code Node (Format for Telegram)**  
    - Paste JavaScript code to:  
      - Create a MarkdownV2 table with columns Ahead, Behind, Repository.  
      - Escape repo names properly.  
      - Format numbers zero-padded and right-aligned.  
      - Include summary counts and date header.  
    - Connect input from Loop Over Items node (output branch after Combine Results).

15. **Create Telegram Node (Send Message)**  
    - Parameters:  
      - Chat ID: fixed Telegram chat ID (e.g., 6083752603).  
      - Text: set dynamically from input JSON `telegramMessage`.  
      - Additional fields: disable attribution.  
    - Credentials: Telegram bot credentials.  
    - Connect input from Format for Telegram node.

16. **Add Sticky Notes for Documentation**  
    - Add sticky notes with notes on triggers, filtering, processing logic, and summary as per original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                         |
|----------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| This workflow helps you keep an eye on your GitHub forks, notifying you when they fall behind or pull ahead of their upstream repositories. | General workflow purpose (Sticky Note4)                                                               |
| Triggers: Manual and Telegram bot with command `/forkcheck`. Usage: `/forkcheck` checks up to 1000 repos, `/forkcheck n` checks n repos, others rejected. | Trigger configuration and usage instructions (Sticky Note)                                            |
| Filter repositories block selects only forked repositories. Requires the number of repositories to select from input. | Filter repositories note (Sticky Note1)                                                               |
| Processing logic block manages batch processing, API calls, and data transformation.                      | Processing logic note (Sticky Note2)                                                                   |
| Summarize and send block formats results and sends Telegram notifications.                               | Summarize and send note (Sticky Note3)                                                                |

---

**Disclaimer:**  
The text analyzed and documented above comes exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.