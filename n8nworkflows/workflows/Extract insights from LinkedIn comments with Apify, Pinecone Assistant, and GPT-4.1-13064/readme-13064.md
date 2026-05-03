Extract insights from LinkedIn comments with Apify, Pinecone Assistant, and GPT-4.1

https://n8nworkflows.xyz/workflows/extract-insights-from-linkedin-comments-with-apify--pinecone-assistant--and-gpt-4-1-13064


# Extract insights from LinkedIn comments with Apify, Pinecone Assistant, and GPT-4.1

## 1. Workflow Overview

**Purpose:**  
This workflow scrapes recent LinkedIn post comments from a specified LinkedIn profile using **Apify**, uploads the comment data as JSON files into **Pinecone Assistant** for retrieval, and then enables an interactive **chat-based insights experience** using an **n8n AI Agent** powered by **OpenAI (GPT-4.1-mini)** plus a **Pinecone Assistant tool** for context retrieval.

**Primary use cases:**
- Weekly ingestion of LinkedIn post comments into a RAG-ready store (Pinecone Assistant).
- Asking questions like “Summarize sentiment about topic X” or “What objections do commenters raise?” with citations (post + comment URLs).

### 1.1 Scheduled ingestion (Step 1)
Runs weekly, sets the target LinkedIn profile URL, triggers an Apify actor scrape, then fetches the resulting dataset items.

### 1.2 Filtering and gating (Step 1 continuation)
Keeps only items that represent “post” objects and only those with at least one comment.

### 1.3 Transformation + upload to Pinecone Assistant (Step 2)
Normalizes comments into a minimal JSON structure, converts each post’s comments into a JSON file, and uploads it to Pinecone Assistant with metadata.

### 1.4 Interactive insights chat (Step 3)
A chat trigger starts an AI Agent that uses:
- OpenAI chat model (gpt-4.1-mini) for reasoning and response writing
- Pinecone Assistant tool for retrieval over uploaded comment files

---

## 2. Block-by-Block Analysis

### Block A — Scheduled LinkedIn scraping via Apify (Step 1)
**Overview:** Runs weekly, sets the LinkedIn profile URL, executes an Apify actor to scrape posts and comments, then loads the produced dataset items into n8n.

**Nodes involved:**  
- Run weekly  
- Set LinkedIn url  
- Run actor to scrape data  
- Get dataset items

#### Node: **Run weekly**
- **Type / role:** `Schedule Trigger` — workflow entry point on a weekly cadence.
- **Key configuration:** Interval set to **weeks** (no specific day/time shown; defaults depend on n8n UI configuration at save-time).
- **Outputs:** Sends one trigger item to **Set LinkedIn url**.
- **Edge cases / failures:** Instance timezone differences; schedule misconfiguration; workflow must be active to run.

#### Node: **Set LinkedIn url**
- **Type / role:** `Set` — defines the LinkedIn profile URL to scrape.
- **Key configuration choices:**
  - Sets field `linkedInProfile` to a placeholder string: **“full LinkedIn personal or company profile URL”**.
  - Intended to be replaced with an actual profile/company URL.
- **Expressions/variables:** None (static value).
- **Input:** From **Run weekly**.
- **Output:** To **Run actor to scrape data**.
- **Edge cases / failures:** If left as placeholder or invalid URL, the Apify actor may fail or return empty data.

#### Node: **Run actor to scrape data**
- **Type / role:** `Apify` node — runs an Apify Store Actor to scrape LinkedIn posts/comments.
- **Configuration choices (interpreted):**
  - **Actor:** `harvestapi/linkedin-profile-posts` (shown as “LinkedIn Profile Posts Scraper (No Cookies)”).
  - **Custom input body** (JSON) includes:
    - `includeQuotePosts: true`
    - `includeReposts: false`
    - `maxComments: 10`
    - `maxPosts: 15`
    - `maxReactions: 5`
    - `postedLimit: "month"` (scrape posts within last month)
    - `scrapeComments: true`, `scrapeReactions: false`
    - `targetUrls: ["{{ $json.linkedInProfile }}"]` (pulls from **Set LinkedIn url** output)
- **Credentials:** Apify API token (“Apify account”).
- **Input:** From **Set LinkedIn url**.
- **Output:** To **Get dataset items**; the run output must include `defaultDatasetId`.
- **Edge cases / failures:**
  - Actor run can fail due to Apify quota, invalid token, actor downtime.
  - LinkedIn anti-bot changes may reduce output or break scraping.
  - If output does not include `defaultDatasetId`, the next node will fail.

#### Node: **Get dataset items**
- **Type / role:** `Apify` node — reads items from the actor run’s dataset.
- **Key configuration choices:**
  - Resource: **Datasets**
  - Operation: fetch dataset items
  - `datasetId: {{ $json.defaultDatasetId }}`
  - `limit: 100`
- **Credentials:** Apify API token (“Apify account”).
- **Input:** From **Run actor to scrape data**.
- **Output:** To **Filter posts**.
- **Edge cases / failures:**
  - Dataset may be empty.
  - `defaultDatasetId` missing/undefined leads to expression error.
  - Large datasets truncated at limit=100.

---

### Block B — Filter posts and require comments (Step 1 continuation)
**Overview:** Narrows the dataset items to LinkedIn “post” objects only, and ensures there are comments before proceeding to upload.

**Nodes involved:**  
- Filter posts  
- If has comments

#### Node: **Filter posts**
- **Type / role:** `Filter` — keeps only items where `type === "post"`.
- **Key configuration choices:**
  - Condition: `{{$json.type}}` **equals** `"post"`.
- **Input:** From **Get dataset items**.
- **Output:** To **If has comments**.
- **Edge cases / failures:**
  - If Apify schema changes and `type` differs/missing, posts may be incorrectly excluded.
  - Strict validation enabled; unexpected types can drop items.

#### Node: **If has comments**
- **Type / role:** `IF` — gates only posts with at least one comment.
- **Key configuration choices:**
  - Condition: `{{$json.comments.length}} > 0`
- **Input:** From **Filter posts**.
- **Outputs:**
  - **True** branch is connected to **Extract comments**.
  - **False** branch unused (posts without comments are dropped).
- **Edge cases / failures:**
  - If `comments` is missing/null, `comments.length` may throw. (In practice, many scrapers output `comments: []`, but this is not guaranteed.)

---

### Block C — Normalize, file conversion, and upload to Pinecone Assistant (Step 2)
**Overview:** Reshapes each post item into a compact structure (postId, postUrl, list of comments with URLs), converts it to a JSON file, and uploads it to Pinecone Assistant with metadata so it becomes retrievable in chat.

**Nodes involved:**  
- Extract comments  
- Convert to File  
- Upload file

#### Node: **Extract comments**
- **Type / role:** `Set` — transforms Apify post item into an upload-ready object.
- **Key configuration choices:**
  - `postId = {{$json.id}}`
  - `postUrl = {{$json.linkedinUrl}}`
  - `comments` array built with expression:
    - Maps over `($json.comments || [])`
    - Produces objects: `{ id: c.id, commentUrl: c.linkedinUrl, comment: c.commentary }`
  - Dot notation disabled (keeps literal field names; avoids nested path behaviors).
- **Input:** From **If has comments** (true branch).
- **Output:** To **Convert to File**.
- **Edge cases / failures:**
  - If a comment object lacks `linkedinUrl` or `commentary`, output fields may be null/undefined.
  - If post `id` missing, later filename and externalFileId become invalid.

#### Node: **Convert to File**
- **Type / role:** `Convert to File` — converts each item to a JSON file binary for upload.
- **Key configuration choices:**
  - Mode: **each** (one file per item/post)
  - Operation: **toJson**
  - Filename: `{{$json.postId}}.json`
  - “format” option disabled (no pretty-print)
- **Input:** From **Extract comments**.
- **Output:** To **Upload file**.
- **Edge cases / failures:**
  - Missing `postId` yields bad/empty filename.
  - Very large comment payloads could hit size limits (n8n binary memory limits or Pinecone file limits).

#### Node: **Upload file**
- **Type / role:** `Pinecone Assistant` — uploads file into a Pinecone Assistant as a knowledge source.
- **Key configuration choices:**
  - Resource: **file**
  - Operation: **uploadFile**
  - Assistant target: `name = "n8n-assistant"`, `host = "https://prod-1-data.ke.pinecone.io"`
  - `externalFileId = {{ $('Extract comments').item.json.postId }}`
    - Uses the post ID to create/update a stable file identity in Pinecone (important for dedupe/replace semantics depending on Pinecone behavior).
  - Metadata:
    - `postUrl = {{ $('Extract comments').item.json.postUrl }}`
  - Source tag: `n8n:n8n_nodes_pinecone_assistant:extract_insights_from_linkedin_comments`
- **Credentials:** Pinecone Assistant API key (“Pinecone Assistant account”).
- **Input:** From **Convert to File** (binary file + JSON).
- **Output:** Not connected further (end of ingestion path).
- **Edge cases / failures:**
  - Authentication/permission errors with Pinecone API key.
  - Assistant name/host mismatch (wrong region/host).
  - Upload failures on duplicate `externalFileId` depending on Pinecone’s update/overwrite rules.
  - Network timeouts on large uploads.

---

### Block D — Chat-based insights with retrieval (Step 3)
**Overview:** Exposes a chat endpoint; when a user asks a question, an AI Agent uses a Pinecone Assistant retrieval tool to fetch relevant comment context and the OpenAI model to generate an answer. The agent is instructed to always cite both post and comment URLs.

**Nodes involved:**  
- When chat message received  
- AI Agent  
- OpenAI Chat Model  
- Pinecone Assistant tool

#### Node: **When chat message received**
- **Type / role:** `Chat Trigger` (LangChain) — entry point for interactive chat sessions.
- **Key configuration choices:**
  - Webhook-based trigger (webhookId present).
  - Options left default.
- **Output:** To **AI Agent**.
- **Edge cases / failures:**
  - If the workflow is not active/publicly reachable, chat won’t work.
  - Webhook URL changes across environments; must reconfigure clients calling it.

#### Node: **AI Agent**
- **Type / role:** `Agent` (LangChain) — orchestrates LLM + tools.
- **Key configuration choices:**
  - System message:  
    “You are a helpful LinkedIn social media assistant… You have access to LinkedIn post comments and reference post and comment urls. When responding always include both urls as a citation.”
  - Relies on connected **language model** and **tool** ports.
- **Inputs:**
  - Main input from **When chat message received**.
  - `ai_languageModel` input from **OpenAI Chat Model**.
  - `ai_tool` input from **Pinecone Assistant tool**.
- **Output:** Returns the agent response to the chat channel (standard for chat trigger workflows).
- **Edge cases / failures:**
  - Model/tool miswiring (agent runs without retrieval).
  - The instruction “always include both urls” may be impossible if retrieved chunks lack comment URLs; then responses may violate requirement unless prompting/tooling ensures URLs are present.

#### Node: **OpenAI Chat Model**
- **Type / role:** `LM Chat OpenAI` — provides the chat completion model to the agent.
- **Key configuration choices:**
  - Model: **gpt-4.1-mini**
  - No special options/tools configured.
- **Credentials:** OpenAI API key (“OpenAi account”).
- **Connection:** Sends into **AI Agent** via `ai_languageModel`.
- **Edge cases / failures:**
  - Missing/invalid API key.
  - Model availability or org access restrictions.
  - Rate limits/timeouts.

#### Node: **Pinecone Assistant tool**
- **Type / role:** `Pinecone Assistant Tool` — retrieval tool usable by the agent to query the assistant’s stored files.
- **Key configuration choices:**
  - Assistant target: `name = "n8n-assistant"`, `host = "https://prod-1-data.ke.pinecone.io"`
  - Source tag: `n8n:n8n_nodes_pinecone_assistant:extract_insights_from_linkedin_comments`
- **Credentials:** Pinecone Assistant API key (“Pinecone Assistant account”).
- **Connection:** Feeds into **AI Agent** via `ai_tool`.
- **Edge cases / failures:**
  - Assistant empty/not yet populated (agent retrieval yields nothing).
  - Host/assistant mismatch with where files were uploaded.
  - Retrieval quality depends on Pinecone Assistant chunking/indexing of the uploaded JSON.

---

### Block E — Documentation and guidance (Sticky Notes)
**Overview:** Sticky notes provide step labels and external resources/requirements. They don’t affect execution but are important for reproduction.

**Nodes involved:**  
- Sticky Note7  
- Sticky Note1  
- Sticky Note2  
- Sticky Note

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Run weekly | Schedule Trigger | Weekly workflow entry point | — | Set LinkedIn url | ## Step 1: Scrape comments from LinkedIn and upload to Pinecone Assistant |
| Set LinkedIn url | Set | Define target LinkedIn profile/company URL | Run weekly | Run actor to scrape data | ## Step 1: Scrape comments from LinkedIn and upload to Pinecone Assistant |
| Run actor to scrape data | Apify | Run LinkedIn scraping actor; produces dataset id | Set LinkedIn url | Get dataset items | ## Step 1: Scrape comments from LinkedIn and upload to Pinecone Assistant |
| Get dataset items | Apify | Fetch scraped items from Apify dataset | Run actor to scrape data | Filter posts | ## Step 1: Scrape comments from LinkedIn and upload to Pinecone Assistant |
| Filter posts | Filter | Keep only dataset items that are posts | Get dataset items | If has comments | ## Step 1: Scrape comments from LinkedIn and upload to Pinecone Assistant |
| If has comments | If | Gate: only posts with comments proceed | Filter posts | Extract comments | ## Step 1: Scrape comments from LinkedIn and upload to Pinecone Assistant |
| Extract comments | Set | Normalize post + comments into compact JSON | If has comments | Convert to File | ## Step 2: Extract comments to json file and upload to Pinecone Assistant |
| Convert to File | Convert to File | Serialize each post’s normalized JSON into a .json file | Extract comments | Upload file | ## Step 2: Extract comments to json file and upload to Pinecone Assistant |
| Upload file | Pinecone Assistant | Upload JSON file into Pinecone Assistant with metadata | Convert to File | — | ## Step 2: Extract comments to json file and upload to Pinecone Assistant |
| When chat message received | Chat Trigger (LangChain) | Chat entry point (webhook) for asking questions | — | AI Agent | ## Step 3: Extract insights from LinkedIn comments through chat |
| AI Agent | Agent (LangChain) | Orchestrate model + retrieval tool; respond with citations | When chat message received; OpenAI Chat Model (ai_languageModel); Pinecone Assistant tool (ai_tool) | — | ## Step 3: Extract insights from LinkedIn comments through chat |
| OpenAI Chat Model | OpenAI Chat Model (LangChain) | LLM backend for agent | — | AI Agent (ai_languageModel) | ## Step 3: Extract insights from LinkedIn comments through chat |
| Pinecone Assistant tool | Pinecone Assistant Tool | Retrieval tool over uploaded files | — | AI Agent (ai_tool) | ## Step 3: Extract insights from LinkedIn comments through chat |
| Sticky Note7 | Sticky Note | Template info, prerequisites, setup links | — | — | ![Pinecone logo](https://www.pinecone.io/images/pinecone-logo-for-n8n-templates.png)<br>## Try it out<br>This n8n workflow template lets you extract insights from comments on your LinkedIn posts using Pinecone Assistant, Apify, and OpenAI. It scrapes LinkedIn comments using Apify and then retrieves relevant context from this data using Pinecone Assistant and generates insights with OpenAI, all without the need to train your own LLM.<br><br>### What is Pinecone Assistant?<br>[Pinecone Assistant](https://docs.pinecone.io/guides/assistant/overview) allows you to build production-grade chat and agent-based applications quickly…<br><br>### Prerequisites<br>* A [Pinecone account](https://app.pinecone.io/) and [API key](https://app.pinecone.io/organizations/-/projects/-/keys)<br>* An [Open AI account](https://auth.openai.com/create-account) and [API key](https://platform.openai.com/settings/organization/api-keys)<br>* An [Apify account](https://apify.com/) and [API token](https://console.apify.com/settings/integrations)<br><br>### Setup<br>1. Create a Pinecone Assistant… [here](https://app.pinecone.io/organizations/-/projects/-/assistant) … name `n8n-assistant` …<br>…<br>### Need help?<br>[Pinecone Discord community](https://discord.gg/tJ8V62S3sH) / [file an issue](https://github.com/pinecone-io/n8n-templates/issues/new/choose) |
| Sticky Note1 | Sticky Note | Step label | — | — | ## Step 1: Scrape comments from LinkedIn and upload to Pinecone Assistant |
| Sticky Note2 | Sticky Note | Step label | — | — | ## Step 2: Extract comments to json file and upload to Pinecone Assistant |
| Sticky Note | Sticky Note | Step label | — | — | ## Step 3: Extract insights from LinkedIn comments through chat |

---

## 4. Reproducing the Workflow from Scratch

1. **Create credentials in n8n**
   1. Add **Apify API** credentials (API token from Apify).
   2. Add **OpenAI API** credentials (OpenAI API key).
   3. Add **Pinecone Assistant API** credentials (Pinecone API key with access to Assistants).

2. **Create Pinecone Assistant**
   1. In Pinecone Console, create an Assistant named **`n8n-assistant`**.
   2. Note your Assistant **host/region URL** (this workflow uses `https://prod-1-data.ke.pinecone.io`). Use the host that matches your project.

3. **Build Step 1 (scheduled scraping)**
   1. Add node **Schedule Trigger** named **“Run weekly”**.
      - Set interval to **Weeks**.
   2. Add node **Set** named **“Set LinkedIn url”**.
      - Add field `linkedInProfile` (String) = your full LinkedIn personal/company profile URL (e.g., `https://www.linkedin.com/in/.../` or company page URL).
   3. Connect: **Run weekly → Set LinkedIn url**.
   4. Add node **Apify** named **“Run actor to scrape data”**.
      - Resource: **Actor run** (via the Apify node UI)
      - Actor: `harvestapi/linkedin-profile-posts` (or select by the actor ID used)
      - Input body fields:
        - `scrapeComments: true`
        - `maxPosts: 15`, `maxComments: 10`
        - `postedLimit: "month"`
        - `targetUrls`: set to an expression referencing `{{$json.linkedInProfile}}`
      - Select your **Apify credentials**.
   5. Connect: **Set LinkedIn url → Run actor to scrape data**.
   6. Add node **Apify** named **“Get dataset items”**.
      - Resource: **Datasets**
      - Dataset ID: expression `{{$json.defaultDatasetId}}`
      - Limit: `100`
      - Select your **Apify credentials**.
   7. Connect: **Run actor to scrape data → Get dataset items**.

4. **Build Step 1 continuation (filtering)**
   1. Add node **Filter** named **“Filter posts”**.
      - Condition: `{{$json.type}}` equals `post`
   2. Connect: **Get dataset items → Filter posts**.
   3. Add node **If** named **“If has comments”**.
      - Condition: `{{$json.comments.length}}` greater than `0`
   4. Connect: **Filter posts → If has comments** (true branch used next).

5. **Build Step 2 (transform → file → upload)**
   1. Add node **Set** named **“Extract comments”**.
      - Add `postId` = `{{$json.id}}`
      - Add `postUrl` = `{{$json.linkedinUrl}}`
      - Add `comments` (Array) using an expression that maps comments into:
        - `id: c.id`
        - `commentUrl: c.linkedinUrl`
        - `comment: c.commentary`
      - (Optional but matching): disable dot notation.
   2. Connect: **If has comments (true) → Extract comments**.
   3. Add node **Convert to File** named **“Convert to File”**.
      - Mode: **Each item**
      - Operation: **To JSON**
      - File name: `{{$json.postId}}.json`
   4. Connect: **Extract comments → Convert to File**.
   5. Add node **Pinecone Assistant** named **“Upload file”**.
      - Resource: **File**
      - Operation: **Upload file**
      - Assistant selection:
        - Name: `n8n-assistant`
        - Host: your Pinecone host (must match your project)
      - External File ID: `{{ $('Extract comments').item.json.postId }}`
      - Metadata: add `postUrl = {{ $('Extract comments').item.json.postUrl }}`
      - Select your **Pinecone Assistant credentials**.
   6. Connect: **Convert to File → Upload file**.

6. **Build Step 3 (chat insights with retrieval)**
   1. Add node **Chat Trigger** (LangChain) named **“When chat message received”**.
      - Keep defaults; n8n will generate a webhook endpoint.
   2. Add node **AI Agent** (LangChain) named **“AI Agent”**.
      - Set System Message to:  
        “You are a helpful LinkedIn social media assistant… When responding always include both urls as a citation.”
   3. Connect: **When chat message received → AI Agent**.
   4. Add node **OpenAI Chat Model** (LangChain) named **“OpenAI Chat Model”**.
      - Model: **gpt-4.1-mini**
      - Select your **OpenAI credentials**.
   5. Connect: **OpenAI Chat Model → AI Agent** using the **ai_languageModel** connection type.
   6. Add node **Pinecone Assistant Tool** named **“Pinecone Assistant tool”**.
      - Assistant: `n8n-assistant`
      - Host: same Pinecone host as upload node
      - Select your **Pinecone Assistant credentials**.
   7. Connect: **Pinecone Assistant tool → AI Agent** using the **ai_tool** connection type.

7. **Operational sequence**
   1. Run/activate the workflow and execute the scheduled path at least once to populate Pinecone.
   2. Use the chat endpoint to ask questions once data is uploaded.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Pinecone Assistant overview and concept (managed RAG: chunking, embeddings, storage, query planning, reranking, orchestration). | https://docs.pinecone.io/guides/assistant/overview |
| Create Pinecone Assistant in console (name it `n8n-assistant`). | https://app.pinecone.io/organizations/-/projects/-/assistant |
| Pinecone API keys location. | https://app.pinecone.io/organizations/-/projects/-/keys |
| OpenAI account + API keys. | https://auth.openai.com/create-account ; https://platform.openai.com/settings/organization/api-keys |
| Apify account + API token. | https://apify.com/ ; https://console.apify.com/settings/integrations |
| Suggested first chat question: “Summarize the comments related to [SOME TOPIC YOU TALK ABOUT] and categorize into positive, neutral, and negative.” | From workflow notes |
| Help resources: Pinecone Discord; GitHub issues for n8n templates. | https://discord.gg/tJ8V62S3sH ; https://github.com/pinecone-io/n8n-templates/issues/new/choose |
| Branding image used in notes. | https://www.pinecone.io/images/pinecone-logo-for-n8n-templates.png |