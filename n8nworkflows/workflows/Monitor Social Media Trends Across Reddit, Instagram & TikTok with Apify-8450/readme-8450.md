Monitor Social Media Trends Across Reddit, Instagram & TikTok with Apify

https://n8nworkflows.xyz/workflows/monitor-social-media-trends-across-reddit--instagram---tiktok-with-apify-8450


# Monitor Social Media Trends Across Reddit, Instagram & TikTok with Apify

### 1. Workflow Overview

This workflow automates the monitoring and ranking of social media trends related to the keyword "trottinette" across three platforms: Reddit, Instagram, and TikTok. It leverages Apify's scraping services to gather recent and relevant posts, calculates engagement scores based on platform-specific metrics, integrates the data into a unified ranking, and finally generates and sends a professional HTML dashboard via email summarizing the top content and engagement statistics.

The workflow is logically organized into six main functional blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow execution on a recurring schedule.
- **1.2 Reddit Data Collection and Processing:** Requests Reddit data scraping, waits for processing, retrieves data, and ranks posts by engagement.
- **1.3 Instagram Data Collection and Processing:** Similar to Reddit, but for Instagram posts under specified hashtags.
- **1.4 TikTok Data Collection and Processing:** Scrapes and ranks TikTok videos with hashtag filtering and advanced scoring.
- **1.5 Cross-Platform Data Integration and Ranking:** Combines and ranks top posts from all platforms into a unified top 15 list.
- **1.6 Dashboard Generation and Delivery:** Creates an HTML email dashboard with the ranking and statistics, then sends it via Gmail.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Scheduled Trigger

- **Overview:**  
  Automatically triggers the entire workflow at configured intervals to ensure regular monitoring without manual intervention.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  
  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Configuration: Default interval set (likely daily or periodic, exact timing not specified)  
    - Input: None (start node)  
    - Output: Triggers "Reddit Post" node to start the scraping chain  
    - Edge Cases: Misconfiguration could lead to no triggering; system time issues might cause skipped runs.  
    - Notes: Initiates the whole social media scraping and analysis cycle.

---

#### 2.2 Reddit Data Collection and Processing

- **Overview:**  
  Sends a POST request to Apify’s Reddit scraper to search for posts containing "trottinette" over the past month, waits for data generation, retrieves the data, filters and scores posts, and ranks the top 10 by engagement.

- **Nodes Involved:**  
  - Reddit Post  
  - Reddit Wait  
  - Reddit Get  
  - Sort Reddit

- **Node Details:**

  - **Reddit Post**  
    - Type: HTTP Request  
    - Configuration: POST request to Apify Reddit scraper  
      - Search keyword: "trottinette"  
      - Search posts only (no comments, communities, users)  
      - Sort by top posts in the last month  
      - Limits: max 50 items, max 25 posts, max 5 comments per post  
      - Uses Apify residential proxy  
      - Headers: Content-Type application/json  
      - Auth: Apify API token via HTTP Header  
    - Input: Schedule Trigger output  
    - Output: JSON response including dataset ID for retrieval  
    - Edge Cases: API timeout, rate limiting, proxy failure, malformed response

  - **Reddit Wait**  
    - Type: Wait  
    - Configuration: Waits 30 seconds to allow Apify to process data  
    - Input: Reddit Post output  
    - Output: Triggers Reddit Get  
    - Edge Cases: Insufficient wait time might cause incomplete data retrieval

  - **Reddit Get**  
    - Type: HTTP Request  
    - Configuration: GET request to Apify dataset URL from Reddit Post response  
      - URL constructed with dataset ID from previous response  
      - Auth: Apify HTTP Header  
    - Input: Reddit Wait output  
    - Output: Raw Reddit posts data  
    - Edge Cases: Dataset ID missing or expired, API errors

  - **Sort Reddit**  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Filters only posts (excludes comments)  
      - Scores posts: 1 point per upvote + 2 points per comment  
      - Sorts descending by score  
      - Returns top 10 with rank  
      - Includes debug logs for visibility  
    - Input: Reddit Get output  
    - Output: Ranked Reddit posts with simplified JSON  
    - Edge Cases: Missing fields (upVotes, numberOfComments), empty dataset

---

#### 2.3 Instagram Data Collection and Processing

- **Overview:**  
  Runs Apify Instagram scraper for posts tagged with "trottinette" from the last 7 days, waits for data, retrieves it, filters main posts, and ranks them by engagement (likes and comments).

- **Nodes Involved:**  
  - Insta Post  
  - Insta Wait  
  - Insta Get  
  - Sort Instagram

- **Node Details:**

  - **Insta Post**  
    - Type: HTTP Request  
    - Configuration: POST to Apify Instagram scraper  
      - Direct URL to hashtag page for "trottinette"  
      - Results type: posts  
      - Limit: 15 posts  
      - Only posts newer than 7 days  
      - Headers: Content-Type application/json  
      - Auth: Apify HTTP Header  
    - Input: Sort Reddit output (chain sequence)  
    - Output: JSON response with dataset ID  
    - Edge Cases: API errors, rate limits, no recent posts

  - **Insta Wait**  
    - Type: Wait  
    - Configuration: 30-second wait for data readiness  
    - Input: Insta Post output  
    - Output: Insta Get  
    - Edge Cases: Insufficient wait time

  - **Insta Get**  
    - Type: HTTP Request  
    - Configuration: GET dataset items using dataset ID  
      - Auth: Apify HTTP Header  
    - Input: Insta Wait output  
    - Output: Raw Instagram posts data  
    - Edge Cases: Dataset access issues

  - **Sort Instagram**  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Filters posts excluding carousel items and non-posts  
      - Scores: likes × 1 + comments × 2  
      - Sorts top 10 posts descending by score  
      - Returns ranked posts with rank  
      - Includes debug logs  
    - Input: Insta Get output  
    - Output: Ranked Instagram posts  
    - Edge Cases: Missing likes or comments counts

---

#### 2.4 TikTok Data Collection and Processing

- **Overview:**  
  Initiates TikTok hashtag scraping for "trottinette" content from the last 7 days, waits for data, retrieves it, calculates an advanced engagement score considering likes, comments, shares, and views, and ranks the top 10 videos.

- **Nodes Involved:**  
  - Tiktok Post  
  - TikTok Wait  
  - Tiktok Get  
  - Sort TikTok

- **Node Details:**

  - **Tiktok Post**  
    - Type: HTTP Request  
    - Configuration: POST to Apify TikTok scraper  
      - Hashtag: "trottinette"  
      - Results per page: 20  
      - Oldest post date: 7 days ago  
      - Headers: Content-Type application/json  
      - Auth: Apify HTTP Header  
    - Input: Sort Instagram output  
    - Output: JSON with dataset ID  
    - Edge Cases: API errors, rate limiting, no recent posts

  - **TikTok Wait**  
    - Type: Wait  
    - Configuration: 30 seconds delay  
    - Input: Tiktok Post output  
    - Output: Tiktok Get  
    - Edge Cases: Insufficient wait time

  - **Tiktok Get**  
    - Type: HTTP Request  
    - Configuration: GET dataset items by dataset ID  
      - Auth: Apify HTTP Header  
    - Input: TikTok Wait output  
    - Output: Raw TikTok posts data  
    - Edge Cases: Dataset not ready, API failure

  - **Sort TikTok**  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Extracts posts from response JSON  
      - Scores: likes ×1 + comments ×2 + shares ×3 + views/1000  
      - Filters posts with valid URLs  
      - Sorts descending by score  
      - Returns top 10 ranked posts  
      - Debug logs included  
    - Input: Tiktok Get output  
    - Output: Ranked TikTok posts  
    - Edge Cases: Missing metrics, empty data

---

#### 2.5 Cross-Platform Data Integration and Unified Ranking

- **Overview:**  
  Consolidates the top posts from Reddit, Instagram, and TikTok, standardizes author and content fields, assigns engagement levels based on score thresholds, sorts all posts together, and selects the top 15 for final reporting.

- **Nodes Involved:**  
  - Classement global

- **Node Details:**

  - **Classement global**  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Inputs: outputs from Sort Reddit, Sort Instagram, Sort TikTok  
      - Extracts author information from URLs per platform using regex  
      - Assigns levels based on score: High (≥10,000), Medium (≥1,000), Low (<1,000)  
      - Truncates content previews to 50 characters  
      - Combines all posts into one array  
      - Sorts all posts by score descending  
      - Assigns unified rank (1 to 15)  
      - Logs detailed summary and statistics by platform and level  
      - Returns top 15 unified posts with fields: source, author, content, score, level, url, rank  
    - Input: Sorted posts from all three platforms  
    - Output: Unified ranking list  
    - Edge Cases: Missing or malformed URLs, null scores, empty inputs

---

#### 2.6 Dashboard Generation and Delivery

- **Overview:**  
  Builds a professionally styled HTML email dashboard summarizing hashtag scores by platform and detailed top posts with engagement levels, then sends the email using Gmail OAuth2.

- **Nodes Involved:**  
  - Send a message

- **Node Details:**

  - **Send a message**  
    - Type: Gmail (OAuth2)  
    - Configuration:  
      - Recipient: configured email address ("Your cool mail")  
      - Subject: "Social media monitoring"  
      - Message: Full HTML content including:  
        - Summary table of total scores by platform for hashtag "trottinette"  
        - Interactive table of top 15 posts with clickable rows to original URLs  
        - Visual badges for source and engagement level  
        - Responsive design and modern styling  
        - Timestamp with generation date/time in French locale  
      - Input: Classement global output for data binding inside HTML expressions  
      - Executes once per workflow run  
    - Output: Email sent confirmation  
    - Edge Cases: Gmail OAuth2 credential expiration, email sending failures, malformed HTML

---

### 3. Summary Table

| Node Name       | Node Type           | Functional Role                             | Input Node(s)       | Output Node(s)      | Sticky Note                                                                                                  |
|-----------------|---------------------|---------------------------------------------|---------------------|---------------------|--------------------------------------------------------------------------------------------------------------|
| Schedule Trigger| Schedule Trigger    | Initiates workflow on schedule               | None                | Reddit Post         | # Phase 1: Automated Schedule Activation                                                                     |
| Reddit Post     | HTTP Request        | Sends Reddit scraping request to Apify       | Schedule Trigger    | Reddit Wait         | Post the request on apify                                                                                     |
| Reddit Wait     | Wait                | Pauses for data generation                    | Reddit Post         | Reddit Get          | Wait for the package to generate                                                                              |
| Reddit Get      | HTTP Request        | Retrieves Reddit scraped data                  | Reddit Wait         | Sort Reddit         | Get the package with the info                                                                                 |
| Sort Reddit     | Code                | Scores and ranks Reddit posts                  | Reddit Get          | Insta Post          | Sorts posts                                                                                                   |
| Insta Post      | HTTP Request        | Sends Instagram scraping request to Apify     | Sort Reddit         | Insta Wait          | Post the request on apify                                                                                     |
| Insta Wait      | Wait                | Pauses for Instagram data generation           | Insta Post          | Insta Get           | Wait for the package to generate                                                                              |
| Insta Get       | HTTP Request        | Retrieves Instagram scraped data                | Insta Wait          | Sort Instagram      | Get the package with the info                                                                                 |
| Sort Instagram  | Code                | Scores and ranks Instagram posts                | Insta Get           | Tiktok Post         | Sorts posts                                                                                                   |
| Tiktok Post     | HTTP Request        | Sends TikTok scraping request to Apify         | Sort Instagram      | TikTok Wait         | Post the request on apify                                                                                     |
| TikTok Wait     | Wait                | Pauses for TikTok data generation               | Tiktok Post         | Tiktok Get          | Wait for the package to generate                                                                              |
| Tiktok Get      | HTTP Request        | Retrieves TikTok scraped data                    | TikTok Wait         | Sort TikTok         | Get the package with the info                                                                                 |
| Sort TikTok     | Code                | Scores and ranks TikTok posts                    | Tiktok Get          | Classement global    | Sorts posts                                                                                                   |
| Classement global| Code                | Integrates and ranks posts across all platforms | Sort TikTok         | Send a message      | Sorts all posts + give them points depending on their likes + comments                                        |
| Send a message  | Gmail               | Sends HTML email dashboard with analytics      | Classement global   | None                | Send dahsboard with top 10 of the most engaging post                                                         |
| Sticky Note1    | Sticky Note         | Section label for Reddit                         | None                | None                | # Reddit                                                                                                      |
| Sticky Note2    | Sticky Note         | Section label for Instagram                      | None                | None                | # Instagram                                                                                                   |
| Sticky Note3    | Sticky Note         | Section label for TikTok                         | None                | None                | # TikTok                                                                                                      |
| Sticky Note4    | Sticky Note         | Phase 1 description                              | None                | None                | # Phase 1: Automated Schedule Activation                                                                      |
| Sticky Note5    | Sticky Note         | Phase 1 details and results                      | None                | None                | ### What the system does and Result for Phase 1 (Schedule Trigger)                                           |
| Sticky Note6    | Sticky Note         | Phase 2 description                              | None                | None                | # Phase 2: Reddit Content Discovery and Processing                                                            |
| Sticky Note7    | Sticky Note         | Phase 2 details and results                       | None                | None                | ### What the system does and Result for Reddit processing                                                    |
| Sticky Note8    | Sticky Note         | Phase 3 description                              | None                | None                | # Phase 3: Instagram Content Discovery and Processing                                                         |
| Sticky Note9    | Sticky Note         | Phase 3 details and results                       | None                | None                | ### What the system does and Result for Instagram processing                                                 |
| Sticky Note10   | Sticky Note         | Phase 4 description                              | None                | None                | # Phase 4: TikTok Content Discovery and Processing                                                            |
| Sticky Note11   | Sticky Note         | Phase 4 details and results                       | None                | None                | ### What the system does and Result for TikTok processing                                                    |
| Sticky Note12   | Sticky Note         | Phase 5 description                              | None                | None                | # Phase 5: Cross-Platform Data Integration and Unified Ranking                                                |
| Sticky Note13   | Sticky Note         | Phase 5 details and results                       | None                | None                | ### What the system does and Result for cross-platform integration                                           |
| Sticky Note14   | Sticky Note         | Phase 6 description                              | None                | None                | # Phase 6: Professional Analytics Dashboard Generation and Delivery                                          |
| Sticky Note15   | Sticky Note         | Phase 6 details and results                       | None                | None                | ### What the system does and Result for dashboard creation and delivery                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set desired recurrence interval (e.g., daily at specific time)

2. **Create Reddit Post HTTP Request node**  
   - URL: `https://api.apify.com/v2/acts/trudax~reddit-scraper-lite/runs`  
   - Method: POST  
   - Body: JSON with parameters:  
     ```json
     {
       "searches": ["trottinette"],
       "searchPosts": true,
       "searchComments": false,
       "searchCommunities": false,
       "searchUsers": false,
       "sort": "top",
       "time": "month",
       "maxItems": 50,
       "maxPostCount": 25,
       "maxComments": 5,
       "scrollTimeout": 40,
       "proxy": {
         "useApifyProxy": true,
         "apifyProxyGroups": ["RESIDENTIAL"]
       }
     }
     ```  
   - Headers: Content-Type: application/json  
   - Authentication: HTTP Header Auth with Apify API token  
   - Connect Schedule Trigger output to this node's input

3. **Create Reddit Wait node**  
   - Type: Wait  
   - Duration: 30 seconds  
   - Connect Reddit Post output to Reddit Wait input

4. **Create Reddit Get HTTP Request node**  
   - URL: Use expression `https://api.apify.com/v2/datasets/{{ $json.data.defaultDatasetId }}/items`  
   - Method: GET  
   - Authentication: HTTP Header Auth with Apify token  
   - Connect Reddit Wait output to Reddit Get input

5. **Create Sort Reddit Code node**  
   - Type: Code  
   - Paste JavaScript code that:  
     - Filters posts (json.dataType === 'post')  
     - Scores: upvotes + 2 × comments  
     - Sorts descending, top 10  
     - Adds rank  
   - Connect Reddit Get output to this node

6. **Create Insta Post HTTP Request node**  
   - URL: `https://api.apify.com/v2/acts/apify~instagram-scraper/runs`  
   - Method: POST  
   - Body JSON:  
     ```json
     {
       "directUrls": ["https://www.instagram.com/explore/tags/trottinette/"],
       "resultsType": "posts",
       "resultsLimit": 15,
       "onlyPostsNewerThan": "7 days"
     }
     ```  
   - Headers: Content-Type: application/json  
   - Authentication: HTTP Header Auth with Apify token  
   - Connect Sort Reddit output to Insta Post input

7. **Create Insta Wait node**  
   - Type: Wait  
   - Duration: 30 seconds  
   - Connect Insta Post output to Insta Wait input

8. **Create Insta Get HTTP Request node**  
   - URL: Expression `https://api.apify.com/v2/datasets/{{ $json.data.defaultDatasetId }}/items`  
   - Method: GET  
   - Authentication: HTTP Header Auth with Apify token  
   - Connect Insta Wait output to Insta Get input

9. **Create Sort Instagram Code node**  
   - Type: Code  
   - Paste JavaScript code that:  
     - Filters main posts (exclude carousel items)  
     - Scores: likes + 2 × comments  
     - Sorts descending, top 10  
     - Adds rank  
   - Connect Insta Get output to this node

10. **Create Tiktok Post HTTP Request node**  
    - URL: `https://api.apify.com/v2/acts/clockworks~tiktok-scraper/runs`  
    - Method: POST  
    - Body JSON:  
      ```json
      {
        "hashtags": ["trottinette"],
        "resultsPerPage": 20,
        "oldestPostDateUnified": "7 days"
      }
      ```  
    - Headers: Content-Type: application/json  
    - Authentication: HTTP Header Auth with Apify token  
    - Connect Sort Instagram output to Tiktok Post input

11. **Create TikTok Wait node**  
    - Type: Wait  
    - Duration: 30 seconds  
    - Connect Tiktok Post output to TikTok Wait input

12. **Create Tiktok Get HTTP Request node**  
    - URL: Expression `https://api.apify.com/v2/datasets/{{ $json.data.defaultDatasetId }}/items`  
    - Method: GET  
    - Authentication: HTTP Header Auth with Apify token  
    - Connect TikTok Wait output to Tiktok Get input

13. **Create Sort TikTok Code node**  
    - Type: Code  
    - Paste JavaScript code that:  
      - Extracts posts from JSON array  
      - Scores: likes + 2×comments + 3×shares + views/1000  
      - Filters posts with URLs  
      - Sorts descending, top 10  
      - Adds rank  
    - Connect Tiktok Get output to this node

14. **Create Classement global Code node**  
    - Type: Code  
    - Paste JavaScript code that:  
      - Receives inputs from Sort Reddit, Sort Instagram, Sort TikTok  
      - Normalizes author, content, score, source  
      - Assigns levels based on score thresholds  
      - Combines all posts into a unified list  
      - Sorts and ranks top 15 posts  
      - Adds debug logs  
    - Connect Sort TikTok output to Classement global input  
    - Also connect Sort Reddit and Sort Instagram outputs as additional inputs to Classement global (multi-input support)

15. **Create Send a message Gmail node**  
    - Type: Gmail (OAuth2)  
    - Recipient: your desired email address  
    - Subject: "Social media monitoring"  
    - Message content: Paste the full HTML dashboard template with embedded expressions to consume Classement global data  
    - Authentication: Configure Gmail OAuth2 credentials  
    - Connect Classement global output to this node

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                                                                 |
|------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow leverages Apify actors: Reddit Scraper Lite, Instagram Scraper, TikTok Scraper                     | Apify platform: https://apify.com/                                                             |
| Engagement scoring formulas customized per platform to reflect meaningful interactions                     | Detailed in sticky notes and code comments                                                      |
| Dashboard HTML uses responsive and modern design with inline CSS for professional email presentation       | Suitable for desktop and mobile email clients                                                   |
| Gmail OAuth2 credential must be set up with appropriate scopes for sending emails                           | See n8n documentation for Gmail OAuth2 setup                                                   |
| Wait nodes with 30 seconds delay ensure data availability before retrieval; time may be adjusted as needed | Consider increasing delay if API response times vary or data volume grows                       |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow built with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.