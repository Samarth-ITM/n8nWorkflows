Publish image & video to multiple social media (X, Instagram, Facebook and more)

https://n8nworkflows.xyz/workflows/publish-image---video-to-multiple-social-media--x--instagram--facebook-and-more--3669


# Publish image & video to multiple social media (X, Instagram, Facebook and more)

### 1. Workflow Overview

This workflow automates the publishing of image or video posts to multiple social media platforms (Instagram, LinkedIn, Facebook, X, TikTok, Threads) using a unified form interface and the Upload-Post.com API service. It streamlines content submission, media type detection, and API interaction, providing clear success or failure feedback to users.

The workflow is logically divided into these blocks:

- **1.1 Input Reception**: Captures user input via a form trigger, including platform selection, account profile, caption, media file (image or video), and optional Facebook Page ID.

- **1.2 Media Type Routing**: Determines whether the uploaded file is an image or video using a Switch node, routing the flow accordingly.

- **1.3 API Upload**: Sends the media and metadata to Upload-Post.com API endpoints for photos or videos, using HTTP Request nodes with authentication.

- **1.4 API Response Handling**: Parses the API response to determine success or failure.

- **1.5 User Feedback**: Displays completion forms to the user indicating success or failure of the post upload.

- **1.6 Configuration Notes**: Contains sticky notes with setup instructions and usage tips.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview**: This block initiates the workflow by capturing user inputs from a web form. It collects all necessary data for posting, including platform choice, account name, caption, media file, and optional Facebook Page ID.

- **Nodes Involved**:  
  - On form submission

- **Node Details**:

  - **On form submission**  
    - Type: Form Trigger  
    - Role: Entry point capturing user-submitted data via a web form.  
    - Configuration:  
      - Form title: "Post Publisher"  
      - Fields:  
        - Platform (dropdown): Instagram, LinkedIn, Facebook, X, TikTok, Threads (required)  
        - Account (text): User profile name configured on Upload-Post.com (required)  
        - Caption (textarea): Post text (required)  
        - Upload (file): Single file upload, accepts `.jpg` or `.mp4` (required)  
        - Facebook Id (text): Optional Facebook Page ID for targeted posting  
    - Expressions/Variables: Outputs JSON with all form fields, including file binary data under `Upload`.  
    - Input: Webhook trigger on form submission  
    - Output: Passes form data to the next node  
    - Edge cases:  
      - Missing required fields will prevent submission (handled by form UI)  
      - Unsupported file types rejected by form restrictions  
      - Large file uploads may cause timeouts or failures  

#### 2.2 Media Type Routing

- **Overview**: Determines if the uploaded file is an image or a video by inspecting the MIME type, routing the workflow accordingly to the appropriate upload node.

- **Nodes Involved**:  
  - Video or Photo?

- **Node Details**:

  - **Video or Photo?**  
    - Type: Switch  
    - Role: Routes flow based on MIME type of uploaded file.  
    - Configuration:  
      - Condition 1 (Image): MIME type equals `image/jpeg` → output "Image"  
      - Condition 2 (Video): MIME type equals `video/mp4` → output "Video"  
    - Expressions: Uses `{{$json.Upload.mimetype}}` to evaluate file type.  
    - Input: Form submission node output  
    - Output: Two branches — one to "Post photo" node, one to "Post video" node  
    - Edge cases:  
      - Unsupported MIME types (other than jpeg/mp4) are not handled explicitly and will cause no output branch to trigger, potentially halting the workflow.  
      - MIME type detection depends on accurate file metadata from form upload.  

#### 2.3 API Upload

- **Overview**: Sends the media file and metadata to Upload-Post.com API endpoints for posting photos or videos, using authenticated HTTP requests with multipart form data.

- **Nodes Involved**:  
  - Post photo  
  - Post video

- **Node Details**:

  - **Post photo**  
    - Type: HTTP Request  
    - Role: Uploads image posts to Upload-Post API endpoint `/api/upload_photos`.  
    - Configuration:  
      - Method: POST  
      - URL: `https://api.upload-post.com/api/upload_photos`  
      - Content-Type: multipart/form-data  
      - Authentication: HTTP Header Auth with API key (`Authorization: Apikey YOUR_API_KEY`)  
      - Body parameters:  
        - `title`: Caption from form (`{{$json.Caption}}`)  
        - `user`: Account/profile name (`{{$json.Account}}`)  
        - `platform[]`: Platform selected (`{{$json.Platform}}`)  
        - `photos[]`: Binary file from form upload (`Upload`)  
        - `facebook_id`: Optional Facebook Page ID from form (`$('On form submission').item.json['Facebook Id']`)  
    - Input: Routed from "Video or Photo?" switch node (Image branch)  
    - Output: API response JSON forwarded to "Result Photo" node  
    - Edge cases:  
      - Authentication failure if API key is invalid or missing  
      - API errors due to invalid parameters or file upload issues  
      - Network timeouts or server errors  
      - Missing or incorrect Facebook ID may cause posting errors on Facebook pages  

  - **Post video**  
    - Type: HTTP Request  
    - Role: Uploads video posts to Upload-Post API endpoint `/api/upload`.  
    - Configuration:  
      - Method: POST  
      - URL: `https://api.upload-post.com/api/upload`  
      - Content-Type: multipart/form-data  
      - Authentication: HTTP Header Auth with API key (`Authorization: Apikey YOUR_API_KEY`)  
      - Body parameters:  
        - `title`: Caption from form (`{{$json.Caption}}`)  
        - `user`: Account/profile name (`{{$json.Account}}`)  
        - `platform[]`: Platform selected (`{{$json.Platform}}`)  
        - `video`: Binary file from form upload (`Upload`)  
    - Input: Routed from "Video or Photo?" switch node (Video branch)  
    - Output: API response JSON forwarded to "Result Video" node  
    - Edge cases:  
      - Same as "Post photo" node, plus potential larger file size causing timeouts  
      - No Facebook ID parameter here, so Facebook page posting may not be supported for videos  

#### 2.4 API Response Handling

- **Overview**: Extracts the success status from the API response to determine if the upload succeeded or failed.

- **Nodes Involved**:  
  - Result Photo  
  - Result Video  
  - Success Photo?  
  - Success Video?

- **Node Details**:

  - **Result Photo**  
    - Type: Set  
    - Role: Parses photo upload API response to extract success boolean.  
    - Configuration:  
      - Sets a new field `result` to the boolean value of `results[platform].success` from API response JSON.  
      - Uses expression: `{{$json.results[$('Video or Photo?').item.json.Platform].success.toBoolean()}}`  
    - Input: Output from "Post photo" node  
    - Output: Passes to "Success Photo?" IF node  
    - Edge cases:  
      - If API response structure changes, expression may fail or return undefined  
      - Platform key must exist in response JSON  

  - **Result Video**  
    - Type: Set  
    - Role: Parses video upload API response to extract success boolean.  
    - Configuration:  
      - Same logic as "Result Photo" but for video upload response.  
    - Input: Output from "Post video" node  
    - Output: Passes to "Success Video?" IF node  
    - Edge cases: Same as "Result Photo"  

  - **Success Photo?**  
    - Type: IF  
    - Role: Checks if photo upload was successful (`result == true`).  
    - Configuration:  
      - Condition: `result` equals string `"true"` (case sensitive)  
    - Input: Output from "Result Photo"  
    - Output:  
      - True branch → "OK Photo" form node  
      - False branch → "KO Photo" form node  
    - Edge cases:  
      - If `result` is missing or not `"true"`, treated as failure  

  - **Success Video?**  
    - Type: IF  
    - Role: Checks if video upload was successful (`result == true`).  
    - Configuration: Same as "Success Photo?"  
    - Input: Output from "Result Video"  
    - Output:  
      - True branch → "OK Video" form node  
      - False branch → "KO Video" form node  
    - Edge cases: Same as "Success Photo?"  

#### 2.5 User Feedback

- **Overview**: Displays completion messages to the user based on success or failure of the upload.

- **Nodes Involved**:  
  - OK Photo  
  - KO Photo  
  - OK Video  
  - KO Video

- **Node Details**:

  - **OK Photo**  
    - Type: Form (Completion)  
    - Role: Shows success message after photo upload.  
    - Configuration:  
      - Completion title: "Congratulations!"  
      - Completion message: "Your post has been published correctly"  
    - Input: True branch from "Success Photo?"  
    - Output: Ends workflow for this branch  

  - **KO Photo**  
    - Type: Form (Completion)  
    - Role: Shows error message after photo upload failure.  
    - Configuration:  
      - Completion title: "Oops"  
      - Completion message: "There was an error"  
    - Input: False branch from "Success Photo?"  
    - Output: Ends workflow for this branch  

  - **OK Video**  
    - Type: Form (Completion)  
    - Role: Shows success message after video upload.  
    - Configuration: Same as "OK Photo"  
    - Input: True branch from "Success Video?"  
    - Output: Ends workflow for this branch  

  - **KO Video**  
    - Type: Form (Completion)  
    - Role: Shows error message after video upload failure.  
    - Configuration: Same as "KO Photo"  
    - Input: False branch from "Success Video?"  
    - Output: Ends workflow for this branch  

#### 2.6 Configuration Notes

- **Overview**: Provides users with setup instructions, API key management, and usage tips via sticky notes.

- **Nodes Involved**:  
  - Sticky Note  
  - Sticky Note1

- **Node Details**:

  - **Sticky Note**  
    - Type: Sticky Note  
    - Role: Contains detailed setup instructions for API key, authentication header, profile management, and a note on YouTube integration status.  
    - Content Highlights:  
      - API key location: [Upload-Post Manage Api Keys](https://app.upload-post.com/)  
      - Header setup: `Authorization: Apikey YOUR_API_KEY_HERE`  
      - Profile names correspond to "Account" field in form  
      - YouTube integration pending Google verification  
      - Includes a workflow architecture image  
    - Position: Top-left of canvas for visibility  

  - **Sticky Note1**  
    - Type: Sticky Note  
    - Role: Marketing and feature summary note describing the workflow’s purpose and benefits.  
    - Content Highlights:  
      - Supports TikTok, X, Facebook, YouTube, Instagram, Threads, LinkedIn  
      - Simplifies social media posting for content creators and marketers  
      - Upload once, distribute everywhere  
    - Positioned above main workflow area  

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role                          | Input Node(s)          | Output Node(s)          | Sticky Note                                                                                      |
|---------------------|---------------------|----------------------------------------|-----------------------|-------------------------|------------------------------------------------------------------------------------------------|
| On form submission  | Form Trigger        | Captures user input from web form      | —                     | Video or Photo?          |                                                                                                |
| Video or Photo?     | Switch              | Routes flow based on file MIME type    | On form submission    | Post photo, Post video   |                                                                                                |
| Post photo          | HTTP Request        | Uploads image post via Upload-Post API | Video or Photo? (Image) | Result Photo             |                                                                                                |
| Post video          | HTTP Request        | Uploads video post via Upload-Post API | Video or Photo? (Video) | Result Video             |                                                                                                |
| Result Photo        | Set                 | Extracts success boolean from photo API response | Post photo            | Success Photo?           |                                                                                                |
| Result Video        | Set                 | Extracts success boolean from video API response | Post video            | Success Video?           |                                                                                                |
| Success Photo?      | IF                  | Checks photo upload success            | Result Photo          | OK Photo (true), KO Photo (false) |                                                                                                |
| Success Video?      | IF                  | Checks video upload success            | Result Video          | OK Video (true), KO Video (false) |                                                                                                |
| OK Photo            | Form (Completion)   | Shows success message for photo upload | Success Photo? (true) | —                       |                                                                                                |
| KO Photo            | Form (Completion)   | Shows failure message for photo upload | Success Photo? (false)| —                       |                                                                                                |
| OK Video            | Form (Completion)   | Shows success message for video upload | Success Video? (true) | —                       |                                                                                                |
| KO Video            | Form (Completion)   | Shows failure message for video upload | Success Video? (false)| —                       |                                                                                                |
| Sticky Note         | Sticky Note         | Setup instructions and API key info    | —                     | —                       | ## SETTINGS - Find your API key in your [Upload-Post Manage Api Keys](https://app.upload-post.com/) 10 FREE uploads per month; Set the "Auth Header": Name: Authorization, Value: Apikey YOUR_API_KEY_HERE; Create profiles to manage your social media accounts. The "Profile" you choose will be used in the field "Account" on form submission (eg. test1 or test2). Apr-2025: YouTube integration is currently being verified by Google, and may not work as expected. ![image](https://i.postimg.cc/N0BHdcQ4/uploadpost.png) |
| Sticky Note1        | Sticky Note         | Marketing and feature summary           | —                     | —                       | ## Upload video and photo on TikTok, X, Facebook, Youtube, Instagram, Threads and Linkedin. Simplify your social media workflow with our powerful platform designed for content creators and marketers. Upload your video once and let Upload-Post distribute it across all your connected social media accounts effortlessly. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger Node**  
   - Add a **Form Trigger** node named "On form submission".  
   - Configure form title: "Post Publisher".  
   - Add form fields:  
     - Dropdown "Platform" with options: instagram, linkedin, facebook, x, tiktok, threads (required).  
     - Text "Account" (required), placeholder: "User Profile name set on Upload-post.com".  
     - Textarea "Caption" (required).  
     - File upload "Upload" (required), accept `.jpg,.mp4`, single file only.  
     - Text "Facebook Id" (optional), placeholder: "Facebook page Id (eg. 00000111122222)".  
   - Save and note the webhook URL generated.

2. **Add a Switch Node to Detect Media Type**  
   - Add a **Switch** node named "Video or Photo?".  
   - Add two rules:  
     - Output "Image": Condition where `{{$json.Upload.mimetype}}` equals `image/jpeg`.  
     - Output "Video": Condition where `{{$json.Upload.mimetype}}` equals `video/mp4`.  
   - Connect "On form submission" output to this node.

3. **Add HTTP Request Node for Photo Upload**  
   - Add an **HTTP Request** node named "Post photo".  
   - Set method to POST.  
   - URL: `https://api.upload-post.com/api/upload_photos`.  
   - Content type: multipart/form-data.  
   - Authentication: HTTP Header Auth with your Upload-Post API key.  
     - Header name: `Authorization`  
     - Header value: `Apikey YOUR_API_KEY_HERE` (replace with your actual key).  
   - Body parameters:  
     - `title`: Expression `{{$json.Caption}}`  
     - `user`: Expression `{{$json.Account}}`  
     - `platform[]`: Expression `{{$json.Platform}}`  
     - `photos[]`: Set as binary form data, input field name `Upload`  
     - `facebook_id`: Expression `$('On form submission').item.json['Facebook Id']` (optional)  
   - Connect "Video or Photo?" node's "Image" output to this node.

4. **Add HTTP Request Node for Video Upload**  
   - Add an **HTTP Request** node named "Post video".  
   - Set method to POST.  
   - URL: `https://api.upload-post.com/api/upload`.  
   - Content type: multipart/form-data.  
   - Authentication: HTTP Header Auth with your Upload-Post API key.  
   - Body parameters:  
     - `title`: Expression `{{$json.Caption}}`  
     - `user`: Expression `{{$json.Account}}`  
     - `platform[]`: Expression `{{$json.Platform}}`  
     - `video`: Set as binary form data, input field name `Upload`  
   - Connect "Video or Photo?" node's "Video" output to this node.

5. **Add Set Node to Parse Photo API Response**  
   - Add a **Set** node named "Result Photo".  
   - Add a field named `result` (string).  
   - Set value to expression:  
     `{{$json.results[$('Video or Photo?').item.json.Platform].success.toBoolean()}}`  
   - Connect "Post photo" node output to this node.

6. **Add Set Node to Parse Video API Response**  
   - Add a **Set** node named "Result Video".  
   - Add a field named `result` (string).  
   - Set value to expression:  
     `{{$json.results[$('Video or Photo?').item.json.Platform].success.toBoolean()}}`  
   - Connect "Post video" node output to this node.

7. **Add IF Node to Check Photo Upload Success**  
   - Add an **IF** node named "Success Photo?".  
   - Condition: `result` equals string `"true"` (case sensitive).  
   - Connect "Result Photo" output to this node.

8. **Add IF Node to Check Video Upload Success**  
   - Add an **IF** node named "Success Video?".  
   - Condition: `result` equals string `"true"`.  
   - Connect "Result Video" output to this node.

9. **Add Completion Form Nodes for Photo Success and Failure**  
   - Add a **Form** node named "OK Photo".  
     - Operation: Completion  
     - Completion title: "Congratulations!"  
     - Completion message: "Your post has been published correctly"  
   - Add a **Form** node named "KO Photo".  
     - Operation: Completion  
     - Completion title: "Oops"  
     - Completion message: "There was an error"  
   - Connect "Success Photo?" node:  
     - True output → "OK Photo"  
     - False output → "KO Photo"

10. **Add Completion Form Nodes for Video Success and Failure**  
    - Add a **Form** node named "OK Video".  
      - Same configuration as "OK Photo".  
    - Add a **Form** node named "KO Video".  
      - Same configuration as "KO Photo".  
    - Connect "Success Video?" node:  
      - True output → "OK Video"  
      - False output → "KO Video"

11. **Add Sticky Notes for Setup Instructions (Optional)**  
    - Add a **Sticky Note** with setup instructions:  
      - API key location and usage  
      - Header configuration  
      - Profile/account mapping  
      - YouTube integration note  
      - Include image link if desired  
    - Add a second **Sticky Note** with marketing and feature summary.

12. **Activate the Workflow**  
    - Save and activate the workflow.  
    - Test by submitting the form with image and video files.  
    - Verify success and failure messages appear correctly.

---

### 5. General Notes & Resources

| Note Content                                                                                                                               | Context or Link                                                                                      |
|--------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Upload-Post.com API key can be obtained from the [API Keys dashboard](https://app.upload-post.com/). Free tier allows 10 uploads/month.    | API key management                                                                                   |
| Set HTTP header `Authorization` with value `Apikey YOUR_API_KEY_HERE` in HTTP Request nodes for authentication.                           | Authentication setup                                                                                |
| Profile names configured on Upload-Post.com must match the "Account" field in the form submission.                                         | Account/profile mapping                                                                             |
| YouTube integration is pending Google verification as of April 2025 and may not function as expected.                                      | Platform support update                                                                             |
| Workflow supports posting to Instagram, LinkedIn, Facebook, X, TikTok, and Threads simultaneously.                                         | Multi-platform support                                                                             |
| For customization or consulting, contact via email: info@n3w.it or LinkedIn: https://www.linkedin.com/in/davideboizza/                     | Support and consulting                                                                              |
| Workflow screenshot and architecture diagram available at: ![image](https://i.postimg.cc/N0BHdcQ4/uploadpost.png)                          | Visual reference                                                                                   |

---

This document provides a detailed, structured reference for understanding, reproducing, and maintaining the "Social Media Publisher" n8n workflow, ensuring clarity for both advanced users and automation agents.