Automatically promote your YouTube video on X

https://n8nworkflows.xyz/workflows/automatically-promote-your-youtube-video-on-x-2564


# Automatically promote your YouTube video on X

### 1. Workflow Overview

This n8n workflow automates the promotion of newly uploaded YouTube videos on X (formerly Twitter), ensuring posts are engaging, concise, and compliant with platform limits. It fetches recent videos, generates witty posts using OpenAI GPT-4, validates and rewrites posts to fit X’s 220-character limit, publishes posts on X, and logs all activities in a Google Sheet. Optional notification nodes for Discord, Slack, and Gmail are included but disabled by default.

The workflow is logically divided into these functional blocks:

- **1.1 Input Trigger & Scheduling:** Manual or scheduled trigger to initiate workflow.
- **1.2 YouTube Video Retrieval & Deduplication:** Fetches latest videos and removes duplicates.
- **1.3 Post Generation (OpenAI):** Creates engaging X posts, validates length, and rewrites if necessary.
- **1.4 Google Sheets Logging:** Records post data before and after publishing.
- **1.5 Posting to X:** Publishes posts and updates logs with post URLs.
- **1.6 Optional Notifications:** Sends alerts via Discord, Slack, or Gmail (disabled).

---

### 2. Block-by-Block Analysis

#### 2.1 Input Trigger & Scheduling

- **Overview:** Starts the workflow either manually or on a scheduled interval (every 2 hours, disabled by default).
- **Nodes Involved:**  
  - `When clicking "Test workflow"` (Manual Trigger)  
  - `Check Every 2 Hours` (Schedule Trigger - disabled)

- **Node Details:**

  - **When clicking "Test workflow"**  
    - *Type:* Manual Trigger  
    - *Role:* Allows manual start of the workflow for testing or on-demand runs.  
    - *Connections:* Outputs to `Fetch Latest Videos`.  
    - *Edge cases:* None significant; user-triggered.

  - **Check Every 2 Hours**  
    - *Type:* Schedule Trigger  
    - *Role:* Automated trigger every 2 hours at a random minute.  
    - *Configuration:* Interval every 2 hours; random trigger minute to avoid fixed time hits.  
    - *Disabled:* Yes (off by default).  
    - *Connections:* None (disabled).  
    - *Edge cases:* Potential API rate limits if enabled too frequently; random minute helps distribute load.

---

#### 2.2 YouTube Video Retrieval & Deduplication

- **Overview:** Fetches the latest videos from a specified YouTube channel in the last 20 hours, then removes duplicates based on video ID to avoid reposting content.
- **Nodes Involved:**  
  - `Fetch Latest Videos`  
  - `Set Fields`  
  - `Remove Duplicates`  
  - `If Array is empty?`  
  - `Sticky Note` (instructional)

- **Node Details:**

  - **Fetch Latest Videos**  
    - *Type:* YouTube API node  
    - *Role:* Retrieves up to 2 recent videos published after a dynamic timestamp (20 hours ago).  
    - *Configuration:*  
      - Channel ID: `UC3JB8Cnync-WCDYKwOYSQUg` (user-specified YouTube channel)  
      - Published After: Dynamic ISO date 20 hours prior to current time  
      - Limit: 2 videos  
    - *Credentials:* YouTube OAuth2 (AlexK1919)  
    - *Output:* Video metadata including videoId, title, description.  
    - *Edge cases:* No new videos published, API quota exceeded, invalid channel ID.

  - **Set Fields**  
    - *Type:* Set node  
    - *Role:* Ensures `id.videoId` field is explicitly set for downstream deduplication.  
    - *Configuration:* Copies `id.videoId` from fetched video data, retains other fields.  
    - *Input:* Output from `Fetch Latest Videos`  
    - *Output:* Modified items with explicit videoId.  

  - **Remove Duplicates**  
    - *Type:* Remove Duplicates node  
    - *Role:* Filters out videos already processed in previous executions to avoid reposting.  
    - *Configuration:*  
      - Scope: Workflow-wide memory  
      - History Size: 10,000 (tracks previously seen videoIds)  
      - Dedupe by: `id.videoId`  
    - *Input:* From `Set Fields`  
    - *Output:* Unique new videos only  
    - *Edge cases:* Workflow memory overflow if too many unique videos over time.

  - **If Array is empty?**  
    - *Type:* If node  
    - *Role:* Checks if any new videos remain after deduplication.  
    - *Configuration:* Checks if `Remove Duplicates` output array is empty.  
    - *Branches:*  
      - True (empty): Workflow ends, no posts generated.  
      - False (not empty): Proceeds to post generation.  
    - *Input:* From `Remove Duplicates`  
    - *Output:* To `Generate X Post` if not empty.

  - **Sticky Note (Fetch latest YouTube video and dedupe)**  
    - *Role:* Instructional note for user to input Channel ID.

---

#### 2.3 Post Generation (OpenAI)

- **Overview:** Generates a witty, engaging X post about the video using OpenAI GPT-4, validates the character length, and rewrites the post if it exceeds 220 characters.
- **Nodes Involved:**  
  - `Generate X Post`  
  - `Verify character limit constraints` (If node)  
  - `Rewrite X Post to 220 Characters`  
  - `Set Final Fields`  
  - `Sticky Note` (validation and generation instructions)

- **Node Details:**

  - **Generate X Post**  
    - *Type:* OpenAI GPT-4 node via Langchain integration  
    - *Role:* Creates a humanized, witty X post under 220 characters including YouTube video link at the end.  
    - *Configuration:*  
      - Model: `gpt-4o-mini`  
      - Prompt: Uses video title, description, and video URL dynamically embedded in prompt.  
      - Instructions: No emojis; witty and human tone.  
    - *Input:* New videos from `If Array is empty?` (False branch)  
    - *Output:* JSON with generated post content at `message.content.post`.  
    - *Edge cases:* API key issues, prompt failures, or empty descriptions handled by fallback in prompt.

  - **Verify character limit constraints**  
    - *Type:* If node  
    - *Role:* Checks if generated post exceeds 280 characters (X's limit).  
    - *Condition:* `post.length > 280`  
    - *Branches:*  
      - True: Send post to rewriting node.  
      - False: Proceed to finalization.  
    - *Input:* From `Generate X Post` or `Rewrite X Post to 220 Characters`.

  - **Rewrite X Post to 220 Characters**  
    - *Type:* OpenAI GPT-4 node  
    - *Role:* Modifies the post to ensure it fits within 220 characters, preserving the video link at the end and maintaining tone.  
    - *Configuration:* Similar to `Generate X Post` but with rewrite instructions.  
    - *Input:* From `Verify character limit constraints` (True branch)  
    - *Output:* Rewritten post.  

  - **Set Final Fields**  
    - *Type:* Set node  
    - *Role:* Passes finalized post content along for logging and publishing.  
    - *Input:* From `Verify character limit constraints` (False branch or after rewriting)  
    - *Output:* To Google Sheets logging node.

  - **Sticky Notes**  
    - Provide instructional context for this block regarding validation and generation.

---

#### 2.4 Google Sheets Logging

- **Overview:** Logs the generated post data to a Google Sheet before and after posting, including timestamps, status, and post URLs.
- **Nodes Involved:**  
  - `GS - Add Tweet`  
  - `GS - Update Tweet`  
  - `Sticky Note` (logging instructions)

- **Node Details:**

  - **GS - Add Tweet**  
    - *Type:* Google Sheets node  
    - *Role:* Appends a new row recording the post content, date, time, status ("Post written"), and channel ("X").  
    - *Configuration:*  
      - Document: `AlexK1919 Social Posts` Google Sheet (ID: `1Ql9TGAzZCSdSqrHvkZLcsBPoNMAjNpPVsELkumP2heM`)  
      - Sheet: `Sheet1` (gid=0)  
      - Columns: xid (random ID), date (ISO), time (24h), post, status, channel  
      - Use Append operation  
    - *Input:* From `Set Final Fields`  
    - *Output:* To `Post to X`  
    - *Edge cases:* API quota limits, permission errors.

  - **GS - Update Tweet**  
    - *Type:* Google Sheets node  
    - *Role:* Updates the existing row (matched by xid) after posting to X with status "Posted to X" and post URL.  
    - *Configuration:*  
      - Same Google Sheet and sheet as above  
      - Matching column: xid  
      - Updates date, time, status, post_url (with Twitter URL), channel, and post content.  
    - *Input:* From `Post to X` (post response)  
    - *Output:* None (end of this branch)  
    - *Edge cases:* Row not found, API permission errors.

  - **Sticky Notes**  
    - Highlight the purpose and configuration of Google Sheets logging.

---

#### 2.5 Posting to X

- **Overview:** Publishes the generated post on X using OAuth2 credentials and passes the post ID to update Google Sheets.
- **Nodes Involved:**  
  - `Post to X`  
  - `Sticky Note` (posting instructions)

- **Node Details:**

  - **Post to X**  
    - *Type:* Twitter node (X API)  
    - *Role:* Posts the text generated by OpenAI to X.  
    - *Configuration:*  
      - Text: From `Verify character limit constraints` or rewritten post  
      - Credentials: OAuth2 for user "Alex Kim X account"  
    - *Input:* From `GS - Add Tweet`  
    - *Output:* To `GS - Update Tweet` with post ID and text.  
    - *Edge cases:* Auth errors, rate limits, text formatting errors.

  - **Sticky Note**  
    - Informs that this node posts on X and subsequently updates the Google Sheet.

---

#### 2.6 Optional Notifications

- **Overview:** Sends notifications of the new post via Discord, Slack, or Gmail. These are disabled by default and can be enabled as needed.
- **Nodes Involved:**  
  - `Discord`  
  - `Slack`  
  - `Gmail`  
  - `Sticky Note` (optional functions)

- **Node Details:**

  - **Discord**  
    - *Type:* Discord node  
    - *Role:* Sends a webhook message announcing the new X post.  
    - *Configuration:* Uses webhook authentication; message includes content and URL.  
    - *Input:* Not connected by default.

  - **Slack**  
    - *Type:* Slack node  
    - *Role:* Sends Slack message notification.  
    - *Configuration:* Webhook ID set, no additional customizations.  
    - *Input:* Not connected by default.

  - **Gmail**  
    - *Type:* Gmail node  
    - *Role:* Sends email notifications about the post.  
    - *Configuration:* Uses OAuth2 credentials for Gmail account `AlexK1919 Gmail`.  
    - *Input:* Not connected by default.

  - **Sticky Note**  
    - Marks these nodes as optional notification functions.

---

### 3. Summary Table

| Node Name                      | Node Type                  | Functional Role                          | Input Node(s)                | Output Node(s)               | Sticky Note                                                                                                   |
|--------------------------------|----------------------------|----------------------------------------|-----------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------------|
| When clicking "Test workflow"  | Manual Trigger             | Manual workflow start                   |                             | Fetch Latest Videos          |                                                                                                              |
| Check Every 2 Hours            | Schedule Trigger           | Scheduled workflow start (disabled)    |                             |                             |                                                                                                              |
| Fetch Latest Videos            | YouTube API                | Retrieve recent videos                  | When clicking "Test workflow" | Set Fields                  | Enter your YouTube Channel ID in this node. Find your Channel ID [here](https://youtube.com/account_advanced).|
| Set Fields                    | Set                        | Prepare fields for deduplication        | Fetch Latest Videos          | Remove Duplicates            |                                                                                                              |
| Remove Duplicates             | Remove Duplicates           | Filter out already posted videos        | Set Fields                  | If Array is empty?           |                                                                                                              |
| If Array is empty?            | If                         | Check if new videos exist                | Remove Duplicates            | (True: end), (False: Generate X Post) |                                                                                                              |
| Generate X Post               | OpenAI GPT-4               | Create witty X post about video          | If Array is empty? (False)   | Verify character limit constraints | ## Generate X Post                                                                                             |
| Verify character limit constraints | If                     | Validate post length                     | Generate X Post / Rewrite X Post | Rewrite X Post to 220 Characters / Set Final Fields | ## Validate: Is Post under 240 characters?                                                                     |
| Rewrite X Post to 220 Characters | OpenAI GPT-4             | Rewrite post to fit character limit     | Verify character limit constraints (True) | Verify character limit constraints |                                                                                                              |
| Set Final Fields             | Set                        | Finalize post content for logging      | Verify character limit constraints (False) | GS - Add Tweet             |                                                                                                              |
| GS - Add Tweet               | Google Sheets              | Log new post details                     | Set Final Fields             | Post to X                   | ## Add to Google Sheet                                                                                         |
| Post to X                   | Twitter (X) API             | Publish post on X                        | GS - Add Tweet               | GS - Update Tweet           | ## Post to X and update Google Sheet with Post Link                                                           |
| GS - Update Tweet           | Google Sheets              | Update post status and URL               | Post to X                   |                             |                                                                                                              |
| Discord                     | Discord                    | Optional notification via Discord       |                             |                             | ## Optional functions                                                                                           |
| Slack                       | Slack                      | Optional notification via Slack         |                             |                             |                                                                                                              |
| Gmail                       | Gmail                      | Optional notification via Email         |                             |                             |                                                                                                              |
| Sticky Note (multiple)      | Sticky Note                | Various user instructions and notes     |                             |                             | Covers instruction for Channel ID, generation, validation, posting, logging, optional notifications, branding |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node**  
   - Name: `When clicking "Test workflow"`  
   - Purpose: Manual start.

2. **(Optional) Create Schedule Trigger node**  
   - Name: `Check Every 2 Hours`  
   - Set interval to every 2 hours at a random minute (disabled by default).

3. **Create YouTube node**  
   - Name: `Fetch Latest Videos`  
   - Resource: Video  
   - Operation: List videos from channel  
   - Channel ID: Set your YouTube Channel ID (e.g., `UC3JB8Cnync-WCDYKwOYSQUg`)  
   - Limit: 2  
   - Published After: Expression `={{ new Date(new Date().getTime() - 1200 * 60000).toISOString() }}` (20 hours ago)  
   - Credentials: YouTube OAuth2 (user’s YouTube account).

4. **Create Set node**  
   - Name: `Set Fields`  
   - Assign field `id.videoId` with value `={{ $json.id.videoId }}`  
   - Include other fields unchanged.

5. **Create Remove Duplicates node**  
   - Name: `Remove Duplicates`  
   - Scope: Workflow  
   - History Size: 10000  
   - Dedupe by: `id.videoId`.

6. **Create If node**  
   - Name: `If Array is empty?`  
   - Condition: Check if input array is empty (`{{$json}}` from Remove Duplicates)  
   - True branch: No new videos, stop workflow.  
   - False branch: Proceed to post generation.

7. **Create OpenAI GPT-4 node**  
   - Name: `Generate X Post`  
   - Model: `gpt-4o-mini`  
   - Prompt:  
     ```
     Write an engaging post about my latest YouTube video for X (Twitter) of no more than 220 characters in length. Link to the video at https://youtu.be/{{ $json.id.videoId }} use this title and description: {{ $json.snippet.title }} {{ $json.snippet.description }}. If there is no description available, use your best guess as to the context of the video. Make sure the YouTube link is at the end of the content.
     ```  
   - Assistant role content: `Be witty. Humanize the content. No emojis.`  
   - Credentials: OpenAI API key.

8. **Create If node**  
   - Name: `Verify character limit constraints`  
   - Condition: Check if `{{ $json.message.content.post.length }}` > 280 characters.  
   - True branch: Send to rewriting node.  
   - False branch: Proceed to finalization.

9. **Create OpenAI GPT-4 node**  
   - Name: `Rewrite X Post to 220 Characters`  
   - Model: `gpt-4o-mini`  
   - Prompt:  
     ```
     Rewrite the content so it is less than 220 characters long in total length. Content: {{ $json.message.content.post }} Make sure the YouTube Link is at the end of the content.
     ```  
   - Assistant role content: `Be witty. Humanize the content. No emojis.`  
   - Credentials: OpenAI API key.

10. **Connect rewritten output back to** `Verify character limit constraints` (to re-validate length).

11. **Create Set node**  
    - Name: `Set Final Fields`  
    - Purpose: Pass finalized post data forward unchanged.

12. **Create Google Sheets node**  
    - Name: `GS - Add Tweet`  
    - Operation: Append row  
    - Document ID: Use your Google Sheet ID  
    - Sheet Name: `Sheet1` (or appropriate)  
    - Columns:  
      - xid: random string `={{ Math.random().toString(36).substr(2, 12) }}`  
      - date: current date `={{ new Date().toISOString().split('T')[0] }}`  
      - time: current time 24h format  
      - post: finalized post text  
      - status: "Post written"  
      - channel: "X"  
    - Credentials: Google Sheets OAuth2.

13. **Create Twitter (X) node**  
    - Name: `Post to X`  
    - Text: `={{ $json.message.content.post }}` (from Google Sheets add tweet output)  
    - Credentials: Twitter OAuth2 (your X account).

14. **Create Google Sheets node**  
    - Name: `GS - Update Tweet`  
    - Operation: Append or update row  
    - Match by xid (from `GS - Add Tweet`)  
    - Update columns: status to "Posted to X", add `post_url` as `https://twitter.com/alexkim/status/{{ $json.id }}`, update other fields as needed.  
    - Credentials: Google Sheets OAuth2.

15. **Optional: Create Discord node**  
    - Configure webhook URL  
    - Message template referencing the new post content and URL.

16. **Optional: Create Slack node**  
    - Configure webhook ID  
    - Setup message content.

17. **Optional: Create Gmail node**  
    - Configure OAuth2 credentials  
    - Setup email message content.

18. **Connect nodes according to logical flow:**  
    - Manual trigger → Fetch Latest Videos → Set Fields → Remove Duplicates → If Array is empty? → Generate X Post → Verify character limit constraints → (True) Rewrite X Post → Verify again → (False) Set Final Fields → GS - Add Tweet → Post to X → GS - Update Tweet  
    - Optional notifications can be connected after `GS - Update Tweet` or `Post to X` as desired.

19. **Add Sticky Notes** at relevant points for instructions and documentation.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                | Context or Link                                                                                                    |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| Enter your YouTube Channel ID in the `Fetch Latest Videos` node. Find your Channel ID here: https://youtube.com/account_advanced                          | YouTube setup instruction                                                                                         |
| Google Sheet for logging: https://docs.google.com/spreadsheets/d/1Ql9TGAzZCSdSqrHvkZLcsBPoNMAjNpPVsELkumP2heM/edit#gid=0                                  | Sheet used for logging posts                                                                                       |
| Posts are restricted to 220 characters to ensure compliance with X's limit, leaving room for links and avoiding truncation                                 | Platform compliance                                                                                               |
| OpenAI GPT-4 model used: gpt-4o-mini (smaller variant for efficiency)                                                                                        | Model choice for cost and speed optimization                                                                      |
| Optional notifications via Discord, Slack, and Gmail are disabled by default but can be enabled to notify about new posts                                  | Extensibility and alerting                                                                                         |
| For support or inquiries, contact Alex Kim: https://beacons.ai/alexk1919                                                                                   | Author information                                                                                                |
| Tools like Wikipedia and Calculator nodes exist but are unused or auxiliary (connected as AI tools)                                                        | Potential for future AI enhancements                                                                               |

---

This documentation enables comprehensive understanding, reproduction, and modification of the YouTube to X automated promotion workflow in n8n.