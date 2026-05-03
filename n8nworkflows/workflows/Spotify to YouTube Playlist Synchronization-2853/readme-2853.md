Spotify to YouTube Playlist Synchronization

https://n8nworkflows.xyz/workflows/spotify-to-youtube-playlist-synchronization-2853


# Spotify to YouTube Playlist Synchronization

### 1. Workflow Overview

This workflow synchronizes a YouTube playlist with a Spotify playlist in a one-way manner (Spotify → YouTube). It continuously monitors changes in the Spotify playlist, updates a Supabase database to reflect the current state, and smartly matches Spotify tracks to YouTube videos based on title, artist, and duration. It also maintains the YouTube playlist by removing deleted or unmatched videos and periodically resets unmatched flags to retry matching.

The workflow is logically divided into three main blocks:

- **1.1 Change Detection and Database Synchronization**  
  Detects changes in the Spotify playlist snapshot, compares Spotify tracks with the database, adds new tracks, and marks removed tracks for deletion.

- **1.2 YouTube Video Matching and Playlist Update**  
  For tracks missing YouTube video IDs, searches YouTube for matching videos considering duration tolerance, adds matched videos to the YouTube playlist, updates the database, and notifies via Discord.

- **1.3 YouTube Playlist Maintenance and Recovery**  
  Monitors the YouTube playlist for deleted videos, flags them for re-search, deletes marked entries from the database, and periodically resets unmatched flags to retry matching.

---

### 2. Block-by-Block Analysis

#### 1.1 Change Detection and Database Synchronization

**Overview:**  
This block monitors the Spotify playlist for changes using snapshot IDs, compares the current Spotify tracks with the database entries, adds new tracks to the database, and marks tracks for deletion if they are no longer in the Spotify playlist.

**Nodes Involved:**  
- Every hour (Schedule Trigger)  
- variables (Set)  
- Get playlist snapshot (Spotify)  
- Wait 1 hour (Wait)  
- Get playlist snapshot1 (Spotify)  
- If different snapshot (If)  
- Spotify (Get Tracks)  
- Get all musics (Supabase)  
- Compare Datasets (Compare Spotify tracks vs DB)  
- Add music (Supabase)  
- Update to_delete to true (Supabase)  
- No Operation, do nothing (NoOp)  

**Node Details:**

- **Every hour**  
  - Type: Schedule Trigger  
  - Role: Triggers the workflow every hour to check for playlist changes.  
  - Config: Interval set to every hour.  
  - Outputs to: variables node.  
  - Edge cases: Trigger failure or delay may cause sync lag.

- **variables**  
  - Type: Set  
  - Role: Defines key variables such as Spotify playlist ID, YouTube playlist ID, and Supabase table name.  
  - Config: Hardcoded values for playlist IDs and table name "musics".  
  - Outputs to: Get playlist snapshot.  
  - Edge cases: Incorrect variable values will break downstream API calls.

- **Get playlist snapshot**  
  - Type: Spotify node  
  - Role: Retrieves the current snapshot ID of the Spotify playlist.  
  - Config: Uses variable `spotify_playlist_id`.  
  - Outputs to: Wait 1 hour.  
  - Edge cases: Spotify API auth errors, rate limits.

- **Wait 1 hour**  
  - Type: Wait  
  - Role: Delays execution by 1 hour to allow for snapshot comparison after some time.  
  - Outputs to: Get playlist snapshot1.  
  - Edge cases: Workflow timeout or interruption.

- **Get playlist snapshot1**  
  - Type: Spotify node  
  - Role: Retrieves a second snapshot ID for comparison.  
  - Config: Uses variable `spotify_playlist_id`.  
  - Outputs to: If different snapshot.  
  - Edge cases: Same as first snapshot node.

- **If different snapshot**  
  - Type: If  
  - Role: Compares the two snapshot IDs to detect changes in the Spotify playlist.  
  - Condition: Checks if snapshot IDs differ.  
  - Outputs:  
    - True: Spotify (Get Tracks) and Get all musics (Supabase) nodes.  
    - False: No Operation, do nothing node.  
  - Edge cases: Expression evaluation failure.

- **Spotify (Get Tracks)**  
  - Type: Spotify node  
  - Role: Retrieves all tracks from the Spotify playlist.  
  - Config: Uses variable `spotify_playlist_id`, returns all tracks.  
  - Outputs to: Compare Datasets.  
  - Edge cases: API rate limits, auth errors.

- **Get all musics**  
  - Type: Supabase node  
  - Role: Retrieves all music entries from the Supabase database table.  
  - Config: Uses variable `supabase_table_name`, returns all rows.  
  - Outputs to: Compare Datasets.  
  - Edge cases: Database connection errors.

- **Compare Datasets**  
  - Type: Compare Datasets  
  - Role: Compares Spotify tracks with database entries by matching Spotify track ID to DB ID.  
  - Config: Skips some fields irrelevant for comparison; merges by `track.id` and `id`.  
  - Outputs:  
    - Added tracks → Add music node  
    - Removed tracks → Update to_delete to true node  
  - Edge cases: Data mismatch, expression errors.

- **Add music**  
  - Type: Supabase node  
  - Role: Inserts new Spotify tracks into the database with fields: id, title, artist, duration.  
  - Config: Uses fields from Spotify track data.  
  - Outputs to: data node (prepares data for next steps).  
  - Edge cases: DB insert errors, duplicate keys.

- **Update to_delete to true**  
  - Type: Supabase node  
  - Role: Marks tracks as to_delete = TRUE in the database when removed from Spotify playlist.  
  - Config: Filters by track ID.  
  - Outputs: None.  
  - Edge cases: DB update failures.

- **No Operation, do nothing**  
  - Type: NoOp  
  - Role: Placeholder when no changes detected.  
  - Outputs: None.

---

#### 1.2 YouTube Video Matching and Playlist Update

**Overview:**  
This block processes tracks without YouTube video IDs, searches YouTube for matching videos based on title and artist, filters results by duration tolerance (±10%), adds matched videos to the YouTube playlist, updates the database with the video ID, and sends notifications via Discord. Tracks with no match are marked as "NOTFOUND".

**Nodes Involved:**  
- variables1 (Set)  
- Get all musics not in youtube playlist (Supabase)  
- data (Set)  
- Loop Over Items (SplitInBatches)  
- Search video (YouTube)  
- If no result (If)  
- Aggregate (Aggregate)  
- If video duration ~= music duration (If)  
- Get video duration (YouTube)  
- Add music to playlist (YouTube)  
- Add youtube id to row (Supabase)  
- Set youtube id to NOTFOUND if no matching (Supabase)  
- Loop Over Items1 (SplitInBatches)  
- data1 (Set)  
- Discord (Discord)  
- Discord1 (Discord)  
- Sticky Note (explanatory)

**Node Details:**

- **variables1**  
  - Type: Set  
  - Role: Defines variables for Spotify playlist ID, YouTube playlist ID, and Supabase table name.  
  - Outputs to: Get all musics not in youtube playlist.  
  - Edge cases: Incorrect variables break downstream nodes.

- **Get all musics not in youtube playlist**  
  - Type: Supabase node  
  - Role: Retrieves all tracks from DB where `youtube_video_id` is null and `to_delete` is FALSE (i.e., tracks needing YouTube matching).  
  - Outputs to: data node.  
  - Edge cases: DB connection errors.

- **data**  
  - Type: Set  
  - Role: Prepares and normalizes track data fields (title, artist, duration, id, supabase_table_name, youtube_playlist_id) for processing.  
  - Outputs to: Loop Over Items.  
  - Edge cases: Expression errors.

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Processes tracks one by one to avoid API rate limits.  
  - Outputs to: Search video and Aggregate nodes.  
  - Edge cases: Batch size misconfiguration.

- **Search video**  
  - Type: YouTube node  
  - Role: Searches YouTube videos using query "title - artist", limits to top 5 results.  
  - Outputs to: If no result.  
  - Edge cases: YouTube API quota limits, auth errors.

- **If no result**  
  - Type: If  
  - Role: Checks if search returned no results.  
  - Outputs:  
    - True: Aggregate node (to mark NOTFOUND)  
    - False: Loop Over Items (to check video duration)  
  - Edge cases: Expression failures.

- **Aggregate**  
  - Type: Aggregate  
  - Role: Aggregates search results for further processing.  
  - Outputs to: Set youtube id to NOTFOUND if no matching.  
  - Edge cases: Data aggregation errors.

- **If video duration ~= music duration**  
  - Type: If  
  - Role: Compares YouTube video duration with Spotify track duration allowing ±10% tolerance (actually -5s to +20s).  
  - Outputs:  
    - True: Add music to playlist  
    - False: Loop Over Items (to check next video)  
  - Edge cases: Parsing ISO 8601 duration string may fail if format unexpected.

- **Get video duration**  
  - Type: YouTube node  
  - Role: Retrieves detailed video info including duration for comparison.  
  - Outputs to: If video duration ~= music duration.  
  - Edge cases: API errors.

- **Add music to playlist**  
  - Type: YouTube node  
  - Role: Adds the matched YouTube video to the specified YouTube playlist.  
  - Config: Uses video ID from Get video duration node, playlist ID from variables.  
  - Outputs to: Add youtube id to row and Discord nodes.  
  - Edge cases: Playlist API quota limits, permission errors.

- **Add youtube id to row**  
  - Type: Supabase node  
  - Role: Updates the database row with the matched YouTube video ID.  
  - Config: Filters by track ID.  
  - Outputs to: Loop Over Items1 and Discord nodes.  
  - Edge cases: DB update failures.

- **Set youtube id to NOTFOUND if no matching**  
  - Type: Supabase node  
  - Role: Marks tracks with no matching YouTube video as "NOTFOUND" in the database.  
  - Outputs to: Loop Over Items1 and Discord1 nodes.  
  - Edge cases: DB update failures.

- **Loop Over Items1**  
  - Type: SplitInBatches  
  - Role: Processes matched or unmatched tracks for notifications.  
  - Outputs to: data1 node and Discord nodes.  
  - Edge cases: Batch size misconfiguration.

- **data1**  
  - Type: Set  
  - Role: Prepares data for notifications (title, artist, duration, id, playlist ID).  
  - Outputs to: Search video and Discord nodes.  
  - Edge cases: Expression errors.

- **Discord**  
  - Type: Discord node  
  - Role: Sends notification message for successfully added videos with title and YouTube link.  
  - Config: Uses webhook authentication.  
  - Edge cases: Discord webhook failures.

- **Discord1**  
  - Type: Discord node  
  - Role: Sends notification message for tracks with no matching YouTube video.  
  - Config: Uses webhook authentication.  
  - Edge cases: Discord webhook failures.

- **Sticky Note**  
  - Content: Explains the YouTube matching logic, duration tolerance, and automation details.

---

#### 1.3 YouTube Playlist Maintenance and Recovery

**Overview:**  
This block maintains the YouTube playlist by detecting deleted videos, marking them for re-search, deleting database entries flagged for deletion, and periodically resetting "NOTFOUND" flags to retry matching.

**Nodes Involved:**  
- Every day at midnight (Schedule Trigger)  
- variables3 (Set)  
- Get all musics that should be in playlist (Supabase)  
- Get playlist items (YouTube)  
- Check for deleted videos (Compare Datasets)  
- Set youtube_video_id to null (Supabase)  
- Get all musics to be deleted (Supabase)  
- Playlist items to be deleted (Compare Datasets)  
- Remove Duplicates (Remove Duplicates)  
- Remove video from playlist (YouTube)  
- Delete music (Supabase)  
- Reset NOTFOUND id to NULL (Supabase)  
- variables4 (Set)  
- Every month (Schedule Trigger)  

**Node Details:**

- **Every day at midnight**  
  - Type: Schedule Trigger  
  - Role: Triggers daily maintenance tasks.  
  - Outputs to: variables3 node.  
  - Edge cases: Trigger failures.

- **variables3**  
  - Type: Set  
  - Role: Defines variables for playlist and table names.  
  - Outputs to: Get all musics that should be in playlist, Get playlist items, Get all musics to be deleted.  
  - Edge cases: Incorrect variables.

- **Get all musics that should be in playlist**  
  - Type: Supabase node  
  - Role: Retrieves all musics with valid YouTube video IDs (not null or NOTFOUND).  
  - Outputs to: Check for deleted videos.  
  - Edge cases: DB errors.

- **Get playlist items**  
  - Type: YouTube node  
  - Role: Retrieves all items from the YouTube playlist.  
  - Outputs to: Playlist items to be deleted and Check for deleted videos.  
  - Edge cases: API quota limits.

- **Check for deleted videos**  
  - Type: Compare Datasets  
  - Role: Compares DB YouTube video IDs with actual videos in the playlist to detect deleted videos.  
  - Outputs to: Set youtube_video_id to null.  
  - Edge cases: Data mismatch.

- **Set youtube_video_id to null**  
  - Type: Supabase node  
  - Role: Resets YouTube video ID to null for deleted videos (except those marked NOTFOUND).  
  - On error: Continues regular output to avoid workflow stop.  
  - Outputs: None.  
  - Edge cases: DB update failures.

- **Get all musics to be deleted**  
  - Type: Supabase node  
  - Role: Retrieves all musics marked `to_delete = TRUE` and with YouTube video ID not equal to NOTFOUND.  
  - Outputs to: Playlist items to be deleted.  
  - Edge cases: DB errors.

- **Playlist items to be deleted**  
  - Type: Compare Datasets  
  - Role: Compares playlist items with musics to be deleted to find which playlist items to remove.  
  - Outputs to: Remove Duplicates and Check for deleted videos.  
  - Edge cases: Data mismatch.

- **Remove Duplicates**  
  - Type: Remove Duplicates  
  - Role: Removes duplicate entries from the list of playlist items to delete.  
  - Outputs to: Remove video from playlist.  
  - Edge cases: Incorrect field comparison.

- **Remove video from playlist**  
  - Type: YouTube node  
  - Role: Deletes playlist items from YouTube playlist based on playlistItemId.  
  - Outputs to: Delete music.  
  - Edge cases: API permission errors.

- **Delete music**  
  - Type: Supabase node  
  - Role: Deletes music entries from the database that have been removed from the playlist and marked for deletion.  
  - Outputs: None.  
  - Edge cases: DB delete failures.

- **Reset NOTFOUND id to NULL**  
  - Type: Supabase node  
  - Role: Periodically resets `youtube_video_id` from "NOTFOUND" to null to retry matching.  
  - Outputs: None.  
  - Edge cases: DB update failures.

- **variables4**  
  - Type: Set  
  - Role: Defines variables for monthly reset.  
  - Outputs to: Reset NOTFOUND id to NULL.  
  - Edge cases: Incorrect variables.

- **Every month**  
  - Type: Schedule Trigger  
  - Role: Triggers monthly reset of NOTFOUND flags.  
  - Outputs to: variables4.  
  - Edge cases: Trigger failures.

---

### 3. Summary Table

| Node Name                      | Node Type           | Functional Role                                     | Input Node(s)                     | Output Node(s)                          | Sticky Note                                                                                                  |
|--------------------------------|---------------------|----------------------------------------------------|----------------------------------|---------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Every hour                     | Schedule Trigger    | Triggers hourly sync check                          |                                  | variables                             | ## Check for any modification in the spotify playlist with snapshot_id<br>### If you want to change the checking interval, make sure to change the trigger AND the wait node |
| variables                     | Set                 | Defines playlist and table variables                | Every hour                      | Get playlist snapshot                 |                                                                                                              |
| Get playlist snapshot          | Spotify             | Gets current Spotify playlist snapshot ID          | variables                       | Wait 1 hour                          |                                                                                                              |
| Wait 1 hour                   | Wait                | Delays for 1 hour before re-checking snapshot      | Get playlist snapshot           | Get playlist snapshot1                |                                                                                                              |
| Get playlist snapshot1         | Spotify             | Gets second snapshot ID for comparison              | Wait 1 hour                    | If different snapshot                |                                                                                                              |
| If different snapshot          | If                  | Checks if Spotify playlist snapshot changed         | Get playlist snapshot1          | Spotify, Get all musics, No Operation |                                                                                                              |
| Spotify                      | Spotify             | Retrieves all tracks from Spotify playlist          | If different snapshot (true)    | Compare Datasets                     |                                                                                                              |
| Get all musics                | Supabase            | Retrieves all music entries from database           | If different snapshot (true)    | Compare Datasets                     |                                                                                                              |
| Compare Datasets              | Compare Datasets    | Compares Spotify tracks with DB entries             | Spotify, Get all musics         | Add music, Update to_delete to true, No Operation | # Spotify-Database Synchronization<br>## Operation:<br>- ## Compares Spotify playlist tracks against database entries<br>- ## Adds missing tracks to database<br>- ## Marks database entries for deletion when removed from Spotify playlist |
| Add music                    | Supabase            | Adds new Spotify tracks to database                  | Compare Datasets                | data                               |                                                                                                              |
| Update to_delete to true      | Supabase            | Marks removed tracks as to_delete = TRUE             | Compare Datasets                |                                   |                                                                                                              |
| No Operation, do nothing      | NoOp                | Placeholder when no changes detected                 | If different snapshot (false)   |                                   |                                                                                                              |
| variables1                   | Set                 | Defines variables for YouTube matching               | Every day at noon + 1mn         | Get all musics not in youtube playlist |                                                                                                              |
| Get all musics not in youtube playlist | Supabase            | Gets tracks needing YouTube video matching           | variables1                     | data                               |                                                                                                              |
| data                         | Set                 | Prepares track data for processing                    | Get all musics not in youtube playlist | Loop Over Items                    |                                                                                                              |
| Loop Over Items              | SplitInBatches      | Processes tracks one by one for YouTube search       | data                          | Search video, Aggregate             | # Match Spotify Tracks to YouTube Videos<br>## This part finds the best YouTube video for a Spotify track using the YouTube Data API v3. It searches with the track title and artist, retrieves the top 5 videos, and selects the first one with a duration within ±10% of the Spotify track length. The matched video is added to a YouTube playlist, and its ID is saved in the database.<br>## Operation:<br>- ## Uses Spotify data (title + artist) for search.<br>- ## Ensures duration accuracy (±10% tolerance).<br>- ## Automates playlist updates and database storage. |
| Search video                 | YouTube             | Searches YouTube videos for each track               | Loop Over Items               | If no result                      |                                                                                                              |
| If no result                 | If                  | Checks if YouTube search returned no results         | Search video                  | Aggregate (true), Loop Over Items (false) |                                                                                                              |
| Aggregate                   | Aggregate           | Aggregates search results                             | If no result (true)           | Set youtube id to NOTFOUND if no matching |                                                                                                              |
| If video duration ~= music duration | If                  | Compares video duration with Spotify track duration  | Get video duration            | Add music to playlist (true), Loop Over Items (false) |                                                                                                              |
| Get video duration           | YouTube             | Retrieves YouTube video duration                      | Loop Over Items               | If video duration ~= music duration |                                                                                                              |
| Add music to playlist        | YouTube             | Adds matched video to YouTube playlist                | If video duration ~= music duration (true) | Add youtube id to row, Discord |                                                                                                              |
| Add youtube id to row        | Supabase            | Updates DB with matched YouTube video ID              | Add music to playlist         | Loop Over Items1, Discord          |                                                                                                              |
| Set youtube id to NOTFOUND if no matching | Supabase            | Marks tracks with no YouTube match as NOTFOUND        | Aggregate                    | Loop Over Items1, Discord1         |                                                                                                              |
| Loop Over Items1             | SplitInBatches      | Processes matched/unmatched tracks for notifications | Add youtube id to row, Set youtube id to NOTFOUND if no matching | data1                              |                                                                                                              |
| data1                       | Set                 | Prepares data for Discord notifications                | Loop Over Items1              | Search video, Discord, Discord1    |                                                                                                              |
| Discord                     | Discord             | Sends notification for added videos                    | Add youtube id to row         |                                   | ## Optional notifications (you can use the chat of your choice)                                              |
| Discord1                    | Discord             | Sends notification for unmatched tracks                | Set youtube id to NOTFOUND if no matching |                                   |                                                                                                              |
| Every day at midnight        | Schedule Trigger    | Triggers daily YouTube playlist maintenance            |                              | variables3                        |                                                                                                              |
| variables3                  | Set                 | Defines variables for maintenance tasks                 | Every day at midnight         | Get all musics that should be in playlist, Get playlist items, Get all musics to be deleted |                                                                                                              |
| Get all musics that should be in playlist | Supabase            | Gets musics with valid YouTube video IDs               | variables3                   | Check for deleted videos          |                                                                                                              |
| Get playlist items           | YouTube             | Gets all items from YouTube playlist                     | variables3                   | Playlist items to be deleted, Check for deleted videos |                                                                                                              |
| Check for deleted videos     | Compare Datasets    | Detects deleted YouTube videos by comparing DB and playlist | Get all musics that should be in playlist, Get playlist items | Set youtube_video_id to null       |                                                                                                              |
| Set youtube_video_id to null | Supabase            | Resets YouTube video ID to null for deleted videos      | Check for deleted videos      |                                   |                                                                                                              |
| Get all musics to be deleted | Supabase            | Gets musics marked for deletion                          | variables3                   | Playlist items to be deleted      |                                                                                                              |
| Playlist items to be deleted | Compare Datasets    | Finds playlist items to delete based on DB flags         | Get playlist items, Get all musics to be deleted | Remove Duplicates, Check for deleted videos |                                                                                                              |
| Remove Duplicates            | Remove Duplicates   | Removes duplicate playlist items to delete               | Playlist items to be deleted  | Remove video from playlist        |                                                                                                              |
| Remove video from playlist   | YouTube             | Deletes playlist items from YouTube playlist              | Remove Duplicates            | Delete music                     |                                                                                                              |
| Delete music                | Supabase            | Deletes music entries marked for deletion                 | Remove video from playlist    |                                   |                                                                                                              |
| Reset NOTFOUND id to NULL    | Supabase            | Resets NOTFOUND flags monthly to retry matching           | variables4                   |                                   |                                                                                                              |
| variables4                  | Set                 | Defines variables for monthly reset                        | Every month                  | Reset NOTFOUND id to NULL         |                                                                                                              |
| Every month                 | Schedule Trigger    | Triggers monthly reset of NOTFOUND flags                   |                              | variables4                       |                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger "Every hour"**  
   - Type: Schedule Trigger  
   - Parameters: Interval set to every hour.

2. **Create Set node "variables"**  
   - Define variables:  
     - spotify_playlist_id = "4fjIxvQt8aQrQZs4XqvsmR"  
     - youtube_playlist_id = "PLjmwnzu1gWRsnW6icKeUyvbaK9-Cs8oom"  
     - supabase_table_name = "musics"

3. **Connect "Every hour" → "variables"**

4. **Create Spotify node "Get playlist snapshot"**  
   - Operation: Get playlist  
   - Playlist ID: `={{ $json.spotify_playlist_id }}`

5. **Connect "variables" → "Get playlist snapshot"**

6. **Create Wait node "Wait 1 hour"**  
   - Unit: hours  
   - Amount: 1

7. **Connect "Get playlist snapshot" → "Wait 1 hour"**

8. **Create Spotify node "Get playlist snapshot1"**  
   - Operation: Get playlist  
   - Playlist ID: `={{ $('variables').item.json.spotify_playlist_id }}`

9. **Connect "Wait 1 hour" → "Get playlist snapshot1"**

10. **Create If node "If different snapshot"**  
    - Condition: Check if `$('Get playlist snapshot').item.json.snapshot_id` NOT EQUALS `$json.snapshot_id` (from Get playlist snapshot1)  
    - True branch: proceed to Spotify tracks and DB fetch  
    - False branch: No operation

11. **Connect "Get playlist snapshot1" → "If different snapshot"**

12. **Create Spotify node "Spotify"**  
    - Operation: Get Tracks  
    - Playlist ID: `={{ $('variables').item.json.spotify_playlist_id }}`  
    - Return All: true

13. **Create Supabase node "Get all musics"**  
    - Operation: Get All  
    - Table: `={{ $('variables').item.json.supabase_table_name }}`  
    - Return All: true

14. **Connect "If different snapshot" True → "Spotify" and "Get all musics"**

15. **Create Compare Datasets node "Compare Datasets"**  
    - Merge by fields: `track.id` (Spotify) and `id` (DB)  
    - Skip fields: title, artists, duration, youtube_video_id, added_at, added_by, is_local, primary_color, video_thumbnail

16. **Connect "Spotify" and "Get all musics" → "Compare Datasets"**

17. **Create Supabase node "Add music"**  
    - Operation: Insert  
    - Table: same as variables  
    - Fields: id, title, artist, duration from Spotify track data

18. **Create Supabase node "Update to_delete to true"**  
    - Operation: Update  
    - Table: same as variables  
    - Filter: id equals removed track id  
    - Set field: to_delete = TRUE

19. **Create NoOp node "No Operation, do nothing"**

20. **Connect "Compare Datasets" outputs:**  
    - Added → "Add music"  
    - Removed → "Update to_delete to true"  
    - Unchanged → "No Operation, do nothing"

21. **Connect "Add music" → "data" (Set node)**  
    - Prepare fields: title, artist, duration, id, supabase_table_name, youtube_playlist_id

22. **Create Supabase node "Get all musics not in youtube playlist"**  
    - Operation: Get All  
    - Filter: youtube_video_id IS NULL AND to_delete IS FALSE  
    - Table: variables.supabase_table_name

23. **Create Set node "variables1"**  
    - Same variables as "variables"

24. **Create Schedule Trigger "Every day at noon + 1mn"**  
    - Triggers daily at 12:01

25. **Connect "Every day at noon + 1mn" → "variables1" → "Get all musics not in youtube playlist" → "data"**

26. **Create SplitInBatches node "Loop Over Items"**  
    - Batch size: 1 (default)

27. **Connect "data" → "Loop Over Items"**

28. **Create YouTube node "Search video"**  
    - Operation: Search videos  
    - Query: `={{ $json.title }} - {{ $json.artist }}`  
    - Limit: 5

29. **Connect "Loop Over Items" → "Search video"**

30. **Create If node "If no result"**  
    - Condition: Check if search results empty

31. **Connect "Search video" → "If no result"**

32. **Create Aggregate node "Aggregate"**  
    - Aggregate all item data

33. **Connect "If no result" True → "Aggregate"**

34. **Create Supabase node "Set youtube id to NOTFOUND if no matching"**  
    - Operation: Update  
    - Table: variables.supabase_table_name  
    - Filter: id equals current track id  
    - Set youtube_video_id = "NOTFOUND"

35. **Connect "Aggregate" → "Set youtube id to NOTFOUND if no matching"**

36. **Create SplitInBatches node "Loop Over Items1"**

37. **Connect "Set youtube id to NOTFOUND if no matching" → "Loop Over Items1"**

38. **Create Set node "data1"**  
    - Prepare fields for notifications

39. **Connect "Loop Over Items1" → "data1"**

40. **Create Discord node "Discord1"**  
    - Sends message for unmatched tracks  
    - Content: `No match for : {{ $('data1').first().json.title }}`  
    - Auth: Webhook

41. **Connect "Set youtube id to NOTFOUND if no matching" → "Discord1"**

42. **Connect "Loop Over Items1" → "Discord1"**

43. **Create YouTube node "Get video duration"**  
    - Operation: Get video details  
    - Video ID: from search results

44. **Connect "If no result" False → "Get video duration"**

45. **Create If node "If video duration ~= music duration"**  
    - Condition: Video duration within -5s to +20s of Spotify track duration

46. **Connect "Get video duration" → "If video duration ~= music duration"**

47. **Create YouTube node "Add music to playlist"**  
    - Operation: Add video to playlist  
    - Playlist ID: variables.youtube_playlist_id  
    - Video ID: from Get video duration

48. **Connect "If video duration ~= music duration" True → "Add music to playlist"**

49. **Create Supabase node "Add youtube id to row"**  
    - Operation: Update  
    - Table: variables.supabase_table_name  
    - Filter: id equals current track id  
    - Set youtube_video_id to added video ID

50. **Connect "Add music to playlist" → "Add youtube id to row"**

51. **Create Discord node "Discord"**  
    - Sends message for added videos  
    - Content: `Added : {{ $json.title }} (https://www.youtube.com/watch?v={{ $json.youtube_video_id }})`  
    - Auth: Webhook

52. **Connect "Add youtube id to row" → "Discord"**

53. **Connect "Add youtube id to row" → "Loop Over Items1"** (to continue processing)

54. **Connect "If video duration ~= music duration" False → "Loop Over Items"** (to check next video)

55. **Create Schedule Trigger "Every day at midnight"**  
    - Triggers daily maintenance

56. **Create Set node "variables3"**  
    - Same variables as before

57. **Connect "Every day at midnight" → "variables3"**

58. **Create Supabase node "Get all musics that should be in playlist"**  
    - Filter: youtube_video_id NOT NULL and NOT "NOTFOUND"

59. **Create YouTube node "Get playlist items"**  
    - Operation: Get all playlist items  
    - Playlist ID: variables.youtube_playlist_id

60. **Connect "variables3" → "Get all musics that should be in playlist" and "Get playlist items"**

61. **Create Compare Datasets node "Check for deleted videos"**  
    - Merge by youtube_video_id (DB) and contentDetails.videoId (YouTube)

62. **Connect "Get all musics that should be in playlist" and "Get playlist items" → "Check for deleted videos"**

63. **Create Supabase node "Set youtube_video_id to null"**  
    - Update DB to set youtube_video_id = null for deleted videos (excluding NOTFOUND)

64. **Connect "Check for deleted videos" → "Set youtube_video_id to null"**

65. **Create Supabase node "Get all musics to be deleted"**  
    - Filter: to_delete = TRUE and youtube_video_id != "NOTFOUND"

66. **Connect "variables3" → "Get all musics to be deleted"**

67. **Create Compare Datasets node "Playlist items to be deleted"**  
    - Merge by snippet.resourceId.videoId (YouTube) and youtube_video_id (DB)

68. **Connect "Get playlist items" and "Get all musics to be deleted" → "Playlist items to be deleted"**

69. **Create Remove Duplicates node "Remove Duplicates"**  
    - Compare by different.youtube_video_id.inputB

70. **Connect "Playlist items to be deleted" → "Remove Duplicates"**

71. **Create YouTube node "Remove video from playlist"**  
    - Operation: Delete playlist item  
    - PlaylistItemId: from Remove Duplicates

72. **Connect "Remove Duplicates" → "Remove video from playlist"**

73. **Create Supabase node "Delete music"**  
    - Delete DB entries matching removed videos and to_delete = TRUE

74. **Connect "Remove video from playlist" → "Delete music"**

75. **Create Schedule Trigger "Every month"**  
    - Triggers monthly reset

76. **Create Set node "variables4"**  
    - Same variables as before

77. **Connect "Every month" → "variables4"**

78. **Create Supabase node "Reset NOTFOUND id to NULL"**  
    - Update DB: set youtube_video_id = null where youtube_video_id = "NOTFOUND"

79. **Connect "variables4" → "Reset NOTFOUND id to NULL"**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                         |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| # Spotify to YouTube Playlist Synchronization<br>## A workflow that maintains a YouTube playlist in sync with a Spotify playlist, featuring smart video matching and persistent synchronization.<br><br>## Key Features<br>- One-way Sync: Spotify playlist → YouTube playlist (additions and deletions)<br>- Continuous Monitoring: Automatic synchronization (every hour by default)<br>- Smart Video Matching: Considers video length and content relevance<br>- Auto-Recovery: Automatically handles deleted YouTube videos<br>- Database Backup: Persistent storage using Supabase<br><br>## Prerequisites<br>1. Supabase project with table `musics` as defined<br>2. Empty YouTube playlist recommended<br>3. Configured credentials for YouTube, Spotify, and Supabase APIs<br>4. Properly set variables nodes<br>5. Activate the workflow! | Workflow description and prerequisites (Sticky Note2)                                                  |
| # Match Spotify Tracks to YouTube Videos<br>This part finds the best YouTube video for a Spotify track using the YouTube Data API v3. It searches with the track title and artist, retrieves the top 5 videos, and selects the first one with a duration within ±10% of the Spotify track length. The matched video is added to a YouTube playlist, and its ID is saved in the database. | Sticky Note explaining YouTube matching logic                                                        |
| # Spotify-Database Synchronization<br>Compares Spotify playlist tracks against database entries, adds missing tracks, and marks removed tracks for deletion.                                                                                                                                                                                                       | Sticky Note3                                                                                          |
| # Daily Force Check<br>Forces daily comparison between Spotify playlist and database state, bypassing playlist modification checks. Useful for initial setup, processing pending tracks, and continuing sync attempts for unmatched tracks.                                                                                                                           | Sticky Note4                                                                                          |
| # Workflow 1: Main Sync Process<br>Change Detection and Video Matching steps summarized.                                                                                                                                                                                                                                                                             | Sticky Note7                                                                                          |
| # Workflow 2: YouTube Maintenance<br>Monitors YouTube playlist for removed videos, flags removed videos for re-search, and handles deletion of marked videos.                                                                                                                                                                                                      | Sticky Note8                                                                                          |
| # Workflow 3: Recovery Process<br>Clears "NOTFOUND" flags periodically to re-search previously unmatched tracks.                                                                                                                                                                                                                                                   | Sticky Note6                                                                                          |
| Optional notifications can be customized to any chat platform by replacing the Discord node or webhook.                                                                                                                                                                                                                                                            | Sticky Note5                                                                                          |

---

This documentation provides a detailed, stepwise understanding of the Spotify to YouTube Playlist Synchronization workflow, enabling reproduction, modification, and troubleshooting by advanced users and AI agents alike.