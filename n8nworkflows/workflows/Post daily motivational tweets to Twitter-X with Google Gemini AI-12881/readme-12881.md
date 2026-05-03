Post daily motivational tweets to Twitter/X with Google Gemini AI

https://n8nworkflows.xyz/workflows/post-daily-motivational-tweets-to-twitter-x-with-google-gemini-ai-12881


# Post daily motivational tweets to Twitter/X with Google Gemini AI

## 1. Workflow Overview

**Title:** Post daily motivational tweets to Twitter/X with Google Gemini AI  
**Workflow name (in n8n):** Daily Motivational Tweets

**Purpose:**  
This workflow runs daily, asks **Google Gemini** to generate **exactly 3 motivational tweet texts**, parses the model output into individual items, removes duplicates (intended to avoid reposting), then posts each item to **Twitter/X** with a delay between posts.

**Typical use cases:**
- Automated daily social posting for creators/brands
- Consistent “motivational quote” account operation
- Prompt-driven content generation with controlled tweet lengths and tone

### Logical Blocks
**1.1 Scheduled Start**  
Runs once per day at a configured hour.

**1.2 AI Quote Generation (Gemini via LangChain)**  
Uses a strict prompt to produce a JSON array of 3 strings.

**1.3 Parsing & Deduplication**  
Converts the JSON array into 3 separate n8n items and attempts to filter out previously posted quotes.

**1.4 Tweet Posting Loop with Delay**  
Iterates over the quotes, posts one tweet at a time, waits, then continues until all quotes are posted.

---

## 2. Block-by-Block Analysis

### Block 1.1 — Scheduled Start
**Overview:** Triggers the workflow daily at a fixed hour and hands control to the quote generation chain.  
**Nodes involved:** Schedule Trigger

#### Node: **Schedule Trigger**
- **Type / role:** `n8n-nodes-base.scheduleTrigger` — entry point (time-based trigger).
- **Configuration (interpreted):**
  - Runs every day at **09:00** (server/workflow timezone as configured in n8n instance).
- **Inputs / outputs:**
  - **Output →** `Generate Quotes`
- **Version notes:** typeVersion **1.3**
- **Edge cases / failures:**
  - Timezone misunderstandings (server vs user expectation).
  - If n8n instance is paused/offline at trigger time, the run may be missed depending on n8n scheduling behavior.

---

### Block 1.2 — AI Quote Generation (Gemini via LangChain)
**Overview:** Uses a LangChain “LLM Chain” node connected to a Gemini Chat Model to generate 3 distinct motivational tweet texts formatted as a JSON array of strings.  
**Nodes involved:** Google Gemini Chat Model, Generate Quotes

#### Node: **Google Gemini Chat Model**
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` — provides the language model to downstream LangChain nodes.
- **Configuration (interpreted):**
  - Sets **topP = 0.8** (affects randomness / diversity of outputs).
  - Uses **Google PaLM/Gemini API credentials** (`googlePalmApi`).
- **Connections:**
  - **AI Language Model output →** `Generate Quotes` (to its `ai_languageModel` input).
- **Version notes:** typeVersion **1**
- **Edge cases / failures:**
  - Credential/auth errors (invalid API key, project not enabled, billing).
  - Rate limits / quota exhaustion.
  - Model output not respecting strict formatting (JSON-only requirement), despite prompt.

#### Node: **Generate Quotes**
- **Type / role:** `@n8n/n8n-nodes-langchain.chainLlm` — runs a single prompt against the connected model and returns generated text.
- **Configuration (interpreted):**
  - Prompt instructs:
    - **Exactly 3** outputs
    - Different style and length:
      - <80 chars, 120–180 chars, 180–220 chars
    - No hashtags, no emojis, no quotes, no markdown, no lists/numbering
    - **Return ONLY a valid JSON array of strings**
- **Connections:**
  - **Input ←** `Schedule Trigger`
  - **Model input ←** `Google Gemini Chat Model`
  - **Output →** `Parse Quotes`
- **Version notes:** typeVersion **1.7**
- **Edge cases / failures:**
  - Model returns non-JSON (extra text, markdown fences, trailing commentary).
  - Length constraints may not be respected; nothing in-workflow validates tweet length.
  - If the node output field name changes (LangChain node variations), downstream parsing may break.

---

### Block 1.3 — Parsing & Deduplication
**Overview:** Parses the LLM output string into structured items (one quote per item), then attempts to remove duplicates (intended to avoid reposting content).  
**Nodes involved:** Parse Quotes, Remove Duplicates

#### Node: **Parse Quotes**
- **Type / role:** `n8n-nodes-base.function` — custom JavaScript to parse the LLM output into items.
- **Configuration (interpreted):**
  - Reads `items[0].json.text` as the raw LLM output.
  - If missing, throws: **“No output from LLM Chain. Check your Gemini node.”**
  - Removes markdown-style JSON fences: ```json … ```
  - `JSON.parse(output)` expecting an **array of strings**
  - Emits one item per quote: `{ quote: q.trim() }`
- **Connections:**
  - **Input ←** `Generate Quotes`
  - **Output →** `Remove Duplicates`
- **Version notes:** typeVersion **1**
- **Edge cases / failures:**
  - If `items[0].json.text` is not present (some LLM nodes output `response`, `output`, etc.), parsing fails immediately.
  - If Gemini returns invalid JSON, `JSON.parse` throws (workflow fails).
  - If Gemini returns an object instead of an array, mapping will break logically.
  - Quotes containing leading/trailing whitespace are trimmed (good), but embedded newlines remain unless Gemini avoids them.

#### Node: **Remove Duplicates**
- **Type / role:** `n8n-nodes-base.function` — filters out quotes already “posted”.
- **Configuration (interpreted):**
  - Builds a Set from current node input items: `new Set(items.map(i => i.json.value))`
  - Then returns: `$items("Parse Quotes").filter(i => !posted.has(i.json.quote))`
- **Connections:**
  - **Input ←** `Parse Quotes`
  - **Output →** `Loop Over Items`
- **Version notes:** typeVersion **1**
- **Important behavior note (likely bug / mismatch):**
  - Incoming items from `Parse Quotes` have `{ quote: ... }`, **not** `{ value: ... }`.
  - Therefore `items.map(i => i.json.value)` will likely produce `[undefined, undefined, ...]`, making `posted` contain only `undefined`.
  - The filter then compares `posted.has(i.json.quote)`; since set contains `undefined`, almost all quotes pass through (deduplication not effective).
- **Edge cases / failures:**
  - Expression `$items("Parse Quotes")` relies on that node existing and having run in the same execution.
  - Deduplication is not persistent across days/runs; it only compares within the current execution unless extended with storage (Data Store, DB, Google Sheet, etc.).

---

### Block 1.4 — Tweet Posting Loop with Delay
**Overview:** Iterates through the quote items and posts them to X one by one, waiting between each post to space them out.  
**Nodes involved:** Loop Over Items, Post Tweet, Wait Between Tweets

#### Node: **Loop Over Items**
- **Type / role:** `n8n-nodes-base.splitInBatches` — controls batch/iteration loop.
- **Configuration (interpreted):**
  - Uses default batching options (batch size not explicitly set in parameters shown).
  - Standard pattern:
    - **Input 0:** items to iterate
    - **Output 1 (“next batch” path):** the current item(s) to process
    - **Output 0:** signals completion (unused here)
- **Connections:**
  - **Input ←** `Remove Duplicates` and later **←** `Wait Between Tweets` (loop-back)
  - **Output (index 1) →** `Post Tweet`
  - **Output (index 0) →** not connected
- **Version notes:** typeVersion **3**
- **Edge cases / failures:**
  - If no incoming items, nothing is posted.
  - If batch size defaults unexpectedly (depending on n8n defaults), it may post more than one per iteration; typically you want batch size = 1 for tweet-by-tweet spacing.

#### Node: **Post Tweet**
- **Type / role:** `n8n-nodes-base.twitter` — posts a tweet to Twitter/X.
- **Configuration (interpreted):**
  - Tweet text: `{{$json.quote}}` (note: includes a trailing space in the template).
  - Uses **Twitter OAuth2** credentials named **“X account”**.
- **Connections:**
  - **Input ←** `Loop Over Items` (output index 1)
  - **Output →** `Wait Between Tweets`
- **Version notes:** typeVersion **2**
- **Edge cases / failures:**
  - Auth errors (revoked token, missing scopes, expired OAuth).
  - Twitter API errors:
    - Duplicate content rejection (X often rejects identical tweets)
    - Length violations (prompt aims to constrain length, but not enforced)
    - Rate limits / spam detection
  - If `quote` is missing/empty, you may post an empty tweet attempt (likely rejected by API).

#### Node: **Wait Between Tweets**
- **Type / role:** `n8n-nodes-base.wait` — pauses workflow execution before continuing loop.
- **Configuration (interpreted):**
  - No explicit wait duration configured in shown parameters (likely using node defaults or UI-configured wait settings not represented here).
  - Has a webhookId (internal resume mechanism).
- **Connections:**
  - **Input ←** `Post Tweet`
  - **Output →** `Loop Over Items` (to fetch next batch and continue)
- **Version notes:** typeVersion **1**
- **Edge cases / failures:**
  - If wait duration is not set, spacing may be 0 or behave unexpectedly depending on node configuration.
  - Wait nodes create paused executions; high volume can increase execution storage usage.
  - If n8n restarts while many waits are pending, resume behavior depends on n8n configuration.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Daily time-based entry point | — | Generate Quotes | ## Daily Motivational Tweets (Gemini + LLM Chain) … (full sticky content applies) |
| Google Gemini Chat Model | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Provides Gemini model to LangChain | — | Generate Quotes (ai_languageModel) | ## Daily Motivational Tweets (Gemini + LLM Chain) … (full sticky content applies) |
| Generate Quotes | @n8n/n8n-nodes-langchain.chainLlm | Prompt Gemini to generate 3 tweets as JSON array | Schedule Trigger; Google Gemini Chat Model | Parse Quotes | ## 1. Quote Generation |
| Parse Quotes | n8n-nodes-base.function | Parse JSON array to items `{quote}` | Generate Quotes | Remove Duplicates | ## 2. Data Processing |
| Remove Duplicates | n8n-nodes-base.function | Filter out duplicates (intended) | Parse Quotes | Loop Over Items | ## 2. Data Processing |
| Loop Over Items | n8n-nodes-base.splitInBatches | Iterate quotes one-by-one | Remove Duplicates; Wait Between Tweets | Post Tweet (output index 1) | ## 3. Tweet Posting |
| Post Tweet | n8n-nodes-base.twitter | Publish tweet to X | Loop Over Items | Wait Between Tweets | ## 3. Tweet Posting |
| Wait Between Tweets | n8n-nodes-base.wait | Delay between tweet posts; loop-back | Post Tweet | Loop Over Items | ## 3. Tweet Posting |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation / canvas note | — | — | ## Daily Motivational Tweets (Gemini + LLM Chain)… includes setup/customization guidance |
| Sticky Note1 | n8n-nodes-base.stickyNote | Section header note | — | — | ## 1. Quote Generation |
| Sticky Note2 | n8n-nodes-base.stickyNote | Section header note | — | — | ## 2. Data Processing |
| Sticky Note3 | n8n-nodes-base.stickyNote | Section header note | — | — | ## 3. Tweet Posting |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n  
   - Name it: **Daily Motivational Tweets** (or your preferred name).
   - Ensure workflow is set to **Active** when ready.

2) **Add Schedule Trigger**
   - Node: **Schedule Trigger**
   - Configure: run **daily at 09:00** (adjust hour/timezone as desired).
   - Connect **Schedule Trigger → Generate Quotes** (step 4).

3) **Add Google Gemini Chat Model**
   - Node: **Google Gemini Chat Model** (LangChain)
   - Set options:
     - **topP = 0.8**
   - Credentials:
     - Create/select **Google Gemini(PaLM) API** credential (API key / project access).
   - This node will connect to the LLM Chain node as its model provider.

4) **Add Generate Quotes (LLM Chain)**
   - Node: **LLM Chain** (LangChain `chainLlm`)
   - Prompt type: **Define**
   - Paste prompt (key requirements):
     - Generate exactly **3** motivational tweet texts with specified length ranges
     - No hashtags/emojis/quotes/markdown/lists
     - Output **ONLY** a valid **JSON array of strings**
   - Connect model:
     - Connect **Google Gemini Chat Model → Generate Quotes** via the **AI Language Model** connection.
   - Connect flow:
     - **Schedule Trigger → Generate Quotes**
     - **Generate Quotes → Parse Quotes**

5) **Add Parse Quotes (Function)**
   - Node: **Function**
   - Code logic:
     - Read LLM output from `items[0].json.text`
     - Strip ```json fences if present
     - `JSON.parse` into array
     - Return items as `{ json: { quote: ... } }`
   - Connect **Parse Quotes → Remove Duplicates**

6) **Add Remove Duplicates (Function)**
   - Node: **Function**
   - Configure it to filter duplicates.
   - Important: as provided, it references `i.json.value` (likely incorrect). To reproduce *exactly*, keep it; to make it functional, store/compare against actual posted history (Data Store/DB) or at least compare within current batch using `quote`.
   - Connect **Remove Duplicates → Loop Over Items**

7) **Add Loop Over Items (Split In Batches)**
   - Node: **Split In Batches**
   - Set **Batch Size = 1** (recommended for “post then wait then post next”).
   - Connect:
     - **Loop Over Items (output 1) → Post Tweet**
     - **Wait Between Tweets → Loop Over Items** (loop-back to continue)
   - Leave **output 0** unconnected (it indicates completion).

8) **Add Post Tweet (Twitter/X)**
   - Node: **Twitter** (operation: post tweet/status)
   - Text field: `{{$json.quote}}`
   - Credentials:
     - Set up **Twitter OAuth2** credential with an X developer app and required scopes/permissions for posting.
   - Connect **Post Tweet → Wait Between Tweets**

9) **Add Wait Between Tweets**
   - Node: **Wait**
   - Configure wait duration (e.g., 10–30 minutes) according to your posting strategy.
   - Connect **Wait Between Tweets → Loop Over Items** to continue the batch loop.

10) **(Optional but recommended) Add validation**
   - Add a node before posting to enforce:
     - tweet length
     - non-empty content
     - no accidental JSON artifacts
   - This isn’t in the provided workflow, but prevents common runtime failures.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This workflow automatically generates and posts 3 motivational quotes daily on Twitter/X using Google Gemini AI… Triggers daily… Parses… Posts each quote… Setup: connect Google Gemini API key; connect Twitter/X account; adjust Cron and Wait…” | From canvas sticky note: **Daily Motivational Tweets (Gemini + LLM Chain)** |
| Sections labeled “## 1. Quote Generation”, “## 2. Data Processing”, “## 3. Tweet Posting” | From the three colored section sticky notes |

Disclaimer (provided by user): Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.