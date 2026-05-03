Remote Job Updates Pipeline with RemoteOK, Airtable, and Telegram

https://n8nworkflows.xyz/workflows/remote-job-updates-pipeline-with-remoteok--airtable--and-telegram-4383


# Remote Job Updates Pipeline with RemoteOK, Airtable, and Telegram

### 1. Workflow Overview

This workflow automates the fetching, processing, storage, and notification of remote job listings sourced from RemoteOK’s public API. It is designed for users or organizations who want to maintain an updated, clean, and formatted list of remote job postings, store them in Airtable for further management or analysis, and optionally broadcast new listings via Telegram.

The workflow is divided into the following logical blocks:

- **1.1 Schedule Trigger:** Periodically triggers the workflow to fetch fresh job data.
- **1.2 Data Retrieval:** Sends an HTTP GET request to RemoteOK API to retrieve the latest remote job listings.
- **1.3 Data Cleaning & Filtering:** Filters out metadata, extracts relevant job fields, and sanitizes text content to remove HTML and special characters.
- **1.4 Data Parsing & Salary Formatting:** Parses cleaned JSON strings safely, creates human-readable salary strings, and adds diagnostic flags.
- **1.5 Data Upsertion in Airtable:** Inserts new or updates existing job records in an Airtable base using job IDs as unique keys.
- **1.6 Message Formatting & Notification:** Formats job entries into user-friendly Telegram messages and sends them to a predefined Telegram chat.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule Trigger

- **Overview:**  
  Initiates the workflow execution at regular intervals (default is every minute) to ensure timely updates of remote job listings.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  
  - **Type:** Schedule Trigger  
  - **Role:** Periodic workflow starter  
  - **Configuration:** Default interval trigger (one-minute intervals)  
  - **Input / Output:** No input; outputs a trigger event to "Remote ok" node  
  - **Failures:** Misconfiguration of schedule may lead to no triggers  
  - **Version:** 1.2

---

#### 2.2 Data Retrieval

- **Overview:**  
  Fetches raw job listings data from RemoteOK API as JSON.

- **Nodes Involved:**  
  - Remote ok

- **Node Details:**  
  - **Type:** HTTP Request  
  - **Role:** GET request to `https://remoteok.com/api`  
  - **Configuration:** Response expected in JSON format  
  - **Input / Output:** Input from Schedule Trigger; output raw API JSON data to "Cleaning the received input" node  
  - **Potential Issues:** Network timeouts, API rate limiting, or JSON parse errors from malformed responses  
  - **Version:** 4.2

---

#### 2.3 Data Cleaning & Filtering

- **Overview:**  
  Filters out non-job metadata, extracts relevant job fields, and cleans text data to remove HTML tags, special characters, and unwanted formatting.

- **Nodes Involved:**  
  - Cleaning the received input  
  - Clean text1

- **Node Details:**  
  - **Cleaning the received input**  
    - Type: Code (JavaScript)  
    - Role: Filters out the first metadata item lacking an `id` and extracts key job fields into a clean JSON object with a fixed "source" field ("Remote OK")  
    - Input: Raw API JSON array  
    - Output: List of job objects with standardized fields  
    - Edge cases: Missing or malformed fields handled by explicit mapping  
    - Version: 2  
  - **Clean text1**  
    - Type: Code (JavaScript)  
    - Role: Cleans text fields by replacing HTML entities, removing tags, normalizing whitespace, and isolating URLs on separate lines  
    - Key Expressions: Regex replacements for HTML character entities and tags  
    - Input: Output of previous node  
    - Output: Objects with `cleaned_text` fields or error/warning messages  
    - Potential Failures: Unexpected data structure or regex failure; handled with try/catch  
    - Version: 2

---

#### 2.4 Data Parsing & Salary Formatting

- **Overview:**  
  Parses cleaned JSON strings into structured objects, safely handling errors, and constructs human-readable salary strings from numeric min/max salary data. Additionally, flags jobs with port numbers greater than 5000.

- **Nodes Involved:**  
  - Text-clean  
  - Salary to string  
  - Code5

- **Node Details:**  
  - **Text-clean**  
    - Type: Code (JavaScript)  
    - Role: Attempts to parse `cleaned_text` JSON strings to extract inner `json` objects; returns errors if parsing fails  
    - Input: Cleaned text objects  
    - Output: Parsed JSON objects or error reports  
    - Edge cases: Malformed JSON strings handled by returning error info instead of failing the workflow  
    - Version: 2  
  - **Salary to string**  
    - Type: Code (JavaScript)  
    - Role: Converts numeric salary ranges into readable string formats like "From 50000", "Up to 70000", or "50000 - 70000"  
    - Input: Parsed job objects  
    - Output: Job objects augmented with `salary_string` field  
    - Edge cases: Handles missing or partial salary data gracefully  
    - Version: 2  
  - **Code5**  
    - Type: Code (JavaScript)  
    - Role: Adds a boolean flag `high-port` set to true if the job's `data.port` field is greater than 5000 (diagnostic logic)  
    - Input: Jobs with salary strings  
    - Output: Jobs with possible additional flag  
    - Edge cases: Jobs missing `data.port` are unaffected  
    - Version: 2

---

#### 2.5 Data Upsertion in Airtable

- **Overview:**  
  Upserts job listings into a defined Airtable base and table, using the job ID as the unique key to avoid duplicates.

- **Nodes Involved:**  
  - RemoteOK Jobs

- **Node Details:**  
  - Type: Airtable node  
  - Role: Upsert operation to insert or update records based on job ID  
  - Configuration:  
    - Base ID: `appzlt8d6rIix61J9`  
    - Table ID: `tblVWkqYX387ikg7e`  
    - Mapping of job fields to Airtable columns (id, url, logo, tags, source, company, location, position, salary_max, salary_min, description)  
    - Matching column: id  
    - Typecast enabled to ensure field type consistency  
  - Credentials: Airtable Personal Access Token  
  - Input: Jobs with processed fields and flags  
  - Output: Upsert response (not used further downstream)  
  - Failures: Authentication errors, API rate limits, field mapping errors  
  - Version: 2.1

---

#### 2.6 Message Formatting & Notification

- **Overview:**  
  Converts the Airtable job records into formatted strings suitable for Telegram messages and sends the messages to a specified Telegram chat.

- **Nodes Involved:**  
  - Table to a single message  
  - Telegram1

- **Node Details:**  
  - **Table to a single message**  
    - Type: Code (JavaScript)  
    - Role: Formats each job into a multi-line message with job title, company, location, salary range, truncated description, application URL, and source site  
    - Input: Job records array from Airtable (fields extracted from `json.fields`)  
    - Output: Array of objects each containing a `message` string  
    - Edge cases: Defaults to "N/A" or "No Title" for missing fields, truncates descriptions to 1000 characters  
    - Version: 2  
  - **Telegram1**  
    - Type: Telegram node  
    - Role: Sends the formatted message string to a Telegram chat using a bot  
    - Configuration:  
      - Chat ID: 858663295  
      - Text: Dynamic from `message` field  
      - Credentials: Telegram Bot API credentials configured for "Telegram account 2"  
    - Input: Formatted message objects  
    - Output: Telegram API response  
    - Failures: Invalid token, chat ID errors, message length limits, network errors  
    - Version: 1.2

---

### 3. Summary Table

| Node Name                 | Node Type          | Functional Role                            | Input Node(s)         | Output Node(s)            | Sticky Note                                                                                                                |
|---------------------------|--------------------|-------------------------------------------|-----------------------|---------------------------|----------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger          | Schedule Trigger   | Periodically triggers the workflow        | -                     | Remote ok                 | ## Schedule Trigger **Purpose**: Triggers the workflow at regular intervals to fetch the latest job listings.              |
| Remote ok                | HTTP Request       | Fetches job listings from RemoteOK API    | Schedule Trigger       | Cleaning the received input | ## Remote ok (HTTP Request) **Purpose**: Sends a GET request to https://remoteok.com/api to retrieve job listings.          |
| Cleaning the received input | Code               | Filters metadata, extracts relevant fields | Remote ok             | Clean text1               | ## Cleaning of texts **Purpose**: Cleans HTML tags and special characters from the job descriptions for better readability. |
| Clean text1              | Code               | Cleans text fields removing HTML & special chars | Cleaning the received input | Text-clean                | (See Cleaning of texts sticky note covering multiple nodes)                                                                 |
| Text-clean               | Code               | Parses cleaned JSON safely, handles errors | Clean text1            | Salary to string          | (See Cleaning of texts sticky note covering multiple nodes)                                                                 |
| Salary to string          | Code               | Creates human-readable salary strings     | Text-clean             | Code5                     | (See Cleaning of texts sticky note covering multiple nodes)                                                                 |
| Code5                    | Code               | Flags entries with high port numbers      | Salary to string       | RemoteOK Jobs             | (See Cleaning of texts sticky note covering multiple nodes)                                                                 |
| RemoteOK Jobs            | Airtable           | Upserts job data into Airtable             | Code5                  | Table to a single message  | (See Cleaning of texts sticky note covering multiple nodes)                                                                 |
| Table to a single message | Code               | Formats jobs into Telegram-friendly messages | RemoteOK Jobs          | Telegram1                 | (See Cleaning of texts sticky note covering multiple nodes)                                                                 |
| Telegram1                | Telegram           | Sends formatted messages to Telegram chat | Table to a single message | -                         | (See Cleaning of texts sticky note covering multiple nodes)                                                                 |
| Sticky Note              | Sticky Note        | Overview and workflow purpose              | -                      | -                         | ## Overview Purpose: This workflow fetches remote job listings from RemoteOK, cleans and formats the data, stores it in Airtable, and optionally sends a message via Telegram. |
| Sticky Note1             | Sticky Note        | Schedule trigger explanation                | -                      | -                         | ## Schedule Trigger **Purpose**: Triggers the workflow at regular intervals to fetch the latest job listings.              |
| Sticky Note2             | Sticky Note        | RemoteOK HTTP request explanation           | -                      | -                         | ## Remote ok (HTTP Request) **Purpose**: Sends a GET request to https://remoteok.com/api to retrieve job listings.          |
| Sticky Note3             | Sticky Note        | Explains cleaning, Airtable upsert & Telegram | -                      | -                         | ## Cleaning of texts **Purpose**: Cleans HTML tags and special characters from the job descriptions for better readability. Attempts to parse cleaned job description JSON safely, handling errors gracefully if parsing fails. Generates a human-readable salary string from the salary_min and salary_max fields. Adds a high-port flag to any entry where the port number is greater than 5000 (optional diagnostic/extra logic). Upserts job data into an Airtable table using the job ID as the unique identifier. Purpose: Formats job data into a Telegram-friendly message string. Purpose: Sends the formatted message to a specific Telegram chat using a bot. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Configuration: Default interval (every minute)  
   - Purpose: To start the workflow periodically.

2. **Create HTTP Request Node ("Remote ok")**  
   - Type: HTTP Request  
   - URL: `https://remoteok.com/api`  
   - Method: GET  
   - Response Format: JSON  
   - Connect Schedule Trigger → Remote ok

3. **Create Code Node ("Cleaning the received input")**  
   - Type: Code  
   - JavaScript: Filter out metadata, map fields `id`, `company`, `position`, etc., add `"source": "Remote OK"`  
   - Connect Remote ok → Cleaning the received input

4. **Create Code Node ("Clean text1")**  
   - Type: Code  
   - JavaScript: Clean texts by replacing HTML entities, tags, whitespace normalization, isolate URLs, trim  
   - Connect Cleaning the received input → Clean text1

5. **Create Code Node ("Text-clean")**  
   - Type: Code  
   - JavaScript: Parse cleaned JSON strings safely, return parsed `json` or error info  
   - Connect Clean text1 → Text-clean

6. **Create Code Node ("Salary to string")**  
   - Type: Code  
   - JavaScript: Generate readable salary strings from `salary_min` and `salary_max` fields  
   - Connect Text-clean → Salary to string

7. **Create Code Node ("Code5")**  
   - Type: Code  
   - JavaScript: Add `high-port` flag if `data.port > 5000`  
   - Connect Salary to string → Code5

8. **Create Airtable Node ("RemoteOK Jobs")**  
   - Type: Airtable  
   - Operation: Upsert  
   - Base ID: `appzlt8d6rIix61J9`  
   - Table ID: `tblVWkqYX387ikg7e`  
   - Match column: `id`  
   - Map fields: id, url, logo, tags, source, company, location, position, salary_max, salary_min, description  
   - Enable typecast  
   - Credentials: Airtable Personal Access Token  
   - Connect Code5 → RemoteOK Jobs

9. **Create Code Node ("Table to a single message")**  
   - Type: Code  
   - JavaScript: Format each job into a multi-line message with position, company, location, salary, description (truncated), URL, and source  
   - Input: Airtable record fields from RemoteOK Jobs output  
   - Connect RemoteOK Jobs → Table to a single message

10. **Create Telegram Node ("Telegram1")**  
    - Type: Telegram  
    - Chat ID: `858663295`  
    - Text: `={{ $json.message }}` (dynamic message)  
    - Credentials: Telegram Bot API token (configured as "Telegram account 2")  
    - Connect Table to a single message → Telegram1

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                 |
|----------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow purpose: Fetches remote job listings from RemoteOK, cleans and formats the data, stores in Airtable, and optionally sends Telegram notifications. | Overview sticky note                                                                              |
| Schedule Trigger node triggers execution at regular intervals to keep data fresh.                                    | Sticky Note1                                                                                   |
| RemoteOK API endpoint: https://remoteok.com/api, returns JSON array with job listings and metadata as first item.     | Sticky Note2                                                                                   |
| Cleaning includes HTML entity replacement, tag removal, whitespace normalization, and error handling for text parsing. | Sticky Note3                                                                                   |
| Airtable base and table IDs are specific to this project; credentials need to be set accordingly.                     | Airtable node configuration                                                                     |
| Telegram node requires bot API token with permissions to post in the specified chat.                                 | Telegram node configuration                                                                    |
| Text description truncation to 1000 characters ensures message length fits Telegram limits.                          | Formatting code node                                                                             |

---

**Disclaimer:** The provided text is exclusively extracted from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected material. All data processed is legal and publicly accessible.