Simple Social: Instagram Single Image Post with Facebook API

https://n8nworkflows.xyz/workflows/simple-social--instagram-single-image-post-with-facebook-api-2537


# Simple Social: Instagram Single Image Post with Facebook API

### 1. Workflow Overview

This workflow automates posting a single image with a caption to an Instagram Business account using the Facebook Graph API. It is intended for social media managers, businesses, content creators, and developers aiming to streamline Instagram posting by automating the media upload, publishing, status verification, and notification process.

The workflow is logically divided into these blocks:

- **1.1 Trigger Initialization**: Starts the workflow manually but can be adapted to other triggers.
- **1.2 Parameter Setup**: Defines required parameters like image URL, Instagram Business Account ID, and post caption.
- **1.3 Prepare Instagram Media**: Sends the media and caption to Facebook API for preparation.
- **1.4 Verify Media Preparation**: Checks if the media is processed and ready to publish.
- **1.5 Conditional Publishing Decision**: Routes flow based on media preparation status.
- **1.6 Publish Media on Instagram**: Publishes the prepared media using Facebook API.
- **1.7 Verify Publication Status**: Checks if the media was successfully published.
- **1.8 Conditional Post-Publish Decision**: Routes flow based on publication success.
- **1.9 Email Notifications**: Sends emails on success, failure, or other outcomes for monitoring and alerting.

Additional disabled node to retrieve Instagram Business Account ID is included as a utility.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger Initialization

- **Overview**: Initiates the workflow manually, allowing easy testing or replacement with other triggers.
- **Nodes Involved**: 
  - When clicking ‘Test workflow’
  - Sticky Note (trigger guidance)
- **Node Details**:
  - *When clicking ‘Test workflow’*  
    - Type: Manual Trigger  
    - Role: Initiates the workflow manually.  
    - Config: Default manual trigger, no parameters.  
    - Connections: Outputs to `Instagram params`.  
    - Edge Cases: None; manual trigger avoids automated scheduling or webhook issues.  
  - *Sticky Note*  
    - Role: Advises users to select or change the trigger node as needed.  
    - Content: "Choose the trigger you want for example trigger this workflow from another one"  

#### 1.2 Parameter Setup

- **Overview**: Defines essential input parameters for posting: image URL, Instagram Business Account ID, and caption.
- **Nodes Involved**: 
  - Instagram params
  - Sticky Note3 (parameters explanation)
  - Sticky Note1 (credential reminder)
- **Node Details**:
  - *Instagram params*  
    - Type: Set Node  
    - Role: Sets three string parameters: `image_url`, `instagram_business_account_id`, `instagram_post_caption`.  
    - Configuration: Placeholder values to be customized with actual data.  
    - Connections: Outputs to `Instagram prepare media`.  
    - Expressions: Parameters are later referenced via expressions in API calls.  
    - Edge Cases: Missing or incorrect parameters will cause API errors downstream.  
  - *Sticky Note3*  
    - Role: Documents the parameters in the UI.  
    - Content: "Here we have all parameters for posting in instagram image url, caption and instagram business profile id"  
  - *Sticky Note1*  
    - Role: Reminder to add Facebook API credentials for authentication.  
    - Content: "Add your credential"  

#### 1.3 Prepare Instagram Media

- **Overview**: Sends a POST request to Facebook API to create a media container with the image and caption.
- **Nodes Involved**:  
  - Instagram prepare media
- **Node Details**:
  - *Instagram prepare media*  
    - Type: Facebook Graph API node  
    - Role: Creates an Instagram media container (prepares media for upload).  
    - Configuration:  
      - HTTP Method: POST  
      - Endpoint: `/{instagram_business_account_id}/media`  
      - Query Parameters: `image_url` and `caption` from `Instagram params` node.  
      - Graph API Version: v20.0  
    - Input: Reads parameters from the JSON input (`$json`).  
    - Output: Returns media container ID and status fields.  
    - Edge Cases:  
      - API authorization errors (invalid credentials).  
      - Invalid image URL or unsupported image format errors.  
      - Rate limiting by Instagram API.  
    - Failure Handling: Not explicit; workflow branches after status check.  

#### 1.4 Verify Media Preparation

- **Overview**: Queries the status of the media container to confirm it is ready for publishing.
- **Nodes Involved**:  
  - Instagram check status of media uploaded before
- **Node Details**:
  - *Instagram check status of media uploaded before*  
    - Type: Facebook Graph API node  
    - Role: GET request to check media container status.  
    - Configuration:  
      - Method: GET  
      - Endpoint: `/{media_container_id}` (from previous node output)  
      - Fields: `id`, `status`, `status_code`  
      - API Version: v20.0  
    - Input: Uses media container ID from the previous node's JSON output.  
    - Output: Status fields indicating readiness.  
    - Edge Cases:  
      - API errors or timeouts.  
      - Media container not found or deleted.  

#### 1.5 Conditional Publishing Decision

- **Overview**: Checks if the media preparation status code equals "FINISHED" to decide if publishing should proceed.
- **Nodes Involved**:  
  - If media status is finished
- **Node Details**:
  - *If media status is finished*  
    - Type: If Node  
    - Role: Conditional branching based on media preparation status.  
    - Configuration: Checks if `$json.status_code === "FINISHED"`  
    - True Output: Connects to `Instagram publish media` to proceed with publishing.  
    - False Output: Connects to `Send Email` node to notify failure.  
    - Edge Cases:  
      - Status codes other than "FINISHED" may indicate processing or errors.  
      - Expression evaluation errors if JSON data missing.  

#### 1.6 Publish Media on Instagram

- **Overview**: Publishes the prepared media container to Instagram via the Facebook API.
- **Nodes Involved**:  
  - Instagram publish media
- **Node Details**:
  - *Instagram publish media*  
    - Type: Facebook Graph API node  
    - Role: POST request to publish media container.  
    - Configuration:  
      - Method: POST  
      - Endpoint: `/{instagram_business_account_id}/media_publish`  
      - Query Parameter: `creation_id` set to media container `id` from preparation step.  
      - API Version: v20.0  
    - Input: Takes Instagram Business ID and creation ID from previous node outputs.  
    - Output: Returns publication ID and status.  
    - Edge Cases:  
      - Publishing errors due to rate limits or invalid creation ID.  
      - Authorization failures.  

#### 1.7 Verify Publication Status

- **Overview**: Checks if the published media's status is "PUBLISHED" to confirm success.
- **Nodes Involved**:  
  - Instagram check status of media published before
- **Node Details**:
  - *Instagram check status of media published before*  
    - Type: Facebook Graph API node  
    - Role: GET request to check published media status.  
    - Configuration:  
      - Method: GET  
      - Endpoint: `/{media_published_id}` from previous node output.  
      - Fields: `id`, `status`, `status_code`  
      - API Version: v20.0  
    - Input: Uses media publish response `id`.  
    - Output: Status fields for final publication.  
    - Edge Cases:  
      - API failure or delayed status updates.  

#### 1.8 Conditional Post-Publish Decision

- **Overview**: Branches workflow depending on whether the media status code equals "PUBLISHED".
- **Nodes Involved**:  
  - If media status is finished1
- **Node Details**:
  - *If media status is finished1*  
    - Type: If Node  
    - Role: Conditional check if media is successfully published.  
    - Configuration: Checks `$json.status_code === "PUBLISHED"`  
    - True Output: Connects to `Send Email1` node (success notification).  
    - False Output: Connects to `Send Email2` node (failure notification).  
    - Edge Cases:  
      - Incorrect or delayed status may misroute flow.  

#### 1.9 Email Notifications

- **Overview**: Sends email notifications based on upload or publishing outcomes for monitoring.
- **Nodes Involved**:  
  - Send Email  
  - Send Email1  
  - Send Email2  
  - Sticky Note5, Sticky Note7, Sticky Note8 (email notes)
- **Node Details**:
  - *Send Email*  
    - Type: Email Send node  
    - Role: Sends email on unsuccessful media preparation.  
    - Configuration: User-defined SMTP or email credentials required.  
    - Connected to "If media status is finished" false branch.  
  - *Send Email1*  
    - Type: Email Send node  
    - Role: Sends email on successful media publishing.  
    - Connected to "If media status is finished1" true branch.  
  - *Send Email2*  
    - Type: Email Send node  
    - Role: Sends email on unsuccessful media publishing.  
    - Connected to "If media status is finished1" false branch.  
  - *Sticky Notes*  
    - Sticky Note5 content: "You can send email for unsuccessful upload or what you want, you can trigger another workflow or another node"  
    - Sticky Note7 content: "You can send email for unsuccessful publishing or what you want, you can trigger another workflow or another node"  
    - Sticky Note8 content: "You can send email for successfull publishing or what you want, you can trigger another workflow or another node"  
  - Edge Cases:  
    - Email sending failures due to credential issues or SMTP downtime.  

#### Utility Node (Disabled)

- **Overview**: Optional node to retrieve Instagram Business Account ID from Facebook Graph API.
- **Nodes Involved**:  
  - Node just for retrieve id of instagram page (disabled)  
  - Sticky Note2 explaining usage
- **Node Details**:
  - *Node just for retrieve id of instagram page*  
    - Type: Facebook Graph API node  
    - Role: GET request to `/me` endpoint with fields `name` and `connected_instagram_account`.  
    - Use: Helps users find the Instagram Business Account ID if unknown.  
    - Disabled by default; can be enabled for manual use.  
  - *Sticky Note2*  
    - Content: "You can use this node if you want to retrieve the instagram id. Add it to the workflow or use manually"  

---

### 3. Summary Table

| Node Name                              | Node Type               | Functional Role                                      | Input Node(s)                      | Output Node(s)                          | Sticky Note                                                                                              |
|--------------------------------------|-------------------------|-----------------------------------------------------|----------------------------------|---------------------------------------|--------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’         | Manual Trigger          | Initiates the workflow manually                      |                                  | Instagram params                      | Choose the trigger you want for example trigger this workflow from another one                          |
| Instagram params                      | Set                     | Sets image URL, Instagram business ID, and caption | When clicking ‘Test workflow’    | Instagram prepare media               | Here we have all parameters for posting in instagram image url, caption and instagram business profile id |
| Instagram prepare media               | Facebook Graph API      | Creates Instagram media container                    | Instagram params                 | Instagram check status of media uploaded before | Add your credential                                                                                      |
| Instagram check status of media uploaded before | Facebook Graph API      | Checks media container upload status                 | Instagram prepare media          | If media status is finished            |                                                                                                        |
| If media status is finished           | If                      | Checks if media upload is finished                   | Instagram check status of media uploaded before | Instagram publish media, Send Email         |                                                                                                        |
| Instagram publish media               | Facebook Graph API      | Publishes media container on Instagram               | If media status is finished      | Instagram check status of media published before |                                                                                                        |
| Instagram check status of media published before | Facebook Graph API      | Checks published media status                         | Instagram publish media          | If media status is finished1           |                                                                                                        |
| If media status is finished1          | If                      | Checks if media is published                          | Instagram check status of media published before | Send Email1, Send Email2                  |                                                                                                        |
| Send Email                          | Email Send               | Sends email on unsuccessful media upload             | If media status is finished (false) |                                   | You can send email for unsuccessful upload or what you want, you can trigger another workflow or another node |
| Send Email1                         | Email Send               | Sends email on successful media publishing            | If media status is finished1 (true) |                                   | You can send email for successfull publishing or what you want, you can trigger another workflow or another node |
| Send Email2                         | Email Send               | Sends email on unsuccessful media publishing          | If media status is finished1 (false) |                                   | You can send email for unsuccessful publishing or what you want, you can trigger another workflow or another node |
| Node just for retrive id of instagram page (disabled) | Facebook Graph API      | Utility to retrieve Instagram Business Account ID    |                                  |                                       | You can use this node if you want to retrieve the instagram id. Add it to the workflow ore use manually  |
| Sticky Note                         | Sticky Note              | Advice on trigger choice                              |                                  |                                       | Choose the trigger you want for example trigger this workflow from another one                          |
| Sticky Note1                        | Sticky Note              | Reminder to add credentials                           |                                  |                                       | Add your credential                                                                                      |
| Sticky Note2                        | Sticky Note              | Explains Instagram ID retrieval node                 |                                  |                                       | You can use this node if you want to retrieve the instagram id. Add it to the workflow ore use manually  |
| Sticky Note3                        | Sticky Note              | Explains parameters set                               |                                  |                                       | Here we have all parameters for posting in instagram image url, caption and instagram business profile id |
| Sticky Note4                        | Sticky Note              | Explains API permissions, limitations, and rate limit |                                  |                                       | Permissions and limitations details, rate limits, and app review info (detailed content)                |
| Sticky Note5                        | Sticky Note              | Notes on email for unsuccessful upload               |                                  |                                       | You can send email for unsuccessful upload or what you want, you can trigger another workflow or another node |
| Sticky Note6                        | Sticky Note              | Introductory note about the workflow                  |                                  |                                       | Instagram single image Post Workflow with Facebook API - P.S: change default URL in node if needed       |
| Sticky Note7                        | Sticky Note              | Notes on email for unsuccessful publishing           |                                  |                                       | You can send email for unsuccessful publishing or what you want, you can trigger another workflow or another node |
| Sticky Note8                        | Sticky Note              | Notes on email for successful publishing             |                                  |                                       | You can send email for successfull publishing or what you want, you can trigger another workflow or another node |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Position: Start of workflow  
   - No parameters needed  

2. **Create Set Node ("Instagram params")**  
   - Connect from Manual Trigger  
   - Define three string parameters:  
     - `image_url`: The full URL of the image to post (e.g., "https://example.com/image.jpg")  
     - `instagram_business_account_id`: Your Instagram Business Account ID as string  
     - `instagram_post_caption`: Text caption for the Instagram post  
   - Position: After the trigger node  

3. **Add Facebook Graph API Node ("Instagram prepare media")**  
   - Connect from "Instagram params"  
   - Set:  
     - HTTP request method: POST  
     - Node (path): `={{ $json.instagram_business_account_id }}`  
     - Edge: `media`  
     - Query parameters:  
       - `image_url`: `={{ $json.image_url }}`  
       - `caption`: `={{ $json.instagram_post_caption }}`  
     - API Version: v20.0  
   - Credentials: Configure Facebook Graph API credentials with required permissions (see notes)  
   - Position: After "Instagram params"  

4. **Add Facebook Graph API Node ("Instagram check status of media uploaded before")**  
   - Connect from "Instagram prepare media"  
   - Set:  
     - HTTP method: GET  
     - Node (path): `={{ $json.id }}` (media container ID from previous node)  
     - Fields: `id,status,status_code`  
     - API Version: v20.0  
   - Position: After "Instagram prepare media"  

5. **Add If Node ("If media status is finished")**  
   - Connect from "Instagram check status of media uploaded before"  
   - Condition: Check if `$json.status_code === "FINISHED"` (case sensitive, strict)  
   - True branch: Connect to "Instagram publish media"  
   - False branch: Connect to "Send Email" for failure notification  
   - Position: After status check node  

6. **Add Facebook Graph API Node ("Instagram publish media")**  
   - Connect from If Node (true branch)  
   - Set:  
     - HTTP method: POST  
     - Node (path): `={{ $('Instagram params').item.json.instagram_business_account_id }}`  
     - Edge: `media_publish`  
     - Query parameter:  
       - `creation_id`: `={{ $json.id }}` (media container ID from prepare media node)  
     - API Version: v20.0  
   - Position: After If Node (true)  

7. **Add Facebook Graph API Node ("Instagram check status of media published before")**  
   - Connect from "Instagram publish media"  
   - Set:  
     - HTTP method: GET  
     - Node (path): `={{ $json.id }}` (published media ID from previous node)  
     - Fields: `id,status,status_code`  
     - API Version: v20.0  
   - Position: After "Instagram publish media"  

8. **Add If Node ("If media status is finished1")**  
   - Connect from "Instagram check status of media published before"  
   - Condition: Check if `$json.status_code === "PUBLISHED"`  
   - True branch: Connect to "Send Email1" (success notification)  
   - False branch: Connect to "Send Email2" (failure notification)  
   - Position: After status check node  

9. **Add Email Send Node ("Send Email")**  
   - Connect from If Node "If media status is finished" (false branch)  
   - Configure SMTP or other email credentials  
   - Compose email for unsuccessful media preparation notification  
   - Position: Failure branch after first If Node  

10. **Add Email Send Node ("Send Email1")**  
    - Connect from If Node "If media status is finished1" (true branch)  
    - Configure SMTP or other email credentials  
    - Compose email for successful media publishing notification  
    - Position: Success branch after second If Node  

11. **Add Email Send Node ("Send Email2")**  
    - Connect from If Node "If media status is finished1" (false branch)  
    - Configure SMTP or other email credentials  
    - Compose email for unsuccessful media publishing notification  
    - Position: Failure branch after second If Node  

12. **(Optional) Add Facebook Graph API Node ("Node just for retrieve id of instagram page")**  
    - Disabled by default  
    - HTTP method: GET  
    - Node: `me`  
    - Fields: `name`, `connected_instagram_account`  
    - API Version: v20.0  
    - Use this to retrieve Instagram Business Account ID for setup  

13. **Add Sticky Notes**  
    - Add notes at relevant points to explain parameters, permissions, limitations, and email use.  
    - Include credential reminders and API permission explanations.  

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| Workflow video in Italian with English subtitles: https://youtu.be/obWJFJvg_6g | Setup and usage guidance video for this workflow |
| Required Facebook API permissions: `ads_management`, `business_management`, `instagram_basic`, `instagram_content_publish`, `pages_read_engagement`. App review needed for external users. | Facebook Graph API permissions for Instagram publishing |
| Limitations: Only JPEG images supported. No shopping tags, branded content tags, filters, or Instagram TV publishing. Rate limit: max 50 posts per 24 hours. | API limitations and usage constraints |
| To check Instagram API usage limits: GET /{ig-user-id}/content_publishing_limit | Facebook Graph API endpoint for rate limit status |
| Customize trigger node as needed (manual, webhook, or scheduled) | Flexible workflow trigger options |
| Email nodes require configured SMTP or email credentials | Email notification setup requirement |
| Change default Facebook Graph API URL to `graph.instagram.com` if using Instagram API directly | Alternative API endpoint note |
| Ensure credentials are properly configured in n8n for Facebook Graph API and email nodes | Credential management best practice |

---

This structured documentation provides complete understanding of the workflow's components, stepwise reproduction instructions, and key considerations to enable efficient usage and maintenance.