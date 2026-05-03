Workflow for Submitting Changed Sitemap URLs using Google Indexing API and Bing IndexNow

https://n8nworkflows.xyz/workflows/workflow-for-submitting-changed-sitemap-urls-using-google-indexing-api-and-bing-indexnow-8778


# Workflow for Submitting Changed Sitemap URLs using Google Indexing API and Bing IndexNow

### 1. Workflow Overview

This workflow automates the submission of recently changed URLs from a website's sitemap to Google’s Indexing API and Bing’s IndexNow service. Its primary use case is to help SEO professionals or webmasters ensure that search engines are promptly notified about updated or new pages for faster indexing.

The workflow is logically divided into the following main blocks:

- **1.1 Input Reception and Configuration:** Accepts manual or scheduled triggers and sets essential configuration variables like site URL, sitemap URL, batch size, and API keys.
- **1.2 Sitemap Retrieval and Parsing:** Downloads the main sitemap, parses it to extract individual content sitemaps, downloads and parses each of those, and compiles a consolidated list of URLs ordered by last modification date.
- **1.3 Filtering URLs by Last Modified Date:** Filters URLs to only those updated within a configurable number of days.
- **1.4 Google Indexing API Submission:** Checks the indexing status of each URL on Google, determines if an update is needed, and submits update notifications in batches with randomized wait times to avoid throttling.
- **1.5 Bing IndexNow Submission:** Splits URLs into batches, builds the payload according to IndexNow specifications, submits the payload to Bing’s API, and includes jittered waits between requests.
- **1.6 Gates for Selective Execution:** Conditional nodes that ensure either Google or IndexNow submission blocks only run if enabled in the configuration.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Configuration

**Overview:**  
This block activates the workflow either manually or on a weekly schedule and sets user-configurable parameters for the entire process.

**Nodes Involved:**  
- When clicking "Test workflow" (Manual Trigger)  
- Schedule Trigger  
- Config (Set node)  
- Sticky Note1 (Configuration instructions)

**Node Details:**  

- **When clicking "Test workflow"**  
  - Type: Manual Trigger  
  - Role: Allows manual execution for testing or ad-hoc runs.  
  - Inputs: None  
  - Outputs: Connects to Config node.  
  - Edge cases: None, but manual trigger requires user interaction.

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Automatically triggers the workflow every 7 days at 02:20 AM.  
  - Configuration: Runs weekly (`daysInterval=7`), at hour 2, minute 20.  
  - Inputs: None  
  - Outputs: Connects to Config node.  
  - Edge cases: Timezone considerations may affect trigger time.

- **Config**  
  - Type: Set  
  - Role: Defines workflow variables such as:  
    - SITE_URL (string): Base URL of the website  
    - SITEMAP_URL (string): URL of the sitemap.xml file  
    - DAYS_BACK (number): Time window in days to consider URLs as recently updated (default 7)  
    - BATCH_SIZE (number): Max URLs per batch for IndexNow (default 500)  
    - USE_GOOGLE (boolean): Enable Google submission (default true)  
    - USE_INDEXNOW (boolean): Enable Bing IndexNow submission (default true)  
    - INDEXNOW_KEY (string): API key for IndexNow  
    - INDEXNOW_KEY_URL (string): URL where the IndexNow key file is hosted  
  - Inputs: From manual or schedule trigger  
  - Outputs: To Get sitemap.xml node  
  - Edge cases: Missing or incorrect values can cause failures downstream.

- **Sticky Note1**  
  - Type: Sticky Note  
  - Role: Provides detailed instructions on configuring the above variables, including links to Bing Webmaster tools for IndexNow key creation and notes on the default values.

---

#### 2.2 Sitemap Retrieval and Parsing

**Overview:**  
Downloads the main sitemap, parses it to JSON, extracts nested sitemaps, downloads and parses each nested sitemap, and finally extracts and sorts all URLs by last modification date.

**Nodes Involved:**  
- Get sitemap.xml (HTTP Request)  
- Convert sitemap to JSON (XML)  
- Get content-specific sitemaps (SplitOut)  
- Get content of each sitemap (HTTP Request)  
- convert page data to JSON (XML)  
- Force urlset.url to array (Set)  
- Split Out (SplitOut)  
- Sort (Sort)  
- Assign mandatory sitemap fields (Set)  
- Sticky Note (Parsing instructions)

**Node Details:**  

- **Get sitemap.xml**  
  - Type: HTTP Request  
  - Role: Downloads the sitemap.xml from the URL specified in SITEMAP_URL.  
  - Configuration: URL set dynamically from config.  
  - Input: Config node  
  - Output: To Convert sitemap to JSON  
  - Edge cases: HTTP errors, sitemap URL invalid or unreachable.

- **Convert sitemap to JSON**  
  - Type: XML  
  - Role: Converts the downloaded sitemap XML into JSON format.  
  - Input: HTTP Request node  
  - Output: To Get content-specific sitemaps  
  - Edge cases: Malformed XML causing parse failure.

- **Get content-specific sitemaps**  
  - Type: SplitOut  
  - Role: Splits the sitemapindex.sitemap array to process each nested sitemap individually.  
  - Input: Parsed JSON from the previous node  
  - Output: To Get content of each sitemap  
  - Edge cases: Empty or missing sitemapindex.sitemap array.

- **Get content of each sitemap**  
  - Type: HTTP Request  
  - Role: Downloads each nested sitemap URL.  
  - Input: Each sitemap URL from the SplitOut node.  
  - Output: To convert page data to JSON  
  - Edge cases: HTTP errors for nested sitemaps.

- **convert page data to JSON**  
  - Type: XML  
  - Role: Converts each nested sitemap XML into JSON.  
  - Configuration: explicitArray: false (to simplify JSON structure).  
  - Input: HTTP Request node  
  - Output: To Force urlset.url to array  
  - Edge cases: Malformed XML.

- **Force urlset.url to array**  
  - Type: Set  
  - Role: Ensures that urlset.url is always an array regardless of whether the sitemap contains one or multiple URLs.  
  - Input: parsed JSON  
  - Output: To Split Out  
  - Edge cases: urlset or urlset.url missing.

- **Split Out**  
  - Type: SplitOut  
  - Role: Splits the array of URLs so each URL can be processed individually.  
  - Input: Set node  
  - Output: To Sort  
  - Edge cases: Empty array handling.

- **Sort**  
  - Type: Sort  
  - Role: Sorts URLs in descending order by their lastmod date to prioritize recent updates.  
  - Input: SplitOut  
  - Output: To Assign mandatory sitemap fields  
  - Edge cases: Missing lastmod values.

- **Assign mandatory sitemap fields**  
  - Type: Set  
  - Role: Maps and ensures lastmod and loc fields are explicitly assigned for each item.  
  - Input: Sorted URLs  
  - Output: To Filter: lastmod within DAYS_BACK  
  - Edge cases: Missing loc or lastmod fields.

- **Sticky Note**  
  - Type: Sticky Note  
  - Role: Notes that this section parses the sitemap and orders URLs by lastmod; no configuration changes needed here.

---

#### 2.3 Filtering URLs by Last Modified Date

**Overview:**  
Filters URLs to only those updated within the configured number of DAYS_BACK (default 7 days).

**Nodes Involved:**  
- Filter: lastmod within DAYS_BACK (Code)  
- Sticky Note2 (Google submission instructions)

**Node Details:**  

- **Filter: lastmod within DAYS_BACK**  
  - Type: Code  
  - Role: Filters the input URLs, keeping only those with a lastmod date within the last DAYS_BACK days.  
  - JavaScript logic:  
    - Reads DAYS_BACK from config  
    - Computes cutoff date  
    - Filters items having lastmod >= cutoff date  
  - Input: URLs with lastmod and loc  
  - Output: To Google and IndexNow gates  
  - Edge cases: Missing or invalid lastmod dates; returns empty list if none match.

- **Sticky Note2**  
  - Type: Sticky Note  
  - Role: Provides detailed instructions on setting up Google Service Account credentials required for the Google Indexing API nodes, including necessary permissions and Search Console user settings.

---

#### 2.4 Google Indexing API Submission

**Overview:**  
Loops over filtered URLs, checks their current indexing status on Google, and submits URLs that have new updates. Includes waits with jitter to avoid API throttling.

**Nodes Involved:**  
- Gate: Google (Code)  
- Loop Over Items (Google) (SplitInBatches)  
- Check status (Google) (HTTP Request)  
- is new? (Google) (If)  
- URL Updated (Google) (HTTP Request)  
- Wait (Google jitter) (Wait)  
- Sticky Note2 (Credentials instructions)

**Node Details:**  

- **Gate: Google**  
  - Type: Code  
  - Role: Passes input items only if USE_GOOGLE is true in config; otherwise, skips Google submission.  
  - Input: Filtered URLs  
  - Output: Loop Over Items (Google) or empty  
  - Edge cases: Incorrect config disables Google block.

- **Loop Over Items (Google)**  
  - Type: SplitInBatches  
  - Role: Processes URLs one at a time for Google API calls.  
  - Input: From Gate: Google  
  - Output: To Check status (Google)  
  - Edge cases: Large URL sets cause longer runtimes.

- **Check status (Google)**  
  - Type: HTTP Request  
  - Role: Queries Google Indexing API to get metadata about the URL’s indexing status.  
  - URL: `https://indexing.googleapis.com/v3/urlNotifications/metadata?url={{ encodeURIComponent($json.loc) }}`  
  - Authentication: Predefined Google Service Account credential  
  - On error: Continues workflow (does not fail)  
  - Outputs:  
    - Main: If success, to is new? (Google)  
    - Error path: To URL Updated (Google) (fallback)  
  - Edge cases: API auth errors, rate limits, network failures.

- **is new? (Google)**  
  - Type: If  
  - Role: Compares lastmod date with Google's latestUpdate.notifyTime to decide if URL needs re-submission.  
  - Condition: lastmod > latestUpdate.notifyTime  
  - True output: To URL Updated (Google)  
  - False output: Ends loop for current item (no update)  
  - Edge cases: Missing notifyTime field, date parse errors.

- **URL Updated (Google)**  
  - Type: HTTP Request  
  - Role: Submits URL update notification to Google Indexing API.  
  - URL: `https://indexing.googleapis.com/v3/urlNotifications:publish`  
  - Method: POST  
  - JSON Body: `{ "url": ..., "type": "URL_UPDATED" }`  
  - Authentication: Predefined Google Service Account credential  
  - Output: To Wait (Google jitter)  
  - Edge cases: API rate limits, auth errors.

- **Wait (Google jitter)**  
  - Type: Wait  
  - Role: Adds a randomized delay between 0.25 and 1 second to avoid API throttling.  
  - Parameter: `amount = 0.25 + random * 0.75` seconds  
  - Output: Back to Loop Over Items (Google) for next URL.

- **Sticky Note2** (see above)

---

#### 2.5 Bing IndexNow Submission

**Overview:**  
Submits batches of URLs updated within the configured time frame to Bing’s IndexNow API, including batch splitting and payload construction according to API specs.

**Nodes Involved:**  
- Gate: IndexNow (Code)  
- Split In Batches (IndexNow ≤500) (SplitInBatches)  
- Build IndexNow payload (Code)  
- IndexNow Submit (HTTP Request)  
- Wait (IndexNow jitter) (Wait)  
- Sticky Note3 (IndexNow notes)

**Node Details:**  

- **Gate: IndexNow**  
  - Type: Code  
  - Role: Passes input URLs only if USE_INDEXNOW is true; otherwise skips IndexNow submission.  
  - Input: Filtered URLs  
  - Output: To Split In Batches (IndexNow ≤500)  
  - Edge cases: Disabled in config disables this block.

- **Split In Batches (IndexNow ≤500)**  
  - Type: SplitInBatches  
  - Role: Splits URLs into batches with max size set by BATCH_SIZE config, capped at 500 per IndexNow API requirements.  
  - Batch size expression: `Math.min(BATCH_SIZE, 500)`  
  - Input: URLs from Gate: IndexNow  
  - Output: To Build IndexNow payload  
  - Edge cases: Empty input, batch size misconfiguration.

- **Build IndexNow payload**  
  - Type: Code  
  - Role:  
    - Sanitizes and filters URLs to only absolute http/https URLs matching SITE_URL host  
    - Removes duplicates  
    - Constructs JSON payload with keys: host, key, keyLocation, urlList  
  - Inputs: Batch of URLs, plus config variables SITE_URL, INDEXNOW_KEY, INDEXNOW_KEY_URL  
  - Output: To IndexNow Submit  
  - Edge cases: No valid URLs after filtering results in empty output (skips submission).

- **IndexNow Submit**  
  - Type: HTTP Request  
  - Role: POSTs JSON payload to `https://api.indexnow.org/indexnow` as per IndexNow API spec.  
  - Sends full JSON body as constructed.  
  - Output: To Wait (IndexNow jitter)  
  - Edge cases: API errors, network issues, invalid key or URL.

- **Wait (IndexNow jitter)**  
  - Type: Wait  
  - Role: Adds a randomized delay between 0.25 and 1 second after each batch submission.  
  - Output: Loops back to Split In Batches for next batch.

- **Sticky Note3**  
  - Type: Sticky Note  
  - Role: Indicates no configuration changes are necessary in this block.

---

### 3. Summary Table

| Node Name                    | Node Type           | Functional Role                            | Input Node(s)                             | Output Node(s)                      | Sticky Note                                                                                                                             |
|------------------------------|---------------------|--------------------------------------------|------------------------------------------|-----------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| When clicking "Test workflow" | Manual Trigger      | Manual execution trigger                    | None                                     | Config                            |                                                                                                                                         |
| Schedule Trigger              | Schedule Trigger    | Scheduled weekly trigger                    | None                                     | Config                            |                                                                                                                                         |
| Config                       | Set                 | Sets configuration variables                | Manual Trigger, Schedule Trigger         | Get sitemap.xml                   | Config File instructions with key setup and usage notes                                                                                 |
| Get sitemap.xml              | HTTP Request        | Downloads sitemap.xml                        | Config                                   | Convert sitemap to JSON           |                                                                                                                                         |
| Convert sitemap to JSON      | XML                 | Parses sitemap XML to JSON                   | Get sitemap.xml                          | Get content-specific sitemaps     | Parsing sitemap and ordering URLs; no user edits needed                                                                                 |
| Get content-specific sitemaps| SplitOut            | Splits main sitemap into nested sitemaps    | Convert sitemap to JSON                   | Get content of each sitemap       |                                                                                                                                         |
| Get content of each sitemap  | HTTP Request        | Downloads each nested sitemap                | Get content-specific sitemaps            | convert page data to JSON         |                                                                                                                                         |
| convert page data to JSON    | XML                 | Parses nested sitemap XML to JSON            | Get content of each sitemap               | Force urlset.url to array         |                                                                                                                                         |
| Force urlset.url to array    | Set                 | Ensures urlset.url is always an array        | convert page data to JSON                 | Split Out                       |                                                                                                                                         |
| Split Out                   | SplitOut            | Splits array of URLs                          | Force urlset.url to array                 | Sort                            |                                                                                                                                         |
| Sort                        | Sort                | Sorts URLs descending by lastmod date        | Split Out                               | Assign mandatory sitemap fields   |                                                                                                                                         |
| Assign mandatory sitemap fields | Set               | Assigns lastmod and loc fields explicitly    | Sort                                    | Filter: lastmod within DAYS_BACK |                                                                                                                                         |
| Filter: lastmod within DAYS_BACK | Code             | Filters URLs updated within DAYS_BACK days   | Assign mandatory sitemap fields          | Gate: Google, Gate: IndexNow      | Google Autosubmitting instructions with credential setup details                                                                        |
| Gate: Google                | Code                | Enables/disables Google submission           | Filter: lastmod within DAYS_BACK          | Loop Over Items (Google)          |                                                                                                                                         |
| Loop Over Items (Google)    | SplitInBatches      | Processes URLs one by one for Google API     | Gate: Google                            | Check status (Google)             |                                                                                                                                         |
| Check status (Google)       | HTTP Request        | Checks Google indexing status of URL         | Loop Over Items (Google)                  | is new? (Google), URL Updated (Google) (error path) | Google Service Account credential setup instructions included                                                                            |
| is new? (Google)            | If                  | Determines if URL needs re-submission        | Check status (Google)                    | URL Updated (Google)              |                                                                                                                                         |
| URL Updated (Google)        | HTTP Request        | Submits URL update notification to Google   | is new? (Google), Check status (Google) | Wait (Google jitter)              |                                                                                                                                         |
| Wait (Google jitter)        | Wait                | Random delay 0.25-1s between Google requests | URL Updated (Google)                     | Loop Over Items (Google)          |                                                                                                                                         |
| Gate: IndexNow              | Code                | Enables/disables Bing IndexNow submission    | Filter: lastmod within DAYS_BACK          | Split In Batches (IndexNow ≤500) |                                                                                                                                         |
| Split In Batches (IndexNow ≤500) | SplitInBatches  | Splits URLs into batches for IndexNow         | Gate: IndexNow                         | Build IndexNow payload            |                                                                                                                                         |
| Build IndexNow payload      | Code                | Constructs IndexNow JSON payload per batch   | Split In Batches (IndexNow ≤500)          | IndexNow Submit                  |                                                                                                                                         |
| IndexNow Submit             | HTTP Request        | Submits payload to Bing IndexNow API          | Build IndexNow payload                   | Wait (IndexNow jitter)            |                                                                                                                                         |
| Wait (IndexNow jitter)      | Wait                | Random delay 0.25-1s between IndexNow batches | IndexNow Submit                        | Split In Batches (IndexNow ≤500) |                                                                                                                                         |
| Sticky Note1                | Sticky Note         | Configuration instructions                    | None                                     | None                            | Config File instructions including Bing Webmaster link https://www.bing.com/indexnow/getstarted                                         |
| Sticky Note                 | Sticky Note         | Parsing sitemap explanation                    | None                                     | None                            | Parsing the sitemap and generating a list of the urls order by last modification date. No edits needed                                  |
| Sticky Note2                | Sticky Note         | Google credentials and setup instructions      | None                                     | None                            | Details Google Service Account creation, roles, key setup, and Search Console permissions.                                               |
| Sticky Note3                | Sticky Note         | IndexNow submission note                        | None                                     | None                            | No changes needed here for IndexNow submission                                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node:** Name it "When clicking 'Test workflow'". No parameters.

2. **Create Schedule Trigger node:** Name it "Schedule Trigger".  
   - Set interval to every 7 days.  
   - Set trigger time to 02:20 AM.

3. **Create Set node:** Name it "Config".  
   - Create variables:  
     - SITE_URL (string, default empty)  
     - SITEMAP_URL (string, default empty)  
     - DAYS_BACK (number, default 7)  
     - BATCH_SIZE (number, default 500)  
     - USE_GOOGLE (boolean, default true)  
     - USE_INDEXNOW (boolean, default true)  
     - INDEXNOW_KEY (string, default empty)  
     - INDEXNOW_KEY_URL (string, default empty)

4. **Connect "When clicking 'Test workflow'" and "Schedule Trigger" nodes to "Config".**

5. **Create HTTP Request node:** Name it "Get sitemap.xml".  
   - URL parameter: `={{ $json.SITEMAP_URL }}` (dynamic from Config).  
   - Method: GET (default).  
   - No authentication.

6. **Create XML node:** Name it "Convert sitemap to JSON".  
   - No special options needed.

7. **Create SplitOut node:** Name it "Get content-specific sitemaps".  
   - Set field to split: `sitemapindex.sitemap`.

8. **Create HTTP Request node:** Name it "Get content of each sitemap".  
   - URL parameter: `={{ $json.loc }}`.  
   - Method: GET.

9. **Create XML node:** Name it "convert page data to JSON".  
   - Option explicitArray: false.

10. **Create Set node:** Name it "Force urlset.url to array".  
    - Assignment:  
      - `urlset.url` = expression:  
        `={{ $json.urlset && $json.urlset.url ? ($json.urlset.url[0] ? $json.urlset.url : [$json.urlset.url]) : [] }}`

11. **Create SplitOut node:** Name it "Split Out".  
    - Field to split: `urlset.url`.

12. **Create Sort node:** Name it "Sort".  
    - Sort by field: `lastmod` in descending order.

13. **Create Set node:** Name it "Assign mandatory sitemap fields".  
    - Assign:  
      - lastmod = `={{ $json.lastmod }}`  
      - loc = `={{ $json.loc }}`

14. **Create Code node:** Name it "Filter: lastmod within DAYS_BACK".  
    - JavaScript code:
    ```javascript
    const daysBack = $items('Config')[0].json.DAYS_BACK || 7;
    const cutoff = new Date(Date.now() - daysBack*24*60*60*1000);
    const items = $items().filter(i => {
      const lm = i.json.lastmod ? new Date(i.json.lastmod) : null;
      return lm && lm >= cutoff;}).map(i => ({ json: { loc: i.json.loc, lastmod: i.json.lastmod } }));
    return items;
    ```

15. **Create Code node:** Name it "Gate: Google".  
    - JavaScript code:
    ```javascript
    const cfg = $items('Config')[0]?.json ?? {};
    const on = (cfg.USE_GOOGLE === true) || String(cfg.USE_GOOGLE).toLowerCase() === 'true';
    return on ? $items() : [];
    ```

16. **Create SplitInBatches node:** Name it "Loop Over Items (Google)".  
    - Defaults are fine (process one item per batch).

17. **Create HTTP Request node:** Name it "Check status (Google)".  
    - URL:  
      `=https://indexing.googleapis.com/v3/urlNotifications/metadata?url={{ encodeURIComponent($json.loc) }}`  
    - Authentication: Predefined credential type → Google Service Account API (needs to be created).  
    - On error: Continue (do not fail).  
    - Full HTTP response enabled.

18. **Create If node:** Name it "is new? (Google)".  
    - Condition: Check if lastmod (`$('Loop Over Items (Google)').item.json.lastmod`) is after `$json.body.latestUpdate.notifyTime`.

19. **Create HTTP Request node:** Name it "URL Updated (Google)".  
    - URL: `https://indexing.googleapis.com/v3/urlNotifications:publish`  
    - Method: POST  
    - JSON Body:  
      ```json
      { "url": "{{ $('Loop Over Items (Google)').item.json.loc }}", "type": "URL_UPDATED" }
      ```  
    - Authentication: Same Google Service Account credential.

20. **Create Wait node:** Name it "Wait (Google jitter)".  
    - Unit: seconds  
    - Amount: expression: `={{ (0.25 + Math.random()*0.75).toFixed(2) }}`

21. **Connect nodes for Google block as follows:**  
    - Filter: lastmod within DAYS_BACK → Gate: Google  
    - Gate: Google → Loop Over Items (Google)  
    - Loop Over Items (Google) → Check status (Google)  
    - Check status (Google) → is new? (Google) (true path) → URL Updated (Google) → Wait (Google jitter) → Loop Over Items (Google)  
    - Check status (Google) → is new? (Google) (false path) → (end for that URL)  
    - Check status (Google) error path → URL Updated (Google)

22. **Create Code node:** Name it "Gate: IndexNow".  
    - JavaScript code:
    ```javascript
    const cfg = $items('Config')[0]?.json ?? {};
    const on = (cfg.USE_INDEXNOW === true) || String(cfg.USE_INDEXNOW).toLowerCase() === 'true';
    return on ? $items() : [];
    ```

23. **Create SplitInBatches node:** Name it "Split In Batches (IndexNow ≤500)".  
    - Batch size expression:  
      `={{ Math.min(($items('Config')[0].json.BATCH_SIZE || 500), 500) }}`

24. **Create Code node:** Name it "Build IndexNow payload".  
    - JavaScript code:
    ```javascript
    const batch = $items();
    if (!batch.length) return [];

    let urls = batch
      .map(i => (i && i.json && i.json.loc) ? String(i.json.loc).trim() : null)
      .filter(u => !!u);

    urls = urls.filter(u => /^https?:\/\//i.test(u));

    const siteUrl = $items('Config')[0].json.SITE_URL || '';
    const m = siteUrl.match(/^https?:\/\/([^/]+)/i);
    const host = m ? m[1] : '';

    urls = urls.filter(u => {
      const h = (u.match(/^https?:\/\/([^/]+)/i) || [])[1] || '';
      return h.toLowerCase() === host.toLowerCase();
    });

    urls = Array.from(new Set(urls));

    if (!urls.length || !host) return [];

    return [{
      json: {
        host,
        key: $items('Config')[0].json.INDEXNOW_KEY,
        keyLocation: $items('Config')[0].json.INDEXNOW_KEY_URL,
        urlList: urls
      }
    }];
    ```

25. **Create HTTP Request node:** Name it "IndexNow Submit".  
    - URL: `https://api.indexnow.org/indexnow`  
    - Method: POST  
    - Body type: JSON, send full body `$json` from previous node.

26. **Create Wait node:** Name it "Wait (IndexNow jitter)".  
    - Unit: seconds  
    - Amount: expression: `={{ (0.25 + Math.random()*0.75).toFixed(2) }}`

27. **Connect nodes for IndexNow block:**  
    - Filter: lastmod within DAYS_BACK → Gate: IndexNow  
    - Gate: IndexNow → Split In Batches (IndexNow ≤500)  
    - Split In Batches (IndexNow ≤500) → Build IndexNow payload  
    - Build IndexNow payload → IndexNow Submit  
    - IndexNow Submit → Wait (IndexNow jitter) → Split In Batches (IndexNow ≤500)

28. **Connect Config → Get sitemap.xml → Convert sitemap to JSON → Get content-specific sitemaps → Get content of each sitemap → convert page data to JSON → Force urlset.url to array → Split Out → Sort → Assign mandatory sitemap fields → Filter: lastmod within DAYS_BACK**

29. **Add Sticky Notes in appropriate places with the provided content to guide users on configuration and credentials setup.**

30. **Create Google Service Account credentials:**  
    - Via Google Cloud Console, create a service account with Owner role.  
    - Generate JSON key, configure credential in n8n with service account email and private key.  
    - Enable scope: `https://www.googleapis.com/auth/indexing`  
    - Add service account email to Google Search Console users with Owner rights.

31. **For IndexNow key:**  
    - Create key using Bing Webmaster tools at https://www.bing.com/indexnow/getstarted  
    - Host file at URL defined in INDEXNOW_KEY_URL (usually `https://www.example.com/<INDEXNOW_KEY>`).

---

### 5. General Notes & Resources

| Note Content                                                                                                                    | Context or Link                                               |
|---------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| Config File variables must be updated before running: SITE_URL, SITEMAP_URL, INDEXNOW_KEY, INDEXNOW_KEY_URL                      | Sticky Note1 in workflow; Bing IndexNow key creation: https://www.bing.com/indexnow/getstarted |
| Google Service Account credentials require Owner role, JSON key setup, and adding the service account email as Owner in Search Console | Sticky Note2; Google Cloud Console and Search Console setup details: https://console.cloud.google.com/iam-admin/serviceaccounts and https://search.google.com/search-console/users |
| The workflow uses random jittered waits between API calls to reduce chance of rate limiting or throttling                       | Wait nodes in Google and IndexNow submission blocks            |
| No user edits are needed in the sitemap parsing block unless sitemap structure changes                                           | Sticky Note on sitemap parsing block                            |
| The workflow supports toggling Google and IndexNow submission independently via USE_GOOGLE and USE_INDEXNOW config flags         | Config node variables                                           |

---

**Disclaimer:** The content above is generated exclusively from an n8n workflow automation. It complies strictly with content policies and contains no illegal or offensive elements. All processed data are legal and publicly accessible.