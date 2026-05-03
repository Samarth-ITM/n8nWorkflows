WordPress Blog to Google Sheets Sync (Posts, Categories, Tags, Media)

https://n8nworkflows.xyz/workflows/wordpress-blog-to-google-sheets-sync--posts--categories--tags--media--9064


# WordPress Blog to Google Sheets Sync (Posts, Categories, Tags, Media)

### 1. Workflow Overview

This workflow, titled **"WordPress Blog to Google Sheets Sync (Posts, Categories, Tags, Media)"**, is designed to synchronize various elements of a WordPress site (posts, categories, tags, and media) into a Google Sheets spreadsheet. It supports both scheduled automatic runs and manual triggering via webhook.

The workflow is logically divided into these main blocks:

- **1.1 Triggering (Input Reception):** Supports both scheduled and webhook triggers to start the sync process.
- **1.2 Posts Sync Loop:** Handles fetching WordPress posts in a paginated manner, splitting and updating them into the "Posts" Google Sheets tab.
- **1.3 Categories Sync:** Fetches all categories once and updates the "Categories" Google Sheets tab.
- **1.4 Tags Sync:** Fetches all tags once and updates the "Tags" Google Sheets tab.
- **1.5 Media Sync Loop:** Handles fetching WordPress media library items in a paginated manner, splitting and updating them into the "Media" Google Sheets tab.

Each syncing block uses pagination where needed to ensure complete data retrieval from the WordPress REST API, and updates the corresponding Google Sheets tabs using append-or-update operations keyed by unique IDs.

---

### 2. Block-by-Block Analysis

#### 2.1 Triggering (Input Reception)

**Overview:**  
Starts the workflow either by a scheduled trigger that runs periodically or by a manual webhook trigger to refresh data on demand.

**Nodes Involved:**  
- Schedule Trigger  
- Webhook

**Node Details:**

- **Schedule Trigger**  
  - Type: `ScheduleTrigger`  
  - Runs at a default interval (configured as every minute or as set) to initiate sync automatically.  
  - No input; outputs trigger data to initialize variables.  
  - Potential failures: misconfigured schedule, workflow disabled.

- **Webhook**  
  - Type: `Webhook`  
  - Allows manual triggering by external HTTP request at the path `6071c43c-0641-4049-8162-1d8fdd1b088a`.  
  - No authentication configured, so open endpoint (consider securing in production).  
  - Outputs to initialize variables.  
  - Possible edge case: unauthorized or malformed requests.

---

#### 2.2 Posts Sync Loop

**Overview:**  
Fetches posts from the WordPress REST API in batches (pagination), splits the response to individual posts, and updates or appends each post into the Google Sheets "Posts" tab. It loops through pages until all posts are processed.

**Nodes Involved:**  
- Variables (initial pagination variables for posts)  
- Loop Vars (increment pagination)  
- Get Posts (HTTP request to WP REST API)  
- Split Out (split posts array)  
- Update Posts (Google Sheets appendOrUpdate)  
- If (pagination check to continue looping)

**Node Details:**

- **Variables**  
  - Type: `Set`  
  - Initializes pagination variables: base URL to posts endpoint, fields requested, per_page count (10), and current_page (0).  
  - Used as starting point for pagination loop.

- **Loop Vars**  
  - Type: `Code`  
  - Increments the current page number for pagination.  
  - Tries to read previous iteration's vars; if none, uses initial Variables node.  
  - Returns updated current_page.

- **Get Posts**  
  - Type: `HttpRequest`  
  - Makes authenticated Basic Auth HTTP GET request to WordPress posts endpoint using tokens from credentials.  
  - Requests specific fields and embeds media and taxonomy terms.  
  - Query parameters include per_page, page (current_page), order by date descending.  
  - Configured to retrieve full HTTP response to access headers (for total pages).  
  - Failure modes: network issues, authentication failure, API rate limiting, malformed responses.

- **Split Out**  
  - Type: `SplitOut`  
  - Splits the array of posts in response body into individual items for separate processing.

- **Update Posts**  
  - Type: `GoogleSheets`  
  - Append or update operation on "Posts" sheet/tab.  
  - Maps post fields (id, date, link, slug, tags, title, excerpt, categories, featured_media) to columns.  
  - Uses "id" as matching column for update.  
  - Requires Google Sheets OAuth2 credentials.  
  - Edge cases: sheet access issues, data type mismatches, API limits.

- **If**  
  - Type: `If`  
  - Checks if current page number >= total pages from the HTTP response header `x-wp-totalpages`.  
  - If false, loops back to Loop Vars to get next page; if true, ends posts pagination loop and triggers categories and media sync blocks.  
  - Edge cases: missing headers, type conversion errors.

---

#### 2.3 Categories Sync

**Overview:**  
Fetches all categories from WordPress once and updates the "Categories" sheet.

**Nodes Involved:**  
- Get Categories (HTTP request)  
- Update Categories (Google Sheets appendOrUpdate)

**Node Details:**

- **Get Categories**  
  - Type: `HttpRequest`  
  - Authenticated Basic Auth GET request to WordPress categories endpoint.  
  - Runs once per workflow execution (executeOnce flag).  
  - Retrieves all category details.  
  - Failures: authentication errors, API errors.

- **Update Categories**  
  - Type: `GoogleSheets`  
  - Append or update on "Categories" Google Sheet tab.  
  - Maps category id, slug, title, and description.  
  - Uses "category_id" as matching key.  
  - Requires Google Sheets OAuth2 credentials.

---

#### 2.4 Tags Sync

**Overview:**  
Fetches all tags from WordPress once and updates the "Tags" sheet.

**Nodes Involved:**  
- HTTP Request (to get tags)  
- Split Out2 (split tags array)  
- Append or update row in sheet (Google Sheets appendOrUpdate)

**Node Details:**

- **HTTP Request**  
  - Type: `HttpRequest`  
  - Basic Auth GET request to WordPress tags endpoint.  
  - Requests 100 tags per page and fields limited to id, name, slug.  
  - Runs once per execution.  
  - Failure modes similar to other HTTP requests.

- **Split Out2**  
  - Type: `SplitOut`  
  - Splits tags array into individual items.

- **Append or update row in sheet**  
  - Type: `GoogleSheets`  
  - Append or update on "Tags" Google Sheet tab.  
  - Maps tag_id, name, slug fields.  
  - Uses "tag_id" as matching column.  
  - Requires Google Sheets OAuth2 credentials.

---

#### 2.5 Media Sync Loop

**Overview:**  
Fetches media library items from WordPress with pagination, splits each media item, and updates the "Media" Google Sheets tab. Loops until all media pages are processed.

**Nodes Involved:**  
- Variables2 (initial pagination variables for media)  
- Loop Vars2 (increment pagination)  
- Get Media (HTTP request to WP REST API media endpoint)  
- Split Out1 (split media array)  
- Update Media (Google Sheets appendOrUpdate)  
- If1 (pagination check to continue looping)

**Node Details:**

- **Variables2**  
  - Type: `Set`  
  - Initializes base_url for media endpoint, per_page count (10), and current_page (0).  
  - Starting point for pagination loop.

- **Loop Vars2**  
  - Type: `Code`  
  - Increments current_page number.  
  - Reads previous iteration vars or initial Variables2 node.  
  - Returns updated current_page.

- **Get Media**  
  - Type: `HttpRequest`  
  - Basic Auth GET request to media endpoint with pagination parameters.  
  - Excludes media items with parent 0 (likely unattached media).  
  - Requests full HTTP response for headers.  
  - Potential failures: authentication, network, API limits.

- **Split Out1**  
  - Type: `SplitOut`  
  - Splits media array into individual media items.

- **Update Media**  
  - Type: `GoogleSheets`  
  - Append or update on "Media" sheet tab.  
  - Maps media_id, post_id, alt_text, source_url, slug, title, and dimensions (width, height).  
  - Uses "media_id" as matching column.  
  - Requires Google Sheets OAuth2 credentials.

- **If1**  
  - Type: `If`  
  - Checks if current_page >= total pages from response header `x-wp-totalpages`.  
  - Loops back to Loop Vars2 if more pages remain or ends if done.  
  - Edge cases: missing headers, pagination logic errors.

---

### 3. Summary Table

| Node Name                    | Node Type            | Functional Role                          | Input Node(s)         | Output Node(s)                  | Sticky Note                                                                                 |
|------------------------------|----------------------|----------------------------------------|-----------------------|-------------------------------|---------------------------------------------------------------------------------------------|
| Schedule Trigger              | ScheduleTrigger      | Periodic workflow starter               |                       | Variables                     | Two ways to start<br>Schedule runs daily, webhook for manual refresh.                       |
| Webhook                      | Webhook              | Manual workflow starter                 |                       | Variables                     | Two ways to start<br>Schedule runs daily, webhook for manual refresh.                       |
| Variables                    | Set                  | Initialize posts pagination variables   | Webhook, Schedule     | Loop Vars                     | Initialise pagination variables for REST API requests.                                     |
| Loop Vars                   | Code                 | Increment posts pagination page number  | Variables, If          | Get Posts                    | Check if we’ve reached the last page of posts. If not, continue loop.                       |
| Get Posts                   | HttpRequest          | Fetch paginated posts from WordPress    | Loop Vars              | Split Out                    | Fetch posts from WP, split them, and upsert into 'Posts' sheet.                            |
| Split Out                   | SplitOut             | Split posts array into single items     | Get Posts              | Update Posts                 | Fetch posts from WP, split them, and upsert into 'Posts' sheet.                            |
| Update Posts                | GoogleSheets         | Upsert posts into Google Sheets         | Split Out              | If                          | Check if we’ve reached the last page of posts. If not, continue loop.                       |
| If                          | If                   | Pagination check for posts loop          | Update Posts           | Get Categories, Variables2, HTTP Request, Loop Vars | Check if we’ve reached the last page of posts. If not, continue loop.                       |
| Get Categories              | HttpRequest          | Fetch all categories once                 | If                     | Update Categories            | Sync categories into 'Categories' sheet                                                    |
| Update Categories           | GoogleSheets         | Upsert categories into Google Sheets     | Get Categories         |                             | Sync categories into 'Categories' sheet                                                    |
| HTTP Request (tags)          | HttpRequest          | Fetch all tags once                       | If                     | Split Out2                  | Fetch tags and update 'Tags' sheet.                                                       |
| Split Out2                  | SplitOut             | Split tags array into single items        | HTTP Request (tags)    | Append or update row in sheet | Fetch tags and update 'Tags' sheet.                                                       |
| Append or update row in sheet| GoogleSheets         | Upsert tags into Google Sheets            | Split Out2             |                             | Fetch tags and update 'Tags' sheet.                                                       |
| Variables2                  | Set                  | Initialize media pagination variables     | Get Categories          | Loop Vars2                   | Fetch media library (with pagination) and sync into 'Media' sheet.                         |
| Loop Vars2                  | Code                 | Increment media pagination page number    | Variables2, If1         | Get Media                   | Stop or continue looping until all media pages are processed.                             |
| Get Media                   | HttpRequest          | Fetch paginated media items from WordPress | Loop Vars2             | Split Out1                  | Fetch media library (with pagination) and sync into 'Media' sheet.                         |
| Split Out1                  | SplitOut             | Split media array into single items        | Get Media               | Update Media                | Fetch media library (with pagination) and sync into 'Media' sheet.                         |
| Update Media                | GoogleSheets         | Upsert media items into Google Sheets       | Split Out1              | If1                         | Stop or continue looping until all media pages are processed.                             |
| If1                         | If                   | Pagination check for media loop             | Update Media            | Loop Vars2                   | Stop or continue looping until all media pages are processed.                             |
| Sticky Note                 | StickyNote           | Instructional note                        |                       |                             | Two ways to start<br>Schedule runs daily, webhook for manual refresh.                       |
| Sticky Note1                | StickyNote           | Instructional note                        |                       |                             | Initialise pagination variables for REST API requests.                                     |
| Sticky Note2                | StickyNote           | Instructional note                        |                       |                             | Fetch posts from WP, split them, and upsert into 'Posts' sheet.                            |
| Sticky Note3                | StickyNote           | Instructional note                        |                       |                             | Check if we’ve reached the last page of posts. If not, continue loop.                       |
| Sticky Note4                | StickyNote           | Instructional note                        |                       |                             | Sync categories into 'Categories' sheet                                                    |
| Sticky Note5                | StickyNote           | Instructional note                        |                       |                             | Fetch tags and update 'Tags' sheet.                                                       |
| Sticky Note6                | StickyNote           | Instructional note                        |                       |                             | Fetch media library (with pagination) and sync into 'Media' sheet.                         |
| Sticky Note7                | StickyNote           | Instructional note                        |                       |                             | Stop or continue looping until all media pages are processed.                             |
| Sticky Note8                | StickyNote           | Video guide reference                      |                       |                             | Video Guide<br>@[youtube](czSMWyD6f-0)                                                    |
| Sticky Note9                | StickyNote           | Google Sheets template link                |                       |                             | Google Sheets Template<br>https://docs.google.com/spreadsheets/d/1i4h5kybiYLTT0wTzXbWN7rxp-DV6dOyNcwqgg8I5Rv4/edit?usp=sharing |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**

   - **Schedule Trigger**  
     - Type: ScheduleTrigger  
     - Configure interval as required (e.g., every day or every hour).  
     - No credentials needed.

   - **Webhook**  
     - Type: Webhook  
     - Set path to a unique identifier (e.g., `6071c43c-0641-4049-8162-1d8fdd1b088a`).  
     - No authentication configured (consider securing it).  
     - No credentials needed.

2. **Set Up Posts Pagination Variables:**

   - **Set Node ("Variables")**  
     - Create a Set node to initialize:  
       - `base_url`: `https://ozwebexpert.com/wp-json/wp/v2/posts`  
       - `fields`: `id,slug,link,date,title,excerpt,categories,tags,featured_media,_embedded`  
       - `per_page`: `10`  
       - `current_page`: `0`  
     - No credentials needed.

3. **Posts Pagination Loop:**

   - **Code Node ("Loop Vars")**  
     - JavaScript to increment `current_page` by 1 each iteration, reading from previous iteration or initial Variables node.

   - **HTTP Request Node ("Get Posts")**  
     - Method: GET  
     - URL: Use `base_url` from variables.  
     - Query parameters:  
       - `per_page`: from JSON variable  
       - `page`: current_page variable  
       - `_embed`: `wp:featuredmedia,wp:term`  
       - `_fields`: from variables  
       - `orderby`: `date`  
       - `order`: `desc`  
     - Authentication: HTTP Basic with WordPress token (create HTTP Basic Auth credentials).  
     - Response format: Full response JSON including headers.

   - **SplitOut Node ("Split Out")**  
     - Field to split: `body` (array of posts).

   - **Google Sheets Node ("Update Posts")**  
     - Operation: Append or Update  
     - Document ID: Google Sheets document ID  
     - Sheet name: "Posts" (gid=0)  
     - Mapping columns: `id` (matching), and fields mapped from post JSON: date, link, slug, tags, title (rendered), excerpt (rendered), categories (first category), featured_media.  
     - Authentication: Google Sheets OAuth2 credentials.

   - **If Node ("If")**  
     - Condition: Current page number >= `x-wp-totalpages` header from Get Posts response.  
     - If false: connect back to Loop Vars for next page.  
     - If true: continue to Categories Sync block.

4. **Categories Sync:**

   - **HTTP Request Node ("Get Categories")**  
     - GET request to `https://ozwebexpert.com/wp-json/wp/v2/categories`.  
     - Authentication: HTTP Basic Auth WordPress token.  
     - Execute once per workflow run.

   - **Google Sheets Node ("Update Categories")**  
     - Operation: Append or Update  
     - Document ID: same Google Sheets ID  
     - Sheet name: "Categories" (gid=1002593339)  
     - Map fields: `category_id` (id), `title` (name), `description`, `slug`.  
     - Matching column: `category_id`.  
     - Authentication: Google Sheets OAuth2.

5. **Tags Sync:**

   - **HTTP Request Node ("HTTP Request")**  
     - GET request to `https://ozwebexpert.com/wp-json/wp/v2/tags`.  
     - Query parameters: `per_page=100`, `_fields=id,name,slug`.  
     - Authentication: HTTP Basic Auth WordPress token.  
     - Execute once.

   - **SplitOut Node ("Split Out2")**  
     - Split field: `body` (tags array).

   - **Google Sheets Node ("Append or update row in sheet")**  
     - Append or Update operation on "Tags" sheet (gid=1137750815).  
     - Map fields: `tag_id` (id), `name`, `slug`.  
     - Matching column: `tag_id`.  
     - Authentication: Google Sheets OAuth2.

6. **Media Sync Pagination Variables:**

   - **Set Node ("Variables2")**  
     - Initialize for media sync:  
       - `base_url`: `https://ozwebexpert.com/wp-json/wp/v2/media`  
       - `per_page`: `10`  
       - `current_page`: `0`

7. **Media Pagination Loop:**

   - **Code Node ("Loop Vars2")**  
     - Increment `current_page` by 1 each iteration.

   - **HTTP Request Node ("Get Media")**  
     - GET request to media endpoint with parameters:  
       - `per_page`, `page` from variables  
       - `parent_exclude=0` (exclude unattached media)  
     - Authentication: HTTP Basic Auth WordPress token.  
     - Full response JSON with headers.

   - **SplitOut Node ("Split Out1")**  
     - Split field: `body` (media array).

   - **Google Sheets Node ("Update Media")**  
     - Append or Update operation on "Media" sheet (gid=1436699446).  
     - Map fields: `media_id` (id), `post_id` (post), `alt` (alt_text), `url` (source_url), `slug`, `title` (rendered), `width` and `height` (full size).  
     - Matching column: `media_id`.  
     - Authentication: Google Sheets OAuth2.

   - **If Node ("If1")**  
     - Condition: current_page >= `x-wp-totalpages` header from Get Media response.  
     - If false, loop back to Loop Vars2 for next page.  
     - If true, end media pagination.

8. **Connect all nodes according to the logical flow:**

   - Schedule Trigger → Variables  
   - Webhook → Variables  
   - Variables → Loop Vars → Get Posts → Split Out → Update Posts → If  
   - If (false) → Loop Vars  
   - If (true) → Get Categories, Variables2, HTTP Request (tags)  
   - Get Categories → Update Categories  
   - HTTP Request (tags) → Split Out2 → Append or update row in sheet  
   - Variables2 → Loop Vars2 → Get Media → Split Out1 → Update Media → If1  
   - If1 (false) → Loop Vars2  
   - If1 (true) → End

9. **Credentials Setup:**

   - Create HTTP Basic Auth credentials for WordPress site with valid API token or user credentials.  
   - Create Google Sheets OAuth2 credentials with access to the target spreadsheet.

10. **Default Values & Constraints:**

    - Pagination limit (`per_page`) is 10 for posts and media to control request size.  
    - Tags request uses `per_page=100` to fetch all tags in one go.  
    - Sheets must have the specified tabs ("Posts", "Categories", "Tags", "Media") with matching columns.

11. **Optional: Add Sticky Notes for Documentation**

    - Add notes describing each block and trigger method for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                                                                                                                             |
|----------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------|
| Two ways to start: Schedule runs daily, webhook for manual refresh.                          | Workflow design principle                                                                                                                   |
| Initialise pagination variables for REST API requests.                                      | Posts and Media pagination setup                                                                                                           |
| Fetch posts from WP, split them, and upsert into 'Posts' sheet.                             | Posts sync block                                                                                                                           |
| Check if we’ve reached the last page of posts. If not, continue loop.                       | Posts pagination control                                                                                                                   |
| Sync categories into 'Categories' sheet                                                    | Categories one-time fetch and update                                                                                                      |
| Fetch tags and update 'Tags' sheet.                                                       | Tags one-time fetch and update                                                                                                            |
| Fetch media library (with pagination) and sync into 'Media' sheet.                         | Media pagination and sync                                                                                                                  |
| Stop or continue looping until all media pages are processed.                             | Media pagination control                                                                                                                   |
| Video Guide: @[youtube](czSMWyD6f-0)                                                       | Visual guide for the workflow setup                                                                                                       |
| Google Sheets Template: https://docs.google.com/spreadsheets/d/1i4h5kybiYLTT0wTzXbWN7rxp-DV6dOyNcwqgg8I5Rv4/edit?usp=sharing | Sample Google Sheets template used for syncing                                                                                             |

---

**Disclaimer:**  
The text above is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.