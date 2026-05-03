Scrape Shopify store leads with ScraperCity and send contacts to Slack

https://n8nworkflows.xyz/workflows/scrape-shopify-store-leads-with-scrapercity-and-send-contacts-to-slack-14058


# Scrape Shopify store leads with ScraperCity and send contacts to Slack

# 1. Workflow Overview

This workflow manually launches a Shopify lead scraping job through ScraperCity, waits until the remote scrape finishes, downloads the resulting CSV, converts it into structured lead records, removes duplicates, and sends each unique lead to a Slack channel.

Its main use case is outbound prospecting or market research: you define a target country and desired lead volume, and the workflow automates collection plus team delivery of store contact data.

## 1.1 Trigger and Search Configuration

The workflow starts with a manual trigger and a Set node that defines all search parameters used later in the ScraperCity API call: ecommerce platform, country, number of leads, whether to include emails and phones, and the Slack destination channel.

## 1.2 Scrape Job Submission

The configured parameters are sent to ScraperCity through an authenticated HTTP POST request. The returned scrape `runId` is extracted and preserved so all later polling and download requests can reference the same job.

## 1.3 Scrape Status Polling Loop

A loop waits 60 seconds between status checks, then calls ScraperCity’s status endpoint until the job reports `SUCCEEDED`. If the job is still running or in any non-success state, execution is routed back into the loop.

## 1.4 Result Download and Lead Parsing

Once the scrape completes, the workflow downloads the result file as text, assumes CSV content, and parses it in a Code node. The parser normalizes several possible column names into a consistent lead schema and skips rows without an email address.

## 1.5 Deduplication and Slack Delivery

Parsed leads are deduplicated by email, then each unique lead is posted to Slack as a formatted block message containing store name, email, phone, website, country, and social profiles where available.

---

# 2. Block-by-Block Analysis

## 2.1 Trigger and Search Configuration

### Overview
This block initializes the workflow and defines the business parameters that control the scrape request and downstream Slack routing. It is the only human-editable control surface required for normal usage.

### Nodes Involved
- When clicking 'Execute workflow'
- Configure Search Parameters

### Node Details

#### When clicking 'Execute workflow'
- **Type and technical role:** `n8n-nodes-base.manualTrigger`; manual entry point for on-demand execution.
- **Configuration choices:** No parameters are required; execution begins only when a user runs the workflow manually.
- **Key expressions or variables used:** None.
- **Input and output connections:** No input; outputs to **Configure Search Parameters**.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** No runtime failure in itself; this node simply cannot be used for scheduled or webhook-driven automation unless replaced.
- **Sub-workflow reference:** None.

#### Configure Search Parameters
- **Type and technical role:** `n8n-nodes-base.set`; defines static configuration values used by the API request and Slack delivery.
- **Configuration choices:** Creates the following fields:
  - `platform` = `shopify`
  - `countryCode` = `US`
  - `totalLeads` = `500`
  - `includeEmails` = `true`
  - `includePhones` = `true`
  - `slackChannel` = `#shopify-leads`
- **Key expressions or variables used:** Static values only.
- **Input and output connections:** Input from **When clicking 'Execute workflow'**; output to **Start Shopify Lead Scrape**.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - Invalid country code or unsupported platform may cause the downstream API request to fail or return unexpected data.
  - Excessively large `totalLeads` values may increase scrape duration or exceed API limits.
  - Slack channel name must match a channel visible to the Slack credential used later.
- **Sub-workflow reference:** None.

---

## 2.2 Scrape Job Submission

### Overview
This block submits the actual scrape request to ScraperCity and stores the returned job identifier. It also carries forward the Slack channel so downstream nodes do not depend on the original configuration item shape.

### Nodes Involved
- Start Shopify Lead Scrape
- Store Run ID

### Node Details

#### Start Shopify Lead Scrape
- **Type and technical role:** `n8n-nodes-base.httpRequest`; authenticated POST request to launch a ScraperCity lead scrape.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://app.scrapercity.com/api/v1/scrape/store-leads`
  - Body type: JSON
  - Authentication: generic credential type using HTTP Header Auth
- **Key expressions or variables used:**
  - `platform` from `{{$json.platform}}`
  - `countryCode` from `{{$json.countryCode}}`
  - `emails` from `{{$json.includeEmails}}`
  - `phones` from `{{$json.includePhones}}`
  - `totalLeads` from `{{$json.totalLeads}}`
- **Input and output connections:** Input from **Configure Search Parameters**; output to **Store Run ID**.
- **Version-specific requirements:** Type version `4.2`; requires a valid n8n HTTP Header Auth credential.
- **Edge cases or potential failure types:**
  - Missing or invalid ScraperCity API key will return authentication failures.
  - API may reject malformed payload values.
  - Network errors, rate limits, or temporary 5xx responses are possible.
  - If ScraperCity changes the response structure and does not return `runId`, downstream logic breaks.
- **Sub-workflow reference:** None.

#### Store Run ID
- **Type and technical role:** `n8n-nodes-base.set`; persists the ScraperCity `runId` and the Slack channel in a clean item schema.
- **Configuration choices:**
  - `runId` = `{{$json.runId}}`
  - `slackChannel` = `{{$('Configure Search Parameters').item.json.slackChannel}}`
- **Key expressions or variables used:**
  - Current node input: `runId`
  - Cross-node expression: `$('Configure Search Parameters').item.json.slackChannel`
- **Input and output connections:** Input from **Start Shopify Lead Scrape**; output to **Poll Loop Controller**.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - If the HTTP response has no `runId`, this node creates an empty string or expression error depending on actual response shape.
  - Cross-node expression assumes the original Set node produced a single matching item.
- **Sub-workflow reference:** None.

---

## 2.3 Scrape Status Polling Loop

### Overview
This block implements asynchronous job monitoring. It waits between checks, requests the scrape status, and branches based on completion state, looping until the job finishes successfully.

### Nodes Involved
- Poll Loop Controller
- Wait 60 Seconds
- Check Scrape Status
- Is Scrape Complete?

### Node Details

#### Poll Loop Controller
- **Type and technical role:** `n8n-nodes-base.splitInBatches`; used here as a loop controller rather than for batching a list.
- **Configuration choices:**
  - `reset: false`
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input from **Store Run ID**
  - Also receives loop-back input from the false branch of **Is Scrape Complete?**
  - Output to **Wait 60 Seconds**
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**
  - This looping pattern depends on correct wiring; miswiring can create an infinite immediate loop or stop execution.
  - Because no explicit max retry count exists, a job stuck forever may keep polling indefinitely.
- **Sub-workflow reference:** None.

#### Wait 60 Seconds
- **Type and technical role:** `n8n-nodes-base.wait`; pauses execution before each status poll.
- **Configuration choices:** Wait amount = `60` seconds.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from **Poll Loop Controller**; output to **Check Scrape Status**.
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**
  - Longer waits delay result delivery.
  - Very short waits could create unnecessary API pressure or hit rate limits.
  - On self-hosted instances, wait execution behavior depends on queue/execution persistence settings.
- **Sub-workflow reference:** None.

#### Check Scrape Status
- **Type and technical role:** `n8n-nodes-base.httpRequest`; authenticated GET request to retrieve the scrape job status.
- **Configuration choices:**
  - URL: `https://app.scrapercity.com/api/v1/scrape/status/{{ $('Store Run ID').item.json.runId }}`
  - Authentication: generic credential type using HTTP Header Auth
- **Key expressions or variables used:**
  - Cross-node reference to `$('Store Run ID').item.json.runId`
- **Input and output connections:** Input from **Wait 60 Seconds**; output to **Is Scrape Complete?**
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases or potential failure types:**
  - Invalid or expired run ID returns not found or API errors.
  - Missing authentication causes 401/403 responses.
  - If ScraperCity returns a different status property name, the IF condition will fail.
- **Sub-workflow reference:** None.

#### Is Scrape Complete?
- **Type and technical role:** `n8n-nodes-base.if`; evaluates whether the scrape finished successfully.
- **Configuration choices:**
  - Strict type validation
  - String equality comparison: `{{$json.status}} == "SUCCEEDED"`
- **Key expressions or variables used:**
  - `{{$json.status}}`
- **Input and output connections:**
  - Input from **Check Scrape Status**
  - True output to **Download Lead Results**
  - False output back to **Poll Loop Controller**
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**
  - Status values such as `FAILED`, `CANCELLED`, or unexpected text are all treated as false and will continue looping forever.
  - A safer production pattern would handle terminal failure states explicitly.
- **Sub-workflow reference:** None.

---

## 2.4 Result Download and Lead Parsing

### Overview
After successful completion, this block fetches the result file and transforms raw CSV into normalized lead items. It is also where filtering logic removes rows lacking email addresses.

### Nodes Involved
- Download Lead Results
- Parse CSV and Format Leads

### Node Details

#### Download Lead Results
- **Type and technical role:** `n8n-nodes-base.httpRequest`; downloads the completed scrape results from ScraperCity.
- **Configuration choices:**
  - URL: `https://app.scrapercity.com/api/downloads/{{ $('Store Run ID').item.json.runId }}`
  - Response format: text
  - Authentication: generic credential type using HTTP Header Auth
- **Key expressions or variables used:**
  - Cross-node reference to `$('Store Run ID').item.json.runId`
- **Input and output connections:** Input from the true branch of **Is Scrape Complete?**; output to **Parse CSV and Format Leads**.
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases or potential failure types:**
  - Download may fail if the result is not yet available even after status says complete.
  - If the endpoint returns binary or structured JSON instead of text, the parser may not work.
  - Authentication/network issues remain possible.
- **Sub-workflow reference:** None.

#### Parse CSV and Format Leads
- **Type and technical role:** `n8n-nodes-base.code`; parses CSV text manually and maps rows into normalized lead objects.
- **Configuration choices:**
  - Reads text from `data` or `body`
  - Splits by newline
  - Uses a custom `parseLine()` function to support quoted fields
  - Lowercases and underscore-normalizes headers
  - Supports multiple source column variants for email, phone, website, and social links
  - Filters out rows missing email
- **Key expressions or variables used:**
  - `const csvText = $input.first().json.data || $input.first().json.body || ''`
  - Email fallback chain:
    - `lead.email`
    - `lead.email_address`
    - `lead.contact_email`
  - Store name fallback chain:
    - `lead.store_name`
    - `lead.name`
    - `lead.shop_name`
    - `lead.domain`
    - `'Unknown Store'`
- **Input and output connections:** Input from **Download Lead Results**; output to **Remove Duplicate Leads**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - CSV parser is simple and may fail on complex escaping rules such as embedded escaped quotes.
  - Uses `split('\n')`, so carriage returns in Windows CSV may leave `\r` artifacts unless trimmed adequately.
  - Returns an empty array for empty downloads, malformed text, or files with only headers.
  - It intentionally discards rows without email, even if phone data exists.
- **Sub-workflow reference:** None.

---

## 2.5 Deduplication and Slack Delivery

### Overview
This final block removes duplicate lead records and sends each surviving lead to Slack in a rich text block format. It is the delivery stage consumed by the sales or research team.

### Nodes Involved
- Remove Duplicate Leads
- Post Lead to Slack

### Node Details

#### Remove Duplicate Leads
- **Type and technical role:** `n8n-nodes-base.removeDuplicates`; removes duplicates using selected fields.
- **Configuration choices:**
  - Compare mode: selected fields
  - Field used: `email`
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from **Parse CSV and Format Leads**; output to **Post Lead to Slack**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - Leads sharing an email across multiple storefront records will collapse into one item.
  - Duplicate detection is only based on exact normalized email text; the Code node helps by lowercasing emails first.
- **Sub-workflow reference:** None.

#### Post Lead to Slack
- **Type and technical role:** `n8n-nodes-base.slack`; posts each lead as a Slack block message to a channel selected by name.
- **Configuration choices:**
  - Message destination mode: channel
  - Channel selected by name from expression
  - Message type: block
  - `unfurl_links: false`
  - Block content includes store name, email, phone, website, country, Instagram, Facebook, Twitter
- **Key expressions or variables used:**
  - Channel: `{{$('Store Run ID').item.json.slackChannel}}`
  - Message body includes:
    - `{{$json.storeName}}`
    - `{{$json.email}}`
    - `{{$json.phone || '_not available_'}}`
    - `{{$json.website || '_not available_'}}`
    - `{{$json.country || '_not available_'}}`
    - `{{$json.instagram || '_not available_'}}`
    - `{{$json.facebook || '_not available_'}}`
    - `{{$json.twitter || '_not available_'}}`
- **Input and output connections:** Input from **Remove Duplicate Leads**; no downstream node.
- **Version-specific requirements:** Type version `2.3`; requires Slack OAuth credentials with access to the target channel.
- **Edge cases or potential failure types:**
  - Slack credential may lack permission to post to the selected channel.
  - Channel lookup by name may fail if the channel is private, renamed, or not visible to the app.
  - Large lead volumes can generate many Slack messages and may hit posting limits or become noisy.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking 'Execute workflow' | Manual Trigger | Manual start of the workflow |  | Configure Search Parameters | ## Trigger and search setup<br>Manual trigger that kicks off the workflow and a configuration node that sets the scrape parameters (platform, country, lead count, email flag). |
| Configure Search Parameters | Set | Defines scrape and Slack routing parameters | When clicking 'Execute workflow' | Start Shopify Lead Scrape | ## Trigger and search setup<br>Manual trigger that kicks off the workflow and a configuration node that sets the scrape parameters (platform, country, lead count, email flag). |
| Start Shopify Lead Scrape | HTTP Request | Submits lead scrape job to ScraperCity | Configure Search Parameters | Store Run ID | ## Scrape job submission<br>Sends the scrape request to the ScraperCity API and stores the returned run ID along with the target Slack channel for use throughout the rest of the workflow. |
| Store Run ID | Set | Stores run ID and Slack channel for downstream reuse | Start Shopify Lead Scrape | Poll Loop Controller | ## Scrape job submission<br>Sends the scrape request to the ScraperCity API and stores the returned run ID along with the target Slack channel for use throughout the rest of the workflow. |
| Poll Loop Controller | Split In Batches | Controls repeated polling loop | Store Run ID; Is Scrape Complete? (false) | Wait 60 Seconds | ## Scrape status polling loop<br>Repeatedly waits 60 seconds, checks the job status via the ScraperCity API, and loops back if the scrape is not yet complete, advancing only when the job finishes. |
| Wait 60 Seconds | Wait | Pauses between status checks | Poll Loop Controller | Check Scrape Status | ## Scrape status polling loop<br>Repeatedly waits 60 seconds, checks the job status via the ScraperCity API, and loops back if the scrape is not yet complete, advancing only when the job finishes. |
| Check Scrape Status | HTTP Request | Queries current ScraperCity job status | Wait 60 Seconds | Is Scrape Complete? | ## Scrape status polling loop<br>Repeatedly waits 60 seconds, checks the job status via the ScraperCity API, and loops back if the scrape is not yet complete, advancing only when the job finishes. |
| Is Scrape Complete? | If | Branches on completion state | Check Scrape Status | Download Lead Results; Poll Loop Controller | ## Scrape status polling loop<br>Repeatedly waits 60 seconds, checks the job status via the ScraperCity API, and loops back if the scrape is not yet complete, advancing only when the job finishes. |
| Download Lead Results | HTTP Request | Downloads completed CSV results | Is Scrape Complete? | Parse CSV and Format Leads | ## Lead download and parsing<br>Downloads the completed CSV result file from ScraperCity and parses it into structured lead objects ready for deduplication. |
| Parse CSV and Format Leads | Code | Parses CSV text and normalizes leads | Download Lead Results | Remove Duplicate Leads | ## Lead download and parsing<br>Downloads the completed CSV result file from ScraperCity and parses it into structured lead objects ready for deduplication. |
| Remove Duplicate Leads | Remove Duplicates | Removes duplicate leads by email | Parse CSV and Format Leads | Post Lead to Slack | ## Dedup and Slack delivery<br>Removes any duplicate leads from the parsed results and posts each unique lead as a message to the configured Slack channel. |
| Post Lead to Slack | Slack | Sends each unique lead to Slack | Remove Duplicate Leads |  | ## Dedup and Slack delivery<br>Removes any duplicate leads from the parsed results and posts each unique lead as a message to the configured Slack channel. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: **Scrape Shopify store leads and deliver new contacts to Slack via ScraperCity**.

2. **Add a Manual Trigger node**
   - Node type: **Manual Trigger**
   - Leave default settings.
   - This will be the workflow entry point.

3. **Add a Set node named `Configure Search Parameters`**
   - Connect it after the Manual Trigger.
   - Add these fields:
     - `platform` → String → `shopify`
     - `countryCode` → String → `US`
     - `totalLeads` → Number → `500`
     - `includeEmails` → Boolean → `true`
     - `includePhones` → Boolean → `true`
     - `slackChannel` → String → `#shopify-leads`
   - This node is where future users change targeting without editing API nodes.

4. **Create the ScraperCity credential**
   - In n8n Credentials, add **HTTP Header Auth**.
   - Configure it with the header expected by ScraperCity for API access.
   - The exact header name/value format depends on ScraperCity’s API documentation and your account-issued key.
   - Save the credential.

5. **Add an HTTP Request node named `Start Shopify Lead Scrape`**
   - Connect it after `Configure Search Parameters`.
   - Configure:
     - Method: `POST`
     - URL: `https://app.scrapercity.com/api/v1/scrape/store-leads`
     - Authentication: **Generic Credential Type**
     - Generic Auth Type: **HTTP Header Auth**
     - Select your ScraperCity credential
     - Send Body: enabled
     - Specify Body: **JSON**
   - Use this JSON body:
     ```json
     {
       "platform": "{{ $json.platform }}",
       "countryCode": "{{ $json.countryCode }}",
       "emails": {{ $json.includeEmails }},
       "phones": {{ $json.includePhones }},
       "totalLeads": {{ $json.totalLeads }}
     }
     ```
   - In n8n this should be entered as an expression-capable JSON body, not plain static text.

6. **Add a Set node named `Store Run ID`**
   - Connect it after `Start Shopify Lead Scrape`.
   - Add:
     - `runId` → String → `={{ $json.runId }}`
     - `slackChannel` → String → `={{ $('Configure Search Parameters').item.json.slackChannel }}`
   - This node standardizes downstream references.

7. **Add a Split In Batches node named `Poll Loop Controller`**
   - Connect it after `Store Run ID`.
   - Keep default batching behavior.
   - In options, ensure `reset` is `false`.
   - This node is used as the loop anchor.

8. **Add a Wait node named `Wait 60 Seconds`**
   - Connect it after `Poll Loop Controller`.
   - Set wait amount to **60 seconds**.

9. **Add an HTTP Request node named `Check Scrape Status`**
   - Connect it after `Wait 60 Seconds`.
   - Configure:
     - Method: `GET` (default is fine)
     - URL:
       `=https://app.scrapercity.com/api/v1/scrape/status/{{ $('Store Run ID').item.json.runId }}`
     - Authentication: **Generic Credential Type**
     - Generic Auth Type: **HTTP Header Auth**
     - Use the same ScraperCity credential.

10. **Add an IF node named `Is Scrape Complete?`**
    - Connect it after `Check Scrape Status`.
    - Set a single condition:
      - Left value: `={{ $json.status }}`
      - Operator: **equals**
      - Right value: `SUCCEEDED`
    - If your n8n version exposes condition settings, keep strict type validation enabled.

11. **Create the polling loop connection**
    - Connect the **false** output of `Is Scrape Complete?` back to `Poll Loop Controller`.
    - Connect the **true** output forward to the download stage in the next step.
    - This creates a repeating wait → check → evaluate cycle.

12. **Add an HTTP Request node named `Download Lead Results`**
    - Connect it from the **true** branch of `Is Scrape Complete?`.
    - Configure:
      - URL:
        `=https://app.scrapercity.com/api/downloads/{{ $('Store Run ID').item.json.runId }}`
      - Authentication: **Generic Credential Type**
      - Generic Auth Type: **HTTP Header Auth**
      - Use the same ScraperCity credential
      - Response format: **Text**
    - This is important because the next node expects plain CSV text, not binary.

13. **Add a Code node named `Parse CSV and Format Leads`**
    - Connect it after `Download Lead Results`.
    - Language: JavaScript.
    - Paste this logic equivalent:
      - Read the download response from `data` or `body`
      - Return an empty array if the response is empty or not a string
      - Split CSV into lines
      - Parse each line while respecting quoted fields
      - Normalize headers to lowercase with underscores
      - Map rows into:
        - `storeName`
        - `email`
        - `phone`
        - `website`
        - `instagram`
        - `facebook`
        - `twitter`
        - `country`
      - Skip rows without an email
      - Lowercase and trim the email
    - Use the following code:
     ```javascript
     // Parse CSV text into lead objects
     const csvText = $input.first().json.data || $input.first().json.body || '';

     if (!csvText || typeof csvText !== 'string') {
       return [];
     }

     const lines = csvText.trim().split('\n');
     if (lines.length < 2) return [];

     function parseLine(line) {
       const result = [];
       let current = '';
       let inQuotes = false;
       for (let i = 0; i < line.length; i++) {
         const ch = line[i];
         if (ch === '"') {
           inQuotes = !inQuotes;
         } else if (ch === ',' && !inQuotes) {
           result.push(current.trim());
           current = '';
         } else {
           current += ch;
         }
       }
       result.push(current.trim());
       return result;
     }

     const headers = parseLine(lines[0]).map(h => h.toLowerCase().replace(/\s+/g, '_'));
     const leads = [];

     for (let i = 1; i < lines.length; i++) {
       if (!lines[i].trim()) continue;
       const values = parseLine(lines[i]);
       const lead = {};
       headers.forEach((h, idx) => {
         lead[h] = values[idx] || '';
       });

       const email = lead.email || lead.email_address || lead.contact_email || '';
       if (!email) continue;

       leads.push({
         json: {
           storeName: lead.store_name || lead.name || lead.shop_name || lead.domain || 'Unknown Store',
           email: email.toLowerCase().trim(),
           phone: lead.phone || lead.phone_number || lead.contact_phone || '',
           website: lead.website || lead.domain || lead.url || '',
           instagram: lead.instagram || lead.instagram_url || '',
           facebook: lead.facebook || lead.facebook_url || '',
           twitter: lead.twitter || lead.twitter_url || '',
           country: lead.country || lead.country_code || ''
         }
       });
     }

     return leads;
     ```

14. **Add a Remove Duplicates node named `Remove Duplicate Leads`**
    - Connect it after `Parse CSV and Format Leads`.
    - Configure:
      - Compare mode: **Selected Fields**
      - Field to compare: `email`

15. **Create the Slack credential**
    - Add a Slack credential in n8n, typically OAuth2 for the Slack node.
    - Authorize it against the workspace where leads should be posted.
    - Ensure the Slack app can post to the target channel.
    - If posting to a private channel, make sure the app is added to that channel.

16. **Add a Slack node named `Post Lead to Slack`**
    - Connect it after `Remove Duplicate Leads`.
    - Configure:
      - Resource/action according to the Slack node version for posting a message
      - Destination selection mode: **Channel**
      - Channel selection mode: **By Name**
      - Channel value:
        `={{ $('Store Run ID').item.json.slackChannel }}`
      - Message type: **Block**
      - Disable link unfurling
    - Add one section block with this expression text:
      ```text
      =*New Shopify Lead: {{ $json.storeName }}*

      *Email:* {{ $json.email }}
      *Phone:* {{ $json.phone || '_not available_' }}
      *Website:* {{ $json.website || '_not available_' }}
      *Country:* {{ $json.country || '_not available_' }}
      *Instagram:* {{ $json.instagram || '_not available_' }}
      *Facebook:* {{ $json.facebook || '_not available_' }}
      *Twitter:* {{ $json.twitter || '_not available_' }}
      ```

17. **Verify final connection order**
    - Manual Trigger → Configure Search Parameters → Start Shopify Lead Scrape → Store Run ID → Poll Loop Controller → Wait 60 Seconds → Check Scrape Status → Is Scrape Complete?
    - `Is Scrape Complete?` true → Download Lead Results → Parse CSV and Format Leads → Remove Duplicate Leads → Post Lead to Slack
    - `Is Scrape Complete?` false → Poll Loop Controller

18. **Test the workflow**
    - Execute manually.
    - Confirm the initial API call returns a `runId`.
    - Wait for one or more polling cycles.
    - Confirm the final Slack messages appear in the configured channel.

19. **Recommended hardening improvements**
    - Add explicit handling for statuses such as `FAILED` or `CANCELLED`.
    - Add a maximum retry counter to avoid infinite polling.
    - Consider adding error notifications to Slack if ScraperCity fails.
    - If CSV complexity grows, replace the custom parser with a dedicated CSV parsing approach.

### Credential and integration expectations

#### ScraperCity
- Requires an API key configured through **HTTP Header Auth**.
- All three HTTP Request nodes should use the same credential:
  - `Start Shopify Lead Scrape`
  - `Check Scrape Status`
  - `Download Lead Results`

#### Slack
- Requires a valid Slack credential with permission to post messages.
- The configured `slackChannel` value must match a reachable channel name.

### Sub-workflow setup
- This workflow contains **no sub-workflow nodes**.
- There are **no additional entry points** besides the Manual Trigger.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Scrape Shopify store leads and deliver new contacts to Slack via ScraperCity | Workflow title/context |
| How it works: 1. The workflow is triggered manually and search parameters (platform, country, lead count, email inclusion) are configured. 2. A scrape job is submitted to the ScraperCity API, and the returned run ID is stored for later reference. 3. A polling loop waits 60 seconds between each status check, querying the ScraperCity API until the scrape job is marked complete. 4. Once complete, the lead results CSV is downloaded from ScraperCity and parsed into structured lead objects. 5. Duplicate leads are removed, and each unique lead is posted as a message to the configured Slack channel. | Main workflow note |
| Setup steps: Create a ScraperCity account and obtain your API key from the dashboard. Add your ScraperCity API key as an n8n credential (HTTP Header Auth) and attach it to the HTTP Request nodes. Configure the Slack node with a valid Slack OAuth credential and set the target channel name. Open `Configure Search Parameters` and set your desired platform, countryCode, totalLeads, and includeEmails values. Open `Store Run ID` and confirm the slackChannel value matches your intended Slack channel. Run the workflow manually and verify leads appear in Slack. | Main workflow note |
| Customization: Adjust the wait duration in `Wait 60 Seconds` for faster or slower polling based on typical scrape completion times. Modify `Configure Search Parameters` to target different countries or lead volumes without touching any other node. Extend `Parse CSV and Format Leads` to map additional CSV fields into the Slack message for richer notifications. | Main workflow note |