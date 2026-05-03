Automatically Import your Meta Threads Posts into Notion

https://n8nworkflows.xyz/workflows/automatically-import-your-meta-threads-posts-into-notion-2802


# Automatically Import your Meta Threads Posts into Notion

### 1. Workflow Overview

This workflow automates the retrieval of Meta Threads posts and imports them into a Notion database for archival, analysis, and AI training purposes. It is designed for social media managers, content creators, and data analysts who want to maintain an organized, searchable record of their Threads content.

The workflow is logically divided into the following blocks:

- **1.1 Authentication & Token Management:** Handles obtaining and refreshing the Meta Threads API access token.
- **1.2 Post Retrieval & Filtering:** Fetches posts from the Meta Threads API, filters relevant posts, and extracts key data.
- **1.3 Data Processing & Deduplication:** Processes posts, compares with existing Notion entries to avoid duplicates.
- **1.4 Notion Integration & Page Creation:** Creates or updates pages in Notion with post content and media embeds.
- **1.5 Scheduling & Control Flow:** Automates periodic execution and controls workflow branching based on data conditions.

---

### 2. Block-by-Block Analysis

#### 2.1 Authentication & Token Management

**Overview:**  
This block obtains a long-lived access token for the Meta Threads API and refreshes it as needed to maintain uninterrupted access.

**Nodes Involved:**  
- Get Posts Schedule  
- Refresh Token  
- Run This First to Get Long Live Access Token  

**Node Details:**  

- **Get Posts Schedule**  
  - *Type:* Schedule Trigger  
  - *Role:* Triggers the workflow at regular intervals (default: every minute) to automate post retrieval.  
  - *Configuration:* Interval trigger with default settings.  
  - *Inputs:* None  
  - *Outputs:* Triggers "Refresh Token" node.  
  - *Edge Cases:* If workflow is paused or schedule misconfigured, posts won't be fetched automatically.

- **Refresh Token**  
  - *Type:* HTTP Request  
  - *Role:* Refreshes the long-lived access token using the existing token.  
  - *Configuration:*  
    - URL: `https://graph.threads.net/refresh_access_token`  
    - Query Parameters:  
      - `grant_type=th_refresh_token`  
      - `access_token=Your Threads Long-Live Token` (to be replaced with actual token)  
  - *Inputs:* Trigger from schedule node.  
  - *Outputs:* Provides refreshed access token JSON for downstream nodes.  
  - *Edge Cases:* Token expiration, invalid token, or network errors may cause failure.

- **Run This First to Get Long Live Access Token**  
  - *Type:* HTTP Request  
  - *Role:* One-time node to exchange a short-lived token for a long-lived token.  
  - *Configuration:*  
    - URL: `https://graph.threads.net/access_token`  
    - Query Parameters:  
      - `grant_type=th_exchange_token`  
      - `client_secret=Threads App Secret` (replace with actual secret)  
      - `access_token=Short Live Access Token` (replace with actual token)  
  - *Inputs:* Manual trigger only.  
  - *Outputs:* Returns long-lived access token for use in "Refresh Token".  
  - *Edge Cases:* Invalid client secret or short-lived token will cause failure.

---

#### 2.2 Post Retrieval & Filtering

**Overview:**  
This block fetches posts from the Meta Threads API for a specified user and date range, filters posts by media type, and extracts relevant fields.

**Nodes Involved:**  
- Get Post  
- Get Post ID  
- Loop Over Items  
- Threads / Root  
- Threads / Comments  
- Root's Filter  
- Comment's Filter  
- Merge  

**Node Details:**  

- **Get Post**  
  - *Type:* HTTP Request  
  - *Role:* Retrieves posts from the Meta Threads API for the specified Threads ID and date range.  
  - *Configuration:*  
    - URL: `https://graph.threads.net/v1.0/<Your Threads ID>/threads`  
    - Query Parameters:  
      - `fields`: id, media_product_type, media_type, media_url, permalink, owner, username, text, timestamp, shortcode, thumbnail_url, children, is_quote_post  
      - `since`: Dynamic expression for date (default: one day ago)  
      - `access_token`: Uses refreshed token from "Refresh Token" node  
  - *Inputs:* Access token from "Refresh Token".  
  - *Outputs:* JSON data containing posts.  
  - *Edge Cases:* Invalid Threads ID, expired token, or API rate limits.

- **Get Post ID**  
  - *Type:* Function  
  - *Role:* Filters posts by media type (TEXT_POST, IMAGE, VIDEO, CAROUSEL_ALBUM, AUDIO) and extracts id, permalink, and timestamp.  
  - *Configuration:* Custom JavaScript filtering and mapping.  
  - *Inputs:* Output from "Get Post".  
  - *Outputs:* Array of filtered post objects with selected fields.  
  - *Edge Cases:* Empty or malformed API response.

- **Loop Over Items**  
  - *Type:* Split In Batches  
  - *Role:* Processes posts in batches to avoid API or memory overload.  
  - *Configuration:* Default batch size.  
  - *Inputs:* Filtered posts from "Get Post ID".  
  - *Outputs:* Batches to "Threads / Root" and "Threads / Comments".  
  - *Edge Cases:* Large datasets may require batch size tuning.

- **Threads / Root**  
  - *Type:* HTTP Request  
  - *Role:* Fetches detailed data for each post ID including media and children posts.  
  - *Configuration:*  
    - URL: `https://graph.threads.net/v1.0/{{ $json.id }}`  
    - Fields: id, media_product_type, media_type, media_url, children{media_url}, permalink, owner, username, text, timestamp, children  
    - Access token from "Refresh Token".  
  - *Inputs:* Post IDs from "Loop Over Items".  
  - *Outputs:* Detailed post JSON.  
  - *Edge Cases:* Missing posts or API errors.

- **Threads / Comments**  
  - *Type:* HTTP Request  
  - *Role:* Retrieves conversation/comments related to each post.  
  - *Configuration:*  
    - URL: `https://graph.threads.net/v1.0/{{ $json.id }}/conversation`  
    - Fields: id, text, username, permalink, timestamp, media_product_type, media_type, media_url, children{media_url}  
    - Access token from "Refresh Token".  
  - *Inputs:* Post IDs from "Loop Over Items".  
  - *Outputs:* Comments JSON.  
  - *Edge Cases:* Posts without comments or API failures.

- **Root's Filter**  
  - *Type:* Code  
  - *Role:* Processes root post data, extracts media URLs, and normalizes structure.  
  - *Configuration:* JavaScript code to extract media URLs from children or main post.  
  - *Inputs:* Output from "Threads / Root".  
  - *Outputs:* Normalized post objects with id, username, text, timestamp, media_type, media_urls, permalink.  
  - *Edge Cases:* Posts without media or children.

- **Comment's Filter**  
  - *Type:* Code  
  - *Role:* Filters comments by username (hardcoded as 'geekaz'), extracts text and media URLs.  
  - *Configuration:* JavaScript code with username filter and media extraction.  
  - *Inputs:* Output from "Threads / Comments".  
  - *Outputs:* Filtered comment objects with text and media_urls.  
  - *Edge Cases:* No comments or username mismatch.

- **Merge**  
  - *Type:* Merge  
  - *Role:* Combines filtered root posts and comments into a single data stream.  
  - *Inputs:* Outputs from "Root's Filter" and "Comment's Filter".  
  - *Outputs:* Combined post data for further processing.  
  - *Edge Cases:* Mismatched data lengths or empty inputs.

---

#### 2.3 Data Processing & Deduplication

**Overview:**  
This block checks if posts already exist in the Notion database to prevent duplicate imports.

**Nodes Involved:**  
- Threads Post  
- Threads ID  
- Extract IDs  
- Compare IDs  
- If Post Exist  

**Node Details:**  

- **Threads Post**  
  - *Type:* Code  
  - *Role:* Constructs Notion-compatible blocks from post text and media URLs for page creation.  
  - *Configuration:* JavaScript code that creates paragraph blocks for text and embed blocks for media URLs.  
  - *Inputs:* Merged post data.  
  - *Outputs:* Post object with id, permalink, username, timestamp, and blocks array.  
  - *Edge Cases:* Posts without text or media.

- **Threads ID**  
  - *Type:* Notion  
  - *Role:* Retrieves all existing pages from the Notion database to get current Threads IDs.  
  - *Configuration:*  
    - Operation: Get All database pages  
    - Database ID: User-specified Notion database  
    - Credential: Notion API OAuth2  
  - *Inputs:* Output from "Threads Post".  
  - *Outputs:* List of existing Notion pages.  
  - *Edge Cases:* API rate limits, invalid credentials.

- **Extract IDs**  
  - *Type:* Function  
  - *Role:* Extracts Threads IDs from existing Notion pages to build a list of imported post IDs.  
  - *Configuration:* JavaScript code parsing Notion page properties.  
  - *Inputs:* Output from "Threads ID".  
  - *Outputs:* JSON object with array of existing Threads IDs.  
  - *Edge Cases:* Missing or malformed properties.

- **Compare IDs**  
  - *Type:* Function  
  - *Role:* Checks if the current post ID already exists in Notion to avoid duplicates.  
  - *Configuration:* Compares new post ID with extracted IDs list.  
  - *Inputs:* Current post from "Threads Post" and existing IDs from "Extract IDs".  
  - *Outputs:* Boolean flag `isExist`.  
  - *Edge Cases:* Empty existing IDs list.

- **If Post Exist**  
  - *Type:* Switch  
  - *Role:* Routes workflow based on whether the post already exists in Notion.  
  - *Configuration:*  
    - If `isExist` is false → proceed to create page  
    - If `isExist` is true → skip creation (go to "Replace Me" node)  
  - *Inputs:* Output from "Compare IDs".  
  - *Outputs:* Two branches: "Create Page" and "Replace Me".  
  - *Edge Cases:* Incorrect boolean evaluation.

---

#### 2.4 Notion Integration & Page Creation

**Overview:**  
This block creates new pages in the Notion database for new posts and uploads associated media blocks.

**Nodes Involved:**  
- Create Page  
- Upload Medias  
- Replace Me  

**Node Details:**  

- **Create Page**  
  - *Type:* Notion  
  - *Role:* Creates a new page in the specified Notion database with post metadata.  
  - *Configuration:*  
    - Database ID: User's Notion database  
    - Properties:  
      - Name (title): post permalink URL  
      - Threads ID (rich text): post ID  
      - Post Date (date): post timestamp (timezone America/Los_Angeles, no time)  
      - Source (multi-select): fixed value "Threads"  
      - Import Check (checkbox): unchecked by default  
      - Username (rich text): post author username  
    - Credential: Notion API OAuth2  
  - *Inputs:* Posts flagged as new from "If Post Exist".  
  - *Outputs:* Created page data including page ID.  
  - *Edge Cases:* Invalid database ID, missing properties, API errors.

- **Upload Medias**  
  - *Type:* HTTP Request  
  - *Role:* Uploads media blocks (images, videos, embeds) as children blocks under the newly created Notion page.  
  - *Configuration:*  
    - URL: `https://api.notion.com/v1/blocks/{{ pageId }}/children`  
    - Method: PATCH  
    - Body: JSON containing blocks from "Threads Post" node  
    - Headers: Authorization with bearer token, Content-Type application/json, Notion API version 2022-06-28  
  - *Inputs:* Page ID from "Create Page" and blocks from "Threads Post".  
  - *Outputs:* Confirmation of media upload.  
  - *Edge Cases:* Invalid page ID, token expiration, API limits.

- **Replace Me**  
  - *Type:* No Operation (NoOp)  
  - *Role:* Placeholder node for posts that already exist; effectively skips further processing.  
  - *Inputs:* Posts flagged as existing from "If Post Exist".  
  - *Outputs:* None (ends branch).  
  - *Edge Cases:* None.

---

#### 2.5 Scheduling & Control Flow

**Overview:**  
This block manages the overall execution flow, triggering the workflow periodically and controlling data flow between nodes.

**Nodes Involved:**  
- Get Posts Schedule  
- Refresh Token  
- Get Post  
- Get Post ID  
- Loop Over Items  
- Threads / Root  
- Threads / Comments  
- Root's Filter  
- Comment's Filter  
- Merge  
- Threads Post  
- Threads ID  
- Extract IDs  
- Compare IDs  
- If Post Exist  
- Create Page  
- Upload Medias  
- Replace Me  

**Node Details:**  
This block is essentially the orchestration of all other blocks, ensuring data flows correctly from token refresh through post retrieval, filtering, deduplication, and Notion page creation. The schedule trigger initiates the process, and the switch node "If Post Exist" controls branching.

---

### 3. Summary Table

| Node Name                      | Node Type           | Functional Role                                   | Input Node(s)               | Output Node(s)              | Sticky Note                                                                                                  |
|--------------------------------|---------------------|--------------------------------------------------|-----------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------------|
| Get Posts Schedule             | Schedule Trigger    | Periodically triggers workflow                    | None                        | Refresh Token               |                                                                                                              |
| Refresh Token                 | HTTP Request        | Refreshes Meta Threads API access token           | Get Posts Schedule          | Get Post                   | ## Refresh Token Update your long live token here / 在此放上剛剛拿到的長期 Token [Check Facebook Docs Refresh Token](https://developers.facebook.com/docs/threads/get-started/long-lived-tokens/) |
| Get Post                     | HTTP Request        | Retrieves posts from Meta Threads API              | Refresh Token               | Get Post ID                | ## Set your Theads ID & Post Time Chage the your with your Threads ID to get your posts / 請先透過上方教學獲取 Threads ID Set the time of the Post you wanna get / 設置抓取的貼文時間 [Check Facebook Docs Post API](https://developers.facebook.com/docs/threads/threads-media) since is scrape the post after the date / since 為抓取日期之後的貼文 until is scrape the post before the date / until 為抓取日期之前的貼文 since can set {{ new Date(new Date().setDate(new Date().getDate() - 1)).toISOString().split('T')[0] }} it will scrape the post since one day ago |
| Get Post ID                  | Function            | Filters posts by media type and extracts key fields | Get Post                   | Loop Over Items            |                                                                                                              |
| Loop Over Items              | Split In Batches    | Processes posts in batches                         | Get Post ID                 | Threads / Root, Threads / Comments |                                                                                                              |
| Threads / Root               | HTTP Request        | Fetches detailed post data                         | Loop Over Items             | Root's Filter              |                                                                                                              |
| Threads / Comments           | HTTP Request        | Fetches comments/conversation for posts           | Loop Over Items             | Comment's Filter           | ## Comment's Filter Set your Threads Username                                                                 |
| Root's Filter                | Code                | Normalizes root post data and extracts media URLs | Threads / Root              | Merge                      |                                                                                                              |
| Comment's Filter             | Code                | Filters comments by username and extracts media   | Threads / Comments          | Merge                      |                                                                                                              |
| Merge                       | Merge               | Combines root posts and comments                   | Root's Filter, Comment's Filter | Threads Post             |                                                                                                              |
| Threads Post                | Code                | Creates Notion blocks for post content and media  | Merge                       | Threads ID                 |                                                                                                              |
| Threads ID                  | Notion              | Retrieves existing Notion pages                    | Threads Post                | Extract IDs                | ## Set Notion Acc Set your notion account and database you wanna save the post                               |
| Extract IDs                 | Function            | Extracts Threads IDs from Notion pages             | Threads ID                  | Compare IDs                |                                                                                                              |
| Compare IDs                 | Function            | Checks if post already exists in Notion            | Extract IDs                 | If Post Exist              |                                                                                                              |
| If Post Exist               | Switch              | Routes based on post existence                      | Compare IDs                 | Create Page, Replace Me    |                                                                                                              |
| Create Page                 | Notion              | Creates new page in Notion database                 | If Post Exist (Create Page) | Upload Medias              | ## Create Page Before create page, please the properties of your post by your demands                        |
| Upload Medias               | HTTP Request        | Uploads media blocks to Notion page                 | Create Page                 | Replace Me                 | ## Support Medias It also can scrape the Threads Media like Images and Videos Update your Notion token here: bearer <your notion token> |
| Replace Me                 | NoOp                | Placeholder for existing posts                      | If Post Exist (Update Page) | None                      |                                                                                                              |
| Run This First to Get Long Live Access Token | HTTP Request | One-time exchange of short-lived token for long-lived token | Manual trigger             | None                      | ## Get Threads API Access Token Get Threads Access Token Tutorial and ID/教學 [Link](https://nijialin.com/2024/08/17/python-threads-sdk-introduction/) Please get your access token and Threads ID first before you start (It only need to run once) |
| Sticky Note (multiple)      | Sticky Note         | Various instructional notes                         | None                        | None                      | See individual notes in Sticky Note column above                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Configure to run at desired interval (e.g., every minute).

2. **Create HTTP Request Node "Refresh Token"**  
   - URL: `https://graph.threads.net/refresh_access_token`  
   - Query Parameters:  
     - `grant_type=th_refresh_token`  
     - `access_token=Your Threads Long-Live Token` (replace with actual token)  
   - Connect Schedule Trigger → Refresh Token.

3. **Create HTTP Request Node "Get Post"**  
   - URL: `https://graph.threads.net/v1.0/<Your Threads ID>/threads`  
   - Query Parameters:  
     - `fields=id,media_product_type,media_type,media_url,permalink,owner,username,text,timestamp,shortcode,thumbnail_url,children,is_quote_post`  
     - `since={{ new Date(new Date().setDate(new Date().getDate() - 1)).toISOString().split('T')[0] }}` (adjust as needed)  
     - `access_token={{ $json.access_token }}` (from Refresh Token)  
   - Connect Refresh Token → Get Post.

4. **Create Function Node "Get Post ID"**  
   - JavaScript code to filter posts by media_type (TEXT_POST, IMAGE, VIDEO, CAROUSEL_ALBUM, AUDIO) and extract id, permalink, timestamp.  
   - Connect Get Post → Get Post ID.

5. **Create Split In Batches Node "Loop Over Items"**  
   - Default batch size.  
   - Connect Get Post ID → Loop Over Items.

6. **Create HTTP Request Node "Threads / Root"**  
   - URL: `https://graph.threads.net/v1.0/{{ $json.id }}`  
   - Query Parameters:  
     - `fields=id,media_product_type,media_type,media_url,children{media_url},permalink,owner,username,text,timestamp,children`  
     - `access_token={{ $('Refresh Token').first().json.access_token }}`  
   - Connect Loop Over Items (first output) → Threads / Root.

7. **Create HTTP Request Node "Threads / Comments"**  
   - URL: `https://graph.threads.net/v1.0/{{ $json.id }}/conversation`  
   - Query Parameters:  
     - `fields=id,text,username,permalink,timestamp,media_product_type,media_type,media_url,children{media_url}`  
     - `access_token={{ $('Refresh Token').first().json.access_token }}`  
   - Connect Loop Over Items (second output) → Threads / Comments.

8. **Create Code Node "Root's Filter"**  
   - JavaScript to extract media URLs from children or main post and normalize data structure.  
   - Connect Threads / Root → Root's Filter.

9. **Create Code Node "Comment's Filter"**  
   - JavaScript to filter comments by username (replace 'geekaz' with your username) and extract text and media URLs.  
   - Connect Threads / Comments → Comment's Filter.

10. **Create Merge Node "Merge"**  
    - Mode: Merge by data.  
    - Connect Root's Filter and Comment's Filter → Merge.

11. **Create Code Node "Threads Post"**  
    - JavaScript to create Notion blocks for text and media embeds from merged posts.  
    - Connect Merge → Threads Post.

12. **Create Notion Node "Threads ID"**  
    - Operation: Get All database pages  
    - Database ID: Your Notion database ID  
    - Credential: Your Notion API OAuth2 credentials  
    - Connect Threads Post → Threads ID.

13. **Create Function Node "Extract IDs"**  
    - JavaScript to extract Threads IDs from Notion pages.  
    - Connect Threads ID → Extract IDs.

14. **Create Function Node "Compare IDs"**  
    - JavaScript to check if current post ID exists in extracted IDs.  
    - Connect Extract IDs → Compare IDs.

15. **Create Switch Node "If Post Exist"**  
    - Condition: If `isExist` is false → output "Create Page" branch; if true → output "Replace Me" branch.  
    - Connect Compare IDs → If Post Exist.

16. **Create Notion Node "Create Page"**  
    - Operation: Create database page  
    - Database ID: Your Notion database ID  
    - Properties:  
      - Name (title): post permalink  
      - Threads ID (rich text): post ID  
      - Post Date (date): post timestamp (timezone America/Los_Angeles, no time)  
      - Source (multi-select): "Threads"  
      - Import Check (checkbox): unchecked  
      - Username (rich text): post username  
    - Credential: Notion API OAuth2  
    - Connect If Post Exist (Create Page output) → Create Page.

17. **Create HTTP Request Node "Upload Medias"**  
    - URL: `https://api.notion.com/v1/blocks/{{ $('Create Page').item.json.id }}/children`  
    - Method: PATCH  
    - Body (JSON): `{ "children": $('Threads Post').last().json.blocks }`  
    - Headers:  
      - Authorization: `bearer Your Notion Token`  
      - Content-Type: `application/json`  
      - Notion-Version: `2022-06-28`  
    - Connect Create Page → Upload Medias.

18. **Create NoOp Node "Replace Me"**  
    - Placeholder node for existing posts to end branch.  
    - Connect If Post Exist (Update Page output) → Replace Me.  
    - Connect Upload Medias → Replace Me.

19. **Add Sticky Notes**  
    - Add instructional sticky notes at relevant points for token setup, API docs, Notion setup, media support, and author contact.

20. **Credential Setup**  
    - Configure credentials for Notion API (OAuth2) and ensure you have your Threads long-lived token and Threads ID ready.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Get Threads API Access Token Tutorial and ID/教學                                                    | [https://nijialin.com/2024/08/17/python-threads-sdk-introduction/](https://nijialin.com/2024/08/17/python-threads-sdk-introduction/) |
| Refresh Token Documentation                                                                         | [https://developers.facebook.com/docs/threads/get-started/long-lived-tokens/](https://developers.facebook.com/docs/threads/get-started/long-lived-tokens/) |
| Meta Threads Post API Documentation                                                                 | [https://developers.facebook.com/docs/threads/threads-media](https://developers.facebook.com/docs/threads/threads-media) |
| Creator Contact for Support                                                                         | Instagram: [https://www.threads.net/@geekaz?hl=zh-tw](https://www.threads.net/@geekaz?hl=zh-tw)      |
| Notion API Version Used                                                                             | 2022-06-28                                                                                         |
| Workflow Creator                                                                                   | Geekaz                                                                                            |

---

This documentation provides a comprehensive understanding of the workflow’s structure, logic, and configuration, enabling advanced users and AI agents to reproduce, modify, and troubleshoot the workflow effectively.