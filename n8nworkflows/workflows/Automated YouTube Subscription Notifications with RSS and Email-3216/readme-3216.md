Automated YouTube Subscription Notifications with RSS and Email

https://n8nworkflows.xyz/workflows/automated-youtube-subscription-notifications-with-rss-and-email-3216


# Automated YouTube Subscription Notifications with RSS and Email

### 1. Workflow Overview

This workflow automates notifications for new YouTube videos from the channels you are subscribed to by sending email alerts. It targets users who want to avoid manually checking YouTube feeds and prefer receiving timely email notifications for each new video.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger & Subscription Retrieval:** Periodically triggers the workflow and fetches the list of YouTube subscriptions using the YouTube Data API.
- **1.2 Subscription Filtering:** Filters subscriptions to keep only channels with new videos and optionally excludes specified channels.
- **1.3 Video Retrieval via RSS:** For each filtered channel, retrieves the latest 15 videos using the channel’s RSS feed to avoid API quota overuse.
- **1.4 Video Filtering & Details Enrichment:** Filters videos published since the last workflow run, fetches detailed video data (thumbnail, duration), and filters out shorts.
- **1.5 Email Notification:** Sends a formatted email for each new video with a clickable thumbnail linking to the video.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Subscription Retrieval

- **Overview:**  
  This block initiates the workflow on a schedule (default every hour) and retrieves the authenticated user's YouTube subscriptions via the YouTube Data v3 API.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Get my subscriptions  
  - Check for errors  
  - If the HTTP request failed, throw the error  
  - Split out subscriptions to process individually

- **Node Details:**

  - **Schedule Trigger**  
    - *Type:* Schedule Trigger  
    - *Role:* Starts workflow execution every hour at minute 47 by default.  
    - *Configuration:* Interval set to 1 hour, trigger at minute 47.  
    - *Inputs:* None (trigger node)  
    - *Outputs:* Connects to "Get my subscriptions"  
    - *Edge Cases:* If scheduling is changed improperly, workflow may not trigger as expected.

  - **Get my subscriptions**  
    - *Type:* HTTP Request  
    - *Role:* Calls YouTube Data API to fetch the authenticated user's subscriptions (max 50 per request).  
    - *Configuration:*  
      - URL: `https://www.googleapis.com/youtube/v3/subscriptions`  
      - Query parameters: `mine=true`, `part=snippet,contentDetails`, `maxResults=50`  
      - Pagination enabled to handle multiple pages using `nextPageToken`.  
      - Authentication: YouTube OAuth2 credentials required.  
    - *Inputs:* From Schedule Trigger  
    - *Outputs:* To "Check for errors"  
    - *Edge Cases:* API quota limits, OAuth token expiration, network errors.

  - **Check for errors**  
    - *Type:* If  
    - *Role:* Checks if the API response contains an error object.  
    - *Configuration:* Condition checks if `$json.error` exists.  
    - *Inputs:* From "Get my subscriptions"  
    - *Outputs:*  
      - If error: to "If the HTTP request failed, throw the error"  
      - Else: to "Split out subscriptions to process individually"  
    - *Edge Cases:* Misinterpretation of error object, false negatives.

  - **If the HTTP request failed, throw the error**  
    - *Type:* Stop and Error  
    - *Role:* Stops workflow execution and throws a formatted error message.  
    - *Configuration:* Error message includes status code and message from the API error.  
    - *Inputs:* From "Check for errors" (error branch)  
    - *Outputs:* None (terminates workflow)  
    - *Edge Cases:* None; ensures graceful failure reporting.

  - **Split out subscriptions to process individually**  
    - *Type:* Split Out  
    - *Role:* Splits the array of subscription items into individual items for downstream processing.  
    - *Configuration:* Field to split out: `items`  
    - *Inputs:* From "Check for errors" (success branch)  
    - *Outputs:* To "Only keep channels with unwatched videos"  
    - *Edge Cases:* Empty subscriptions list results in no downstream processing.

---

#### 1.2 Subscription Filtering

- **Overview:**  
  Filters the list of subscriptions to only those channels with new videos and optionally excludes specific channels by ID.

- **Nodes Involved:**  
  - Only keep channels with unwatched videos  
  - Filter out channels

- **Node Details:**

  - **Only keep channels with unwatched videos**  
    - *Type:* Filter  
    - *Role:* Keeps only subscriptions where `contentDetails.newItemCount` > 0, indicating new videos.  
    - *Configuration:* Condition: `$json.contentDetails.newItemCount > 0`  
    - *Inputs:* From "Split out subscriptions to process individually"  
    - *Outputs:* To "Filter out channels"  
    - *Edge Cases:* May miss videos if YouTube's newItemCount is inaccurate or delayed.

  - **Filter out channels** (Optional)  
    - *Type:* Filter  
    - *Role:* Excludes channels whose IDs are in a manually maintained blocklist.  
    - *Configuration:* Condition: Channel ID not in array `["exampleChannelId1", "exampleChannelId2"]`  
    - *Inputs:* From "Only keep channels with unwatched videos"  
    - *Outputs:* To "Get latest 15 videos of each channel"  
    - *Edge Cases:* Requires manual update of channel IDs; misconfiguration may exclude desired channels.

---

#### 1.3 Video Retrieval via RSS

- **Overview:**  
  Retrieves the latest 15 videos for each filtered channel using the channel's RSS feed instead of the YouTube API to conserve quota.

- **Nodes Involved:**  
  - Get latest 15 videos of each channel

- **Node Details:**

  - **Get latest 15 videos of each channel**  
    - *Type:* RSS Feed Read  
    - *Role:* Fetches the RSS feed for a channel to get the 15 latest videos.  
    - *Configuration:*  
      - URL template: `https://www.youtube.com/feeds/videos.xml?channel_id={{ $json.snippet.resourceId.channelId }}`  
    - *Inputs:* From "Filter out channels"  
    - *Outputs:* To "Keep only videos published since last run"  
    - *Edge Cases:* RSS feed availability issues, malformed feed, network errors.

---

#### 1.4 Video Filtering & Details Enrichment

- **Overview:**  
  Filters videos published since the last workflow run, enriches video data by fetching details from YouTube API, and filters out shorts (videos shorter than 61 seconds or missing duration).

- **Nodes Involved:**  
  - Keep only videos published since last run  
  - Get video details  
  - Filter out shorts

- **Node Details:**

  - **Keep only videos published since last run**  
    - *Type:* Filter  
    - *Role:* Keeps videos published after the last workflow execution time.  
    - *Configuration:*  
      - Condition compares video `pubDate` to the timestamp of the last run minus the schedule interval (default 1 hour).  
      - Uses expressions to dynamically calculate the cutoff time.  
    - *Inputs:* From "Get latest 15 videos of each channel"  
    - *Outputs:* To "Get video details"  
    - *Edge Cases:* If workflow was stopped for a long time, videos published during downtime are missed.

  - **Get video details**  
    - *Type:* YouTube node  
    - *Role:* Retrieves detailed video information including thumbnails and duration.  
    - *Configuration:*  
      - Operation: Get video by ID  
      - Video ID extracted from RSS feed item ID by removing prefix `yt:video:`  
      - Parts requested: `contentDetails`, `snippet`, `id`  
      - Credentials: YouTube OAuth2  
    - *Inputs:* From "Keep only videos published since last run"  
    - *Outputs:* To "Filter out shorts"  
    - *Edge Cases:* API quota limits, invalid video IDs, token expiration.

  - **Filter out shorts**  
    - *Type:* If  
    - *Role:* Filters out videos that are shorts (duration ≤ 61 seconds) or missing duration (to exclude some live broadcasts).  
    - *Configuration:*  
      - Condition: Either duration does not exist OR duration in seconds > 61  
      - Uses ISO 8601 duration parsing to seconds.  
    - *Inputs:* From "Get video details"  
    - *Outputs:*  
      - True branch: To "Send an email for each new video"  
      - False branch: Discards shorts  
    - *Edge Cases:* Some live broadcasts without duration may be included or excluded incorrectly.

---

#### 1.5 Email Notification

- **Overview:**  
  Sends an email notification for each new video passing all filters, including a clickable thumbnail linking to the video.

- **Nodes Involved:**  
  - Send an email for each new video

- **Node Details:**

  - **Send an email for each new video**  
    - *Type:* Email Send  
    - *Role:* Sends a formatted HTML email with video title and thumbnail linking to the YouTube video.  
    - *Configuration:*  
      - Subject: Channel title from video snippet  
      - HTML body: Includes centered video title and an image with the highest resolution thumbnail available, clickable linking to the video URL.  
      - From and To email addresses configured with SMTP credentials.  
      - Attribution disabled.  
    - *Inputs:* From "Filter out shorts" (true branch)  
    - *Outputs:* None (end node)  
    - *Edge Cases:* SMTP authentication errors, invalid email addresses, email formatting issues.

---

### 3. Summary Table

| Node Name                         | Node Type           | Functional Role                                   | Input Node(s)                        | Output Node(s)                          | Sticky Note                                                                                              |
|----------------------------------|---------------------|-------------------------------------------------|------------------------------------|---------------------------------------|--------------------------------------------------------------------------------------------------------|
| Schedule Trigger                 | Schedule Trigger    | Initiates workflow every hour                    | None                               | Get my subscriptions                  | Default frequency: every hour. Changing it here is enough to adjust check frequency.                    |
| Get my subscriptions            | HTTP Request        | Fetches YouTube subscriptions via API           | Schedule Trigger                   | Check for errors                      | Get my subscriptions from the YouTube Data v3 API. Quota cost ~1 per 50 subscriptions per run.          |
| Check for errors                | If                  | Checks for errors in API response                 | Get my subscriptions              | If the HTTP request failed, throw error / Split out subscriptions to process individually |                                                                                                        |
| If the HTTP request failed, throw the error | Stop and Error      | Stops workflow on API error                       | Check for errors (error branch)   | None                                  |                                                                                                        |
| Split out subscriptions to process individually | Split Out           | Splits subscription list into individual items  | Check for errors (success branch) | Only keep channels with unwatched videos |                                                                                                        |
| Only keep channels with unwatched videos | Filter              | Keeps channels with new videos                    | Split out subscriptions           | Filter out channels                   | It's not a perfect indicator but helps reduce channels to process.                                      |
| Filter out channels             | Filter              | Excludes specified channels by ID (optional)    | Only keep channels with unwatched videos | Get latest 15 videos of each channel | Manually filter out channels. To find channel ID: description → Share channel → Copy channel ID.        |
| Get latest 15 videos of each channel | RSS Feed Read       | Retrieves latest 15 videos via RSS feed          | Filter out channels               | Keep only videos published since last run | Get the 15 latest videos of each channel with RSS to save API quota.                                    |
| Keep only videos published since last run | Filter              | Filters videos published after last run          | Get latest 15 videos              | Get video details                    | Dynamically uses Schedule Trigger's last execution time to filter videos.                               |
| Get video details               | YouTube             | Fetches video details (thumbnail, duration)      | Keep only videos published since last run | Filter out shorts                   | Call YouTube API for thumbnails and duration to filter shorts.                                         |
| Filter out shorts               | If                  | Filters out videos shorter than 61 seconds       | Get video details                 | Send an email for each new video (true) | Shorts are excluded by default. Some live broadcasts without duration are filtered out here.            |
| Send an email for each new video | Email Send          | Sends notification email with video info         | Filter out shorts (true branch)  | None                                | Configure your email here. Click thumbnail in email to watch video.                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Set interval to 1 hour, trigger at minute 47 (default).  
   - This node starts the workflow periodically.

2. **Add an HTTP Request node named "Get my subscriptions":**  
   - URL: `https://www.googleapis.com/youtube/v3/subscriptions`  
   - Query parameters:  
     - `mine` = `true`  
     - `part` = `snippet,contentDetails`  
     - `maxResults` = `50`  
   - Enable pagination with `pageToken` parameter, complete when `nextPageToken` is absent.  
   - Authentication: Select YouTube OAuth2 API credentials.  
   - Connect Schedule Trigger output to this node.

3. **Add an If node named "Check for errors":**  
   - Condition: Check if `$json.error` exists.  
   - Connect "Get my subscriptions" output to this node.

4. **Add a Stop and Error node named "If the HTTP request failed, throw the error":**  
   - Error message:  
     ```
     Status code: {{ $json.error.code }}
     Message: {{ $json.error.message }}
     ```  
   - Connect "Check for errors" error output to this node.

5. **Add a Split Out node named "Split out subscriptions to process individually":**  
   - Field to split out: `items`  
   - Connect "Check for errors" success output to this node.

6. **Add a Filter node named "Only keep channels with unwatched videos":**  
   - Condition: `$json.contentDetails.newItemCount > 0`  
   - Connect "Split out subscriptions to process individually" output to this node.

7. **Add a Filter node named "Filter out channels" (optional):**  
   - Condition: Channel ID NOT in array of blocked channel IDs (e.g., `["exampleChannelId1", "exampleChannelId2"]`)  
   - Use expression to compare `$json.snippet.resourceId.channelId`  
   - Connect "Only keep channels with unwatched videos" output to this node.

8. **Add an RSS Feed Read node named "Get latest 15 videos of each channel":**  
   - URL: `https://www.youtube.com/feeds/videos.xml?channel_id={{ $json.snippet.resourceId.channelId }}`  
   - Connect "Filter out channels" output to this node.

9. **Add a Filter node named "Keep only videos published since last run":**  
   - Condition: Video `pubDate` is after the last run timestamp minus the schedule interval.  
   - Use expression referencing `$('Schedule Trigger').item.json.timestamp` and subtract interval.  
   - Connect "Get latest 15 videos of each channel" output to this node.

10. **Add a YouTube node named "Get video details":**  
    - Operation: Get video by ID  
    - Video ID: Extract from RSS feed item ID by removing prefix `yt:video:` (expression: `={{ $json.id.replace("yt:video:", "") }}`)  
    - Parts: `contentDetails`, `snippet`, `id`  
    - Credentials: YouTube OAuth2 API  
    - Connect "Keep only videos published since last run" output to this node.

11. **Add an If node named "Filter out shorts":**  
    - Condition: Either duration does not exist OR duration in seconds > 61  
    - Use expression to parse ISO 8601 duration to seconds, e.g., `Duration.fromISO($json.contentDetails.duration).as('seconds') > 61`  
    - Connect "Get video details" output to this node.

12. **Add an Email Send node named "Send an email for each new video":**  
    - Subject: `={{ $json.snippet.channelTitle }}`  
    - HTML Body:  
      ```html
      <h1 style="text-align: center;">{{ $json.snippet.title }}</h1>
      <a href="https://www.youtube.com/watch?v={{ $json.id }}">
        <img src="{{ $json.snippet.thumbnails[Object.keys($json.snippet.thumbnails)[Object.keys($json.snippet.thumbnails).length - 1]].url }}" alt="Watch on YouTube" style="width:100%; height:auto; max-width:640px; display:block; margin: 10px auto;">
      </a>
      ```  
    - From Email: Set to an email address authorized by your SMTP credentials.  
    - To Email: Set to your desired recipient email address.  
    - Disable attribution.  
    - Credentials: SMTP credentials configured.  
    - Connect "Filter out shorts" true output to this node.

13. **Connect "Filter out shorts" false output to nowhere (discard shorts).**

14. **Set up credentials:**  
    - Configure YouTube OAuth2 API credentials with appropriate scopes for subscriptions and video details.  
    - Configure SMTP credentials for sending emails.

15. **Optional adjustments:**  
    - Modify Schedule Trigger interval as desired.  
    - Update "Filter out channels" node with channel IDs to exclude.  
    - Remove "Filter out shorts" node if you want to include shorts.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                      | Context or Link                                                                                 |
|---------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Default frequency: every hour. Changing it in the Schedule Trigger node is sufficient to adjust check frequency.                                  | Sticky Note near Schedule Trigger node                                                         |
| YouTube Data API quota cost is approximately 1 per 50 subscriptions per run, well within the default 10,000/day quota.                           | Sticky Note near "Get my subscriptions" node                                                  |
| Using RSS feeds for latest videos avoids high quota costs associated with YouTube API search requests.                                            | Sticky Note near "Get latest 15 videos of each channel" node                                   |
| Video thumbnails and duration are fetched from YouTube API to enable email formatting and shorts filtering.                                       | Sticky Note near "Get video details" node                                                     |
| To find a channel's ID for filtering, go to the channel's description, click "Share channel," then "Copy channel ID."                             | Sticky Note near "Filter out channels" node                                                   |
| Email notifications include a clickable thumbnail linking directly to the video on YouTube.                                                       | Sticky Note near "Send an email for each new video" node                                      |
| If the n8n instance is offline for a period, videos published during downtime will not trigger email notifications due to timestamp filtering.    | Caveat mentioned in workflow description                                                      |
| Workflow created with n8n version 1.84.0                                                                                                         | Workflow metadata                                                                            |

---

This documentation provides a complete and detailed reference for understanding, reproducing, and modifying the "Automated YouTube Subscription Notifications with RSS and Email" workflow. It covers all nodes, logic blocks, configuration details, and potential failure points to ensure robust operation and easy maintenance.