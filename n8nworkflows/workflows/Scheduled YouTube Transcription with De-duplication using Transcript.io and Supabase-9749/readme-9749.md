Scheduled YouTube Transcription with De-duplication using Transcript.io and Supabase

https://n8nworkflows.xyz/workflows/scheduled-youtube-transcription-with-de-duplication-using-transcript-io-and-supabase-9749


# Scheduled YouTube Transcription with De-duplication using Transcript.io and Supabase

### 1. Workflow Overview

This workflow automates the process of finding recent YouTube videos from specified channels, checking for duplicates in a Supabase database, transcribing new videos using the youtube-transcript.io API, and saving the transcription results back to the database. It is designed to run on a schedule, efficiently managing video data ingestion, deduplication, transcription, and storage.

The workflow is logically divided into four main functional blocks:

- **1.1 Fetch Recent Videos:** Retrieve YouTube channel IDs, generate RSS feed URLs, fetch recent videos per channel, and filter videos by a configurable maximum age.

- **1.2 Filter New Content:** Check each video's URL against the Supabase database to exclude already processed videos, filtering out duplicates and optionally filtering out YouTube Shorts.

- **1.3 Transcription Processing:** Extract and validate YouTube video IDs, call the youtube-transcript.io API to obtain transcripts, handle transcription success or failure, and parse the transcript data.

- **1.4 Save to Database:** Merge transcription results with video metadata and insert new content entries into the Supabase content queue table.

---

### 2. Block-by-Block Analysis

#### 1.1 Fetch Recent Videos

- **Overview:**  
  This block initiates the workflow by pulling a predefined list of YouTube channel IDs, iterating over them to construct RSS feed URLs for each channel, fetching recent videos via RSS, and filtering out videos older than the maximum allowed age.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Set Max Video Age  
  - Channels To Track  
  - Split Out  
  - Channel Info  
  - Verify Channel ID + Create RSS Link  
  - Channel Info + Channel ID  
  - Find Channel's Videos (RSS Feed Read)  
  - Loop Over Each Channel  
  - Add Client & Age Rules  
  - Data with Raw Date  
  - Date & Time  
  - Merge Formatted Date  
  - Is Video Recent Enough?  
  - Loop Over Recent Videos  

- **Node Details:**

  1. **Schedule Trigger**  
     - Type: Trigger node  
     - Role: Starts the workflow on a scheduled basis (default triggers at 1 AM daily)  
     - Configuration: Interval set to trigger daily at hour 1  
     - Edge Cases: None specific, but schedule misconfiguration may lead to unintended run times.

  2. **Set Max Video Age**  
     - Type: Set node  
     - Role: Defines the max age (in days) of videos to consider (default 60 days)  
     - Key Variables: `max_content_age_days` set to "60" (string)  
     - Edge Cases: Setting a very low or very high value affects filtering.

  3. **Channels To Track**  
     - Type: Set node  
     - Role: Holds an array of YouTube channel IDs to monitor  
     - Configuration: `source_identifier` assigned an array of channel IDs  
     - Edge Cases: Empty or invalid channel arrays will halt further processing.

  4. **Split Out**  
     - Type: SplitOut node  
     - Role: Splits the array of channel IDs into individual items for iteration  
     - Edge Cases: Empty inputs produce no outputs.

  5. **Channel Info**  
     - Type: Set node  
     - Role: Assigns the current channel ID to `youtubeChannels` field for processing  
     - Input: Single channel ID from Split Out  
     - Output: Passes channel ID for verification.

  6. **Verify Channel ID + Create RSS Link**  
     - Type: Code node  
     - Role: Validates that channel IDs start with "UC" and have length 24; constructs the YouTube RSS feed URL accordingly  
     - Key Logic: Returns `channelId`, `rssUrl` (e.g. `https://www.youtube.com/feeds/videos.xml?channel_id=...`), plus passes through metadata  
     - Edge Cases: Invalid channel IDs are filtered out by returning null.

  7. **Channel Info + Channel ID**  
     - Type: Set node  
     - Role: Passes RSS URL downstream  
     - Key Variables: `rssUrl` from previous node

  8. **Find Channel's Videos**  
     - Type: RSS Feed Read node  
     - Role: Reads RSS feed to fetch latest videos for the channel  
     - Input: RSS feed URL  
     - Edge Cases: Network errors, invalid RSS, or empty feeds.

  9. **Loop Over Each Channel**  
     - Type: SplitInBatches node  
     - Role: Processes each channel’s videos in batches for rate limiting and performance  
     - Edge Cases: Batch size configuration affects processing speed and resource usage.

  10. **Add Client & Age Rules**  
      - Type: Merge node (combineAll)  
      - Role: Combines channel video data with max age and other rules for filtering  
      - Input: Video data and max age configuration

  11. **Data with Raw Date**  
      - Type: Set node  
      - Role: Prepares raw video publish date for formatting

  12. **Date & Time**  
      - Type: DateTime node  
      - Role: Formats the video publish date to a standard string (yyyy-MM-dd)

  13. **Merge Formatted Date**  
      - Type: Merge node (combineByPosition)  
      - Role: Combines formatted date back with the original video data

  14. **Is Video Recent Enough?**  
      - Type: If node  
      - Role: Filters videos published after or equal to the date calculated by subtracting `max_content_age_days` from current date  
      - Key Expressions: Compares `standardizedPubDate` against `DateTime.now().minus({ days: max_content_age_days })`  
      - Edge Cases: Incorrect date formats, timezone issues.

  15. **Loop Over Recent Videos**  
      - Type: SplitInBatches node  
      - Role: Iterates over filtered recent videos to process further downstream

---

#### 1.2 Filter New Content

- **Overview:**  
  This block checks each candidate video URL against the Supabase database to detect duplicates. It filters out videos already present in the database and optionally filters out YouTube Shorts.

- **Nodes Involved:**  
  - author + title + link + pubDate  
  - Check if URL Is In Database (Supabase)  
  - Merge DB Check Result  
  - Is Video Already in Database?  
  - Discard URL  
  - Filter Out YouTube Shorts  
  - Rename Original URL  
  - New Video Information  
  - Loop Over New Videos  

- **Node Details:**

  1. **author + title + link + pubDate**  
     - Type: Set node  
     - Role: Extracts and structures key video metadata for database queries  
     - Fields: author, title, link, pubDate  
     - Input: Video data  
     - Output: Structured metadata for checking

  2. **Check if URL Is In Database**  
     - Type: Supabase node  
     - Role: Queries `content_queue_1` table filtering by `source_url` equal to current video link  
     - Configuration: Returns all matching entries to detect duplicates  
     - Credentials: Requires valid Supabase API credentials  
     - On Error: Continues to output to avoid workflow halt  
     - Edge Cases: Network issues, auth errors, malformed queries

  3. **Merge DB Check Result**  
     - Type: Merge node (combineByPosition)  
     - Role: Combines query results with video metadata for conditional filtering

  4. **Is Video Already in Database?**  
     - Type: If node  
     - Role: Checks if the `source_url` exists in database results (i.e., video processed before)  
     - Condition: `source_url` exists (non-null)  
     - Output:  
       - True branch: Video exists → Discard URL node  
       - False branch: Video is new → continue processing

  5. **Discard URL**  
     - Type: Code node  
     - Role: Explicitly discards items by returning empty array  
     - Effect: Stops further processing of duplicates

  6. **Filter Out YouTube Shorts**  
     - Type: If node  
     - Role: Filters out videos whose URL contains "youtube.com/shorts" (unless user deletes this condition to include shorts)  
     - Edge Cases: Incorrect URL formats or missing URL fields

  7. **Rename Original URL**  
     - Type: Set node  
     - Role: Renames `link` field to `original_url` for consistent processing downstream  
     - Input: Video data

  8. **New Video Information**  
     - Type: Set node  
     - Role: Rebuilds video metadata for transcription and storage after filtering  
     - Fields: author, title, link, pubDate, plus any other fields preserved

  9. **Loop Over New Videos**  
     - Type: SplitInBatches node  
     - Role: Processes filtered new videos individually for transcription

---

#### 1.3 Transcription Processing

- **Overview:**  
  This block extracts clean YouTube video IDs from URLs, validates them, calls the youtube-transcript.io API for transcription, handles success or failure, and parses the returned transcript data for further use.

- **Nodes Involved:**  
  - Rename URL  
  - try to get video_id again (Code)  
  - Find Video ID  
  - Is Video ID valid?  
  - Clean Up URL  
  - Was Video ID Found? (If node)  
  - Merge Video ID With Video Data  
  - Get Transcript from API (HTTP Request)  
  - Transcript Worked? (If node)  
  - Official Captions/Transcript (Code)  
  - Transcript Failed (Stop and Error)  

- **Node Details:**

  1. **Rename URL**  
     - Type: Set node  
     - Role: Sets `url` field from `original_url` for uniformity before ID extraction

  2. **try to get video_id again**  
     - Type: Code node  
     - Role: Uses regex to extract YouTube video ID from various URL formats  
     - Output: JSON with original URL, video ID, validity flag, and cleaned URL  
     - Edge Cases: Invalid URLs, no match returns null video ID

  3. **Find Video ID**  
     - Type: Set node  
     - Role: Extracts video ID using regex directly from `original_url` field  
     - Stores video_id field for downstream use

  4. **Is Video ID valid?**  
     - Type: Set node  
     - Role: Sets boolean `is_valid` indicating whether video ID extraction succeeded

  5. **Clean Up URL**  
     - Type: Set node  
     - Role: Creates a clean YouTube video URL using the extracted video ID

  6. **Was Video ID Found?**  
     - Type: If node  
     - Role: Checks `is_valid` flag  
     - True branch: Proceed with transcription  
     - False branch: Attempts re-extraction via Rename URL → try to get video_id again

  7. **Merge Video ID With Video Data**  
     - Type: Merge node (combineByPosition)  
     - Role: Combines video metadata with extracted video ID info for the API call

  8. **Get Transcript from API**  
     - Type: HTTP Request node  
     - Role: Sends POST request to `https://www.youtube-transcript.io/api/transcripts` with video ID in JSON body  
     - Authentication: Uses `youtube-transcript.io` API key via HTTP Header Auth credential  
     - Retry: Enabled with 5-second delay on failure  
     - Output: Raw API response as text  
     - Edge Cases: API service unavailable, rate limits, invalid video IDs, videos without captions  
     - Failure Handling: Will not error the workflow, but relies on downstream node to detect success.

  9. **Transcript Worked?**  
     - Type: If node  
     - Role: Checks if API response statusMessage equals "OK" (indicating success)  
     - True branch: Proceed to parse transcript  
     - False branch: Stop execution with error "Transcript Failed"

  10. **Official Captions/Transcript**  
      - Type: Code node  
      - Role: Parses JSON transcript data from API, concatenates transcript text segments into a single string  
      - Output: JSON with `id` and `transcript` string  
      - Edge Cases: Malformed JSON, empty transcript arrays

  11. **Transcript Failed**  
      - Type: Stop and Error node  
      - Role: Stops workflow execution for the current item with a descriptive error message  
      - Effect: Prevents saving failed transcripts

---

#### 1.4 Save to Database

- **Overview:**  
  This block merges finalized transcript data with video metadata and inserts the new content record into the Supabase database. It enforces a wait delay to control throughput.

- **Nodes Involved:**  
  - Add Transcript to Video Data (Merge)  
  - Add to Content Queue Table (Supabase)  
  - Wait  

- **Node Details:**

  1. **Add Transcript to Video Data**  
     - Type: Merge node (combineByPosition)  
     - Role: Combines transcript text with video metadata for database insertion

  2. **Add to Content Queue Table**  
     - Type: Supabase node  
     - Role: Inserts a new row into `content_queue_1` table with fields:  
       - `content_type`: fixed "youtube"  
       - `title`, `source_url`, `content_snippet` (transcript), `status` ("new"), `published_date`, `creator`  
     - Credentials: Requires valid Supabase API credentials  
     - On Error: Continues to avoid stopping workflow on DB errors  
     - Edge Cases: DB connection failures, constraint violations

  3. **Wait**  
     - Type: Wait node  
     - Role: Delays the workflow for 20 seconds after insertion to prevent rate-limiting or overload  
     - Configuration: Fixed 20 seconds delay

---

### 3. Summary Table

| Node Name                     | Node Type             | Functional Role                             | Input Node(s)                      | Output Node(s)                      | Sticky Note                                                                                                   |
|-------------------------------|----------------------|---------------------------------------------|----------------------------------|-----------------------------------|--------------------------------------------------------------------------------------------------------------|
| Schedule Trigger               | Schedule Trigger     | Starts workflow on schedule                  | -                                | Set Max Video Age                  | How often do you want this to run? **PRO TIP**: Replace this with a schedule node!                            |
| Set Max Video Age              | Set                  | Sets max video age (default 60 days)         | Schedule Trigger                 | Channels To Track, Max Content Age Days | Filter out Youtube Videos posted X days ago. Default = 60 days.                                             |
| Channels To Track              | Set                  | Holds list of YouTube channel IDs             | Set Max Video Age                | Split Out                        | Stores the Channel IDs of the youtube channels you are tracking. Find channel IDs for free at https://www.tunepocket.com/youtube-channel-id-finder |
| Split Out                     | SplitOut             | Splits channels array into individual items   | Channels To Track                | Channel Info                     | Splits all of the websites into their own items so that they go into the loop one at a time                   |
| Channel Info                  | Set                  | Assigns current channel ID                     | Split Out                      | Verify Channel ID + Create RSS Link |                                                                                                              |
| Verify Channel ID + Create RSS Link | Code                | Validates channel ID and creates RSS URL      | Channel Info                   | Channel Info + Channel ID         |                                                                                                              |
| Channel Info + Channel ID      | Set                  | Passes RSS URL downstream                      | Verify Channel ID + Create RSS Link | Loop Over Each Channel            | Convert to a valid RSS feed link                                                                              |
| Loop Over Each Channel         | SplitInBatches       | Processes each channel in batches              | Channel Info + Channel ID       | Add Client & Age Rules, Find Channel's Videos | Part 1: Get Recent Videos                                                                                     |
| Find Channel's Videos          | RSS Feed Read        | Reads RSS feed to fetch videos                  | Loop Over Each Channel          | Loop Over Each Channel (loop)     |                                                                                                              |
| Add Client & Age Rules         | Merge                | Combines video data with max age rules         | Loop Over Each Channel, Max Content Age Days | Data with Raw Date                |                                                                                                              |
| Data with Raw Date             | Set                  | Prepares raw publish date                        | Add Client & Age Rules          | Date & Time, Merge Formatted Date |                                                                                                              |
| Date & Time                   | DateTime             | Formats publish date                             | Data with Raw Date              | Merge Formatted Date              |                                                                                                              |
| Merge Formatted Date           | Merge                | Combines formatted date with video data         | Date & Time, Data with Raw Date | Is Video Recent Enough?           | Filter out videos older than your preferred number of days. Default = 60 days                                |
| Is Video Recent Enough?        | If                   | Filters videos by age                            | Merge Formatted Date            | Loop Over Recent Videos           |                                                                                                              |
| Loop Over Recent Videos        | SplitInBatches       | Processes recent videos                          | Is Video Recent Enough?         | author + title + link + pubDate, Loop Over New Videos | Part 2: Filter for New Content Only                                                                           |
| author + title + link + pubDate | Set                  | Extracts video metadata                          | Loop Over Recent Videos         | Check if URL Is In Database, Merge DB Check Result |                                                                                                              |
| Check if URL Is In Database    | Supabase             | Checks database for duplicate videos            | author + title + link + pubDate | Merge DB Check Result             | You must add your Supabase credentials to this node and the 'Add to Content Queue Table' node                 |
| Merge DB Check Result          | Merge                | Combines DB check results with video metadata   | author + title + link + pubDate, Check if URL Is In Database | Is Video Already in Database?  |                                                                                                              |
| Is Video Already in Database?  | If                   | Filters out videos already in DB                 | Merge DB Check Result           | Discard URL (if exists), Loop Over Recent Videos (if new) |                                                                                                              |
| Discard URL                   | Code                 | Discards duplicate videos                        | Is Video Already in Database? (true) | Loop Over Recent Videos            |                                                                                                              |
| Filter Out YouTube Shorts      | If                   | Filters out YouTube Shorts videos                | Loop Over New Videos            | Rename Original URL (if allowed), Loop Over New Videos (if shorts) | Filter Out Youtube Shorts? Default = Will NOT extract transcript from Youtube Shorts. Delete condition to include shorts |
| Rename Original URL            | Set                  | Renames `link` to `original_url`                 | Filter Out YouTube Shorts       | Find Video ID, New Video Information |                                                                                                              |
| New Video Information          | Set                  | Prepares video metadata for transcription         | Rename Original URL             | Merge Video ID With Video Data     |                                                                                                              |
| Rename URL                   | Set                  | Sets `url` from `original_url`                    | Was Video ID Found? (false)     | try to get video_id again           |                                                                                                              |
| try to get video_id again      | Code                 | Extracts YouTube video ID using regex             | Rename URL                     | Merge Video ID With Video Data     |                                                                                                              |
| Find Video ID                 | Set                  | Extracts video ID from URL using regex            | Rename Original URL             | Is Video ID valid?                 |                                                                                                              |
| Is Video ID valid?            | Set                  | Sets boolean flag if video ID is valid            | Find Video ID                  | Clean Up URL                      |                                                                                                              |
| Clean Up URL                  | Set                  | Creates a clean YouTube video URL                  | Is Video ID valid?              | Was Video ID Found?                |                                                                                                              |
| Was Video ID Found?           | If                   | Checks if video ID extraction succeeded            | Clean Up URL                   | Merge Video ID With Video Data (if true), Rename URL (if false) |                                                                                                              |
| Merge Video ID With Video Data | Merge                | Merges video data with video ID info               | New Video Information, try to get video_id again or Was Video ID Found? | Get Transcript from API, Add Transcript to Video Data |                                                                                                              |
| Get Transcript from API        | HTTP Request         | Calls youtube-transcript.io API for transcription | Merge Video ID With Video Data | Transcript Worked?                | Use your youtube-transcript.io API key here: https://www.youtube-transcript.io/                               |
| Transcript Worked?             | If                   | Checks if transcription succeeded                   | Get Transcript from API        | Official Captions/Transcript (if OK), Transcript Failed (if not) |                                                                                                              |
| Official Captions/Transcript   | Code                 | Parses and concatenates transcript text             | Transcript Worked? (true)       | Add Transcript to Video Data        |                                                                                                              |
| Transcript Failed             | Stop and Error       | Stops workflow on transcription failure            | Transcript Worked? (false)      | -                                 |                                                                                                              |
| Add Transcript to Video Data   | Merge                | Combines transcript with video metadata             | Official Captions/Transcript, Merge Video ID With Video Data | Add to Content Queue Table           |                                                                                                              |
| Add to Content Queue Table     | Supabase             | Inserts new video and transcript into database      | Add Transcript to Video Data    | Wait                             | You must add your Supabase credentials to this node and the 'Check if URL Is In Database' node               |
| Wait                         | Wait                 | Delays 20 seconds between inserts                   | Add to Content Queue Table      | Loop Over New Videos              |                                                                                                              |
| Discard URL                   | Code                 | Discards duplicates                                 | Is Video Already in Database? (true) | Loop Over Recent Videos           |                                                                                                              |
| Sticky Notes (various nodes)   | Sticky Note          | Provides workflow guidance, setup instructions, and part descriptions | -                              | -                               | Multiple sticky notes guide the user on setup, filtering logic, API keys, and workflow customization          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger** node  
   - Type: Schedule Trigger  
   - Set to run daily at 1 AM (or your preferred schedule).

2. **Create Set node "Set Max Video Age"**  
   - Assign variable: `max_content_age_days` = "60" (string)  
   - Connect from Schedule Trigger.

3. **Create Set node "Channels To Track"**  
   - Assign variable: `source_identifier` = array of YouTube Channel IDs (e.g., ["UCaEkuhQejDMyindRnUbISIg", "UCIPPMRA040LQr5QPyJEbmXA"])  
   - Connect from "Set Max Video Age".

4. **Create SplitOut node "Split Out"**  
   - Field to split out: `source_identifier`  
   - Connect from "Channels To Track".

5. **Create Set node "Channel Info"**  
   - Assign variable: `youtubeChannels` = current split channel ID item  
   - Connect from "Split Out".

6. **Create Code node "Verify Channel ID + Create RSS Link"**  
   - Validate channel ID format (starts with "UC" and length 24)  
   - If valid, output `channelId`, `rssUrl` = `https://www.youtube.com/feeds/videos.xml?channel_id=${channelId}` plus metadata  
   - Else, return null to skip  
   - Connect from "Channel Info".

7. **Create Set node "Channel Info + Channel ID"**  
   - Assign variable `rssUrl` from previous output  
   - Connect from "Verify Channel ID + Create RSS Link".

8. **Create SplitInBatches node "Loop Over Each Channel"**  
   - Connect from "Channel Info + Channel ID".

9. **Create RSS Feed Read node "Find Channel's Videos"**  
   - URL: `{{$json.rssUrl}}`  
   - Connect from "Loop Over Each Channel".

10. **Create Merge node "Add Client & Age Rules"**  
    - Mode: combineAll  
    - Inputs: "Loop Over Each Channel" and "Set Max Video Age"  
    - Connect "Find Channel's Videos" to "Loop Over Each Channel" input.

11. **Create Set node "Data with Raw Date"**  
    - Pass through all fields, no changes  
    - Connect from "Add Client & Age Rules".

12. **Create DateTime node "Date & Time"**  
    - Input date: `{{$json.pubDate || $json.isoDate}}`  
    - Format: `yyyy-MM-dd`  
    - Output field: `standardizedPubDate`  
    - Connect from "Data with Raw Date".

13. **Create Merge node "Merge Formatted Date"**  
    - Mode: combineByPosition  
    - Inputs: "Date & Time" and "Data with Raw Date"  
    - Connect accordingly.

14. **Create If node "Is Video Recent Enough?"**  
    - Condition: `standardizedPubDate` is after or equals `DateTime.now().minus({ days: max_content_age_days })`  
    - Connect from "Merge Formatted Date".

15. **Create SplitInBatches node "Loop Over Recent Videos"**  
    - Connect from True branch of "Is Video Recent Enough?".

16. **Create Set node "author + title + link + pubDate"**  
    - Assign variables: author, title, link, pubDate from video data  
    - Connect from "Loop Over Recent Videos".

17. **Create Supabase node "Check if URL Is In Database"**  
    - Operation: getAll from table `content_queue_1`  
    - Filter: `source_url` equals `{{$json.link}}`  
    - Credentials: Add your Supabase API credentials  
    - On error: continue output  
    - Connect from "author + title + link + pubDate".

18. **Create Merge node "Merge DB Check Result"**  
    - Mode: combineByPosition  
    - Inputs: "author + title + link + pubDate" and "Check if URL Is In Database"  
    - Connect accordingly.

19. **Create If node "Is Video Already in Database?"**  
    - Condition: `source_url` exists in DB result  
    - Connect from "Merge DB Check Result".

20. **Create Code node "Discard URL"**  
    - Code: `return [];` to discard duplicate videos  
    - Connect from True branch of "Is Video Already in Database?".

21. **Create If node "Filter Out YouTube Shorts"**  
    - Condition: `link` exists AND does not contain "youtube.com/shorts"  
    - Connect from False branch of "Is Video Already in Database?".

22. **Create Set node "Rename Original URL"**  
    - Rename `link` to `original_url`  
    - Connect from True branch of "Filter Out YouTube Shorts".

23. **Create Set node "New Video Information"**  
    - Assign author, title, link, pubDate, preserving other fields  
    - Connect from "Rename Original URL".

24. **Create SplitInBatches node "Loop Over New Videos"**  
    - Connect from "New Video Information".

25. **Create Set node "Rename URL"**  
    - Assign `url` = `original_url`  
    - Connect from False branch of "Was Video ID Found?".

26. **Create Code node "try to get video_id again"**  
    - Extract video ID using regex from `url` field, output video_id, is_valid, clean_url  
    - Connect from "Rename URL".

27. **Create Set node "Find Video ID"**  
    - Extract video_id from `original_url` using regex  
    - Connect from "Rename Original URL".

28. **Create Set node "Is Video ID valid?"**  
    - Assign boolean `is_valid` based on regex match success  
    - Connect from "Find Video ID".

29. **Create Set node "Clean Up URL"**  
    - Assign `clean_url` based on `video_id`  
    - Connect from "Is Video ID valid?".

30. **Create If node "Was Video ID Found?"**  
    - Condition: `is_valid` == true  
    - Connect from "Clean Up URL".

31. **Create Merge node "Merge Video ID With Video Data"**  
    - Combine video metadata and video ID data  
    - Connect from True branch of "Was Video ID Found?" and from "try to get video_id again".

32. **Create HTTP Request node "Get Transcript from API"**  
    - URL: `https://www.youtube-transcript.io/api/transcripts`  
    - Method: POST  
    - Body JSON: `{ "ids": ["{{$json.video_id}}"] }`  
    - Authentication: HTTP Header Auth with your youtube-transcript.io API key  
    - Retry On Fail: true, wait 5 seconds  
    - Connect from "Merge Video ID With Video Data".

33. **Create If node "Transcript Worked?"**  
    - Condition: `$json.statusMessage` equals "OK"  
    - Connect from "Get Transcript from API".

34. **Create Code node "Official Captions/Transcript"**  
    - Parse JSON transcript data, concatenate transcript text  
    - Connect from True branch of "Transcript Worked?".

35. **Create Stop and Error node "Transcript Failed"**  
    - Error message: "Transcript Failed"  
    - Connect from False branch of "Transcript Worked?".

36. **Create Merge node "Add Transcript to Video Data"**  
    - Combine transcript text with video metadata  
    - Connect from "Official Captions/Transcript" and "Merge Video ID With Video Data".

37. **Create Supabase node "Add to Content Queue Table"**  
    - Insert row into `content_queue_1` with fields:  
      `content_type` = "youtube", `title`, `source_url`, `content_snippet` (transcript), `status` = "new", `published_date`, `creator`  
    - Credentials: Same Supabase credentials as earlier  
    - On Error: continue output  
    - Connect from "Add Transcript to Video Data".

38. **Create Wait node "Wait"**  
    - Delay: 20 seconds  
    - Connect from "Add to Content Queue Table".

39. **Connect Wait node back to "Loop Over New Videos"**  
    - Enables batch processing with delay between inserts.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                               | Context or Link                                                                                       |
|------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| **Advanced YouTube Transcriptor**: Runs scheduled to find new YouTube videos, deduplicate, transcribe, and save. Setup involves max days, channels, API key, and Supabase credentials. | Detailed sticky note at workflow start.                                                           |
| Find YouTube Channel IDs for free using: https://www.tunepocket.com/youtube-channel-id-finder                                                               | Sticky note on Channels To Track node.                                                             |
| youtube-transcript.io API Key: Provides 25 free YouTube transcripts per month; only works on videos with captions. Get API key at https://www.youtube-transcript.io/ | Sticky note near Get Transcript from API node.                                                    |
| To transcribe YouTube Shorts: Delete the second filter condition in the "Filter Out YouTube Shorts" node                                                   | Sticky note on Filter Out YouTube Shorts node.                                                     |
| Workflow schedule can be changed via the Schedule Trigger node                                                                                              | Sticky note near Schedule Trigger node.                                                            |

---

**Disclaimer:** The provided content comes exclusively from an automated workflow created with n8n, strictly respecting current content policies and containing no illegal, offensive, or protected elements. All processed data is legal and public.