Send Slack notifications when a new release is published for public Github repos

https://n8nworkflows.xyz/workflows/send-slack-notifications-when-a-new-release-is-published-for-public-github-repos-2353


# Send Slack notifications when a new release is published for public Github repos

---
### 1. Workflow Overview

This workflow automates the monitoring of public GitHub repositories for new releases and sends notifications to a Slack channel when a new release is detected. It is designed for users who want to keep track of updates on multiple GitHub repositories without manual checking.

**Logical Blocks:**

- **1.1 Scheduled Trigger:** Initiates the workflow execution daily.
- **1.2 Repository Configuration:** Defines the list of GitHub repositories to monitor.
- **1.3 Fetch Latest Release:** Retrieves the latest release information for each configured repository.
- **1.4 Release Recency Check:** Determines if the fetched release was published within the last 24 hours.
- **1.5 Slack Notification:** Sends a formatted Slack message announcing new releases.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  This block initiates the workflow once every day, serving as the entry point for periodic execution.

- **Nodes Involved:**  
  - Daily Trigger

- **Node Details:**

  - **Daily Trigger**  
    - **Type & Role:** Schedule Trigger — starts the workflow on a daily basis.  
    - **Configuration:** Set to trigger daily using default interval settings.  
    - **Expressions/Variables:** None.  
    - **Input/Output:** No input; outputs a single execution trigger.  
    - **Version Requirements:** n8n v1.2 or later recommended for Schedule Trigger improvements.  
    - **Edge Cases:** Workflow will not run if n8n service is down during trigger time; no retries on missed executions.

---

#### 1.2 Repository Configuration

- **Overview:**  
  Defines the GitHub repositories to check. Provides a JSON array with organization and repository names used downstream.

- **Nodes Involved:**  
  - RepoConfig  
  - Sticky Note (Setup instruction note)

- **Node Details:**

  - **RepoConfig**  
    - **Type & Role:** Code node — outputs a hardcoded JSON array of repo configurations.  
    - **Configuration:** JavaScript code returns an array of objects, each with `"github-org"` and `"github-repo"` keys.  
      Example entries:  
      ```json
      [
        { "github-org": "n8n-io", "github-repo": "n8n" },
        { "github-org": "home-assistant", "github-repo": "core" }
      ]
      ```  
    - **Expressions/Variables:** Used in HTTP request URL construction.  
    - **Input/Output:** No input; output is the repo array, which feeds into the HTTP Request node.  
    - **Edge Cases:** Misconfigured JSON will cause downstream failures; ensure valid JSON and correct org/repo names.  
    - **Sticky Note:** Guides user to update this JSON array to add/remove repositories.

  - **Sticky Note**  
    - **Type & Role:** Annotation — provides setup instructions for the RepoConfig node.  
    - **Content:** "Setup repos here to check releases for. Add a new json object to the array setting the org and repo, these will be used by the following nodes."

---

#### 1.3 Fetch Latest Release

- **Overview:**  
  For each repository in the configuration, fetches the most recent release data from the GitHub API.

- **Nodes Involved:**  
  - Fetch Github Repo Releases

- **Node Details:**

  - **Fetch Github Repo Releases**  
    - **Type & Role:** HTTP Request — queries GitHub REST API for the latest release of a repo.  
    - **Configuration:**  
      - HTTP Method: GET (default)  
      - URL constructed dynamically using expressions referencing current item's `"github-org"` and `"github-repo"`.  
      - URL Template: `https://api.github.com/repos/{{ $json["github-org"] }}/{{ $json["github-repo"] }}/releases/latest`  
    - **Expressions/Variables:** Uses `$json["github-org"]` and `$json["github-repo"]` from the current item in iteration.  
    - **Input/Output:** Input from `RepoConfig` node; outputs release JSON data.  
    - **Version Requirements:** Requires n8n v4.2 or later for HTTP Request node version compatibility.  
    - **Edge Cases:**  
      - GitHub API rate limiting can cause 403 errors if unauthenticated or too many requests.  
      - Repos without any releases will return 404 or empty results.  
      - Network timeouts or API downtime can cause failures.  
      - No authentication is configured — public repos only; for private repos, OAuth token needed.

---

#### 1.4 Release Recency Check

- **Overview:**  
  Filters fetched releases to only those published within the last 24 hours.

- **Nodes Involved:**  
  - Wether Release is new (If node)

- **Node Details:**

  - **Wether Release is new**  
    - **Type & Role:** If node — evaluates if the release date is within the last day.  
    - **Configuration:**  
      - Condition: release `published_at` date is after (more recent than) current time minus 1 day.  
      - Expression for left value: `={{ $json.published_at.toDateTime() }}` — converts release publish date string to DateTime.  
      - Expression for right value: `={{ DateTime.utc().minus(1, 'days') }}` — current UTC time minus 24 hours.  
      - Combines conditions with AND (only one condition here).  
    - **Expressions/Variables:** Uses `published_at` field from release JSON.  
    - **Input/Output:** Input from HTTP Request node; routes output to either Slack notification node (true branch) or no output (false branch).  
    - **Version Requirements:** Requires n8n v2 or later for advanced IF node date/time comparisons.  
    - **Edge Cases:**  
      - Releases without `published_at` field cause expression failures.  
      - Timezone issues mitigated by using UTC consistently.  
      - If release date is exactly 24 hours ago, it will not pass (strictly after).  
      - If no releases found, node does not emit true branch.

---

#### 1.5 Slack Notification

- **Overview:**  
  Sends a Slack message to a configured channel with details of the new release.

- **Nodes Involved:**  
  - Send Message  
  - Sticky Note1 (Slack setup instructions)

- **Node Details:**

  - **Send Message**  
    - **Type & Role:** Slack node — posts a message to a Slack channel notifying about the new release.  
    - **Configuration:**  
      - Channel: `#dk-test` (by name) — user must customize this to their Slack channel.  
      - Message text includes:  
        - Celebration emoji :tada:  
        - Repo name from `RepoConfig` node JSON (`github-repo`)  
        - Release name from `Fetch Github Repo Releases` node JSON (`name`)  
        - First 500 characters of the release body (description)  
        - Release URL  
      - Markdown formatting enabled (`mrkdwn: true`).  
    - **Expressions/Variables:**  
      - Repo name: `$('RepoConfig').item.json["github-repo"]`  
      - Release name: `$('Fetch Github Repo Releases').item.json["name"]`  
      - Release body snippet: `$json.body.slice(0, 500)`  
      - Release URL: `$('Fetch Github Repo Releases').item.json["url"]`  
    - **Input/Output:** Input from If node true branch; output is Slack post confirmation.  
    - **Credentials:** Requires Slack OAuth2 credentials setup with permission to post messages.  
    - **Edge Cases:**  
      - Slack API rate limits or connectivity issues may cause failures.  
      - Invalid channel name or credentials cause errors.  
      - Release body might be null or empty — slicing will handle gracefully.  
    - **Sticky Note1:** Instructs user to customize Slack notification settings.

  - **Sticky Note1**  
    - **Type & Role:** Annotation — guides user on updating the Slack notification node.  
    - **Content:** "Setup Slack notification\n\nUpdate this node to customise your Slack notification."

---

### 3. Summary Table

| Node Name                 | Node Type             | Functional Role                | Input Node(s)       | Output Node(s)             | Sticky Note                                                                                       |
|---------------------------|-----------------------|------------------------------|---------------------|----------------------------|-------------------------------------------------------------------------------------------------|
| Daily Trigger             | Schedule Trigger      | Starts workflow daily         | —                   | RepoConfig                 |                                                                                                 |
| RepoConfig                | Code                  | Defines repo list             | Daily Trigger       | Fetch Github Repo Releases | Setup repos here to check releases for. Add a new json object to the array setting the org and repo, these will be used by the following nodes |
| Fetch Github Repo Releases | HTTP Request          | Fetches latest release info  | RepoConfig          | Wether Release is new       |                                                                                                 |
| Wether Release is new     | If                    | Checks release recency       | Fetch Github Repo Releases | Send Message           |                                                                                                 |
| Send Message              | Slack                 | Sends Slack notification     | Wether Release is new | —                         | Setup Slack notification. Update this node to customise your Slack notification                  |
| Sticky Note               | Sticky Note           | Instruction for RepoConfig   | —                   | —                          | Setup repos here to check releases for. Add a new json object to the array setting the org and repo, these will be used by the following nodes |
| Sticky Note1              | Sticky Note           | Instruction for Slack node   | —                   | —                          | Setup Slack notification. Update this node to customise your Slack notification                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**
   - Name: `Daily Trigger`
   - Type: Schedule Trigger
   - Set to trigger once every day (default daily interval).
   - Connect output to the next node.

3. **Add a Code node:**
   - Name: `RepoConfig`
   - Type: Code
   - Paste the following JavaScript code in the `Code` parameter:
     ```javascript
     return [
       { "github-org": "n8n-io", "github-repo": "n8n" },
       { "github-org": "home-assistant", "github-repo": "core" }
     ];
     ```
   - Connect input to `Daily Trigger`.
   - This node outputs the list of repositories to check.

4. **Add an HTTP Request node:**
   - Name: `Fetch Github Repo Releases`
   - Type: HTTP Request
   - HTTP Method: GET
   - URL: Use an expression with template syntax:
     ```
     https://api.github.com/repos/{{$json["github-org"]}}/{{$json["github-repo"]}}/releases/latest
     ```
   - Authentication: None (for public repos). For private repos, configure OAuth credentials for GitHub.
   - Connect input to `RepoConfig`.

5. **Add an If node:**
   - Name: `Wether Release is new`
   - Type: If
   - Set condition to check if the release's `published_at` date is after the current time minus 24 hours:
     - Condition Type: Date Time
     - Left Value: `={{ $json.published_at.toDateTime() }}`
     - Right Value: `={{ DateTime.utc().minus(1, 'days') }}`
     - Operator: After
   - Connect input to `Fetch Github Repo Releases`.

6. **Add a Slack node:**
   - Name: `Send Message`
   - Type: Slack
   - Authentication: Set up Slack OAuth2 credentials with permissions to post messages.
   - Channel: Set to the desired channel name (e.g., `#dk-test`).
   - Message Text:
     ```
     :tada: New release for *{{ $('RepoConfig').item.json["github-repo"] }}* - {{ $('Fetch Github Repo Releases').item.json["name"] }}

     {{ $json.body.slice(0, 500) }}

     {{ $('Fetch Github Repo Releases').item.json["url"] }}
     ```
   - Enable Markdown formatting.
   - Connect input from the `true` output of the `Wether Release is new` node.

7. **Optional: Add Sticky Note nodes for documentation**  
   - Add notes near `RepoConfig` and `Send Message` nodes with instructions for updating repository list and Slack notification settings.

8. **Validate the workflow by running it manually or waiting for the scheduled trigger.**

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                                                     |
|-------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Update the `RepoConfig` node JSON to include GitHub repositories you want to monitor.           | Setup instruction embedded in Sticky Note near `RepoConfig`.                                      |
| Ensure your Slack credentials have the required permissions to post messages to the target channel. | Slack OAuth2 credential configuration required for `Send Message` node.                           |
| GitHub API rate limits may apply; consider adding authentication if monitoring many repos.       | GitHub REST API documentation: https://docs.github.com/en/rest/reference/repos#releases          |
| Timezone consistency ensured by using UTC for date comparisons in the If node.                   | Leveraging n8n's DateTime functions for robust time handling.                                     |
| This workflow is designed for public repositories; private repositories require authenticated API requests. | Modify HTTP Request node to include a GitHub token in headers if necessary.                        |
| Slack message formatting uses Markdown with emojis for visual emphasis.                          | Slack message formatting guide: https://api.slack.com/reference/surfaces/formatting               |

---