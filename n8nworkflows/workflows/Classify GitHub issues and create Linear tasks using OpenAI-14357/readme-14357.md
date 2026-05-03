Classify GitHub issues and create Linear tasks using OpenAI

https://n8nworkflows.xyz/workflows/classify-github-issues-and-create-linear-tasks-using-openai-14357


# Classify GitHub issues and create Linear tasks using OpenAI

# 1. Workflow Overview

This workflow monitors GitHub issue activity, uses an AI model to classify newly opened issues, creates a corresponding Linear issue, and posts a feedback comment back on GitHub.

Its main purpose is to automate issue triage for teams that receive GitHub issues and want a consistent, low-touch way to:
- classify issues as **Bug**, **Feature**, or **Question**
- assign a numeric priority
- prepare labels
- create a task in **Linear**
- acknowledge the issue publicly in **GitHub**

The workflow contains one active entry point and three main logical blocks.

## 1.1 Input Reception and Filtering
The workflow starts from a **GitHub Trigger** listening to issue-related events. It then filters events so only issues whose action is `opened` continue. After that, it extracts a small normalized payload from the GitHub webhook.

## 1.2 AI Classification and Output Normalization
The cleaned issue title and description are sent to an **Information Extractor** node backed by an **OpenAI Chat Model**. The AI returns structured JSON containing issue type, priority, and labels. A **Code** node then validates and normalizes that response.

## 1.3 Task Creation and GitHub Feedback
The original issue data and AI-enriched classification are merged. The workflow then creates a new issue in **Linear** and posts a comment on the original GitHub issue summarizing the AI triage result.

---

# 2. Block-by-Block Analysis

## 2.1 Block: Input Reception and Filtering

### Overview
This block receives GitHub webhook events, restricts processing to newly opened issues, and extracts the issue fields needed by downstream AI and task creation steps. It prevents unrelated GitHub events from reaching the rest of the flow.

### Nodes Involved
- Github Trigger
- If
- Edit Fields

### Node Details

#### Github Trigger
- **Type and technical role:** `n8n-nodes-base.githubTrigger`  
  Event-based trigger node that starts the workflow whenever configured GitHub repository events occur.
- **Configuration choices:**
  - Listens to GitHub events:
    - `issues`
    - `issue_comment`
  - Owner and repository are expected to be configured through GitHub credentials and repository selection.
- **Key expressions or variables used:**  
  Downstream nodes access webhook payload fields such as:
  - `$json.body.action`
  - `$json.body.issue.title`
  - `$json.body.issue.body`
  - `$json.body.issue.url`
  - `$json.body.issue.user.login`
  - `$json.body.issue.number`
- **Input and output connections:**
  - No input; this is the workflow entry point.
  - Output goes to **If**.
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:**
  - GitHub webhook registration/authentication issues
  - Owner/repository not configured
  - Unexpected payload shape for certain event types
  - Because `issue_comment` is also subscribed, events of that type will trigger the workflow, but the downstream `If` node filters by `action == opened`; if the payload differs, expressions may still be evaluated unexpectedly depending on event shape
- **Sub-workflow reference:** None

#### If
- **Type and technical role:** `n8n-nodes-base.if`  
  Conditional filter node used to process only newly opened issues.
- **Configuration choices:**
  - Condition checks whether `{{$json.body.action}}` equals `opened`
  - Strict type validation enabled
  - Case-sensitive comparison
- **Key expressions or variables used:**
  - `={{ $json.body.action }}`
- **Input and output connections:**
  - Input from **Github Trigger**
  - True output goes to **Edit Fields**
  - False branch is unused
- **Version-specific requirements:** `typeVersion: 2.3`
- **Edge cases or potential failure types:**
  - If the incoming event payload does not contain `body.action`, the condition may evaluate as false or behave unexpectedly
  - Since no false branch is connected, non-matching events are silently dropped
- **Sub-workflow reference:** None

#### Edit Fields
- **Type and technical role:** `n8n-nodes-base.set`  
  Field-mapping node that builds a simplified, normalized issue object for later use.
- **Configuration choices:**
  - Creates these fields:
    - `title` = GitHub issue title
    - `description` = GitHub issue body
    - `issue_url` = GitHub issue API URL
    - `author` = GitHub username
- **Key expressions or variables used:**
  - `={{ $json.body.issue.title }}`
  - `={{ $json.body.issue.body }}`
  - `={{ $json.body.issue.url }}`
  - `={{ $json.body.issue.user.login }}`
- **Input and output connections:**
  - Input from **If**
  - Output goes to:
    - **Information Extractor**
    - **Merge** on input 1
- **Version-specific requirements:** `typeVersion: 3.4`
- **Edge cases or potential failure types:**
  - Empty or null `issue.body`
  - Missing `issue.user.login`
  - `issue_url` stores the API URL, not the browser-facing HTML URL; that is acceptable here but may not be ideal for user-facing comments or links
- **Sub-workflow reference:** None

---

## 2.2 Block: AI Classification and Output Normalization

### Overview
This block sends the issue title and description to an LLM with a strict extraction schema, then sanitizes the returned structure for safe downstream usage. It ensures priority values are constrained and labels are always an array.

### Nodes Involved
- Information Extractor
- OpenAI Chat Model
- Code in JavaScript

### Node Details

#### Information Extractor
- **Type and technical role:** `@n8n/n8n-nodes-langchain.informationExtractor`  
  LangChain-based structured extraction node that prompts an LLM to classify the issue and produce machine-readable JSON.
- **Configuration choices:**
  - Prompt instructs the model to act as an AI issue triage assistant
  - Strict output requirements:
    - valid JSON only
    - no explanations
    - no markdown
  - Enforces classification schema:
    - `type`: one of `Bug | Feature | Question`
    - `priority`: number 1 to 4
    - `labels`: array of short strings
  - Manual schema is defined for:
    - `type` as string
    - `priority` as number
    - `labels` as array of strings
- **Key expressions or variables used:**
  - `{{ $json.title }}`
  - `{{ $json.description }}`
- **Input and output connections:**
  - Main input from **Edit Fields**
  - AI language model input from **OpenAI Chat Model**
  - Main output to **Code in JavaScript**
- **Version-specific requirements:** `typeVersion: 1.2`
- **Edge cases or potential failure types:**
  - Model may still return malformed JSON
  - Model may return a string instead of already parsed JSON
  - Title or description may be empty, reducing classification quality
  - If the LangChain/OpenAI integration is not available in the n8n version, this node will not function
- **Sub-workflow reference:** None

#### OpenAI Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  LLM provider node that supplies the model used by the Information Extractor.
- **Configuration choices:**
  - Model selected: `gpt-4.1-mini`
  - No additional options configured
  - No built-in tools enabled
- **Key expressions or variables used:** None
- **Input and output connections:**
  - No main input
  - AI language model output connected to **Information Extractor**
- **Version-specific requirements:** `typeVersion: 1.3`
- **Edge cases or potential failure types:**
  - Missing or invalid OpenAI credentials
  - Model unavailable in the connected OpenAI account or region
  - Token/rate limit errors
  - Provider-side timeouts
- **Sub-workflow reference:** None

#### Code in JavaScript
- **Type and technical role:** `n8n-nodes-base.code`  
  Custom normalization and safety layer for the AI output.
- **Configuration choices:**
  - Reads `let output = $json.output`
  - If `output` is a string, attempts `JSON.parse(output)`
  - Converts `output.priority` to a number
  - Applies safe fallback priority:
    - valid values: 1, 2, 3, 4
    - otherwise fallback to `3`
  - Maps numeric priority to labels:
    - 1 → Urgent
    - 2 → High
    - 3 → Medium
    - 4 → Low
  - Returns enriched JSON with:
    - `output`
    - `priority`
    - `priorityLabel`
    - `type`
    - `typeLabel`
    - `labels`
- **Key expressions or variables used:**
  - `$json.output`
  - `Number(output.priority)`
  - `Array.isArray(output.labels) ? output.labels : []`
- **Input and output connections:**
  - Input from **Information Extractor**
  - Output to **Merge** on input 0
- **Version-specific requirements:** `typeVersion: 2`
- **Edge cases or potential failure types:**
  - Throws `Invalid JSON from AI` if a string output cannot be parsed
  - Does not validate `output.type` against allowed values; invalid type could pass through
  - Labels are normalized to an empty array if malformed, but they are not used later
  - If `output` itself is missing, the script may fail when accessing `output.priority`
- **Sub-workflow reference:** None

---

## 2.3 Block: Task Creation and GitHub Feedback

### Overview
This block combines the original issue content with AI classification data, creates a new issue in Linear, and posts a confirmation comment in GitHub. It is the final execution block where external system writes occur.

### Nodes Involved
- Merge
- Create an issue
- Create a comment on an issue

### Node Details

#### Merge
- **Type and technical role:** `n8n-nodes-base.merge`  
  Combines the AI-normalized item with the original issue fields into a single payload.
- **Configuration choices:**
  - Mode: `combine`
  - Combination strategy: `combineByPosition`
- **Key expressions or variables used:** None directly in configuration
- **Input and output connections:**
  - Input 0 from **Code in JavaScript**
  - Input 1 from **Edit Fields**
  - Output to **Create an issue**
- **Version-specific requirements:** `typeVersion: 3.2`
- **Edge cases or potential failure types:**
  - Combine-by-position assumes both branches emit the same number of items in the same order
  - If one branch errors or emits zero items, downstream creation will fail or not run
- **Sub-workflow reference:** None

#### Create an issue
- **Type and technical role:** `n8n-nodes-base.linear`  
  Creates a new issue/task in Linear from the merged GitHub + AI data.
- **Configuration choices:**
  - Title set from `{{$json.title}}`
  - Team ID hardcoded as placeholder: `YOUR_LINEAR_TEAM_ID`
  - Additional fields:
    - `priorityId` = `{{$json.priority}}`
    - `description` = `{{ $('Merge').item.json.description }}`
- **Key expressions or variables used:**
  - `={{$json.title}}`
  - `={{$json.priority}}`
  - `={{ $('Merge').item.json.description }}`
- **Input and output connections:**
  - Input from **Merge**
  - Output to **Create a comment on an issue**
- **Version-specific requirements:** `typeVersion: 1.1`
- **Edge cases or potential failure types:**
  - Placeholder `YOUR_LINEAR_TEAM_ID` must be replaced or the node will fail
  - Linear credentials must be configured
  - Numeric priority mapping must match what the Linear node expects in the installed version
  - The labels generated by AI are not sent to Linear, so expected label/tag automation is incomplete
  - Description may be empty if the original issue body was empty
- **Sub-workflow reference:** None

#### Create a comment on an issue
- **Type and technical role:** `n8n-nodes-base.github`  
  Writes a follow-up comment to the original GitHub issue to show that triage occurred and a Linear issue was created.
- **Configuration choices:**
  - Operation: `createComment`
  - Comment body thanks the reporter and includes:
    - AI type label
    - AI priority label
    - confirmation that a Linear task was created
  - Uses issue number from the original GitHub trigger payload
  - Owner and repository must be configured
- **Key expressions or variables used:**
  - `{{ $('Code in JavaScript').item.json.typeLabel }}`
  - `{{ $('Code in JavaScript').item.json.priorityLabel }}`
  - `={{ $('Github Trigger').item.json.body.issue.number }}`
- **Input and output connections:**
  - Input from **Create an issue**
  - No downstream output
- **Version-specific requirements:** `typeVersion: 1.1`
- **Edge cases or potential failure types:**
  - GitHub credentials missing or insufficient comment permissions
  - Owner/repository not configured
  - If Linear creation fails, this node does not run
  - If the code node output is unavailable due to prior error, expressions will fail
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Github Trigger | n8n-nodes-base.githubTrigger | Starts the workflow on GitHub issue-related events |  | If | # GitHub Issue → AI Triage & Task Creation<br>This workflow automates the process of analyzing and organizing new GitHub issues using AI. Whenever a new issue is created, the workflow evaluates its content, classifies it (e.g., Bug, Feature, or Question), and prepares structured data such as priority and labels. Based on this analysis, a task is automatically created, helping teams streamline issue management and reduce manual triage effort.<br>By leveraging AI, teams can ensure consistent classification, faster response times, and better organization of incoming issues. This is especially useful for growing projects where managing issues manually becomes time-consuming and error-prone.<br>### How it works<br>- Trigger fires when a new GitHub issue is created<br>- Basic filtering ensures only relevant issues are processed<br>- Issue content is cleaned and structured<br>- AI analyzes and classifies the issue type<br>- Code node formats the AI output<br>- Data is merged and used to create a task<br>- A comment is added back to the issue for traceability<br>### Setup<br>- Connect GitHub credentials<br>- Add AI model credentials (Groq/OpenAI)<br>- Configure classification prompt inside AI node<br>- Map fields for issue title, description, and labels<br>- Set up Linear/GitHub task creation fields properly<br>## Step 1 : Capture & Filter Issues<br>Triggers on new GitHub issues, filters it, and prepares structured data (title, description) for AI processing. |
| If | n8n-nodes-base.if | Filters for newly opened issues only | Github Trigger | Edit Fields | # GitHub Issue → AI Triage & Task Creation<br>This workflow automates the process of analyzing and organizing new GitHub issues using AI. Whenever a new issue is created, the workflow evaluates its content, classifies it (e.g., Bug, Feature, or Question), and prepares structured data such as priority and labels. Based on this analysis, a task is automatically created, helping teams streamline issue management and reduce manual triage effort.<br>By leveraging AI, teams can ensure consistent classification, faster response times, and better organization of incoming issues. This is especially useful for growing projects where managing issues manually becomes time-consuming and error-prone.<br>### How it works<br>- Trigger fires when a new GitHub issue is created<br>- Basic filtering ensures only relevant issues are processed<br>- Issue content is cleaned and structured<br>- AI analyzes and classifies the issue type<br>- Code node formats the AI output<br>- Data is merged and used to create a task<br>- A comment is added back to the issue for traceability<br>### Setup<br>- Connect GitHub credentials<br>- Add AI model credentials (Groq/OpenAI)<br>- Configure classification prompt inside AI node<br>- Map fields for issue title, description, and labels<br>- Set up Linear/GitHub task creation fields properly<br>## Step 1 : Capture & Filter Issues<br>Triggers on new GitHub issues, filters it, and prepares structured data (title, description) for AI processing. |
| Edit Fields | n8n-nodes-base.set | Extracts title, description, issue URL, and author into a simpler payload | If | Information Extractor, Merge | # GitHub Issue → AI Triage & Task Creation<br>This workflow automates the process of analyzing and organizing new GitHub issues using AI. Whenever a new issue is created, the workflow evaluates its content, classifies it (e.g., Bug, Feature, or Question), and prepares structured data such as priority and labels. Based on this analysis, a task is automatically created, helping teams streamline issue management and reduce manual triage effort.<br>By leveraging AI, teams can ensure consistent classification, faster response times, and better organization of incoming issues. This is especially useful for growing projects where managing issues manually becomes time-consuming and error-prone.<br>### How it works<br>- Trigger fires when a new GitHub issue is created<br>- Basic filtering ensures only relevant issues are processed<br>- Issue content is cleaned and structured<br>- AI analyzes and classifies the issue type<br>- Code node formats the AI output<br>- Data is merged and used to create a task<br>- A comment is added back to the issue for traceability<br>### Setup<br>- Connect GitHub credentials<br>- Add AI model credentials (Groq/OpenAI)<br>- Configure classification prompt inside AI node<br>- Map fields for issue title, description, and labels<br>- Set up Linear/GitHub task creation fields properly<br>## Step 1 : Capture & Filter Issues<br>Triggers on new GitHub issues, filters it, and prepares structured data (title, description) for AI processing. |
| Information Extractor | @n8n/n8n-nodes-langchain.informationExtractor | Uses an LLM to classify issue type, priority, and labels as structured JSON | Edit Fields, OpenAI Chat Model | Code in JavaScript | ## Step 2 : AI Classification & Formatting<br>AI classifies the issue (Bug, Feature, Question) and assigns priority. Code node formats this output for downstream task creation. |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Supplies the OpenAI model to the extraction node |  | Information Extractor | ## Step 2 : AI Classification & Formatting<br>AI classifies the issue (Bug, Feature, Question) and assigns priority. Code node formats this output for downstream task creation. |
| Code in JavaScript | n8n-nodes-base.code | Parses and normalizes AI output with fallback priority handling | Information Extractor | Merge | ## Step 2 : AI Classification & Formatting<br>AI classifies the issue (Bug, Feature, Question) and assigns priority. Code node formats this output for downstream task creation. |
| Merge | n8n-nodes-base.merge | Combines original issue fields with AI-enriched classification | Code in JavaScript, Edit Fields | Create an issue | ## Step 3 : Task Creation & Feedback<br>Processed data is merged and used to create a task. A comment is added to the GitHub issue for visibility and tracking. |
| Create an issue | n8n-nodes-base.linear | Creates a corresponding issue in Linear | Merge | Create a comment on an issue | ## Step 3 : Task Creation & Feedback<br>Processed data is merged and used to create a task. A comment is added to the GitHub issue for visibility and tracking. |
| Create a comment on an issue | n8n-nodes-base.github | Posts a confirmation comment back to the original GitHub issue | Create an issue |  | ## Step 3 : Task Creation & Feedback<br>Processed data is merged and used to create a task. A comment is added to the GitHub issue for visibility and tracking. |
| Sticky Note6 | n8n-nodes-base.stickyNote | Canvas documentation note |  |  | # GitHub Issue → AI Triage & Task Creation<br>This workflow automates the process of analyzing and organizing new GitHub issues using AI. Whenever a new issue is created, the workflow evaluates its content, classifies it (e.g., Bug, Feature, or Question), and prepares structured data such as priority and labels. Based on this analysis, a task is automatically created, helping teams streamline issue management and reduce manual triage effort.<br>By leveraging AI, teams can ensure consistent classification, faster response times, and better organization of incoming issues. This is especially useful for growing projects where managing issues manually becomes time-consuming and error-prone.<br>### How it works<br>- Trigger fires when a new GitHub issue is created<br>- Basic filtering ensures only relevant issues are processed<br>- Issue content is cleaned and structured<br>- AI analyzes and classifies the issue type<br>- Code node formats the AI output<br>- Data is merged and used to create a task<br>- A comment is added back to the issue for traceability<br>### Setup<br>- Connect GitHub credentials<br>- Add AI model credentials (Groq/OpenAI)<br>- Configure classification prompt inside AI node<br>- Map fields for issue title, description, and labels<br>- Set up Linear/GitHub task creation fields properly |
| Sticky Note7 | n8n-nodes-base.stickyNote | Canvas documentation note for Step 1 |  |  | ## Step 1 : Capture & Filter Issues<br>Triggers on new GitHub issues, filters it, and prepares structured data (title, description) for AI processing. |
| Sticky Note8 | n8n-nodes-base.stickyNote | Canvas documentation note for Step 2 |  |  | ## Step 2 : AI Classification & Formatting<br>AI classifies the issue (Bug, Feature, Question) and assigns priority. Code node formats this output for downstream task creation. |
| Sticky Note9 | n8n-nodes-base.stickyNote | Canvas documentation note for Step 3 |  |  | ## Step 3 : Task Creation & Feedback<br>Processed data is merged and used to create a task. A comment is added to the GitHub issue for visibility and tracking. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and give it the title:  
   **Classify GitHub issues and create Linear tasks using OpenAI**

2. **Add a GitHub Trigger node**
   - Node type: **GitHub Trigger**
   - Configure GitHub credentials with access to the target repository.
   - Select the repository owner.
   - Select the repository.
   - Under events, enable:
     - `issues`
     - `issue_comment`
   - Keep other options at default unless your environment requires specific webhook settings.

3. **Add an If node**
   - Node type: **If**
   - Connect **Github Trigger → If**
   - Create one condition:
     - Left value: `{{ $json.body.action }}`
     - Operator: `equals`
     - Right value: `opened`
   - Keep strict validation enabled if available.
   - Leave the false branch unconnected unless you want explicit handling for ignored events.

4. **Add a Set node named “Edit Fields”**
   - Node type: **Set**
   - Connect **If (true output) → Edit Fields**
   - Add these fields:
     1. `title` as string: `{{ $json.body.issue.title }}`
     2. `description` as string: `{{ $json.body.issue.body }}`
     3. `issue_url` as string: `{{ $json.body.issue.url }}`
     4. `author` as string: `{{ $json.body.issue.user.login }}`
   - Leave other options at default.

5. **Add an OpenAI Chat Model node**
   - Node type: **OpenAI Chat Model**
   - Configure OpenAI credentials.
   - Select model: `gpt-4.1-mini`
   - Leave tools disabled.
   - Leave advanced options at default unless you need temperature or token tuning.

6. **Add an Information Extractor node**
   - Node type: **Information Extractor**
   - Connect **Edit Fields → Information Extractor**
   - Connect **OpenAI Chat Model → Information Extractor** through the AI language model connection.
   - Set the extraction prompt text to:

     ```text
     You are an AI issue triage assistant.

     STRICT RULES:
     - Output ONLY valid JSON
     - No explanation
     - No markdown
     - No extra text

     Classify the issue:

     type must be one of:
     Bug | Feature | Question

     priority must be a NUMBER:
     1 = Urgent
     2 = High
     3 = Medium
     4 = Low

     Also generate 2–4 short labels.

     Return format:

     {
       "type": "Bug",
       "priority": 1,
       "labels": ["crash", "login"]
     }

     Issue Title:
     {{ $json.title }}

     Issue Description:
     {{ $json.description }}
     ```

   - Set schema mode to **manual**.
   - Define the schema with:
     - `type`: string
     - `priority`: number
     - `labels`: array of strings

7. **Add a Code node named “Code in JavaScript”**
   - Node type: **Code**
   - Connect **Information Extractor → Code in JavaScript**
   - Use JavaScript mode.
   - Paste this logic:

     ```javascript
     let output = $json.output;

     // 1. Parse if AI returned string
     if (typeof output === "string") {
       try {
         output = JSON.parse(output);
       } catch (e) {
         throw new Error("Invalid JSON from AI");
       }
     }

     // 2. Normalize values
     const priorityNum = Number(output.priority);

     // 3. Fallback safety
     const safePriority = [1,2,3,4].includes(priorityNum) ? priorityNum : 3;

     // 4. Priority label mapping
     const priorityMap = {
       1: "Urgent",
       2: "High",
       3: "Medium",
       4: "Low"
     };

     return [{
       ...$json,
       output,
       priority: safePriority,
       priorityLabel: priorityMap[safePriority],
       type: output.type,
       typeLabel: output.type,
       labels: Array.isArray(output.labels) ? output.labels : []
     }];
     ```

   - This node is important because it:
     - parses malformed stringified output
     - forces a safe priority
     - creates user-friendly labels for the GitHub comment

8. **Add a Merge node**
   - Node type: **Merge**
   - Connect:
     - **Code in JavaScript → Merge** as input 1
     - **Edit Fields → Merge** as input 2
   - Set mode to **Combine**
   - Set combine strategy to **Combine by Position**
   - This ensures AI output and original GitHub fields are available together in one item.

9. **Add a Linear node named “Create an issue”**
   - Node type: **Linear**
   - Connect **Merge → Create an issue**
   - Configure Linear credentials.
   - Choose the operation that creates an issue.
   - Set:
     - **Title** = `{{ $json.title }}`
     - **Team ID** = your actual Linear team ID
   - Under additional fields:
     - **Priority ID** = `{{ $json.priority }}`
     - **Description** = `{{ $('Merge').item.json.description }}`
   - Replace the placeholder `YOUR_LINEAR_TEAM_ID` with a valid team ID before activation.

10. **Add a GitHub node named “Create a comment on an issue”**
    - Node type: **GitHub**
    - Connect **Create an issue → Create a comment on an issue**
    - Configure GitHub credentials with permission to write issue comments.
    - Set operation to **Create Comment**
    - Set owner and repository to the same target repository used in the trigger.
    - Set issue number to:
      - `{{ $('Github Trigger').item.json.body.issue.number }}`
    - Set the comment body to:

      ```text
      Thanks for opening this issue!

      Our AI triage system classified this as:

      Type: {{ $('Code in JavaScript').item.json.typeLabel }}
      Priority: {{ $('Code in JavaScript').item.json.priorityLabel }}

      A Linear task has been created for the team.
      ```

11. **Optionally add sticky notes for maintainability**
    - Add one general note describing the full process.
    - Add one note above the trigger/filter section.
    - Add one note above the AI section.
    - Add one note above the Linear/comment section.

12. **Credential setup checklist**
    - **GitHub Trigger credentials**
      - Must allow repository webhook setup and read access to issue events
    - **GitHub node credentials**
      - Must allow issue comment creation
    - **OpenAI credentials**
      - Must allow access to `gpt-4.1-mini`
    - **Linear credentials**
      - Must allow issue creation in the chosen team

13. **Test the workflow**
    - Open a new issue in the configured GitHub repository.
    - Confirm the trigger fires.
    - Confirm the If node passes only when action is `opened`.
    - Verify the Information Extractor returns structured output.
    - Verify the Code node returns:
      - `type`
      - `priority`
      - `priorityLabel`
      - `labels`
    - Verify a Linear issue is created.
    - Verify a comment appears on the GitHub issue.

14. **Recommended hardening improvements when rebuilding**
    - Restrict the trigger to only `issues` if `issue_comment` events are not needed
    - Validate `type` in the Code node so unexpected model output cannot pass through
    - Use `issue.html_url` instead of API `url` if a human-readable issue link is needed
    - Send AI-generated labels to GitHub or Linear if label automation is part of the intended design
    - Add an error branch or Error Trigger workflow for failures in AI or Linear creation

15. **There are no sub-workflows in this design**
    - No Execute Workflow node is present
    - No external n8n workflow dependency must be created

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| GitHub Issue → AI Triage & Task Creation: This workflow automates analysis and organization of new GitHub issues using AI, classifies them, prepares priority and labels, creates a task, and comments back for traceability. | General workflow note |
| Setup notes mention connecting GitHub credentials, adding AI model credentials (Groq/OpenAI), configuring the classification prompt, mapping issue fields, and setting up Linear/GitHub task creation fields properly. | General setup guidance |
| Step 1 : Capture & Filter Issues — Triggers on new GitHub issues, filters it, and prepares structured data (title, description) for AI processing. | Input/filtering block |
| Step 2 : AI Classification & Formatting — AI classifies the issue (Bug, Feature, Question) and assigns priority. Code node formats this output for downstream task creation. | AI block |
| Step 3 : Task Creation & Feedback — Processed data is merged and used to create a task. A comment is added to the GitHub issue for visibility and tracking. | Output block |

## Additional implementation observations
- The workflow title and behavior indicate GitHub-to-Linear triage automation, but the current implementation does **not** apply AI-generated labels anywhere.
- The trigger listens to both `issues` and `issue_comment`, but only `opened` actions are processed. This is broader than necessary unless future expansion is planned.
- The comment confirms that a Linear task was created, but it does not include the created Linear issue ID or URL. Adding that would improve traceability.