Monitor YouTube Channels and Send Daily Updates to Discord via RSS (No API Key)

https://n8nworkflows.xyz/workflows/monitor-youtube-channels-and-send-daily-updates-to-discord-via-rss--no-api-key--11212


# Monitor YouTube Channels and Send Daily Updates to Discord via RSS (No API Key)

### 1. Workflow Overview

This n8n workflow automates the monitoring of specified YouTube channels and sends daily updates about new videos posted within the last 24 hours to a Discord channel via webhook, without requiring a YouTube API key. It leverages YouTube’s publicly available RSS feeds for channels and schedules daily checks, providing a lightweight, no-authentication-required solution for content tracking.

The workflow consists of three main logical blocks:

- **1.1 Input Reception and Setup:** Defines the YouTube channel IDs to monitor and triggers the workflow daily at a fixed time.
- **1.2 RSS Feed Processing:** Splits the list of channel IDs, fetches their respective YouTube RSS feeds, and parses the new video entries.
- **1.3 Filtering and Notification:** Filters videos published within the last 24 hours and sends formatted notifications to a Discord channel using a webhook, pacing messages to avoid potential rate limits.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Setup

**Overview:**  
This block sets the workflow to run automatically every day at 8:30 AM and defines the list of YouTube channel IDs to monitor.

**Nodes Involved:**  
- Daily Trigger  
- Define Channel IDs

**Node Details:**

- **Daily Trigger**  
  - *Type & Role:* Schedule Trigger node; starts workflow execution daily at a specific time.  
  - *Configuration:* Set to trigger at 8:30 AM every day.  
  - *Expressions/Variables:* None.  
  - *Connections:* Output connected to “Define Channel IDs”.  
  - *Version Notes:* v1.2 or later for schedule rule configuration.  
  - *Potential Failures:* Rare; possible scheduling misconfiguration or instance downtime.

- **Define Channel IDs**  
  - *Type & Role:* Set node; defines an array variable containing YouTube channel IDs.  
  - *Configuration:* Assigns an array of strings representing YouTube Channel IDs (not handles).  
  - *Expressions/Variables:* Static array of channel IDs, e.g., `["UCxcDzs-4quJV4QsairlFYNg", ...]`.  
  - *Connections:* Input from “Daily Trigger”, output to “Split Channel IDs”.  
  - *Version Notes:* v3.4 for array assignment support.  
  - *Potential Failures:* Manual entry errors in channel IDs can cause broken feed URLs downstream.

---

#### 1.2 RSS Feed Processing

**Overview:**  
Takes the array of channel IDs, splits it into individual items, and fetches the RSS feed for each channel to retrieve recent video metadata.

**Nodes Involved:**  
- Split Channel IDs  
- Fetch Youtube RSS

**Node Details:**

- **Split Channel IDs**  
  - *Type & Role:* SplitOut node; transforms an array of channel IDs into individual items for parallel processing.  
  - *Configuration:* Splits on the field `channelids` which holds the array.  
  - *Expressions/Variables:* Uses the field `channelids` from the "Define Channel IDs" node.  
  - *Connections:* Input from “Define Channel IDs”, output to “Fetch Youtube RSS”.  
  - *Version Notes:* v1 standard.  
  - *Potential Failures:* If `channelids` is empty or incorrectly typed, no feeds will be fetched.

- **Fetch Youtube RSS**  
  - *Type & Role:* RSS Feed Read node; fetches and parses the RSS XML feed for each channel’s videos.  
  - *Configuration:* URL constructed dynamically as `https://www.youtube.com/feeds/videos.xml?channel_id={{ $json.channelids }}` to target the channel’s feed.  
  - *Expressions/Variables:* Uses expression to inject channel ID into URL.  
  - *Connections:* Input from “Split Channel IDs”, output to “Filter Last 24h”.  
  - *Version Notes:* v1.2 or later for RSS feed reading.  
  - *Potential Failures:*  
    - Invalid or mistyped channel IDs result in 404 or empty feeds.  
    - Network issues or YouTube RSS service downtime.  
    - Rate limits unlikely but possible if many channels queried simultaneously.

---

#### 1.3 Filtering and Notification

**Overview:**  
Filters videos to those published within the last 24 hours and sends notifications to Discord with a controlled delay between messages to avoid rate limits.

**Nodes Involved:**  
- Filter Last 24h  
- Wait 2 sec  
- Discord Notification

**Node Details:**

- **Filter Last 24h**  
  - *Type & Role:* Filter node; checks if the video’s publication date is within the last 24 hours.  
  - *Configuration:* Uses a dateTime condition comparing the `isoDate` field converted to a date string with current date minus one day.  
  - *Expressions/Variables:*  
    - Left value: `{{$json.isoDate.toDate().format('yyyy-MM-dd')}}`  
    - Right value: `{{$now.minus({ days:1 }).format('yyyy-MM-dd')}}`  
  - *Connections:* Input from “Fetch Youtube RSS”, output to “Wait 2 sec”.  
  - *Version Notes:* Uses v2.2 for enhanced date/time validation.  
  - *Potential Failures:*  
    - Parsing errors if `isoDate` is missing or malformed.  
    - Timezone differences may cause slight inaccuracies.

- **Wait 2 sec**  
  - *Type & Role:* Wait node; introduces a 2-second delay between messages to Discord.  
  - *Configuration:* 2 seconds delay.  
  - *Expressions/Variables:* None.  
  - *Connections:* Input from “Filter Last 24h”, output to “Discord Notification”.  
  - *Version Notes:* v1.1 or later for delay configuration.  
  - *Potential Failures:* Delay failure unlikely; workflow execution time may increase with many messages.

- **Discord Notification**  
  - *Type & Role:* Discord node; posts messages to a Discord channel via webhook.  
  - *Configuration:*  
    - Content formatted as:  
      ```
      Title: **{{ $json.title }}**  
      Date: `{{ $json.pubDate }}`  
      Link: {{ $json.link }}
      ```  
    - Authentication via Discord Webhook credentials.  
  - *Expressions/Variables:* Uses `title`, `pubDate`, and `link` from RSS feed item JSON.  
  - *Connections:* Input from “Wait 2 sec”.  
  - *Version Notes:* v2 for webhook support.  
  - *Potential Failures:*  
    - Invalid or revoked webhook URL causes authentication errors.  
    - Discord rate limits if messages are sent too quickly (mitigated by wait node).  
    - Content formatting errors if fields are missing.

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role                          | Input Node(s)        | Output Node(s)         | Sticky Note                                                                                                                 |
|---------------------|---------------------|----------------------------------------|----------------------|------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| Daily Trigger       | Schedule Trigger     | Triggers workflow daily at 8:30 AM     |                      | Define Channel IDs      | ## 0. Daily Trigger<br>Workflow executes daily at 8:30 am.                                                                 |
| Define Channel IDs  | Set                 | Defines array of YouTube channel IDs   | Daily Trigger        | Split Channel IDs       | ## 1. Define Channels<br>Add the YouTube Channel IDs here.                                                                   |
| Split Channel IDs   | SplitOut            | Splits channel ID array into items     | Define Channel IDs   | Fetch Youtube RSS       | ## 2. Process Feeds<br>Split the list and fetch RSS data.                                                                    |
| Fetch Youtube RSS   | RSS Feed Read       | Fetches YouTube RSS feed per channel   | Split Channel IDs    | Filter Last 24h         | ## 2. Process Feeds<br>Split the list and fetch RSS data.                                                                    |
| Filter Last 24h     | Filter              | Filters videos from last 24 hours      | Fetch Youtube RSS    | Wait 2 sec              | ## 3. Filter & Notify<br>Keep only last 24h videos.                                                                           |
| Wait 2 sec          | Wait                | Delays message sending by 2 seconds    | Filter Last 24h      | Discord Notification    | ## 3. Filter & Notify<br>Keep only last 24h videos.                                                                           |
| Discord Notification| Discord             | Sends video info to Discord webhook    | Wait 2 sec           |                        | ## 3. Filter & Notify<br>Keep only last 24h videos.                                                                           |
| Sticky Note3        | Sticky Note         | Describes daily trigger block           |                      |                        | ## 0. Daily Trigger<br>Workflow executes daily at 8:30 am.                                                                   |
| Chapter 1           | Sticky Note         | Notes for channel definition block      |                      |                        | ## 1. Define Channels<br>Add the YouTube Channel IDs here.                                                                   |
| Chapter 2           | Sticky Note         | Notes for feed processing block         |                      |                        | ## 2. Process Feeds<br>Split the list and fetch RSS data.                                                                    |
| Chapter 3           | Sticky Note         | Notes for filtering and notification block |                  |                        | ## 3. Filter & Notify<br>Keep only last 24h videos.                                                                           |
| Sticky Note         | Sticky Note         | YouTube monitor setup instructions      |                      |                        | ## 📺 YouTube Monitor Setup<br>This workflow tracks specific YouTube channels and notifies Discord of new videos from last 24h.|
| Sticky Note5        | Sticky Note         | Demo configuration description          |                      |                        | ## 🟢 Demo Configuration (Spanish AI Channels)<br>This template is pre-loaded with top Spanish-speaking AI channels.          |
| Sticky Note4        | Sticky Note         | Author bio and contact info              |                      |                        | ## Hey, I'm Vasyl Pavlyuchok<br>I design AI-powered automation systems for founders and agencies.<br>See LinkedIn & YouTube.  |
| Sticky Note7        | Sticky Note         | Author profile image                     |                      |                        | ![Vasyl Pavlyuchok](https://assets.zyrosite.com/mp86MKQeprTxrnEW/profile-pic-2025_10-Yan0j4jpxeUzzGlm.png)                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node:**  
   - Name: `Daily Trigger`  
   - Set to trigger daily at 8:30 AM (hour=8, minute=30).

2. **Create a Set Node:**  
   - Name: `Define Channel IDs`  
   - Add a new field named `channelids` of type Array.  
   - Enter the list of YouTube channel IDs you want to monitor, e.g.:  
     ```["UCxcDzs-4quJV4QsairlFYNg", "UC7iDdffeN0GW3VDehzHq24w", "UC_RovKmk0OCbuZjA8f08opw", "UCl5-lvQyfILb-l2abPk4Ntg"]```  
   - Connect `Daily Trigger` output to this node’s input.

3. **Create a SplitOut Node:**  
   - Name: `Split Channel IDs`  
   - Configure to split on field `channelids`.  
   - Connect output of `Define Channel IDs` to this node.

4. **Create an RSS Feed Read Node:**  
   - Name: `Fetch Youtube RSS`  
   - For the URL parameter, use an expression:  
     ```
     https://www.youtube.com/feeds/videos.xml?channel_id={{ $json.channelids }}
     ```  
   - Connect output of `Split Channel IDs` to this node.

5. **Create a Filter Node:**  
   - Name: `Filter Last 24h`  
   - Add a condition:  
     - Operation: DateTime “afterOrEquals”  
     - Left Value: `{{$json.isoDate.toDate().format('yyyy-MM-dd')}}` (expression)  
     - Right Value: `{{$now.minus({ days:1 }).format('yyyy-MM-dd')}}` (expression)  
   - Connect output of `Fetch Youtube RSS` to this node.

6. **Create a Wait Node:**  
   - Name: `Wait 2 sec`  
   - Set wait time to 2 seconds.  
   - Connect output of `Filter Last 24h` to this node.

7. **Create a Discord Node:**  
   - Name: `Discord Notification`  
   - Set authentication to ‘Webhook’ and select/create a Discord webhook credential.  
   - Set message content to:  
     ```
     Title: **{{ $json.title }}**  
     Date: `{{ $json.pubDate }}`  
     Link: {{ $json.link }}
     ```  
   - Connect output of `Wait 2 sec` to this node.

8. **Credentials Setup:**  
   - For the Discord node, create or provide credentials for a Discord webhook with the appropriate permissions to post messages to your target channel.

9. **Test Workflow Execution:**  
   - Run the workflow manually or wait for the trigger time.  
   - Verify that for each channel, new videos published in the last 24 hours post messages in your Discord channel.

10. **Optional:** Add sticky notes for documentation and clarity, reflecting the blocks:  
    - “Daily Trigger” explanation  
    - Channel ID definition instructions  
    - Feed processing notes  
    - Filtering and notification notes  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                  | Context or Link                                                                                         |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| This workflow tracks specific YouTube channels and notifies Discord of new videos from the last 24 hours using RSS feeds—no API key required.                  | Workflow purpose                                                                                       |
| How to find YouTube Channel ID: use an online finder or inspect page source searching for `channel_id`.                                                       | https://www.youtube.com/ (search channel page source)                                                |
| Author: Vasyl Pavlyuchok, AI-powered automation systems designer. Connect on LinkedIn and YouTube for more automation and AI content.                        | LinkedIn: https://www.linkedin.com/in/vasyl-pavlyuchok/ <br> YouTube: https://www.youtube.com/@vasylpavlyuchok |
| Demo configuration includes Spanish AI channels such as Alejavi Rivera, academIArtificial, Futurepedia, and Inteligencia Artificial.                          | Demo channels in “Define Channel IDs” node                                                           |
| To avoid Discord rate-limiting, a 2-second delay is introduced between message postings.                                                                       | See “Wait 2 sec” node configuration                                                                  |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, a workflow automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.