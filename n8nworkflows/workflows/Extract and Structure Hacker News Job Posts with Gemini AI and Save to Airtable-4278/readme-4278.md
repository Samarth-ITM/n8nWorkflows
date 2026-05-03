Extract and Structure Hacker News Job Posts with Gemini AI and Save to Airtable

https://n8nworkflows.xyz/workflows/extract-and-structure-hacker-news-job-posts-with-gemini-ai-and-save-to-airtable-4278


# Extract and Structure Hacker News Job Posts with Gemini AI and Save to Airtable

### 1. Workflow Overview

This workflow automates the extraction, structuring, and storage of job posts from Hacker News "Ask HN: Who is hiring" submissions. It is designed for users who want to collect and organize up-to-date job listings from Hacker News efficiently, leveraging AI-powered text processing and structured output parsing to transform raw job post data into well-defined fields. The processed job data is then saved into an Airtable base for easy management and further use.

Logical blocks in this workflow are:

- **1.1 Scheduled Trigger & Data Retrieval:** Periodically queries Hacker News search API to find recent "Who is hiring" posts.
- **1.2 Post Filtering & Fetching Details:** Filters posts within the last 30 days and retrieves detailed data for main posts and their child job posts.
- **1.3 Text Extraction & Cleaning:** Extracts raw text from job posts and cleans it to remove HTML entities, excessive whitespace, and other artifacts.
- **1.4 AI Processing & Structured Parsing:** Uses Google Gemini AI chat model and a structured output parser to convert cleaned text into a JSON schema representing job details.
- **1.5 Data Post-processing & Storage:** Converts salary info to numeric form and writes the enriched job data into Airtable.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Data Retrieval

**Overview:**  
This block triggers the workflow every 2 minutes and queries the Algolia-powered Hacker News search API for "Ask HN: Who is hiring" posts, retrieving up to 30 results per query.

**Nodes Involved:**  
- Schedule Trigger  
- Search for Who is hiring posts  
- Split Out

**Node Details:**

- **Schedule Trigger**  
  - *Type & Role:* Time-based trigger node initiating workflow every 2 minutes.  
  - *Configuration:* Interval set to 2 minutes.  
  - *Input/Output:* No input; outputs trigger event.  
  - *Failure Cases:* Possible scheduling misfires if n8n instance is down or paused.

- **Search for Who is hiring posts**  
  - *Type & Role:* HTTP Request node querying Algolia API for posts with title "Ask HN: Who is hiring".  
  - *Configuration:* POST method to Algolia endpoint with JSON body specifying query, pagination, filters, and headers for authentication and user agent. Uses HTTP Header Authentication credential for Algolia app.  
  - *Key Expressions:* Query string is `"Ask HN: Who is hiring"`.  
  - *Input/Output:* Input from Schedule Trigger; outputs JSON with array field `hits`.  
  - *Failure Cases:* API authentication errors, network timeouts, API rate limits.

- **Split Out**  
  - *Type & Role:* Splits the array `hits` from the previous node into individual items for processing each post separately.  
  - *Configuration:* Field to split out is `hits`.  
  - *Input/Output:* Input from Search for Who is hiring posts; outputs one item per post.  
  - *Failure Cases:* Empty or malformed `hits` field.

---

#### 2.2 Post Filtering & Fetching Details

**Overview:**  
Filters posts to only those created within the last 30 days, fetches main post details from Hacker News API, and splits out child job postings for further processing.

**Nodes Involved:**  
- Get relevant data  
- Get latest post  
- HN API: Get Main Post  
- Split out children (jobs)  
- HI API: Get the individual job post

**Node Details:**

- **Get relevant data**  
  - *Type & Role:* Set node extracting key fields (`title`, `createdAt`, `updatedAt`, `storyId`) from split posts for filtering.  
  - *Configuration:* Assigns these fields from the JSON input.  
  - *Input/Output:* Input from Split Out; outputs simplified JSON item.  
  - *Failure Cases:* Missing expected fields in input JSON.

- **Get latest post**  
  - *Type & Role:* Filter node to allow only posts created within the past 30 days.  
  - *Configuration:* Condition checks if `createdAt` is after current date minus 30 days.  
  - *Input/Output:* Input from Get relevant data; outputs filtered posts.  
  - *Failure Cases:* Date parsing errors if `createdAt` format is unexpected.

- **HN API: Get Main Post**  
  - *Type & Role:* HTTP Request node fetching full post details from Hacker News API using `storyId`.  
  - *Configuration:* GET request to `https://hacker-news.firebaseio.com/v0/item/{{ $json.storyId }}.json?print=pretty`.  
  - *Input/Output:* Input from Get latest post; outputs detailed post JSON including child IDs (`kids`).  
  - *Failure Cases:* API downtime, invalid storyId, rate limits.

- **Split out children (jobs)**  
  - *Type & Role:* Splits `kids` array (child comment IDs) into individual job posts for detailed retrieval.  
  - *Configuration:* Field to split out is `kids`.  
  - *Input/Output:* Input from HN API: Get Main Post; outputs individual child IDs.  
  - *Failure Cases:* Missing or empty `kids` field.

- **HI API: Get the individual job post**  
  - *Type & Role:* HTTP Request node fetching each child job post detail by ID from Hacker News API.  
  - *Configuration:* GET request to `https://hacker-news.firebaseio.com/v0/item/{{ $json.kids }}.json?print=pretty`.  
  - *Input/Output:* Input from Split out children (jobs); outputs job post JSON including `text`.  
  - *Failure Cases:* Invalid child ID, API errors.

---

#### 2.3 Text Extraction & Cleaning

**Overview:**  
Extracts the raw post text from each job post and performs cleaning to remove HTML entities, extra spaces, and format URLs clearly for downstream AI processing.

**Nodes Involved:**  
- Extract text  
- Clean text  
- Limit for testing (optional)

**Node Details:**

- **Extract text**  
  - *Type & Role:* Set node to isolate the `text` field from the job post JSON.  
  - *Configuration:* Assigns `text` field from input JSON.  
  - *Input/Output:* Input from HI API: Get the individual job post; outputs JSON with `text`.  
  - *Failure Cases:* Missing `text` field in some posts.

- **Clean text**  
  - *Type & Role:* Code node running JavaScript to sanitize and normalize raw text from job posts.  
  - *Configuration:* Custom JS code replaces HTML entities (`&#x2F;`, `&#x27;`, etc.) with characters, removes HTML tags, normalizes whitespace, separates URLs with newlines, and trims. Also handles errors gracefully, returning partial cleaning or error info per item.  
  - *Input/Output:* Input from Extract text; outputs JSON with `cleaned_text` field or error info.  
  - *Failure Cases:* Unexpected input structure, runtime errors inside JS; code is defensive.

- **Limit for testing (optional)**  
  - *Type & Role:* Limit node restricting number of processed items to 5 for testing or development.  
  - *Configuration:* Max items set to 5.  
  - *Input/Output:* Input from Clean text; outputs limited subset.  
  - *Failure Cases:* None critical; can be removed in production.

---

#### 2.4 AI Processing & Structured Parsing

**Overview:**  
This block uses Google Gemini AI to transform cleaned job post text into structured JSON data following a predefined schema. The output parser validates and enforces the schema.

**Nodes Involved:**  
- Google Gemini Chat Model  
- Structured Output Parser  
- Trun into structured data (chainLlm)

**Node Details:**

- **Google Gemini Chat Model**  
  - *Type & Role:* AI language model node calling Google Gemini (PaLM) for chat-based text extraction.  
  - *Configuration:* Uses model `models/gemini-2.0-flash`. Sends cleaned text as prompt to extract JSON data.  
  - *Credentials:* Google Palm API credentials required.  
  - *Input/Output:* Input from Limit for testing (optional); outputs AI-generated response.  
  - *Failure Cases:* API authentication issues, rate limits, model downtime, malformed prompts.

- **Structured Output Parser**  
  - *Type & Role:* Langchain output parser node validating AI output against manual JSON schema for job posts.  
  - *Configuration:* Schema includes fields such as `company`, `title`, `location`, `type` (enum), `work_location` (enum), `salary`, `description`, `apply_url`, `company_url`.  
  - *Input/Output:* Receives AI output and parses it into structured JSON for downstream use.  
  - *Failure Cases:* Schema mismatch, invalid AI output, parsing errors.

- **Trun into structured data**  
  - *Type & Role:* Chain LLM node integrating AI output parser with Gemini chat model results.  
  - *Configuration:* Uses input schema and AI messages to define extraction prompt. Has output parser enabled.  
  - *Input/Output:* Input from Google Gemini Chat Model and Structured Output Parser outputs; outputs structured job data.  
  - *Failure Cases:* Combined errors from AI or parser nodes.

---

#### 2.5 Data Post-processing & Storage

**Overview:**  
The final block converts salary strings into numeric values for easier analysis and saves the structured job data into Airtable for persistent storage and management.

**Nodes Involved:**  
- Code  
- Write results to airtable

**Node Details:**

- **Code**  
  - *Type & Role:* JavaScript function node converting salary textual ranges into average numeric values.  
  - *Configuration:* Parses salary strings with potential "$", "k" suffixes, and ranges separated by dashes, returning average or single numeric value. Adds `salary_numeric` field to JSON.  
  - *Input/Output:* Input from Trun into structured data; outputs enriched JSON with numeric salary.  
  - *Failure Cases:* Unexpected salary formats, missing values result in null.

- **Write results to airtable**  
  - *Type & Role:* Airtable node to create new records in a specific Airtable base and table.  
  - *Configuration:* Uses Airtable Personal Access Token credential. Maps fields from structured AI output to Airtable columns: Type, Title, Salary, Company, Location, Apply_url, Description, company_url, work_location.  
  - *Input/Output:* Input from Code node; outputs Airtable write results.  
  - *Failure Cases:* Credential errors, API rate limits, Airtable schema mismatches.

---

### 3. Summary Table

| Node Name                     | Node Type                                | Functional Role                        | Input Node(s)                   | Output Node(s)                  | Sticky Note                                                                                                                                                            |
|-------------------------------|-----------------------------------------|-------------------------------------|--------------------------------|--------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger               | Schedule Trigger                        | Starts workflow every 2 minutes     | -                              | Search for Who is hiring posts  |                                                                                                                                                                      |
| Search for Who is hiring posts | HTTP Request                           | Queries Hacker News Algolia search  | Schedule Trigger               | Split Out                      | Uses Algolia API with HTTP Header Auth for search of "Ask HN: Who is hiring" posts                                                                                   |
| Split Out                     | Split Out                              | Splits hits array into individual posts | Search for Who is hiring posts | Get relevant data               |                                                                                                                                                                      |
| Get relevant data             | Set                                    | Extracts fields for filtering       | Split Out                     | Get latest post                |                                                                                                                                                                      |
| Get latest post               | Filter                                 | Filters posts created in last 30 days | Get relevant data             | HN API: Get Main Post          |                                                                                                                                                                      |
| HN API: Get Main Post         | HTTP Request                           | Fetches full Hacker News post data  | Get latest post               | Split out children (jobs)       |                                                                                                                                                                      |
| Split out children (jobs)     | Split Out                              | Splits child comment IDs into job posts | HN API: Get Main Post         | HI API: Get the individual job post |                                                                                                                                                                      |
| HI API: Get the individual job post | HTTP Request                    | Fetches individual job post details | Split out children (jobs)     | Extract text                  |                                                                                                                                                                      |
| Extract text                 | Set                                    | Extracts raw text from job post     | HI API: Get the individual job post | Clean text                   |                                                                                                                                                                      |
| Clean text                   | Code                                   | Cleans HTML entities and formatting | Extract text                  | Limit for testing (optional)   |                                                                                                                                                                      |
| Limit for testing (optional) | Limit                                  | Limits data to 5 items for testing  | Clean text                    | Google Gemini Chat Model       | Optional node to speed up development/testing                                                                                                                        |
| Google Gemini Chat Model      | Langchain LM Chat                      | AI processes cleaned text to JSON   | Limit for testing (optional)  | Trun into structured data       | Requires Google Palm API credentials                                                                                                                                  |
| Structured Output Parser      | Langchain Output Parser                | Validates AI output against schema  | Google Gemini Chat Model       | Trun into structured data       | Output schema defines job post fields including enums                                                                                                                |
| Trun into structured data    | Langchain Chain LLM                    | Combines AI model and parser to extract structured data | Google Gemini Chat Model, Structured Output Parser | Code                     |                                                                                                                                                                      |
| Code                         | Code                                   | Converts salary strings to numeric  | Trun into structured data     | Write results to airtable       |                                                                                                                                                                      |
| Write results to airtable    | Airtable                               | Saves structured job data to Airtable | Code                         | -                              | Uses Airtable Personal Access Token credential; maps job fields into Airtable columns                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set to run every 2 minutes.

2. **Create HTTP Request Node "Search for Who is hiring posts"**  
   - Method: POST  
   - URL: `https://uj5wyc0l7x-dsn.algolia.net/1/indexes/Item_dev_sort_date/query`  
   - Add HTTP Header Auth credential with Algolia app credentials.  
   - Set JSON body with query `"Ask HN: Who is hiring"`, pagination, filters, and headers as per Algolia API requirements.

3. **Create Split Out Node "Split Out"**  
   - Field to split out: `hits`  
   - Connect output from "Search for Who is hiring posts".

4. **Create Set Node "Get relevant data"**  
   - Assign fields: `title` = `$json.title`, `createdAt` = `$json.created_at`, `updatedAt` = `$json.updated_at`, `storyId` = `$json.story_id`  
   - Connect output from "Split Out".

5. **Create Filter Node "Get latest post"**  
   - Condition: `createdAt` is after current time minus 30 days  
   - Connect output from "Get relevant data".

6. **Create HTTP Request Node "HN API: Get Main Post"**  
   - Method: GET  
   - URL: `https://hacker-news.firebaseio.com/v0/item/{{ $json.storyId }}.json?print=pretty`  
   - Connect output from "Get latest post".

7. **Create Split Out Node "Split out children (jobs)"**  
   - Field to split out: `kids`  
   - Connect output from "HN API: Get Main Post".

8. **Create HTTP Request Node "HI API: Get the individual job post"**  
   - Method: GET  
   - URL: `https://hacker-news.firebaseio.com/v0/item/{{ $json.kids }}.json?print=pretty`  
   - Connect output from "Split out children (jobs)".

9. **Create Set Node "Extract text"**  
   - Assign `text` = `$json.text`  
   - Connect output from "HI API: Get the individual job post".

10. **Create Code Node "Clean text"**  
    - Paste provided JavaScript code that cleans HTML entities, replaces special characters, and normalizes whitespace and URLs.  
    - Connect output from "Extract text".

11. **Create Limit Node "Limit for testing (optional)"**  
    - Set max items to 5 (optional, for testing)  
    - Connect output from "Clean text".

12. **Create Langchain LM Chat Node "Google Gemini Chat Model"**  
    - Model: `models/gemini-2.0-flash`  
    - Connect output from "Limit for testing (optional)".  
    - Add Google Palm API credentials.

13. **Create Langchain Output Parser Node "Structured Output Parser"**  
    - Set schema manually with JSON schema defining job fields (`company`, `title`, `location`, `type`, `work_location`, `salary`, `description`, `apply_url`, `company_url`).  
    - Connect output from "Google Gemini Chat Model" as AI output parser.

14. **Create Langchain Chain LLM Node "Trun into structured data"**  
    - Use cleaned text as prompt input.  
    - Enable output parser referencing the "Structured Output Parser".  
    - Connect outputs from both "Google Gemini Chat Model" and "Structured Output Parser".

15. **Create Code Node "Code"**  
    - Paste provided JavaScript to convert salary string ranges to numeric values, adding a `salary_numeric` field.  
    - Connect output from "Trun into structured data".

16. **Create Airtable Node "Write results to airtable"**  
    - Operation: Create record  
    - Base: Select your Airtable base (e.g., "HN Who is hiring?")  
    - Table: Select the target table  
    - Map fields from structured AI output to Airtable columns: Type, Title, Salary, Company, Location, Apply_url, Description, company_url, work_location  
    - Use Airtable Personal Access Token credential.  
    - Connect output from "Code".

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                           |
|---------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| The workflow uses Google Gemini (PaLM) API for AI text extraction and requires Google Palm API credentials.   | https://cloud.google.com/vertex-ai/docs/generative-ai/learn/vertex-ai-generative-ai-overview              |
| Airtable Personal Access Token is used for authentication; ensure token has write permissions on target base. | https://support.airtable.com/docs/generating-an-api-key                                                    |
| Workflow processes only posts created in the last 30 days to maintain relevancy.                              | Date filter is configurable in "Get latest post" filter node.                                            |
| The cleaning code is defensive and logs errors without stopping workflow; useful for handling malformed posts. |                                                                                                          |
| Limiting processed items during development helps speed up iteration; remove Limit node in production.        |                                                                                                          |
| Hacker News Firebase API is used to fetch individual posts and comments, which may occasionally be slow or rate limited. | https://github.com/HackerNews/API                                                                         |
| Algolia API is used to search Hacker News posts efficiently.                                                  | https://hn.algolia.com/api                                                                                 |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly follows applicable content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.