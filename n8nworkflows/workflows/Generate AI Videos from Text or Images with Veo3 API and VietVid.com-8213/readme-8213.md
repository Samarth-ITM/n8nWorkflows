Generate AI Videos from Text or Images with Veo3 API and VietVid.com

https://n8nworkflows.xyz/workflows/generate-ai-videos-from-text-or-images-with-veo3-api-and-vietvid-com-8213


# Generate AI Videos from Text or Images with Veo3 API and VietVid.com

### 1. Workflow Overview

This workflow automates the creation of AI-generated videos using the Veo3 engine via the VietVid.com API. It is designed for users who want to generate videos from textual prompts or optional images by submitting a form, then automatically polling the API until the video is ready, and finally returning the video URLs for download or embedding.

**Target Use Cases:**  
- Content creators generating quick AI videos from text or images  
- Educators producing visual materials easily  
- Businesses automating video content creation for marketing or presentations  

**Logical Blocks:**  
- **1.1 Input Reception:** User submits video generation parameters via a web form.  
- **1.2 Video Request Submission:** The workflow sends a POST request to VietVid.com API to start video generation.  
- **1.3 Video Status Polling:** The workflow waits and repeatedly checks the status of video processing until completion.  
- **1.4 Video Link Return:** Once the video is ready, the workflow outputs the 720p and 1080p video URLs.  
- **1.5 User Guidance & Documentation:** Sticky notes provide step-by-step instructions, API details, and best practices.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block captures user input via a form trigger node, collecting essential parameters to generate the video.

**Nodes Involved:**  
- Enter prompt infomation (formTrigger)

**Node Details:**  
- **Type:** Form Trigger  
- **Role:** Provides a web form interface for users to enter video prompt, optional image URL, aspect ratio, model choice, and API key.  
- **Configuration:**  
  - Form Title: "Veo 3 video generator"  
  - Fields:  
    - text_prompt (text input, placeholder: “The cat playing in the garden”)  
    - ImageURL [optional] (text input, placeholder URL)  
    - api_Token (text input for API key)  
    - aspect_Ratio (dropdown: 16:9, 9:16)  
    - model (dropdown: veo3, veo3_fast)  
  - Required fields: aspect_Ratio, model  
- **Expressions/Variables:** User inputs are referenced later to build API requests.  
- **Input:** HTTP form submission  
- **Output:** JSON object with user inputs to next node  
- **Edge Cases:**  
  - Missing required fields will prevent submission  
  - Invalid API key format may cause auth errors downstream  
  - Empty image URL is allowed (text-to-video mode)  
- **Version-specific:** Uses version 2.2 of Form Trigger node  

---

#### 2.2 Video Request Submission

**Overview:**  
This block submits user inputs to the VietVid.com API to initiate video generation.

**Nodes Involved:**  
- Submit request

**Node Details:**  
- **Type:** HTTP Request  
- **Role:** Sends POST request to the generate endpoint with user-provided prompt and settings.  
- **Configuration:**  
  - Method: POST  
  - URL: https://vietvid.com/api/n8n/generate  
  - Headers:  
    - Content-Type: application/json  
    - Authorization: Bearer {{api_Token}} (from form input)  
  - JSON Body built dynamically using expressions referencing formTrigger outputs:  
    ```  
    {  
      "prompt": "{{ text_prompt }}",  
      "model": "{{ model }}",  
      "watermark": "",  
      "imageUrls": ["{{ ImageURL [optional] }}"],  
      "aspectRatio": "{{ aspect_Ratio }}",  
      "seeds": 50000  
    }  
    ```  
- **Input:** JSON from form node  
- **Output:** JSON response including taskId for status polling  
- **Expressions:** Dynamic templating to insert user inputs  
- **Edge Cases:**  
  - HTTP errors: invalid API key (401), malformed JSON, network issues  
  - Empty or invalid prompt may cause rejection by API  
  - Image URL invalid or unreachable may affect video generation  
- **Version-specific:** HTTP Request node v4.2  

---

#### 2.3 Video Status Polling

**Overview:**  
This block repeatedly waits and checks the video creation status by polling the API every 10 seconds until the video is ready.

**Nodes Involved:**  
- Wait for Video Processing  
- Check Video status  
- Check video available (720p or 1080p) [IF node]

**Node Details:**  

- **Wait for Video Processing:**  
  - Type: Wait node  
  - Role: Delays workflow execution by 10 seconds between status checks  
  - Configuration: 10 seconds delay  
  - Input: After video request submission or failed status check  
  - Output: Triggers status check HTTP request  

- **Check Video status:**  
  - Type: HTTP Request  
  - Role: Sends GET request to video status endpoint with taskId to retrieve current video status.  
  - Configuration:  
    - URL: https://vietvid.com/api/n8n/video-status/{{ taskId }} (taskId from Submit request response)  
    - Method: GET (default)  
  - Output: JSON with keys like status, videoUrl, hdVideoUrl  
  - Edge cases:  
    - Network errors, API unavailability  
    - Status “processing” indicates video not ready  
    - Possible rate limits from API  

- **Check video available (720p or 1080p):**  
  - Type: IF node  
  - Role: Checks if videoUrl exists and successFlag equals 1 in the API response  
  - Condition:  
    - Exists check on videoUrl  
    - successFlag == 1  
  - If TRUE: Proceed to Return video link  
  - If FALSE: Loop back to Wait node for another delay and check  
  - Edge cases:  
    - API response missing expected keys  
    - false successFlag or incomplete data causing loop delays  

---

#### 2.4 Video Link Return

**Overview:**  
Once the video is completed, this block extracts and stores the 720p and 1080p video URLs for downstream use.

**Nodes Involved:**  
- Return video link

**Node Details:**  
- **Type:** Set node  
- **Role:** Assigns output JSON fields with direct video link URLs from API response  
- **Configuration:**  
  - Creates two variables:  
    - 720p_link = videoUrl (always available)  
    - 1080p_link = hdVideoUrl (available only for 16:9 aspect ratio)  
- **Input:** JSON from Check video available node (API response)  
- **Output:** JSON with standardized video URL fields for further processing or user display  
- **Edge Cases:**  
  - 1080p_link may be null for 9:16 or 1:1 aspect ratios  
  - Missing videoUrl would indicate failure or incomplete processing  

---

#### 2.5 User Guidance & Documentation

**Overview:**  
Sticky Note nodes provide comprehensive documentation, usage instructions, and step-by-step guidance embedded within the workflow.

**Nodes Involved:**  
- Sticky Note - Overview  
- Sticky Note - Setup  
- Sticky Note - Step 1  
- Sticky Note - Step 2  
- Sticky Note - Step 4  
- Sticky Note - Step 5  
- Sticky Note - Step 3  

**Node Details:**  
These nodes contain markdown-formatted text covering:  
- Workflow purpose and pricing advantages  
- Setup instructions, including API key retrieval and form configuration  
- Detailed description of each step: obtaining API key, entering prompt, submitting request, polling status, and retrieving video links  
- HTTP request details and expected API responses  
- Tips for optimal prompt design and typical processing times  

They are positioned alongside respective functional nodes for user clarity and ease of maintenance. No technical configuration beyond content text.

---

### 3. Summary Table

| Node Name                      | Node Type          | Functional Role                        | Input Node(s)               | Output Node(s)                | Sticky Note                                                                                                                                                                                                                           |
|--------------------------------|--------------------|-------------------------------------|-----------------------------|------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Sticky Note - Overview          | Sticky Note        | Workflow purpose and pricing info   |                             |                              | ## Overview This workflow uses the VietVid.com Veo3 engine to create AI-powered videos from text or images...                                                                                                                      |
| Sticky Note - Setup             | Sticky Note        | Setup instructions and tips         |                             |                              | ## Setup Instructions 1. Get API Key...                                                                                                                                                                                             |
| Sticky Note - Step 1            | Sticky Note        | Instructions: Get API key           |                             |                              | ## STEP 1 - GET API KEY (YOURAPIKEY) - Create an account [here](https://vietvid.com/api) and obtain your API key...                                                                                                                |
| Sticky Note - Step 2            | Sticky Note        | Instructions: Enter prompt info     |                             |                              | ## STEP 2 - ENTER PROMPT INFORMATION Fill out the required fields: Prompt, Image URL (optional), Aspect Ratio, Seeds (optional), API Key...                                                                                        |
| Sticky Note - Step 3            | Sticky Note        | Instructions: Submit request        |                             |                              | ## STEP 3 - SUBMIT REQUEST (AUTO POST) The workflow will automatically POST your inputs to VietVid.com to generate the video...                                                                                                     |
| Sticky Note - Step 4            | Sticky Note        | Instructions: Wait for video creation |                             |                              | ## STEP 4 - WAIT FOR VIDEO CREATION The workflow will automatically check every 10 seconds until the video URL is available...                                                                                                     |
| Sticky Note - Step 5            | Sticky Note        | Instructions: Receive completed link |                             |                              | ## STEP 5 - RECEIVE COMPLETED VIDEO LINK When status = completed, the API returns video URL...                                                                                                                                       |
| Enter prompt infomation         | Form Trigger       | Capture user input for video params |                             | Submit request               |                                                                                                                                                                                                                                     |
| Submit request                 | HTTP Request       | POST video generation request       | Enter prompt infomation     | Wait for Video Processing    |                                                                                                                                                                                                                                     |
| Wait for Video Processing       | Wait               | Delay 10 seconds between status polls | Submit request, Check video available (false branch) | Check Video status           |                                                                                                                                                                                                                                     |
| Check Video status             | HTTP Request       | GET video status from API            | Wait for Video Processing   | Check video available         |                                                                                                                                                                                                                                     |
| Check video available (720p or 1080p) | IF                 | Check if video URL is ready          | Check Video status          | Return video link (true), Wait for Video Processing (false) |                                                                                                                                                                                                                                     |
| Return video link              | Set                | Output final video URLs              | Check video available       |                              |                                                                                                                                                                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node** named `Enter prompt infomation`:
   - Set form title to "Veo 3 video generator".
   - Add form fields:  
     - Text input `text_prompt` with placeholder "The cat playing in the garden".  
     - Text input `ImageURL [optional]` with placeholder e.g. "https://vietvid.com/example.jpg".  
     - Text input `api_Token` for API key with placeholder "47f33b46*************47f33b46".  
     - Dropdown `aspect_Ratio` with options: "16:9" and "9:16", required.  
     - Dropdown `model` with options: "veo3" and "veo3_fast", required.  
   - Use version 2.2 or higher.

2. **Add an HTTP Request node** named `Submit request`:
   - Method: POST  
   - URL: `https://vietvid.com/api/n8n/generate`  
   - Headers:  
     - Content-Type: application/json  
     - Authorization: Bearer {{ $json["api_Token"] }} (use expression referencing form input)  
   - Body: Raw JSON with parameters:  
     ```json
     {
       "prompt": "{{ $json["text_prompt"] }}",
       "model": "{{ $json["model"] }}",
       "watermark": "",
       "imageUrls": ["{{ $json["ImageURL [optional]"] }}"],
       "aspectRatio": "{{ $json["aspect_Ratio"] }}",
       "seeds": 50000
     }
     ```  
   - Set to send JSON body.  
   - Connect output of form trigger to this node.

3. **Add a Wait node** named `Wait for Video Processing`:
   - Set delay to 10 seconds.  
   - Connect output of `Submit request` to this node.

4. **Add an HTTP Request node** named `Check Video status`:
   - Method: GET  
   - URL: `https://vietvid.com/api/n8n/video-status/{{ $json["task"]["taskId"] }}` (use expression to extract taskId from Submit request response)  
   - Connect output of `Wait for Video Processing` to this node.

5. **Add an IF node** named `Check video available (720p or 1080p)`:
   - Condition: Check if `videoUrl` exists and `successFlag` equals 1 in the JSON response.  
   - Connect output of `Check Video status` to this IF node.

6. **Add a Set node** named `Return video link`:
   - Create two output variables:  
     - `720p_link` = `{{ $json["videoUrl"] }}`  
     - `1080p_link` = `{{ $json["hdVideoUrl"] }}`  
   - Connect the TRUE branch of the IF node to this node.

7. **Loop back:**  
   - Connect the FALSE branch of the IF node back to `Wait for Video Processing` node to continue polling.

8. **Add Sticky Notes** for documentation:  
   - Step 1: How to get API key from VietVid.com (include https://vietvid.com/api link)  
   - Step 2: Instructions for filling the form fields.  
   - Step 3: Details on the POST request to generate video.  
   - Step 4: Explanation about polling with GET request every 10 seconds.  
   - Step 5: Notes on video links returned and usage.  
   - Overview and Setup notes for general workflow explanation and prerequisites.

9. **Credentials:**  
   - No special credential node required; API key is passed dynamically from form input in Authorization header.

10. **Test the workflow:**  
    - Activate and run the workflow.  
    - Access the webhook URL from the form trigger node.  
    - Submit prompt and API key to see video generation process and final URLs.

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                  |
|------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| Videos start at $0.4 per 8-second clip and use credit packages with no expiration date.                     | Pricing Advantage (Sticky Note - Overview)       |
| API documentation and account registration available at https://vietvid.com/api                            | Step 1 - Get API Key instructions                |
| Typical video generation times: 720p ~1–3 minutes, 1080p ~2–3 minutes.                                       | Step 4 - Wait for Video Creation                  |
| For better AI video quality, enhance prompts with style, actions, and length details.                       | Setup Instructions sticky note                    |
| 1080p videos only available for 16:9 aspect ratio; others return null for hdVideoUrl                        | Step 5 - Receive Completed Video Link             |

---

**Disclaimer:** The provided text is extracted exclusively from an automated workflow created with n8n, an integration and automation tool. The process strictly adheres to content policies and contains no illegal or protected materials. All data handled is legal and public.