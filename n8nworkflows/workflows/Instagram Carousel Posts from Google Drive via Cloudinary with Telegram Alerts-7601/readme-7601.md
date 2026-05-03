Instagram Carousel Posts from Google Drive via Cloudinary with Telegram Alerts

https://n8nworkflows.xyz/workflows/instagram-carousel-posts-from-google-drive-via-cloudinary-with-telegram-alerts-7601


# Instagram Carousel Posts from Google Drive via Cloudinary with Telegram Alerts

### 1. Workflow Overview

This workflow automates the creation and publishing of Instagram Carousel Posts using images stored in Google Drive, with media uploads managed through Cloudinary, and notifications sent via Telegram. It is designed for social media managers or automation specialists who want to streamline the posting process for carousel content on Instagram, integrating Google Drive as a source repository for images and Cloudinary for optimized media hosting.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Initialization:** Manual trigger to start the workflow, with preparation of input data such as Instagram Account ID, Google Drive links for images, and Instagram post text.
- **1.2 Media Processing Loop:** Iterates over each image link from Google Drive, downloads the file, uploads it to Cloudinary, posts it as a carousel item using Facebook Graph API, and gathers media IDs.
- **1.3 Carousel Assembly and Publishing:** After all media items are processed, creates the Instagram carousel post referencing the uploaded media IDs, publishes the carousel post, and sends a Telegram notification about the successful post.
- **1.4 Timing Controls:** Wait nodes are placed strategically to handle API rate limits and ensure asynchronous operations complete before proceeding.
- **1.5 Error Handling and Continuity:** Some nodes are configured to continue on error to maintain workflow progress despite individual failures.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Initialization

- **Overview:**  
  This block is the entry point where the workflow is manually triggered. It sets up essential input data fields such as Instagram Account ID, Google Drive photo links for each carousel image, and the Instagram caption content.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Prepare Data (Set)  
  - Sticky Note (instructions)

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Starts workflow on user command  
    - Configuration: Default, no parameters  
    - Inputs: None  
    - Outputs: One output to “Prepare Data”  
    - Failures: None expected  
    - Sub-workflow: None

  - **Prepare Data**  
    - Type: Set Node  
    - Role: Initializes variables for Instagram Account ID, Google Drive URLs (3 poses), Instagram caption content, and Facebook Graph API token placeholder  
    - Configuration: Sets empty strings for keys 'instagram_content', 'node (Instagram Account ID)', 'pose_1_drive_fotolink', 'pose_2_drive_fotolink', 'pose_3_drive_fotolink', and 'Facebook Graph'  
    - Expressions: None (static empty values)  
    - Inputs: From Manual Trigger  
    - Outputs: To “Loop Over Items1”  
    - Failures: None expected  
    - Sub-workflow: None

  - **Sticky Note**  
    - Content: Instructions for the user to insert Instagram Account ID, Google Drive links for images, and Instagram post text  
    - Position: Near “Prepare Data” node for user visibility

#### 2.2 Media Processing Loop

- **Overview:**  
  This block loops over each image URL specified in the previous step, downloads the file from Google Drive, uploads it to Cloudinary, creates a carousel media item on Instagram, and collects media IDs to assemble the carousel later.

- **Nodes Involved:**  
  - Loop Over Items1 (SplitInBatches)  
  - Download file1 (Google Drive)  
  - Upload images to Cloudinary (HTTP Request)  
  - Create each Carousel Picture (Facebook Graph API)  
  - Edit Fields1 (Set)  
  - Collect Media IDs (Code)  
  - Wait before Carousel (Wait)  
  - Sticky Notes (Cloudinary and GDrive instructions)

- **Node Details:**

  - **Loop Over Items1**  
    - Type: SplitInBatches  
    - Role: Iterates over each image URL from the prepared data, enabling batch processing of media  
    - Configuration: Default batch size (not specified, defaults to 1)  
    - Inputs: From “Prepare Data”  
    - Outputs: Two outputs: one to “Collect Media IDs” (for media ID aggregation), one to “Download file1” (to process each image)  
    - Failures: None expected  
    - Sub-workflow: None

  - **Download file1**  
    - Type: Google Drive Node (Download File)  
    - Role: Downloads the image file from Google Drive using the URL from current batch item  
    - Configuration:  
      - File ID extracted from URL via expression `={{ $json.item_url }}`  
      - On error: Continue workflow despite download failures  
    - Credentials: Google Drive OAuth2 with a configured Vertical Google Drive account  
    - Inputs: From “Loop Over Items1”  
    - Outputs: To “Upload images to Cloudinary”  
    - Failures: Possible errors include invalid file URL, file access permission denied, or network errors  
    - Version: 3

  - **Upload images to Cloudinary**  
    - Type: HTTP Request  
    - Role: Uploads downloaded image binary data to Cloudinary under a preset for n8n uploads  
    - Configuration:  
      - HTTP POST to `https://api.cloudinary.com/v1_1/dd4rbdyco/image/upload`  
      - Multipart form-data with binary field “file” from downloaded file data  
      - Upload preset: “N8N Upload”  
      - Authentication: HTTP Basic Auth via stored Cloudinary credentials  
      - On error: Continue workflow despite upload failures  
    - Inputs: Binary data from “Download file1”  
    - Outputs: To “Create each Carousel Picture”  
    - Failures: Possible errors include invalid credentials, API limits, network errors  
    - Version: 4.2

  - **Create each Carousel Picture**  
    - Type: Facebook Graph API  
    - Role: Creates a carousel media item on Instagram referencing the uploaded image URL from Cloudinary  
    - Configuration:  
      - POST to `/media` edge of Instagram Account ID node  
      - Query parameters:  
        - `image_url` set to `={{ $json.url }}`, the Cloudinary URL  
        - `is_carousel_item` set to `true`  
      - API version: v22.0  
      - On error: Continue workflow on failure  
    - Credentials: Facebook Graph account credentials  
    - Inputs: From “Upload images to Cloudinary”  
    - Outputs: To “Edit Fields1”  
    - Failures: Possible errors include Instagram API rate limits, invalid token, malformed parameters  
    - Version: 1

  - **Edit Fields1**  
    - Type: Set  
    - Role: Extracts and sets relevant fields for further processing and aggregation  
    - Configuration: Sets:  
      - `media_id` from the Facebook Graph API response `id` field  
      - `pose_number` from downloaded file JSON field `pose_number`  
      - `instagram_content` from downloaded file JSON field `instagram_content`  
      - `Instagram Account ID` from downloaded file JSON field `node (Instagram Account ID)`  
      - `airtable id` from downloaded file JSON field `airtable id`  
    - Inputs: From “Create each Carousel Picture”  
    - Outputs: To “Loop Over Items1” (loop continuation)  
    - Failures: Expression evaluation errors if fields missing  
    - Version: 3.4

  - **Collect Media IDs**  
    - Type: Code  
    - Role: Aggregates all media IDs from the loop to prepare for carousel assembly  
    - Configuration:  
      - Collects all `media_id` values from input items  
      - Extracts common data like Instagram content, Instagram Account ID, Facebook Graph token, Airtable ID from first item  
      - Outputs JSON with aggregated media IDs and total count  
    - Inputs: From “Loop Over Items1”  
    - Outputs: To “Wait before Carousel”  
    - Failures: None expected if input items structured correctly  
    - Version: 2

  - **Wait before Carousel**  
    - Type: Wait  
    - Role: Delays execution for 20 seconds to ensure media processing completion before carousel creation  
    - Configuration: Wait 20 seconds  
    - Inputs: From “Collect Media IDs”  
    - Outputs: To “Edit Carousel”  
    - Failures: None expected  
    - Version: 1.1

  - **Sticky Note1 & Sticky Note2**  
    - Content: Reminders to add Cloudinary and Google Drive account credentials  
    - Positioned near relevant nodes for user awareness

#### 2.3 Carousel Assembly and Publishing

- **Overview:**  
  After all media items are uploaded and their IDs collected, this block assembles the carousel post referencing all media IDs, publishes it on Instagram, and sends an alert via Telegram.

- **Nodes Involved:**  
  - Edit Carousel (Facebook Graph API)  
  - Wait before Upload to Instagram (Wait)  
  - Publish Carousel to Instagram (Facebook Graph API)  
  - Send Update Message (Telegram)  
  - Sticky Note3 (Telegram Bot ID instruction)

- **Node Details:**

  - **Edit Carousel**  
    - Type: Facebook Graph API  
    - Role: Creates an Instagram carousel post referencing the collected media IDs  
    - Configuration:  
      - POST to `/media` edge of Instagram Account ID node  
      - Query parameters:  
        - `media_type` = `CAROUSEL`  
        - `children` = list of media IDs from `media_ids`  
        - `caption` = Instagram post text content  
      - API version: v22.0  
    - Credentials: Facebook Graph account credentials  
    - Inputs: From “Wait before Carousel”  
    - Outputs: To “Wait before Upload to Instagram”  
    - Failures: API errors due to invalid media IDs, caption length, or token issues  
    - Version: 1

  - **Wait before Upload to Instagram**  
    - Type: Wait  
    - Role: Waits 15 seconds to ensure Instagram processes the carousel creation before publishing  
    - Configuration: Wait 15 seconds  
    - Inputs: From “Edit Carousel”  
    - Outputs: To “Publish Carousel to Instagram”  
    - Failures: None expected  
    - Version: 1.1

  - **Publish Carousel to Instagram**  
    - Type: Facebook Graph API  
    - Role: Publishes the assembled carousel post on Instagram  
    - Configuration:  
      - POST to `/media_publish` edge of Instagram Account ID node  
      - Query parameter `creation_id` set to ID from “Collect Media IDs” node  
      - API version: v22.0  
      - On error: Continue despite errors  
    - Credentials: Facebook Graph account credentials  
    - Inputs: From “Wait before Upload to Instagram”  
    - Outputs: To “Send Update Message”  
    - Failures: Possible failures include permissions, token expiry, or API limits  
    - Version: 1

  - **Send Update Message**  
    - Type: Telegram  
    - Role: Sends a Telegram notification to a specified chat indicating successful carousel upload  
    - Configuration:  
      - Message text: `The Carousel post {{ instagram_content }} has been successfully uploaded to Instagram.`  
      - Chat ID: `7754721939` (configured Telegram chat)  
    - Credentials: Telegram Bot API credentials  
    - Inputs: From “Publish Carousel to Instagram”  
    - Outputs: None  
    - Failures: Possible failures include invalid bot token, chat ID, or network issues  
    - Version: 1.2

  - **Sticky Note3**  
    - Content: Reminder to add Telegram Bot ID credentials  
    - Positioned near “Send Update Message” node

---

### 3. Summary Table

| Node Name                 | Node Type                    | Functional Role                                  | Input Node(s)                | Output Node(s)                | Sticky Note                                               |
|---------------------------|------------------------------|-------------------------------------------------|------------------------------|------------------------------|-----------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger               | Entry point to start the workflow                | None                         | Prepare Data                 |                                                           |
| Prepare Data              | Set                          | Initializes input variables for Instagram and media links | When clicking ‘Execute workflow’ | Loop Over Items1             | Prepare Data: Insert Instagram Account ID, Google Drive links, Instagram content |
| Loop Over Items1          | SplitInBatches               | Iterates over media links for batch processing   | Prepare Data                 | Collect Media IDs, Download file1 |                                                           |
| Download file1            | Google Drive                 | Downloads each image file from Google Drive      | Loop Over Items1             | Upload images to Cloudinary  | Add Gdrive Account                                         |
| Upload images to Cloudinary | HTTP Request                 | Uploads downloaded images to Cloudinary           | Download file1               | Create each Carousel Picture | Add Cloudinary Account                                    |
| Create each Carousel Picture | Facebook Graph API           | Creates carousel media item on Instagram          | Upload images to Cloudinary  | Edit Fields1                 |                                                           |
| Edit Fields1              | Set                          | Sets fields from API responses for aggregation   | Create each Carousel Picture | Loop Over Items1             |                                                           |
| Collect Media IDs         | Code                         | Aggregates media IDs for carousel assembly       | Loop Over Items1             | Wait before Carousel         |                                                           |
| Wait before Carousel      | Wait                         | Delay to ensure readiness before carousel creation | Collect Media IDs            | Edit Carousel                |                                                           |
| Edit Carousel             | Facebook Graph API           | Creates Instagram carousel post referencing media IDs | Wait before Carousel         | Wait before Upload to Instagram |                                                           |
| Wait before Upload to Instagram | Wait                         | Delay before publishing the carousel post         | Edit Carousel                | Publish Carousel to Instagram |                                                           |
| Publish Carousel to Instagram | Facebook Graph API           | Publishes the carousel post on Instagram          | Wait before Upload to Instagram | Send Update Message          |                                                           |
| Send Update Message       | Telegram                     | Sends notification of successful Instagram post  | Publish Carousel to Instagram | None                        | Add Telegram Bot ID                                       |
| Sticky Note               | Sticky Note                  | Instructions to insert required input data       | None                         | None                        | Prepare Data instructions                                 |
| Sticky Note1              | Sticky Note                  | Reminder to add Cloudinary credentials            | None                         | None                        | Add Cloudinary Account                                    |
| Sticky Note2              | Sticky Note                  | Reminder to add Google Drive credentials          | None                         | None                        | Add Gdrive Account                                       |
| Sticky Note3              | Sticky Note                  | Reminder to add Telegram Bot credentials          | None                         | None                        | Add Telegram Bot ID                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Name: `When clicking ‘Execute workflow’`  
   - Type: Manual Trigger  
   - No special parameters.

2. **Create Set Node for Input Preparation:**  
   - Name: `Prepare Data`  
   - Type: Set  
   - Add fields with empty string values:  
     - `instagram_content` (string)  
     - `node (Instagram Account ID)` (string)  
     - `pose_1_drive_fotolink` (string)  
     - `pose_2_drive_fotolink` (string)  
     - `pose_3_drive_fotolink` (string)  
     - `Facebook Graph` (string)  
   - Connect output of Manual Trigger to this node.

3. **Create SplitInBatches Node:**  
   - Name: `Loop Over Items1`  
   - Type: SplitInBatches  
   - No special batch size configured (defaults to 1)  
   - Connect output of `Prepare Data` to this node.

4. **Create Google Drive Download Node:**  
   - Name: `Download file1`  
   - Type: Google Drive (Download File)  
   - Parameter: `fileId` set with expression `={{ $json.item_url }}` (URL from batch item)  
   - Set onError to “Continue” to avoid workflow stop on failure  
   - Credential: Configure with Google Drive OAuth2 credentials (e.g., Vertical Google Drive account)  
   - Connect one output of `Loop Over Items1` to this node.

5. **Create HTTP Request Node for Cloudinary Upload:**  
   - Name: `Upload images to Cloudinary`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.cloudinary.com/v1_1/dd4rbdyco/image/upload`  
   - Authentication: HTTP Basic Auth with Cloudinary credentials  
   - Body Type: multipart-form-data  
   - Body Parameters:  
     - `file` as formBinaryData, input field: binary data from `Download file1`  
     - `upload_preset` = `N8N Upload`  
   - On error: Continue  
   - Connect output of `Download file1` to this node.

6. **Create Facebook Graph API Node to Create Carousel Picture:**  
   - Name: `Create each Carousel Picture`  
   - Type: Facebook Graph API  
   - HTTP Method: POST  
   - API Version: v22.0  
   - Node: Expression `={{ $('Download file1').item.json['node (Instagram Account ID)'] }}`  
   - Edge: `media`  
   - Query Parameters:  
     - `image_url` = expression `={{ $json.url }}` (Cloudinary URL)  
     - `is_carousel_item` = `true`  
   - Credentials: Facebook Graph API credentials  
   - On error: Continue  
   - Connect output of `Upload images to Cloudinary` to this node.

7. **Create Set Node to Edit Fields:**  
   - Name: `Edit Fields1`  
   - Type: Set  
   - Assign fields from `Create each Carousel Picture` and `Download file1`:  
     - `media_id` = `={{ $json.id }}` (from Facebook response)  
     - `pose_number` = `={{ $('Download file1').item.json.pose_number }}`  
     - `instagram_content` = `={{ $('Download file1').item.json.instagram_content }}`  
     - `Instagram Account ID` = `={{ $('Download file1').item.json['node (Instagram Account ID)'] }}`  
     - `airtable id` = `={{ $('Download file1').item.json['airtable id'] }}`  
   - Connect output of `Create each Carousel Picture` to this node.

8. **Connect output of `Edit Fields1` back to the second output of `Loop Over Items1`** to allow aggregated data collection.

9. **Create Code Node to Collect Media IDs:**  
   - Name: `Collect Media IDs`  
   - Type: Code  
   - Code: JavaScript to collect all `media_id` fields from all incoming items, plus common fields from first item (Instagram Account ID, instagram_content, Facebook Graph token, airtable id).  
   - Connect second output of `Loop Over Items1` to this node.

10. **Create Wait Node to Delay Before Carousel Creation:**  
    - Name: `Wait before Carousel`  
    - Type: Wait  
    - Amount: 20 seconds  
    - Connect output of `Collect Media IDs` to this node.

11. **Create Facebook Graph API Node to Edit Carousel:**  
    - Name: `Edit Carousel`  
    - Type: Facebook Graph API  
    - HTTP Method: POST  
    - API Version: v22.0  
    - Node: Expression for Instagram Account ID from collected data  
    - Edge: `media`  
    - Query Parameters:  
      - `media_type` = `CAROUSEL`  
      - `children` = list of media IDs collected  
      - `caption` = Instagram caption content  
    - Credentials: Facebook Graph API credentials  
    - Connect output of `Wait before Carousel` to this node.

12. **Create Wait Node Before Publishing Carousel:**  
    - Name: `Wait before Upload to Instagram`  
    - Type: Wait  
    - Amount: 15 seconds  
    - Connect output of `Edit Carousel` to this node.

13. **Create Facebook Graph API Node to Publish Carousel:**  
    - Name: `Publish Carousel to Instagram`  
    - Type: Facebook Graph API  
    - HTTP Method: POST  
    - API Version: v22.0  
    - Node: Instagram Account ID from collected data  
    - Edge: `media_publish`  
    - Query Parameter: `creation_id` from `Collect Media IDs` node output  
    - Credentials: Facebook Graph API credentials  
    - On error: Continue  
    - Connect output of `Wait before Upload to Instagram` to this node.

14. **Create Telegram Node to Send Update Message:**  
    - Name: `Send Update Message`  
    - Type: Telegram  
    - Chat ID: `7754721939` (or your target chat)  
    - Text: `The Carousel post {{ instagram_content }} has been successfully uploaded to Instagram.`  
    - Credentials: Telegram Bot API credentials  
    - Connect output of `Publish Carousel to Instagram` to this node.

15. **Add Sticky Notes:**  
    - Near `Prepare Data`: Instructions to add Instagram Account ID, Google Drive links, Instagram content  
    - Near `Download file1`: Reminder to add Google Drive credentials  
    - Near `Upload images to Cloudinary`: Reminder to add Cloudinary credentials  
    - Near `Send Update Message`: Reminder to add Telegram Bot ID credentials

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                         |
|-----------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| This workflow requires valid credentials for Google Drive OAuth2, Cloudinary HTTP Basic Auth, Facebook Graph API, and Telegram Bot API. | Credential setup in n8n required before execution       |
| Instagram API limits and Facebook Graph API permissions may cause failures; monitor API quotas.     | Facebook Graph API documentation: https://developers.facebook.com/docs/graph-api/ |
| Cloudinary upload preset "N8N Upload" must be configured in your Cloudinary dashboard.              | Cloudinary Upload Presets: https://cloudinary.com/documentation/upload_presets |
| Telegram bot and chat ID must correspond to a valid Telegram bot and authorized chat.               | Telegram Bot API: https://core.telegram.org/bots/api   |
| Wait nodes are used to handle API propagation delays and rate limits; adjust timing if errors occur.| Adjust wait durations based on API response times       |

---

**Disclaimer:**  
This document is an analysis and detailed description of an n8n workflow automating Instagram Carousel posts using Google Drive, Cloudinary, and Telegram. All data handled is legal and public. This workflow respects content policies and does not contain illegal, offensive, or protected material.