Form-Based X/Twitter Poster

https://n8nworkflows.xyz/workflows/form-based-x-twitter-poster-4415


# Form-Based X/Twitter Poster

### 1. Workflow Overview

This workflow automates the process of posting content to X (formerly Twitter) via a web form submission. It allows users to enter text and optionally upload media (images or videos), processes the submitted data, uploads media if present, posts the tweet, and finally shows a customized confirmation message.

The workflow is structured into the following logical blocks:

- **1.1 Input Reception:** Captures user input from a web form with text and optional media.
- **1.2 Media Extraction and Conditional Handling:** Extracts media metadata and determines whether to upload media or post text-only.
- **1.3 Media Upload to X:** Uploads media to X if present, preparing it for tweet attachment.
- **1.4 Posting to X:** Posts the tweet with or without media.
- **1.5 Confirmation Display:** Shows a thank-you message after successful submission.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block receives user input via a web form titled "Post to X" with fields for post content and media upload.

**Nodes Involved:**  
- On form submission

**Node Details:**  

- **On form submission**  
  - Type: `Form Trigger`  
  - Role: Initiates the workflow when the form is submitted by a user.  
  - Configuration:  
    - Form Title: "Post to X"  
    - Fields:  
      - "Post Content" (textarea, required)  
      - "Media" (file upload, single file, accepts multiple image and video formats: .jpg, .png, .mp4, .mov, .jpeg, .gif, .heif, .webp, .tiff)  
    - Button Label: "Submit"  
    - Append Attribution: Disabled  
  - Inputs: HTTP webhook triggered by form submission  
  - Outputs: JSON with form fields and binary data if media uploaded  
  - Version: 2.2  
  - Edge cases: Missing required field (post content) will prevent submission; unsupported media formats are not accepted by the form field; large file uploads may cause timeouts or errors.

---

#### 1.2 Media Extraction and Conditional Handling

**Overview:**  
Extracts media metadata (MIME type and media type) from the form submission and decides the next step depending on media presence.

**Nodes Involved:**  
- Extract Media Details  
- If Image Exists

**Node Details:**  

- **Extract Media Details**  
  - Type: `Code` (JavaScript)  
  - Role: Parses form data to extract post content and identify media type (image, video, audio) based on MIME type. Passes binary media data forward if present.  
  - Configuration:  
    - Script extracts "Post Content" and first media file details.  
    - Evaluates MIME type to classify media as IMAGE, VIDEO, AUDIO, or null if none.  
    - Outputs JSON with `content`, `mime_type`, `media_type` and binary data under `media` key if available.  
  - Inputs: Output of "On form submission" (form JSON and binary data)  
  - Outputs: Processed JSON and binary data (if any)  
  - Version: 2  
  - Edge cases: Missing or corrupted media data; unsupported MIME types; absence of media leads to null media_type. Expression errors if input data malformed.

- **If Image Exists**  
  - Type: `If`  
  - Role: Conditional branching based on presence of media MIME type.  
  - Configuration:  
    - Condition: checks if `mime_type` exists and is non-empty in JSON.  
  - Inputs: Output of "Extract Media Details"  
  - Outputs:  
    - True branch: media exists → proceeds to upload media  
    - False branch: no media → proceeds to tweet text only  
  - Version: 2.2  
  - Edge cases: MIME type detection failure could misroute the flow; empty strings or null values affect branching logic.

---

#### 1.3 Media Upload to X

**Overview:**  
Uploads media file to X’s media upload endpoint to obtain a media ID for attaching to the tweet.

**Nodes Involved:**  
- Upload Media (X)

**Node Details:**  

- **Upload Media (X)**  
  - Type: `HTTP Request`  
  - Role: Sends a multipart/form-data POST request to X’s media upload API.  
  - Configuration:  
    - URL: `https://upload.twitter.com/1.1/media/upload.json?media_category=TWEET_IMAGE`  
    - Method: POST  
    - Authentication: Predefined credential of type `twitterOAuth1Api` (OAuth 1.0 for media upload)  
    - Body: Multipart form-data with binary media attached under "media" parameter  
    - Response format: JSON expected  
  - Inputs: Binary media data and JSON from "If Image Exists" true branch  
  - Outputs: JSON with media upload response including `media_id_string`  
  - Credentials: OAuth1 credentials named "X OAuth - Akhil"  
  - Version: 4.2  
  - Edge cases: Authentication failures; media size limits; unsupported media category; network timeouts; malformed binary data; API quota limits.

---

#### 1.4 Posting to X

**Overview:**  
Posts the tweet with text and optional media attachment using OAuth2 API.

**Nodes Involved:**  
- X (tweet with media)  
- X1 (tweet text only)

**Node Details:**  

- **X**  
  - Type: `Twitter` (n8n node)  
  - Role: Posts a tweet with text content and media attachment (media ID).  
  - Configuration:  
    - Text: taken from extracted content  
    - Attachments: uses media ID from upload response (`media_id_string`)  
    - Authentication: OAuth2 credentials  
  - Inputs: Output of "Upload Media (X)"  
  - Outputs: Twitter API response for posted tweet  
  - Credentials: OAuth2 credentials named "X OAuth2 - Akhil"  
  - Version: 2  
  - Edge cases: Invalid media ID; tweet content length limits; API rate limiting; auth token expiration.

- **X1**  
  - Type: `Twitter` (n8n node)  
  - Role: Posts a tweet with text only (no media).  
  - Configuration:  
    - Text: extracted content from JSON  
    - No attachments  
    - Authentication: OAuth2 credentials  
  - Inputs: False branch of "If Image Exists" (no media)  
  - Outputs: Twitter API response  
  - Credentials: OAuth2 credentials named "X OAuth2 - Akhil"  
  - Version: 2  
  - Edge cases: Tweet length limits; auth failures; API quota.

---

#### 1.5 Confirmation Display

**Overview:**  
Displays a custom thank-you message to the user after successful posting.

**Nodes Involved:**  
- End Form   
- Sticky Note (comment)

**Node Details:**  

- **End Form**  
  - Type: `Form` (completion operation)  
  - Role: Sends completion message and title back to user interface after workflow finishes.  
  - Configuration:  
    - Completion Title: "Thank you so much for sharing your experience on X! 🖤"  
    - Completion Message: "We truly appreciate your support and are so glad we could make a positive impact. Your words mean the world to us!"  
  - Inputs: From both "X" and "X1" nodes (tweet posted with or without media)  
  - Outputs: HTTP response to form submitter  
  - Version: 1  
  - Edge cases: Failure to reach this node would leave user without feedback.

- **Sticky Note**  
  - Type: `Sticky Note`  
  - Role: Provides a visible comment in the workflow canvas explaining the confirmation message block.  
  - Content: "## Customize Confirmation Message \nDisplays thank-you message post-submission."  
  - Position: Visually associated with "End Form" node  
  - No inputs or outputs.

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role                      | Input Node(s)         | Output Node(s)        | Sticky Note                                             |
|---------------------|---------------------|------------------------------------|-----------------------|-----------------------|---------------------------------------------------------|
| On form submission  | Form Trigger        | Receive form submission input       | —                     | Extract Media Details  |                                                         |
| Extract Media Details| Code                | Extract post content & media info   | On form submission    | If Image Exists        |                                                         |
| If Image Exists      | If                  | Branch depending on media presence  | Extract Media Details | Upload Media (X), X1   |                                                         |
| Upload Media (X)     | HTTP Request        | Upload media to X API                | If Image Exists (true)| X                     |                                                         |
| X                    | Twitter             | Post tweet with media               | Upload Media (X)      | End Form               |                                                         |
| X1                   | Twitter             | Post tweet text only                | If Image Exists (false)| End Form               |                                                         |
| End Form             | Form                 | Show confirmation message           | X, X1                 | —                      | ## Customize Confirmation Message \nDisplays thank-you message post-submission. |
| Sticky Note          | Sticky Note         | Comment on confirmation message     | —                     | —                      | ## Customize Confirmation Message \nDisplays thank-you message post-submission. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "On form submission" node:**  
   - Type: Form Trigger  
   - Configure form title: "Post to X"  
   - Add fields:  
     - Textarea field labeled "Post Content" (required)  
     - File upload field labeled "Media" (accept multiple image/video types: .jpg, .png, .mp4, etc; only single file allowed)  
   - Set button label to "Submit"  
   - Disable append attribution  
   - Save credentials if needed (none here as this is trigger)

2. **Add "Extract Media Details" node:**  
   - Type: Code (JavaScript)  
   - Paste script to extract post content and media metadata as per the provided logic (classify media type by MIME, pass binary data forward)  
   - Connect output of "On form submission" to this node

3. **Add "If Image Exists" node for branching:**  
   - Type: If  
   - Condition: Check if JSON property `mime_type` exists and is non-empty  
   - Connect output of "Extract Media Details" to this node

4. **Add "Upload Media (X)" node:**  
   - Type: HTTP Request  
   - URL: `https://upload.twitter.com/1.1/media/upload.json?media_category=TWEET_IMAGE`  
   - Method: POST  
   - Authentication: Use OAuth1 credentials configured for X (Twitter) media upload  
   - Send body: multipart-form-data  
   - Body parameter: Add form binary data parameter with input field name "media" connected to binary data named "media" from previous node  
   - Connect "true" branch of "If Image Exists" to this node

5. **Add "X" node for posting tweet with media:**  
   - Type: Twitter node  
   - Text: Use expression to get content `={{ $('Extract Media Details').first().json.content }}`  
   - Additional Fields: Attachments field with `={{ $json.media_id_string }}` from previous node output  
   - Authentication: OAuth2 credentials for X (Twitter) posting  
   - Connect output of "Upload Media (X)" to this node

6. **Add "X1" node for posting tweet text only:**  
   - Type: Twitter node  
   - Text: Use expression `={{ $json.content }}` from "Extract Media Details"  
   - Additional fields: none (no media)  
   - Authentication: OAuth2 credentials for X (Twitter) posting  
   - Connect "false" branch of "If Image Exists" to this node

7. **Add "End Form" node to show confirmation:**  
   - Type: Form node (completion operation)  
   - Configure completion title: "Thank you so much for sharing your experience on X! 🖤"  
   - Configure completion message: "We truly appreciate your support and are so glad we could make a positive impact. Your words mean the world to us!"  
   - Connect outputs of both "X" and "X1" nodes to this node

8. **Add "Sticky Note" near "End Form" node:**  
   - Type: Sticky Note  
   - Content:  
     ```
     ## Customize Confirmation Message 
     Displays thank-you message post-submission.
     ```
   - No connections needed

9. **Credential Setup:**  
   - Set up OAuth1 credential for Twitter media upload API (named e.g., "X OAuth - Akhil")  
   - Set up OAuth2 credential for Twitter posting API (named e.g., "X OAuth2 - Akhil")  
   - Ensure credentials have required scopes and valid tokens

10. **Test the workflow:**  
    - Submit form with text only: should post text tweet and show confirmation  
    - Submit form with text and supported media: should upload media, post tweet with attachment, show confirmation  
    - Handle errors by checking logs and edge cases (timeouts, auth errors)

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                               |
|-------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| The workflow is designed for posting to X (Twitter) using both OAuth1 (media upload) and OAuth2 (tweet posting) APIs | Important for credential setup and API compatibility           |
| Supported media formats in the form: .jpg, .png, .mp4, .mov, .jpeg, .gif, .heif, .webp, .tiff   | Ensures correct file type handling at upload                   |
| Media upload uses Twitter’s legacy API endpoint for media uploads with multipart/form-data      | https://developer.twitter.com/en/docs/twitter-api/v1/media/upload-api/overview |
| OAuth1 is required for media upload, while OAuth2 is used for posting tweets                     | Twitter API v2 limitations/requirements                        |
| Confirmation message can be customized in the "End Form" node parameters                        | Supports branding and user experience customization            |

---

**Disclaimer:**  
The text provided is exclusively derived from an n8n automation workflow. It strictly respects content policies and contains no illegal or offensive elements. All processed data is legal and public.