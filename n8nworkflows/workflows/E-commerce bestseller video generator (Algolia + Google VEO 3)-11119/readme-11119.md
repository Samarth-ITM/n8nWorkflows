E-commerce bestseller video generator (Algolia + Google VEO 3)

https://n8nworkflows.xyz/workflows/e-commerce-bestseller-video-generator--algolia---google-veo-3--11119


# E-commerce bestseller video generator (Algolia + Google VEO 3)

### 1. Workflow Overview

This workflow automates the creation of a promotional video for the weekly bestseller product in an e-commerce store using Algolia and Google VEO 3. It targets e-commerce businesses leveraging Algolia for product search and Google Vertex AI's VEO 3.0 model for video generation. The workflow runs weekly (or manually for testing), identifies the top-selling product, verifies if a video already exists, validates the product image, generates a cinematic video if needed, uploads the video to Supabase storage, and updates the Algolia product record with the video URL. Notifications are sent to administrators if product images are missing or broken.

The workflow's logic is grouped into these blocks:

- **1.1 Trigger Block:** Initiates the workflow weekly or manually.
- **1.2 Bestseller Retrieval:** Queries Algolia for the top-selling product.
- **1.3 Video Existence Check:** Determines if the product already has a video.
- **1.4 Image Validation:** Checks presence and accessibility of product image.
- **1.5 Video Generation:** Converts image, sends it to Google VEO 3 API, waits for processing.
- **1.6 Video Handling and Upload:** Retrieves generated video, uploads to Supabase.
- **1.7 Algolia Update:** Updates product record to mark video presence and URL.
- **1.8 Notification:** Alerts admins on missing or broken product images.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger Block

- **Overview:** Starts the workflow either manually or on a scheduled weekly basis (Monday 9:00 PM). The manual trigger is primarily for testing.
- **Nodes Involved:** 
  - Every week on monday morning (disabled)
  - Test Trigger while building the workflow
  - Sticky Note4 (documentation)

- **Node Details:**

  - **Every week on monday morning**
    - Type: Schedule Trigger
    - Configuration: Runs weekly on Monday at 21:00 (9:00 PM)
    - Input: None (time-triggered)
    - Output: Starts workflow
    - Disabled: Yes (used if enabled for automation)
    - Edge cases: Misconfigured timezone or schedule may delay or skip runs.

  - **Test Trigger while building the workflow**
    - Type: Manual Trigger
    - Configuration: No parameters, triggers workflow on demand
    - Input: None
    - Output: Starts workflow
    - Use for immediate testing or debugging.

  - **Sticky Note4**
    - Describes the purpose: To launch workflow manually or automatically each Monday at 9:00 PM.

#### 1.2 Bestseller Retrieval

- **Overview:** Queries Algolia product index to retrieve the weekly bestseller product (top-ranked product).
- **Nodes Involved:**
  - Get weekly bestseller from Algolia
  - Sticky Note5

- **Node Details:**

  - **Get weekly bestseller from Algolia**
    - Type: HTTP Request
    - Configuration:
      - POST to Algolia search API endpoint for product index.
      - Body parameter: `hitsPerPage=1` to get only top product.
      - Auth: Algolia API key (Bearer token and custom HTTP authentication)
    - Input: Trigger node output
    - Output: JSON with product hits array
    - Edge cases:
      - Authorization failure (invalid API keys)
      - Network errors
      - Empty hits if no products or misconfigured index
    - Sticky Note5 explains the purpose: retrieve top-ranked product.

#### 1.3 Video Existence Check

- **Overview:** Checks if the bestseller already has a video to avoid redundant processing.
- **Nodes Involved:**
  - If no video for that product
  - No Operation, do nothing
  - Sticky Note6, Sticky Note

- **Node Details:**

  - **If no video for that product**
    - Type: If node
    - Condition: Checks if `hits[0].hasVideo` is false
    - Input: Bestseller product JSON
    - Output: 
      - True branch: product has no video, proceed
      - False branch: product has video, skip generation
    - Edge cases: Missing `hasVideo` field or unexpected data format.

  - **No Operation, do nothing**
    - Type: NoOp
    - Runs if product already has a video
    - Purpose: Explicitly do nothing to end flow here.

  - **Sticky Note6**
    - Explains: Avoid regenerating video if product already has one.

  - **Sticky Note**
    - Reiterates: Workflow does nothing if video exists.

#### 1.4 Image Validation

- **Overview:** Validates presence and accessibility of product image before video generation.
- **Nodes Involved:**
  - Image URL is present
  - Check image availability
  - Image is present and valid
  - Tell admin that bestseller has no image
  - Tell admin that bestseller has broken image
  - Sticky Note1, Sticky Note2, Sticky Note7

- **Node Details:**

  - **Image URL is present**
    - Type: If node
    - Condition: Checks if `hits[0].images[0]` is not empty
    - True: proceed to check image accessibility
    - False: send notification to admin about missing image
    - Edge cases: image array missing or empty.

  - **Check image availability**
    - Type: HTTP Request
    - Sends GET request to the image URL
    - Options: Full HTTP response for status code capture
    - On error: continues error output (do not fail workflow)
    - Edge cases: network errors, 404 or other HTTP errors.

  - **Image is present and valid**
    - Type: If node
    - Condition: HTTP response status code equals 200
    - True: proceed to video generation steps
    - False: notify admin of broken image

  - **Tell admin that bestseller has no image**
    - Type: Gmail node
    - Sends email to admin@example.com
    - Subject and body include product SKU
    - Purpose: Alert human to missing product image

  - **Tell admin that bestseller has broken image**
    - Type: Gmail node
    - Similar to above, alerts admin about inaccessible image

  - **Sticky Notes:**
    - Sticky Note1: Error handling for missing image
    - Sticky Note2: Error handling for broken image
    - Sticky Note7: Purpose of image validation is to ensure image presence and accessibility

#### 1.5 Video Generation

- **Overview:** Converts the product image to base64, submits it to Google VEO 3 API for cinematic video generation, and waits for completion.
- **Nodes Involved:**
  - Convert image to base64 for VEO 3
  - Generate video with Google VEO 3
  - Wait for video generation
  - Checking if we're done
  - Finished ?
  - Wait more
  - Sticky Note8

- **Node Details:**

  - **Convert image to base64 for VEO 3**
    - Type: Extract From File
    - Operation: binaryToProperty (convert image binary to base64 property)
    - Input: product image from previous node (fetched image data)
    - Output: base64 encoded image string for API

  - **Generate video with Google VEO 3**
    - Type: HTTP Request
    - Sends POST request to Google Vertex AI VEO 3.0 long-running predict endpoint
    - Body includes:
      - Prompt describing cinematic video style
      - Image encoded as base64 JPEG
      - Parameters like aspect ratio (9:16), duration (6s), sample count, resize mode, and storage URI on Google Cloud Storage bucket
    - Auth: Google Cloud OAuth2 credentials
    - Output: Operation name to track generation progress

  - **Wait for video generation**
    - Type: Wait
    - Waits 60 seconds before checking status

  - **Checking if we're done**
    - Type: HTTP Request
    - Calls Google VEO 3 API operation fetch endpoint with operation name
    - Auth: Google Cloud OAuth2
    - Output: JSON including `done` boolean and video info if complete

  - **Finished ?**
    - Type: If node
    - Checks if `done` is true
    - True branch: proceed to extract video info
    - False branch: wait more

  - **Wait more**
    - Type: Wait
    - Waits 30 seconds then loops back to check status again (polling)

  - **Sticky Note8**
    - Describes purpose: Transform product image into promotional video using Google VEO 3.0 AI

#### 1.6 Video Handling and Upload

- **Overview:** Extracts video storage details, downloads the video from Google Cloud Storage, uploads it to Supabase storage, aggregates data for Algolia update.
- **Nodes Involved:**
  - Get video name and storage bucket
  - Aggregate
  - Downloading the MP4 file
  - Drop video in Supabase Bucket
  - Sticky Note9

- **Node Details:**

  - **Get video name and storage bucket**
    - Type: Set node
    - Parses the Google Cloud Storage URI (`gs://bucketName/objectName`) from video response
    - Assigns `bucketName` and `objectName` variables for subsequent download

  - **Aggregate**
    - Type: Aggregate
    - Aggregates all incoming item data into a single array (prepares for batch processing)

  - **Downloading the MP4 file**
    - Type: HTTP Request
    - GET request to Google Cloud Storage public URL for the video file
    - Auth: Google Cloud Storage OAuth2
    - Downloads the MP4 binary data

  - **Drop video in Supabase Bucket**
    - Type: HTTP Request
    - POST request to Supabase Storage API
    - Uploads the MP4 binary data to a Supabase storage bucket under product objectID filename
    - Auth: Supabase API key
    - Content-Type: binary data

  - **Sticky Note9**
    - Explains: Updates Algolia product to mark video presence and add video public URL for publishing on store

#### 1.7 Algolia Update

- **Overview:** Updates the Algolia product record to flag it as having a video and attach the new video URL.
- **Nodes Involved:**
  - Index in Algolia

- **Node Details:**

  - **Index in Algolia**
    - Type: HTTP Request
    - POST request to Algolia batch update API for product index
    - Body includes partialUpdateObject with:
      - `objectID` of the product
      - `videos` array containing the Supabase public video URL
      - `hasVideo` set to true
    - Auth: Algolia HTTP Custom Auth
    - Edge cases: API errors, auth failures, data conflicts

#### 1.8 Notification

- **Overview:** Sends email notifications to administrators when the bestseller product is missing an image or has a broken image.
- **Nodes Involved:**
  - Tell admin that bestseller has no image
  - Tell admin that bestseller has broken image

- **Node Details:**

  - Both nodes:
    - Type: Gmail node
    - Send email to admin@example.com
    - Subject and message include product SKU
    - Auth: Gmail OAuth2
    - Edge cases: Gmail auth errors, email delivery issues

---

### 3. Summary Table

| Node Name                          | Node Type                 | Functional Role                                        | Input Node(s)                     | Output Node(s)                     | Sticky Note                                                                                          |
|-----------------------------------|---------------------------|-------------------------------------------------------|----------------------------------|-----------------------------------|-----------------------------------------------------------------------------------------------------|
| Every week on monday morning       | Schedule Trigger          | Weekly trigger to start workflow                       | None                             | Get weekly bestseller from Algolia (indirect via manual trigger) | Sticky Note4: Workflow trigger explanation                                                          |
| Test Trigger while building the workflow | Manual Trigger            | Manual trigger for testing                             | None                             | Get weekly bestseller from Algolia | Sticky Note4                                                                                         |
| Get weekly bestseller from Algolia | HTTP Request             | Retrieves top-selling product from Algolia            | Trigger nodes                   | If no video for that product       | Sticky Note5: Bestseller identification                                                            |
| If no video for that product       | If                       | Checks if product already has a video                  | Get weekly bestseller from Algolia | Image URL is present / No Operation | Sticky Note6: Video presence check <br> Sticky Note: Do nothing if video exists                      |
| No Operation, do nothing           | NoOp                     | Ends flow if video exists                              | If no video for that product     | None                              | Sticky Note: See above                                                                               |
| Image URL is present               | If                       | Checks if product image URL exists                      | If no video for that product     | Check image availability / Tell admin no image | Sticky Note7: Image URL validation                                                                |
| Check image availability           | HTTP Request             | Validates image URL accessibility                       | Image URL is present             | Image is present and valid / Tell admin broken image | Sticky Note1: Missing image error handling <br> Sticky Note2: Broken image error handling           |
| Image is present and valid         | If                       | Checks HTTP status 200 for image                        | Check image availability         | Convert image to base64 for VEO 3 / Tell admin broken image | Sticky Note7                                                                                         |
| Tell admin that bestseller has no image | Gmail                   | Notification for missing image                          | Image URL is present (false)     | None                              | Sticky Note1                                                                                         |
| Tell admin that bestseller has broken image | Gmail                   | Notification for broken image                           | Check image availability (false)/ Image is present and valid (false) | None                              | Sticky Note2                                                                                         |
| Convert image to base64 for VEO 3  | Extract From File        | Converts image binary to base64 for video generation   | Image is present and valid       | Generate video with Google VEO 3   |                                                                                                     |
| Generate video with Google VEO 3   | HTTP Request             | Sends image and prompt to Google VEO 3 for video gen   | Convert image to base64 for VEO 3 | Wait for video generation          | Sticky Note8: Video generation purpose                                                             |
| Wait for video generation          | Wait                     | Waits 60 seconds before checking video generation status | Generate video with Google VEO 3 | Checking if we're done             |                                                                                                     |
| Checking if we're done             | HTTP Request             | Checks Google VEO 3 operation status                    | Wait for video generation        | Finished ?                       |                                                                                                     |
| Finished ?                        | If                       | Determines if video generation is complete             | Checking if we're done           | Get video name and storage bucket / Wait more |                                                                                                     |
| Wait more                        | Wait                     | Waits 30 seconds before re-checking video status       | Finished ? (false branch)        | Checking if we're done             |                                                                                                     |
| Get video name and storage bucket  | Set                      | Parses storage bucket and object name from video URI   | Finished ? (true branch)         | Aggregate                        |                                                                                                     |
| Aggregate                        | Aggregate                | Aggregates video data for batch processing              | Get video name and storage bucket | Downloading the MP4 file          |                                                                                                     |
| Downloading the MP4 file           | HTTP Request             | Downloads generated MP4 video from Google Cloud Storage | Aggregate                      | Drop video in Supabase Bucket     |                                                                                                     |
| Drop video in Supabase Bucket      | HTTP Request             | Uploads MP4 video to Supabase storage                   | Downloading the MP4 file          | Index in Algolia                  |                                                                                                     |
| Index in Algolia                  | HTTP Request             | Updates Algolia product record with video URL and flag | Drop video in Supabase Bucket    | None                              | Sticky Note9: Algolia product update purpose                                                       |
| Sticky Note                      | Sticky Note              | Notes if video exists, workflow does nothing            | None                           | None                             | "If there is already a video for that product, we do nothing"                                      |
| Sticky Note1                     | Sticky Note              | Notes error handling for missing image                  | None                           | None                             | "Error handling: a best selling product has no image, tell a human"                                |
| Sticky Note2                     | Sticky Note              | Notes error handling for broken image                   | None                           | None                             | "Error handling: a best selling product has a broken image, tell a human"                          |
| Sticky Note3                     | Sticky Note              | Overview and explanation of the workflow                | None                           | None                             | Long description of workflow purpose and requirements                                              |
| Sticky Note4                     | Sticky Note              | Explanation of workflow trigger                          | None                           | None                             | "Workflow Trigger: manual or scheduled launch"                                                    |
| Sticky Note5                     | Sticky Note              | Explanation of bestseller identification                 | None                           | None                             | "Query Algolia index for top-ranked product"                                                      |
| Sticky Note6                     | Sticky Note              | Explanation of video presence check                      | None                           | None                             | "Avoid regenerating video if it exists"                                                          |
| Sticky Note7                     | Sticky Note              | Explanation of image URL validation                       | None                           | None                             | "Ensure product image URL exists and is accessible"                                              |
| Sticky Note8                     | Sticky Note              | Explanation of video generation block                     | None                           | None                             | "Transform product image into promotional video using Google VEO 3.0"                            |
| Sticky Note9                     | Sticky Note              | Explanation of Algolia product update                      | None                           | None                             | "Mark product as having video and add public URL"                                                |
| Sticky Note10                    | Sticky Note              | Link to video walkthrough                                 | None                           | None                             | "# 🎙️ Video Walkthrough @[youtube](sjzush0DYFU)"                                                 |
| Sticky Note12                    | Sticky Note              | Contact info for questions or feedback                    | None                           | None                             | "📧 emir.belkahia@gmail.com  \n💼 [linkedin.com/in/emirbelkahia](https://www.linkedin.com/in/emirbelkahia/)" |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**
   - Add a **Schedule Trigger** node named `Every week on monday morning`.
     - Set to trigger every week on Monday at 21:00 (9:00 PM).
     - Leave disabled initially for manual testing.
   - Add a **Manual Trigger** node named `Test Trigger while building the workflow` for manual workflow start.

2. **Add Algolia Bestseller Query:**
   - Add an **HTTP Request** node named `Get weekly bestseller from Algolia`.
   - Configure as POST to Algolia search API endpoint of your product index:  
     `https://ALGOLIA_APP_ID.algolia.net/1/indexes/unicorns_prod_products/query`
   - Body: `{ "params": "hitsPerPage=1" }`
   - Set authentication:
     - Use HTTP Bearer Auth with Algolia API token.
     - Use HTTP Custom Auth for Algolia app ID.
   - Connect both triggers to this node.

3. **Add Video Existence Check:**
   - Add an **If** node `If no video for that product`.
   - Condition: Check if `{{$json.hits[0].hasVideo}}` is `false`.
   - True (no video): proceed to image validation.
   - False (video exists): connect to a **No Operation** node `No Operation, do nothing`.

4. **Add Image URL Presence Check:**
   - Add an **If** node `Image URL is present`.
   - Condition: Check if `{{$json.hits[0].images[0]}}` is not empty.
   - True: continue to check image availability.
   - False: connect to a **Gmail** node `Tell admin that bestseller has no image`.
     - Configure Gmail OAuth2 credentials.
     - Send email to admin with subject and body including product SKU.

5. **Add Image Availability Check:**
   - Add an **HTTP Request** node `Check image availability`.
   - GET request to `{{$json.hits[0].images[0]}}`.
   - Enable full HTTP response capture.
   - On error, continue workflow.
   - Connect true branch to **If** node `Image is present and valid`.

6. **Add Image Validity Check:**
   - Add an **If** node `Image is present and valid`.
   - Condition: Check if statusCode equals 200.
   - True: proceed to video generation.
   - False: connect to **Gmail** node `Tell admin that bestseller has broken image`.
     - Similar configuration as missing image notification.

7. **Convert Image to Base64:**
   - Add an **Extract From File** node `Convert image to base64 for VEO 3`.
   - Operation: binary to property.
   - Input: product image binary data.

8. **Generate Video with Google VEO 3:**
   - Add an **HTTP Request** node `Generate video with Google VEO 3`.
   - POST to Google VEO 3 predictLongRunning endpoint, e.g.:  
     `https://us-central1-aiplatform.googleapis.com/v1/projects/PROJECT-ID-GOOGLE-CLOUD/locations/us-central1/publishers/google/models/veo-3.0-generate-001:predictLongRunning`
   - Body JSON with prompt, base64 image, parameters (aspect ratio, duration, storage URI).
   - Use Google Cloud OAuth2 credentials.
   - Store operation name from response.

9. **Wait for Video Generation:**
   - Add a **Wait** node `Wait for video generation` for 60 seconds.

10. **Check Video Generation Status:**
    - Add an **HTTP Request** node `Checking if we're done`.
    - POST to Google API fetchPredictOperation endpoint with operation name.
    - Use Google Cloud OAuth2 credentials.

11. **Evaluate Completion:**
    - Add an **If** node `Finished ?`.
    - Condition: `$json.done` is true.
    - If true: proceed to get video info.
    - If false: connect to **Wait** node `Wait more` for 30 seconds.
    - From `Wait more`, loop back to `Checking if we're done` (polling).

12. **Extract Video Storage Info:**
    - Add a **Set** node `Get video name and storage bucket`.
    - Parse `bucketName` and `objectName` from `response.videos[0].gcsUri` using regex.

13. **Aggregate Video Data:**
    - Add an **Aggregate** node `Aggregate`.
    - Aggregate all item data into an array.

14. **Download Video from Google Cloud Storage:**
    - Add an **HTTP Request** node `Downloading the MP4 file`.
    - GET request to:  
      `https://storage.googleapis.com/{{ $json.data[0].bucketName }}/{{ $json.data[0].objectName }}`
    - Use Google Cloud Storage OAuth2 credentials.

15. **Upload Video to Supabase Storage:**
    - Add an **HTTP Request** node `Drop video in Supabase Bucket`.
    - POST to Supabase storage API endpoint:  
      `https://YOUR_SUPABASE_PROJECT.supabase.co/storage/v1/object/unicorns-images/{{ $('Get weekly bestseller from Algolia').item.json.hits[0].objectID }}.mp4`
    - Send binary MP4 data.
    - Use Supabase API key credentials.

16. **Update Algolia Product Record:**
    - Add an **HTTP Request** node `Index in Algolia`.
    - POST to Algolia batch API:  
      `https://ALGOLIA_APP_ID-dsn.algolia.net/1/indexes/unicorns_prod_products/batch`
    - Body includes partial update to set:
      - `hasVideo`=true
      - `videos` array includes Supabase public URL
    - Use Algolia HTTP Custom Auth credentials.

17. **Add Notification Nodes:**
    - Configure Gmail nodes for missing and broken image alerts, as above.

18. **Add Sticky Notes for Documentation:**
    - Add sticky notes at relevant positions to describe workflow blocks and notes as per original workflow.

19. **Test Workflow:**
    - Use manual trigger to verify each step.
    - Once verified, enable schedule trigger for automatic weekly runs.

---

### 5. General Notes & Resources

| Note Content                                                                                                                         | Context or Link                                                                                                     |
|--------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| Automated Weekly Video Generation for E-commerce Bestsellers leveraging Algolia and Google VEO 3.0 AI.                              | Workflow description sticky note summarizing purpose and requirements.                                             |
| Use manual execution during setup to test workflow immediately instead of waiting for scheduled runs.                                | Sticky Note3 content                                                                                               |
| VEO 3.0 outputs videos to Google Cloud Storage; Supabase is used for simpler storage pricing and frontend integration.              | Sticky Note3 explanation                                                                                           |
| Gmail node sends notifications to admin@example.com when bestseller images are missing or broken.                                   | Notification nodes and Sticky Notes 1 & 2                                                                           |
| Video generation prompt uses cinematic style with soft white lighting, neutral background, shallow depth of field, centered framing.| In Generate video with Google VEO 3 node JSON body                                                                 |
| Contact for questions or feedback: emir.belkahia@gmail.com, LinkedIn [linkedin.com/in/emirbelkahia](https://www.linkedin.com/in/emirbelkahia/) | Sticky Note12                                                                                                      |
| Video walkthrough available at YouTube video ID: sjzush0DYFU                                                                         | Sticky Note10                                                                                                      |

---

**Disclaimer:** The text and workflow analysis above are based exclusively on an automated n8n workflow. All data handled is legal and public. No illegal or protected content is involved.