Automatically Post Latest Reddit Content to Discord with Image Extraction

https://n8nworkflows.xyz/workflows/automatically-post-latest-reddit-content-to-discord-with-image-extraction-5105


# Automatically Post Latest Reddit Content to Discord with Image Extraction

### 1. Workflow Overview

This workflow automates the process of fetching the latest Reddit posts from a specified subreddit, filters out moderator announcement posts, extracts direct image URLs from the posts if available, and posts the content along with the image to a Discord channel via webhook. It is designed to run on a scheduled interval, making it suitable for continuously updating a Discord community with fresh Reddit content in near real-time.

The workflow is logically divided into four main blocks:

- **1.1 Scheduling:** Triggers the workflow every 15 minutes to check for new Reddit posts.
- **1.2 Fetch and Filter Reddit Posts:** Retrieves the latest posts from a subreddit and filters out announcement posts by moderators.
- **1.3 Post Fetching and URL Extraction:** For each filtered post, fetches detailed post data and extracts a direct image URL if present.
- **1.4 Posting to Discord:** Sends the post title, image URL, and original Reddit URL to a Discord channel using a webhook.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduling

- **Overview:**  
  Initiates the workflow on a timed schedule every 15 minutes to automate periodic checks for new Reddit posts.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Sticky Note4 (Scheduling explanation)

- **Node Details:**

  - **Schedule Trigger**  
    - *Type:* Trigger node  
    - *Role:* Starts workflow every 15 minutes.  
    - *Configuration:* Interval set to every 15 minutes.  
    - *Input:* None (trigger node).  
    - *Output:* Triggers "Fetch latest reddit posts" node.  
    - *Edge Cases:* If n8n instance is down or paused, scheduling is missed; no retry logic here.

  - **Sticky Note4**  
    - *Type:* Sticky Note  
    - *Content:* Explains the scheduling setup and frequency.  
    - *Purpose:* Documentation aid only.

---

#### 1.2 Fetch and Filter Reddit Posts

- **Overview:**  
  Fetches the three latest posts from the configured subreddit ("news") and filters out posts authored by a specific moderator (identified by author_fullname "t2_6l4z3"), which are considered announcement posts.

- **Nodes Involved:**  
  - Fetch latest reddit posts  
  - Filter out announcement posts  
  - Sticky Note (Fetch and filter reddit posts explanation)

- **Node Details:**

  - **Fetch latest reddit posts**  
    - *Type:* Reddit node (Get All posts)  
    - *Role:* Retrieves the latest 3 posts from subreddit "news".  
    - *Configuration:*  
      - Subreddit: "news"  
      - Limit: 3 posts  
      - Filters: None applied here (default).  
    - *Credentials:* Reddit OAuth2 (configured with client credentials).  
    - *Input:* Trigger from Schedule Trigger node.  
    - *Output:* Sends posts to "Filter out announcement posts".  
    - *Edge Cases:* API rate limits, invalid credentials, subreddit not found, network failures.

  - **Filter out announcement posts**  
    - *Type:* If node  
    - *Role:* Filters out posts where author_fullname equals "t2_6l4z3" (a mod account).  
    - *Configuration:* Condition checks if `author_fullname` equals "t2_6l4z3".  
    - *Input:* Receives posts from "Fetch latest reddit posts".  
    - *Output:*  
      - True branch (author matches mod): no output (empty array), effectively filtering posts out.  
      - False branch (author different): passes post to "Fetch Reddit Post" for further processing.  
    - *Edge Cases:* If author_fullname field is missing or changed, filtering may fail.

  - **Sticky Note**  
    - *Type:* Sticky Note  
    - *Content:* Describes fetching posts and filtering out mod announcements.  
    - *Purpose:* Documentation only.

---

#### 1.3 Post Fetching and URL Extraction

- **Overview:**  
  For each filtered post, fetches detailed post data by post ID to access media metadata, then extracts a direct image URL if available from Reddit's media metadata.

- **Nodes Involved:**  
  - Fetch Reddit Post  
  - Extract Image URL  
  - Sticky Note2 (Post fetching and URL extraction explanation)

- **Node Details:**

  - **Fetch Reddit Post**  
    - *Type:* Reddit node (Get single post)  
    - *Role:* Retrieves detailed information about a single post using its post ID.  
    - *Configuration:*  
      - Subreddit: "news" (same as earlier)  
      - Post ID: Expression `={{ $('Filter out announcement posts').item.json.id }}` (post ID from filtered post)  
      - Operation: Get single post.  
    - *Credentials:* Reddit OAuth2 (same credentials as before).  
    - *Input:* Receives post ID from the false branch of "Filter out announcement posts".  
    - *Output:* Sends detailed post JSON to "Extract Image URL".  
    - *Edge Cases:* Invalid or deleted post ID, API errors, network issues.

  - **Extract Image URL**  
    - *Type:* Code node (JavaScript)  
    - *Role:* Parses post JSON to extract a direct image URL from Reddit media metadata, if present.  
    - *Configuration:*  
      - JavaScript code inspects `media_metadata` JSON property, extracts first image's URL, replaces HTML encoded "&amp;" with "&", and constructs a direct URL in the format `https://i.redd.it/{imageName}.{extension}`.  
      - Returns `{ directUrl: URL | null }`.  
    - *Input:* Receives detailed post JSON from "Fetch Reddit Post".  
    - *Output:* Sends an object with `directUrl` property to "Send to discord".  
    - *Edge Cases:*  
      - Posts without media metadata (returns null URL).  
      - Posts with unsupported media types or malformed metadata.  
      - Multiple media entries: only the first is considered.  
      - Could fail if Reddit JSON structure changes.

  - **Sticky Note2**  
    - *Type:* Sticky Note  
    - *Content:* Explains the post fetching step and image URL extraction.  
    - *Purpose:* Documentation only.

---

#### 1.4 Posting to Discord

- **Overview:**  
  Sends a message to a Discord channel including the Reddit post title, the extracted direct image URL if any, and the original Reddit post URL.

- **Nodes Involved:**  
  - Send to discord  
  - Sticky Note3 (Posting to platforms explanation)

- **Node Details:**

  - **Send to discord**  
    - *Type:* Discord node (Webhook)  
    - *Role:* Posts a message to a Discord channel via webhook authentication.  
    - *Configuration:*  
      - Content:  
        ```
        New post on reddit:
        {{ $('Fetch Reddit Post').item.json.title }}
        {{ $json.directUrl }}
        {{ $('Filter out announcement posts').item.json.url_overridden_by_dest }}
        ```  
      - Authentication: Webhook  
      - Webhook ID: configured in n8n instance  
    - *Credentials:* Discord Webhook API credentials.  
    - *Input:* Receives image URL from "Extract Image URL" node and uses expressions to access post title and original Reddit URL.  
    - *Output:* None (end node).  
    - *Edge Cases:*  
      - Discord webhook errors (invalid URL, revoked permissions).  
      - Message length limits.  
      - Null or empty image URLs still included in message.  
      - Network or API failures.

  - **Sticky Note3**  
    - *Type:* Sticky Note  
    - *Content:* Notes that this sends direct image + context to social media platforms (here Discord).  
    - *Purpose:* Documentation only.

---

#### Additional Notes on Setup

- **Sticky Note1** details setup instructions:  
  - Create Reddit app and obtain client ID and secret.  
  - Configure redirect URL to n8n base URL for OAuth2.  
  - Set the subreddit name in both Reddit nodes (currently "news").  
  - Add a Discord webhook URL for message posting.

---

### 3. Summary Table

| Node Name                  | Node Type               | Functional Role                       | Input Node(s)               | Output Node(s)                  | Sticky Note                                                    |
|----------------------------|-------------------------|------------------------------------|-----------------------------|---------------------------------|---------------------------------------------------------------|
| Schedule Trigger            | Schedule Trigger        | Triggers workflow every 15 minutes | None                        | Fetch latest reddit posts       | ## Scheduling - Set up the frequency of time to check for new posts |
| Fetch latest reddit posts   | Reddit                  | Fetches latest 3 posts from subreddit | Schedule Trigger            | Filter out announcement posts   | ## Fetch and filter reddit posts - Fetches latest post from a subreddit, filters out mod posts. |
| Filter out announcement posts | If                     | Filters out mod announcement posts  | Fetch latest reddit posts    | Fetch Reddit Post (false branch) | ## Fetch and filter reddit posts - Filters out posts by author_fullname "t2_6l4z3" |
| Fetch Reddit Post           | Reddit                  | Fetches detailed post data by ID    | Filter out announcement posts (false branch) | Extract Image URL              | ## Post fetching and URL extraction - Fetches per post data, extracts direct URL for image. |
| Extract Image URL           | Code                    | Extracts direct image URL from media metadata | Fetch Reddit Post          | Send to discord                 | ## Post fetching and URL extraction - Extracts direct image URL from Reddit media metadata |
| Send to discord             | Discord                 | Sends formatted message to Discord  | Extract Image URL            | None                          | ## Post To platforms - Send Direct image + context to social media platforms |
| Sticky Note                 | Sticky Note             | Documentation                      | None                        | None                          | ## Fetch and filter reddit posts - Fetches latest post from a subreddit, filters out mod posts. |
| Sticky Note1                | Sticky Note             | Setup instructions                 | None                        | None                          | ## SETUP - Reddit app, credentials, subreddit config, Discord webhook setup |
| Sticky Note2                | Sticky Note             | Documentation                      | None                        | None                          | ## Post fetching and URL extraction - Fetches per post data, extracts direct URL for image. |
| Sticky Note3                | Sticky Note             | Documentation                      | None                        | None                          | ## Post To platforms - Send Direct image + context to social media platforms |
| Sticky Note4                | Sticky Note             | Documentation                      | None                        | None                          | ## Scheduling - Set up the frequency of time to check for new posts |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node:**  
   - Type: Schedule Trigger  
   - Set interval to every 15 minutes.

2. **Create Reddit Node to Fetch Latest Posts:**  
   - Type: Reddit  
   - Operation: Get All  
   - Subreddit: "news" (or your target subreddit)  
   - Limit: 3 posts  
   - Connect output of Schedule Trigger to input of this node.  
   - Credentials: Configure Reddit OAuth2 credentials with client ID, secret, and redirect URL set to your n8n instance URL.

3. **Create If Node to Filter Announcement Posts:**  
   - Type: If  
   - Condition: Check if `author_fullname` equals `t2_6l4z3` (mod account ID to filter out).  
   - Connect output of Reddit fetch node to input of this If node.

4. **Create Reddit Node to Fetch Single Post Details:**  
   - Type: Reddit  
   - Operation: Get  
   - Subreddit: "news"  
   - Post ID: Expression `={{ $json["id"] }}` from the false branch of If node (posts not filtered out).  
   - Connect false output branch of If node to this node.  
   - Use same Reddit OAuth2 credentials.

5. **Create Code Node to Extract Image URL:**  
   - Type: Code  
   - Language: JavaScript  
   - Paste the following code:
     ```javascript
     return items.map(item => {
       const mediaMetadata = item.json.media_metadata;
       if (!mediaMetadata) return { directUrl: null };

       const mediaKey = Object.keys(mediaMetadata)[0];
       const media = mediaMetadata[mediaKey];
       if (!media || !media.s || !media.s.u) return { directUrl: null };

       let url = media.s.u.replace(/&amp;/g, '&');

       const match = url.match(/([a-z0-9]+)\.(jpg|png|gif)/i);
       if (!match) return { directUrl: null };

       const imageName = match[1];
       const extension = match[2];

       const directUrl = `https://i.redd.it/${imageName}.${extension}`;

       return { directUrl };
     });
     ```
   - Connect output of single post fetch node to this code node.

6. **Create Discord Node to Send Message:**  
   - Type: Discord  
   - Authentication: Webhook  
   - Configure webhook URL or select existing Discord webhook credentials.  
   - Content:
     ```
     New post on reddit:
     {{ $('Fetch Reddit Post').item.json.title }}
     {{ $json.directUrl }}
     {{ $('Filter out announcement posts').item.json.url_overridden_by_dest }}
     ```
   - Connect output of Code node (image URL extractor) to this Discord node.

7. **Add Sticky Notes for Documentation (Optional):**  
   - Create sticky notes with the content provided in the existing workflow for scheduling, fetching/filtering posts, extracting URLs, posting to Discord, and setup instructions.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Setup requires creating a Reddit app with OAuth2 credentials; redirect URL must be set to your n8n instance. | See Reddit API documentation: https://www.reddit.com/prefs/apps                                  |
| Discord webhook must be created in your Discord server and configured in n8n credentials.                      | Discord webhook setup guide: https://support.discord.com/hc/en-us/articles/228383668-Intro-to-Webhooks |
| The workflow filters out posts by a specific moderator account ID (author_fullname "t2_6l4z3"); update as needed. | The ID corresponds to Reddit user; change if subreddit mods differ.                             |
| This workflow uses Reddit’s media_metadata JSON which may change if Reddit updates their API format.          | Monitor Reddit API changes to maintain URL extraction logic.                                     |
| Rate limits on Reddit API and Discord webhook calls may apply; consider handling retries or throttling.      | n8n documentation on error handling and rate limiting: https://docs.n8n.io/integrations/builtin/core-nodes/error-handling/ |

---

**Disclaimer:**  
The provided content is exclusively from an automated workflow created with n8n, an integration and automation tool. It complies strictly with current content policies and contains no illegal, offensive, or protected material. All data processed is legal and publicly accessible.