Check and deactivate broken job apply URLs with Postgres and Google Sheets

https://n8nworkflows.xyz/workflows/check-and-deactivate-broken-job-apply-urls-with-postgres-and-google-sheets-14995


# Check and deactivate broken job apply URLs with Postgres and Google Sheets

The user has provided a workflow JSON and metadata (title, description, etc.) and wants a comprehensive, structured, self-contained reference document in English, following a specific format. The document must have sections:

1. Workflow Overview
2. Block-by-Block Analysis
3. Summary Table (nodes)
4. Reproducing the Workflow from Scratch (step-by-step numbered list)
5. General Notes & Resources

We must analyze each node, connections, logic, expressions, edge cases. Provide details on each node. Then summary table: Node Name, Node Type, Functional Role, Input Node(s), Output Node(s), Sticky Note. Use sticky note content where relevant. Must not include raw JSON except for examples if needed.

We must not skip any nodes. Must group logically.

The workflow includes the following nodes (from JSON):

1. Main Sticky (stickyNote)
2. Every 3 Days (scheduleTrigger)
3. Fetch Active Jobs (postgres)
4. Prepare URLs (code)
5. Check URLs (httpRequest)
6. Find Dead Jobs (code)
7. Has Dead Jobs? (if)
8. Mark Inactive (Postgres) (postgres)
9. Mark Inactive (Google Sheet) (googleSheets)
10. Section 1 (stickyNote)
11. Section 2 (stickyNote)
12. Section 3 (stickyNote)

Connections:
- Every 3 Days -> Fetch Active Jobs
- Fetch Active Jobs -> Prepare URLs
- Prepare URLs -> Check URLs
- Check URLs -> Find Dead Jobs
- Find Dead Jobs -> Has Dead Jobs?
- Has Dead Jobs? -> Mark Inactive (Postgres) (True branch)
- Mark Inactive (Postgres) -> Mark Inactive (Google Sheet)
- The false branch of Has Dead Jobs? (no dead jobs) presumably ends (no connection, so it just stops)

We need to analyze each node.

**Node details:**

**Main Sticky**: stickyNote with color 2, width 500, height 600, content includes description of the workflow.

**Section 1**, **Section 2**, **Section 3**: sticky notes for sections.

**Every 3 Days**: scheduleTrigger, interval 72 hours (3 days). Should be typeVersion 1.1.

**Fetch Active Jobs**: Postgres node, operation executeQuery, query: SELECT job_hash, apply_url, company, job_title FROM jobs WHERE status = 'active' ORDER BY created_at ASC. Type version 2.5. Needs Postgres credentials configured.

**Prepare URLs**: Code node, JavaScript filtering of jobs. Takes all input items ($input.all()), filters out rows with missing, malformed, or non-http URLs. It logs skipped items. Returns an array of valid items. If no valid jobs, returns [{ json: { _no_jobs: true } }]. Type version 2.

**Check URLs**: HTTP Request node, method HEAD, URL expression: ={{ $json.apply_url }}, options: timeout 5000 ms, batch size 5, maxRedirects 5, neverError true, fullResponse true. On error continue regular output. Type version 4.2.

**Find Dead Jobs**: Code node, analyzing HTTP responses. It references cfgItems from Prepare URLs node ($('Prepare URLs')). It checks status codes, error messages for DNS failure, connection refused, 404, 410, and soft-404 redirect detection. If dead jobs found, returns array; else returns single item with _no_dead_jobs: true and counts. Type version 2.

**Has Dead Jobs?**: IF node, checks condition: $json._no_dead_jobs not equals true. So true branch when there are dead jobs; false branch when _no_dead_jobs is true (i.e., no dead jobs). The conditions settings: leftValue = {{ $json._no_dead_jobs }}, operation boolean notEquals rightValue true. Type version 2.

**Mark Inactive (Postgres)**: Postgres node, operation executeQuery, query: UPDATE jobs SET status = 'inactive' WHERE job_hash = '{{ $json.job_hash }}'; Note: uses expression interpolation. Should be parameterized; note injection risk. Type version 2.5. On error continue regular output.

**Mark Inactive (Google Sheet)**: Google Sheets node, operation appendOrUpdate, sheetName and documentId placeholder YOUR_RESOURCE_ID_HERE, matchingColumns: ["job_hash"], columns mapping: job_hash expression {{ $json.job_hash }}, success column? Actually the columns spec shows columns mapping with job_hash plus many removed columns (like Job Title, Company, etc.) and a column "success" which is not removed. mappingMode: autoMapInputData, matchingColumns: ["job_hash"]. This will update rows where job_hash matches, updating the "success" column? But "success" column is there, but no mapping value provided, perhaps will be auto-mapped. Actually columns.value includes job_hash mapping; success column has no explicit mapping but appears. It might be that the node will update the row's "success" column with default empty. We'll document that.

Edge cases: In Google Sheets node, missing Document ID and Sheet Name need to be filled. Also, matchingColumns must be configured correctly.

**General notes**:

- The workflow is designed to be run automatically every 3 days.
- Ensure that Postgres credentials are set.
- Ensure that Google Sheets credentials are configured and sheet ID is provided.
- The soft-404 detection logic can be customized.
- The workflow uses an IF node to branch only when dead jobs are found.
- There is a risk of SQL injection due to using expression interpolation in the Postgres update query; advisable to use parameterized queries.
- The HTTP Request node's "continue on error" and "neverError" settings ensure that error responses are captured for analysis, rather than causing the node to fail.
- The Check URLs node uses batch size of 5 and max redirects 5; these can be adjusted.

**Connections:**

- The flow: Every 3 Days -> Fetch Active Jobs -> Prepare URLs -> Check URLs -> Find Dead Jobs -> Has Dead Jobs? -> Mark Inactive (Postgres) -> Mark Inactive (Google Sheet). The false branch of Has Dead Jobs? has no connection; the workflow ends.

**Sticky notes**:

- Main Sticky: Contains description. Should be included in summary table under each node? Only nodes that have sticky notes covering them. The sticky notes: Section 1 covers "Every 3 Days" and "Fetch Active Jobs". Section 2 covers "Prepare URLs", "Check URLs", "Find Dead Jobs". Section 3 covers "Has Dead Jobs?" and "Mark Inactive (Postgres)" and "Mark Inactive (Google Sheet)". The Main Sticky covers the entire workflow.

Thus for each node, we need to assign sticky note content that covers it. So:

- Every 3 Days: Section 1 sticky note.
- Fetch Active Jobs: Section 1 sticky note.
- Prepare URLs: Section 2 sticky note.
- Check URLs: Section 2 sticky note.
- Find Dead Jobs: Section 2 sticky note.
- Has Dead Jobs?: Section 3 sticky note.
- Mark Inactive (Postgres): Section 3 sticky note.
- Mark Inactive (Google Sheet): Section 3 sticky note.
- Main Sticky covers all nodes? Could include it as general note.

We'll place the appropriate sticky content in each node row.

Now write the document.

**Section 1: Workflow Overview**: Provide concise high-level explanation: automated job link validation, runs every 3 days, fetches active jobs from Postgres, filters URLs, HEAD checks, identifies dead links, updates status in Postgres and Google Sheets. Include logical blocks: 1. Trigger & Extraction, 2. Validation & Processing, 3. Reporting & Sync.

**Section 2: Block-by-Block Analysis**: For each block, list nodes involved and details.

- Block 1: Trigger and Extraction. Nodes: Every 3 Days, Fetch Active Jobs.
- Block 2: Validation and Processing. Nodes: Prepare URLs, Check URLs, Find Dead Jobs.
- Block 3: Reporting and Sync. Nodes: Has Dead Jobs?, Mark Inactive (Postgres), Mark Inactive (Google Sheet).

Add details for each node.

**Edge Cases**:
- In Prepare URLs, if no valid jobs, output _no_jobs sentinel, causing Find Dead Jobs to output _no_dead_jobs sentinel and workflow ends.
- In Check URLs, if HEAD request times out, it captures error info.
- In Find Dead Jobs, uses soft-404 detection; customizing detection logic.

**Potential Failure Types**:
- DB connection issues.
- Google Sheets authentication errors.
- HTTP request timeouts.
- SQL injection risk.
- Rate limiting if too many URLs.

**Sub-workflows**: None.

Now produce the summary table.

List nodes with Node Name, Node Type, Functional Role, Input Node(s), Output Node(s), Sticky Note.

Will include sticky note content: For nodes covered by Section 1, the note content: "## 1. Trigger and Extraction\nSchedule the automated run and retrieve active job records from your PostgreSQL database."

For Section 2: "## 2. Validation and Processing\nFilter invalid URLs, perform connectivity checks, and identify dead links using custom JavaScript logic."

For Section 3: "## 3. Reporting and Sync\nUpdate the status of verified dead jobs in both the source database and the reporting spreadsheet."

The Main Sticky note content: includes description and setup instructions. Possibly mention for each node that the main sticky applies.

Will add a row for each node.

**Reproducing the Workflow**: Step-by-step numbered list.

Will outline:
1. Add schedule trigger node.
2. Add Postgres node.
3. Add Code node for Prepare URLs.
4. Add HTTP Request node for Check URLs.
5. Add Code node for Find Dead Jobs.
6. Add IF node.
7. Add Postgres node for Mark Inactive (Postgres).
8. Add Google Sheets node for Mark Inactive.
9. Connect nodes in order.
10. Configure credentials.
11. Set parameters (query, expressions, options).
12. Adjust settings for error handling.

Will include specific parameter values.

**General Notes & Resources**: Could note documentation link, note about adjusting soft-404 logic, branding.

We'll prepare final answer.

Need to ensure not to include raw JSON except minimal examples, no "tutorial" term.

Will produce final document accordingly.## 1. Workflow Overview  

**Purpose** – This workflow automatically keeps a job‑listing database clean by scanning every active job’s application URL, detecting broken or silently‑redirected (“soft‑404”) links, and deactivating those records in both a PostgreSQL (Supabase) database and a Google Sheets log. It runs unattended on a 3‑day cadence, eliminating the need for manual URL checks.

**Logical blocks**

| Block | Functional role | Nodes |
|------|----------------|-------|
| **1.1 Trigger & Extraction** | Schedule the scan and pull all active jobs from Postgres. | `Every 3 Days` → `Fetch Active Jobs` |
| **1.2 Validation & Processing** | Filter out malformed URLs, test each URL with a HEAD request, and classify dead jobs (hard 404/410, DNS failure, connection refused, soft‑404 redirects). | `Prepare URLs` → `Check URLs` → `Find Dead Jobs` |
| **1.3 Reporting & Sync** | Decide whether any dead jobs exist and, if so, mark them `inactive` in Postgres and update the corresponding Google Sheets rows. | `Has Dead Jobs?` → `Mark Inactive (Postgres)` → `Mark Inactive (Google Sheet)` |

If no dead jobs are found, the workflow ends cleanly without any writes.

---

## 2. Block‑by‑Block Analysis  

### Block 1 – Trigger & Extraction  

#### Overview  
A schedule trigger starts the workflow every 72 hours. The Postgres node then runs a SQL query that returns all active job records required for URL checking.

| Node | Type | Technical role | Configuration | Key expressions / variables | Input | Output | Edge cases / failure types |
|------|------|----------------|--------------|----------------------------|-------|--------|-----------------------------|
| **Every 3 Days** | `scheduleTrigger` | Initiates the workflow on a fixed interval. | Interval = every 72 hours (`hoursInterval: 72`). | — | — | `Fetch Active Jobs` | No payload; failures only if n8n scheduler is disabled. |
| **Fetch Active Jobs** | `postgres` (v2.5) | Retrieves active job rows from the `jobs` table. | Operation: **Execute Query**<br>Query: `SELECT job_hash, apply_url, company, job_title FROM jobs WHERE status = 'active' ORDER BY created_at ASC;`<br>Options: default. | — | `Every 3 Days` | `Prepare URLs` | • Invalid Postgres credentials.<br>• Network/timeout errors.<br>• Empty result set (still passes an empty array). |

#### Sticky notes covering this block  
- **Section 1 Sticky Note**: “## 1. Trigger and Extraction – Schedule the automated run and retrieve active job records from your PostgreSQL database.”

---

### Block 2 – Validation & Processing  

#### Overview  
This block sanitises the URL list, performs a lightweight HEAD request for each URL, and then analyses the responses to decide whether a job link is dead. Soft‑404 detection looks for redirects that lead to a different path (e.g., `/jobs` or `/careers`).

| Node | Type | Technical role | Configuration | Key expressions / variables | Input | Output | Edge cases / failure types |
|------|------|----------------|--------------|----------------------------|-------|--------|-----------------------------|
| **Prepare URLs** | `code` (v2) | Filters out rows with empty, too‑short, or non‑HTTP URLs; creates a clean list for checking. | JS code iterates `$input.all()`, trims `apply_url`, discards entries where URL is missing, `<10` chars, or does not start with `http`. Logs skip count. If no valid jobs remain, returns `[{ json: { _no_jobs: true } }]`. | `$json.apply_url`, `$json.job_title` | `Fetch Active Jobs` | `Check URLs` (valid items) or sentinel `_no_jobs` | • Unexpected null values in `apply_url`.<br>• All jobs filtered out → sentinel triggers early exit later. |
| **Check URLs** | `httpRequest` (v4.2) | Sends a HEAD request to each `apply_url` and captures full response information, including redirects and errors. | Method: **HEAD**<br>URL: `={{ $json.apply_url }}`<br>Options: timeout = 5000 ms, maxRedirects = 5, batchSize = 5, neverError = true, fullResponse = true<br>Error handling: **continueRegularOutput**. | `$json.apply_url` | `Prepare URLs` | `Find Dead Jobs` | • Timeouts or network failures are captured in the output object (`error` field).<br>• If the host is unreachable, `error` will contain `ENOTFOUND`/`ECONNREFUSED`. |
| **Find Dead Jobs** | `code` (v2) | Analyses each HTTP response, classifies a job as dead when: <br>• HTTP status 404/410.<br>• DNS failure (`ENOTFOUND`).<br>• Connection refused (`ECONNREFUSED`).<br>• Redirect (301/302/307) to a path that does **not** contain the original path and ends in `/jobs`, `/careers`, or `/` (soft‑404).<br>Returns a list of dead jobs, or a sentinel `_no_dead_jobs` when none are found. | References `$input.all()` (HTTP results) and `$('Prepare URLs').all()` (original job data). Uses `new URL()` for path parsing. | `$json.statusCode`, `$json.headers.location`, `$json.error`, `$json.job_hash`, `$json.apply_url` | `Check URLs` | `Has Dead Jobs?` (dead jobs list or `_no_dead_jobs` object) | • Redirect header may be relative; resolved with `new URL(location, originalUrl)`.<br>• Potential injection if `apply_url` contains malformed data – still safe because only URL parsing is used.<br>• If both `_no_jobs` sentinel is present, early exit returns `_no_dead_jobs`. |

#### Sticky notes covering this block  
- **Section 2 Sticky Note**: “## 2. Validation and Processing – Filter invalid URLs, perform connectivity checks, and identify dead links using custom JavaScript logic.”

---

### Block 3 – Reporting & Sync  

#### Overview  
An IF node checks whether any dead jobs were detected. If yes, the workflow updates each dead job’s status to `inactive` in Postgres and then mirrors the change in the linked Google Sheet. When no dead jobs are found, the workflow terminates without writing anything.

| Node | Type | Technical role | Configuration | Key expressions / variables | Input | Output | Edge cases / failure types |
|------|------|----------------|--------------|----------------------------|-------|--------|-----------------------------|
| **Has Dead Jobs?** | `if` (v2) | Branches based on the presence of dead jobs. | Condition: `$json._no_dead_jobs` **not equal to** `true`. Left‑value: `={{ $json._no_dead_jobs }}`, operation: `notEquals`, right‑value: `true`. | — | `Find Dead Jobs` | True branch → `Mark Inactive (Postgres)`<br>False branch → (no connection, workflow ends) | • If `_no_dead_jobs` is `undefined` (e.g., the node receives an array of dead jobs), condition evaluates to **true** and proceeds. |
| **Mark Inactive (Postgres)** | `postgres` (v2.5) | Executes an UPDATE statement that sets `status = 'inactive'` for the matching `job_hash`. | Operation: **Execute Query**<br>Query: `UPDATE jobs SET status = 'inactive' WHERE job_hash = '{{ $json.job_hash }}';`<br>Error handling: **continueRegularOutput**. | `$json.job_hash` | `Has Dead Jobs?` (True) | `Mark Inactive (Google Sheet)` | • Uses string interpolation – susceptible to SQL injection if `job_hash` contains malicious characters (highly unlikely for a hash).<br>• Database connectivity or permission errors.<br>• Each dead job is processed sequentially (one UPDATE per item). |
| **Mark Inactive (Google Sheet)** | `googleSheets` (v4) | Updates the Google Sheet row whose `job_hash` matches the dead job, setting the `success` column (auto‑mapped) and preserving other columns. | Operation: **appendOrUpdate**<br>Matching columns: `["job_hash"]`<br>Column mapping: `job_hash = {{ $json.job_hash }}`<br>Document & sheet IDs: placeholders (`YOUR_RESOURCE_ID_HERE`) – must be replaced with actual IDs.<br>Options: default. | `$json.job_hash` | `Mark Inactive (Postgres)` | — | • Missing or invalid Google Sheets credentials.<br>• Incorrect Document ID or Sheet Name → node fails.<br>• If the sheet lacks a column named `success`, the update will create it (or you may need to map the correct column). |

#### Sticky notes covering this block  
- **Section 3 Sticky Note**: “## 3. Reporting and Sync – Update the status of verified dead jobs in both the source database and the reporting spreadsheet.”

---

## 3. Summary Table  

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|-----------|-----------|----------------|---------------|----------------|-------------|
| **Every 3 Days** | `scheduleTrigger` | Starts the workflow every 72 hours. | — | `Fetch Active Jobs` | ## 1. Trigger and Extraction – Schedule the automated run and retrieve active job records from your PostgreSQL database. |
| **Fetch Active Jobs** | `postgres` (v2.5) | Executes SQL to get all active jobs. | `Every 3 Days` | `Prepare URLs` | ## 1. Trigger and Extraction – Schedule the automated run and retrieve active job records from your PostgreSQL database. |
| **Prepare URLs** | `code` (v2) | Filters out rows with missing or invalid URLs. | `Fetch Active Jobs` | `Check URLs` | ## 2. Validation and Processing – Filter invalid URLs, perform connectivity checks, and identify dead links using custom JavaScript logic. |
| **Check URLs** | `httpRequest` (v4.2) | Sends HEAD requests to each URL, captures response & errors. | `Prepare URLs` | `Find Dead Jobs` | ## 2. Validation and Processing – Filter invalid URLs, perform connectivity checks, and identify dead links using custom JavaScript logic. |
| **Find Dead Jobs** | `code` (v2) | Analyses responses, identifies dead jobs (hard failures & soft‑404). | `Check URLs` | `Has Dead Jobs?` | ## 2. Validation and Processing – Filter invalid URLs, perform connectivity checks, and identify dead links using custom JavaScript logic. |
| **Has Dead Jobs?** | `if` (v2) | Routes execution to update nodes only when dead jobs exist. | `Find Dead Jobs` | True → `Mark Inactive (Postgres)`<br>False → (none) | ## 3. Reporting and Sync – Update the status of verified dead jobs in both the source database and the reporting spreadsheet. |
| **Mark Inactive (Postgres)** | `postgres` (v2.5) | Sets `status = 'inactive'` for each dead job. | `Has Dead Jobs?` (True) | `Mark Inactive (Google Sheet)` | ## 3. Reporting and Sync – Update the status of verified dead jobs in both the source database and the reporting spreadsheet. |
| **Mark Inactive (Google Sheet)** | `googleSheets` (v4) | Mirrors the status change in a Google Sheet row. | `Mark Inactive (Postgres)` | — | ## 3. Reporting and Sync – Update the status of verified dead jobs in both the source database and the reporting spreadsheet. |
| **Main Sticky** | `stickyNote` | High‑level description & setup tips for the entire workflow. | — | — | ## Automate Job Link Validation and Deactivation – Maintain data hygiene by automatically scanning job application URLs and disabling broken links. Setup: configure Postgres credentials, authenticate Google Sheets, update Resource ID & Sheet Name, adjust validation logic in Find Dead Jobs. |
| **Section 1** | `stickyNote` | Describes Block 1 (Trigger & Extraction). | — | — | ## 1. Trigger and Extraction – Schedule the automated run and retrieve active job records from your PostgreSQL database. |
| **Section 2** | `stickyNote` | Describes Block 2 (Validation & Processing). | — | — | ## 2. Validation and Processing – Filter invalid URLs, perform connectivity checks, and identify dead links using custom JavaScript logic. |
| **Section 3** | `stickyNote` | Describes Block 3 (Reporting & Sync). | — | — | ## 3. Reporting and Sync – Update the status of verified dead jobs in both the source database and the reporting spreadsheet. |

---

## 4. Reproducing the Workflow from Scratch  

Follow these steps to recreate the workflow in a fresh n8n instance. All parameter values are provided; placeholders must be replaced with your own IDs/credentials.

1. **Create a new workflow** and name it *Check and deactivate broken job apply URLs with Postgres and Google Sheets*.

2. **Add Schedule Trigger**  
   - Node type: `scheduleTrigger` (v1.1)  
   - Label: `Every 3 Days`  
   - Rule: interval → field `hours` → `hoursInterval: 72`  

3. **Add Postgres Node – Fetch Active Jobs**  
   - Node type: `postgres` (v2.5)  
   - Label: `Fetch Active Jobs`  
   - Operation: **Execute Query**  
   - Query:  
     ```sql
     SELECT job_hash, apply_url, company, job_title
     FROM jobs
     WHERE status = 'active'
     ORDER BY created_at ASC;
     ```  
   - Connect the output of `Every 3 Days` → input of `Fetch Active Jobs`.  
   - Configure Postgres credentials (host, database, user, password).

4. **Add Code Node – Prepare URLs**  
   - Node type: `code` (v2)  
   - Label: `Prepare URLs`  
   - Language: JavaScript  
   - Paste the following code (unchanged):  
     ```javascript
     // Filter out jobs with empty or invalid URLs
     const jobs = $input.all();
     const valid = [];
     const skipped = [];

     for (const item of jobs) {
       const url = (item.json.apply_url || '').trim();
       if (!url || url.length < 10 || !url.startsWith('http')) {
         skipped.push(item.json.job_title || 'unknown');
         continue;
       }
       valid.push({ json: { ...item.json } });
     }

     if (skipped.length > 0) {
       console.log('Skipped ' + skipped.length + ' jobs with invalid URLs');
     }
     console.log('Checking ' + valid.length + ' job URLs');

     if (valid.length === 0) {
       return [{ json: { _no_jobs: true } }];
     }
     return valid;
     ```  
   - Connect `Fetch Active Jobs` → `Prepare URLs`.

5. **Add HTTP Request Node – Check URLs**  
   - Node type: `httpRequest` (v4.2)  
   - Label: `Check URLs`  
   - Method: **HEAD**  
   - URL: `={{ $json.apply_url }}`  
   - Options:  
     - Timeout: `5000` ms  
     - Batching → Batch size: `5`  
     - Redirect → Max Redirects: `5`  
     - Response → Never error: `true`  
     - Response → Full response: `true`  
   - Error handling: **Continue** (regular output).  
   - Connect `Prepare URLs` → `Check URLs`.

6. **Add Code Node – Find Dead Jobs**  
   - Node type: `code` (v2)  
   - Label: `Find Dead Jobs`  
   - Language: JavaScript  
   - Paste the following code (unchanged):  
     ```javascript
     const httpItems = $input.all();
     const cfgItems = $('Prepare URLs').all();

     if (cfgItems.length === 1 && cfgItems[0].json._no_jobs) {
       return [{ json: { _no_dead_jobs: true } }];
     }

     const deadJobs = [];
     let aliveCount = 0, errorCount = 0;

     for (let i = 0; i < httpItems.length; i++) {
       const http = httpItems[i].json;
       const job = cfgItems[i] ? cfgItems[i].json : {};
       if (!job.job_hash) continue;

       const statusCode = http.statusCode || http.status || null;
       const errMsg = JSON.stringify(http.error || http.message || '').toLowerCase();

       let isDead = false;
       let reason = '';

       if (errMsg.includes('enotfound') || errMsg.includes('getaddrinfo')) {
         isDead = true; reason = 'DNS_FAIL';
       } else if (errMsg.includes('econnrefused')) {
         isDead = true; reason = 'CONN_REFUSED';
       } else if (statusCode === 404 || statusCode === 410) {
         isDead = true; reason = 'HTTP_' + statusCode;
       } else if ((statusCode === 301 || statusCode === 302 || statusCode === 307) && http.headers && http.headers.location) {
         const originalPath = new URL(job.apply_url).pathname;
         const redirectPath = new URL(http.headers.location, job.apply_url).pathname;
         if (!redirectPath.includes(originalPath) && (
           redirectPath.endsWith('/jobs') ||
           redirectPath.endsWith('/careers') ||
           redirectPath === '/'
         )) {
           isDead = true; reason = 'SOFT_404_REDIRECT';
         } else {
           aliveCount++;
         }
       } else if (http.error) {
         errorCount++;
       } else {
         aliveCount++;
       }

       if (isDead) {
         console.log('DEAD [' + reason + ']: ' + job.company + ' | ' + job.job_title);
         deadJobs.push(job);
       }
     }

     console.log('Alive: ' + aliveCount + ' | Dead: ' + deadJobs.length + ' | Errors: ' + errorCount);

     if (deadJobs.length === 0) {
       return [{ json: { _no_dead_jobs: true, alive: aliveCount, errors: errorCount } }];
     }
     return deadJobs.map(function(j) { return { json: j }; });
     ```  
   - Connect `Check URLs` → `Find Dead Jobs`.

7. **Add IF Node – Has Dead Jobs?**  
   - Node type: `if` (v2)  
   - Label: `Has Dead Jobs?`  
   - Condition type: **Boolean**  
   - Left value: `={{ $json._no_dead_jobs }}`  
   - Operation: **not equal**  
   - Right value: `true`  
   - Connect `Find Dead Jobs` → `Has Dead Jobs?`.  
   - The **True** branch goes to the next node; the **False** branch ends the workflow (no connection).

8. **Add Postgres Node – Mark Inactive (Postgres)**  
   - Node type: `postgres` (v2.5)  
   - Label: `Mark Inactive (Postgres)`  
   - Operation: **Execute Query**  
   - Query:  
     ```sql
     UPDATE jobs SET status = 'inactive' WHERE job_hash = '{{ $json.job_hash }}';
     ```  
   - Error handling: **Continue** (regular output).  
   - Connect the **True** output of `Has Dead Jobs?` → input of this node.

9. **Add Google Sheets Node – Mark Inactive (Google Sheet)**  
   - Node type: `googleSheets` (v4)  
   - Label: `Mark Inactive (Google Sheet)`  
   - Operation: **Append or Update**  
   - Document ID: Replace `YOUR_RESOURCE_ID_HERE` with the actual Google Sheets file ID.  
   - Sheet Name: Replace `YOUR_RESOURCE_ID_HERE` with the sheet name or ID.  
   - Matching columns: `["job_hash"]`  
   - Column mapping: `job_hash = {{ $json.job_hash }}` (auto‑map can be left enabled).  
   - Connect `Mark Inactive (Postgres)` → `Mark Inactive (Google Sheet)`.  
   - Add Google Sheets OAuth2 credentials and grant the necessary scopes (spreadsheet read/write).

10. **Add Sticky Notes (optional but recommended)**  
    - Create a sticky note labelled **Main Sticky** with the content from the original workflow (overview, how it works, setup instructions, customization).  
    - Create three section sticky notes (Section 1, Section 2, Section 3) positioned near the respective groups, using the original text:  
      - Section 1: “## 1. Trigger and Extraction …”  
      - Section 2: “## 2. Validation and Processing …”  
      - Section 3: “## 3. Reporting and Sync …”

11. **Verify & Test**  
    - Run the workflow manually to confirm the scheduler, Postgres query, URL checks, and updates work as expected.  
    - Inspect logs from `Prepare URLs` and `Find Dead Jobs` to see how many URLs were skipped or marked dead.  
    - Confirm that rows in both Postgres and Google Sheets reflect the `inactive` status for dead links.

12. **Security & Best‑Practice Adjustments**  
    - Replace the interpolated `job_hash` in the Postgres UPDATE with a parameterized query if your n8n version supports it (to avoid SQL injection).  
    - Limit the batch size and timeout in the HTTP Request node if you experience rate‑limiting from remote job sites.  
    - Adjust the soft‑404 detection heuristics (e.g., add more redirect‑path patterns) to match the ATS platforms you monitor.

---

## 5. General Notes & Resources  

| Note Content | Context or Link |
|--------------|-----------------|
| The workflow uses **Supabase** (Postgres) as its source database. Any Postgres‑compatible instance works, but column names (`job_hash`, `apply_url`, `job_title`, `status`) must match. | N/A |
| Google Sheets integration requires a **Google OAuth2 credential** with spreadsheet read/write scope. Ensure the sheet contains a column named `job_hash` for row matching. | N/A |
| Soft‑404 detection currently checks for redirects ending in `/jobs`, `/careers`, or `/`. Add or remove paths in the `Find Dead Jobs` code block to accommodate other ATS behaviors (e.g., `/job-listings`). | N/A |
| The `Check URLs` node’s **Continue on Error** and **Never Error** options keep the workflow running even when individual URLs fail, allowing the subsequent code node to analyse the error objects. | N/A |
| If you experience high latency or many timeouts, increase the **timeout** value or reduce the **batch size** in the HTTP Request node. | N/A |
| For large datasets (thousands of rows), consider splitting the `Fetch Active Jobs` query or adding pagination, as the workflow currently processes all active jobs in a single batch. | N/A |
| The original workflow JSON contains placeholder values (`YOUR_RESOURCE_ID_HERE`) for the Google Sheets document and sheet ID. Replace these before execution. | N/A |
| The `Main Sticky` note contains a full description of the workflow, setup steps, and customization hints. It is a useful reference for onboarding new team members. | N/A |

---