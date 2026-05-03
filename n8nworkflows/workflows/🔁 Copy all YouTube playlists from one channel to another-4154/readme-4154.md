🔁 Copy all YouTube playlists from one channel to another

https://n8nworkflows.xyz/workflows/---copy-all-youtube-playlists-from-one-channel-to-another-4154


# 🔁 Copy all YouTube playlists from one channel to another

### 1. Workflow Overview

This workflow automates the copying of all user-created YouTube playlists from one channel (origin) to another channel (target). It systematically retrieves all playlists from the source channel, extracts their videos while filtering out private videos, creates corresponding playlists on the target channel, and populates them with the same videos.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Manual trigger to start the workflow.
- **1.2 Retrieve and Format Playlists:** Fetch playlists from origin channel and format their data.
- **1.3 Loop Over Playlists and Retrieve Videos:** Iterate over each playlist to fetch its items (videos).
- **1.4 Filter and Prepare Videos:** Remove private videos and prepare data structures.
- **1.5 Create Playlists in Target Channel:** Create new playlists on the target channel matching origin playlists.
- **1.6 Add Videos to Target Playlists:** Add videos to the newly created playlists, handling potential failures gracefully.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** This block starts the workflow manually.
- **Nodes Involved:**  
  - When clicking ‘Test workflow’
- **Node Details:**

  | Node Name                        | Details                                                                                                                                            |
  |---------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------|
  | When clicking ‘Test workflow’    | - Type: Manual Trigger<br>- Role: Initiates workflow execution on demand.<br>- Config: No parameters.<br>- Output: Triggers next node.<br>- Inputs: None.<br>- Outputs: Connects to "Get all playlists from origin channel".<br>- Failure Modes: None expected as it is manual.<br>- Version: 1. |

#### 2.2 Retrieve and Format Playlists

- **Overview:** Fetches all playlists from the origin YouTube channel and formats them to extract essential fields.
- **Nodes Involved:**  
  - Get all playlists from origin channel  
  - Format fields properly  
  - Loop over playlists  
- **Node Details:**

  | Node Name                     | Details                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
  |-------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
  | Get all playlists from origin channel | - Type: YouTube node (playlist resource)<br>- Role: Retrieves all playlists from the origin channel.<br>- Config: Set to return all playlists without filters.<br>- Credentials: Uses YouTube OAuth2 for origin channel.<br>- Output: List of playlists with full metadata.<br>- Inputs: From manual trigger.<br>- Outputs: To "Format fields properly".<br>- Failure Modes: API quota exceeded, network issues, credential auth failures.<br>- Retry: Enabled with 5s wait between tries.<br>- Version: 1. |
  | Format fields properly          | - Type: Set node<br>- Role: Extracts and formats only playlist id and title.<br>- Config: Assigns 'id' from `$json.id` and 'title' from `$json.snippet.title`.<br>- Inputs: From "Get all playlists from origin channel".<br>- Outputs: To "Loop over playlists".<br>- Failure Modes: Expression errors if fields missing.<br>- Version: 3.4.                                                                                                                                                        |
  | Loop over playlists            | - Type: SplitInBatches<br>- Role: Processes playlists one at a time to avoid API overload.<br>- Config: Defaults, no batch size explicitly set (implies default batch size 1).<br>- Inputs: From "Format fields properly".<br>- Outputs: To "Get playlist items from original playlist" and "Create new playlist in the target channel".<br>- Failure Modes: None specific, but large number of playlists may slow execution.<br>- Version: 3.                                                                                       |

#### 2.3 Loop Over Playlists and Retrieve Videos

- **Overview:** For each playlist, fetch all its video items along with status details.
- **Nodes Involved:**  
  - Get playlist items from original playlist  
  - Remove private items  
  - Keep only the video id  
- **Node Details:**

  | Node Name                      | Details                                                                                                                                                                                                                                                                                                                                                              |
  |--------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
  | Get playlist items from original playlist | - Type: YouTube node (playlistItem resource)<br>- Role: Retrieves all videos in the current playlist.<br>- Config: Return all items, fields: contentDetails and status, playlistId dynamically set to current playlist id.<br>- Credentials: Origin channel YouTube OAuth2.<br>- Inputs: From "Loop over playlists".<br>- Outputs: To "Remove private items".<br>- Failure Modes: API quota limits, playlistId invalid, network issues.<br>- Retry: Enabled, 5s wait.<br>- Version: 1 |
  | Remove private items            | - Type: Filter node<br>- Role: Filters out videos marked as private (`status.privacyStatus != "private"`).<br>- Config: Condition on `$json.status.privacyStatus` not equal to "private".<br>- Inputs: From "Get playlist items from original playlist".<br>- Outputs: To "Keep only the video id".<br>- Failure Modes: If status missing, expression may fail.<br>- Version: 2.2.                                                                                   |
  | Keep only the video id          | - Type: Set node<br>- Role: Extracts video id from contentDetails for further processing.<br>- Config: Sets `video_id` = `$json.contentDetails.videoId`.<br>- Inputs: From "Remove private items".<br>- Outputs: To "Join the playlist and video id in the same item".<br>- Failure Modes: If contentDetails or videoId missing, expression fails.<br>- Version: 3.4.                                                                                     |

#### 2.4 Create Playlists in Target Channel and Combine Data

- **Overview:** Creates a new playlist in the target channel with the same title as the origin playlist, prepares data to associate videos with new playlists.
- **Nodes Involved:**  
  - Create new playlist in the target channel  
  - Keep only the playlist id  
  - Keep only the playlist id (from origin for merging)  
  - Join the playlist and video id in the same item  
- **Node Details:**

  | Node Name                      | Details                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
  |--------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
  | Create new playlist in the target channel | - Type: YouTube node (playlist resource, create operation)<br>- Role: Creates a new playlist on the target channel, using origin playlist title.<br>- Config: `title` set dynamically from `$json.title`.<br>- Credentials: Target channel YouTube OAuth2.<br>- Inputs: From "Loop over playlists".<br>- Outputs: To "Keep only the playlist id" (for new playlist).<br>- Failure Modes: API quota exceeded, invalid credential, title conflicts.<br>- Retry: Disabled.<br>- Version: 1.           |
  | Keep only the playlist id (target) | - Type: Set node<br>- Role: Extracts only the new playlist id from creation response.<br>- Config: Assigns `playlist_id` = `$json.id`.<br>- Inputs: From "Create new playlist in the target channel".<br>- Outputs: To "Join the playlist and video id in the same item" (as one side).<br>- Failure Modes: Missing id in response.<br>- Version: 3.4.                                                                                                                                                  |
  | Keep only the playlist id (origin) | - Type: Set node<br>- Role: Extracts original playlist id from playlist metadata.<br>- Config: Assigns `playlist_id` = `$json.id`.<br>- Inputs: From "Loop over playlists".<br>- Outputs: To "Join the playlist and video id in the same item" (other side).<br>- Failure Modes: Missing id.<br>- Version: 3.4.                                                                                                                                                                                         |
  | Join the playlist and video id in the same item | - Type: Merge node (combine mode)<br>- Role: Combines videos (`video_id`) and playlist ids (`playlist_id`) into a unified data structure for target playlist additions.<br>- Config: Combine mode = combineAll.<br>- Inputs: From "Keep only the playlist id" (origin) and "Keep only the video id".<br>- Outputs: To "Add items to target playlist".<br>- Failure Modes: Data mismatch or empty inputs.<br>- Version: 3.1.                                                                                          |

#### 2.5 Add Videos to Target Playlists

- **Overview:** Adds each video to the corresponding newly created playlist on the target channel. Errors during addition do not stop the workflow.
- **Nodes Involved:**  
  - Add items to target playlist
- **Node Details:**

  | Node Name                    | Details                                                                                                                                                                                                                                                                                                                                                       |
  |------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
  | Add items to target playlist  | - Type: YouTube node (playlistItem resource)<br>- Role: Adds videos to the new playlist on the target channel.<br>- Config: `videoId` set from `$json.video_id`, `playlistId` set from `$json.playlist_id`.<br>- Credentials: Target channel YouTube OAuth2.<br>- Inputs: From "Join the playlist and video id in the same item".<br>- Outputs: To "Loop over playlists" (to continue processing).<br>- Failure Modes: May fail if video is unavailable, already added, quota exceeded.<br>- onError: Continue execution.<br>- Retry: Enabled.<br>- Version: 1. |

---

### 3. Summary Table

| Node Name                          | Node Type              | Functional Role                                 | Input Node(s)                          | Output Node(s)                             | Sticky Note                                                                                         |
|-----------------------------------|------------------------|------------------------------------------------|--------------------------------------|--------------------------------------------|---------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’      | Manual Trigger         | Starts the workflow                             | None                                 | Get all playlists from origin channel       |                                                                                                   |
| Get all playlists from origin channel | YouTube                | Retrieves all playlists from origin channel    | When clicking ‘Test workflow’         | Format fields properly                       | 👆👆👆 Add your origin channel credential here                                                      |
| Format fields properly             | Set                    | Extracts playlist id and title                  | Get all playlists from origin channel | Loop over playlists                         |                                                                                                   |
| Loop over playlists               | SplitInBatches          | Processes playlists one by one                  | Format fields properly                 | Get playlist items from original playlist, Create new playlist in the target channel |                                                                                                   |
| Get playlist items from original playlist | YouTube                | Retrieves videos in each playlist                | Loop over playlists                   | Remove private items                        | 👆👆👆 Add your origin channel credential here                                                      |
| Remove private items              | Filter                  | Filters out private videos                       | Get playlist items from original playlist | Keep only the video id                   |                                                                                                   |
| Keep only the video id            | Set                     | Extracts video id from playlist items           | Remove private items                  | Join the playlist and video id in the same item |                                                                                                   |
| Create new playlist in the target channel | YouTube                | Creates new playlists in target channel         | Loop over playlists                   | Keep only the playlist id (target)          | 👆👆👆 Add your target channel credential here                                                      |
| Keep only the playlist id (target) | Set                     | Extracts new playlist id                         | Create new playlist in the target channel | Join the playlist and video id in the same item | 👆👆👆 Add your target channel credential here                                                      |
| Keep only the playlist id (origin) | Set                     | Extracts original playlist id                    | Loop over playlists                   | Join the playlist and video id in the same item |                                                                                                   |
| Join the playlist and video id in the same item | Merge                   | Combines video and playlist ids for addition    | Keep only the playlist id (origin), Keep only the video id | Add items to target playlist               |                                                                                                   |
| Add items to target playlist      | YouTube                | Adds videos to new playlists                     | Join the playlist and video id in the same item | Loop over playlists                         | 👆👆👆 Add your target channel credential here                                                      |
| Sticky Note                      | Sticky Note             | Workflow description and setup instructions     | None                                 | None                                       | ## 🔁 COPY ALL YOUTUBE PLAYLISTS FROM ONE CHANNEL TO ANOTHER<br>### ℹ️ ABOUT THIS WORKFLOW<br>This workflow will copy all user-created playlists from one YouTube channel to another.<br>### 🛠️ SETUP<br>1. Create two YouTube credentials: one for each channel.<br>2. Add the credentials to the appropriate nodes, as indicated.<br>3. Click "Test workflow" to run.<br>### ⚠️ KNOWN LIMITATIONS<br>- Does not copy automatic playlists.<br>- Watch API quota usage. |
| Sticky Note1                    | Sticky Note             | Add origin channel credential reminder           | None                                 | None                                       | ## 👆👆👆<br>## ADD YOUR ORIGIN CHANNEL CREDENTIAL HERE                                          |
| Sticky Note2                    | Sticky Note             | Add origin channel credential reminder           | None                                 | None                                       | ## 👆👆👆<br>## ADD YOUR ORIGIN CHANNEL CREDENTIAL HERE                                          |
| Sticky Note3                    | Sticky Note             | Add target channel credential reminder           | None                                 | None                                       | ## 👆👆👆<br>## ADD YOUR TARGET CHANNEL CREDENTIAL HERE                                          |
| Sticky Note4                    | Sticky Note             | Add target channel credential reminder           | None                                 | None                                       | ## ADD YOUR TARGET CHANNEL CREDENTIAL HERE<br>## 👇👇👇                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name it: `When clicking ‘Test workflow’`  
   - No special parameters.

2. **Add a YouTube node to get playlists from origin channel**  
   - Name: `Get all playlists from origin channel`  
   - Resource: `playlist`  
   - Operation: default (list)  
   - Return All: true  
   - Credentials: Set OAuth2 credentials for the origin YouTube channel.  
   - Connect input from `When clicking ‘Test workflow’`.

3. **Add a Set node to format playlist fields**  
   - Name: `Format fields properly`  
   - Assignments:  
     - `id` = `{{$json["id"]}}`  
     - `title` = `{{$json["snippet"]["title"]}}`  
   - Connect input from `Get all playlists from origin channel`.

4. **Add a SplitInBatches node to loop over playlists**  
   - Name: `Loop over playlists`  
   - Use default batch size (1) to process one playlist at a time.  
   - Connect input from `Format fields properly`.

5. **Add a YouTube node to get playlist items**  
   - Name: `Get playlist items from original playlist`  
   - Resource: `playlistItem`  
   - Operation: `getAll`  
   - Return All: true  
   - Part: `contentDetails`, `status`  
   - Playlist ID: Set expression to `{{$json["id"]}}` (from current playlist)  
   - Credentials: Use origin channel OAuth2 credentials.  
   - Connect input from second output of `Loop over playlists`.

6. **Add a Filter node to remove private videos**  
   - Name: `Remove private items`  
   - Condition: `$json.status.privacyStatus` `!=` `private` (case sensitive, strict)  
   - Connect input from `Get playlist items from original playlist`.

7. **Add a Set node to keep only video IDs**  
   - Name: `Keep only the video id`  
   - Assignment: `video_id` = `{{$json["contentDetails"]["videoId"]}}`  
   - Connect input from `Remove private items`.

8. **Add a YouTube node to create playlists in target channel**  
   - Name: `Create new playlist in the target channel`  
   - Resource: `playlist`  
   - Operation: `create`  
   - Title: Set expression `{{$json["title"]}}` (from current playlist)  
   - Credentials: Use target channel OAuth2 credentials.  
   - Connect input from first output of `Loop over playlists`.

9. **Add a Set node to keep only playlist id (target)**  
   - Name: `Keep only the playlist id` (target)  
   - Assignment: `playlist_id` = `{{$json["id"]}}`  
   - Connect input from `Create new playlist in the target channel`.

10. **Add a Set node to keep only playlist id (origin)**  
    - Name: `Keep only the playlist id` (origin)  
    - Assignment: `playlist_id` = `{{$json["id"]}}`  
    - Connect input from `Loop over playlists`.

11. **Add a Merge node to combine playlist and video ids**  
    - Name: `Join the playlist and video id in the same item`  
    - Mode: `combine`  
    - Combine By: `combineAll`  
    - Connect inputs from:  
      - Left input: `Keep only the playlist id` (origin) node  
      - Right input: `Keep only the video id` node

12. **Add a YouTube node to add videos to target playlist**  
    - Name: `Add items to target playlist`  
    - Resource: `playlistItem`  
    - Parameters:  
      - `playlistId` = `{{$json["playlist_id"]}}`  
      - `videoId` = `{{$json["video_id"]}}`  
    - Credentials: Target channel OAuth2 credentials  
    - On Error: Continue (to prevent workflow stop on failures)  
    - Retry: Enabled with 5 seconds wait  
    - Connect input from `Join the playlist and video id in the same item`.

13. **Connect output of `Add items to target playlist` back to `Loop over playlists`**  
    - To continue processing next playlist batch.

14. **Add Sticky Notes** as reminders for:  
    - Adding origin channel credentials to relevant nodes (`Get all playlists from origin channel`, `Get playlist items from original playlist`).  
    - Adding target channel credentials to relevant nodes (`Create new playlist in the target channel`, `Add items to target playlist`).  
    - Workflow description including limitations and setup instructions.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| This workflow does not copy YouTube automatic playlists like Watch Later or Liked Videos due to API limitations. | Workflow Sticky Note |
| Be mindful of your YouTube API quota, as copying many playlists/videos may exhaust it. Request quota increase if necessary. | Workflow Sticky Note |
| Setup requires two YouTube OAuth2 credentials: one for origin channel, one for target channel. | Workflow Sticky Note |
| Official YouTube API docs for playlists and playlistItems can help understand resource limitations and structures: https://developers.google.com/youtube/v3/docs/ | External Resource |
| Retry and error handling settings in YouTube nodes help mitigate transient network or quota issues. | Best Practice Note |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.