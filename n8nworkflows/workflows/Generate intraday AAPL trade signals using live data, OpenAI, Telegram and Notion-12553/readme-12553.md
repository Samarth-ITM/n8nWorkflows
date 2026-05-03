Generate intraday AAPL trade signals using live data, OpenAI, Telegram and Notion

https://n8nworkflows.xyz/workflows/generate-intraday-aapl-trade-signals-using-live-data--openai--telegram-and-notion-12553


# Generate intraday AAPL trade signals using live data, OpenAI, Telegram and Notion

## 1. Workflow Overview

**Purpose:** Poll live intraday market data for **AAPL** every ~5 minutes, compute deterministic technical signals (trend + momentum), ask an OpenAI-powered decision agent for a strict **APPROVE / WAIT / REJECT** verdict, then **notify via Telegram** and **log to Notion**. A separate error path emails failures via Gmail.

**Target use cases:**
- Intraday monitoring and decision support (not trade execution)
- Repeatable signal evaluation with auditable indicator logic (non-AI) and rule-based decisioning (AI)

### 1.1 Market Data Polling (Twelve Data)
Schedules a recurring run and fetches the latest **5-min price/volume**, **RSI(14)**, and **EMA(20)** for AAPL.

### 1.2 Signal Computation (Deterministic)
Merges the three data streams, extracts the latest values, and computes **trend** (price vs EMA) and **momentum** (RSI thresholds).

### 1.3 AI Trade Decision (OpenAI via LangChain nodes)
Sends the computed snapshot to an AI agent constrained by a strict system policy. A structured output parser enforces JSON output.

### 1.4 Routing, Notifications, and Audit Logging
Routes by verdict (APPROVE vs non-approved) to Telegram messages and logs decisions to a Notion database.

### 1.5 Error Handling
Any node failure triggers an email alert with node name, error message, and timestamp.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Market Data Polling (AAPL 5-min)
**Overview:** Runs on a schedule and queries three endpoints (RSI, EMA, and time series) so downstream logic operates on a synchronized snapshot.

**Nodes involved:**
- Schedule Market Data Polling (AAPL 5-Min)
- Fetch AAPL 5-Minute Price & Volume Series
- Fetch AAPL 14-Period RSI (5-Minute)
- Fetch AAPL 20-Period EMA (5-Minute)1
- Merge RSI, Price, and EMA Streams

#### Node: Schedule Market Data Polling (AAPL 5-Min)
- **Type / role:** Schedule Trigger; starts workflow periodically.
- **Config (interpreted):** Interval-based schedule (intended 5 minutes per sticky note, though JSON shows an interval rule placeholder).
- **Connections:** Outputs to the three HTTP request nodes in parallel.
- **Failure modes / edge cases:**
  - Misconfigured interval (workflow runs too often / not at all).
  - Timezone considerations if later extended to market-hours logic (not implemented here).

#### Node: Fetch AAPL 5-Minute Price & Volume Series
- **Type / role:** HTTP Request; fetches latest AAPL OHLCV for 5-min candles.
- **Config (interpreted):** HTTP request with default options; URL/params not shown in JSON export (must be configured).
- **Expected output shape (implied by Code node):**
  - `json.values[0].close`
  - `json.values[0].volume`
- **Connections:** Feeds Merge input **1** (middle input).
- **Failure modes / edge cases:**
  - 401/403 (missing/invalid Twelve Data API key).
  - Rate limiting (429) on frequent polling.
  - API schema differences (e.g., values array missing/empty).
  - Time series ordering: code assumes `[0]` is the latest candle.

#### Node: Fetch AAPL 14-Period RSI (5-Minute)
- **Type / role:** HTTP Request; fetches RSI(14) for 5-min timeframe.
- **Expected output shape (implied):**
  - `json.values[0].rsi`
- **Connections:** Feeds Merge input **0** (first input).
- **Failure modes / edge cases:** Same as above; plus RSI endpoint may return strings that must parse to number.

#### Node: Fetch AAPL 20-Period EMA (5-Minute)1
- **Type / role:** HTTP Request; fetches EMA(20) for 5-min timeframe.
- **Expected output shape (implied):**
  - `json.values[0].ema`
- **Connections:** Feeds Merge input **2** (third input).
- **Failure modes / edge cases:** Same as above.

#### Node: Merge RSI, Price, and EMA Streams
- **Type / role:** Merge (multi-input); combines 3 incoming items into a single merged set for deterministic computation.
- **Config (interpreted):** `numberInputs: 3` meaning it expects three separate incoming branches.
- **Connections:** Output to “Compute Trend & Momentum Signals”.
- **Failure modes / edge cases:**
  - If any upstream branch fails or returns no item, merge may not produce the expected combined structure.
  - Data misalignment (not truly synchronized timestamps) if the API responses represent different “latest” points.

**Sticky note(s) applicable to this block:**
- “## Market Data Polling … pulls the latest AAPL price, volume, RSI, and EMA data from Twelve Data…”
- Global header note describing the workflow’s overall behavior.
- Credential/security note about using credentials/env vars (applies broadly).

---

### Block 2.2 — Signal Computation (Deterministic)
**Overview:** Converts raw indicator outputs into a compact snapshot: price, volume, RSI, EMA plus derived categorical signals for trend and momentum.

**Nodes involved:**
- Compute Trend & Momentum Signals

#### Node: Compute Trend & Momentum Signals
- **Type / role:** Code node; deterministic transformation and rule derivation.
- **Key logic (interpreted):**
  - Reads:
    - `rsi = Number(items[0].json.values[0].rsi)`
    - `candle = items[1].json.values[0]` → `price = Number(candle.close)`, `volume = Number(candle.volume)`
    - `ema = Number(items[2].json.values[0].ema)`
  - Trend:
    - `BULLISH` if `price > ema`
    - `BEARISH` if `price < ema`
    - else `NEUTRAL`
  - Momentum:
    - `BULLISH` if `rsi >= 60`
    - `BEARISH` if `rsi <= 40`
    - else `NEUTRAL`
  - Outputs one item:
    - `{ price, ema, rsi, volume, trend, momentum }`
- **Connections:** Output to AI agent “Evaluate Trade Decision from Signals (AI)”.
- **Failure modes / edge cases:**
  - `items[n].json.values[0]` missing → runtime exception.
  - Non-numeric strings → `Number()` yields `NaN` (downstream AI may still respond, but logic becomes unreliable).
  - Assumes merge input ordering: RSI is `items[0]`, price/volume is `items[1]`, EMA is `items[2]`. If merge behavior or upstream ordering changes, signals become incorrect.

**Sticky note(s):**
- “## Signal Computation … Applies deterministic logic to derive trend and momentum signals before AI evaluation.”

---

### Block 2.3 — AI Trade Decision (Rule-based via OpenAI)
**Overview:** Uses an AI agent with strict system rules to map computed signals to a decision verdict. A structured output parser constrains the output format.

**Nodes involved:**
- Evaluate Trade Decision from Signals (AI)
- LLM Engine for Trade Decision Reasoning
- Structured Trade Decision Output Parser

#### Node: LLM Engine for Trade Decision Reasoning
- **Type / role:** OpenAI Chat Model (LangChain integration).
- **Config (interpreted):**
  - Model: **gpt-4o**
  - Credentials: “OpenAi account 2”
  - Responses API disabled
- **Connections:** Connected to the Agent node via **ai_languageModel**.
- **Failure modes / edge cases:**
  - Auth/quota errors from OpenAI.
  - Model availability changes (gpt-4o access, region, or account limits).
  - Latency/timeouts during peak usage.

#### Node: Structured Trade Decision Output Parser
- **Type / role:** Structured output parser; enforces JSON schema-like structure for agent outputs.
- **Config (interpreted):**
  - Example schema indicates the agent returns an array with an object containing:
    - `verdict`, `confidence`, `reason`
- **Connections:** Connected to the Agent node via **ai_outputParser**.
- **Failure modes / edge cases:**
  - If the LLM outputs non-JSON or deviates from schema, parsing fails.
  - The example shows an **array** wrapper; downstream nodes reference `$json.output[0]`, so the agent must produce output in that shape.

#### Node: Evaluate Trade Decision from Signals (AI)
- **Type / role:** LangChain Agent; applies system rules and returns structured verdict.
- **Prompt behavior (interpreted):**
  - User prompt injects the computed snapshot: `{{ JSON.stringify($json, null, 2) }}`
  - Requires output **ONLY valid JSON**:
    ```json
    {
      "verdict": "APPROVE | WAIT | REJECT",
      "confidence": "HIGH | MEDIUM | LOW",
      "reason": "one concise sentence"
    }
    ```
  - System message enforces deterministic conservative behavior and explicit decision logic:
    - APPROVE when `trend=BULLISH` AND `volume is high` AND `momentum=BULLISH`
    - WAIT when `trend=BULLISH` AND `volume is high` AND `momentum=NEUTRAL`
    - REJECT when `trend=BEARISH`, else REJECT
- **Config notes:**
  - `maxIterations: 30` (agent can loop internally; higher cost/latency risk).
  - `hasOutputParser: true` (uses the structured parser node).
- **Connections:**
  - Main output to:
    - Route Trade Based on Verdict = APPROVE
    - Log Trade Decision to Notion (Market Signals DB)
- **Failure modes / edge cases:**
  - “Volume is high” is **not computed** in the deterministic layer; only a numeric `volume` is provided. The agent may interpret “high” inconsistently unless the prompt/system message defines a threshold (it does not).
  - Output shape expectations: downstream uses `$json.output[0].verdict`; if the agent outputs a plain object instead of an array/object structure, the IF/Telegram/Notion expressions break.
  - Agent iteration count may cause longer runtimes or increased OpenAI usage.

**Sticky note(s):**
- “## AI Trade Decision … strict, rule-based AI agent … structured trade verdict…”
- Credential/security note (OpenAI credential management).

---

### Block 2.4 — Trade Routing, Telegram Alerts, and Notion Logging
**Overview:** Logs every AI decision to Notion, and sends Telegram alerts on both approve and non-approve paths.

**Nodes involved:**
- Route Trade Based on Verdict = APPROVE
- Send Trade Alert — APPROVED Path (Telegram)
- Send Trade Alert — NON-APPROVED Path (Telegram)
- Log Trade Decision to Notion (Market Signals DB)

#### Node: Route Trade Based on Verdict = APPROVE
- **Type / role:** IF node; branches based on verdict.
- **Condition (interpreted):**
  - Checks: `{{ $json.output[0].verdict }} == "APPROVE"`
- **Connections:**
  - True branch → “Send Trade Alert — APPROVED Path (Telegram)”
  - False branch → “Send Trade Alert — NON-APPROVED Path (Telegram)”
- **Failure modes / edge cases:**
  - If `$json.output` is missing or not an array → expression error.
  - Case sensitivity: expects exactly `"APPROVE"`.

#### Node: Send Trade Alert — APPROVED Path (Telegram)
- **Type / role:** Telegram node; sends message to a chat.
- **Config (interpreted):**
  - Sends a formatted message using:
    - `{{ $json.output[0].verdict }}`
    - `{{ $json.output[0].confidence }}`
    - `{{ $json.output[0].reason }}`
  - Credentials: “saurabh Telegram account”
- **Connections:** Terminal node (no downstream).
- **Failure modes / edge cases:**
  - Bot not authorized in target chat, wrong chat ID (not shown in parameters), revoked token.
  - Telegram formatting: message begins with emoji; otherwise plain text.

#### Node: Send Trade Alert — NON-APPROVED Path (Telegram)
- **Type / role:** Telegram node; sends message for WAIT/REJECT outcomes.
- **Config / expressions:** Same structure as approved path.
- **Failure modes:** Same as above.

#### Node: Log Trade Decision to Notion (Market Signals DB)
- **Type / role:** Notion node; creates a database page to store decision outcomes.
- **Config (interpreted):**
  - Resource: **Database Page**
  - Database: “Market signal analysis” (ID provided)
  - Property mapping:
    - Notion property “Analysis data” (Title) set to:
      ```
      {{ verdict }}
      {{ confidence }}
      {{ reason }}
      ```
      via `{{ $json.output[0].verdict }}`, etc.
  - Credentials: “Notion account 2”
- **Connections:** Terminal node (in parallel with routing).
- **Failure modes / edge cases:**
  - Notion permission issues (integration not shared with the database).
  - Property name/type mismatch (“Analysis data|title” must exactly match the Notion database schema).
  - Expression failures if `output[0]` missing.

**Sticky note(s):**
- “## Trade Routing & Alerts … sends real-time alerts to Telegram…”
- “## Telegram Trade Alerts … Both approved and non-approved decisions are sent…”
- “## Audit Logging … Stores all trade decisions…”
- Credential/security note (Telegram/Notion).

---

### Block 2.5 — Error Handling (Email on Failure)
**Overview:** Any execution error triggers a Gmail email with node name, error message, and timestamp.

**Nodes involved:**
- Workflow Error Handler
- Send a message1

#### Node: Workflow Error Handler
- **Type / role:** Error Trigger; starts when the workflow fails.
- **Connections:** To Gmail send node.
- **Failure modes / edge cases:**
  - Only triggers on workflow errors; logic-level “bad data” that doesn’t throw won’t alert.

#### Node: Send a message1
- **Type / role:** Gmail node; sends error notification email.
- **Config (interpreted):**
  - To: `user@example.com` (placeholder)
  - Subject: “Workflow Error Alert”
  - Body includes:
    - `{{ $json.node.name }}`
    - `{{ $json.error.message }}`
    - `{{ $now.toISO() }}`
  - Email type: text
  - Credentials: “Gmail credentials”
- **Failure modes / edge cases:**
  - Gmail OAuth expires/revoked; domain policy blocks programmatic sending.
  - Placeholder recipient not updated.

**Sticky note(s):**
- “## Error Handling Sends alerts when the workflow fails”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Market Data Polling (AAPL 5-Min) | Schedule Trigger | Start workflow every interval | — | Fetch AAPL 20-Period EMA (5-Minute)1; Fetch AAPL 5-Minute Price & Volume Series; Fetch AAPL 14-Period RSI (5-Minute) | ## Market Data Polling  Runs on a fixed 5-minute schedule and pulls the latest AAPL price, volume, RSI, and EMA data from Twelve Data. This ensures all downstream logic works on fresh, synchronized market inputs. |
| Fetch AAPL 5-Minute Price & Volume Series | HTTP Request | Pull 5-min OHLCV series | Schedule Market Data Polling (AAPL 5-Min) | Merge RSI, Price, and EMA Streams | ## Market Data Polling  Runs on a fixed 5-minute schedule and pulls the latest AAPL price, volume, RSI, and EMA data from Twelve Data. This ensures all downstream logic works on fresh, synchronized market inputs. |
| Fetch AAPL 14-Period RSI (5-Minute) | HTTP Request | Pull RSI(14) values | Schedule Market Data Polling (AAPL 5-Min) | Merge RSI, Price, and EMA Streams | ## Market Data Polling  Runs on a fixed 5-minute schedule and pulls the latest AAPL price, volume, RSI, and EMA data from Twelve Data. This ensures all downstream logic works on fresh, synchronized market inputs. |
| Fetch AAPL 20-Period EMA (5-Minute)1 | HTTP Request | Pull EMA(20) values | Schedule Market Data Polling (AAPL 5-Min) | Merge RSI, Price, and EMA Streams | ## Market Data Polling  Runs on a fixed 5-minute schedule and pulls the latest AAPL price, volume, RSI, and EMA data from Twelve Data. This ensures all downstream logic works on fresh, synchronized market inputs. |
| Merge RSI, Price, and EMA Streams | Merge | Combine indicator streams | Fetch AAPL 14-Period RSI (5-Minute); Fetch AAPL 5-Minute Price & Volume Series; Fetch AAPL 20-Period EMA (5-Minute)1 | Compute Trend & Momentum Signals  | ## Signal Computation  Combines RSI, price, volume, and EMA into a single market snapshot. Applies deterministic logic to derive trend and momentum signals before AI evaluation. |
| Compute Trend & Momentum Signals  | Code | Compute trend/momentum labels | Merge RSI, Price, and EMA Streams | Evaluate Trade Decision from Signals (AI) | ## Signal Computation  Combines RSI, price, volume, and EMA into a single market snapshot. Applies deterministic logic to derive trend and momentum signals before AI evaluation. |
| LLM Engine for Trade Decision Reasoning | OpenAI Chat Model (LangChain) | LLM backend for agent | — | Evaluate Trade Decision from Signals (AI) (ai_languageModel) | ## AI Trade Decision  Uses a strict, rule-based AI agent to evaluate computed signals and return a structured trade verdict with confidence and a concise reasoning statement. |
| Structured Trade Decision Output Parser  | Structured Output Parser (LangChain) | Enforce structured JSON output | — | Evaluate Trade Decision from Signals (AI) (ai_outputParser) | ## AI Trade Decision  Uses a strict, rule-based AI agent to evaluate computed signals and return a structured trade verdict with confidence and a concise reasoning statement. |
| Evaluate Trade Decision from Signals (AI) | LangChain Agent | Apply decision rules to signals | Compute Trend & Momentum Signals  | Route Trade Based on Verdict = APPROVE; Log Trade Decision to Notion (Market Signals DB) | ## AI Trade Decision  Uses a strict, rule-based AI agent to evaluate computed signals and return a structured trade verdict with confidence and a concise reasoning statement. |
| Route Trade Based on Verdict = APPROVE | IF | Branch APPROVE vs other | Evaluate Trade Decision from Signals (AI) | Send Trade Alert — APPROVED Path (Telegram); Send Trade Alert — NON-APPROVED Path (Telegram) | ## Trade Routing & Alerts  Routes trades based on the AI verdict and sends real-time alerts to Telegram so decisions are visible immediately without opening n8n. |
| Send Trade Alert — APPROVED Path (Telegram) | Telegram | Notify approved verdict | Route Trade Based on Verdict = APPROVE (true) | — | ## Telegram Trade Alerts  Delivers real-time trade verdicts directly to Telegram. Both approved and non-approved decisions are sent, ensuring visibility without opening n8n. Ideal for fast reaction and continuous monitoring. |
| Send Trade Alert — NON-APPROVED Path (Telegram) | Telegram | Notify WAIT/REJECT verdict | Route Trade Based on Verdict = APPROVE (false) | — | ## Telegram Trade Alerts  Delivers real-time trade verdicts directly to Telegram. Both approved and non-approved decisions are sent, ensuring visibility without opening n8n. Ideal for fast reaction and continuous monitoring. |
| Log Trade Decision to Notion (Market Signals DB) | Notion | Persist decision for audit | Evaluate Trade Decision from Signals (AI) | — | ## Audit Logging  Stores all trade decisions for analysis and traceability. |
| Workflow Error Handler | Error Trigger | Start error workflow | — | Send a message1 | ## Error Handling  Sends alerts when the workflow fails |
| Send a message1 | Gmail | Email error details | Workflow Error Handler | — | ## Error Handling  Sends alerts when the workflow fails |
| Sticky Note | Sticky Note | Comment / documentation | — | — |  |
| Sticky Note1 | Sticky Note | Comment / documentation | — | — |  |
| Sticky Note2 | Sticky Note | Comment / documentation | — | — |  |
| Sticky Note3 | Sticky Note | Comment / documentation | — | — |  |
| Sticky Note4 | Sticky Note | Comment / documentation | — | — |  |
| Sticky Note5 | Sticky Note | Comment / documentation | — | — |  |
| Sticky Note6 | Sticky Note | Comment / documentation | — | — |  |
| Sticky Note7 | Sticky Note | Comment / documentation | — | — |  |
| Sticky Note8 | Sticky Note | Comment / documentation | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it: **“Automated Stock Trade Signals Using Live Market Data and Telegram Alerts”** (or your preferred name).

2) **Add trigger**
- Add node: **Schedule Trigger**
- Configure it for **every 5 minutes** (Cron or Interval).  
  *Important:* Ensure the schedule is actually set to 5 minutes (the provided JSON’s interval rule appears incomplete).

3) **Add market data HTTP requests (Twelve Data)**
Create three **HTTP Request** nodes and connect them from the Schedule Trigger in parallel:

3.1) **HTTP Request — Time series (price & volume)**
- Node name: “Fetch AAPL 5-Minute Price & Volume Series”
- Method: GET
- URL: Twelve Data time series endpoint (example pattern):
  - `/time_series?symbol=AAPL&interval=5min&apikey=...`
- Response: JSON
- Make sure the response includes a `values` array with latest candle at index 0 (or adjust code accordingly).

3.2) **HTTP Request — RSI(14)**
- Node name: “Fetch AAPL 14-Period RSI (5-Minute)”
- Method: GET
- URL: Twelve Data RSI endpoint (example pattern):
  - `/rsi?symbol=AAPL&interval=5min&time_period=14&apikey=...`
- Response: JSON with `values[0].rsi`.

3.3) **HTTP Request — EMA(20)**
- Node name: “Fetch AAPL 20-Period EMA (5-Minute)1”
- Method: GET
- URL: Twelve Data EMA endpoint (example pattern):
  - `/ema?symbol=AAPL&interval=5min&time_period=20&apikey=...`
- Response: JSON with `values[0].ema`.

4) **Add Merge node**
- Add node: **Merge**
- Name: “Merge RSI, Price, and EMA Streams”
- Set **Number of Inputs = 3**
- Connect:
  - RSI HTTP → Merge input 1 (index 0)
  - Time series HTTP → Merge input 2 (index 1)
  - EMA HTTP → Merge input 3 (index 2)

5) **Add Code node for deterministic signals**
- Add node: **Code**
- Name: “Compute Trend & Momentum Signals”
- Paste logic (adapt as needed) that:
  - reads `items[0]` RSI, `items[1]` candle, `items[2]` EMA
  - outputs `{ price, ema, rsi, volume, trend, momentum }`
- Connect Merge → Code.

6) **Add OpenAI model (LangChain)**
- Add node: **OpenAI Chat Model** (LangChain)
- Name: “LLM Engine for Trade Decision Reasoning”
- Choose model: **gpt-4o**
- Set OpenAI credentials in n8n:
  - Create/choose **OpenAI API credential** with your API key.

7) **Add Structured Output Parser**
- Add node: **Structured Output Parser**
- Name: “Structured Trade Decision Output Parser”
- Configure an example matching your intended structure. In this workflow, downstream expects:
  - `output[0].verdict`, `output[0].confidence`, `output[0].reason`
  so ensure the parser/agent produces an array with one object (or adjust downstream expressions).

8) **Add AI Agent node**
- Add node: **AI Agent** (LangChain agent)
- Name: “Evaluate Trade Decision from Signals (AI)”
- Connect:
  - Code node → Agent (main)
  - OpenAI Chat Model node → Agent (AI Language Model connection)
  - Structured Output Parser node → Agent (AI Output Parser connection)
- Configure:
  - Prompt includes the computed JSON snapshot (stringified).
  - System message contains the strict decision rules (as in the workflow).
  - Max iterations: set as desired (30 in the workflow).

9) **Add Notion logging**
- Add node: **Notion**
- Name: “Log Trade Decision to Notion (Market Signals DB)”
- Resource: **Database Page**
- Select your Notion database (share it with the Notion integration first).
- Map the **Title** property (named “Analysis data” in the workflow) to:
  - `{{ $json.output[0].verdict }}`
  - `{{ $json.output[0].confidence }}`
  - `{{ $json.output[0].reason }}`
- Connect Agent → Notion (in parallel with routing).

10) **Add IF routing node**
- Add node: **IF**
- Name: “Route Trade Based on Verdict = APPROVE”
- Condition: String equals:
  - Left: `{{ $json.output[0].verdict }}`
  - Right: `APPROVE`
- Connect Agent → IF.

11) **Add Telegram alerts (two nodes)**
11.1) Telegram (approved)
- Node: **Telegram**
- Name: “Send Trade Alert — APPROVED Path (Telegram)”
- Credentials: Create/select Telegram bot credential.
- Configure chat (chat ID) and message text using:
  - `{{ $json.output[0].verdict }}`, `confidence`, `reason`
- Connect IF True → Telegram (approved)

11.2) Telegram (non-approved)
- Node: **Telegram**
- Name: “Send Trade Alert — NON-APPROVED Path (Telegram)”
- Same credentials and same message template
- Connect IF False → Telegram (non-approved)

12) **Add error handling**
12.1) Error trigger
- Add node: **Error Trigger**
- Name: “Workflow Error Handler”

12.2) Gmail send
- Add node: **Gmail** → “Send Email”
- Name: “Send a message1”
- Configure OAuth2 Gmail credentials.
- To: your real email
- Subject: “Workflow Error Alert”
- Body includes:
  - `{{ $json.node.name }}`
  - `{{ $json.error.message }}`
  - `{{ $now.toISO() }}`
- Connect Error Trigger → Gmail node.

13) **Credentials checklist**
- Twelve Data: store API key in n8n credentials or environment variables; reference it in HTTP node (avoid hardcoding).
- OpenAI: API key in OpenAI credential.
- Telegram: bot token in Telegram credential; ensure bot is in the target chat.
- Notion: integration token in Notion credential; share DB with integration.
- Gmail: OAuth2 credential authorized for sending.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This workflow runs on a scheduled 5-minute interval… fetches… from Twelve Data… deterministic calculation layer… AI does not calculate indicators… logs to Notion… designed for monitoring and decision support—not autonomous trading execution.” | Sticky note: “## 📈 Automated Stock Trade Signals Using Live Market Data and Telegram Alerts” |
| “Required Credentials & Security… Use environment variables or n8n credentials for all API keys. Never hard-code secrets in HTTP nodes. Limit API permissions to read-only where possible.” | Sticky note: “## 🔐 Required Credentials & Security” |
| Disclaimer: *Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n…* | Provided in prompt (applies to the whole document) |