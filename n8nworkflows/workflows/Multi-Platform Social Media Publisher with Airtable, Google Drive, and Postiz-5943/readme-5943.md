Multi-Platform Social Media Publisher with Airtable, Google Drive, and Postiz

https://n8nworkflows.xyz/workflows/multi-platform-social-media-publisher-with-airtable--google-drive--and-postiz-5943


# Multi-Platform Social Media Publisher with Airtable, Google Drive, and Postiz

### 1. Workflow Overview

This workflow automates multi-platform social media content publishing integrating Airtable, Google Drive, and Postiz. It is designed to:

- Fetch media files (videos/images) and social media post content from Airtable.
- Download media files from Google Drive.
- Upload media files to Postiz storage to generate platform-compatible URLs.
- Clean and validate social media content to prevent JSON formatting errors.
- Post content with media references to multiple social media platforms using Postiz API.
- Update Airtable with Postiz media paths for tracking and future reference.

The workflow is logically divided into two main pipelines:

- **1.1 Media Upload Pipeline:** Handles media fetching, downloading from Google Drive, uploading to Postiz, and updating Airtable with Postiz URLs.
- **1.2 Content Posting Pipeline:** Fetches social content, validates and cleans it, retrieves social media integrations from Postiz, routes by platform, cleans platform-specific content, and posts to each social network via Postiz API.

There is also a specialized **1.3 Video Posting Sub-Workflow** focusing on video content distribution with dedicated cleaning and routing.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Media Upload Pipeline

**Overview:**  
This block downloads video and image files from Google Drive based on Airtable references, uploads them to Postiz storage, and updates Airtable records with the resulting Postiz URLs. This ensures all media is stored in Postiz and ready for posting.

**Nodes Involved:**  
- 🎯 Upload (Webhook)  
- Edit Fields (Set)  
- 📊 Content Database (Media) (Airtable)  
- 📥 Download Video from Drive (Google Drive)  
- 📥 Download Image from Drive (Google Drive)  
- 📥 Download x image from Drive (Google Drive)  
- 📹 Video Upload to Postiz (HTTP Request)  
- 🖼️ Image Upload to Postiz (HTTP Request)  
- 🖼️ Image Upload to Postiz2 (HTTP Request)  
- 💾 Save Video Path (Airtable)  
- 💾 Save Image Path (Airtable)  
- 💾 Save Image Path1 (Airtable)  
- 📝 Workflow Documentation (Sticky Note)

**Node Details:**

- **🎯 Upload (Webhook)**
  - Type: Webhook, Entry trigger node.
  - Configuration: Listens on path `328d95ae-3de0-41bf-a8bf-52071bdb36d3`.
  - Role: Starts the media upload pipeline upon external HTTP request.
  - Outputs: Connects to "📊 Content Database (Media)".

- **Edit Fields (Set)**
  - Type: Set node.
  - Configuration: Sets the Airtable RecordId query parameter to a fixed value (e.g., "recuoYjg4icStHsMK").
  - Role: Prepares input data for Airtable content fetch.
  - Inputs: Manual trigger or upstream webhook.
  - Outputs: Connects to "📊 Content Database (Media)".

- **📊 Content Database (Media) (Airtable)**
  - Type: Airtable node.
  - Configuration: Reads record from table "Youtube tool" with ID from input query RecordId.
  - Role: Fetches media metadata (file IDs for video/image) from Airtable.
  - Outputs: Connects to Google Drive download nodes.

- **📥 Download Video from Drive (Google Drive)**
  - Type: Google Drive node.
  - Configuration: Downloads video file from Google Drive using file ID from Airtable field "Short form Video".
  - Role: Retrieves binary video file.
  - Outputs: Connects to "📹 Video Upload to Postiz".

- **📥 Download Image from Drive (Google Drive)**
  - Type: Google Drive node.
  - Configuration: Downloads image file from Google Drive using file ID from Airtable field "Image for socials".
  - Role: Retrieves binary image file for social posts.
  - Outputs: Connects to "🖼️ Image Upload to Postiz".

- **📥 Download x image from Drive (Google Drive)**
  - Type: Google Drive node.
  - Configuration: Same as above, likely duplicate for alternative image usage.
  - Outputs: Connects to "🖼️ Image Upload to Postiz2".

- **📹 Video Upload to Postiz (HTTP Request)**
  - Type: HTTP Request.
  - Configuration: POST multipart-form-data to Postiz `.../upload` endpoint with video binary data.
  - Role: Uploads video file to Postiz storage.
  - Outputs: Connects to "💾 Save Video Path".

- **🖼️ Image Upload to Postiz (HTTP Request)**
  - Type: HTTP Request.
  - Configuration: POST multipart-form-data to Postiz upload endpoint with image binary data.
  - Role: Uploads image file to Postiz storage.
  - Outputs: Connects to "💾 Save Image Path".

- **🖼️ Image Upload to Postiz2 (HTTP Request)**
  - Type: HTTP Request.
  - Configuration: Duplicate image upload node for alternative image usage.
  - Outputs: Connects to "💾 Save Image Path1".

- **💾 Save Video Path (Airtable)**
  - Type: Airtable node.
  - Configuration: Updates original Airtable record with Postiz video path returned from video upload.
  - Role: Maintains data consistency by linking Postiz video URL with Airtable record.

- **💾 Save Image Path (Airtable)**
  - Type: Airtable node.
  - Configuration: Updates Airtable record with Postiz image path from upload.
  - Role: Tracks image media in Airtable for social posts.

- **💾 Save Image Path1 (Airtable)**
  - Type: Airtable node.
  - Duplicate updating of image path for alternate usage.
  
- **📝 Workflow Documentation (Sticky Note)**
  - Contains detailed summary and critical notes about media upload pipeline.
  - Notes critical constraints such as "Cannot use external URLs in Postiz posts," "Files must exist in Google Drive first," and data flow explanation.

**Edge Cases & Failures:**
- Google Drive download failures (file not found, permissions).
- Postiz upload failures (authentication, file size limits).
- Airtable update failures (record locked or missing).
- Missing or invalid RecordId.
- Binary data issues corrupting uploads.

---

#### 1.2 Content Posting Pipeline

**Overview:**  
This pipeline fetches social media post content from Airtable, validates and cleans the content to ensure JSON formatting compliance, retrieves social media integration IDs from Postiz, routes content by platform, cleans platform-specific content, and posts to each social media platform via Postiz API.

**Nodes Involved:**
- Post (Webhook)
- 📊 Content Database (Posts) (Airtable)
- 🔍 Content Validator & Cleaner1 (Code)
- 📋 Content Availability Check1 (IF)
- ❌ Content Error1 (Stop and Error)
- 🔗 integrations (HTTP Request)
- 🔀 Platform Router (Switch)
- 🧹 Instagram Content Cleaner (Code)
- 🧹 LinkedIn Content Cleaner (Code)
- 🧹 Facebook Content Cleaner (Code)
- 🐦 X/Twitter Posts (HTTP Request)
- 🐦 X/Twitter Alt Account (HTTP Request)
- 💼 LinkedIn Publisher (HTTP Request)
- 📘 Facebook Publisher (HTTP Request)
- 📸 Instagram Publisher (HTTP Request)
- 📊 Fixed Results Processor (Code)
- 📝 Complete Workflow Guide (Sticky Note)

**Node Details:**

- **Post (Webhook)**
  - Type: Webhook
  - Configuration: Listens on path `7263d416-6333-429d-9767-528fc6dced38`.
  - Role: Entry point for content posting workflow.

- **📊 Content Database (Posts) (Airtable)**
  - Fetches platform-specific post content fields from Airtable for the given RecordId.
  - Fields include captions and text for Instagram, Twitter, LinkedIn, Facebook, etc.

- **🔍 Content Validator & Cleaner1 (Code)**
  - Validates presence of content and media for each platform.
  - Cleans content by removing line breaks, tabs, extra spaces, escaping quotes and backslashes.
  - Enforces character limits per platform (e.g., Twitter 280 chars).
  - Produces validation results, cleaned content, and media availability flags.

- **📋 Content Availability Check1 (IF)**
  - Checks if content is valid and no critical errors exist.
  - Routes to error handler if content is missing or invalid.

- **❌ Content Error1 (Stop and Error)**
  - Stops workflow with an error message if no valid content is found.
  - Useful for debugging and preventing unnecessary API calls.

- **🔗 integrations (HTTP Request)**
  - Fetches all connected social media integrations from Postiz.
  - Returns platform integration IDs necessary for posting.

- **🔀 Platform Router (Switch)**
  - Routes integration IDs to platform-specific branches:
    - Instagram
    - Twitter/X (main and alt)
    - LinkedIn
    - Facebook
    - YouTube (in main switch, but YouTube posting handled in video workflow)
  - Uses unique integration IDs configured in Postiz.

- **🧹 Instagram Content Cleaner (Code)**
  - Removes line breaks, tabs, and multiple spaces from Instagram caption.
  - Prevents JSON parsing errors in Instagram API calls.

- **🧹 LinkedIn Content Cleaner (Code)**
  - Cleans LinkedIn post content with same logic as Instagram cleaner.

- **🧹 Facebook Content Cleaner (Code)**
  - Cleans Facebook post content similarly.

- **🐦 X/Twitter Posts (HTTP Request)**
  - Posts to main Twitter/X account using Postiz API.
  - Uses cleaned 'twitter single' content and associated postiz image.
  - Immediate post with 1 minute delay.

- **🐦 X/Twitter Alt Account (HTTP Request)**
  - Posts to alternate Twitter/X account.
  - Duplicate logic with different integration ID.

- **💼 LinkedIn Publisher (HTTP Request)**
  - Posts professional content to LinkedIn Pages.
  - Uses cleaned LinkedIn post content and image.

- **📘 Facebook Publisher (HTTP Request)**
  - Posts to Facebook with cleaned content and images.
  - Supports both images and videos.

- **📸 Instagram Publisher (HTTP Request)**
  - Posts visual content to Instagram feed.
  - Requires cleaned caption and image.

- **📊 Fixed Results Processor (Code)**
  - Aggregates and maps posting results.
  - Tracks success/failure counts per platform and account.
  - Extracts post IDs and error messages.
  - Enhances debugging and logs summary.

- **📝 Complete Workflow Guide (Sticky Note)**
  - Provides detailed overview of content posting process.
  - Lists critical notes about content cleaning, API rate limits, platform coverage, and troubleshooting.

**Edge Cases & Failures:**
- Content missing or empty → routed to error node.
- JSON formatting errors due to uncleaned content → API failures.
- Integration ID mismatch or missing in Postiz → posting fails.
- API rate limits (30 requests/hour) → may cause throttling errors.
- Missing media paths in Airtable → posts without media or errors.
- Network or authentication errors on Postiz API calls.

---

#### 1.3 Video Posting Sub-Workflow

**Overview:**  
Specialized sub-workflow for video content publishing across Instagram, Facebook, and YouTube using Postiz API. Includes video content fetching, cleaning, routing, and posting.

**Nodes Involved:**  
- Video (Webhook)  
- 📊 Content Database (Video) (Airtable)  
- 🔗 integrations (Branch 2) (HTTP Request)  
- 🔀 Video Platform Router (Switch)  
- 🧹 Instagram Video Cleaner (Code)  
- 🧹 Facebook Video Cleaner (Code)  
- 📹 Video Upload to Postiz (from media pipeline, referenced)  
- 📹 Instagram Video Publisher (HTTP Request)  
- 📘 Facebook Video Publisher (HTTP Request)  
- 🎬 YouTube Publisher (HTTP Request)  
- 📝 Video Workflow Overview (Sticky Note)

**Node Details:**

- **Video (Webhook)**
  - Trigger for video posting workflow.
  - Listens on path `26fd48c7-7ef9-44ad-9816-feedd75426f4`.

- **📊 Content Database (Video) (Airtable)**
  - Fetches video-specific content and media paths from Airtable.

- **🔗 integrations (Branch 2) (HTTP Request)**
  - Fetches platform integration IDs for video platforms.

- **🔀 Video Platform Router (Switch)**
  - Routes based on integration ID to:
    - Instagram (video)
    - Facebook (video)
    - YouTube (video)

- **🧹 Instagram Video Cleaner (Code)**
  - Cleans Instagram video captions to avoid JSON errors.

- **🧹 Facebook Video Cleaner (Code)**
  - Cleans Facebook video captions.

- **📹 Instagram Video Publisher (HTTP Request)**
  - Posts video content to Instagram via Postiz API.

- **📘 Facebook Video Publisher (HTTP Request)**
  - Posts video content to Facebook.

- **🎬 YouTube Publisher (HTTP Request)**
  - Posts video content to YouTube.
  - Note: Uses image fields currently; may require configuration update for actual video.

- **📝 Video Workflow Overview (Sticky Note)**
  - Detailed description of video posting workflow, platform coverage, and known issues (YouTube posting broken).

**Edge Cases & Failures:**
- Video media missing or not uploaded to Postiz.
- YouTube posting configuration incomplete or incorrect.
- API errors from Postiz.
- Content validation failures.
- Integration ID mismatches.

---

### 3. Summary Table

| Node Name                      | Node Type            | Functional Role                                  | Input Node(s)                     | Output Node(s)                               | Sticky Note                                                                                                                                            |
|--------------------------------|----------------------|-------------------------------------------------|----------------------------------|----------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------|
| 🎯 Upload                      | Webhook              | Entry point for media upload workflow            | (external trigger)               | 📊 Content Database (Media)                   |                                                                                                                                                        |
| Edit Fields                    | Set                  | Sets Airtable RecordId parameter                  | When clicking ‘Execute workflow’  | 📊 Content Database (Media)                   |                                                                                                                                                        |
| 📊 Content Database (Media)    | Airtable             | Fetches media metadata from Airtable              | Edit Fields / 🎯 Upload          | 📥 Download Video from Drive, 📥 Download Image from Drive, 📥 Download x image from Drive |                                                                                                                                                        |
| 📥 Download Video from Drive   | Google Drive         | Downloads video binary from Google Drive          | 📊 Content Database (Media)      | 📹 Video Upload to Postiz                     |                                                                                                                                                        |
| 📥 Download Image from Drive   | Google Drive         | Downloads image binary from Google Drive          | 📊 Content Database (Media)      | 🖼️ Image Upload to Postiz                     |                                                                                                                                                        |
| 📥 Download x image from Drive | Google Drive         | Downloads alternate image binary                   | 📊 Content Database (Media)      | 🖼️ Image Upload to Postiz2                    |                                                                                                                                                        |
| 📹 Video Upload to Postiz      | HTTP Request         | Uploads video file to Postiz storage               | 📥 Download Video from Drive     | 💾 Save Video Path                            |                                                                                                                                                        |
| 🖼️ Image Upload to Postiz      | HTTP Request         | Uploads image file to Postiz storage               | 📥 Download Image from Drive     | 💾 Save Image Path                            |                                                                                                                                                        |
| 🖼️ Image Upload to Postiz2     | HTTP Request         | Uploads alternate image to Postiz                  | 📥 Download x image from Drive   | 💾 Save Image Path1                           |                                                                                                                                                        |
| 💾 Save Video Path             | Airtable             | Updates Airtable record with Postiz video URL     | 📹 Video Upload to Postiz        | (end)                                        |                                                                                                                                                        |
| 💾 Save Image Path             | Airtable             | Updates Airtable record with Postiz image URL     | 🖼️ Image Upload to Postiz        | (end)                                        |                                                                                                                                                        |
| 💾 Save Image Path1            | Airtable             | Updates Airtable record with alternate Postiz image URL | 🖼️ Image Upload to Postiz2       | (end)                                        |                                                                                                                                                        |
| 📝 Workflow Documentation      | Sticky Note          | Describes media upload pipeline and critical notes| (none)                         | (none)                                       | Provides detailed overview of media upload pipeline; critical notes on file handling and Postiz usage                                                  |
| Post                          | Webhook              | Entry point for content posting workflow           | (external trigger)               | 📊 Content Database (Posts)                   |                                                                                                                                                        |
| 📊 Content Database (Posts)    | Airtable             | Fetches social media post content from Airtable   | Post                           | 🔍 Content Validator & Cleaner1               |                                                                                                                                                        |
| 🔍 Content Validator & Cleaner1| Code                 | Cleans and validates all platform content          | 📊 Content Database (Posts)      | 📋 Content Availability Check1                |                                                                                                                                                        |
| 📋 Content Availability Check1 | IF                   | Checks if content is valid and routes accordingly  | 🔍 Content Validator & Cleaner1  | 🔗 integrations / ❌ Content Error1           |                                                                                                                                                        |
| ❌ Content Error1              | Stop and Error       | Stops workflow if no valid content                  | 📋 Content Availability Check1  | (end)                                        |                                                                                                                                                        |
| 🔗 integrations               | HTTP Request         | Fetches Postiz social media integrations           | 📋 Content Availability Check1  | 🔀 Platform Router                            |                                                                                                                                                        |
| 🔀 Platform Router             | Switch               | Routes each integration to appropriate platform    | 🔗 integrations                | Instagram Content Cleaner, Twitter posts, LinkedIn Cleaner, Facebook Cleaner |                                                                                                                                                        |
| 🧹 Instagram Content Cleaner   | Code                 | Cleans Instagram post content to fix JSON errors   | 🔀 Platform Router (instagram)  | 📸 Instagram Publisher                        |                                                                                                                                                        |
| 🧹 LinkedIn Content Cleaner    | Code                 | Cleans LinkedIn post content                        | 🔀 Platform Router (linkedin)   | 💼 LinkedIn Publisher                         |                                                                                                                                                        |
| 🧹 Facebook Content Cleaner    | Code                 | Cleans Facebook post content                        | 🔀 Platform Router (facebook)   | 📘 Facebook Publisher                         |                                                                                                                                                        |
| 🐦 X/Twitter Posts             | HTTP Request         | Posts to main Twitter/X account                     | 🔀 Platform Router (x main)     | 📊 Fixed Results Processor                     |                                                                                                                                                        |
| 🐦 X/Twitter Alt Account       | HTTP Request         | Posts to alternate Twitter/X account                | 🔀 Platform Router (x alt)      | 📊 Fixed Results Processor                     |                                                                                                                                                        |
| 💼 LinkedIn Publisher          | HTTP Request         | Posts content to LinkedIn Pages                      | 🧹 LinkedIn Content Cleaner      | 📊 Fixed Results Processor                     |                                                                                                                                                        |
| 📘 Facebook Publisher          | HTTP Request         | Posts content to Facebook                            | 🧹 Facebook Content Cleaner      | 📊 Fixed Results Processor                     |                                                                                                                                                        |
| 📸 Instagram Publisher         | HTTP Request         | Posts content to Instagram                           | 🧹 Instagram Content Cleaner     | 📊 Fixed Results Processor                     |                                                                                                                                                        |
| 📊 Fixed Results Processor     | Code                 | Aggregates post results and logs summary            | All social posts                | (end)                                        | Provides detailed success/failure reporting per platform and account                                                                                   |
| 📝 Complete Workflow Guide     | Sticky Note          | Describes content posting workflow and troubleshooting| (none)                         | (none)                                       | Complete overview of social media posting process, platform coverage, critical cleaning notes, and troubleshooting                                    |
| Video                        | Webhook              | Entry point for video posting workflow               | (external trigger)               | 📊 Content Database (Video)                   |                                                                                                                                                        |
| 📊 Content Database (Video)    | Airtable             | Fetches video content and metadata                   | Video                         | 🔗 integrations (Branch 2)                     |                                                                                                                                                        |
| 🔗 integrations (Branch 2)    | HTTP Request         | Fetches integrations for video platforms             | 📊 Content Database (Video)      | 🔀 Video Platform Router                       |                                                                                                                                                        |
| 🔀 Video Platform Router       | Switch               | Routes to Instagram / Facebook / YouTube video posts| 🔗 integrations (Branch 2)      | Instagram Video Cleaner, Facebook Video Cleaner, YouTube Publisher |                                                                                                                                                        |
| 🧹 Instagram Video Cleaner     | Code                 | Cleans Instagram video captions                      | 🔀 Video Platform Router (instagram) | 📹 Instagram Video Publisher              |                                                                                                                                                        |
| 🧹 Facebook Video Cleaner      | Code                 | Cleans Facebook video captions                        | 🔀 Video Platform Router (facebook) | 📘 Facebook Video Publisher                |                                                                                                                                                        |
| 📹 Instagram Video Publisher   | HTTP Request         | Posts video content to Instagram                      | 🧹 Instagram Video Cleaner       | (end)                                        |                                                                                                                                                        |
| 📘 Facebook Video Publisher    | HTTP Request         | Posts video content to Facebook                        | 🧹 Facebook Video Cleaner        | (end)                                        |                                                                                                                                                        |
| 🎬 YouTube Publisher           | HTTP Request         | Posts video content to YouTube (currently uses image fields) | 🔀 Video Platform Router (youtube) | (end)                                     | May require configuration update for true video uploads; currently partially broken.                                                                  |
| 📝 Video Workflow Overview     | Sticky Note          | Describes video posting workflow and known issues    | (none)                         | (none)                                       | Details video platforms covered, data flow, and technical notes                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

**Step 1: Set Up Credentials**
- Configure Airtable credentials with Personal Access Token.
- Configure Google Drive OAuth2 credentials.
- Configure HTTP Header Auth credentials for Postiz API.

**Step 2: Create Media Upload Pipeline**

1. Create a Webhook node named "🎯 Upload" with path `/328d95ae-3de0-41bf-a8bf-52071bdb36d3`.
2. Create a "Set" node named "Edit Fields" that sets the parameter `query.RecordId` to the specific Airtable record ID (e.g., `recuoYjg4icStHsMK`).
3. Connect "🎯 Upload" to "Edit Fields".
4. Create "📊 Content Database (Media)" Airtable node:
   - Operation: Read
   - Base: Select your Airtable base.
   - Table: "Youtube tool"
   - Record ID: `={{ $json.query.RecordId }}`
5. Connect "Edit Fields" to "📊 Content Database (Media)".
6. Create three Google Drive nodes:
   - "📥 Download Video from Drive": download operation for video file using field "Short form Video".
   - "📥 Download Image from Drive": download operation for image file using field "Image for socials".
   - "📥 Download x image from Drive": download operation for same or alternate image.
7. Connect "📊 Content Database (Media)" to all three download nodes.
8. Create three HTTP Request nodes for uploading:
   - "📹 Video Upload to Postiz"
   - "🖼️ Image Upload to Postiz"
   - "🖼️ Image Upload to Postiz2"
   - Use POST method to Postiz `/api/public/v1/upload` endpoint.
   - Set Content-Type to multipart-form-data.
   - Attach binary data from respective Google Drive downloads.
9. Connect video download node to video upload; image download nodes to respective image upload nodes.
10. Create three Airtable update nodes:
    - "💾 Save Video Path" updates record with video upload response path.
    - "💾 Save Image Path" updates record with image upload response path.
    - "💾 Save Image Path1" updates record for alternate image path.
11. Connect respective Postiz upload nodes to these Airtable update nodes.

**Step 3: Create Content Posting Pipeline**

1. Create a Webhook node "Post" with path `/7263d416-6333-429d-9767-528fc6dced38`.
2. Create "📊 Content Database (Posts)" Airtable node:
   - Reads from same base/table.
   - Uses record ID from webhook input.
3. Connect "Post" webhook to "📊 Content Database (Posts)".
4. Create a "Code" node "🔍 Content Validator & Cleaner1" that:
   - Validates presence of platform content fields.
   - Cleans content (removes line breaks, tabs, escapes quotes/backslashes).
   - Enforces platform-specific character limits.
5. Connect Airtable node to validator node.
6. Create an "IF" node "📋 Content Availability Check1" that:
   - Checks `validation.hasContent` is true and no errors.
7. Connect validator to IF node.
8. Create a "Stop and Error" node "❌ Content Error1" with error message "No valid content found for posting".
9. Connect false branch of IF node to error node.
10. Create HTTP Request node "🔗 integrations" to fetch Postiz integrations.
11. Connect true branch of IF node to integrations node.
12. Create a "Switch" node "🔀 Platform Router" with rules matching Postiz integration IDs for Instagram, Twitter (main and alt), LinkedIn, Facebook, YouTube.
13. Connect integrations node to switch.
14. For each platform branch, create content cleaner code nodes:
    - Instagram: "🧹 Instagram Content Cleaner"
    - LinkedIn: "🧹 LinkedIn Content Cleaner"
    - Facebook: "🧹 Facebook Content Cleaner"
15. Connect each cleaner node to respective HTTP Request publisher node:
    - Instagram: "📸 Instagram Publisher"
    - Twitter main: "🐦 X/Twitter Posts"
    - Twitter alt: "🐦 X/Twitter Alt Account"
    - LinkedIn: "💼 LinkedIn Publisher"
    - Facebook: "📘 Facebook Publisher"
16. All publisher nodes send POST requests to Postiz `/api/public/v1/posts`.
    - Use JSON body with immediate posting type "now" and 1-minute delay.
    - Include cleaned content and Postiz image path.
17. Connect all publisher nodes to "📊 Fixed Results Processor" code node.
18. The Fixed Results Processor aggregates and summarizes posting results.

**Step 4: Create Video Posting Sub-Workflow**

1. Create a Webhook node "Video" with path `/26fd48c7-7ef9-44ad-9816-feedd75426f4`.
2. Create Airtable node "📊 Content Database (Video)" fetching video content and media.
3. Connect "Video" to Airtable node.
4. Create HTTP Request node "🔗 integrations (Branch 2)" to fetch video platform integrations.
5. Connect Airtable to integrations.
6. Create "Switch" node "🔀 Video Platform Router" with rules routing to Instagram, Facebook, YouTube.
7. Connect integrations node to switch.
8. Create content cleaner nodes for Instagram and Facebook video captions.
9. Connect these to respective publisher nodes for Instagram video and Facebook video.
10. Create YouTube publisher node configured to post video content (note: may require adjustment).
11. Connect YouTube output branch directly to YouTube publisher.
12. Reference video upload from media upload pipeline for media paths.

**Step 5: Add Sticky Notes**

- Add "📝 Workflow Documentation" note describing media upload pipeline.
- Add "📝 Complete Workflow Guide" note describing social media posting workflow.
- Add "📝 Video Workflow Overview" note describing video posting workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                | Context or Link                                         |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| Cannot use external URLs in Postiz posts; all media must be uploaded to Postiz storage for referencing in posts.                                                                                                                                                             | Workflow documentation sticky note                      |
| Content cleaning nodes are critical to prevent "JSON parameter needs to be valid JSON" errors on all social platforms. They remove line breaks, tabs, multiple spaces, and escape problematic characters.                                                                      | Complete Workflow Guide sticky note                      |
| API rate limit for Postiz posting is 30 requests/hour; ensure spacing of workflow execution to avoid throttling.                                                                                                                                                             | Complete Workflow Guide sticky note                      |
| Video posting to YouTube currently uses image fields and not actual video files; configuration requires refinement to support proper video uploads.                                                                                                                         | Video Workflow Overview sticky note                      |
| Integration IDs are unique per Postiz deployment and must be updated in Switch nodes to match actual connected platforms.                                                                                                                                                    | Platform Router node notes                               |
| Postiz API endpoints used: `/api/public/v1/upload` for media uploads and `/api/public/v1/posts` for posting content. Authentication uses HTTP Header Auth credentials configured in n8n.                                                                                      | Workflow documentation and HTTP Request node notes      |
| The Fixed Results Processor node aggregates success/failure per platform and account, providing detailed insights into posting results for debugging and monitoring.                                                                                                        | Fixed Results Processor node notes                       |

---

**Disclaimer:**  
The provided content originates solely from an automated workflow created with n8n, adhering strictly to content policies. No illegal, offensive, or copyrighted materials are included. All manipulated data is legal and publicly accessible.