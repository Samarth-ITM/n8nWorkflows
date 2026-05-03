Orchestrate Web Crawls with Scrapyd and Automated Data Enrichment

https://n8nworkflows.xyz/workflows/orchestrate-web-crawls-with-scrapyd-and-automated-data-enrichment-8552


# Orchestrate Web Crawls with Scrapyd and Automated Data Enrichment

### 1. Workflow Overview

This workflow automates the orchestration of web crawling jobs using Scrapyd, a service for running Scrapy spiders remotely. It manages the entire lifecycle: launching a spider job with specified parameters, monitoring its progress until completion, retrieving and enriching the scraped data, and finally returning structured JSON results through a webhook. Optional debug outputs such as job logs, HTML page dumps, and screenshots are also collected and combined for further inspection.

The logical blocks in this workflow are:

- **1.1 Job Launch and Trigger**: Starts the workflow manually and initiates a Scrapyd spider job with configured parameters.
- **1.2 Job Monitoring and Status Checking**: Polls Scrapyd to monitor the job status until it completes.
- **1.3 Data Retrieval and Enrichment**: Once the job is finished, retrieves the scraped items, processes them to deduplicate and enrich with structured fields.
- **1.4 Debug Artifacts Collection**: Optionally collects job logs, screenshots, and HTML files, combining them for debugging or archival.
- **1.5 Response Delivery**: Sends the final enriched data back to the caller via a webhook response.

---

### 2. Block-by-Block Analysis

#### 2.1 Job Launch and Trigger

- **Overview**: Initiates the workflow manually and starts a Scrapyd spider job with parameters such as project name, spider type, config path, and search parameters.
- **Nodes Involved**: When clicking ‘Execute workflow’, run_job, job_list

##### Node Details

- **When clicking ‘Execute workflow’**
  - Type: Manual Trigger
  - Role: Entry point for manual execution.
  - Configuration: No parameters; triggers workflow execution on demand.
  - Inputs: None
  - Outputs: run_job
  - Edge cases: User must manually trigger; no automation without manual start.

- **run_job**
  - Type: HTTP Request
  - Role: Sends a POST request to Scrapyd’s schedule endpoint to start a spider job.
  - Configuration:
    - URL: `http://localhost:6800/schedule.json`
    - Method: POST
    - Content-Type: form-urlencoded
    - Body parameters: project ("null"), spider ("generic_list"), config_path (path to YAML config), project_id, query parameters (q, sci, prs, pages)
  - Inputs: When clicking ‘Execute workflow’
  - Outputs: job_list
  - Edge cases:
    - Scrapyd service must be running and accessible at localhost:6800.
    - Job parameters must be valid; invalid params may cause scheduling failure.
    - Network or server errors may cause request failure.

- **job_list**
  - Type: HTTP Request
  - Role: Fetches the list of current Scrapyd jobs to track the launched job.
  - Configuration:
    - URL: `http://localhost:6800/listjobs.json`
    - Sends query parameter project = "null"
  - Inputs: run_job
  - Outputs: filter_job
  - Edge cases:
    - Scrapyd must respond with valid JSON.
    - Network issues may cause failure.
    - Project parameter "null" must match Scrapyd configuration.

---

#### 2.2 Job Monitoring and Status Checking

- **Overview**: Polls the Scrapyd job list repeatedly to check if the targeted job has finished. Waits and retries if not finished.
- **Nodes Involved**: filter_job, check_job_status, Wait3, check_items, job_log

##### Node Details

- **filter_job**
  - Type: Code
  - Role: Filters Scrapyd job list for the specific job id to check if finished.
  - Configuration:
    - JavaScript code extracts job ID from run_job node or workflow variable.
    - Searches only in 'finished' jobs array.
    - Returns status object indicating if job is finished.
  - Inputs: job_list
  - Outputs: check_job_status
  - Edge cases:
    - Job ID must be available or set in flow context.
    - If job not found, returns `{ match: false }`—workflow branches accordingly.

- **check_job_status**
  - Type: If (Conditional)
  - Role: Branches workflow based on job completion status.
  - Configuration:
    - Condition: `$json.match` not equal to false means job finished.
  - Inputs: filter_job
  - Outputs:
    - True: check_items, job_log
    - False: Wait3
  - Edge cases:
    - Condition must correctly interpret JSON match field.
    - Misinterpretation leads to infinite wait or premature termination.

- **Wait3**
  - Type: Wait
  - Role: Pauses workflow briefly before retrying job status check.
  - Configuration: Default wait time (not explicitly set, default behavior).
  - Inputs: check_job_status (false branch)
  - Outputs: job_list (loops back)
  - Edge cases:
    - Excessive waiting may cause workflow timeout.
    - No explicit retry counter; infinite loop possible if job never finishes.

- **check_items**
  - Type: HTTP Request
  - Role: Retrieves the scraped items JSONL file from Scrapyd after job completion.
  - Configuration:
    - URL constructed dynamically using `$json.items_url` from job info.
  - Inputs: check_job_status (true branch)
  - Outputs: Filter-result, If-job-check
  - Edge cases:
    - URL must be valid and items file accessible.
    - Network or file missing errors possible.

- **job_log**
  - Type: HTTP Request
  - Role: Retrieves the job log file for debugging.
  - Configuration:
    - URL constructed dynamically using `$json.log_url`.
  - Inputs: check_job_status (true branch)
  - Outputs: none connected downstream here.
  - Edge cases:
    - Log file may be large or unavailable.
    - Network issues can cause failure.

---

#### 2.3 Data Retrieval and Enrichment

- **Overview**: Parses the scraped JSONL data, deduplicates entries by URL (keeping the cheapest price), extracts structured fields (part numbers, make, model, etc.), enriches with metadata (domain, source, timestamp), and sorts results.
- **Nodes Involved**: Filter-result

##### Node Details

- **Filter-result**
  - Type: Code
  - Role: Processes raw JSONL scraped results into enriched, clean JSON objects.
  - Configuration:
    - JavaScript code:
      - Normalizes input from raw string, JSON wrapper, or base64 binary.
      - Parses each JSONL line safely.
      - Extracts price, domain, URL path-based fields (id, partNo, make, model, partName).
      - Deduplicates by URL, keeping the lowest price.
      - Sorts by ascending price.
      - Returns structured array of JSON objects.
  - Inputs: check_items
  - Outputs: Respond to Webhook
  - Edge cases:
    - Malformed JSON lines handled gracefully.
    - Missing or invalid price fields handled with nulls.
    - URLs that cannot be parsed may yield incomplete metadata.

---

#### 2.4 Debug Artifacts Collection

- **Overview**: Optionally collects job screenshots and HTML page dumps, splits lists into batches for downloading, and aggregates the files for easier downstream use.
- **Nodes Involved**: If-job-check, screenshots, HTML, Split-screenshot, Split HTML, loop-screenshots, loop-html, DL-screenshots, DL-html, combine-screenshots, combine-html

##### Node Details

- **If-job-check**
  - Type: If (Conditional)
  - Role: Checks if `data` field exists and is "on" to proceed with debug collection.
  - Configuration:
    - Condition: checks if `$json.data` exists.
  - Inputs: check_items
  - Outputs: screenshots, HTML
  - Edge cases:
    - If condition fails, debug files are skipped.
    - Misconfigured input may bypass debug collection.

- **screenshots**
  - Type: HTTP Request
  - Role: Lists screenshot files from Scrapyd debug folder using job ID.
  - Configuration:
    - URL constructed with job ID.
    - Sends Authorization header with Bearer token from environment variable.
  - Inputs: If-job-check
  - Outputs: Split-screenshot
  - Edge cases:
    - Authorization token must be valid.
    - Screenshots folder may be empty or inaccessible.

- **Split-screenshot**
  - Type: SplitOut
  - Role: Splits screenshot file list into individual items for batch processing.
  - Configuration:
    - Field to split: "files"
  - Inputs: screenshots
  - Outputs: loop-screenshots
  - Edge cases:
    - Empty or missing 'files' field causes no output.

- **loop-screenshots**
  - Type: SplitInBatches
  - Role: Processes screenshots in batches (default batch size).
  - Inputs: Split-screenshot
  - Outputs: combine-screenshots, DL-screenshots
  - Edge cases:
    - Batch size defaults used; tuning might be needed for large sets.

- **DL-screenshots**
  - Type: HTTP Request
  - Role: Downloads each screenshot file using its path.
  - Configuration:
    - URL fixed to Scrapyd loader endpoint.
    - Sends Authorization header.
    - Query param 'path' set dynamically.
  - Inputs: loop-screenshots
  - Outputs: loop-screenshots (looping for next batch)
  - Edge cases:
    - Network or auth failure may cause partial downloads.

- **combine-screenshots**
  - Type: Aggregate
  - Role: Combines all downloaded screenshots into a single dataset.
  - Configuration:
    - Aggregates the binary 'data' fields.
  - Inputs: loop-screenshots
  - Outputs: none connected downstream here.
  - Edge cases:
    - Large binary data may impact performance.

- **HTML**
  - Type: HTTP Request
  - Role: Lists HTML dump files from Scrapyd debug folder using job ID.
  - Configuration:
    - URL constructed with job ID.
    - Sends Authorization header.
  - Inputs: If-job-check
  - Outputs: Split HTML
  - Edge cases:
    - Similar auth and availability constraints as screenshots.

- **Split HTML**
  - Type: SplitOut
  - Role: Splits HTML file list into individual items.
  - Inputs: HTML
  - Outputs: loop-html
  - Edge cases:
    - Empty 'files' field leads to no output.

- **loop-html**
  - Type: SplitInBatches
  - Role: Processes HTML files in batches.
  - Inputs: Split HTML
  - Outputs: combine-html, DL-html
  - Edge cases:
    - Batch size tuning may be required.

- **DL-html**
  - Type: HTTP Request
  - Role: Downloads each HTML file using its path.
  - Configuration:
    - URL pointing to Scrapyd loader endpoint.
    - Sends Authorization header.
    - Query param 'path' set dynamically.
    - On error configured to continue regular output.
  - Inputs: loop-html
  - Outputs: loop-html (for next batch)
  - Edge cases:
    - Errors handled gracefully; partial downloads possible.

- **combine-html**
  - Type: Aggregate
  - Role: Combines downloaded HTML files into a single dataset.
  - Configuration:
    - Aggregates binary 'data' fields.
  - Inputs: loop-html
  - Outputs: none connected downstream here.
  - Edge cases:
    - Large binary data may impact performance.

---

#### 2.5 Response Delivery

- **Overview**: Sends the enriched and structured JSON data back to the caller via an HTTP webhook response.
- **Nodes Involved**: Respond to Webhook

##### Node Details

- **Respond to Webhook**
  - Type: Respond to Webhook
  - Role: Final node to send output data as HTTP response.
  - Configuration:
    - No special parameters; responds with input JSON.
  - Inputs: Filter-result
  - Outputs: None
  - Edge cases:
    - Must be invoked as part of a webhook-triggered workflow to respond properly.
    - If used standalone, response might not be delivered.

---

### 3. Summary Table

| Node Name                   | Node Type             | Functional Role                          | Input Node(s)                    | Output Node(s)                        | Sticky Note                                                                                                                       |
|-----------------------------|-----------------------|----------------------------------------|---------------------------------|-------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger        | Manual entry point                      | None                            | run_job                             |                                                                                                                                  |
| run_job                     | HTTP Request          | Schedule Scrapyd spider job             | When clicking ‘Execute workflow’ | job_list                           |                                                                                                                                  |
| job_list                    | HTTP Request          | Retrieve list of Scrapyd jobs           | run_job                         | filter_job                         |                                                                                                                                  |
| filter_job                  | Code                  | Filter job list for target job finished | job_list                        | check_job_status                   |                                                                                                                                  |
| check_job_status            | If                    | Check if job is finished                | filter_job                      | check_items, job_log (true branch), Wait3 (false branch) |                                                                                                                                  |
| Wait3                      | Wait                  | Wait before retrying job status check  | check_job_status (false)         | job_list                           |                                                                                                                                  |
| check_items                | HTTP Request          | Retrieve scraped items                   | check_job_status (true)          | Filter-result, If-job-check         |                                                                                                                                  |
| job_log                    | HTTP Request          | Retrieve job log for debugging           | check_job_status (true)          | None                              | You job log and you can send via email or save it locally                                                                       |
| Filter-result              | Code                  | Enrich, deduplicate, and sort data       | check_items                     | Respond to Webhook                 |                                                                                                                                  |
| Respond to Webhook          | Respond to Webhook    | Send enriched JSON data response         | Filter-result                   | None                              | Add your webhook here to get the respond in JSON                                                                                 |
| If-job-check               | If                    | Check if debug data collection is on     | check_items                    | screenshots, HTML                 | You can set a warning here if job result were empty                                                                              |
| screenshots                | HTTP Request          | List screenshot files for job            | If-job-check                   | Split-screenshot                  | This will combine the page screenshots files and you can send the result via email or save locally                              |
| Split-screenshot           | SplitOut              | Split screenshots list for batch processing | screenshots                  | loop-screenshots                 | This will combine the page screenshots files and you can send the result via email or save locally                              |
| loop-screenshots           | SplitInBatches        | Batch process screenshot downloads       | Split-screenshot              | combine-screenshots, DL-screenshots | This will combine the page screenshots files and you can send the result via email or save locally                              |
| DL-screenshots             | HTTP Request          | Download individual screenshot           | loop-screenshots              | loop-screenshots                 | This will combine the page screenshots files and you can send the result via email or save locally                              |
| combine-screenshots        | Aggregate             | Combine downloaded screenshots           | loop-screenshots              | None                            | This will combine the page screenshots files and you can send the result via email or save locally                              |
| HTML                      | HTTP Request          | List HTML files for job                   | If-job-check                   | Split HTML                      | This will combine the html files and you can send the result via email or save locally                                          |
| Split HTML                | SplitOut              | Split HTML file list for batch processing | HTML                         | loop-html                      | This will combine the html files and you can send the result via email or save locally                                          |
| loop-html                 | SplitInBatches        | Batch process HTML downloads              | Split HTML                   | combine-html, DL-html            | This will combine the html files and you can send the result via email or save locally                                          |
| DL-html                   | HTTP Request          | Download individual HTML file             | loop-html                    | loop-html                      | This will combine the html files and you can send the result via email or save locally                                          |
| combine-html              | Aggregate             | Combine downloaded HTML files             | loop-html                    | None                            | This will combine the html files and you can send the result via email or save locally                                          |
| Sticky Note               | Sticky Note           | Documentation and overview note           | None                         | None                            | ## 🟢 Job Orchestration... [Full detailed explanation with GitHub repo link]                                                   |
| Sticky Note1              | Sticky Note           | Instruction related to webhook response   | None                         | None                            | Add your webhook here to get the respond in JSON                                                                                |
| Sticky Note2              | Sticky Note           | Info on combining HTML files              | None                         | None                            | This will combine the html files and you can send the result via email or save locally                                          |
| Sticky Note3              | Sticky Note           | Info on combining screenshots             | None                         | None                            | This will combine the page screenshots files and you can send the result via email or save locally                              |
| Sticky Note4              | Sticky Note           | Warning setting if job result empty       | None                         | None                            | You can set a warning here if job result were empty                                                                             |
| Sticky Note5              | Sticky Note           | Info on job log handling                   | None                         | None                            | You job log and you can send via email or save it locally                                                                       |
| Sticky Note6              | Sticky Note           | Monitoring job overview                    | None                         | None                            | ## Monitoring job                                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**
   - Type: Manual Trigger
   - Name: When clicking ‘Execute workflow’
   - No parameters needed.
   - This node starts the workflow manually.

2. **Create HTTP Request Node to Run Job**
   - Type: HTTP Request
   - Name: run_job
   - Method: POST
   - URL: `http://localhost:6800/schedule.json`
   - Content-Type: form-urlencoded
   - Body parameters:
     - project: "null"
     - spider: "generic_list"
     - config_path: "/data/sites/project001/config.yaml"
     - project_id: "project001"
     - q: "null"
     - sci: "123"
     - prs: "2"
     - pages: "2"
   - Connect input from Manual Trigger node.

3. **Create HTTP Request Node to List Jobs**
   - Type: HTTP Request
   - Name: job_list
   - Method: GET
   - URL: `http://localhost:6800/listjobs.json`
   - Query parameter: project = "null"
   - Connect input from run_job.

4. **Create Code Node to Filter Job**
   - Type: Code
   - Name: filter_job
   - JavaScript:
     ```javascript
     const jobId = $('run_job').first().json.jobid || $flow.get('jobId');
     const first = $input.first().json;
     const root = Array.isArray(first) ? first[0] : first;
     const hit = (root.finished || []).find(j => j.id === jobId);
     if (hit) {
       return [{ json: hit }];
     } else {
       return [{ json: { match: false, lookedFor: jobId } }];
     }
     ```
   - Connect input from job_list.

5. **Create If Node to Check Job Status**
   - Type: If
   - Name: check_job_status
   - Condition: `$json.match` is not equal to "false"
   - Connect input from filter_job.
   - True branch connects to check_items and job_log.
   - False branch connects to Wait3.

6. **Create Wait Node**
   - Type: Wait
   - Name: Wait3
   - Default wait parameters (default delay)
   - Connect input from check_job_status (false branch).
   - Connect output to job_list (loop for retry).

7. **Create HTTP Request Node to Retrieve Items**
   - Type: HTTP Request
   - Name: check_items
   - Method: GET
   - URL: `http://localhost:6800{{$json.items_url}}`
   - Connect input from check_job_status (true branch).

8. **Create HTTP Request Node to Retrieve Job Log**
   - Type: HTTP Request
   - Name: job_log
   - Method: GET
   - URL: `http://localhost:6800{{$json.log_url}}`
   - Connect input from check_job_status (true branch).

9. **Create If Node to Check Debug Option**
   - Type: If
   - Name: If-job-check
   - Condition: `$json.data` exists
   - Connect input from check_items.
   - True branch connects to screenshots and HTML nodes.

10. **Create HTTP Request Node to List Screenshots**
    - Type: HTTP Request
    - Name: screenshots
    - Method: GET
    - URL: `http://localhost:8080/files/list?subpath=debug/{{$('run_job').item.json.jobid}}/screenshots`
    - Add HTTP Header: Authorization: Bearer `$LOADER_API_TOKEN` (set as environment variable)
    - Connect input from If-job-check.

11. **Create SplitOut Node for Screenshots**
    - Type: SplitOut
    - Name: Split-screenshot
    - Field to split: "files"
    - Connect input from screenshots.

12. **Create SplitInBatches Node for Screenshots**
    - Type: SplitInBatches
    - Name: loop-screenshots
    - Connect input from Split-screenshot.

13. **Create HTTP Request Node to Download Screenshot**
    - Type: HTTP Request
    - Name: DL-screenshots
    - Method: GET
    - URL: `http://localhost:8080/files/get`
    - Query parameter: path = `{{$json.path}}`
    - Header: Authorization: Bearer `$LOADER_API_TOKEN`
    - On error: Continue Regular Output
    - Connect input from loop-screenshots.
    - Output loops back to loop-screenshots for next batch.

14. **Create Aggregate Node for Screenshots**
    - Type: Aggregate
    - Name: combine-screenshots
    - Aggregate field: `data` (include binaries)
    - Connect input from loop-screenshots.

15. **Create HTTP Request Node to List HTML Files**
    - Type: HTTP Request
    - Name: HTML
    - Method: GET
    - URL: `http://localhost:8080/files/list?subpath=debug/{{$('run_job').item.json.jobid}}/html`
    - Header: Authorization: Bearer `$LOADER_API_TOKEN`
    - Connect input from If-job-check.

16. **Create SplitOut Node for HTML**
    - Type: SplitOut
    - Name: Split HTML
    - Field to split: "files"
    - Connect input from HTML.

17. **Create SplitInBatches Node for HTML**
    - Type: SplitInBatches
    - Name: loop-html
    - Connect input from Split HTML.

18. **Create HTTP Request Node to Download HTML**
    - Type: HTTP Request
    - Name: DL-html
    - Method: GET
    - URL: `http://localhost:8080/files/get`
    - Query parameter: path = `{{$json.path}}`
    - Header: Authorization: Bearer `$LOADER_API_TOKEN`
    - On error: Continue Regular Output
    - Connect input from loop-html.
    - Output loops back to loop-html.

19. **Create Aggregate Node for HTML**
    - Type: Aggregate
    - Name: combine-html
    - Aggregate field: `data` (include binaries)
    - Connect input from loop-html.

20. **Create Code Node to Filter and Enrich Results**
    - Type: Code
    - Name: Filter-result
    - JavaScript: Use the provided code that normalizes input, parses JSONL, deduplicates by URL (keeping cheapest price), enriches with structured fields, sorts by price, and returns array of clean JSON objects.
    - Connect input from check_items.

21. **Create Respond to Webhook Node**
    - Type: Respond to Webhook
    - Name: Respond to Webhook
    - Connect input from Filter-result.

22. **Set Credentials**
    - For HTTP Request nodes that require authorization (screenshots, DL-screenshots, HTML, DL-html), configure environment variable `LOADER_API_TOKEN` with a valid bearer token.
    - Scrapyd endpoints assumed to be running locally without auth.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                              | Context or Link                                                                                 |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| ## 🟢 Job Orchestration: This workflow automates the full lifecycle of running a Scrapy spider via Scrapyd: launching the job, monitoring until completion, collecting results, and outputting clean structured JSON. Full backend code and setup is in my private repo. | https://github.com/PersianGru?tab=repositories                                                  |
| Add your webhook here to get the respond in JSON                                                                                                                                                                                                                          | Sticky Note near Respond to Webhook node                                                        |
| This will combine the html files and you can send the result via email or save locally                                                                                                                                                                                     | Sticky Note near HTML file combination nodes                                                    |
| This will combine the page screenshots files and you can send the result via email or save locally                                                                                                                                                                         | Sticky Note near screenshot combination nodes                                                  |
| You can set a warning here if job result were empty                                                                                                                                                                                                                       | Sticky Note near If-job-check node                                                             |
| You job log and you can send via email or save it locally                                                                                                                                                                                                                  | Sticky Note near job_log node                                                                  |
| ## Monitoring job                                                                                                                                                                                                                                                          | Sticky Note near job monitoring nodes                                                          |

---

**Disclaimer:**  
The provided text is exclusively derived from an n8n automated workflow. It complies strictly with current content policies and contains no illegal, offensive, or protected content. All data handled is legal and publicly available.