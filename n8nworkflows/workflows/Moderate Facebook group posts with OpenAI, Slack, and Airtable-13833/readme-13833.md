Moderate Facebook group posts with OpenAI, Slack, and Airtable

https://n8nworkflows.xyz/workflows/moderate-facebook-group-posts-with-openai--slack--and-airtable-13833


# Moderate Facebook group posts with OpenAI, Slack, and Airtable

This reference document provides a detailed analysis of the **Facebook Group Auto-Moderation** n8n workflow.

---

### 1. Workflow Overview
The purpose of this workflow is to automate the moderation of Facebook Group posts using AI. It monitors incoming posts via webhooks, evaluates the content for policy violations (spam, scams, hate speech, etc.) using OpenAI, and takes automated actions. 

**Logical Blocks:**
*   **1.1 Data Ingestion:** Receives raw Facebook Webhook data and splits multiple changes into individual items.
*   **1.2 Pre-processing:** Normalizes data fields and iterates through posts one by one.
*   **1.3 AI Evaluation:** Analyzes the post content using a Large Language Model to determine safety and severity.
*   **1.4 Routing & Logging:** Filters posts based on the AI's verdict; safe posts are ignored, while violations are logged to Airtable and alerted on Slack.
*   **1.5 Automated Enforcement:** For high-severity violations, the workflow attempts to hide the post on Facebook via API and reports the outcome.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Ingestion & Splitting
**Overview:** Captures the webhook from Facebook and ensures that if multiple changes are sent in one payload, they are handled individually.
*   **Nodes Involved:** `Receive Facebook group`, `Extracts Posts`.
*   **Node Details:**
    *   **Receive Facebook group (Webhook):** Listens for `POST` requests at the `/fb-group-posts` path.
    *   **Extracts Posts (Split Out):** Targets the array `body.entry[0].changes` to separate individual post events.

#### 2.2 Pre-processing & Iteration
**Overview:** Cleans the incoming data for the AI and ensures a sequential processing loop.
*   **Nodes Involved:** `Process Posts`, `Normalize post data`.
*   **Node Details:**
    *   **Process Posts (Split In Batches):** Controls the flow to process posts one by one, preventing API rate limits and ensuring data integrity.
    *   **Normalize post data (Set):** Extracts `post_id`, `message`, `user_id`, `user_name`, and `created_time` into a flat structure.

#### 2.3 AI Evaluation
**Overview:** The core "brain" of the workflow that categorizes the content.
*   **Nodes Involved:** `AI Content moderation`, `Merge Ai result with post data`, `Parse moderation result`.
*   **Node Details:**
    *   **AI Content moderation (OpenAI):** Uses GPT-4-Turbo. Prompted to return a JSON object containing `violation` (boolean), `category`, `severity`, and `reason`.
    *   **Merge Ai result with post data (Merge):** Combines the AI's output with the original post metadata.
    *   **Parse moderation result (Code):** A JavaScript snippet that safely parses the AI's string response into JSON and handles parsing failures by defaulting to a "safe" status.

#### 2.4 Routing & Logging
**Overview:** Directs the workflow based on the AI's findings.
*   **Nodes Involved:** `Violations?`, `Violation alert notify`, `Log violations`.
*   **Node Details:**
    *   **Violations? (If):** Checks if the `violation` field is `true`. If `false`, it loops back to `Process Posts`.
    *   **Violation alert notify (Slack):** Sends formatted post details and the AI's reasoning to a specific Slack channel.
    *   **Log violations (Airtable):** Records the post ID, message, user info, and violation details into a tracking base.

#### 2.5 Automated Enforcement
**Overview:** Takes direct action on Facebook for serious offenses.
*   **Nodes Involved:** `Severity high?`, `Hide Facebook Post`, `Hide Post Failed`, `Auto Hide Success Alert`, `Auto hide failure alert`.
*   **Node Details:**
    *   **Severity high? (If):** Proceeds only if the AI marked the severity as "high".
    *   **Hide Facebook Post (HTTP Request):** Performs a `POST` to the Facebook Graph API (`/v19.0/{postId}`) with `is_hidden: true`. Requires an environment variable `FB_PAGE_ACCESS_TOKEN`.
    *   **Hide Post Failed (If):** Checks if the HTTP status code is NOT 200.
    *   **Success/Failure Alerts (Slack):** Notifies the team whether the automated hiding succeeded or if manual intervention is required.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Receive Facebook group | Webhook | Entry Point | - | Extracts Posts | It receives new posts made in the Facebook group using a webhook. |
| Extracts Posts | Split Out | Data Unpacking | Receive Facebook group | Process Posts | This step separates each post so they can be processed one by one. |
| Process Posts | SplitInBatches | Iteration Control | Extracts Posts, Violations?, Log violations | Normalize post data | Post Moderation Process: Automatically checks Facebook group posts one by one. |
| Normalize post data | Set | Data Mapping | Process Posts | AI Moderation, Merge | Each post is cleaned and prepared so important details are easy to work with. |
| AI Content moderation | OpenAI | AI Analysis | Normalize post data | Merge | The AI decides whether the post is safe or breaks group rules. |
| Merge Ai result | Merge | Data Join | AI Moderation, Normalize post data | Parse moderation result | - |
| Parse moderation result| Code | Data Formatting | Merge | Violations? | - |
| Violations? | If | Logic Gate | Parse moderation result | Violation alert, Process Posts | Safe posts are ignored and the workflow continues with the next post. |
| Severity high? | If | Logic Gate | Violations? | Hide Facebook Post | This process handles serious rule-breaking posts. |
| Violation alert notify| Slack | Notification | Violations? | Log violations | Sends a Slack alert to moderators with post details. |
| Log violations | Airtable | Data Logging | Violation alert notify | Process Posts | Saves the violation details in Airtable for reporting. |
| Hide Facebook Post | HTTP Request | API Enforcement | Severity high? | Hide Post Failed | The system tries to hide it automatically on Facebook. |
| Hide Post Failed | If | Error Handling | Hide Facebook Post | Auto hide failure, Auto hide success | After the request, the system checks whether the post was hidden successfully. |
| Auto Hide Success | Slack | Notification | Hide Post Failed | - | The team is then notified in Slack about the result. |
| Auto hide failure | Slack | Notification | Hide Post Failed | - | If auto-hide fails, moderators are informed to take manual action. |

---

### 4. Reproducing the Workflow from Scratch

1.  **Webhook Setup:** Create a Webhook node named `Receive Facebook group`. Set HTTP Method to `POST` and note the path.
2.  **Unpack Data:** Add a `Split Out` node to extract the `body.entry[0].changes` array.
3.  **Batching:** Connect a `Split In Batches` node to the output of the Split Out node to handle items sequentially.
4.  **Field Mapping:** Add a `Set` node. Map properties: `post_id`, `message`, `user_id`, `user_name`, and `created_time`.
5.  **AI Integration:** 
    *   Add an `OpenAI` node.
    *   Configure Model: `gpt-4-turbo`.
    *   Prompt: Instruct the AI to act as a moderator, check for spam/scams, and return a specific JSON schema (`violation`, `category`, `severity`, `reason`).
6.  **Data Merging:** Use a `Merge` node to join the AI's JSON response with the original data from the `Set` node.
7.  **Parsing:** Add a `Code` node. Use JavaScript to `JSON.parse` the AI's text output and combine it into a single object containing both post data and AI analysis.
8.  **Filtering Logic:**
    *   **If Node (Violations?):** Check if `violation` is true.
    *   **If True:** Branch to Slack and Airtable nodes.
    *   **If False:** Loop back to the `Split In Batches` node.
9.  **Alerts & Logs:**
    *   **Slack:** Configure a message with the post message and AI reason.
    *   **Airtable:** Map fields to a table named "FacebookGroup Moderation".
10. **Enforcement Logic:**
    *   **If Node (Severity high?):** Check if `severity` equals `high`.
    *   **HTTP Request:** Setup a `POST` request to `https://graph.facebook.com/v19.0/{{postId}}`. Use a Bearer Token for authorization. Set the body to `{"is_hidden": true}`.
    *   **Error Handling:** Use an `If` node to check if the `statusCode` is 200. Send a Slack alert for success or failure accordingly.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Facebook API Requirement** | Requires a valid Page Access Token stored in `FB_PAGE_ACCESS_TOKEN` environment variable. |
| **Airtable Schema** | The table should contain fields: User Name, User Id, Post Id, Message, Category, Severity, Reason. |
| **Batch Processing** | This workflow is designed for high-volume groups; the "Split in Batches" node is critical to avoid AI rate limits. |