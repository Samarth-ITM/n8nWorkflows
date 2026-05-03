Meta Ads Bulk Launcher with Google Drive integration and Slack Notifications

https://n8nworkflows.xyz/workflows/meta-ads-bulk-launcher-with-google-drive-integration-and-slack-notifications-8276


# Meta Ads Bulk Launcher with Google Drive integration and Slack Notifications

### 1. Workflow Overview

This workflow, titled **Meta Ads Bulk Launcher with Google Drive integration and Slack Notifications**, automates the bulk launching of Meta (Facebook) Ads using assets stored on Google Drive. It processes images and videos from Google Drive folders, creates corresponding ad creatives and ad sets in Meta Ads Manager, publishes them, and sends Slack notifications for success and error events. The workflow is designed for marketing teams or agencies managing multiple ad campaigns with bulk media assets, ensuring streamlined ad creation, error handling, and real-time notifications.

The workflow is structured into these logical blocks:

- **1.1 Input Reception**: Captures ad campaign input via a form trigger.
- **1.2 Data Validation and Preparation**: Validates Google Drive folder access, checks naming conventions, downloads assets, and separates images/videos.
- **1.3 Ad Copy and Account Setup**: Formats ad copy, retrieves and confirms Facebook Ad Account ID.
- **1.4 Attribution and Ad Set Creation**: Chooses attribution model and creates multiple ad sets.
- **1.5 Asset Processing and Creative Creation**: Processes images and videos, renames binaries, and creates creatives on Meta.
- **1.6 Publishing and Post-Processing**: Publishes ads to Facebook and manages waiting loops.
- **1.7 Merging and Linking**: Merges creative data and links creatives with ad sets.
- **1.8 Notifications and Error Handling**: Sends Slack notifications about success or errors and manages workflow stop on errors.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Receives user input via a form and initiates the workflow.
- **Nodes Involved:** `Submit Form`, `Format Ad Copy`.
- **Node Details:**

  - **Submit Form**
    - Type: Form Trigger  
    - Role: Entry point capturing input data such as campaign details, folder links, attribution, etc.  
    - Config: Webhook ID set for external form integration.  
    - Input: External form submission webhook.  
    - Output: Passes form data to `Format Ad Copy`.  
    - Edge cases: Missing required form fields, invalid webhook trigger.

  - **Format Ad Copy**
    - Type: Code  
    - Role: Processes and formats the ad copy text from form input, possibly sanitizing or structuring it for Facebook Ads.  
    - Config: JavaScript code for formatting.  
    - Input: Data from `Submit Form`.  
    - Output: Passes formatted ad copy to `Get Ad Account ID`.  
    - Edge cases: Expression errors if input format unexpected.

#### 1.2 Data Validation and Preparation

- **Overview:** Validates access to Google Drive folder, checks naming conventions, downloads media files, and determines media type.
- **Nodes Involved:** `Get Ad Account ID`, `Confirm Ad Account`, `Analyze Folder`, `Check Naming`, `No Access to Folder`, `Download Files`, `Video or Image`, `Error - Filename`, `Stop and Error`.
- **Node Details:**

  - **Get Ad Account ID**
    - Type: Google Sheets  
    - Role: Retrieves the Facebook Ad Account ID from a spreadsheet based on input.  
    - Config: Google Sheets node with retry on fail and wait between tries to handle API limits or delays.  
    - Input: Formatted ad copy data.  
    - Output: To `Confirm Ad Account`.  
    - Edge cases: Sheet not found, permission errors.

  - **Confirm Ad Account**
    - Type: Filter  
    - Role: Validates the retrieved ad account ID exists and is valid.  
    - Input: Data from `Get Ad Account ID`.  
    - Output: If valid, branches to `Analyze Folder` and `Choose Correct Attribution`.  
    - Edge cases: Invalid or missing Ad Account ID.

  - **Analyze Folder**
    - Type: Google Drive  
    - Role: Lists contents of the specified Google Drive folder to verify access and enumerate files.  
    - Input: Confirmed ad account data.  
    - Output: To `Check Naming`.  
    - Edge cases: Folder inaccessible or missing.

  - **Check Naming**
    - Type: If  
    - Role: Checks if files in the folder follow expected naming conventions.  
    - Input: Folder contents.  
    - Output: If naming invalid, routes to `No Access to Folder`; if valid, proceeds to `Download Files`.  
    - Edge cases: Unexpected file names or formats.

  - **No Access to Folder**
    - Type: Stop and Error  
    - Role: Stops workflow if folder access/naming is invalid to avoid further processing.

  - **Download Files**
    - Type: Google Drive  
    - Role: Downloads files from the Google Drive folder for processing.  
    - Input: Validated folder files.  
    - Output: To `Video or Image`.  
    - Edge cases: Download failures, file corruption.

  - **Video or Image**
    - Type: Switch  
    - Role: Determines if the downloaded file is a video or image or if there's an error in the filename.  
    - Input: Downloaded files.  
    - Output: Routes to `Format Naming - Image`, `Format Naming - Video`, or `Error - Filename`.  
    - Edge cases: Unsupported file types, malformed filenames.

  - **Error - Filename**
    - Type: Slack  
    - Role: Sends Slack notification about filename errors.  
    - Input: Files flagged by `Video or Image`.  
    - Output: To `Stop and Error`.  
    - Edge cases: Slack API errors.

  - **Stop and Error**
    - Type: Stop and Error  
    - Role: Terminates workflow due to critical error.

#### 1.3 Ad Copy and Account Setup

- **Overview:** Continues with account confirmation and attribution selection.
- **Nodes Involved:** `Choose Correct Attribution`, `Create Ad_set (1dc)`, `Create Ad_set (7dc)`, `Create Ad_set (7dc1dv)`.
- **Node Details:**

  - **Choose Correct Attribution**
    - Type: Switch  
    - Role: Selects one of three attribution models based on input parameters.  
    - Input: Confirmed ad account and form data.  
    - Output: Routes to one of the three `Create Ad_set` nodes.  
    - Edge cases: Invalid attribution input.

  - **Create Ad_set (1dc), Create Ad_set (7dc), Create Ad_set (7dc1dv)**
    - Type: Facebook Graph API  
    - Role: Creates ad sets in Facebook Ads Manager according to chosen attribution.  
    - Config: Uses Facebook OAuth credentials, retries on failure.  
    - Input: Attribution selection data.  
    - Output: All connect to `Format ID`.  
    - Edge cases: API rate limits, invalid ad set parameters, auth failures.

#### 1.4 Attribution and Ad Set Creation

- **Overview:** Formats Facebook ad set IDs and merges creatives.
- **Nodes Involved:** `Format ID`, `Merge All`.
- **Node Details:**

  - **Format ID**
    - Type: Set  
    - Role: Formats or prepares ad set IDs received from Facebook API for further processing.  
    - Input: Ad set creation response.  
    - Output: To `Merge All`.  
    - Edge cases: Missing or malformed ID data.

  - **Merge All**
    - Type: Merge  
    - Role: Combines multiple streams of creative data and ad set data into a single stream for linking.  
    - Input: Creative merges and formatted ad set IDs.  
    - Output: To `linking ad_creatives with adsets`.  
    - Edge cases: Data mismatch or missing inputs.

#### 1.5 Asset Processing and Creative Creation

- **Overview:** Processes media files, renames binaries, creates ad creatives for images and videos, and merges creative data.
- **Nodes Involved:** `Format Naming - Image`, `Format Naming - Video`, `Rename Image Binary Top Image`, `Rename Image Binary Top Image1`, `Loop Over Items1`, `Loop Over Items`, `Creating ad_Creatives for images`, `Creating ad_Creatives for videos`, `Merge Creative`.
- **Node Details:**

  - **Format Naming - Image**, **Format Naming - Video**
    - Type: Set  
    - Role: Standardizes naming conventions for images and videos before processing.  
    - Input: Files from `Video or Image`.  
    - Output: To `Rename Image Binary Top Image` or `Rename Image Binary Top Image1` respectively.  
    - Edge cases: Naming conflicts.

  - **Rename Image Binary Top Image**, **Rename Image Binary Top Image1**
    - Type: Code  
    - Role: Modifies binary data of media files to set correct filenames or metadata for Facebook upload.  
    - Input: Formatted naming data.  
    - Output: To `Loop Over Items1` (images) or `Loop Over Items` (videos).  
    - Edge cases: Binary data corruption.

  - **Loop Over Items1** (images), **Loop Over Items** (videos)
    - Type: Split In Batches  
    - Role: Processes media files in batches to avoid API rate limits or memory overload.  
    - Input: Renamed media files.  
    - Output: To `Creating ad_Creatives for images` and `Publish to Facebook` or `Creating ad_Creatives for videos` and `Publish to Facebook1`.  
    - Edge cases: Large batch sizes causing timeout.

  - **Creating ad_Creatives for images**, **Creating ad_Creatives for videos**
    - Type: Facebook Graph API  
    - Role: Creates ad creatives in Facebook Ads Manager for images and videos.  
    - Config: Retries on failure, uses Facebook OAuth.  
    - Input: Batch media data.  
    - Output: To `Merge Creative`.  
    - Edge cases: Upload failures, invalid media.

  - **Merge Creative**
    - Type: Merge  
    - Role: Aggregates creatives from image and video creation nodes into a unified data stream.  
    - Input: Outputs from both creatives nodes.  
    - Output: To `Merge All`.  
    - Edge cases: Missing or incompatible creative data.

#### 1.6 Publishing and Post-Processing

- **Overview:** Publishes ads and manages wait loops to handle asynchronous Facebook responses.
- **Nodes Involved:** `Publish to Facebook`, `Publish to Facebook1`, `Wait1`, `Wait`.
- **Node Details:**

  - **Publish to Facebook**, **Publish to Facebook1**
    - Type: Facebook Graph API  
    - Role: Publishes the image and video ads to Facebook respectively.  
    - Config: Retry on fail, wait between tries for rate limiting.  
    - Input: Batches from loops.  
    - Output: To `Wait1` and `Wait`.  
    - Edge cases: API limit exceeded, publishing errors.

  - **Wait1**, **Wait**
    - Type: Wait  
    - Role: Introduces delays to allow Facebook to process publishing requests before continuing.  
    - Config: Webhook IDs for synchronization.  
    - Input: Publishing nodes.  
    - Output: Loops back to batch nodes (`Loop Over Items1`, `Loop Over Items`).  
    - Edge cases: Timeouts if wait is insufficient.

#### 1.7 Merging and Linking

- **Overview:** Links ad creatives with ad sets and sends success notification.
- **Nodes Involved:** `linking ad_creatives with adsets`, `Success Notification`.
- **Node Details:**

  - **linking ad_creatives with adsets**
    - Type: Facebook Graph API  
    - Role: Associates created ad creatives with their respective ad sets to finalize ad setup.  
    - Config: Multiple retries, Facebook OAuth.  
    - Input: Merged creatives and ad sets.  
    - Output: To `Success Notification`.  
    - Edge cases: Link failures, API errors.

  - **Success Notification**
    - Type: Slack  
    - Role: Sends a Slack message upon successful ad creation and linking.  
    - Config: Slack webhook configured.  
    - Input: Confirmation from linking node.  
    - Edge cases: Slack webhook failures.

#### 1.8 Notifications and Error Handling

- **Overview:** Handles errors globally and specific filename errors with Slack notifications and workflow stops.
- **Nodes Involved:** `Error Trigger`, `Error - General`, `Error - Filename`, `Stop and Error`, `Error - Filename`.
- **Node Details:**

  - **Error Trigger**
    - Type: Error Trigger  
    - Role: Captures any unhandled errors in the workflow execution.  
    - Output: Routes to `Error - General`.  
    - Edge cases: None (designed to catch all errors).

  - **Error - General**
    - Type: Slack  
    - Role: Sends a Slack message detailing the general error encountered.  
    - Input: From `Error Trigger`.  
    - Edge cases: Slack API errors.

  - **Error - Filename**
    - Type: Slack  
    - Role: Sends Slack notifications about filename-specific errors encountered during media processing.  
    - Input: From `Video or Image` node’s error branch.  
    - Output: To `Stop and Error`.  
    - Edge cases: Slack webhook failures.

  - **Stop and Error**
    - Type: Stop and Error  
    - Role: Terminates workflow execution upon a critical error.

---

### 3. Summary Table

| Node Name                          | Node Type             | Functional Role                                  | Input Node(s)                      | Output Node(s)                            | Sticky Note                            |
|-----------------------------------|-----------------------|-------------------------------------------------|----------------------------------|------------------------------------------|--------------------------------------|
| Submit Form                       | Form Trigger          | Entry point, receives ad campaign input         | -                                | Format Ad Copy                           |                                      |
| Format Ad Copy                   | Code                  | Formats ad copy from form input                  | Submit Form                     | Get Ad Account ID                        |                                      |
| Get Ad Account ID                | Google Sheets         | Retrieves Facebook Ad Account ID                 | Format Ad Copy                 | Confirm Ad Account                       |                                      |
| Confirm Ad Account               | Filter                | Validates Ad Account ID                          | Get Ad Account ID              | Analyze Folder, Choose Correct Attribution |                                      |
| Analyze Folder                  | Google Drive          | Lists folder contents to check access            | Confirm Ad Account            | Check Naming                            |                                      |
| Check Naming                   | If                    | Validates file naming conventions                 | Analyze Folder                | No Access to Folder, Download Files      |                                      |
| No Access to Folder              | Stop and Error        | Stops workflow on invalid folder access          | Check Naming                 | -                                        |                                      |
| Download Files                  | Google Drive          | Downloads media assets                            | Check Naming                 | Video or Image                          |                                      |
| Video or Image                 | Switch                | Determines media type (image/video/error)         | Download Files               | Format Naming - Image, Format Naming - Video, Error - Filename |                                      |
| Format Naming - Image           | Set                   | Formats image file naming                          | Video or Image               | Rename Image Binary Top Image            |                                      |
| Format Naming - Video           | Set                   | Formats video file naming                          | Video or Image               | Rename Image Binary Top Image1           |                                      |
| Rename Image Binary Top Image   | Code                  | Renames image binary file for upload              | Format Naming - Image         | Loop Over Items1                         |                                      |
| Rename Image Binary Top Image1  | Code                  | Renames video binary file for upload              | Format Naming - Video         | Loop Over Items                         |                                      |
| Loop Over Items1                | Split In Batches      | Batches image files for processing                 | Rename Image Binary Top Image | Creating ad_Creatives for images, Publish to Facebook |                                      |
| Loop Over Items                 | Split In Batches      | Batches video files for processing                 | Rename Image Binary Top Image1 | Creating ad_Creatives for videos, Publish to Facebook1 |                                      |
| Creating ad_Creatives for images | Facebook Graph API    | Creates Facebook ad creatives for images          | Loop Over Items1             | Merge Creative                          |                                      |
| Creating ad_Creatives for videos | Facebook Graph API    | Creates Facebook ad creatives for videos          | Loop Over Items              | Merge Creative                          |                                      |
| Merge Creative                 | Merge                 | Merges image and video creatives                   | Creating ad_Creatives for images, Creating ad_Creatives for videos | Merge All                              |                                      |
| Merge All                     | Merge                 | Merges all creative and ad set data                | Merge Creative, Format ID    | linking ad_creatives with adsets         |                                      |
| linking ad_creatives with adsets | Facebook Graph API    | Links creatives to ad sets                         | Merge All                   | Success Notification                    |                                      |
| Success Notification          | Slack                 | Sends Slack success notification                    | linking ad_creatives with adsets | -                                      |                                      |
| Choose Correct Attribution      | Switch                | Selects attribution model and routes accordingly  | Confirm Ad Account          | Create Ad_set (1dc), Create Ad_set (7dc), Create Ad_set (7dc1dv) |                                      |
| Create Ad_set (1dc)            | Facebook Graph API    | Creates an ad set with 1dc attribution              | Choose Correct Attribution   | Format ID                              |                                      |
| Create Ad_set (7dc)            | Facebook Graph API    | Creates an ad set with 7dc attribution              | Choose Correct Attribution   | Format ID                              |                                      |
| Create Ad_set (7dc1dv)         | Facebook Graph API    | Creates an ad set with 7dc1dv attribution           | Choose Correct Attribution   | Format ID                              |                                      |
| Format ID                     | Set                   | Formats ad set IDs                                 | Create Ad_set (1dc), Create Ad_set (7dc), Create Ad_set (7dc1dv) | Merge All                              |                                      |
| Publish to Facebook           | Facebook Graph API    | Publishes image ads                                | Loop Over Items1, Loop Over Items | Wait1                                |                                      |
| Publish to Facebook1          | Facebook Graph API    | Publishes video ads                                | Loop Over Items, Loop Over Items1 | Wait                                 |                                      |
| Wait1                        | Wait                  | Waits after publishing image ads                   | Publish to Facebook          | Loop Over Items1                        |                                      |
| Wait                         | Wait                  | Waits after publishing video ads                   | Publish to Facebook1         | Loop Over Items                         |                                      |
| Error Trigger                | Error Trigger         | Catches all errors                                | -                          | Error - General                        |                                      |
| Error - General             | Slack                 | Sends Slack notification on general errors         | Error Trigger               | -                                      |                                      |
| Error - Filename            | Slack                 | Sends Slack notification about filename errors     | Video or Image              | Stop and Error                        |                                      |
| Stop and Error              | Stop and Error        | Stops workflow on critical errors                   | Error - Filename            | -                                      |                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node**
   - Name: `Submit Form`
   - Set up a webhook URL to receive campaign input (e.g., folder links, ad copy, attribution).
   - No credentials needed.
   - Connect output to `Format Ad Copy`.

2. **Add a Code Node to Format Ad Copy**
   - Name: `Format Ad Copy`
   - Write JavaScript code to sanitize and structure ad copy from form data.
   - Connect output to `Get Ad Account ID`.

3. **Add a Google Sheets Node to Get Facebook Ad Account ID**
   - Name: `Get Ad Account ID`
   - Configure to read the sheet containing Ad Account IDs.
   - Credentials: Google Sheets OAuth.
   - Enable retry on failure with 5-second wait.
   - Connect output to `Confirm Ad Account`.

4. **Add a Filter Node to Confirm Ad Account**
   - Name: `Confirm Ad Account`
   - Set condition to check if Ad Account ID is valid.
   - Connect true branch to `Analyze Folder` and `Choose Correct Attribution`.
   - Leave false branch unconnected or add error handling as desired.

5. **Add a Google Drive Node to Analyze Folder**
   - Name: `Analyze Folder`
   - Configure to list files in the specified Google Drive folder.
   - Credentials: Google Drive OAuth.
   - Connect output to `Check Naming`.

6. **Add an If Node to Check Naming**
   - Name: `Check Naming`
   - Implement condition to verify file naming conventions.
   - True branch: Connect to `Download Files`.
   - False branch: Connect to `No Access to Folder`.

7. **Add a Stop and Error Node**
   - Name: `No Access to Folder`
   - Configure to stop workflow with error message.

8. **Add a Google Drive Node to Download Files**
   - Name: `Download Files`
   - Configure to download files from the folder.
   - Connect output to `Video or Image`.

9. **Add a Switch Node to Determine Media Type**
   - Name: `Video or Image`
   - Configure to route files based on extension or metadata:
     - Image files to `Format Naming - Image`
     - Video files to `Format Naming - Video`
     - Invalid files to `Error - Filename`

10. **Add Slack Node for Filename Errors**
    - Name: `Error - Filename`
    - Configure Slack webhook to notify about naming errors.
    - Connect to `Stop and Error`.

11. **Add Stop and Error Node**
    - Name: `Stop and Error`
    - Stops workflow execution.

12. **Add Set Nodes to Format Naming**
    - Name: `Format Naming - Image` and `Format Naming - Video`
    - Set necessary properties to standardize naming.
    - Connect outputs respectively to `Rename Image Binary Top Image` and `Rename Image Binary Top Image1`.

13. **Add Code Nodes to Rename Image Binary**
    - Name: `Rename Image Binary Top Image` and `Rename Image Binary Top Image1`
    - Write JavaScript to rename binary files appropriately for upload.
    - Connect outputs to `Loop Over Items1` (images) and `Loop Over Items` (videos).

14. **Add Split In Batches Nodes**
    - Name: `Loop Over Items1` and `Loop Over Items`
    - Configure batch sizes to optimize API calls.
    - Connect outputs to `Creating ad_Creatives for images` and `Publish to Facebook` (images), or `Creating ad_Creatives for videos` and `Publish to Facebook1` (videos).

15. **Add Facebook Graph API Nodes to Create Ad Creatives**
    - Name: `Creating ad_Creatives for images` and `Creating ad_Creatives for videos`
    - Configure with Facebook OAuth credentials.
    - Enable retry on failure.
    - Connect outputs to `Merge Creative`.

16. **Add Merge Node for Creatives**
    - Name: `Merge Creative`
    - Merge outputs from image and video creatives.
    - Connect to `Merge All`.

17. **Add Set Node to Format Ad Set IDs**
    - Name: `Format ID`
    - Format ad set IDs received from Facebook.
    - Connect to `Merge All`.

18. **Add Merge Node for All Data**
    - Name: `Merge All`
    - Merge creative and ad set data.
    - Connect output to `linking ad_creatives with adsets`.

19. **Add Facebook Graph API Node to Link Creatives with Ad Sets**
    - Name: `linking ad_creatives with adsets`
    - Configure to associate creatives with ad sets.
    - Enable retry on failure.
    - Connect to `Success Notification`.

20. **Add Slack Node for Success Notification**
    - Name: `Success Notification`
    - Configure Slack webhook for success messages.

21. **Add Switch Node to Choose Attribution**
    - Name: `Choose Correct Attribution`
    - Configure to route based on attribution input.
    - Connect outputs to `Create Ad_set (1dc)`, `Create Ad_set (7dc)`, and `Create Ad_set (7dc1dv)`.

22. **Add Facebook Graph API Nodes to Create Ad Sets**
    - Name: `Create Ad_set (1dc)`, `Create Ad_set (7dc)`, `Create Ad_set (7dc1dv)`
    - Configure with Facebook OAuth credentials.
    - Enable retry on failure.
    - Connect outputs to `Format ID`.

23. **Add Facebook Graph API Nodes to Publish Ads**
    - Name: `Publish to Facebook` (for images) and `Publish to Facebook1` (for videos)
    - Configure with Facebook OAuth credentials.
    - Enable retry on failure.
    - Connect outputs to respective `Wait1` and `Wait`.

24. **Add Wait Nodes**
    - Name: `Wait1` and `Wait`
    - Configure appropriate delay durations.
    - Connect outputs back to batch loops (`Loop Over Items1` and `Loop Over Items`).

25. **Add Error Trigger Node**
    - Name: `Error Trigger`
    - Connect output to `Error - General`.

26. **Add Slack Node for General Errors**
    - Name: `Error - General`
    - Configure Slack webhook for error messages.

27. **Add Sticky Notes as Needed**
    - Add sticky notes for documentation or reminders within n8n interface.

---

### 5. General Notes & Resources

| Note Content                                                                                                                             | Context or Link                                        |
|------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------|
| Workflow tagged under "Templates" for easy reuse and sharing within the n8n community.                                                  | Workflow metadata                                      |
| The workflow uses retry and wait mechanisms extensively to handle API rate limits and asynchronous processing by Facebook APIs.         | Implementation detail in Facebook Graph API nodes     |
| Slack notifications cover both success and error events, ensuring real-time monitoring of the workflow execution status.                | Slack nodes                                            |
| Google Drive integration expects proper OAuth credentials and folder permissions; naming conventions must be strictly followed.         | Google Drive nodes and Check Naming node               |
| Facebook Graph API calls require a valid OAuth2 credential with appropriate permissions for Ads Management.                              | Facebook Graph API nodes                                |
| The workflow is inactive (`active: false`), requiring manual activation before use.                                                      | Workflow metadata                                      |
| For bulk processing, batch size in Split In Batches nodes should be configured according to API limits and server resources.             | Split In Batches nodes                                 |
| Webhook IDs are assigned to Wait nodes for synchronization; care must be taken not to reuse these IDs across workflows.                  | Wait nodes                                             |

---

This concludes the comprehensive technical reference and reproduction guide for the **Meta Ads Bulk Launcher with Google Drive integration and Slack Notifications** workflow.