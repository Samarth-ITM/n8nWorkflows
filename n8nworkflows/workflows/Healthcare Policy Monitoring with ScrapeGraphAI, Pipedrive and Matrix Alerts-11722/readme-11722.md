Healthcare Policy Monitoring with ScrapeGraphAI, Pipedrive and Matrix Alerts

https://n8nworkflows.xyz/workflows/healthcare-policy-monitoring-with-scrapegraphai--pipedrive-and-matrix-alerts-11722


# Healthcare Policy Monitoring with ScrapeGraphAI, Pipedrive and Matrix Alerts

### 1. Workflow Overview

This workflow automates monitoring of new healthcare policy documents published by major U.S. health agencies (HHS, CMS, FDA). It runs daily to scrape policy updates, processes and deduplicates the data, then alerts the policy team and logs new policies into Pipedrive CRM for tracking.

The workflow divides logically into these blocks:

- **1.1 Trigger & URL Preparation:** Scheduled execution initiates the workflow and defines the list of agency URLs to scrape.
- **1.2 Scraping Engine:** Each URL is processed in parallel by ScrapeGraphAI to extract recent policy entries with title, date, summary, and link.
- **1.3 Data Processing & Change Detection:** Normalizes data formats, filters for new policies published within 24 hours, merges results, and removes duplicates.
- **1.4 Storage & Alerts:** Sends new policy alerts via Matrix messaging and creates or updates deals in Pipedrive to track policy items.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & URL Preparation

**Overview:**  
This block starts the workflow on a daily schedule and generates a configurable list of healthcare policy source URLs for scraping.

**Nodes Involved:**  
- Daily Policy Check (Schedule Trigger)  
- Define Policy URLs (Code)

**Node Details:**

- **Daily Policy Check**  
  - Type: Schedule Trigger  
  - Role: Initiates workflow every 24 hours (default)  
  - Configuration: Interval set to 24 hours; adjustable cron or interval rules possible  
  - Connections: Outputs to Define Policy URLs  
  - Edge Cases: Workflow won't trigger if n8n is down; check time zone implications for scheduling  

- **Define Policy URLs**  
  - Type: Code  
  - Role: Returns an array of JSON objects defining URLs and source labels for scraping  
  - Configuration: Hardcoded list includes three URLs (HHS, CMS, FDA) with source names; editable to add/remove sources  
  - Key Expression: Returns array mapped as `{ json: { url, source } }`  
  - Input: Trigger node output  
  - Output: Array of URL objects to Split URLs  
  - Edge Cases: URL changes or deprecation require manual code update  

---

#### 1.2 Scraping Engine

**Overview:**  
Splits the URL array into individual items and runs AI-powered scraping on each page in parallel, extracting structured policy entries.

**Nodes Involved:**  
- Split URLs (SplitInBatches)  
- Fetch Policy Data (ScrapeGraphAI)

**Node Details:**

- **Split URLs**  
  - Type: SplitInBatches  
  - Role: Splits input array into individual items for parallel processing  
  - Configuration: Default batch settings (1 item per batch)  
  - Input: Output of Define Policy URLs  
  - Output: Single JSON URL objects to Fetch Policy Data  
  - Edge Cases: Large input arrays may require batch size tuning to avoid rate limits or overload  

- **Fetch Policy Data**  
  - Type: ScrapeGraphAI  
  - Role: Uses LLM prompt to extract recent policy entries from each webpage  
  - Configuration:  
    - Prompt instructs extraction of title, date, summary, link, and carries forward source label  
    - Filters for entries published within last 7 days if date info is available  
    - Uses input URL dynamically via expression `={{ $json.url }}`  
  - Input: Single URL JSON item  
  - Output: JSON array of policy entries for normalization  
  - Edge Cases:  
    - Website layout or content changes could reduce extraction accuracy, though AI approach is resilient  
    - API or credential errors with ScrapeGraphAI  
    - Network timeouts or rate limits  

---

#### 1.3 Data Processing & Change Detection

**Overview:**  
Normalizes and standardizes scraped data fields, flags recently published policies, merges parallel results, deduplicates by title, and routes only new items forward.

**Nodes Involved:**  
- Normalize Data (Code)  
- Merge Results (Merge)  
- Deduplicate Policies (Code)  
- Is New Policy? (IF)

**Node Details:**

- **Normalize Data**  
  - Type: Code  
  - Role: Harmonizes field names (`title`, `date`, `summary`, `link`, `source`), converts dates to ISO-8601, adds boolean `isNew` if published within 24 hours  
  - Key Logic: Compares publication date timestamp to current time minus 24 hours  
  - Input: Array of policy entries from ScrapeGraphAI  
  - Output: Standardized JSON objects to Merge Results  
  - Edge Cases: Missing or malformed date fields default to current time; may cause false positives  

- **Merge Results**  
  - Type: Merge  
  - Role: Combines outputs of all parallel scraping branches into a single array  
  - Configuration: Mode set to "combine" (append inputs)  
  - Input: Multiple outputs from Normalize Data nodes (due to parallelism)  
  - Output: Unified list of all policy entries to Deduplicate Policies  
  - Edge Cases: If any branch fails or delays, may hold up merging  

- **Deduplicate Policies**  
  - Type: Code  
  - Role: Removes duplicate policies by case-insensitive title matching  
  - Logic: Uses a Set to track seen lowercase titles, filters duplicates out  
  - Input: Merged policy list  
  - Output: Deduplicated list to IF node  
  - Edge Cases: Near-duplicate titles with slight variations may not be caught  

- **Is New Policy?**  
  - Type: IF  
  - Role: Routes only policies flagged as `isNew=true` down true path for alerting and storage  
  - Condition: Boolean check on `isNew` field  
  - Input: Deduplicated policy list  
  - Output: True → Prepare Matrix Message & Upsert Policy Deal; False → discarded silently  
  - Edge Cases: Policies with incorrect or missing `isNew` flag may be misrouted  

---

#### 1.4 Storage & Alerts

**Overview:**  
Formats and sends real-time alerts to a Matrix chat room and creates or updates policy deals in Pipedrive CRM.

**Nodes Involved:**  
- Prepare Matrix Message (Set)  
- Send Matrix Alert (Matrix)  
- Upsert Policy Deal (Pipedrive)

**Node Details:**

- **Prepare Matrix Message**  
  - Type: Set  
  - Role: Constructs a concise alert message containing policy title, source agency, publication date, and link  
  - Input: Items routed from IF node (new policies only)  
  - Output: Structured message JSON to Send Matrix Alert  
  - Edge Cases: Missing fields may produce incomplete messages  

- **Send Matrix Alert**  
  - Type: Matrix  
  - Role: Posts prepared message to configured Matrix room for policy team notification  
  - Configuration: Requires Matrix credentials and target room ID  
  - Input: Prepared message from Set node  
  - Output: Confirmation of message sent  
  - Edge Cases: Authentication failures, network errors, or invalid room ID cause alert failures  

- **Upsert Policy Deal**  
  - Type: Pipedrive  
  - Role: Creates or updates a deal in Pipedrive CRM representing the new policy item  
  - Configuration:  
    - Sets deal title from policy title  
    - Labels deal as “Policy”  
    - Sets status to “open”  
    - Visibility set for all users (visible_to = 3)  
  - Input: Same new policy items as Matrix alert  
  - Output: Deal creation/update confirmation  
  - Edge Cases: API authentication errors, missing required fields, or quota limits may cause failures  

---

### 3. Summary Table

| Node Name           | Node Type               | Functional Role                     | Input Node(s)        | Output Node(s)              | Sticky Note                                                                                                                      |
|---------------------|-------------------------|-----------------------------------|----------------------|----------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| Workflow Overview   | Sticky Note             | Describes high-level workflow     |                      |                            | ## How it works and setup steps explanation                                                                                     |
| Section – Trigger & URL Prep | Sticky Note       | Explains initial trigger and URL setup |                      |                            | ## Trigger & URL Preparation details                                                                                           |
| Section – Scraping  | Sticky Note             | Explains scraping engine details  |                      |                            | ## Scraping Engine explanation                                                                                                 |
| Section – Processing & Detection | Sticky Note     | Explains data normalization and detection |                      |                            | ## Data Processing & Change Detection overview                                                                                 |
| Section – Storage & Alerts | Sticky Note         | Explains storage and notification |                      |                            | ## Storage & Notification details                                                                                              |
| Daily Policy Check  | Schedule Trigger        | Starts workflow every 24 hours    |                      | Define Policy URLs          | See Section – Trigger & URL Prep                                                                                               |
| Define Policy URLs  | Code                    | Defines list of policy URLs        | Daily Policy Check    | Split URLs                 | See Section – Trigger & URL Prep                                                                                               |
| Split URLs          | SplitInBatches          | Splits URL array for parallel scrape | Define Policy URLs    | Fetch Policy Data, Merge Results | See Section – Scraping                                                                                                         |
| Fetch Policy Data   | ScrapeGraphAI           | Extracts policy data via AI prompt | Split URLs           | Normalize Data             | See Section – Scraping                                                                                                         |
| Normalize Data      | Code                    | Normalizes fields, flags recent items | Fetch Policy Data     | Merge Results              | See Section – Processing & Detection                                                                                           |
| Merge Results       | Merge                   | Combines parallel scrape outputs  | Normalize Data, Split URLs (second output) | Deduplicate Policies    | See Section – Processing & Detection                                                                                           |
| Deduplicate Policies | Code                    | Removes duplicate policies by title | Merge Results        | Is New Policy?             | See Section – Processing & Detection                                                                                           |
| Is New Policy?      | IF                      | Routes only new policies forward  | Deduplicate Policies  | Prepare Matrix Message, Upsert Policy Deal | See Section – Processing & Detection                                                                                           |
| Prepare Matrix Message | Set                    | Formats alert message              | Is New Policy? (true) | Send Matrix Alert          | See Section – Storage & Alerts                                                                                                |
| Send Matrix Alert   | Matrix                  | Sends alert to Matrix room         | Prepare Matrix Message |                            | See Section – Storage & Alerts                                                                                                |
| Upsert Policy Deal  | Pipedrive               | Creates/updates deal in CRM        | Is New Policy? (true) |                            | See Section – Storage & Alerts                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**  
   - Name: `Daily Policy Check`  
   - Set interval to every 24 hours (or desired interval via cron/expression).  
   - Connect output to next node.

3. **Add a Code node:**  
   - Name: `Define Policy URLs`  
   - Paste the following JavaScript code:
     ```javascript
     const urls = [
       { url: 'https://www.hhs.gov/about/news/index.html', source: 'HHS' },
       { url: 'https://www.cms.gov/newsroom', source: 'CMS' },
       { url: 'https://www.fda.gov/news-events/fda-voices', source: 'FDA' }
     ];
     return urls.map(obj => ({ json: obj }));
     ```
   - Connect input from `Daily Policy Check`.

4. **Add a SplitInBatches node:**  
   - Name: `Split URLs`  
   - Default batch size 1 (process one URL at a time).  
   - Connect input from `Define Policy URLs`.

5. **Add ScrapeGraphAI node:**  
   - Name: `Fetch Policy Data`  
   - Credential: Add and select your ScrapeGraphAI credentials.  
   - Configure parameters:  
     - `websiteUrl` set to expression `={{ $json.url }}`  
     - `userPrompt` set as:  
       `"Extract the most recent healthcare policy or regulation entries from this page. For each entry return a JSON object with: title, date, summary, link, and keep the provided source value from input. Only include items published in the last 7 days if the information is available."`  
   - Connect input from `Split URLs`.

6. **Add a Code node:**  
   - Name: `Normalize Data`  
   - Code:
     ```javascript
     const ONE_DAY = 24 * 60 * 60 * 1000;
     const items = $input.all();
     return items.map(item => {
       const d = item.json;
       const obj = {
         title: d.title || d.name || 'Untitled Policy',
         date: d.date || new Date().toISOString(),
         summary: d.summary || d.description || '',
         link: d.link || d.url || '',
         source: d.source || item.json.source || 'unknown'
       };
       obj.isNew = (new Date(obj.date)).getTime() > Date.now() - ONE_DAY;
       return { json: obj };
     });
     ```
   - Connect input from `Fetch Policy Data`.

7. **Add a Merge node:**  
   - Name: `Merge Results`  
   - Mode: Combine (append inputs)  
   - Connect two inputs:  
     - Main input from `Normalize Data`  
     - Secondary input from `Split URLs` (second output, to ensure synchronization)  

8. **Add a Code node:**  
   - Name: `Deduplicate Policies`  
   - Code:
     ```javascript
     const seen = new Set();
     const out = [];
     for (const item of $input.all()) {
       const key = (item.json.title || '').toLowerCase();
       if (!seen.has(key)) {
         seen.add(key);
         out.push(item);
       }
     }
     return out;
     ```
   - Connect input from `Merge Results`.

9. **Add an IF node:**  
   - Name: `Is New Policy?`  
   - Condition: Boolean check where `{{$json.isNew}} == true`  
   - Connect input from `Deduplicate Policies`.

10. **Add a Set node:**  
    - Name: `Prepare Matrix Message`  
    - Configure to build a message with fields: title, source, date, link from input JSON.  
    - Example fields to set (adjust as needed):
      - `message` = `"New policy: {{$json.title}} from {{$json.source}} published on {{$json.date}}. Link: {{$json.link}}"`  
    - Connect true output of `Is New Policy?` to this node.

11. **Add a Matrix node:**  
    - Name: `Send Matrix Alert`  
    - Operation: Send message  
    - Credentials: Add and select your Matrix account credentials and specify the room ID.  
    - Connect input from `Prepare Matrix Message`.

12. **Add a Pipedrive node:**  
    - Name: `Upsert Policy Deal`  
    - Operation: Create or update a deal  
    - Credentials: Add Pipedrive API credentials  
    - Parameters:  
      - Title: Expression `{{$json.title}}`  
      - Additional fields:  
        - Label: "Policy"  
        - Status: "open"  
        - Visibility: "3" (visible to all users)  
    - Connect true output of `Is New Policy?` parallel to `Prepare Matrix Message` node.

13. **Activate the workflow and test manually.**

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                                                           |
|------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| This workflow uses ScrapeGraphAI for AI-driven scraping, which is more resilient to website layout changes than CSS selectors. | https://scrapegraph.ai/                                                                                   |
| Matrix messaging provides real-time alerts to policy teams without needing dashboard checks.          | https://matrix.org/                                                                                        |
| Pipedrive is used to treat policies as actionable CRM deals, allowing task assignment and tracking.   | https://pipedrive.com/                                                                                     |
| Setup requires adding credentials for ScrapeGraphAI, Matrix, and Pipedrive before activation.         | See Workflow Overview sticky note for setup steps                                                        |
| Customization tips: add/remove URLs in `Define Policy URLs` Code node; adjust schedule in `Daily Policy Check`. |                                                                                                          |

---

This structured reference enables understanding, modification, and reliable reproduction of the Healthcare Policy Monitoring workflow. It includes all nodes, key logic, connections, configurations, and notes on possible edge cases or failure points.