Extract High-Engagement YouTube Keyword Trends for Content Strategy

https://n8nworkflows.xyz/workflows/extract-high-engagement-youtube-keyword-trends-for-content-strategy-8091


# Extract High-Engagement YouTube Keyword Trends for Content Strategy

### 1. Workflow Overview

This workflow extracts and analyzes trending YouTube keywords based on high-engagement videos in Germany to support content strategy decisions. It fetches the most popular videos, computes an engagement metric to identify quality interactions, and aggregates video tags into a consolidated keyword list.

The workflow is organized into these logical blocks:

- **1.1 Input Reception:** Entry point via manual trigger.
- **1.2 Data Acquisition:** Fetches most popular YouTube videos in Germany using YouTube Data API v3.
- **1.3 Data Preparation:** Splits the API response so each video is processed individually.
- **1.4 Engagement Calculation & Context Extraction:** Extracts key video metadata and calculates engagement rate.
- **1.5 Ranking & Filtering:** Sorts videos by engagement rate and selects the top 20.
- **1.6 Tags Aggregation:** Merges tags from top-ranked videos into a single array.
- **1.7 Keyword Summary:** Concatenates all tags into a clean string representing trending keywords.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Starts the workflow manually via the n8n interface for testing or ad hoc execution.
- **Nodes Involved:** 
  - When clicking ‘Execute workflow’
  - Sticky: Workflow Start

- **Node Details:**

  - **When clicking ‘Execute workflow’**
    - Type: Manual Trigger node
    - Role: Entry point; initiates workflow execution when triggered manually.
    - Configuration: Default manual trigger, no parameters.
    - Input: None
    - Output: Passes empty data to next node.
    - Edge cases: None; manual user action required.
  
  - **Sticky: Workflow Start**
    - Type: Sticky Note
    - Role: Documentation; describes the workflow start and notes possibility of replacing manual trigger with Cron/Webhook.
    - Input/Output: None
    - Edge cases: None

---

#### 1.2 Data Acquisition

- **Overview:** Calls YouTube Data API to fetch the 50 most popular videos in Germany with statistics and snippet details.
- **Nodes Involved:** 
  - HTTP Request – YouTube Most Popular
  - Sticky: API Call

- **Node Details:**

  - **HTTP Request – YouTube Most Popular**
    - Type: HTTP Request node
    - Role: Retrieves YouTube video data using YouTube Data API v3.
    - Configuration:
      - Method: GET (default)
      - URL: https://www.googleapis.com/youtube/v3/videos
      - Query parameters:
        - part = statistics,snippet (request video stats and metadata)
        - chart = mostPopular (popular videos chart)
        - regionCode = DE (Germany)
        - maxResults = 50 (limit results)
        - key = APIKEY (YouTube API key credential, must be replaced with a valid key)
      - Headers:
        - User-Agent: Mozilla/5.0 (to mimic browser request)
        - Accept: application/json
    - Input: Trigger node output
    - Output: JSON response containing an `items` array with video data
    - Edge cases:
      - API key invalid or quota exceeded → authentication or rate limit errors
      - Network timeouts or HTTP errors
      - Empty or malformed response
      - Regional or parameter changes affecting data availability

  - **Sticky: API Call**
    - Type: Sticky Note
    - Role: Documents API call purpose and included data fields
    - Input/Output: None

---

#### 1.3 Data Preparation

- **Overview:** Splits the bulk API response array so that each video item can be processed individually downstream.
- **Nodes Involved:**
  - Split Out – Items
  - Sticky: Split Items

- **Node Details:**

  - **Split Out – Items**
    - Type: SplitOut node
    - Role: Takes the array from the API response (`items[]`) and outputs each video as a separate item.
    - Configuration:
      - Field to split out: `items`
    - Input: JSON object with `items` array
    - Output: Multiple items, each representing one video's data
    - Edge cases:
      - Empty or missing `items` array leads to no output items
      - Unexpected structure breaks splitting

  - **Sticky: Split Items**
    - Type: Sticky Note
    - Role: Explains that this node splits the array so each video is processed separately.
    - Input/Output: None

---

#### 1.4 Engagement Calculation & Context Extraction

- **Overview:** Extracts key metadata fields and calculates an engagement rate metric per video to quantify interaction quality.
- **Nodes Involved:**
  - Set – Derive Context
  - Sticky: Context & KPIs

- **Node Details:**

  - **Set – Derive Context**
    - Type: Set node
    - Role: Extracts title, channel name, published date, views, likes, comments; calculates engagement rate.
    - Configuration:
      - Fields extracted from item JSON:
        - title (from snippet.title)
        - channelTitle (from snippet.channelTitle)
        - publishedAt (from snippet.publishedAt)
        - views (from statistics.viewCount, converted to number)
        - likes (from statistics.likeCount, converted to number)
        - comments (from statistics.commentCount, converted to number)
      - Calculated field:
        - engagementRate = (likes + comments) / (views + 1)
          - Division by (views + 1) avoids divide-by-zero errors
    - Key expressions:
      - Use of expressions to convert string counts to numbers
      - Mathematical calculation for engagementRate
    - Input: One video item
    - Output: Enriched item with context and KPIs
    - Edge cases:
      - Missing or null statistics fields (likes, comments, views) cause calculation errors or NaN
      - Division by zero avoided by adding 1 to views
      - Type conversion failures if values are not numeric strings

  - **Sticky: Context & KPIs**
    - Type: Sticky Note
    - Role: Documents the extraction and calculation logic around engagement metrics.
    - Input/Output: None

---

#### 1.5 Ranking & Filtering

- **Overview:** Sorts videos by engagement rate descending and limits to top 20 to focus analysis on highly engaging content.
- **Nodes Involved:**
  - Item Lists – Top by Engagement
  - Sticky: Ranking & Limit

- **Node Details:**

  - **Item Lists – Top by Engagement**
    - Type: ItemLists node
    - Role: Sorts the incoming items by engagementRate and keeps the top 20.
    - Configuration:
      - Operation: Sort
      - Sort criteria: engagementRate (descending)
      - Limit: Top 20 items after sorting (implicit from sticky note; actual node config may require manual limiting downstream or implicit)
    - Input: Items with engagementRate field
    - Output: Sorted and truncated item list
    - Edge cases:
      - Items missing engagementRate field sorted unpredictably
      - Less than 20 items results in smaller output set

  - **Sticky: Ranking & Limit**
    - Type: Sticky Note
    - Role: Explains sorting by engagementRate and limiting to top 20 items.
    - Input/Output: None

---

#### 1.6 Tags Aggregation

- **Overview:** Aggregates all tags from the top 20 videos and merges them into one array.
- **Nodes Involved:**
  - Aggregate – Collect Tags
  - Sticky: Collect Tags

- **Node Details:**

  - **Aggregate – Collect Tags**
    - Type: Aggregate node
    - Role: Collects and merges all `snippet.tags` arrays from the top videos into a single list.
    - Configuration:
      - Options: mergeLists = true (flattens arrays into one)
      - Field to aggregate: tags (from snippet.tags)
    - Input: Top 20 video items with tags arrays
    - Output: Single item with merged tags list
    - Edge cases:
      - Some videos may have missing or empty tags arrays
      - Duplicate tags not removed by default (may require downstream de-duplication if needed)

  - **Sticky: Collect Tags**
    - Type: Sticky Note
    - Role: Describes aggregation of tags from top videos.
    - Input/Output: None

---

#### 1.7 Keyword Summary

- **Overview:** Concatenates all collected tags into one continuous string to form a clean, trend-focused keyword/hashtag list.
- **Nodes Involved:**
  - Summarize – Tag Summary
  - Sticky: Keyword Summary

- **Node Details:**

  - **Summarize – Tag Summary**
    - Type: Summarize node
    - Role: Concatenates the merged tags array into a single string.
    - Configuration:
      - Field to summarize: tags
      - Aggregation method: concatenate
    - Input: Single item with merged tags array
    - Output: Single item with concatenated keyword string
    - Edge cases:
      - Empty tags list results in empty string
      - No delimiter specified, so output is direct concatenation (may require separator in future)

  - **Sticky: Keyword Summary**
    - Type: Sticky Note
    - Role: Explains creation of a clean list of trending hashtags/keywords from tags.
    - Input/Output: None

---

### 3. Summary Table

| Node Name                     | Node Type             | Functional Role                 | Input Node(s)             | Output Node(s)               | Sticky Note                                                                                   |
|-------------------------------|-----------------------|--------------------------------|---------------------------|-----------------------------|----------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger        | Workflow entry/start           | —                         | HTTP Request – YouTube Most Popular | Workflow Start: Entry point; run manually or replace with Cron/Webhook                       |
| Sticky: Workflow Start         | Sticky Note           | Documentation                  | —                         | —                           | Workflow Start: Entry point. Run via **Execute Workflow**. Replace manual trigger later       |
| HTTP Request – YouTube Most Popular | HTTP Request         | Fetch YouTube most popular videos | When clicking ‘Execute workflow’ | Split Out – Items            | API Call: Fetches most popular videos in Germany with statistics and snippet                 |
| Sticky: API Call               | Sticky Note           | Documentation                  | —                         | —                           | API Call: YouTube Data API v3, includes statistics and snippet                              |
| Split Out – Items              | SplitOut              | Splits response array into items | HTTP Request – YouTube Most Popular | Set – Derive Context          | Split Items: Splits `items[]` so each video is an individual item                           |
| Sticky: Split Items            | Sticky Note           | Documentation                  | —                         | —                           | Split Items: Splits array for individual processing                                         |
| Set – Derive Context           | Set                   | Extracts fields, calculates engagement | Split Out – Items          | Item Lists – Top by Engagement | Context & KPIs: Extracts key fields and calculates engagement rate                          |
| Sticky: Context & KPIs         | Sticky Note           | Documentation                  | —                         | —                           | Extracts title, channel, publishedAt, views, likes, comments. Calculates engagementRate     |
| Item Lists – Top by Engagement | ItemLists             | Sorts items by engagement, limits top 20 | Set – Derive Context       | Aggregate – Collect Tags      | Ranking & Limit: Sorts by engagementRate descending, keeps top 20                           |
| Sticky: Ranking & Limit        | Sticky Note           | Documentation                  | —                         | —                           | Sorts by engagementRate and keeps only Top 20                                              |
| Aggregate – Collect Tags       | Aggregate             | Merges tags arrays from top videos | Item Lists – Top by Engagement | Summarize – Tag Summary       | Collect Tags: Aggregates all snippet.tags into single array                                |
| Sticky: Collect Tags           | Sticky Note           | Documentation                  | —                         | —                           | Aggregates all snippet.tags from Top 20 videos                                             |
| Summarize – Tag Summary        | Summarize             | Concatenates all tags into one string | Aggregate – Collect Tags   | —                           | Keyword Summary: Concatenates tags into clean list of trending hashtags/keywords           |
| Sticky: Keyword Summary        | Sticky Note           | Documentation                  | —                         | —                           | Concatenates all collected tags into one string – list of trending hashtags/keywords       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name: When clicking ‘Execute workflow’  
   - Purpose: Manual start of the workflow.

2. **Add an HTTP Request node**  
   - Name: HTTP Request – YouTube Most Popular  
   - Connect: Output of Manual Trigger → Input of HTTP Request  
   - Configure:  
     - URL: https://www.googleapis.com/youtube/v3/videos  
     - Method: GET (default)  
     - Query Parameters:  
       - `part` = `statistics,snippet`  
       - `chart` = `mostPopular`  
       - `regionCode` = `DE`  
       - `maxResults` = `50`  
       - `key` = `<Your YouTube API Key here>` (use credentials or set directly)  
     - Headers:  
       - `User-Agent`: `Mozilla/5.0`  
       - `Accept`: `application/json`

3. **Add a SplitOut node**  
   - Name: Split Out – Items  
   - Connect: HTTP Request → SplitOut  
   - Configure:  
     - Field to split out: `items`

4. **Add a Set node**  
   - Name: Set – Derive Context  
   - Connect: SplitOut → Set  
   - Configure fields:  
     - `title`: Expression extracting `{{$json["snippet"]["title"]}}`  
     - `channelTitle`: `{{$json["snippet"]["channelTitle"]}}`  
     - `publishedAt`: `{{$json["snippet"]["publishedAt"]}}`  
     - `views`: Convert to number from `{{$json["statistics"]["viewCount"]}}`  
     - `likes`: Convert to number from `{{$json["statistics"]["likeCount"]}}`  
     - `comments`: Convert to number from `{{$json["statistics"]["commentCount"]}}`  
     - `engagementRate`: Expression: `({{ $json.likes + $json.comments }}) / ({{ $json.views + 1 }})`

5. **Add an ItemLists node**  
   - Name: Item Lists – Top by Engagement  
   - Connect: Set → ItemLists  
   - Configure:  
     - Operation: Sort  
     - Sort by: `engagementRate` descending  
     - Limit to top 20 items (if limit not available in node, filter downstream or via code)

6. **Add an Aggregate node**  
   - Name: Aggregate – Collect Tags  
   - Connect: ItemLists → Aggregate  
   - Configure:  
     - Options: Enable `mergeLists` (to flatten arrays)  
     - Field to aggregate: `tags` (from `snippet.tags` in each item)

7. **Add a Summarize node**  
   - Name: Summarize – Tag Summary  
   - Connect: Aggregate → Summarize  
   - Configure:  
     - Field to summarize: `tags`  
     - Aggregation: Concatenate

8. **Optionally add Sticky Notes** at appropriate positions to document each block as described in the workflow overview.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------|
| This workflow currently uses a manual trigger for execution; recommended to replace with Cron or Webhook for automation. | Sticky: Workflow Start                          |
| YouTube Data API v3 requires a valid API key with enabled quota for `videos.list` endpoint.                    | https://developers.google.com/youtube/v3/docs/videos/list |
| Engagement rate formula avoids division by zero by adding 1 to views count.                                   | Sticky: Context & KPIs                          |
| Sorting and limiting results to top 20 reduces noise and focuses on meaningful keyword trends.                  | Sticky: Ranking & Limit                         |
| Aggregated tags may contain duplicates; further post-processing can be added if unique tags are desired.       | Aggregate – Collect Tags node considerations   |

---

**Disclaimer:** The provided text is exclusively derived from an automated n8n workflow process. It complies strictly with current content policies and contains no illegal or protected content. All data processed is legal and publicly accessible.