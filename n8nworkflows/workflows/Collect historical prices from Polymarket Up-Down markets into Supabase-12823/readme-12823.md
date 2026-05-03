Collect historical prices from Polymarket Up/Down markets into Supabase

https://n8nworkflows.xyz/workflows/collect-historical-prices-from-polymarket-up-down-markets-into-supabase-12823


# Collect historical prices from Polymarket Up/Down markets into Supabase

## 1. Workflow Overview

This workflow collects **historical price data** for Polymarket “Up/Down” *serial markets* (e.g., hourly Bitcoin Up or Down), and stores the results in **Supabase** for later SQL/analytics usage. It is split into two main execution paths:

### 1.1 Series discovery & event cataloging (run once per series)
Takes a **market slug**, resolves it to an **event**, derives the **series**, fetches **all events in the series**, and stores each event’s metadata (at least `EventId` and `Closed`) in an internal **n8n Data Table**.

### 1.2 Batch historical price extraction (rerunnable, batch-of-100)
Reads up to **100 unprocessed events**, fetches each event’s **UP/DOWN token IDs** and **end time**, skips events that are not closed, then queries the Polymarket CLOB **prices-history** endpoint and writes arrays of timestamps/prices into **Supabase**. It marks processed rows so the workflow can be rerun to continue where it stopped.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Input reception (Slug) → event → series → all series events
**Overview:** Accepts a Polymarket market slug, resolves it to an event ID, then to a series ID, fetches all events in that series, and prepares them for insertion into an n8n table.

**Nodes involved:**
- Form to include slug of the market
- Find the Event ID from the Slug
- Find the Series ID from the slug
- Find All the Events for that Series ID
- Split Events into Itens
- Register Events in Table Polymarket_Btc_1h_Event_List_1

#### Node: Form to include slug of the market
- **Type / role:** `Form Trigger` — interactive entry point to collect the slug.
- **Config:** One required field labeled: *“Slug of the Polymarket Market you want to get data”*.
- **Key variables:** Output JSON contains the slug under that exact label.
- **Connections:** Outputs to **Find the Event ID from the Slug**.
- **Failure/edge cases:** Empty/invalid slug will cause downstream HTTP calls to return 404 or unexpected payload.

#### Node: Find the Event ID from the Slug
- **Type / role:** `HTTP Request` — calls Polymarket Gamma API to resolve slug.
- **Config:** `GET https://gamma-api.polymarket.com/markets/slug/{{slug}}`
  - Uses expression:  
    `https://gamma-api.polymarket.com/markets/slug/{{ $json['Slug of the Polymarket Market you want to get data'] }}`
- **Connections:** Outputs to **Find the Series ID from the slug**.
- **Failure/edge cases:**
  - 404 if slug not found.
  - Response shape changes (expects `events[0].id` downstream).
  - Rate limiting/network errors.

#### Node: Find the Series ID from the slug
- **Type / role:** `HTTP Request` — fetches event detail to access series.
- **Config:** `GET https://gamma-api.polymarket.com/events/{{ $json.events[0].id }}`
- **Connections:** Outputs to **Find All the Events for that Series ID**.
- **Failure/edge cases:**
  - If `events` array is missing/empty, expression `events[0].id` fails.
  - Event payload must contain `series[0].id` for next step.

#### Node: Find All the Events for that Series ID
- **Type / role:** `HTTP Request` — fetches the series and all its events.
- **Config:** `GET https://gamma-api.polymarket.com/series/{{ $json.series[0].id }}`
- **Connections:** Outputs to **Split Events into Itens**.
- **Failure/edge cases:** Missing `series[0].id`, large series payload, intermittent API failures.

#### Node: Split Events into Itens
- **Type / role:** `Split Out` — converts the `events` array into one item per event.
- **Config:** Splits field `events`.
- **Connections:** Outputs to **Register Events in Table Polymarket_Btc_1h_Event_List_1**.
- **Failure/edge cases:** If `events` is not an array, node produces no items or errors.

#### Node: Register Events in Table Polymarket_Btc_1h_Event_List_1
- **Type / role:** `Data Table` — writes event metadata into an internal n8n table.
- **Config choices (interpreted):**
  - Table: `Polymarket_Btc_1h_Event_List_1`
  - Writes columns:
    - `EventId` = `{{ $json.id }}`
    - `Closed` = `{{ $json.closed }}`
  - Mapping mode: define fields explicitly.
- **Connections:** Terminal in this branch (no next node).
- **Failure/edge cases:**
  - If the table schema doesn’t contain those columns (or column types differ).
  - Potential duplicates if the table doesn’t enforce uniqueness and this is run multiple times for same series.

---

### Block 2.2 — Batch processing trigger & selecting unprocessed events
**Overview:** Starts the historical extraction path manually, and fetches up to 100 unprocessed events from the n8n table.

**Nodes involved:**
- Start Getting Prices Data
- Fetch 100 unprocessed events

#### Node: Start Getting Prices Data
- **Type / role:** `Manual Trigger` — manual entry point to start extraction.
- **Connections:** Outputs to **Fetch 100 unprocessed events**.
- **Failure/edge cases:** None (manual start).

#### Node: Fetch 100 unprocessed events
- **Type / role:** `Data Table (get)` — reads batch from internal table.
- **Config choices:**
  - Operation: `get`
  - Limit: `100`
  - Filter: `Processed` **isEmpty**
  - `executeOnce: true` (node-level setting): intended to avoid repeated re-execution behavior during a single run.
- **Connections:** Outputs to **Fetch UP/DOWN tokens and end time**.
- **Failure/edge cases:**
  - If your table uses `Processado` instead of `Processed` (note later node uses Portuguese field), this filter may never match.
  - If `Processed` exists but is boolean with default `false`, “isEmpty” won’t match; you’d need “equals false” instead.

---

### Block 2.3 — Enrich event rows with token IDs & end time; skip open markets
**Overview:** For each event, fetches Polymarket event details to extract the market’s end time and UP/DOWN token IDs, stores them back into the internal table, then filters out events that are still open.

**Nodes involved:**
- Fetch UP/DOWN tokens and end time
- Store UP/DOWN tokens and end time
- Check if market has closed
- Mark open markets as processed

#### Node: Fetch UP/DOWN tokens and end time
- **Type / role:** `HTTP Request` — gets event details from Gamma API.
- **Config:**
  - `GET https://gamma-api.polymarket.com/events/{{ $json.EventId }}`
  - `onError: continueRegularOutput` (important): workflow continues even if request fails.
- **Connections:** Outputs to **Store UP/DOWN tokens and end time**.
- **Failure/edge cases:**
  - If event row doesn’t have `EventId`, request URL becomes invalid.
  - If API errors, downstream may fail due to missing `markets[0]`.
  - Assumes `markets[0].clobTokenIds` exists and is JSON string.

#### Node: Store UP/DOWN tokens and end time
- **Type / role:** `Data Table (update)` — writes extracted fields back into the internal table.
- **Config choices:**
  - Updates row(s) where `EventId` equals `{{ $json.id }}` (note: this uses the HTTP response’s `id`)
  - Writes:
    - `MarketId` = `{{ $json.markets[0].id }}`
    - `EndTime` = `{{ $json.markets[0].endDate }}`
    - `TokenUp` = `{{ JSON.parse($json.markets[0].clobTokenIds)[0] }}`
    - `TokenDw` = `{{ JSON.parse($json.markets[0].clobTokenIds)[1] }}`
- **Connections:** Outputs to **Check if market has closed**.
- **Failure/edge cases:**
  - If `clobTokenIds` is already an array (not a JSON string), `JSON.parse()` will error.
  - If `markets` is empty, indexing `[0]` fails.
  - Filter mismatch if your Data Table’s key is not `EventId` or if the stored `EventId` differs from returned `id`.

#### Node: Check if market has closed
- **Type / role:** `IF` — decides whether to process prices or skip.
- **Condition:** `Closed` **notEquals** `"true"`
  - Left: `{{ $json.Closed }}`
  - Right: `true` (as a string)
- **Connections:**
  - **True path** → Mark open markets as processed
  - **False path** → Convert end time to Unix timestamp
- **Important logic note:** As written, this IF checks the **Data Table field `Closed`**, but after “Store UP/DOWN tokens and end time” the active item is the **update result**, which may not include `Closed`. If `Closed` is missing, the comparison may behave unexpectedly.
- **Failure/edge cases:**
  - `Closed` stored as boolean vs string `"true"` leads to mismatch.
  - Inverted logic risk: name implies “has closed”, but condition is “Closed != true”.

#### Node: Mark open markets as processed
- **Type / role:** `Data Table (update)` — sets `Processed = true` for open markets, to skip them in later batches.
- **Config:**
  - Filter: `EventId` equals `{{ $json.EventId }}`
  - Update: `Processed = true`
- **Connections:** None forward from this node (ends for open markets).
- **Failure/edge cases:**
  - If the current item doesn’t contain `EventId`, filter won’t match.
  - Field naming inconsistency: this uses `Processed`, later there is also `Processado`.

---

### Block 2.4 — Time window calculation (EndTime → unix; StartTime = EndTime - 3600)
**Overview:** Converts the market end date into Unix seconds, and computes a start timestamp (default 1 hour earlier) for querying price history.

**Nodes involved:**
- Convert end time to Unix timestamp
- Convert end time to numeric and set start time

#### Node: Convert end time to Unix timestamp
- **Type / role:** `Date & Time` — formats a date to Unix timestamp.
- **Config:**
  - Input date: `{{ $json.EndTime }}`
  - Operation: `formatDate`
  - Format: `X` (Unix seconds)
  - Output field: `EndTimeUnix`
- **Connections:** Outputs to **Convert end time to numeric and set start time**.
- **Failure/edge cases:**
  - Invalid/empty `EndTime` will fail conversion or produce null.
  - Timezone interpretation depends on input ISO string; verify Gamma returns expected format.

#### Node: Convert end time to numeric and set start time
- **Type / role:** `Set` — enforces numeric type and derives start time.
- **Config:**
  - `EndTimeUnix` (number) = `{{ $json.EndTimeUnix }}`
  - `StartTimeUnix` (number) = `{{ $json.EndTimeUnix - 3600 }}`
- **Connections:** Outputs to **Fetch price history**.
- **Failure/edge cases:**
  - If `EndTimeUnix` is a string, subtraction usually coerces, but can break if non-numeric.
  - Duration hard-coded to 1 hour; must be changed for other market durations.

---

### Block 2.5 — Fetch price history → transform arrays → write to Supabase → mark processed
**Overview:** Queries Polymarket CLOB price history for the UP token, extracts time/price arrays, inserts them into Supabase, then marks the event processed.

**Nodes involved:**
- Fetch price history
- Split prices and timestamps into arrays
- Store in Supabase
- Mark as processed

#### Node: Fetch price history
- **Type / role:** `HTTP Request` — calls Polymarket CLOB API for historical prices.
- **Config:**
  - `GET https://clob.polymarket.com/prices-history`
  - Query params:
    - `market` = `{{ $('Store UP/DOWN tokens and end time').item.json.TokenUp }}`
    - `startTs` = `{{ $json.StartTimeUnix }}`
    - `endTs` = `{{ $json.EndTimeUnix }}`
  - `onError: continueRegularOutput`
- **Connections:** Outputs to **Split prices and timestamps into arrays**.
- **Failure/edge cases:**
  - Cross-node reference `$('Store UP/DOWN tokens and end time').item.json.TokenUp` assumes item alignment; in batch/parallel runs this can mismatch.
  - If API returns empty `history`, downstream arrays will be empty.
  - API may require specific headers in future; currently public.

#### Node: Split prices and timestamps into arrays
- **Type / role:** `Set` — converts `history` array of objects into two arrays.
- **Config:**
  - `t` (array) = `{{ $json.history.map(i => i.t) }}`
  - `p` (array) = `{{ $json.history.map(i => i.p) }}`
- **Connections:** Outputs to **Store in Supabase**.
- **Failure/edge cases:** If `history` is missing/not an array, `.map` throws.

#### Node: Store in Supabase
- **Type / role:** `Supabase` — inserts a row into Supabase table.
- **Config:**
  - Credentials: Supabase API credential named **“Polymarket”**
  - Target table: `HYST_BTC_UP_DW_1H`
  - Fields mapped:
    - `price` = `{{ $json.p }}`
    - `time_of_price` = `{{ $json.p }}` (**likely a bug; should probably be `{{ $json.t }}`**)
    - `EventId` = `{{ $('Check if market has closed').item.json.EventId }}`
- **Connections:** Outputs to **Mark as processed**.
- **Failure/edge cases:**
  - Insert fails if table schema differs (array types, not-null constraints).
  - `time_of_price` is populated with prices, not times; will fail type validation if column is `TIMESTAMPTZ[]`.
  - EventId reference via `$('Check if market has closed')...` can mismatch due to item linking.

#### Node: Mark as processed
- **Type / role:** `Data Table (update)` — flags items as processed and loops back to fetch next batch.
- **Config:**
  - Updates table: `Polymarket_Btc_1h_Event_List` (**different table than earlier: `_1` vs no `_1`**)
  - Filter: `Market_Id` equals `{{ $json.MarketId }}`
  - Update: `Processado = true` (Portuguese field name)
- **Connections:** Loops back to **Fetch 100 unprocessed events**.
- **Failure/edge cases (major consistency risks):**
  - Uses a **different table** (`Polymarket_Btc_1h_Event_List`) than earlier nodes (`Polymarket_Btc_1h_Event_List_1`).
  - Uses different column names (`Market_Id`, `Processado`) vs (`MarketId`, `Processed`).
  - The current item coming from Supabase insert likely does **not** include `MarketId`, so the filter can be empty and update nothing.

---

### Block 2.6 — Sticky notes (documentation embedded in canvas)
**Overview:** Sticky notes describe purpose, usage instructions, and configuration cautions.

**Nodes involved (sticky notes):**
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4

(Sticky notes do not affect execution, but should be preserved for maintainers.)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Form to include slug of the market | Form Trigger | Collect market slug | — | Find the Event ID from the Slug | ## Fetches and stores all event IDs in a series based on a user-provided event slug. |
| Find the Event ID from the Slug | HTTP Request | Resolve slug → event | Form to include slug of the market | Find the Series ID from the slug | ## Fetches and stores all event IDs in a series based on a user-provided event slug. |
| Find the Series ID from the slug | HTTP Request | Resolve event → series | Find the Event ID from the Slug | Find All the Events for that Series ID | ## Fetches and stores all event IDs in a series based on a user-provided event slug. |
| Find All the Events for that Series ID | HTTP Request | Fetch all events in series | Find the Series ID from the slug | Split Events into Itens | ## Fetches and stores all event IDs in a series based on a user-provided event slug. |
| Split Events into Itens | Split Out | One item per event | Find All the Events for that Series ID | Register Events in Table Polymarket_Btc_1h_Event_List_1 | ## Fetches and stores all event IDs in a series based on a user-provided event slug. |
| Register Events in Table Polymarket_Btc_1h_Event_List_1 | Data Table | Store event list (EventId, Closed) | Split Events into Itens | — | ## Fetches and stores all event IDs in a series based on a user-provided event slug. |
| Start Getting Prices Data | Manual Trigger | Start batch extraction | — | Fetch 100 unprocessed events | ## Extracts price data and stores it in Supabase in batches of 100. |
| Fetch 100 unprocessed events | Data Table (get) | Read 100 unprocessed events | Start Getting Prices Data; Mark as processed | Fetch UP/DOWN tokens and end time | ## Extracts price data and stores it in Supabase in batches of 100. |
| Fetch UP/DOWN tokens and end time | HTTP Request | Get token IDs/end time from event | Fetch 100 unprocessed events | Store UP/DOWN tokens and end time | ## Extracts price data and stores it in Supabase in batches of 100. |
| Store UP/DOWN tokens and end time | Data Table (update) | Persist MarketId/EndTime/TokenUp/TokenDw | Fetch UP/DOWN tokens and end time | Check if market has closed | ## Extracts price data and stores it in Supabase in batches of 100. |
| Check if market has closed | IF | Branch: skip open vs process closed | Store UP/DOWN tokens and end time | Mark open markets as processed; Convert end time to Unix timestamp | ## Extracts price data and stores it in Supabase in batches of 100. |
| Mark open markets as processed | Data Table (update) | Mark open markets as processed to skip | Check if market has closed | — | ## Extracts price data and stores it in Supabase in batches of 100. |
| Convert end time to Unix timestamp | Date & Time | EndTime → EndTimeUnix | Check if market has closed | Convert end time to numeric and set start time | ## Extracts price data and stores it in Supabase in batches of 100. |
| Convert end time to numeric and set start time | Set | Ensure numeric; StartTimeUnix = EndTimeUnix-3600 | Convert end time to Unix timestamp | Fetch price history | ## Adjust the StartTimeUnix to match the selected market In this example, StartTimeUnix is set to 1 hour before EndTimeUnix, since the sample market used has a one-hour duration. If the market is longer or shorter, adjust this value to the appropriate time window. |
| Fetch price history | HTTP Request | Query CLOB prices-history | Convert end time to numeric and set start time | Split prices and timestamps into arrays | ## Extracts price data and stores it in Supabase in batches of 100. |
| Split prices and timestamps into arrays | Set | Create arrays t[] and p[] | Fetch price history | Store in Supabase | ## Extracts price data and stores it in Supabase in batches of 100. |
| Store in Supabase | Supabase | Insert arrays into Supabase | Split prices and timestamps into arrays | Mark as processed | ## Extracts price data and stores it in Supabase in batches of 100. |
| Mark as processed | Data Table (update) | Mark processed; loop | Store in Supabase | Fetch 100 unprocessed events | ## Extracts price data and stores it in Supabase in batches of 100. |
| Sticky Note | Sticky Note | Canvas documentation | — | — | |
| Sticky Note1 | Sticky Note | Canvas documentation | — | — | |
| Sticky Note2 | Sticky Note | Canvas documentation | — | — | |
| Sticky Note3 | Sticky Note | Canvas documentation | — | — | |
| Sticky Note4 | Sticky Note | Canvas documentation | — | — | |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Data Table for events (internal n8n table)**
   - Create an n8n **Data Table** (example name: `Polymarket_Btc_1h_Event_List_1`).
   - Include fields (recommended):
     - `EventId` (string, ideally unique)
     - `Closed` (string or boolean; choose one consistently)
     - `Processed` (boolean, nullable or default null if you want `isEmpty` filter to work)
     - `MarketId` (string)
     - `EndTime` (string/date)
     - `TokenUp` (string)
     - `TokenDw` (string)

2) **Create Supabase table**
   - In Supabase SQL editor, create (example from sticky note):
     ```sql
     CREATE TABLE HYST_BTC_UP_DW_1H (
         EventId TEXT NOT NULL,
         price NUMERIC[] NOT NULL,
         time_of_price TIMESTAMPTZ[] NOT NULL
     );
     ```
   - Ensure your insertion format matches the node output (arrays).

3) **Create node: “Form to include slug of the market” (Form Trigger)**
   - Form title: `Polymarket Slug`
   - Add a required text field labeled exactly:
     - `Slug of the Polymarket Market you want to get data`

4) **Create node: “Find the Event ID from the Slug” (HTTP Request)**
   - Method: GET
   - URL expression:
     - `https://gamma-api.polymarket.com/markets/slug/{{ $json['Slug of the Polymarket Market you want to get data'] }}`

5) **Create node: “Find the Series ID from the slug” (HTTP Request)**
   - Method: GET
   - URL expression:
     - `https://gamma-api.polymarket.com/events/{{ $json.events[0].id }}`

6) **Create node: “Find All the Events for that Series ID” (HTTP Request)**
   - Method: GET
   - URL expression:
     - `https://gamma-api.polymarket.com/series/{{ $json.series[0].id }}`

7) **Create node: “Split Events into Itens” (Split Out)**
   - Field to split out: `events`

8) **Create node: “Register Events in Table …” (Data Table → insert/upsert)**
   - Point to your events Data Table.
   - Map:
     - `EventId` = `{{$json.id}}`
     - `Closed` = `{{$json.closed}}`

9) **Create node: “Start Getting Prices Data” (Manual Trigger)**

10) **Create node: “Fetch 100 unprocessed events” (Data Table → get)**
   - Table: the same events table as step (1)
   - Limit: 100
   - Filter: `Processed` is empty (or adjust to your chosen semantics)

11) **Create node: “Fetch UP/DOWN tokens and end time” (HTTP Request)**
   - Method: GET
   - URL: `https://gamma-api.polymarket.com/events/{{ $json.EventId }}`
   - Error handling: set to **Continue on Fail** if you want batch resilience.

12) **Create node: “Store UP/DOWN tokens and end time” (Data Table → update)**
   - Table: events table
   - Filter: `EventId` equals the event id (ideally use `{{$json.id}}` from the HTTP response)
   - Set:
     - `MarketId` = `{{$json.markets[0].id}}`
     - `EndTime` = `{{$json.markets[0].endDate}}`
     - `TokenUp` = `{{ JSON.parse($json.markets[0].clobTokenIds)[0] }}`
     - `TokenDw` = `{{ JSON.parse($json.markets[0].clobTokenIds)[1] }}`

13) **Create node: “Check if market has closed” (IF)**
   - Condition based on your `Closed` type:
     - If `Closed` is boolean: check `Closed is true`
     - If string: check `Closed equals "true"`
   - Route:
     - If **open** → mark processed (skip)
     - If **closed** → continue to time conversion

14) **Create node: “Mark open markets as processed” (Data Table → update)**
   - Filter by `EventId`
   - Set `Processed = true`

15) **Create node: “Convert end time to Unix timestamp” (Date & Time)**
   - Date: `{{$json.EndTime}}`
   - Format to: Unix seconds (`X`)
   - Output: `EndTimeUnix`

16) **Create node: “Convert end time to numeric and set start time” (Set)**
   - `EndTimeUnix` as number
   - `StartTimeUnix` as number = `EndTimeUnix - 3600` (adjust for market duration)

17) **Create node: “Fetch price history” (HTTP Request)**
   - Method: GET
   - URL: `https://clob.polymarket.com/prices-history`
   - Query params:
     - `market` = TokenUp
     - `startTs` = StartTimeUnix
     - `endTs` = EndTimeUnix

18) **Create node: “Split prices and timestamps into arrays” (Set)**
   - `t` = `{{$json.history.map(i => i.t)}}`
   - `p` = `{{$json.history.map(i => i.p)}}`

19) **Create Supabase credential + node: “Store in Supabase”**
   - Create Supabase API credential (URL + service key or appropriate API key).
   - Node operation: insert row into `HYST_BTC_UP_DW_1H`
   - Fields:
     - `EventId` = current event id
     - `price` = `{{$json.p}}`
     - `time_of_price` = `{{$json.t}}` (recommended correction; the provided workflow maps `p` by mistake)

20) **Create node: “Mark as processed” (Data Table → update)**
   - Update the same events table used in step (10)
   - Filter by `EventId` (recommended)
   - Set `Processed = true`
   - Connect back to **Fetch 100 unprocessed events** to continue next batch.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Extracts price data and stores it in Supabase in batches of 100.” | Sticky note covering the batch processing path |
| “Fetches and stores all event IDs in a series based on a user-provided event slug.” | Sticky note covering the slug→series discovery path |
| Slug Example image | https://imgur.com/StX3juA.pngfull-width |
| Adjust StartTimeUnix based on market duration (example uses 1 hour) | Sticky note near time window calculation |
| Supabase table SQL (HYST_BTC_UP_DW_1H with arrays) | Embedded in sticky note text (see Block 2.6) |

--- 

### Important consistency issues to resolve if you want this to run reliably
- Standardize **one** events table name (currently `_1` vs non-`_1` are mixed).
- Standardize the processed flag and key names (`Processed` vs `Processado`, `MarketId` vs `Market_Id`, etc.).
- Fix Supabase mapping: `time_of_price` should use `t` array, not `p`.
- Avoid cross-node item references where possible (use fields carried forward in the same item) to prevent batch item misalignment.