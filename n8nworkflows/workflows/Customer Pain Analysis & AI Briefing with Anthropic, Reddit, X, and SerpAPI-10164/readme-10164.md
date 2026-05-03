Customer Pain Analysis & AI Briefing with Anthropic, Reddit, X, and SerpAPI

https://n8nworkflows.xyz/workflows/customer-pain-analysis---ai-briefing-with-anthropic--reddit--x--and-serpapi-10164


# Customer Pain Analysis & AI Briefing with Anthropic, Reddit, X, and SerpAPI

### 1. Workflow Overview

This workflow, titled **Customer Pain Analysis & AI Briefing with Anthropic, Reddit, X, and SerpAPI**, is designed to automate the collection, analysis, and briefing of customer pain points related to HVAC (Heating, Ventilation, and Air Conditioning) customer service. It aggregates data from multiple sources—Google via SerpAPI, Reddit, and X (formerly Twitter)—to identify prevalent customer complaints and sentiments. The workflow then synthesizes this intelligence into a professional executive summary and sends it via email to the sales or executive team.

The workflow consists of the following logical blocks:

- **1.1 Input Reception**: Captures user input via a form specifying keywords, subreddits, and search queries.
- **1.2 Data Ingestion**: Parallel fetching of raw data from Google (SerpAPI), Reddit, and X (Twitter) APIs.
- **1.3 Data Filtering and Labeling**: Cleans, filters, and standardizes data from each source, adding source labels.
- **1.4 Data Merging**: Combines the cleaned data streams into a unified dataset.
- **1.5 Core Analysis**: Applies custom logic to categorize customer complaints into pain points and assign sentiment scores.
- **1.6 Aggregation and Deduplication**: Removes duplicates and calculates statistics such as frequency counts and average sentiment.
- **1.7 Summary Retrieval and AI Processing**: Retrieves aggregated data, sends it to Anthropic's Claude model for generation of a formatted executive summary.
- **1.8 Final Dispatch and Logging**: Sends the executive summary via Gmail and logs the search parameters and results in Google Sheets.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Captures structured input from a user via a web form to specify search parameters like location, keywords, and targeted subreddits.

**Nodes Involved:**  
- Form

**Node Details:**  

- **Form**  
  - Type: Form Trigger  
  - Configuration:  
    - Form titled "Customer intelligence Briefing"  
    - Fields capturing user's name, email, location, SerpAPI query, X (Twitter) query, and two subreddit names  
  - Inputs: Webhook trigger on form submission  
  - Outputs: JSON with form responses for downstream use  
  - Edge Cases: Missing required fields; invalid email format; user input errors  
  - Sticky Notes: Guidance on query construction and subreddit input validation (Sticky Note)

---

#### 1.2 Data Ingestion

**Overview:**  
Parallel fetching of raw data from three sources: Google (via SerpAPI), Reddit, and X (Twitter). Each source fetches data based on user inputs.

**Nodes Involved:**  
- Search Google (SerpAPI)  
- Search Reddit (Reddit OAuth2 API)  
- Search X (HTTP Request to twitterapi.io)  
- List Creator (splits subreddit input into individual items)

**Node Details:**  

- **Search Google**  
  - Type: SerpAPI Node  
  - Configuration: Queries Google with user-supplied keywords and location (from form)  
  - Credentials: SerpAPI account  
  - Output: Raw search results including organic results  
  - Edge Cases: API limits, invalid queries, location errors  
  - Sticky Note1: Explains the strategic context role of the SERP API data stream  

- **List Creator**  
  - Type: Code  
  - Configuration: Converts the two subreddit strings from form data into individual JSON items for iteration  
  - Input: Form data  
  - Output: Array of subreddit names  
  - Edge Cases: Empty or malformed subreddit names  
  - Sticky Note2: Describes looping structure for Reddit data ingestion  

- **Search Reddit**  
  - Type: Reddit OAuth2 API  
  - Configuration: Iterates over each subreddit from List Creator; fetches up to 50 posts per subreddit  
  - Credentials: Reddit OAuth2 API  
  - Input: Subreddit names from List Creator  
  - Output: Raw Reddit post data  
  - Edge Cases: Subreddit does not exist, API rate limit, private subreddits  

- **Search X**  
  - Type: HTTP Request  
  - Configuration: Calls external API (twitterapi.io) with user-defined Twitter query; uses HTTP header authentication  
  - Credentials: HTTP Header Auth with X/Twitter Demo account  
  - Output: Raw tweets data  
  - Edge Cases: Rate limits, 429 errors, invalid queries  
  - Sticky Note3: Explains use of external API to circumvent Twitter API limits and importance of query filters  

---

#### 1.3 Data Filtering and Labeling

**Overview:**  
Processes raw data from each source to filter out low-quality or irrelevant items, standardizes data structure, and labels each item with its source.

**Nodes Involved:**  
- Filter & Label Google (Code)  
- Filter & Label Reddit (Code)  
- Filter & Label X (Code)

**Node Details:**  

- **Filter & Label Google**  
  - Type: Code  
  - Configuration: Extracts title, snippet, and link from Google organic results; adds source label 'SERP API (Web)'  
  - Input: Search Google node output  
  - Output: Cleaned, labeled data array  
  - Edge Cases: Missing organic results, malformed data  

- **Filter & Label Reddit**  
  - Type: Code  
  - Configuration: Filters posts with score > 5; extracts title, selftext, and URL; labels source as 'Reddit'; combines title and text into full_text  
  - Input: Search Reddit node output  
  - Output: Filtered and labeled Reddit posts  
  - Edge Cases: No posts passing score filter, missing fields  

- **Filter & Label X**  
  - Type: Code  
  - Configuration: Extracts tweets; filters out low engagement (likes <= 5); standardizes structure; labels source as 'Twitter (External API)'  
  - Input: Search X node output  
  - Output: Filtered and labeled tweets  
  - Edge Cases: No tweets or empty array, missing fields  

---

#### 1.4 Data Merging

**Overview:**  
Combines the three cleaned data streams (Google, Reddit, X) into a single unified dataset for analysis.

**Nodes Involved:**  
- Merge

**Node Details:**  

- **Merge**  
  - Type: Merge (Append mode)  
  - Configuration: Accepts three inputs from Filter & Label Google, Filter & Label Reddit, and Filter & Label X  
  - Input: Three separate datasets  
  - Output: Single combined dataset for downstream processing  
  - Edge Cases: One or more inputs empty or missing  

---

#### 1.5 Core Analysis

**Overview:**  
Applies custom JavaScript logic to categorize each complaint into defined pain points based on keyword matching and assigns sentiment scores. Generates unique keys for deduplication.

**Nodes Involved:**  
- Categorization & Sentiment (Code)

**Node Details:**  

- **Categorization & Sentiment**  
  - Type: Code  
  - Configuration:  
    - Converts full_text to lowercase  
    - Checks keywords to assign pain points such as 'Call Hold/Availability', 'Scheduling Inefficiency', 'Receptionist Tone/Quality', and 'Automated System Frustration'  
    - Assigns negative sentiment scores based on severity of keywords  
    - Creates a unique key combining pain point and first 50 chars of text (cleaned)  
  - Input: Merged dataset  
  - Output: Dataset with pain_point, sentiment_score, unique_key fields added  
  - Edge Cases: Text missing or empty, overlapping keywords, false positives  

---

#### 1.6 Aggregation and Deduplication

**Overview:**  
Removes duplicate complaints using unique keys, counts occurrences per pain point and source, computes average sentiment, and formats a summary string with statistics and examples. Prepares data for logging and AI briefing.

**Nodes Involved:**  
- Deduplicate, Count, and Format (Code)

**Node Details:**  

- **Deduplicate, Count, and Format**  
  - Type: Code  
  - Configuration:  
    - Uses a Map to track unique complaints by unique_key  
    - Aggregates counts for pain points and sources  
    - Calculates average sentiment and rounds it  
    - Builds a multi-line summary string including total unique complaints, sentiment intensity, pain point frequencies, source distribution, and top 5 complaint examples  
    - Prepares an array of objects suitable for Google Sheets logging, embedding summary string for later retrieval  
  - Input: Categorized dataset  
  - Output: Aggregated data array for logging and summary retrieval  
  - Edge Cases: Empty input, missing keys, zero counts  

---

#### 1.7 Summary Retrieval and AI Processing

**Overview:**  
Retrieves the aggregated summary string, checks for valid data, and sends it to Anthropic Claude AI for generation of a professional HTML executive briefing with structured sections, tables, and analysis.

**Nodes Involved:**  
- Get Summary (Code)  
- Executive Email (Anthropic AI)

**Node Details:**  

- **Get Summary**  
  - Type: Code  
  - Configuration:  
    - Retrieves the summary string from the "Deduplicate, Count, and Format" node's output using $items()  
    - Returns an error message if no data found  
    - Provides fallback text if no market intelligence data is present  
  - Input: Deduplicate, Count, and Format output  
  - Output: Object containing the summary string for AI consumption  
  - Edge Cases: Missing or empty summary string  

- **Executive Email**  
  - Type: Anthropic AI (LangChain node)  
  - Configuration:  
    - Uses Claude model `claude-haiku-4-5-20251001`  
    - System prompt instructs strict HTML output formatting without markdown or tags  
    - Generates sections: Opportunity Statement, Top 3 Pain Points with AI feature suggestions, Source Trust Assessment with styled HTML table  
    - Receives raw summary string as input message  
  - Input: Summary string from Get Summary  
  - Output: HTML-formatted executive summary  
  - Credentials: Anthropic API  
  - Edge Cases: LLM model failure, malformed input, API rate limits  

---

#### 1.8 Final Dispatch and Logging

**Overview:**  
Sends the generated HTML executive summary via Gmail to the user and logs the search parameters and analytics summary in Google Sheets for audit and reproducibility.

**Nodes Involved:**  
- Send Email (Gmail)  
- Log Search Details (Google Sheets)

**Node Details:**  

- **Send Email**  
  - Type: Gmail node  
  - Configuration:  
    - Recipient email address from form input  
    - Email subject includes user name from form  
    - Email body contains raw HTML content generated by Anthropic node  
    - Does not append attribution  
  - Input: HTML content from Executive Email node  
  - Credentials: Gmail OAuth2 account  
  - Edge Cases: Invalid email, Gmail API errors, HTML rendering issues  

- **Log Search Details**  
  - Type: Google Sheets  
  - Configuration:  
    - Appends a new row with columns: Pain_Point, Count, Latest_Source, Execution_Date, Average_Sentiment, Summary_Sample_Example  
    - Sheet ID and name configured to a specific Google Sheet for audit trail  
  - Input: Aggregated summary data from Deduplicate, Count, and Format node  
  - Credentials: Google Sheets OAuth2 account  
  - Edge Cases: API limits, sheet access permissions  

---

### 3. Summary Table

| Node Name                | Node Type                          | Functional Role                              | Input Node(s)                   | Output Node(s)                            | Sticky Note                                                                                                 |
|--------------------------|----------------------------------|----------------------------------------------|--------------------------------|-------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Form                     | Form Trigger                     | Captures user inputs for search parameters   | —                              | Search Google, List Creator, Search X     | Guide for constructing search queries and subreddit inputs with example queries and links (Sticky Note)      |
| Search Google             | SerpAPI Node                    | Fetches Google search results for keywords   | Form                           | Filter & Label Google                      | Data ingestion and strategic context for SERP API (Sticky Note1)                                            |
| Filter & Label Google     | Code                            | Cleans and labels Google data                 | Search Google                  | Merge                                     |                                                                                                             |
| List Creator             | Code                            | Splits subreddit inputs into individual items| Form                           | Search Reddit                             | Data ingestion and looping explanation for Reddit (Sticky Note2)                                            |
| Search Reddit             | Reddit OAuth2 API               | Fetches posts from specified subreddits      | List Creator                  | Filter & Label Reddit                      |                                                                                                             |
| Filter & Label Reddit     | Code                            | Filters and labels Reddit posts               | Search Reddit                  | Merge                                     |                                                                                                             |
| Search X                 | HTTP Request                   | Fetches tweets from X/Twitter external API   | Form                           | Filter & Label X                          | Use of external API to avoid rate limits and error handling (Sticky Note3)                                  |
| Filter & Label X          | Code                            | Filters and labels tweets                      | Search X                      | Merge                                     |                                                                                                             |
| Merge                    | Merge (Append)                  | Combines data streams into unified dataset   | Filter & Label Google, Reddit, X| Categorization & Sentiment                |                                                                                                             |
| Categorization & Sentiment| Code                            | Categorizes complaints and scores sentiment  | Merge                         | Deduplicate, Count, and Format              | Core analytical engine applying hybrid categorization logic (Sticky Note4)                                  |
| Deduplicate, Count, and Format| Code                      | Deduplicates, aggregates, formats summary    | Categorization & Sentiment    | Get Summary, Log Search Details            |                                                                                                             |
| Get Summary               | Code                            | Retrieves summary string for AI briefing      | Deduplicate, Count, and Format | Executive Email                            | Retrieves and sanitizes summary data for AI consumption (Sticky Note5)                                      |
| Executive Email           | Anthropic AI (LangChain)        | Generates HTML executive summary via LLM     | Get Summary                   | Send Email                                 | Produces formatted AI briefing email (Sticky Note5)                                                        |
| Send Email               | Gmail                           | Sends executive summary email                  | Executive Email               | —                                         |                                                                                                             |
| Log Search Details        | Google Sheets                   | Logs search parameters and analytics          | Deduplicate, Count, and Format | —                                         | Maintains audit trail of search inputs and results (Sticky Note6)                                           |
| Sticky Note               | Sticky Note                     | Provides query construction guidance           | —                            | —                                         | See content in respective sticky notes above                                                               |
| Sticky Note1              | Sticky Note                     | Explains SERP API data ingestion               | —                            | —                                         |                                                                                                             |
| Sticky Note2              | Sticky Note                     | Explains Reddit data ingestion structure      | —                            | —                                         |                                                                                                             |
| Sticky Note3              | Sticky Note                     | Explains X/Twitter API data ingestion          | —                            | —                                         |                                                                                                             |
| Sticky Note4              | Sticky Note                     | Describes core analytical engines              | —                            | —                                         |                                                                                                             |
| Sticky Note5              | Sticky Note                     | Describes final delivery system & LLM usage   | —                            | —                                         |                                                                                                             |
| Sticky Note6              | Sticky Note                     | Explains purpose of logging node               | —                            | —                                         |                                                                                                             |
| Sticky Note8              | Sticky Note                     | High-level workflow overview                    | —                            | —                                         |                                                                                                             |
| Sticky Note12             | Sticky Note                     | Support contact link                            | —                            | —                                         | [Connect on LinkedIn](https://www.linkedin.com/in/bhuvaneshhhh/)                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node ("Form"):**  
   - Add fields: Name (text, required), Email (email, required), Location (text, required), SerpAPI query (text, required), X/Twitter query (text, required), Subreddit #1 (text, required), Subreddit #2 (text, required).  
   - Set webhook to trigger on submission.

2. **Create SerpAPI Node ("Search Google"):**  
   - Set operation: Get All results.  
   - Query: Use expression referencing form's SerpAPI query field.  
   - Location: Use expression referencing form's Location field.  
   - Credentials: Connect SerpAPI credential.

3. **Create Code Node ("Filter & Label Google"):**  
   - Input: Connect from "Search Google".  
   - Code: Extract `title`, `snippet`, and `link` from `organic_results`.  
   - Label each item with source: 'SERP API (Web)'.  
   - Output array of cleaned results.

4. **Create Code Node ("List Creator"):**  
   - Input: Connect from "Form".  
   - Code: Return array of objects with `subreddit` keys from the two subreddit form fields.  
   - This splits input for iteration.

5. **Create Reddit Node ("Search Reddit"):**  
   - Operation: Get All posts.  
   - Subreddit: Use expression referencing `subreddit` from "List Creator".  
   - Limit: 50 posts.  
   - Credentials: Connect Reddit OAuth2.

6. **Create Code Node ("Filter & Label Reddit"):**  
   - Input: Connect from "Search Reddit".  
   - Code: Filter posts with score > 5; extract `title`, `selftext`, `url`; label source as 'Reddit'; combine title and text into `full_text`.

7. **Create HTTP Request Node ("Search X"):**  
   - URL: `https://api.twitterapi.io/twitter/tweet/advanced_search`  
   - Query Params:  
     - `query`: expression from form's X/Twitter query field  
     - `queryType`: "Latest"  
   - Authentication: HTTP Header Auth with Twitter demo credentials.

8. **Create Code Node ("Filter & Label X"):**  
   - Input: Connect from "Search X".  
   - Code: Extract tweets; filter based on likes > 5; standardize fields; label source 'Twitter (External API)'.

9. **Create Merge Node ("Merge"):**  
   - Set to append mode with 3 inputs.  
   - Connect outputs from "Filter & Label Google", "Filter & Label Reddit", and "Filter & Label X".

10. **Create Code Node ("Categorization & Sentiment"):**  
    - Input: Connect from "Merge".  
    - Code: Implement keyword-based categorization into pain points and sentiment scoring. Generate unique key from pain point and text snippet.

11. **Create Code Node ("Deduplicate, Count, and Format"):**  
    - Input: Connect from "Categorization & Sentiment".  
    - Code: Deduplicate items by unique key; count pain points and sources; calculate average sentiment; create summary string; prepare output for logging.

12. **Create Code Node ("Get Summary"):**  
    - Input: Connect from "Deduplicate, Count, and Format".  
    - Code: Retrieve summary string using $items(); handle no data case.

13. **Create Anthropic Node ("Executive Email"):**  
    - Model: `claude-haiku-4-5-20251001`  
    - System prompt: Provide instructions for strict HTML executive summary generation including opportunity statement, pain points, source trust table, and key insights.  
    - Input message: summary string from "Get Summary".  
    - Credentials: Anthropic API.

14. **Create Gmail Node ("Send Email"):**  
    - Input: Connect from "Executive Email".  
    - Recipient: Use form's email field.  
    - Subject: Include user's name from form.  
    - Message: Use raw HTML output from Anthropic node.  
    - Credentials: Gmail OAuth2.

15. **Create Google Sheets Node ("Log Search Details"):**  
    - Input: Connect from "Deduplicate, Count, and Format".  
    - Operation: Append.  
    - Map columns to pain point, counts, latest source, execution date (current date), average sentiment, summary sample example.  
    - Credentials: Google Sheets OAuth2.

---

### 5. General Notes & Resources

| Note Content                                                                                                           | Context or Link                                                                                     |
|------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Query construction guide for form inputs including examples and pro tips for SERP API, X/Twitter, and Reddit queries.  | See Sticky Note attached to Form node; includes links: https://serpapi.com/search-api and https://github.com/igorbrigadir/twitter-advanced-search |
| Data ingestion from SERP API establishes strategic context by collecting authoritative web commentary.                  | Sticky Note1                                                                                       |
| Reddit ingestion uses iterative looping on subreddits to gather community complaints, filtering low-engagement posts. | Sticky Note2                                                                                       |
| Twitter/X ingestion uses an external API to bypass standard API rate limits and filters for high-engagement tweets.   | Sticky Note3                                                                                       |
| Core analytical engine applies hybrid keyword categorization and sentiment scoring, avoiding costly LLM calls.        | Sticky Note4                                                                                       |
| Final delivery includes retrieving summary from memory, generating a professional HTML brief via Anthropic, and sending email. | Sticky Note5                                                                                       |
| Search logging node creates an audit trail for reproducibility and debugging.                                          | Sticky Note6                                                                                       |
| Workflow overview and logic explained in Sticky Note8.                                                                |                                                                                                   |
| Support contact link provided for workflow questions and assistance.                                                  | [LinkedIn Profile](https://www.linkedin.com/in/bhuvaneshhhh/)                                     |

---

**Disclaimer:** The text provided derives exclusively from an automated workflow built with n8n, respecting all content policies. It contains no illegal, offensive, or protected elements. All data handled is legal and public.