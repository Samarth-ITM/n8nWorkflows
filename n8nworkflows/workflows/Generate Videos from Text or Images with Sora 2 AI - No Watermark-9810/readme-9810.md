Generate Videos from Text or Images with Sora 2 AI - No Watermark

https://n8nworkflows.xyz/workflows/generate-videos-from-text-or-images-with-sora-2-ai---no-watermark-9810


# Generate Videos from Text or Images with Sora 2 AI - No Watermark

---

# Reference Document for "Generate Videos from Text or Images with Sora 2 AI - No Watermark" Workflow

---

### 1. Workflow Overview

This n8n workflow enables users to generate professional AI videos from either text descriptions or images without watermarks using the Sora 2 AI model via the Kie.AI API. Users submit requests through a web form, choosing between text-to-video or image-to-video generation paths. The workflow manages file uploads, API interactions, polling for video readiness, downloading the final video, and optional Telegram delivery.

**Target Use Cases:**  
- Content creators wanting quick AI-generated videos from prompts or images.  
- Automation of video creation pipelines without watermark constraints.  
- Integration with messaging platforms for automated video delivery.

**Logical Blocks:**  

- **1.1 Input Reception:** Receives user input via a form with text prompt, optional image, aspect ratio, and quality options.  
- **1.2 Routing Logic:** Determines whether the request is text-only or includes an image and directs the flow accordingly.  
- **1.3 Image Upload (Image-to-Video only):** Uploads user image to ImgBB to obtain a publicly accessible URL for the Sora 2 API.  
- **1.4 Video Generation Request:** Sends API requests to Kie.AI's Sora 2 model for either text-to-video or image-to-video generation.  
- **1.5 Polling for Completion:** Implements a wait-check loop to poll the video generation status until completion.  
- **1.6 Video Downloading:** Retrieves the generated video file once ready.  
- **1.7 Delivery:** Optionally sends the video to a Telegram chat.  
- **1.8 Documentation & Best Practices:** Provides embedded setup instructions, workflow explanation notes, and prompt guidance.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception

**Overview:**  
Captures user input from a web form including a video description prompt, optional starting image, aspect ratio selection, and video quality choice.

**Nodes Involved:**  
- On form submission

**Node Details:**  

- **On form submission**  
  - *Type:* Form Trigger  
  - *Role:* Entry point accepting user input via a customized web form.  
  - *Configuration:*  
    - Form title: "🎬 AI Video Generator - Sora 2"  
    - Fields:  
      - Textarea for "Video Description (Prompt)" (required)  
      - Dropdown for "Aspect Ratio" with options: landscape, portrait (required)  
      - Dropdown for "Video Quality" with options: standard, hd (required)  
      - File upload for "Upload Starting Image (Optional)" accepting .jpg, .jpeg, .png, .webp  
    - Webhook ID assigned for external access.  
    - No attribution appended.  
  - *Input Connections:* None (trigger node)  
  - *Output Connections:* To "Check: Has Image?" node  
  - *Edge Cases:*  
    - Missing required fields will prevent submission.  
    - Uploaded files must be valid image types; invalid files rejected by form.  
  - *Version:* 2.3

---

#### 1.2 Routing Logic

**Overview:**  
Determines the workflow path based on whether the user uploaded an image or not.

**Nodes Involved:**  
- Check: Has Image?  
- 🚦 Router Logic (Sticky Note)

**Node Details:**  

- **Check: Has Image?**  
  - *Type:* Switch Node  
  - *Role:* Checks if the binary field for the uploaded image exists to route to image or text video generation paths.  
  - *Configuration:*  
    - Two outputs:  
      - "No Image - Text to Video" if no image uploaded (binary data missing)  
      - "Has Image - Upload First" if image uploaded (binary data exists)  
    - Uses expression to detect presence of binary data field named `Upload_Starting_Image__Optional_`.  
  - *Input Connections:* From "On form submission"  
  - *Output Connections:*  
    - "No Image - Text to Video" → "TEXT TO VIDEO" node (text path)  
    - "Has Image - Upload First" → "Upload to ImgBB" node (image path)  
  - *Edge Cases:*  
    - Binary field detection might fail if form structure changes.  
    - Empty or corrupted file upload could cause false positives.  
  - *Version:* 3.2

---

#### 1.3 Image Upload (Image-to-Video only)

**Overview:**  
Uploads the user’s image to ImgBB to obtain a public URL required by the Sora 2 API for image-to-video generation.

**Nodes Involved:**  
- Upload to ImgBB  
- 🎨 Image-to-Video Path (Sticky Note)

**Node Details:**  

- **Upload to ImgBB**  
  - *Type:* HTTP Request  
  - *Role:* Sends the uploaded image as multipart form data to ImgBB API for hosting.  
  - *Configuration:*  
    - POST to https://api.imgbb.com/1/upload  
    - Query parameters:  
      - `key`: API key (must be replaced with user's ImgBB API key)  
      - `expiration`: 0 (indefinite)  
    - Body: binary form data with field "image" mapped from uploaded file  
  - *Input Connections:* From "Check: Has Image?" (when image exists)  
  - *Output Connections:* To "IMAGE TO VIDEO" node  
  - *Edge Cases:*  
    - Invalid or missing ImgBB API key causes authentication failure.  
    - Rate limits on free ImgBB API tier may cause upload failures.  
    - Large images may fail due to size restrictions.  
  - *Version:* 4.2

---

#### 1.4 Video Generation Request

**Overview:**  
Sends video generation requests to Kie.AI's Sora 2 API with appropriate inputs for either text-to-video or image-to-video.

**Nodes Involved:**  
- TEXT TO VIDEO  
- IMAGE TO VIDEO  
- 🚦 Router Logic (contextual)  
- 💬 Text-to-Video Path (Sticky Note)  
- 🎨 Image-to-Video Path (Sticky Note)

**Node Details:**  

- **TEXT TO VIDEO**  
  - *Type:* HTTP Request  
  - *Role:* Submits a text prompt with aspect ratio and quality to the Sora 2 text-to-video model.  
  - *Configuration:*  
    - POST to https://api.kie.ai/api/v1/jobs/createTask  
    - JSON body includes:  
      - model: "sora-2-text-to-video"  
      - input: prompt (from form), aspect_ratio, quality (from form)  
    - Uses generic HTTP header authentication with credential "Kie Ai(Veo and more)"  
  - *Input:* From "Check: Has Image?" (no image branch)  
  - *Output:* To "Wait (Text)" node  
  - *Edge Cases:*  
    - Invalid API key or expired token causes auth errors.  
    - Malformed prompts might cause API errors or unexpected results.  
    - API downtime or rate limits may cause request failures.  
  - *Version:* 4.2

- **IMAGE TO VIDEO**  
  - *Type:* HTTP Request  
  - *Role:* Submits an image URL, prompt, aspect ratio, and quality to the Sora 2 image-to-video model.  
  - *Configuration:*  
    - POST to https://api.kie.ai/api/v1/jobs/createTask  
    - JSON body includes:  
      - model: "sora-2-image-to-video"  
      - input:  
        - prompt (from "Check: Has Image?" node JSON)  
        - image_urls: array with the ImgBB URL from previous node  
        - aspect_ratio, quality (from "Check: Has Image?" node JSON)  
    - Uses same generic HTTP header authentication as above  
  - *Input:* From "Upload to ImgBB"  
  - *Output:* To "Wait (Image)" node  
  - *Edge Cases:*  
    - ImgBB URL must be valid and accessible, otherwise API rejects request.  
    - Same auth and prompt issues as text path.  
  - *Version:* 4.2

---

#### 1.5 Polling for Completion

**Overview:**  
Implements a loop to wait and check repeatedly until the video generation is complete, preventing API overload and respecting rate limits.

**Nodes Involved:**  
- Wait (Text)  
- Check Status (Text)  
- Is Ready? (Text)  
- Wait (Image)  
- Check Status (Image)  
- Is Ready? (Image)  
- ⏱️ Polling Explained (Sticky Note)

**Node Details:**  

- **Wait (Text)**  
  - *Type:* Wait  
  - *Role:* Pauses workflow for 30 seconds before checking status again.  
  - *Input:* From "TEXT TO VIDEO" (first) or "Is Ready? (Text)" (loop)  
  - *Output:* To "Check Status (Text)"  
  - *Edge Cases:* N/A  
  - *Version:* 1.1

- **Check Status (Text)**  
  - *Type:* HTTP Request  
  - *Role:* Queries Kie.AI API for current job status of text-to-video task.  
  - *Configuration:*  
    - GET https://api.kie.ai/api/v1/jobs/recordInfo?taskId={{ $json.data.taskId }}  
    - Uses same HTTP header auth credential  
  - *Input:* From "Wait (Text)"  
  - *Output:* To "Is Ready? (Text)"  
  - *Edge Cases:*  
    - Failure to retrieve status due to network or auth errors.  
  - *Version:* 4.2

- **Is Ready? (Text)**  
  - *Type:* If  
  - *Role:* Checks if the job state equals "success".  
  - *Configuration:*  
    - Condition: `{{$json.data.state}} == "success"`  
  - *Input:* From "Check Status (Text)"  
  - *Output:*  
    - If true: to "Download Video (Text)"  
    - If false: back to "Wait (Text)" to continue polling  
  - *Edge Cases:*  
    - Unexpected states or missing `state` field cause loop to continue indefinitely or possible errors.  
  - *Version:* 2.2

- **Wait (Image)**  
  - *Type:* Wait  
  - *Role:* Same as Wait (Text) but for image-to-video path.  
  - *Input:* From "IMAGE TO VIDEO" or "Is Ready? (Image)"  
  - *Output:* To "Check Status (Image)"  
  - *Version:* 1.1

- **Check Status (Image)**  
  - *Type:* HTTP Request  
  - *Role:* Same as Check Status (Text), but for image-to-video.  
  - *Configuration:* Same as above with taskId from image job.  
  - *Input:* From "Wait (Image)"  
  - *Output:* To "Is Ready? (Image)"  
  - *Version:* 4.2

- **Is Ready? (Image)**  
  - *Type:* If  
  - *Role:* Same as Is Ready? (Text) but for image-to-video.  
  - *Input:* From "Check Status (Image)"  
  - *Output:*  
    - If ready: to "Download Video (Image)"  
    - Else: back to "Wait (Image)"  
  - *Version:* 2.2

---

#### 1.6 Video Downloading

**Overview:**  
Downloads the generated video file from the URL returned by the API once generation is complete.

**Nodes Involved:**  
- Download Video (Text)  
- Download Video (Image)

**Node Details:**  

- **Download Video (Text)**  
  - *Type:* HTTP Request  
  - *Role:* Fetches the video file from the URL extracted from the API response JSON.  
  - *Configuration:*  
    - URL dynamically set as `={{ JSON.parse($json.data.resultJson).resultUrls[0] }}`  
    - No authentication needed  
  - *Input:* From "Is Ready? (Text)" (success branch)  
  - *Output:* To "Send Text to Video" (Telegram delivery)  
  - *Edge Cases:*  
    - Invalid or expired URLs lead to download failure.  
  - *Version:* 4.2

- **Download Video (Image)**  
  - *Type:* HTTP Request  
  - *Role:* Same as Download Video (Text) but for image-to-video.  
  - *Input:* From "Is Ready? (Image)" (success branch)  
  - *Output:* To "Send Image to Video"  
  - *Version:* 4.2

---

#### 1.7 Delivery (Optional)

**Overview:**  
Sends the downloaded video files as attachments via Telegram to a configured chat ID.

**Nodes Involved:**  
- Send Text to Video  
- Send Image to Video  
- 📤 OUTPUT & DELIVERY (Sticky Note)

**Node Details:**  

- **Send Text to Video**  
  - *Type:* Telegram Node  
  - *Role:* Sends the downloaded text-to-video file to Telegram chat.  
  - *Configuration:*  
    - chatId: expression `=YOUR_CHAT_ID` (must be replaced with actual chat ID)  
    - Operation: sendVideo with binaryData true  
    - Caption: "Here's your text to video" with MarkdownV2 parse mode  
    - Credential: Telegram API credential named "Imagebananabot" (user must configure)  
  - *Input:* From "Download Video (Text)"  
  - *Output:* None (end node)  
  - *Edge Cases:*  
    - Invalid chat ID or Telegram credential causes delivery failure.  
  - *Version:* 1.2

- **Send Image to Video**  
  - *Type:* Telegram Node  
  - *Role:* Same as above but for image-generated video.  
  - *Input:* From "Download Video (Image)"  
  - *Output:* None  
  - *Version:* 1.2

---

#### 1.8 Documentation & Best Practices

**Overview:**  
Sticky notes embedded in the workflow provide setup instructions, detailed explanations of each logical path, polling mechanism, and best practices for prompt writing and video quality optimization.

**Nodes Involved:**  
- 📖 Setup Instructions  
- 💬 Text-to-Video Path  
- 🎨 Image-to-Video Path  
- ⏱️ Polling Explained  
- 💡 Best Practices  
- 🚦 Router Logic (explains routing)  
- 📤 Delivery Options  

**Details:**  

- Provide stepwise setup guidance including API key acquisition, credential configuration, and optional Telegram bot setup.  
- Describe routing logic and flow sequences for each video generation path.  
- Explain the polling mechanism's purpose and benefits.  
- Tips for writing effective prompts and optimizing video quality and image selection.  
- Notes about removing Telegram nodes if delivery via Telegram is not desired.  

---

### 3. Summary Table

| Node Name            | Node Type             | Functional Role                            | Input Node(s)           | Output Node(s)              | Sticky Note                                                                                                                                                                                  |
|----------------------|-----------------------|--------------------------------------------|-------------------------|-----------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| On form submission   | Form Trigger          | Receives user inputs via web form           | None                    | Check: Has Image?            | Form trigger: Users submit video requests via web form                                                                                                                                       |
| Check: Has Image?    | Switch                | Routes flow based on image upload presence  | On form submission      | TEXT TO VIDEO, Upload to ImgBB | Routes workflow based on whether user uploaded an image; acts as traffic controller                                                                                                          |
| Upload to ImgBB      | HTTP Request          | Uploads user image to ImgBB for hosting     | Check: Has Image?       | IMAGE TO VIDEO               | Uploads user image to ImgBB for hosting (required for Sora API)                                                                                                                              |
| TEXT TO VIDEO        | HTTP Request          | Sends text prompt to Sora 2 API             | Check: Has Image?       | Wait (Text)                 | Sends text prompt to Sora 2 API for video generation                                                                                                                                          |
| IMAGE TO VIDEO       | HTTP Request          | Sends image URL + prompt to Sora 2 API      | Upload to ImgBB         | Wait (Image)                | Sends image URL + prompt to Sora 2 API for image-to-video generation                                                                                                                          |
| Wait (Text)          | Wait                  | Pauses 30s before polling text video status | TEXT TO VIDEO, Is Ready? (Text) | Check Status (Text)       | Wait 30 seconds for video generation to complete                                                                                                                                              |
| Check Status (Text)  | HTTP Request          | Checks text-to-video generation status      | Wait (Text)             | Is Ready? (Text)             | Queries if video is ready                                                                                                                                                                     |
| Is Ready? (Text)     | If                    | Checks if text video generation is complete | Check Status (Text)     | Download Video (Text), Wait (Text) | Checks if video generation is complete, loops back to wait if not ready                                                                                                                      |
| Download Video (Text)| HTTP Request          | Downloads generated text-to-video file       | Is Ready? (Text)        | Send Text to Video           | Downloads the generated video from text-to-video path                                                                                                                                         |
| Send Text to Video   | Telegram               | Sends text-generated video via Telegram     | Download Video (Text)   | None                        | Sends video to Telegram chat                                                                                                                                                                  |
| Wait (Image)         | Wait                  | Pauses 30s before polling image video status| IMAGE TO VIDEO, Is Ready? (Image) | Check Status (Image)      | Wait 30 seconds for image-to-video generation to complete                                                                                                                                     |
| Check Status (Image) | HTTP Request          | Checks image-to-video generation status     | Wait (Image)            | Is Ready? (Image)            | Queries if video is ready                                                                                                                                                                     |
| Is Ready? (Image)    | If                    | Checks if image video generation is complete| Check Status (Image)    | Download Video (Image), Wait (Image) | Checks if image-to-video is complete, loops back to wait if not ready                                                                                                                       |
| Download Video (Image)| HTTP Request         | Downloads generated image-to-video file      | Is Ready? (Image)       | Send Image to Video          | Downloads the generated video from image-to-video path                                                                                                                                         |
| Send Image to Video  | Telegram               | Sends image-generated video via Telegram    | Download Video (Image)  | None                        | Sends video to Telegram chat                                                                                                                                                                  |
| 📖 Setup Instructions| Sticky Note           | Workflow setup and API configuration guide  | None                    | None                        | Detailed setup instructions including API keys, credentials, and Telegram bot                                                                                                                 |
| 💬 Text-to-Video Path| Sticky Note           | Explains text-to-video generation flow      | None                    | None                        | Details text-to-video path steps, timing, and example prompts                                                                                                                                 |
| 🎨 Image-to-Video Path| Sticky Note           | Explains image-to-video generation flow     | None                    | None                        | Details image-to-video path steps, ImgBB upload, and tips                                                                                                                                     |
| ⏱️ Polling Explained | Sticky Note           | Explains polling wait-check-loop mechanism  | None                    | None                        | Explains why the workflow waits and polls in loops for completion                                                                                                                           |
| 💡 Best Practices    | Sticky Note           | Prompt writing and quality optimization tips| None                    | None                        | Advice on writing effective prompts and choosing quality settings                                                                                                                            |
| 🚦 Router Logic      | Sticky Note           | Explains routing decision based on image upload | None                 | None                        | Describes switch node logic deciding between text and image video paths                                                                                                                      |
| 📤 Delivery Options  | Sticky Note           | Explains output delivery options and Telegram setup | None              | None                        | Describes how to deliver videos via Telegram or alternatives, optional removal of Telegram nodes                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node: "On form submission"**  
   - Type: Form Trigger  
   - Configure form with these fields:  
     - Textarea labeled "Video Description (Prompt)" (required)  
     - Dropdown "Aspect Ratio" with options: landscape, portrait (required)  
     - Dropdown "Video Quality" with options: standard, hd (required)  
     - File upload "Upload Starting Image (Optional)" accepting .jpg, .jpeg, .png, .webp  
   - Set webhook ID or leave auto-generated.  
   - Disable append attribution.

2. **Create Switch Node: "Check: Has Image?"**  
   - Type: Switch (Version 3)  
   - Add two outputs:  
     - "No Image - Text to Video" if `Upload_Starting_Image__Optional_` binary data does not exist.  
     - "Has Image - Upload First" if binary data exists.  
   - Use expression to test presence of binary data field from form.

3. **Create HTTP Request Node: "Upload to ImgBB"**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: https://api.imgbb.com/1/upload  
   - Query parameters:  
     - key = your ImgBB API key (replace placeholder)  
     - expiration = 0  
   - Body parameters (multipart-form-data):  
     - Field "image" mapped to binary data from form upload.  
   - Connect "Has Image - Upload First" output of Switch to this node.

4. **Create HTTP Request Node: "TEXT TO VIDEO"**  
   - Method: POST  
   - URL: https://api.kie.ai/api/v1/jobs/createTask  
   - Authentication: HTTP Header Auth credential (named e.g. "Kie Ai(Veo and more)") with header "Authorization: Bearer YOUR_TOKEN"  
   - Request body (JSON):  
     ```json
     {
       "model": "sora-2-text-to-video",
       "input": {
         "prompt": "{{ $json['Video Description (Prompt)'] }}",
         "aspect_ratio": "{{ $json['Aspect Ratio'] }}",
         "quality": "{{ $json['Video Quality'] }}"
       }
     }
     ```  
   - Connect "No Image - Text to Video" output of Switch to this node.

5. **Create HTTP Request Node: "IMAGE TO VIDEO"**  
   - Method: POST  
   - URL: https://api.kie.ai/api/v1/jobs/createTask  
   - Authentication: Same HTTP Header Auth credential as above.  
   - Request body (JSON):  
     ```json
     {
       "model": "sora-2-image-to-video",
       "input": {
         "prompt": "{{ $('Check: Has Image?').item.json['Video Description (Prompt)'] }}",
         "image_urls": ["{{ $json.data.url }}"],
         "aspect_ratio": "{{ $('Check: Has Image?').item.json['Aspect Ratio'] }}",
         "quality": "{{ $('Check: Has Image?').item.json['Video Quality'] }}"
       }
     }
     ```  
   - Connect "Upload to ImgBB" output to this node.

6. **Create Wait Nodes:**  
   - "Wait (Text)" and "Wait (Image)"  
   - Configure both to wait 30 seconds.  
   - Connect "TEXT TO VIDEO" to "Wait (Text)"  
   - Connect "IMAGE TO VIDEO" to "Wait (Image)"

7. **Create HTTP Request Nodes for Status Checking:**  
   - "Check Status (Text)" and "Check Status (Image)"  
   - Method: GET  
   - URLs:  
     - Text: `https://api.kie.ai/api/v1/jobs/recordInfo?taskId={{ $json.data.taskId }}`  
     - Image: same URL pattern with taskId from respective response  
   - Authentication: HTTP Header Auth credential as above.  
   - Connect "Wait (Text)" → "Check Status (Text)"  
   - Connect "Wait (Image)" → "Check Status (Image)"

8. **Create If Nodes to Check Completion:**  
   - "Is Ready? (Text)" and "Is Ready? (Image)"  
   - Condition: `{{$json.data.state}} == "success"`  
   - If true, proceed to download video  
   - If false, loop back to corresponding Wait node to continue polling.

9. **Create HTTP Request Nodes to Download Video:**  
   - "Download Video (Text)" and "Download Video (Image)"  
   - Method: GET  
   - URL from `={{ JSON.parse($json.data.resultJson).resultUrls[0] }}`  
   - Connect success output of "Is Ready? (Text)" to "Download Video (Text)"  
   - Connect success output of "Is Ready? (Image)" to "Download Video (Image)"

10. **(Optional) Create Telegram Nodes for Video Delivery:**  
    - "Send Text to Video" and "Send Image to Video"  
    - Operation: sendVideo  
    - chatId: replace `YOUR_CHAT_ID` with your Telegram chat ID  
    - Enable binary data sending  
    - Caption: "Here's your text to video" or "Here's your image to video"  
    - Credential: Telegram API credential configured with your bot token  
    - Connect "Download Video (Text)" → "Send Text to Video"  
    - Connect "Download Video (Image)" → "Send Image to Video"

11. **Add Sticky Notes (Optional) for Documentation:**  
    - Add notes describing setup instructions, routing logic, polling mechanism, best practices, and delivery options for clarity and future maintenance.

12. **Credential Setup:**  
    - Create HTTP Header Auth credential for Kie.AI API:  
      - Name: `Kie Ai(Veo and more)`  
      - Header Name: `Authorization`  
      - Header Value: `Bearer YOUR_API_KEY`  
    - Create Telegram API credential with your bot token if using Telegram delivery.  
    - Obtain ImgBB API key and replace placeholder in "Upload to ImgBB" node.

13. **Activate Workflow and Test:**  
    - Activate the workflow.  
    - Use "Test URL" from On form submission node to submit sample requests.  
    - Monitor logs and Telegram for video delivery.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                         | Context or Link                                                   |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| Generate professional AI videos without watermarks in minutes with Sora 2 AI by Kie.AI. Supports text-to-video and image-to-video with standard or HD quality.                                                                                      | Workflow purpose                                                  |
| Setup instructions include obtaining API keys from https://kie.ai and https://api.imgbb.com/, configuring HTTP Header Auth credentials, and optionally setting up Telegram bot for notifications.                                                    | Setup Instructions sticky note                                    |
| Telegram chat ID can be obtained using @get_id_bot on Telegram; replace `YOUR_CHAT_ID` in Telegram nodes accordingly.                                                                                                                               | Delivery Options sticky note                                       |
| Polling mechanism uses 30-second wait intervals to avoid API rate limits and ensure video generation completion before download.                                                                                                                    | Polling Explained sticky note                                     |
| Best practices for prompt writing include specificity, detail on camera movement, lighting, style, and technical specs such as 4K quality.                                                                                                         | Best Practices sticky note                                        |
| Image-to-video requires high-quality image uploads in JPG, PNG, or WebP formats and uses ImgBB for hosting. Free ImgBB tier has rate limits.                                                                                                        | Image-to-Video Path sticky note                                   |
| For advanced prompt generation, check out ChatGPT prompt generator for TikTok/Reels/Shorts scripts: https://chatgpt.com/g/g-68ee8b5b87f8819191126b5543bd9bd1-prompt-generator-ugc                                                                     | Best Practices sticky note (external resource link)               |
| Contact bilsimaging.com at contact@bilsimaging.com for advanced setup or support.                                                                                                                                                                    | Provided in setup sticky note                                     |

---

**Disclaimer:**  
The provided text and workflow are entirely generated from an n8n automated integration respecting all applicable content policies with no illegal, offensive, or protected material. All data processed are legal and publicly accessible.

---