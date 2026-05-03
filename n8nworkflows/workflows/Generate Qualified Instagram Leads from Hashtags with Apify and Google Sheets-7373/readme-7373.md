Generate Qualified Instagram Leads from Hashtags with Apify and Google Sheets

https://n8nworkflows.xyz/workflows/generate-qualified-instagram-leads-from-hashtags-with-apify-and-google-sheets-7373


# Generate Qualified Instagram Leads from Hashtags with Apify and Google Sheets

### 1. Workflow Overview

This n8n workflow automates the generation of qualified Instagram leads by leveraging hashtags. It extracts hashtags from a Google Sheet, scrapes Instagram posts containing these hashtags via the Apify platform, processes and analyzes post captions for language and hashtag content, aggregates unique usernames from the posts, scrapes detailed user profiles, and finally filters users based on follower count criteria to produce a targeted list of Instagram leads.

The workflow is logically divided into five blocks:

- **1.1 Input Reception and Hashtag Preparation:** Retrieves hashtags from Google Sheets and constructs Instagram hashtag URLs for scraping.
- **1.2 Instagram Hashtag Posts Scraping and Caption Analysis:** Uses Apify API to scrape posts, then analyzes captions to detect hashtags, links, and language attributes.
- **1.3 Filtering Posts Based on Content:** Applies conditions to filter posts that contain only hashtags or are English-language dominant.
- **1.4 Usernames Aggregation and Profile Scraping:** Aggregates usernames from filtered posts, removes duplicates, and scrapes user profile data from Instagram via Apify.
- **1.5 User Filtering by Follower Count:** Filters scraped user profiles to keep only those with followers within a specified range.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Hashtag Preparation

**Overview:**  
This block initiates the workflow by manually triggering it and fetching a list of hashtags from a Google Sheets document. It then formats these hashtags into Instagram hashtag URLs compatible with Apify scraping.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Get list of Hashtags (Google Sheets)  
- Aggregate Hashtags (Aggregate)  
- make hashtag links (Code)  

**Node Details:**  

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Workflow entry point to start execution manually  
  - Configuration: No parameters, simple manual trigger  
  - Inputs: None  
  - Outputs: Triggers "Get list of Hashtags"  
  - Failures: None expected  
  - Notes: Standard manual trigger node  

- **Get list of Hashtags**  
  - Type: Google Sheets  
  - Role: Reads hashtags from a specified Google Sheet and Sheet tab  
  - Configuration: Uses OAuth2 credentials, requires correct Spreadsheet ID and Sheet ID  
  - Key Parameters: Spreadsheet and Sheet IDs provided as secured variables  
  - Inputs: Trigger from manual trigger  
  - Outputs: Rows containing hashtags, expected under field "Hashtag"  
  - Failures: Google API authentication errors, quota limits, invalid IDs  
  - Notes: Requires valid Google Sheets OAuth2 credentials setup  

- **Aggregate Hashtags**  
  - Type: Aggregate  
  - Role: Collects and aggregates all hashtag entries from Google Sheets into a list  
  - Configuration: Aggregates field “Hashtag” from input data  
  - Inputs: Google Sheets output  
  - Outputs: Aggregated array of hashtags  
  - Failures: Empty or malformed input can cause issues  
  - Notes: Ensures unique list for next step  

- **make hashtag links**  
  - Type: Code (JavaScript)  
  - Role: Converts each hashtag string into a formatted Instagram hashtag URL for Apify scraping  
  - Configuration: Custom JavaScript code that encodes hashtags and builds URLs  
  - Key Expressions: Uses `$input.first().json.Hashtag` and `encodeURIComponent()`  
  - Inputs: Aggregated hashtag list  
  - Outputs: Single JSON object with "startUrls" array of URLs  
  - Failures: Empty input array, malformed hashtags, or encoding errors  
  - Notes: Adds trailing slash as required by Instagram URLs  

---

#### 1.2 Instagram Hashtag Posts Scraping and Caption Analysis

**Overview:**  
This block scrapes Instagram posts related to the prepared hashtag URLs using an Apify actor via HTTP POST, then formats and analyzes post captions to extract hashtags, remove links, and detect language characteristics.

**Nodes Involved:**  
- Scrape instagram hashtag posts (HTTP Request)  
- Format captions, usernames and data (Code)  

**Node Details:**  

- **Scrape instagram hashtag posts**  
  - Type: HTTP Request  
  - Role: Calls Apify API to run an Instagram hashtag scraping actor synchronously and retrieves dataset items  
  - Configuration: POST request with JSON body specifying max items (300), start URLs, and a date filter ("until": "2025-06-20")  
  - Headers: Includes Accept: application/json and Authorization with Bearer token (APIFY_API_KEY)  
  - Inputs: Output from "make hashtag links" node  
  - Outputs: JSON array of Instagram posts with fields like caption, title, owner.username, etc.  
  - Failures: API authentication failure, rate limiting, network issues, malformed request body  
  - Notes: Requires valid Apify API key stored as environment variable or credential  

- **Format captions, usernames and data**  
  - Type: Code (JavaScript)  
  - Role: Processes each Instagram post’s caption to:  
    - Extract original caption or title  
    - Count hashtags, remove URLs and hashtags to isolate text  
    - Analyze language presence (English, French, Spanish) via keyword matching  
    - Detect presence of accents and scripts (Arabic/CJK)  
    - Calculate ratios and flags for filtering  
  - Key Expressions: Regular expressions for hashtags, URLs, language keyword arrays, ASCII ratio calculations  
  - Inputs: Instagram posts JSON array  
  - Outputs: Enhanced JSON with analysis fields: hashtagCount, hasHashtags, textNoTags, enHits, frHits, esHits, enRatio, frRatio, esRatio, asciiRatio, etc.  
  - Failures: Malformed captions, missing fields causing runtime errors in regex or arrays  
  - Notes: Critical for downstream filtering logic  

---

#### 1.3 Filtering Posts Based on Content

**Overview:**  
This block applies logical conditions to filter posts: one to select posts containing only hashtags and no other text, and another to keep only posts predominantly in English.

**Nodes Involved:**  
- IF — Hashtags Only (If)  
- IF — English Only (If)  

**Node Details:**  

- **IF — Hashtags Only**  
  - Type: If  
  - Role: Filters posts that have hashtags and no other text content (i.e., textNoTags is empty)  
  - Condition: Checks if `hasHashtags` is true and `textNoTags` is an empty trimmed string  
  - Inputs: Output from "Format captions, usernames and data"  
  - Outputs: Two branches: true (passes filter), false (fails filter)  
  - Failure: Expression evaluation errors if fields missing or unexpectedly typed  
  - Notes: Posts passing here go directly to username deduplication; others are filtered further  

- **IF — English Only**  
  - Type: If  
  - Role: Filters posts where English keywords are detected and French and Spanish keywords are absent  
  - Condition: `enHits > 0 && frHits === 0 && esHits === 0`  
  - Inputs: False branch from "IF — Hashtags Only" node  
  - Outputs: True branch continues, false branch discarded  
  - Failures: Same as above, expression evaluation errors  
  - Notes: Ensures only English-language posts proceed downstream  

---

#### 1.4 Usernames Aggregation and Profile Scraping

**Overview:**  
This block aggregates all usernames from the filtered posts, removes duplicates, and scrapes detailed Instagram profile data for each username via Apify.

**Nodes Involved:**  
- Remove Duplicate Usernames (Remove Duplicates)  
- Combine all usernames (Aggregate)  
- Scrape instagram Profiles (HTTP Request)  

**Node Details:**  

- **Remove Duplicate Usernames**  
  - Type: Remove Duplicates  
  - Role: Removes duplicate Instagram usernames from the filtered posts  
  - Configuration: Compares on the field "owner.username"  
  - Inputs: True branches from both "IF — Hashtags Only" and "IF — English Only"  
  - Outputs: Unique username list, passed to aggregation  
  - Failures: Missing field errors if "owner.username" is absent  
  - Notes: Prevents redundant profile scraping calls  

- **Combine all usernames**  
  - Type: Aggregate  
  - Role: Aggregates the unique usernames into an array for batch processing  
  - Configuration: Aggregates field "owner.username"  
  - Inputs: Output of "Remove Duplicate Usernames"  
  - Outputs: JSON object with "username" array  
  - Failures: Empty input arrays cause empty output  
  - Notes: Prepares usernames for Apify profile scraping API  

- **Scrape instagram Profiles**  
  - Type: HTTP Request  
  - Role: Calls Apify API to scrape profile details for a list of Instagram usernames synchronously  
  - Configuration: POST request with JSON body containing usernames array, headers include Authorization with APIFY_API_KEY  
  - Inputs: JSON list of usernames from "Combine all usernames"  
  - Outputs: Detailed user profile data including follower counts  
  - Failures: API auth errors, rate limits, malformed username array, network issues  
  - Notes: Requires valid Apify API key and actor availability  

---

#### 1.5 User Filtering by Follower Count

**Overview:**  
This final block filters scraped user profiles, retaining only those users whose follower count falls within a targeted range (10,000 to 100,000 followers).

**Nodes Involved:**  
- If followers are in a certain range continue (If)  

**Node Details:**  

- **If followers are in a certain range continue**  
  - Type: If  
  - Role: Filters user profiles where `followersCount` is between 10,000 and 100,000 inclusive  
  - Conditions: `followersCount >= 10000 && followersCount <= 100000`  
  - Inputs: Output from "Scrape instagram Profiles"  
  - Outputs: True branch contains filtered qualified leads, false branch discarded  
  - Failures: Missing or non-numeric `followersCount` fields may cause errors  
  - Notes: This filtering step defines the quality criteria for leads  

---

### 3. Summary Table

| Node Name                      | Node Type          | Functional Role                                  | Input Node(s)                       | Output Node(s)                    | Sticky Note                                                                                                    |
|--------------------------------|--------------------|-------------------------------------------------|-----------------------------------|----------------------------------|---------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’| Manual Trigger     | Entry point to start the workflow manually      | —                                 | Get list of Hashtags             |                                                                                                               |
| Get list of Hashtags            | Google Sheets      | Fetches hashtags list from Google Sheets        | When clicking ‘Execute workflow’  | Aggregate Hashtags               |                                                                                                               |
| Aggregate Hashtags             | Aggregate          | Aggregates all hashtags into a list              | Get list of Hashtags              | make hashtag links               |                                                                                                               |
| make hashtag links              | Code               | Builds Instagram hashtag URLs for scraping      | Aggregate Hashtags               | Scrape instagram hashtag posts  | ## Read Hashtags & Prepare Instagram Links - Retrieve hashtags, build URLs for scraping                       |
| Scrape instagram hashtag posts | HTTP Request       | Calls Apify to scrape Instagram posts by hashtag| make hashtag links               | Format captions, usernames and data |                                                                                                               |
| Format captions, usernames and data | Code          | Analyzes captions, extracts hashtags and language| Scrape instagram hashtag posts  | IF — Hashtags Only              | ## Analyze Post Captions - Extract & process captions, analyze language, hashtags                              |
| IF — Hashtags Only             | If                 | Filters posts that contain only hashtags         | Format captions, usernames and data | Remove Duplicate Usernames (true branch), IF — English Only (false branch) |                                                                                                               |
| IF — English Only              | If                 | Filters posts to include only English-language   | IF — Hashtags Only (false branch)| Remove Duplicate Usernames       |                                                                                                               |
| Remove Duplicate Usernames     | Remove Duplicates  | Removes duplicate Instagram usernames            | IF — Hashtags Only (true) and IF — English Only (true) | Combine all usernames           |                                                                                                               |
| Combine all usernames          | Aggregate          | Aggregates usernames into an array                | Remove Duplicate Usernames       | Scrape instagram Profiles       | # Gather & Filter User Profiles - Combine usernames, scrape profiles, filter by followers                     |
| Scrape instagram Profiles      | HTTP Request       | Calls Apify to scrape Instagram user profiles    | Combine all usernames            | If followers are in a certain range continue |                                                                                                               |
| If followers are in a certain range continue | If       | Filters users by follower count range             | Scrape instagram Profiles        | — (final output of qualified leads) |                                                                                                               |
| Sticky Note                   | Sticky Note        | Workflow overview summary                         | —                               | —                              | # Workflow Overview - Start: Get hashtags from Sheets, scrape posts, analyze captions, gather & filter users    |
| Sticky Note1                  | Sticky Note        | Hashtag input and URL preparation explanation    | —                               | —                              | ## Read Hashtags & Prepare Instagram Links - Retrieve hashtags, build URLs for scraping                       |
| Sticky Note2                  | Sticky Note        | Caption analysis explanation                      | —                               | —                              | ## Analyze Post Captions - Extract & process captions, analyze language, hashtags                              |
| Sticky Note3                  | Sticky Note        | User profile gathering and filtering explanation | —                               | —                              | # Gather & Filter User Profiles - Combine usernames, scrape profiles, filter by followers                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Name: "When clicking ‘Execute workflow’"  
   - Purpose: Entry point to manually start the workflow  

2. **Add a Google Sheets Node**  
   - Name: "Get list of Hashtags"  
   - Credentials: Configure Google Sheets OAuth2 with valid account  
   - Parameters: Set Spreadsheet ID and Sheet ID containing your hashtags  
   - Output: Expects rows with a "Hashtag" field  

3. **Add an Aggregate Node**  
   - Name: "Aggregate Hashtags"  
   - Parameters: Aggregate the "Hashtag" field from input rows into a list  

4. **Add a Code Node**  
   - Name: "make hashtag links"  
   - Code:  
     ```javascript
     const tags = $input.first().json.Hashtag || [];
     const startUrls = tags.map(t => `https://www.instagram.com/explore/tags/${encodeURIComponent(t)}/`);
     return [{ startUrls }];
     ```
   - Purpose: Convert hashtags to Instagram hashtag URLs for scraping  

5. **Add an HTTP Request Node**  
   - Name: "Scrape instagram hashtag posts"  
   - Method: POST  
   - URL: `https://api.apify.com/v2/acts/culc72xb7MP3EbaeX/run-sync-get-dataset-items`  
   - Headers:  
     - Accept: application/json  
     - Authorization: Bearer <Your APIFY_API_KEY> (set as credential or environment variable)  
   - Body (JSON):  
     ```json
     {
       "maxItems": 300,
       "startUrls": {{ $json.startUrls }},
       "until": "2025-06-20"
     }
     ```
   - Purpose: Scrape Instagram posts for the hashtags  

6. **Add a Code Node**  
   - Name: "Format captions, usernames and data"  
   - Paste the provided JavaScript code that:  
     - Extracts captions  
     - Counts and strips hashtags and URLs  
     - Analyzes language hits and ratios  
     - Flags accents and scripts  
   - Purpose: Prepare data for filtering  

7. **Add an If Node**  
   - Name: "IF — Hashtags Only"  
   - Condition:  
     - Expression: `{{$json.hasHashtags && ($json.textNoTags || '').trim() === ''}}` is true  
   - Purpose: Filter posts that contain only hashtags  

8. **Add an If Node**  
   - Name: "IF — English Only"  
   - Condition:  
     - Expression: `{{$json.enHits > 0 && $json.frHits === 0 && $json.esHits === 0}}` is true  
   - Input: Connect from the false branch of "IF — Hashtags Only"  
   - Purpose: Filter English-only posts  

9. **Add a Remove Duplicates Node**  
   - Name: "Remove Duplicate Usernames"  
   - Compare on field: "owner.username"  
   - Input: Connect both true branches of "IF — Hashtags Only" and "IF — English Only" to this node (merge branches before this node)  
   - Purpose: Remove duplicate Instagram usernames  

10. **Add an Aggregate Node**  
    - Name: "Combine all usernames"  
    - Aggregate the "owner.username" field into an array  
    - Input: Output of "Remove Duplicate Usernames"  

11. **Add an HTTP Request Node**  
    - Name: "Scrape instagram Profiles"  
    - Method: POST  
    - URL: `https://api.apify.com/v2/acts/dSCLg0C3YEZ83HzYX/run-sync-get-dataset-items`  
    - Headers:  
      - Accept: application/json  
      - Authorization: Bearer <Your APIFY_API_KEY>  
    - Body (JSON):  
      ```json
      {
        "usernames": {{ $json.username.map(u => `"${u}"`).join(", ") }}
      }
      ```  
    - Purpose: Scrape profile details for usernames  

12. **Add an If Node**  
    - Name: "If followers are in a certain range continue"  
    - Conditions:  
      - followersCount >= 10000  
      - followersCount <= 100000  
    - Purpose: Filter profiles by follower count range  

13. **Connect all nodes in the order described above** starting from the manual trigger to the final follower count filter node.

14. **Credential Setup:**  
    - Google Sheets OAuth2: Configure with access to the target spreadsheet  
    - Apify API Key: Store securely as environment variable or credentials for HTTP Request nodes  

15. **Optional:** Add Sticky Notes nodes for documentation and clarity at strategic points, copying content from the workflow’s sticky notes.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                       |
|----------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| Workflow starts by retrieving hashtags from Google Sheets and builds URLs for Instagram hashtag pages.               | Sticky Note1 in workflow                                                                              |
| Captions are analyzed for hashtags, language detection (English, French, Spanish), and presence of only hashtags.    | Sticky Note2 in workflow                                                                              |
| Usernames collected are deduplicated, then detailed profile scraping is done, with filtering by follower count.      | Sticky Note3 in workflow                                                                              |
| Workflow overview summarizing entire process from input to qualified leads.                                          | Sticky Note (workflow overview)                                                                       |
| Apify actors used: Instagram hashtag scraper (`culc72xb7MP3EbaeX`) and Instagram profile scraper (`dSCLg0C3YEZ83HzYX`). | Docs: https://docs.apify.com/actors/ and https://apify.com/store                                         |
| Ensure API keys and OAuth credentials are set securely and have appropriate permissions.                             | General best practice for API integrations                                                            |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, a workflow automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected material. All processed data is legal and publicly accessible.