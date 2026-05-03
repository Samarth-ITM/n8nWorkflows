Check job apply URLs and deactivate dead links in Postgres and Google Sheets

https://n8nworkflows.xyz/workflows/check-job-apply-urls-and-deactivate-dead-links-in-postgres-and-google-sheets-14807


# Check job apply URLs and deactivate dead links in Postgres and Google Sheets

# Workflow Documentation: Check job apply URLs and deactivate dead links

### 1. Workflow Overview
This workflow is designed to maintain data hygiene for a job application database. It automatically scans a list of active job application URLs to identify "dead" links (404s, DNS failures, or redirects to homepages) and marks them as inactive in both a PostgreSQL database (Supabase) and a Google Sheet.

The logic is divided into three primary functional blocks:
- **1.1 Trigger & Extraction:** Handles the scheduling and retrieval of active job records.
- **1.2 Validation & Processing:** Cleans the URL data, performs network requests, and executes logic to determine if a link is truly broken.
- **1.3 Reporting & Sync:** Updates the status of the identified dead links across all integrated platforms.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Extraction
**Overview:** Initiates the process on a recurring schedule and fetches the current list of active jobs to be validated.

- **Nodes Involved:** `⏰ Every 3 Days`, `📥 Fetch Active Jobs`
- **Node Details:**
    - **⏰ Every 3 Days**
        - *Type:* Schedule Trigger
        - *Configuration:* Set to run every 72 hours.
        - *Output:* Triggers the workflow.
    - **📥 Fetch Active Jobs**
        - *Type:* Postgres Node
        - *Configuration:* Executes a `SELECT` query to retrieve `job_hash`, `apply_url`, `company`, and `job_title` where the status is 'active'.
        - *Input:* Triggered by the schedule.
        - *Output:* A list of active job records.
        - *Potential Failures:* Database connection timeout or authentication errors.

#### 2.2 Validation & Processing
**Overview:** Pre-filters the data to avoid waste and uses HTTP requests and custom JavaScript to detect dead links, including "soft 404s".

- **Nodes Involved:** `🔧 Prepare URLs`, `🔗 Check URLs`, `🧠 Find Dead Jobs`, `🔀 Has Dead Jobs?`
- **Node Details:**
    - **🔧 Prepare URLs**
        - *Type:* Code Node (JavaScript)
        - *Configuration:* Filters out items with empty URLs, URLs shorter than 10 characters, or those not starting with "http".
        - *Output:* A cleaned list of jobs; returns a special `_no_jobs` flag if the list is empty.
    - **🔗 Check URLs**
        - *Type:* HTTP Request Node
        - *Configuration:* 
            - Method: `HEAD` (to minimize bandwidth).
            - Timeout: 5000ms.
            - Batch Size: 5.
            - Redirects: Follows up to 5 redirects.
            - Error Handling: `continueRegularOutput` (prevents workflow crash on 404/500).
        - *Input:* URLs from "Prepare URLs".
        - *Output:* HTTP status codes and headers.
    - **🧠 Find Dead Jobs**
        - *Type:* Code Node (JavaScript)
        - *Configuration:* Complex logic to identify "dead" status based on:
            - DNS failures (`enotfound`, `getaddrinfo`).
            - Connection refused (`econnrefused`).
            - Hard 404/410 status codes.
            - **Soft 404s:** Detects if a redirect leads away from the specific job path toward a generic `/jobs` or `/careers` page.
        - *Output:* A list of jobs confirmed as dead.
    - **🔀 Has Dead Jobs?**
        - *Type:* IF Node
        - *Configuration:* Checks if the result from "Find Dead Jobs" is NOT the `_no_dead_jobs` flag.
        - *Input:* Processed results from the logic node.

#### 2.3 Reporting & Sync
**Overview:** Synchronizes the "inactive" status across the primary database and the reporting sheet.

- **Nodes Involved:** `❌ Mark Inactive (Supabase)`, `📊 Mark Inactive (Google Sheet)`
- **Node Details:**
    - **❌ Mark Inactive (Supabase)**
        - *Type:* Postgres Node
        - *Configuration:* Executes an `UPDATE` query setting `status = 'inactive'` where the `job_hash` matches the current item.
        - *Input:* Items passing the "Has Dead Jobs?" check.
        - *Potential Failures:* SQL syntax errors or permission issues.
    - **📊 Mark Inactive (Google Sheet)**
        - *Type:* Google Sheets Node
        - *Configuration:* 
            - Operation: `appendOrUpdate`.
            - Matching Column: `job_hash`.
            - Value updated: Sets the status to inactive (via mapping).
        - *Input:* Confirmed dead jobs from the Postgres update.
        - *Potential Failures:* Expired OAuth2 token, missing Sheet ID.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| ⏰ Every 3 Days | Schedule Trigger | Workflow Initiation | - | 📥 Fetch Active Jobs | Trigger & Extraction |
| 📥 Fetch Active Jobs | Postgres | Data Retrieval | ⏰ Every 3 Days | 🔧 Prepare URLs | Trigger & Extraction |
| 🔧 Prepare URLs | Code | Data Cleaning | 📥 Fetch Active Jobs | 🔗 Check URLs | Validation & Processing |
| 🔗 Check URLs | HTTP Request | Connectivity Test | 🔧 Prepare URLs | 🧠 Find Dead Jobs | Validation & Processing |
| 🧠 Find Dead Jobs | Code | Death Logic Analysis | 🔗 Check URLs | 🔀 Has Dead Jobs? | Validation & Processing |
| 🔀 Has Dead Jobs? | IF | Decision Gateway | 🧠 Find Dead Jobs | ❌ Mark Inactive (Supabase) | Validation & Processing |
| ❌ Mark Inactive (Supabase) | Postgres | Database Sync | 🔀 Has Dead Jobs? | 📊 Mark Inactive (Google Sheet) | Reporting & Sync |
| 📊 Mark Inactive (Google Sheet) | Google Sheets | Spreadsheet Sync | ❌ Mark Inactive (Supabase) | - | Reporting & Sync |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger:** Create a **Schedule Trigger** node. Set the interval to 72 hours (3 days).
2.  **Data Retrieval:** Add a **Postgres** node. Set the operation to "Execute Query" and use: `SELECT job_hash, apply_url, company, job_title FROM jobs WHERE status = 'active' ORDER BY created_at ASC;`.
3.  **Preprocessing:** Add a **Code** node. Implement the JavaScript logic to filter out empty strings or URLs not starting with `http`. Ensure it returns a `_no_jobs` flag if the array is empty.
4.  **URL Validation:** Add an **HTTP Request** node.
    - Method: `HEAD`.
    - URL: `{{ $json.apply_url }}`.
    - In "Options", enable "Never Error" and "Full Response".
    - Set "Batching" size to 5.
5.  **Analysis Logic:** Add a **Code** node. Implement the logic to check for `statusCode === 404`, DNS errors (`enotfound`), and redirect path comparisons (original path vs redirect path) to identify soft 404s. Return only the `job` objects that are dead.
6.  **Conditional Check:** Add an **IF** node. Check if the input contains the `_no_dead_jobs` flag. Route "False" (meaning dead jobs exist) to the next step.
7.  **DB Update:** Add a **Postgres** node. Set the operation to "Execute Query" and use: `UPDATE jobs SET status = 'inactive' WHERE job_hash = '{{ $json.job_hash }}';`.
8.  **Sheet Update:** Add a **Google Sheets** node. 
    - Operation: `appendOrUpdate`.
    - Document ID and Sheet Name: Provide your specific resource IDs.
    - Matching Column: `job_hash`.
    - Mapping: Map the `job_hash` and the updated status.

**Credential Requirements:**
- **Postgres:** Host, Database, User, Password, Port.
- **Google Sheets:** OAuth2 Connection.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Maintain data hygiene by automatically scanning job application URLs and disabling broken links. | High-level Workflow Goal |
| Adjust the URL validation logic in the 'Find Dead Jobs' node to fit specific site behaviors. | Customization Advice |
| Consolidate user-specific values in a Set node at the workflow start for easier config. | Architecture Tip |