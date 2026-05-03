Generate Branded Article Images with GPT-4o, FAL Flux and Google Drive

https://n8nworkflows.xyz/workflows/generate-branded-article-images-with-gpt-4o--fal-flux-and-google-drive-8064


# Generate Branded Article Images with GPT-4o, FAL Flux and Google Drive

### 1. Workflow Overview

This workflow automates the creation of branded article images for web content by combining AI-powered image generation, image processing, and cloud storage. It is designed for content creators, marketers, and web publishers who want to generate sharp, customized article hero images automatically, incorporating a dynamic subject and a brand logo.

The workflow is logically divided into the following blocks:
- **1.1 Input Reception:** Accepts a topic (`subject`) and a reference image URL.
- **1.2 AI Prompt Creation:** Uses GPT-4o to generate a concise, detailed prompt for image generation, enforcing no text or logos.
- **1.3 Image Generation Request:** Sends the prompt and reference image to the FAL Flux AI image-to-image generation API, initiating a job and returning a `request_id`.
- **1.4 Polling for Completion:** Waits and polls the FAL Flux API until the job status is `COMPLETED`.
- **1.5 Post-Processing:** Downloads the generated image, resizes it, fetches and resizes the company logo, and composites the logo onto the image.
- **1.6 Output Saving:** Saves the final composited image to a specified Google Drive folder with a dynamic filename.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Starts the workflow manually and sets initial input variables.
- **Nodes Involved:**  
  - When clicking ‘Execute workflow’  
  - Link image

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - *Type:* Manual Trigger  
    - *Role:* Entry point to start the workflow manually.  
    - *Config:* No parameters; triggers workflow on user command.  
    - *Input/Output:* No input; outputs empty JSON to next node.  
    - *Edge Cases:* None.  

  - **Link image**  
    - *Type:* Set node  
    - *Role:* Defines initial input data: `image_url` (reference image URL) and `subject` (topic string).  
    - *Config:* Hardcoded example values:  
      - `image_url`: https://surecctv.com/uploads/images/2021/07/600x600-1627381898-single_product1-nk1080hir100af.jpg  
      - `subject`: "technology"  
    - *Expressions:* Uses static strings.  
    - *Input:* From manual trigger  
    - *Output:* JSON with `image_url` and `subject` fields.  
    - *Edge Cases:* The image URL should be accessible; otherwise, image generation may fail.

---

#### 2.2 AI Prompt Creation

- **Overview:** GPT-4o generates a concise, sharp image prompt based on the input topic, tailored for AI image generation without text or logos.
- **Nodes Involved:**  
  - Create prompt for generate image

- **Node Details:**

  - **Create prompt for generate image**  
    - *Type:* OpenAI (LangChain node)  
    - *Role:* Generates an AI prompt for the image model.  
    - *Config:*  
      - Model: GPT-4o  
      - Message content template:  
        ```
        Based on the topic: {{ $json.subject }}

        Create an image prompt so I can put it into the Image AI Model to create a sharp, detailed illustration. The image prompt must be under 70 words, sharp and detailed, but no text or logos can be added to the image (Important).

        The size of the article image for the website will be: 800x500 pixels
        ```  
      - No extra options set.  
    - *Expressions:* Uses `{{ $json.subject }}` from the input JSON.  
    - *Input:* Receives JSON with `subject` from "Link image" node.  
    - *Output:* JSON message content with generated prompt.  
    - *Credentials:* Requires OpenAI API key credentials.  
    - *Edge Cases:* API rate limits, invalid credentials, or malformed input.

---

#### 2.3 Image Generation Request

- **Overview:** Sends the generated prompt and reference image to the FAL Flux image-to-image API to queue an image generation job.
- **Nodes Involved:**  
  - generate image

- **Node Details:**

  - **generate image**  
    - *Type:* HTTP Request  
    - *Role:* Posts to FAL Flux `/image-to-image` endpoint to start image generation.  
    - *Config:*  
      - Method: POST  
      - URL: https://queue.fal.run/fal-ai/flux/dev/image-to-image  
      - Headers: Authorization with API key (placeholder `Key xxxxx`)  
      - JSON body uses:  
        - `prompt`: extracted from GPT-4o output `message.content` (stringified and trimmed)  
        - `image_url`: from "Link image" node's JSON  
        - `image_size`: 800×500 pixels  
        - `num_inference_steps`: 45 (controls generation detail)  
        - `guidance_scale`: 4.5 (controls prompt adherence)  
        - `num_images`: 1  
        - `enable_safety_checker`: true (to avoid unsafe content)  
    - *Expressions:*  
      - `prompt` uses expression: `{{ JSON.stringify($json.message.content).slice(1, -1) }}`  
      - `image_url` uses `{{JSON.stringify($('Link image').item.json.image_url)}}`  
    - *Input:* Receives prompt JSON from "Create prompt for generate image".  
    - *Output:* Returns JSON containing `request_id` for job tracking.  
    - *Edge Cases:* API key invalid, network errors, malformed request, rate limiting.

---

#### 2.4 Polling for Completion

- **Overview:** Polls the FAL Flux API at intervals to check if the image generation job has completed.
- **Nodes Involved:**  
  - Wait  
  - check image finish  
  - If3  
  - Get image link

- **Node Details:**

  - **Wait**  
    - *Type:* Wait node  
    - *Role:* Delays execution for 2 seconds between polling requests.  
    - *Config:* Wait amount = 2 seconds  
    - *Input:* Receives JSON with `request_id` from "generate image" or from "If3" loop.  
    - *Output:* Passes input unchanged.  
    - *Edge Cases:* Too short interval may hit API rate limits; too long slows workflow.

  - **check image finish**  
    - *Type:* HTTP Request  
    - *Role:* GET request to check job status.  
    - *Config:*  
      - URL: `https://queue.fal.run/fal-ai/flux/requests/{{ $json.request_id }}/status`  
      - Header: Authorization with API key (placeholder `Key xxxx`)  
    - *Input:* Receives JSON with `request_id` from "Wait" or "generate image".  
    - *Output:* Returns JSON with a `status` field (e.g., "COMPLETED").  
    - *Edge Cases:* API errors, invalid request_id, auth failures.

  - **If3**  
    - *Type:* If node  
    - *Role:* Checks if status equals "COMPLETED".  
    - *Config:* Condition: `$json.status == "COMPLETED"`  
    - *Input:* Receives JSON from "check image finish" node.  
    - *Output:*  
      - True branch: proceeds to "Get image link"  
      - False branch: loops back to "Wait" for another poll  
    - *Edge Cases:* Status never becomes "COMPLETED" → consider timeout branch (not implemented here).

  - **Get image link**  
    - *Type:* HTTP Request  
    - *Role:* Fetches the completed image metadata and download URLs.  
    - *Config:*  
      - URL: `https://queue.fal.run/fal-ai/flux/requests/{{ $json.request_id }}`  
      - Header: Authorization with API key (placeholder `Key xxxxxx`)  
    - *Input:* Receives JSON with `request_id` from "If3".  
    - *Output:* JSON containing an `images` array with URLs.  
    - *Edge Cases:* API failure, missing images array.

---

#### 2.5 Post-Processing

- **Overview:** Downloads the generated image, resizes it, downloads and resizes the company logo, merges both images.
- **Nodes Involved:**  
  - Download image  
  - resize image  
  - Get company's logo  
  - resize logo  
  - Merge2  
  - Merge Binary Items  
  - Composite image and logo

- **Node Details:**

  - **Download image**  
    - *Type:* HTTP Request  
    - *Role:* Downloads the generated image binary file.  
    - *Config:* URL taken from `images[0].url` in previous node output.  
    - *Input:* Receives JSON with image URLs from "Get image link".  
    - *Output:* Binary image data.  
    - *Edge Cases:* URL invalid or inaccessible, network errors.

  - **resize image**  
    - *Type:* Edit Image node  
    - *Role:* Resizes the downloaded image to width 800 pixels (height auto to preserve aspect ratio).  
    - *Config:* Operation: resize, width: 800, height not set (auto), no extra options.  
    - *Input:* Binary data from "Download image".  
    - *Output:* Resized image binary.  
    - *Edge Cases:* Image corruption, unsupported formats.

  - **Get company's logo**  
    - *Type:* HTTP Request  
    - *Role:* Downloads the company logo image from Google Drive (or any public URL).  
    - *Config:* URL: direct Google Drive download link (placeholder `xxxxx`)  
    - *Input:* Output from "resize image" node (used to synchronize flow).  
    - *Output:* Binary logo image data.  
    - *Edge Cases:* URL access restrictions, invalid file.

  - **resize logo**  
    - *Type:* Edit Image node  
    - *Role:* Resizes the company logo to fixed dimensions 68.4 × 52.6 pixels.  
    - *Config:* Operation: resize, width: 68.4, height: 52.6.  
    - *Input:* Binary logo image from "Get company's logo".  
    - *Output:* Resized logo binary.  
    - *Edge Cases:* Same as resize image.

  - **Merge2**  
    - *Type:* Merge node  
    - *Role:* Merges two input streams (resized main image and resized logo) into one output for further processing.  
    - *Config:* Default merge (combine inputs).  
    - *Inputs:*  
      - Input 1: resized main image from "resize image"  
      - Input 2: resized logo from "resize logo"  
    - *Output:* Combined JSON and binary data for further compositing.  
    - *Edge Cases:* Input mismatch, empty data streams.

  - **Merge Binary Items**  
    - *Type:* Code node  
    - *Role:* Consolidates binary image data from multiple inputs into a single item with separate binary properties (`data0`, `data1`).  
    - *Config:* JavaScript code that collects binaries from both inputs.  
    - *Input:* Output from "Merge2".  
    - *Output:* Single item with two binary fields for compositing.  
    - *Edge Cases:* Missing binaries, code errors.

  - **Composite image and logo**  
    - *Type:* Edit Image node  
    - *Role:* Composites the resized logo onto the resized main image at horizontal position X=728px.  
    - *Config:*  
      - Operation: composite  
      - PositionX: 728 (right side)  
      - Composite binary: uses `data1` (logo) over `data0` (main image)  
    - *Input:* Binary data from "Merge Binary Items".  
    - *Output:* Final composited image binary.  
    - *Edge Cases:* Misaligned images, transparency issues.

---

#### 2.6 Output Saving

- **Overview:** Saves the final composited image to a specified folder on Google Drive with a dynamically generated file name.
- **Nodes Involved:**  
  - Save on drive

- **Node Details:**

  - **Save on drive**  
    - *Type:* Google Drive node  
    - *Role:* Uploads the final image binary to Google Drive.  
    - *Config:*  
      - File name: dynamic expression `={{ $('Loop Over Items').first().json.ten }}` (user to replace or provide)  
      - Drive ID: "My Drive"  
      - Folder ID: user-defined folder (placeholder given)  
      - Input field: `data0` binary from "Composite image and logo"  
    - *Credentials:* Google Drive OAuth2 credentials required.  
    - *Edge Cases:* OAuth token expiry, insufficient permissions, invalid folder ID.

---

### 3. Summary Table

| Node Name                 | Node Type             | Functional Role                          | Input Node(s)                 | Output Node(s)               | Sticky Note                                                                                                                      |
|---------------------------|-----------------------|----------------------------------------|------------------------------|------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger         | Manual workflow start                   | -                            | Link image                   |                                                                                                                                 |
| Link image                | Set                   | Sets initial `image_url` and `subject` | When clicking ‘Execute workflow’ | Create prompt for generate image |                                                                                                                                 |
| Create prompt for generate image | OpenAI (LangChain)     | Generates image prompt from subject    | Link image                   | generate image               | Requires OpenAI credentials (GPT-4o model)                                                                                      |
| generate image            | HTTP Request          | Sends prompt and image_url to FAL Flux | Create prompt for generate image | Wait                        | Requires FAL API key; POST to `/image-to-image`                                                                                  |
| Wait                      | Wait                  | Waits 2 seconds before polling          | generate image, If3           | check image finish           | Polling interval can be adjusted to avoid rate limits                                                                           |
| check image finish        | HTTP Request          | Checks if image generation job completed | Wait                        | If3                         | Requires FAL API key; GET job status                                                                                            |
| If3                       | If                    | Checks if status == COMPLETED            | check image finish            | Get image link (true), Wait (false) | Add timeout branch if needed                                                                                                    |
| Get image link            | HTTP Request          | Retrieves image URLs after completion   | If3                         | Download image               | Requires FAL API key                                                                                                            |
| Download image            | HTTP Request          | Downloads generated image binary        | Get image link               | resize image, Get company's logo |                                                                                                                                 |
| resize image              | Edit Image            | Resizes main image to width 800px       | Download image               | Merge2                      | Final image size for article hero                                                                                              |
| Get company's logo         | HTTP Request          | Downloads company logo                   | resize image                | resize logo                  | Google Drive direct download link                                                                                              |
| resize logo               | Edit Image            | Resizes logo to ~68×53 px                | Get company's logo           | Merge2                      |                                                                                                                                 |
| Merge2                    | Merge                 | Merges resized image and logo streams   | resize image, resize logo    | Merge Binary Items           |                                                                                                                                 |
| Merge Binary Items        | Code                  | Consolidates binaries into one item     | Merge2                      | Composite image and logo     |                                                                                                                                 |
| Composite image and logo  | Edit Image            | Composites logo onto main image at X=728px | Merge Binary Items           | Save on drive               | Logo position and size customizable                                                                                            |
| Save on drive             | Google Drive          | Saves final image to Google Drive       | Composite image and logo     | -                           | Requires Google Drive OAuth2; dynamic filename expression                                                                       |
| resize logo, Composite image and logo, Merge2, Merge Binary Items | Multiple              | Branding and image compositing steps   | See above                   | See above                   | Branding nodes place logo at right side; tweak position/size as needed                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Start workflow manually.

2. **Add a Set Node ("Link image")**  
   - Define two fields:  
     - `image_url`: Set to a public image URL (e.g., reference image)  
     - `subject`: Set to your article topic (e.g., "technology")  
   - Connect Manual Trigger → Set node.

3. **Add OpenAI Node ("Create prompt for generate image")**  
   - Use LangChain OpenAI node.  
   - Model: GPT-4o  
   - Message template:  
     ```
     Based on the topic: {{ $json.subject }}

     Create an image prompt so I can put it into the Image AI Model to create a sharp, detailed illustration. The image prompt must be under 70 words, sharp and detailed, but no text or logos can be added to the image (Important).

     The size of the article image for the website will be: 800x500 pixels
     ```  
   - Connect Set node → OpenAI node.  
   - Set OpenAI credentials with your API key.

4. **Add HTTP Request Node ("generate image")**  
   - Method: POST  
   - URL: https://queue.fal.run/fal-ai/flux/dev/image-to-image  
   - Headers: Authorization: `Key YOUR_FAL_API_KEY`  
   - JSON Body:  
     ```json
     {
       "prompt": "{{ JSON.stringify($json.message.content).slice(1, -1) }}",
       "image_url": {{JSON.stringify($('Link image').item.json.image_url)}},
       "image_size": {"width": 800, "height": 500},
       "num_inference_steps": 45,
       "guidance_scale": 4.5,
       "num_images": 1,
       "enable_safety_checker": true
     }
     ```  
   - Connect OpenAI node → HTTP Request node.

5. **Add Wait Node ("Wait")**  
   - Wait time: 2 seconds  
   - Connect "generate image" → Wait node.

6. **Add HTTP Request Node ("check image finish")**  
   - Method: GET  
   - URL: `https://queue.fal.run/fal-ai/flux/requests/{{ $json.request_id }}/status`  
   - Headers: Authorization: `Key YOUR_FAL_API_KEY`  
   - Connect Wait node → check image finish node.

7. **Add If Node ("If3")**  
   - Condition: `$json.status == "COMPLETED"`  
   - Connect check image finish → If node.

8. **Add HTTP Request Node ("Get image link")**  
   - Method: GET  
   - URL: `https://queue.fal.run/fal-ai/flux/requests/{{ $json.request_id }}`  
   - Headers: Authorization: `Key YOUR_FAL_API_KEY`  
   - Connect If node (true branch) → Get image link node.

9. **Loop False Branch**  
   - Connect If node (false branch) → Wait node (to poll again).

10. **Add HTTP Request Node ("Download image")**  
    - URL: `={{ $json.images[0].url }}` (expression)  
    - Connect Get image link → Download image node.

11. **Add Edit Image Node ("resize image")**  
    - Operation: Resize  
    - Width: 800 pixels  
    - Connect Download image → resize image node.

12. **Add HTTP Request Node ("Get company's logo")**  
    - URL: direct Google Drive download link or public URL of logo  
    - Connect resize image → Get company's logo node.

13. **Add Edit Image Node ("resize logo")**  
    - Operation: Resize  
    - Width: 68.4 px  
    - Height: 52.6 px  
    - Connect Get company's logo → resize logo node.

14. **Add Merge Node ("Merge2")**  
    - Connect resize image → Merge2 (input 1)  
    - Connect resize logo → Merge2 (input 2)

15. **Add Code Node ("Merge Binary Items")**  
    - JavaScript code:  
      ```javascript
      const images = $input.all().reduce((acc, item, index) => {
        acc[`data${index}`] = item.binary.data;
        return acc;
      }, {});
      return [{ json: {}, binary: images }];
      ```  
    - Connect Merge2 → Merge Binary Items.

16. **Add Edit Image Node ("Composite image and logo")**  
    - Operation: Composite  
    - PositionX: 728  
    - Data property names:  
      - Base image: `data0`  
      - Composite image (logo): `data1`  
    - Connect Merge Binary Items → Composite image and logo.

17. **Add Google Drive Node ("Save on drive")**  
    - Operation: Upload file  
    - Filename: Use expression like `={{ $('Loop Over Items').first().json.ten }}` or replace with your own field.  
    - Folder ID: Set your Google Drive folder ID to save image.  
    - Input field name: `data0` (composited image binary)  
    - Connect Composite image and logo → Save on drive.  
    - Set up Google Drive OAuth2 credentials.

---

### 5. General Notes & Resources

| Note Content                                                                                                      | Context or Link                                                                                                  |
|------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Workflow inputs: `subject` (topic) and `image_url` must be updated for your use case.                            | Input variables in "Link image" node.                                                                           |
| Ensure OpenAI GPT-4o API credentials are configured correctly in the "Create prompt for generate image" node.    | OpenAI API setup.                                                                                                |
| FAL API keys must be set in all HTTP Request nodes accessing FAL Flux endpoints (`generate image`, `check image finish`, `Get image link`). | FAL Flux API key placement.                                                                                       |
| Google Drive OAuth2 credentials and folder ID must be set to save images properly.                               | Google Drive node configuration.                                                                                 |
| Logo URL should be a publicly accessible direct download link from Google Drive or any image hosting service.    | "Get company's logo" node URL configuration.                                                                     |
| Final image dimensions fixed at 800×500 pixels for typical web article hero images; adjust as needed.            | Resize parameters in "resize image" node.                                                                        |
| The GPT prompt explicitly forbids text or logos in the generated image to avoid artifacts.                       | Prompt text in "Create prompt for generate image".                                                               |
| Polling interval is 2 seconds in the Wait node; increase if API rate limits are hit or decrease for faster updates. | "Wait" node configuration.                                                                                        |
| If the status never reaches `COMPLETED`, consider adding timeout and error handling branches to prevent infinite loops. | Suggested improvement for robustness.                                                                             |
| Branding: Logo is composited at X=728 pixels horizontally on an 800px wide image, size 68.4×52.6 px. Adjust for your layout. | "resize logo" and "Composite image and logo" node parameters.                                                    |

---

**Disclaimer:**  
The content is extracted exclusively from an automated n8n workflow. All data handled is legal and public. This document does not contain any illegal, offensive, or protected content.