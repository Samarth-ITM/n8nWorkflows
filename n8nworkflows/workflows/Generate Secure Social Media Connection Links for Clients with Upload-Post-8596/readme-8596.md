Generate Secure Social Media Connection Links for Clients with Upload-Post

https://n8nworkflows.xyz/workflows/generate-secure-social-media-connection-links-for-clients-with-upload-post-8596


# Generate Secure Social Media Connection Links for Clients with Upload-Post

### 1. Workflow Overview

This workflow is designed for agencies or social media managers to securely onboard clients for social media management without requiring clients’ passwords. It automates the creation of a client user in Upload-Post, generates a secure, branded one-hour JWT connection link, and sends this link via Telegram to the client. Once the client connects their social accounts through Upload-Post’s Connect page, they can submit posts via a simple publisher form that uploads videos to multiple social platforms (Facebook, Instagram, TikTok, YouTube) using Upload-Post.

Logical blocks in the workflow:

- **1.1 Initialization Trigger and User Setup:** Manual trigger to start the process, creating or reusing a user in Upload-Post.
- **1.2 JWT Link Generation and Notification:** Generates a secure JWT connect link branded with a logo and sends it to the client via Telegram.
- **1.3 Client Post Submission Handling:** Waits for the client to submit a post via a form, then uploads the video post to selected platforms via Upload-Post.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization Trigger and User Setup

- **Overview:**  
This block starts the workflow manually and ensures a user profile exists in Upload-Post, either creating a new one or reusing an existing user.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Create user (Upload-Post node)

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow manually for testing or manual execution.  
    - Configuration: Default manual trigger, no parameters.  
    - Input: None  
    - Output: Triggers "Create user" node.  
    - Edge cases: None typical; user must manually trigger to initiate.  

  - **Create user**  
    - Type: Upload-Post API node (resource: users, operation: createUser)  
    - Role: Creates or reuses a user profile on Upload-Post identified by "add_user_name".  
    - Configuration:  
      - Parameter `newUser` set to "add_user_name" (likely a fixed or variable username).  
      - Uses Upload-Post API credentials named "Smoker".  
    - Input: Triggered by manual node  
    - Output: Passes user data to JWT generation node.  
    - Edge cases:  
      - API errors (network, authentication) could cause failure.  
      - User creation may fail if username already exists — workflow mentions reuse, but the node config implies creation only, so potential error if user exists.  
    - Version requirements: Upload-Post API version 1.

---

#### 2.2 JWT Link Generation and Notification

- **Overview:**  
Generates a secure JWT Connect Accounts link branded with a logo, valid for 1 hour, and sends it via Telegram to the client.

- **Nodes Involved:**  
  - Generate jwt for platform integration (Upload-Post node)  
  - Send a text message (Telegram node)

- **Node Details:**

  - **Generate jwt for platform integration**  
    - Type: Upload-Post API node (resource: users, operation: generateJwt)  
    - Role: Produces a JWT URL for the client to securely connect their social accounts.  
    - Configuration:  
      - User: "add_user_name" (same as created user)  
      - Logo Image: URL to a logo image (`https://tattooservices.es/wp-content/uploads/2020/07/logo-community-manager.png`) to brand the Connect page.  
      - Uses same Upload-Post API credentials ("Smoker").  
    - Input: Receives from "Create user" node.  
    - Output: Provides a JSON object containing `access_url` (the JWT link).  
    - Edge cases:  
      - API failures or invalid user references.  
      - Logo URL must be accessible; otherwise, branding might fail or default.  
    - Version: Upload-Post API v1.

  - **Send a text message**  
    - Type: Telegram node  
    - Role: Sends the generated JWT connect link to the client via Telegram chat.  
    - Configuration:  
      - Text message template: `"Url for connect accounts generated: {{ $json.access_url }}"` dynamically inserts the generated URL.  
      - Chat ID: `-4127128831` (Telegram group or user chat).  
      - Credentials: Telegram Bot API credentials ("Telegram account").  
    - Input: Receives from JWT generation node.  
    - Output: None (end node).  
    - Edge cases:  
      - Telegram API errors (invalid token, chat ID issues).  
      - Network or rate limits.  
    - Version: Telegram node v1.2.

---

#### 2.3 Client Post Submission Handling

- **Overview:**  
Receives client-submitted posts via a form trigger, then uploads the submitted video to multiple social platforms through Upload-Post.

- **Nodes Involved:**  
  - On form submission (Form Trigger node)  
  - Upload a video (Upload-Post node)

- **Node Details:**

  - **On form submission**  
    - Type: Form Trigger  
    - Role: Webhook that captures client submissions from a post publisher form.  
    - Configuration:  
      - Form Title: "Post Publisher"  
      - Fields:  
        - Upload-Post Account (required text, profile name)  
        - Description (required textarea)  
        - Upload (required single file, accepts `.jpg` and `.mp4`)  
        - Facebook Id (optional text, for Facebook Page ID)  
      - Webhook ID: Automatically generated (used to receive form submissions).  
    - Input: External HTTP POST from form submission.  
    - Output: Passes form data to "Upload a video" node.  
    - Edge cases:  
      - Missing required fields cause form validation failure (client-side or server-side).  
      - Invalid file types or sizes might cause upload errors downstream.  
      - Facebook Id optional but must be valid if provided.  
    - Version: Form Trigger v2.2.

  - **Upload a video**  
    - Type: Upload-Post API node (resource: video upload)  
    - Role: Uploads the submitted video and posts it to multiple social media platforms via Upload-Post.  
    - Configuration:  
      - User: dynamically set from form field "Upload-Post Account" (`=add_user_name` expression from form data).  
      - Title: taken from form "Description" field.  
      - Video: uses uploaded file from "Upload" field.  
      - Platforms: Facebook, Instagram, TikTok, YouTube (all enabled).  
      - Facebook Page ID: optionally passed from form "Facebook Id" field.  
      - Operation: "uploadVideo".  
      - Credentials: Upload-Post API named "Smoker".  
    - Input: Receives form submission JSON.  
    - Output: None (end node).  
    - Edge cases:  
      - Upload-Post API errors (authentication, invalid token).  
      - Invalid user profile name causes upload failure.  
      - Invalid or missing media file causes failure.  
      - Facebook Page ID must be valid if provided, otherwise Facebook posting might fail.  
    - Version: Upload-Post API v1.

---

### 3. Summary Table

| Node Name                      | Node Type                  | Functional Role                                  | Input Node(s)                 | Output Node(s)                      | Sticky Note                                                                                                              |
|-------------------------------|----------------------------|-------------------------------------------------|-------------------------------|------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| About this workflow            | Sticky Note                | Describes workflow purpose, usage, and tips    | None                          | None                               | Designed for agencies and social media managers: creates secure Connect Accounts page and sends links via Telegram.       |
| When clicking ‘Execute workflow’ | Manual Trigger             | Starts workflow manually                        | None                          | Create user                        |                                                                                                                          |
| Create user                   | Upload-Post API            | Creates or reuses Upload-Post user profile     | When clicking ‘Execute workflow’ | Generate jwt for platform integration |                                                                                                                          |
| Generate jwt for platform integration | Upload-Post API            | Generates secure branded JWT connect link      | Create user                   | Send a text message                |                                                                                                                          |
| Send a text message           | Telegram                   | Sends JWT connect link via Telegram             | Generate jwt for platform integration | None                               |                                                                                                                          |
| On form submission            | Form Trigger               | Receives client post submissions via form      | None                          | Upload a video                    |                                                                                                                          |
| Upload a video                | Upload-Post API            | Uploads video post to multiple social platforms | On form submission            | None                               |                                                                                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add node: Manual Trigger  
   - Name: `When clicking ‘Execute workflow’`  
   - No parameters needed.

2. **Create Upload-Post ‘Create user’ Node**  
   - Add Upload-Post node  
   - Name: `Create user`  
   - Resource: `users`  
   - Operation: `createUser`  
   - Parameter: `newUser` set to a fixed or input username (e.g., `"add_user_name"`)  
   - Attach Upload-Post API credentials (e.g., "Smoker").  
   - Connect output of Manual Trigger node to input of this node.

3. **Create Upload-Post ‘Generate JWT’ Node**  
   - Add Upload-Post node  
   - Name: `Generate jwt for platform integration`  
   - Resource: `users`  
   - Operation: `generateJwt`  
   - Parameter `user` set to `"add_user_name"` (same as create)  
   - Parameter `logoImage` set to `"https://tattooservices.es/wp-content/uploads/2020/07/logo-community-manager.png"`  
   - Attach same Upload-Post API credentials.  
   - Connect output of `Create user` node to input of this node.

4. **Create Telegram Node to Send Message**  
   - Add Telegram node  
   - Name: `Send a text message`  
   - Operation: Send Message (text)  
   - Text: Use expression `=Url for connect accounts generated: {{ $json.access_url }}` to insert JWT URL  
   - Chat ID: `-4127128831` (replace with your target Telegram chat ID)  
   - Attach Telegram API credentials (Telegram Bot).  
   - Connect output of JWT generation node to input of this node.

5. **Create Form Trigger Node for Client Post Submission**  
   - Add Form Trigger node  
   - Name: `On form submission`  
   - Configure form:  
     - Title: `Post Publisher`  
     - Fields:  
       - Text field: "Upload-Post Account" (required)  
       - Textarea: "Description" (required)  
       - File upload: "Upload" (required, accept `.jpg`, `.mp4`, single file)  
       - Text field: "Facebook Id" (optional)  
   - No input connections (webhook trigger).

6. **Create Upload-Post ‘Upload a video’ Node**  
   - Add Upload-Post node  
   - Name: `Upload a video`  
   - Resource: video upload or relevant resource in Upload-Post  
   - Operation: `uploadVideo`  
   - Parameters:  
     - `user`: Set from form field "Upload-Post Account" (use expression `{{$json["Upload-Post Account"]}}` or `add_user_name` if fixed)  
     - `title`: Set to form field "Description" (`{{$json.Description}}`)  
     - `video`: Set to uploaded file from form field "Upload"  
     - `platform`: Select Facebook, Instagram, TikTok, YouTube  
     - `facebookPageId`: Optional, set from form field "Facebook Id"  
   - Attach Upload-Post API credentials.  
   - Connect output of form trigger to this node.

7. **Test Workflow**  
   - Manually trigger the first part to create user, generate JWT, and send Telegram message.  
   - Submit the form to the webhook URL provided by the form trigger node to test video upload.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                           | Context or Link                                                                                       |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| The Connect link generated expires after 1 hour (TTL). Regenerate as needed for security.                                                                             | Workflow description.                                                                                |
| Brand the Connect page with your own `brandName` and `logoImage` for client trust and recognition.                                                                   | Workflow description.                                                                                |
| Clients can publish posts after connecting accounts via Upload-Post Dashboard, API, or custom n8n flows using their profile name.                                   | Workflow description.                                                                                |
| Telegram Bot is used for notification; can be replaced with Email/Gmail node if preferred.                                                                            | Workflow description.                                                                                |
| Upload-Post API credentials must be configured properly with API key named (example) "Smoker" for all Upload-Post nodes.                                              | Credentials setup.                                                                                   |
| For Telegram Node, ensure Bot token and chat ID correspond to your Telegram bot and client chat/group.                                                                | Credentials setup.                                                                                   |
| More info on Upload-Post platform: https://app.upload-post.com/dashboard                                                                                             | External resource for post management.                                                              |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.