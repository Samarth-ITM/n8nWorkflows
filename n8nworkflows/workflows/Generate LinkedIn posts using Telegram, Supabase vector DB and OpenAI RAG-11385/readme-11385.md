Generate LinkedIn posts using Telegram, Supabase vector DB and OpenAI RAG

https://n8nworkflows.xyz/workflows/generate-linkedin-posts-using-telegram--supabase-vector-db-and-openai-rag-11385


# Generate LinkedIn posts using Telegram, Supabase vector DB and OpenAI RAG

## 1. Workflow Overview

**Purpose:** This n8n workflow builds and uses a personal “viral LinkedIn posts” knowledge base. It has two modules:
1) **Collect viral posts**: a Telegram bot accepts a LinkedIn post URL, scrapes the post text, generates embeddings, and stores it in a **Supabase vector store**.  
2) **Generate new posts (RAG)**: a web form collects a hook, then a multi-agent AI pipeline analyzes the hook, produces a post outline, retrieves similar posts from the vector DB (RAG), and generates a final LinkedIn post that can be **auto-published** to LinkedIn.

### 1.1 Viral Post Collection (Telegram → Scrape → Vector DB)
- Trigger on Telegram message
- Validate URL contains `linkedin.com`
- Fetch LinkedIn HTML and extract post commentary
- Load as document, embed with OpenAI, insert into Supabase vector table
- Notify user of success/failure

### 1.2 Post Generation (Form → Agents → RAG → LinkedIn publish)
- Trigger via n8n Form
- Agent 1: analyze hook into structured fields (topic/niche/tone/key points)
- Agent 2: generate a 5-part outline (hook/problem/value/solution/CTA)
- Agent 3: retrieve similar posts from Supabase as a tool (RAG) and ghostwrite a post
- Publish via LinkedIn node

---

## 2. Block-by-Block Analysis

### Block A — Telegram Intake & URL Validation
**Overview:** Receives a Telegram message, shows a typing indicator, and routes execution based on whether the message looks like a LinkedIn URL.  
**Nodes involved:** `On Telegram Message`, `Typing....`, `If`, `Wrong URL`

#### Node: On Telegram Message
- **Type / role:** `telegramTrigger` — entry point for Telegram bot messages.
- **Config:** Listens to **message** updates.
- **Key data used later:** `$json.message.text`, `$json.message.chat.id`
- **Outputs:** Feeds both `If` and `Typing....` in parallel.
- **Failure modes:** Telegram credential revoked; webhook not registered; bot not started by user.

#### Node: Typing....
- **Type / role:** `telegram` — sends chat action for UX.
- **Config:** Operation `sendChatAction`, `chatId = {{ $json.message.chat.id }}`
- **Inputs:** From trigger message.
- **Failure modes:** Telegram API errors (403 if bot blocked, 400 bad chat id).

#### Node: If
- **Type / role:** `if` — validates message content.
- **Condition:**  
  `{{ $json.message.text && $json.message.text.includes("linkedin.com") }}`
- **Routing:**
  - **True** → `LinkedIn Post URL`
  - **False** → `Wrong URL`
- **Edge cases:** URLs without `linkedin.com` (shorteners), empty text messages, media-only messages.

#### Node: Wrong URL
- **Type / role:** `telegram` — user feedback.
- **Config:** Sends “⚠️ Please provide a valid LinkedIn post URL” to  
  `chatId = {{ $('On Telegram Message').item.json.message.chat.id }}`
- **Edge cases:** The node references the trigger node by name; renaming the trigger breaks this expression.

---

### Block B — Scraping LinkedIn Post Content
**Overview:** Fetches the URL HTML, extracts the post text via CSS selector, and handles scraping failures.  
**Nodes involved:** `LinkedIn Post URL`, `Scrap Content`, `Unable to Scrape`

#### Node: LinkedIn Post URL
- **Type / role:** `httpRequest` — fetch HTML of the provided LinkedIn URL.
- **Config:**  
  - `url = {{ $('On Telegram Message').item.json.message.text }}`
  - `onError: continueErrorOutput` (important: prevents workflow from stopping on HTTP error)
- **Outputs:**
  - Main success output → `Scrap Content`
  - Error output → `Unable to Scrape` (via the second connection path)
- **Failure modes / edge cases:**
  - LinkedIn blocks scraping (401/403, bot detection, requires cookies/login).
  - Redirects, regional pages, consent walls.
  - Non-HTML responses, rate limits, timeouts.
  - If URL points to a post that requires authentication, extraction will likely fail.

#### Node: Scrap Content
- **Type / role:** `html` — extracts post content from HTML.
- **Config:** Operation `extractHtmlContent` with:
  - Key: `Post Content`
  - CSS selector: `[data-test-id="main-feed-activity-card__commentary"]`
- **Input:** HTML from `LinkedIn Post URL`.
- **Output:** JSON containing extracted “Post Content”.
- **Failure modes / edge cases:**
  - Selector changes (LinkedIn frequently changes DOM).
  - Multiple matches or empty match -> blank content.
  - Content loaded dynamically (not in initial HTML).

#### Node: Unable to Scrape
- **Type / role:** `telegram` — failure notification.
- **Config:** Sends “😶 Scraping failed for this LinkedIn post” to the trigger chat id.
- **When it runs:** When HTTP request errors (and potentially when scrape produces unusable content, though the current logic only wires HTTP error output).
- **Gap:** There is no explicit “empty extracted content” check; an empty scrape may still proceed to storage.

---

### Block C — Vector DB Ingestion (Supabase + OpenAI Embeddings)
**Overview:** Converts scraped post text into embeddings and inserts it into a Supabase vector table for later RAG retrieval.  
**Nodes involved:** `Upload Document`, `Data Loader`, `Embeddings`, `Code`, `✅ Post Scrapped Sucessfully`

#### Node: Data Loader
- **Type / role:** `documentDefaultDataLoader` (LangChain) — converts incoming item data into a “Document” for vector insertion.
- **Config:** Defaults (no custom mapping shown).
- **Connection:** Provides `ai_document` input to `Upload Document`.
- **Edge cases:** If “Post Content” is missing/empty, the produced document may be empty or poorly structured depending on defaults.

#### Node: Embeddings
- **Type / role:** `embeddingsOpenAi` — generates embedding vectors.
- **Config:** Uses OpenAI credentials “n8n - Vekrro”; default embedding model depends on node defaults/version.
- **Connection:** Provides `ai_embedding` to `Upload Document`.
- **Failure modes:** OpenAI auth errors, quota limits, rate limiting, model deprecation.

#### Node: Upload Document
- **Type / role:** `vectorStoreSupabase` (LangChain) — inserts documents + embeddings into Supabase.
- **Mode:** `insert`
- **Table:** `linkedin_post`
- **Embedding batch size:** 500
- **Dependencies:** Requires both `ai_document` (from Data Loader) and `ai_embedding` (from Embeddings).
- **Failure modes / edge cases:**
  - Supabase credentials invalid.
  - Table missing or schema mismatch (vector column dimensions must match embedding output).
  - Row-level security policies blocking inserts.
  - Large content leading to token/size constraints upstream.

#### Node: Code
- **Type / role:** `code` — reshapes output for the next step.
- **JS:** `return $('Upload Document').all()[0]`
- **Meaning:** Returns the first item from the Upload Document node’s output, ignoring current input.
- **Edge cases:**
  - If `Upload Document` outputs 0 items, this throws.
  - Strong coupling to node name; renaming breaks it.

#### Node: ✅ Post Scrapped Sucessfully
- **Type / role:** `telegram` — success confirmation.
- **Config:** Sends “✅ Post Scraped successfully” to trigger chat id.
- **Note:** Spelling in node name is “Scrapped Sucessfully”, message says “Scraped”.

---

### Block D — Web Form Entry for Post Generation
**Overview:** Provides a web form endpoint to input a hook (and optional image), initiating the AI generation chain.  
**Nodes involved:** `LinkedIn Form`

#### Node: LinkedIn Form
- **Type / role:** `formTrigger` — second entry point (independent of Telegram).
- **Config:**
  - Path: `linkedin-post`
  - Title: “LinkedIn - Bhavy Shekhaliya”
  - Button label: “Generate”
  - Ignore bots: enabled
  - Fields:
    - `Hook` (textarea, required)
    - `Post Image` (optional; type not explicitly set in JSON, appears as a default text field)
- **Output:** Feeds `Hook Analyse Agent`.
- **Failure modes:** Public form abuse if not protected; path conflicts; bot protection limitations.

---

### Block E — Multi-Agent Analysis + Structured Outputs
**Overview:** Uses LLM agents with structured output parsers to convert free text into machine-usable JSON, then builds an outline.  
**Nodes involved:** `Hook Analyse Agent`, `Structured Output Parser`, `Post Structure Agent`, `Structured Output Parser.`, `4o mini`, `2.5-flash`, `5 nano`

#### Node: 4o mini
- **Type / role:** `lmChatOpenAi` — primary LLM for agents.
- **Model:** `gpt-4o-mini`
- **Connections:** Supplies `ai_languageModel` to all three agents as index 0 (primary).
- **Failure modes:** OpenAI credential issues, rate limits.

#### Node: 2.5-flash
- **Type / role:** `lmChatGoogleGemini` — fallback LLM for agents.
- **Connections:** Supplies `ai_languageModel` to agents as index 1 (fallback).
- **Failure modes:** Google credential/config issues (not shown in JSON), quota limits.

#### Node: Hook Analyse Agent
- **Type / role:** `agent` — extracts structured hook metadata.
- **Prompt:** Analyzes `{{ $json.Hook }}` and extracts:
  - topic
  - niche/industry
  - emotion/tone
  - 3–5 key points
- **Config:** `needsFallback: true`, `hasOutputParser: true`
- **Inputs:** Main input from `LinkedIn Form`; LLM from `4o mini` + fallback `2.5-flash`; parser from `Structured Output Parser`.
- **Output:** Feeds `Post Structure Agent`.
- **Failure modes:** Model returns non-JSON; parser auto-fix may still fail on very malformed outputs.

#### Node: Structured Output Parser
- **Type / role:** `outputParserStructured` — enforces JSON schema for Hook analysis.
- **AutoFix:** true
- **Schema example:** topic/niche/tone/key_points array.
- **LLM dependency:** Uses `5 nano` as `ai_languageModel` (shared across parsers).
- **Edge cases:** If the hook is very short/ambiguous, keys may be hallucinated; schema still forces fields.

#### Node: Post Structure Agent
- **Type / role:** `agent` — generates a 5-section outline from analyzed hook.
- **Prompt inputs:**  
  `Topic: {{ $json.output.topic }}`, etc.
- **Config:** `needsFallback: true`, `hasOutputParser: true`
- **Parser:** `Structured Output Parser.`
- **Output:** Feeds `Post Generator Agent`.
- **Potential issue:** In the Post Generator prompt, **Problem** is mapped to `{{ $json.output.structure.value }}` (likely a mistake; should be `.problem`).

#### Node: Structured Output Parser.
- **Type / role:** structured parser for outline.
- **AutoFix:** true
- **Schema example:** `{ structure: { hook, problem, value, solution, cta } }`
- **LLM dependency:** `5 nano`
- **Edge cases:** If the agent produces extra keys, they may be dropped/rewritten.

#### Node: 5 nano
- **Type / role:** `lmChatOpenAi` — used by output parsers for auto-fix / schema correction.
- **Model:** `gpt-5-nano`
- **Connections:** Feeds all three structured parsers.
- **Failure modes:** Access to GPT-5 models may require specific OpenAI account entitlements.

---

### Block F — RAG Retrieval + Post Generation + LinkedIn Publishing
**Overview:** Retrieves similar viral posts from Supabase vector store as a tool (RAG) and generates the final post, then publishes it to LinkedIn.  
**Nodes involved:** `Supabase Vector Store`, `Embeddings.`, `Post Generator Agent`, `Structured Output Parser1`, `Create a post`

#### Node: Embeddings.
- **Type / role:** `embeddingsOpenAi` — embeddings for retrieval queries.
- **Config:** Default options (credentials not explicitly attached in JSON here; may use default/implicit credentials in your n8n instance).
- **Connection:** Provides `ai_embedding` to `Supabase Vector Store`.
- **Failure modes:** Missing credential assignment (common cause of runtime error), rate limits.

#### Node: Supabase Vector Store
- **Type / role:** `vectorStoreSupabase` — configured as a LangChain “tool” for retrieval.
- **Mode:** `retrieve-as-tool`
- **Table:** `linkedin_post`
- **topK:** 5
- **Tool description:** “You are helpful assistant”
- **Connection:** Exposed as `ai_tool` to `Post Generator Agent`.
- **Failure modes / edge cases:**
  - Table not populated yet → low-quality retrieval.
  - Embedding dimension mismatch between ingestion and retrieval.
  - RLS policies block select operations.

#### Node: Post Generator Agent
- **Type / role:** `agent` — final ghostwriter using RAG.
- **Prompt:** Describes a 3-step DB analysis process and then generates a post using outline fields:
  - Hook: `{{ $json.output.structure.hook }}`
  - Problem: `{{ $json.output.structure.value }}` (likely incorrect mapping)
  - Value: `{{ $json.output.structure.value }}`
  - Solution: `{{ $json.output.structure.solution }}`
  - CTA: `{{ $json.output.structure.cta }}`
- **Inputs:**
  - Main: from `Post Structure Agent`
  - LLM: `4o mini` primary, `2.5-flash` fallback
  - Tool: `Supabase Vector Store` (RAG)
  - Output parser: `Structured Output Parser1`
- **Failure modes:** Tool call failures, low recall if embeddings mismatch, agent may ignore tool if not compelled strongly enough.

#### Node: Structured Output Parser1
- **Type / role:** structured parser for final post.
- **Schema example:** `{ "Post Content": "Post Content" }`
- **LLM dependency:** `5 nano`
- **Edge cases:** If you want rich output (hashtags, line breaks, etc.), ensure the parser schema allows it and the agent puts content under `Post Content`.

#### Node: Create a post
- **Type / role:** `linkedIn` — publishes generated content to LinkedIn.
- **Config:** Minimal (`additionalFields` empty). The workflow JSON does not show the exact text mapping; you typically map `Post Content` into the LinkedIn node’s text field.
- **Failure modes:** OAuth permissions, token expiry, missing “w_member_social”/posting permissions, API limitations for images.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| On Telegram Message | telegramTrigger | Entry point for Telegram scraper module | — | If; Typing.... | # LinkedIn Post Generator with Viral Content Vector Database (full overview + setup notes incl. BotFather link https://t.me/BotFather) |
| Typing.... | telegram | Sends “typing” chat action | On Telegram Message | — | ## 📌 Viral LinkedIn Post Vector Database - Transform any LinkedIn post URL into actionable insights... |
| If | if | Validates message contains LinkedIn URL | On Telegram Message | LinkedIn Post URL (true); Wrong URL (false) | ## 📌 Viral LinkedIn Post Vector Database - Transform any LinkedIn post URL into actionable insights... |
| Wrong URL | telegram | Notifies invalid URL | If (false) | — | ## 📌 Viral LinkedIn Post Vector Database - Transform any LinkedIn post URL into actionable insights... |
| LinkedIn Post URL | httpRequest | Fetch LinkedIn post HTML | If (true) | Scrap Content (success); Unable to Scrape (error) | ## 📌 Viral LinkedIn Post Vector Database - Transform any LinkedIn post URL into actionable insights... |
| Scrap Content | html | Extracts post commentary via CSS selector | LinkedIn Post URL | Upload Document | ## 📌 Viral LinkedIn Post Vector Database - Transform any LinkedIn post URL into actionable insights... |
| Unable to Scrape | telegram | Notifies scraping failure | LinkedIn Post URL (error output) | — | ## 📌 Viral LinkedIn Post Vector Database - Transform any LinkedIn post URL into actionable insights... |
| Upload Document | vectorStoreSupabase (LangChain) | Inserts scraped post into Supabase vector table | Scrap Content (+ Data Loader ai_document + Embeddings ai_embedding) | Code | ## 📌 Viral LinkedIn Post Vector Database - Transform any LinkedIn post URL into actionable insights... |
| Data Loader | documentDefaultDataLoader (LangChain) | Converts extracted text into document | (implicit flow into Upload Document via ai_document) | Upload Document (ai_document) | ## 📌 Viral LinkedIn Post Vector Database - Transform any LinkedIn post URL into actionable insights... |
| Embeddings | embeddingsOpenAi (LangChain) | Creates embeddings for insertion | (implicit flow into Upload Document via ai_embedding) | Upload Document (ai_embedding) | ## 📌 Viral LinkedIn Post Vector Database - Transform any LinkedIn post URL into actionable insights... |
| Code | code | Returns first output item from Upload Document | Upload Document | ✅ Post Scrapped Sucessfully | ## 📌 Viral LinkedIn Post Vector Database - Transform any LinkedIn post URL into actionable insights... |
| ✅ Post Scrapped Sucessfully | telegram | Confirms scrape+store success | Code | — | ## 📌 Viral LinkedIn Post Vector Database - Transform any LinkedIn post URL into actionable insights... |
| LinkedIn Form | formTrigger | Entry point for generation module via web form | — | Hook Analyse Agent | ## Generate LinkedIn Post using LinkedIn Post Vector Store |
| Hook Analyse Agent | agent (LangChain) | Extracts topic/niche/tone/key points from hook | LinkedIn Form | Post Structure Agent | ## Generate LinkedIn Post using LinkedIn Post Vector Store |
| Structured Output Parser | outputParserStructured (LangChain) | Enforces JSON schema for hook analysis | 5 nano (ai_languageModel) | Hook Analyse Agent (ai_outputParser) | ## Generate LinkedIn Post using LinkedIn Post Vector Store |
| Post Structure Agent | agent (LangChain) | Builds 5-part post outline | Hook Analyse Agent | Post Generator Agent | ## Generate LinkedIn Post using LinkedIn Post Vector Store |
| Structured Output Parser. | outputParserStructured (LangChain) | Enforces JSON schema for outline | 5 nano (ai_languageModel) | Post Structure Agent (ai_outputParser) | ## Generate LinkedIn Post using LinkedIn Post Vector Store |
| Post Generator Agent | agent (LangChain) | RAG-based final post writing | Post Structure Agent (+ Supabase Vector Store tool) | Create a post | ## Generate LinkedIn Post using LinkedIn Post Vector Store |
| Structured Output Parser1 | outputParserStructured (LangChain) | Enforces JSON schema for final post content | 5 nano (ai_languageModel) | Post Generator Agent (ai_outputParser) | ## Generate LinkedIn Post using LinkedIn Post Vector Store |
| 4o mini | lmChatOpenAi (LangChain) | Primary LLM for agents | — | Hook Analyse Agent; Post Structure Agent; Post Generator Agent | ## Generate LinkedIn Post using LinkedIn Post Vector Store |
| 2.5-flash | lmChatGoogleGemini (LangChain) | Fallback LLM for agents | — | Hook Analyse Agent; Post Structure Agent; Post Generator Agent | ## Generate LinkedIn Post using LinkedIn Post Vector Store |
| 5 nano | lmChatOpenAi (LangChain) | LLM used by parsers for schema auto-fix | — | Structured Output Parser; Structured Output Parser.; Structured Output Parser1 | ## Generate LinkedIn Post using LinkedIn Post Vector Store |
| Supabase Vector Store | vectorStoreSupabase (LangChain) | Retrieval tool (topK=5) for RAG | Embeddings. (ai_embedding) | Post Generator Agent (ai_tool) | ## Generate LinkedIn Post using LinkedIn Post Vector Store |
| Embeddings. | embeddingsOpenAi (LangChain) | Creates embeddings for retrieval queries | — | Supabase Vector Store (ai_embedding) | ## Generate LinkedIn Post using LinkedIn Post Vector Store |
| Create a post | linkedIn | Publishes generated post to LinkedIn | Post Generator Agent | — | ## Generate LinkedIn Post using LinkedIn Post Vector Store |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n with two separate entry points: Telegram Trigger and Form Trigger.

### Module 1 — Telegram scraper → Supabase vector insert
2. Add **Telegram Trigger** node named **On Telegram Message**  
   - Updates: `message`  
   - Credentials: Telegram bot token (create via [@BotFather](https://t.me/BotFather)).

3. Add **Telegram** node named **Typing....**  
   - Operation: `Send Chat Action`  
   - Chat ID: `{{ $json.message.chat.id }}`  
   - Connect: `On Telegram Message → Typing....`

4. Add **If** node named **If**  
   - Condition (Boolean true):  
     `{{ $json.message.text && $json.message.text.includes("linkedin.com") }}`  
   - Connect: `On Telegram Message → If`

5. Add **Telegram** node named **Wrong URL**  
   - Send message text: `⚠️ Please provide a valid LinkedIn post URL`  
   - Chat ID: `{{ $('On Telegram Message').item.json.message.chat.id }}`  
   - Connect: `If (false) → Wrong URL`

6. Add **HTTP Request** node named **LinkedIn Post URL**  
   - URL: `{{ $('On Telegram Message').item.json.message.text }}`  
   - Set **Error Handling** to: “Continue on Fail” / `onError=continueErrorOutput`  
   - Connect: `If (true) → LinkedIn Post URL`

7. Add **HTML** node named **Scrap Content**  
   - Operation: Extract HTML Content  
   - CSS selector: `[data-test-id="main-feed-activity-card__commentary"]`  
   - Key name: `Post Content`  
   - Connect: `LinkedIn Post URL (success) → Scrap Content`

8. Add **Telegram** node named **Unable to Scrape**  
   - Text: `😶 Scraping failed for this LinkedIn post`  
   - Chat ID: `{{ $('On Telegram Message').item.json.message.chat.id }}`  
   - Connect: `LinkedIn Post URL (error output) → Unable to Scrape`

9. Add **Document Default Data Loader** (LangChain) node named **Data Loader**  
   - Leave defaults unless you want to map specific fields into the document content/metadata.

10. Add **OpenAI Embeddings** (LangChain) node named **Embeddings**  
   - Credentials: OpenAI API key  
   - Keep default model/options (or select your preferred embedding model).

11. Add **Supabase Vector Store** (LangChain) node named **Upload Document**  
   - Mode: `Insert`  
   - Table name: `linkedin_post`  
   - Credentials: Supabase (URL + service role key recommended for server-side inserts)  
   - Connect:
   - `Scrap Content → Upload Document (main)`
   - `Data Loader → Upload Document (ai_document)`
   - `Embeddings → Upload Document (ai_embedding)`

12. Add **Code** node named **Code**  
   - JS: `return $('Upload Document').all()[0]`  
   - Connect: `Upload Document → Code`

13. Add **Telegram** node named **✅ Post Scrapped Sucessfully**  
   - Text: `✅ Post Scraped successfully`  
   - Chat ID: `{{ $('On Telegram Message').item.json.message.chat.id }}`  
   - Connect: `Code → ✅ Post Scrapped Sucessfully`

### Module 2 — Form → Agents → RAG → LinkedIn publish
14. Add **Form Trigger** node named **LinkedIn Form**  
   - Path: `linkedin-post`  
   - Title: “LinkedIn - Bhavy Shekhaliya”  
   - Field 1: `Hook` (Textarea, required)  
   - Field 2: `Post Image` (optional)  
   - Connect: `LinkedIn Form → Hook Analyse Agent`

15. Add LLM nodes:
   - **OpenAI Chat Model** named **4o mini**: select model `gpt-4o-mini`, set OpenAI credentials.
   - **Google Gemini Chat Model** named **2.5-flash**: configure Google/Gemini credentials; use as fallback.
   - **OpenAI Chat Model** named **5 nano**: select model `gpt-5-nano` (requires access), used by parsers.

16. Add **Structured Output Parser** node (LangChain) named **Structured Output Parser**  
   - Auto-fix: enabled  
   - Provide JSON schema example with `topic`, `niche`, `tone`, `key_points[]`  
   - Connect: `5 nano → Structured Output Parser (ai_languageModel)`

17. Add **Agent** node named **Hook Analyse Agent**  
   - Prompt: hook analysis (topic/niche/tone/key points) using `{{ $json.Hook }}`  
   - Enable: `hasOutputParser`, `needsFallback`  
   - Connect:
     - `LinkedIn Form → Hook Analyse Agent (main)`
     - `4o mini → Hook Analyse Agent (ai_languageModel index 0)`
     - `2.5-flash → Hook Analyse Agent (ai_languageModel index 1)`
     - `Structured Output Parser → Hook Analyse Agent (ai_outputParser)`

18. Add **Structured Output Parser** node named **Structured Output Parser.** for the outline schema  
   - Auto-fix enabled; schema example with `structure.hook/problem/value/solution/cta`  
   - Connect: `5 nano → Structured Output Parser. (ai_languageModel)`

19. Add **Agent** node named **Post Structure Agent**  
   - Prompt: produce 5-section outline using `{{ $json.output.* }}` from Hook agent  
   - Connect: `Hook Analyse Agent → Post Structure Agent (main)`  
   - LLM connections: `4o mini` (primary) + `2.5-flash` (fallback)  
   - Parser connection: `Structured Output Parser.` → agent.

20. Add **OpenAI Embeddings** node named **Embeddings.** (for retrieval)  
   - Set OpenAI credentials (explicitly assign to avoid runtime credential errors).

21. Add **Supabase Vector Store** node named **Supabase Vector Store**  
   - Mode: `Retrieve as tool`  
   - Table: `linkedin_post`  
   - topK: 5  
   - Connect: `Embeddings. → Supabase Vector Store (ai_embedding)`

22. Add **Structured Output Parser** node named **Structured Output Parser1** for final output  
   - Schema example: `{ "Post Content": "..." }`  
   - Connect: `5 nano → Structured Output Parser1 (ai_languageModel)`

23. Add **Agent** node named **Post Generator Agent**  
   - Prompt: instruct to query vector DB for similar posts + generate final post using outline fields  
   - Connect:
     - `Post Structure Agent → Post Generator Agent (main)`
     - `4o mini` (primary LLM) + `2.5-flash` (fallback)
     - `Supabase Vector Store → Post Generator Agent (ai_tool)`
     - `Structured Output Parser1 → Post Generator Agent (ai_outputParser)`

24. Add **LinkedIn** node named **Create a post**  
   - Configure LinkedIn OAuth2 credentials (LinkedIn app with posting permissions).  
   - Map the generated content (typically `{{ $json.output["Post Content"] }}` or whatever field your agent emits) into the LinkedIn post text field.  
   - Connect: `Post Generator Agent → Create a post`

**Recommended fix while reproducing:** In the **Post Generator Agent** prompt, map:
- `Problem: {{ $json.output.structure.problem }}` (not `.value`), unless intentional.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Create Telegram bot via BotFather and use the token in Telegram credentials | https://t.me/BotFather |
| Supabase setup: enable vector extension and create `linkedin_post` with correct vector dimensions | Mentioned in the workflow notes (Supabase vector DB requirement) |
| Workflow concept: two modules (Telegram scraper + web form generation with RAG) | Sticky note “LinkedIn Post Generator with Viral Content Vector Database” |
| LinkedIn posting is optional; you can remove the LinkedIn node to only return formatted text | Mentioned in sticky note “Stage 4: Publication” |
| Scraping relies on a LinkedIn DOM selector that may break if LinkedIn changes markup | CSS selector `[data-test-id="main-feed-activity-card__commentary"]` |

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.