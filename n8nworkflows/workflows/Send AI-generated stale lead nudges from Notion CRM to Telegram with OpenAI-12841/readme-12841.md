Send AI-generated stale lead nudges from Notion CRM to Telegram with OpenAI

https://n8nworkflows.xyz/workflows/send-ai-generated-stale-lead-nudges-from-notion-crm-to-telegram-with-openai-12841


# Send AI-generated stale lead nudges from Notion CRM to Telegram with OpenAI

## 1. Workflow Overview

**Purpose:** This workflow automatically detects “stale” leads in a Notion CRM (deals not edited for *X* days), generates a short AI-written nudge in a “sales manager” tone using OpenAI, and sends the nudge to the responsible sales agent via Telegram (or a fallback Telegram chat).

**Typical use cases**
- Daily sales hygiene: remind agents to follow up on neglected deals.
- Lightweight coaching: AI tone adapts to pipeline stage, inactivity, and budget.
- Team monitoring: unassigned or unmapped leads go to a general Telegram chat.

### 1.1 Entry & Scheduling
Two entry points:
- Scheduled daily run (every 24 hours)
- Manual run for testing

### 1.2 Configuration
Centralized settings node defining inactivity threshold, max alerts per run, Telegram fallback chat, and the AI “coach persona”.

### 1.3 Data Retrieval (Notion)
Fetches:
- “Agents” database pages (to map agent email → Telegram ID/handle/name)
- “Deals” database pages filtered to “active leads” (excluding closed/archived statuses)

### 1.4 Core Logic: Identify stale leads + match to agents
Code block:
- Builds an **agentMap** keyed by agent email
- Computes **days_inactive** from Notion timestamps
- Determines target Telegram chat (agent-specific or fallback)
- Sorts by most stale and limits to max alerts

### 1.5 Loop + AI + Telegram Action
For each stale lead:
- OpenAI generates a punchy nudge message
- Telegram sends an HTML-formatted alert including a Notion link
- Loop continues until all selected stale leads are processed

---

## 2. Block-by-Block Analysis

### Block A — Entry & Configuration

**Overview:** Starts the workflow either on a daily schedule or manually, then sets global configuration values used throughout the workflow.

**Nodes involved:**
- Run Every Morning
- Manual Trigger
- 📝 CONFIGURATION

#### Node: Run Every Morning
- **Type / Role:** Schedule Trigger (`n8n-nodes-base.scheduleTrigger`) — automatic entry point.
- **Configuration:** Runs every 24 hours (interval on hours field).
- **Connections:** Outputs to **📝 CONFIGURATION**.
- **Edge cases / failures:**
  - n8n instance timezone can affect “morning” expectation (it’s “every 24h” rather than “at 08:00”).
  - If the instance was down, executions won’t “catch up” unless configured externally.

#### Node: Manual Trigger
- **Type / Role:** Manual Trigger (`n8n-nodes-base.manualTrigger`) — testing entry point.
- **Configuration:** No parameters.
- **Connections:** Outputs to **📝 CONFIGURATION**.
- **Edge cases / failures:** None (only runs when manually executed in editor).

#### Node: 📝 CONFIGURATION
- **Type / Role:** Set node (`n8n-nodes-base.set`) — defines constants used by later nodes.
- **Configuration choices (interpreted):**
  - `DAYS_INACTIVE_LIMIT` (number): 7
  - `MAX_ALERTS_PER_RUN` (number): 5
  - `TELEGRAM_CHAT_ID` (string): `-1+1234567890` (placeholder-style; must be replaced with a real chat ID)
  - `COACH_PERSONA` (string): “You are a tough but fair Sales Manager.”
- **Key variables used later:**
  - `$('📝 CONFIGURATION').first().json.DAYS_INACTIVE_LIMIT`
  - `$('📝 CONFIGURATION').first().json.MAX_ALERTS_PER_RUN`
  - `$('📝 CONFIGURATION').first().json.TELEGRAM_CHAT_ID`
  - `$('📝 CONFIGURATION').first().json.COACH_PERSONA`
- **Connections:** Outputs to **📂 Get Agents**.
- **Edge cases / failures:**
  - If `TELEGRAM_CHAT_ID` is not a valid numeric chat ID (Telegram typically expects an integer), Telegram node may fail.
  - If values are missing or wrong types, downstream expressions/code may behave unexpectedly.

---

### Block B — Notion Data Sources

**Overview:** Retrieves the Agents list and filtered Deals list from Notion databases.

**Nodes involved:**
- 📂 Get Agents
- 📂 Get Active Leads

#### Node: 📂 Get Agents
- **Type / Role:** Notion node (`n8n-nodes-base.notion`) — reads all pages from the Agents database.
- **Configuration choices:**
  - Resource: **Database Page**
  - Operation: **Get All**
  - Database: selected via list UI (currently empty in the JSON export)
- **Connections:**
  - Input from **📝 CONFIGURATION**
  - Output to **📂 Get Active Leads**
- **Version-specific:** Node typeVersion `2.2` (Notion node v2 line; property shapes differ from older node versions).
- **Edge cases / failures:**
  - Missing/invalid Notion credentials (401).
  - Database ID not set (workflow will error).
  - Property naming mismatches: code later expects possible names like `Email`, `Telegram ID`, etc.

#### Node: 📂 Get Active Leads
- **Type / Role:** Notion node (`n8n-nodes-base.notion`) — reads deal pages filtered by pipeline stage.
- **Configuration choices:**
  - Resource: **Database Page**
  - Operation: **Get All**
  - Filter type: manual, match **allFilters**
  - Filters (Pipeline Stage status must NOT equal):
    - `Active User`
    - `Closed Won`
    - `Lost/Archived`
  - `executeOnce: true` (node-level setting shown in export): typically used to avoid re-fetching during testing; in production it may prevent repeated fetches in the same execution context but still fetch on each run.
- **Connections:**
  - Input from **📂 Get Agents**
  - Output to **🗓️ Filter & Map Agents**
- **Edge cases / failures:**
  - Notion status names must match exactly (“Pipeline Stage|status” + values).
  - If Notion schema differs (e.g., property not a Status), filter can return nothing or error.

---

### Block C — Logic Core (stale detection + agent mapping)

**Overview:** Builds an email→agent map from Agents DB, computes “days inactive” from each deal’s last edit timestamp, assigns each stale deal to an agent’s Telegram chat (or fallback), sorts, and limits results.

**Nodes involved:**
- 🗓️ Filter & Map Agents
- 🔄 Loop Stale Leads

#### Node: 🗓️ Filter & Map Agents
- **Type / Role:** Code node (`n8n-nodes-base.code`) — transforms datasets and filters leads.
- **Configuration choices:**
  - Uses JavaScript to:
    1. Pull all items from **📂 Get Agents**: `$('📂 Get Agents').all()`
    2. Build `agentMap` keyed by normalized email
    3. Iterate over incoming deal items (`items`) from **📂 Get Active Leads**
    4. Determine inactivity from `property_timestamp` or `last_edited_time`
    5. If `diffDays >= DAYS_INACTIVE_LIMIT`, mark as stale and attempt assignment
    6. Extract assigned manager email from `property_assigned_manager` or `Assigned Manager`
    7. If match found: set `agent_tag` and `target_chat_id`; else set `agent_tag='Unassigned'` and `target_chat_id=null`
    8. Sort by `days_inactive` descending and keep only top `MAX_ALERTS_PER_RUN`
- **Key variables / outputs added to each stale deal item:**
  - `days_inactive` (number)
  - `agent_tag` (string; e.g. `@username`, agent name, or `Unassigned`)
  - `target_chat_id` (telegram chat id or null)
- **Connections:**
  - Input from **📂 Get Active Leads**
  - Output to **🔄 Loop Stale Leads**
- **Edge cases / failures:**
  - **Property name drift:** The code checks multiple property keys (`property_email`, `Email`, etc.). If your Notion DB uses different names/types, mapping fails and agents won’t be found.
  - **Assigned Manager shape:** Notion People/Relation/Email fields can serialize differently; if email isn’t present, agent matching fails.
  - **Timestamp source:** If neither `property_timestamp` nor `last_edited_time` exists, the deal is silently ignored.
  - **Timezone/clock:** Uses JS `new Date()`; Notion timestamps are ISO strings—generally safe, but inactivity calculation may differ by ~1 day around boundaries.
  - **Telegram ID type:** extracted as `number` or text; Telegram node may require an integer-like value.

#### Node: 🔄 Loop Stale Leads
- **Type / Role:** Split In Batches (`n8n-nodes-base.splitInBatches`) — iterates over stale leads one by one (or batch-sized).
- **Configuration choices:**
  - Default batch size (not explicitly set in parameters).
- **Connections:**
  - Input from **🗓️ Filter & Map Agents**
  - **Second output** (index 1) goes to **🤖 AI Coach** (this is the “per-batch items” output).
  - After **📢 Send Nudge**, control returns to **🔄 Loop Stale Leads** to continue.
- **Edge cases / failures:**
  - If there are 0 items, downstream AI/Telegram won’t run (expected).
  - If batch size is changed, OpenAI/Telegram nodes must be able to handle multiple items per execution.

---

### Block D — AI message generation + Telegram delivery

**Overview:** For each stale lead item, generates a tailored nudge message via OpenAI and sends it to the right Telegram chat with an HTML-formatted message and a Notion link.

**Nodes involved:**
- 🤖 AI Coach
- 📢 Send Nudge

#### Node: 🤖 AI Coach
- **Type / Role:** OpenAI (LangChain) node (`@n8n/n8n-nodes-langchain.openAi`) — generates the nudge text.
- **Configuration choices:**
  - Model: `gpt-4o-mini`
  - Prompt is a single user message assembled from:
    - Persona from config: `$('📝 CONFIGURATION').first().json.COACH_PERSONA`
    - Deal context fields (company name, inactivity, budget, stage, agent tag)
    - Rules controlling tone and content
- **Key expressions used:**
  - Company name fallback chain:  
    `{{ $json.property_company_name || $json.properties?.Name?.title?.[0]?.plain_text || 'Unknown' }}`
  - Budget fallback chain:  
    `{{ $json.property_estimated_value || $json.properties?.['Estimated Value']?.number || 'Unknown' }}`
  - Stage fallback chain:  
    `{{ $json.property_pipeline_stage || $json.properties?.['Pipeline Stage']?.status?.name || 'Unknown' }}`
- **Connections:**
  - Input from **🔄 Loop Stale Leads** (per-item loop output)
  - Output to **📢 Send Nudge**
- **Version-specific:** This is the LangChain-based OpenAI node line (different from the classic OpenAI node). Ensure the n8n instance has this node available.
- **Edge cases / failures:**
  - OpenAI credential missing/invalid.
  - Model name not available for the account/region.
  - Budget comparisons in the instructions (“IF Budget > 1000”) depend on the model interpreting the number; if budget is `'Unknown'` it may behave unpredictably.

#### Node: 📢 Send Nudge
- **Type / Role:** Telegram node (`n8n-nodes-base.telegram`) — sends the final message.
- **Configuration choices:**
  - Operation: send message (implied by parameters)
  - `chatId` expression:
    - If item has `target_chat_id`, send to that
    - Else fallback to config `TELEGRAM_CHAT_ID`
    - Expression:  
      `{{ $json.target_chat_id ? $json.target_chat_id : $('📝 CONFIGURATION').first().json.TELEGRAM_CHAT_ID }}`
  - `text` uses HTML formatting, includes:
    - Header “Stale Lead Alert!”
    - The AI message from one of these possible fields: `$json.text || $json.message?.content || $json.output`
    - A Notion link using: `{{ $('🔄 Loop Stale Leads').first().json.url }}`
  - Additional fields:
    - `parse_mode: HTML`
    - `appendAttribution: false`
- **Connections:**
  - Input from **🤖 AI Coach**
  - Output back to **🔄 Loop Stale Leads** to continue processing next item
- **Edge cases / failures:**
  - Telegram auth/bot token invalid, bot not in chat, or missing permissions.
  - `TELEGRAM_CHAT_ID` format invalid.
  - Notion URL expression: `$('🔄 Loop Stale Leads').first().json.url` may be wrong if the item doesn’t contain `url` or if `.first()` does not correspond to the current loop item. In many cases you want `{{ $json.url }}` (the current item) instead.
  - HTML parsing: any unexpected characters (e.g., unescaped `<`) in AI output could break formatting.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Run Every Morning | Schedule Trigger | Scheduled entry point | — | 📝 CONFIGURATION | ## 1. Setup  \nDuplicate the Notion Template first:  \n[👉 Get the Template](https://probable-banana-3c9.notion.site/AI-Sales-Coach-System-n8n-Companion-v-1-2ee6bbcb3d0b811694b6d5ab51652670?pvs=143)  \nThen set your **Telegram Chat ID** and the **Persona** in the configuration node. |
| Manual Trigger | Manual Trigger | Manual entry point for testing | — | 📝 CONFIGURATION | ## 1. Setup  \nDuplicate the Notion Template first:  \n[👉 Get the Template](https://probable-banana-3c9.notion.site/AI-Sales-Coach-System-n8n-Companion-v-1-2ee6bbcb3d0b811694b6d5ab51652670?pvs=143)  \nThen set your **Telegram Chat ID** and the **Persona** in the configuration node. |
| 📝 CONFIGURATION | Set | Stores global settings/constants | Run Every Morning, Manual Trigger | 📂 Get Agents | ## 1. Setup  \nDuplicate the Notion Template first:  \n[👉 Get the Template](https://probable-banana-3c9.notion.site/AI-Sales-Coach-System-n8n-Companion-v-1-2ee6bbcb3d0b811694b6d5ab51652670?pvs=143)  \nThen set your **Telegram Chat ID** and the **Persona** in the configuration node. |
| 📂 Get Agents | Notion | Fetch Agents DB pages | 📝 CONFIGURATION | 📂 Get Active Leads | ## 2. Data Sources  \nFetches your **Agents list** (to find Telegram IDs) and **Active Deals** (to find stale leads). |
| 📂 Get Active Leads | Notion | Fetch active deals (excluding closed stages) | 📂 Get Agents | 🗓️ Filter & Map Agents | ## 2. Data Sources  \nFetches your **Agents list** (to find Telegram IDs) and **Active Deals** (to find stale leads). |
| 🗓️ Filter & Map Agents | Code | Map agents, compute inactivity, select top stale leads | 📂 Get Active Leads | 🔄 Loop Stale Leads | ## 3. Logic Core  \nThis code maps Leads to Agents by Email, calculates "Days Inactive", and sorts the worst offenders. |
| 🔄 Loop Stale Leads | Split In Batches | Iterate over stale leads for per-item processing | 🗓️ Filter & Map Agents; (loop) 📢 Send Nudge | 🤖 AI Coach (output 1); (loop control) | ## 3. Logic Core  \nThis code maps Leads to Agents by Email, calculates "Days Inactive", and sorts the worst offenders. |
| 🤖 AI Coach | OpenAI (LangChain) | Generate Telegram nudge text from deal context | 🔄 Loop Stale Leads | 📢 Send Nudge | ## 4. AI & Action  \nOpenAI writes a custom message based on the deal size and stage. Then sends it via Telegram. |
| 📢 Send Nudge | Telegram | Send HTML alert to agent or fallback chat | 🤖 AI Coach | 🔄 Loop Stale Leads | ## 4. AI & Action  \nOpenAI writes a custom message based on the deal size and stage. Then sends it via Telegram. |
| Sticky Note 1 | Sticky Note | Comment block | — | — | ## 1. Setup  \nDuplicate the Notion Template first:  \n[👉 Get the Template](https://probable-banana-3c9.notion.site/AI-Sales-Coach-System-n8n-Companion-v-1-2ee6bbcb3d0b811694b6d5ab51652670?pvs=143)  \nThen set your **Telegram Chat ID** and the **Persona** in the configuration node. |
| Sticky Note 2 | Sticky Note | Comment block | — | — | ## 2. Data Sources  \nFetches your **Agents list** (to find Telegram IDs) and **Active Deals** (to find stale leads). |
| Sticky Note 3 | Sticky Note | Comment block | — | — | ## 3. Logic Core  \nThis code maps Leads to Agents by Email, calculates "Days Inactive", and sorts the worst offenders. |
| Sticky Note 4 | Sticky Note | Comment block | — | — | ## 4. AI & Action  \nOpenAI writes a custom message based on the deal size and stage. Then sends it via Telegram. |
| Main Sticky | Sticky Note | Comment block | — | — | # 🤖 AI Sales Coach  \nThis workflow acts as a relentless but helpful sales manager. It wakes up every morning, scans your Notion CRM for leads that haven't been touched in X days, and uses AI to generate personalized "nudges" to the specific sales agent in charge via Telegram.  \n\n### Key Features  \n* **Smart Mapping:** Links Notion Users to Telegram IDs automatically.  \n* **Context Aware:** The AI knows if the deal is High Value or Cold, and adjusts the tone.  \n* **Direct Nudging:** Tags the specific agent responsible (@alex) instead of spamming everyone. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add Trigger (Schedule):**
   - Node: *Schedule Trigger*
   - Configure interval: every **24 hours**
   - Name it: **Run Every Morning**
3. **Add Trigger (Manual):**
   - Node: *Manual Trigger*
   - Name it: **Manual Trigger**
4. **Add Configuration node:**
   - Node: *Set*
   - Name: **📝 CONFIGURATION**
   - Add fields:
     - Number: `DAYS_INACTIVE_LIMIT` = `7`
     - Number: `MAX_ALERTS_PER_RUN` = `5`
     - String: `TELEGRAM_CHAT_ID` = *(your fallback chat id, e.g. `-100...` for groups)*
     - String: `COACH_PERSONA` = `You are a tough but fair Sales Manager.`
   - Connect **Run Every Morning → 📝 CONFIGURATION**
   - Connect **Manual Trigger → 📝 CONFIGURATION**
5. **Add Notion node for Agents:**
   - Node: *Notion*
   - Name: **📂 Get Agents**
   - Credentials: connect Notion OAuth/Token with access to the CRM pages
   - Resource: *Database Page*
   - Operation: *Get All*
   - Database: select your **Agents** database
   - Connect **📝 CONFIGURATION → 📂 Get Agents**
6. **Add Notion node for Deals:**
   - Node: *Notion*
   - Name: **📂 Get Active Leads**
   - Resource: *Database Page*
   - Operation: *Get All*
   - Database: select your **Deals** database
   - Add filters (Match: **All**):
     - `Pipeline Stage` (Status) **does not equal** `Active User`
     - `Pipeline Stage` (Status) **does not equal** `Closed Won`
     - `Pipeline Stage` (Status) **does not equal** `Lost/Archived`
   - Connect **📂 Get Agents → 📂 Get Active Leads**
7. **Add Code node to map/filter:**
   - Node: *Code*
   - Name: **🗓️ Filter & Map Agents**
   - Paste logic equivalent to:
     - Build agentMap by agent email with Telegram ID + handle/name
     - For each deal, compute inactivity from `last_edited_time` (or your chosen timestamp property)
     - Keep deals where `days_inactive >= DAYS_INACTIVE_LIMIT`
     - Extract assigned manager email and match to agentMap
     - Add `agent_tag` and `target_chat_id`
     - Sort desc by days inactive and slice to `MAX_ALERTS_PER_RUN`
   - Connect **📂 Get Active Leads → 🗓️ Filter & Map Agents**
8. **Add loop node:**
   - Node: *Split In Batches*
   - Name: **🔄 Loop Stale Leads**
   - Keep default batch size (or set to 1 explicitly to ensure one deal at a time)
   - Connect **🗓️ Filter & Map Agents → 🔄 Loop Stale Leads**
9. **Add OpenAI node (LangChain):**
   - Node: *OpenAI* (LangChain)
   - Name: **🤖 AI Coach**
   - Credentials: configure OpenAI API key in n8n
   - Model: `gpt-4o-mini`
   - Prompt/message content should include:
     - Persona from config
     - Deal context (client, days inactive, budget, stage, agent tag)
     - Rules for stage-based phrases, sarcasm, greed, and format (start with agent tag)
   - Connect **🔄 Loop Stale Leads (items output) → 🤖 AI Coach**
10. **Add Telegram send node:**
    - Node: *Telegram*
    - Name: **📢 Send Nudge**
    - Credentials: Telegram Bot token
    - Chat ID: expression that uses per-item `target_chat_id` else config fallback
    - Text: HTML message including AI output and a Notion link (prefer `{{ $json.url }}` if your deal items contain `url`)
    - Parse mode: HTML
    - Connect **🤖 AI Coach → 📢 Send Nudge**
11. **Close the loop:**
    - Connect **📢 Send Nudge → 🔄 Loop Stale Leads** (so it continues to the next item)
12. **Validate Notion properties align with the code:**
    - Agents DB must expose:
      - Email
      - Telegram ID
      - Telegram name/handle (optional)
      - Display name (optional)
    - Deals DB should expose:
      - Pipeline Stage (Status)
      - Estimated Value (Number) (optional but used)
      - Assigned Manager (must allow extracting an email)
13. **Test with Manual Trigger**, then enable the workflow.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Duplicate the Notion Template first | https://probable-banana-3c9.notion.site/AI-Sales-Coach-System-n8n-Companion-v-1-2ee6bbcb3d0b811694b6d5ab51652670?pvs=143 |
| Set your Telegram Chat ID and Persona in the configuration node | Applies to **📝 CONFIGURATION** |
| Workflow concept: “AI Sales Coach” that scans Notion daily and nudges agents via Telegram | From the “Main Sticky” description |