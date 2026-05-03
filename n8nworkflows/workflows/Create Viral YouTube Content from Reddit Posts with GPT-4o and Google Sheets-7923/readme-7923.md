Create Viral YouTube Content from Reddit Posts with GPT-4o and Google Sheets

https://n8nworkflows.xyz/workflows/create-viral-youtube-content-from-reddit-posts-with-gpt-4o-and-google-sheets-7923


# Create Viral YouTube Content from Reddit Posts with GPT-4o and Google Sheets

### 1. Workflow Overview

This workflow automates the process of creating viral YouTube content ideas by scraping Reddit posts from the r/confession subreddit, filtering and scoring them for viral potential using GPT-4o, and then storing the high-potential posts in a Google Sheets spreadsheet. It is designed as Phase 1 in a larger pipeline aimed at creating YouTube Shorts from Reddit content.

**Logical blocks:**

- **1.1 Input Reception and Reddit Data Fetching:** Manual trigger and HTTP request to pull Reddit posts.
- **1.2 Data Cleaning and Filtering:** Filtering out irrelevant or undesirable posts (e.g., political or NSFW content).
- **1.3 Virality Potential Filtering:** Applying numeric thresholds on upvotes, comments, and NSFW flags.
- **1.4 Duplicate Detection:** Comparing new posts against existing spreadsheet entries to avoid redundancy.
- **1.5 Virality Scoring with GPT-4o:** Using OpenAI GPT-4o to assign a virality score based on content.
- **1.6 Final Filtering and Google Sheets Storage:** Filtering posts by minimum virality score and writing them to Google Sheets.
- **1.7 Documentation and Setup Notes:** Sticky notes providing workflow overview, instructions, and phase context.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Reddit Data Fetching

- **Overview:** Initiates workflow manually and fetches the latest 30 hot posts from r/confession subreddit.
- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Get AskReddit Posts (HTTP Request)

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point, manual start for testing or execution  
    - Config: No parameters, simple trigger  
    - Connections: Outputs to "Get AskReddit Posts"  
    - Failures: None expected; manual trigger

  - **Get AskReddit Posts**  
    - Type: HTTP Request  
    - Role: Fetches Reddit posts JSON data from r/confession hot posts, limited to 30  
    - Config:  
      - URL: `https://www.reddit.com/r/confession/hot.json?limit=30`  
      - Headers: User-Agent set to "n8n-askreddit-scraper" to respect Reddit API guidelines  
      - Output: JSON response parsed  
    - Connections: Input from manual trigger; output to "Filter AskReddit Data"  
    - Failures: HTTP errors, rate limiting or network issues; Reddit API rate limits may apply  
    - Notes: Uses Reddit public API anonymously; no OAuth credentials needed  

#### 1.2 Data Cleaning and Filtering

- **Overview:** Removes posts containing certain keywords and maps relevant fields for further processing.
- **Nodes Involved:**  
  - Filter AskReddit Data (Code)  
  - Filter for Virality Potential (Code)

- **Node Details:**

  - **Filter AskReddit Data**  
    - Type: Code (JavaScript)  
    - Role: Filters out posts with titles containing "trump", "president", "war", "israel" (case-insensitive)  
    - Config:  
      - Filters `items[0].json.data.children` (Reddit API response structure)  
      - Maps fields: post_id, title, selftext, score, num_comments, over_18  
    - Connections: Input from HTTP Request; output to "Filter for Virality Potential"  
    - Failures: Expression errors if Reddit JSON structure changes; empty result if no posts pass filter  

  - **Filter for Virality Potential**  
    - Type: Code (JavaScript)  
    - Role: Filters posts to keep only those with score > 60, comments > 40, and not NSFW (over_18 = false)  
    - Config: Simple numeric and boolean filtering on previous output  
    - Connections: Input from Filter AskReddit Data; output to "Merge" and "Get row(s) in sheet"  
    - Failures: None expected if input is valid; filters aggressively  

#### 1.3 Duplicate Detection

- **Overview:** Retrieves existing Reddit post IDs from Google Sheets and filters out duplicates from new posts.
- **Nodes Involved:**  
  - Get row(s) in sheet (Google Sheets)  
  - Merge (Merge node combining filtered posts and existing rows)  
  - Filter Duplicates (Code)

- **Node Details:**

  - **Get row(s) in sheet**  
    - Type: Google Sheets  
    - Role: Retrieves all rows from the target spreadsheet and sheet (ASKREDDIT tab)  
    - Config:  
      - Document ID and Sheet ID set to specific Google Sheet  
      - No filters applied, gets entire sheet data once per workflow execution  
    - Connections: Input from "Filter for Virality Potential" (parallel)  
    - Failures: API errors, auth errors if credentials invalid or quota exceeded  
    - Credentials: Google Sheets OAuth2 account  
    - Notes: Executes once per workflow run  

  - **Merge**  
    - Type: Merge  
    - Role: Combines newly filtered Reddit posts and existing spreadsheet rows for duplicate checking  
    - Config: Default settings with two inputs  
    - Connections: Inputs from "Filter for Virality Potential" and "Get row(s) in sheet"  
    - Output to "Filter Duplicates"  
    - Failures: None expected; empty inputs may result in empty output  

  - **Filter Duplicates**  
    - Type: Code (JavaScript)  
    - Role: Filters out Reddit posts whose post_id already exists in spreadsheet's "Reddit Post ID" column  
    - Config:  
      - Extracts post_id from Reddit posts and compares to existing rows' Reddit Post ID field  
      - Uses Set for efficient lookup  
    - Connections: Input from Merge; output to "Add Numbering"  
    - Failures: Type coercion or missing fields could cause errors; empty existing rows means no duplicates filtered  

#### 1.4 Virality Scoring with GPT-4o

- **Overview:** Assigns a virality score (1–10) to each post using OpenAI's GPT-4o model evaluating emotional engagement and relatability.
- **Nodes Involved:**  
  - Add Numbering (Code)  
  - GIVE VIRALITY SCORE (OpenAI node)  
  - Merge1 (Merge)  
  - Add Virality Score (Code)  
  - Filter by Score (Code)

- **Node Details:**

  - **Add Numbering**  
    - Type: Code (JavaScript)  
    - Role: Adds a sequential number field to each post for indexing  
    - Config: Iterates items, adds `number` property starting at 1  
    - Connections: Input from Filter Duplicates; output to GIVE VIRALITY SCORE and Merge1 (second input)  
    - Failures: None expected  

  - **GIVE VIRALITY SCORE**  
    - Type: OpenAI (LangChain)  
    - Role: Sends each post’s title to GPT-4o to rate virality potential (1–10)  
    - Config:  
      - Model: "gpt-4o"  
      - Message prompt: instructs model to score based on emotional engagement, curiosity, relatability, returning only the number  
    - Connections: Input from Add Numbering; output to Merge1  
    - Credentials: OpenAI API key  
    - Failures: API rate limits, network errors, invalid API key, unexpected model output formats  

  - **Merge1**  
    - Type: Merge  
    - Role: Combines outputs from Add Numbering and GIVE VIRALITY SCORE by position (index) to align original data with GPT scores  
    - Config: Mode: Combine, Combine By: Position  
    - Connections: Inputs from Add Numbering (index 1) and GIVE VIRALITY SCORE (index 0); output to Add Virality Score  
    - Failures: Mismatch in item counts may cause misalignment  

  - **Add Virality Score**  
    - Type: Code (JavaScript)  
    - Role: Extracts virality score from GPT message content and adds it as a new field `virality_score` in JSON  
    - Config: Trims GPT response content; sets `virality_score` property  
    - Connections: Input from Merge1; output to Filter by Score  
    - Failures: Malformed GPT response or empty content could cause undefined scores  

  - **Filter by Score**  
    - Type: Code (JavaScript)  
    - Role: Filters posts to keep only those with virality_score >= 6 (threshold for viral potential)  
    - Config: Parses `virality_score` as integer, applies threshold  
    - Connections: Input from Add Virality Score; output to Write confession DETAILS  
    - Failures: Parsing errors if score is not numeric; strict filter may discard all items  

#### 1.5 Final Storage in Google Sheets

- **Overview:** Writes the filtered viral posts into a Google Sheet with detailed metadata.
- **Nodes Involved:**  
  - Write confession DETAILS (Google Sheets)

- **Node Details:**

  - **Write confession DETAILS**  
    - Type: Google Sheets  
    - Role: Appends new viral posts to the Google Sheet tab "ASKREDDIT" with multiple columns including date, type, number, status, upvotes, comments, question, Reddit post ID, and virality score  
    - Config:  
      - Document ID: Specific Google Sheet ID  
      - Sheet Name: ASKREDDIT (by ID 1850949286)  
      - Columns mapped and values set with expressions (e.g., current date, post fields)  
      - Operation: Append rows  
    - Connections: Input from Filter by Score  
    - Credentials: Google Sheets OAuth2 account  
    - Failures: API errors, auth failures, quota limits, malformed data errors  

#### 1.6 Documentation and Setup Notes

- **Overview:** Provides descriptive sticky notes explaining the workflow purpose, setup instructions, and context.
- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1

- **Node Details:**

  - **Sticky Note**  
    - Type: Sticky Note  
    - Role: Visual note indicating this is Phase 1 that scrapes Reddit data and writes to Excel (Google Sheets)  
    - Connections: None  

  - **Sticky Note1**  
    - Type: Sticky Note  
    - Role: Detailed description of workflow objectives, setup steps, API credentials requirements, scoring explanation, and usage instructions  
    - Connections: None  
    - Content emphasizes manual or scheduled runs and phase integration  

---

### 3. Summary Table

| Node Name                | Node Type               | Functional Role                             | Input Node(s)                        | Output Node(s)                  | Sticky Note                                                                                                   |
|--------------------------|-------------------------|---------------------------------------------|------------------------------------|--------------------------------|--------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger          | Manual start of workflow                    | -                                  | Get AskReddit Posts             |                                                                                                              |
| Get AskReddit Posts       | HTTP Request            | Fetches Reddit posts JSON                   | When clicking ‘Test workflow’       | Filter AskReddit Data           |                                                                                                              |
| Filter AskReddit Data     | Code                    | Filters out unwanted posts by keywords, maps fields | Get AskReddit Posts                | Filter for Virality Potential   |                                                                                                              |
| Filter for Virality Potential | Code                    | Filters posts by score, comments, NSFW flag | Filter AskReddit Data              | Merge, Get row(s) in sheet      |                                                                                                              |
| Get row(s) in sheet       | Google Sheets           | Retrieves existing rows from Google Sheets | Filter for Virality Potential      | Merge                          |                                                                                                              |
| Merge                    | Merge                   | Combines new posts and existing sheet data | Filter for Virality Potential, Get row(s) in sheet | Filter Duplicates              |                                                                                                              |
| Filter Duplicates         | Code                    | Removes duplicate posts based on Reddit post ID | Merge                            | Add Numbering                  |                                                                                                              |
| Add Numbering             | Code                    | Adds sequential numbering to posts          | Filter Duplicates                  | GIVE VIRALITY SCORE, Merge1    |                                                                                                              |
| GIVE VIRALITY SCORE       | OpenAI (GPT-4o)         | Assigns virality score using GPT-4o          | Add Numbering                     | Merge1                        |                                                                                                              |
| Merge1                   | Merge                   | Combines original data with GPT virality scores | GIVE VIRALITY SCORE, Add Numbering | Add Virality Score             |                                                                                                              |
| Add Virality Score        | Code                    | Extracts and adds virality score field       | Merge1                           | Filter by Score                |                                                                                                              |
| Filter by Score           | Code                    | Keeps posts with virality score ≥ 6          | Add Virality Score                | Write confession DETAILS       |                                                                                                              |
| Write confession DETAILS  | Google Sheets           | Appends filtered viral posts to Google Sheets | Filter by Score                  | -                              |                                                                                                              |
| Sticky Note              | Sticky Note             | Notes that this is Phase 1 for scraping/writing | -                               | -                              | THIS IS THE FIRST PHASE. IT SCRAPS REDDIT DATA AND WRITES IN EXCEL                                           |
| Sticky Note1             | Sticky Note             | Detailed workflow description and setup instructions | -                               | -                              | 📝 Description\n\nThis workflow automates the collection, filtering, and scoring of trending AskReddit posts for viral potential. It pulls posts from Reddit, removes duplicates, calculates a custom virality score, and writes the final candidates into Google Sheets for later use in content creation.\n\nThis is Phase 1 of the AskReddit → YouTube Shorts automation pipeline. It prepares clean, high-quality data that can be used in the next phases (script generation, AI video creation, and publishing).\n\n⚙️ Setup Steps\n\nImport Workflow into your n8n instance.\n\nReddit API:\n\nAdd your Reddit API credentials in the \"Get AskReddit Posts\" node.\n\nGoogle Sheets:\n\nConnect your Google account.\n\nPoint the \"Write Candidates\" node to your target Google Sheet.\n\nVirality Scoring:\n\nThe \"Add Virality Score\" node assigns weights (e.g., upvotes, comments).\n\nAdjust the scoring logic as needed for your niche.\n\nRun Workflow:\n\nExecute manually or schedule with Cron.\n\nVerify that trending AskReddit posts appear in your sheet, scored and cleaned. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Start workflow manually for testing or execution  

2. **Create HTTP Request Node: "Get AskReddit Posts"**  
   - URL: `https://www.reddit.com/r/confession/hot.json?limit=30`  
   - Headers: Add `User-Agent` header with value `"n8n-askreddit-scraper"`  
   - Response Format: JSON  
   - Connect output of Manual Trigger to this node  

3. **Create Code Node: "Filter AskReddit Data"**  
   - Function: Filter posts that do not contain keywords "trump", "president", "war", "israel" (case-insensitive) in title  
   - Map fields: post_id, title, selftext, score, num_comments, over_18  
   - Input: Output of "Get AskReddit Posts"  
   - Output: Filtered list of posts  

4. **Create Code Node: "Filter for Virality Potential"**  
   - Filter posts where:  
     - score > 60  
     - num_comments > 40  
     - over_18 === false (exclude NSFW)  
   - Input: Output from "Filter AskReddit Data"  

5. **Create Google Sheets Node: "Get row(s) in sheet"**  
   - Operation: Read all rows from target spreadsheet  
   - Document ID: Your Google Sheet ID containing previous Reddit posts  
   - Sheet Name: Use sheet/tab ID corresponding to 'ASKREDDIT'  
   - Credentials: Configure Google Sheets OAuth2 credentials  
   - Set to execute once per run  
   - Connect in parallel with "Filter for Virality Potential" output  

6. **Create Merge Node: "Merge"**  
   - Combine the filtered posts and existing spreadsheet rows  
   - Inputs:  
     - Input 1: Output from "Filter for Virality Potential"  
     - Input 2: Output from "Get row(s) in sheet"  

7. **Create Code Node: "Filter Duplicates"**  
   - Remove posts whose post_id already exists in spreadsheet "Reddit Post ID" column  
   - Input: Output from "Merge"  

8. **Create Code Node: "Add Numbering"**  
   - Add sequential `number` property starting at 1 to each filtered post  
   - Input: Output from "Filter Duplicates"  

9. **Create OpenAI Node: "GIVE VIRALITY SCORE"**  
   - Model: GPT-4o  
   - Prompt: "You're an expert in viral content analysis. Rate the virality potential (1–10) of this Reddit question based on emotional engagement, curiosity, and relatability: {{$json.title}}. Respond with just the number."  
   - Credentials: Configure with your OpenAI API key  
   - Input: Output from "Add Numbering"  

10. **Create Merge Node: "Merge1"**  
    - Mode: Combine by Position  
    - Inputs:  
      - Input 1: Output from "GIVE VIRALITY SCORE"  
      - Input 2: Output from "Add Numbering"  

11. **Create Code Node: "Add Virality Score"**  
    - Extract the GPT score from message content and add as `virality_score` property  
    - Input: Output from "Merge1"  

12. **Create Code Node: "Filter by Score"**  
    - Filter posts with `virality_score` >= 6  
    - Input: Output from "Add Virality Score"  

13. **Create Google Sheets Node: "Write confession DETAILS"**  
    - Operation: Append rows to Google Sheets  
    - Document ID: Same Google Sheet as above  
    - Sheet Name: ASKREDDIT tab  
    - Map columns: Date (current date), TYPE ("CONFESSION"), Number, Status ("TODO"), Upvotes, Answer 1 (selftext), Comments, Question (title), Reddit Post ID, Virality Score  
    - Credentials: Google Sheets OAuth2 credentials  
    - Input: Output from "Filter by Score"  

14. **(Optional) Add Sticky Notes**  
    - Add visual notes for workflow phase and setup instructions to assist users  

15. **Connect all nodes according to the sequence above**  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Context or Link                                                                                          |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| THIS IS THE FIRST PHASE. IT SCRAPS REDDIT DATA AND WRITES IN EXCEL                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Workflow sticky note emphasizing this is phase 1 of automation                                          |
| 📝 Description\n\nThis workflow automates the collection, filtering, and scoring of trending AskReddit posts for viral potential. It pulls posts from Reddit, removes duplicates, calculates a custom virality score, and writes the final candidates into Google Sheets for later use in content creation.\n\nThis is Phase 1 of the AskReddit → YouTube Shorts automation pipeline. It prepares clean, high-quality data that can be used in the next phases (script generation, AI video creation, and publishing).\n\n⚙️ Setup Steps\n\nImport Workflow into your n8n instance.\n\nReddit API:\n\nAdd your Reddit API credentials in the "Get AskReddit Posts" node.\n\nGoogle Sheets:\n\nConnect your Google account.\n\nPoint the "Write Candidates" node to your target Google Sheet.\n\nVirality Scoring:\n\nThe "Add Virality Score" node assigns weights (e.g., upvotes, comments).\n\nAdjust the scoring logic as needed for your niche.\n\nRun Workflow:\n\nExecute manually or schedule with Cron.\n\nVerify that trending AskReddit posts appear in your sheet, scored and cleaned. | Workflow sticky note with detailed description and setup instructions                                    |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow built with n8n, a workflow automation tool. This process strictly respects content policies and contains no illegal, offensive, or protected elements. All data processed is lawful and publicly accessible.