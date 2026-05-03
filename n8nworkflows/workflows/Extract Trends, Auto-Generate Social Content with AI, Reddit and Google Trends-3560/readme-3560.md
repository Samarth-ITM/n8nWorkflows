Extract Trends, Auto-Generate Social Content with AI, Reddit and Google Trends

https://n8nworkflows.xyz/workflows/extract-trends--auto-generate-social-content-with-ai--reddit-and-google-trends-3560


# Extract Trends, Auto-Generate Social Content with AI, Reddit and Google Trends

### 1. Workflow Overview

This workflow automates the extraction of trending topics from multiple sources (Reddit, Twitter/X, Google Trends, and news), scores these trends for relevance using OpenAI's GPT models, and generates tailored social media content for various platforms. It then posts the generated content to LinkedIn, Instagram, and Facebook, while logging and storing trend data and posting results in Google Sheets for tracking.

The workflow is logically divided into two main blocks:

- **1.1 Trend Extraction and Scoring**  
  Fetches trends and user-generated content (UGC) from Reddit, Twitter/X, Google Trends, and news sources; processes and merges this data; uses OpenAI to score trends for relevance; and stores the scored trends in Google Sheets.

- **1.2 Content Generation and Posting**  
  Retrieves the latest scored trends from Google Sheets, selects the top trend, generates platform-specific social media content using OpenAI, optionally generates images, and posts the content to LinkedIn, Instagram, and Facebook.

---

### 2. Block-by-Block Analysis

#### 2.1 Trend Extraction and Scoring

**Overview:**  
This block gathers trending data from multiple sources, processes and merges it, analyzes audience mood and news sentiment using AI, scores the trends for relevance, and stores the results for later use.

**Nodes Involved:**  
- Daily Trigger Idea  
- Set Default Inputs  
- Fetch Breaking News  
- Extract News Headlines  
- Analyze News Sentiment  
- Fix News Sentiment Output  
- Fetch Twitter Mentions  
- Translate Tweets to English  
- Fix Translated Tweets Output  
- Detect Audience Mood  
- Fix Audience Mood Output  
- Fetch Twitter Trends  
- Posts from Reddit  
- Log Reddit Response  
- Exctract Reddit Trends  
- SERP-Google Trends  
- If Fallback  
- Extract Google Trends2  
- Extract Google Trends  
- Google Trends Fallback  
- Notify Google Trends Failure  
- Combine All Data  
- Merge Items into Single Item  
- Combine Trends and UGCc  
- Prepare Trend Scoring Input  
- Trend Relevance Scoring (Wow Factor)  
- Parse Trend Scores  
- Store Selected Trend

**Node Details:**

- **Daily Trigger Idea**  
  - Type: Schedule Trigger  
  - Role: Initiates the workflow daily.  
  - Configuration: Runs at a default interval (daily).  
  - Inputs: None  
  - Outputs: Set Default Inputs  
  - Failures: None expected unless scheduling misconfigured.

- **Set Default Inputs**  
  - Type: Set  
  - Role: Initializes default parameters such as `brand_voice` ("funny"), `platforms` (list of social platforms), and `campaign_type` ("standard").  
  - Inputs: Trigger node  
  - Outputs: Multiple trend fetching nodes  
  - Failures: None expected.

- **Fetch Breaking News**  
  - Type: HTTP Request  
  - Role: Fetches top technology news headlines from NewsAPI.  
  - Configuration: Authenticated with NewsAPI credentials; queries category "technology" and language "en".  
  - Inputs: Set Default Inputs  
  - Outputs: Extract News Headlines  
  - Failures: API key invalid, rate limits, network errors.

- **Extract News Headlines**  
  - Type: Set  
  - Role: Extracts news headlines from the API response into a simplified array of titles.  
  - Inputs: Fetch Breaking News  
  - Outputs: Analyze News Sentiment  
  - Failures: Missing or malformed API response.

- **Analyze News Sentiment**  
  - Type: OpenAI (LangChain)  
  - Role: Uses GPT-4o-mini to analyze overall sentiment of news headlines, classifying as positive, negative, or neutral.  
  - Inputs: Extract News Headlines  
  - Outputs: Fix News Sentiment Output  
  - Failures: OpenAI API errors, prompt failures.

- **Fix News Sentiment Output**  
  - Type: Set  
  - Role: Extracts and normalizes the sentiment result from OpenAI's response.  
  - Inputs: Analyze News Sentiment  
  - Outputs: Combine All Data  
  - Failures: Unexpected response structure.

- **Fetch Twitter Mentions**  
  - Type: HTTP Request  
  - Role: Fetches recent mentions of a specified Twitter user (e.g., "ewaraha") using Twitter API.  
  - Configuration: Uses Twitter OAuth1 credentials.  
  - Inputs: Set Default Inputs  
  - Outputs: Translate Tweets to English  
  - Failures: Auth errors, rate limits.

- **Translate Tweets to English**  
  - Type: OpenAI (LangChain)  
  - Role: Translates fetched tweets (possibly in Thai) to English using GPT-4o-mini.  
  - Inputs: Fetch Twitter Mentions  
  - Outputs: Fix Translated Tweets Output  
  - Failures: OpenAI API errors, translation inaccuracies.

- **Fix Translated Tweets Output**  
  - Type: Set  
  - Role: Normalizes the translated tweets output from OpenAI.  
  - Inputs: Translate Tweets to English  
  - Outputs: Detect Audience Mood  
  - Failures: Unexpected response format.

- **Detect Audience Mood**  
  - Type: OpenAI (LangChain)  
  - Role: Analyzes translated tweets to detect audience mood (e.g., excited, frustrated).  
  - Inputs: Fix Translated Tweets Output  
  - Outputs: Fix Audience Mood Output  
  - Failures: OpenAI API errors.

- **Fix Audience Mood Output**  
  - Type: Set  
  - Role: Normalizes audience mood output from OpenAI and passes translated tweets forward.  
  - Inputs: Detect Audience Mood  
  - Outputs: Combine All Data  
  - Failures: Unexpected response format.

- **Fetch Twitter Trends**  
  - Type: HTTP Request  
  - Role: Fetches latest tweets matching a query ("manus") from Twitter API.  
  - Inputs: Set Default Inputs  
  - Outputs: Combine All Data  
  - Failures: Auth errors, rate limits.

- **Posts from Reddit**  
  - Type: Reddit Node  
  - Role: Fetches top "hot" posts from the "technology" subreddit.  
  - Inputs: Set Default Inputs  
  - Outputs: Log Reddit Response  
  - Failures: Reddit API auth errors, rate limits.

- **Log Reddit Response**  
  - Type: Code  
  - Role: Logs Reddit response data for debugging.  
  - Inputs: Posts from Reddit  
  - Outputs: Exctract Reddit Trends  
  - Failures: None expected.

- **Exctract Reddit Trends**  
  - Type: Code  
  - Role: Extracts titles and scores from Reddit posts into a structured array.  
  - Inputs: Log Reddit Response  
  - Outputs: Combine All Data  
  - Failures: Unexpected data format.

- **SERP-Google Trends**  
  - Type: HTTP Request  
  - Role: Fetches trending topics from Google Trends via SerpApi.  
  - Configuration: Uses SerpApi credentials; geo set to US.  
  - Inputs: Set Default Inputs  
  - Outputs: If Fallback  
  - Failures: API key invalid, rate limits.

- **If Fallback**  
  - Type: If  
  - Role: Checks if Google Trends API response status is "Success".  
  - Inputs: SERP-Google Trends  
  - Outputs: Extract Google Trends2 (on success), Google Trends Fallback (on failure)  
  - Failures: Logic errors if response structure changes.

- **Extract Google Trends2**  
  - Type: Code  
  - Role: Extracts trending search queries and traffic volume from Google Trends response.  
  - Inputs: If Fallback (success branch)  
  - Outputs: Extract Google Trends  
  - Failures: Missing fields or unexpected response format.

- **Extract Google Trends**  
  - Type: Set  
  - Role: Prepares extracted Google Trends data for merging.  
  - Inputs: Extract Google Trends2  
  - Outputs: Combine All Data  
  - Failures: None expected.

- **Google Trends Fallback**  
  - Type: Set  
  - Role: Provides an empty fallback array for Google Trends data if API call fails.  
  - Inputs: If Fallback (failure branch)  
  - Outputs: Notify Google Trends Failure  
  - Failures: None expected.

- **Notify Google Trends Failure**  
  - Type: Gmail  
  - Role: Sends an email notification if Google Trends API fails.  
  - Inputs: Google Trends Fallback  
  - Outputs: None (end)  
  - Failures: Gmail auth errors, email delivery issues.

- **Combine All Data**  
  - Type: Merge  
  - Role: Merges data from multiple sources (news sentiment, audience mood, tweets, Reddit trends, Google Trends, Twitter mentions).  
  - Inputs: Fetch Twitter Trends, Fix Audience Mood Output, Fix News Sentiment Output, Extract Google Trends, Exctract Reddit Trends  
  - Outputs: Merge Items into Single Item  
  - Failures: Data mismatch or missing inputs.

- **Merge Items into Single Item**  
  - Type: Code  
  - Role: Combines multiple JSON objects into a single unified JSON object, normalizing fields like `audience_mood`.  
  - Inputs: Combine All Data  
  - Outputs: Combine Trends and UGCc  
  - Failures: Unexpected data structures.

- **Combine Trends and UGCc**  
  - Type: Code  
  - Role: Combines all trends and user-generated content into a unified array with consistent naming; determines `brand_voice` based on `audience_mood`.  
  - Inputs: Merge Items into Single Item  
  - Outputs: Prepare Trend Scoring Input  
  - Failures: Missing or malformed data arrays.

- **Prepare Trend Scoring Input**  
  - Type: Code  
  - Role: Simplifies audience mood and prepares data for trend scoring, ensuring defaults for missing values.  
  - Inputs: Combine Trends and UGCc  
  - Outputs: Trend Relevance Scoring (Wow Factor)  
  - Failures: None expected.

- **Trend Relevance Scoring (Wow Factor)**  
  - Type: OpenAI (LangChain)  
  - Role: Uses GPT-4o-mini to score each trend's relevance (1-10) for social media campaigns based on brand voice and audience mood.  
  - Inputs: Prepare Trend Scoring Input  
  - Outputs: Parse Trend Scores  
  - Failures: OpenAI API errors, prompt failures.

- **Parse Trend Scores**  
  - Type: Code  
  - Role: Parses OpenAI's JSON response to extract trend scores into a structured array; provides fallback if no trends found.  
  - Inputs: Trend Relevance Scoring (Wow Factor)  
  - Outputs: Store Selected Trend  
  - Failures: Parsing errors, unexpected response format.

- **Store Selected Trend**  
  - Type: Google Sheets  
  - Role: Appends scored trends, brand voice, audience mood, and timestamp to a Google Sheets document for tracking.  
  - Inputs: Parse Trend Scores  
  - Outputs: Retrieve Latest Trend Scores  
  - Failures: Google Sheets API errors, permission issues.

---

#### 2.2 Content Generation and Posting

**Overview:**  
This block retrieves the latest scored trends, selects the top trend, generates social media content tailored to multiple platforms using OpenAI, optionally generates images, and posts the content to LinkedIn, Instagram, and Facebook.

**Nodes Involved:**  
- Retrieve Latest Trend Scores  
- Parse Retrieved Trend Scores  
- Select Top Trends  
- Generate Social Media Content  
- Parse Social Media Content  
- Generate Images  
- Post to LinkedIn  
- Post to Instagram  
- Post to Facebook

**Node Details:**

- **Retrieve Latest Trend Scores**  
  - Type: Google Sheets  
  - Role: Reads the latest trend scores and associated metadata from Google Sheets.  
  - Inputs: Store Selected Trend  
  - Outputs: Parse Retrieved Trend Scores  
  - Failures: Google Sheets API errors, permission issues.

- **Parse Retrieved Trend Scores**  
  - Type: Code  
  - Role: Parses the Google Sheets row data, splitting semicolon-separated trends and scores into arrays and preparing for selection.  
  - Inputs: Retrieve Latest Trend Scores  
  - Outputs: Select Top Trends  
  - Failures: Malformed sheet data.

- **Select Top Trends**  
  - Type: Code  
  - Role: Selects the trend with the highest score from the parsed trend scores; provides defaults if none available.  
  - Inputs: Parse Retrieved Trend Scores  
  - Outputs: Generate Social Media Content  
  - Failures: Empty or missing trend scores.

- **Generate Social Media Content**  
  - Type: OpenAI (LangChain)  
  - Role: Generates platform-specific social media content (Twitter/X, LinkedIn, TikTok, YouTube Shorts) based on the selected trend, brand voice, and audience mood.  
  - Inputs: Select Top Trends  
  - Outputs: Parse Social Media Content  
  - Failures: OpenAI API errors, prompt failures.

- **Parse Social Media Content**  
  - Type: Code  
  - Role: Parses the OpenAI-generated JSON content; provides fallback content if parsing fails or required fields are missing.  
  - Inputs: Generate Social Media Content  
  - Outputs: Generate Images, Post to LinkedIn, Post to Instagram  
  - Failures: JSON parsing errors.

- **Generate Images**  
  - Type: OpenAI (LangChain) Image Generation  
  - Role: Generates images based on TikTok and YouTube Shorts scripts for enhanced social media posts.  
  - Inputs: Parse Social Media Content  
  - Outputs: None (end)  
  - Failures: OpenAI API errors, image generation limits.

- **Post to LinkedIn**  
  - Type: LinkedIn Node  
  - Role: Posts generated LinkedIn content as an organization post.  
  - Inputs: Parse Social Media Content  
  - Outputs: None (end)  
  - Failures: LinkedIn API auth errors, posting limits.

- **Post to Instagram**  
  - Type: Facebook Graph API Node  
  - Role: Posts generated content/photos to Instagram via Facebook Graph API.  
  - Inputs: Parse Social Media Content  
  - Outputs: None (end)  
  - Failures: Facebook/Instagram API auth errors, permission issues.

- **Post to Facebook**  
  - *Note:* Although mentioned in the description, no explicit node for posting to Facebook is present in the provided workflow JSON. This may require manual addition or is handled externally.

---

### 3. Summary Table

| Node Name                    | Node Type                      | Functional Role                              | Input Node(s)                 | Output Node(s)                    | Sticky Note                                                                                      |
|------------------------------|--------------------------------|----------------------------------------------|------------------------------|----------------------------------|-------------------------------------------------------------------------------------------------|
| Daily Trigger Idea            | Schedule Trigger               | Triggers workflow daily                      | None                         | Set Default Inputs               |                                                                                                 |
| Set Default Inputs            | Set                           | Sets default parameters (brand voice, etc.) | Daily Trigger Idea            | Fetch Breaking News, Fetch Twitter Mentions, Fetch Twitter Trends, SERP-Google Trends, Posts from Reddit |                                                                                                 |
| Fetch Breaking News           | HTTP Request                  | Fetches top technology news headlines        | Set Default Inputs            | Extract News Headlines           |                                                                                                 |
| Extract News Headlines        | Set                           | Extracts news headlines from API response    | Fetch Breaking News           | Analyze News Sentiment           |                                                                                                 |
| Analyze News Sentiment        | OpenAI (LangChain)            | Analyzes sentiment of news headlines          | Extract News Headlines        | Fix News Sentiment Output        |                                                                                                 |
| Fix News Sentiment Output     | Set                           | Normalizes news sentiment output              | Analyze News Sentiment        | Combine All Data                 |                                                                                                 |
| Fetch Twitter Mentions        | HTTP Request                  | Fetches Twitter mentions for user             | Set Default Inputs            | Translate Tweets to English      |                                                                                                 |
| Translate Tweets to English   | OpenAI (LangChain)            | Translates tweets to English                   | Fetch Twitter Mentions        | Fix Translated Tweets Output     |                                                                                                 |
| Fix Translated Tweets Output  | Set                           | Normalizes translated tweets output           | Translate Tweets to English   | Detect Audience Mood             |                                                                                                 |
| Detect Audience Mood          | OpenAI (LangChain)            | Detects audience mood from tweets             | Fix Translated Tweets Output  | Fix Audience Mood Output         |                                                                                                 |
| Fix Audience Mood Output      | Set                           | Normalizes audience mood output                | Detect Audience Mood          | Combine All Data                 |                                                                                                 |
| Fetch Twitter Trends          | HTTP Request                  | Fetches latest tweets matching query          | Set Default Inputs            | Combine All Data                 |                                                                                                 |
| Posts from Reddit             | Reddit Node                   | Fetches hot posts from subreddit               | Set Default Inputs            | Log Reddit Response             |                                                                                                 |
| Log Reddit Response           | Code                          | Logs Reddit response for debugging             | Posts from Reddit             | Exctract Reddit Trends           |                                                                                                 |
| Exctract Reddit Trends        | Code                          | Extracts titles and scores from Reddit posts  | Log Reddit Response           | Combine All Data                 |                                                                                                 |
| SERP-Google Trends            | HTTP Request                  | Fetches Google Trends data via SerpApi        | Set Default Inputs            | If Fallback                    |                                                                                                 |
| If Fallback                  | If                            | Checks Google Trends API success               | SERP-Google Trends            | Extract Google Trends2 / Google Trends Fallback |                                                                                                 |
| Extract Google Trends2        | Code                          | Extracts trending queries and volumes          | If Fallback (success)         | Extract Google Trends            |                                                                                                 |
| Extract Google Trends         | Set                           | Prepares Google Trends data                     | Extract Google Trends2        | Combine All Data                 |                                                                                                 |
| Google Trends Fallback        | Set                           | Provides empty fallback for failed Google Trends | If Fallback (failure)         | Notify Google Trends Failure     |                                                                                                 |
| Notify Google Trends Failure  | Gmail                         | Sends email notification on Google Trends failure | Google Trends Fallback        | None                           |                                                                                                 |
| Combine All Data              | Merge                         | Merges all trend and sentiment data            | Multiple (Twitter, Reddit, News, Google Trends) | Merge Items into Single Item      |                                                                                                 |
| Merge Items into Single Item  | Code                          | Combines multiple JSON objects into one        | Combine All Data              | Combine Trends and UGCc          |                                                                                                 |
| Combine Trends and UGCc       | Code                          | Combines trends and UGC into unified array     | Merge Items into Single Item  | Prepare Trend Scoring Input      |                                                                                                 |
| Prepare Trend Scoring Input   | Code                          | Prepares data for trend scoring                 | Combine Trends and UGCc       | Trend Relevance Scoring (Wow Factor) |                                                                                                 |
| Trend Relevance Scoring (Wow Factor) | OpenAI (LangChain)            | Scores trends for relevance (1-10)              | Prepare Trend Scoring Input   | Parse Trend Scores              |                                                                                                 |
| Parse Trend Scores            | Code                          | Parses OpenAI trend scoring response            | Trend Relevance Scoring       | Store Selected Trend             |                                                                                                 |
| Store Selected Trend          | Google Sheets                 | Stores scored trends in Google Sheets           | Parse Trend Scores            | Retrieve Latest Trend Scores     |                                                                                                 |
| Retrieve Latest Trend Scores  | Google Sheets                 | Retrieves latest trend scores from Google Sheets | Store Selected Trend          | Parse Retrieved Trend Scores     |                                                                                                 |
| Parse Retrieved Trend Scores  | Code                          | Parses Google Sheets data into structured array | Retrieve Latest Trend Scores  | Select Top Trends               |                                                                                                 |
| Select Top Trends             | Code                          | Selects top scoring trend                        | Parse Retrieved Trend Scores  | Generate Social Media Content    |                                                                                                 |
| Generate Social Media Content | OpenAI (LangChain)            | Generates platform-specific social media content | Select Top Trends             | Parse Social Media Content       |                                                                                                 |
| Parse Social Media Content    | Code                          | Parses OpenAI-generated social media content    | Generate Social Media Content | Generate Images, Post to LinkedIn, Post to Instagram |                                                                                                 |
| Generate Images              | OpenAI (LangChain Image)      | Generates images for TikTok and YouTube Shorts  | Parse Social Media Content    | None                           |                                                                                                 |
| Post to LinkedIn              | LinkedIn Node                 | Posts content to LinkedIn organization page     | Parse Social Media Content    | None                           |                                                                                                 |
| Post to Instagram             | Facebook Graph API            | Posts photos/content to Instagram                | Parse Social Media Content    | None                           |                                                                                                 |
| Sticky Note5                 | Sticky Note                   | Describes Trend Extraction and Scoring block    | None                         | None                           | # Trend Extraction and Scoring Workflow 🛠️ ... (see content in workflow)                         |
| Sticky Note                  | Sticky Note                   | Describes Content Generation block               | None                         | None                           | # 📋 Content Generation ... (see content in workflow)                                           |
| Sticky Note1                 | Sticky Note                   | Describes Posting to Social Media block          | None                         | None                           | # 📋 Posting to Social Media ... (see content in workflow)                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node**  
   - Name: `Daily Trigger Idea`  
   - Type: Schedule Trigger  
   - Configuration: Set to run daily (default interval).

2. **Add a Set Node**  
   - Name: `Set Default Inputs`  
   - Type: Set  
   - Assign variables:  
     - `user_input` = "" (empty string)  
     - `brand_voice` = "funny"  
     - `platforms` = `"twitter,linkedin,instagram,tiktok,youtube"`  
     - `campaign_type` = `"standard"`  
   - Connect from `Daily Trigger Idea`.

3. **Add HTTP Request Node for NewsAPI**  
   - Name: `Fetch Breaking News`  
   - Type: HTTP Request  
   - URL: `https://newsapi.org/v2/top-headlines`  
   - Query Parameters:  
     - `category` = "technology"  
     - `language` = "en"  
   - Authentication: Use Google Sheets API key or NewsAPI key stored in n8n credentials.  
   - Connect from `Set Default Inputs`.

4. **Add Set Node to Extract Headlines**  
   - Name: `Extract News Headlines`  
   - Type: Set  
   - Assign `news_headlines` to array of article titles extracted from the previous node's JSON response.  
   - Connect from `Fetch Breaking News`.

5. **Add OpenAI Node for News Sentiment**  
   - Name: `Analyze News Sentiment`  
   - Type: OpenAI (LangChain)  
   - Model: `gpt-4o-mini` or preferred GPT model  
   - Prompt: Analyze sentiment of news headlines, classify as positive, negative, or neutral.  
   - Connect from `Extract News Headlines`.

6. **Add Set Node to Fix News Sentiment Output**  
   - Name: `Fix News Sentiment Output`  
   - Type: Set  
   - Extract sentiment string from OpenAI response.  
   - Connect from `Analyze News Sentiment`.

7. **Add HTTP Request Node for Twitter Mentions**  
   - Name: `Fetch Twitter Mentions`  
   - Type: HTTP Request  
   - URL: `https://api.twitterapi.io/twitter/user/mentions`  
   - Query Parameter: `userName` = your target username (e.g., "ewaraha")  
   - Authentication: Twitter OAuth1 credentials  
   - Connect from `Set Default Inputs`.

8. **Add OpenAI Node to Translate Tweets**  
   - Name: `Translate Tweets to English`  
   - Type: OpenAI (LangChain)  
   - Model: `gpt-4o-mini`  
   - Prompt: Translate tweets from Thai to English.  
   - Connect from `Fetch Twitter Mentions`.

9. **Add Set Node to Fix Translation Output**  
   - Name: `Fix Translated Tweets Output`  
   - Type: Set  
   - Extract translated text from OpenAI response.  
   - Connect from `Translate Tweets to English`.

10. **Add OpenAI Node to Detect Audience Mood**  
    - Name: `Detect Audience Mood`  
    - Type: OpenAI (LangChain)  
    - Model: `gpt-4o-mini`  
    - Prompt: Analyze translated tweets to detect audience mood (e.g., excited, frustrated).  
    - Connect from `Fix Translated Tweets Output`.

11. **Add Set Node to Fix Audience Mood Output**  
    - Name: `Fix Audience Mood Output`  
    - Type: Set  
    - Extract audience mood string from OpenAI response and pass translated tweets forward.  
    - Connect from `Detect Audience Mood`.

12. **Add HTTP Request Node for Twitter Trends**  
    - Name: `Fetch Twitter Trends`  
    - Type: HTTP Request  
    - URL: `https://api.twitterapi.io/twitter/tweet/advanced_search`  
    - Query Parameters:  
      - `query` = "manus" (or your keyword)  
      - `queryType` = "Latest"  
      - `exclude` = "hashtags"  
    - Authentication: Twitter OAuth1 credentials  
    - Connect from `Set Default Inputs`.

13. **Add Reddit Node to Fetch Posts**  
    - Name: `Posts from Reddit`  
    - Type: Reddit  
    - Subreddit: "technology"  
    - Limit: 3  
    - Category: "hot"  
    - Authentication: Reddit OAuth2 credentials  
    - Connect from `Set Default Inputs`.

14. **Add Code Node to Log Reddit Response**  
    - Name: `Log Reddit Response`  
    - Type: Code  
    - Code: Log JSON for debugging.  
    - Connect from `Posts from Reddit`.

15. **Add Code Node to Extract Reddit Trends**  
    - Name: `Exctract Reddit Trends`  
    - Type: Code  
    - Code: Extract titles and scores from Reddit posts into array.  
    - Connect from `Log Reddit Response`.

16. **Add HTTP Request Node for Google Trends (SerpApi)**  
    - Name: `SERP-Google Trends`  
    - Type: HTTP Request  
    - URL: `https://serpapi.com/search?engine=google_trends_trending_now`  
    - Query Parameters:  
      - `engine` = "google_trends"  
      - `geo` = "US"  
      - `api_key` = Use SerpApi credentials in n8n  
      - `q` = "trending_searches"  
    - Authentication: SerpApi credentials  
    - Connect from `Set Default Inputs`.

17. **Add If Node to Check Google Trends Success**  
    - Name: `If Fallback`  
    - Type: If  
    - Condition: `$json.search_metadata.status == "Success"`  
    - Connect from `SERP-Google Trends`.

18. **Add Code Node to Extract Google Trends**  
    - Name: `Extract Google Trends2`  
    - Type: Code  
    - Code: Extract trending searches and traffic volume from Google Trends response.  
    - Connect from `If Fallback` (success branch).

19. **Add Set Node to Format Google Trends**  
    - Name: `Extract Google Trends`  
    - Type: Set  
    - Prepare extracted Google Trends data for merging.  
    - Connect from `Extract Google Trends2`.

20. **Add Set Node for Google Trends Fallback**  
    - Name: `Google Trends Fallback`  
    - Type: Set  
    - Assign empty array to `google_trends` for fallback.  
    - Connect from `If Fallback` (failure branch).

21. **Add Gmail Node to Notify on Google Trends Failure**  
    - Name: `Notify Google Trends Failure`  
    - Type: Gmail  
    - Send email alert if Google Trends API fails.  
    - Connect from `Google Trends Fallback`.

22. **Add Merge Node to Combine All Data**  
    - Name: `Combine All Data`  
    - Type: Merge  
    - Number of inputs: 5 (from Twitter Trends, Fix Audience Mood Output, Fix News Sentiment Output, Extract Google Trends, Exctract Reddit Trends)  
    - Connect respective outputs accordingly.

23. **Add Code Node to Merge Items into Single Item**  
    - Name: `Merge Items into Single Item`  
    - Type: Code  
    - Code: Combine multiple JSON objects into one, normalize fields.  
    - Connect from `Combine All Data`.

24. **Add Code Node to Combine Trends and UGC**  
    - Name: `Combine Trends and UGCc`  
    - Type: Code  
    - Code: Combine all trends and UGC into a unified array; determine brand voice based on audience mood.  
    - Connect from `Merge Items into Single Item`.

25. **Add Code Node to Prepare Trend Scoring Input**  
    - Name: `Prepare Trend Scoring Input`  
    - Type: Code  
    - Code: Simplify audience mood and prepare data for scoring.  
    - Connect from `Combine Trends and UGCc`.

26. **Add OpenAI Node for Trend Relevance Scoring**  
    - Name: `Trend Relevance Scoring (Wow Factor)`  
    - Type: OpenAI (LangChain)  
    - Model: `gpt-4o-mini`  
    - Prompt: Score each trend 1-10 for relevance given brand voice and audience mood; return JSON array.  
    - Connect from `Prepare Trend Scoring Input`.

27. **Add Code Node to Parse Trend Scores**  
    - Name: `Parse Trend Scores`  
    - Type: Code  
    - Code: Parse OpenAI response JSON to extract trend scores; fallback if empty.  
    - Connect from `Trend Relevance Scoring (Wow Factor)`.

28. **Add Google Sheets Node to Store Selected Trend**  
    - Name: `Store Selected Trend`  
    - Type: Google Sheets  
    - Operation: Append row with columns Timestamp, Trend, Score, BrandVoice, AudienceMood.  
    - Connect from `Parse Trend Scores`.

---

29. **Add Google Sheets Node to Retrieve Latest Trend Scores**  
    - Name: `Retrieve Latest Trend Scores`  
    - Type: Google Sheets  
    - Operation: Read rows from the same sheet used for storing trends.  
    - Connect from `Store Selected Trend`.

30. **Add Code Node to Parse Retrieved Trend Scores**  
    - Name: `Parse Retrieved Trend Scores`  
    - Type: Code  
    - Code: Parse semicolon-separated trends and scores into arrays.  
    - Connect from `Retrieve Latest Trend Scores`.

31. **Add Code Node to Select Top Trends**  
    - Name: `Select Top Trends`  
    - Type: Code  
    - Code: Select trend with highest score; provide defaults if none.  
    - Connect from `Parse Retrieved Trend Scores`.

32. **Add OpenAI Node to Generate Social Media Content**  
    - Name: `Generate Social Media Content`  
    - Type: OpenAI (LangChain)  
    - Model: `gpt-4o-mini`  
    - Prompt: Generate content for Twitter/X, LinkedIn, TikTok, YouTube Shorts based on selected idea, brand voice, audience mood.  
    - Connect from `Select Top Trends`.

33. **Add Code Node to Parse Social Media Content**  
    - Name: `Parse Social Media Content`  
    - Type: Code  
    - Code: Parse OpenAI JSON response; fallback content if parsing fails.  
    - Connect from `Generate Social Media Content`.

34. **Add OpenAI Image Generation Node**  
    - Name: `Generate Images`  
    - Type: OpenAI (LangChain Image)  
    - Prompt: Generate images based on TikTok and YouTube Shorts scripts.  
    - Connect from `Parse Social Media Content`.

35. **Add LinkedIn Node to Post Content**  
    - Name: `Post to LinkedIn`  
    - Type: LinkedIn  
    - Post as: Organization  
    - Text: Use parsed LinkedIn content.  
    - Connect from `Parse Social Media Content`.

36. **Add Facebook Graph API Node to Post to Instagram**  
    - Name: `Post to Instagram`  
    - Type: Facebook Graph API  
    - Edge: photos  
    - Node: Your Instagram Business Account ID  
    - Connect from `Parse Social Media Content`.

37. **(Optional) Add Facebook Posting Node**  
    - Not present in original workflow; add if needed.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                       |
|----------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| This workflow automates trend extraction and social media content creation using OpenAI, Reddit, Twitter/X, and Google Trends. | Workflow description and use case.                                                                   |
| Setup requires API credentials for Reddit, SerpApi (Google Trends), OpenAI, Twitter/X, LinkedIn, Instagram/Facebook, and Google Sheets. | Setup instructions section.                                                                          |
| Google Sheets must have columns: Timestamp, Trend, Score, BrandVoice, AudienceMood (case-sensitive). | Setup instructions.                                                                                  |
| OpenAI model used is `gpt-4o-mini` by default; prompts can be customized for brand voice and tone. | Customization tips.                                                                                   |
| Posting to TikTok and YouTube Shorts is logged for manual posting; image generation is optional.   | Posting block notes.                                                                                  |
| Sticky notes in the workflow provide detailed explanations of the trend extraction and content generation blocks. | Sticky notes nodes `Sticky Note5`, `Sticky Note`, and `Sticky Note1`.                               |
| For Google Trends, SerpApi is used; fallback email notification is sent if API fails.               | Google Trends nodes and fallback logic.                                                              |
| Twitter API calls use OAuth 1.0a credentials with write permissions.                                | Twitter API nodes.                                                                                   |
| LinkedIn API requires OAuth 2.0 with `w_organization_social` scope.                                 | LinkedIn node.                                                                                       |
| Instagram/Facebook posting uses Facebook Graph API with appropriate permissions.                    | Facebook Graph API node.                                                                             |

---

This comprehensive documentation enables advanced users and AI agents to understand, reproduce, and customize the workflow effectively, while anticipating potential failure points and integration requirements.