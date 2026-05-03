Index your website using IndexNow and your XML sitemap

https://n8nworkflows.xyz/workflows/index-your-website-using-indexnow-and-your-xml-sitemap-13775


# Index your website using IndexNow and your XML sitemap

## 1. Workflow Overview

This workflow automates URL submission to the **IndexNow** protocol by reading a website’s XML sitemap, extracting all listed URLs, filtering only pages updated after a chosen date, and sending the resulting list to the IndexNow API.

Typical use cases:
- Notify search engines about newly published or recently updated pages
- Reduce indexing delays for SEO-sensitive content sites
- Run a recurring indexing routine from a sitemap without manually curating URLs

### 1.1 Scheduled Execution
The workflow starts on a schedule and initializes runtime configuration values such as the sitemap URL, the IndexNow key, and the date threshold used to determine which pages are “recent”.

### 1.2 Sitemap Retrieval and Parsing
The workflow downloads the XML sitemap, converts the XML into structured JSON, and splits the sitemap entries into one item per URL record.

### 1.3 URL Filtering and Aggregation
Only pages whose `lastmod` value is newer than the configured threshold are kept. Those matching URLs are then aggregated into a single array suitable for the IndexNow API payload.

### 1.4 IndexNow Submission
The workflow sends the filtered list of URLs to `https://api.indexnow.org/indexnow` using the configured host, key, and key file location.

---

## 2. Block-by-Block Analysis

## 2.1 Scheduled Execution and Configuration

**Overview:**  
This block triggers the workflow automatically and defines the core runtime settings. It centralizes the values needed downstream so the rest of the workflow can refer to a single source of truth.

**Nodes Involved:**  
- Run Daily Indexing
- Configuration

### Node: Run Daily Indexing
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Entry-point trigger that launches the workflow on a recurring schedule.
- **Configuration choices:**  
  Configured with an interval-based rule. The current JSON indicates a default interval setup, intended to run regularly; the sticky note suggests daily execution.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: none, this is the trigger
  - Output: Configuration
- **Version-specific requirements:**  
  `typeVersion: 1.3`
- **Edge cases or potential failure types:**  
  - Workflow inactive: no executions occur
  - Misconfigured schedule: runs too often or not at expected times
  - Timezone expectations may differ from operator assumptions
- **Sub-workflow reference:**  
  None

### Node: Configuration
- **Type and technical role:** `n8n-nodes-base.set`  
  Creates static and dynamic configuration fields used across the workflow.
- **Configuration choices:**  
  Defines three fields:
  - `sitemap_url`: XML sitemap endpoint, currently `https://example.com/sitemap.xml`
  - `modified_after`: dynamic ISO timestamp using `{{$now.minus(1, 'week').toISO()}}`
  - `indexnow_key`: static IndexNow key string
- **Key expressions or variables used:**  
  - `{{$now.minus(1, 'week').toISO()}}`
- **Input and output connections:**  
  - Input: Run Daily Indexing
  - Output: Read Sitemap
- **Version-specific requirements:**  
  `typeVersion: 3.4`
- **Edge cases or potential failure types:**  
  - Placeholder `sitemap_url` not replaced in production
  - Invalid `modified_after` string causing date comparison issues later
  - Incorrect or revoked IndexNow key
  - If the website does not host the key file at the expected location, IndexNow may reject submissions
- **Sub-workflow reference:**  
  None

---

## 2.2 Sitemap Retrieval and Parsing

**Overview:**  
This block downloads the XML sitemap and transforms it into a JSON structure n8n can process. It then expands the array of sitemap URLs into separate items for filtering.

**Nodes Involved:**  
- Read Sitemap
- Parse XML
- Split Out URLs

### Node: Read Sitemap
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Fetches the sitemap document from the configured URL.
- **Configuration choices:**  
  Uses the URL from the Configuration node:
  - `{{$('Configuration').item.json.sitemap_url}}`
  Default request options are used.
- **Key expressions or variables used:**  
  - `{{$('Configuration').item.json.sitemap_url}}`
- **Input and output connections:**  
  - Input: Configuration
  - Output: Parse XML
- **Version-specific requirements:**  
  `typeVersion: 4.3`
- **Edge cases or potential failure types:**  
  - 404/403/500 from the sitemap endpoint
  - Redirect or CDN restrictions
  - Timeout on large sitemaps
  - Non-XML response body
  - Compressed sitemap formats not handled if the endpoint returns an unsupported format
- **Sub-workflow reference:**  
  None

### Node: Parse XML
- **Type and technical role:** `n8n-nodes-base.xml`  
  Converts the downloaded XML sitemap into JSON.
- **Configuration choices:**  
  Uses default XML parsing options.
- **Key expressions or variables used:**  
  None directly.
- **Input and output connections:**  
  - Input: Read Sitemap
  - Output: Split Out URLs
- **Version-specific requirements:**  
  `typeVersion: 1`
- **Edge cases or potential failure types:**  
  - Malformed XML
  - Namespace-heavy sitemap structures may parse into unexpected field paths
  - Very large sitemap payloads can increase memory usage
- **Sub-workflow reference:**  
  None

### Node: Split Out URLs
- **Type and technical role:** `n8n-nodes-base.splitOut`  
  Splits the parsed sitemap URL array into one item per URL entry.
- **Configuration choices:**  
  Splits the field:
  - `urlset.url`
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Input: Parse XML
  - Output: Filter Last Modified Pages
- **Version-specific requirements:**  
  `typeVersion: 1`
- **Edge cases or potential failure types:**  
  - If the parsed sitemap does not contain `urlset.url`, no useful items will be produced
  - Sitemap index files (`<sitemapindex>`) are not supported by this field path as configured
  - If some sitemap entries lack expected subfields like `loc` or `lastmod`, downstream logic may fail or skip them
- **Sub-workflow reference:**  
  None

---

## 2.3 URL Filtering and Aggregation

**Overview:**  
This block keeps only URLs modified after the configured threshold and converts the surviving `loc` values into a single array. That array matches the `urlList` structure expected by the IndexNow API.

**Nodes Involved:**  
- Filter Last Modified Pages
- Aggregate URLs

### Node: Filter Last Modified Pages
- **Type and technical role:** `n8n-nodes-base.filter`  
  Filters sitemap items based on their `lastmod` value.
- **Configuration choices:**  
  Uses a date-time comparison:
  - Left value: `{{$json.lastmod}}`
  - Operator: `after`
  - Right value: `{{$('Configuration').item.json.modified_after}}`
  Strict type validation is enabled.
- **Key expressions or variables used:**  
  - `{{$json.lastmod}}`
  - `{{$('Configuration').item.json.modified_after}}`
- **Input and output connections:**  
  - Input: Split Out URLs
  - Output: Aggregate URLs
- **Version-specific requirements:**  
  `typeVersion: 2.3`
- **Edge cases or potential failure types:**  
  - Missing `lastmod` values
  - Invalid or non-ISO date strings in sitemap entries
  - Strict validation may reject mismatched types
  - Timezone differences between sitemap dates and threshold date may affect inclusion
- **Sub-workflow reference:**  
  None

### Node: Aggregate URLs
- **Type and technical role:** `n8n-nodes-base.aggregate`  
  Collects the filtered `loc` field values into a single output field named `urlList`.
- **Configuration choices:**  
  Aggregates the field:
  - source field: `loc`
  - output field: `urlList`
- **Key expressions or variables used:**  
  None directly.
- **Input and output connections:**  
  - Input: Filter Last Modified Pages
  - Output: Send URLs to IndexNow
- **Version-specific requirements:**  
  `typeVersion: 1`
- **Edge cases or potential failure types:**  
  - If no pages pass the filter, the aggregate output may be empty, depending on execution behavior
  - Missing `loc` fields create incomplete payloads
  - Large URL lists may approach API payload limits
- **Sub-workflow reference:**  
  None

---

## 2.4 IndexNow Submission

**Overview:**  
This block builds and sends the IndexNow-compliant payload. It uses the configured sitemap domain as the host and constructs the public key file URL required by IndexNow verification.

**Nodes Involved:**  
- Send URLs to IndexNow

### Node: Send URLs to IndexNow
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends a POST request to the IndexNow API with the final URL batch.
- **Configuration choices:**  
  - URL: `https://api.indexnow.org/indexnow`
  - Method: `POST`
  - Request body enabled
  - Body parameters:
    - `host`: `{{$('Configuration').item.json.sitemap_url.extractDomain()}}`
    - `key`: `{{$('Configuration').item.json.indexnow_key}}`
    - `keyLocation`: `=https://{{ $('Configuration').item.json.sitemap_url.extractDomain() }}/{{ $('Configuration').item.json.indexnow_key }}.txt`
    - `urlList`: `{{$json.urlList}}`
- **Key expressions or variables used:**  
  - `{{$('Configuration').item.json.sitemap_url.extractDomain()}}`
  - `{{$('Configuration').item.json.indexnow_key}}`
  - `=https://{{ $('Configuration').item.json.sitemap_url.extractDomain() }}/{{ $('Configuration').item.json.indexnow_key }}.txt`
  - `{{$json.urlList}}`
- **Input and output connections:**  
  - Input: Aggregate URLs
  - Output: none
- **Version-specific requirements:**  
  `typeVersion: 4.3`
- **Edge cases or potential failure types:**  
  - IndexNow rejects requests if the key file is not publicly accessible at `keyLocation`
  - `extractDomain()` behavior depends on the n8n expression environment supporting that helper
  - Empty `urlList` may cause a no-op or API rejection
  - Host and URLs must match ownership expectations of IndexNow
  - Rate limits or network failures may interrupt submission
- **Sub-workflow reference:**  
  None

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Run Daily Indexing | Schedule Trigger | Starts the workflow on a recurring schedule |  | Configuration | ## Index your website using IndexNow and XML sitemap\nStop waiting for search engines to discover your content updates. This workflow automates the process of notifying search engines (Bing, Yandex, etc.) about new or updated pages using the IndexNow protocol.\nBy parsing your XML sitemap and filtering for pages modified within a specific timeframe (e.g., the last week), it ensures your crawl budget is used efficiently and your latest content appears in search results almost instantly.\n### Setup steps:\n1. **Host your Key:** Generate an IndexNow key and upload the `.txt` file to your website's root directory.\n2. **Setup Configuration Node:**\n* Enter your `sitemap_url` (e.g., https://yoursite.com/sitemap.xml).\n* Enter your `indexnow_key`.\n* Set the `modified_after` variable to define your lookback window (e.g., -7d or a specific ISO date).\n3. **Schedule:** Adjust the Schedule Trigger to your preferred frequency (default is daily).\n4. **Activate:** Once configured, turn the workflow on to start proactive indexing.\nFor a deep dive into how this workflow works and why it's a game-changer for your SEO strategy, check out our full guide on n8nplaybook.com.\n[Read the full Playbook Article here](https://n8nplaybook.com/post/2026/02/automate-indexnow-seo-n8n-workflow/) |
| Configuration | Set | Stores sitemap URL, modified-after threshold, and IndexNow key | Run Daily Indexing | Read Sitemap | ## Index your website using IndexNow and XML sitemap\nStop waiting for search engines to discover your content updates. This workflow automates the process of notifying search engines (Bing, Yandex, etc.) about new or updated pages using the IndexNow protocol.\nBy parsing your XML sitemap and filtering for pages modified within a specific timeframe (e.g., the last week), it ensures your crawl budget is used efficiently and your latest content appears in search results almost instantly.\n### Setup steps:\n1. **Host your Key:** Generate an IndexNow key and upload the `.txt` file to your website's root directory.\n2. **Setup Configuration Node:**\n* Enter your `sitemap_url` (e.g., https://yoursite.com/sitemap.xml).\n* Enter your `indexnow_key`.\n* Set the `modified_after` variable to define your lookback window (e.g., -7d or a specific ISO date).\n3. **Schedule:** Adjust the Schedule Trigger to your preferred frequency (default is daily).\n4. **Activate:** Once configured, turn the workflow on to start proactive indexing.\nFor a deep dive into how this workflow works and why it's a game-changer for your SEO strategy, check out our full guide on n8nplaybook.com.\n[Read the full Playbook Article here](https://n8nplaybook.com/post/2026/02/automate-indexnow-seo-n8n-workflow/) |
| Read Sitemap | HTTP Request | Downloads the XML sitemap | Configuration | Parse XML | ## Get URLs from website\nRead sitemap.xml, parse it and split out URLs from it |
| Parse XML | XML | Converts sitemap XML into JSON | Read Sitemap | Split Out URLs | ## Get URLs from website\nRead sitemap.xml, parse it and split out URLs from it |
| Split Out URLs | Split Out | Expands sitemap entries into individual items | Parse XML | Filter Last Modified Pages | ## Get URLs from website\nRead sitemap.xml, parse it and split out URLs from it |
| Filter Last Modified Pages | Filter | Keeps only URLs newer than the configured date | Split Out URLs | Aggregate URLs | ## Prepare URLs from submission\nFilter last modified URLs and convert them into array |
| Aggregate URLs | Aggregate | Builds the final `urlList` array for submission | Filter Last Modified Pages | Send URLs to IndexNow | ## Prepare URLs from submission\nFilter last modified URLs and convert them into array |
| Send URLs to IndexNow | HTTP Request | Sends the IndexNow batch request | Aggregate URLs |  | ## Submit |
| Sticky Note | Sticky Note | Documentation / canvas note |  |  |  |
| Sticky Note1 | Sticky Note | Documentation / canvas note |  |  |  |
| Sticky Note2 | Sticky Note | Documentation / canvas note |  |  |  |
| Sticky Note3 | Sticky Note | Documentation / canvas note |  |  |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**  
   Name it: **Index your website using IndexNow and XML sitemap**.

2. **Add a Schedule Trigger node**
   - Node type: **Schedule Trigger**
   - Name: **Run Daily Indexing**
   - Configure it with a recurring interval
   - Set it to run daily, or another cadence that matches your publishing frequency

3. **Add a Set node**
   - Node type: **Set**
   - Name: **Configuration**
   - Create these fields:
     1. `sitemap_url` as String  
        Example: `https://example.com/sitemap.xml`
     2. `modified_after` as String  
        Expression: `{{$now.minus(1, 'week').toISO()}}`
     3. `indexnow_key` as String  
        Example: your IndexNow key value
   - Connect **Run Daily Indexing → Configuration**

4. **Prepare the IndexNow key on your website**
   - Generate an IndexNow key
   - Create a file named `<your-key>.txt`
   - Put the key content in that file as required by IndexNow
   - Upload that file to the root of your website so it is accessible at:  
     `https://yourdomain.com/<your-key>.txt`
   - This is mandatory for successful verification

5. **Add an HTTP Request node to fetch the sitemap**
   - Node type: **HTTP Request**
   - Name: **Read Sitemap**
   - Method: `GET` is sufficient
   - URL expression: `{{$('Configuration').item.json.sitemap_url}}`
   - Leave default options unless your sitemap requires redirects, custom headers, or special SSL handling
   - Connect **Configuration → Read Sitemap**

6. **Add an XML node**
   - Node type: **XML**
   - Name: **Parse XML**
   - Use default parsing options
   - Connect **Read Sitemap → Parse XML**

7. **Add a Split Out node**
   - Node type: **Split Out**
   - Name: **Split Out URLs**
   - Field to split out: `urlset.url`
   - This assumes your sitemap is a standard URL set and not a sitemap index
   - Connect **Parse XML → Split Out URLs**

8. **Add a Filter node**
   - Node type: **Filter**
   - Name: **Filter Last Modified Pages**
   - Add one condition:
     - Left Value: `{{$json.lastmod}}`
     - Operator: **Date & Time → after**
     - Right Value: `{{$('Configuration').item.json.modified_after}}`
   - Keep strict validation enabled if you want malformed dates to fail fast
   - Connect **Split Out URLs → Filter Last Modified Pages**

9. **Add an Aggregate node**
   - Node type: **Aggregate**
   - Name: **Aggregate URLs**
   - Aggregate field:
     - Input field: `loc`
     - Rename output field: enabled
     - Output field name: `urlList`
   - This creates a single item containing an array of URLs
   - Connect **Filter Last Modified Pages → Aggregate URLs**

10. **Add the final HTTP Request node for IndexNow**
    - Node type: **HTTP Request**
    - Name: **Send URLs to IndexNow**
    - Method: `POST`
    - URL: `https://api.indexnow.org/indexnow`
    - Enable request body
    - Add body parameters:
      1. `host`  
         Expression: `{{$('Configuration').item.json.sitemap_url.extractDomain()}}`
      2. `key`  
         Expression: `{{$('Configuration').item.json.indexnow_key}}`
      3. `keyLocation`  
         Expression: `=https://{{ $('Configuration').item.json.sitemap_url.extractDomain() }}/{{ $('Configuration').item.json.indexnow_key }}.txt`
      4. `urlList`  
         Expression: `{{$json.urlList}}`
    - Connect **Aggregate URLs → Send URLs to IndexNow**

11. **Validate the sitemap structure**
    - Run the workflow manually once
    - Inspect the output of **Parse XML**
    - Confirm that URLs are actually located at `urlset.url`
    - If your sitemap uses a different structure or namespace handling, adjust the Split Out field path

12. **Validate the last modification format**
    - Check that each URL item includes `lastmod`
    - Confirm that `lastmod` is in a date format the Filter node can compare
    - If not, add a transformation node before filtering to normalize date strings

13. **Test the final payload**
    - Execute the workflow
    - Inspect the output of **Aggregate URLs** and verify that `urlList` is a proper array of absolute URLs
    - Verify that the host extracted from the sitemap URL matches the domain owning the URLs

14. **Activate the workflow**
    - After successful testing, activate the workflow so the Schedule Trigger runs automatically

### Credential configuration
This workflow, as provided, does **not** require authenticated credentials for its nodes:
- The sitemap is fetched anonymously
- The IndexNow API submission is performed without OAuth/API credential objects in n8n

However, you may need to configure HTTP options if:
- Your sitemap is protected
- Your site blocks certain user agents
- Your environment requires proxy or custom TLS behavior

### Default values and constraints
- `sitemap_url` should point directly to a standard XML sitemap
- `modified_after` should be a valid ISO datetime string if used with the current Filter configuration
- `indexnow_key` must correspond to a valid text file hosted at the website root
- All URLs in `urlList` should belong to the same verified host for reliable IndexNow acceptance

### Sub-workflow setup
No sub-workflows are used in this workflow.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Stop waiting for search engines to discover your content updates. This workflow automates the process of notifying search engines (Bing, Yandex, etc.) about new or updated pages using the IndexNow protocol. By parsing your XML sitemap and filtering for pages modified within a specific timeframe, it helps use crawl budget efficiently and accelerate appearance in search results. | Workflow purpose |
| Host your IndexNow key as a `.txt` file in the root of your site before activating the workflow. | Operational requirement |
| Suggested configuration fields: `sitemap_url`, `indexnow_key`, and `modified_after`. | Setup guidance |
| Suggested schedule: daily, but it can be changed to match site update frequency. | Operating cadence |
| Full Playbook article | https://n8nplaybook.com/post/2026/02/automate-indexnow-seo-n8n-workflow/ |