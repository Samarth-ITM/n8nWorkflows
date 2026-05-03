Hacker News Job Listing Scraper and Parser

https://n8nworkflows.xyz/workflows/hacker-news-job-listing-scraper-and-parser-2924


# Hacker News Job Listing Scraper and Parser

### 1. Workflow Overview

This workflow automates the scraping and parsing of the monthly "Who is Hiring" thread from Hacker News (HN). It targets job seekers, recruiters, and analysts interested in tech job market trends by transforming unstructured job listing comments into structured, analyzable data.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Initial Search**: Triggering the workflow manually and querying the Algolia-powered HN search API to find recent "Ask HN: Who is hiring?" posts.
- **1.2 Filtering and Main Post Retrieval**: Filtering posts to the last 30 days and retrieving the full main post data from the official HN API.
- **1.3 Job Listings Extraction**: Splitting the main post’s child comments (individual job listings) and fetching each job post’s full content.
- **1.4 Text Cleaning and Preparation**: Extracting raw text from job posts and cleaning it to remove HTML entities, excessive whitespace, and formatting inconsistencies.
- **1.5 AI-Powered Structuring**: Using OpenAI GPT-4o-mini to parse cleaned text into a predefined structured JSON schema representing job details.
- **1.6 Output and Storage**: Writing the structured job listing data into an Airtable base for further use.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Initial Search

- **Overview:**  
  This block initiates the workflow manually and performs a search query on the Algolia-powered Hacker News API to find recent "Ask HN: Who is hiring?" posts.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Search for Who is hiring posts (HTTP Request)  
  - Split Out (Split Out)  
  - Sticky Note (Instructional)

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow on user command.  
    - Inputs: None  
    - Outputs: Triggers the next node.  
    - Edge Cases: None.

  - **Search for Who is hiring posts**  
    - Type: HTTP Request  
    - Role: Queries Algolia API with a POST request to search for posts titled exactly "Ask HN: Who is hiring".  
    - Configuration:  
      - URL: Algolia search endpoint  
      - Method: POST  
      - Body: JSON with query `"Ask HN: Who is hiring"` (with quotes for exact match), filters for stories only, pagination, and other Algolia-specific parameters.  
      - Authentication: HTTP Header Auth with Algolia API key.  
      - Headers: Includes user-agent, origin, referer, and Algolia-specific headers.  
    - Inputs: Trigger from manual node.  
    - Outputs: JSON response containing search hits.  
    - Edge Cases: API rate limits, network errors, invalid API key.  
    - Sticky Note: Instructions on how to obtain the cURL command from hn.algolia.com and set up the HTTP node.

  - **Split Out**  
    - Type: Split Out  
    - Role: Splits the array of search hits into individual items for further processing.  
    - Configuration: Splits on the `hits` field from the Algolia response.  
    - Inputs: Output from Search for Who is hiring posts.  
    - Outputs: Individual post objects.  
    - Edge Cases: Empty hits array.

  - **Sticky Note**  
    - Content: Instructions on using hn.algolia.com to get the cURL command and set up API key authentication.

---

#### 1.2 Filtering and Main Post Retrieval

- **Overview:**  
  Filters the posts to only those created within the last 30 days and retrieves the full main post data from the official Hacker News API.

- **Nodes Involved:**  
  - Get relevant data (Set)  
  - Get latest post (Filter)  
  - HN API: Get Main Post (HTTP Request)  
  - Sticky Note1 (Instructional)  
  - Sticky Note3 (Example data)

- **Node Details:**

  - **Get relevant data**  
    - Type: Set  
    - Role: Extracts and assigns key fields (`title`, `createdAt`, `updatedAt`, `storyId`) from each hit for easier processing.  
    - Inputs: Individual post from Split Out.  
    - Outputs: Simplified JSON with relevant metadata.  
    - Edge Cases: Missing fields in input JSON.

  - **Get latest post**  
    - Type: Filter  
    - Role: Filters posts to only those created within the last 30 days based on `createdAt` date.  
    - Configuration: Condition checks if `createdAt` is after current date minus 30 days.  
    - Inputs: Output from Get relevant data.  
    - Outputs: Filtered posts.  
    - Edge Cases: Date parsing errors, no posts in last 30 days.

  - **HN API: Get Main Post**  
    - Type: HTTP Request  
    - Role: Retrieves the full main post JSON from Hacker News API using the `storyId`.  
    - Configuration: URL templated with `storyId` parameter.  
    - Inputs: Filtered post from Get latest post.  
    - Outputs: Full main post data including child comment IDs (`kids`).  
    - Edge Cases: API downtime, invalid storyId, network errors.

  - **Sticky Note1**  
    - Content: Links and examples for Hacker News API endpoints for stories and comments.

  - **Sticky Note3**  
    - Content: Example cleaned JSON data for posts with fields like title, createdAt, updatedAt, storyId.

---

#### 1.3 Job Listings Extraction

- **Overview:**  
  Splits the main post’s child comments (which represent individual job listings) and fetches each job post’s full content.

- **Nodes Involved:**  
  - Split out children (jobs) (Split Out)  
  - HI API: Get the individual job post (HTTP Request)

- **Node Details:**

  - **Split out children (jobs)**  
    - Type: Split Out  
    - Role: Splits the `kids` array from the main post into individual job post IDs.  
    - Inputs: Main post JSON from HN API: Get Main Post.  
    - Outputs: Individual job post IDs.  
    - Edge Cases: Missing or empty `kids` array.

  - **HI API: Get the individual job post**  
    - Type: HTTP Request  
    - Role: Retrieves each individual job post comment JSON from Hacker News API using the job post ID.  
    - Configuration: URL templated with `kids` (job post ID).  
    - Inputs: Individual job post ID from Split out children (jobs).  
    - Outputs: Full job post JSON including raw text.  
    - Edge Cases: API errors, missing job post, network issues.

---

#### 1.4 Text Cleaning and Preparation

- **Overview:**  
  Extracts raw text from each job post and cleans it by removing HTML entities, excessive whitespace, and formatting inconsistencies to prepare for AI parsing.

- **Nodes Involved:**  
  - Extract text (Set)  
  - Clean text (Code)  
  - Limit for testing (optional) (Limit)  
  - Sticky Note2 (Instructional)

- **Node Details:**

  - **Extract text**  
    - Type: Set  
    - Role: Extracts the `text` field from the job post JSON into a new field for processing.  
    - Inputs: Job post JSON from HI API: Get the individual job post.  
    - Outputs: JSON with `text` field.  
    - Edge Cases: Missing `text` field.

  - **Clean text**  
    - Type: Code (JavaScript)  
    - Role: Cleans the extracted text by:  
      - Replacing HTML entities (`&#x2F;`, `&#x27;`, etc.)  
      - Removing HTML tags  
      - Normalizing whitespace and line breaks  
      - Formatting URLs on separate lines  
    - Inputs: JSON with `text` field.  
    - Outputs: JSON with `cleaned_text` field.  
    - Edge Cases: Unexpected input structure, partial cleaning fallback, logs errors to console.

  - **Limit for testing (optional)**  
    - Type: Limit  
    - Role: Limits the number of items processed downstream to 5 for testing purposes.  
    - Inputs: Cleaned text JSON.  
    - Outputs: Limited subset of cleaned job posts.  
    - Edge Cases: None.

  - **Sticky Note2**  
    - Content: Explanation of the data structure used for AI output with example JSON schema.

---

#### 1.5 AI-Powered Structuring

- **Overview:**  
  Uses OpenAI GPT-4o-mini to parse the cleaned job listing text into a structured JSON format according to a predefined schema.

- **Nodes Involved:**  
  - OpenAI Chat Model (Langchain LLM Chat)  
  - Structured Output Parser (Langchain Output Parser Structured)

- **Node Details:**

  - **OpenAI Chat Model**  
    - Type: Langchain LLM Chat (OpenAI)  
    - Role: Sends cleaned text to GPT-4o-mini model to extract structured job listing data.  
    - Configuration: Model set to `gpt-4o-mini`, no additional options.  
    - Inputs: Cleaned text from Limit node.  
    - Outputs: Raw AI response with structured data.  
    - Credentials: OpenAI API key required.  
    - Edge Cases: API rate limits, malformed input, model errors.

  - **Structured Output Parser**  
    - Type: Langchain Output Parser Structured  
    - Role: Parses the AI response into a strict JSON schema with fields like `company`, `title`, `location`, `type`, `work_location`, `salary`, `description`, `apply_url`, `company_url`.  
    - Configuration: Manual JSON schema defining expected fields and types.  
    - Inputs: AI response from OpenAI Chat Model.  
    - Outputs: Validated structured JSON data.  
    - Edge Cases: Parsing errors if AI output deviates from schema.

---

#### 1.6 Output and Storage

- **Overview:**  
  Writes the structured job listing data into an Airtable base for storage, analysis, or integration with other tools.

- **Nodes Involved:**  
  - Write results to airtable (Airtable)

- **Node Details:**

  - **Write results to airtable**  
    - Type: Airtable  
    - Role: Creates new records in a specified Airtable base and table with the structured job listing data.  
    - Configuration:  
      - Base and Table selected from existing Airtable account.  
      - Columns mapped explicitly from structured JSON fields (`title`, `company`, `location`, `type`, `salary`, `description`, `apply_url`, `company_url`).  
      - Mapping mode: Define below (manual mapping).  
    - Inputs: Structured JSON data from Trun into structured data node.  
    - Outputs: Confirmation of record creation.  
    - Credentials: Airtable Personal Access Token required.  
    - Edge Cases: API limits, invalid field mapping, network errors.

---

### 3. Summary Table

| Node Name                      | Node Type                          | Functional Role                          | Input Node(s)                  | Output Node(s)                  | Sticky Note                                                                                                                      |
|--------------------------------|----------------------------------|----------------------------------------|-------------------------------|--------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’   | Manual Trigger                   | Starts workflow manually                | None                          | Search for Who is hiring posts  |                                                                                                                                 |
| Search for Who is hiring posts  | HTTP Request                    | Queries Algolia HN API for hiring posts| When clicking ‘Test workflow’ | Split Out                      | Go to https://hn.algolia.com - filter by "Ask HN: Who is hiring?" (important with quotes for full match) ...                    |
| Split Out                      | Split Out                       | Splits Algolia hits array into items   | Search for Who is hiring posts | Get relevant data              |                                                                                                                                 |
| Get relevant data              | Set                            | Extracts key fields from hits           | Split Out                     | Get latest post                | Clean the result example JSON snippet                                                                                           |
| Get latest post                | Filter                         | Filters posts from last 30 days         | Get relevant data             | HN API: Get Main Post          |                                                                                                                                 |
| HN API: Get Main Post          | HTTP Request                   | Retrieves full main post data            | Get latest post               | Split out children (jobs)      | Go to HN API https://github.com/HackerNews/API - example endpoints for story and comment                                         |
| Split out children (jobs)      | Split Out                      | Splits main post’s kids array into jobs | HN API: Get Main Post         | HI API: Get the individual job post |                                                                                                                                 |
| HI API: Get the individual job post | HTTP Request                   | Retrieves individual job post data      | Split out children (jobs)     | Extract text                  |                                                                                                                                 |
| Extract text                  | Set                            | Extracts raw text from job post          | HI API: Get the individual job post | Clean text                  |                                                                                                                                 |
| Clean text                    | Code (JavaScript)               | Cleans and normalizes raw text           | Extract text                  | Limit for testing (optional)   |                                                                                                                                 |
| Limit for testing (optional)  | Limit                          | Limits number of items for testing       | Clean text                   | Trun into structured data      |                                                                                                                                 |
| OpenAI Chat Model             | Langchain LLM Chat (OpenAI)    | Parses cleaned text into structured data | Limit for testing (optional)  | Structured Output Parser       | Data structure explained with example JSON                                                                                      |
| Structured Output Parser      | Langchain Output Parser Structured | Validates and parses AI output to JSON schema | OpenAI Chat Model            | Trun into structured data      |                                                                                                                                 |
| Trun into structured data     | Langchain Chain LLM            | Finalizes structured data output         | Structured Output Parser      | Write results to airtable      |                                                                                                                                 |
| Write results to airtable     | Airtable                       | Writes structured job listings to Airtable | Trun into structured data     | None                         |                                                                                                                                 |
| Sticky Note                   | Sticky Note                    | Instructional notes                      | None                         | None                         | See detailed notes in respective blocks                                                                                         |
| Sticky Note1                  | Sticky Note                    | Instructional notes                      | None                         | None                         | See detailed notes in respective blocks                                                                                         |
| Sticky Note2                  | Sticky Note                    | Instructional notes                      | None                         | None                         | See detailed notes in respective blocks                                                                                         |
| Sticky Note3                  | Sticky Note                    | Instructional notes                      | None                         | None                         | See detailed notes in respective blocks                                                                                         |
| Sticky Note4                  | Sticky Note                    | Instructional notes                      | None                         | None                         | Overview and project credits with links                                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: When clicking ‘Test workflow’  
   - Type: Manual Trigger  
   - No parameters needed.

2. **Create HTTP Request Node for Algolia Search**  
   - Name: Search for Who is hiring posts  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://uj5wyc0l7x-dsn.algolia.net/1/indexes/Item_dev_sort_date/query`  
   - Authentication: HTTP Header Auth with Algolia API key (set up credentials).  
   - Headers: Include required headers such as `Accept`, `Origin`, `Referer`, `User-Agent`, and Algolia-specific headers (`x-algolia-agent`, `x-algolia-application-id`).  
   - Body (JSON):  
     ```json
     {
       "query": "\"Ask HN: Who is hiring\"",
       "analyticsTags": ["web"],
       "page": 0,
       "hitsPerPage": 30,
       "minWordSizefor1Typo": 4,
       "minWordSizefor2Typos": 8,
       "advancedSyntax": true,
       "ignorePlurals": false,
       "clickAnalytics": true,
       "minProximity": 7,
       "numericFilters": [],
       "tagFilters": [["story"], []],
       "typoTolerance": "min",
       "queryType": "prefixNone",
       "restrictSearchableAttributes": ["title", "comment_text", "url", "story_text", "author"],
       "getRankingInfo": true
     }
     ```
   - Connect output of Manual Trigger to this node.

3. **Create Split Out Node**  
   - Name: Split Out  
   - Type: Split Out  
   - Field to split out: `hits`  
   - Connect output of Search for Who is hiring posts to this node.

4. **Create Set Node to Extract Relevant Data**  
   - Name: Get relevant data  
   - Type: Set  
   - Assign fields:  
     - `title` = `{{$json.title}}`  
     - `createdAt` = `{{$json.created_at}}`  
     - `updatedAt` = `{{$json.updated_at}}`  
     - `storyId` = `{{$json.story_id}}`  
   - Connect output of Split Out to this node.

5. **Create Filter Node to Get Latest Posts**  
   - Name: Get latest post  
   - Type: Filter  
   - Condition: `createdAt` is after current date minus 30 days (`{{$now.minus({days: 30})}}`)  
   - Connect output of Get relevant data to this node.

6. **Create HTTP Request Node to Get Main Post from HN API**  
   - Name: HN API: Get Main Post  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://hacker-news.firebaseio.com/v0/item/{{$json.storyId}}.json?print=pretty`  
   - Connect output of Get latest post to this node.

7. **Create Split Out Node to Split Children (Job Posts)**  
   - Name: Split out children (jobs)  
   - Type: Split Out  
   - Field to split out: `kids`  
   - Connect output of HN API: Get Main Post to this node.

8. **Create HTTP Request Node to Get Individual Job Post**  
   - Name: HI API: Get the individual job post  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://hacker-news.firebaseio.com/v0/item/{{$json.kids}}.json?print=pretty`  
   - Connect output of Split out children (jobs) to this node.

9. **Create Set Node to Extract Text**  
   - Name: Extract text  
   - Type: Set  
   - Assign field: `text` = `{{$json.text}}`  
   - Connect output of HI API: Get the individual job post to this node.

10. **Create Code Node to Clean Text**  
    - Name: Clean text  
    - Type: Code (JavaScript)  
    - Paste the provided JavaScript code that cleans HTML entities, tags, whitespace, and formats URLs.  
    - Connect output of Extract text to this node.

11. **Create Limit Node (Optional for Testing)**  
    - Name: Limit for testing (optional)  
    - Type: Limit  
    - Max Items: 5  
    - Connect output of Clean text to this node.

12. **Create OpenAI Chat Model Node**  
    - Name: OpenAI Chat Model  
    - Type: Langchain LLM Chat (OpenAI)  
    - Model: `gpt-4o-mini`  
    - Credentials: Set up OpenAI API credentials.  
    - Input: Use cleaned text from Limit node.  
    - Connect output of Limit for testing (optional) to this node.

13. **Create Structured Output Parser Node**  
    - Name: Structured Output Parser  
    - Type: Langchain Output Parser Structured  
    - Schema: Paste the provided JSON schema defining fields like company, title, location, type, salary, description, apply_url, company_url.  
    - Connect AI output from OpenAI Chat Model to this node.

14. **Create Chain LLM Node to Finalize Structured Data**  
    - Name: Trun into structured data  
    - Type: Langchain Chain LLM  
    - Input: Use cleaned text and parsed output.  
    - Connect output of Structured Output Parser to this node.

15. **Create Airtable Node to Write Results**  
    - Name: Write results to airtable  
    - Type: Airtable  
    - Operation: Create  
    - Base: Select or paste Airtable base ID (e.g., `appM2JWvA5AstsGdn`)  
    - Table: Select or paste table ID (e.g., `tblGvcOjqbliwM7AS`)  
    - Map fields from structured JSON output to Airtable columns (`title`, `company`, `location`, `type`, `salary`, `description`, `apply_url`, `company_url`).  
    - Credentials: Set up Airtable Personal Access Token.  
    - Connect output of Trun into structured data to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                          | Context or Link                                                                                         |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Go to https://hn.algolia.com - filter by "Ask HN: Who is hiring?" (important with quotes for full match). Use Chrome Network Tab to find API call and copy as cURL for HTTP node setup. API key must be set manually.                  | Sticky Note on Algolia API setup                                                                       |
| Hacker News API documentation and example endpoints for stories and comments: https://github.com/HackerNews/API                                                                                                                    | Sticky Note1                                                                                           |
| Data structure used for AI output: JSON schema with fields company, title, location, type, salary, description, apply_url, company_url. Feel free to customize.                                                                     | Sticky Note2                                                                                           |
| Example cleaned JSON data for posts with fields like title, createdAt, updatedAt, storyId.                                                                                                                                          | Sticky Note3                                                                                           |
| Overview and project credits: Scraper for monthly HN Who is Hiring post using Algolia Search and Hacker News API. Requires OpenAI account for text structuring and Airtable for storage. Base template link: https://airtable.com/appM2JWvA5AstsGdn/shrAuo78cJt5C2laR | Sticky Note4                                                                                           |

---

This documentation provides a detailed, stepwise understanding of the "Hacker News Job Listing Scraper and Parser" workflow, enabling users and AI agents to reproduce, modify, and troubleshoot the process effectively.