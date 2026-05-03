Schedule Social Media Posts from Google Sheets to Twitter & Instagram

https://n8nworkflows.xyz/workflows/schedule-social-media-posts-from-google-sheets-to-twitter---instagram-6342


# Schedule Social Media Posts from Google Sheets to Twitter & Instagram

### 1. Workflow Overview

This workflow automates social media posting by scheduling posts from a Google Sheet to Twitter (X) and Instagram. It targets social media managers and content teams who want to streamline publishing content stored in Google Sheets with minimal manual intervention.

The workflow is logically divided into the following blocks:

- **1.1 Scheduling and Input Fetching:** Periodically triggers the workflow and fetches the first “Pending” post row from Google Sheets.
- **1.2 Instagram Caption and Image Generation:** Prepares Instagram post content by combining text fields and generating an HTML-based image using an external API.
- **1.3 Social Media Posting:** Posts the content as a tweet on Twitter and as an image + caption on Instagram via Facebook Graph API.
- **1.4 Post-Publish Update:** Updates the Google Sheet row status to prevent reposting.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduling and Input Fetching

**Overview:**  
This block triggers the workflow every 4 hours and fetches the next post to publish by reading the first row marked as "Pending" in the Google Sheet. It acts as the entry point and input provider for subsequent operations.

**Nodes Involved:**  
- Schedule Trigger  
- Get row(s) in sheet

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Periodically starts the workflow every 4 hours.  
  - Configuration: Interval set to 4 hours.  
  - Input/Output: No input; output connects to "Get row(s) in sheet".  
  - Edge Cases: Workflow won't run if n8n instance is offline; misconfigured interval can cause unintended frequency.

- **Get row(s) in sheet**  
  - Type: Google Sheets Node  
  - Role: Retrieves the first row where the "Status" column equals "Pending".  
  - Configuration: Uses filter on "Status" column, returns only first match, targets a specific Google Sheet and Sheet1 tab by gid=0.  
  - Credentials: Google Sheets OAuth2 account linked.  
  - Input/Output: Input from Schedule Trigger; outputs post data to "Post on Twitter" and "Insta post caption".  
  - Edge Cases: Errors if sheet ID is wrong or credentials expire; no rows with "Pending" yield no output; rate limits on Google Sheets API.

---

#### 2.2 Instagram Caption and Image Generation

**Overview:**  
Prepares the Instagram caption by concatenating multiple fields and creates an HTML layout for the post image. Then, it sends this HTML to the HCTI.io API to generate a 1080x1080 image suitable for Instagram.

**Nodes Involved:**  
- Insta post caption  
- HCTI Image

**Node Details:**

- **Insta post caption**  
  - Type: Set  
  - Role: Combines "Caption", "Desc", and "Hashtags" fields into a single `final_caption`. Also generates an HTML string describing the post's visual layout.  
  - Configuration: Uses expressions to concatenate fields and embed them in styled HTML.  
  - Input/Output: Input from "Get row(s) in sheet"; output connects to "HCTI Image".  
  - Edge Cases: Missing fields may cause incomplete captions; malformed HTML if unexpected characters exist.

- **HCTI Image**  
  - Type: HTTP Request  
  - Role: Sends the HTML to hcti.io API to convert it into an image.  
  - Configuration: POST request with form-urlencoded body including HTML, viewport width/height (1080x1080), authenticated via HTTP Basic with stored credential.  
  - Input/Output: Input from "Insta post caption"; output connects to "Create Insta post".  
  - Credentials: HTTP Basic Auth credential named "Unnamed credential 2".  
  - Edge Cases: API limits or failures; invalid HTML may produce errors; authentication failures; network timeouts.

---

#### 2.3 Social Media Posting

**Overview:**  
Posts the social media content. It posts the text as a tweet on Twitter, then uploads the generated image and caption to Instagram via the Facebook Graph API in two steps: media container creation and publishing.

**Nodes Involved:**  
- Post on Twitter  
- Create Insta post  
- Post On Instagram

**Node Details:**

- **Post on Twitter**  
  - Type: HTTP Request  
  - Role: Posts the caption text as a tweet on Twitter (X).  
  - Configuration: POST to Twitter API v2 endpoint `/2/tweets` with OAuth1 authentication, body contains text from the "Caption" field.  
  - Input/Output: Input from "Get row(s) in sheet"; no explicit output connections (ends branch).  
  - Credentials: OAuth1 API credentials for Twitter.  
  - Edge Cases: Twitter API rate limits; invalid OAuth credentials; text length restrictions.

- **Create Insta post**  
  - Type: Facebook Graph API  
  - Role: Uploads the image URL from HCTI response and caption to Instagram Content Publishing API, creating a media container.  
  - Configuration: POST to `/{InstagramAccountID}/media` endpoint, includes image URL, caption, and Facebook access token.  
  - Input/Output: Input from "HCTI Image"; output connects to "Post On Instagram".  
  - Credentials: Facebook Graph API token.  
  - Edge Cases: Token expiration; incorrect Instagram account ID; image URL not accessible; API errors.

- **Post On Instagram**  
  - Type: Facebook Graph API  
  - Role: Publishes the media container created in the previous step to the Instagram feed.  
  - Configuration: POST to `/{InstagramAccountID}/media_publish` endpoint with `creation_id` and access token.  
  - Input/Output: Input from "Create Insta post"; output connects to "Update Status Posted".  
  - Credentials: Facebook Graph API token.  
  - Edge Cases: Token expiration; invalid container ID; API errors.

---

#### 2.4 Post-Publish Update

**Overview:**  
Updates the original Google Sheet row to mark the post as published by modifying the "Status" column to include a timestamp, preventing reposting.

**Nodes Involved:**  
- Update Status Posted

**Node Details:**

- **Update Status Posted**  
  - Type: Google Sheets  
  - Role: Updates the row identified by `RowID` to set "Status" to "Posted on [current date and time]".  
  - Configuration: Uses `RowID` from "Get row(s) in sheet" to match row; updates "Status" with current timestamp.  
  - Credentials: Google Sheets OAuth2 API credential.  
  - Input/Output: Input from "Post On Instagram"; no outputs (workflow end).  
  - Edge Cases: RowID mismatch; credential expiration; API errors.

---

### 3. Summary Table

| Node Name           | Node Type            | Functional Role                                          | Input Node(s)          | Output Node(s)            | Sticky Note                                                                                                     |
|---------------------|----------------------|----------------------------------------------------------|------------------------|---------------------------|-----------------------------------------------------------------------------------------------------------------|
| Schedule Trigger     | Schedule Trigger     | Periodically triggers the workflow every 4 hours         | None                   | Get row(s) in sheet       | This workflow is triggered automatically every 4 hours.                                                        |
| Get row(s) in sheet  | Google Sheets        | Fetches first "Pending" row from Google Sheet             | Schedule Trigger       | Post on Twitter, Insta post caption | ## Google Sheet Fetches the first row from the Google Sheet where the 'Status' column is marked as 'Pending'. This provides the content for the social media posts. |
| Insta post caption   | Set                  | Combines caption, description, hashtags; prepares HTML    | Get row(s) in sheet    | HCTI Image                | ## Create a Post Caption for the Instagram post Prepares the data for the Instagram post. It combines the 'Caption', 'Description', and 'Hashtags' from the Google Sheet into a single `final_caption`. It also generates an HTML structure to be converted into the post image. |
| HCTI Image           | HTTP Request         | Sends HTML to hcti.io API to generate Instagram image     | Insta post caption     | Create Insta post         | ## Create Image using HTMLCSSIMAGE API Sends the generated HTML to the HTML/CSS to Image API (hcti.io) to create a 1080x1080px image for the Instagram post. |
| Post on Twitter      | HTTP Request         | Posts tweet text to Twitter (X)                           | Get row(s) in sheet    | None                      | ## Post a Tweet **Posts the content from the 'Caption' column of the Google Sheet as a new tweet on X (Twitter). |
| Create Insta post    | Facebook Graph API   | Uploads image and caption to Instagram (media container) | HCTI Image             | Post On Instagram         | ## POST on Instagram Step 1/2 for Instagram Posting. This node uploads the image generated in the previous step to the Instagram Content Publishing API. This creates a media container but does not publish it yet. |
| Post On Instagram    | Facebook Graph API   | Publishes Instagram media container to feed               | Create Insta post      | Update Status Posted      | ## POST on Instagram Step 2/2 for Instagram Posting. This node takes the container ID from the previous step ('Create Insta post') and publishes the media to the Instagram feed. |
| Update Status Posted | Google Sheets        | Updates row status in Google Sheet to mark as Posted      | Post On Instagram      | None                      | ## Update the Google sheet Updates the Google Sheet row that was just used. It changes the 'Status' column to 'Posted on [current date and time]' to prevent it from being posted again. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set Interval to every 4 hours  
   - No credentials needed  
   - Connect output to next node.

2. **Create a Google Sheets node named "Get row(s) in sheet"**  
   - Operation: Read rows  
   - Sheet: Set sheet name or gid=0 (Sheet1)  
   - Document ID: Your Google Sheet ID  
   - Filter: Status column equals "Pending"  
   - Option: Return first match only  
   - Credential: Connect your Google Sheets OAuth2 credentials  
   - Connect input from Schedule Trigger  
   - Connect outputs to "Post on Twitter" and "Insta post caption".

3. **Create an HTTP Request node named "Post on Twitter"**  
   - Method: POST  
   - URL: `https://api.x.com/2/tweets`  
   - Body Parameters: `text` = expression `{{$json.Caption}}`  
   - Authentication: OAuth1 configured with your Twitter credentials  
   - Connect input from "Get row(s) in sheet"  
   - Executes once per workflow run.

4. **Create a Set node named "Insta post caption"**  
   - Set two fields:  
     - `final_caption` = `{{$json["Caption"] + "  ..................                     " + "[" + $json["Desc"] + "]" + "  ..................                     " + $json["Hashtags"]}}`  
     - `HTML` = a long HTML string embedding `$json.Caption` for the styled Instagram post (copy from original workflow)  
   - Connect input from "Get row(s) in sheet"  
   - Connect output to "HCTI Image".

5. **Create an HTTP Request node named "HCTI Image"**  
   - Method: POST  
   - URL: `https://hcti.io/v1/image`  
   - Content Type: Form URL Encoded  
   - Body Parameters:  
     - `HTML` = `{{$json.HTML}}`  
     - `viewport_width` = `1080`  
     - `viewport_height` = `1080`  
   - Authentication: HTTP Basic Auth with your HCTI.io username and key  
   - Connect input from "Insta post caption"  
   - Connect output to "Create Insta post".

6. **Create a Facebook Graph API node named "Create Insta post"**  
   - HTTP Request Method: POST  
   - Graph API Version: v19.0  
   - Node: Your Instagram Business Account ID  
   - Edge: `media`  
   - Query Parameters:  
     - `image_url` = `{{$json.url}}` (URL returned by HCTI Image node)  
     - `caption` = `{{$node["Insta post caption"].json.final_caption}}` (from Insta post caption node)  
     - `access_token` = Your Facebook Page/Instagram token  
   - Connect input from "HCTI Image"  
   - Connect output to "Post On Instagram".

7. **Create a Facebook Graph API node named "Post On Instagram"**  
   - HTTP Request Method: POST  
   - Graph API Version: v19.0  
   - Node: Your Instagram Business Account ID  
   - Edge: `media_publish`  
   - Query Parameters:  
     - `creation_id` = `{{$json.id}}` (ID from "Create Insta post" node)  
     - `access_token` = Your Facebook token  
   - Connect input from "Create Insta post"  
   - Connect output to "Update Status Posted".

8. **Create a Google Sheets node named "Update Status Posted"**  
   - Operation: Update  
   - Sheet: same as "Get row(s) in sheet"  
   - Document ID: Your Google Sheet ID  
   - Columns to update:  
     - `RowID` = `{{$node["Get row(s) in sheet"].json.RowID}}` (to identify row)  
     - `Status` = `Posted on [current date and time]` using expression `{{'Posted on ' + $now.toLocaleString()}}`  
   - Credential: Same Google Sheets OAuth2 credential  
   - Connect input from "Post On Instagram".

9. **Add Sticky Notes** (optional) for documentation at various points.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                                    |
|----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| This workflow runs automatically every 4 hours to post scheduled social media content.              | Workflow scheduling detail                                                                                         |
| Google Sheets must have columns: RowID, Caption, Desc, Hashtags, Status; use 'Pending' to queue posts. | Google Sheets data structure                                                                                        |
| HCTI.io API is used to convert HTML into Instagram post image; requires HTTP Basic Auth credentials. | HCTI.io documentation: https://hcti.io/                                                                           |
| Twitter (X) posting uses OAuth1 authentication, posting only text content.                          | Twitter API v2 docs: https://developer.twitter.com/en/docs/twitter-api/tweets/manage-tweets/api-reference/post-tweets |
| Instagram posting requires a business account linked to Facebook Page and a valid Graph API token. | Facebook Graph API docs: https://developers.facebook.com/docs/instagram-api/guides/content-publishing/            |
| Facebook/Instagram tokens expire; consider long-lived tokens or refresh mechanisms.                 | Facebook Developer token management                                                                                 |
| The HTML template in "Insta post caption" node can be customized to change Instagram image style.  | Customization point for branding                                                                                    |
| The Google Sheet update prevents duplicate posting by marking rows as posted with timestamp.        | Ensures idempotency                                                                                                |

---

**Disclaimer:** The content above is generated exclusively from the provided n8n workflow JSON. It complies with content policies and does not contain any illegal or sensitive data.