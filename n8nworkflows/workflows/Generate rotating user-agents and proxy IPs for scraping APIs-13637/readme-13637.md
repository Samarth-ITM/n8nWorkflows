Generate rotating user-agents and proxy IPs for scraping APIs

https://n8nworkflows.xyz/workflows/generate-rotating-user-agents-and-proxy-ips-for-scraping-apis-13637


# Generate rotating user-agents and proxy IPs for scraping APIs

# 1. Workflow Overview

This workflow generates a rotating combination of:

- random user-agent strings fetched from a public website
- proxy connection settings provided manually

It then uses those values to call a target API through a proxy, with an optional verification branch that checks which IP address and user-agent are actually seen externally.

Typical use case: sending multiple HTTP requests to public APIs or scraping-oriented endpoints while varying both the `user-agent` header and the outbound IP address, assuming the proxy provider supports rotating IPs.

## 1.1 Entry and Configuration

The workflow starts manually and immediately splits into two branches:

- one branch fetches and prepares a pool of user-agent strings
- the other provides proxy credentials and port values

These two branches are later merged so that each selected user-agent can be paired with the same proxy settings.

## 1.2 User-Agent Collection and Preparation

This block downloads a browser list page, extracts user-agent strings from HTML, splits them into individual items, cleans formatting issues, randomizes order, and keeps only a configurable number of entries.

This produces a small rotating set of candidate user-agents.

## 1.3 Combination of Proxy Settings with Selected User-Agents

The workflow combines the selected user-agent items with the proxy credential item so each outgoing request item contains both:

- `clean_user-agent`
- `proxy_username`
- `proxy_password`
- `proxy_port`

## 1.4 External Request Execution

From the merged data, the workflow fans out into two parallel HTTP actions:

- a validation request to Cloudflare’s trace endpoint to inspect the visible IP and user-agent
- the actual target API request using the same proxy and `user-agent` header

## 1.5 Observability / Validation

The Cloudflare response is parsed to expose:

- the detected public IP address
- the detected user-agent

This branch is informational and helps verify that proxy rotation and header injection are working as expected.

---

# 2. Block-by-Block Analysis

## 2.1 Entry and Manual Configuration

### Overview
This block provides the workflow entry point and manually defines the proxy credentials used by later HTTP requests. It is the only place where the user must supply proxy-specific values directly in the workflow.

### Nodes Involved
- `Manual trigger`
- `SET your proxy connection details here`

### Node Details

#### Manual trigger
- **Type and technical role:** `n8n-nodes-base.manualTrigger`; starts the workflow manually from the editor.
- **Configuration choices:** No custom parameters; default manual execution trigger.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - No input
  - Outputs to:
    - `user-agents`
    - `SET your proxy connection details here`
- **Version-specific requirements:** Type version `1`; standard manual trigger behavior.
- **Edge cases or potential failure types:** None at runtime beyond normal manual execution constraints.
- **Sub-workflow reference:** None.

#### SET your proxy connection details here
- **Type and technical role:** `n8n-nodes-base.set`; creates proxy configuration fields as static values.
- **Configuration choices:** Defines three string fields:
  - `proxy_username`
  - `proxy_password`
  - `proxy_port`
- **Key expressions or variables used:** None; static placeholders (`xxxxx`) must be replaced.
- **Input and output connections:**  
  - Input from `Manual trigger`
  - Output to `Merge`
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - Leaving placeholder values unchanged will cause proxy authentication or connection failure.
  - Incorrect port format may produce proxy connection errors.
  - Hardcoding secrets in the workflow is operationally risky; credentials are visible to anyone with workflow access.
- **Sub-workflow reference:** None.

---

## 2.2 User-Agent Collection and Preparation

### Overview
This block fetches a public HTML page containing browser user-agent strings, extracts them from the DOM, converts them into one item per user-agent, normalizes whitespace, shuffles the list, and limits the number of selected items.

This is the core generation mechanism for varying the `user-agent` header.

### Nodes Involved
- `user-agents`
- `Extract user-agent values`
- `Split Out`
- `clean the return to line in user-agent`
- `random sort`
- `Take X random user-agents`

### Node Details

#### user-agents
- **Type and technical role:** `n8n-nodes-base.httpRequest`; downloads the source page containing browser user-agent strings.
- **Configuration choices:**
  - Sends an HTTP request to `https://www.useragentstring.com/pages/Browserlist/`
  - Uses default request options
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input from `Manual trigger`
  - Output to `Extract user-agent values`
- **Version-specific requirements:** Type version `4.3`.
- **Edge cases or potential failure types:**
  - Remote site may be down or rate-limited.
  - HTML structure may change, breaking extraction downstream.
  - SSL/network/DNS failures may occur.
- **Sub-workflow reference:** None.

#### Extract user-agent values
- **Type and technical role:** `n8n-nodes-base.html`; parses HTML and extracts specific text content from matching elements.
- **Configuration choices:**
  - Operation: extract HTML content
  - CSS selector: `ul li a`
  - Output key: `userAgents`
  - `returnArray: true`, producing an array of extracted strings
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input from `user-agents`
  - Output to `Split Out`
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or potential failure types:**
  - If the upstream response is not HTML, extraction may fail or produce no matches.
  - If page markup changes, `userAgents` may be empty.
  - Overly broad selector may pull unexpected values if the source page changes.
- **Sub-workflow reference:** None.

#### Split Out
- **Type and technical role:** `n8n-nodes-base.splitOut`; turns an array field into multiple individual items.
- **Configuration choices:**
  - Field to split out: `userAgents`
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input from `Extract user-agent values`
  - Output to `clean the return to line in user-agent`
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - If `userAgents` is missing or empty, no useful output is produced.
  - Unexpected non-array data may cause failure or empty behavior depending on n8n runtime handling.
- **Sub-workflow reference:** None.

#### clean the return to line in user-agent
- **Type and technical role:** `n8n-nodes-base.set`; normalizes extracted user-agent strings by removing newlines and collapsing repeated whitespace.
- **Configuration choices:**
  - Creates `clean_user-agent`
  - Uses an expression to sanitize the current item’s `userAgents` value
- **Key expressions or variables used:**
  - `{{ $json.userAgents.replace(/\n/g, ' ').replace(/\s+/g, ' ').trim() }}`
- **Input and output connections:**  
  - Input from `Split Out`
  - Output to `random sort`
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - If `userAgents` is not a string, `.replace(...)` will fail.
  - Empty strings may still pass through and later be used as invalid headers.
- **Sub-workflow reference:** None.

#### random sort
- **Type and technical role:** `n8n-nodes-base.sort`; randomizes item order.
- **Configuration choices:**
  - Sort type: `random`
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input from `clean the return to line in user-agent`
  - Output to `Take X random user-agents`
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - Randomization is non-deterministic; repeated runs will produce different subsets.
  - If there are very few source user-agents, apparent rotation may be weak.
- **Sub-workflow reference:** None.

#### Take X random user-agents
- **Type and technical role:** `n8n-nodes-base.limit`; keeps only the first N items after randomization.
- **Configuration choices:**
  - `maxItems: 5`
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input from `random sort`
  - Output to `Merge`
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - If fewer than 5 items are available, all available items pass through.
  - Changing this value directly changes request fan-out volume downstream.
- **Sub-workflow reference:** None.

---

## 2.3 Combination of Proxy Settings with Selected User-Agents

### Overview
This block merges the single proxy settings item with the list of selected user-agent items. The result is a set of items where each one contains both proxy credentials and one cleaned user-agent string.

### Nodes Involved
- `Merge`

### Node Details

#### Merge
- **Type and technical role:** `n8n-nodes-base.merge`; combines data from two branches.
- **Configuration choices:**
  - Mode: `combine`
  - Combine by: `combineAll`
- **Key expressions or variables used:** None inside the node configuration.
- **Input and output connections:**  
  - Input 0 from `SET your proxy connection details here`
  - Input 1 from `Take X random user-agents`
  - Outputs to:
    - `Check used IP/user-agent with cloudflare`
    - `Targeted API`
- **Version-specific requirements:** Type version `3.2`.
- **Edge cases or potential failure types:**
  - If either branch produces no items, downstream execution may not happen as expected.
  - `combineAll` behavior depends on item counts; here it effectively propagates the proxy fields across the selected user-agent items, but this should be validated after changes.
  - If the proxy branch is modified to emit multiple items, output cardinality may change significantly.
- **Sub-workflow reference:** None.

---

## 2.4 External Request Execution

### Overview
This block performs two outbound HTTP operations using the merged item data. Both rely on dynamically injected proxy settings and the cleaned user-agent header.

### Nodes Involved
- `Check used IP/user-agent with cloudflare`
- `Targeted API`

### Node Details

#### Check used IP/user-agent with cloudflare
- **Type and technical role:** `n8n-nodes-base.httpRequest`; sends a verification request through the proxy to confirm the externally visible IP and user-agent.
- **Configuration choices:**
  - URL: `https://cloudflare.com/cdn-cgi/trace`
  - Method: `POST`
  - Proxy option:
    - `http://{{ $json.proxy_username }}:{{ $json.proxy_password }}@gate.decodo.com:{{ $json.proxy_port }}`
  - Full response enabled
  - Sends header:
    - `user-agent: {{ $json["clean_user-agent"] }}`
- **Key expressions or variables used:**
  - Proxy:
    - `{{ $json.proxy_username }}`
    - `{{ $json.proxy_password }}`
    - `{{ $json.proxy_port }}`
  - Header:
    - `{{ $json["clean_user-agent"] }}`
- **Input and output connections:**  
  - Input from `Merge`
  - Output to `IP address and user-agent used`
- **Version-specific requirements:** Type version `4.4`.
- **Edge cases or potential failure types:**
  - Proxy auth failures
  - Proxy server timeout or connection refusal
  - Cloudflare may reject unusual traffic patterns or POST usage in some contexts
  - If proxy provider rotates differently than expected, returned IP may repeat
  - Full response shape may differ from assumptions if n8n version or node options change
- **Sub-workflow reference:** None.

#### Targeted API
- **Type and technical role:** `n8n-nodes-base.httpRequest`; placeholder node representing the actual API to call using rotating user-agent and proxy settings.
- **Configuration choices:**
  - URL: `API_URL` placeholder
  - Proxy option:
    - `http://{{ $json.proxy_username }}:{{ $json.proxy_password }}@gate.decodo.com:{{ $json.proxy_port }}`
  - Sends header:
    - `user-agent: {{ $json["clean_user-agent"] }}`
- **Key expressions or variables used:**
  - Proxy:
    - `{{ $json.proxy_username }}`
    - `{{ $json.proxy_password }}`
    - `{{ $json.proxy_port }}`
  - Header:
    - `{{ $json["clean_user-agent"] }}`
- **Input and output connections:**  
  - Input from `Merge`
  - No downstream node connected
- **Version-specific requirements:** Type version `4.4`.
- **Edge cases or potential failure types:**
  - `API_URL` must be replaced or the node will fail.
  - Target API may block proxy traffic entirely.
  - Some APIs require additional headers, authentication, query parameters, cookies, or body content.
  - Some endpoints validate browser headers beyond `user-agent`; adding only one header may not be sufficient.
  - Rate limiting may still apply despite IP/header rotation.
- **Sub-workflow reference:** None.

---

## 2.5 Observability / Validation

### Overview
This block parses the Cloudflare trace response and extracts a human-readable IP address and user-agent value. It is purely diagnostic and does not affect the target API execution.

### Nodes Involved
- `IP address and user-agent used`

### Node Details

#### IP address and user-agent used
- **Type and technical role:** `n8n-nodes-base.set`; extracts values from the textual response body using regular expressions.
- **Configuration choices:**
  - Creates:
    - `Ip_address`
    - `user-agent`
- **Key expressions or variables used:**
  - `{{ $json.data.match(/ip=([^\n]+)/)[1] }}`
  - `{{ $json.data.match(/uag=([^\n]+)/)[1] }}`
- **Input and output connections:**  
  - Input from `Check used IP/user-agent with cloudflare`
  - No downstream node connected
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - Assumes the response text is present in `$json.data`.
  - If Cloudflare changes field names or format, regex matching will fail.
  - If `.match(...)` returns `null`, indexing `[1]` throws an expression error.
  - Because the request uses full response mode, any change in node output structure across versions should be validated.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Manual trigger | Manual Trigger | Starts workflow manually |  | user-agents; SET your proxy connection details here | # Generate multiple dynamic user-agent & IP address for scraping APIs using proxy  \nUseful for scraping only/API data.  \nThis workflow will give you the ability to bypass the IP address limitation control of **some** publicly available APIs by using a different couple of “user-agent” and “IP address” for each call to the targeted API.  \nYou can therefore place this workflow before any HTTP request node.  \nIf you're using Decodo proxy server (or any other proxy service provider) you can use the Residential proxy with "session type" as "rotating" if you want to have `one IP address per API call`.  \nCredentials usually work like this in any proxy service provider: `http://username:password@gate.decodo.com:PORT` |
| user-agents | HTTP Request | Downloads public page containing user-agent strings | Manual trigger | Extract user-agent values | # Generate multiple dynamic user-agent & IP address for scraping APIs using proxy  \nUseful for scraping only/API data. |
| Extract user-agent values | HTML | Extracts user-agent strings from page HTML | user-agents | Split Out | # Generate multiple dynamic user-agent & IP address for scraping APIs using proxy |
| Split Out | Split Out | Converts array of extracted user-agents into one item per value | Extract user-agent values | clean the return to line in user-agent | # Generate multiple dynamic user-agent & IP address for scraping APIs using proxy |
| clean the return to line in user-agent | Set | Normalizes whitespace/newlines in each user-agent string | Split Out | random sort | # Generate multiple dynamic user-agent & IP address for scraping APIs using proxy |
| random sort | Sort | Randomizes user-agent order | clean the return to line in user-agent | Take X random user-agents | # Generate multiple dynamic user-agent & IP address for scraping APIs using proxy |
| Take X random user-agents | Limit | Keeps only a configurable number of random user-agents | random sort | Merge | ## Update the number of user-agent you want to use |
| SET your proxy connection details here | Set | Stores proxy credentials and port | Manual trigger | Merge | ## Add proxy credentials here |
| Merge | Merge | Combines selected user-agents with proxy settings | SET your proxy connection details here; Take X random user-agents | Check used IP/user-agent with cloudflare; Targeted API | # Generate multiple dynamic user-agent & IP address for scraping APIs using proxy  \n### 1) The proxy connection details  \nConfigure these fields in `SET your proxy connection details here`: `proxy_username`, `proxy_password`, `proxy_port`  \n### 2) Number of user-agents needed  \nConfigure the count in `Take X random user-agents` |
| Check used IP/user-agent with cloudflare | HTTP Request | Verifies actual outbound IP and user-agent via proxy | Merge | IP address and user-agent used | ## Informative node to show with couple IP_address/user-agent are being used |
| IP address and user-agent used | Set | Parses Cloudflare trace response into readable fields | Check used IP/user-agent with cloudflare |  | ## Informative node to show with couple IP_address/user-agent are being used |
| Targeted API | HTTP Request | Calls the actual destination API using proxy and random user-agent | Merge |  | ## Add the HTTP node for the API you want to call |
| Sticky Note | Sticky Note | Documentation note |  |  |  |
| Sticky Note1 | Sticky Note | Documentation note |  |  |  |
| Sticky Note2 | Sticky Note | Documentation note |  |  |  |
| Sticky Note3 | Sticky Note | Documentation note |  |  |  |
| Sticky Note4 | Sticky Note | Documentation note |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Generate rotating user-agents and proxy IPs for scraping APIs`.
   - Keep execution mode at the default unless your environment requires something else.

2. **Add a `Manual Trigger` node**
   - Node type: `Manual Trigger`
   - Leave default settings.
   - This will be the workflow entry point.

3. **Add a `Set` node for proxy settings**
   - Name it: `SET your proxy connection details here`
   - Connect `Manual trigger` → `SET your proxy connection details here`
   - Add three string fields:
     - `proxy_username`
     - `proxy_password`
     - `proxy_port`
   - Fill them with your actual proxy provider values.
   - For Decodo-style formatting, the later proxy URL will be built as:
     - `http://username:password@gate.decodo.com:PORT`

4. **Add an `HTTP Request` node to fetch user-agents**
   - Name it: `user-agents`
   - Connect `Manual trigger` → `user-agents`
   - Configure:
     - Method: default `GET`
     - URL: `https://www.useragentstring.com/pages/Browserlist/`
   - Keep other options at default.

5. **Add an `HTML` node to extract the user-agent strings**
   - Name it: `Extract user-agent values`
   - Connect `user-agents` → `Extract user-agent values`
   - Configure:
     - Operation: `Extract HTML Content`
     - Add one extraction value:
       - Key: `userAgents`
       - CSS Selector: `ul li a`
       - Return Array: enabled
   - This should return an array field containing multiple user-agent strings.

6. **Add a `Split Out` node**
   - Name it: `Split Out`
   - Connect `Extract user-agent values` → `Split Out`
   - Configure:
     - Field to Split Out: `userAgents`
   - This converts the array into one item per user-agent.

7. **Add a `Set` node to clean formatting**
   - Name it: `clean the return to line in user-agent`
   - Connect `Split Out` → `clean the return to line in user-agent`
   - Add one string field:
     - Name: `clean_user-agent`
     - Value as expression:
       - `{{ $json.userAgents.replace(/\n/g, ' ').replace(/\s+/g, ' ').trim() }}`
   - This removes line breaks and compresses repeated whitespace.

8. **Add a `Sort` node for randomization**
   - Name it: `random sort`
   - Connect `clean the return to line in user-agent` → `random sort`
   - Configure:
     - Sort type: `Random`

9. **Add a `Limit` node**
   - Name it: `Take X random user-agents`
   - Connect `random sort` → `Take X random user-agents`
   - Configure:
     - Maximum items: `5`
   - Change this value if you want more or fewer rotated requests per run.

10. **Add a `Merge` node**
    - Name it: `Merge`
    - Connect:
      - `SET your proxy connection details here` → `Merge` input 1
      - `Take X random user-agents` → `Merge` input 2
    - Configure:
      - Mode: `Combine`
      - Combine By: `Combine All`
    - Verify after a test run that each output item contains:
      - `clean_user-agent`
      - `proxy_username`
      - `proxy_password`
      - `proxy_port`

11. **Add a verification `HTTP Request` node**
    - Name it: `Check used IP/user-agent with cloudflare`
    - Connect `Merge` → `Check used IP/user-agent with cloudflare`
    - Configure:
      - URL: `https://cloudflare.com/cdn-cgi/trace`
      - Method: `POST`
      - Send Headers: enabled
      - Add header:
        - Name: `user-agent`
        - Value: `{{ $json["clean_user-agent"] }}`
      - In Options, set Proxy to:
        - `http://{{ $json.proxy_username }}:{{ $json.proxy_password }}@gate.decodo.com:{{ $json.proxy_port }}`
      - Enable full response in the response options so the output structure includes the response payload used by the next parsing node.

12. **Add a `Set` node to parse Cloudflare trace output**
    - Name it: `IP address and user-agent used`
    - Connect `Check used IP/user-agent with cloudflare` → `IP address and user-agent used`
    - Add two string fields:
      - `Ip_address` with expression:
        - `{{ $json.data.match(/ip=([^\n]+)/)[1] }}`
      - `user-agent` with expression:
        - `{{ $json.data.match(/uag=([^\n]+)/)[1] }}`
    - Run a test and confirm that the upstream node exposes the response body in `$json.data`.
    - If your n8n version returns a different field path, adjust the expressions accordingly.

13. **Add the actual API request node**
    - Name it: `Targeted API`
    - Connect `Merge` → `Targeted API`
    - Node type: `HTTP Request`
    - Configure:
      - URL: replace `API_URL` with your real endpoint
      - Method: choose the method required by your API
      - Send Headers: enabled
      - Add header:
        - Name: `user-agent`
        - Value: `{{ $json["clean_user-agent"] }}`
      - In Options, set Proxy to:
        - `http://{{ $json.proxy_username }}:{{ $json.proxy_password }}@gate.decodo.com:{{ $json.proxy_port }}`
    - Add any additional API-specific requirements:
      - authentication
      - query parameters
      - request body
      - content type
      - cookies
      - timeout settings

14. **Add optional sticky notes for documentation**
    - One note around the full flow describing purpose and setup expectations.
    - One near the proxy set node: `Add proxy credentials here`
    - One near the limit node: `Update the number of user-agent you want to use`
    - One near the target API node: `Add the HTTP node for the API you want to call`
    - One near the Cloudflare parsing nodes: `Informative node to show which IP_address/user-agent are being used`

15. **Test the workflow**
    - Execute the workflow manually.
    - Confirm:
      - user-agents are fetched correctly
      - at least several clean user-agent strings are generated
      - the `Merge` output contains both proxy and user-agent data
      - Cloudflare trace returns expected `ip=` and `uag=` values
      - the target API request succeeds

16. **Validate proxy behavior**
    - If your proxy service supports rotating sessions, repeated items should often show different outbound IPs.
    - If IPs are not changing:
      - verify proxy product type
      - verify session/rotation settings at the provider level
      - check whether your provider expects additional username modifiers for rotation
      - confirm port and gateway host are correct

17. **Harden the workflow for production use**
    - Replace static proxy secrets in the `Set` node with environment variables, credentials, or another secure source if possible.
    - Consider adding:
      - error handling branches
      - retry logic
      - fallback user-agent source
      - filtering of empty or malformed user-agents
      - pacing/rate-limiting between API calls

18. **Important implementation notes**
    - No dedicated n8n credential object is used in this workflow as provided; proxy credentials are assembled into the proxy URL string dynamically.
    - There is no sub-workflow in this design.
    - There is only one entry point: `Manual trigger`.
    - The Cloudflare branch is optional but strongly recommended during setup and debugging.

## Expected Input/Output Behavior

### Workflow input
- Manual start only
- No external payload required

### Intermediate merged item shape
Each merged item is expected to contain at least:
- `clean_user-agent`
- `proxy_username`
- `proxy_password`
- `proxy_port`

### Verification output
The informational branch should produce:
- `Ip_address`
- `user-agent`

### Target API execution pattern
The `Targeted API` node will execute once per selected user-agent item, which means:
- with `maxItems = 5`, the API is called 5 times per workflow run, unless upstream extraction returns fewer than 5 valid values.

## Common Modification Points

- **Change number of requests per run:** adjust `Take X random user-agents`
- **Change user-agent source:** replace `user-agents` URL and revise CSS selector in `Extract user-agent values`
- **Use another proxy provider:** keep the same dynamic proxy pattern but change gateway host and possibly authentication format
- **Apply to another HTTP node:** copy the same header and proxy expressions into any other `HTTP Request` node in the workflow