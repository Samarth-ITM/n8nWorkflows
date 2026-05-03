Generate Instagram Content from Top Trends with AI Image Generation

https://n8nworkflows.xyz/workflows/generate-instagram-content-from-top-trends-with-ai-image-generation-2803


# Generate Instagram Content from Top Trends with AI Image Generation

### 1. Workflow Overview

This workflow automates the generation and publishing of Instagram content inspired by current top trends, leveraging AI for image analysis, caption creation, and image generation. It targets social media managers, content creators, and marketing teams aiming to maintain an active Instagram presence with fresh, AI-generated posts based on trending hashtags.

The workflow is logically divided into three main blocks:

- **1.1 Content Discovery & Filtering**  
  Scrapes trending Instagram posts from specified hashtags, filters out videos and duplicates, and checks the database to avoid reposting existing content.

- **1.2 AI Content Analysis & Generation**  
  Analyzes filtered images using AI to generate descriptive content, creates engaging Instagram captions, and generates new AI images inspired by the analyzed content.

- **1.3 Automated Publishing & Notifications**  
  Publishes the generated content to Instagram Business Account, monitors the publishing status, and sends notifications about success or failure via Telegram.

---

### 2. Block-by-Block Analysis

#### 1.1 Content Discovery & Filtering

**Overview:**  
This block collects trending Instagram posts from two hashtags (#blender3d and #isometric) using the Instagram Scraper API via RapidAPI. It filters out video posts, merges the results, and checks the PostgreSQL database to avoid duplicates.

**Nodes Involved:**  
- Schedule Trigger1  
- Replicate params  
- Instagram params  
- Telegram Params  
- Rapid Api params  
- get top trends on instagram #isometric  
- get  top trends on instagram #blender3d  
- filter the image content  
- filter the image content-2  
- merge the array content  
- Loop Over Items  
- Check Data on Database Is Exist  
- If Data is Exist  
- insert data on db  
- send error message to telegram  

**Node Details:**

- **Schedule Trigger1**  
  - Type: Schedule Trigger  
  - Role: Initiates workflow execution twice daily at 13:05 and 19:05 (server time Europe/Istanbul).  
  - Config: Cron expression "5 13,19 * * *".  
  - Edge cases: Trigger misfire if server time zone changes or cron expression invalid.

- **Replicate params**  
  - Type: Set  
  - Role: Stores Replicate API token for image generation.  
  - Config: Empty string placeholder for token, to be filled by user.  
  - Edge cases: Missing or invalid token will cause downstream HTTP request failures.

- **Instagram params**  
  - Type: Set  
  - Role: Stores Instagram Business Account ID.  
  - Config: Empty string placeholder for account ID.  
  - Edge cases: Incorrect ID will cause API errors during publishing.

- **Telegram Params**  
  - Type: Set  
  - Role: Stores Telegram chat ID for notifications.  
  - Config: Empty string placeholder for chat ID.  
  - Edge cases: Invalid chat ID will cause Telegram notification failures.

- **Rapid Api params**  
  - Type: Set  
  - Role: Stores RapidAPI key for Instagram Scraper API.  
  - Config: Empty string placeholder for key.  
  - Edge cases: Missing or invalid key will cause API request failures.

- **get top trends on instagram #isometric**  
  - Type: HTTP Request  
  - Role: Fetches top posts for hashtag "isometric" from Instagram Scraper API.  
  - Config: Uses RapidAPI host and key headers; query parameters include hashtag and feed_type=top.  
  - Edge cases: API rate limits, network errors, or invalid keys.

- **get  top trends on instagram #blender3d**  
  - Type: HTTP Request  
  - Role: Fetches top posts for hashtag "blender3d" similarly.  
  - Config: Same as above with different hashtag.  
  - Edge cases: Same as above.

- **filter the image content** and **filter the image content-2**  
  - Type: Code  
  - Role: Filters out video posts, maps relevant fields (id, caption text, code, thumbnail URL, tag) into simplified objects.  
  - Config: JavaScript filtering for `!item.is_video`.  
  - Edge cases: Unexpected API response structure or missing fields.

- **merge the array content**  
  - Type: Merge  
  - Role: Combines filtered arrays from both hashtags into one stream.  
  - Config: Default merge mode.  
  - Edge cases: Empty inputs or data type mismatches.

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Processes each trending post individually in batches.  
  - Config: Default batch size (1).  
  - Edge cases: Large data sets may cause performance issues.

- **Check Data on Database Is Exist**  
  - Type: PostgreSQL  
  - Role: Queries `top_trends` table to check if content code already exists.  
  - Config: Select operation with where clause on `code` column.  
  - Edge cases: DB connection errors, query failures.

- **If Data is Exist**  
  - Type: If  
  - Role: Branches workflow based on whether the database query returned results (i.e., content exists).  
  - Config: Checks if query result is not empty.  
  - Edge cases: Expression errors if input data malformed.

- **insert data on db**  
  - Type: PostgreSQL  
  - Role: Inserts new trending post data into `top_trends` table if not already present.  
  - Config: Inserts fields: tag, code, prompt, isposted=false, thumbnail_url.  
  - Edge cases: DB insert errors, constraint violations.

- **send error message to telegram**  
  - Type: Telegram  
  - Role: Sends notification if PostgreSQL query fails.  
  - Config: Uses Telegram chat ID from parameters.  
  - Edge cases: Telegram API errors.

---

#### 1.2 AI Content Analysis & Generation

**Overview:**  
This block analyzes the inserted trending image using OpenAI's GPT-4 Vision to generate a descriptive content summary, then creates an Instagram caption based on that description. Finally, it generates a new AI image inspired by the description using Replicate's Flux model.

**Nodes Involved:**  
- Analyze Image and give the content  
- Analyze Content And Generate Instagram Caption  
- Generate image on flux  

**Node Details:**

- **Analyze Image and give the content**  
  - Type: OpenAI (Langchain) - Image Analysis  
  - Role: Generates a clear, concise description of the object in the image focusing on physical features.  
  - Config: Model "gpt-4o-mini", input is thumbnail URL from current item.  
  - Key expression: Uses thumbnail URL from Loop Over Items node.  
  - Edge cases: Image URL invalid, OpenAI API errors, rate limits.

- **Analyze Content And Generate Instagram Caption**  
  - Type: OpenAI (Langchain) - Chat Completion  
  - Role: Summarizes the image description into a short, engaging Instagram caption with relevant hashtags.  
  - Config: Model "gpt-4o-mini", prompt includes hashtags #Blender3D, #3DArt, #DigitalArt, #3DModeling, #ArtCommunity.  
  - Key expression: Uses content description from previous node.  
  - Edge cases: API errors, prompt formatting issues.

- **Generate image on flux**  
  - Type: HTTP Request  
  - Role: Calls Replicate API to generate a new 3D isometric image based on the analyzed content description.  
  - Config: POST to Replicate Flux model endpoint with detailed prompt including rendering style and technical specs; Authorization header uses Replicate token.  
  - Key expression: Prompt dynamically constructed from analyzed content with whitespace cleanup.  
  - Edge cases: API token invalid, network errors, model errors, timeouts.

---

#### 1.3 Automated Publishing & Notifications

**Overview:**  
This block handles publishing the generated image and caption to Instagram Business Account, monitors the media upload and post status, and sends Telegram notifications about success or failure.

**Nodes Involved:**  
- Prepare data on Instagram  
- Check Status Of Media Before Uploaded  
- If media status is finished  
- Publish Media on Instagram  
- Check status of post   
- If media status is finished1  
- Telegram  
- Telegram1  
- Telegram2  

**Node Details:**

- **Prepare data on Instagram**  
  - Type: Facebook Graph API  
  - Role: Creates Instagram media object with generated image URL and caption.  
  - Config: POST to `/media` edge of Instagram Business Account ID; parameters include `image_url` and `caption`.  
  - Key expression: Uses generated image URL from Replicate and caption from AI node.  
  - Edge cases: API errors, invalid account ID, image URL issues.

- **Check Status Of Media Before Uploaded**  
  - Type: Facebook Graph API  
  - Role: Polls media object status to check if upload is finished.  
  - Config: GET request for media ID fields including status and status_code.  
  - Edge cases: API errors, media ID invalid.

- **If media status is finished**  
  - Type: If  
  - Role: Branches based on whether media upload status_code equals "FINISHED".  
  - Edge cases: Expression errors, unexpected status codes.

- **Publish Media on Instagram**  
  - Type: Facebook Graph API  
  - Role: Publishes the media object to Instagram feed.  
  - Config: POST to `/media_publish` edge with creation_id parameter.  
  - Edge cases: API errors, publishing failures.

- **Check status of post**  
  - Type: Facebook Graph API  
  - Role: Polls published post status to check if it is "PUBLISHED".  
  - Config: GET request for post ID fields including status and status_code.  
  - Edge cases: API errors.

- **If media status is finished1**  
  - Type: If  
  - Role: Branches based on whether post status_code equals "PUBLISHED".  
  - Edge cases: Expression errors.

- **Telegram**  
  - Type: Telegram  
  - Role: Sends notification if media upload failed before finishing.  
  - Config: Message "Video upload edilmeden önce bir problem oldu" (Problem before video upload).  
  - Edge cases: Telegram API errors.

- **Telegram1**  
  - Type: Telegram  
  - Role: Sends notification on successful Instagram content sharing.  
  - Config: Message "Instagram Content is shared".  
  - Edge cases: Telegram API errors.

- **Telegram2**  
  - Type: Telegram  
  - Role: Sends notification if there was a problem during Instagram content upload.  
  - Config: Message "There was a problem when execution a upload content to instagram".  
  - Edge cases: Telegram API errors.

---

### 3. Summary Table

| Node Name                         | Node Type                  | Functional Role                                      | Input Node(s)                      | Output Node(s)                         | Sticky Note                                                                                              |
|----------------------------------|----------------------------|-----------------------------------------------------|----------------------------------|--------------------------------------|--------------------------------------------------------------------------------------------------------|
| Schedule Trigger1                 | Schedule Trigger           | Starts workflow on schedule                          |                                  | Replicate params                     | ## Schedule Your Time To  Post                                                                           |
| Replicate params                 | Set                        | Stores Replicate API token                           | Schedule Trigger1                | Instagram params                    | ## Guide \n** [Guide](https://replicate.com) of getting  of Replicate Token                            |
| Instagram params                 | Set                        | Stores Instagram Business Account ID                 | Replicate params                | Telegram Params                    | ## Guide \n** [Guide](https://docs.matillion.com/metl/docs/6957316//) of getting  of Instagram Business Account Id |
| Telegram Params                 | Set                        | Stores Telegram chat ID                              | Instagram params                | Rapid Api params                   | ## Guide \n** [Guide](https://rapidapi.com/i-yqerddkq0t/api/telegram92/tutorials/how-to-get-the-id-of-a-telegram-channel,-chat,-user-or-bot%3F) of Getting  of Telegram Chat Id |
| Rapid Api params                | Set                        | Stores RapidAPI key                                 | Telegram Params                 | get top trends on instagram #isometric, get  top trends on instagram #blender3d | ## Guide \n** [Guide](https://docs.rapidapi.com/docs/keys-and-key-rotation) of Getting  of Rapid Api Key |
| get top trends on instagram #isometric | HTTP Request               | Fetches top posts for #isometric                     | Rapid Api params               | filter the image content            | ## Getting Top Trend Posts On Instagram\n** Change the topic you want to get on http request           |
| get  top trends on instagram #blender3d | HTTP Request               | Fetches top posts for #blender3d                      | Rapid Api params               | filter the image content-2          | ## Getting Top Trend Posts On Instagram\n** Change the topic you want to get on http request           |
| filter the image content         | Code                       | Filters out videos and maps relevant fields          | get top trends on instagram #isometric | merge the array content             |                                                                                                        |
| filter the image content-2       | Code                       | Filters out videos and maps relevant fields          | get  top trends on instagram #blender3d | merge the array content             |                                                                                                        |
| merge the array content          | Merge                      | Merges filtered arrays from both hashtags            | filter the image content, filter the image content-2 | Loop Over Items                   |                                                                                                        |
| Loop Over Items                 | SplitInBatches             | Processes each trending post individually             | merge the array content         | Check Data on Database Is Exist     | ## Looping Data And Checking For Is Exist On Database\n**We are checking until find a data we did not insert because we don't want to create content about in same content |
| Check Data on Database Is Exist  | PostgreSQL                 | Checks if content code exists in DB                   | Loop Over Items                | If Data is Exist, send error message to telegram | ## Warning\n** Don't forgot the create top_trends table                                                |
| If Data is Exist                | If                         | Branches based on DB existence                         | Check Data on Database Is Exist | Loop Over Items (if exists), insert data on db (if not) |                                                                                                        |
| insert data on db               | PostgreSQL                 | Inserts new trending post data                         | If Data is Exist               | Analyze Image and give the content  |                                                                                                        |
| send error message to telegram  | Telegram                   | Sends error notification on DB query failure          | Check Data on Database Is Exist |                                  |                                                                                                        |
| Analyze Image and give the content | OpenAI (Langchain)         | Generates descriptive content of image                 | insert data on db              | Analyze Content And Generate Instagram Caption | ## Analyze Post Content\n** We are analyzing the image\n** We are generating a instagram caption by content\n** Then we are generating the image |
| Analyze Content And Generate Instagram Caption | OpenAI (Langchain)         | Creates Instagram caption from description             | Analyze Image and give the content | Generate image on flux             |                                                                                                        |
| Generate image on flux          | HTTP Request               | Generates AI image based on description                | Analyze Content And Generate Instagram Caption | Prepare data on Instagram         |                                                                                                        |
| Prepare data on Instagram       | Facebook Graph API         | Creates Instagram media object with image and caption | Generate image on flux         | Check Status Of Media Before Uploaded | ## Publish On Instagram And Send Message When Published via Telegram                                    |
| Check Status Of Media Before Uploaded | Facebook Graph API         | Checks media upload status                              | Prepare data on Instagram      | If media status is finished         |                                                                                                        |
| If media status is finished     | If                         | Branches if media upload finished                       | Check Status Of Media Before Uploaded | Publish Media on Instagram (if finished), Telegram (if not) |                                                                                                        |
| Publish Media on Instagram      | Facebook Graph API         | Publishes media to Instagram feed                       | If media status is finished    | Check status of post                |                                                                                                        |
| Check status of post            | Facebook Graph API         | Checks post publishing status                           | Publish Media on Instagram     | If media status is finished1        |                                                                                                        |
| If media status is finished1    | If                         | Branches if post status is "PUBLISHED"                  | Check status of post           | Telegram1 (if published), Telegram2 (if error) |                                                                                                        |
| Telegram                       | Telegram                   | Sends notification if media upload failed               | If media status is finished    |                                  |                                                                                                        |
| Telegram1                      | Telegram                   | Sends notification on successful Instagram post         | If media status is finished1   |                                  |                                                                                                        |
| Telegram2                      | Telegram                   | Sends notification if Instagram post upload failed      | If media status is finished1   |                                  |                                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Cron expression: `5 13,19 * * *` (runs at 13:05 and 19:05 daily)  
   - Position: Start of workflow.

2. **Create Set node "Replicate params"**  
   - Add string field `replicate_token` (empty by default)  
   - Connect from Schedule Trigger.

3. **Create Set node "Instagram params"**  
   - Add string field `instagram_business_account_id` (empty by default)  
   - Connect from Replicate params.

4. **Create Set node "Telegram Params"**  
   - Add string field `telegram_chat_id` (empty by default)  
   - Connect from Instagram params.

5. **Create Set node "Rapid Api params"**  
   - Add string field `x-rapid-api-key` (empty by default)  
   - Connect from Telegram Params.

6. **Create two HTTP Request nodes for Instagram Scraper API:**  
   - Node 1: "get top trends on instagram #isometric"  
     - Method: GET  
     - URL: `https://instagram-scraper-api2.p.rapidapi.com/v1/hashtag`  
     - Query parameters: `hashtag=isometric`, `feed_type=top`  
     - Headers: `x-rapidapi-host=instagram-scraper-api2.p.rapidapi.com`, `x-rapidapi-key={{ $json['x-rapid-api-key'] }}`  
     - Connect from Rapid Api params.

   - Node 2: "get top trends on instagram #blender3d"  
     - Same as above, but `hashtag=blender3d`  
     - Connect from Rapid Api params.

7. **Create two Code nodes to filter images:**  
   - Node 1: "filter the image content"  
     - Input: Output of #isometric trends node  
     - JS code filters out videos and maps fields: id, prompt (caption text), content_code (code), thumbnail_url, tag (hashtag name)  
     - Connect from "get top trends on instagram #isometric".

   - Node 2: "filter the image content-2"  
     - Same as above for #blender3d trends node  
     - Connect from "get top trends on instagram #blender3d".

8. **Create Merge node "merge the array content"**  
   - Merge mode: default  
   - Connect inputs from both filter code nodes.

9. **Create SplitInBatches node "Loop Over Items"**  
   - Default batch size (1)  
   - Connect from Merge node.

10. **Create PostgreSQL node "Check Data on Database Is Exist"**  
    - Operation: Select  
    - Table: `top_trends` (schema: public)  
    - Where clause: `code = {{$json.content_code}}`  
    - Connect from Loop Over Items (main output 2).

11. **Create If node "If Data is Exist"**  
    - Condition: Check if query result is not empty (`!$json.isEmpty()`)  
    - Connect from PostgreSQL node.

12. **Connect If node branches:**  
    - True branch: Connect back to Loop Over Items (to skip existing content)  
    - False branch: Connect to PostgreSQL node "insert data on db".

13. **Create PostgreSQL node "insert data on db"**  
    - Operation: Insert  
    - Table: `top_trends`  
    - Columns: tag, code, prompt, isposted (false), thumbnail_url  
    - Connect from False branch of If node.

14. **Create OpenAI node "Analyze Image and give the content"**  
    - Resource: Image  
    - Operation: Analyze  
    - Model: gpt-4o-mini  
    - Image URL: `{{$json.thumbnail_url}}` from Loop Over Items  
    - Prompt: Detailed instructions to describe object features only  
    - Connect from insert data on db.

15. **Create OpenAI node "Analyze Content And Generate Instagram Caption"**  
    - Resource: Chat Completion  
    - Model: gpt-4o-mini  
    - Prompt: Summarize description into Instagram caption with hashtags (#Blender3D, #3DArt, #DigitalArt, #3DModeling, #ArtCommunity)  
    - Connect from Analyze Image node.

16. **Create HTTP Request node "Generate image on flux"**  
    - Method: POST  
    - URL: `https://api.replicate.com/v1/models/black-forest-labs/flux-schnell/predictions`  
    - Headers: Authorization Bearer token from Replicate params, Prefer: wait  
    - Body: JSON with prompt constructed from analyzed content (cleaned text), output_format jpg, quality 100  
    - Connect from Analyze Content And Generate Instagram Caption.

17. **Create Facebook Graph API node "Prepare data on Instagram"**  
    - Operation: POST to `/media` edge of Instagram Business Account ID  
    - Parameters: image_url from generated image, caption from AI caption node  
    - Connect from Generate image on flux.

18. **Create Facebook Graph API node "Check Status Of Media Before Uploaded"**  
    - Operation: GET media status fields (id, status, status_code)  
    - Node ID from Prepare data on Instagram output  
    - Connect from Prepare data on Instagram.

19. **Create If node "If media status is finished"**  
    - Condition: status_code equals "FINISHED"  
    - Connect from Check Status Of Media Before Uploaded.

20. **Connect If node branches:**  
    - True branch: Connect to Facebook Graph API node "Publish Media on Instagram"  
    - False branch: Connect to Telegram node "Telegram" (error notification).

21. **Create Facebook Graph API node "Publish Media on Instagram"**  
    - Operation: POST to `/media_publish` edge with creation_id from media object  
    - Connect from True branch of If node.

22. **Create Facebook Graph API node "Check status of post"**  
    - Operation: GET post status fields (id, status, status_code)  
    - Node ID from Publish Media node output  
    - Connect from Publish Media on Instagram.

23. **Create If node "If media status is finished1"**  
    - Condition: status_code equals "PUBLISHED"  
    - Connect from Check status of post.

24. **Connect If node branches:**  
    - True branch: Connect to Telegram node "Telegram1" (success notification)  
    - False branch: Connect to Telegram node "Telegram2" (failure notification).

25. **Create Telegram nodes:**  
    - "Telegram": Sends "Video upload edilmeden önce bir problem oldu" on media upload failure  
    - "Telegram1": Sends "Instagram Content is shared" on success  
    - "Telegram2": Sends "There was a problem when execution a upload content to instagram" on failure  
    - All use chat ID from Telegram Params.

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                                                                                   |
|-----------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| All Credentials You Need: Instagram Business Account Id, Telegram Chat Id, Rapid Api Key, Replicate Token        | Sticky Note2                                                                                                     |
| Guide for Instagram Business Account Id                                                                         | [Guide](https://docs.matillion.com/metl/docs/6957316//)                                                         |
| Guide for Telegram Chat Id                                                                                       | [Guide](https://rapidapi.com/i-yqerddkq0t/api/telegram92/tutorials/how-to-get-the-id-of-a-telegram-channel,-chat,-user-or-bot%3F) |
| Guide for Rapid Api Key                                                                                          | [Guide](https://docs.rapidapi.com/docs/keys-and-key-rotation)                                                    |
| Guide for Replicate Token                                                                                        | [Guide](https://replicate.com)                                                                                   |
| Warning: Create Telegram bot and send a message to bot first                                                    | Sticky Note7                                                                                                     |
| Warning: Create `top_trends` table in PostgreSQL with specified schema                                          | Sticky Note12 and Sticky Note13                                                                                  |
| Warning: Subscribe to Instagram Scraper API on RapidAPI                                                         | [Instagram Scraper Api](https://rapidapi.com/social-api1-instagram/api/instagram-scraper-api2/playground/apiendpoint_a45552b2-9850-4da9-b5cb-bbdd3ac2199d) |
| Warning: Instagram Scraper API rate limit is 500 requests/month for free tier                                    | [Rate Limit](https://rapidapi.com/social-api1-instagram/api/instagram-scraper-api2)                              |
| Facebook Scraper API Guide                                                                                       | [Guide](https://rapidapi.com/social-api1-instagram/api/instagram-scraper-api2/playground/apiendpoint_a45552b2-9850-4da9-b5cb-bbdd3ac2199d) |
| Workflow Description: Automates discovery, AI analysis, generation, and publishing of Instagram content          | Sticky Note14                                                                                                    |
| Schedule Your Time To Post                                                                                        | Sticky Note3                                                                                                     |
| Analyze Post Content: Image analysis, caption generation, image generation                                       | Sticky Note                                                                                                      |
| Publish On Instagram And Send Message When Published via Telegram                                               | Sticky Note1                                                                                                     |

---

This document fully describes the workflow structure, node configurations, and setup instructions to enable reproduction, modification, and troubleshooting by advanced users or automation agents.