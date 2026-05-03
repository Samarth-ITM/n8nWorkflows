Retrieve and answer Gmail email queries with Llama 3.2, mxbai-embed, and Qdrant

https://n8nworkflows.xyz/workflows/retrieve-and-answer-gmail-email-queries-with-llama-3-2--mxbai-embed--and-qdrant-14268


# Retrieve and answer Gmail email queries with Llama 3.2, mxbai-embed, and Qdrant

# 1. Workflow Overview

This workflow builds and uses a lightweight email-answering system backed by a vector database and local AI models.

Its purpose is twofold:

1. **Knowledge ingestion:** watch a FAQ JSON file in Google Drive, convert each FAQ entry into embeddings using a local embedding model, and store them in Qdrant.
2. **Email answering:** monitor Gmail for incoming messages, embed the email text, retrieve the most relevant FAQ entries from Qdrant, ask a local LLM to draft a reply, wrap that reply in branded HTML, and send it as a Gmail thread reply.

The workflow is designed for use cases such as:

- automated student support or helpdesk email replies
- FAQ-driven email response systems
- local/private RAG-style support automation using LM Studio + Qdrant + n8n

## 1.1 Knowledge Base Ingestion

This block starts when a specific Google Drive file changes. It downloads the file, parses the JSON FAQ payload, checks whether the target Qdrant collection exists, recreates it if necessary, generates embeddings for each FAQ item, and upserts them into Qdrant.

## 1.2 Vector Database Management

This block isolates collection lifecycle management: read the target collection name from environment variables, check whether it exists, delete it if present, then recreate it cleanly before loading new vectors.

## 1.3 Incoming Email Processing

This block starts when a new Gmail message arrives. It extracts sender/thread/message metadata, builds an embedding from the email body, fetches the collection name, and combines both streams so Qdrant can perform a semantic search.

## 1.4 Retrieval and AI Response Generation

This block queries Qdrant for the top 3 similar FAQ entries, feeds them plus the original email content into a local LLM, and asks it to produce HTML body content only.

## 1.5 Email Rendering and Reply Delivery

This block inserts the generated HTML body into a branded email template and replies directly in the original Gmail thread.

---

# 2. Block-by-Block Analysis

## Block 1 — Knowledge Base Ingestion

### Overview
This block monitors a specific Google Drive file and transforms its JSON content into structured FAQ records suitable for vectorization. It prepares the raw source data used later for semantic retrieval.

### Nodes Involved
- Google Drive Trigger
- Download file
- Extract from File
- Data Fetch
- Data Preprocessing

### Node Details

#### 1. Google Drive Trigger
- **Type and role:** `n8n-nodes-base.googleDriveTrigger`; entry point that polls Google Drive for updates on one specific file.
- **Configuration choices:**
  - Trigger mode: `specificFile`
  - File watched: a fixed Google Drive file ID (`11HH12Rrdy6RGSBsHM8Ci31OYM0VfHZCr`)
  - Polling: default interval because `pollTimes.item` is present but not customized in detail
- **Key expressions or variables used:** none in parameters beyond the fixed file reference
- **Input and output connections:**
  - No input
  - Outputs to:
    - `Download file`
    - `Fetch Collection Name`
- **Version-specific requirements:** typeVersion `1`
- **Edge cases / failures:**
  - Google OAuth2 token expiry or missing scope
  - Wrong or inaccessible file ID
  - Trigger delay due to polling instead of instant push
  - If the watched file is replaced rather than updated, the file ID may change and the trigger may stop being relevant
- **Sub-workflow reference:** none

#### 2. Download file
- **Type and role:** `n8n-nodes-base.googleDrive`; downloads the updated file contents from Google Drive.
- **Configuration choices:**
  - Operation: `download`
  - File ID: `={{ $json.id }}`
  - This means it uses the file ID emitted by the trigger event
- **Key expressions or variables used:**
  - `$json.id`
- **Input and output connections:**
  - Input from `Google Drive Trigger`
  - Output to `Extract from File`
- **Version-specific requirements:** typeVersion `3`
- **Edge cases / failures:**
  - Download permission denied
  - File metadata event exists but content download fails
  - Binary output may not match expected format if the watched file is not actually JSON
- **Sub-workflow reference:** none

#### 3. Extract from File
- **Type and role:** `n8n-nodes-base.extractFromFile`; parses the downloaded file content into JSON.
- **Configuration choices:**
  - Operation: `fromJson`
- **Key expressions or variables used:** none
- **Input and output connections:**
  - Input from `Download file`
  - Output to `Data Fetch`
- **Version-specific requirements:** typeVersion `1.1`
- **Edge cases / failures:**
  - Invalid JSON file
  - Unexpected file encoding
  - Large file parsing failures
- **Sub-workflow reference:** none

#### 4. Data Fetch
- **Type and role:** `n8n-nodes-base.set`; normalizes parsed JSON to a single `data` array field.
- **Configuration choices:**
  - Creates field `data`
  - Value: `={{ $json.data.data }}`
  - Assumes the parsed file structure contains `data.data`
- **Key expressions or variables used:**
  - `$json.data.data`
- **Input and output connections:**
  - Input from `Extract from File`
  - Output to `Data Preprocessing`
- **Version-specific requirements:** typeVersion `3.4`
- **Edge cases / failures:**
  - Expression breaks if file shape differs from expected nested structure
  - If `data.data` is not an array, downstream code will fail
- **Sub-workflow reference:** none

#### 5. Data Preprocessing
- **Type and role:** `n8n-nodes-base.code`; iterates over the FAQ array and converts each item into a Qdrant-ready record.
- **Configuration choices:**
  - Reads input array from `$input.first().json.data`
  - Reads target collection from environment variable `QDRANT_COLLECTION`
  - For each source FAQ item, returns:
    - `id`
    - `topic`
    - `question`
    - `answer`
    - `embeddingText` as `[topic] question`
    - `collection`
- **Key expressions or variables used:**
  - `$input.first().json.data`
  - `$env.QDRANT_COLLECTION`
- **Input and output connections:**
  - Input from `Data Fetch`
  - Output to `Embedding Generation`
- **Version-specific requirements:** typeVersion `2`
- **Edge cases / failures:**
  - Missing environment variable
  - Non-array input
  - Missing FAQ fields such as `id`, `topic`, or `question`
  - Duplicate IDs can cause point overwrites in Qdrant
- **Sub-workflow reference:** none

---

## Block 2 — Vector Database Management

### Overview
This block manages Qdrant collection lifecycle to ensure the FAQ index is rebuilt cleanly whenever the source file changes. It prevents stale or duplicate data by deleting and recreating the collection when it already exists.

### Nodes Involved
- Fetch Collection Name
- Check If Collection Exists
- If
- Delete Collection
- Create Collection
- Embedding Generation
- Upsert Points

### Node Details

#### 6. Fetch Collection Name
- **Type and role:** `n8n-nodes-base.code`; resolves the Qdrant collection name from environment variables.
- **Configuration choices:**
  - Reads `$env.QDRANT_COLLECTION`
  - Returns `{ collection: collection }`
- **Key expressions or variables used:**
  - `$env.QDRANT_COLLECTION`
- **Input and output connections:**
  - Input from `Google Drive Trigger`
  - Output to `Check If Collection Exists`
- **Version-specific requirements:** typeVersion `2`
- **Edge cases / failures:**
  - Missing environment variable yields undefined collection names
- **Sub-workflow reference:** none

#### 7. Check If Collection Exists
- **Type and role:** `n8n-nodes-qdrant.qdrant`; checks whether the target Qdrant collection already exists.
- **Configuration choices:**
  - Operation: `collectionExists`
  - Collection name: `={{ $json.collection }}`
- **Key expressions or variables used:**
  - `$json.collection`
- **Input and output connections:**
  - Input from `Fetch Collection Name`
  - Output to `If`
- **Version-specific requirements:** Qdrant node typeVersion `1`
- **Edge cases / failures:**
  - Qdrant auth or connectivity issues
  - Empty collection name
- **Sub-workflow reference:** none

#### 8. If
- **Type and role:** `n8n-nodes-base.if`; branches based on whether the collection exists.
- **Configuration choices:**
  - Condition checks `={{ $json.result.exists }}` equals `true`
  - True branch means collection exists
  - False branch means it does not exist
- **Key expressions or variables used:**
  - `$json.result.exists`
- **Input and output connections:**
  - Input from `Check If Collection Exists`
  - True output to `Delete Collection`
  - False output to `Create Collection`
- **Version-specific requirements:** typeVersion `2.2`
- **Edge cases / failures:**
  - If Qdrant response shape changes, expression may break
  - Strict boolean comparison can fail if API returns string-like values
- **Sub-workflow reference:** none

#### 9. Delete Collection
- **Type and role:** `n8n-nodes-qdrant.qdrant`; deletes the existing Qdrant collection before recreation.
- **Configuration choices:**
  - Operation: `deleteCollection`
  - Collection name: `{{ $json.collection }}`
- **Key expressions or variables used:**
  - `$json.collection`
- **Input and output connections:**
  - Input from `If` true branch
  - Output to `Create Collection`
- **Version-specific requirements:** typeVersion `1`
- **Edge cases / failures:**
  - Delete race conditions if collection is in use
  - Collection may disappear between existence check and delete
- **Sub-workflow reference:** none

#### 10. Create Collection
- **Type and role:** `n8n-nodes-qdrant.qdrant`; creates a fresh Qdrant collection configured for embedding vectors.
- **Configuration choices:**
  - Operation: `createCollection`
  - Collection name: `{{ $json.collection }}`
  - Vector config:
    - `size: 1024`
    - `distance: Cosine`
  - `shardNumber: 1`
  - `replicationFactor: 1`
- **Key expressions or variables used:**
  - `$json.collection`
- **Input and output connections:**
  - Input from:
    - `If` false branch
    - `Delete Collection`
  - No explicit downstream connection in this workflow
- **Version-specific requirements:** typeVersion `1`
- **Edge cases / failures:**
  - The vector size must match the embedding model output dimension
  - If `mxbai-embed-large-v1` output dimension differs from 1024 in the actual LM Studio setup, all upserts/searches will fail
  - Qdrant permission/network failures
- **Sub-workflow reference:** none

#### 11. Embedding Generation
- **Type and role:** `n8n-nodes-base.httpRequest`; requests embeddings from a locally hosted OpenAI-compatible embedding endpoint.
- **Configuration choices:**
  - Method: `POST`
  - URL: `http://192.168.56.1:1234/v1/embeddings`
  - JSON body:
    - model: `text-embedding-mxbai-embed-large-v1`
    - input: `{{ $json.embeddingText }}`
  - Sends `Content-Type: application/json`
- **Key expressions or variables used:**
  - `$json.embeddingText`
- **Input and output connections:**
  - Input from `Data Preprocessing`
  - Output to `Upsert Points`
- **Version-specific requirements:** typeVersion `4.3`
- **Edge cases / failures:**
  - LM Studio server not running
  - Wrong host IP when n8n runs in Docker
  - Timeout on local model inference
  - Response shape mismatch if endpoint is not OpenAI-compatible
- **Sub-workflow reference:** none

#### 12. Upsert Points
- **Type and role:** `n8n-nodes-qdrant.qdrant`; inserts or updates one vector point per FAQ item in Qdrant.
- **Configuration choices:**
  - Resource: `point`
  - Operation: `upsertPoints`
  - Collection name from `Data Preprocessing`
  - Points payload includes:
    - `id`
    - payload fields: `id`, `topic`, `question`, `answer`, `embeddingText`
    - vector from `Embedding Generation`
- **Key expressions or variables used:**
  - `$('Data Preprocessing').item.json.id`
  - `$('Data Preprocessing').item.json.topic`
  - `$('Data Preprocessing').item.json.question`
  - `$('Data Preprocessing').item.json.answer`
  - `$('Data Preprocessing').item.json.embeddingText`
  - `$('Data Preprocessing').item.json.collection`
  - `$('Embedding Generation').item.json.data[0].embedding`
- **Input and output connections:**
  - Input from `Embedding Generation`
  - No downstream node
- **Version-specific requirements:** typeVersion `1`
- **Edge cases / failures:**
  - Expression coupling assumes one embedding result per incoming FAQ item
  - ID type must be valid for Qdrant
  - Vector dimension mismatch with collection schema
  - Qdrant rejects malformed JSON if quoting/escaping fails
- **Sub-workflow reference:** none

---

## Block 3 — Incoming Email Processing

### Overview
This block receives Gmail messages, extracts key email metadata, generates a semantic embedding from the email body, and prepares the collection name needed for retrieval. It sets up both the query vector and the Qdrant target.

### Nodes Involved
- Gmail Trigger
- Fields Preparation
- Query Embedding
- Collection Name Fetch
- Merge

### Node Details

#### 13. Gmail Trigger
- **Type and role:** `n8n-nodes-base.gmailTrigger`; starts the email-answering branch whenever a new Gmail message is detected.
- **Configuration choices:**
  - Poll mode: every minute
  - No specific filters configured
- **Key expressions or variables used:** none
- **Input and output connections:**
  - No input
  - Output to `Fields Preparation`
- **Version-specific requirements:** typeVersion `1.3`
- **Edge cases / failures:**
  - Gmail OAuth2 token expiry
  - Polling may reprocess messages depending on Gmail trigger state behavior
  - Without filters, unrelated inbox mail may trigger the workflow
- **Sub-workflow reference:** none

#### 14. Fields Preparation
- **Type and role:** `n8n-nodes-base.set`; extracts and reshapes email fields needed later for search and reply.
- **Configuration choices:**
  - `from` = `{{ $json.To }}`
  - `name` = first quoted display name from `From`, then strips periods and trims
  - `to` = email address captured from angle brackets in `From`
  - `email_id` = Gmail message ID
  - `thread_id` = Gmail thread ID
  - `subject` = subject line
  - `email_body` = Gmail snippet
- **Key expressions or variables used:**
  - `$json.To`
  - `$json.From.match(/\"(.+?)\"/)[1].replace('.', '').trim()`
  - `$json.From.match(/<(.+?)>/)[1]`
  - `$json.id`
  - `$json.threadId`
  - `$json.Subject`
  - `$json.snippet`
- **Input and output connections:**
  - Input from `Gmail Trigger`
  - Outputs to:
    - `Query Embedding`
    - `Collection Name Fetch`
- **Version-specific requirements:** typeVersion `3.4`
- **Edge cases / failures:**
  - Regex parsing for sender name/email is fragile
  - `From` may not contain quotes or angle brackets
  - Gmail snippet may truncate the real question, reducing retrieval quality
  - `from` and `to` field naming is counterintuitive here
- **Sub-workflow reference:** none

#### 15. Query Embedding
- **Type and role:** `n8n-nodes-base.httpRequest`; converts the incoming email text into an embedding for semantic search.
- **Configuration choices:**
  - Method: `POST`
  - URL: `http://192.168.56.1:1234/v1/embeddings`
  - Body model: `text-embedding-mxbai-embed-large-v1`
  - Input text: `{{ $json.email_body }}`
- **Key expressions or variables used:**
  - `$json.email_body`
- **Input and output connections:**
  - Input from `Fields Preparation`
  - Output to `Merge`
- **Version-specific requirements:** typeVersion `4.3`
- **Edge cases / failures:**
  - Same LM Studio connectivity risks as ingestion embeddings
  - Truncated email snippet may generate incomplete embeddings
- **Sub-workflow reference:** none

#### 16. Collection Name Fetch
- **Type and role:** `n8n-nodes-base.code`; retrieves the same Qdrant collection name for the email-search branch.
- **Configuration choices:**
  - Reads `$env.QDRANT_COLLECTION`
  - Returns `{ collection: collection }`
- **Key expressions or variables used:**
  - `$env.QDRANT_COLLECTION`
- **Input and output connections:**
  - Input from `Fields Preparation`
  - Output to `Merge`
- **Version-specific requirements:** typeVersion `2`
- **Edge cases / failures:**
  - Missing environment variable
- **Sub-workflow reference:** none

#### 17. Merge
- **Type and role:** `n8n-nodes-base.merge`; combines embedding output and collection name into one item for Qdrant search.
- **Configuration choices:**
  - Mode: `combine`
  - Combine by: `combineByPosition`
  - `alwaysOutputData: true`
- **Key expressions or variables used:** none directly
- **Input and output connections:**
  - Input 0 from `Query Embedding`
  - Input 1 from `Collection Name Fetch`
  - Output to `Similarity Search`
- **Version-specific requirements:** typeVersion `3.2`
- **Edge cases / failures:**
  - Positional combine assumes one item from each branch
  - If one branch emits zero items or multiple items, merge can misalign data
- **Sub-workflow reference:** none

---

## Block 4 — Retrieval and AI Response Generation

### Overview
This block uses the query embedding to search Qdrant for the top matching FAQ points, then prompts a local LLM to draft an email reply based only on the retrieved results and the incoming email.

### Nodes Involved
- Similarity Search
- Message a model

### Node Details

#### 18. Similarity Search
- **Type and role:** `n8n-nodes-qdrant.qdrant`; performs vector similarity search against the configured collection.
- **Configuration choices:**
  - Resource: `search`
  - Operation: `queryPoints`
  - Limit: `3`
  - Score threshold: `0.6`
  - Query vector: `={{ JSON.stringify($json.data[0].embedding) }}`
  - Collection name: `={{ $json.collection }}`
- **Key expressions or variables used:**
  - `$json.data[0].embedding`
  - `$json.collection`
- **Input and output connections:**
  - Input from `Merge`
  - Output to `Message a model`
- **Version-specific requirements:** typeVersion `1`
- **Edge cases / failures:**
  - Search fails if collection doesn’t exist yet
  - Search fails if vector length does not match collection vector size
  - Score threshold of 0.6 may return fewer than 3 points
  - Downstream LLM prompt assumes points `[0]`, `[1]`, and `[2]` exist
- **Sub-workflow reference:** none

#### 19. Message a model
- **Type and role:** `@n8n/n8n-nodes-langchain.openAi`; sends a chat-style prompt to an OpenAI-compatible model endpoint, configured here for a local Llama model.
- **Configuration choices:**
  - Model: `llama-3.2-3b-instruct`
  - Credential type: OpenAI API credential
  - Prompt contains:
    - a long **system instruction** defining role, university context, strict tone rules, HTML-only output rules, fallback behavior, and prohibited disclosures
    - a **user message** containing:
      - student name
      - student email body
      - top 3 retrieved answers with scores, topics, matched questions, and answer texts
- **Key expressions or variables used:**
  - `$('Fields Preparation').item.json.name`
  - `$('Fields Preparation').item.json.email_body`
  - `$json.result.points[0].score`
  - `$json.result.points[0].payload.topic`
  - `$json.result.points[0].payload.question`
  - `$json.result.points[0].payload.answer`
  - same pattern for points 1 and 2
- **Input and output connections:**
  - Input from `Similarity Search`
  - Output to `Email Format`
- **Version-specific requirements:** typeVersion `2`
- **Edge cases / failures:**
  - Local model server unreachable or misconfigured
  - If the OpenAI credential points to OpenAI cloud instead of LM Studio-compatible endpoint, model ID likely fails
  - Prompt breaks when fewer than 3 Qdrant results are returned
  - LLM may produce malformed or partial HTML despite instructions
- **Sub-workflow reference:** none

---

## Block 5 — Email Rendering and Reply Delivery

### Overview
This block takes the model-generated HTML fragment, injects it into a branded email template, and replies to the original Gmail message.

### Nodes Involved
- Email Format
- Send Repy Email

### Node Details

#### 20. Email Format
- **Type and role:** `n8n-nodes-base.set`; creates a full HTML email document containing branding, contact details, footer, and the LLM-generated response body.
- **Configuration choices:**
  - Sets `reply_email` to a complete HTML document
  - Injects LLM output at:
    - `{{ $json.output[0].content[0].text }}`
- **Key expressions or variables used:**
  - `$json.output[0].content[0].text`
- **Input and output connections:**
  - Input from `Message a model`
  - Output to `Send Repy Email`
- **Version-specific requirements:** typeVersion `3.4`
- **Edge cases / failures:**
  - Output path assumes a specific LangChain/OpenAI node response structure
  - If model output is empty or malformed, email body may be broken
  - Full HTML document may not be necessary for Gmail reply rendering, but should still work
- **Sub-workflow reference:** none

#### 21. Send Repy Email
- **Type and role:** `n8n-nodes-base.gmail`; replies to the original Gmail message using the generated HTML.
- **Configuration choices:**
  - Operation: `reply`
  - Message: `={{ $json.reply_email }}`
  - Message ID: `={{ $('Fields Preparation').item.json.email_id }}`
  - Options:
    - `appendAttribution: false`
    - `replyToSenderOnly: true`
- **Key expressions or variables used:**
  - `$json.reply_email`
  - `$('Fields Preparation').item.json.email_id`
- **Input and output connections:**
  - Input from `Email Format`
  - No downstream connection
- **Version-specific requirements:** typeVersion `2.1`
- **Edge cases / failures:**
  - Gmail credential permission problems
  - Reply may fail if original message ID is unavailable or stale
  - HTML sanitization/rendering differences in Gmail
- **Sub-workflow reference:** none

---

## Block 6 — Sticky Notes / Embedded Documentation

### Overview
These nodes do not execute workflow logic, but they document deployment assumptions, architecture, and important warnings. They are essential for reproducing the setup correctly.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5
- Sticky Note6
- Sticky Note7
- Sticky Note8

### Node Details

#### 22. Sticky Note
- **Type and role:** `n8n-nodes-base.stickyNote`; high-level project documentation.
- **Configuration choices:** contains setup, flow summary, and customization notes.
- **Input and output connections:** none
- **Version-specific requirements:** typeVersion `1`
- **Edge cases / failures:** none
- **Sub-workflow reference:** none

#### 23. Sticky Note1
- **Type and role:** sticky note; labels the ingestion area.
- **Configuration choices:** “Knowledge Base Ingestion” description.
- **Input and output connections:** none
- **Version-specific requirements:** typeVersion `1`
- **Edge cases / failures:** none
- **Sub-workflow reference:** none

#### 24. Sticky Note2
- **Type and role:** sticky note; warning for file selection.
- **Configuration choices:** instructs explicit FAQ JSON file ID selection.
- **Input and output connections:** none
- **Version-specific requirements:** typeVersion `1`
- **Edge cases / failures:** none
- **Sub-workflow reference:** none

#### 25. Sticky Note3
- **Type and role:** sticky note; labels vector database management area.
- **Configuration choices:** notes collection recreation and FAQ upserts.
- **Input and output connections:** none
- **Version-specific requirements:** typeVersion `1`
- **Edge cases / failures:** none
- **Sub-workflow reference:** none

#### 26. Sticky Note4
- **Type and role:** sticky note; labels AI reply generation area.
- **Configuration choices:** describes model drafting and Gmail reply.
- **Input and output connections:** none
- **Version-specific requirements:** typeVersion `1`
- **Edge cases / failures:** none
- **Sub-workflow reference:** none

#### 27. Sticky Note5
- **Type and role:** sticky note; operational warning for LLM server availability.
- **Configuration choices:** reminds that LM Studio must be running.
- **Input and output connections:** none
- **Version-specific requirements:** typeVersion `1`
- **Edge cases / failures:** none
- **Sub-workflow reference:** none

#### 28. Sticky Note6
- **Type and role:** sticky note; networking warning for embedding endpoint.
- **Configuration choices:** reminds that Docker cannot usually use `localhost` to reach host services.
- **Input and output connections:** none
- **Version-specific requirements:** typeVersion `1`
- **Edge cases / failures:** none
- **Sub-workflow reference:** none

#### 29. Sticky Note7
- **Type and role:** sticky note; labels incoming email processing area.
- **Configuration choices:** describes Gmail trigger, embedding, and Qdrant search path.
- **Input and output connections:** none
- **Version-specific requirements:** typeVersion `1`
- **Edge cases / failures:** none
- **Sub-workflow reference:** none

#### 30. Sticky Note8
- **Type and role:** sticky note; duplicate networking warning for embedding endpoint in ingestion area.
- **Configuration choices:** same Docker/local IP reminder.
- **Input and output connections:** none
- **Version-specific requirements:** typeVersion `1`
- **Edge cases / failures:** none
- **Sub-workflow reference:** none

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Google Drive Trigger | n8n-nodes-base.googleDriveTrigger | Watches a specific Google Drive FAQ file for changes |  | Download file; Fetch Collection Name | ## How it works<br>1. **Knowledge Ingestion**: Monitors Google Drive for FAQ file updates, converts them into embeddings, and stores them in Qdrant.<br>2. **Email Trigger**: Activates automatically whenever a new email arrives in your connected Gmail inbox.<br>3. **Semantic Search**: Converts the incoming email question into an embedding and queries Qdrant to find the most relevant FAQ match.<br>4. **AI Reply**: Uses a local LLM via LM Studio to draft a polite, context-aware response and sends it as a threaded reply in Gmail.<br><br>## Setup Process<br>- [ ] Run n8n and Qdrant using Docker.<br>- [ ] Start your LM Studio local server on port `1234` with `mxbai-embed-large-v1` and `llama-3.2-3b-instruct` models.<br>- [ ] Configure Google OAuth2 credentials for both Gmail and Google Drive.<br>- [ ] Update the URL in the embedding nodes and the OpenAI node to use your LM Studio's local IP (do not use `localhost`).<br>- [ ] Select your specific FAQ JSON file in both the **Google Drive Trigger** and **Download File** nodes.<br><br>## Customization<br>- **AI Persona**: Edit the system prompt in the **Message a Model** node to change the AI's tone and formatting.<br>- **Search Strictness**: Adjust the minimum similarity threshold in the Qdrant search node to ensure high-quality matching.<br>- **Triggers**: Swap Gmail for another email provider or customer support tool if needed.<br>## Knowledge Base Ingestion<br>Monitors Google Drive for updates, downloads the FAQ file, and transforms the text into vector embeddings.<br>You must explicitly select your FAQ JSON file ID for the workflow to pull the correct data. |
| Download file | n8n-nodes-base.googleDrive | Downloads the watched FAQ file | Google Drive Trigger | Extract from File | ## Knowledge Base Ingestion<br>Monitors Google Drive for updates, downloads the FAQ file, and transforms the text into vector embeddings.<br>You must explicitly select your FAQ JSON file ID for the workflow to pull the correct data. |
| Extract from File | n8n-nodes-base.extractFromFile | Parses the downloaded FAQ JSON file | Download file | Data Fetch | ## Knowledge Base Ingestion<br>Monitors Google Drive for updates, downloads the FAQ file, and transforms the text into vector embeddings. |
| Create Collection | n8n-nodes-qdrant.qdrant | Creates a fresh Qdrant collection | If; Delete Collection |  | ## Vector Database Management<br>Checks for existing Qdrant collections, recreates them to prevent duplicates, and upserts the new FAQ <br>The embedding URL must point to your host machine's actual local IP address so Docker can reach it. |
| Check If Collection Exists | n8n-nodes-qdrant.qdrant | Checks whether the target Qdrant collection already exists | Fetch Collection Name | If | ## Vector Database Management<br>Checks for existing Qdrant collections, recreates them to prevent duplicates, and upserts the new FAQ |
| If | n8n-nodes-base.if | Branches depending on Qdrant collection existence | Check If Collection Exists | Delete Collection; Create Collection | ## Vector Database Management<br>Checks for existing Qdrant collections, recreates them to prevent duplicates, and upserts the new FAQ |
| Delete Collection | n8n-nodes-qdrant.qdrant | Deletes the existing Qdrant collection before recreation | If | Create Collection | ## Vector Database Management<br>Checks for existing Qdrant collections, recreates them to prevent duplicates, and upserts the new FAQ |
| Upsert Points | n8n-nodes-qdrant.qdrant | Inserts FAQ embeddings and payloads into Qdrant | Embedding Generation |  | ## Vector Database Management<br>Checks for existing Qdrant collections, recreates them to prevent duplicates, and upserts the new FAQ |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Watches Gmail for new incoming emails |  | Fields Preparation | ## How it works<br>1. **Knowledge Ingestion**: Monitors Google Drive for FAQ file updates, converts them into embeddings, and stores them in Qdrant.<br>2. **Email Trigger**: Activates automatically whenever a new email arrives in your connected Gmail inbox.<br>3. **Semantic Search**: Converts the incoming email question into an embedding and queries Qdrant to find the most relevant FAQ match.<br>4. **AI Reply**: Uses a local LLM via LM Studio to draft a polite, context-aware response and sends it as a threaded reply in Gmail.<br><br>## Setup Process<br>- [ ] Run n8n and Qdrant using Docker.<br>- [ ] Start your LM Studio local server on port `1234` with `mxbai-embed-large-v1` and `llama-3.2-3b-instruct` models.<br>- [ ] Configure Google OAuth2 credentials for both Gmail and Google Drive.<br>- [ ] Update the URL in the embedding nodes and the OpenAI node to use your LM Studio's local IP (do not use `localhost`).<br>- [ ] Select your specific FAQ JSON file in both the **Google Drive Trigger** and **Download File** nodes.<br><br>## Customization<br>- **AI Persona**: Edit the system prompt in the **Message a Model** node to change the AI's tone and formatting.<br>- **Search Strictness**: Adjust the minimum similarity threshold in the Qdrant search node to ensure high-quality matching.<br>- **Triggers**: Swap Gmail for another email provider or customer support tool if needed.<br>## Incoming Email Processing<br>Catches incoming emails, generates an embedding for the user's question, and searches Qdrant for the best answer. |
| Merge | n8n-nodes-base.merge | Combines query embedding and collection name | Query Embedding; Collection Name Fetch | Similarity Search | ## Incoming Email Processing<br>Catches incoming emails, generates an embedding for the user's question, and searches Qdrant for the best answer. |
| Fetch Collection Name | n8n-nodes-base.code | Reads Qdrant collection name from env for ingestion | Google Drive Trigger | Check If Collection Exists | ## Vector Database Management<br>Checks for existing Qdrant collections, recreates them to prevent duplicates, and upserts the new FAQ |
| Data Preprocessing | n8n-nodes-base.code | Converts FAQ items into embedding/upsert-ready records | Data Fetch | Embedding Generation | ## Knowledge Base Ingestion<br>Monitors Google Drive for updates, downloads the FAQ file, and transforms the text into vector embeddings. |
| Embedding Generation | n8n-nodes-base.httpRequest | Requests embeddings for FAQ entries from LM Studio | Data Preprocessing | Upsert Points | The embedding URL must point to your host machine's actual local IP address so Docker can reach it. |
| Data Fetch | n8n-nodes-base.set | Extracts FAQ array from parsed JSON structure | Extract from File | Data Preprocessing | ## Knowledge Base Ingestion<br>Monitors Google Drive for updates, downloads the FAQ file, and transforms the text into vector embeddings. |
| Query Embedding | n8n-nodes-base.httpRequest | Requests embedding for incoming email text | Fields Preparation | Merge | ## Incoming Email Processing<br>Catches incoming emails, generates an embedding for the user's question, and searches Qdrant for the best answer.<br>The embedding URL must point to your host machine's actual local IP address so Docker can reach it. |
| Fields Preparation | n8n-nodes-base.set | Extracts sender, thread, message, and body fields from Gmail event | Gmail Trigger | Query Embedding; Collection Name Fetch | ## Incoming Email Processing<br>Catches incoming emails, generates an embedding for the user's question, and searches Qdrant for the best answer. |
| Collection Name Fetch | n8n-nodes-base.code | Reads Qdrant collection name from env for search | Fields Preparation | Merge | ## Incoming Email Processing<br>Catches incoming emails, generates an embedding for the user's question, and searches Qdrant for the best answer. |
| Similarity Search | n8n-nodes-qdrant.qdrant | Searches Qdrant for top similar FAQ entries | Merge | Message a model | ## Incoming Email Processing<br>Catches incoming emails, generates an embedding for the user's question, and searches Qdrant for the best answer.<br>## AI Reply Generation<br>Prompts the local LLM to draft a professional HTML response and sends it back to the original email thread. |
| Send Repy Email | n8n-nodes-base.gmail | Sends Gmail threaded reply with generated HTML | Email Format |  | ## AI Reply Generation<br>Prompts the local LLM to draft a professional HTML response and sends it back to the original email thread. |
| Message a model | @n8n/n8n-nodes-langchain.openAi | Generates the HTML reply body using local Llama | Similarity Search | Email Format | ## AI Reply Generation<br>Prompts the local LLM to draft a professional HTML response and sends it back to the original email thread.<br>Ensure LM Studio is actively running the LLaMA model server; otherwise, the response generation will fail. |
| Email Format | n8n-nodes-base.set | Wraps LLM output in a branded HTML email template | Message a model | Send Repy Email | ## AI Reply Generation<br>Prompts the local LLM to draft a professional HTML response and sends it back to the original email thread. |
| Sticky Note | n8n-nodes-base.stickyNote | Embedded workflow documentation |  |  |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Visual documentation for ingestion block |  |  |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Visual warning about FAQ file selection |  |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Visual documentation for vector DB block |  |  |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Visual documentation for AI reply block |  |  |  |
| Sticky Note5 | n8n-nodes-base.stickyNote | Visual warning about LM Studio LLaMA availability |  |  |  |
| Sticky Note6 | n8n-nodes-base.stickyNote | Visual warning about Docker-to-host embedding URL |  |  |  |
| Sticky Note7 | n8n-nodes-base.stickyNote | Visual documentation for email processing block |  |  |  |
| Sticky Note8 | n8n-nodes-base.stickyNote | Visual warning about Docker-to-host embedding URL |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

Below is a full rebuild sequence for n8n.

## Prerequisites

1. **Deploy supporting services**
   - Run **n8n**
   - Run **Qdrant**
   - Run **LM Studio** locally with an OpenAI-compatible API server exposed on port `1234`

2. **Load local models in LM Studio**
   - Embedding model: `text-embedding-mxbai-embed-large-v1`
   - Chat model: `llama-3.2-3b-instruct`

3. **Prepare credentials**
   - **Google Drive OAuth2**
   - **Gmail OAuth2**
   - **Qdrant REST API**
   - **OpenAI-compatible credential** pointing to your LM Studio server, not necessarily OpenAI cloud

4. **Set environment variable in n8n**
   - `QDRANT_COLLECTION=<your_collection_name>`

5. **Prepare the FAQ JSON file**
   - The workflow expects a structure where parsed content can be accessed as `data.data`
   - Each FAQ record should contain at least:
     - `id`
     - `topic`
     - `question`
     - `answer`

---

## Build the ingestion branch

### 1. Add `Google Drive Trigger`
1. Create a **Google Drive Trigger** node.
2. Set:
   - **Trigger On**: `Specific File`
   - **File to Watch**: choose the FAQ JSON file by file ID
3. Attach your **Google Drive OAuth2** credential.

### 2. Add `Download file`
1. Create a **Google Drive** node.
2. Set:
   - **Operation**: `Download`
   - **File ID**: `={{ $json.id }}`
3. Use the same **Google Drive OAuth2** credential.
4. Connect `Google Drive Trigger -> Download file`.

### 3. Add `Extract from File`
1. Create an **Extract From File** node.
2. Set:
   - **Operation**: `From JSON`
3. Connect `Download file -> Extract from File`.

### 4. Add `Data Fetch`
1. Create a **Set** node named `Data Fetch`.
2. Add one assignment:
   - **Name**: `data`
   - **Type**: `Array`
   - **Value**: `={{ $json.data.data }}`
3. Connect `Extract from File -> Data Fetch`.

### 5. Add `Data Preprocessing`
1. Create a **Code** node named `Data Preprocessing`.
2. Paste this logic conceptually:
   - Read `const data = $input.first().json.data`
   - Read `const collection = $env.QDRANT_COLLECTION`
   - Return one item per FAQ row with:
     - `id`
     - `topic`
     - `question`
     - `answer`
     - `embeddingText` = `[topic] question`
     - `collection`
3. Connect `Data Fetch -> Data Preprocessing`.

---

## Build the collection-management branch

### 6. Add `Fetch Collection Name`
1. Create a **Code** node.
2. Return:
   - `collection = $env.QDRANT_COLLECTION`
3. Connect `Google Drive Trigger -> Fetch Collection Name`.

### 7. Add `Check If Collection Exists`
1. Create a **Qdrant** node.
2. Set:
   - **Operation**: `Collection Exists`
   - **Collection Name**: `={{ $json.collection }}`
3. Attach **Qdrant REST API** credentials.
4. Connect `Fetch Collection Name -> Check If Collection Exists`.

### 8. Add `If`
1. Create an **If** node.
2. Add boolean condition:
   - Left value: `={{ $json.result.exists }}`
   - Operation: `equals`
   - Right value: `true`
3. Connect `Check If Collection Exists -> If`.

### 9. Add `Delete Collection`
1. Create a **Qdrant** node.
2. Set:
   - **Operation**: `Delete Collection`
   - **Collection Name**: `{{ $json.collection }}`
3. Use the same Qdrant credential.
4. Connect the **true** branch of `If -> Delete Collection`.

### 10. Add `Create Collection`
1. Create a **Qdrant** node.
2. Set:
   - **Operation**: `Create Collection`
   - **Collection Name**: `{{ $json.collection }}`
   - **Vectors**:
     - `size: 1024`
     - `distance: Cosine`
   - **Shard Number**: `1`
   - **Replication Factor**: `1`
3. Connect:
   - `If` false branch -> `Create Collection`
   - `Delete Collection` -> `Create Collection`

> Important: confirm your embedding model actually outputs **1024 dimensions**. If not, change the vector size here.

---

## Build FAQ embedding and Qdrant loading

### 11. Add `Embedding Generation`
1. Create an **HTTP Request** node.
2. Configure:
   - **Method**: `POST`
   - **URL**: `http://<your-host-ip>:1234/v1/embeddings`
   - **Send Headers**: yes
   - **Headers**:
     - `Content-Type: application/json`
   - **Send Body**: yes
   - **Body Type**: JSON
   - **JSON Body**:
     - `model`: `text-embedding-mxbai-embed-large-v1`
     - `input`: `{{ $json.embeddingText }}`
3. Connect `Data Preprocessing -> Embedding Generation`.

> Do not use `localhost` if n8n is inside Docker. Use the actual host IP or a working host alias.

### 12. Add `Upsert Points`
1. Create a **Qdrant** node.
2. Set:
   - **Resource**: `Point`
   - **Operation**: `Upsert Points`
   - **Collection Name**: `={{ $('Data Preprocessing').item.json.collection }}`
3. Configure the **Points** JSON so each item contains:
   - `id` from `Data Preprocessing`
   - `payload.id`
   - `payload.topic`
   - `payload.question`
   - `payload.answer`
   - `payload.embeddingText`
   - `vector` from `$('Embedding Generation').item.json.data[0].embedding`
4. Connect `Embedding Generation -> Upsert Points`.

---

## Build the email-processing branch

### 13. Add `Gmail Trigger`
1. Create a **Gmail Trigger** node.
2. Set polling to:
   - **Every Minute**
3. Attach **Gmail OAuth2** credential.
4. Leave filters empty unless you want inbox restrictions.

### 14. Add `Fields Preparation`
1. Create a **Set** node.
2. Add these fields:
   - `from` = `={{ $json.To }}`
   - `name` = `={{ $json.From.match(/\"(.+?)\"/)[1].replace('.', '').trim() }}`
   - `to` = `={{ $json.From.match(/<(.+?)>/)[1] }}`
   - `email_id` = `={{ $json.id }}`
   - `thread_id` = `={{ $json.threadId }}`
   - `subject` = `={{ $json.Subject }}`
   - `email_body` = `={{ $json.snippet }}`
3. Connect `Gmail Trigger -> Fields Preparation`.

> Recommended hardening: replace the regex-based sender parsing with safer fallback logic because not all `From` headers have quoted names.

### 15. Add `Query Embedding`
1. Create an **HTTP Request** node.
2. Configure exactly like the ingestion embedding node, except:
   - `input` = `{{ $json.email_body }}`
3. Connect `Fields Preparation -> Query Embedding`.

### 16. Add `Collection Name Fetch`
1. Create a **Code** node.
2. Return the env collection:
   - `collection = $env.QDRANT_COLLECTION`
3. Connect `Fields Preparation -> Collection Name Fetch`.

### 17. Add `Merge`
1. Create a **Merge** node.
2. Set:
   - **Mode**: `Combine`
   - **Combine By**: `Position`
   - **Always Output Data**: enabled
3. Connect:
   - `Query Embedding -> Merge` as input 1
   - `Collection Name Fetch -> Merge` as input 2

---

## Build retrieval and generation

### 18. Add `Similarity Search`
1. Create a **Qdrant** node.
2. Set:
   - **Resource**: `Search`
   - **Operation**: `Query Points`
   - **Collection Name**: `={{ $json.collection }}`
   - **Query**: `={{ JSON.stringify($json.data[0].embedding) }}`
   - **Limit**: `3`
   - **Score Threshold**: `0.6`
3. Connect `Merge -> Similarity Search`.

> Recommended hardening: add a fallback branch if fewer than 3 points are returned.

### 19. Add `Message a model`
1. Create an **OpenAI / LangChain OpenAI** node.
2. Point its credential/base URL to your **LM Studio** OpenAI-compatible endpoint.
3. Choose model:
   - `llama-3.2-3b-instruct`
4. Add two prompt messages:
   - **System message**:
     - Define the AI as Crestwood University’s support email assistant
     - Instruct it to use only retrieved answers
     - Require fallback wording if none are relevant
     - Require HTML body only
   - **User message**:
     - Include:
       - student name from `Fields Preparation`
       - student email body
       - top 3 retrieved answers with score, topic, matched question, answer
5. Connect `Similarity Search -> Message a model`.

> Important: the prompt currently assumes exactly 3 retrieved points. If Qdrant returns fewer, this node can fail due to invalid expressions.

---

## Build email templating and reply

### 20. Add `Email Format`
1. Create a **Set** node.
2. Add one string field:
   - `reply_email`
3. Set it to a full HTML template containing:
   - page wrapper
   - Crestwood University header
   - body content placeholder using:
     - `{{ $json.output[0].content[0].text }}`
   - contact information
   - signature
   - footer
4. Connect `Message a model -> Email Format`.

### 21. Add `Send Repy Email`
1. Create a **Gmail** node.
2. Set:
   - **Operation**: `Reply`
   - **Message**: `={{ $json.reply_email }}`
   - **Message ID**: `={{ $('Fields Preparation').item.json.email_id }}`
3. Options:
   - `appendAttribution: false`
   - `replyToSenderOnly: true`
4. Use the same **Gmail OAuth2** credential.
5. Connect `Email Format -> Send Repy Email`.

---

## Connection map summary

### Ingestion branch
1. `Google Drive Trigger -> Download file`
2. `Download file -> Extract from File`
3. `Extract from File -> Data Fetch`
4. `Data Fetch -> Data Preprocessing`
5. `Data Preprocessing -> Embedding Generation`
6. `Embedding Generation -> Upsert Points`

### Collection lifecycle branch
7. `Google Drive Trigger -> Fetch Collection Name`
8. `Fetch Collection Name -> Check If Collection Exists`
9. `Check If Collection Exists -> If`
10. `If (true) -> Delete Collection`
11. `Delete Collection -> Create Collection`
12. `If (false) -> Create Collection`

### Email-answering branch
13. `Gmail Trigger -> Fields Preparation`
14. `Fields Preparation -> Query Embedding`
15. `Fields Preparation -> Collection Name Fetch`
16. `Query Embedding -> Merge`
17. `Collection Name Fetch -> Merge`
18. `Merge -> Similarity Search`
19. `Similarity Search -> Message a model`
20. `Message a model -> Email Format`
21. `Email Format -> Send Repy Email`

---

## Required credentials and endpoint behavior

### Google Drive OAuth2
- Must allow file metadata access and file download for the watched FAQ file.

### Gmail OAuth2
- Must support trigger polling and message replies.

### Qdrant REST API
- Must point to the correct Qdrant instance.
- The collection used must match `QDRANT_COLLECTION`.

### OpenAI-compatible credential for LM Studio
- Should target the local LM Studio server base URL.
- Must support:
  - `/v1/chat/completions` or equivalent for the LLM node
  - model name `llama-3.2-3b-instruct`

### HTTP embedding endpoints
- Both embedding nodes must call:
  - `http://<reachable-host-ip>:1234/v1/embeddings`
- The embedding model must be loaded in LM Studio.

---

## Recommended improvements when rebuilding

1. **Guard against missing Qdrant hits**
   - Add an IF or Code node after `Similarity Search`
   - If fewer than 3 points exist, construct a safer prompt

2. **Use full email body instead of snippet**
   - Gmail snippet is often truncated
   - Retrieval quality will improve if you fetch the full plain text body

3. **Harden sender parsing**
   - Replace regex-only parsing with fallback expressions

4. **Wait for collection creation before upserts**
   - In some environments, ingestion and collection recreation may race
   - Consider merging collection creation success with FAQ processing before upserts begin

5. **Fix naming typo**
   - `Send Repy Email` should likely be renamed `Send Reply Email`

6. **Add error handling**
   - Use Error Trigger or dedicated branches for:
     - LM Studio unavailable
     - Qdrant unavailable
     - malformed FAQ JSON
     - Gmail/API auth failures

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Run n8n and Qdrant using Docker. | Deployment note |
| Start LM Studio local server on port `1234` with `mxbai-embed-large-v1` and `llama-3.2-3b-instruct` models. | Runtime dependency |
| Configure Google OAuth2 credentials for both Gmail and Google Drive. | Credential setup |
| Update the embedding-node URLs and the OpenAI-compatible model endpoint to use your LM Studio local IP, not `localhost`, when n8n runs in Docker. | Docker networking note |
| Select the specific FAQ JSON file in both the Google Drive Trigger and file download path. | Data source setup |
| AI persona, formatting, and tone are controlled mainly by the system prompt in `Message a model`. | Customization |
| Search strictness is controlled by the Qdrant similarity threshold, currently `0.6`. | Retrieval tuning |
| Gmail can be replaced by another email provider or support tool if equivalent trigger/send nodes are used. | Architecture flexibility |
| Crestwood University branding appears throughout the prompt and HTML template. | Branding note |
| Portal link used in the template: [https://portal.crestwooduniversity.edu](https://portal.crestwooduniversity.edu) | Template link |
| Support email used in the template: `support@crestwooduniversity.edu` | Template contact |
| Helpdesk email mentioned in prompt context: `helpdesk@crestwooduniversity.edu` | Prompt context |