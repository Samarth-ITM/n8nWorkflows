Publishing Videos Across Multiple Platforms with Blotato (Instagram, YouTube)

https://n8nworkflows.xyz/workflows/publishing-videos-across-multiple-platforms-with-blotato--instagram--youtube--10298


# Publishing Videos Across Multiple Platforms with Blotato (Instagram, YouTube)

### 1. Workflow Overview

This n8n workflow automates the publishing of videos across multiple social media platforms via the Blotato service. It targets users who maintain a video content inventory in Google Sheets and want to streamline video distribution to platforms including Instagram, YouTube, TikTok, Facebook, LinkedIn, Threads, Twitter, Bluesky, and Pinterest. The workflow has two main functional parts:

- **1.1 Scheduled Video Publishing:**  
  Automatically triggered daily at a set hour, it pulls videos marked as ready from Google Sheets, uploads them to Blotato, and publishes simultaneously to multiple social platforms. After successful posting, it updates the sheet status to prevent re-posting.

- **1.2 Manual Upload & Publishing (Test Workflow):**  
  A form-triggered workflow enables manual upload of video files (via Dropbox or other hosts), from which it obtains a shareable URL and metadata, then calls a subworkflow to publish the video across the same platforms.

Logical Blocks:

- **1.1 Input Reception & Trigger:** Schedule Trigger node initiates the process.
- **1.2 Data Retrieval & Configuration:** Reads Google Sheets, sets social media account IDs.
- **1.3 Video Upload to Blotato:** Uploads the video URL to Blotato to prepare media.
- **1.4 Social Media Publishing:** HTTP requests post the video to each connected platform.
- **1.5 Status Update:** Updates the Google Sheet marking the video as posted.
- **1.6 Manual Upload & Subworkflow Invocation:** Handles manual video upload via form and calls sub-workflows for upload and publishing.
- **1.7 Subworkflow for Video to Social:** Handles publishing given a video URL and metadata, referencing credentials and account IDs dynamically.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception & Trigger

- **Overview:**  
  Initiates the scheduled publishing process every day at 22:00 by triggering the workflow.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  
  - **Schedule Trigger**  
    - Type: Trigger node, fires workflow at specified times.  
    - Configuration: Trigger at 22:00 daily.  
    - Input: None (trigger node).  
    - Output: Triggers downstream nodes.  
    - Potential Failures: Misconfigured schedule, workflow disabled.  
    - Version: 1.2

#### Block 1.2: Data Retrieval & Configuration

- **Overview:**  
  Pulls video entries from Google Sheets where videos are marked "Ready" to post, and sets the social media account IDs used for posting.

- **Nodes Involved:**  
  - Get my video (Google Sheets)  
  - Assign Social Media IDs (Set node)

- **Node Details:**  
  - **Get my video**  
    - Type: Google Sheets node, reads rows with "Ready" in "ReadyToPost" column.  
    - Config: Spreadsheet ID and Sheet name provided; filter on "ReadyToPost" = "Ready".  
    - Input: Trigger from Schedule Trigger.  
    - Output: JSON with video URL, description, title, etc.  
    - Failures: Auth errors, sheet not found, no matches.  
    - Version: 4.5  
    - Credential: Google Sheets OAuth2 Account.

  - **Assign Social Media IDs**  
    - Type: Set node, hardcodes social media account IDs from Blotato.  
    - Config: JSON object with IDs for Instagram, YouTube, TikTok, Facebook, Threads, Twitter, LinkedIn, Pinterest, Bluesky, and Pinterest board IDs.  
    - Input: From "Get my video".  
    - Output: Passes social media IDs and video data forward.  
    - Failure: Missing or incorrect IDs will cause posting errors.  
    - Version: 3.4

#### Block 1.3: Video Upload to Blotato

- **Overview:**  
  Uploads the video URL from Google Sheets to Blotato to register media before posting.

- **Nodes Involved:**  
  - Upload Video to Blotato

- **Node Details:**  
  - **Upload Video to Blotato**  
    - Type: HTTP Request, POST to Blotato /v2/media endpoint.  
    - Config: Sends video URL as body parameter "url".  
    - Headers: Requires "blotato-api-key" header (must be set manually or via a centralized credentials data table).  
    - Input: Receives video URL and IDs from previous node.  
    - Output: Returns a media object with a URL used in social posts.  
    - Failures: API key invalid, network issues, malformed URL.  
    - Version: 4.2

#### Block 1.4: Social Media Publishing

- **Overview:**  
  Posts the uploaded video to all configured social media platforms via Blotato API calls.

- **Nodes Involved:**  
  - INSTAGRAM  
  - YOUTUBE  
  - TIKTOK  
  - FACEBOOK  
  - THREADS  
  - LINKEDIN  
  - BLUESKY  
  - PINTEREST  
  - TWITTER

- **Node Details (common patterns):**  
  - Type: HTTP Request nodes, POST to Blotato /v2/posts endpoint.  
  - Config:  
    - Account IDs pulled from "Assign Social Media IDs".  
    - Target type set per platform.  
    - Content includes text description and media URLs from the uploaded video response.  
    - YouTube includes title, privacy settings, and subscriber notification flags.  
    - TikTok includes privacy, AI-generated flags, comment and duet/stitch settings.  
    - Pinterest includes a board ID.  
    - Bluesky and Pinterest use a fixed image URL as media.  
    - Headers include Blotato API key.  
  - Input: Video media URL from "Upload Video to Blotato".  
  - Output: API response for each post.  
  - Failures: API key issues, invalid IDs, platform-specific content restrictions, network issues, rate limiting.  
  - Version: 4.2  
  - Important: All nodes require the Blotato API key header; hardcoded or centralized credential references are recommended.

#### Block 1.5: Status Update

- **Overview:**  
  After all posts are sent, updates the "ReadyToPost" column in Google Sheets from "Ready" to "Finished" to prevent re-posting.

- **Nodes Involved:**  
  - Update Video Sheet

- **Node Details:**  
  - **Update Video Sheet**  
    - Type: Google Sheets node, update operation.  
    - Config: Matches rows by "No" column, updates "ReadyToPost" field to "Finished".  
    - Input: Triggered after all social post HTTP requests complete.  
    - Output: Confirmation of sheet update.  
    - Failures: Sheet permissions, row matching errors, concurrent updates.  
    - Version: 4.5  
    - Credential: Google Sheets OAuth2 Account.

#### Block 1.6: Manual Upload & Subworkflow Invocation (Test Workflow)

- **Overview:**  
  Provides form-based manual upload of a video file, uploads it to Dropbox (or another host), sets URL, title, and description, then triggers a dedicated subworkflow to publish the media.

- **Nodes Involved:**  
  - Upload File (Form Trigger)  
  - Call '[SUB] Dropbox upload link' (Subworkflow)  
  - Set URL, Title & Description (Set node)  
  - Call '[SUB] Video to Social' (Subworkflow)  
  - Test Workflow Completed (NoOp)

- **Node Details:**  
  - **Upload File**  
    - Type: Form Trigger, accepts file upload, video title, and description.  
    - Config: Three form fields: file (required), title (required), description (required).  
    - Output: Form data payload.  
    - Failures: Missing required fields, network issues.

  - **Call '[SUB] Dropbox upload link'**  
    - Type: Subworkflow execution node.  
    - Config: Calls a subworkflow that uploads the file to Dropbox and returns a sharable URL.  
    - Input: Form data from "Upload File".  
    - Output: Dropbox share link.

  - **Set URL, Title & Description**  
    - Type: Set node to map the Dropbox URL and metadata to expected keys.  
    - Config: Sets "url", "Title", "Description" from previous nodes.  
    - Output: Prepared data for subworkflow.

  - **Call '[SUB] Video to Social'**  
    - Type: Subworkflow execution node.  
    - Config: Publishes the video URL to social media using the subworkflow.  
    - Input: URL, Title, Description from previous node.  
    - Output: Confirmation.

  - **Test Workflow Completed**  
    - Type: NoOp node, marks workflow end.  
    - Output: None.

#### Block 1.7: Subworkflow: Video to Social Publisher

- **Overview:**  
  A reusable subworkflow that accepts video URL and metadata as inputs and publishes to social media using Blotato API. It references a credentials data table for the Blotato API key and uses assigned social media account IDs.

- **Nodes Involved:**  
  - Media to publish (Execute Workflow Trigger)  
  - get blotato key (Data Table)  
  - Assign Social Media IDs2 (Set node)  
  - Upload Video to Blotato2 (HTTP Request)  
  - INSTAGRAM2, YOUTUBE2, TIKTOK2, FACEBOOK2, THREADS2, TWITTER2, LINKEDIN2, BLUESKY2, PINTEREST2 (HTTP Requests)  
  - Social Posts Completed (NoOp)

- **Node Details:**  
  - **Media to publish**  
    - Type: Execute Workflow Trigger node; receives input from external workflows.  
    - Input: URL, Title, Description.  
    - Output: Triggers downstream publishing nodes.

  - **get blotato key**  
    - Type: Data Table node; retrieves Blotato API key from a stored credentials table.  
    - Config: Filter on service = "blotato".  
    - Output: Provides token to all HTTP request nodes.  
    - Failures: Missing or incorrect data table, no credentials.

  - **Assign Social Media IDs2**  
    - Same function as Assign Social Media IDs but with different hardcoded IDs for this subworkflow context.

  - **Upload Video to Blotato2**  
    - Same as Upload Video to Blotato but uses the token from the credentials data table dynamically.

  - **Social Platform HTTP Nodes (INSTAGRAM2, YOUTUBE2, etc.)**  
    - Same as main workflow, but all use the API key from the data table node for better maintainability.

  - **Social Posts Completed**  
    - Marks end of the subworkflow; could be replaced with another node to trigger further actions.

- **Version:** HTTP nodes mainly version 4.2, Google Sheets and Data Table nodes version 1 or higher.

- **Failure Modes:** Missing credentials, API key errors, invalid account IDs, network issues.

---

### 3. Summary Table

| Node Name                  | Node Type              | Functional Role                     | Input Node(s)                 | Output Node(s)                                  | Sticky Note                                                                                      |
|----------------------------|------------------------|-----------------------------------|------------------------------|------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Schedule Trigger           | Schedule Trigger       | Initiates scheduled publishing    | None                         | Get my video                                   |                                                                                                 |
| Get my video               | Google Sheets          | Retrieves ready videos             | Schedule Trigger             | Assign Social Media IDs                         | ## Data Source Pulls video info from Google Sheets                                              |
| Assign Social Media IDs    | Set                    | Sets social media account IDs      | Get my video                 | Upload Video to Blotato                         | ## Configuration Social account IDs and media upload                                            |
| Upload Video to Blotato    | HTTP Request           | Uploads video URL to Blotato       | Assign Social Media IDs      | INSTAGRAM, YOUTUBE, TIKTOK, FACEBOOK, THREADS, LINKEDIN, BLUESKY, PINTEREST, Update Video Sheet, TWITTER | ## Blotato Key Copy from Blotato! Get account then go to API settings and plug into nodes       |
| INSTAGRAM                 | HTTP Request           | Posts video to Instagram           | Upload Video to Blotato      |                                                    | ## Social Publishing Publish to all connected platforms                                         |
| YOUTUBE                   | HTTP Request           | Posts video to YouTube             | Upload Video to Blotato      |                                                    | ## Social Publishing Publish to all connected platforms                                         |
| TIKTOK                    | HTTP Request           | Posts video to TikTok              | Upload Video to Blotato      |                                                    | ## Social Publishing Publish to all connected platforms                                         |
| FACEBOOK                  | HTTP Request           | Posts video to Facebook            | Upload Video to Blotato      |                                                    | ## Social Publishing Publish to all connected platforms                                         |
| THREADS                   | HTTP Request           | Posts video to Threads             | Upload Video to Blotato      |                                                    | ## Social Publishing Publish to all connected platforms                                         |
| LINKEDIN                  | HTTP Request           | Posts video to LinkedIn            | Upload Video to Blotato      |                                                    | ## Social Publishing Publish to all connected platforms                                         |
| BLUESKY                   | HTTP Request           | Posts video to Bluesky             | Upload Video to Blotato      |                                                    | ## Social Publishing Publish to all connected platforms                                         |
| PINTEREST                 | HTTP Request           | Posts video to Pinterest           | Upload Video to Blotato      |                                                    | ## Social Publishing Publish to all connected platforms                                         |
| TWITTER                   | HTTP Request           | Posts video to Twitter             | Upload Video to Blotato      |                                                    | ## Social Publishing Publish to all connected platforms                                         |
| Update Video Sheet        | Google Sheets          | Marks video as posted in sheet     | Upload Video to Blotato      |                                                    | ## Update Video sheet Updates Basic sheet1 ReadyToPost from 'Ready' to 'Finished'               |
| Upload File               | Form Trigger           | Accepts manual video upload        | None                        | Call '[SUB] Dropbox upload link'               | ## What does this do? Test Workflow: form upload to Dropbox then to subworkflow                  |
| Call '[SUB] Dropbox upload link' | Execute Workflow | Uploads file to Dropbox            | Upload File                 | Set URL, Title & Description                    |                                                                                                 |
| Set URL, Title & Description | Set                  | Prepares input data for subworkflow | Call '[SUB] Dropbox upload link' | Call '[SUB] Video to Social'                    |                                                                                                 |
| Call '[SUB] Video to Social' | Execute Workflow      | Calls subworkflow to publish video | Set URL, Title & Description | Test Workflow Completed                         |                                                                                                 |
| Test Workflow Completed   | NoOp                   | Marks manual test workflow end     | Call '[SUB] Video to Social' | None                                           |                                                                                                 |
| Media to publish          | Execute Workflow Trigger | Entry point for subworkflow        | External trigger            | get blotato key                                | ## Sub Trigger How we trigger this node triggered by another workflow                            |
| get blotato key           | Data Table             | Retrieves Blotato API key          | Media to publish            | Assign Social Media IDs2                        | ## Data table Gets Blotato Key                                                                 |
| Assign Social Media IDs2  | Set                    | Sets social media account IDs      | get blotato key             | Upload Video to Blotato2                        | ## Configuration Social account IDs and media upload                                            |
| Upload Video to Blotato2  | HTTP Request           | Uploads video URL to Blotato       | Assign Social Media IDs2     | INSTAGRAM2, YOUTUBE2, TIKTOK2, FACEBOOK2, THREADS2, TWITTER2 | ## Blotato Key from data table Make sure all these nodes reference the key from data table      |
| INSTAGRAM2                | HTTP Request           | Posts video to Instagram           | Upload Video to Blotato2     | Social Posts Completed                          |                                                                                                 |
| YOUTUBE2                  | HTTP Request           | Posts video to YouTube             | Upload Video to Blotato2     | Social Posts Completed                          |                                                                                                 |
| TIKTOK2                   | HTTP Request           | Posts video to TikTok              | Upload Video to Blotato2     | Social Posts Completed                          |                                                                                                 |
| FACEBOOK2                 | HTTP Request           | Posts video to Facebook            | Upload Video to Blotato2     | Social Posts Completed                          |                                                                                                 |
| THREADS2                  | HTTP Request           | Posts video to Threads             | Upload Video to Blotato2     | Social Posts Completed                          |                                                                                                 |
| TWITTER2                  | HTTP Request           | Posts video to Twitter             | Upload Video to Blotato2     | Social Posts Completed                          |                                                                                                 |
| LINKEDIN2                 | HTTP Request           | Posts video to LinkedIn            | Upload Video to Blotato2     | Social Posts Completed                          |                                                                                                 |
| BLUESKY2                  | HTTP Request           | Posts video to Bluesky             | Upload Video to Blotato2     | Social Posts Completed                          |                                                                                                 |
| PINTEREST2                | HTTP Request           | Posts video to Pinterest           | Upload Video to Blotato2     | Social Posts Completed                          |                                                                                                 |
| Social Posts Completed    | NoOp                   | Marks subworkflow completion       | INSTAGRAM2, YOUTUBE2, etc.  | None                                           | ## This node does nothing Replace it with any node you need after publishing                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Set trigger to daily at 22:00 (or your preferred time).  
   - No credentials needed.

3. **Add a Google Sheets node ("Get my video"):**  
   - Operation: Read rows from spreadsheet.  
   - Connect Google Sheets OAuth2 credentials.  
   - Set document ID to your Google Sheet containing video data.  
   - Set sheet name to the relevant tab (e.g., "Sheet1").  
   - Add a filter where "ReadyToPost" column equals "Ready" to select videos ready for publishing.

4. **Add a Set node ("Assign Social Media IDs"):**  
   - Mode: Raw JSON  
   - Input JSON object with your Blotato social media account IDs, including instagram_id, youtube_id, tiktok_id, facebook_id, facebook_page_id, threads_id, twitter_id, linkedin_id, pinterest_id, pinterest_board_id, bluesky_id.  
   - Input node: Connect from "Get my video".

5. **Add an HTTP Request node ("Upload Video to Blotato"):**  
   - Method: POST  
   - URL: https://backend.blotato.com/v2/media  
   - Body Parameters: Add parameter "url" with value from previous node's video URL field (e.g., `{{$node["Get my video"].json["URL VIDEO"]}}`)  
   - Headers: Add header "blotato-api-key" with your API key value.  
   - Connect from "Assign Social Media IDs".

6. **Add multiple HTTP Request nodes for each social platform:**  
   - Platforms: Instagram, YouTube, TikTok, Facebook, Threads, LinkedIn, Bluesky, Pinterest, Twitter.  
   - URL: https://backend.blotato.com/v2/posts  
   - Method: POST  
   - For each, build JSON body using expressions to set accountId from "Assign Social Media IDs" node, targetType per platform, content text from video description, and mediaUrls referencing the URL from "Upload Video to Blotato" node.  
   - Include platform-specific fields (e.g., YouTube title and privacyStatus, TikTok privacyLevel and flags, Pinterest boardId).  
   - Add header "blotato-api-key" with your API key.  
   - Connect all from "Upload Video to Blotato".

7. **Add a Google Sheets node ("Update Video Sheet"):**  
   - Operation: Update rows in the same sheet.  
   - Match rows by unique identifier column ("No").  
   - Update the "ReadyToPost" column from "Ready" to "Finished".  
   - Connect from one of the last social post nodes or from all in parallel if using a join.

8. **For manual upload testing:**  
   - Add a Form Trigger node ("Upload File") with fields for file upload, video title, and description.  
   - Add an Execute Workflow node to call a subworkflow that uploads the file to Dropbox (or another host) and returns a shareable link.  
   - Add a Set node to map the Dropbox URL, title, and description into parameters expected by the subworkflow.  
   - Add another Execute Workflow node to call the "[SUB] Video to Social" subworkflow passing the URL and metadata.  
   - End with a NoOp node.

9. **Create the "[SUB] Video to Social" subworkflow:**  
   - Add an Execute Workflow Trigger node as entry point.  
   - Add a Data Table node to fetch Blotato API key from credentials table.  
   - Add a Set node for social media IDs specific to this subworkflow.  
   - Add HTTP Request node to upload video URL to Blotato using the key from data table.  
   - Add HTTP Request nodes to post to each social media platform, configured similarly to the main workflow but referencing the API key dynamically.  
   - End with a NoOp node.

10. **Credentials Setup:**  
    - Google Sheets OAuth2 credentials connected wherever Google Sheets nodes are used.  
    - Blotato API key either hardcoded in HTTP nodes header or stored in a Data Table for dynamic retrieval (recommended).  
    - Dropbox or other file hosting credentials for manual upload subworkflow.

11. **Validation and Testing:**  
    - Test scheduled publishing by setting the schedule trigger time close to current time.  
    - Test manual upload and publishing via form trigger.  
    - Validate API key correctness and account IDs.  
    - Monitor for errors such as authentication failures, missing data, or API rate limits.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                | Context or Link                                   |
|---------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------|
| Blotato IDs: Get your social media IDs after connecting accounts in Blotato settings.                                                       | https://my.blotato.com/settings                   |
| Blotato API Key: Get your API key from Blotato API settings and use it in all HTTP request nodes.                                           | https://my.blotato.com/settings/api               |
| Video Sheet Setup: Create a Google Sheet with columns "URL VIDEO" and "ReadyToPost" where "Ready" indicates videos ready to post.          |                                                  |
| Subworkflow Design: Use the "[SUB] Video to Social" subworkflow to publish videos given URL and metadata; it references credentials table. |                                                  |
| Test Workflow: Use the manual upload form trigger workflow to validate subworkflow integration; uploads file to Dropbox and posts videos.  |                                                  |
| API Rate Limits and Errors: Prepare for possible API rate limits or auth errors from Blotato or social platforms, handle retries if needed.|                                                  |
| Replace NoOp node after publishing with notifications or logging nodes as needed for extended workflows.                                    |                                                  |

---

This document provides a detailed reference to understand, reproduce, and maintain the "Publishing Videos Across Multiple Platforms with Blotato" workflow in n8n, enabling efficient multi-platform video publishing automation.