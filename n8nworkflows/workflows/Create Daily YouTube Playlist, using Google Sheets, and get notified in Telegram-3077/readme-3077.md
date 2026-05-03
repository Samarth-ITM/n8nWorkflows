Create Daily YouTube Playlist, using Google Sheets, and get notified in Telegram

https://n8nworkflows.xyz/workflows/create-daily-youtube-playlist--using-google-sheets--and-get-notified-in-telegram-3077


# Create Daily YouTube Playlist, using Google Sheets, and get notified in Telegram

### 1. Workflow Overview

This workflow automates the creation of a daily YouTube playlist composed of the latest videos from a curated list of YouTube channels, managed via Google Sheets. It targets users who want to streamline their video consumption by automatically aggregating recent uploads from selected channels into a single playlist, while also deleting the previous day’s playlist to avoid clutter. Additionally, it sends a Telegram notification upon completion.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger & Playlist Management**: Initiates the daily process, deletes the previous day’s playlist, and creates a new playlist for the current day.
- **1.2 Channel Data Retrieval**: Reads the list of YouTube channels from Google Sheets and fetches their recent videos published within the last 24 hours.
- **1.3 Video Filtering & Playlist Population**: Filters out upcoming live broadcasts and adds the valid videos to the newly created playlist.
- **1.4 Playlist ID Management**: Saves the new playlist ID back to Google Sheets for reference and deletion on the next run.
- **1.5 Notification**: Sends a Telegram message to notify the user that the playlist has been updated.
- **1.6 Channel List Creation (Separate Workflow)**: A sub-workflow to populate channel details in the Google Sheet based on channel usernames; intended to be run manually when updating the channel list.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Playlist Management

- **Overview:**  
  This block triggers the workflow daily at 07:15 AM (Asia/Manila timezone), deletes the previous day’s playlist (except on the first run), and creates a new playlist named with the current date in YYMMDD format followed by "AI News".

- **Nodes Involved:**  
  - `0715 Trigger`  
  - `Google Sheets` (to fetch yesterday’s playlist ID)  
  - `Delete Old Playlist`  
  - `Create Playlist`  
  - `Save Playlist ID`

- **Node Details:**

  - **0715 Trigger**  
    - *Type:* Schedule Trigger  
    - *Role:* Starts the workflow daily at 07:15 AM.  
    - *Config:* Timezone set to Asia/Manila, triggers once per day at 7:15.  
    - *Inputs:* None  
    - *Outputs:* Connects to `Create Playlist` and `Google Sheets`.  
    - *Edge Cases:* If the workflow is run for the first time, the deletion of the old playlist will fail because no previous playlist ID exists; user must disconnect the deletion leg initially.

  - **Google Sheets (Fetch Yesterday’s Playlist ID)**  
    - *Type:* Google Sheets (Read with filter)  
    - *Role:* Retrieves the playlist ID for the "AI News" playlist group from the Google Sheet to identify the playlist to delete.  
    - *Config:* Filters rows where "Playlist Group" equals "AI News".  
    - *Inputs:* From `0715 Trigger`  
    - *Outputs:* Connects to `Delete Old Playlist`.  
    - *Edge Cases:* If no matching row is found, deletion node will receive no valid playlist ID, causing failure.

  - **Delete Old Playlist**  
    - *Type:* YouTube (Playlist Delete)  
    - *Role:* Deletes the playlist identified by the fetched playlist ID.  
    - *Config:* Uses playlist ID from the Google Sheets node output (`{{$json['New Playlist ID']}}`).  
    - *Inputs:* From `Google Sheets`  
    - *Outputs:* None (end of deletion branch)  
    - *Edge Cases:* Playlist ID missing or invalid causes API error; first run requires disconnecting this node.

  - **Create Playlist**  
    - *Type:* YouTube (Playlist Create)  
    - *Role:* Creates a new playlist titled with the current date in YYMMDD format plus "AI News".  
    - *Config:* Title expression: `{{$today.format('yyLLdd')}} AI News`  
    - *Inputs:* From `0715 Trigger`  
    - *Outputs:* Connects to `Save Playlist ID`.  
    - *Edge Cases:* API quota limits or auth errors possible.

  - **Save Playlist ID**  
    - *Type:* Google Sheets (Update)  
    - *Role:* Updates the Google Sheet with the new playlist ID under the "AI News" playlist group for future reference.  
    - *Config:* Updates row where "Playlist Group" = "AI News" with new playlist ID from `Create Playlist`.  
    - *Inputs:* From `Create Playlist`  
    - *Outputs:* Connects to `Read Channel Names`.  
    - *Edge Cases:* Sheet access or write permission errors.

---

#### 1.2 Channel Data Retrieval

- **Overview:**  
  Reads the list of YouTube channels from a Google Sheet and fetches the latest videos published in the last 24 hours for each channel.

- **Nodes Involved:**  
  - `Read Channel Names`  
  - `Get Videos`  
  - `Split Out`

- **Node Details:**

  - **Read Channel Names**  
    - *Type:* Google Sheets (Read)  
    - *Role:* Reads the list of channels from the specified Google Sheet tab "AI News Channels".  
    - *Config:* Reads all rows from the sheet with columns including Channel User Name, Channel Name, Channel Link, Channel ID.  
    - *Inputs:* From `Save Playlist ID`  
    - *Outputs:* Connects to `Get Videos`.  
    - *Edge Cases:* Sheet access errors, empty rows, or missing Channel IDs.

  - **Get Videos**  
    - *Type:* HTTP Request  
    - *Role:* Calls YouTube Data API v3 to search for videos published in the last 24 hours on each channel.  
    - *Config:*  
      - URL: `https://www.googleapis.com/youtube/v3/search`  
      - Query parameters:  
        - `part=snippet`  
        - `publishedAfter={{ $now.minus(1, 'day') }}` (ISO 8601 date 24 hours ago)  
        - `maxResults=5` (fetches up to 5 videos per channel)  
        - `channel_id={{ $('Read Channel Names').item.json['Channel Id'] }}`  
        - `order=date` (most recent first)  
        - `key=AddYourAPIKeyHere` (YouTube API key placeholder)  
    - *Inputs:* From `Read Channel Names` (iterates per channel)  
    - *Outputs:* Connects to `Split Out`.  
    - *Edge Cases:* API quota exceeded, invalid API key, missing channel ID, network errors.

  - **Split Out**  
    - *Type:* Split Out  
    - *Role:* Splits the array of videos returned by the API into individual items for processing.  
    - *Config:* Splits on the `items` field from the API response.  
    - *Inputs:* From `Get Videos`  
    - *Outputs:* Connects to `Filter Out Upcoming`.  
    - *Edge Cases:* Empty video list results in no output items.

---

#### 1.3 Video Filtering & Playlist Population

- **Overview:**  
  Filters out upcoming live broadcasts and adds the remaining videos to the newly created playlist.

- **Nodes Involved:**  
  - `Filter Out Upcoming`  
  - `YouTube` (Add video to playlist)

- **Node Details:**

  - **Filter Out Upcoming**  
    - *Type:* Filter  
    - *Role:* Excludes videos where `snippet.liveBroadcastContent` equals "upcoming" (i.e., scheduled live streams).  
    - *Config:* Condition: `snippet.liveBroadcastContent != "upcoming"`  
    - *Inputs:* From `Split Out`  
    - *Outputs:* Connects to `YouTube` node for adding videos.  
    - *Edge Cases:* Videos without `liveBroadcastContent` field may pass through; ensure field presence.

  - **YouTube (Add video to playlist)**  
    - *Type:* YouTube (Playlist Item Create)  
    - *Role:* Adds each filtered video to the playlist created earlier.  
    - *Config:*  
      - `videoId={{ $('Split Out').item.json.id.videoId }}`  
      - `playlistId={{ $('Create Playlist').item.json.id }}`  
    - *Inputs:* From `Filter Out Upcoming`  
    - *Outputs:* Connects to `Telegram` notification.  
    - *Edge Cases:* API quota limits, invalid video or playlist IDs, auth errors.

---

#### 1.4 Playlist ID Management

- **Overview:**  
  Saves the newly created playlist ID back to Google Sheets to enable deletion on the next run.

- **Nodes Involved:**  
  - `Save Playlist ID` (already described in 1.1)

- **Node Details:**  
  See 1.1 for details.

---

#### 1.5 Notification

- **Overview:**  
  Sends a Telegram message to notify the user that the AI News playlist has been updated.

- **Nodes Involved:**  
  - `Telegram`

- **Node Details:**

  - **Telegram**  
    - *Type:* Telegram Node  
    - *Role:* Sends a text notification message.  
    - *Config:* Message text: "Your AI News YT Playlist has been updated."  
    - *Inputs:* From `YouTube` (Add video to playlist) node  
    - *Outputs:* None (end of workflow)  
    - *Credentials:* Requires Telegram Bot API credentials.  
    - *Edge Cases:* Telegram API errors, invalid bot token, network issues.

---

#### 1.6 Channel List Creation (Separate Workflow)

- **Overview:**  
  This sub-workflow is designed to be run manually to populate the Google Sheet with channel details based on the channel usernames (e.g., "@CorporalStock"). It fetches channel info from YouTube API and updates the sheet accordingly.

- **Nodes Involved:**  
  - `When clicking ‘Test workflow’` (Manual Trigger)  
  - `Read Channel Names1` (Google Sheets Read)  
  - `Get Channels` (YouTube API Search for channel)  
  - `Update Google Sheet` (Update channel info)  
  - Sticky Notes: `Sticky Note6`, `Sticky Note7` (instructions and explanations)

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - *Type:* Manual Trigger  
    - *Role:* Starts the sub-workflow manually.  
    - *Inputs:* None  
    - *Outputs:* Connects to `Read Channel Names1`.

  - **Read Channel Names1**  
    - *Type:* Google Sheets (Read)  
    - *Role:* Reads the list of channel usernames from the Google Sheet.  
    - *Config:* Reads from the same sheet and tab as main workflow.  
    - *Inputs:* From manual trigger  
    - *Outputs:* Connects to `Get Channels`.

  - **Get Channels**  
    - *Type:* HTTP Request  
    - *Role:* Calls YouTube API v3 search endpoint to find channel details by username.  
    - *Config:*  
      - URL: `https://www.googleapis.com/youtube/v3/search`  
      - Query parameters:  
        - `q={{ $json['Channel User Name'] }}`  
        - `type=channel`  
        - `maxResults=1`  
        - `part=snippet`  
        - `key=AIzaSyARU7upVG5hzoaMHIMaBEXjcYtayo8vPJ4` (API key placeholder)  
    - *Inputs:* From `Read Channel Names1`  
    - *Outputs:* Connects to `Update Google Sheet`.  
    - *Edge Cases:* API quota limits, invalid usernames, no results.

  - **Update Google Sheet**  
    - *Type:* Google Sheets (Update)  
    - *Role:* Updates the Google Sheet row with channel ID, name, and link based on API response.  
    - *Config:* Updates row matching `row_number` with:  
      - `Link=https://www.youtube.com/{{ Channel User Name }}`  
      - `Channel Id` from API response  
      - `Channel Name` from API response  
    - *Inputs:* From `Get Channels`  
    - *Outputs:* None (end of sub-workflow)  
    - *Edge Cases:* Sheet write permission errors, missing API data.

---

### 3. Summary Table

| Node Name               | Node Type             | Functional Role                                | Input Node(s)            | Output Node(s)          | Sticky Note                                                                                                         |
|-------------------------|-----------------------|-----------------------------------------------|--------------------------|-------------------------|---------------------------------------------------------------------------------------------------------------------|
| 0715 Trigger            | Schedule Trigger      | Starts workflow daily at 07:15 AM              | None                     | Create Playlist, Google Sheets |                                                                                                                     |
| Google Sheets           | Google Sheets (Read)  | Fetch yesterday’s playlist ID                   | 0715 Trigger             | Delete Old Playlist      |                                                                                                                     |
| Delete Old Playlist     | YouTube Playlist      | Deletes yesterday’s playlist                    | Google Sheets            | None                    | ## Delete Yesterday's Playlist: This section deletes the playlist created yesterday. (do not include this on your first run; or, your workflow will stop) |
| Create Playlist         | YouTube Playlist      | Creates today’s playlist                         | 0715 Trigger             | Save Playlist ID         | ## Create New AI News Playlist: This section creates today's playlist.                                              |
| Save Playlist ID        | Google Sheets (Update)| Saves new playlist ID for future deletion       | Create Playlist          | Read Channel Names       |                                                                                                                     |
| Read Channel Names      | Google Sheets (Read)  | Reads list of channels from Google Sheet        | Save Playlist ID          | Get Videos               | ## Getting the Videos: This section grabs the videos.                                                               |
| Get Videos              | HTTP Request          | Fetches recent videos from each channel         | Read Channel Names       | Split Out                |                                                                                                                     |
| Split Out               | Split Out             | Splits video list into individual items         | Get Videos               | Filter Out Upcoming      |                                                                                                                     |
| Filter Out Upcoming     | Filter                | Filters out upcoming live broadcasts             | Split Out                | YouTube (Add video)      |                                                                                                                     |
| YouTube (Add video)     | YouTube PlaylistItem  | Adds videos to the created playlist              | Filter Out Upcoming      | Telegram                 | ## Add Videos to Playlist: This section adds videos to the playlist created today.                                  |
| Telegram                | Telegram              | Sends notification of playlist update            | YouTube (Add video)      | None                    | ## Notification of Completion (optional)                                                                            |
| When clicking ‘Test workflow’ | Manual Trigger    | Starts channel list creation sub-workflow        | None                     | Read Channel Names1      |                                                                                                                     |
| Read Channel Names1     | Google Sheets (Read)  | Reads channel usernames for sub-workflow          | When clicking ‘Test workflow’ | Get Channels          | ## Create your Channel List: This section needs to be put into its own workflow: this workflow gathers information needed to gather videos for the playlist. This workflow only needs to be run when a new channel name is added to the Google Sheet. |
| Get Channels            | HTTP Request          | Fetches channel details by username               | Read Channel Names1      | Update Google Sheet      |                                                                                                                     |
| Update Google Sheet     | Google Sheets (Update)| Updates channel info in Google Sheet               | Get Channels             | None                    |                                                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets:**
   - Create a Google Sheet with two tabs:
     - "AI News Channels" with columns: `Channel User Name`, `Channel Name`, `Channel Link`, `Channel ID`.
     - "PlaylistId" with columns: `Playlist Group`, `New Playlist ID`.
   - Populate "AI News Channels" with channel usernames (e.g., "@CorporalStock") in the `Channel User Name` column.

2. **Set up Credentials:**
   - Configure Google Sheets OAuth2 credentials with access to the above sheet.
   - Configure YouTube OAuth2 credentials with permissions to manage playlists.
   - Configure Telegram Bot API credentials for sending messages.
   - Obtain a YouTube Data API key for HTTP Request nodes.

3. **Create Nodes:**

   - **Schedule Trigger Node:**
     - Type: Schedule Trigger
     - Set to trigger daily at 07:15 AM (Asia/Manila timezone).

   - **Google Sheets (Read) Node:**
     - Reads from "PlaylistId" tab.
     - Filter rows where `Playlist Group` = "AI News".
     - Used to fetch yesterday’s playlist ID.

   - **YouTube Playlist Delete Node:**
     - Operation: Delete playlist.
     - Playlist ID: Use expression to get `New Playlist ID` from Google Sheets node.
     - Connect input from Google Sheets node.

   - **YouTube Playlist Create Node:**
     - Operation: Create playlist.
     - Title: Expression `{{$today.format('yyLLdd')}} AI News`.
     - Connect input from Schedule Trigger node.

   - **Google Sheets (Update) Node:**
     - Updates "PlaylistId" tab.
     - Match row where `Playlist Group` = "AI News".
     - Update `New Playlist ID` with ID from Create Playlist node.
     - Connect input from Create Playlist node.

   - **Google Sheets (Read) Node:**
     - Reads from "AI News Channels" tab.
     - Connect input from Google Sheets Update node.

   - **HTTP Request Node (Get Videos):**
     - URL: `https://www.googleapis.com/youtube/v3/search`
     - Query parameters:
       - `part=snippet`
       - `publishedAfter={{ $now.minus(1, 'day') }}`
       - `maxResults=5`
       - `channel_id={{ $json['Channel Id'] }}`
       - `order=date`
       - `key=YourYouTubeAPIKey`
     - Connect input from Read Channel Names node.
     - Set to run for each item (per channel).

   - **Split Out Node:**
     - Field to split out: `items`.
     - Connect input from Get Videos node.

   - **Filter Node:**
     - Condition: `snippet.liveBroadcastContent` NOT EQUAL to "upcoming".
     - Connect input from Split Out node.

   - **YouTube Playlist Item Create Node:**
     - Operation: Add video to playlist.
     - Video ID: `{{$json.id.videoId}}`
     - Playlist ID: `{{ $('Create Playlist').item.json.id }}`
     - Connect input from Filter node.

   - **Telegram Node:**
     - Send message: "Your AI News YT Playlist has been updated."
     - Connect input from YouTube Playlist Item Create node.

4. **Sub-Workflow for Channel List Creation:**

   - **Manual Trigger Node:** To start the sub-workflow manually.

   - **Google Sheets (Read) Node:** Reads "AI News Channels" tab for channel usernames.

   - **HTTP Request Node (Get Channels):**
     - URL: `https://www.googleapis.com/youtube/v3/search`
     - Query parameters:
       - `q={{ $json['Channel User Name'] }}`
       - `type=channel`
       - `maxResults=1`
       - `part=snippet`
       - `key=YourYouTubeAPIKey`
     - Connect input from Google Sheets Read node.

   - **Google Sheets (Update) Node:**
     - Updates "AI News Channels" tab.
     - Match row by `row_number`.
     - Update columns:
       - `Channel Name` from API response.
       - `Channel Id` from API response.
       - `Link` constructed as `https://www.youtube.com/{{ Channel User Name }}`.
     - Connect input from Get Channels node.

5. **Important Setup Notes:**
   - On the first run of the main workflow, disconnect the `Delete Old Playlist` node to avoid errors due to missing previous playlist.
   - Replace all placeholder API keys with valid keys.
   - Ensure all OAuth2 credentials are authorized and tested.
   - The date format in playlist title ensures unique daily playlists.
   - The Google Sheet must be accessible and shared with the OAuth2 service account or user.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                      |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow is designed for cord-cutters or users who want to automate the aggregation of recent YouTube videos from selected channels into a daily playlist, simplifying video consumption. It supports channels whether subscribed or not.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Workflow Purpose                                                                                   |
| The playlist title includes the date in YYMMDD format to avoid conflicts with existing playlists.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Playlist Naming Convention                                                                          |
| The workflow sends a Telegram notification upon successful playlist update, providing user feedback.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Telegram Notification                                                                              |
| The sub-workflow "Create your Channel List" must be run manually whenever new channels are added by their usernames (e.g., "@ChannelName") to populate channel details in the Google Sheet.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Channel List Management Instructions                                                              |
| First-time run caution: Disconnect the "Delete Yesterday's Playlist" node to prevent workflow failure due to missing playlist ID.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | First Run Setup Note                                                                              |
| YouTube Data API quota limits and authentication errors are common failure points; monitor API usage and refresh credentials as needed.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | API Quota and Auth Considerations                                                                 |
| Google Sheets must have proper sharing and permission settings to allow read/write operations by the workflow's OAuth2 credentials.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Google Sheets Permissions                                                                         |
| Telegram Bot must be configured with correct API token and chat ID to receive notifications.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Telegram Bot Setup                                                                               |
| For more information on YouTube Data API, visit: https://developers.google.com/youtube/v3/docs/search/list                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | YouTube API Documentation                                                                         |
| For Google Sheets API details, see: https://developers.google.com/sheets/api/reference/rest                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Google Sheets API Documentation                                                                  |
| Telegram Bot API reference: https://core.telegram.org/bots/api                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Telegram API Documentation                                                                       |

---

This document provides a complete, structured reference for understanding, reproducing, and maintaining the "Create Daily YouTube Playlist" workflow using n8n, Google Sheets, YouTube API, and Telegram notifications.