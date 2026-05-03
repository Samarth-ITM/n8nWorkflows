Batch Delete Posts and Featured Images in WordPress

https://n8nworkflows.xyz/workflows/batch-delete-posts-and-featured-images-in-wordpress-4976


# Batch Delete Posts and Featured Images in WordPress

### 1. Workflow Overview

This workflow automates the batch deletion of WordPress posts and their associated featured images. It targets scenarios where multiple posts with a specific status (e.g., "pending") must be removed efficiently, ensuring that any linked featured media is also deleted to prevent orphaned files.

The workflow is logically divided into these blocks:

- **1.1 Input Reception**: Manual trigger node to initiate the batch deletion process with the WordPress site URL configured.

- **1.2 Post Retrieval**: HTTP request to fetch posts filtered by status (pending) and ordered by date ascending.

- **1.3 Post Filtering**: Limits processing to the first post in the batch for controlled deletion cycles.

- **1.4 Conditional Routing**: Determines if the post has a featured image (media) associated.

- **1.5 Media Handling**: If a featured image exists, retrieves and deletes this media item before deleting the post.

- **1.6 Post Deletion**: Deletes the post directly if no featured image is linked or after deleting the media.

- **1.7 Documentation and Notes**: Sticky notes providing guidance for approvals, workflow functionality, expansion ideas, and configuration reminders.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Starts the workflow manually and provides the WordPress site URL to be used in subsequent API requests.

- **Nodes Involved:**  
  - `Change your Domain here` (Manual Trigger)

- **Node Details:**  
  - *Type:* Manual Trigger  
  - *Configuration:* No input parameters; contains pinned data with `wpUrl` set to `https://setyourwordpresshere.com` by default, which must be customized by the user.  
  - *Expressions:* Used in downstream HTTP requests to dynamically build API endpoints using `{{$node["Change your Domain here"].item.json.wpUrl}}`.  
  - *Connections:* Output to `get post` node.  
  - *Edge Cases:* If the URL is incorrect or inaccessible, HTTP requests will fail downstream. User must replace the placeholder domain.  
  - *Sub-workflows:* None.

#### 2.2 Post Retrieval

- **Overview:**  
  Fetches all posts from the WordPress REST API with status "pending", ordered ascending by date. This collects candidates for deletion.

- **Nodes Involved:**  
  - `get post`

- **Node Details:**  
  - *Type:* HTTP Request  
  - *Configuration:*  
    - Method: GET (default, no explicit method set)  
    - URL: `{{ $node["Change your Domain here"].item.json.wpUrl }}/wp-json/wp/v2/posts`  
    - Query parameters (sent as body parameters due to node config):  
      - `status` = "pending"  
      - `order` = "asc"  
    - Authentication: **Must be configured by user** to access WP REST API securely.  
  - *Expressions:* Uses domain from `Change your Domain here` node.  
  - *Connections:* Output to `Filter`.  
  - *Edge Cases:*  
    - Authentication failure or insufficient permissions.  
    - No posts returned or API rate limiting.  
  - *Version:* 4.2 HTTP Request node.

#### 2.3 Post Filtering

- **Overview:**  
  Filters posts to process only the first item in the batch, enabling a stepwise deletion approach.

- **Nodes Involved:**  
  - `Filter`

- **Node Details:**  
  - *Type:* Filter node  
  - *Configuration:* Condition: item index `<` 1 (only the first post passes filter)  
  - *Connections:* Output to `Has Img` node.  
  - *Edge Cases:* If no posts exist, no further processing occurs.  
  - *Version:* 2.1 Filter node.

#### 2.4 Conditional Routing (Featured Image Check)

- **Overview:**  
  Checks if the filtered post has a featured image (`featured_media` field) attached.

- **Nodes Involved:**  
  - `Has Img`

- **Node Details:**  
  - *Type:* If node  
  - *Configuration:* Condition: `featured_media` from Filter node JSON output is not equal to 0  
  - *Connections:*  
    - True branch: to `get img` (retrieve media)  
    - False branch: to `delete post` (delete post directly)  
  - *Edge Cases:*  
    - Missing or malformed `featured_media` field may cause expression errors.  
  - *Version:* 2.1.

#### 2.5 Media Handling (Retrieve and Delete Featured Image)

- **Overview:**  
  If a featured image exists, this block retrieves the media object and then deletes it before deleting the post.

- **Nodes Involved:**  
  - `get img`  
  - `delete img`  
  - `delete post with img`

- **Node Details:**  

  - `get img`  
    - *Type:* HTTP Request  
    - *Configuration:*  
      - Method: GET (default)  
      - URL: `{{ $node["Change your Domain here"].item.json.wpUrl }}/wp-json/wp/v2/media/{{ $json.featured_media }}`  
    - *Connections:* Output to `delete img`  
    - *Edge Cases:* Media may not exist or API may return 404.  

  - `delete img`  
    - *Type:* HTTP Request  
    - *Configuration:*  
      - Method: DELETE  
      - URL: `{{ $node["Change your Domain here"].item.json.wpUrl }}/wp-json/wp/v2/media/{{ $json.id }}`  
      - Body parameter: `force=true` to permanently delete  
    - *Connections:* Output to `delete post with img`  
    - *Edge Cases:*  
      - Failure to delete media (permissions, not found).  
      - If media deletion fails, post deletion might still proceed causing orphaned references.

  - `delete post with img`  
    - *Type:* HTTP Request  
    - *Configuration:*  
      - Method: DELETE  
      - URL: `{{ $node["Change your Domain here"].item.json.wpUrl }}/wp-json/wp/v2/posts/{{ $('Filter').item.json.id }}`  
      - Body parameter: `force=true`  
    - *Connections:* None (end node for this branch)  
    - *Edge Cases:* Same as other deletions (auth, permissions).

#### 2.6 Post Deletion (No Featured Image)

- **Overview:**  
  Deletes the post directly if no featured image is attached.

- **Nodes Involved:**  
  - `delete post`

- **Node Details:**  
  - *Type:* HTTP Request  
  - *Configuration:*  
    - Method: DELETE  
    - URL: `{{ $node["Change your Domain here"].item.json.wpUrl }}/wp-json/wp/v2/posts/{{ $('Filter').item.json.id }}`  
    - Body parameter: `force=true`  
  - *Connections:* None (end node for this branch)  
  - *Edge Cases:* Similar to post deletion with image (auth issues, not found).

#### 2.7 Documentation and Notes

- **Overview:**  
  Contains multiple sticky notes that provide guidance for users on setup, approval process, workflow logic, and possible expansions.

- **Nodes Involved:**  
  - `Sticky Note`  
  - `Sticky Note1`  
  - `Sticky Note2`  
  - `Sticky Note3`  
  - `Sticky Note4`  
  - `Sticky Note5`

- **Node Details:**  
  - *Type:* Sticky Note  
  - *Content Highlights:*  
    - Approval process recommendation (manual or via external app/webhook).  
    - Router explanation (detecting featured image).  
    - Magic logic of deleting media before post.  
    - Trigger explanation and domain configuration reminder.  
    - Retrieval of posts explanation including auth requirement.  
    - Expansion suggestion to log deleted posts to external storage.  
  - *Connections:* None, purely informational.  

---

### 3. Summary Table

| Node Name              | Node Type          | Functional Role                           | Input Node(s)          | Output Node(s)          | Sticky Note                                                                                     |
|------------------------|--------------------|-----------------------------------------|-----------------------|-------------------------|------------------------------------------------------------------------------------------------|
| Change your Domain here | Manual Trigger     | Entry point & domain configuration      | -                     | get post                | ## Trigger This workflow is set up for bulk/batch deletion of many WordPress posts...          |
| get post               | HTTP Request       | Retrieve pending posts ordered ascending| Change your Domain here| Filter                  | ## Get Your Posts **IMPORTANT:** Be sure to add your authentication for WordPress in the HTTP Request node... |
| Filter                 | Filter             | Limit to first post for processing       | get post               | Has Img                 |                                                                                                |
| Has Img                | If                 | Check if post has featured image         | Filter                 | get img, delete post    | ## Router This step detects if the post has a featured image associated.                       |
| get img                | HTTP Request       | Retrieve featured image media             | Has Img (true branch)  | delete img              |                                                                                                |
| delete img             | HTTP Request       | Delete featured image media               | get img                | delete post with img    | ## This is the Magic If the post has a featured media associated, the workflow will first delete that media... |
| delete post with img    | HTTP Request       | Delete post after media deletion          | delete img             | -                       | ## This is the Magic If the post has a featured media associated, the workflow will first delete that media... |
| delete post            | HTTP Request       | Delete post directly (no media)           | Has Img (false branch) | -                       |                                                                                                |
| Sticky Note            | Sticky Note        | Approval process recommendations          | -                      | -                       | ## Approvals Built out your approval process here...                                          |
| Sticky Note1           | Sticky Note        | Suggest storing deleted posts externally | -                      | -                       | ## Expansion You might consider storing the results of the deleted posts...                    |
| Sticky Note2           | Sticky Note        | Router logic explanation                   | -                      | -                       | ## Router This step detects if the post has a featured image associated.                       |
| Sticky Note3           | Sticky Note        | Trigger explanation and domain config     | -                      | -                       | ## Trigger This workflow is set up for bulk/batch deletion of many WordPress posts...          |
| Sticky Note4           | Sticky Note        | Magic logic explanation                    | -                      | -                       | ## This is the Magic If the post has a featured media associated, the workflow will first delete that media... |
| Sticky Note5           | Sticky Note        | Get posts explanation and auth reminder  | -                      | -                       | ## Get Your Posts **IMPORTANT:** Be sure to add your authentication for WordPress in the HTTP Request node... |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node** named `Change your Domain here`:  
   - No input required.  
   - Add pinned data property: `wpUrl` with default value `"https://setyourwordpresshere.com"`.  
   - This node starts the workflow.

2. **Add HTTP Request node** named `get post`:  
   - Method: GET (default).  
   - URL: `={{ $node["Change your Domain here"].item.json.wpUrl }}/wp-json/wp/v2/posts`.  
   - Set body parameters (sent as query parameters):  
     - `status` = `pending`  
     - `order` = `asc`  
   - Configure authentication with WordPress credentials to permit API calls.  
   - Connect `Change your Domain here` → `get post`.

3. **Add Filter node** named `Filter`:  
   - Condition: item index `<` 1 (process only first post).  
   - Connect `get post` → `Filter`.

4. **Add If node** named `Has Img`:  
   - Condition: field `featured_media` from `Filter` node output is not equal to 0.  
   - Connect `Filter` → `Has Img`.

5. **Add HTTP Request node** named `get img`:  
   - Method: GET (default).  
   - URL: `={{ $node["Change your Domain here"].item.json.wpUrl }}/wp-json/wp/v2/media/{{ $('Filter').item.json.featured_media }}`.  
   - Connect `Has Img` true output → `get img`.

6. **Add HTTP Request node** named `delete img`:  
   - Method: DELETE.  
   - URL: `={{ $node["Change your Domain here"].item.json.wpUrl }}/wp-json/wp/v2/media/{{ $json.id }}` (uses output from `get img`).  
   - Body parameter: `force` = `true` (to force permanent deletion).  
   - Connect `get img` → `delete img`.

7. **Add HTTP Request node** named `delete post with img`:  
   - Method: DELETE.  
   - URL: `={{ $node["Change your Domain here"].item.json.wpUrl }}/wp-json/wp/v2/posts/{{ $('Filter').item.json.id }}`.  
   - Body parameter: `force` = `true`.  
   - Connect `delete img` → `delete post with img`.

8. **Add HTTP Request node** named `delete post`:  
   - Method: DELETE.  
   - URL: `={{ $node["Change your Domain here"].item.json.wpUrl }}/wp-json/wp/v2/posts/{{ $('Filter').item.json.id }}`.  
   - Body parameter: `force` = `true`.  
   - Connect `Has Img` false output → `delete post`.

9. **Add Sticky Notes** (optional but recommended) for documentation at logical positions:  
   - For approval process guidance near the trigger node.  
   - For router explanation near the `Has Img` node.  
   - For workflow logic and magic explanations near media deletion nodes.  
   - For reminders about authentication and domain change near `Change your Domain here` and `get post`.  
   - For expansion suggestions near the deletion nodes.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                             | Context or Link                                                                                         |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Built out your approval process here. You can split this workflow into two: the first to collect and propose posts for deletion, and the second triggered by an external app (Slack, email) webhook to confirm deletion with identifiable post info (id, featured_media). | Sticky Note near input node "Change your Domain here".                                               |
| This workflow must be authenticated with WordPress REST API credentials that allow post and media deletion. Ensure OAuth2 or Basic Auth credentials are properly set in HTTP Request nodes.                                              | Sticky Note near `get post` node.                                                                     |
| Posts are fetched with status "pending" and ordered ascending; modify parameters in `get post` node to adjust filters (e.g., categories, status).                                                                                       | Sticky Note near `get post` node.                                                                     |
| The workflow first deletes featured media if it exists, then deletes the post to avoid orphaned media files.                                                                                                                            | Sticky Note near media deletion nodes.                                                                |
| Consider logging deleted posts and media to external storage (Airtable, Google Sheets, databases) for audit trails and record-keeping.                                                                                                | Sticky Note near deletion nodes.                                                                       |

---

**Disclaimer:**  
The text provided is exclusively from an n8n automated workflow. All content adheres strictly to relevant content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.