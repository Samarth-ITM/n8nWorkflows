Monitor Favorite YouTube Channels Through RSS feeds and Receive Notifications

https://n8nworkflows.xyz/workflows/monitor-favorite-youtube-channels-through-rss-feeds-and-receive-notifications-3003


# Monitor Favorite YouTube Channels Through RSS feeds and Receive Notifications

### 1. Workflow Overview

This workflow automates monitoring of favorite YouTube channels via their RSS feeds and sends notifications about new videos through Telegram and email. It supports both user-submitted and default channel lists, fetches detailed video data from the YouTube Data API, and formats notifications with responsive HTML templates. The workflow is structured into the following logical blocks:

- **1.1 Input Reception & Channel List Preparation**  
  Handles user input via form submission or uses default YouTube channel IDs. Prepares and splits channel IDs for processing.

- **1.2 RSS Feed Creation & Video Retrieval**  
  Constructs RSS feed URLs for each channel, reads the latest videos (up to 15 per channel), and labels videos as recent based on a configurable time window (default 3 days).

- **1.3 Video Filtering & Data Enrichment**  
  Filters only recent videos, fetches detailed video information from the YouTube Data API, and prepares data for notifications.

- **1.4 Notification Preparation & Sending**  
  Prepares and sends notifications via Telegram (with thumbnails and links) and email in two formats: individual emails per video and a single digest email listing all new videos.

- **1.5 Auxiliary & Configuration Nodes**  
  Includes workflow variables, scheduling triggers, and sticky notes for documentation and setup guidance.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Channel List Preparation

**Overview:**  
This block receives YouTube channel IDs either from a user form submission or defaults, then prepares the list for further processing.

**Nodes Involved:**  
- On form submission  
- Default YouTube Channel Ids  
- YouTube Channel Ids  
- Get Channel Ids  
- Create List of Channel Ids  

**Node Details:**

- **On form submission**  
  - Type: Form Trigger  
  - Role: Entry point for user input of YouTube channel IDs via a textarea form field.  
  - Configuration: Form titled "RSS Feed for YouTube Channels" with a textarea for channel IDs array input.  
  - Input: HTTP webhook triggered on form submit.  
  - Output: JSON containing user input.  
  - Edge Cases: Empty or malformed input; no fallback if user input is invalid.  
  - Notes: Response mode set to last node.

- **Default YouTube Channel Ids**  
  - Type: Set  
  - Role: Provides a fallback list of default YouTube channel IDs if user input is empty.  
  - Configuration: Sets an array of three default channel IDs.  
  - Input: From "On form submission" or "YouTube Channel Ids".  
  - Output: JSON with "Default YouTube Channel Ids" array.  
  - Edge Cases: None significant; static data.

- **YouTube Channel Ids**  
  - Type: Set  
  - Role: Initializes an empty string field "YouTube Channel Ids" to prepare for input or default assignment.  
  - Input: From schedule trigger or manual trigger.  
  - Output: JSON with empty string field.  
  - Edge Cases: None.

- **Get Channel Ids**  
  - Type: Set  
  - Role: Chooses between user-submitted channel IDs or defaults based on presence of user input.  
  - Configuration: Uses expression to check if user input array length > 0; if yes, uses it; else uses default list.  
  - Input: From "Default YouTube Channel Ids".  
  - Output: JSON with "ids" array containing final channel IDs.  
  - Edge Cases: Empty arrays, malformed input.

- **Create List of Channel Ids**  
  - Type: SplitOut  
  - Role: Splits the array of channel IDs into individual items, each with a field "youtube_channel_id".  
  - Input: From "Get Channel Ids".  
  - Output: Multiple items, each representing one channel ID.  
  - Edge Cases: Empty input array results in no output items.

---

#### 2.2 RSS Feed Creation & Video Retrieval

**Overview:**  
This block constructs RSS feed URLs for each channel and reads the latest videos from these feeds.

**Nodes Involved:**  
- Create RSS Feed URLs1  
- RSS Read - Max 15 Latest Videos per Channel  
- Label New Videos  
- Get New Videos  

**Node Details:**

- **Create RSS Feed URLs1**  
  - Type: Set  
  - Role: Creates the YouTube RSS feed URL for each channel ID.  
  - Configuration: Uses expression to build URL: `https://www.youtube.com/feeds/videos.xml?channel_id={{ $json.youtube_channel_id }}`  
  - Input: From "Create List of Channel Ids".  
  - Output: JSON with "rss_feed_url" string.  
  - Edge Cases: Invalid or empty channel IDs produce invalid URLs.

- **RSS Read - Max 15 Latest Videos per Channel**  
  - Type: RSS Feed Read  
  - Role: Reads up to 15 latest videos from the constructed RSS feed URL.  
  - Configuration: URL set dynamically from "rss_feed_url". SSL verification enabled.  
  - Input: From "Create RSS Feed URLs1".  
  - Output: Video items with metadata including publication date and ID.  
  - Edge Cases: Network errors, invalid RSS feeds, empty feeds.

- **Label New Videos**  
  - Type: Code  
  - Role: Processes video items to label those published within the last 3 days as recent.  
  - Configuration: JavaScript code uses Luxon DateTime to compare publication dates with a 3-day threshold. Extracts video ID from RSS ID string.  
  - Input: From "RSS Read - Max 15 Latest Videos per Channel".  
  - Output: Video items with added boolean property "recent_videos".  
  - Edge Cases: Missing or malformed date or ID fields; code logs errors and returns original item on failure.

- **Get New Videos**  
  - Type: Filter  
  - Role: Filters only videos marked as recent (recent_videos == true).  
  - Configuration: Condition checks boolean true on "recent_videos" property.  
  - Input: From "Label New Videos".  
  - Output: Only recent video items.  
  - Edge Cases: No recent videos results in empty output.

---

#### 2.3 Video Filtering & Data Enrichment

**Overview:**  
This block enriches filtered recent videos by fetching detailed video information from the YouTube Data API.

**Nodes Involved:**  
- Workflow Variables  
- Merge  
- Create YouTube API URL  
- Get YouTube Video Details  
- Prepare YouTube Data  

**Node Details:**

- **Workflow Variables**  
  - Type: Set  
  - Role: Sets variables needed for API calls, including Google API key and extracts VIDEO_ID from video item ID.  
  - Configuration:  
    - GOOGLE_API_KEY: placeholder string "[Add-Your-Google-API-Key-Here]" to be replaced by user.  
    - VIDEO_ID: extracted by splitting the RSS ID string.  
  - Input: From "Get New Videos".  
  - Output: JSON with GOOGLE_API_KEY and VIDEO_ID.  
  - Edge Cases: Missing or invalid API key; malformed video ID.

- **Create YouTube API URL**  
  - Type: Code  
  - Role: Constructs YouTube Data API URL to fetch detailed video info using VIDEO_ID and GOOGLE_API_KEY.  
  - Configuration: JavaScript code builds URL with parts: snippet, contentDetails, status, statistics, player, topicDetails.  
  - Input: From "Workflow Variables".  
  - Output: JSON with "youtubeUrl" string.  
  - Edge Cases: Missing VIDEO_ID or GOOGLE_API_KEY throws error.

- **Get YouTube Video Details**  
  - Type: HTTP Request  
  - Role: Calls YouTube Data API using constructed URL to retrieve detailed video metadata.  
  - Configuration: URL set dynamically from "youtubeUrl".  
  - Input: From "Create YouTube API URL".  
  - Output: JSON with detailed video data.  
  - Edge Cases: API quota exceeded, invalid API key, network errors.

- **Prepare YouTube Data**  
  - Type: Set  
  - Role: Extracts and restructures key video data fields for notification use.  
  - Configuration: Sets fields such as video ID, title, description, embed HTML, and standard thumbnail object from API response.  
  - Input: From "Get YouTube Video Details".  
  - Output: JSON with simplified video data structure.  
  - Edge Cases: Missing fields in API response.

- **Merge**  
  - Type: Merge  
  - Role: Combines enriched video data with original video items for further processing.  
  - Configuration: Mode "combine" with join mode "enrichInput1" by matching IDs.  
  - Input: From "Prepare YouTube Data" and "Workflow Variables".  
  - Output: Merged video data items.  
  - Edge Cases: Mismatched IDs cause incomplete merges.

---

#### 2.4 Notification Preparation & Sending

**Overview:**  
This block prepares notification messages and sends them via Telegram and email in two formats: multiple individual emails and a single digest email.

**Nodes Involved:**  
- Prepare For Telegram Message  
- Telegram  
- Create Email per Video  
- Multiple Emails  
- One List Object  
- Create One Email for All Videos  
- Single Email  

**Node Details:**

- **Prepare For Telegram Message**  
  - Type: Set  
  - Role: Prepares the first video item’s data for Telegram notification, including ID, title, thumbnail URL, and link.  
  - Configuration: Assigns these fields explicitly from merged video data.  
  - Input: From "Merge".  
  - Output: JSON with fields for Telegram node.  
  - Edge Cases: Empty video list results in no data.

- **Telegram**  
  - Type: Telegram  
  - Role: Sends a photo message to Telegram chat with video thumbnail and caption including title and link.  
  - Configuration:  
    - Operation: sendPhoto  
    - Chat ID: from environment variable TELEGRAM_CHAT_ID  
    - Photo URL: from thumbnail URL field  
    - Caption: includes video title and link  
  - Input: From "Prepare For Telegram Message".  
  - Output: Telegram API response.  
  - Edge Cases: Invalid chat ID, network errors, invalid photo URL.

- **Create Email per Video**  
  - Type: Chain LLM (OpenAI)  
  - Role: Generates responsive HTML email cards for each video individually using OpenAI GPT-4o-mini model.  
  - Configuration:  
    - Input JSON: list of video items  
    - Prompt: detailed instructions for HTML email card design optimized for email clients  
  - Input: From "Merge".  
  - Output: HTML email content per video.  
  - Edge Cases: API errors, rate limits, malformed input.

- **Multiple Emails**  
  - Type: Gmail  
  - Role: Sends individual emails for each new video using generated HTML content.  
  - Configuration:  
    - Recipient: static email "joe@example.com" (to be customized)  
    - Subject: "Latest YouTube Videos from Your Favorite Channels"  
    - Message: HTML from "Create Email per Video"  
    - Credentials: Gmail OAuth2 configured  
  - Input: From "Create Email per Video".  
  - Output: Gmail API response.  
  - Edge Cases: Authentication errors, quota limits.

- **One List Object**  
  - Type: Aggregate  
  - Role: Aggregates all video items into a single JSON object for digest email.  
  - Input: From "Merge".  
  - Output: Single aggregated JSON object.  
  - Edge Cases: Empty input results in empty aggregation.

- **Create One Email for All Videos**  
  - Type: Chain LLM (OpenAI)  
  - Role: Generates a single responsive HTML email listing all new videos using OpenAI GPT-4o-mini.  
  - Configuration:  
    - Input JSON: aggregated video data  
    - Prompt: similar to individual email but for multiple videos in one email  
  - Input: From "One List Object".  
  - Output: HTML email content for digest.  
  - Edge Cases: API errors, rate limits.

- **Single Email**  
  - Type: Gmail  
  - Role: Sends the single digest email with all new videos.  
  - Configuration:  
    - Recipient: static email "joe@example.com"  
    - Subject: "Latest YouTube Videos from Your Favorite Channels"  
    - Message: HTML from "Create One Email for All Videos"  
    - Credentials: Gmail OAuth2 configured  
  - Input: From "Create One Email for All Videos".  
  - Output: Gmail API response.  
  - Edge Cases: Authentication errors, quota limits.

---

#### 2.5 Auxiliary & Configuration Nodes

**Overview:**  
Nodes providing scheduling, manual triggers, and documentation notes.

**Nodes Involved:**  
- Every Day  
- When clicking ‘Test workflow’ (disabled)  
- Sticky Notes (various)  

**Node Details:**

- **Every Day**  
  - Type: Schedule Trigger  
  - Role: Triggers workflow execution daily to check for new videos.  
  - Configuration: Interval set to daily.  
  - Output: Triggers "YouTube Channel Ids" node.  
  - Edge Cases: None.

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger (disabled)  
  - Role: Allows manual triggering for testing.  
  - Edge Cases: Disabled by default.

- **Sticky Notes**  
  - Type: Sticky Note  
  - Role: Provide inline documentation, instructions, and links for users.  
  - Content includes:  
    - Workflow description and key features  
    - API documentation links  
    - Setup instructions  
    - Node group explanations  
  - Edge Cases: None.

---

### 3. Summary Table

| Node Name                          | Node Type                     | Functional Role                                     | Input Node(s)                     | Output Node(s)                                | Sticky Note                                                                                          |
|-----------------------------------|-------------------------------|----------------------------------------------------|----------------------------------|-----------------------------------------------|----------------------------------------------------------------------------------------------------|
| On form submission                | Form Trigger                  | Receives user input of YouTube channel IDs         | (Webhook trigger)                | Default YouTube Channel Ids                    |                                                                                                    |
| Default YouTube Channel Ids       | Set                           | Provides default YouTube channel IDs                | On form submission, YouTube Channel Ids | Get Channel Ids                              |                                                                                                    |
| YouTube Channel Ids               | Set                           | Initializes empty YouTube Channel Ids field         | Every Day                       | Default YouTube Channel Ids                    |                                                                                                    |
| Get Channel Ids                  | Set                           | Chooses user input or default channel IDs           | Default YouTube Channel Ids      | Create List of Channel Ids                      |                                                                                                    |
| Create List of Channel Ids        | SplitOut                      | Splits channel ID array into individual items       | Get Channel Ids                  | Create RSS Feed URLs1                           |                                                                                                    |
| Create RSS Feed URLs1             | Set                           | Builds RSS feed URLs for each channel               | Create List of Channel Ids       | RSS Read - Max 15 Latest Videos per Channel    |                                                                                                    |
| RSS Read - Max 15 Latest Videos per Channel | RSS Feed Read               | Reads latest videos from RSS feeds                   | Create RSS Feed URLs1            | Label New Videos                               |                                                                                                    |
| Label New Videos                  | Code                          | Labels videos as recent if published within 3 days | RSS Read - Max 15 Latest Videos  | Get New Videos                                 |                                                                                                    |
| Get New Videos                   | Filter                        | Filters only recent videos                            | Label New Videos                | Workflow Variables, Merge                       |                                                                                                    |
| Workflow Variables               | Set                           | Sets API key and extracts VIDEO_ID                   | Get New Videos                  | Create YouTube API URL                          | 💡 YouTube Variables https://cloud.google.com/docs/get-started/access-apis                          |
| Create YouTube API URL           | Code                          | Constructs YouTube Data API URL                       | Workflow Variables             | Get YouTube Video Details                       | ## YouTube Video Details https://developers.google.com/youtube/v3/docs https://www.googleapis.com/youtube/v3/videos |
| Get YouTube Video Details        | HTTP Request                  | Fetches detailed video info from YouTube API         | Create YouTube API URL          | Prepare YouTube Data                            |                                                                                                    |
| Prepare YouTube Data             | Set                           | Extracts key video fields for notifications          | Get YouTube Video Details       | Merge                                          |                                                                                                    |
| Merge                           | Merge                         | Combines enriched video data with original items     | Prepare YouTube Data, Workflow Variables | Create Email per Video, One List Object, Prepare For Telegram Message |                                                                                                    |
| Prepare For Telegram Message     | Set                           | Prepares data for Telegram notification               | Merge                          | Telegram                                       | ## Send Latest Videos as Telegram Message                                                          |
| Telegram                        | Telegram                      | Sends video thumbnail and link via Telegram          | Prepare For Telegram Message    |                                               |                                                                                                    |
| Create Email per Video           | Chain LLM (OpenAI)            | Generates individual HTML email cards per video      | Merge                          | Multiple Emails                                | ## Send Email for Each Latest Video (Multiple Emails)                                              |
| Multiple Emails                 | Gmail                         | Sends individual emails for each new video           | Create Email per Video          |                                               |                                                                                                    |
| One List Object                 | Aggregate                     | Aggregates all videos into one object for digest     | Merge                          | Create One Email for All Videos                 |                                                                                                    |
| Create One Email for All Videos | Chain LLM (OpenAI)            | Generates single HTML email listing all new videos   | One List Object                | Single Email                                   | ## Send Email with a List of Latest Videos (One email only)                                        |
| Single Email                   | Gmail                         | Sends digest email with all new videos                | Create One Email for All Videos |                                               |                                                                                                    |
| Every Day                      | Schedule Trigger              | Triggers workflow daily                               |                                  | YouTube Channel Ids                            | ## ⌚Set Your Schedule                                                                              |
| When clicking ‘Test workflow’   | Manual Trigger (disabled)     | Manual trigger for testing                            |                                  | YouTube Channel Ids1                           |                                                                                                    |
| YouTube Channel Ids1            | Set                           | Provides default channel IDs for manual test         | When clicking ‘Test workflow’   | Create List of Channel Ids1                     |                                                                                                    |
| Create List of Channel Ids1     | SplitOut                     | Splits default channel IDs for manual test           | YouTube Channel Ids1            | Create RSS Feed URLs                            |                                                                                                    |
| Create RSS Feed URLs            | Set                          | Builds RSS feed URLs for manual test                  | Create List of Channel Ids1     | RSS Read - Max 15 Latest Videos per Channel1   |                                                                                                    |
| RSS Read - Max 15 Latest Videos per Channel1 | RSS Feed Read              | Reads latest videos for manual test                    | Create RSS Feed URLs            | Label New Videos                               |                                                                                                    |
| Sticky Notes                   | Sticky Note                  | Documentation and instructions                        |                                  |                                               | Multiple sticky notes provide detailed workflow description, API links, and usage tips.            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node ("Every Day")**  
   - Set to trigger daily at your preferred time.

2. **Create a Set node ("YouTube Channel Ids")**  
   - Initialize an empty string field "YouTube Channel Ids".

3. **Create a Set node ("Default YouTube Channel Ids")**  
   - Assign an array field "Default YouTube Channel Ids" with your default channel IDs.

4. **Create a Set node ("Get Channel Ids")**  
   - Use an expression to set "ids" array:  
     `={{ $json["YouTube Channel Ids"].length > 0 ? $json["YouTube Channel Ids"] : $json["Default YouTube Channel Ids"] }}`

5. **Create a SplitOut node ("Create List of Channel Ids")**  
   - Split the "ids" array into individual items with field name "youtube_channel_id".

6. **Create a Set node ("Create RSS Feed URLs1")**  
   - For each item, set "rss_feed_url" to:  
     `https://www.youtube.com/feeds/videos.xml?channel_id={{ $json.youtube_channel_id }}`

7. **Create an RSS Feed Read node ("RSS Read - Max 15 Latest Videos per Channel")**  
   - Set URL to `={{ $json.rss_feed_url }}`  
   - Limit to max 15 items per feed.

8. **Create a Code node ("Label New Videos")**  
   - Paste JavaScript code that:  
     - Extracts video ID from RSS ID string  
     - Compares publication date to current date minus 3 days (using Luxon or native Date)  
     - Adds boolean "recent_videos" flag  
   - Handle errors gracefully.

9. **Create a Filter node ("Get New Videos")**  
   - Filter items where `recent_videos == true`.

10. **Create a Set node ("Workflow Variables")**  
    - Set "GOOGLE_API_KEY" to your YouTube Data API key (replace placeholder).  
    - Extract "VIDEO_ID" from video ID string.

11. **Create a Code node ("Create YouTube API URL")**  
    - JavaScript code to build YouTube API URL using VIDEO_ID and GOOGLE_API_KEY, requesting parts: snippet, contentDetails, status, statistics, player, topicDetails.

12. **Create an HTTP Request node ("Get YouTube Video Details")**  
    - Use URL from previous node to fetch video details.

13. **Create a Set node ("Prepare YouTube Data")**  
    - Extract and assign fields: id, snippet.title, snippet.description, player.embedHtml, snippet.thumbnails.standard.

14. **Create a Merge node ("Merge")**  
    - Combine enriched video data with original items by matching IDs.

15. **Create a Set node ("Prepare For Telegram Message")**  
    - Prepare first video item’s id, title, thumbnail URL, and link for Telegram.

16. **Create a Telegram node ("Telegram")**  
    - Operation: sendPhoto  
    - Chat ID: from environment variable TELEGRAM_CHAT_ID  
    - Photo: thumbnail URL  
    - Caption: includes video title and link  
    - Configure Telegram credentials.

17. **Create a Chain LLM node ("Create Email per Video")**  
    - Use OpenAI GPT-4o-mini model.  
    - Input: JSON list of videos.  
    - Prompt: Generate responsive HTML email cards per video with specified design requirements.

18. **Create a Gmail node ("Multiple Emails")**  
    - Send individual emails per video.  
    - Recipient: your email address.  
    - Subject: "Latest YouTube Videos from Your Favorite Channels".  
    - Use Gmail OAuth2 credentials.

19. **Create an Aggregate node ("One List Object")**  
    - Aggregate all video items into one JSON object.

20. **Create a Chain LLM node ("Create One Email for All Videos")**  
    - Use OpenAI GPT-4o-mini model.  
    - Input: aggregated JSON data.  
    - Prompt: Generate a single responsive HTML email listing all videos.

21. **Create a Gmail node ("Single Email")**  
    - Send digest email with all new videos.  
    - Recipient and subject same as above.  
    - Use Gmail OAuth2 credentials.

22. **Create a Form Trigger node ("On form submission")**  
    - Configure form with textarea for YouTube channel IDs array input.  
    - Connect to "Default YouTube Channel Ids" node.

23. **Connect nodes as per logical flow:**  
    - Schedule Trigger → YouTube Channel Ids → Default YouTube Channel Ids → Get Channel Ids → Create List of Channel Ids → Create RSS Feed URLs1 → RSS Read → Label New Videos → Get New Videos → Workflow Variables → Create YouTube API URL → Get YouTube Video Details → Prepare YouTube Data → Merge → Prepare For Telegram Message → Telegram  
    - Merge → Create Email per Video → Multiple Emails  
    - Merge → One List Object → Create One Email for All Videos → Single Email

24. **Set environment variables:**  
    - TELEGRAM_CHAT_ID for Telegram node.  
    - GOOGLE_API_KEY in Workflow Variables node.  
    - OpenAI API key in Chain LLM nodes.  
    - Gmail OAuth2 credentials for Gmail nodes.  
    - Telegram API credentials for Telegram node.

25. **Optional:** Add sticky notes for documentation and instructions.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                          |
|----------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Workflow automates monitoring and notifications for YouTube channels via RSS feeds with Telegram and email alerts.   | Workflow purpose                                                                                         |
| YouTube Data API documentation: https://developers.google.com/youtube/v3/docs                                        | API reference for video details retrieval                                                               |
| Google Cloud API key setup guide: https://cloud.google.com/docs/get-started/access-apis                                | Instructions for obtaining API keys                                                                     |
| OpenAI GPT-4o-mini model used for generating responsive HTML email templates                                         | AI content generation                                                                                     |
| Telegram bot token and chat ID required for Telegram notifications                                                   | Telegram integration setup                                                                               |
| Gmail OAuth2 credentials required for sending emails                                                                | Email sending setup                                                                                       |
| Default monitoring window for new videos is 3 days; adjustable in "Label New Videos" code node                       | Customization note                                                                                       |
| Email templates use safe, inline CSS and are optimized for mobile and desktop email clients                          | Email formatting best practices                                                                           |
| Environment variables used: TELEGRAM_CHAT_ID                                                                         | Secure configuration                                                                                      |
| The workflow includes a manual trigger node (disabled) for testing purposes                                          | Testing convenience                                                                                       |
| Sticky notes throughout the workflow provide inline documentation and setup tips                                     | User guidance                                                                                             |

---

This structured documentation provides a comprehensive understanding of the workflow’s design, node configurations, data flow, and integration points, enabling advanced users and AI agents to reproduce, modify, or troubleshoot the workflow effectively.