Triage GitHub issues with Gemini AI, auto-label them, and send Slack alerts

https://n8nworkflows.xyz/workflows/triage-github-issues-with-gemini-ai--auto-label-them--and-send-slack-alerts-13874


# Triage GitHub issues with Gemini AI, auto-label them, and send Slack alerts

# 1. Workflow Overview

This workflow automates GitHub issue triage using Gemini AI. When a new GitHub issue is opened, the workflow receives the webhook payload, extracts the relevant issue fields, sends the issue content to Gemini for classification, applies labels back to the GitHub issue, posts an AI-generated comment, routes alerts to Slack depending on urgency, and logs the triaged issue to Google Sheets.

Primary use cases:
- Automating first-pass issue categorization
- Standardizing labels such as issue type and priority
- Alerting teams faster for urgent issues
- Building a lightweight issue analytics log in Sheets

## 1.1 Input Reception and Issue Field Extraction

This block receives the GitHub webhook event and normalizes the payload into a smaller, easier-to-use internal structure.

## 1.2 AI Classification with Gemini

This block sends the issue title and body to Gemini, requesting a strict JSON response containing issue type, priority, and a summary.

## 1.3 GitHub Issue Update

This block writes the AI output back into GitHub by adding labels and posting a triage comment on the issue.

## 1.4 Priority-Based Routing and Logging

This block decides whether the issue is urgent, sends a Slack alert to the appropriate destination, and appends the issue metadata to Google Sheets.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Issue Field Extraction

**Overview:**  
This block starts the workflow when GitHub sends an issue event. It extracts the specific fields needed downstream so later nodes do not have to reference the full GitHub payload repeatedly.

**Nodes Involved:**
- GitHub Issue Webhook
- Extract Issue Data

### Node: GitHub Issue Webhook

- **Type and technical role:**  
  `n8n-nodes-base.webhook`  
  Entry-point node that listens for incoming HTTP POST requests from GitHub.

- **Configuration choices:**  
  - HTTP method: `POST`
  - Path: `github-issue-webhook`
  - Webhook ID is present internally, but the functional endpoint is based on the path.

- **Key expressions or variables used:**  
  None in configuration. It exposes the raw inbound payload as `$json`.

- **Input and output connections:**  
  - Input: none
  - Output: `Extract Issue Data`

- **Version-specific requirements:**  
  Type version `2`.

- **Edge cases or potential failure types:**  
  - GitHub webhook may not be configured to send `Issues` events.
  - If GitHub sends actions other than `opened`, the workflow still accepts them because there is no filter node.
  - Missing or malformed request body would break downstream expressions.
  - No signature validation is implemented, so authenticity checking is not enforced in the workflow itself.

- **Sub-workflow reference:**  
  None.

### Node: Extract Issue Data

- **Type and technical role:**  
  `n8n-nodes-base.set`  
  Normalizes the incoming webhook payload into a concise data object.

- **Configuration choices:**  
  The node creates these fields:
  - `issue_title` from `body.issue.title`
  - `issue_body` from `body.issue.body`
  - `issue_number` from `body.issue.number`
  - `issue_url` from `body.issue.html_url`
  - `author` from `body.issue.user.login`
  - `repo_name` from `body.repository.name`
  - `repo_owner` from `body.repository.owner.login`

- **Key expressions or variables used:**  
  - `{{ $json.body.issue.title }}`
  - `{{ $json.body.issue.body }}`
  - `{{ $json.body.issue.number }}`
  - `{{ $json.body.issue.html_url }}`
  - `{{ $json.body.issue.user.login }}`
  - `{{ $json.body.repository.name }}`
  - `{{ $json.body.repository.owner.login }}`

- **Input and output connections:**  
  - Input: `GitHub Issue Webhook`
  - Output: `Classify with Gemini AI`

- **Version-specific requirements:**  
  Type version `3.4`.

- **Edge cases or potential failure types:**  
  - If the webhook payload shape changes or the request is not a GitHub issue event, expressions may resolve to `undefined`.
  - Empty issue bodies are possible on GitHub; Gemini prompt still works, but classification quality may decrease.
  - If GitHub sends payloads for edited, reopened, or labeled issues, the extracted fields will still exist, causing unintended reprocessing.

- **Sub-workflow reference:**  
  None.

---

## 2.2 AI Classification with Gemini

**Overview:**  
This block sends the issue title and body to Gemini and requests structured output. The workflow then parses the response into simple fields for later API updates and routing.

**Nodes Involved:**
- Classify with Gemini AI
- Parse AI Response

### Node: Classify with Gemini AI

- **Type and technical role:**  
  `n8n-nodes-base.httpRequest`  
  Calls the Gemini Generate Content API directly.

- **Configuration choices:**  
  - Method: `POST`
  - URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent`
  - Body format: JSON
  - Authentication: predefined credential type `googlePalmApi`
  - Prompt explicitly asks for JSON only, no markdown
  - `generationConfig.responseMimeType` is set to `application/json`
  - A JSON schema is provided requiring:
    - `type`
    - `priority`
    - `summary`

- **Key expressions or variables used:**  
  The prompt interpolates:
  - `{{ $json.issue_title }}`
  - `{{ $json.issue_body }}`

- **Input and output connections:**  
  - Input: `Extract Issue Data`
  - Output: `Parse AI Response`

- **Version-specific requirements:**  
  Type version `4.2`.  
  Requires the Google PaLM / Gemini credential type supported by the installed n8n version.

- **Edge cases or potential failure types:**  
  - Invalid or missing Gemini API credential.
  - Model availability or API quota/rate-limit issues.
  - Long issue content may exceed request size or token limits.
  - Gemini may still return unexpected output despite schema guidance.
  - Network timeouts or API transient errors.
  - If the issue body contains unusual characters, the request still usually succeeds, but prompt fidelity may vary.

- **Sub-workflow reference:**  
  None.

### Node: Parse AI Response

- **Type and technical role:**  
  `n8n-nodes-base.set`  
  Parses the Gemini text response into structured fields used by GitHub, Slack, and Google Sheets.

- **Configuration choices:**  
  It creates:
  - `ai_type`
  - `ai_priority`
  - `ai_summary`
  It keeps previous fields because `includeOtherFields` is enabled.

- **Key expressions or variables used:**  
  - `{{ JSON.parse($json.candidates[0].content.parts[0].text).type }}`
  - `{{ JSON.parse($json.candidates[0].content.parts[0].text).priority }}`
  - `{{ JSON.parse($json.candidates[0].content.parts[0].text).summary }}`

- **Input and output connections:**  
  - Input: `Classify with Gemini AI`
  - Output: `Add Labels to Issue`

- **Version-specific requirements:**  
  Type version `3.4`.

- **Edge cases or potential failure types:**  
  - If `candidates[0]` is absent, expressions fail.
  - If Gemini returns non-JSON or invalid JSON, `JSON.parse` throws an error.
  - If the API response structure changes, parsing fails.
  - If any expected property is missing, downstream nodes may generate malformed labels or comments.

- **Sub-workflow reference:**  
  None.

---

## 2.3 GitHub Issue Update

**Overview:**  
This block pushes the AI triage results back into GitHub. It first adds labels derived from the classification, then posts a human-readable AI triage comment to the issue.

**Nodes Involved:**
- Add Labels to Issue
- Post AI Comment

### Node: Add Labels to Issue

- **Type and technical role:**  
  `n8n-nodes-base.httpRequest`  
  Calls the GitHub Issues Labels API to attach labels to the issue.

- **Configuration choices:**  
  - Method: `POST`
  - URL dynamically built from owner, repo, and issue number
  - Authentication: generic credential type using HTTP Header Auth
  - Body sends two labels:
    - issue type in lowercase
    - priority label prefixed with `priority:`

- **Key expressions or variables used:**  
  - URL:  
    `https://api.github.com/repos/{{ $json.repo_owner }}/{{ $json.repo_name }}/issues/{{ $json.issue_number }}/labels`
  - JSON body:  
    `{{ $json.ai_type.toLowerCase() }}`  
    `{{ $json.ai_priority.toLowerCase() }}`

- **Input and output connections:**  
  - Input: `Parse AI Response`
  - Output: `Post AI Comment`

- **Version-specific requirements:**  
  Type version `4.2`.

- **Edge cases or potential failure types:**  
  - Missing or invalid GitHub personal access token.
  - Token lacks permission to write issue labels.
  - Repository owner or name expressions resolve incorrectly.
  - Some label names may not exist in the repository; GitHub may create them automatically in some contexts depending on endpoint behavior and permissions, but not always as expected in all governance setups.
  - Spaces and capitalization in labels are transformed; for example `Feature Request` becomes `feature request`, which may not match an existing repository label convention.
  - Rate limits on the GitHub API.

- **Sub-workflow reference:**  
  None.

### Node: Post AI Comment

- **Type and technical role:**  
  `n8n-nodes-base.httpRequest`  
  Creates an issue comment summarizing the AI classification.

- **Configuration choices:**  
  - Method: `POST`
  - URL dynamically targets the issue comments endpoint
  - Authentication: generic HTTP Header Auth
  - Comment body includes:
    - Type
    - Priority
    - Summary
    - A note that the content was auto-generated

- **Key expressions or variables used:**  
  - URL:  
    `https://api.github.com/repos/{{ $json.repo_owner }}/{{ $json.repo_name }}/issues/{{ $json.issue_number }}/comments`
  - Body uses:
    - `{{ $json.ai_type }}`
    - `{{ $json.ai_priority }}`
    - `{{ $json.ai_summary }}`

- **Input and output connections:**  
  - Input: `Add Labels to Issue`
  - Output: `Is Critical or High?`

- **Version-specific requirements:**  
  Type version `4.2`.

- **Edge cases or potential failure types:**  
  - Authentication or permission issues on GitHub.
  - Posting can fail if issue comments are restricted by repository policy or token scope.
  - Special characters in AI summary may require correct JSON escaping; n8n usually handles this, but malformed upstream values can still be problematic.
  - If label creation succeeded but comment creation fails, the workflow leaves GitHub partially updated.

- **Sub-workflow reference:**  
  None.

---

## 2.4 Priority-Based Routing and Logging

**Overview:**  
This block checks issue priority and routes it to one of two Slack destinations. Regardless of branch, the issue is also appended to Google Sheets for later analysis.

**Nodes Involved:**
- Is Critical or High?
- Alert Urgent Issues
- Log to Issue Tracker
- Log to Google Sheets

### Node: Is Critical or High?

- **Type and technical role:**  
  `n8n-nodes-base.if`  
  Branches the workflow based on priority.

- **Configuration choices:**  
  - Condition type: string equals
  - Case-sensitive: false
  - Current condition only checks whether `ai_priority` equals `Critical`
  - Combinator is `or`, but there is only one condition configured

- **Key expressions or variables used:**  
  - `{{ $json.ai_priority }}`

- **Input and output connections:**  
  - Input: `Post AI Comment`
  - True output: `Alert Urgent Issues` and `Log to Google Sheets`
  - False output: `Log to Issue Tracker` and `Log to Google Sheets`

- **Version-specific requirements:**  
  Type version `2`.

- **Edge cases or potential failure types:**  
  - The node name and sticky note imply `Critical or High`, but configuration checks only `Critical`. High-priority issues currently go to the non-urgent branch.
  - If Gemini returns `critical`, `CRITICAL`, or variant spellings, case-insensitive mode helps, but unexpected values still route to false.
  - If `ai_priority` is missing, the item goes to the false branch.

- **Sub-workflow reference:**  
  None.

### Node: Alert Urgent Issues

- **Type and technical role:**  
  `n8n-nodes-base.slack`  
  Sends an urgent Slack message for issues classified as urgent.

- **Configuration choices:**  
  - Sends a formatted message containing:
    - issue type
    - repository
    - priority
    - issue title
    - author
    - AI analysis
    - issue URL
  - The node has a `webhookId` internally, but message sending is via Slack integration settings in the node.

- **Key expressions or variables used:**  
  - `{{ $json.ai_type }}`
  - `{{ $json.repo_name }}`
  - `{{ $json.ai_priority }}`
  - `{{ $json.issue_title }}`
  - `{{ $json.author }}`
  - `{{ $json.ai_summary }}`
  - `{{ $json.issue_url }}`

- **Input and output connections:**  
  - Input: `Is Critical or High?` true branch
  - Output: none

- **Version-specific requirements:**  
  Type version `2.2`.

- **Edge cases or potential failure types:**  
  - Slack authentication may be incomplete depending on node setup.
  - Channel destination is not explicitly shown in the exported parameters, so the connected Slack app configuration must support the intended posting behavior.
  - Slack formatting may degrade if fields are empty.
  - If this branch fails, logging to Sheets may still proceed independently because both nodes are connected in parallel from the IF node.

- **Sub-workflow reference:**  
  None.

### Node: Log to Issue Tracker

- **Type and technical role:**  
  `n8n-nodes-base.slack`  
  Sends a standard informational Slack message for non-urgent issues.

- **Configuration choices:**  
  - Authentication: OAuth2
  - Uses Slack credentials named `Slack account`
  - Message includes:
    - type
    - priority
    - issue title
    - author
    - summary
    - issue URL

- **Key expressions or variables used:**  
  - `{{ $json.ai_type }}`
  - `{{ $json.ai_priority }}`
  - `{{ $json.issue_title }}`
  - `{{ $json.author }}`
  - `{{ $json.ai_summary }}`
  - `{{ $json.issue_url }}`

- **Input and output connections:**  
  - Input: `Is Critical or High?` false branch
  - Output: none

- **Version-specific requirements:**  
  Type version `2.2`.

- **Edge cases or potential failure types:**  
  - OAuth2 token expiry or revoked Slack access.
  - Channel routing is not explicit in the exported parameters; posting destination must be validated in the node UI.
  - Missing text fields reduce message usefulness.
  - High-priority issues currently land here because the IF node does not test for `High`.

- **Sub-workflow reference:**  
  None.

### Node: Log to Google Sheets

- **Type and technical role:**  
  `n8n-nodes-base.googleSheets`  
  Appends every triaged issue to a spreadsheet row.

- **Configuration choices:**  
  - Operation: `append`
  - Sheet name: `Sheet1`
  - Mapping mode: define fields explicitly
  - Columns mapped:
    - `URL`
    - `Date`
    - `Type`
    - `Issue`
    - `Author`
    - `Summary`
    - `Priority`
    - `Repository`
  - Date is generated with current workflow time using ISO format
  - `documentId` is empty in the provided workflow, so it is not yet fully configured

- **Key expressions or variables used:**  
  - `{{ $json.issue_url }}`
  - `{{ $now.toISO() }}`
  - `{{ $json.ai_type }}`
  - `{{ $json.issue_title }}`
  - `{{ $json.author }}`
  - `{{ $json.ai_summary }}`
  - `{{ $json.ai_priority }}`
  - `{{ $json.repo_name }}`

- **Input and output connections:**  
  - Input: `Is Critical or High?` true branch and false branch
  - Output: none

- **Version-specific requirements:**  
  Type version `4.5`.

- **Edge cases or potential failure types:**  
  - Spreadsheet document ID is blank, so the node will fail until a Google Sheets file is selected.
  - OAuth2 permission issues.
  - Sheet name must exist exactly as `Sheet1`.
  - Column headers in the target sheet must align with mapping expectations.
  - Concurrent writes are usually handled, but large bursts may still trigger rate limits or retries.

- **Sub-workflow reference:**  
  None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | `n8n-nodes-base.stickyNote` | Canvas documentation / overview note |  |  | ### GitHub issue triage with Gemini AI<br>Automatically classifies new issues, adds labels, and routes alerts by priority.<br><br>### How it works<br>1. A **webhook** fires when a new issue is opened on your repo<br>2. **Gemini AI** reads the title and body, then classifies the type (Bug, Feature, Question, etc.) and priority (Critical → Low)<br>3. Labels are added to the issue via the GitHub API, and an AI summary is posted as a comment<br>4. A **Switch** routes Critical/High to #urgent-issues and Medium/Low to #issue-tracker on Slack<br>5. Every triaged issue is logged to **Google Sheets** for tracking<br><br>### Setup steps<br>1. Set the webhook URL in your GitHub repo → Settings → Webhooks → select "Issues" events<br>2. Add your Gemini API key as HTTP Header Auth credentials (`x-goog-api-key`)<br>3. Create a GitHub personal access token with `issues:write` scope and add as HTTP Header Auth<br>4. Connect Slack and Google Sheets OAuth2 credentials<br>5. Create Slack channels: `#urgent-issues` and `#issue-tracker`<br><br>### Customization<br>- Edit the categories in the Gemini prompt to fit your project<br>- Change the priority threshold in the Switch node<br>- Add more Slack channels for different teams |
| Sticky Note1 | `n8n-nodes-base.stickyNote` | Canvas documentation for intake block |  |  | ## Capture new issues<br>Webhook receives the GitHub event payload. The Set node picks out the fields we actually need for classification. |
| Sticky Note2 | `n8n-nodes-base.stickyNote` | Canvas documentation for AI block |  |  | ## AI classification<br>Gemini reads the issue and returns type, priority, and a one-line summary as JSON. The next node parses the response. |
| Sticky Note3 | `n8n-nodes-base.stickyNote` | Canvas documentation for GitHub update block |  |  | ## Update the issue<br>POSTs labels and an AI analysis comment back to GitHub via the REST API. |
| Sticky Note4 | `n8n-nodes-base.stickyNote` | Canvas documentation for routing/logging block |  |  | ## Route alerts & log<br>Critical/High → #urgent-issues, Medium/Low → #issue-tracker. Everything gets appended to Sheets for analytics. |
| GitHub Issue Webhook | `n8n-nodes-base.webhook` | Receives GitHub issue webhook events |  | Extract Issue Data | ## Capture new issues<br>Webhook receives the GitHub event payload. The Set node picks out the fields we actually need for classification. |
| Extract Issue Data | `n8n-nodes-base.set` | Extracts relevant issue and repository fields | GitHub Issue Webhook | Classify with Gemini AI | ## Capture new issues<br>Webhook receives the GitHub event payload. The Set node picks out the fields we actually need for classification. |
| Classify with Gemini AI | `n8n-nodes-base.httpRequest` | Calls Gemini to classify issue type and priority | Extract Issue Data | Parse AI Response | ## AI classification<br>Gemini reads the issue and returns type, priority, and a one-line summary as JSON. The next node parses the response. |
| Parse AI Response | `n8n-nodes-base.set` | Parses Gemini JSON output into reusable fields | Classify with Gemini AI | Add Labels to Issue | ## AI classification<br>Gemini reads the issue and returns type, priority, and a one-line summary as JSON. The next node parses the response. |
| Add Labels to Issue | `n8n-nodes-base.httpRequest` | Adds AI-derived labels to the GitHub issue | Parse AI Response | Post AI Comment | ## Update the issue<br>POSTs labels and an AI analysis comment back to GitHub via the REST API. |
| Post AI Comment | `n8n-nodes-base.httpRequest` | Posts an AI-generated triage comment to GitHub | Add Labels to Issue | Is Critical or High? | ## Update the issue<br>POSTs labels and an AI analysis comment back to GitHub via the REST API. |
| Is Critical or High? | `n8n-nodes-base.if` | Routes issues based on priority | Post AI Comment | Alert Urgent Issues; Log to Google Sheets; Log to Issue Tracker; Log to Google Sheets | ## Route alerts & log<br>Critical/High → #urgent-issues, Medium/Low → #issue-tracker. Everything gets appended to Sheets for analytics. |
| Alert Urgent Issues | `n8n-nodes-base.slack` | Sends urgent Slack notification | Is Critical or High? |  | ## Route alerts & log<br>Critical/High → #urgent-issues, Medium/Low → #issue-tracker. Everything gets appended to Sheets for analytics. |
| Log to Issue Tracker | `n8n-nodes-base.slack` | Sends standard Slack notification | Is Critical or High? |  | ## Route alerts & log<br>Critical/High → #urgent-issues, Medium/Low → #issue-tracker. Everything gets appended to Sheets for analytics. |
| Log to Google Sheets | `n8n-nodes-base.googleSheets` | Appends issue triage data to spreadsheet | Is Critical or High? |  | ## Route alerts & log<br>Critical/High → #urgent-issues, Medium/Low → #issue-tracker. Everything gets appended to Sheets for analytics. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Sticky Note for the overall description.**  
   Paste this content:
   - GitHub issue triage with Gemini AI
   - Automatically classifies new issues, adds labels, and routes alerts by priority
   - Include setup steps for GitHub webhook, Gemini API key, GitHub token, Slack, and Google Sheets
   - Include customization notes for categories, routing thresholds, and Slack channels

3. **Add a Webhook node named `GitHub Issue Webhook`.**
   - Type: `Webhook`
   - HTTP Method: `POST`
   - Path: `github-issue-webhook`
   - Leave other options default unless you want authentication or response customization.
   - This becomes the workflow trigger.

4. **Configure GitHub to call this webhook.**
   - In your GitHub repository:
     - Go to **Settings → Webhooks**
     - Add a webhook pointing to your n8n webhook URL plus `/github-issue-webhook`
     - Set content type to JSON
     - Select at least the **Issues** event
   - If using production mode, use the production webhook URL from n8n.

5. **Add a Sticky Note above the intake area.**  
   Suggested content:  
   `## Capture new issues`  
   `Webhook receives the GitHub event payload. The Set node picks out the fields we actually need for classification.`

6. **Add a Set node named `Extract Issue Data`.**
   - Connect `GitHub Issue Webhook` → `Extract Issue Data`
   - Create these fields:
     - `issue_title` as string: `{{ $json.body.issue.title }}`
     - `issue_body` as string: `{{ $json.body.issue.body }}`
     - `issue_number` as number: `{{ $json.body.issue.number }}`
     - `issue_url` as string: `{{ $json.body.issue.html_url }}`
     - `author` as string: `{{ $json.body.issue.user.login }}`
     - `repo_name` as string: `{{ $json.body.repository.name }}`
     - `repo_owner` as string: `{{ $json.body.repository.owner.login }}`

7. **Add a Sticky Note above the AI area.**  
   Suggested content:  
   `## AI classification`  
   `Gemini reads the issue and returns type, priority, and a one-line summary as JSON. The next node parses the response.`

8. **Create Gemini credentials in n8n.**
   - Use the credential type supported by your n8n version for Google PaLM / Gemini, referenced by this workflow as `googlePalmApi`.
   - The note says to use header auth with `x-goog-api-key`; in practice, configure the supported Gemini credential in n8n so the HTTP Request node can authenticate properly.
   - Ensure the API key has access to the Gemini model endpoint.

9. **Add an HTTP Request node named `Classify with Gemini AI`.**
   - Connect `Extract Issue Data` → `Classify with Gemini AI`
   - Method: `POST`
   - URL:  
     `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent`
   - Authentication: predefined credential type using your Gemini credential
   - Send Body: enabled
   - Body Content Type: JSON
   - JSON body:
     - Include `contents.parts.text` asking Gemini to classify the issue
     - Include the title and body from the previous node
     - Request JSON only
     - Set `generationConfig.responseMimeType` to `application/json`
     - Provide a schema requiring:
       - `type`
       - `priority`
       - `summary`

10. **Use this prompt logic inside the Gemini request body.**
    - Ask for:
      - `type`: one of `Bug`, `Feature Request`, `Documentation`, `Enhancement`, `Question`
      - `priority`: one of `Critical`, `High`, `Medium`, `Low`
      - `summary`: one sentence analysis
    - Inject:
      - `{{ $json.issue_title }}`
      - `{{ $json.issue_body }}`

11. **Add a Set node named `Parse AI Response`.**
    - Connect `Classify with Gemini AI` → `Parse AI Response`
    - Enable keeping previous fields (`includeOtherFields`)
    - Add:
      - `ai_type` = `{{ JSON.parse($json.candidates[0].content.parts[0].text).type }}`
      - `ai_priority` = `{{ JSON.parse($json.candidates[0].content.parts[0].text).priority }}`
      - `ai_summary` = `{{ JSON.parse($json.candidates[0].content.parts[0].text).summary }}`

12. **Add a Sticky Note above the GitHub update area.**  
   Suggested content:  
   `## Update the issue`  
   `POSTs labels and an AI analysis comment back to GitHub via the REST API.`

13. **Create GitHub credentials for API access.**
    - Use HTTP Header Auth credentials.
    - Add a GitHub personal access token.
    - The note specifies `issues:write` scope.
    - In practice, ensure the token can:
      - add labels
      - create issue comments
    - Common header format:
      - `Authorization: Bearer <token>`
      - and `Accept: application/vnd.github+json` if desired

14. **Add an HTTP Request node named `Add Labels to Issue`.**
    - Connect `Parse AI Response` → `Add Labels to Issue`
    - Method: `POST`
    - Authentication: generic credential type → HTTP Header Auth
    - URL:
      `https://api.github.com/repos/{{ $json.repo_owner }}/{{ $json.repo_name }}/issues/{{ $json.issue_number }}/labels`
    - JSON body:
      - `labels` array containing:
        - `{{ $json.ai_type.toLowerCase() }}`
        - `priority:{{ $json.ai_priority.toLowerCase() }}`

15. **Add an HTTP Request node named `Post AI Comment`.**
    - Connect `Add Labels to Issue` → `Post AI Comment`
    - Method: `POST`
    - Authentication: generic credential type → HTTP Header Auth
    - URL:
      `https://api.github.com/repos/{{ $json.repo_owner }}/{{ $json.repo_name }}/issues/{{ $json.issue_number }}/comments`
    - JSON body should set `body` to a formatted message including:
      - Type
      - Priority
      - Summary
      - Auto-generated notice

16. **Add a Sticky Note above the routing area.**  
   Suggested content:  
   `## Route alerts & log`  
   `Critical/High → #urgent-issues, Medium/Low → #issue-tracker. Everything gets appended to Sheets for analytics.`

17. **Add an IF node named `Is Critical or High?`.**
    - Connect `Post AI Comment` → `Is Critical or High?`
    - Create a string condition:
      - Left value: `{{ $json.ai_priority }}`
      - Operator: equals
      - Right value: `Critical`
    - Set case sensitivity to false

18. **Important reproduction note:**  
    The node name suggests it should match both `Critical` and `High`, but the provided workflow only checks `Critical`.  
    - If you want to reproduce the workflow exactly, keep only `Critical`.
    - If you want behavior aligned with the label and notes, add a second OR condition for `High`.

19. **Create Slack credentials in n8n.**
    - Use Slack OAuth2 credentials.
    - Connect the Slack workspace where alerts should be posted.
    - Ensure the app has permission to post messages to the intended channels.
    - Create or confirm channels:
      - `#urgent-issues`
      - `#issue-tracker`

20. **Add a Slack node named `Alert Urgent Issues`.**
    - Connect from the **true** output of `Is Critical or High?`
    - Configure it to send a message with:
      - urgent indicator
      - type
      - repository
      - priority
      - issue title
      - author
      - AI summary
      - issue link
    - Make sure the channel is set to your urgent destination in the node UI, since the exported JSON does not explicitly show the channel parameter.

21. **Add a Slack node named `Log to Issue Tracker`.**
    - Connect from the **false** output of `Is Critical or High?`
    - Use Slack OAuth2 authentication
    - Configure a standard triage message with:
      - type
      - priority
      - issue title
      - author
      - summary
      - link
    - Set the channel to your standard issue-tracking destination in the node UI.

22. **Create Google Sheets credentials in n8n.**
    - Use Google OAuth2 credentials with spreadsheet access.
    - Prepare a spreadsheet with a sheet named `Sheet1`.
    - Add these column headers in the first row:
      - `Date`
      - `Repository`
      - `Issue`
      - `Author`
      - `Type`
      - `Priority`
      - `Summary`
      - `URL`

23. **Add a Google Sheets node named `Log to Google Sheets`.**
    - Connect it from both outputs of `Is Critical or High?`
      - true branch → `Log to Google Sheets`
      - false branch → `Log to Google Sheets`
    - Operation: `Append`
    - Select your spreadsheet document
    - Sheet name: `Sheet1`
    - Map fields:
      - `URL` = `{{ $json.issue_url }}`
      - `Date` = `{{ $now.toISO() }}`
      - `Type` = `{{ $json.ai_type }}`
      - `Issue` = `{{ $json.issue_title }}`
      - `Author` = `{{ $json.author }}`
      - `Summary` = `{{ $json.ai_summary }}`
      - `Priority` = `{{ $json.ai_priority }}`
      - `Repository` = `{{ $json.repo_name }}`

24. **Fill in the missing Google Sheets document selection.**
    - The exported workflow has an empty document ID.
    - You must explicitly choose the spreadsheet in the node; otherwise the workflow will fail.

25. **Test the workflow with a sample GitHub issue.**
    - Open a new issue in the target repository.
    - Confirm:
      - webhook receives the payload
      - Gemini returns valid JSON
      - labels are added
      - AI comment appears on the issue
      - the correct Slack branch runs
      - a row is appended to the spreadsheet

26. **Optionally harden the workflow before production use.**
    - Add an IF node after the webhook to process only `action = opened`
    - Add error handling for invalid Gemini output
    - Add retries or continue-on-fail behavior for Slack and Sheets
    - Normalize labels to repository conventions
    - Add GitHub webhook signature validation

**Sub-workflow setup:**  
This workflow does **not** use any sub-workflow or Execute Workflow node. No sub-workflow configuration is required.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| GitHub issue triage with Gemini AI: automatically classifies new issues, adds labels, and routes alerts by priority. | Overall workflow purpose |
| Setup note: configure your GitHub repository webhook under Settings → Webhooks and subscribe to Issues events. | GitHub repository configuration |
| Setup note: add Gemini API access in n8n credentials; the canvas note mentions header auth with `x-goog-api-key`. | Gemini authentication |
| Setup note: create a GitHub personal access token with issue-writing permissions and use it in HTTP Header Auth credentials. | GitHub API authentication |
| Setup note: connect Slack and Google Sheets OAuth2 credentials. | External service setup |
| Setup note: create Slack channels `#urgent-issues` and `#issue-tracker`. | Slack workspace preparation |
| Customization note: edit the Gemini prompt categories to match your own project taxonomy. | AI classification design |
| Customization note: change the priority threshold in the routing node if you want different escalation behavior. | Priority routing |
| Customization note: add more Slack channels for team-specific routing if needed. | Slack extension |
| Important implementation mismatch: the notes describe a “Switch” routing Critical/High versus Medium/Low, but the actual workflow uses an IF node and currently checks only `Critical`. | Behavior discrepancy to review |
| Important implementation gap: Google Sheets logging is not fully configured because the spreadsheet document ID is blank. | Must be completed before use |