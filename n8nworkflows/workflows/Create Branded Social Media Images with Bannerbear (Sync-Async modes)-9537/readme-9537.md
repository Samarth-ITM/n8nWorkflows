Create Branded Social Media Images with Bannerbear (Sync/Async modes)

https://n8nworkflows.xyz/workflows/create-branded-social-media-images-with-bannerbear--sync-async-modes--9537


# Create Branded Social Media Images with Bannerbear (Sync/Async modes)

### 1. Workflow Overview

This workflow automates the creation of branded social media images using Bannerbear's API, supporting both synchronous and asynchronous operation modes. It is designed for marketers, social media managers, or developers who want to generate customized images dynamically with Bannerbear templates, choosing between immediate synchronous image generation or deferred asynchronous generation via webhook callbacks.

The workflow is logically divided into the following blocks:

- **1.1 Input Setup & Mode Selection:** Manual trigger and parameter setting including API key, template ID, text content, and call mode (sync or async).
- **1.2 Synchronous Image Creation:** If sync mode is selected, directly request image generation from Bannerbear's sync API and extract image URLs.
- **1.3 Asynchronous Image Creation:** If async mode is selected, request image generation via Bannerbear's async API and handle webhook callbacks for image completion.
- **1.4 Webhook Listener for Async Mode:** Receives the webhook event from Bannerbear when image creation is completed and extracts the image details.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Setup & Mode Selection

- **Overview:**  
  This block initializes the workflow with user-defined parameters and decides which image creation mode (sync or async) to proceed with.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - SetParameters (Set)  
  - IfSynchrounousCall (If)

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow manually. No parameters needed.  
    - Inputs: None  
    - Outputs: Connects to SetParameters  
    - Failure modes: None (user-initiated)

  - **SetParameters**  
    - Type: Set  
    - Role: Defines key parameters for Bannerbear API, template, texts, and call mode.  
    - Configuration:  
      - `bannerbear_api_key`: Placeholder for user's Bannerbear API key  
      - `banner_bear_template_id`: Placeholder for Bannerbear template ID  
      - `title`: Text for main title on the image  
      - `subtitle`: Text for subtitle/pretitle on the image  
      - `call_mode`: String, either `"sync"` or `"async"`  
    - Inputs: Manual Trigger output  
    - Outputs: Connects to IfSynchrounousCall  
    - Edge cases: User must replace placeholders with valid keys and IDs; missing or invalid keys cause auth failures.

  - **IfSynchrounousCall**  
    - Type: If (conditional branch)  
    - Role: Routes workflow based on `call_mode` parameter.  
    - Configuration: Checks if `call_mode` equals `"sync"`  
    - Inputs: SetParameters output  
    - Outputs:  
      - True branch: To synchronous image creation  
      - False branch: To asynchronous image creation  
    - Edge cases: Case sensitivity and type validation are loose; invalid or missing `call_mode` leads to default async branch.

---

#### 2.2 Synchronous Image Creation

- **Overview:**  
  Directly calls Bannerbear’s synchronous API endpoint to generate an image immediately and then extracts relevant image URLs and metadata.

- **Nodes Involved:**  
  - SynchronouslyCreateImage (HTTP Request)  
  - GetImageUrlAndSize (Set)  
  - Sticky Note2 (documentation)

- **Node Details:**

  - **SynchronouslyCreateImage**  
    - Type: HTTP Request  
    - Role: Makes a POST request to Bannerbear's synchronous API endpoint `https://sync.api.bannerbear.com/v2/images`.  
    - Configuration:  
      - Body JSON includes:  
        - `template`: Template ID from SetParameters  
        - `modifications`: Title and pretitle text from SetParameters  
        - `webhook_url`: Null (no webhook for sync mode)  
      - Headers: Authorization Bearer token from SetParameters  
      - Retries on failure with 5-second intervals  
    - Inputs: IfSynchrounousCall True output  
    - Outputs: Connects to GetImageUrlAndSize  
    - Edge cases: Auth failures, API rate limits, network timeouts, invalid template IDs.

  - **GetImageUrlAndSize**  
    - Type: Set  
    - Role: Extracts and formats important fields from Bannerbear's sync API response: UID, status, PNG/JPG URLs, width, height.  
    - Configuration: Uses expressions to map fields from the HTTP response JSON.  
    - Inputs: Output from SynchronouslyCreateImage  
    - Outputs: None (end of sync branch)  
    - Edge cases: Missing fields in response cause expression errors.

  - **Sticky Note2**  
    - Type: Sticky Note  
    - Content: "## Synchronous call (specific API URL)"  
    - Purpose: Documentation for user clarity on sync call method.

---

#### 2.3 Asynchronous Image Creation

- **Overview:**  
  Initiates image creation via Bannerbear’s asynchronous API through an n8n Bannerbear node. The result is delivered later via a webhook event.

- **Nodes Involved:**  
  - AsynchronouslyCreateImage (Bannerbear Node)  
  - GetUidAndStatus (Set)  
  - Sticky Note3 (documentation)

- **Node Details:**

  - **AsynchronouslyCreateImage**  
    - Type: Bannerbear (native node)  
    - Role: Sends an async image generation request to Bannerbear API using provided template and modifications.  
    - Configuration:  
      - Template ID and modifications (title and subtitle) sourced from SetParameters JSON via expressions.  
      - Credentials: Uses stored Bannerbear API key node credential.  
      - No webhook URL set here; webhook is configured separately.  
    - Inputs: IfSynchrounousCall False output  
    - Outputs: Connects to GetUidAndStatus  
    - Edge cases: Credential misconfiguration, API key invalid, network issues.

  - **GetUidAndStatus**  
    - Type: Set  
    - Role: Extracts UID and status fields from the async response, which is initial creation confirmation data.  
    - Configuration: Maps JSON fields using expressions.  
    - Inputs: Output from AsynchronouslyCreateImage  
    - Outputs: None (intermediate step)  
    - Edge cases: Missing or malformed API response.

  - **Sticky Note3**  
    - Type: Sticky Note  
    - Content: "## Asynchronous call"  
    - Purpose: User documentation for async call mode.

---

#### 2.4 Webhook Listener for Async Mode

- **Overview:**  
  Listens for Bannerbear webhook POST requests signaling that an asynchronous image creation has completed, then extracts image information.

- **Nodes Involved:**  
  - Webhook_OnImageCreated (Webhook)  
  - GetCompletedImageInfo (Set)  
  - Sticky Note4 (documentation)

- **Node Details:**

  - **Webhook_OnImageCreated**  
    - Type: Webhook  
    - Role: Receives POST requests from Bannerbear's `image_created` webhook event.  
    - Configuration:  
      - Path: Unique webhook path (`ac6a6723-1876-4f43-af4c-411a0f1f4ad3`)  
      - HTTP Method: POST  
      - Active only if workflow is active and webhook URL is registered in Bannerbear settings  
    - Inputs: External HTTP requests from Bannerbear  
    - Outputs: Connects to GetCompletedImageInfo  
    - Edge cases: Webhook not registered properly in Bannerbear, inactive workflow, incorrect URL, HTTP method mismatch.

  - **GetCompletedImageInfo**  
    - Type: Set  
    - Role: Parses webhook payload to extract UID, status, image URLs (PNG/JPG), and dimensions.  
    - Configuration: JSON mapping expressions from webhook body.  
    - Inputs: Webhook_OnImageCreated output  
    - Outputs: None (end of async branch)  
    - Edge cases: Missing or malformed webhook data.

  - **Sticky Note4**  
    - Type: Sticky Note  
    - Content:  
      ```
      ## Webhook Async mode
      In order to declare the webhook in BannerBear:
      - The webhook must be of type POST
      - You must use the production URL
      - Workflow must be active
      ```
    - Purpose: Critical configuration instructions to ensure webhook triggers correctly.

---

### 3. Summary Table

| Node Name                 | Node Type          | Functional Role                                 | Input Node(s)                 | Output Node(s)              | Sticky Note                                                                                         |
|---------------------------|--------------------|------------------------------------------------|------------------------------|-----------------------------|---------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger     | Starts workflow manually                        | None                         | SetParameters               |                                                                                                   |
| SetParameters             | Set                | Defines Bannerbear API keys, template ID, texts, call mode | When clicking ‘Execute workflow’ | IfSynchrounousCall          |                                                                                                   |
| IfSynchrounousCall        | If                 | Routes workflow to sync or async image creation | SetParameters                | SynchronouslyCreateImage, AsynchronouslyCreateImage |                                                                                                   |
| SynchronouslyCreateImage  | HTTP Request       | Calls Bannerbear sync API to create image       | IfSynchrounousCall (true)    | GetImageUrlAndSize          |                                                                                                   |
| GetImageUrlAndSize        | Set                | Extracts image URLs and metadata from sync response | SynchronouslyCreateImage     | None                        |                                                                                                   |
| AsynchronouslyCreateImage | Bannerbear         | Calls Bannerbear async API to create image      | IfSynchrounousCall (false)   | GetUidAndStatus             |                                                                                                   |
| GetUidAndStatus           | Set                | Extracts UID and status from async creation response | AsynchronouslyCreateImage    | None                        |                                                                                                   |
| Webhook_OnImageCreated    | Webhook            | Receives Bannerbear webhook on image creation   | External HTTP POST            | GetCompletedImageInfo       |                                                                                                   |
| GetCompletedImageInfo     | Set                | Extracts image info from webhook payload         | Webhook_OnImageCreated       | None                        |                                                                                                   |
| Sticky Note               | Sticky Note        | Setup instructions and workflow overview         | None                         | None                        | "## How to set up this workflow..." with detailed Bannerbear API key and webhook setup instructions |
| Sticky Note2              | Sticky Note        | Describes synchronous call method                 | None                         | None                        | "## Synchronous call (specific API URL)"                                                          |
| Sticky Note3              | Sticky Note        | Describes asynchronous call method                | None                         | None                        | "## Asynchronous call"                                                                             |
| Sticky Note4              | Sticky Note        | Describes webhook async mode setup                | None                         | None                        | "## Webhook Async mode" with instructions on webhook POST type and URL                           |
| Sticky Note12             | Sticky Note        | Notes on choosing sync or async call               | None                         | None                        | "## Choose between synchronous and asynchronous call"                                            |
| Sticky Note6              | Sticky Note        | Notes on setting Bannerbear parameters and data   | None                         | None                        | "## Set BannerBear parameters & data"                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Name: "When clicking ‘Execute workflow’"  
   - No parameters needed.

2. **Create Set Node for Parameters**  
   - Type: Set  
   - Name: "SetParameters"  
   - Mode: Raw JSON  
   - JSON content example:  
     ```json
     {
       "bannerbear_api_key": "your_bannerbear_api_key",
       "banner_bear_template_id": "your_banner_bear_template_id",
       "title": "This image is AI generated",
       "subtitle": "Made in n8n with BannerBear",
       "call_mode": "sync"
     }
     ```  
   - Connect Manual Trigger output to this node.

3. **Create If Node to Choose Call Mode**  
   - Type: If  
   - Name: "IfSynchrounousCall"  
   - Condition: Check if expression `{{$json.call_mode}}` equals `"sync"` (case-sensitive)  
   - Connect SetParameters output to this node.

4. **Create HTTP Request Node for Sync Call**  
   - Type: HTTP Request  
   - Name: "SynchronouslyCreateImage"  
   - Method: POST  
   - URL: `https://sync.api.bannerbear.com/v2/images`  
   - Headers:  
     - `Authorization`: `Bearer {{$json.bannerbear_api_key}}`  
   - Body (JSON):  
     ```json
     {
       "template": "{{$json.banner_bear_template_id}}",
       "modifications": [
         {
           "name": "title",
           "text": "{{$json.title}}",
           "color": null,
           "background": null
         },
         {
           "name": "pretitle",
           "text": "{{$json.subtitle}}",
           "background": null
         }
       ],
       "webhook_url": null,
       "transparent": false,
       "metadata": null
     }
     ```  
   - Enable "Send Body" as JSON  
   - Retry on failure: enabled, with 5 seconds between tries  
   - Connect If node True output to this node.

5. **Create Set Node to Extract Image Info (Sync)**  
   - Type: Set  
   - Name: "GetImageUrlAndSize"  
   - Mode: Raw JSON  
   - Content:  
     ```json
     {
       "uid": "{{$json.uid}}",
       "status": "{{$json.status}}",
       "image_url_png": "{{$json.image_url_png}}",
       "image_url_jpg": "{{$json.image_url_jpg}}",
       "width": {{$json.width}},
       "height": {{$json.height}}
     }
     ```  
   - Connect HTTP Request output to this node.

6. **Create Bannerbear Node for Async Call**  
   - Type: Bannerbear  
   - Name: "AsynchronouslyCreateImage"  
   - Template ID: `{{$json.banner_bear_template_id}}` (expression)  
   - Modifications:  
     - `title`: `{{$json.title}}`  
     - `pretitle`: `{{$json.subtitle}}`  
   - Credentials: Add Bannerbear API credentials with your API key  
   - Connect If node False output to this node.

7. **Create Set Node to Extract UID and Status (Async)**  
   - Type: Set  
   - Name: "GetUidAndStatus"  
   - Mode: Raw JSON  
   - Content:  
     ```json
     {
       "uid": "{{$json.uid}}",
       "status": "{{$json.status}}"
     }
     ```  
   - Connect Bannerbear node output to this node.

8. **Create Webhook Node for Async Callback**  
   - Type: Webhook  
   - Name: "Webhook_OnImageCreated"  
   - HTTP Method: POST  
   - Path: Generate a unique webhook path (e.g., UUID or descriptive string)  
   - Save and activate workflow for the webhook to work.

9. **Register Webhook URL in Bannerbear**  
   - Go to Bannerbear settings > Webhooks  
   - Create webhook with:  
     - URL: Your n8n webhook URL (production URL from Webhook node)  
     - Event: `image_created`  
     - Method: POST

10. **Create Set Node to Extract Completed Image Info from Webhook**  
    - Type: Set  
    - Name: "GetCompletedImageInfo"  
    - Mode: Raw JSON  
    - Content:  
      ```json
      {
        "uid": "{{$json.uid}}",
        "status": "{{$json.status}}",
        "image_url_png": "{{$json.image_url_png}}",
        "image_url_jpg": "{{$json.image_url_jpg}}",
        "width": {{$json.width}},
        "height": {{$json.height}}
      }
      ```  
    - Connect Webhook output to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                        | Context or Link                                                                                                                                                                                                                                              |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| How to set up this workflow: 1. Get Bannerbear credentials by signing up at bannerbear.com, creating a project and template, copying API key and template ID; 2. Configure SetParameters node with your credentials and texts; 3. For async mode, activate webhook node, copy production URL, and register webhook in Bannerbear with event `image_created`. Alternative webhook setup via POSTMAN included. | Detailed setup instructions are included in the first sticky note node within the workflow.                                                                                                                             |
| For asynchronous mode, ensure the webhook uses HTTP POST, the workflow is active, and the production webhook URL is registered in Bannerbear settings for image creation to trigger the webhook correctly.                                                                                                                                                                       | Sticky Note4 content: critical for webhook async mode configuration.                                                                                                                                                 |
| Bannerbear API documentation: https://www.bannerbear.com/docs/api/                                                                                                                                                                                                                                                                                                                  | Official API docs for details on template modifications, webhooks, and API usage.                                                                                                                                      |
| Credentials setup in n8n requires creating Bannerbear API key credential with your API key for the Bannerbear node to authenticate properly.                                                                                                                                                                                                                                      | Credential management in n8n under Credentials section.                                                                                                                                                               |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly follows current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.