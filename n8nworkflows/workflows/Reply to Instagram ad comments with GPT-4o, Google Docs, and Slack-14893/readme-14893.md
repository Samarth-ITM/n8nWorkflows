Reply to Instagram ad comments with GPT-4o, Google Docs, and Slack

https://n8nworkflows.xyz/workflows/reply-to-instagram-ad-comments-with-gpt-4o--google-docs--and-slack-14893


# Reply to Instagram ad comments with GPT-4o, Google Docs, and Slack

# Workflow Documentation: Reply to Instagram Ad Comments with GPT-4o

### 1. Workflow Overview

This workflow automates the management of comments on Instagram Advertisements. It uses a combination of the Facebook Graph API, LLMs (via OpenRouter), and a Google Docs knowledge base to classify incoming comments and provide context-aware responses.

The primary goal is to automate positive engagement and lead qualification while alerting human agents to negative feedback.

**Logical Blocks:**
- **1.1 Input Reception & Initial Filtering:** Captures webhooks from Instagram and filters out noise or self-comments.
- **1.2 Data Enrichment:** Retrieves specific comment details and associated Ad content (captions/permalinks) via the Graph API.
- **1.3 AI Classification:** Uses GPT-4o to categorize the comment as a Query, Positive Review, Negative Review, or Spam.
- **1.4 Conditional Response Branching:** Routes the workflow based on the classification to either generate an automated reply or trigger a Slack alert.
- **1.5 Response Generation & Execution:** Uses a RAG (Retrieval-Augmented Generation) approach with Google Docs to answer questions or send a thank-you note, then posts the reply back to Instagram.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Initial Filtering
**Overview:** Receives the raw webhook from Meta and ensures the event is actually a comment on an Advertisement and not sent by the page owner.
- **Nodes Involved:** `Trigger on Instagram Comment`, `Filter`.
- **Node Details:**
    - **Trigger on Instagram Comment (Webhook):** Listens for POST requests from Meta.
    - **Filter (Filter):** 
        - *Conditions:* Checks if `field` equals "comments", `media_product_type` equals "AD", and the username does not equal the Page ID.
        - *Failure Case:* If the event is not an ad comment, the workflow stops here.

#### 2.2 Data Enrichment
**Overview:** Gathers the full text of the comment and the context of the ad it was posted on.
- **Nodes Involved:** `Get Commnet Details`, `If`, `Get Ad Data`.
- **Node Details:**
    - **Get Commnet Details (HTTP Request):** Calls Graph API (`/v25.0/{comment_id}`) to get `from`, `text`, and `id`.
    - **If (If):** Validates if the comment text actually exists before proceeding.
    - **Get Ad Data (HTTP Request):** Calls Graph API using the media ID to retrieve the ad `caption` and `permalink`.
        - *Authentication:* Uses `httpQueryAuth`.

#### 2.3 Data Cleaning & AI Classification
**Overview:** Standardizes the data and uses AI to determine the intent of the user.
- **Nodes Involved:** `Clean Data`, `Comment Classifier`, `Structured Output Parser`, `OpenRouter Chat Model`.
- **Node Details:**
    - **Clean Data (Set):** Maps disparate API responses into clean variables: `post_text`, `post_permalink`, `comment_text`, `comment_id`, and `commentator_nusername`.
    - **Comment Classifier (AI Agent):** A LangChain agent that reads the ad context and comment to assign one of four labels: `QUERY`, `POSITIVE_REVIEW`, `NEGATIVE_REVIEW`, or `SPAM`.
    - **Structured Output Parser:** Ensures the AI returns a valid JSON object with `classification` and `confidence` keys.
    - **OpenRouter Chat Model:** Provides the GPT-4o engine for processing.

#### 2.4 Conditional Response Branching
**Overview:** Directs the flow based on the AI's classification.
- **Nodes Involved:** `Switch`.
- **Node Details:**
    - **Switch (Switch):**
        - `POSITIVE_REVIEW` $\rightarrow$ Route to Comment Generator.
        - `QUERY` $\rightarrow$ Route to Comment Generator.
        - `NEGATIVE_REVIEW` $\rightarrow$ Route to Inform User (Slack).
        - `SPAM` $\rightarrow$ End (No action).

#### 2.5 Response Generation & Execution
**Overview:** Drafts a human-like response and posts it back to Instagram.
- **Nodes Involved:** `Comment Generator`, `Knowledge Base`, `Reply to Comment`, `Inform User`.
- **Node Details:**
    - **Comment Generator (AI Agent):** Uses a specialized system prompt to draft a reply. It adheres to language matching (Urdu/English) and a "human" tone.
    - **Knowledge Base (Google Docs Tool):** A tool attached to the agent. The agent calls this *only* if the classification is `QUERY` to find pricing, FAQs, or service details.
    - **Reply to Comment (HTTP Request):** POSTs the final text to the `/replies` endpoint of the specific comment ID via Graph API.
    - **Inform User (Slack):** Sends a high-priority alert to a specific Slack user when a `NEGATIVE_REVIEW` is detected, including the commenter's handle and text.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Trigger on Instagram Comment | Webhook | Entry point for Meta events | - | Filter | AI Instagram Comment Automation... |
| Filter | Filter | Filters for Ad comments | Trigger on Instagram Comment | Get Commnet Details | Trigger and initial filter |
| Get Commnet Details | HTTP Request | Fetches comment metadata | Filter | If | Fetch comment details |
| If | If | Validates text existence | Get Commnet Details | Get Ad Data | Evaluate conditions |
| Get Ad Data | HTTP Request | Fetches Ad caption/link | If | Clean Data | Evaluate conditions |
| Clean Data | Set | Normalizes data variables | Get Ad Data | Comment Classifier | Data cleaning and classification |
| Comment Classifier | AI Agent | Categorizes comment intent | Clean Data | Switch | Data cleaning and classification |
| Structured Output Parser | Output Parser | Forces JSON format | - | Comment Classifier | Data cleaning and classification |
| OpenRouter Chat Model | LLM | GPT-4o Processing engine | - | Classifier/Generator | Data cleaning and classification |
| Switch | Switch | Routes based on category | Comment Classifier | Generator/Slack | Conditional response branching |
| Comment Generator | AI Agent | Drafts the public reply | Switch | Reply to Comment | Conditional response branching |
| Knowledge Base | Google Docs Tool | RAG for factual answers | - | Comment Generator | Conditional response branching |
| Reply to Comment | HTTP Request | Posts reply to Instagram | Comment Generator | - | Conditional response branching |
| Inform User | Slack | Alerts staff of negative reviews | Switch | - | Conditional response branching |

---

### 4. Reproducing the Workflow from Scratch

**1. Trigger & Filter Setup**
- Create a **Webhook** node. Set method to `POST`. Copy the URL to your Meta App Dashboard.
- Create a **Filter** node. Add three conditions:
    - `body.entry[0].changes[0].value.from.username` $\neq$ `YOUR_PAGE_ID`
    - `body.entry[0].changes[0].field` $=$ `comments`
    - `body.entry[0].changes[0].value.media.media_product_type` $=$ `AD`

**2. API Data Retrieval**
- Create an **HTTP Request** node (`Get Commnet Details`). URL: `https://graph.facebook.com/v25.0/{{ $json.body.entry[0].changes[0].value.id }}`. Query parameters: `fields=from,text,id`.
- Create an **If** node to check if `{{ $json.text }}` exists.
- Create an **HTTP Request** node (`Get Ad Data`). URL: `https://graph.facebook.com/v25.0/{{ $('Filter').item.json.body.entry[0].changes[0].value.media.id }}`. Query parameters: `fields=alt_text,caption,id,media_type,permalink`.

**3. Data Structuring & AI Setup**
- Create a **Set** node (`Clean Data`) to define variables: `post_text`, `post_permalink`, `comment_text`, `comment_id`, and `commentator_nusername`.
- Create an **AI Agent** (`Comment Classifier`). 
    - Connect an **OpenRouter Chat Model** (Model: `openai/gpt-4o`).
    - Connect a **Structured Output Parser** with JSON schema: `{"classification": "string", "confidence": "string"}`.
    - Input the provided system prompt specifying categories (QUERY, POSITIVE_REVIEW, NEGATIVE_REVIEW, SPAM).

**4. Routing & Response Logic**
- Create a **Switch** node. Set rules to match the output of the classifier (`$json.output.classification`).
- For the `QUERY` and `POSITIVE_REVIEW` paths:
    - Create an **AI Agent** (`Comment Generator`).
    - Connect the same **OpenRouter Chat Model**.
    - Connect a **Google Docs Tool** (`Knowledge Base`). Set the operation to `get` and provide the specific Doc URL.
    - Input the system prompt detailing the persona ("RankFlow Digital Agency") and language rules.
- For the `NEGATIVE_REVIEW` path:
    - Create a **Slack** node. Configure the message to include the commenter's name and the text of the comment.

**5. Final Execution**
- Create an **HTTP Request** node (`Reply to Comment`). 
    - Method: `POST`. 
    - URL: `https://graph.facebook.com/v25.0/{{ $('Clean Data').first().json.comment_id }}/replies`.
    - Body (form-urlencoded): `message` = `{{ $json.output }}`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Workflow handles Urdu, English, and mixed languages automatically via GPT-4o. | Language Support |
| Requires Facebook Graph API Token with `pages_manage_metadata` and `instagram_manage_comments` permissions. | API Credentials |
| The Knowledge Base is a specific Google Doc containing agency FAQs. | RAG Source |
| Uses OpenRouter as a gateway to access GPT-4o. | LLM Provider |