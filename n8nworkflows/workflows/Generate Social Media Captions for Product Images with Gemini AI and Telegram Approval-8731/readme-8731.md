Generate Social Media Captions for Product Images with Gemini AI and Telegram Approval

https://n8nworkflows.xyz/workflows/generate-social-media-captions-for-product-images-with-gemini-ai-and-telegram-approval-8731


# Generate Social Media Captions for Product Images with Gemini AI and Telegram Approval

### 1. Workflow Overview

This workflow is designed to automate the generation, approval, and posting of social media captions and hashtags for product images using Google Drive, Google Gemini AI, Telegram, and Twitter. It targets e-commerce or marketing teams who want to streamline content creation and publishing with AI assistance and human approval.

The workflow is logically divided into these blocks:

- **1.1 Input Reception**: Detect new product images uploaded to a specific Google Drive folder and download the image.
- **1.2 AI Image Analysis**: Use Google Gemini AI to analyze the image and extract structured descriptive data.
- **1.3 AI Caption & Hashtag Generation**: Use Google Gemini AI to generate a catchy caption and trending hashtags based on the image analysis.
- **1.4 Telegram Approval Flow**: Send the image with generated caption and hashtags to a Telegram user for approval, regeneration, or discard decision.
- **1.5 Posting and Confirmation**: Upon approval, upload the image to Twitter, post the tweet with the approved caption and hashtags, and notify via Telegram.
- **1.6 Regeneration or Discard Handling**: If the user requests regeneration, repeat caption generation; if discard, notify and stop.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Watches a designated Google Drive folder for new product image uploads, downloads the image for processing.
- **Nodes Involved:** Google Drive Trigger, Product_Img
- **Node Details:**

  - **Google Drive Trigger**
    - Type: Google Drive Trigger
    - Role: Monitors specific Google Drive folder for new files every minute.
    - Config: Watches folder ID `1A1Nmg58RBv969MqOIrvTWFlKbkgiKZBE` (named Product_images).
    - Input: None (trigger node).
    - Output: Metadata of newly uploaded file including link.
    - Edge Cases: Folder permission issues, API quota limits, delayed triggers.
  
  - **Product_Img**
    - Type: Google Drive (download)
    - Role: Downloads the triggered image file using its `webViewLink`.
    - Config: Uses OAuth2 credentials for Google Drive.
    - Input: Output of Google Drive Trigger (file link).
    - Output: Binary image data for AI analysis.
    - Edge Cases: File access errors, download failures.

#### 1.2 AI Image Analysis

- **Overview:** Analyzes the downloaded product image via Google Gemini AI to extract structured descriptive data.
- **Nodes Involved:** Analyze image, Temp_Field, Formatting_Block
- **Node Details:**

  - **Analyze image**
    - Type: Google Gemini (LangChain)
    - Role: Sends the product image to Gemini AI for structured JSON analysis describing objects, subject, colors, background, style, mood, and use cases.
    - Config: Uses `models/gemini-2.5-pro` model; input type is binary image.
    - Input: Binary data from Product_Img.
    - Output: Raw AI response with JSON inside text.
    - Edge Cases: API errors, timeouts, malformed response.
  
  - **Temp_Field**
    - Type: Set node
    - Role: Temporarily assigns a “name” field extracted from the AI response content (for intermediate handling).
    - Config: Extracts first candidate’s text part.
    - Input: Output of Analyze image.
    - Output: JSON with assigned name.
  
  - **Formatting_Block**
    - Type: Code (JavaScript)
    - Role: Cleans the raw AI JSON string by removing code fences, parses it to JSON, and wraps it under an `output` key for downstream nodes.
    - Config: JS code removes ```json and ``` fences, parses JSON.
    - Input: Output of Temp_Field.
    - Output: Structured JSON describing image.
    - Edge Cases: Parsing errors if AI response malformed.

#### 1.3 AI Caption & Hashtag Generation

- **Overview:** Sends the cleaned structured image description to Gemini AI flash model to generate a short caption and five trending hashtags in JSON format.
- **Nodes Involved:** Message a model, Output_Cleaner, Product_Img_1, Send_Photo, Ask_For_Approval
- **Node Details:**

  - **Message a model**
    - Type: Google Gemini (LangChain)
    - Role: Takes the image description JSON and instructs Gemini AI to generate a catchy caption (max 20 words) and five hashtags in JSON only.
    - Config: Uses `models/gemini-2.5-flash`; message content template injects fields from previous JSON.
    - Input: Output of Formatting_Block.
    - Output: Raw AI response with caption and hashtags JSON.
    - Edge Cases: Model response malformed, API errors.
  
  - **Output_Cleaner**
    - Type: Code (JavaScript)
    - Role: Cleans AI text response similarly by removing code fences and parsing JSON for use in Telegram message.
    - Input: Output of Message a model.
    - Output: JSON with `output` key containing caption and hashtags.
    - Edge Cases: Parsing failures.
  
  - **Product_Img_1**
    - Type: Google Drive (download)
    - Role: Downloads the original image again for sending to Telegram.
    - Input: Original file link from Google Drive Trigger.
    - Output: Binary image data.
  
  - **Send_Photo**
    - Type: Telegram
    - Role: Sends the downloaded product image to a specified Telegram chat, captioning it "Uploaded Photo".
    - Config: Uses Telegram API credentials; sends binary image.
    - Input: Output of Product_Img_1.
    - Output: Message sent confirmation.
  
  - **Ask_For_Approval**
    - Type: Telegram
    - Role: Sends the generated caption and hashtags to Telegram chat and waits for user approval, with options to approve or not.
    - Config: Sends formatted message with caption and hashtags, waits for approval response.
    - Input: Output of Send_Photo.
    - Output: Approval result boolean.
    - Edge Cases: User timeout, no response, malformed approval data.

#### 1.4 Telegram Approval Flow

- **Overview:** Receives user approval decision from Telegram to either proceed with posting, regenerate caption, or discard the image.
- **Nodes Involved:** Condition_Checker, Product_Img2, Request_Media_ID, Create Tweet, Send_Confirmation, Regenerate_Discard, Condition_Checker_1, Formatting_Block (reuse), Send_Confirmation1
- **Node Details:**

  - **Condition_Checker**
    - Type: If node
    - Role: Checks if Telegram user approved (true).
    - Input: Output of Ask_For_Approval.
    - Output: 
      - True branch: proceed to download image again and post.
      - False branch: send regenerate/discard options.
  
  - **Product_Img2**
    - Type: Google Drive (download)
    - Role: Downloads the product image again for Twitter upload.
    - Input: Original Google Drive link.
  
  - **Request_Media_ID**
    - Type: HTTP Request
    - Role: Uploads the image to Twitter media upload endpoint to get a media_id for tweeting with image.
    - Config: Uses Twitter OAuth1 credentials; multipart-form-data POST with binary image.
    - Input: Output of Product_Img2 (binary).
    - Output: Twitter media_id.
    - Edge Cases: Twitter API rate limits, upload failures.
  
  - **Create Tweet**
    - Type: Twitter node
    - Role: Posts the tweet with caption and hashtags, attaching uploaded media using media_id.
    - Config: Uses Twitter OAuth2 credentials; text from Output_Cleaner output; attaches media.
    - Input: Output of Request_Media_ID.
    - Output: Tweet confirmation.
  
  - **Send_Confirmation**
    - Type: Telegram
    - Role: Sends confirmation message to Telegram user that tweet was posted.
  
  - **Regenerate_Discard**
    - Type: Telegram
    - Role: Sends Telegram message asking user to choose between regenerating caption or discarding image with two approval buttons.
    - Output: Approval result.
  
  - **Condition_Checker_1**
    - Type: If node
    - Role: Checks if user chose "Regenerate" (true) or "Discard" (false).
    - True branch: loops back to Formatting_Block to regenerate caption.
    - False branch: sends discard confirmation and ends.
  
  - **Send_Confirmation1**
    - Type: Telegram
    - Role: Sends notification that image has been discarded.
    - Ends workflow.

#### 1.5 Posting and Confirmation

- Covered above in Create Tweet and Send_Confirmation nodes.

---

### 3. Summary Table

| Node Name           | Node Type                   | Functional Role                                      | Input Node(s)          | Output Node(s)           | Sticky Note                                                                                                                               |
|---------------------|-----------------------------|-----------------------------------------------------|------------------------|--------------------------|-------------------------------------------------------------------------------------------------------------------------------------------|
| Google Drive Trigger | Google Drive Trigger         | Trigger on new file in Google Drive folder          | -                      | Product_Img              |                                                                                                                                           |
| Product_Img          | Google Drive (download)      | Download the triggered image                         | Google Drive Trigger   | Analyze image            |                                                                                                                                           |
| Analyze image        | Google Gemini (Image Analysis) | Analyze image and extract structured JSON           | Product_Img            | Temp_Field               |                                                                                                                                           |
| Temp_Field           | Set                         | Extract text part from AI response                   | Analyze image          | Formatting_Block         |                                                                                                                                           |
| Formatting_Block     | Code (JavaScript)            | Clean and parse AI JSON response                      | Temp_Field, Condition_Checker_1 | Message a model, Message a model (loop) |                                                                                                                                           |
| Message a model      | Google Gemini (Text)          | Generate caption and hashtags from image description | Formatting_Block       | Output_Cleaner           |                                                                                                                                           |
| Output_Cleaner       | Code (JavaScript)            | Clean and parse caption and hashtag JSON             | Message a model        | Product_Img_1            |                                                                                                                                           |
| Product_Img_1        | Google Drive (download)      | Download image for Telegram photo                     | Output_Cleaner         | Send_Photo               |                                                                                                                                           |
| Send_Photo           | Telegram                    | Send image to Telegram chat                           | Product_Img_1          | Ask_For_Approval         |                                                                                                                                           |
| Ask_For_Approval     | Telegram                    | Send caption and hashtags, wait for approval         | Send_Photo             | Condition_Checker        |                                                                                                                                           |
| Condition_Checker    | If                          | Check approval decision                               | Ask_For_Approval       | Product_Img2 / Regenerate_Discard |                                                                                                                                           |
| Product_Img2         | Google Drive (download)      | Download image for Twitter upload                     | Condition_Checker (true) | Request_Media_ID         |                                                                                                                                           |
| Request_Media_ID     | HTTP Request                | Upload image to Twitter, get media_id                 | Product_Img2           | Create Tweet             | ## Request_Media_ID Node This node is used to upload the image to twitter server and this request will return us with a **Media_ID**, which is used to display product image in tweet. |
| Create Tweet         | Twitter                    | Post tweet with caption, hashtags, and media          | Request_Media_ID       | Send_Confirmation        |                                                                                                                                           |
| Send_Confirmation    | Telegram                    | Notify user tweet posted successfully                 | Create Tweet           | -                        |                                                                                                                                           |
| Regenerate_Discard   | Telegram                    | Ask user to regenerate caption or discard image       | Condition_Checker (false) | Condition_Checker_1      |                                                                                                                                           |
| Condition_Checker_1  | If                          | Check user choice for regeneration or discard         | Regenerate_Discard     | Formatting_Block / Send_Confirmation1 |                                                                                                                                           |
| Send_Confirmation1   | Telegram                    | Notify user image discarded                            | Condition_Checker_1 (false) | -                      |                                                                                                                                           |
| Sticky Note          | Sticky Note                 | Overview and explanation content                       | -                      | -                        | # Social Media Bot: AI Captioning, Approval & Auto-Tweet This n8n template demonstrates a complete AI-assisted social media workflow. It automatically generates captions and hashtags for product images, allows human approval through Telegram, and posts approved content to Twitter. |
| Sticky Note1         | Sticky Note                 | Explains Request_Media_ID node                         | -                      | -                        | ## Request_Media_ID Node This node is used to upload the image to twitter server and this request will return us with a **Media_ID**, which is used to display product image in tweet. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Google Drive Trigger node:**
   - Type: Google Drive Trigger
   - Parameters: Set `event` to `fileCreated`.
   - Poll every minute.
   - Set `triggerOn` to `specificFolder`.
   - Provide folder ID of the product images folder.
   - Attach Google Drive OAuth2 credentials.

2. **Add a Google Drive node to download the new file:**
   - Type: Google Drive
   - Operation: Download
   - File ID: Use the expression `{{$json.webViewLink}}` from the trigger.
   - Use same Google Drive credentials.

3. **Add a Google Gemini node to analyze image:**
   - Type: LangChain Google Gemini
   - Model: `models/gemini-2.5-pro`
   - Operation: Analyze image
   - Input type: Binary (image from previous node)
   - Text prompt: Provide detailed JSON schema requesting objects, subject, colors, background, style, mood, and use cases.
   - Use Google Gemini API credentials.

4. **Add a Set node (Temp_Field):**
   - Extract the first candidate's content from AI response and assign it to a temporary string field `name`.

5. **Add a Code node (Formatting_Block):**
   - Use JavaScript to strip code fences ```json and ``` from the AI text response.
   - Parse cleaned string into JSON.
   - Output wrapped inside an `output` key.

6. **Add a Google Gemini node to generate caption and hashtags:**
   - Model: `models/gemini-2.5-flash`
   - Input: Use the parsed JSON fields from the previous node as template variables.
   - Prompt: Ask to generate exactly one caption (max 20 words) and five hashtags in JSON format.
   - Use Google Gemini API credentials.

7. **Add another Code node (Output_Cleaner):**
   - Clean code fences and parse the text response JSON.
   - Output under an `output` key.

8. **Add another Google Drive node (Product_Img_1):**
   - Download the original image again using the trigger file link for Telegram sending.

9. **Add a Telegram node (Send_Photo):**
   - Operation: Send Photo
   - Chat ID: Set your Telegram chat ID.
   - Binary data: Enable to send image.
   - Message caption: "Uploaded Photo"
   - Use Telegram Bot credentials.

10. **Add a Telegram node (Ask_For_Approval):**
    - Operation: Send and Wait
    - Chat ID: Same as above.
    - Message: Include caption and hashtags from Output_Cleaner node.
    - Approval type: Double approval buttons (approve/disapprove).
    - Use Telegram credentials.

11. **Add an If node (Condition_Checker):**
    - Condition: Check if `{{$json.data.approved}}` is true.
    - True branch: proceed to next step.
    - False branch: send regenerate/discard options.

12. **On True branch:**
    - Add Google Drive node (Product_Img2) to download image again.
    - Add HTTP Request node (Request_Media_ID):
      - POST to `https://upload.twitter.com/1.1/media/upload.json`
      - Content type: multipart-form-data
      - Body parameter: binary media data.
      - Use Twitter OAuth1 credentials.
    - Add Twitter node (Create Tweet):
      - Text: Caption and hashtags from Output_Cleaner.
      - Attach media using `media_id_string` from previous step.
      - Use Twitter OAuth2 credentials.
    - Add Telegram node (Send_Confirmation):
      - Message: "✅ Your tweet has been successfully posted!"
      - Use Telegram credentials.

13. **On False branch:**
    - Add Telegram node (Regenerate_Discard):
      - Operation: Send and Wait
      - Message: "Confirm whether to regenerate or discard the image and stop the process."
      - Approval buttons: "Regenerate" and "Discard".
    - Add If node (Condition_Checker_1):
      - Check if approved (regenerate) or not (discard).
      - True branch (Regenerate): Connect back to Formatting_Block to regenerate caption and hashtags.
      - False branch (Discard): Add Telegram node (Send_Confirmation1) to notify image discarded and stop workflow.

14. **Add Sticky Notes as needed to explain workflow sections and key nodes.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                           | Context or Link                                   |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------|
| # Social Media Bot: AI Captioning, Approval & Auto-Tweet This n8n template demonstrates a complete AI-assisted social media workflow. It automatically generates captions and hashtags for product images, allows human approval through Telegram, and posts approved content to Twitter. | Sticky Note at start of workflow                  |
| ## Request_Media_ID Node This node is used to upload the image to twitter server and this request will return us with a **Media_ID**, which is used to display product image in tweet.                                                 | Sticky Note near Request_Media_ID node            |
| Requirements: Google Drive account, Gemini API credentials, Telegram bot with approval, Twitter Developer credentials.                                                                                                               | Workflow description and sticky notes             |
| Customization ideas: Replace Google Drive with Dropbox/Airtable, swap Twitter for LinkedIn/Instagram, extend Telegram approval for team voting/multi-step review.                                                                     | Sticky Note content                                |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created using n8n, a workflow automation tool. This processing strictly respects content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.