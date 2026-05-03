Extract Public Email Addresses From Any Website Using Firecrawl

https://n8nworkflows.xyz/workflows/extract-public-email-addresses-from-any-website-using-firecrawl-5786


# Extract Public Email Addresses From Any Website Using Firecrawl

### 1. Workflow Overview

This workflow, titled **"The Recap AI - Email Scraper"**, automates the extraction of public email addresses from any given website URL using the Firecrawl API. It is designed primarily for users who need to gather contact emails from websites, such as marketers, researchers, or sales teams.

The workflow is logically divided into the following functional blocks:

- **1.1 Input Reception:** Captures user input for the target website URL via a web form.
- **1.2 Initial Website Mapping:** Uses Firecrawl’s `/map` endpoint to discover key pages related to contact or team information.
- **1.3 Batch Email Scraping:** Submits discovered URLs to Firecrawl’s `/scrape/batch` endpoint to extract and normalize email addresses.
- **1.4 Polling and Result Gathering:** Periodically fetches the scrape results and checks scrape status, with retry logic and rate limit handling.
- **1.5 Final Output Preparation:** Aggregates and formats the extracted email addresses into a clean output array.
- **1.6 Error Handling:** Detects excessive retries and stops the workflow with an error message.

A sticky note summarizes the workflow’s purpose and high-level approach, reinforcing its use of Firecrawl API endpoints and the formatting of results.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block collects the website URL input from the user via a simple form interface.

**Nodes Involved:**  
- `form_trigger`

**Node Details:**  
- **form_trigger**  
  - Type: Form Trigger node  
  - Role: Captures user input through a web form  
  - Configuration:  
    - Form title: "Email Scraper"  
    - One mandatory text field labeled "Website Url" with placeholder "https://aitools.inc"  
  - Key variables: User input available as `$json["Website Url"]`  
  - Input: None (entry point)  
  - Output: Connected to `map_website` node  
  - Failure modes: User may submit invalid URL format; no explicit validation beyond required field  
  - Version: 2.2

---

#### 2.2 Initial Website Mapping

**Overview:**  
This block uses Firecrawl’s `/map` endpoint to search the submitted website for pages likely to contain contact information, such as "about", "contact", "company", "authors", and "team" pages. It limits to 5 results.

**Nodes Involved:**  
- `map_website`

**Node Details:**  
- **map_website**  
  - Type: HTTP Request node  
  - Role: Posts to Firecrawl `/map` API to get relevant URLs  
  - Configuration:  
    - URL: `https://api.firecrawl.dev/v1/map`  
    - Method: POST  
    - Body (JSON):  
      ```json
      {
        "url": "{{ $json['Website Url'] }}",
        "search": "about contact company authors team",
        "limit": 5
      }
      ```  
    - Headers: Content-Type: application/json  
    - Authentication: HTTP Header Auth with Firecrawl API key  
    - Retry: max 5 tries, 5 seconds between tries, retry on failure enabled  
  - Input: Output of `form_trigger`  
  - Output: Provides mapped URLs in JSON property `links` to `start_batch_scrape`  
  - Failure modes: HTTP errors, authentication failures, API rate limiting, invalid input URLs  
  - Version: 4.2

---

#### 2.3 Batch Email Scraping

**Overview:**  
Sends the identified URLs to Firecrawl’s `/scrape/batch` API endpoint to extract email addresses, applying normalization rules for common obfuscations and deduplication.

**Nodes Involved:**  
- `start_batch_scrape`

**Node Details:**  
- **start_batch_scrape**  
  - Type: HTTP Request node  
  - Role: Initiates batch scraping for emails on identified pages  
  - Configuration:  
    - URL: `https://api.firecrawl.dev/v1/batch/scrape`  
    - Method: POST  
    - Body (JSON): Includes:  
      - `urls`: JSON serialized array of URLs from previous node's output (`$json.links`)  
      - `formats`: ["markdown", "json"]  
      - `proxy`: "stealth" (to mask scraping origin)  
      - `jsonOptions`: Contains a detailed prompt for email extraction and schema for validation  
    - Headers: Content-Type: application/json  
    - Authentication: HTTP Header Auth with Firecrawl API key  
  - Input: Output from `map_website`  
  - Output: Provides batch scrape job ID for polling  
  - Failure modes: Rate limits, malformed URLs, API errors  
  - Version: 4.2

---

#### 2.4 Polling and Result Gathering

**Overview:**  
This block periodically checks the status of the batch scraping job and fetches results when completed. It manages retries and rate limiting waits.

**Nodes Involved:**  
- `rate_limit_wait`  
- `fetch_scrape_results`  
- `check_scrape_completed`  
- `check_retry_count`  
- `too_many_attempts_error`

**Node Details:**  
- **rate_limit_wait**  
  - Type: Wait node  
  - Role: Pauses execution between polling attempts to avoid API rate limiting  
  - Configuration: Default wait (duration not explicitly set, defaults to seconds)  
  - Input: From `start_batch_scrape` or retry path  
  - Output: To `fetch_scrape_results`  
  - Failure modes: None (controlled wait)  
  - Version: 1.1

- **fetch_scrape_results**  
  - Type: HTTP Request node  
  - Role: Fetches the scrape results by job ID from Firecrawl API  
  - Configuration:  
    - URL: Dynamic endpoint with job ID: `https://api.firecrawl.dev/v1/batch/scrape/{{ $('start_batch_scrape').item.json.id }}`  
    - Method: GET  
    - Headers: Content-Type: application/json  
    - Authentication: HTTP Header Auth with Firecrawl API key  
  - Input: From `rate_limit_wait`  
  - Output: JSON containing scrape status and results to `check_scrape_completed`  
  - Failure modes: Network errors, invalid job ID, API errors  
  - Version: 4.2

- **check_scrape_completed**  
  - Type: If node  
  - Role: Tests if the scrape job status is "completed"  
  - Configuration: Condition: `$json.status === 'completed'`  
  - Input: From `fetch_scrape_results`  
  - Output:  
    - True branch: `set_result` node (process results)  
    - False branch: `check_retry_count` node (retry logic)  
  - Failure modes: Unexpected API response structure  
  - Version: 2.2

- **check_retry_count**  
  - Type: If node  
  - Role: Checks if the workflow has retried more than 12 times (to avoid infinite loops)  
  - Configuration: Condition: `$runIndex >= 12`  
  - Input: From `check_scrape_completed` false branch  
  - Output:  
    - True branch: `too_many_attempts_error` (stop workflow)  
    - False branch: `rate_limit_wait` (retry polling)  
  - Failure modes: N/A  
  - Version: 2.2

- **too_many_attempts_error**  
  - Type: Stop and Error node  
  - Role: Stops workflow execution and returns error message after too many retries  
  - Configuration: Error message: "Too many retries when attempting to scrape website."  
  - Input: From `check_retry_count` true branch  
  - Output: None (terminates workflow)  
  - Failure modes: N/A  
  - Version: 1

---

#### 2.5 Final Output Preparation

**Overview:**  
Once scraping completes successfully, this block consolidates all extracted email addresses, filters duplicates, and formats the result as an array accessible for further use.

**Nodes Involved:**  
- `set_result`

**Node Details:**  
- **set_result**  
  - Type: Set node  
  - Role: Aggregates and formats the scraped email addresses into a single array property  
  - Configuration:  
    - Assigns a new field named `scraped_email_addresses` as an array  
    - Expression:  
      ```javascript
      ($node["fetch_scrape_results"].json.data || [])
        .flatMap(item => item?.json?.email_addresses || [])
        .filter(email => typeof email === 'string' && email.trim())
      ```  
    - This flattens nested arrays of emails, filters out empty or invalid entries  
  - Input: From `check_scrape_completed` true branch  
  - Output: Final structured output of email addresses  
  - Failure modes: Missing or malformed data structure in API response  
  - Version: 3.4

---

#### 2.6 Documentation & Notes

**Nodes Involved:**  
- `Sticky Note1`

**Node Details:**  
- **Sticky Note1**  
  - Type: Sticky Note  
  - Role: Describes the workflow purpose and approach for user clarity  
  - Content:  
    ```
    ## Scrape Public Email Addresses

    - Takes a website's home page url as input to the automation
    - Uses Firecrawl's `/map` and `/scrape/batch` endpoints to scrape and extract email addresses that exist on the website's HTML
    - Formats the results in array
    ```  
  - Position: Top-left of the canvas for prominence  
  - Version: 1

---

### 3. Summary Table

| Node Name             | Node Type              | Functional Role                          | Input Node(s)            | Output Node(s)             | Sticky Note                                                   |
|-----------------------|------------------------|----------------------------------------|--------------------------|----------------------------|---------------------------------------------------------------|
| form_trigger          | Form Trigger           | Receive website URL input from user    | -                        | map_website                |                                                               |
| map_website           | HTTP Request           | Map website pages likely to contain contacts | form_trigger             | start_batch_scrape          |                                                               |
| start_batch_scrape    | HTTP Request           | Initiate batch scraping for emails     | map_website              | rate_limit_wait             |                                                               |
| rate_limit_wait       | Wait                   | Pause between polling attempts         | start_batch_scrape, check_retry_count (false) | fetch_scrape_results       |                                                               |
| fetch_scrape_results  | HTTP Request           | Fetch batch scrape results              | rate_limit_wait          | check_scrape_completed      |                                                               |
| check_scrape_completed| If                     | Check if scrape job is completed       | fetch_scrape_results     | set_result (true), check_retry_count (false) |                                                               |
| check_retry_count     | If                     | Check retry count to avoid infinite loops | check_scrape_completed (false) | too_many_attempts_error (true), rate_limit_wait (false) |                                                               |
| too_many_attempts_error | Stop and Error         | Stop workflow on too many retries      | check_retry_count (true) | -                          |                                                               |
| set_result            | Set                    | Aggregate and format final email array | check_scrape_completed (true) | -                          |                                                               |
| Sticky Note1          | Sticky Note            | Workflow purpose and approach summary  | -                        | -                          | ## Scrape Public Email Addresses - Takes a website’s home page url as input to the automation - Uses Firecrawl’s `/map` and `/scrape/batch` endpoints to scrape and extract email addresses that exist on the website’s HTML - Formats the results in array |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node**  
   - Name: `form_trigger`  
   - Configure form with title: "Email Scraper"  
   - Add one required text field:  
     - Label: "Website Url"  
     - Placeholder: "https://aitools.inc"  
   - This node will serve as the entry point for the workflow.

2. **Create an HTTP Request node to map website pages**  
   - Name: `map_website`  
   - Connect `form_trigger` output to this node’s input  
   - Set HTTP Method: POST  
   - URL: `https://api.firecrawl.dev/v1/map`  
   - Body Content-Type: JSON  
   - JSON Body:  
     ```json
     {
       "url": "{{ $json['Website Url'] }}",
       "search": "about contact company authors team",
       "limit": 5
     }
     ```  
   - Add header: Content-Type: application/json  
   - Authentication: HTTP Header Auth with Firecrawl API key credentials  
   - Enable retry on failure: max 5 tries, 5 seconds wait between tries.

3. **Create an HTTP Request node to start batch scraping**  
   - Name: `start_batch_scrape`  
   - Connect output of `map_website` to input here  
   - HTTP Method: POST  
   - URL: `https://api.firecrawl.dev/v1/batch/scrape`  
   - Body Content-Type: JSON  
   - JSON Body:  
     ```json
     {
       "urls": {{ JSON.stringify($json.links) }},
       "formats": ["markdown", "json"],
       "proxy": "stealth",
       "jsonOptions": {
         "prompt": "Extract every unique, fully-qualified email address found in the supplied web page. Normalize common obfuscations where “@” appears as “(at)”, “[at]”, “{at}”, “ at ”, “&#64;” and “.” appears as “(dot)”, “[dot]”, “{dot}”, “ dot ”, “&#46;”. Convert variants such as “user(at)example(dot)com” or “user at example dot com” to “user@example.com”. Ignore addresses hidden inside HTML comments, <script>, or <style> blocks. Deduplicate case-insensitively. The addresses shown in the example output below (e.g., “user@example.com”, “info@example.com”, “support@sample.org”) are placeholders; include them only if they genuinely exist on the web page.",
         "schema": {
           "type": "object",
           "properties": {
             "email_addresses": {
               "type": "array",
               "items": {
                 "type": "string",
                 "format": "email",
                 "description": "A valid email address found and extracted from the page"
               },
               "description": "An array of all email addresses found on the web page"
             }
           },
           "required": ["emails"]
         }
       }
     }
     ```  
   - Headers: Content-Type: application/json  
   - Authentication: HTTP Header Auth with Firecrawl API key

4. **Create a Wait node for rate limiting**  
   - Name: `rate_limit_wait`  
   - Connect output of `start_batch_scrape` to input here  
   - No specific duration configured (default wait)

5. **Create an HTTP Request node to fetch scrape results**  
   - Name: `fetch_scrape_results`  
   - Connect output of `rate_limit_wait` to input here  
   - HTTP Method: GET  
   - URL: `https://api.firecrawl.dev/v1/batch/scrape/{{ $('start_batch_scrape').item.json.id }}` (dynamic)  
   - Headers: Content-Type: application/json  
   - Authentication: HTTP Header Auth with Firecrawl API key

6. **Create an If node to check scrape completion**  
   - Name: `check_scrape_completed`  
   - Connect output of `fetch_scrape_results` to input here  
   - Condition: Check if `$json.status === 'completed'`  
   - True branch connects to `set_result` node (next step)  
   - False branch connects to `check_retry_count` node (next step)

7. **Create an If node to check retry count**  
   - Name: `check_retry_count`  
   - Connect false output of `check_scrape_completed` here  
   - Condition: `$runIndex >= 12`  
   - True branch connects to `too_many_attempts_error` node  
   - False branch connects back to `rate_limit_wait` node (creates retry loop)

8. **Create a Stop and Error node for too many retries**  
   - Name: `too_many_attempts_error`  
   - Connect true output of `check_retry_count` here  
   - Error message: "Too many retries when attempting to scrape website."  
   - This node terminates the workflow on failure

9. **Create a Set node to prepare final results**  
   - Name: `set_result`  
   - Connect true output of `check_scrape_completed` here  
   - Add field: `scraped_email_addresses` (Array)  
   - Value (Expression):  
     ```javascript
     ($node["fetch_scrape_results"].json.data || [])
       .flatMap(item => item?.json?.email_addresses || [])
       .filter(email => typeof email === 'string' && email.trim())
     ```  
   - Outputs the clean array of all extracted emails

10. **(Optional) Add a Sticky Note node**  
    - Place at top-left for documentation  
    - Content:  
      ```
      ## Scrape Public Email Addresses

      - Takes a website's home page url as input to the automation
      - Uses Firecrawl's `/map` and `/scrape/batch` endpoints to scrape and extract email addresses that exist on the website's HTML
      - Formats the results in array
      ```

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                   |
|-------------------------------------------------------------------------------------------------|--------------------------------------------------|
| Workflow uses Firecrawl API (https://firecrawl.dev) for website mapping and email scraping      | Firecrawl official API documentation              |
| Handles common email obfuscations like `(at)`, `[dot]` and converts them to standard emails     | Improves accuracy of email extraction             |
| Retry logic with 12 attempts max prevents infinite polling cycles                                | Protects from indefinite waits or failures        |
| Proxy mode "stealth" used to disguise scraping requests                                         | Useful for avoiding bot detection                  |
| Sticky note summarizing workflow purpose is placed on canvas for user clarity                   | Enhances maintainability and onboarding            |
| Workflow is inactive by default (`active: false`) — remember to activate before production use  | n8n workflow setting                               |

---

**Disclaimer:**  
The provided text is extracted exclusively from an automated workflow built with n8n, a workflow automation tool. It strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.