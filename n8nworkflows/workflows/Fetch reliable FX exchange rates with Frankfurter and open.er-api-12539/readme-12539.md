Fetch reliable FX exchange rates with Frankfurter and open.er-api

https://n8nworkflows.xyz/workflows/fetch-reliable-fx-exchange-rates-with-frankfurter-and-open-er-api-12539


# Fetch reliable FX exchange rates with Frankfurter and open.er-api

## 1. Workflow Overview

**Title:** Fetch reliable FX exchange rates with Frankfurter and open.er-api  
**Purpose:** Fetch FX rates for a configurable **base currency** and a list of **target currencies**, using **multiple public providers** sequentially. The workflow merges results into a single consistent schema and **fails if it cannot achieve full coverage** (no partial success).  
**Use cases:** pricing engines, invoicing, reporting, e-commerce multi-currency conversion, internal finance tools that require deterministic coverage.

### 1.1 Input Reception (two entry points)
- Accepts inputs either from a **Manual Trigger** (example) or as a **called sub-workflow** (Execute Workflow Trigger).

### 1.2 Input Validation & Normalization
- Validates `baseCurrency` and `currencies`.
- Normalizes to uppercase and removes the base from the target list to avoid redundant requests.

### 1.3 Initialize Internal FX State (with optional static rates)
- Creates a canonical internal `fx` object.
- Optionally injects static “always win” peg rates (currently commented).

### 1.4 Coverage Tracking + Provider 1 (Frankfurter)
- Tracks which requested currencies are missing/invalid.
- Calls Frankfurter, validates base consistency, normalizes the response, merges rates, then checks coverage.

### 1.5 Fallback Provider 2 (open.er-api.com)
- If coverage is incomplete after provider 1, calls open.er-api.com, normalizes, merges, checks coverage again.
- If still incomplete, stops with an error.

### 1.6 Optional Trim + Final Output
- If `trim` is true, returns only the base + requested currencies (and sources for those).
- Otherwise returns the full merged set.
- Ends in a NoOp node (acts as the output node).

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception

**Overview:** Provides two ways to start the workflow: manual testing with preset inputs, or invocation as a sub-workflow with passed-in inputs.

**Nodes Involved:**
- Manual Trigger (Example)
- Set Currencies
- Fetch FX Rates (Execute Workflow Trigger)

#### Node: **Manual Trigger (Example)**
- **Type / Role:** `Manual Trigger` — starts execution manually for testing.
- **Configuration:** No parameters.
- **Outputs:** Connects to **Set Currencies**.
- **Edge cases:** None; only used interactively.

#### Node: **Set Currencies**
- **Type / Role:** `Set` — provides example input payload (base currency, currency list, trim flag).
- **Configuration choices:**
  - `trim`: `false`
  - `baseCurrency`: `"AED"`
  - `currencies`: array of many ISO codes (includes `"AED"` too; validation step removes it)
- **Outputs:** Connects to **Validate & Normalize Inputs**.
- **Edge cases:**
  - If the array string were malformed (it’s stored as an array value), downstream validation would fail.

#### Node: **Fetch FX Rates**
- **Type / Role:** `Execute Workflow Trigger` — enables this workflow to be called by another workflow.
- **Configuration choices:**
  - Defines expected inputs: `trim` (boolean), `baseCurrency` (string), `currencies` (array).
- **Outputs:** Connects to **Validate & Normalize Inputs**.
- **Version-specific notes:** Requires n8n’s “Execute Workflow Trigger” capability (this is not the same as “Execute Workflow” node).
- **Edge cases:**
  - Caller may omit or mis-type inputs; handled by validation node.

---

### Block 2 — Input Validation & Normalization

**Overview:** Ensures required inputs exist and are typed correctly, uppercases codes, and removes base currency from target list.

**Nodes Involved:**
- Validate & Normalize Inputs
- If Valid Data
- Stop and Error

#### Node: **Validate & Normalize Inputs**
- **Type / Role:** `Code` — validates and normalizes incoming JSON.
- **Key logic:**
  - `trim` is taken as-is from input.
  - `baseCurrency` uppercased: `baseCurrencyRaw?.toUpperCase()`
  - Validates:
    - baseCurrency exists and is a string
    - currencies is an array
  - Uppercases all currencies and **removes baseCurrency from the currencies list** (so requested symbols exclude base).
- **Outputs:** Always outputs one item with:
  - `valid` (boolean)
  - `reason` (string if invalid)
  - `trim`, `baseCurrency`, `currencies` (filtered list)
- **Connections:** To **If Valid Data**.
- **Edge cases / failure types:**
  - If `baseCurrencyRaw` is not a string but has `.toUpperCase` (rare), could throw; current code uses optional chaining, so it becomes `undefined` and fails validation safely.
  - If currency elements are not strings, `.toUpperCase()` will throw. (This workflow assumes currency codes are strings.)

#### Node: **If Valid Data**
- **Type / Role:** `If` — gates the workflow on validation success.
- **Condition:**
  - `{{ $json.valid }}` is `true`
- **Outputs / connections:**
  - **True:** → **Initialize FX State + Static Rates**
  - **False:** → **Stop and Error**
- **Edge cases:** If `$json.valid` is missing, strict boolean true check evaluates false → error branch.

#### Node: **Stop and Error**
- **Type / Role:** `Stop and Error` — terminates workflow with a message.
- **Error message expression:** `Error: {{ $json.reason }}`
- **Inputs:** From invalid branch of **If Valid Data**.
- **Failure types:** Expression is safe as long as `reason` exists; otherwise prints `undefined`.

---

### Block 3 — Initialize Internal FX State + Coverage Baseline

**Overview:** Creates internal state `fx` with base/asOf/rates/sources and computes initial coverage (missing/invalid). Static rates can be injected and take precedence later.

**Nodes Involved:**
- Initialize FX State + Static Rates
- Initialize Coverage Tracking

#### Node: **Initialize FX State + Static Rates**
- **Type / Role:** `Code` — initializes canonical FX state and optional static rates.
- **Configuration choices / behavior:**
  - Sets `fx.base` = validated `baseCurrency` (uppercased)
  - Sets `fx.asOf` = current ISO timestamp
  - Sets `fx.rates` and `fx.sources` to empty objects
  - Contains commented example for static USD peg rates; only applies if `base === 'USD'`
- **Outputs:** Passes through prior validation fields plus `fx`.
- **Connections:** → **Initialize Coverage Tracking**
- **Edge cases:**
  - Static rates (if enabled) must be positive finite numbers; merge logic later only accepts rates `> 0`.

#### Node: **Initialize Coverage Tracking**
- **Type / Role:** `Code` — computes which requested symbols are missing/invalid in the current `fx.rates`.
- **Key expressions / references:**
  - Uses `$('Validate & Normalize Inputs').first().json` as “source of truth”
- **Logic:**
  - For each requested symbol (already excludes base), checks:
    - missing if `rate == null`
    - invalid if not finite or `<= 0`
  - Produces:
    - `fx` (current running state)
    - `missing[]`, `invalid[]`, `complete` boolean
- **Connections:** → **Frankfurter**
- **Edge cases:**
  - If validation node output is absent (e.g., node renamed), expressions break.
  - If `requestedSymbols` contains non-strings, loop still works but matching keys may fail.

---

### Block 4 — Provider 1: Frankfurter Fetch, Normalize, Merge, Coverage Check

**Overview:** Calls Frankfurter, ensures the provider’s base matches the requested base, normalizes response, merges rates without overwriting existing ones, and checks if coverage is complete.

**Nodes Involved:**
- Frankfurter
- If base correct 1
- Normalize Frankfurter
- Handle Wrong Base 1
- Merge Rates & Check Coverage (1)
- Coverage Complete 1?

#### Node: **Frankfurter**
- **Type / Role:** `HTTP Request` — fetch latest FX rates.
- **Configuration choices:**
  - URL: `https://api.frankfurter.dev/v1/latest`
  - Query parameter: `base={{ $json.fx.base }}`
  - `retryOnFail`: enabled
  - `onError`: `continueRegularOutput` (does not hard-fail; passes error response downstream)
- **Inputs:** From **Initialize Coverage Tracking** (`fx.base` is available).
- **Outputs:** Raw HTTP response JSON (or error payload depending on n8n behavior).
- **Edge cases / failure types:**
  - Network/DNS/timeouts; retry helps but not infinite.
  - API may return non-JSON or error shape; normalization handles missing `.rates`.
  - If the API ignores/changes base, the workflow detects mismatch.

#### Node: **If base correct 1**
- **Type / Role:** `If` — verifies Frankfurter response base matches requested base.
- **Condition:**
  - `{{ $json.base }} == {{ $('Initialize Coverage Tracking').first().json.fx.base }}`
- **Outputs / connections:**
  - **True:** → **Normalize Frankfurter**
  - **False:** → **Handle Wrong Base 1**
- **Edge cases:**
  - If Frankfurter returns error without `base`, condition fails → wrong-base handler, which will mark provider not ok.

#### Node: **Normalize Frankfurter**
- **Type / Role:** `Code` — transforms provider response into normalized schema.
- **Logic:**
  - If response object missing `rates`: returns `{ ok:false, provider:'frankfurter', error:'invalid_response' }`
  - Else returns:
    - `ok: true`
    - `provider: 'frankfurter'`
    - `base: res.base.toUpperCase()`
    - `asOf: res.date || null`
    - `rates: res.rates`
- **Connections:** → **Merge Rates & Check Coverage (1)**
- **Edge cases:**
  - If `res.base` is missing, `base` becomes `''` and later base check in merge prevents merging.

#### Node: **Handle Wrong Base 1**
- **Type / Role:** `Code` — produces a normalized “failed provider” payload when base mismatch happens.
- **Output:** `{ ok:false, provider:'frankfurter', error:'invalid_basecurrency' }`
- **Connections:** → **Merge Rates & Check Coverage (1)**

#### Node: **Merge Rates & Check Coverage (1)**
- **Type / Role:** `Code` — merges provider rates into running `fx` state, then recomputes coverage.
- **Key references:**
  - Requested base/symbols: `$('Validate & Normalize Inputs').first().json...`
  - Running FX state: `$('Initialize Coverage Tracking').first().json.fx`
  - Provider normalized payload: `$input.first().json`
- **Merge rules:**
  - Merge only if `provider.ok` and `provider.base === requestedBase`
  - Accept rate only if finite number and `> 0`
  - **Do not overwrite** existing rates (`static wins if already set`)
  - Record `sources[sym] = provider.provider` for merged symbols
- **Outputs:**
  - `fx: { base, asOf, rates, sources }`
  - `complete`, `missing[]`, `invalid[]`
- **Connections:** → **Coverage Complete 1?**
- **Edge cases:**
  - If provider returned rates as strings, they are ignored (not numbers).
  - If requested symbol is not present in provider output, remains missing.

#### Node: **Coverage Complete 1?**
- **Type / Role:** `If` — decides whether to stop early or call fallback provider.
- **Condition:** `{{ $json.complete }}` is `true`
- **Outputs / connections:**
  - **True:** → **If Trim**
  - **False:** → **open.er-api.com**
- **Edge cases:** If `complete` missing, goes to fallback.

---

### Block 5 — Provider 2: open.er-api.com Fetch, Normalize, Merge, Coverage Check

**Overview:** Fallback provider queried only if coverage is incomplete after Frankfurter. Same normalization/merge pattern; if still incomplete, workflow fails.

**Nodes Involved:**
- open.er-api.com
- If base correct  2
- Normalize open.er-api.com
- Handle Wrong Base 2
- Merge Rates & Check Coverage (2)
- Coverage Complete 2?
- Stop and Error3

#### Node: **open.er-api.com**
- **Type / Role:** `HTTP Request` — fetch latest rates for base.
- **Configuration choices:**
  - URL expression: `https://open.er-api.com/v6/latest/{{$json.fx.base}}`
  - `retryOnFail`: enabled
  - `onError`: `continueRegularOutput`
- **Inputs:** From **Coverage Complete 1?** false branch, carrying `fx.base`.
- **Connections:** → **If base correct  2**
- **Edge cases:**
  - Provider may return `result: "error"`; normalization handles this.
  - Rate limit/temporary blocks; will pass downstream as error-shaped response.

#### Node: **If base correct  2**
- **Type / Role:** `If` — verifies base matches requested base.
- **Condition:**
  - `{{ $json.base_code }} == {{ $('Merge Rates & Check Coverage (1)').first().json.fx.base }}`
- **Outputs / connections:**
  - **True:** → **Normalize open.er-api.com**
  - **False:** → **Handle Wrong Base 2**
- **Edge cases:** If error response lacks `base_code`, it routes to wrong-base handler.

#### Node: **Normalize open.er-api.com**
- **Type / Role:** `Code` — transforms response to normalized schema.
- **Logic:**
  - If `res.result !== 'success'` or missing `rates`: returns `{ ok:false, provider:'open.er-api.com', error: res.error_type || 'invalid_response' }`
  - Else returns:
    - `ok:true`
    - `provider:'open.er-api.com'`
    - `base: res.base_code.toUpperCase()`
    - `asOf: res.time_last_update_utc || res.time_last_update_unix || null`
    - `rates: res.rates`
- **Connections:** → **Merge Rates & Check Coverage (2)**
- **Edge cases:** If `rates` include base itself; merge code permits, but does not overwrite existing ones.

#### Node: **Handle Wrong Base 2**
- **Type / Role:** `Code` — emits normalized failure payload due to base mismatch.
- **Output:** `{ ok:false, provider:'open.er-api.com', error:'invalid_basecurrency' }`
- **Connections:** → **Merge Rates & Check Coverage (2)**

#### Node: **Merge Rates & Check Coverage (2)**
- **Type / Role:** `Code` — merges fallback provider rates into FX state from step 1 merge.
- **Key references:**
  - Running FX: `$('Merge Rates & Check Coverage (1)').first().json.fx`
  - Requested base/symbols from validation node
- **Merge rules:** Same as merge (1): only merge if `ok` and base matches; never overwrite existing.
- **Outputs:** `fx`, `complete`, `missing`, `invalid`
- **Connections:** → **Coverage Complete 2?**
- **Edge cases:** Same as merge (1).

#### Node: **Coverage Complete 2?**
- **Type / Role:** `If` — final coverage decision.
- **Condition:** `{{ $json.complete }}` is `true`
- **Outputs / connections:**
  - **True:** → **If Trim**
  - **False:** → **Stop and Error3**

#### Node: **Stop and Error3**
- **Type / Role:** `Stop and Error` — fails workflow if full coverage is still not reached.
- **Message:** `Unable to fetch all currencies.`
- **Edge cases:** None.

---

### Block 6 — Optional Trim and Final Output

**Overview:** If `trim` is enabled, returns only the requested base + requested currencies (and their sources). Otherwise passes full merged FX object.

**Nodes Involved:**
- If Trim
- Trim
- Final Output

#### Node: **If Trim**
- **Type / Role:** `If` — controls trimming.
- **Condition:**
  - `{{ $('Validate & Normalize Inputs').first().json.trim }}` is `true`
- **Outputs / connections:**
  - **True:** → **Trim**
  - **False:** → **Final Output**
- **Edge cases:** If `trim` is undefined, it routes to Final Output (no trim).

#### Node: **Trim**
- **Type / Role:** `Code` — enforces output to include only requested currencies.
- **Key references:**
  - Requested base/currencies from `Validate & Normalize Inputs`
  - Current merged FX from `$input.first().json.fx`
- **Logic:**
  - Always includes base with rate `1`
  - Includes only requested currencies that have numeric rates
  - Trims `sources` to only included symbols
  - Keeps `fx.asOf`
- **Connections:** → **Final Output**
- **Edge cases:**
  - If a requested currency is missing (should not happen if coverage is complete), it will be omitted silently—however the workflow should have errored earlier if incomplete.

#### Node: **Final Output**
- **Type / Role:** `NoOp` — acts as terminal/output node.
- **Inputs:** From either **If Trim** false branch or **Trim**.
- **Edge cases:** None.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Manual Trigger (Example) | Manual Trigger | Manual test entry point | — | Set Currencies | ## Inputs; * `trim` (boolean) * `baseCurrency` (string, e.g. "EUR") * `currencies` (array of string ISO currency codes, e.g ["USD","EUR","IDR","AED"]); Use Manual Trigger or Call from another workflow |
| Set Currencies | Set | Example input builder | Manual Trigger (Example) | Validate & Normalize Inputs | ## Inputs; * `trim` (boolean) * `baseCurrency` (string, e.g. "EUR") * `currencies` (array of string ISO currency codes, e.g ["USD","EUR","IDR","AED"]); Use Manual Trigger or Call from another workflow |
| Fetch FX Rates | Execute Workflow Trigger | Sub-workflow entry point | — | Validate & Normalize Inputs | ## Inputs; * `trim` (boolean) * `baseCurrency` (string, e.g. "EUR") * `currencies` (array of string ISO currency codes, e.g ["USD","EUR","IDR","AED"]); Use Manual Trigger or Call from another workflow |
| Validate & Normalize Inputs | Code | Validate inputs, uppercase, remove base from targets | Set Currencies; Fetch FX Rates | If Valid Data | # Multi-provider FX Rates Fetcher (workflow description block) |
| If Valid Data | If | Gate execution on validation | Validate & Normalize Inputs | Initialize FX State + Static Rates; Stop and Error | # Multi-provider FX Rates Fetcher (workflow description block) |
| Stop and Error | Stop and Error | Fail on invalid inputs | If Valid Data (false) | — | # Multi-provider FX Rates Fetcher (workflow description block) |
| Initialize FX State + Static Rates | Code | Create `fx` state; optional static rates | If Valid Data (true) | Initialize Coverage Tracking | ## Optional static rates * You can define fixed FX rates here. Static rates always override provider results. e.g. if (base === 'USD') { staticRates.AED = 3.6725; staticMeta.AED = 'static_usd_peg'; } |
| Initialize Coverage Tracking | Code | Compute missing/invalid coverage baseline | Initialize FX State + Static Rates | Frankfurter | # Multi-provider FX Rates Fetcher (workflow description block) |
| Frankfurter | HTTP Request | Provider 1 fetch | Initialize Coverage Tracking | If base correct 1 | # Multi-provider FX Rates Fetcher (workflow description block) |
| If base correct 1 | If | Validate provider base matches requested | Frankfurter | Normalize Frankfurter; Handle Wrong Base 1 | # Multi-provider FX Rates Fetcher (workflow description block) |
| Normalize Frankfurter | Code | Normalize Frankfurter schema | If base correct 1 (true) | Merge Rates & Check Coverage (1) | # Multi-provider FX Rates Fetcher (workflow description block) |
| Handle Wrong Base 1 | Code | Normalize failure for base mismatch | If base correct 1 (false) | Merge Rates & Check Coverage (1) | # Multi-provider FX Rates Fetcher (workflow description block) |
| Merge Rates & Check Coverage (1) | Code | Merge provider 1 rates, recompute coverage | Normalize Frankfurter; Handle Wrong Base 1 | Coverage Complete 1? | # Multi-provider FX Rates Fetcher (workflow description block) |
| Coverage Complete 1? | If | Early stop if complete else fallback | Merge Rates & Check Coverage (1) | If Trim; open.er-api.com | # Multi-provider FX Rates Fetcher (workflow description block) |
| open.er-api.com | HTTP Request | Provider 2 fetch (fallback) | Coverage Complete 1? (false) | If base correct  2 | ## Optional expand providers * Clone this nodes to expand your fetcher with another provider and connect the clones to Stop and Error3 node. |
| If base correct  2 | If | Validate fallback provider base | open.er-api.com | Normalize open.er-api.com; Handle Wrong Base 2 | ## Optional expand providers * Clone this nodes to expand your fetcher with another provider and connect the clones to Stop and Error3 node. |
| Normalize open.er-api.com | Code | Normalize open.er-api.com schema | If base correct  2 (true) | Merge Rates & Check Coverage (2) | ## Optional expand providers * Clone this nodes to expand your fetcher with another provider and connect the clones to Stop and Error3 node. |
| Handle Wrong Base 2 | Code | Normalize failure for base mismatch | If base correct  2 (false) | Merge Rates & Check Coverage (2) | ## Optional expand providers * Clone this nodes to expand your fetcher with another provider and connect the clones to Stop and Error3 node. |
| Merge Rates & Check Coverage (2) | Code | Merge provider 2 rates, recompute coverage | Normalize open.er-api.com; Handle Wrong Base 2 | Coverage Complete 2? | ## Optional expand providers * Clone this nodes to expand your fetcher with another provider and connect the clones to Stop and Error3 node. |
| Coverage Complete 2? | If | Final completeness check | Merge Rates & Check Coverage (2) | If Trim; Stop and Error3 | ## Optional expand providers * Clone this nodes to expand your fetcher with another provider and connect the clones to Stop and Error3 node. |
| Stop and Error3 | Stop and Error | Fail if still incomplete | Coverage Complete 2? (false) | — | ## Optional expand providers * Clone this nodes to expand your fetcher with another provider and connect the clones to Stop and Error3 node. |
| If Trim | If | Decide whether to trim output | Coverage Complete 1? (true); Coverage Complete 2? (true) | Trim; Final Output | # Multi-provider FX Rates Fetcher (workflow description block) |
| Trim | Code | Keep only requested currencies and base | If Trim (true) | Final Output | # Multi-provider FX Rates Fetcher (workflow description block) |
| Final Output | NoOp | Terminal output | If Trim (false); Trim | — | # Multi-provider FX Rates Fetcher (workflow description block) |
| Sticky Note6 | Sticky Note | Documentation | — | — | # Multi-provider FX Rates Fetcher (contains overall description, setup, requirements) |
| Sticky Note | Sticky Note | Inputs documentation | — | — | ## Inputs (trim/baseCurrency/currencies; manual or sub-workflow) |
| Sticky Note1 | Sticky Note | Static rates guidance | — | — | ## Optional static rates (static overrides provider) |
| Sticky Note2 | Sticky Note | Provider expansion guidance | — | — | ## Optional expand providers (clone nodes, connect to Stop and Error3) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** named: *Fetch reliable FX exchange rates with Frankfurter and open.er-api*.

2. **Add entry point A (manual test):**
   1. Add node **Manual Trigger** named **Manual Trigger (Example)**.
   2. Add node **Set** named **Set Currencies** with fields:
      - `trim` (Boolean) = `false`
      - `baseCurrency` (String) = `AED`
      - `currencies` (Array) = include your ISO codes (can include the base; it will be removed later)
   3. Connect **Manual Trigger (Example)** → **Set Currencies**.

3. **Add entry point B (called as sub-workflow):**
   1. Add node **Execute Workflow Trigger** named **Fetch FX Rates**.
   2. Define inputs:
      - `trim` (Boolean)
      - `baseCurrency` (String)
      - `currencies` (Array)
   3. (No incoming connection needed; it is a trigger.)

4. **Add validation node:**
   1. Add **Code** node named **Validate & Normalize Inputs**.
   2. Paste logic equivalent to:
      - uppercase `baseCurrency`
      - ensure `currencies` is an array
      - uppercase all currency codes
      - remove base currency from the currency list
      - output `{valid, reason, trim, baseCurrency, currencies}`
   3. Connect **Set Currencies** → **Validate & Normalize Inputs**.
   4. Connect **Fetch FX Rates** → **Validate & Normalize Inputs**.

5. **Gate on validity:**
   1. Add **If** node named **If Valid Data**.
   2. Condition: Boolean true check on `{{$json.valid}}`.
   3. Connect **Validate & Normalize Inputs** → **If Valid Data**.
   4. Add **Stop and Error** node named **Stop and Error** with message expression: `Error: {{ $json.reason }}`.
   5. Connect **If Valid Data (false)** → **Stop and Error**.

6. **Initialize FX state (and optional static rates):**
   1. Add **Code** node named **Initialize FX State + Static Rates** to create:
      - `fx.base`
      - `fx.asOf` = now
      - `fx.rates` = `{}` (optionally prefill static peg rates)
      - `fx.sources` = `{}` (metadata for static rates)
   2. Connect **If Valid Data (true)** → **Initialize FX State + Static Rates**.
   3. (Optional) Add static rates inside the code block; ensure values are positive finite numbers.

7. **Initialize coverage tracking:**
   1. Add **Code** node named **Initialize Coverage Tracking**.
   2. Make it read the requested base/symbols from **Validate & Normalize Inputs** (using node reference) and compute `missing`, `invalid`, `complete`.
   3. Connect **Initialize FX State + Static Rates** → **Initialize Coverage Tracking**.

8. **Provider 1 (Frankfurter):**
   1. Add **HTTP Request** node named **Frankfurter**:
      - URL: `https://api.frankfurter.dev/v1/latest`
      - Enable “Send Query Parameters”
      - Add query parameter `base` = `{{$json.fx.base}}`
      - Set **Retry on Fail** = enabled
      - Set **On Error** = *Continue (regular output)*
   2. Connect **Initialize Coverage Tracking** → **Frankfurter**.
   3. Add **If** node named **If base correct 1**:
      - Condition: `{{$json.base}}` equals `{{ $('Initialize Coverage Tracking').first().json.fx.base }}`
   4. Connect **Frankfurter** → **If base correct 1**.
   5. Add **Code** node **Normalize Frankfurter** to emit a normalized object:
      - `{ ok, provider, base, asOf, rates }` or `{ ok:false, error }`
   6. Add **Code** node **Handle Wrong Base 1** to emit `{ ok:false, provider:'frankfurter', error:'invalid_basecurrency' }`.
   7. Connect **If base correct 1 (true)** → **Normalize Frankfurter**.
   8. Connect **If base correct 1 (false)** → **Handle Wrong Base 1**.

9. **Merge & coverage check after provider 1:**
   1. Add **Code** node named **Merge Rates & Check Coverage (1)**:
      - Start from running FX in **Initialize Coverage Tracking**
      - Merge normalized provider rates if `ok` and base matches
      - Do not overwrite existing rates
      - Recompute `missing/invalid/complete`
   2. Connect **Normalize Frankfurter** → **Merge Rates & Check Coverage (1)**.
   3. Connect **Handle Wrong Base 1** → **Merge Rates & Check Coverage (1)**.
   4. Add **If** node **Coverage Complete 1?** with condition `{{$json.complete}}` is true.
   5. Connect **Merge Rates & Check Coverage (1)** → **Coverage Complete 1?**.

10. **Provider 2 (open.er-api.com fallback):**
    1. Add **HTTP Request** node named **open.er-api.com**:
       - URL: `https://open.er-api.com/v6/latest/{{$json.fx.base}}` (expression)
       - Retry on fail enabled
       - On Error continue regular output
    2. Connect **Coverage Complete 1? (false)** → **open.er-api.com**.
    3. Add **If** node **If base correct  2**:
       - Condition: `{{$json.base_code}}` equals `{{ $('Merge Rates & Check Coverage (1)').first().json.fx.base }}`
    4. Connect **open.er-api.com** → **If base correct  2**.
    5. Add **Code** node **Normalize open.er-api.com** (normalize success/error into `{ok, provider, base, asOf, rates}`).
    6. Add **Code** node **Handle Wrong Base 2** to emit `{ ok:false, provider:'open.er-api.com', error:'invalid_basecurrency' }`.
    7. Connect **If base correct  2 (true)** → **Normalize open.er-api.com**.
    8. Connect **If base correct  2 (false)** → **Handle Wrong Base 2**.

11. **Merge & coverage check after provider 2:**
    1. Add **Code** node **Merge Rates & Check Coverage (2)**:
       - Start from FX output of **Merge Rates & Check Coverage (1)**
       - Merge only missing symbols (by “do not overwrite” rule)
       - Recompute coverage
    2. Connect **Normalize open.er-api.com** → **Merge Rates & Check Coverage (2)**.
    3. Connect **Handle Wrong Base 2** → **Merge Rates & Check Coverage (2)**.
    4. Add **If** node **Coverage Complete 2?** with condition `{{$json.complete}}` is true.
    5. Connect **Merge Rates & Check Coverage (2)** → **Coverage Complete 2?**.
    6. Add **Stop and Error** node **Stop and Error3** with message: `Unable to fetch all currencies.`
    7. Connect **Coverage Complete 2? (false)** → **Stop and Error3**.

12. **Optional trim and final output:**
    1. Add **If** node **If Trim**:
       - Condition: `{{ $('Validate & Normalize Inputs').first().json.trim }}` is true
    2. Connect **Coverage Complete 1? (true)** → **If Trim**.
    3. Connect **Coverage Complete 2? (true)** → **If Trim**.
    4. Add **Code** node **Trim**:
       - Build `fx.rates` and `fx.sources` containing only the requested base + requested currencies
       - Force base rate to `1`
    5. Add **NoOp** node **Final Output**.
    6. Connect **If Trim (true)** → **Trim** → **Final Output**.
    7. Connect **If Trim (false)** → **Final Output**.

13. **Credentials**
    - None required (both APIs are public). Ensure the HTTP Request nodes do not use auth unless your environment enforces it.

14. **(Optional) Expanding providers**
    - Clone the provider 2 pattern (HTTP Request → base check → normalize → merge → coverage check).
    - Route the final “still incomplete” branch to **Stop and Error3** (or to another provider if adding more fallbacks).