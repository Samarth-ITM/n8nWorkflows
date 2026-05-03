Instagram Auto-Publisher for Lightroom Photos (AI Captions)

https://n8nworkflows.xyz/workflows/instagram-auto-publisher-for-lightroom-photos--ai-captions--9694


# Instagram Auto-Publisher for Lightroom Photos (AI Captions)

### 1. Workflow Overview

This n8n workflow automates the process of publishing photos from Adobe Lightroom to Instagram with AI-generated captions. It is designed to:

- Select the next unposted photo from a data table of Lightroom photos.
- Generate an Instagram caption using ALT text and EXIF metadata processed by an AI model.
- Publish the photo via the Instagram Graph API.
- Update the data table to mark the photo as posted.

The workflow is logically divided into five main blocks:

1.1 Input Reception & Selection  
1.2 AI Caption Generation  
1.3 Instagram Authentication  
1.4 Image Publishing  
1.5 Data Table Update

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Selection

**Overview:**  
This block fetches the next Lightroom photo that has not yet been posted on Instagram. It filters the photos data table to retrieve only unposted photos, sorts them by creation date, and limits the selection to one photo for processing.

**Nodes Involved:**  
- Schedule Trigger  
- Get row(s)  
- Sort  
- Limit  
- Sticky Note1

**Node Details:**

- **Schedule Trigger**  
  - Type: Trigger  
  - Role: Initiates the workflow on a defined schedule (specific hours and minutes).  
  - Configuration: Triggers at 11:40, 14:10, 18:10, and 20:10 hours.  
  - Inputs: None (trigger)  
  - Outputs: Connects to "Get row(s)"  
  - Edge cases: Trigger timing misconfiguration might lead to missed runs.

- **Get row(s)**  
  - Type: Data Table  
  - Role: Fetches rows where `ig_posted_at` is empty, i.e., photos not published.  
  - Configuration: Filter condition on `ig_posted_at` being empty, returns all matching rows. Uses a specific data table ID for Lightroom photos.  
  - Inputs: From Schedule Trigger  
  - Outputs: Connects to "Sort"  
  - Edge cases: Empty data table or no unposted rows results in no further processing.

- **Sort**  
  - Type: Sort  
  - Role: Sorts photos by their creation date to pick the oldest/unposted first.  
  - Configuration: Sorted ascending by `createdAt`.  
  - Inputs: From Get row(s)  
  - Outputs: Connects to "Limit"  
  - Edge cases: Missing or malformed creation dates could affect sorting.

- **Limit**  
  - Type: Limit  
  - Role: Restricts the flow to a single photo (or a set limit) to prevent bulk posting.  
  - Configuration: Default limit of 1 (can be adjusted).  
  - Inputs: From Sort  
  - Outputs: Connects to "Message a model" (next block)  
  - Edge cases: Limit set to zero or very high numbers might cause no action or overload.

- **Sticky Note1**  
  - Type: Sticky Note  
  - Role: Documentation indicating this is Step 1 - select the next image to publish.

---

#### 2.2 AI Caption Generation

**Overview:**  
Generates an Instagram caption for the selected photo by using ALT text and EXIF metadata as input to an AI language model (Anthropic Claude).

**Nodes Involved:**  
- Message a model  
- Params  
- Sticky Note2

**Node Details:**

- **Message a model**  
  - Type: AI (Anthropic via LangChain)  
  - Role: Generates a 3-line Instagram caption following strict formatting and SEO rules based on photo metadata.  
  - Configuration: Uses the "claude-sonnet-4-5-20250929" model. Input message includes detailed instructions and extracts ALT text and EXIF from the selected photo.  
  - Expressions:  
    - ALT text: `{{ $('Limit').item.json.alt }}`  
    - EXIF data parsed from `lr_asset` JSON payload.  
  - Inputs: From Limit  
  - Outputs: Connects to Params  
  - Edge cases:  
    - AI service downtime or rate limits.  
    - Malformed JSON in EXIF data causing parsing errors.  
    - Output exceeding 220 characters or formatting errors.

- **Params**  
  - Type: Set  
  - Role: Holds fixed parameters such as Instagram Business Account ID, Facebook Page ID, n8n instance domain, Lightroom catalog ID. These are reused in subsequent HTTP requests.  
  - Configuration: Four string fields configured with user-specific values.  
  - Inputs: From Message a model  
  - Outputs: Connects to get access_token  
  - Edge cases: Missing or incorrect parameter values will cause API failures downstream.

- **Sticky Note2**  
  - Type: Sticky Note  
  - Role: Documentation indicating Step 2 - Generate the description with AI.

---

#### 2.3 Instagram Authentication

**Overview:**  
Fetches the Facebook long-lived access token and retrieves the Instagram Business Account ID required for media posting.

**Nodes Involved:**  
- get access_token  
- Get instagram id  
- Sticky Note3

**Node Details:**

- **get access_token**  
  - Type: HTTP Request  
  - Role: Retrieves a valid Facebook access token for the Facebook Page using the configured Facebook ID.  
  - Configuration: GET request to `https://graph.facebook.com/v23.0/{FB id}?fields=access_token` with HTTP Bearer authentication.  
  - Inputs: From Params  
  - Outputs: Connects to Get instagram id  
  - Edge cases:  
    - Expired or invalid token causing authentication failure.  
    - Network issues or API rate limits.

- **Get instagram id**  
  - Type: HTTP Request  
  - Role: Retrieves Instagram Business Account ID linked to the Facebook Page.  
  - Configuration: GET request to `https://graph.facebook.com/v23.0/{FB id}?fields=instagram_business_account{id,username}` with authentication.  
  - Inputs: From get access_token  
  - Outputs: Connects to Create container  
  - Edge cases:  
    - Missing Instagram Business Account linkage.  
    - Permission errors or invalid tokens.

- **Sticky Note3**  
  - Type: Sticky Note  
  - Role: Documentation indicating Step 3 - Instagram authentication.

---

#### 2.4 Image Publishing

**Overview:**  
Creates a media container on Instagram with the photo URL and generated caption, then publishes the media.

**Nodes Involved:**  
- Create container  
- Publish image  
- Sticky Note4

**Node Details:**

- **Create container**  
  - Type: HTTP Request  
  - Role: Creates an Instagram media container with the image URL, caption, and alt text.  
  - Configuration: POST request to `https://graph.facebook.com/v23.0/{instagram_business_account_id}/media` with access token. Body includes:  
    - `image_url`: Constructed from n8n domain webhook URL with Lightroom catalog and asset IDs as query parameters.  
    - `caption`: AI-generated caption from "Message a model".  
    - `alt_text`: ALT text from photo metadata.  
  - Inputs: From Get instagram id  
  - Outputs: Connects to Publish image  
  - Edge cases:  
    - Invalid image URL or inaccessible image causing container creation failure.  
    - Exceeding Instagram API limits or errors in caption formatting.

- **Publish image**  
  - Type: HTTP Request  
  - Role: Publishes the media container on Instagram.  
  - Configuration: POST request to `https://graph.facebook.com/v23.0/{instagram_business_account_id}/media_publish` with `creation_id` referring to the container created. Uses Bearer authentication.  
  - Inputs: From Create container  
  - Outputs: Connects to Update row(s)  
  - Edge cases:  
    - Publish failures due to API errors.  
    - Rate limits or token expiry.

- **Sticky Note4**  
  - Type: Sticky Note  
  - Role: Documentation indicating Step 4 - Publish the image.

---

#### 2.5 Data Table Update

**Overview:**  
Updates the Lightroom photos data table to mark the photo as posted, saving Instagram post ID, caption, and timestamp.

**Nodes Involved:**  
- Update row(s)  
- Sticky Note5

**Node Details:**

- **Update row(s)**  
  - Type: Data Table  
  - Role: Marks the photo as posted by updating `ig_posted_at` with current timestamp, `ig_id` with Instagram post ID, and `ig_caption` with generated caption.  
  - Configuration: Updates the row matching the current photo's ID.  
  - Inputs: From Publish image  
  - Outputs: None (end of workflow)  
  - Edge cases:  
    - Failure to update data table leads to inconsistent state.  
    - Concurrent updates could produce race conditions.

- **Sticky Note5**  
  - Type: Sticky Note  
  - Role: Documentation indicating Step 5 - Update Data Table.

---

### 3. Summary Table

| Node Name        | Node Type                     | Functional Role                      | Input Node(s)          | Output Node(s)       | Sticky Note                                                                                          |
|------------------|-------------------------------|------------------------------------|-----------------------|---------------------|----------------------------------------------------------------------------------------------------|
| Schedule Trigger | Schedule Trigger              | Initiates workflow on schedule     | -                     | Get row(s)           |                                                                                                    |
| Get row(s)       | Data Table                    | Fetch unposted Lightroom photos    | Schedule Trigger       | Sort                 |                                                                                                    |
| Sort             | Sort                         | Sort photos by creation date       | Get row(s)             | Limit                |                                                                                                    |
| Limit            | Limit                        | Limit to next photo to process      | Sort                   | Message a model      |                                                                                                    |
| Message a model  | Anthropic AI (LangChain)     | Generate Instagram caption          | Limit                  | Params               |                                                                                                    |
| Params           | Set                          | Store fixed parameters              | Message a model        | get access_token      |                                                                                                    |
| get access_token | HTTP Request                 | Get Facebook access token           | Params                  | Get instagram id     |                                                                                                    |
| Get instagram id | HTTP Request                 | Retrieve Instagram Business Account | get access_token        | Create container     |                                                                                                    |
| Create container | HTTP Request                 | Create Instagram media container    | Get instagram id       | Publish image        |                                                                                                    |
| Publish image    | HTTP Request                 | Publish media on Instagram          | Create container       | Update row(s)        |                                                                                                    |
| Update row(s)    | Data Table                   | Mark photo as posted in data table  | Publish image          | -                    |                                                                                                    |
| Sticky Note      | Sticky Note                  | Workflow overview and instructions  | -                      | -                    | ### Instagram Auto-Publisher (Lightroom → IG) Purpose, setup requirements and notes                |
| Sticky Note1     | Sticky Note                  | Step 1 - Select image to publish    | -                      | -                    | ## STEP1 - Select the next image to publish                                                       |
| Sticky Note2     | Sticky Note                  | Step 2 - Generate AI caption        | -                      | -                    | ## STEP2 - Generate the description with AI                                                      |
| Sticky Note3     | Sticky Note                  | Step 3 - Instagram authentication   | -                      | -                    | ## STEP3 - Instagram auth                                                                        |
| Sticky Note4     | Sticky Note                  | Step 4 - Publish the image           | -                      | -                    | ## STEP4 - Publish the image                                                                    |
| Sticky Note5     | Sticky Note                  | Step 5 - Update Data Table           | -                      | -                    | ## STEP5 - Update Data Table                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger**  
   - Node Type: Schedule Trigger  
   - Parameters: Set trigger times at 11:40, 14:10, 18:10, and 20:10.  
   - No credentials needed.

2. **Create Data Table Node "Get row(s)"**  
   - Node Type: Data Table (operation: get)  
   - Filter: `ig_posted_at` is empty (to find unposted photos).  
   - Data Table ID: Your Lightroom photos data table ID.  
   - Connect Schedule Trigger → Get row(s).

3. **Create Sort Node**  
   - Node Type: Sort  
   - Sort by field: `createdAt` ascending.  
   - Connect Get row(s) → Sort.

4. **Create Limit Node**  
   - Node Type: Limit  
   - Limit: 1 (or desired max posts per run).  
   - Connect Sort → Limit.

5. **Create AI Model Node "Message a model"**  
   - Type: Anthropic AI via LangChain (model: "claude-sonnet-4-5-20250929").  
   - Credentials: Anthropic API key.  
   - Input message: Include instructions for caption generation using expressions:  
     - ALT: `{{$node["Limit"].json["alt"]}}`  
     - EXIF: parse JSON from `lr_asset` field.  
   - Connect Limit → Message a model.

6. **Create Set Node "Params"**  
   - Node Type: Set  
   - Fields to configure:  
     - instagram_business_account_id (string)  
     - FB id (Facebook Page id, string)  
     - n8n instance domain (string)  
     - LR catalog ID (string)  
   - Connect Message a model → Params.

7. **Create HTTP Request Node "get access_token"**  
   - Method: GET  
   - URL: `https://graph.facebook.com/v23.0/{{ $json["FB id"] }}?fields=access_token`  
   - Authentication: HTTP Bearer with valid Facebook token credential.  
   - Connect Params → get access_token.

8. **Create HTTP Request Node "Get instagram id"**  
   - Method: GET  
   - URL: `https://graph.facebook.com/v23.0/{{ $json["FB id"] }}?fields=instagram_business_account{id,username}`  
   - Authentication: HTTP Bearer with Facebook token.  
   - Connect get access_token → Get instagram id.

9. **Create HTTP Request Node "Create container"**  
   - Method: POST  
   - URL: `https://graph.facebook.com/v23.0/{{ $json.instagram_business_account.id }}/media`  
   - Authentication: HTTP Bearer with Facebook token.  
   - Body parameters (JSON):  
     - `image_url`: Construct using n8n domain + webhook endpoint + Lightroom catalog and asset IDs from Limit node.  
     - `caption`: Output text from Message a model.  
     - `alt_text`: ALT text from Limit node.  
   - Connect Get instagram id → Create container.

10. **Create HTTP Request Node "Publish image"**  
    - Method: POST  
    - URL: `https://graph.facebook.com/v23.0/{{ $json.instagram_business_account.id }}/media_publish`  
    - Authentication: HTTP Bearer with Facebook token.  
    - Body: `creation_id` from Create container response.  
    - Enable retry on fail with delay (e.g., 3 seconds).  
    - Connect Create container → Publish image.

11. **Create Data Table Node "Update row(s)"**  
    - Operation: Update  
    - Filter to row with photo ID matching Limit node's current photo.  
    - Columns to update:  
      - `ig_id`: post id from Publish image response.  
      - `ig_caption`: caption text from Message a model.  
      - `ig_posted_at`: current timestamp (`{{$now}}`).  
    - Connect Publish image → Update row(s).

12. **Add Sticky Notes** (optional but recommended for clarity)  
    - Place Sticky Notes describing each step as per the original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------|
| Workflow requires Instagram Business or Creator Account linked to a Facebook Page with Graph API v23.0 long-lived token. | Instagram and Facebook API documentation for token management and permissions.                                  |
| Publicly reachable n8n instance domain is mandatory for Lightroom image webhook URL, enabling image access.             | Ensure n8n instance has a static domain or use tunneling services if needed.                                    |
| Lightroom catalog and asset IDs must be available and accurate in the Data Table for image retrieval and metadata.    | Lightroom API or export tools to maintain Data Table with accurate metadata.                                   |
| AI-generated captions follow strict no-inference and SEO-focused rules to maintain authenticity and optimize engagement. | Anthropic Claude model usage via LangChain integration in n8n.                                                 |
| Instagram API rate limits and error handling should be monitored to avoid posting failures or token expiry issues.    | Facebook Developer Portal and Instagram Graph API rate limits documentation.                                   |
| Scheduling times can be customized to preferred posting windows to maximize audience engagement.                      | Adjust Schedule Trigger node parameters accordingly.                                                           |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow built with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.