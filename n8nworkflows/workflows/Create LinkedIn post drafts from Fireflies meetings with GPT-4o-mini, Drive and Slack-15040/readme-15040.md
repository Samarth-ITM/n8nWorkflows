Create LinkedIn post drafts from Fireflies meetings with GPT-4o-mini, Drive and Slack

https://n8nworkflows.xyz/workflows/create-linkedin-post-drafts-from-fireflies-meetings-with-gpt-4o-mini--drive-and-slack-15040


# Create LinkedIn post drafts from Fireflies meetings with GPT-4o-mini, Drive and Slack

# Workflow Reference: LinkedIn Post Generator from Fireflies Meetings

## 1. Workflow Overview
This workflow automates the transformation of business meeting transcripts into professional LinkedIn posts. It triggers automatically when a Fireflies.ai transcription is completed, processes the transcript data, utilizes AI to draft a high-engagement post, and delivers the result to both Google Drive and Slack for review.

The logic is divided into four primary functional blocks:
- **1.1 Webhook Receipt and Validation:** Captures the event from Fireflies and ensures a valid meeting ID is present.
- **1.2 Config, Transcript Fetch, and Processing:** Manages administrative settings, retrieves the full GraphQL transcript, and cleans the data for AI consumption.
- **1.3 AI LinkedIn Post Writing:** Employs GPT-4o-mini with a specific ghostwriting prompt to create a structured post based on the meeting's actual content.
- **1.4 Doc Assembly and Delivery:** Formats the final output into a formal document and a Slack notification.

---

## 2. Block-by-Block Analysis

### 2.1 Webhook Receipt and Validation
**Overview:** Acts as the entry point, receiving notifications from Fireflies and validating that the payload contains the necessary identifier to proceed.

- **Nodes Involved:** 
  - `1. Webhook — Fireflies Transcript Done`
  - `2. Code — Extract Meeting ID`
  - `3. IF — Valid Meeting ID?`
  - `4. Set — Invalid Webhook Skip`
- **Node Details:**
  - **Webhook:** Listens for `POST` requests on the path `fireflies-transcript-done`.
  - **Code (Extract Meeting ID):** A JavaScript node that handles varying Fireflies payload structures (checking `body`, `data`, or `payload` keys) to find the `meetingId`. Returns an `isValid` boolean.
  - **IF Node:** Checks if `isValid` is true. If false, it routes to the skip node.
  - **Set (Invalid Webhook Skip):** Terminates the execution for invalid requests with a status message.
- **Edge Cases:** Fireflies may send webhooks for events other than "Transcription completed"; the Code node is designed to capture the `eventType` for logging.

### 2.2 Config, Transcript Fetch, and Processing
**Overview:** Centralizes user configuration and transforms raw GraphQL API data into a structured format suitable for a prompt.

- **Nodes Involved:**
  - `5. Set — Config Values`
  - `6. HTTP — Fetch Transcript`
  - `7. Code — Process Transcript Data`
  - `8. IF — Transcript Ready?`
  - `9. Set — Transcript Not Ready Skip`
- **Node Details:**
  - **Set (Config Values):** Stores global variables: API keys, Google Drive Folder IDs, Slack channels, and Author/Company branding details.
  - **HTTP Request:** Executes a GraphQL query to `https://api.fireflies.ai/graphql`. It requests the transcript, summaries, sentiment analytics, and participant lists using a Bearer token.
  - **Code (Process Transcript Data):** A complex JS node that:
    - Combines individual sentences into a `fullText` transcript.
    - Filters specific lines tagged as "pricing", "task", or "question".
    - Truncates the transcript to 5,000 characters to prevent AI token overflow.
    - Formats dates and creates a standardized document title.
  - **IF Node:** Checks the `hasData` flag to ensure the transcript is actually available before calling the AI.
- **Edge Cases:** API timeouts or transcripts not yet fully processed by Fireflies' backend may result in the "Not Ready" path.

### 2.3 AI LinkedIn Post Writing
**Overview:** Uses a Large Language Model to act as a professional ghostwriter, converting raw meeting data into a "scroll-stopping" social media post.

- **Nodes Involved:**
  - `10. AI Agent — Write LinkedIn Post`
  - `11. OpenAI — GPT-4o-mini Model`
- **Node Details:**
  - **AI Agent:** Uses a sophisticated system prompt. It defines a strict structure: Hook $\rightarrow$ Blank Line $\rightarrow$ Paragraph $\rightarrow$ Emoji-based learnings $\rightarrow$ CTA $\rightarrow$ Hashtags $\rightarrow$ Sign-off.
  - **OpenAI Model:** Configured with `gpt-4o-mini`, a temperature of `0.8` (for creativity), and a max token limit of `700`.
- **Constraints:** The prompt explicitly forbids mentioning that the meeting was recorded/transcribed and mandates a length of 180–280 words.

### 2.4 Doc Assembly and Delivery
**Overview:** Prepares the final strings for storage and notification, ensuring the user has both a permanent record and an immediate alert.

- **Nodes Involved:**
  - `12. Code — Build Doc and Slack Message`
  - `13. Google Drive — Save LinkedIn Post`
  - `14. Slack — Send Post Preview`
- **Node Details:**
  - **Code (Build Doc/Slack):** Assembles the `docContent` (containing the post and the original meeting metadata/links) and a truncated `slackMessage` (first 350 characters).
  - **Google Drive:** Uses the `create` operation to save the formatted text as a new file in the specified folder.
  - **Slack:** Sends a formatted Markdown message to the designated channel with a preview and a link to the Fireflies transcript.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 1. Webhook... | Webhook | Trigger | None | 2. Code... | Webhook Receipt and Validation |
| 2. Code... | Code | ID Extraction | 1. Webhook... | 3. IF... | Webhook Receipt and Validation |
| 3. IF... | IF | Validation Gate | 2. Code... | 5. Set (Config), 4. Set (Skip) | Webhook Receipt and Validation |
| 4. Set... | Set | Error Handler | 3. IF... | None | Webhook Receipt and Validation |
| 5. Set... | Set | Configuration | 3. IF... | 6. HTTP... | Config, Transcript Fetch, and Processing |
| 6. HTTP... | HTTP Request | Data Retrieval | 5. Set... | 7. Code... | Config, Transcript Fetch, and Processing |
| 7. Code... | Code | Data Cleaning | 6. HTTP... | 8. IF... | Config, Transcript Fetch, and Processing |
| 8. IF... | IF | Readiness Gate | 7. Code... | 10. AI Agent, 9. Set (Skip) | Config, Transcript Fetch, and Processing |
| 9. Set... | Set | Error Handler | 8. IF... | None | Config, Transcript Fetch, and Processing |
| 10. AI Agent... | AI Agent | Content Creation | 8. IF... | 12. Code... | AI LinkedIn Post Writing |
| 11. OpenAI... | Chat Model | LLM Provider | None (linked to 10) | 10. AI Agent | AI LinkedIn Post Writing |
| 12. Code... | Code | Formatting | 10. AI Agent | 13. GDrive, 14. Slack | Doc Assembly and Delivery |
| 13. Google Drive... | Google Drive | Storage | 12. Code... | None | Doc Assembly and Delivery |
| 14. Slack... | Slack | Notification | 12. Code... | None | Doc Assembly and Delivery |

---

## 4. Reproducing the Workflow from Scratch

### Step 1: Trigger and Validation
1. Create a **Webhook** node. Set path to `fireflies-transcript-done` and method to `POST`.
2. Add a **Code** node to extract the `meetingId`. Use logic to check for `raw.meetingId`, `raw.body.meetingId`, etc.
3. Add an **IF** node to check if the extracted `meetingId` exists.
4. (Optional) Connect a **Set** node to the `false` output of the IF node for logging invalid requests.

### Step 2: Configuration and Data Retrieval
5. Create a **Set** node. Add string values for: `firefliesApiKey`, `googleDriveFolderId`, `slackChannel`, `authorName`, `authorTitle`, and `companyName`. Map `meetingId` from node 2.
6. Add an **HTTP Request** node. 
   - Method: `POST`, URL: `https://api.fireflies.ai/graphql`.
   - Headers: `Authorization: Bearer {{ $json.firefliesApiKey }}`.
   - Body (JSON): A GraphQL query for `transcript(id: $id)` including `sentences`, `summary`, and `analytics`.
7. Add a **Code** node to process the response. Implement logic to concatenate `sentences`, extract `summary.keywords`, and truncate the text to 5,000 chars.
8. Add an **IF** node to verify that `hasData` is true.

### Step 3: AI Generation
9. Add an **AI Agent** node. 
   - Set prompt to the ghostwriter persona.
   - Use expressions to inject `authorName`, `overview`, `bulletPoints`, and `transcriptExcerpt`.
   - Define the strict structure (Hook, Paragraph, Emoji List, CTA, Hashtags).
10. Connect an **OpenAI Chat Model** node to the AI Agent. Select `gpt-4o-mini`, set temperature to `0.8` and Max Tokens to `700`.

### Step 4: Delivery
11. Add a **Code** node to build the final outputs.
    - Create `docContent` (Full text with metadata).
    - Create `slackMessage` (Truncated preview + link).
12. Add a **Google Drive** node. Set operation to `Create` (File). Use `docTitle` as name and `docContent` as content.
13. Add a **Slack** node. Set operation to `Post Message`. Use `slackMessage` expression.

### Credentials Required:
- **OpenAI API Key** (for GPT-4o-mini).
- **Google Drive OAuth2** (for file creation).
- **Slack OAuth2** (for channel posting).
- **Fireflies API Key** (inserted in the Config Set node).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Fireflies Webhook Setup | Go to Settings $\rightarrow$ Developer Settings $\rightarrow$ Webhooks and paste the n8n production URL. |
| Model Choice | `gpt-4o-mini` is used for a balance of cost-efficiency and high-quality reasoning for short-form content. |
| Token Management | Transcript is capped at 5,000 characters in node 7 to avoid OpenAI context window errors. |