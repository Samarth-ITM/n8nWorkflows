Aggregate multi-source job boards into Supabase and Google Sheets

https://n8nworkflows.xyz/workflows/aggregate-multi-source-job-boards-into-supabase-and-google-sheets-14806


# Aggregate multi-source job boards into Supabase and Google Sheets

# Technical Documentation: Multi-Source Job Board Aggregator

## 1. Workflow Overview
This workflow automates the collection, normalization, and synchronization of job postings from various Applicant Tracking Systems (ATS) and global job boards into a centralized database (Supabase) and a tracking spreadsheet (Google Sheets).

The system is designed to handle highly inconsistent data structures across different platforms, employing a custom normalization engine to ensure all output data follows a strict, unified schema.

### Logical Blocks:
- **1.1 Source Discovery & Trigger:** Schedules the execution and defines the list of target companies and their respective ATS platforms.
- **1.2 Request & Normalization:** Constructs API endpoints, handles pagination, fetches raw data, and transforms diverse JSON payloads into a standardized format.
- **1.3 Storage & Output:** Performs "upsert" operations (update or insert) to prevent duplication in Supabase and Google Sheets.

---

## 2. Block-by-Block Analysis

### 2.1 Source Discovery & Trigger
**Overview:** Initiates the workflow on a schedule and provides the configuration array of target job boards.

- **Nodes Involved:** `⏰ Daily 8AM IST`, `📋 Company List`, `🔄 Loop Batches (5)`
- **Node Details:**
    - **⏰ Daily 8AM IST (Schedule Trigger):** Triggers the workflow daily at 08:00 IST.
    - **📋 Company List (Code):** A JavaScript node acting as a configuration registry. It returns an array of objects containing `ats` (platform type), `slug` (company identifier), and `company` (display name). It categorizes sources into Global Boards, Greenhouse, SmartRecruiters, Ashby, Workable, Lever, and RemoteOK.
    - **🔄 Loop Batches (5) (Split In Batches):** Manages the flow of source configurations in batches of 5 to prevent API rate limiting or timeouts during the HTTP request phase.

### 2.2 Request & Normalization
**Overview:** Translates company configurations into actual API calls and parses the resulting raw data.

- **Nodes Involved:** `🔧 Prepare Request`, `🌐 HTTP Request`, `🔍 Parse + Enrich + Filter1`
- **Node Details:**
    - **🔧 Prepare Request (Code):** 
        - **Technical Role:** URL and Header Constructor.
        - **Logic:** Maps the `ats` type to a specific API endpoint. Implements pagination logic for *SmartRecruiters* (offset-based) and *Himalayas* (page-based).
        - **Outputs:** A list of URLs and corresponding HTTP headers (User-Agent, Accept) required for each source.
    - **🌐 HTTP Request (HTTP Request):** 
        - **Configuration:** Uses dynamic expressions for URL and headers. Set to "Continue on Fail" to ensure one offline API doesn't stop the entire aggregation.
        - **Batching:** Configured with a batch size of 5 for stability.
    - **🔍 Parse + Enrich + Filter1 (Code):** 
        - **Technical Role:** The "Normalization Engine."
        - **Key Logic:** 
            - **Salary Parsing:** Converts various formats (numeric, strings, nested objects) into a human-readable string (e.g., "$100k–$120k/yr USD").
            - **Location Mapping:** Uses ISO country maps and city lists (e.g., `INDIA_CITIES`) to standardize "Bengaluru" to "India" or "Remote" to "Global".
            - **Domain Filtering:** Only keeps jobs matching `ALLOWED_DOMAINS` (e.g., Product Manager, Full Stack Engineer, Data Analyst).
            - **Deduplication:** Generates a `job_hash` based on ATS, Company, Title, and Location to identify unique postings.
            - **HTML/JSON-LD Parsing:** Specifically extracts `@type: JobPosting` data from raw HTML pages.

### 2.3 Storage & Output
**Overview:** Commits the cleaned data to external storage systems.

- **Nodes Involved:** `💾 Upsert to Supabase`, `📊 Write to Google Sheet`
- **Node Details:**
    - **💾 Upsert to Supabase (Postgres):** 
        - **Operation:** Upsert.
        - **Matching Column:** `job_hash`. This ensures that if a job is updated on the ATS, it is updated in the DB rather than duplicated.
        - **Data Mapped:** Title, Company, Location, Country, Work Mode, Salary, etc.
    - **📊 Write to Google Sheet (Google Sheets):** 
        - **Operation:** Append or Update.
        - **Matching Column:** `job_hash`.
        - **Connection:** Syncs the same standardized fields as the Supabase node.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| ⏰ Daily 8AM IST | Schedule Trigger | Execution Trigger | - | 📋 Company List | Source Discovery & Trigger |
| 📋 Company List | Code | Source Configuration | ⏰ Daily 8AM IST | 🔄 Loop Batches (5) | Source Discovery & Trigger |
| 🔄 Loop Batches (5) | Split In Batches | Flow Control/Batching | 📋 Company List, 📊 Write to Google Sheet | 🔧 Prepare Request | Source Discovery & Trigger |
| 🔧 Prepare Request | Code | URL/Header Generator | 🔄 Loop Batches (5) | 🌐 HTTP Request | Request & Normalization |
| 🌐 HTTP Request | HTTP Request | Data Acquisition | 🔧 Prepare Request | 🔍 Parse + Enrich + Filter1 | Request & Normalization |
| 🔍 Parse + Enrich + Filter1 | Code | Normalization & Filtering | 🌐 HTTP Request | 💾 Upsert to Supabase | Request & Normalization |
| 💾 Upsert to Supabase | Postgres | DB Synchronization | 🔍 Parse + Enrich + Filter1 | 📊 Write to Google Sheet | Storage & Output |
| 📊 Write to Google Sheet | Google Sheets | Sheet Synchronization | 💾 Upsert to Supabase | 🔄 Loop Batches (5) | Storage & Output |

---

## 4. Reproducing the Workflow from Scratch

### Step 1: Trigger and Configuration
1. Add a **Schedule Trigger** node set to 8 AM daily.
2. Add a **Code Node** (`📋 Company List`). Paste the JavaScript array containing the source configurations (ATS, Slug, Company).
3. Add a **Split In Batches** node. Set `Batch Size` to 5. Connect the Code node to this.

### Step 2: Request Pipeline
4. Add a **Code Node** (`🔧 Prepare Request`). Implement logic to map `ats` types to URLs (e.g., Greenhouse API, Lever API). Include the pagination loops for SmartRecruiters and Himalayas.
5. Add an **HTTP Request** node. 
    - URL: `{{ $json._url }}`.
    - Headers: Set to "JSON" and use `{{ JSON.stringify($json._headers) }}`.
    - Set "Error Handling" to **Continue (Regular Output)**.

### Step 3: Normalization Engine
6. Add a **Code Node** (`🔍 Parse + Enrich + Filter1`). Implement the following JS logic:
    - Create a `parseSalary` function to handle mixed currency and numeric types.
    - Create a `makeHash` function to generate a unique ID based on `ats|company|title|location`.
    - Define an `ALLOWED_DOMAINS` array to filter job titles.
    - Map raw response fields (e.g., `j.absolute_url` for Greenhouse) to standard fields (`apply_url`).

### Step 4: Destination Setup
7. Add a **Postgres Node** (Supabase).
    - Operation: **Upsert**.
    - Column Matching: `job_hash`.
    - Map all standardized fields from the previous Code node.
    - *Credential:* Provide Supabase Connection details.
8. Add a **Google Sheets Node**.
    - Operation: **Append or Update**.
    - Column Matching: `job_hash`.
    - Document ID: Provide your target Sheet ID.
    - *Credential:* Provide Google OAuth2 credentials.
9. Connect the Google Sheets node back to the **Split In Batches** node to complete the loop.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Deduplication Strategy** | The `job_hash` is critical. It prevents the database from filling with duplicate entries when the same job is found across multiple pages or subsequent daily runs. |
| **Domain Filtering** | To change the types of jobs collected, modify the `ALLOWED_DOMAINS` array inside the `Parse + Enrich + Filter1` node. |
| **Rate Limiting** | Batch size is set to 5 to avoid `429 Too Many Requests` errors from strict ATS APIs. |