Generate Curated Local News Digests with Gemini AI for Discord

https://n8nworkflows.xyz/workflows/generate-curated-local-news-digests-with-gemini-ai-for-discord-8991


# Generate Curated Local News Digests with Gemini AI for Discord

### 1. Workflow Overview

This n8n workflow automates the generation and delivery of curated local news digests for the Phoenix, Scottsdale, and Paradise Valley areas to a Discord channel. It runs daily at 8 AM, aggregates news articles from multiple RSS feeds (including local news sites and Reddit subreddits), deduplicates and filters these articles, scores their local relevance using Google Gemini AI, ranks the articles by relevance, formats the top articles into a Discord-friendly message, and sends the digest via a Discord webhook.

The workflow’s logic is grouped into the following blocks:

- **1.1 Scheduled Trigger:** Runs the workflow daily at 8 AM.
- **1.2 RSS Feed Aggregation:** Collects news articles from five RSS sources.
- **1.3 Merge and Deduplicate:** Combines all feed items and removes duplicates based on URLs or titles, filtering articles mentioning target locations.
- **1.4 Prepare for AI Scoring:** Prepares article titles in bulk to send to Gemini AI for relevance scoring.
- **1.5 Gemini AI Relevance Scoring:** Sends article titles to Gemini AI to obtain local relevance scores.
- **1.6 Score Processing and Ranking:** Parses Gemini’s response, applies fallback scoring, and sorts articles by relevance.
- **1.7 Limit and Format for Discord:** Selects the top N articles (default 5), formats them into a Discord message respecting character limits.
- **1.8 Send to Discord:** Posts the formatted news digest to a Discord channel via webhook.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:** Triggers the workflow every day at 8:00 AM to start the news digest process.
- **Nodes Involved:**  
  - Daily 8AM Trigger

- **Node Details:**  
  - **Daily 8AM Trigger:**  
    - Type: Schedule Trigger  
    - Configuration: Uses a cron expression `0 8 * * *` to fire daily at 8 AM server time.  
    - Inputs: None (trigger node)  
    - Outputs: Starts five parallel RSS feed read nodes.  
    - Edge Cases: Cron syntax errors; time zone mismatches; workflow not running at the scheduled time if n8n instance is down.

#### 1.2 RSS Feed Aggregation

- **Overview:** Reads news articles from five RSS feeds covering local news websites and Reddit communities related to the Phoenix area.
- **Nodes Involved:**  
  - RSS Phoenix New Times  
  - RSS AZ Free News  
  - RSS Reddit Phoenix  
  - RSS Reddit Scottsdale  
  - RSS Reddit Arizona

- **Node Details:**  
  - Each node is an **RSS Feed Read** node configured with a specific RSS URL:  
    - Phoenix New Times: https://www.phoenixnewtimes.com/phoenix/Rss.xml  
    - AZ Free News: https://azfreenews.com/feed/  
    - Reddit Phoenix: https://www.reddit.com/r/Phoenix/.rss  
    - Reddit Scottsdale: https://www.reddit.com/r/Scottsdale/.rss  
    - Reddit Arizona: https://www.reddit.com/r/Arizona/.rss  
  - Inputs: Trigger from the Daily 8AM Trigger node.  
  - Outputs: Each outputs its RSS items to be merged.  
  - Edge Cases: RSS feed unavailability; malformed RSS data; network timeouts; rate-limiting on Reddit API; RSS feed structural changes.

#### 1.3 Merge and Deduplicate

- **Overview:** Combines all five RSS feed outputs into a single dataset, then deduplicates articles by URL or title, filtering those mentioning specified local locations.
- **Nodes Involved:**  
  - Merge All RSS Feeds  
  - Deduplicate & Prepare

- **Node Details:**  
  - **Merge All RSS Feeds:**  
    - Type: Merge (mode: append)  
    - Configuration: Accepts 5 inputs (from each RSS feed node).  
    - Inputs: RSS feeds  
    - Outputs: Combined list of all articles.  
    - Edge Cases: Empty inputs if some feeds fail; handling large data volumes.

  - **Deduplicate & Prepare:**  
    - Type: Code (JavaScript) node  
    - Configuration:  
      - Deduplicates articles by URL or title.  
      - Extracts fields: title, URL, description, source, publication date.  
      - Checks if article text mentions any of the target locations: Phoenix, Scottsdale, Paradise Valley, Arizona, Tempe, Mesa, Chandler.  
      - Keeps max 50 most recent articles after sorting by publication date descending.  
    - Inputs: Merged RSS feed items.  
    - Outputs: Deduplicated and filtered articles.  
    - Key Variables: `uniqueArticles` map keyed by URL or title; `mentionsLocation` boolean flag.  
    - Edge Cases: Missing or malformed date fields; articles without URLs; possible false negatives in location mentions due to text variations.

#### 1.4 Prepare for AI Scoring

- **Overview:** Prepares a concise text payload containing only article titles to minimize token usage before sending to Gemini AI for relevance scoring.
- **Nodes Involved:**  
  - Prepare for Gemini

- **Node Details:**  
  - Type: Code (JavaScript) node  
  - Configuration:  
    - Maps articles from input.  
    - Creates a numbered list of article titles as a single string.  
    - Returns both the original article array and the prepared `titlesText`.  
  - Inputs: Deduplicated article list.  
  - Outputs: Object including articles array and text for scoring.  
  - Edge Cases: Empty article list; titles with unusual characters; encoding issues.

#### 1.5 Gemini AI Relevance Scoring

- **Overview:** Sends the prepared article titles to Google’s Gemini AI API to score each article’s local relevance according to defined criteria.
- **Nodes Involved:**  
  - Gemini Relevance Scoring

- **Node Details:**  
  - Type: HTTP Request node  
  - Configuration:  
    - POST request to Gemini API endpoint:  
      `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash-lite:generateContent?key=YOUR_KEY_HERE` (replace with real API key)  
    - JSON body includes prompt asking Gemini to score 0-100 each title based on:  
      - Local relevance (50%)  
      - Community impact (30%)  
      - News value (20%)  
    - Generation config limits temperature and response tokens for consistent output.  
  - Inputs: Prepared titles and article count from previous node.  
  - Outputs: Gemini’s JSON response containing scores as a JSON array string.  
  - Edge Cases: Invalid or missing API key; quota exceeded; response timeouts; malformed response; network errors.  
  - Notes: Sticky note reminds user to replace placeholder API key with a valid one.

#### 1.6 Score Processing and Ranking

- **Overview:** Parses Gemini’s response to extract scores, applies fallback scoring if parsing fails, attaches scores to articles, and sorts them by relevance descending.
- **Nodes Involved:**  
  - Process Scores  
  - Sort by Relevance

- **Node Details:**  
  - **Process Scores:**  
    - Type: Code (JavaScript) node  
    - Configuration:  
      - Tries to extract JSON array of scores from Gemini’s response text.  
      - If parsing fails, assigns default scores: 75 if article mentions locations, else 50.  
      - Combines articles with their relevance scores.  
    - Inputs: Gemini AI response and original articles from Prepare for Gemini.  
    - Outputs: Articles enriched with `relevanceScore` attribute.  
    - Edge Cases: Parsing errors; mismatch in number of scores vs articles.  
  - **Sort by Relevance:**  
    - Type: Sort node  
    - Configuration: Sorts by numeric field `relevanceScore` descending.  
    - Inputs: Scored articles.  
    - Outputs: Sorted articles.  
    - Edge Cases: Missing or non-numeric scores.

#### 1.7 Limit and Format for Discord

- **Overview:** Restricts the output to the top N articles (default 5), formats them into a clean Discord message with emojis and stripped HTML descriptions, and splits the message if it exceeds Discord’s character limits.
- **Nodes Involved:**  
  - Limit to Top 5  
  - Format Discord Message

- **Node Details:**  
  - **Limit to Top 5:**  
    - Type: Limit node  
    - Configuration: Limits the number of articles passed forward to 5 (adjustable).  
  - **Format Discord Message:**  
    - Type: Code (JavaScript) node  
    - Configuration:  
      - Uses article data to create a digest message including:  
        - Today's date formatted as weekday, year, month, day.  
        - Ranking emojis for top 3 articles (🥇, 🥈, 🥉), then numeric for others.  
        - Titles, description (HTML stripped), and URL links.  
        - Divider lines and footer text.  
      - Breaks message into chunks under 1900 characters to comply with Discord limits.  
      - Returns multiple message objects to be sent individually.  
    - Edge Cases: Articles without descriptions; very long descriptions; message splitting logic correctness.

#### 1.8 Send to Discord

- **Overview:** Sends the formatted news digest messages to Discord via a webhook.
- **Nodes Involved:**  
  - Send to Discord

- **Node Details:**  
  - Type: Discord node (Webhook authentication)  
  - Configuration:  
    - Uses provided Discord webhook credentials.  
    - Sends each message chunk as separate Discord messages.  
  - Inputs: Formatted message content objects.  
  - Outputs: None (final node).  
  - Edge Cases: Invalid webhook URL; Discord API rate limiting; failed message sends.

---

### 3. Summary Table

| Node Name               | Node Type              | Functional Role                      | Input Node(s)                               | Output Node(s)                | Sticky Note                                                                                   |
|-------------------------|------------------------|------------------------------------|---------------------------------------------|------------------------------|----------------------------------------------------------------------------------------------|
| Daily 8AM Trigger       | Schedule Trigger       | Starts workflow daily at 8AM       | None                                        | RSS Phoenix New Times, RSS AZ Free News, RSS Reddit Phoenix, RSS Reddit Scottsdale, RSS Reddit Arizona | Change the cron expression to adjust schedule. Current: 8AM daily                            |
| RSS Phoenix New Times    | RSS Feed Read          | Reads Phoenix New Times RSS feed   | Daily 8AM Trigger                           | Merge All RSS Feeds           | Set Your News Feeds Here. Make sure they are rss. See examples below for link structures.     |
| RSS AZ Free News         | RSS Feed Read          | Reads AZ Free News RSS feed        | Daily 8AM Trigger                           | Merge All RSS Feeds           | Set Your News Feeds Here. Make sure they are rss. See examples below for link structures.     |
| RSS Reddit Phoenix       | RSS Feed Read          | Reads Reddit /r/Phoenix RSS feed   | Daily 8AM Trigger                           | Merge All RSS Feeds           | Set Your News Feeds Here. Make sure they are rss. See examples below for link structures.     |
| RSS Reddit Scottsdale    | RSS Feed Read          | Reads Reddit /r/Scottsdale RSS feed| Daily 8AM Trigger                           | Merge All RSS Feeds           | Set Your News Feeds Here. Make sure they are rss. See examples below for link structures.     |
| RSS Reddit Arizona       | RSS Feed Read          | Reads Reddit /r/Arizona RSS feed   | Daily 8AM Trigger                           | Merge All RSS Feeds           | Set Your News Feeds Here. Make sure they are rss. See examples below for link structures.     |
| Merge All RSS Feeds      | Merge                  | Merges all RSS feed articles       | RSS Phoenix New Times, RSS AZ Free News, RSS Reddit Phoenix, RSS Reddit Scottsdale, RSS Reddit Arizona | Deduplicate & Prepare        |                                                                                              |
| Deduplicate & Prepare    | Code                   | Deduplicates and filters articles  | Merge All RSS Feeds                         | Prepare for Gemini            |                                                                                              |
| Prepare for Gemini       | Code                   | Prepares article titles batch for Gemini AI | Deduplicate & Prepare                         | Gemini Relevance Scoring      |                                                                                              |
| Gemini Relevance Scoring | HTTP Request           | Sends titles to Gemini AI for scoring | Prepare for Gemini                          | Process Scores                | Set Your Gemini API Key. Get for free! Link: https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash-lite:generateContent?key=YOUR_KEY_HERE |
| Process Scores           | Code                   | Parses Gemini scores and attaches to articles | Gemini Relevance Scoring                   | Sort by Relevance             |                                                                                              |
| Sort by Relevance        | Sort                   | Sorts articles by descending relevance | Process Scores                             | Limit to Top 5                |                                                                                              |
| Limit to Top 5           | Limit                  | Limits number of articles to send  | Sort by Relevance                           | Format Discord Message        | Set how many articles you want. Default is 5                                               |
| Format Discord Message   | Code                   | Formats articles into Discord messages | Limit to Top 5                             | Send to Discord               | Might need changing for other sending locations, telegram whatsapp etc.                     |
| Send to Discord          | Discord (Webhook)      | Sends news digest to Discord channel | Format Discord Message                      | None                         | Receive your relevant news articles                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Cron expression: `0 8 * * *` (runs daily at 8 AM)  
   - Name: `Daily 8AM Trigger`

2. **Add Five RSS Feed Read Nodes**  
   - Type: RSS Feed Read  
   - Configure each with the following URLs:  
     - `https://www.phoenixnewtimes.com/phoenix/Rss.xml` (Phoenix New Times)  
     - `https://azfreenews.com/feed/` (AZ Free News)  
     - `https://www.reddit.com/r/Phoenix/.rss` (Reddit Phoenix)  
     - `https://www.reddit.com/r/Scottsdale/.rss` (Reddit Scottsdale)  
     - `https://www.reddit.com/r/Arizona/.rss` (Reddit Arizona)  
   - Connect all from `Daily 8AM Trigger` (main output) to each RSS node’s input.

3. **Add a Merge Node**  
   - Type: Merge  
   - Set number of inputs to 5  
   - Connect all five RSS nodes’ outputs to the Merge node inputs.

4. **Add a Code Node to Deduplicate & Prepare**  
   - Name: `Deduplicate & Prepare`  
   - Paste the provided JavaScript code that:  
     - Deduplicates articles by URL/title  
     - Filters by location mention (Phoenix, Scottsdale, etc.)  
     - Keeps max 50 newest articles  
   - Connect Merge node output to this node input.

5. **Add a Code Node to Prepare for Gemini Scoring**  
   - Name: `Prepare for Gemini`  
   - Paste the provided JavaScript code that creates a numbered list of article titles for efficient scoring.  
   - Connect `Deduplicate & Prepare` output to this node.

6. **Add an HTTP Request Node for Gemini AI**  
   - Name: `Gemini Relevance Scoring`  
   - Method: POST  
   - URL:  
     `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash-lite:generateContent?key=YOUR_KEY_HERE` (replace `YOUR_KEY_HERE` with your actual API key)  
   - Body Content-Type: JSON  
   - Paste the JSON body expression from the workflow which includes the prompt and generation config.  
   - Connect `Prepare for Gemini` output here.

7. **Add a Code Node to Process Scores**  
   - Name: `Process Scores`  
   - Paste the JavaScript code that parses Gemini’s JSON array of scores, applies fallback scoring, and combines scores with articles.  
   - Connect Gemini HTTP Request node output to this node input.

8. **Add a Sort Node**  
   - Name: `Sort by Relevance`  
   - Configure to sort by `relevanceScore` field descending.  
   - Connect `Process Scores` output here.

9. **Add a Limit Node**  
   - Name: `Limit to Top 5`  
   - Set max items to 5 (or desired number)  
   - Connect `Sort by Relevance` output here.

10. **Add a Code Node to Format Discord Message**  
    - Name: `Format Discord Message`  
    - Paste the JavaScript code that:  
      - Builds a multi-line Discord message with emojis, stripped HTML descriptions, links, date header, and footer.  
      - Splits message into chunks <1900 characters to comply with Discord limits.  
    - Connect `Limit to Top 5` output here.

11. **Add a Discord Node to Send Message**  
    - Name: `Send to Discord`  
    - Authentication: Webhook  
    - Enter your Discord webhook URL or credentials.  
    - Map the content field to `{{$json.content}}` from the input.  
    - Connect `Format Discord Message` output here.

12. **Verify and Test**  
    - Replace placeholder API keys and webhook URLs with actual credentials.  
    - Test the workflow execution and check Discord for delivered messages.  
    - Adjust cron timing, limit count, or RSS feeds as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                                 |
|-----------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| Set Your Gemini API Key. Get for free! Link: https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash-lite:generateContent?key=YOUR_KEY_HERE | Gemini AI Relevance Scoring node requires a valid API key.                                                     |
| Set Your News Feeds Here. Make sure they are rss. See examples below for link structures.                 | Applies to all RSS nodes – ensure feeds are valid RSS format for successful ingestion.                         |
| Set how many articles you want. Default is 5                                                              | Limit to Top 5 node can be adjusted to increase or decrease articles sent in the digest.                      |
| Might need changing for other sending locations, telegram whatsapp etc.                                   | Format Discord Message node is tailored for Discord message formatting; other platforms may require changes.  |
| Receive your relevant news articles                                                                       | Final output is sent to Discord channel via webhook.                                                          |

---

**Disclaimer:** The provided content is derived exclusively from an automated workflow created with n8n, respecting all current content policies and containing no illegal, offensive, or protected material. All data processed is legal and public.