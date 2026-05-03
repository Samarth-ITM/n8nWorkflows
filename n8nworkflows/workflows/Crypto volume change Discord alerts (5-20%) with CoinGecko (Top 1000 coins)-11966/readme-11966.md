Crypto volume change Discord alerts (5-20%) with CoinGecko (Top 1000 coins)

https://n8nworkflows.xyz/workflows/crypto-volume-change-discord-alerts--5-20---with-coingecko--top-1000-coins--11966


# Crypto volume change Discord alerts (5-20%) with CoinGecko (Top 1000 coins)

## 1. Workflow Overview

**Purpose:**  
This workflow monitors **24h trading volume changes** for the **Top ~1000 cryptocurrencies** (via CoinGecko market pages), compares the current snapshot with a previously stored snapshot, and sends **Discord webhook alerts** when volume changes exceed configured thresholds. It also updates the stored snapshot for the next run.

**Target use cases:**
- Periodic monitoring of unusual volume spikes/drops across many coins
- Separate alerting streams for **Large / Mid / Small caps** (or three cohorts) with different thresholds (5%, 10%, 20%)
- “Top movers” digest messages posted to Discord

### 1.1 Scheduling & Market Data Ingestion (Pages 1–4)
Triggered on a schedule; fetches multiple CoinGecko “markets” pages in parallel and merges them.

### 1.2 Normalization & Snapshot Read
Processes/normalizes fetched market data, then reads the previous snapshot from an n8n **Data Table**.

### 1.3 Change Computation (three thresholds)
Computes volume deltas against the stored snapshot, then routes into three parallel cohorts (labeled >5%, >10%, >20%).

### 1.4 Segmentation, Alert Decision, Top Movers, Discord Formatting & Send
For each cohort: filter a cap segment, check if alerts exist, pick top 20 movers, format Discord payload(s), and send via webhook in a loop/batch.

### 1.5 Snapshot Maintenance (delete old + insert new)
Deletes existing stored rows (snapshot) then refetches market pages and inserts the new snapshot into the Data Table (so the next run has a baseline).

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & Fetch Top-1000 Market Pages (1–4)
**Overview:** Runs on a schedule and fetches four CoinGecko pages (likely 250 coins/page) in parallel, then merges them into a single stream.  
**Nodes involved:** `Schedule Trigger`, `Fetch Page 1`, `Fetch Page 2`, `Fetch Page 3`, `Fetch Page 4`, `Merge`

#### Node: Schedule Trigger
- **Type/role:** Schedule Trigger — entry point.
- **Config (interpreted):** Schedule details are not present in JSON (parameters empty). In practice it must be set (e.g., every X minutes).
- **Outputs:** Fans out to `Fetch Page 1..4`.
- **Edge cases:** If schedule misconfigured or disabled, nothing runs.

#### Nodes: Fetch Page 1 / 2 / 3 / 4
- **Type/role:** HTTP Request — calls CoinGecko markets endpoint/pages.
- **Config:** Empty in JSON → URL, query params, headers not included. Typical setup would be CoinGecko `/coins/markets` with paging (`page=1..4`) and `per_page=250`, `vs_currency=usd`, and `order=market_cap_desc`.
- **Inputs:** From `Schedule Trigger`.
- **Outputs:** Each goes into a different input index of `Merge`.
- **Edge cases/failures:**
  - 429 rate limiting, timeouts, DNS/network errors
  - API schema changes (missing `total_volume`, `market_cap`, etc.)
  - If you rely on CoinGecko API key/Pro endpoints, auth errors may occur

#### Node: Merge
- **Type/role:** Merge — combines multiple page result sets.
- **Config:** Not shown; based on connections, it accepts **4 inputs** (indexes 0–3). Likely set to “Append” mode.
- **Input:** From `Fetch Page 1..4`
- **Output:** To `Data Processing`
- **Edge cases:** If one page fails, merge may output partial data or fail depending on merge mode.

---

### Block 2 — Normalize Current Snapshot & Load Previous Snapshot
**Overview:** Converts merged CoinGecko payload into a consistent per-coin structure, then loads stored rows from a Data Table to compare against.  
**Nodes involved:** `Data Processing`, `Get rows`

#### Node: Data Processing
- **Type/role:** Code — transforms API responses into a normalized list.
- **Config:** Code not included. Typically:
  - Flatten all pages into one array
  - Map each coin to `{ id, symbol, name, market_cap, total_volume, current_price, ... }`
  - Possibly compute helper fields (rank, cap bucket, etc.)
- **Input:** `Merge`
- **Output:** `Get rows`
- **Edge cases:**
  - Expression/code errors if input structure differs (e.g., array nesting)
  - Missing/null numeric fields causing NaN in later calculations

#### Node: Get rows
- **Type/role:** Data Table — reads previous stored snapshot rows.
- **Config:** Not shown; `executeOnce: true` indicates it only runs once per workflow execution (normal).
- **Input:** From `Data Processing`
- **Outputs:** Feeds **three** comparison nodes in parallel:
  - `Compare Volume & Calculate Changes (>5%)`
  - `Compare Volume & Calculate Changes (>10%)`
  - `Compare Volume & Calculate Changes (>20%)`
- **Edge cases:**
  - Data Table not configured (no table selected)
  - Empty table on first run → comparisons must handle “no baseline”

---

### Block 3 — Compare Volume & Calculate Changes (Three Parallel Threshold Paths)
**Overview:** For each threshold path, compares current volume vs stored volume and outputs items exceeding the threshold.  
**Nodes involved:** `Compare Volume & Calculate Changes (>5%)`, `Compare Volume & Calculate Changes (>10%)`, `Compare Volume & Calculate Changes (>20%)`

#### Node: Compare Volume & Calculate Changes (>5%)
- **Type/role:** Code — compute % change and filter for threshold.
- **Config:** Code not included; likely:
  - Join current items with stored items by coin id
  - `changePct = (currentVol - prevVol) / prevVol * 100`
  - Filter abs(changePct) >= 5
- **Input:** `Get rows`
- **Output:** `Filter For Large Caps`
- **Edge cases:** prevVol=0 or missing → division by zero; new coins not in baseline.

#### Node: Compare Volume & Calculate Changes (>10%)
- Same as above, threshold ~10%.
- **Output:** `Filter For Mid Caps`

#### Node: Compare Volume & Calculate Changes (>20%)
- Same as above, threshold ~20%.
- **Output:** `Filter For Small Caps`

---

### Block 4 — Large Caps Alert Stream (5% path)
**Overview:** Filters items to “Large Cap” cohort, checks if there are any alerts, selects top 20 movers, formats Discord messages, and posts them via webhook in a loop.  
**Nodes involved:** `Filter For Large Caps`, `Check for volume alerts`, `Top 20 volume movers`, `Format discord message`, `Loop Over Items`, `Send Large Cap Alerts to Discord`

#### Node: Filter For Large Caps
- **Type/role:** Code — cohort filter (likely by `market_cap` thresholds).
- **Input:** `Compare Volume & Calculate Changes (>5%)`
- **Output:** `Check for volume alerts`
- **Edge cases:** Missing market_cap; incorrect thresholds leading to empty or huge sets.

#### Node: Check for volume alerts
- **Type/role:** IF — gate: proceed only if any items exist.
- **Config:** Empty in JSON; typical condition is “Items exist” or `{{$items().length > 0}}`.
- **Input:** `Filter For Large Caps`
- **Output:** True → `Top 20 volume movers` (False path not connected)
- **Edge cases:** If condition mis-set, may never alert or always alert.

#### Node: Top 20 volume movers
- **Type/role:** Code — sorts by absolute % change or volume delta and keeps top 20.
- **Input:** `Check for volume alerts` (true)
- **Output:** `Format discord message`

#### Node: Format discord message
- **Type/role:** Code — builds Discord webhook payload(s).
- **Typical output:** `{ content: "...", embeds: [...] }` or per-item messages.
- **Input:** `Top 20 volume movers`
- **Output:** `Loop Over Items`
- **Edge cases:** Discord payload limits (content length, embeds, rate limits).

#### Node: Loop Over Items
- **Type/role:** Split In Batches — iterates through formatted messages.
- **Config:** Empty; default batch size may be 1.
- **Connections:**  
  - **Input 0:** from `Format discord message`  
  - **Output 1:** to `Send Large Cap Alerts to Discord`  
  - **Output 0:** unused (in JSON it’s empty), which is unusual but workable depending on how you wired it.
- **Edge cases:** Miswired output indexes can prevent sending.

#### Node: Send Large Cap Alerts to Discord
- **Type/role:** HTTP Request — sends webhook POST to Discord.
- **Config:** Empty; should be POST to Discord webhook URL, JSON body from previous node.
- **Input:** `Loop Over Items` (second output)
- **Output:** Back to `Loop Over Items` input (to continue looping).
- **Edge cases:** Discord 429 rate limit; invalid webhook URL; payload rejected (400).

---

### Block 5 — Mid Caps Alert Stream (10% path)
**Overview:** Same pattern as Large Caps, but for Mid Caps and a different threshold.  
**Nodes involved:** `Filter For Mid Caps`, `Check for volume alerts1`, `Top 20 volume movers1`, `Format discord message1`, `Loop Over Items1`, `Send Mid Caps Alerts to Discord`

- All node roles mirror Block 4.
- **Key difference:** Stream uses the “>10%” comparison output and sends via `Send Mid Caps Alerts to Discord`.

**Notable wiring detail:**  
`Send Mid Caps Alerts to Discord` outputs back to `Loop Over Items1` to continue batches (standard loop pattern).

---

### Block 6 — Small Caps Alert Stream (20% path)
**Overview:** Same pattern, but for Small Caps and a different threshold; additionally this stream triggers snapshot cleanup (delete rows) before rebuilding the stored snapshot.  
**Nodes involved:** `Filter For Small Caps`, `Check for volume alerts2`, `Top 20 volume movers2`, `Format discord message2`, `Loop Over Items2`, `Send Large Caps Alerts to Discord`, `Delete row(s)`, `No Operation, do nothing`

#### Node: Loop Over Items2 (special)
- **Type/role:** Split In Batches
- **Connections:**
  - **Output 0:** goes to `Delete row(s)` (this likely fires after batching completes, but the wiring suggests it may run immediately per batch depending on how n8n evaluates outputs)
  - **Output 1:** goes to `Send Large Caps Alerts to Discord`
- **Risk:** This is easy to get wrong: you usually want deletion **after** sending (or after formatting), not in parallel.

#### Node: Send Large Caps Alerts to Discord
- **Type/role:** HTTP Request — Discord webhook sender for this stream (name suggests “Large Caps” but connected to small-cap stream).
- **Output:** Feeds back into `Loop Over Items2` to continue loop.
- **Edge cases:** same Discord limits/rate limiting.

#### Node: Delete row(s)
- **Type/role:** Data Table — deletes existing stored snapshot rows.
- **Input:** `Loop Over Items2` output 0
- **Output:** `No Operation, do nothing`
- **Edge cases:** Wrong deletion filter could wipe unexpected rows; table selection missing.

#### Node: No Operation, do nothing
- **Type/role:** NoOp — used as a sequencing junction.
- **Config:** `executeOnce: true` (runs once).
- **Output:** Triggers the next fetch set (`Fetch Page `, `Fetch Page 8`, `Fetch Page 9`, `Fetch Page 10`) in parallel.
- **Edge cases:** If upstream never reaches this node (due to IF gating or batch wiring), snapshot won’t refresh.

---

### Block 7 — Rebuild Stored Snapshot (Pages 5/8/9/10) and Insert
**Overview:** Refetches pages (another set of 4), merges and processes them, then inserts rows into the Data Table as the new baseline snapshot.  
**Nodes involved:** `Fetch Page `, `Fetch Page 8`, `Fetch Page 9`, `Fetch Page 10`, `Merge1`, `Data Processing1`, `Insert row`

#### Nodes: Fetch Page  / Fetch Page 8 / Fetch Page 9 / Fetch Page 10
- **Type/role:** HTTP Request
- **Config:** Empty; `executeOnce: true` on all four. They only run once even if triggered multiple times.
- **Input:** From `No Operation, do nothing`
- **Output:** Into `Merge1` input indexes 0–3.

#### Node: Merge1
- **Type/role:** Merge — combines the four fetched pages.
- **Input:** The four fetch nodes above
- **Output:** `Data Processing1`

#### Node: Data Processing1
- **Type/role:** Code — normalizes data for insertion into Data Table (baseline snapshot).
- **Output:** `Insert row`
- **Edge cases:** Must match table schema (column names/types).

#### Node: Insert row
- **Type/role:** Data Table — inserts new snapshot rows.
- **Config:** Not shown; `executeOnce: false` means it will insert per item (normal).
- **Edge cases:** schema mismatch; duplicates if delete didn’t happen; table not selected.

---

### Block 8 — Sticky Notes (Documentation Layers)
**Overview:** Sticky notes exist but their content is empty in the provided JSON, so they don’t add guidance.  
**Nodes involved:** `Sticky Note`, `Sticky Note1`, `Sticky Note8`, `Sticky Note9`, `Sticky Note10`, `Sticky Note11`, `Sticky Note12`, `Sticky Note13`

- **Config:** `content: ""` for all.
- **Effect:** None at runtime.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | Schedule Trigger | Entry point / periodic run | — | Fetch Page 1, Fetch Page 2, Fetch Page 3, Fetch Page 4 |  |
| Fetch Page 1 | HTTP Request | Fetch CoinGecko markets page | Schedule Trigger | Merge |  |
| Fetch Page 2 | HTTP Request | Fetch CoinGecko markets page | Schedule Trigger | Merge |  |
| Fetch Page 3 | HTTP Request | Fetch CoinGecko markets page | Schedule Trigger | Merge |  |
| Fetch Page 4 | HTTP Request | Fetch CoinGecko markets page | Schedule Trigger | Merge |  |
| Merge | Merge | Append/merge fetched pages | Fetch Page 1/2/3/4 | Data Processing |  |
| Data Processing | Code | Normalize current market snapshot | Merge | Get rows |  |
| Get rows | Data Table | Read previous snapshot rows | Data Processing | Compare Volume & Calculate Changes (>5%), (>10%), (>20%) |  |
| Compare Volume & Calculate Changes (>5%) | Code | Compute volume deltas; filter >=5% | Get rows | Filter For Large Caps |  |
| Filter For Large Caps | Code | Segment large caps | Compare Volume & Calculate Changes (>5%) | Check for volume alerts |  |
| Check for volume alerts | IF | Gate if alerts exist | Filter For Large Caps | Top 20 volume movers |  |
| Top 20 volume movers | Code | Sort + select top 20 movers | Check for volume alerts | Format discord message |  |
| Format discord message | Code | Build Discord payload(s) | Top 20 volume movers | Loop Over Items |  |
| Loop Over Items | Split In Batches | Iterate sending messages | Format discord message | Send Large Cap Alerts to Discord |  |
| Send Large Cap Alerts to Discord | HTTP Request | POST to Discord webhook | Loop Over Items | Loop Over Items |  |
| Compare Volume & Calculate Changes (>10%) | Code | Compute volume deltas; filter >=10% | Get rows | Filter For Mid Caps |  |
| Filter For Mid Caps | Code | Segment mid caps | Compare Volume & Calculate Changes (>10%) | Check for volume alerts1 |  |
| Check for volume alerts1 | IF | Gate if alerts exist | Filter For Mid Caps | Top 20 volume movers1 |  |
| Top 20 volume movers1 | Code | Sort + select top 20 movers | Check for volume alerts1 | Format discord message1 |  |
| Format discord message1 | Code | Build Discord payload(s) | Top 20 volume movers1 | Loop Over Items1 |  |
| Loop Over Items1 | Split In Batches | Iterate sending messages | Format discord message1 | Send Mid Caps Alerts to Discord |  |
| Send Mid Caps Alerts to Discord | HTTP Request | POST to Discord webhook | Loop Over Items1 | Loop Over Items1 |  |
| Compare Volume & Calculate Changes (>20%) | Code | Compute volume deltas; filter >=20% | Get rows | Filter For Small Caps |  |
| Filter For Small Caps | Code | Segment small caps | Compare Volume & Calculate Changes (>20%) | Check for volume alerts2 |  |
| Check for volume alerts2 | IF | Gate if alerts exist | Filter For Small Caps | Top 20 volume movers2 |  |
| Top 20 volume movers2 | Code | Sort + select top 20 movers | Check for volume alerts2 | Format discord message2 |  |
| Format discord message2 | Code | Build Discord payload(s) | Top 20 volume movers2 | Loop Over Items2 |  |
| Loop Over Items2 | Split In Batches | Iterate sending + trigger snapshot refresh path | Format discord message2 | Delete row(s), Send Large Caps Alerts to Discord |  |
| Send Large Caps Alerts to Discord | HTTP Request | POST to Discord webhook | Loop Over Items2 | Loop Over Items2 |  |
| Delete row(s) | Data Table | Delete old snapshot | Loop Over Items2 | No Operation, do nothing |  |
| No Operation, do nothing | NoOp | Sequencing junction | Delete row(s) | Fetch Page , Fetch Page 8, Fetch Page 9, Fetch Page 10 |  |
| Fetch Page  | HTTP Request | Fetch CoinGecko markets page (snapshot rebuild) | No Operation, do nothing | Merge1 |  |
| Fetch Page 8 | HTTP Request | Fetch CoinGecko markets page (snapshot rebuild) | No Operation, do nothing | Merge1 |  |
| Fetch Page 9 | HTTP Request | Fetch CoinGecko markets page (snapshot rebuild) | No Operation, do nothing | Merge1 |  |
| Fetch Page 10 | HTTP Request | Fetch CoinGecko markets page (snapshot rebuild) | No Operation, do nothing | Merge1 |  |
| Merge1 | Merge | Append/merge rebuild pages | Fetch Page /8/9/10 | Data Processing1 |  |
| Data Processing1 | Code | Normalize for insertion | Merge1 | Insert row |  |
| Insert row | Data Table | Insert new snapshot rows | Data Processing1 | — |  |
| Sticky Note | Sticky Note | Comment layer | — | — |  |
| Sticky Note1 | Sticky Note | Comment layer | — | — |  |
| Sticky Note8 | Sticky Note | Comment layer | — | — |  |
| Sticky Note9 | Sticky Note | Comment layer | — | — |  |
| Sticky Note10 | Sticky Note | Comment layer | — | — |  |
| Sticky Note11 | Sticky Note | Comment layer | — | — |  |
| Sticky Note12 | Sticky Note | Comment layer | — | — |  |
| Sticky Note13 | Sticky Note | Comment layer | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create “Schedule Trigger” (Schedule Trigger node)**
   - Set interval (e.g., every 15 minutes / hourly).
   - This is the only entry point.

2) **Create 4 HTTP Request nodes for CoinGecko pages**
   - Names: `Fetch Page 1`, `Fetch Page 2`, `Fetch Page 3`, `Fetch Page 4`
   - Method: **GET**
   - URL (example using public endpoint):  
     `https://api.coingecko.com/api/v3/coins/markets`
   - Query params (example):
     - `vs_currency=usd`
     - `order=market_cap_desc`
     - `per_page=250`
     - `page=1` (then 2/3/4)
     - `sparkline=false`
     - `price_change_percentage=24h` (optional)
   - Connect `Schedule Trigger` → each fetch node.

3) **Create a “Merge” node**
   - Mode: **Append** (so arrays are concatenated)
   - Connect each Fetch Page node into Merge inputs 0..3 (any order is fine if you append).

4) **Create “Data Processing” (Code node)**
   - Purpose: flatten merged results and output **one item per coin** with at least:
     - `id` (CoinGecko id)
     - `symbol`, `name`
     - `market_cap`
     - `total_volume` (24h volume)
   - Connect: `Merge` → `Data Processing`.

5) **Create a Data Table to store baseline snapshot**
   - In n8n, create/select a **Data Table** (e.g., `coingecko_volume_snapshot`).
   - Columns recommended:
     - `id` (string, unique key ideally)
     - `symbol` (string)
     - `name` (string)
     - `market_cap` (number)
     - `total_volume` (number)
     - `timestamp` (date/time or number)
   - Add node `Get rows` (Data Table) configured to read all rows from that table.
   - Connect: `Data Processing` → `Get rows`.

6) **Create 3 Code nodes for comparisons**
   - Names:
     - `Compare Volume & Calculate Changes (>5%)`
     - `Compare Volume & Calculate Changes (>10%)`
     - `Compare Volume & Calculate Changes (>20%)`
   - Each should:
     - Build a lookup map from Data Table rows by `id`
     - For each current coin:
       - If baseline missing: decide to skip or treat as “no alert”
       - If baseline volume is 0: skip or handle safely
       - Compute `changePct`
     - Filter by threshold (>=5/10/20)
   - Connect: `Get rows` → each compare node (3 parallel outputs).

7) **Create 3 cap-segmentation Code nodes**
   - `Filter For Large Caps` connected from >5% path
   - `Filter For Mid Caps` connected from >10% path
   - `Filter For Small Caps` connected from >20% path
   - Implement market cap thresholds (example):
     - Large: `market_cap >= 10e9`
     - Mid: `1e9 <= market_cap < 10e9`
     - Small: `< 1e9`
   - Connect compare → corresponding filter node.

8) **Create 3 IF nodes to gate empty results**
   - `Check for volume alerts`, `Check for volume alerts1`, `Check for volume alerts2`
   - Condition example: “Number of items” > 0 (or equivalent)
   - Connect filter → IF.
   - Use only the **true** output to continue.

9) **Create “Top 20 volume movers” Code nodes (3)**
   - Sort by `Math.abs(changePct)` descending (or volume delta) and keep first 20.
   - Connect each IF true → top movers.

10) **Create “Format discord message” Code nodes (3)**
   - Build Discord webhook JSON.
   - Options:
     - One message containing a ranked list
     - Or one message per coin item (then batch send)
   - Connect top movers → formatter.

11) **Create “Split In Batches” nodes (3)**
   - `Loop Over Items`, `Loop Over Items1`, `Loop Over Items2`
   - Batch size: 1 (safe for Discord) or more if you bundle.
   - Connect formatter → split in batches.

12) **Create 3 Discord webhook senders (HTTP Request)**
   - `Send Large Cap Alerts to Discord`
   - `Send Mid Caps Alerts to Discord`
   - `Send Large Caps Alerts to Discord` (despite naming, it’s used in third stream)
   - Configure each:
     - Method: **POST**
     - URL: Discord webhook URL (per channel/segment)
     - Send JSON body from incoming item
     - Set `Content-Type: application/json`
   - Wire loop pattern:
     - SplitInBatches “next batch” output → HTTP Request
     - HTTP Request → SplitInBatches input (to continue)

13) **Snapshot refresh (delete then insert)**
   - Add `Delete row(s)` (Data Table) configured to delete all rows (or by table criteria).
   - Add `No Operation, do nothing` (NoOp) as a junction.
   - Wire from the third loop (`Loop Over Items2`) to deletion **after** sending is complete (recommended).
     - In your provided wiring, `Loop Over Items2` output 0 goes to delete; verify it runs when intended.

14) **Re-fetch pages for snapshot rebuild**
   - Create 4 HTTP Request nodes:
     - `Fetch Page ` (page 7 or 1—name is ambiguous; configure as needed)
     - `Fetch Page 8` (page 8)
     - `Fetch Page 9` (page 9)
     - `Fetch Page 10` (page 10)
   - Set `executeOnce` if you want them only once per run (optional).
   - Connect: `No Operation, do nothing` → all four fetch nodes.

15) **Merge + process + insert**
   - Add `Merge1` (Append mode) to combine 4 pages.
   - Add `Data Processing1` Code node to normalize into Data Table row format.
   - Add `Insert row` (Data Table) to insert each coin as a row into the snapshot table.
   - Connect: fetches → Merge1 → Data Processing1 → Insert row.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sticky notes exist but are empty in the provided workflow JSON. | No additional embedded documentation was provided. |
| The workflow depends on CoinGecko market page fetching (multiple pages) and a local Data Table snapshot to compute deltas. | Consider handling first-run behavior (no baseline) and CoinGecko 429 rate limits. |

