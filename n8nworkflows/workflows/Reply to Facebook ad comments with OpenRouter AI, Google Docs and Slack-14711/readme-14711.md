Reply to Facebook ad comments with OpenRouter AI, Google Docs and Slack

https://n8nworkflows.xyz/workflows/reply-to-facebook-ad-comments-with-openrouter-ai--google-docs-and-slack-14711


# Reply to Facebook ad comments with OpenRouter AI, Google Docs and Slack

# Workflow Documentation: Reply to Facebook Ad Comments with OpenRouter AI, Google Docs, and Slack

### 1. Workflow Overview

This workflow is designed for digital agencies and businesses to automate the management of comments on Facebook and Instagram advertisement posts. It leverages AI to classify the sentiment and intent of comments, providing automated helpful responses for queries and positive feedback, while alerting human operators via Slack when negative reviews are detected.

The logic is divided into five primary functional blocks:
- **1.1 Input Reception & Filtering:** Captures incoming webhooks and ensures only relevant ad comments (not from the page owner) are processed.
- **1.2 Data Retrieval:** Fetches detailed metadata for both the specific comment and the parent Facebook post.
- **1.3 AI Classification:** Analyzes the context and content to categorize the comment into specific intent types.
- **1.4 Response Orchestration:** A decision-making layer that determines whether to generate an AI reply or trigger an alert.
- **1.5 Action Execution:** Executes the final response via the Facebook API or sends a notification to Slack.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Filtering
**Overview:** Acts as the entry point, filtering out noise to ensure the AI only processes legitimate customer comments on active ads.
- **Nodes Involved:** `Webhook`, `Filleter Author Comment and Reqular Post`
- **Node Details:**
  - **Webhook:** Receives POST requests from Facebook.
  - **Filleter Author Comment and Reqular Post (Filter):** 
    - **Logic:** Validates three conditions: 1) The commenter is NOT the page owner, 2) The item is a "comment", and 3) The post promotion status is "active".
    - **Failure Type:** If any condition fails, the workflow stops here.

#### 2.2 Data Retrieval
**Overview:** Gathers the necessary text and URLs from the Facebook Graph API to provide the AI with full context.
- **Nodes Involved:** `Get Commnet Details`, `Skip If Coment Contains Attachment`, `Get FB Post Data`
- **Node Details:**
  - **Get Commnet Details (HTTP Request):** Calls `graph.facebook.com` using the `comment_id` to retrieve the message, author, and permalink.
  - **Skip If Coment Contains Attachment (Switch):** Routes the flow. If the comment contains an attachment, it is ignored; if it contains a text message, it proceeds.
  - **Get FB Post Data (HTTP Request):** Calls `graph.facebook.com` using the `post.id` to retrieve the original ad copy (message) and URL.

#### 2.3 Data Cleaning and AI Classification
**Overview:** Normalizes the gathered data and uses a Large Language Model (LLM) to determine the intent of the user.
- **Nodes Involved:** `Clean Data`, `Comment Classifier`, `OpenRouter Chat Model`, `Structured Output Parser`
- **Node Details:**
  - **Clean Data (Set):** Maps complex JSON paths to simple variables (e.g., `post_text`, `comment_text`, `comment_id`).
  - **Comment Classifier (AI Agent):** Uses a detailed system prompt to categorize comments into `QUERY`, `POSITIVE_REVIEW`, `NEGATIVE_REVIEW`, or `SPAM`.
  - **OpenRouter Chat Model:** Provides the LLM backbone for the agent.
  - **Structured Output Parser:** Forces the AI to return a strict JSON object containing `classification` and `confidence`.

#### 2.4 Processing and Response Decision
**Overview:** Directs the workflow based on the AI's classification.
- **Nodes Involved:** `Switch`
- **Node Details:**
  - **Switch:** 
    - `POSITIVE_REVIEW` $\rightarrow$ Route to Comment Generator.
    - `QUERY` $\rightarrow$ Route to Comment Generator.
    - `NEGATIVE_REVIEW` $\rightarrow$ Route to Slack Notification.
    - `SPAM` $\rightarrow$ Terminate (No action).

#### 2.5 Generate Response or Notification
**Overview:** The final stage where the AI either writes a public reply based on a knowledge base or notifies a human.
- **Nodes Involved:** `Comment Generator`, `Knowledge Base`, `Reply to Comment`, `Inform User`
- **Node Details:**
  - **Comment Generator (AI Agent):** 
    - Uses a system prompt to craft human-like replies.
    - Logic: If the category is `QUERY`, it triggers the `Knowledge Base` tool. If it's `POSITIVE_REVIEW`, it replies warmly without tools.
  - **Knowledge Base (Google Docs Tool):** Retrieves specific business information (pricing, FAQs) from a designated Google Doc.
  - **Reply to Comment (HTTP Request):** Sends a POST request to the Facebook Graph API to publish the generated text as a reply.
  - **Inform User (Slack):** Sends a high-priority alert to a Slack channel with the commenter's name, the comment text, and links to the post for manual intervention.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Webhook | Webhook | Trigger | - | Filleter Author... | Initial trigger and filter |
| Filleter Author... | Filter | Validation | Webhook | Get Commnet Details | Initial trigger and filter |
| Get Commnet Details | HTTP Request | Data Fetching | Filleter Author... | Skip If Coment... | Fetch Facebook details |
| Skip If Coment... | Switch | Logic Gate | Get Commnet Details | Get FB Post Data | Fetch Facebook details |
| Get FB Post Data | HTTP Request | Data Fetching | Skip If Coment... | Clean Data | Fetch Facebook details |
| Clean Data | Set | Data Normalization | Get FB Post Data | Comment Classifier | Data cleaning and AI classification |
| Comment Classifier | AI Agent | Sentiment Analysis | Clean Data | Switch | Data cleaning and AI classification |
| OpenRouter Chat Model | AI Model | LLM Provider | - | Classifier / Generator | - |
| Structured Output Parser | Output Parser | JSON Enforcement | - | Comment Classifier | - |
| Switch | Switch | Logic Router | Comment Classifier | Generator / Inform User | processing and response decision |
| Comment Generator | AI Agent | Response Author | Switch | Reply to Comment | Generate response or notification |
| Knowledge Base | Google Docs | RAG Tool | - | Comment Generator | Generate response or notification |
| Reply to Comment | HTTP Request | Action | Comment Generator | - | Generate response or notification |
| Inform User | Slack | Alerting | Switch | - | Generate response or notification |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Trigger and Initial Filtering
1. Create a **Webhook** node (Method: POST).
2. Create a **Filter** node. Configure it to check:
   - `body.entry[0].changes[0].value.from.name` $\neq$ [Your Page Name]
   - `body.entry[0].changes[0].value.item` $=$ `comment`
   - `body.entry[0].changes[0].value.post.promotion_status` $=$ `active`

#### Step 2: Data Collection
1. Add an **HTTP Request** node to call `https://graph.facebook.com/v25.0/{{$json.body.entry[0].changes[0].value.comment_id}}`. Query parameters: `fields=message,attachment,from,permalink_url`.
2. Add a **Switch** node to check if `attachment` exists. If yes $\rightarrow$ stop; if `message` exists $\rightarrow$ proceed.
3. Add another **HTTP Request** node to call `https://graph.facebook.com/v25.0/{{$json.body.entry[0].changes[0].value.post.id}}`. Query parameters: `fields=message,permalink_url`.

#### Step 3: Data Normalization and Classification
1. Add a **Set** node to create variables: `post_text`, `post_permalink_url`, `comment_text`, `comment_permalink_url`, `comment_id`, and `commentator_name`.
2. Add an **AI Agent** node ("Comment Classifier"). 
   - Connect an **OpenRouter Chat Model**.
   - Connect a **Structured Output Parser** with a JSON schema for `classification` and `confidence`.
   - Set the system prompt to define categories: `QUERY`, `POSITIVE_REVIEW`, `NEGATIVE_REVIEW`, and `SPAM`.

#### Step 4: Routing and Action
1. Add a **Switch** node. Create four outputs based on the `classification` value.
2. For `POSITIVE_REVIEW` and `QUERY`: 
   - Connect to another **AI Agent** ("Comment Generator").
   - Attach a **Google Docs Tool** as a tool for the agent to access a specific Doc ID.
   - Set the system prompt to handle language matching and brand voice.
   - Connect the agent's output to an **HTTP Request** node (POST to `.../comment_id/comments`) with the `message` body.
3. For `NEGATIVE_REVIEW`:
   - Connect to a **Slack** node. Use expressions to include the commenter's name and the post link.
4. For `SPAM`:
   - Leave unconnected.

#### Step 5: Credentials Configuration
- **Facebook Graph API:** Use "Query Auth" or "OAuth2" credentials.
- **OpenRouter:** API Key.
- **Google Docs:** OAuth2 credentials for the specific document.
- **Slack:** Slack API Token/Webhook.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Auto-Reply to Facebook Ad Comments with AI - Built for Agencies & Businesses | General Workflow Purpose |
| Supports Urdu, English, and mixed language responses | Language Capability |
| RAG Implementation via Google Docs for FAQ/Pricing | Knowledge Base Logic |
| High-priority human intervention for negative sentiment | Quality Control Strategy |