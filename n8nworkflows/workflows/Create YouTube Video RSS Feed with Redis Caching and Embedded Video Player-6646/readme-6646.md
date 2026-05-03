Create YouTube Video RSS Feed with Redis Caching and Embedded Video Player

https://n8nworkflows.xyz/workflows/create-youtube-video-rss-feed-with-redis-caching-and-embedded-video-player-6646


# Create YouTube Video RSS Feed with Redis Caching and Embedded Video Player

### 1. Workflow Overview

This workflow creates a custom YouTube video RSS feed with Redis caching and embedded video players, designed for users who want to follow specific YouTube channels through RSS readers. It pulls recent videos (excluding Shorts and videos older than one week) from configured YouTube channels, enriches video data via the YouTube API, caches video information to reduce API calls, and constructs a fully featured RSS feed including embedded playable videos.

The workflow is logically divided into these blocks:

- **1.1 Input Reception**: Receives webhook calls from RSS clients to trigger the workflow.
- **1.2 Channel Setup and RSS Feed Retrieval**: Defines YouTube channels, fetches their RSS feeds, and filters videos.
- **1.3 Redis Cache Lookup**: Checks cached video data to avoid redundant API calls.
- **1.4 Video Data Enrichment**: For uncached videos, fetches detailed video info including embedded player HTML.
- **1.5 RSS Item Construction and Caching**: Builds RSS items per video, caches new items in Redis.
- **1.6 RSS Feed Assembly and Response**: Merges cached and new items, sorts them, builds the final RSS XML, and responds to the webhook.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block triggers the workflow when an RSS client accesses the webhook URL and returns the constructed RSS XML feed.

- **Nodes Involved:**  
  - RSS Webhook  
  - Respond to Webhook

- **Node Details:**

  - **RSS Webhook**  
    - Type: Webhook node, receives external HTTP requests  
    - Config: Path set to a unique webhook ID; response mode set to use a response node  
    - Inputs: None (entry point)  
    - Outputs: Connected to "Set Channels" node to start processing  
    - Edge Cases: Unauthorized or malformed webhook calls; ensure webhook URL is kept private or secured as needed.

  - **Respond to Webhook**  
    - Type: Respond to Webhook node  
    - Config: Responds with content-type `application/xml`; body set dynamically from JSON property `feed`  
    - Inputs: Receives final RSS feed XML string from "Build RSS Feed" node  
    - Outputs: HTTP response to the caller  
    - Edge Cases: If the feed property is empty or malformed, the response will be invalid XML.

---

#### 1.2 Channel Setup and RSS Feed Retrieval

- **Overview:**  
  Defines which YouTube channels to process, fetches their RSS feeds, and filters videos to exclude Shorts and videos older than one week.

- **Nodes Involved:**  
  - Set Channels  
  - Make Each Channel One Item  
  - RSS Read YouTube Channel Feeds  
  - Filter Out Shorts and Old Videos

- **Node Details:**

  - **Set Channels**  
    - Type: Set node (raw JSON mode)  
    - Config: Contains JSON array with YouTube channel IDs and optionally channel names  
    - Outputs: An object with a `channels` array  
    - Edge Cases: Incorrect or missing channel IDs will cause downstream failures or empty feeds.

  - **Make Each Channel One Item**  
    - Type: SplitOut node  
    - Config: Splits the `channels` array into individual items for processing  
    - Inputs: Receives channels object from "Set Channels"  
    - Outputs: One item per channel ID  
    - Edge Cases: Empty `channels` array results in no output.

  - **RSS Read YouTube Channel Feeds**  
    - Type: RSS Feed Read node  
    - Config: URL templated with channel ID to fetch YouTube RSS feed  
    - Inputs: Each channel item  
    - Outputs: List of videos per channel  
    - Edge Cases: Network errors, invalid channel ID, or feed rate limit errors.

  - **Filter Out Shorts and Old Videos**  
    - Type: Filter node  
    - Config: Filters out videos whose link starts with `https://www.youtube.com/shorts` and videos with `pubDate` older than 7 days  
    - Inputs: Video items from RSS feed  
    - Outputs: Only full-length recent videos  
    - Edge Cases: Videos missing `link` or `pubDate` may be incorrectly filtered.

---

#### 1.3 Redis Cache Lookup

- **Overview:**  
  Attempts to retrieve cached RSS item XML for each video by its unique video ID key in Redis to avoid redundant YouTube API calls.

- **Nodes Involved:**  
  - Get Video Info From Cache  
  - Video in Cache?

- **Node Details:**

  - **Get Video Info From Cache**  
    - Type: Redis node  
    - Operation: `get` by key pattern `yt-rss-item:<video_id>`  
    - Inputs: Filtered video items  
    - Outputs: Adds property `rss_item` if found  
    - Credentials: Requires Redis credentials (host, port, password if applicable)  
    - Edge Cases: Redis connection failures, missing keys (cache misses).

  - **Video in Cache?**  
    - Type: If node  
    - Conditions: Checks if `rss_item` exists in JSON  
    - Inputs: Redis get node output  
    - Outputs:  
      - True branch (cache hit): proceeds to use cached RSS items  
      - False branch (cache miss): triggers video data enrichment  
    - Edge Cases: Expression evaluation failures if `rss_item` property missing or malformed.

---

#### 1.4 Video Data Enrichment

- **Overview:**  
  For videos not found in cache, fetch detailed video info via YouTube API, extract a short description, and prepare enriched RSS items.

- **Nodes Involved:**  
  - Get Video ID  
  - Get Video Details  
  - Extract Short Description  
  - Get relevant RSS fields  
  - Create RSS Items

- **Node Details:**

  - **Get Video ID**  
    - Type: Set node  
    - Config: Extracts video ID from filtered item JSON `id` field  
    - Inputs: Items from cache miss branch  
    - Outputs: Sets a clean `id` property for the YouTube API call  
    - Edge Cases: Invalid or missing ID format.

  - **Get Video Details**  
    - Type: YouTube node  
    - Operation: `get` video resource with parts `snippet` and `player`  
    - Inputs: Video ID  
    - Credentials: Requires Google API credentials (OAuth2 or API key)  
    - Edge Cases: API quota limit, invalid video ID, authentication failures.

  - **Extract Short Description**  
    - Type: Code node (JavaScript)  
    - Function: Extracts the first paragraph (up to first newline) from the full description to use as short description  
    - Inputs: Detailed video info  
    - Outputs: Sets `description` field with short description  
    - Edge Cases: Missing description, no newline characters.

  - **Get relevant RSS fields**  
    - Type: Set node  
    - Config: Maps video snippet and filtered feed fields to RSS item properties including title, link, pubDate, author, thumbnail, description, and embedded player HTML  
    - Inputs: Output from "Extract Short Description" and filtered feed  
    - Outputs: JSON with clean RSS item fields  
    - Edge Cases: Missing nested JSON properties.

  - **Create RSS Items**  
    - Type: Code node  
    - Function: Builds RSS `<item>` XML snippet with title, link, description, embedded player iframe in content, author, and pubDate  
    - Inputs: Structured video info from previous node  
    - Outputs: JSON with `rss_item` string property containing XML snippet  
    - Edge Cases: XML escaping issues if special characters in content.

---

#### 1.5 RSS Item Construction and Caching

- **Overview:**  
  Prepares new RSS items for caching and merges them with cached items. Saves new items in Redis with a TTL of 1 week.

- **Nodes Involved:**  
  - Get Fields to Cache  
  - Cache Video Info  
  - Get New RSS Items  
  - Get Cached RSS Items  
  - Merge RSS Items

- **Node Details:**

  - **Get Fields to Cache**  
    - Type: Set node  
    - Config: Extracts `rss_item` and video `id` fields to prepare for caching  
    - Inputs: RSS items from "Create RSS Items"  
    - Outputs: JSON with fields for Redis set  
    - Edge Cases: Missing `rss_item` or `id`.

  - **Cache Video Info**  
    - Type: Redis node  
    - Operation: `set` key `yt-rss-item:<id>` with value as RSS XML snippet, TTL set to 604800 seconds (7 days)  
    - Inputs: Prepared cache fields  
    - Credentials: Redis credentials must be configured  
    - Edge Cases: Redis connection or write failures.

  - **Get New RSS Items**  
    - Type: Set node  
    - Config: Prepares new RSS items and their `pubDate` for merging  
    - Inputs: Output from "Create RSS Items"  
    - Outputs: JSON with `rss_item` and `pubDate`  
    - Edge Cases: Missing data fields.

  - **Get Cached RSS Items**  
    - Type: Set node  
    - Config: Prepares cached RSS items and their `pubDate` for merging  
    - Inputs: Cached RSS items from "Video in Cache?" true branch  
    - Outputs: JSON with `rss_item` and `pubDate`  
    - Edge Cases: Missing or corrupted cached data.

  - **Merge RSS Items**  
    - Type: Merge node  
    - Operation: Merges cached and new RSS items arrays into one list  
    - Inputs: Cached items and new items  
    - Outputs: Combined list of all RSS items  
    - Edge Cases: Empty inputs on either side.

---

#### 1.6 RSS Feed Assembly and Response

- **Overview:**  
  Sorts all RSS items by publication date descending, builds the full RSS XML feed, and sends it as the webhook response.

- **Nodes Involved:**  
  - Sort by Date  
  - Build RSS Feed  
  - Respond to Webhook

- **Node Details:**

  - **Sort by Date**  
    - Type: Sort node  
    - Config: Sort items descending by `pubDate` field  
    - Inputs: Merged RSS items  
    - Outputs: Sorted list  
    - Edge Cases: Missing or invalid `pubDate` may cause incorrect sorting.

  - **Build RSS Feed**  
    - Type: Code node  
    - Function: Constructs the full RSS XML feed including channel metadata and all item XML snippets concatenated  
    - Inputs: Sorted RSS items  
    - Outputs: JSON with `feed` property containing complete XML  
    - Edge Cases: Malformed individual RSS item XML can break whole feed.

  - **Respond to Webhook**  
    - (Detailed above in 1.1)

---

### 3. Summary Table

| Node Name                      | Node Type              | Functional Role                                    | Input Node(s)                     | Output Node(s)                     | Sticky Note                                                                                                                      |
|--------------------------------|------------------------|---------------------------------------------------|----------------------------------|----------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| RSS Webhook                    | Webhook                | Entry point; receives RSS client requests         | None                             | Set Channels                     | ## 4) Copy the Webhook URL... Copy the production URL from this node and paste it into any RSS reader of your choosing.         |
| Manual trigger for testing     | Manual Trigger         | Testing entry point                                | None                             | Set Channels                     |                                                                                                                                 |
| Set Channels                   | Set                    | Defines YouTube channels to pull                   | RSS Webhook, Manual Trigger      | Make Each Channel One Item       | ## 1) Set Channels to Pull Videos From... How to get a YouTube channel ID explained.                                              |
| Make Each Channel One Item     | SplitOut               | Splits channels array into individual channel items| Set Channels                    | RSS Read YouTube Channel Feeds   | ## Read RSS feeds of YouTube Channels...                                                                                         |
| RSS Read YouTube Channel Feeds | RSS Feed Read          | Fetches YouTube RSS feed per channel                | Make Each Channel One Item       | Filter Out Shorts and Old Videos | ## Read RSS feeds of YouTube Channels...                                                                                         |
| Filter Out Shorts and Old Videos | Filter                 | Filters out Shorts and videos older than 7 days    | RSS Read YouTube Channel Feeds   | Get Video Info From Cache        | ## Read RSS feeds of YouTube Channels...                                                                                         |
| Get Video Info From Cache      | Redis                  | Tries to get cached RSS item for video             | Filter Out Shorts and Old Videos | Video in Cache?                  | ## 2) Add Redis Credentials... Instructions to configure Redis credentials                                                        |
| Video in Cache?                | If                     | Branches on cache hit or miss                       | Get Video Info From Cache        | Get Cached RSS Items (true), Get Video ID (false) | ## Pull Video Info From Uncached Videos...                                                                                        |
| Get Cached RSS Items           | Set                    | Prepares cached RSS items for merging               | Video in Cache? (true branch)    | Merge RSS Items                  | ## Build RSS Feed...                                                                                                              |
| Get Video ID                  | Set                    | Extracts video ID for YouTube API call              | Video in Cache? (false branch)   | Get Video Details               | ## Pull Video Info From Uncached Videos...                                                                                        |
| Get Video Details              | YouTube                | Fetches detailed video info including player embed | Get Video ID                    | Extract Short Description        | ## 3) Add Google Credentials... Requires Google API credentials as per documentation                                              |
| Extract Short Description      | Code                   | Extracts short description from full description    | Get Video Details               | Get relevant RSS fields          | ## Pull Video Info From Uncached Videos...                                                                                        |
| Get relevant RSS fields        | Set                    | Maps video data to RSS item fields                   | Extract Short Description        | Create RSS Items                 | ## Pull Video Info From Uncached Videos...                                                                                        |
| Create RSS Items               | Code                   | Builds RSS XML item string including embedded player| Get relevant RSS fields         | Get Fields to Cache, Get New RSS Items | ## Pull Video Info From Uncached Videos...                                                                                        |
| Get Fields to Cache            | Set                    | Prepares fields for caching in Redis                 | Create RSS Items                 | Cache Video Info                | ## Save RSS Items to Cache...                                                                                                    |
| Cache Video Info               | Redis                  | Caches new RSS item with TTL                          | Get Fields to Cache              | None                           | ## Save RSS Items to Cache...                                                                                                    |
| Get New RSS Items              | Set                    | Prepares new RSS items for merging                    | Create RSS Items                 | Merge RSS Items                 | ## Save RSS Items to Cache...                                                                                                    |
| Merge RSS Items                | Merge                  | Merges cached and new RSS items                       | Get Cached RSS Items, Get New RSS Items | Sort by Date             | ## Build RSS Feed...                                                                                                              |
| Sort by Date                  | Sort                   | Sorts all RSS items descending by pubDate            | Merge RSS Items                 | Build RSS Feed                 | ## Build RSS Feed...                                                                                                              |
| Build RSS Feed                | Code                   | Constructs full RSS feed XML                           | Sort by Date                   | Respond to Webhook             | ## Build RSS Feed...                                                                                                              |
| Respond to Webhook            | Respond to Webhook     | Returns RSS XML feed to requester                      | Build RSS Feed                 | None                          | ## 4) Copy the Webhook URL... Copy the production URL from this node and paste it into any RSS reader of your choosing.         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook Node** named `RSS Webhook`:
   - Set path to a unique identifier (e.g., `ca2154fe-8ea8-4f2e-bb93-897de6df8bb1`).
   - Set response mode to `Response Node`.

2. **Create a Set Node** named `Set Channels`:
   - Mode: Raw JSON.
   - Add JSON property `channels` as an array of objects with at least `id` (YouTube channel ID).
   - Example:
     ```json
     {
       "channels": [
         { "name": "n8n", "id": "UCiHVTkJtWSdc9N3h0nUGWLg" },
         { "name": "NetworkChuck", "id": "UC9x0AN7BWHpCDHSm9NiJFJQ" },
         { "name": "David Bombal", "id": "UCP7WmQ_U4GB3K51Od9QvM0w" }
       ]
     }
     ```
   - Connect `RSS Webhook` to `Set Channels`.

3. **Create a SplitOut Node** named `Make Each Channel One Item`:
   - Field to split out: `channels`.
   - Connect `Set Channels` to this node.

4. **Create an RSS Feed Read Node** named `RSS Read YouTube Channel Feeds`:
   - URL: `https://www.youtube.com/feeds/videos.xml?channel_id={{ $json.id }}`
   - Connect `Make Each Channel One Item` to this node.

5. **Create a Filter Node** named `Filter Out Shorts and Old Videos`:
   - Condition 1: `link` does NOT start with `https://www.youtube.com/shorts`
   - Condition 2: `pubDate` is after [current date - 7 days] (use expression: `new Date(Date.now() - 7 * 24 * 60 * 60 * 1000).toISOString()`)
   - Connect `RSS Read YouTube Channel Feeds` to this node.

6. **Create a Redis Node** named `Get Video Info From Cache`:
   - Operation: `get`
   - Key: `yt-rss-item:{{ $json.id }}`
   - Property name for value: `rss_item`
   - Configure Redis credentials (host, port, password).
   - Connect `Filter Out Shorts and Old Videos` to this node.

7. **Create an If Node** named `Video in Cache?`:
   - Condition: Check if `rss_item` exists (expression: `{{$json.rss_item}}` exists).
   - Connect `Get Video Info From Cache` to this If node.

8. **Create a Set Node** named `Get Cached RSS Items`:
   - Assign fields:
     - `rss_item` from `$json.rss_item`
     - `pubDate` from original filtered item (`$('Filter Out Shorts and Old Videos').item.json.pubDate`)
   - Connect `Video in Cache?` true branch to this node.

9. **Create a Set Node** named `Get Video ID`:
   - Assign `id` from `$('Filter Out Shorts and Old Videos').item.json.id`
   - Connect `Video in Cache?` false branch to this node.

10. **Create a YouTube Node** named `Get Video Details`:
    - Operation: `get`
    - Resource: `video`
    - Video ID: Extract ID from `id` field, e.g., `{{$json.id.split(':')[2]}}`
    - Part: `snippet`, `player`
    - Configure Google API credentials (OAuth2 or API key).
    - Connect `Get Video ID` to this node.

11. **Create a Code Node** named `Extract Short Description`:
    - Run once for each item.
    - Code: Extract substring before first newline in `snippet.description`, set as `description`.
    - Connect `Get Video Details` to this node.

12. **Create a Set Node** named `Get relevant RSS fields`:
    - Map fields from detailed video info and filtered feed:
      - `title`: `snippet.title`
      - `content`: `snippet.description`
      - `thumbnail`: `snippet.thumbnails.standard`
      - `link`: from original filtered feed
      - `pubDate`: from original filtered feed
      - `author`: from original filtered feed
      - `id`: from original RSS feed item
      - `description`: short description extracted
      - `playerEmbedHtml`: `player.embedHtml`
    - Connect `Extract Short Description` to this node.

13. **Create a Code Node** named `Create RSS Items`:
    - Build RSS `<item>` XML string using mapped fields.
    - Include embedded video player iframe in `content:encoded`.
    - Output as `rss_item` property.
    - Connect `Get relevant RSS fields` to this node.

14. **Create a Set Node** named `Get Fields to Cache`:
    - Assign fields:
      - `rss_item` from code node
      - `id` from `Get Video ID`
    - Connect `Create RSS Items` to this node.

15. **Create a Redis Node** named `Cache Video Info`:
    - Operation: `set`
    - Key: `yt-rss-item:{{ $json.id }}`
    - Value: `{{ $json.rss_item }}`
    - TTL: 604800 (7 days)
    - Connect `Get Fields to Cache` to this node.

16. **Create a Set Node** named `Get New RSS Items`:
    - Assign fields:
      - `rss_item` from `Create RSS Items`
      - `pubDate` from `Get relevant RSS fields`
    - Connect `Create RSS Items` to this node.

17. **Create a Merge Node** named `Merge RSS Items`:
    - No special parameters (default merge).
    - Connect `Get Cached RSS Items` to first input.
    - Connect `Get New RSS Items` to second input.

18. **Create a Sort Node** named `Sort by Date`:
    - Sort field: `pubDate`, order descending.
    - Connect `Merge RSS Items` to this node.

19. **Create a Code Node** named `Build RSS Feed`:
    - Build full RSS XML feed including channel metadata.
    - Concatenate all `rss_item` XML snippets.
    - Output property `feed` with XML string.
    - Connect `Sort by Date` to this node.

20. **Connect `Build RSS Feed` to `Respond to Webhook` node**:
    - Create Respond to Webhook node:
      - Response type: Text
      - Content-Type header: `application/xml`
      - Response body: `{{ $json.feed }}`
    - Connect `Build RSS Feed` to `Respond to Webhook`.

21. **Activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                          | Context or Link                                                                                           |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| This workflow creates a personalized RSS feed for YouTube videos from selected channels, allowing playback directly in RSS readers without visiting YouTube. It filters out Shorts and videos older than 1 week.                                       | General project purpose                                                                                   |
| Google API credentials are required to access the YouTube API and enrich video data. See [n8n Google Credentials Documentation](https://docs.n8n.io/integrations/builtin/credentials/google/) for setup instructions.                                 | Credential configuration                                                                                  |
| Redis is used to cache RSS items per video for 7 days to reduce API calls. If hosting n8n with Docker Compose, Redis can be added easily with a simple service definition. Redis credentials typically require host (`redis`), port (`6379`), user/pass. | Redis setup tip                                                                                           |
| To get a YouTube channel ID: visit the channel page, click "more" near description, use "Share Channel" button, then "Copy Channel-ID".                                                                                                              | Channel ID acquisition instructions                                                                       |
| The webhook URL must be copied from the webhook node and pasted into an RSS reader to subscribe to the generated feed.                                                                                                                                | Deployment instructions                                                                                   |
| The RSS feed XML structure and content can be customized by editing code nodes, including title, description, and embedded player HTML.                                                                                                               | Customization hint                                                                                        |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, a no-code automation tool. All content processed is legal and public; no illegal, offensive, or protected material is included.