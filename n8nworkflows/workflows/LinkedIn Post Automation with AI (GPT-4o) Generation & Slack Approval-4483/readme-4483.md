LinkedIn Post Automation with AI (GPT-4o) Generation & Slack Approval

https://n8nworkflows.xyz/workflows/linkedin-post-automation-with-ai--gpt-4o--generation---slack-approval-4483


# LinkedIn Post Automation with AI (GPT-4o) Generation & Slack Approval

---
### 1. Workflow Overview

This workflow automates the creation, approval, and publishing of LinkedIn posts using AI-generated content and Slack-based human approval. It is designed for social media managers or marketing teams who want to streamline LinkedIn content creation with GPT-4o while maintaining quality control through Slack approval. The workflow integrates Google Sheets as a content source, OpenAI for content generation, Slack for approval, LinkedIn API for posting, and handles image uploads and group postings.

The logical blocks are:

- **1.1 Input Reception & Trigger**: Detects new LinkedIn post requests from Google Sheets.
- **1.2 AI Content Generation**: Uses GPT-4o (OpenAI) to create post content based on sheet data.
- **1.3 Slack Approval Process**: Formats and sends the AI-generated content to Slack for human review and edits.
- **1.4 Approval Response Handling**: Receives and processes Slack approval responses via webhook.
- **1.5 Image Upload to LinkedIn**: Manages image upload to LinkedIn if the post contains images.
- **1.6 LinkedIn Publishing**: Publishes the approved post content to LinkedIn profile and groups.
- **1.7 Status Update in Sheets**: Updates the Google Sheet to mark posts as published, preventing duplicates.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Trigger

- **Overview:** Watches a Google Sheet for new items marked as "Pending" to initiate post creation.
- **Nodes Involved:**  
  - Linkedin-Post-Topic (Google Sheets Trigger)  
  - Validate-Status (If node)  
  - Limit (Limit processing to one item at a time)  
  - Edit Fields (Set fields for downstream use)

- **Node Details:**  
  - **Linkedin-Post-Topic**  
    - Type: Google Sheets Trigger  
    - Configured to poll every minute for new rows in a designated sheet.  
    - Triggers workflow on new rows or changes.  
    - Edge cases: API quota limits, empty rows, or malformed data.  
  - **Validate-Status**  
    - Type: If  
    - Filters only rows with status = "Pending" to avoid reprocessing already handled posts.  
    - Edge case: Incorrect or missing status values causing misfiltering.  
  - **Limit**  
    - Type: Limit  
    - Ensures only one post is processed at a time to avoid API overload and rate limits.  
  - **Edit Fields**  
    - Type: Set  
    - Extracts and prepares necessary IDs and fields for the AI generation step.  
    - May fail if expected fields are missing.

#### 1.2 AI Content Generation

- **Overview:** Generates LinkedIn post content using GPT-4o based on input from Google Sheets.
- **Nodes Involved:**  
  - Linedin-Post-Creator (Langchain Agent with GPT-4o)  
  - OpenAI Chat Model  
  - Structured Output Parser  
  - Format-Content (Code node)

- **Node Details:**  
  - **Linedin-Post-Creator**  
    - Type: Langchain Agent  
    - Uses GPT-4o with a custom prompt tailored for LinkedIn post tone and style.  
    - Receives input from Edit Fields.  
    - Edge cases: API authentication failure, prompt errors, or output format issues.  
  - **OpenAI Chat Model**  
    - Type: Language Model (OpenAI)  
    - Configured with API credentials for GPT-4o.  
  - **Structured Output Parser**  
    - Type: Output parser for Langchain  
    - Parses AI output into a structured format for downstream use.  
    - Edge cases: Parsing failures due to unexpected AI output.  
  - **Format-Content**  
    - Type: Code  
    - Cleans and formats AI-generated content for Slack display, handles special characters and newlines.  
    - Edge cases: Code execution errors if unexpected input.

#### 1.3 Slack Approval Process

- **Overview:** Sends the AI-generated LinkedIn post content to a Slack channel for human review and possible editing.
- **Nodes Involved:**  
  - Approval-on-Slack (Slack node)

- **Node Details:**  
  - **Approval-on-Slack**  
    - Type: Slack node  
    - Posts a message with the content and interactive buttons ("Approve As Is", "Submit Edited Version").  
    - Slack webhook URL and credentials must be configured.  
    - The Slack message includes a text input block for editing content inline.  
    - Edge cases: Slack API rate limits, webhook misconfiguration, or message formatting errors.

#### 1.4 Approval Response Handling

- **Overview:** Receives Slack approval or edited content via webhook and processes it for next steps.
- **Nodes Involved:**  
  - Webhook (Slack response receiver)  
  - Set Topics and Images for Processing (Code node)  
  - Format-Response (Code node)  
  - Validate-Response (If node)  
  - Set-Final-Message (Set node)

- **Node Details:**  
  - **Webhook**  
    - Type: Webhook  
    - Configured to receive Slack action payloads on approval or edits.  
    - URL must be registered in Slack app settings.  
    - Edge cases: Payload format changes, security token validation failure.  
  - **Set Topics and Images for Processing**  
    - Type: Code  
    - Extracts the edited content and image URLs from Slack response.  
    - Edge cases: Missing or malformed payload values.  
  - **Format-Response**  
    - Type: Code  
    - Further processes Slack inputs to prepare for validation.  
  - **Validate-Response**  
    - Type: If  
    - Checks that response contains valid content before proceeding.  
    - Prevents empty or invalid approvals from advancing.  
  - **Set-Final-Message**  
    - Type: Set  
    - Prepares content and metadata for LinkedIn publishing.

#### 1.5 Image Upload to LinkedIn

- **Overview:** Handles image upload workflow required by LinkedIn API before post publishing.
- **Nodes Involved:**  
  - Linkedin-User-Detail (HTTP Request)  
  - Register Image (HTTP Request)  
  - Upload-Image-data (Set node)  
  - Get Image Binary (HTTP Request)  
  - Upload Image (HTTP Request)

- **Node Details:**  
  - **Linkedin-User-Detail**  
    - Fetches authenticated LinkedIn user profile ID needed for media uploads and attribution.  
    - Requires OAuth2 LinkedIn credentials.  
    - Edge cases: Token expiration or invalid permissions.  
  - **Register Image**  
    - Initiates media upload process by requesting upload URL from LinkedIn API.  
    - Requires valid image URL from sheet or Slack response.  
  - **Upload-Image-data**  
    - Extracts upload URL and asset ID from LinkedIn API response for subsequent upload steps.  
  - **Get Image Binary**  
    - Downloads image binary from the external URL.  
    - Edge cases: Broken URLs or download failures.  
  - **Upload Image**  
    - Uploads the binary data to LinkedIn media server using prior obtained URL.  
    - Edge cases: Upload failures due to network or API errors.

#### 1.6 LinkedIn Publishing

- **Overview:** Publishes the approved and optionally imaged post to LinkedIn profile and groups.
- **Nodes Involved:**  
  - Linkedin-post (HTTP Request)  
  - Get-Group-id (Google Sheets)  
  - Pick 1 by 1 (SplitInBatches)  
  - Post-Linkedin-Groups (HTTP Request)  
  - Update-Status (Google Sheets)

- **Node Details:**  
  - **Linkedin-post**  
    - Publishes the post content (with media if any) to the user's LinkedIn profile.  
    - Requires OAuth2 LinkedIn credentials and correct API scopes.  
    - Edge cases: API rate limits, invalid post content, or auth errors.  
  - **Get-Group-id**  
    - Retrieves LinkedIn group IDs from a dedicated Google Sheet to allow group posting.  
  - **Pick 1 by 1**  
    - Splits group IDs to post one by one, preventing LinkedIn API rate limit violations.  
  - **Post-Linkedin-Groups**  
    - Posts the content to each LinkedIn group identified.  
    - Configured to continue on errors to avoid halting the whole batch.  
  - **Update-Status**  
    - Updates the original row in Google Sheets marking the post as "Posted" to avoid duplicate processing.

#### 1.7 Status Update in Sheets

- **Overview:** Ensures that posts which have been published are marked accordingly to prevent re-triggering.
- **Nodes Involved:**  
  - Update-Status (Google Sheets)

- **Node Details:**  
  - Updates status column to "Posted".  
  - Edge cases: Failed sheet update causing workflow to reprocess.

---

### 3. Summary Table

| Node Name                   | Node Type                       | Functional Role                         | Input Node(s)             | Output Node(s)            | Sticky Note                                                                                   |
|-----------------------------|--------------------------------|---------------------------------------|---------------------------|---------------------------|----------------------------------------------------------------------------------------------|
| Workflow Overview           | Sticky Note                    | Documentation overview                 |                           |                           |                                                                                              |
| Sheets Trigger Info         | Sticky Note                    | Notes on Google Sheets trigger         |                           |                           |                                                                                              |
| AI Generation               | Sticky Note                    | Notes on AI content generation          |                           |                           |                                                                                              |
| Slack Approval              | Sticky Note                    | Notes on Slack approval process         |                           |                           |                                                                                              |
| Webhook Handler            | Sticky Note                    | Notes on Slack webhook response handling|                           |                           |                                                                                              |
| Image Upload Process       | Sticky Note                    | Notes on LinkedIn image upload          |                           |                           |                                                                                              |
| LinkedIn Publishing        | Sticky Note                    | Notes on LinkedIn post publishing       |                           |                           |                                                                                              |
| Status Update              | Sticky Note                    | Notes on updating status in Sheets      |                           |                           |                                                                                              |
| Linkedin-Post-Topic        | Google Sheets Trigger          | Trigger new LinkedIn posts              |                           | Validate-Status           | Triggers every minute for new posts in your spreadsheet                                      |
| Validate-Status            | If                            | Filters only 'Pending' posts            | Linkedin-Post-Topic       | Limit                     | Filters to process only 'Pending' items - prevents duplicates                                |
| Limit                     | Limit                         | Processes one item at a time            | Validate-Status           | Edit Fields               | Processes one item at a time to prevent API overload                                        |
| Edit Fields                | Set                           | Prepares IDs and fields for AI          | Limit                     | Linedin-Post-Creator      | Extracts post ID for tracking through workflow                                               |
| Linedin-Post-Creator       | Langchain Agent (GPT-4o)      | Generates LinkedIn post content         | Edit Fields               | Format-Content            | Uses GPT-4 to generate LinkedIn content - customize prompt for your brand voice              |
| OpenAI Chat Model          | Language Model (OpenAI)        | Underlying AI model for generation      |                           | Linedin-Post-Creator (ai_languageModel) |                                                                                              |
| Structured Output Parser   | Langchain Output Parser        | Parses AI response into structured form| OpenAI Chat Model         | Linedin-Post-Creator (ai_outputParser) |                                                                                              |
| Format-Content             | Code                          | Formats AI content for Slack            | Linedin-Post-Creator      | Approval-on-Slack         | Formats content for Slack display - handles special characters and newlines                  |
| Approval-on-Slack          | Slack                         | Sends content to Slack for approval     | Format-Content            |                           | Sends for human review - edit the Slack user to change approver                              |
| Webhook                   | Webhook                       | Receives Slack approval responses       |                           | Set Topics and Images for Processing | Receives approval responses - ensure URL is configured in Slack app                         |
| Set Topics and Images for Processing | Code                 | Extracts edited content and image URLs  | Webhook                   | Format-Response           | Extracts post ID and image URL from Slack approval response                                  |
| Format-Response            | Code                          | Processes Slack response data            | Set Topics and Images for Processing | Validate-Response         | Extracts edited content from Slack response                                                  |
| Validate-Response          | If                            | Validates Slack response content presence| Format-Response           | Set-Final-Message         | Ensures response contains content before proceeding                                         |
| Set-Final-Message          | Set                           | Prepares final message for publishing    | Validate-Response         | Linkedin-User-Detail      | Prepares approved content for publishing                                                    |
| Linkedin-User-Detail       | HTTP Request                  | Retrieves LinkedIn user ID               | Set-Final-Message         | Register Image            | Fetches LinkedIn user ID for post attribution                                               |
| Register Image             | HTTP Request                  | Initiates LinkedIn media upload          | Linkedin-User-Detail      | Upload-Image-data         | Initiates LinkedIn media upload - requires valid image URL from sheet                       |
| Upload-Image-data          | Set                           | Extracts upload URL and asset ID         | Register Image            | Get Image Binary          | Extracts upload URL and asset ID from LinkedIn response                                     |
| Get Image Binary           | HTTP Request                  | Downloads image from URL                  | Upload-Image-data         | Upload Image              | Downloads image from URL specified in spreadsheet                                           |
| Upload Image               | HTTP Request                  | Uploads image binary to LinkedIn         | Get Image Binary          | Linkedin-post             | Uploads image binary to LinkedIn servers                                                   |
| Linkedin-post              | HTTP Request                  | Publishes post to LinkedIn profile       | Upload Image, Edit Fields | Get-Group-id              | Publishes to your profile - update with your LinkedIn credentials                           |
| Get-Group-id               | Google Sheets                 | Retrieves LinkedIn group IDs              | Linkedin-post             | Pick 1 by 1               | Retrieves LinkedIn group IDs from Groups sheet                                             |
| Pick 1 by 1               | SplitInBatches                | Processes groups one at a time            | Get-Group-id              | Update-Status, Post-Linkedin-Groups | Processes groups one at a time to avoid rate limits                                      |
| Post-Linkedin-Groups       | HTTP Request                  | Posts content to LinkedIn groups          | Pick 1 by 1               | Pick 1 by 1 (loop)        | Posts to multiple groups - add group IDs in Groups sheet                                   |
| Update-Status              | Google Sheets                 | Marks posts as 'Posted'                    | Pick 1 by 1               |                           | Marks as 'Posted' to prevent reprocessing                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets Trigger Node**  
   - Name: `Linkedin-Post-Topic`  
   - Type: Google Sheets Trigger  
   - Configure to watch the sheet and worksheet where LinkedIn post requests are stored.  
   - Set polling interval to 1 minute.  

2. **Add If Node to Filter Pending Posts**  
   - Name: `Validate-Status`  
   - Type: If  
   - Condition: Status column equals "Pending"  
   - Connect `Linkedin-Post-Topic` output to `Validate-Status` input.

3. **Add Limit Node**  
   - Name: `Limit`  
   - Type: Limit  
   - Set to process 1 item at a time.  
   - Connect `Validate-Status` true output to `Limit`.

4. **Add Set Node to Prepare Fields**  
   - Name: `Edit Fields`  
   - Type: Set  
   - Extract post ID and necessary fields from input data for AI generation.  
   - Connect `Limit` output to this node.

5. **Set up Langchain Agent Node for AI Generation**  
   - Name: `Linedin-Post-Creator`  
   - Type: Langchain Agent  
   - Configure prompt to instruct GPT-4o to generate LinkedIn post content according to brand voice.  
   - Connect `Edit Fields` output to `Linedin-Post-Creator` input.

6. **Add OpenAI Chat Model Node**  
   - Name: `OpenAI Chat Model`  
   - Type: Langchain LM Chat OpenAI  
   - Configure credentials with OpenAI GPT-4o API key.  
   - Connect `Linedin-Post-Creator` to use this model for generation.

7. **Add Structured Output Parser Node**  
   - Name: `Structured Output Parser`  
   - Type: Langchain Structured Output Parser  
   - Connect to parse output from `OpenAI Chat Model` for structured data.

8. **Add Code Node to Format Content for Slack**  
   - Name: `Format-Content`  
   - Type: Code  
   - Write code to clean and format AI content, handle newlines and special chars.  
   - Connect from `Linedin-Post-Creator`.

9. **Add Slack Node for Approval**  
   - Name: `Approval-on-Slack`  
   - Type: Slack  
   - Configure Slack credentials (OAuth2 or webhook).  
   - Set message with interactive blocks: content display, editable text input, approve and edit buttons.  
   - Connect from `Format-Content`.

10. **Create Webhook Node to Receive Slack Responses**  
    - Name: `Webhook`  
    - Type: Webhook  
    - Set the webhook URL and configure Slack app to send action responses to this URL.  

11. **Add Code Node to Extract Slack Response Data**  
    - Name: `Set Topics and Images for Processing`  
    - Type: Code  
    - Extract edited content, image URLs, and other metadata from Slack payload.  
    - Connect `Webhook` output to this node.

12. **Add Code Node to Format Slack Response**  
    - Name: `Format-Response`  
    - Type: Code  
    - Prepare Slack response data for validation.  
    - Connect from `Set Topics and Images for Processing`.

13. **Add If Node to Validate Slack Response Content**  
    - Name: `Validate-Response`  
    - Type: If  
    - Check if content field is not empty or invalid.  
    - Connect from `Format-Response`.

14. **Add Set Node to Prepare Final Message for LinkedIn**  
    - Name: `Set-Final-Message`  
    - Type: Set  
    - Prepare approved content, image references, and user metadata for LinkedIn API.  
    - Connect `Validate-Response` true output to this node.

15. **Add HTTP Request Node to Fetch LinkedIn User Details**  
    - Name: `Linkedin-User-Detail`  
    - Type: HTTP Request  
    - Configure OAuth2 LinkedIn credentials.  
    - Request LinkedIn user profile data for post attribution.  
    - Connect from `Set-Final-Message`.

16. **Add HTTP Request Node to Register Image Upload**  
    - Name: `Register Image`  
    - Type: HTTP Request  
    - Use LinkedIn API to initiate media upload.  
    - Requires valid image URL from Slack response or sheet.  
    - Connect from `Linkedin-User-Detail`.

17. **Add Set Node to Extract Upload Data**  
    - Name: `Upload-Image-data`  
    - Type: Set  
    - Extract upload URL and asset ID from LinkedIn response.  
    - Connect `Register Image` output here.

18. **Add HTTP Request Node to Download Image Binary**  
    - Name: `Get Image Binary`  
    - Type: HTTP Request  
    - Download image using extracted URL.  
    - Connect from `Upload-Image-data`.

19. **Add HTTP Request Node to Upload Image to LinkedIn**  
    - Name: `Upload Image`  
    - Type: HTTP Request  
    - Upload binary data to LinkedIn media server.  
    - Connect from `Get Image Binary`.

20. **Add HTTP Request Node to Publish Post on LinkedIn Profile**  
    - Name: `Linkedin-post`  
    - Type: HTTP Request  
    - Publish post with content and media references.  
    - Connect from `Upload Image` (or `Set-Final-Message` if no image).  

21. **Add Google Sheets Node to Retrieve Group IDs**  
    - Name: `Get-Group-id`  
    - Type: Google Sheets  
    - Fetch group IDs from dedicated sheet for group posting.  
    - Connect from `Linkedin-post`.

22. **Add SplitInBatches Node to Process Groups Sequentially**  
    - Name: `Pick 1 by 1`  
    - Type: SplitInBatches  
    - Configure batch size = 1 to avoid API rate limits.  
    - Connect from `Get-Group-id`.

23. **Add HTTP Request Node to Post to LinkedIn Groups**  
    - Name: `Post-Linkedin-Groups`  
    - Type: HTTP Request  
    - Post content to each LinkedIn group ID.  
    - On error: continue.  
    - Connect from `Pick 1 by 1`.

24. **Add Google Sheets Node to Update Post Status**  
    - Name: `Update-Status`  
    - Type: Google Sheets  
    - Mark original post row as "Posted".  
    - Connect from both `Pick 1 by 1` (successful batch completion) and `Post-Linkedin-Groups`.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                      |
|-------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| Slack message includes interactive elements for content editing inline, improving approval UX. | Slack Block Kit documentation: https://api.slack.com/block-kit |
| GPT-4o prompt can be customized to match brand voice and style for better post relevance.      | OpenAI GPT Models: https://platform.openai.com/docs/models/gpt-4o |
| Rate limiting is handled by limiting processing to one item and splitting batch requests.      | LinkedIn API Rate Limits documentation              |
| Ensure LinkedIn OAuth2 credentials have correct permissions for posting and media uploads.      | LinkedIn API docs: https://learn.microsoft.com/en-us/linkedin/marketing/integrations/community-management/shares/ugc-post-api |
| Google Sheets API quota considerations for frequent polling.                                    | Google Sheets API Limits: https://developers.google.com/sheets/api/limits |
| Slack webhook URL must be kept secure and registered in Slack app for interactive message response.| Slack Interactivity: https://api.slack.com/interactivity |
| Media upload is a multistep process with registration, binary upload, and association.          | LinkedIn Media Upload guide                          |

---

**Disclaimer:**  
The provided content is exclusively derived from an automated n8n workflow. It strictly adheres to current content policies and contains no illegal, offensive, or protected material. All data manipulated is legal and publicly accessible.