Track CoinMarketCap crypto sector pumps with Gemini AI and send digests to Discord

https://n8nworkflows.xyz/workflows/track-coinmarketcap-crypto-sector-pumps-with-gemini-ai-and-send-digests-to-discord-12796


# Track CoinMarketCap crypto sector pumps with Gemini AI and send digests to Discord

## 1. Workflow Overview

**Workflow name:** Track crypto narrative trends and send AI sector analysis to Discord  
**Purpose:** On a schedule, pull the top 200 24h gainers from CoinMarketCap, filter to “meaningful” movers, group them into sectors (AI, DeFi, Meme, etc.) using tag matching, ask **Google Gemini** to explain why sectors are pumping, then post a formatted multi-part digest to **Discord**.

### 1.1 Triggering
Runs automatically on a time schedule (configured as “hourly” in the workflow).

### 1.2 Data Collection & Filtering (CoinMarketCap)
Fetches market listings sorted by 24h % change, then filters tokens by gain, market cap, and volume thresholds.

### 1.3 Sector Classification & Aggregation
Assigns each token to a sector based on CoinMarketCap tags, then computes per-sector stats (count, average gain) and bundles token lists.

### 1.4 AI Research (Gemini)
Creates a research prompt per sector (only sectors with at least 2 tokens), sends it to Gemini, and collects the text analysis.

### 1.5 Digest Formatting & Delivery (Discord)
Builds a long “narrative digest” text, splits it into chunks under Discord’s message limit, and sends each part via a Discord webhook.

---

## 2. Block-by-Block Analysis

### Block 1 — Triggering

**Overview:** Starts the workflow on a recurring schedule.  
**Nodes involved:**  
- Schedule Trigger (Hourly)

#### Node: Schedule Trigger (Hourly)
- **Type / role:** `n8n-nodes-base.scheduleTrigger` — periodic workflow entry point.
- **Configuration (interpreted):** Runs on an interval schedule (the UI label says hourly; the stored rule is a generic interval object).
- **Connections:**
  - **Output →** Fetch Top 200 Gainers from CMC
- **Failure/edge cases:**
  - Misconfigured interval/timezone can cause unexpected run frequency.
  - If n8n instance is down, runs are missed (unless you implement catch-up externally).

---

### Block 2 — Data Collection & Filtering (CMC)

**Overview:** Pulls top gainers from CoinMarketCap and filters to strong/large/liquid movers.  
**Nodes involved:**  
- Fetch Top 200 Gainers from CMC  
- Filter Top Gainers (40% gain, $10M MC, $1M vol)

#### Node: Fetch Top 200 Gainers from CMC
- **Type / role:** `n8n-nodes-base.httpRequest` — HTTP GET to CoinMarketCap “listings/latest”.
- **Configuration choices:**
  - **URL:** `https://pro-api.coinmarketcap.com/v1/cryptocurrency/listings/latest`
  - **Query parameters:**
    - `limit=200`
    - `sort=percent_change_24h`
    - `sort_dir=desc`
    - `convert=USD`
  - **Auth:** “HTTP Header Auth” via n8n **generic credential type** (typically `X-CMC_PRO_API_KEY: <key>`).
- **Input/Output:**
  - **Input:** Trigger
  - **Output:** Raw CMC API response (expects `.json.data` array).
- **Failure/edge cases:**
  - 401/403 if API key missing/invalid.
  - 429 rate limits (CMC free tier credits).
  - Response shape changes or missing fields may break downstream Code nodes.

#### Node: Filter Top Gainers (40% gain, $10M MC, $1M vol)
- **Type / role:** `n8n-nodes-base.code` — transforms the CMC data array into filtered token items.
- **Key logic/config:**
  - Reads `items[0].json.data`.
  - Filters using constants:
    - `MIN_PERCENT_CHANGE = 40`
    - `MIN_MARKET_CAP = 10,000,000`
    - `MIN_VOLUME = 1,000,000`
  - Outputs one n8n item per passing token with fields:
    - `name, symbol, price, percent_change_24h, market_cap, volume_24h, tags`
- **Connections:**
  - **Input ←** Fetch Top 200 Gainers from CMC
  - **Output →** Group Tokens by Sector
- **Failure/edge cases:**
  - If CMC returns an error or unexpected payload, `items[0].json.data` may be undefined → runtime error.
  - If `token.quote.USD` is missing, access will throw.
  - Thresholds are hardcoded; lowering them can drastically increase load on Gemini and Discord message length.

---

### Block 3 — Sector Classification & Aggregation

**Overview:** Assigns each token to a sector based on tag keyword matching, then computes per-sector counts and average gains.  
**Nodes involved:**  
- Group Tokens by Sector

#### Node: Group Tokens by Sector
- **Type / role:** `n8n-nodes-base.code` — classification + aggregation.
- **Configuration choices (interpreted):**
  - Defines a `sectorMapping` dictionary, e.g.:
    - `ai`: `['ai','artificial-intelligence','ai-big-data','gpt']`
    - `defi`: `['defi','dex','lending','yield-farming','derivatives','stablecoin']`
    - `meme`: `['meme','memes','dog-themed']`
    - … plus `depin, gaming, rwa, layer1, layer2, infrastructure`
  - `determineSector(tags)` returns first sector whose keyword is a substring of any tag; otherwise `other`.
  - Creates:
    - `allTokens`: list of tokens with `.sector`
    - `sectorStats`: per sector `{ sector, count, avgGain, tokens }`
  - Sorts `sectorStats` by `count` descending.
- **Output shape (single item):**
  - `totalGainers`
  - `sectorStats` (array)
  - `allTokens` (array)
- **Connections:**
  - **Input ←** Filter Top Gainers
  - **Outputs:**
    - **Main index 0 →** Prepare Gemini Research Prompts
    - **Main index 1 →** Merge Sector Data with Analysis
- **Failure/edge cases:**
  - Tag matching is substring-based; can misclassify (e.g., “oracle” appears in multiple sectors).
  - Some CMC tokens have empty tags → “other”.
  - Because it emits **one item** containing arrays, downstream nodes must handle that structure correctly.

---

### Block 4 — AI Research (Gemini)

**Overview:** Builds one Gemini prompt per sector worth researching and calls Gemini to produce concise catalyst analysis text.  
**Nodes involved:**  
- Prepare Gemini Research Prompts  
- Gemini Sector Research  
- Format Sector Analysis Results

#### Node: Prepare Gemini Research Prompts
- **Type / role:** `n8n-nodes-base.code` — expands sectorStats into multiple prompt items.
- **Key logic/config:**
  - Reads `data = $input.first().json`, then `data.sectorStats`.
  - Selects sectors where `count >= 2` (avoids researching sectors with only 1 token).
  - For each selected sector, builds a prompt containing:
    - sector name, token count, average gain
    - bullet list of tokens and basic metrics
    - explicit questions: drivers, narrative vs coincidence, specific catalysts with dates/sources, risk, confidence
  - Outputs items like:
    - `sector`, `prompt`, `tokenCount`, `avgGain`
- **Connections:**
  - **Input ←** Group Tokens by Sector (main index 0)
  - **Output →** Gemini Sector Research
- **Failure/edge cases:**
  - If no sector has `count >= 2`, output is empty → Gemini won’t run and downstream merge/report will likely fail or be incomplete.
  - Prompt asks Gemini to “search for recent news/sources”; Gemini node may not have browsing/tools enabled, so responses may be generic unless your Gemini setup supports retrieval.

#### Node: Gemini Sector Research
- **Type / role:** `@n8n/n8n-nodes-langchain.googleGemini` — calls Google Gemini model.
- **Configuration choices:**
  - **Model:** `models/gemini-2.5-flash`
  - **Generation config:** `temperature: 0.7`, `maxOutputTokens: 1024`
  - **Message body:** uses `$json.prompt` to populate the request payload.
- **Connections:**
  - **Input ←** Prepare Gemini Research Prompts
  - **Output →** Format Sector Analysis Results
- **Version-specific notes:**
  - This is an n8n LangChain Gemini node; exact fields/options vary by n8n version and node package version.
- **Failure/edge cases:**
  - Auth/credential errors with Google AI Studio key.
  - Model name availability differs by region/project.
  - Quota/rate limits; large number of sectors increases calls.
  - Response shape assumptions downstream (expects `.json.content.parts[0].text`).

#### Node: Format Sector Analysis Results
- **Type / role:** `n8n-nodes-base.code` — converts Gemini responses into `{sector, analysis}` items.
- **Key logic/config (critical issue):**
  - Hardcodes `var sectors = ['meme', 'ai'];`
  - For each Gemini result item `i`, assigns `sector: sectors[i]` and extracts:
    - `geminiResponse = items[i].json.content.parts[0].text`
- **Connections:**
  - **Input ←** Gemini Sector Research
  - **Output →** Merge Sector Data with Analysis
- **Failure/edge cases (important):**
  - **Mismatch risk:** If Gemini returns prompts for sectors other than exactly two items, or in a different order, sector labels will be wrong or `undefined`.
  - It ignores the original `sector` field produced by “Prepare Gemini Research Prompts”, so you lose deterministic mapping.
  - If Gemini node returns a different JSON schema, `.content.parts[0].text` may be undefined.

---

### Block 5 — Merge, Digest Formatting & Discord Delivery

**Overview:** Merges sector stats with AI analysis, builds a long digest string, splits it into Discord-sized parts, then posts via webhook.  
**Nodes involved:**  
- Merge Sector Data with Analysis  
- Create Narrative Digest Report  
- Split Report for Discord (2000 char limit)  
- Send to Discord

#### Node: Merge Sector Data with Analysis
- **Type / role:** `n8n-nodes-base.merge` — combines two inputs.
- **Configuration choices:**
  - **Mode:** Combine
  - **Combine by:** Position (`combineByPosition`)
- **Connections:**
  - **Input 1 (main index 0) ←** Format Sector Analysis Results
  - **Input 2 (main index 1) ←** Group Tokens by Sector
  - **Output →** Create Narrative Digest Report
- **Failure/edge cases (important):**
  - **Cardinality mismatch:** “Group Tokens by Sector” produces **one item**, while “Format Sector Analysis Results” can produce **N items** (one per researched sector). Combine-by-position will:
    - merge item 0 with the single sectorStats item,
    - but subsequent analysis items may have no matching sectorStats item (behavior depends on n8n merge semantics/version).
  - The downstream report node uses `$input.first()`, meaning it will likely only include the **first merged pair**.

#### Node: Create Narrative Digest Report
- **Type / role:** `n8n-nodes-base.code` — renders the final digest text.
- **Key logic/config:**
  - Takes `mergedItem = $input.first().json`
  - Uses:
    - `mergedItem.sectorStats` (array of all sectors/tokens)
    - `mergedItem.analysis` (single analysis string)
    - `mergedItem.totalGainers`
  - Creates a UTC timestamp header.
  - Iterates through **all sectors** and prints performance token-by-token.
  - If `analysis` exists, prints “KEY INSIGHTS” paragraphs derived from `analysis.split('\n\n')` and filters out paragraphs containing `It appears there might be`.
  - Appends filter criteria and disclaimer.
- **Connections:**
  - **Input ←** Merge Sector Data with Analysis
  - **Output →** Split Report for Discord
- **Failure/edge cases:**
  - As written, it applies the **same single `analysis`** to every sector in the loop (because `analysis` isn’t per-sector).
  - If `mergedItem.sectorStats` is missing, the node will throw.
  - Long token lists can exceed Discord limit, requiring more splits.

#### Node: Split Report for Discord (2000 char limit)
- **Type / role:** `n8n-nodes-base.code` — chunks the digest into multiple messages.
- **Key logic/config:**
  - Uses `DISCORD_LIMIT = 1900` (safer than 2000).
  - Splits on the long `separator` line (`━━━━━━━━━...`) and rebuilds messages under the limit.
  - Outputs items with:
    - `content`, `partNumber`, `totalParts`
- **Connections:**
  - **Input ←** Create Narrative Digest Report
  - **Output →** Send to Discord
- **Failure/edge cases:**
  - If a single section exceeds 1900 chars by itself, it still may produce an oversized message (no secondary splitting strategy).
  - Separator must match exactly; if changed upstream, split logic breaks.

#### Node: Send to Discord
- **Type / role:** `n8n-nodes-base.discord` — sends messages to Discord via webhook.
- **Configuration choices:**
  - **Authentication:** webhook
  - **Content field:** `=={{ $json.content }}`
    - This appears to have an extra `==` prefix; likely intended: `={{ $json.content }}`
- **Connections:**
  - **Input ←** Split Report for Discord
  - **Output:** none
- **Failure/edge cases:**
  - Invalid webhook URL/credentials → 401/404.
  - Discord rate limits if many parts are sent quickly.
  - The `=={{ ... }}` typo may cause the literal `==` to appear or break expression parsing depending on n8n expression rules.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note - Overview | n8n-nodes-base.stickyNote | Documentation / overview text |  |  | ## 🔥 Crypto narrative trend tracker; Who is this for? Crypto traders…; Setup steps…; Configuration… |
| Sticky Note - Data Collection | n8n-nodes-base.stickyNote | Section label for CMC fetch/filter/group |  |  | ### 📥 Data Collection & Processing; Fetch gainers from CMC, filter by criteria, and group by sector |
| Sticky Note - AI Analysis | n8n-nodes-base.stickyNote | Section label for Gemini + output |  |  | ### 🤖 AI Analysis & Output; Generate sector research prompts, analyze with Gemini, and send digest to Discord |
| Schedule Trigger (Hourly) | n8n-nodes-base.scheduleTrigger | Scheduled entry point | — | Fetch Top 200 Gainers from CMC |  |
| Fetch Top 200 Gainers from CMC | n8n-nodes-base.httpRequest | Pull top 200 gainers from CoinMarketCap | Schedule Trigger (Hourly) | Filter Top Gainers (40% gain, $10M MC, $1M vol) | ### 📥 Data Collection & Processing; Fetch gainers from CMC, filter by criteria, and group by sector |
| Filter Top Gainers (40% gain, $10M MC, $1M vol) | n8n-nodes-base.code | Filter tokens by gain/market cap/volume | Fetch Top 200 Gainers from CMC | Group Tokens by Sector | ### 📥 Data Collection & Processing; Fetch gainers from CMC, filter by criteria, and group by sector |
| Group Tokens by Sector | n8n-nodes-base.code | Tag-based sector mapping + aggregation | Filter Top Gainers (40% gain, $10M MC, $1M vol) | Prepare Gemini Research Prompts; Merge Sector Data with Analysis | ### 📥 Data Collection & Processing; Fetch gainers from CMC, filter by criteria, and group by sector |
| Prepare Gemini Research Prompts | n8n-nodes-base.code | Create one Gemini prompt per sector (count ≥ 2) | Group Tokens by Sector | Gemini Sector Research | ### 🤖 AI Analysis & Output; Generate sector research prompts, analyze with Gemini, and send digest to Discord |
| Gemini Sector Research | @n8n/n8n-nodes-langchain.googleGemini | Run Gemini analysis for each sector prompt | Prepare Gemini Research Prompts | Format Sector Analysis Results | ### 🤖 AI Analysis & Output; Generate sector research prompts, analyze with Gemini, and send digest to Discord |
| Format Sector Analysis Results | n8n-nodes-base.code | Extract Gemini text and label sector | Gemini Sector Research | Merge Sector Data with Analysis | ### 🤖 AI Analysis & Output; Generate sector research prompts, analyze with Gemini, and send digest to Discord |
| Merge Sector Data with Analysis | n8n-nodes-base.merge | Combine sector stats with AI analysis | Format Sector Analysis Results; Group Tokens by Sector | Create Narrative Digest Report | ### 🤖 AI Analysis & Output; Generate sector research prompts, analyze with Gemini, and send digest to Discord |
| Create Narrative Digest Report | n8n-nodes-base.code | Render formatted digest string | Merge Sector Data with Analysis | Split Report for Discord (2000 char limit) | ### 🤖 AI Analysis & Output; Generate sector research prompts, analyze with Gemini, and send digest to Discord |
| Split Report for Discord (2000 char limit) | n8n-nodes-base.code | Split digest into <1900 char chunks | Create Narrative Digest Report | Send to Discord | ### 🤖 AI Analysis & Output; Generate sector research prompts, analyze with Gemini, and send digest to Discord |
| Send to Discord | n8n-nodes-base.discord | Post each message chunk to Discord | Split Report for Discord (2000 char limit) | — | ### 🤖 AI Analysis & Output; Generate sector research prompts, analyze with Gemini, and send digest to Discord |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **“Track crypto narrative trends and send AI sector analysis to Discord”**
   - (Optional) Add sticky notes with the provided texts for maintainability.

2. **Add node: Schedule Trigger**
   - Node type: **Schedule Trigger**
   - Configure interval to run **hourly** (or your preferred cadence).
   - Connect to the next node.

3. **Add node: HTTP Request (CoinMarketCap)**
   - Node type: **HTTP Request**
   - Method: **GET**
   - URL: `https://pro-api.coinmarketcap.com/v1/cryptocurrency/listings/latest`
   - Enable **Send Query Parameters**
   - Add query params:
     - `limit` = `200`
     - `sort` = `percent_change_24h`
     - `sort_dir` = `desc`
     - `convert` = `USD`
   - Authentication: **Header Auth**
     - Create credential with header `X-CMC_PRO_API_KEY` = your CoinMarketCap key.
   - Connect: **Schedule Trigger → HTTP Request**

4. **Add node: Code (Filter Top Gainers)**
   - Node type: **Code**
   - Paste logic that:
     - Reads `items[0].json.data`
     - Filters by `MIN_PERCENT_CHANGE=40`, `MIN_MARKET_CAP=10000000`, `MIN_VOLUME=1000000`
     - Returns one item per token with `name,symbol,price,percent_change_24h,market_cap,volume_24h,tags`
   - Connect: **HTTP Request → Filter Code**

5. **Add node: Code (Group Tokens by Sector)**
   - Node type: **Code**
   - Implement:
     - `sectorMapping` keyword arrays
     - `determineSector(tags)` substring matching
     - Compute `sectorStats` (count, avgGain, tokens), sort by count desc
     - Output **one item** with `{totalGainers, sectorStats, allTokens}`
   - Connect: **Filter Code → Group Tokens by Sector**

6. **Add node: Code (Prepare Gemini Research Prompts)**
   - Node type: **Code**
   - For each sector where `count >= 2`, create:
     - `prompt` text including tokens and the 5 research questions
     - Include `sector`, `tokenCount`, `avgGain`
   - Connect: **Group Tokens by Sector (main) → Prepare Prompts**

7. **Add node: Google Gemini (LangChain)**
   - Node type: **Google Gemini** (the `@n8n/n8n-nodes-langchain.googleGemini` node)
   - Credentials: Google AI Studio / Gemini API key
   - Model: `models/gemini-2.5-flash`
   - Temperature: `0.7`, Max tokens: `1024`
   - Message content should reference the prompt field (e.g., using `$json.prompt`).
   - Connect: **Prepare Prompts → Gemini**

8. **Add node: Code (Format Sector Analysis Results)**
   - Node type: **Code**
   - Extract Gemini text output and shape to `{sector, analysis}`.
   - Important: to make it robust, map using the original `sector` coming from the prompt item (recommended), not a hardcoded array.
   - Connect: **Gemini → Format Results**

9. **Add node: Merge**
   - Node type: **Merge**
   - Mode: **Combine**
   - Combine by: **Position**
   - Connect inputs:
     - **Input 1:** Format Sector Analysis Results → Merge
     - **Input 2:** Group Tokens by Sector → Merge (use the node’s second output connection/index if you want to preserve the original layout; functionally it’s just a second line into Merge)

10. **Add node: Code (Create Narrative Digest Report)**
    - Node type: **Code**
    - Build the digest string:
      - Header with UTC date
      - Market snapshot
      - For each sector: list tokens and metrics
      - Append AI “Key Insights”
      - Add filters/disclaimer
    - Connect: **Merge → Create Narrative Digest Report**

11. **Add node: Code (Split Report for Discord)**
    - Node type: **Code**
    - Split the report into chunks under ~1900 chars.
    - Output items with `content`.
    - Connect: **Create Narrative Digest Report → Split**

12. **Add node: Discord**
    - Node type: **Discord**
    - Authentication: **Webhook**
      - Create a Discord webhook in your server/channel and paste into n8n credentials.
    - Content: set to the expression referencing chunk text:
      - Recommended: `={{ $json.content }}`
    - Connect: **Split → Discord**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| CoinMarketCap free tier mentions “10K credits/month” and requires an API key. | Sticky Note - Overview |
| Requires a Google Gemini API key from Google AI Studio. | Sticky Note - Overview |
| Requires a Discord webhook in your server. | Sticky Note - Overview |
| Filters and sector mappings are intended to be customized in the Code nodes. | Sticky Note - Overview |

