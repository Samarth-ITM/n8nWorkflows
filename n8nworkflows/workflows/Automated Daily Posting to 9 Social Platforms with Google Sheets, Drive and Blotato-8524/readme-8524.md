Automated Daily Posting to 9 Social Platforms with Google Sheets, Drive and Blotato

https://n8nworkflows.xyz/workflows/automated-daily-posting-to-9-social-platforms-with-google-sheets--drive-and-blotato-8524


# Automated Daily Posting to 9 Social Platforms with Google Sheets, Drive and Blotato

### 1. Workflow Overview

This workflow automates the process of daily posting content (text, images, videos, and other media types) to nine different social media platforms using Google Sheets as the content source, Google Drive for media storage, and Blotato as the unified social media posting API. It is designed to:

- Check a Google Sheet periodically for new content marked "Ready to Post".
- Extract media URLs from Google Drive links.
- Upload media to Blotato.
- Post the media and captions to multiple social platforms simultaneously.
- Update the Google Sheet to mark content as "Posted" to avoid duplication.

The workflow is logically divided into these blocks:

- **1.1 Trigger & Input Fetching:** Scheduling and retrieving the next item ready to post from Google Sheets.
- **1.2 Media Preparation:** Extracting Google Drive file ID and uploading media to Blotato.
- **1.3 Multi-Platform Posting:** Posting the prepared content to nine social platforms via Blotato nodes.
- **1.4 Status Update & Reporting:** Marking the content as posted in Google Sheets and merging results for reporting.
- **1.5 Documentation & Setup Notes:** Sticky Notes throughout the workflow provide detailed instructions and platform-specific guidance.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Input Fetching

- **Overview:**  
  This block triggers the workflow every 3 hours, then fetches a single content entry from Google Sheets where the status is "Ready to Post".

- **Nodes Involved:**  
  - Check Every 3 Hours  
  - Ready to Post Image/Video

- **Node Details:**

  - **Check Every 3 Hours**  
    - Type: Schedule Trigger  
    - Role: Initiates the workflow every 3 hours.  
    - Configuration: Interval set to 3 hours.  
    - Inputs: None (trigger node)  
    - Outputs: Connects to "Ready to Post Image/Video".  
    - Edge Cases: Workflow will idle if no items are marked "Ready to Post". Trigger reliability depends on n8n execution environment uptime.

  - **Ready to Post Image/Video**  
    - Type: Google Sheets  
    - Role: Queries Google Sheet for the first row where "Status" equals "Ready to Post".  
    - Configuration:  
      - Document ID linked to a specific Google Sheet.  
      - Sheet Name set to "Sheet1" (gid=0).  
      - Filter applied on column "Status" with value "Ready to Post".  
      - Returns only the first matching entry to prevent multiple simultaneous posts.  
    - Inputs: From Schedule Trigger.  
    - Outputs: Passes the row data downstream.  
    - Credentials: Google OAuth2 for Sheets access.  
    - Edge Cases:  
      - No matching rows returns empty output, stopping the workflow.  
      - Google Sheets API limits or auth failures may cause errors.

#### 1.2 Media Preparation

- **Overview:**  
  Extracts the Google Drive file ID from the media URL in the sheet, then uploads the media to Blotato.

- **Nodes Involved:**  
  - Get Google Drive ID  
  - Upload Image/Video to BLOTATO

- **Node Details:**

  - **Get Google Drive ID**  
    - Type: Set node  
    - Role: Extracts the unique Google Drive file ID from the media URL string using a regex expression.  
    - Configuration:  
      - Uses expression: `{{$('Ready to Post Image/Video').item.json['Media URL (google drive)'].match(/https:\/\/drive\.google\.com\/file\/d\/([A-Za-z0-9_-]+)/i)[1]}}`  
      - Assigns extracted ID to `final_google_drive_url` variable.  
    - Inputs: From "Ready to Post Image/Video".  
    - Outputs: Passes data with extracted Drive ID.  
    - Edge Cases:  
      - Invalid or malformed URL will cause expression to fail or return undefined.  
      - Missing media URL or changes in Google Drive URL structure may break regex.

  - **Upload Image/Video to BLOTATO**  
    - Type: Blotato API node (custom community node)  
    - Role: Uploads media to Blotato by constructing a direct download URL from the Google Drive file ID.  
    - Configuration:  
      - Media URL set as `https://drive.google.com/uc?export=download&id={{ $json.final_google_drive_url }}`  
      - Resource set to "media".  
    - Inputs: From "Get Google Drive ID".  
    - Outputs: Passes uploaded media info (including Blotato URL) downstream.  
    - Credentials: Blotato API Key (paid feature) required.  
    - Edge Cases:  
      - Google Drive file must be publicly accessible; private files cause upload failure.  
      - File size limit <60MB due to Google Drive constraints.  
      - Network or API rate limits might cause retries or failures.

#### 1.3 Multi-Platform Posting

- **Overview:**  
  Posts the media and caption to nine social media platforms via Blotato API nodes in parallel.

- **Nodes Involved:**  
  - Tiktok [BLOTATO]  
  - Linkedin [BLOTATO]  
  - Facebook [BLOTATO]  
  - Instagram [BLOTATO]  
  - Twitter [BLOTATO]  
  - Youtube [BLOTATO]  
  - Threads [BLOTATO]  
  - Bluesky [BLOTATO]  
  - Pinterest [BLOTATO]

- **Node Details (common patterns):**

  Each node is a Blotato API node configured for a specific social platform:

  - Type: `@blotato/n8n-nodes-blotato.blotato`  
  - Role: Posts content (text and media) to the respective social platform via Blotato.  
  - Configuration commonalities:  
    - `postContentText`: Uses caption from Google Sheet: `={{ $('Ready to Post Image/Video').item.json.Caption }}`  
    - `postContentMediaUrls`: Uses media URL returned from Blotato upload node: `={{ $('Upload Image/Video to BLOTATO').item.json.url }}`  
    - `accountId`: Specific to each social media account (configured via list mode, referencing Blotato accounts).  
    - Platform-specific parameters: e.g., Facebook includes `facebookPageId`, Pinterest includes `pinterestBoardId`.  
  - Inputs: From "Upload Image/Video to BLOTATO".  
  - Outputs: On success or failure, each node outputs to "Final Report" node for merging.  
  - Credentials: Blotato API key required for all.  
  - Error Handling: Most nodes have `onError` set to `continueErrorOutput` to avoid stopping the workflow on platform-specific failures. Some have retries or waitBetweenTries configured.  
  - Edge Cases:  
    - API quota limits or auth errors from Blotato.  
    - Platform-specific content restrictions (e.g., video length, format).  
    - Account-specific warm-up requirements (noted in sticky notes).  
    - Network or service downtime.

- **Platform-Specific Notes:**  
  Detailed in sticky notes, including warm-up guides, content limitations, and helpful documentation links.

#### 1.4 Status Update & Reporting

- **Overview:**  
  Updates the Google Sheet row to mark the content as "Posted" and merges all post results for consolidated reporting.

- **Nodes Involved:**  
  - Update Status to "Posted"  
  - Final Report

- **Node Details:**

  - **Update Status to "Posted"**  
    - Type: Google Sheets  
    - Role: Appends or updates the row in Google Sheet, setting "Status" column to "Posted" for the posted caption.  
    - Configuration:  
      - Matching on "Caption" column to identify the row.  
      - Updates "Status" = "Posted".  
      - Uses same Sheet and Document ID as input fetch node.  
    - Inputs: From "Upload Image/Video to BLOTATO" (after posting attempts).  
    - Credentials: Google OAuth2 for Sheets access.  
    - Edge Cases:  
      - Matching may fail if captions are not unique or changed.  
      - Google Sheets API limits or auth errors.

  - **Final Report**  
    - Type: Merge  
    - Role: Collects outputs from all social media posting nodes to aggregate results.  
    - Configuration:  
      - Configured for 9 inputs (one from each platform node).  
    - Inputs: From all Blotato platform nodes.  
    - Outputs: Can be used to trigger further reporting or logging (not shown in this workflow).  
    - Edge Cases: Missing input if a platform node fails or is disabled.

#### 1.5 Documentation & Setup Notes

- **Overview:**  
  Multiple sticky notes throughout the workflow provide:

  - Setup instructions for Google Sheets, Google Drive, and Blotato API credentials.  
  - Platform-specific posting best practices and links.  
  - Important warnings (e.g., avoiding spam by reusing media, AI content disclosure).  
  - Troubleshooting and support links.

- **Nodes Involved:**  
  - Sticky Note (main instructions)  
  - Sticky Note1 (Google Sheet setup)  
  - Sticky Note2 ("Don't Touch" warning)  
  - Sticky Note3 (Post Everywhere notes)  
  - Sticky Note4 (Blotato credential setup)  
  - Sticky Note5 (Platform-specific notes)  
  - Sticky Note6 (Final setup checklist)  
  - Sticky Note7 (Final report and API dashboard links)

- **Edge Cases:**  
  Sticky notes serve as guidance and do not affect workflow execution but are critical for user understanding and proper configuration.

---

### 3. Summary Table

| Node Name                | Node Type                         | Functional Role                               | Input Node(s)               | Output Node(s)                                               | Sticky Note                                                                                                   |
|--------------------------|----------------------------------|-----------------------------------------------|-----------------------------|--------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Check Every 3 Hours       | Schedule Trigger                 | Triggers workflow every 3 hours                | None                        | Ready to Post Image/Video                                     |                                                                                                              |
| Ready to Post Image/Video | Google Sheets                   | Fetches one "Ready to Post" content item       | Check Every 3 Hours          | Get Google Drive ID                                          | # Setup 1 Configure your Google Sheet                                                                         |
| Get Google Drive ID       | Set                            | Extracts Google Drive file ID from URL         | Ready to Post Image/Video    | Upload Image/Video to BLOTATO                                | # Don't Touch                                                                                                |
| Upload Image/Video to BLOTATO | Blotato API                    | Uploads media to Blotato                        | Get Google Drive ID          | Tiktok [BLOTATO], Linkedin [BLOTATO], Facebook [BLOTATO], Instagram [BLOTATO], Twitter [BLOTATO], Youtube [BLOTATO], Threads [BLOTATO], Bluesky [BLOTATO], Pinterest [BLOTATO], Update Status to "Posted" | # Setup 2 Select your Blotato credential; # Setup 3 Select social accounts and deactivate unused platforms      |
| Tiktok [BLOTATO]          | Blotato API                    | Posts to TikTok platform                        | Upload Image/Video to BLOTATO| Final Report                                                | # Platform Specific Notes - TikTok warmup guide and FAQ link                                                  |
| Linkedin [BLOTATO]        | Blotato API                    | Posts to LinkedIn platform                      | Upload Image/Video to BLOTATO| Final Report                                                | # Platform Specific Notes - LinkedIn best practices and FAQ link                                              |
| Facebook [BLOTATO]        | Blotato API                    | Posts to Facebook platform                      | Upload Image/Video to BLOTATO| Final Report                                                | # Platform Specific Notes - Facebook posting guidelines and FAQs                                             |
| Instagram [BLOTATO]       | Blotato API                    | Posts to Instagram platform                     | Upload Image/Video to BLOTATO| Final Report                                                | # Platform Specific Notes - Instagram posting guidelines and FAQs                                            |
| Twitter [BLOTATO]         | Blotato API                    | Posts to Twitter platform                       | Upload Image/Video to BLOTATO| Final Report                                                | # Platform Specific Notes - Twitter posting guidelines and FAQs                                              |
| Youtube [BLOTATO]         | Blotato API                    | Posts to YouTube platform                       | Upload Image/Video to BLOTATO| Final Report                                                | # Platform Specific Notes - YouTube shorts and posting limits                                                |
| Threads [BLOTATO]         | Blotato API                    | Posts to Threads platform                       | Upload Image/Video to BLOTATO| Final Report                                                | # Platform Specific Notes - Threads posting guidelines and FAQs                                              |
| Bluesky [BLOTATO]         | Blotato API                    | Posts to Bluesky platform                       | Upload Image/Video to BLOTATO| Final Report                                                | # Platform Specific Notes - Bluesky posting guidelines and FAQs                                              |
| Pinterest [BLOTATO]       | Blotato API                    | Posts to Pinterest platform                     | Upload Image/Video to BLOTATO| Final Report                                                | # Platform Specific Notes - Pinterest warmup and board ID instructions                                       |
| Update Status to "Posted" | Google Sheets                  | Updates Google Sheet row status to "Posted"    | Upload Image/Video to BLOTATO| None                                                       |                                                                                                              |
| Final Report              | Merge                         | Aggregates results from all platform posts     | All 9 Blotato platform nodes | None                                                        | # Final Report and API Dashboard link                                                                         |
| Sticky Note               | Sticky Note                   | Setup and usage instructions                    | None                        | None                                                       | # Ready to Post - How template works and setup instructions                                                  |
| Sticky Note1              | Sticky Note                   | Google Sheet setup reminder                      | None                        | None                                                       | # Setup 1 Configure your Google Sheet                                                                         |
| Sticky Note2              | Sticky Note                   | Warning not to modify critical nodes            | None                        | None                                                       | # Don't Touch                                                                                                |
| Sticky Note3              | Sticky Note                   | Important posting warnings and Blotato docs     | None                        | None                                                       | # Post Everywhere - Anti-spam, AI disclosure, docs and support links                                         |
| Sticky Note4              | Sticky Note                   | Blotato credential setup notes                   | None                        | None                                                       | # Setup 2 Select your Blotato credential and troubleshooting checklist                                       |
| Sticky Note5              | Sticky Note                   | Platform-specific posting notes and best practices | None                        | None                                                       | # Platform Specific Notes with links to Blotato help pages                                                  |
| Sticky Note6              | Sticky Note                   | Final setup checklist                             | None                        | None                                                       | # Setup 3 Final checklist - Select accounts, deactivate unused platforms                                    |
| Sticky Note7              | Sticky Note                   | API Dashboard and run log reference              | None                        | None                                                       | # Final Report and API Dashboard link                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Name: `Check Every 3 Hours`  
   - Type: Schedule Trigger  
   - Configure interval to every 3 hours.  

2. **Create Google Sheets Node to Fetch Content**  
   - Name: `Ready to Post Image/Video`  
   - Type: Google Sheets  
   - Credential: Connect your Google Sheets OAuth2 credential with access to the content sheet.  
   - Operation: Read rows with filter  
   - Document ID: Use your target Google Sheet ID (must have columns: Title, Media URL (google drive), Caption, Status).  
   - Sheet Name: Use the first sheet or gid=0.  
   - Filter: Status column equals "Ready to Post".  
   - Options: Return only the first match to avoid multiple simultaneous posts.  
   - Connect output of `Check Every 3 Hours` to this node.  

3. **Create Set Node to Extract Google Drive File ID**  
   - Name: `Get Google Drive ID`  
   - Type: Set  
   - Add a new field named `final_google_drive_url` (string).  
   - Value: Use expression to extract Drive file ID from `Media URL (google drive)` column:  
     ```  
     {{$('Ready to Post Image/Video').item.json['Media URL (google drive)'].match(/https:\/\/drive\.google\.com\/file\/d\/([A-Za-z0-9_-]+)/i)[1]}}  
     ```  
   - Connect output of `Ready to Post Image/Video` to this node.  

4. **Create Blotato Node to Upload Media**  
   - Name: `Upload Image/Video to BLOTATO`  
   - Type: Blotato (community node)  
   - Credential: Use your Blotato API key credential.  
   - Resource: Set to "media".  
   - Media URL: Construct direct download URL using extracted Drive ID:  
     ```  
     https://drive.google.com/uc?export=download&id={{ $json.final_google_drive_url }}  
     ```  
   - Connect output of `Get Google Drive ID` to this node.  

5. **Create Blotato Nodes for Each Social Platform**  
   For each platform below, create a Blotato node configured as follows:  
   - Credential: Same Blotato API key credential.  
   - Platform: Set accordingly (e.g., "tiktok", "linkedin", "facebook", etc.) or leave blank if using default accountId.  
   - Account ID: Select the corresponding Blotato account ID for that platform.  
   - Post Content Text: Use caption from Google Sheet:  
     ```  
     {{$('Ready to Post Image/Video').item.json.Caption}}  
     ```  
   - Post Content Media URLs: Use uploaded media URL:  
     ```  
     {{$('Upload Image/Video to BLOTATO').item.json.url}}  
     ```  
   - For Facebook, add Facebook Page ID parameter.  
   - For Pinterest, add Pinterest Board ID parameter.  
   - Enable error handling to continue on errors. Optionally configure retry attempts and wait times.  
   - Platforms to create nodes for:  
     - Tiktok  
     - Linkedin  
     - Facebook  
     - Instagram  
     - Twitter  
     - Youtube (include YouTube video title from Google Sheet Title column)  
     - Threads  
     - Bluesky  
     - Pinterest  

   - Connect output of `Upload Image/Video to BLOTATO` to each platform node.  

6. **Create Google Sheets Node to Update Status**  
   - Name: `Update Status to "Posted"`  
   - Type: Google Sheets  
   - Credential: Your Google Sheets OAuth2 credential (may differ from read credential).  
   - Operation: Append or Update row.  
   - Document ID and Sheet Name: Same as input fetch node.  
   - Matching Columns: Use "Caption" to find the row to update.  
   - Update the "Status" column to "Posted".  
   - Map "Caption" column from original row to ensure correct matching.  
   - Connect output of `Upload Image/Video to BLOTATO` to this node.  

7. **Create Merge Node for Final Report**  
   - Name: `Final Report`  
   - Type: Merge  
   - Number of inputs: 9 (one for each social platform node).  
   - Connect all platform post output nodes to this merge node.  

8. **Connect Platform Nodes to Final Report Node**  
   - Connect outputs of all Blotato platform posting nodes to the `Final Report` node inputs.

9. **Add Sticky Notes for Documentation**  
   - Add sticky notes containing:  
     - Overview of process and how to use.  
     - Setup instructions for Google Sheets, Google Drive, and Blotato API key creation.  
     - Platform-specific posting notes and best practices with links to Blotato help pages.  
     - Troubleshooting tips and warnings about spamming and AI content disclosure.  
     - Final setup checklist to select accounts and deactivate unused platforms.

10. **Configure Credentials**  
    - Google Sheets OAuth2 with read/write access to your posting sheet.  
    - Blotato API key credential (paid feature).  
    - Ensure Google Drive folder with media is public (anyone with the link).  

11. **Activate Workflow**  
    - Test with a sample row marked "Ready to Post" in your Google Sheet.  
    - Monitor logs and Blotato API dashboard for posting status and errors.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                              |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Ready to Post: Supports text, images, videos, stories, slideshows, and carousels. Workflow checks Google Sheet every 3 hours for new items marked "Ready to Post". Only one item is processed each run to avoid spamming. Media is uploaded to Blotato before posting to multiple platforms.                                                                                                                                                                                                       | Main sticky note in workflow                                                                                 |
| Setup instructions include creating Blotato API key (paid feature), enabling community nodes in n8n, connecting Google Drive and Google Sheets credentials, and making Google Drive media folder public.                                                                                                                                                                                                                                                                                           | Setup sticky notes                                                                                           |
| Platform-specific posting notes with best practices, warm-up guides, media requirements, and troubleshooting FAQs are linked for TikTok, LinkedIn, Facebook, Instagram, Twitter, Threads, Bluesky, YouTube, and Pinterest.                                                                                                                                                                                                                                                                           | Sticky Note5 containing multiple platform help links                                                        |
| Important warnings: Avoid posting the same media repeatedly to prevent spam flagging. Disclose AI-generated content if applicable.                                                                                                                                                                                                                                                                                                                                                                 | Sticky Note3                                                                                                |
| Blotato API dashboard link for monitoring API usage and troubleshooting.                                                                                                                                                                                                                                                                                                                                                                                                                            | https://my.blotato.com/api-dashboard                                                                         |
| Google Drive media files must be public and below 60MB due to Google Drive limitations; use Amazon S3 for larger files.                                                                                                                                                                                                                                                                                                                                                                            | Sticky Note4 troubleshooting checklist                                                                       |
| Blotato community node is a verified community node in n8n; ensure it is installed and configured properly in your n8n instance.                                                                                                                                                                                                                                                                                                                                                                   | Setup notes                                                                                                 |
| Platform warm-up guides (especially TikTok and Pinterest) are necessary for optimal engagement and to prevent account restrictions.                                                                                                                                                                                                                                                                                                                                                                | TikTok warm-up guide: https://help.blotato.com/platforms/tiktok/brand-new-accounts                            |
| The workflow is designed to be modular; disable or remove platform nodes you do not need without impacting the rest of the flow.                                                                                                                                                                                                                                                                                                                                                                   | Setup sticky notes                                                                                           |

---

**Disclaimer:** The content above is generated based solely on the provided n8n workflow JSON. It adheres strictly to current content policies and contains no illegal, offensive, or protected content. All data processed is legal and public.