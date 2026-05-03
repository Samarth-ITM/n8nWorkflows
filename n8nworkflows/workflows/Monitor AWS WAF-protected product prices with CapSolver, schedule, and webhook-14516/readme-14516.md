Monitor AWS WAF-protected product prices with CapSolver, schedule, and webhook

https://n8nworkflows.xyz/workflows/monitor-aws-waf-protected-product-prices-with-capsolver--schedule--and-webhook-14516


# Monitor AWS WAF-protected product prices with CapSolver, schedule, and webhook

# 1. Workflow Overview

This workflow monitors a product page protected by AWS WAF, extracts price and product name from the HTML, compares the current result with previously stored data, and then either builds an alert or records that nothing changed. It supports **two entry points**:

- a **scheduled execution** every 6 hours
- a **POST webhook execution** that returns the result directly to the caller

Its main use cases are:

- tracking price changes on an AWS WAF-protected product page
- running unattended periodic checks
- exposing the same scrape-and-compare logic through an API endpoint
- enabling simple price-drop/increase notifications or downstream integrations

The workflow is logically divided into the following blocks.

## 1.1 Scheduled Trigger and AWS WAF Solving
The scheduled path starts the workflow every 6 hours and obtains an AWS WAF challenge solution via CapSolver before fetching the protected page.

## 1.2 Scheduled Fetch, Extraction, and Comparison
After the WAF challenge is solved, the workflow fetches the product page HTML, extracts the relevant fields, and compares the current price to the last stored price in workflow static data.

## 1.3 Scheduled Change Handling
The workflow branches depending on whether the price changed. It either prepares an alert payload or a no-change payload.

## 1.4 Webhook Trigger and AWS WAF Solving
The webhook path accepts POST requests and launches the same AWS WAF solving process for on-demand checks.

## 1.5 Webhook Fetch, Extraction, and Comparison
The webhook branch fetches the product page, extracts the price and product name, compares them to the stored value, and determines whether the price changed.

## 1.6 Webhook Change Handling and Response
The webhook branch formats the output as either an alert or a no-change result and returns it as JSON through a Respond to Webhook node.

---

# 2. Block-by-Block Analysis

## 2.1 Scheduled Trigger and AWS WAF Solving

**Overview:**  
This block initiates automated monitoring on a fixed cadence and solves the AWS WAF protection challenge before any scraping occurs. It is the entry point for periodic checks.

**Nodes Involved:**  
- Every 6 Hours
- Solve AWS WAF

### Node: Every 6 Hours
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Time-based trigger node that starts the scheduled branch.
- **Configuration choices:**  
  Configured to run every 6 hours using an interval rule on the `hours` field.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: none
  - Output: `Solve AWS WAF`
- **Version-specific requirements:**  
  Uses `typeVersion: 1.3`. Available in recent n8n versions supporting the newer schedule trigger format.
- **Edge cases or potential failure types:**  
  - Workflow must be activated for the trigger to run automatically.
  - Time-based execution may drift slightly depending on server load and instance scheduling behavior.
  - If the n8n instance is down at runtime, the execution is missed unless external recovery logic exists.
- **Sub-workflow reference:**  
  None.

### Node: Solve AWS WAF
- **Type and technical role:** `n8n-nodes-capsolver.capSolver`  
  Uses CapSolver to solve an AWS WAF challenge and obtain solution data, including cookie material used in subsequent HTTP requests.
- **Configuration choices:**  
  - Operation: `AWS WAF`
  - Website URL: `https://YOUR-TARGET-SITE.com/product-page`
  - Uses a CapSolver account credential
- **Key expressions or variables used:**  
  No dynamic expression is used in the configured URL.
- **Input and output connections:**  
  - Input: `Every 6 Hours`
  - Output: `Fetch Product Page`
- **Version-specific requirements:**  
  Uses `typeVersion: 1`. Requires the CapSolver community/custom node to be installed and compatible with the current n8n version.
- **Edge cases or potential failure types:**  
  - Invalid or missing CapSolver API credentials
  - Unsupported or changed AWS WAF challenge format
  - CapSolver timeout or quota exhaustion
  - Incorrect target URL causing a mismatch between solved challenge and fetched page
  - Missing `data.solution.cookie` in the returned payload would break the next node’s header expression
- **Sub-workflow reference:**  
  None.

---

## 2.2 Scheduled Fetch, Extraction, and Comparison

**Overview:**  
This block uses the solved AWS WAF cookie to fetch the protected page, extracts product information from HTML, and compares the current result with persisted workflow state.

**Nodes Involved:**  
- Fetch Product Page
- Extract Data
- Compare Data

### Node: Fetch Product Page
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Performs the actual HTTP request to the protected product page using headers that emulate a browser and include the WAF cookie.
- **Configuration choices:**  
  - URL: `https://YOUR-TARGET-SITE.com/product-page`
  - Sends custom headers
  - User-Agent header is set to a recent Chrome browser string
  - Cookie header is populated from the CapSolver result: `{{$json.data.solution.cookie}}`
  - Response handling is left in standard response mode
- **Key expressions or variables used:**  
  - `={{ $json.data.solution.cookie }}`
- **Input and output connections:**  
  - Input: `Solve AWS WAF`
  - Output: `Extract Data`
- **Version-specific requirements:**  
  Uses `typeVersion: 4.3`, which assumes a modern HTTP Request node schema.
- **Edge cases or potential failure types:**  
  - 403/401/503 responses if the cookie is invalid or expired
  - Redirect loops or anti-bot pages still being returned
  - Header expression failure if CapSolver output shape differs
  - Site may require additional headers such as `accept-language`, `referer`, or session headers
  - Target page may return compressed or script-rendered content not directly useful to the HTML extractor
- **Sub-workflow reference:**  
  None.

### Node: Extract Data
- **Type and technical role:** `n8n-nodes-base.html`  
  Extracts values from the fetched HTML using CSS selectors.
- **Configuration choices:**  
  Operation is `extractHtmlContent` with two extraction keys:
  - `price` from `.product-price, [data-price], .price`
  - `productName` from `h1, .product-title`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: `Fetch Product Page`
  - Output: `Compare Data`
- **Version-specific requirements:**  
  Uses `typeVersion: 1.2`.
- **Edge cases or potential failure types:**  
  - Selectors may not match the real site HTML
  - Multiple matching elements may produce unexpected extraction results depending on page structure
  - If the fetched page is an interstitial, CAPTCHA page, or empty response, extraction may return blank values
  - Dynamic content rendered client-side may not be present in raw HTML
- **Sub-workflow reference:**  
  None.

### Node: Compare Data
- **Type and technical role:** `n8n-nodes-base.code`  
  JavaScript node that parses current and previous prices, stores the latest value in workflow static data, and determines whether the value changed.
- **Configuration choices:**  
  The code:
  - reads `price` and `productName` from the first input item
  - reads prior state from `$workflow.staticData`
  - parses numeric values from price strings using regex
  - updates `staticData.lastPrice` and `staticData.lastChecked`
  - computes:
    - `changed`
    - `direction` (`dropped`, `increased`, or `unchanged`)
    - `diff`
    - `checkedAt`
- **Key expressions or variables used:**  
  - `$workflow.staticData`
  - `$input.first().json.price`
  - `$input.first().json.productName`
- **Input and output connections:**  
  - Input: `Extract Data`
  - Output: `Data Changed?`
- **Version-specific requirements:**  
  Uses Code node `typeVersion: 2`.
- **Edge cases or potential failure types:**  
  - First execution has no previous price, so it always reports `previousPrice: first check` and `changed: false`
  - Regex price parsing is simplistic and may mishandle:
    - comma decimals such as `19,99`
    - thousands separators
    - currencies placed before the value
    - multi-price strings such as ranges or discounts
  - Static data is shared at workflow level, so both scheduled and webhook branches update the same stored value
  - If multiple executions run concurrently, race conditions may affect `lastPrice`
  - If `price` is empty, comparison degrades to null parsing and results in no detected change
- **Sub-workflow reference:**  
  None.

---

## 2.3 Scheduled Change Handling

**Overview:**  
This block checks the comparison result and formats either an alert payload for changed prices or a minimal status payload when no change is detected.

**Nodes Involved:**  
- Data Changed?
- Build Alert
- No Change

### Node: Data Changed?
- **Type and technical role:** `n8n-nodes-base.if`  
  Branches the flow based on the boolean `changed` property.
- **Configuration choices:**  
  Uses a strict boolean condition:
  - left value: `{{$json.changed}}`
  - operation: boolean `true`
- **Key expressions or variables used:**  
  - `={{ $json.changed }}`
- **Input and output connections:**  
  - Input: `Compare Data`
  - True output: `Build Alert`
  - False output: `No Change`
- **Version-specific requirements:**  
  Uses `typeVersion: 2.3`.
- **Edge cases or potential failure types:**  
  - If `changed` is missing or not a strict boolean, the condition may not behave as expected
  - Null comparison outcomes from the Code node effectively flow to the false branch
- **Sub-workflow reference:**  
  None.

### Node: Build Alert
- **Type and technical role:** `n8n-nodes-base.set`  
  Formats a human-readable alert record when a price change is detected.
- **Configuration choices:**  
  Sets:
  - `alert`: formatted message describing direction and price delta
  - `severity`: `deal` if dropped, otherwise `info`
  - `checkedAt`: copied from comparison output
- **Key expressions or variables used:**  
  - `=Price {{ $json.direction }} for {{ $json.productName }}: {{ $json.previousPrice }} → {{ $json.currentPrice }} ({{ $json.direction === 'dropped' ? '-' : '+' }}{{ $json.diff }})`
  - `={{ $json.direction === 'dropped' ? 'deal' : 'info' }}`
  - `={{ $json.checkedAt }}`
- **Input and output connections:**  
  - Input: `Data Changed?` true branch
  - Output: none connected
- **Version-specific requirements:**  
  Uses `typeVersion: 3.4`.
- **Edge cases or potential failure types:**  
  - If `diff` is null or malformed, the message can look odd
  - If previous/current price strings are empty, the alert text may still render but be misleading
  - No downstream delivery node exists in this branch, so the alert is only produced as terminal output of the execution
- **Sub-workflow reference:**  
  None.

### Node: No Change
- **Type and technical role:** `n8n-nodes-base.set`  
  Creates a simple status payload for unchanged checks.
- **Configuration choices:**  
  Sets:
  - `status` = `no_change`
  - `currentPrice`
  - `checkedAt`
- **Key expressions or variables used:**  
  - `={{ $json.currentPrice }}`
  - `={{ $json.checkedAt }}`
- **Input and output connections:**  
  - Input: `Data Changed?` false branch
  - Output: none connected
- **Version-specific requirements:**  
  Uses `typeVersion: 3.4`.
- **Edge cases or potential failure types:**  
  - If `currentPrice` is empty because extraction failed, the output still says `no_change`
  - This branch also terminates with no notification or persistence beyond execution data
- **Sub-workflow reference:**  
  None.

---

## 2.4 Webhook Trigger and AWS WAF Solving

**Overview:**  
This block exposes the monitoring logic through an HTTP endpoint. It receives a POST request, keeps the response open for a later response node, and solves AWS WAF before scraping.

**Nodes Involved:**  
- Receive Monitor Request
- Solve AWS WAF [Webhook]

### Node: Receive Monitor Request
- **Type and technical role:** `n8n-nodes-base.webhook`  
  Entry-point node that receives external POST requests.
- **Configuration choices:**  
  - Path: `price-monitor-aws-waf`
  - Method: `POST`
  - Response mode: `responseNode`
  - Error behavior: `continueRegularOutput`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: none
  - Output: `Solve AWS WAF [Webhook]`
- **Version-specific requirements:**  
  Uses `typeVersion: 2.1`.
- **Edge cases or potential failure types:**  
  - Workflow must be active for production webhook URL to work
  - Test vs production webhook URL differences can confuse setup
  - With `responseNode` mode, failure to reach `Respond to Webhook` may leave the caller waiting until timeout
  - `continueRegularOutput` may allow execution to continue in some error contexts, but it does not guarantee a meaningful response if downstream fails
- **Sub-workflow reference:**  
  None.

### Node: Solve AWS WAF [Webhook]
- **Type and technical role:** `n8n-nodes-capsolver.capSolver`  
  Same CapSolver AWS WAF solving step as the scheduled branch, but triggered by webhook.
- **Configuration choices:**  
  - Operation: `AWS WAF`
  - Website URL: `https://YOUR-TARGET-SITE.com/product-page`
  - Uses the same CapSolver credential
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: `Receive Monitor Request`
  - Output: `Fetch Product Page [Webhook]`
- **Version-specific requirements:**  
  Uses `typeVersion: 1`.
- **Edge cases or potential failure types:**  
  Same as the scheduled CapSolver node:
  - invalid credentials
  - unsupported challenge
  - timeout/rate limit
  - missing cookie in output
- **Sub-workflow reference:**  
  None.

---

## 2.5 Webhook Fetch, Extraction, and Comparison

**Overview:**  
This block fetches the protected page for on-demand requests, extracts the same target fields, and compares them against the stored workflow state.

**Nodes Involved:**  
- Fetch Product Page [Webhook]
- Extract Data [Webhook]
- Compare Data [Webhook]

### Node: Fetch Product Page [Webhook]
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Requests the target product page using the WAF cookie returned by CapSolver.
- **Configuration choices:**  
  Same as the scheduled HTTP request:
  - URL set to the product page
  - browser-style user-agent
  - cookie header from CapSolver output
- **Key expressions or variables used:**  
  - `={{ $json.data.solution.cookie }}`
- **Input and output connections:**  
  - Input: `Solve AWS WAF [Webhook]`
  - Output: `Extract Data [Webhook]`
- **Version-specific requirements:**  
  Uses `typeVersion: 4.3`.
- **Edge cases or potential failure types:**  
  Same as the scheduled branch:
  - invalid/expired cookie
  - anti-bot fallback page
  - header expression failure
  - script-rendered content not available in raw HTML
- **Sub-workflow reference:**  
  None.

### Node: Extract Data [Webhook]
- **Type and technical role:** `n8n-nodes-base.html`  
  Parses the returned HTML and extracts price and product name using CSS selectors.
- **Configuration choices:**  
  Same selectors as scheduled branch:
  - `price`: `.product-price, [data-price], .price`
  - `productName`: `h1, .product-title`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: `Fetch Product Page [Webhook]`
  - Output: `Compare Data [Webhook]`
- **Version-specific requirements:**  
  Uses `typeVersion: 1.2`.
- **Edge cases or potential failure types:**  
  Same selector fragility and empty-page risks as in the scheduled branch.
- **Sub-workflow reference:**  
  None.

### Node: Compare Data [Webhook]
- **Type and technical role:** `n8n-nodes-base.code`  
  Performs the same comparison logic as the scheduled code node and updates shared workflow static data.
- **Configuration choices:**  
  Same JavaScript logic as `Compare Data`, including:
  - regex parsing
  - static data update
  - change direction
  - diff calculation
- **Key expressions or variables used:**  
  - `$workflow.staticData`
  - `$input.first().json.price`
  - `$input.first().json.productName`
- **Input and output connections:**  
  - Input: `Extract Data [Webhook]`
  - Output: `Data Changed? [Webhook]`
- **Version-specific requirements:**  
  Uses `typeVersion: 2`.
- **Edge cases or potential failure types:**  
  - Shares the same workflow-level state as the scheduled branch
  - A webhook call can overwrite the stored baseline used by the schedule path, and vice versa
  - Same parsing limitations and null-handling issues apply
- **Sub-workflow reference:**  
  None.

---

## 2.6 Webhook Change Handling and Response

**Overview:**  
This block turns the comparison result into a webhook response payload. It always routes to a final response node so the caller receives JSON.

**Nodes Involved:**  
- Data Changed? [Webhook]
- Build Alert [Webhook]
- No Change [Webhook]
- Return Scraped Data

### Node: Data Changed? [Webhook]
- **Type and technical role:** `n8n-nodes-base.if`  
  Checks whether the current price differs from the previous stored price.
- **Configuration choices:**  
  Same strict boolean condition as the scheduled branch:
  - `{{$json.changed}}` must be true
- **Key expressions or variables used:**  
  - `={{ $json.changed }}`
- **Input and output connections:**  
  - Input: `Compare Data [Webhook]`
  - True output: `Build Alert [Webhook]`
  - False output: `No Change [Webhook]`
- **Version-specific requirements:**  
  Uses `typeVersion: 2.3`.
- **Edge cases or potential failure types:**  
  Same strict-type concerns as the scheduled IF node.
- **Sub-workflow reference:**  
  None.

### Node: Build Alert [Webhook]
- **Type and technical role:** `n8n-nodes-base.set`  
  Builds the JSON payload returned to the webhook caller when a change is detected.
- **Configuration choices:**  
  Sets:
  - `alert`
  - `severity`
  - `checkedAt`
  using the same expressions as the scheduled alert node
- **Key expressions or variables used:**  
  - `=Price {{ $json.direction }} for {{ $json.productName }}: {{ $json.previousPrice }} → {{ $json.currentPrice }} ({{ $json.direction === 'dropped' ? '-' : '+' }}{{ $json.diff }})`
  - `={{ $json.direction === 'dropped' ? 'deal' : 'info' }}`
  - `={{ $json.checkedAt }}`
- **Input and output connections:**  
  - Input: `Data Changed? [Webhook]` true branch
  - Output: `Return Scraped Data`
- **Version-specific requirements:**  
  Uses `typeVersion: 3.4`.
- **Edge cases or potential failure types:**  
  Same formatting limitations as the scheduled alert node.
- **Sub-workflow reference:**  
  None.

### Node: No Change [Webhook]
- **Type and technical role:** `n8n-nodes-base.set`  
  Builds the JSON payload returned to the webhook caller when no change is detected.
- **Configuration choices:**  
  Sets:
  - `status` = `no_change`
  - `currentPrice`
  - `checkedAt`
- **Key expressions or variables used:**  
  - `={{ $json.currentPrice }}`
  - `={{ $json.checkedAt }}`
- **Input and output connections:**  
  - Input: `Data Changed? [Webhook]` false branch
  - Output: `Return Scraped Data`
- **Version-specific requirements:**  
  Uses `typeVersion: 3.4`.
- **Edge cases or potential failure types:**  
  - If scraping failed silently and produced empty values, the caller can still receive `no_change`
- **Sub-workflow reference:**  
  None.

### Node: Return Scraped Data
- **Type and technical role:** `n8n-nodes-base.respondToWebhook`  
  Sends the final JSON response back to the HTTP caller.
- **Configuration choices:**  
  - Respond with: `json`
  - Response body expression: `={{ JSON.stringify($json) }}`
- **Key expressions or variables used:**  
  - `={{ JSON.stringify($json) }}`
- **Input and output connections:**  
  - Input: `Build Alert [Webhook]`, `No Change [Webhook]`
  - Output: none
- **Version-specific requirements:**  
  Uses `typeVersion: 1.5`.
- **Edge cases or potential failure types:**  
  - If upstream node output contains unexpected circular structures, `JSON.stringify` would fail, though that is unlikely here
  - If any branch does not reach this node, the webhook caller may receive no response or a timeout
  - Some n8n setups can return JSON directly without manual stringify, but this configuration is still workable
- **Sub-workflow reference:**  
  None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Every 6 Hours | Schedule Trigger | Starts scheduled monitoring every 6 hours |  | Solve AWS WAF | ## AWS WAF Scraping — Price & Product Details — CapSolver + Schedule + Webhook<br>### How it works<br>1. Triggers at a regular interval or via a webhook request.<br>2. Solves AWS WAF challenge then makes a request to fetch the product page.<br>3. Extracts product data from the retrieved HTML page.<br>4. Compares the current and previously stored data to detect any changes.<br>5. Sends an alert if data has changed; else logs no change.<br>6. Returns results if triggered via webhook.<br>### Setup steps<br>- [ ] Configure schedule settings in 'Every 6 Hours' node.<br>- [ ] Set up AWS WAF credentials in 'Solve AWS WAF' nodes.<br>- [ ] Input target URL in 'Fetch Product Page' nodes.<br>- [ ] Configure webhook URL in 'Receive Monitor Request' node.<br>### Customization<br>Adjust the target site URL in 'Fetch Product Page' and related nodes to match different sites or specific pages.<br>## Scheduled data scraping<br>Triggers scraping every 6 hours and solves AWS WAF. |
| Solve AWS WAF | CapSolver | Solves AWS WAF and returns cookie data | Every 6 Hours | Fetch Product Page | ## AWS WAF Scraping — Price & Product Details — CapSolver + Schedule + Webhook<br>### How it works<br>1. Triggers at a regular interval or via a webhook request.<br>2. Solves AWS WAF challenge then makes a request to fetch the product page.<br>3. Extracts product data from the retrieved HTML page.<br>4. Compares the current and previously stored data to detect any changes.<br>5. Sends an alert if data has changed; else logs no change.<br>6. Returns results if triggered via webhook.<br>### Setup steps<br>- [ ] Configure schedule settings in 'Every 6 Hours' node.<br>- [ ] Set up AWS WAF credentials in 'Solve AWS WAF' nodes.<br>- [ ] Input target URL in 'Fetch Product Page' nodes.<br>- [ ] Configure webhook URL in 'Receive Monitor Request' node.<br>### Customization<br>Adjust the target site URL in 'Fetch Product Page' and related nodes to match different sites or specific pages.<br>## Scheduled data scraping<br>Triggers scraping every 6 hours and solves AWS WAF. |
| Fetch Product Page | HTTP Request | Fetches the protected product page HTML | Solve AWS WAF | Extract Data | ## AWS WAF Scraping — Price & Product Details — CapSolver + Schedule + Webhook<br>### How it works<br>1. Triggers at a regular interval or via a webhook request.<br>2. Solves AWS WAF challenge then makes a request to fetch the product page.<br>3. Extracts product data from the retrieved HTML page.<br>4. Compares the current and previously stored data to detect any changes.<br>5. Sends an alert if data has changed; else logs no change.<br>6. Returns results if triggered via webhook.<br>### Setup steps<br>- [ ] Configure schedule settings in 'Every 6 Hours' node.<br>- [ ] Set up AWS WAF credentials in 'Solve AWS WAF' nodes.<br>- [ ] Input target URL in 'Fetch Product Page' nodes.<br>- [ ] Configure webhook URL in 'Receive Monitor Request' node.<br>### Customization<br>Adjust the target site URL in 'Fetch Product Page' and related nodes to match different sites or specific pages.<br>## Scheduled fetch and extract<br>Fetches and processes product page data on schedule. |
| Extract Data | HTML | Extracts price and product name from HTML | Fetch Product Page | Compare Data | ## AWS WAF Scraping — Price & Product Details — CapSolver + Schedule + Webhook<br>### How it works<br>1. Triggers at a regular interval or via a webhook request.<br>2. Solves AWS WAF challenge then makes a request to fetch the product page.<br>3. Extracts product data from the retrieved HTML page.<br>4. Compares the current and previously stored data to detect any changes.<br>5. Sends an alert if data has changed; else logs no change.<br>6. Returns results if triggered via webhook.<br>### Setup steps<br>- [ ] Configure schedule settings in 'Every 6 Hours' node.<br>- [ ] Set up AWS WAF credentials in 'Solve AWS WAF' nodes.<br>- [ ] Input target URL in 'Fetch Product Page' nodes.<br>- [ ] Configure webhook URL in 'Receive Monitor Request' node.<br>### Customization<br>Adjust the target site URL in 'Fetch Product Page' and related nodes to match different sites or specific pages.<br>## Scheduled fetch and extract<br>Fetches and processes product page data on schedule. |
| Compare Data | Code | Compares current price with stored previous price | Extract Data | Data Changed? | ## AWS WAF Scraping — Price & Product Details — CapSolver + Schedule + Webhook<br>### How it works<br>1. Triggers at a regular interval or via a webhook request.<br>2. Solves AWS WAF challenge then makes a request to fetch the product page.<br>3. Extracts product data from the retrieved HTML page.<br>4. Compares the current and previously stored data to detect any changes.<br>5. Sends an alert if data has changed; else logs no change.<br>6. Returns results if triggered via webhook.<br>### Setup steps<br>- [ ] Configure schedule settings in 'Every 6 Hours' node.<br>- [ ] Set up AWS WAF credentials in 'Solve AWS WAF' nodes.<br>- [ ] Input target URL in 'Fetch Product Page' nodes.<br>- [ ] Configure webhook URL in 'Receive Monitor Request' node.<br>### Customization<br>Adjust the target site URL in 'Fetch Product Page' and related nodes to match different sites or specific pages.<br>## Scheduled data comparison<br>Compares scraped data to previous entries and checks for changes. |
| Data Changed? | If | Branches based on whether the price changed | Compare Data | Build Alert; No Change | ## AWS WAF Scraping — Price & Product Details — CapSolver + Schedule + Webhook<br>### How it works<br>1. Triggers at a regular interval or via a webhook request.<br>2. Solves AWS WAF challenge then makes a request to fetch the product page.<br>3. Extracts product data from the retrieved HTML page.<br>4. Compares the current and previously stored data to detect any changes.<br>5. Sends an alert if data has changed; else logs no change.<br>6. Returns results if triggered via webhook.<br>### Setup steps<br>- [ ] Configure schedule settings in 'Every 6 Hours' node.<br>- [ ] Set up AWS WAF credentials in 'Solve AWS WAF' nodes.<br>- [ ] Input target URL in 'Fetch Product Page' nodes.<br>- [ ] Configure webhook URL in 'Receive Monitor Request' node.<br>### Customization<br>Adjust the target site URL in 'Fetch Product Page' and related nodes to match different sites or specific pages.<br>## Scheduled data comparison<br>Compares scraped data to previous entries and checks for changes. |
| Build Alert | Set | Formats alert payload for changed price | Data Changed? |  | ## AWS WAF Scraping — Price & Product Details — CapSolver + Schedule + Webhook<br>### How it works<br>1. Triggers at a regular interval or via a webhook request.<br>2. Solves AWS WAF challenge then makes a request to fetch the product page.<br>3. Extracts product data from the retrieved HTML page.<br>4. Compares the current and previously stored data to detect any changes.<br>5. Sends an alert if data has changed; else logs no change.<br>6. Returns results if triggered via webhook.<br>### Setup steps<br>- [ ] Configure schedule settings in 'Every 6 Hours' node.<br>- [ ] Set up AWS WAF credentials in 'Solve AWS WAF' nodes.<br>- [ ] Input target URL in 'Fetch Product Page' nodes.<br>- [ ] Configure webhook URL in 'Receive Monitor Request' node.<br>### Customization<br>Adjust the target site URL in 'Fetch Product Page' and related nodes to match different sites or specific pages.<br>## Scheduled alert or log<br>Generates alert or logs no change if applicable. |
| No Change | Set | Formats no-change payload | Data Changed? |  | ## AWS WAF Scraping — Price & Product Details — CapSolver + Schedule + Webhook<br>### How it works<br>1. Triggers at a regular interval or via a webhook request.<br>2. Solves AWS WAF challenge then makes a request to fetch the product page.<br>3. Extracts product data from the retrieved HTML page.<br>4. Compares the current and previously stored data to detect any changes.<br>5. Sends an alert if data has changed; else logs no change.<br>6. Returns results if triggered via webhook.<br>### Setup steps<br>- [ ] Configure schedule settings in 'Every 6 Hours' node.<br>- [ ] Set up AWS WAF credentials in 'Solve AWS WAF' nodes.<br>- [ ] Input target URL in 'Fetch Product Page' nodes.<br>- [ ] Configure webhook URL in 'Receive Monitor Request' node.<br>### Customization<br>Adjust the target site URL in 'Fetch Product Page' and related nodes to match different sites or specific pages.<br>## Scheduled alert or log<br>Generates alert or logs no change if applicable. |
| Receive Monitor Request | Webhook | Receives POST requests to trigger on-demand scraping |  | Solve AWS WAF [Webhook] | ## AWS WAF Scraping — Price & Product Details — CapSolver + Schedule + Webhook<br>### How it works<br>1. Triggers at a regular interval or via a webhook request.<br>2. Solves AWS WAF challenge then makes a request to fetch the product page.<br>3. Extracts product data from the retrieved HTML page.<br>4. Compares the current and previously stored data to detect any changes.<br>5. Sends an alert if data has changed; else logs no change.<br>6. Returns results if triggered via webhook.<br>### Setup steps<br>- [ ] Configure schedule settings in 'Every 6 Hours' node.<br>- [ ] Set up AWS WAF credentials in 'Solve AWS WAF' nodes.<br>- [ ] Input target URL in 'Fetch Product Page' nodes.<br>- [ ] Configure webhook URL in 'Receive Monitor Request' node.<br>### Customization<br>Adjust the target site URL in 'Fetch Product Page' and related nodes to match different sites or specific pages.<br>## Webhook triggered scraping<br>Handles requests from webhook to initiate scraping. |
| Solve AWS WAF [Webhook] | CapSolver | Solves AWS WAF for webhook-driven checks | Receive Monitor Request | Fetch Product Page [Webhook] | ## AWS WAF Scraping — Price & Product Details — CapSolver + Schedule + Webhook<br>### How it works<br>1. Triggers at a regular interval or via a webhook request.<br>2. Solves AWS WAF challenge then makes a request to fetch the product page.<br>3. Extracts product data from the retrieved HTML page.<br>4. Compares the current and previously stored data to detect any changes.<br>5. Sends an alert if data has changed; else logs no change.<br>6. Returns results if triggered via webhook.<br>### Setup steps<br>- [ ] Configure schedule settings in 'Every 6 Hours' node.<br>- [ ] Set up AWS WAF credentials in 'Solve AWS WAF' nodes.<br>- [ ] Input target URL in 'Fetch Product Page' nodes.<br>- [ ] Configure webhook URL in 'Receive Monitor Request' node.<br>### Customization<br>Adjust the target site URL in 'Fetch Product Page' and related nodes to match different sites or specific pages.<br>## Webhook triggered scraping<br>Handles requests from webhook to initiate scraping. |
| Fetch Product Page [Webhook] | HTTP Request | Fetches protected HTML for webhook checks | Solve AWS WAF [Webhook] | Extract Data [Webhook] | ## AWS WAF Scraping — Price & Product Details — CapSolver + Schedule + Webhook<br>### How it works<br>1. Triggers at a regular interval or via a webhook request.<br>2. Solves AWS WAF challenge then makes a request to fetch the product page.<br>3. Extracts product data from the retrieved HTML page.<br>4. Compares the current and previously stored data to detect any changes.<br>5. Sends an alert if data has changed; else logs no change.<br>6. Returns results if triggered via webhook.<br>### Setup steps<br>- [ ] Configure schedule settings in 'Every 6 Hours' node.<br>- [ ] Set up AWS WAF credentials in 'Solve AWS WAF' nodes.<br>- [ ] Input target URL in 'Fetch Product Page' nodes.<br>- [ ] Configure webhook URL in 'Receive Monitor Request' node.<br>### Customization<br>Adjust the target site URL in 'Fetch Product Page' and related nodes to match different sites or specific pages.<br>## Webhook fetch and extract<br>Fetches and processes product page data via webhook. |
| Extract Data [Webhook] | HTML | Extracts product fields for webhook checks | Fetch Product Page [Webhook] | Compare Data [Webhook] | ## AWS WAF Scraping — Price & Product Details — CapSolver + Schedule + Webhook<br>### How it works<br>1. Triggers at a regular interval or via a webhook request.<br>2. Solves AWS WAF challenge then makes a request to fetch the product page.<br>3. Extracts product data from the retrieved HTML page.<br>4. Compares the current and previously stored data to detect any changes.<br>5. Sends an alert if data has changed; else logs no change.<br>6. Returns results if triggered via webhook.<br>### Setup steps<br>- [ ] Configure schedule settings in 'Every 6 Hours' node.<br>- [ ] Set up AWS WAF credentials in 'Solve AWS WAF' nodes.<br>- [ ] Input target URL in 'Fetch Product Page' nodes.<br>- [ ] Configure webhook URL in 'Receive Monitor Request' node.<br>### Customization<br>Adjust the target site URL in 'Fetch Product Page' and related nodes to match different sites or specific pages.<br>## Webhook fetch and extract<br>Fetches and processes product page data via webhook. |
| Compare Data [Webhook] | Code | Compares current and previous price in webhook path | Extract Data [Webhook] | Data Changed? [Webhook] | ## AWS WAF Scraping — Price & Product Details — CapSolver + Schedule + Webhook<br>### How it works<br>1. Triggers at a regular interval or via a webhook request.<br>2. Solves AWS WAF challenge then makes a request to fetch the product page.<br>3. Extracts product data from the retrieved HTML page.<br>4. Compares the current and previously stored data to detect any changes.<br>5. Sends an alert if data has changed; else logs no change.<br>6. Returns results if triggered via webhook.<br>### Setup steps<br>- [ ] Configure schedule settings in 'Every 6 Hours' node.<br>- [ ] Set up AWS WAF credentials in 'Solve AWS WAF' nodes.<br>- [ ] Input target URL in 'Fetch Product Page' nodes.<br>- [ ] Configure webhook URL in 'Receive Monitor Request' node.<br>### Customization<br>Adjust the target site URL in 'Fetch Product Page' and related nodes to match different sites or specific pages.<br>## Webhook data comparison<br>Compares and evaluates changes in data from webhook requests. |
| Data Changed? [Webhook] | If | Branches webhook response based on detected change | Compare Data [Webhook] | Build Alert [Webhook]; No Change [Webhook] | ## AWS WAF Scraping — Price & Product Details — CapSolver + Schedule + Webhook<br>### How it works<br>1. Triggers at a regular interval or via a webhook request.<br>2. Solves AWS WAF challenge then makes a request to fetch the product page.<br>3. Extracts product data from the retrieved HTML page.<br>4. Compares the current and previously stored data to detect any changes.<br>5. Sends an alert if data has changed; else logs no change.<br>6. Returns results if triggered via webhook.<br>### Setup steps<br>- [ ] Configure schedule settings in 'Every 6 Hours' node.<br>- [ ] Set up AWS WAF credentials in 'Solve AWS WAF' nodes.<br>- [ ] Input target URL in 'Fetch Product Page' nodes.<br>- [ ] Configure webhook URL in 'Receive Monitor Request' node.<br>### Customization<br>Adjust the target site URL in 'Fetch Product Page' and related nodes to match different sites or specific pages.<br>## Webhook data comparison<br>Compares and evaluates changes in data from webhook requests. |
| Build Alert [Webhook] | Set | Formats webhook alert payload | Data Changed? [Webhook] | Return Scraped Data | ## AWS WAF Scraping — Price & Product Details — CapSolver + Schedule + Webhook<br>### How it works<br>1. Triggers at a regular interval or via a webhook request.<br>2. Solves AWS WAF challenge then makes a request to fetch the product page.<br>3. Extracts product data from the retrieved HTML page.<br>4. Compares the current and previously stored data to detect any changes.<br>5. Sends an alert if data has changed; else logs no change.<br>6. Returns results if triggered via webhook.<br>### Setup steps<br>- [ ] Configure schedule settings in 'Every 6 Hours' node.<br>- [ ] Set up AWS WAF credentials in 'Solve AWS WAF' nodes.<br>- [ ] Input target URL in 'Fetch Product Page' nodes.<br>- [ ] Configure webhook URL in 'Receive Monitor Request' node.<br>### Customization<br>Adjust the target site URL in 'Fetch Product Page' and related nodes to match different sites or specific pages.<br>## Webhook alert/log and return<br>Sends an alert or logs and returns data via webhook. |
| No Change [Webhook] | Set | Formats webhook no-change payload | Data Changed? [Webhook] | Return Scraped Data | ## AWS WAF Scraping — Price & Product Details — CapSolver + Schedule + Webhook<br>### How it works<br>1. Triggers at a regular interval or via a webhook request.<br>2. Solves AWS WAF challenge then makes a request to fetch the product page.<br>3. Extracts product data from the retrieved HTML page.<br>4. Compares the current and previously stored data to detect any changes.<br>5. Sends an alert if data has changed; else logs no change.<br>6. Returns results if triggered via webhook.<br>### Setup steps<br>- [ ] Configure schedule settings in 'Every 6 Hours' node.<br>- [ ] Set up AWS WAF credentials in 'Solve AWS WAF' nodes.<br>- [ ] Input target URL in 'Fetch Product Page' nodes.<br>- [ ] Configure webhook URL in 'Receive Monitor Request' node.<br>### Customization<br>Adjust the target site URL in 'Fetch Product Page' and related nodes to match different sites or specific pages.<br>## Webhook alert/log and return<br>Sends an alert or logs and returns data via webhook. |
| Return Scraped Data | Respond to Webhook | Returns JSON payload to webhook caller | Build Alert [Webhook]; No Change [Webhook] |  | ## AWS WAF Scraping — Price & Product Details — CapSolver + Schedule + Webhook<br>### How it works<br>1. Triggers at a regular interval or via a webhook request.<br>2. Solves AWS WAF challenge then makes a request to fetch the product page.<br>3. Extracts product data from the retrieved HTML page.<br>4. Compares the current and previously stored data to detect any changes.<br>5. Sends an alert if data has changed; else logs no change.<br>6. Returns results if triggered via webhook.<br>### Setup steps<br>- [ ] Configure schedule settings in 'Every 6 Hours' node.<br>- [ ] Set up AWS WAF credentials in 'Solve AWS WAF' nodes.<br>- [ ] Input target URL in 'Fetch Product Page' nodes.<br>- [ ] Configure webhook URL in 'Receive Monitor Request' node.<br>### Customization<br>Adjust the target site URL in 'Fetch Product Page' and related nodes to match different sites or specific pages.<br>## Webhook alert/log and return<br>Sends an alert or logs and returns data via webhook. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.  
   Name it something like:  
   `AWS WAF Scraping — Price & Product Details — CapSolver + Schedule + Webhook`

2. **Add a Schedule Trigger node** named `Every 6 Hours`.
   - Node type: **Schedule Trigger**
   - Configure interval mode:
     - Field: `hours`
     - Interval: `6`
   - This node will serve as the scheduled entry point.

3. **Add a CapSolver node** named `Solve AWS WAF`.
   - Node type: **CapSolver**
   - Operation: `AWS WAF`
   - Website URL: your real protected product URL, for example:  
     `https://example.com/product-page`
   - Create or select a **CapSolver API credential**
   - Connect:
     - `Every 6 Hours` → `Solve AWS WAF`

4. **Add an HTTP Request node** named `Fetch Product Page`.
   - Node type: **HTTP Request**
   - Method: `GET` if not otherwise required
   - URL: the same target product page URL used in the CapSolver node
   - Enable **Send Headers**
   - Add headers:
     - `user-agent` = a realistic browser user agent string
     - `Cookie` = `{{ $json.data.solution.cookie }}`
   - Keep response format suitable for passing HTML to the next node
   - Connect:
     - `Solve AWS WAF` → `Fetch Product Page`

5. **Add an HTML node** named `Extract Data`.
   - Node type: **HTML**
   - Operation: `Extract HTML Content`
   - Add extraction values:
     - Key: `price`
       - CSS selector: `.product-price, [data-price], .price`
     - Key: `productName`
       - CSS selector: `h1, .product-title`
   - Adjust selectors to match the real site HTML
   - Connect:
     - `Fetch Product Page` → `Extract Data`

6. **Add a Code node** named `Compare Data`.
   - Node type: **Code**
   - Language: JavaScript
   - Paste logic equivalent to:
     - read `price` and `productName`
     - read previous price from `$workflow.staticData`
     - parse numeric price
     - update `staticData.lastPrice` and `staticData.lastChecked`
     - compute `changed`, `direction`, `diff`, `checkedAt`
   - Use this exact logic if you want the same behavior:

   ```javascript
   const staticData = $workflow.staticData;
   const currentPrice = $input.first().json.price;
   const previousPrice = staticData.lastPrice;
   const productName = $input.first().json.productName || 'Product';

   const parsePrice = (str) => {
     if (!str) return null;
     const match = str.match(/[\d]+\.?\d*/);
     return match ? parseFloat(match[0].replace(',', '')) : null;
   };

   const currentNum = parsePrice(currentPrice);
   const previousNum = parsePrice(previousPrice);

   staticData.lastPrice = currentPrice;
   staticData.lastChecked = new Date().toISOString();

   const changed = previousNum !== null && currentNum !== null && currentNum !== previousNum;
   const direction = changed ? (currentNum < previousNum ? 'dropped' : 'increased') : 'unchanged';
   const diff = changed ? Math.abs(currentNum - previousNum).toFixed(2) : '0';

   return [{
     json: {
       productName,
       currentPrice,
       previousPrice: previousPrice || 'first check',
       changed,
       direction,
       diff: changed ? `$${diff}` : null,
       checkedAt: new Date().toISOString()
     }
   }];
   ```
   - Connect:
     - `Extract Data` → `Compare Data`

7. **Add an IF node** named `Data Changed?`.
   - Node type: **IF**
   - Create one boolean condition:
     - Value 1: `{{ $json.changed }}`
     - Operation: `is true`
   - Keep strict type behavior if available in your node version
   - Connect:
     - `Compare Data` → `Data Changed?`

8. **Add a Set node** named `Build Alert`.
   - Node type: **Set**
   - Add fields:
     - `alert` = `Price {{ $json.direction }} for {{ $json.productName }}: {{ $json.previousPrice }} → {{ $json.currentPrice }} ({{ $json.direction === 'dropped' ? '-' : '+' }}{{ $json.diff }})`
     - `severity` = `{{ $json.direction === 'dropped' ? 'deal' : 'info' }}`
     - `checkedAt` = `{{ $json.checkedAt }}`
   - Connect:
     - `Data Changed?` true output → `Build Alert`

9. **Add another Set node** named `No Change`.
   - Node type: **Set**
   - Add fields:
     - `status` = `no_change`
     - `currentPrice` = `{{ $json.currentPrice }}`
     - `checkedAt` = `{{ $json.checkedAt }}`
   - Connect:
     - `Data Changed?` false output → `No Change`

10. **Add a Webhook node** named `Receive Monitor Request`.
    - Node type: **Webhook**
    - HTTP method: `POST`
    - Path: `price-monitor-aws-waf`
    - Response mode: `Using Respond to Webhook node`
    - If available, set error handling to continue regular output as in the source workflow
    - This creates the second entry point.

11. **Add another CapSolver node** named `Solve AWS WAF [Webhook]`.
    - Same configuration as the scheduled CapSolver node:
      - Operation: `AWS WAF`
      - Website URL: same protected page URL
      - Same CapSolver credential
    - Connect:
      - `Receive Monitor Request` → `Solve AWS WAF [Webhook]`

12. **Add another HTTP Request node** named `Fetch Product Page [Webhook]`.
    - Same configuration as the scheduled HTTP node:
      - URL: target page
      - Headers:
        - `user-agent` = browser string
        - `Cookie` = `{{ $json.data.solution.cookie }}`
    - Connect:
      - `Solve AWS WAF [Webhook]` → `Fetch Product Page [Webhook]`

13. **Add another HTML node** named `Extract Data [Webhook]`.
    - Same configuration as `Extract Data`
    - Connect:
      - `Fetch Product Page [Webhook]` → `Extract Data [Webhook]`

14. **Add another Code node** named `Compare Data [Webhook]`.
    - Use the same JavaScript logic as `Compare Data`
    - Connect:
      - `Extract Data [Webhook]` → `Compare Data [Webhook]`

15. **Add another IF node** named `Data Changed? [Webhook]`.
    - Same condition as the scheduled IF node:
      - `{{ $json.changed }}` is true
    - Connect:
      - `Compare Data [Webhook]` → `Data Changed? [Webhook]`

16. **Add a Set node** named `Build Alert [Webhook]`.
    - Same fields as `Build Alert`
    - Connect:
      - `Data Changed? [Webhook]` true output → `Build Alert [Webhook]`

17. **Add a Set node** named `No Change [Webhook]`.
    - Same fields as `No Change`
    - Connect:
      - `Data Changed? [Webhook]` false output → `No Change [Webhook]`

18. **Add a Respond to Webhook node** named `Return Scraped Data`.
    - Node type: **Respond to Webhook**
    - Respond with: `JSON`
    - Response body: `{{ JSON.stringify($json) }}`
    - Connect:
      - `Build Alert [Webhook]` → `Return Scraped Data`
      - `No Change [Webhook]` → `Return Scraped Data`

19. **Review credentials and URLs.**
    - Ensure both CapSolver nodes use a valid API credential.
    - Ensure both HTTP Request nodes point to the exact same product page as the CapSolver nodes.
    - If the site is locale-sensitive, consider adding extra headers such as:
      - `accept-language`
      - `referer`
      - `accept`

20. **Test the scheduled path manually.**
    - Run from `Every 6 Hours` or from `Solve AWS WAF`
    - Confirm that:
      - CapSolver returns a cookie
      - the page fetch succeeds
      - HTML selectors return `price` and `productName`
      - the first run stores state and returns no change

21. **Test the webhook path.**
    - Use the test webhook URL first
    - Send a `POST` request with any required body, even though this workflow currently does not use incoming payload fields
    - Confirm the response returns either:
      - an alert JSON object, or
      - a no-change JSON object

22. **Activate the workflow** if you want:
    - the schedule to run automatically
    - the production webhook URL to be available

23. **Optional but recommended improvements when rebuilding:**
    - Add explicit error handling after CapSolver and HTTP Request nodes
    - Add validation that extracted `price` is not empty
    - Add a notification node after `Build Alert`, such as Slack, email, Discord, or another webhook
    - Separate static data keys for scheduled and webhook branches if you do not want them sharing the same baseline
    - Improve price parsing for international currency formats

### Credential configuration required
- **CapSolver account credential**
  - API key from CapSolver
  - Must be assigned to both CapSolver nodes

### Sub-workflow setup
- No sub-workflows are used in this workflow.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow has two explicit entry points: one schedule-based and one webhook-based. | Architecture note |
| Both branches use the same workflow static data key (`lastPrice`), so webhook runs and scheduled runs affect the same historical baseline. | Important state-management note |
| The first successful execution does not trigger a change alert because there is no previous price to compare against. | Comparison behavior |
| The current scheduled branch stops after creating `Build Alert` or `No Change`; it does not send notifications externally. | Operational limitation |
| The selectors `.product-price, [data-price], .price` and `h1, .product-title` are generic placeholders and will often need adaptation to the target site. | Extraction note |
| The price parser is simplistic and may fail on international formats such as `1.299,99 €` or on pages showing multiple prices. | Data parsing note |