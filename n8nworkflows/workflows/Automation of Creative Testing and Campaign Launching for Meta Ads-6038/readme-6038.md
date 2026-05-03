Automation of Creative Testing and Campaign Launching for Meta Ads

https://n8nworkflows.xyz/workflows/automation-of-creative-testing-and-campaign-launching-for-meta-ads-6038


# Automation of Creative Testing and Campaign Launching for Meta Ads

### 1. Workflow Overview

This workflow automates the process of creative testing and campaign launching on Meta Ads (Facebook Ads) with a focus on conversion performance (CPA - Cost Per Acquisition). It is designed to:

- Regularly scan a Google Drive folder for new creative assets (images and videos).
- Upload these assets to Meta Ads as ad creatives.
- Build and configure a campaign structure including Campaign, Ad Set, and Ads.
- Log all relevant campaign and creative data to a Google Sheet for tracking and analysis.

The workflow is logically divided into four main blocks:

- **1.1 Scheduled Trigger & Configuration Setup:** Initiates the workflow on a weekly basis and sets all necessary configuration parameters.
- **1.2 Creative Processing Pipeline:** Fetches creatives from Google Drive, distinguishes between videos and images, uploads them to Meta Ads, and builds corresponding ad creatives.
- **1.3 Campaign Assembly:** Creates a new Campaign and Ad Set once per execution and merges creatives with the ad set for ad creation.
- **1.4 Ad Creation & Reporting:** Creates individual Ads for each creative under the ad set and logs the complete data to Google Sheets for reporting.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Configuration Setup

**Overview:**  
This block schedules the workflow to run weekly and sets all key parameters required for Meta Ads API calls and folder identification.

**Nodes Involved:**  
- Schedule Trigger  
- Configuration Meta Ads

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Initiates the workflow every Monday at 3:00 PM (UTC assumed).  
  - Configuration: Weekly interval, triggers on Mondays at 15:00 hours.  
  - Inputs: None (trigger node)  
  - Outputs: Passes control to Configuration Meta Ads node.  
  - Potential Failures: Scheduling misconfiguration, timezone mismatch.

- **Configuration Meta Ads**  
  - Type: Set  
  - Role: Defines static parameters such as Ad Account ID, Facebook Page ID, Campaign Name, Pixel ID, website link, and text for ads.  
  - Configuration: Hardcoded string values to be replaced by user (placeholders like "put_your_ad_account_id_here").  
  - Inputs: From Schedule Trigger  
  - Outputs: Configuration parameters passed to the Files search node.  
  - Edge Cases: Missing or incorrect configuration values will cause API calls to fail downstream.

---

#### 1.2 Creative Processing Pipeline

**Overview:**  
This block fetches creative files from a dedicated Google Drive folder, distinguishes between video and image files, uploads them to Meta Ads, creates video or image ad creatives, and formats them into uniform data packets.

**Nodes Involved:**  
- Files search  
- Download Files  
- Is it a Video? (IF node)  
- Upload Video to FB  
- Upload Image to FB  
- Create Video Creative  
- Create Image Creative  
- Set Video ID  
- Set Image Hash  
- Set Video Packet  
- Set Image Packet  
- Merge Creatives

**Node Details:**

- **Files search**  
  - Type: Google Drive  
  - Role: Queries a specific Google Drive folder for image (.jpg, .png) and video (.mp4) files that are not trashed.  
  - Configuration: Query uses MIME types and parent folder ID `'13WeDNsMdihc79WNZYOGxZvXU9ea7N4X_'`. Returns all matching files.  
  - Inputs: From Configuration Meta Ads  
  - Outputs: List of files to Download Files node.  
  - Failures: Google Drive API auth errors, folder ID invalid or inaccessible.

- **Download Files**  
  - Type: Google Drive (download)  
  - Role: Downloads each file by fileId from Files search.  
  - Inputs: From Files search  
  - Outputs: Passes downloaded file content and metadata to Is it a Video? node.  
  - Failures: API errors, quota exceeded, file missing.

- **Is it a Video?**  
  - Type: IF  
  - Role: Checks if the downloaded file MIME type contains "video" to route processing accordingly.  
  - Inputs: From Download Files  
  - Outputs: True branch to Upload Video to FB; False branch to Upload Image to FB.  
  - Edge cases: MIME type inconsistencies, corrupted metadata.

- **Upload Video to FB**  
  - Type: HTTP Request  
  - Role: Uploads video content as a video asset to Meta Ads via Facebook Graph API endpoint `advideos`.  
  - Configuration: Uses multipart-form-data with binary data field named "data". Authenticated with predefined Facebook credentials.  
  - Inputs: From IF node (True branch)  
  - Outputs: Passes video ID and metadata to Set Video ID node.  
  - Potential Failures: API rate limits, auth token expiration, large file upload timeouts.

- **Set Video ID**  
  - Type: Set  
  - Role: Assigns `video_id`, `original_file_name`, and thumbnail URL from Google Drive to the data object for downstream use.  
  - Inputs: From Upload Video to FB  
  - Outputs: To Create Video Creative  
  - Edge Cases: Missing thumbnail, inconsistent naming.

- **Create Video Creative**  
  - Type: HTTP Request  
  - Role: Creates a video-based ad creative in Meta Ads using the uploaded video ID and page ID.  
  - Configuration: JSON payload specifies `object_story_spec` with `video_data`, call-to-action button "LEARN_MORE" linking to configured website.  
  - Inputs: From Set Video ID  
  - Outputs: Passes created creative ID to Set Video Packet.  
  - Potential Failures: API errors, invalid page ID, missing video asset.

- **Set Video Packet**  
  - Type: Set  
  - Role: Constructs a data packet with `creative_id`, original filename, and MIME type for the video creative.  
  - Inputs: From Create Video Creative  
  - Outputs: To Merge Creatives (first input).  
  - Edge Cases: Missing creative ID, data inconsistencies.

- **Upload Image to FB**  
  - Type: HTTP Request  
  - Role: Uploads image content to Meta Ads via `adimages` endpoint.  
  - Configuration: Multipart-form-data with binary data field "data". Uses Facebook credentials.  
  - Inputs: From IF node (False branch)  
  - Outputs: Passes image upload result to Set Image Hash.  
  - Failures: Similar to video upload (auth, API limits, file size).

- **Set Image Hash**  
  - Type: Set  
  - Role: Extracts `image_hash`, sets original filename and MIME type for the image asset.  
  - Inputs: From Upload Image to FB  
  - Outputs: To Create Image Creative  
  - Edge Cases: Missing image hash, broken upload response.

- **Create Image Creative**  
  - Type: HTTP Request  
  - Role: Creates an image-based ad creative specifying `link_data` with image hash, website link, and primary text.  
  - Inputs: From Set Image Hash  
  - Outputs: To Set Image Packet  
  - Failures: API errors, invalid image hash.

- **Set Image Packet**  
  - Type: Set  
  - Role: Creates a standardized data packet for the image creative including `creative_id`, original filename, MIME type.  
  - Inputs: From Create Image Creative  
  - Outputs: To Merge Creatives (second input).  
  - Edge Cases: Missing fields, data integrity issues.

- **Merge Creatives**  
  - Type: Merge  
  - Role: Combines video and image creative packets into a single stream for the campaign assembly block.  
  - Inputs: From Set Video Packet (first input), Set Image Packet (second input)  
  - Outputs: To Merge node for final combination with Ad Set data.  
  - Edge Cases: Unbalanced inputs, empty streams.

---

#### 1.3 Campaign Assembly

**Overview:**  
This block handles the creation of the campaign and ad set (once per workflow run) and merges these with the processed creatives to prepare for ad creation.

**Nodes Involved:**  
- Run Once (Function)  
- Create Campaign  
- Create Ad Set  
- Save Adset Id  
- Merge  
- Merge Creatives (input from previous block)

**Node Details:**

- **Run Once**  
  - Type: Function  
  - Role: Ensures downstream campaign and ad set creation nodes execute a single time per workflow run.  
  - Configuration: Returns only the first item in the data stream to prevent multiple executions.  
  - Inputs: From Merge Creatives (second main input)  
  - Outputs: To Create Campaign  
  - Edge Cases: If input is empty, may skip campaign creation.

- **Create Campaign**  
  - Type: HTTP Request  
  - Role: Creates a new Meta Ads campaign with objective `OUTCOME_SALES`, status `PAUSED`, and no special ad categories.  
  - Configuration: Uses dynamic campaign name with current date (format ddMMyyyy). Authenticated with Facebook Graph API credentials.  
  - Inputs: From Run Once  
  - Outputs: To Create Ad Set  
  - Failures: Auth errors, campaign name conflicts, API limits.

- **Create Ad Set**  
  - Type: HTTP Request  
  - Role: Creates an ad set under the newly created campaign, configured for daily budget of 500 (currency assumed), billing by impressions, optimized for offsite conversions with pixel and custom event.  
  - Configuration: Targeting US only, status `PAUSED`. Uses pixel ID and custom event type from configuration node.  
  - Inputs: From Create Campaign  
  - Outputs: To Save Adset Id  
  - Failures: Invalid pixel ID, budget errors.

- **Save Adset Id**  
  - Type: Set  
  - Role: Extracts and stores the adset ID for use in ad creation.  
  - Inputs: From Create Ad Set  
  - Outputs: To Merge node (second input)  
  - Edge Cases: Missing or invalid adset ID.

- **Merge**  
  - Type: Merge (combine mode)  
  - Role: Combines the full list of creatives (first input) with the single ad set ID (second input) producing all combinations for ad creation.  
  - Inputs: First input from Merge Creatives, second input from Save Adset Id  
  - Outputs: To Create Ad node  
  - Edge Cases: No creatives or ad set ID will halt downstream processing.

---

#### 1.4 Ad Creation & Reporting

**Overview:**  
This block creates individual ads for each creative/ad set combination and logs all resulting data into a Google Sheet for record-keeping.

**Nodes Involved:**  
- Create Ad  
- Save Full Report to Sheet

**Node Details:**

- **Create Ad**  
  - Type: HTTP Request  
  - Role: Creates an Ad in the Meta Ads account for each creative under the specified ad set, with status `PAUSED`.  
  - Configuration: Uses dynamic ad name (original file name), links creative ID and adset ID.  
  - Inputs: From Merge node  
  - Outputs: To Save Full Report to Sheet  
  - Failures: API errors, invalid creative or adset IDs.

- **Save Full Report to Sheet**  
  - Type: Google Sheets  
  - Role: Appends or updates a row in a Google Sheet with details including Ad ID, creative type, adset ID, campaign ID, creative ID, filename, and timestamp.  
  - Configuration: Uses a specific sheet and document ID; updates based on CreativeID.  
  - Inputs: From Create Ad  
  - Outputs: End of workflow  
  - Failures: Google Sheets API errors, permission issues.

---

### 3. Summary Table

| Node Name            | Node Type             | Functional Role                            | Input Node(s)               | Output Node(s)              | Sticky Note                                                                                  |
|----------------------|-----------------------|------------------------------------------|-----------------------------|-----------------------------|----------------------------------------------------------------------------------------------|
| Schedule Trigger     | Schedule Trigger      | Triggers workflow weekly at Monday 3PM   | None                        | Configuration Meta Ads       | ## Weekly CPA Creative Testing Objective and trigger details                                |
| Configuration Meta Ads | Set                   | Holds config parameters for Meta Ads API | Schedule Trigger            | Files search                 | Same sticky note as Schedule Trigger                                                        |
| Files search         | Google Drive          | Searches Google Drive for creatives       | Configuration Meta Ads      | Download Files               | ## Block 2: Creative Processing Pipeline overview                                           |
| Download Files       | Google Drive          | Downloads files found in search           | Files search                | Is it a Video?               | Same as Files search                                                                        |
| Is it a Video?       | IF                    | Branches flow based on MIME type          | Download Files              | Upload Video to FB, Upload Image to FB | Same as Files search                                                                  |
| Upload Video to FB   | HTTP Request          | Uploads video file to Facebook Ads        | Is it a Video? (true branch)| Set Video ID                 | Same as Files search                                                                        |
| Set Video ID         | Set                   | Sets video ID and metadata                 | Upload Video to FB          | Create Video Creative        | Same as Files search                                                                        |
| Create Video Creative | HTTP Request          | Creates video ad creative on Facebook     | Set Video ID                | Set Video Packet             | Same as Files search                                                                        |
| Set Video Packet     | Set                   | Formats video creative data packet         | Create Video Creative       | Merge Creatives (input 1)    | Same as Files search                                                                        |
| Upload Image to FB   | HTTP Request          | Uploads image file to Facebook Ads        | Is it a Video? (false branch)| Set Image Hash               | Same as Files search                                                                        |
| Set Image Hash       | Set                   | Sets image hash and metadata               | Upload Image to FB          | Create Image Creative        | Same as Files search                                                                        |
| Create Image Creative | HTTP Request          | Creates image ad creative on Facebook     | Set Image Hash              | Set Image Packet             | Same as Files search                                                                        |
| Set Image Packet     | Set                   | Formats image creative data packet         | Create Image Creative       | Merge Creatives (input 2)    | Same as Files search                                                                        |
| Merge Creatives      | Merge                 | Combines image and video creative packets | Set Video Packet, Set Image Packet | Merge, Run Once           | ## Block 3: Campaign Assembly explanation                                                  |
| Run Once             | Function              | Ensures campaign/adset creation runs once | Merge Creatives             | Create Campaign              | Same as Merge Creatives                                                                    |
| Create Campaign      | HTTP Request          | Creates a new campaign                     | Run Once                    | Create Ad Set                | Same as Merge Creatives                                                                    |
| Create Ad Set        | HTTP Request          | Creates ad set under campaign              | Create Campaign             | Save Adset Id                | Same as Merge Creatives                                                                    |
| Save Adset Id        | Set                   | Saves adset ID for merging                  | Create Ad Set               | Merge (input 2)              | Same as Merge Creatives                                                                    |
| Merge                | Merge (combine)       | Combines creatives with adset ID            | Merge Creatives (input 1), Save Adset Id (input 2) | Create Ad                   | Same as Block 4                                                                            |
| Create Ad            | HTTP Request          | Creates ads for each creative/adset combo | Merge                       | Save Full Report to Sheet    | ## Block 4: Ad Creation & Reporting overview                                               |
| Save Full Report to Sheet | Google Sheets       | Logs all ad and creative details            | Create Ad                   | None                        | Same as Create Ad                                                                          |
| Sticky Note          | Sticky Note           | Description of weekly CPA testing           | None                        | None                        | -                                                                                            |
| Sticky Note1         | Sticky Note           | Description of Block 2 Creative Pipeline    | None                        | None                        | -                                                                                            |
| Sticky Note2         | Sticky Note           | Description of Block 3 Campaign Assembly    | None                        | None                        | -                                                                                            |
| Sticky Note3         | Sticky Note           | Description of Block 4 Ad Creation & Reporting | None                    | None                        | -                                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Set interval to weekly.  
   - Trigger at Monday, 15:00 (3 PM).  
   - No inputs.

2. **Create a Set node named "Configuration Meta Ads"**  
   - Add string fields:  
     - `ad_account_id` (your Meta Ads account ID)  
     - `facebook_page_id` (your Facebook page ID)  
     - `campaign_name` (base name for campaigns)  
     - `custom_event_type` (e.g., "ADD_TO_CART")  
     - `website_link` (destination URL)  
     - `pixel_id` (Meta Pixel ID)  
     - `primary_text` (ad primary text)  
     - `headline` (ad headline)  
   - Link from Schedule Trigger.

3. **Add a Google Drive node "Files search"**  
   - Operation: Search files.  
   - Query: `(mimeType='image/jpeg' or mimeType='image/png' or mimeType='video/mp4') and '<your_folder_id>' in parents and trashed=false`  
   - Return all results, select fields: id, name, mimeType, webViewLink, thumbnailLink.  
   - Set Google Drive OAuth2 credentials.  
   - Connect from Configuration Meta Ads.

4. **Add a Google Drive node "Download Files"**  
   - Operation: Download file.  
   - Use expression for `fileId`: `={{ $json.id }}` from Files search.  
   - Set credentials same as Files search.  
   - Connect from Files search.

5. **Add an IF node "Is it a Video?"**  
   - Condition: String contains `video` in `{{$json.mimeType}}`  
   - Connect from Download Files.

6. **Create an HTTP Request node "Upload Video to FB"**  
   - Method: POST  
   - URL: `https://graph-video.facebook.com/v23.0/act_{{ your_ad_account_id }}/advideos` (replace placeholder dynamically)  
   - Body: multipart-form-data, field name `source`, input binary data field `data`.  
   - Use Facebook Graph API credentials.  
   - Connect from IF node (true branch).

7. **Create a Set node "Set Video ID"**  
   - Assign variables:  
     - `video_id` from `{{$json.id}}` (upload response)  
     - `original_file_name` from `{{ $('Download Files').item.json.name }}`  
     - `gdrive_thumbnail` from `{{ $('Files search').item.json.thumbnailLink }}`  
   - Connect from Upload Video to FB.

8. **Create an HTTP Request node "Create Video Creative"**  
   - Method: POST  
   - URL: `https://graph.facebook.com/v23.0/act_{{ your_ad_account_id }}/adcreatives`  
   - JSON body:  
   ```json
   {
     "name": "{{ $json.original_file_name }}",
     "object_story_spec": {
       "page_id": "{{ your_facebook_page_id }}",
       "video_data": {
         "video_id": {{ $json.video_id }},
         "image_url": "{{ $json.gdrive_thumbnail }}",
         "call_to_action": {
           "type": "LEARN_MORE",
           "value": {
             "link": "{{ your_website_link }}"
           }
         }
       }
     }
   }
   ```
   - Authentication: Facebook Graph API credentials  
   - Connect from Set Video ID.

9. **Create a Set node "Set Video Packet"**  
   - Assign:  
     - `creative_id` from `{{$json.id}}` (create video creative response)  
     - `original_file_name` from `{{ $('Download Files').item.json.name }}`  
     - `mimeType` from `{{ $('Download Files').item.json.mimeType }}`  
   - Connect from Create Video Creative.

10. **Create an HTTP Request node "Upload Image to FB"**  
    - Method: POST  
    - URL: `https://graph.facebook.com/v23.0/act_{{ your_ad_account_id }}/adimages`  
    - Body: multipart-form-data, field name `source`, input binary data field `data`.  
    - Authentication: Facebook Graph API credentials  
    - Connect from IF node (false branch).

11. **Create a Set node "Set Image Hash"**  
    - Assign:  
      - `image_hash` from `{{$json.images[Object.keys($json.images)[0]].hash}}` (upload image response)  
      - `original_file_name` from `{{ $('Download Files').item.json.name }}`  
      - `mimeType` from `{{ $('Download Files').item.json.mimeType }}`  
    - Connect from Upload Image to FB.

12. **Create an HTTP Request node "Create Image Creative"**  
    - Method: POST  
    - URL: `https://graph.facebook.com/v23.0/act_{{ your_ad_account_id }}/adcreatives`  
    - JSON body:  
    ```json
    {
      "name": "{{ $json.original_file_name }}",
      "object_story_spec": {
        "page_id": "{{ your_facebook_page_id }}",
        "link_data": {
          "image_hash": "{{ $json.image_hash }}",
          "link": "{{ your_website_link }}",
          "message": "{{ your_primary_text }}"
        }
      }
    }
    ```
    - Authentication: Facebook Graph API credentials  
    - Connect from Set Image Hash.

13. **Create a Set node "Set Image Packet"**  
    - Assign:  
      - `creative_id` from `{{$json.id}}` (create image creative response)  
      - `original_file_name` from `{{ $('Download Files').item.json.name }}`  
      - `mimeType` from `{{ $('Download Files').item.json.mimeType }}`  
    - Connect from Create Image Creative.

14. **Create a Merge node "Merge Creatives"**  
    - Mode: Merge inputs  
    - Input 1: Connect from Set Video Packet  
    - Input 2: Connect from Set Image Packet

15. **Create a Function node "Run Once"**  
    - JavaScript code:  
    ```js
    return [items[0]];
    ```  
    - Connect from Merge Creatives (second output/main).

16. **Create an HTTP Request node "Create Campaign"**  
    - Method: POST  
    - URL: `https://graph.facebook.com/v23.0/act_{{ your_ad_account_id }}/campaigns`  
    - JSON body:  
    ```json
    {
      "name": "{{ your_campaign_name }}_{{ $now.toFormat('ddMMyyyy') }}",
      "objective": "OUTCOME_SALES",
      "status": "PAUSED",
      "special_ad_categories": ["NONE"]
    }
    ```  
    - Authentication: Facebook Graph API credentials  
    - Connect from Run Once.

17. **Create an HTTP Request node "Create Ad Set"**  
    - Method: POST  
    - URL: `https://graph.facebook.com/v23.0/act_{{ your_ad_account_id }}/adsets`  
    - JSON body:  
    ```json
    {
      "name": "{{ your_campaign_name }}_{{ $now.toFormat('ddMMyyyy') }}",
      "campaign_id": "{{ $('Create Campaign').item.json.id }}",
      "status": "PAUSED",
      "daily_budget": "500",
      "billing_event": "IMPRESSIONS",
      "optimization_goal": "OFFSITE_CONVERSIONS",
      "bid_strategy": "LOWEST_COST_WITHOUT_CAP",
      "promoted_object": {
        "pixel_id": "{{ your_pixel_id }}",
        "custom_event_type": "{{ your_custom_event_type }}"
      },
      "targeting": {
        "geo_locations": {
          "countries": ["US"]
        }
      }
    }
    ```  
    - Authentication: Facebook Graph API credentials  
    - Connect from Create Campaign.

18. **Create a Set node "Save Adset Id"**  
    - Assign:  
      - `adset_id` from `{{$json.id}}` (Create Ad Set response)  
    - Connect from Create Ad Set.

19. **Create a Merge node "Merge"**  
    - Mode: Combine (all possible combinations)  
    - Input 1: Connect from Merge Creatives (output 1)  
    - Input 2: Connect from Save Adset Id  
    - This merges creatives with the adset ID for ad creation.

20. **Create an HTTP Request node "Create Ad"**  
    - Method: POST  
    - URL: `https://graph.facebook.com/v23.0/act_{{ your_ad_account_id }}/ads`  
    - JSON body:  
    ```json
    {
      "name": "{{ $json.original_file_name }}",
      "adset_id": "{{ $json.adset_id }}",
      "creative": {
        "creative_id": "{{ $json.creative_id }}"
      },
      "status": "PAUSED"
    }
    ```  
    - Authentication: Facebook Graph API credentials  
    - Connect from Merge.

21. **Create a Google Sheets node "Save Full Report to Sheet"**  
    - Operation: Append or Update  
    - Document ID: Your target Google Sheet ID  
    - Sheet Name: "Creatives" (or your sheet name)  
    - Mapping columns: AdID, Type, AdsetID, FileName, Timestamp, CampaignID, CreativeID (mapped dynamically from JSON fields)  
    - Matching columns: `CreativeID`  
    - Connect from Create Ad.

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow runs weekly every Monday at 3:00 PM to automate CPA-focused creative testing for Meta Ads.    | Sticky Note at workflow start                                                                        |
| All Meta Ads configuration parameters are centralized in the "Configuration Meta Ads" node for easy updates. | Sticky Note near Configuration Meta Ads node                                                        |
| Creative processing handles images (.jpg, .png) and videos (.mp4) distinctly, uploading and creating creatives accordingly. | Sticky Note at Block 2                                                                               |
| Campaign assembly creates Campaign and Ad Set once per week, merging with all creatives for ad generation.  | Sticky Note at Block 3                                                                               |
| Ads creation logs detailed metadata to Google Sheets for ongoing tracking and performance analysis.         | Sticky Note at Block 4                                                                               |
| Facebook Graph API version v23.0 is used consistently for all ad management API calls.                       | Implied throughout HTTP Request nodes                                                              |
| Google Drive and Google Sheets OAuth2 credentials must be set up prior to execution.                         | Credential requirements for corresponding nodes                                                    |
| Ensure folder ID in Google Drive query is correct and accessible by the connected account.                   | Files search node configuration                                                                     |
| Recommended to monitor Facebook API rate limits and errors for large volume runs.                           | General operational note                                                                            |
| Folder ID: `13WeDNsMdihc79WNZYOGxZvXU9ea7N4X_` is used for Google Drive creatives location.                 | Files search query                                                                                   |
| Google Sheet document ID: `1gwBOLHpez5fFX9C2m6PoZcsw5LLcjjbR7448jn6cimw` for logging creatives and ads.     | Save Full Report to Sheet node configuration                                                       |

---

This structured analysis and stepwise reconstruction guide will enable developers or AI agents to fully understand, reproduce, and maintain this Meta Ads automation workflow with clear insight into potential failure points and integration dependencies.

---

**Disclaimer:** The provided text is derived solely from an n8n automated workflow, in compliance with content policies, and does not contain any illegal or protected content. All data processed is lawful and publicly accessible.