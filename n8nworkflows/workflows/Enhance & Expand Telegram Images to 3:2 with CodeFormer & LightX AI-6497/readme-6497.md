Enhance & Expand Telegram Images to 3:2 with CodeFormer & LightX AI

https://n8nworkflows.xyz/workflows/enhance---expand-telegram-images-to-3-2-with-codeformer---lightx-ai-6497


# Enhance & Expand Telegram Images to 3:2 with CodeFormer & LightX AI

### 1. Workflow Overview

This workflow automates the process of enhancing and expanding images received via Telegram to a 3:2 aspect ratio using AI services CodeFormer and LightX AI. It is designed for users who want to upscale and pad images intelligently, ensuring improved quality and a specific format suitable for further use or sharing.

The workflow includes the following logical blocks:

- **1.1 Input Reception and Validation:** Receives images from Telegram users and checks if the image already exists.
- **1.2 AI Upscaling (Replicate):** Upscales the original image using the Replicate AI service.
- **1.3 Image Padding Calculation:** Calculates padding required to convert the image to a 3:2 aspect ratio.
- **1.4 Upload and Expansion Request to LightX AI:** Uploads the padded image to LightX, requests AI-based image expansion, and polls for completion.
- **1.5 Final Image Download and User Notification:** Downloads the final expanded image, uploads it to AWS S3, generates a public URL, and sends the URL back to the user via Telegram.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception and Validation

**Overview:**  
This block captures images sent via Telegram, verifies if the photo file already exists, and prepares input URLs and file names for processing.

**Nodes Involved:**  
- Telegram Trigger1  
- Check If Photo Exists1  
- Generate Input File Name (.jpeg)  
- Upload Original Photo to S1  
- Generate Public URL for Original Image  
- Set Input URL for AI Model

**Node Details:**

- **Telegram Trigger1**  
  - *Type:* Telegram Trigger  
  - *Role:* Entry point that triggers workflow on incoming Telegram media messages.  
  - *Config:* Uses webhook to listen for Telegram image messages.  
  - *Inputs:* External Telegram message.  
  - *Outputs:* Passes message data downstream.  
  - *Failure cases:* Telegram API downtime, webhook misconfiguration.

- **Check If Photo Exists1**  
  - *Type:* IF  
  - *Role:* Checks if the image file already exists in storage to avoid duplicate processing.  
  - *Config:* Conditional check on file existence metadata or S3 presence.  
  - *Inputs:* Telegram Trigger1 output.  
  - *Outputs:* Two branches - "true" for existing file, "false" for new.  
  - *Edge cases:* False negatives if S3 indexing delayed.

- **Generate Input File Name (.jpeg)**  
  - *Type:* Set  
  - *Role:* Generates a sanitized, standardized filename for the original input image.  
  - *Config:* Sets variables for file naming convention ending with `.jpeg`.  
  - *Inputs:* Check If Photo Exists1 (false branch).  
  - *Outputs:* Filename for upload.  
  - *Edge cases:* Filename collisions or invalid characters.

- **Upload Original Photo to S1**  
  - *Type:* AWS S3  
  - *Role:* Uploads the original image to an AWS S3 bucket for persistent storage and later reference.  
  - *Config:* Uses AWS S3 credentials, bucket name, and key from the filename node.  
  - *Inputs:* Generate Input File Name (.jpeg).  
  - *Outputs:* Upload metadata including URL or key.  
  - *Failure types:* AWS authentication errors, network timeouts.

- **Generate Public URL for Original Image**  
  - *Type:* Set  
  - *Role:* Constructs a publicly accessible URL for the uploaded original image.  
  - *Config:* Combines S3 bucket URL with file path.  
  - *Inputs:* Upload Original Photo to S1 output.  
  - *Outputs:* Public URL variable.  
  - *Edge cases:* Incorrect URL formatting.

- **Set Input URL for AI Model**  
  - *Type:* Set  
  - *Role:* Sets the input URL that will be sent to the AI upscaling model.  
  - *Config:* Uses the public URL generated for the original image.  
  - *Inputs:* Generate Public URL for Original Image.  
  - *Outputs:* Passes input URL downstream.  

---

#### 1.2 AI Upscaling (Replicate)

**Overview:**  
This block sends the original image URL to Replicate’s AI upscaling model, waits for the process, and downloads the upscaled image.

**Nodes Involved:**  
- Generate Output File Name (.png)  
- AI Upscale Image via Replicate  
- Wait After Upscale  
- Download Upscaled Image from Replicate1

**Node Details:**

- **Generate Output File Name (.png)**  
  - *Type:* Set  
  - *Role:* Generates the output filename for the upscaled image, ending with `.png`.  
  - *Config:* Sets filename variables accordingly.  
  - *Input:* Set Input URL for AI Model.  
  - *Output:* Filename for AI upscaled image.

- **AI Upscale Image via Replicate**  
  - *Type:* HTTP Request  
  - *Role:* Calls Replicate API to upscale the image at the input URL.  
  - *Config:* Uses API credentials, sends POST with image URL, model parameters.  
  - *Input:* Generate Output File Name (.png).  
  - *Output:* Task ID or job info for upscaling.  
  - *Failure types:* API rate limit, invalid URL, network errors.

- **Wait After Upscale**  
  - *Type:* Wait  
  - *Role:* Delays processing to allow AI upscale time to complete.  
  - *Config:* Fixed wait time or dynamic polling interval.  
  - *Input:* AI Upscale Image via Replicate output.  
  - *Output:* Triggers next node after wait.

- **Download Upscaled Image from Replicate1**  
  - *Type:* HTTP Request  
  - *Role:* Retrieves the upscaled image from Replicate once ready.  
  - *Config:* GET request to Replicate result URL, with retry logic on failure.  
  - *Input:* Wait After Upscale output.  
  - *Output:* Binary image data for further processing.  
  - *Edge cases:* Timeout if upscale not ready, invalid response.

---

#### 1.3 Image Padding Calculation

**Overview:**  
Calculates padding needed to convert the upscaled image to a 3:2 aspect ratio, preparing data for the next image expansion step.

**Nodes Involved:**  
- Calculate Padding for 3:2 Ratio  
- Get File Size (in Bytes)  

**Node Details:**

- **Calculate Padding for 3:2 Ratio**  
  - *Type:* Code  
  - *Role:* Custom JavaScript code calculates the padding dimensions required for a 3:2 aspect ratio based on image dimensions.  
  - *Config:* Runs once per image, computes padding on width or height.  
  - *Input:* Download Upscaled Image from Replicate1.  
  - *Output:* Padding dimensions and related metadata.  
  - *Edge cases:* Non-standard image dimensions, zero or negative padding.

- **Get File Size (in Bytes)**  
  - *Type:* Code  
  - *Role:* Calculates the binary size of the upscaled image file.  
  - *Config:* Reads binary data length.  
  - *Input:* Download Upscaled Image from Replicate1.  
  - *Output:* File size numeric value.  

---

#### 1.4 Upload and Expansion Request to LightX AI

**Overview:**  
Uploads the padded image to LightX AI, requests an AI expansion to fill the padding, and iteratively checks expansion status until completion.

**Nodes Involved:**  
- Extract Upload Link from LightX Response  
- Request Upload Link from LightX  
- Merge Data for Upload  
- Upload File to LightX via PUT  
- Request AI Expansion from LightX  
- Iterate Until Expansion is Ready  
- Wait Before Checking Status Again  
- Check LightX Expansion Status  
- Check If Expansion Is Complete  
- Generate Output Filename (.jpeg) Final

**Node Details:**

- **Request Upload Link from LightX**  
  - *Type:* HTTP Request  
  - *Role:* Requests a pre-signed upload URL from LightX for the image upload.  
  - *Config:* Authenticated API call with file size and metadata.  
  - *Input:* Get File Size (in Bytes).  
  - *Output:* Upload URL and parameters.

- **Extract Upload Link from LightX Response**  
  - *Type:* Set  
  - *Role:* Extracts the actual pre-signed URL from the JSON response for file upload.  
  - *Input:* Request Upload Link from LightX.  
  - *Output:* Upload URL string.

- **Merge Data for Upload**  
  - *Type:* Merge  
  - *Role:* Combines image binary data, upload URL, and padding data into one payload for upload.  
  - *Input:* Calculate Padding for 3:2 Ratio, Extract Upload Link from LightX Response, Download Upscaled Image from Replicate1.  
  - *Output:* Complete upload payload.

- **Upload File to LightX via PUT**  
  - *Type:* HTTP Request  
  - *Role:* Uploads the padded image binary to LightX using the pre-signed URL via HTTP PUT.  
  - *Input:* Merge Data for Upload.  
  - *Output:* Upload confirmation response.  
  - *Failure types:* Network errors, expired URLs.

- **Request AI Expansion from LightX**  
  - *Type:* HTTP Request  
  - *Role:* Initiates the AI-based image expansion process on LightX.  
  - *Input:* Upload File to LightX via PUT.  
  - *Output:* Expansion task ID or status.

- **Iterate Until Expansion is Ready**  
  - *Type:* Split In Batches  
  - *Role:* Controls the iteration loop for polling expansion status.  
  - *Input:* Request AI Expansion from LightX.  
  - *Output:* Loops through status checks or proceeds when ready.

- **Wait Before Checking Status Again**  
  - *Type:* Wait  
  - *Role:* Delays between polling attempts to avoid rate limits or excessive requests.  
  - *Input:* Iterate Until Expansion is Ready (false branch).  
  - *Output:* Triggers status check.

- **Check LightX Expansion Status**  
  - *Type:* HTTP Request  
  - *Role:* Polls LightX API for current status of the image expansion task.  
  - *Input:* Wait Before Checking Status Again.  
  - *Output:* Current expansion status.

- **Check If Expansion Is Complete**  
  - *Type:* IF  
  - *Role:* Branches workflow depending on whether LightX expansion has completed successfully.  
  - *Input:* Check LightX Expansion Status.  
  - *Output:* True branch for completion, false branch to continue polling.

- **Generate Output Filename (.jpeg) Final**  
  - *Type:* Set  
  - *Role:* Sets the filename for the final expanded image for consistent storage and retrieval.  
  - *Input:* Check If Expansion Is Complete (true branch).  
  - *Output:* Filename for final image download.

---

#### 1.5 Final Image Download and User Notification

**Overview:**  
Downloads the final expanded image from LightX, uploads it to AWS S3, generates a public URL, and sends that URL back to the Telegram user.

**Nodes Involved:**  
- Download Final Photo from LightX  
- Upload Final Photo to S1  
- Generate Public URL for Final Image  
- Send Final URL to User via Telegram

**Node Details:**

- **Download Final Photo from LightX**  
  - *Type:* HTTP Request  
  - *Role:* Downloads the final expanded image binary data from LightX API.  
  - *Input:* Generate Output Filename (.jpeg) Final.  
  - *Output:* Binary image data for final upload.

- **Upload Final Photo to S1**  
  - *Type:* AWS S3  
  - *Role:* Uploads the final expanded image to AWS S3 bucket for permanent storage.  
  - *Input:* Download Final Photo from LightX.  
  - *Output:* Upload metadata including URL.

- **Generate Public URL for Final Image**  
  - *Type:* Set  
  - *Role:* Constructs a public-facing URL for the final expanded image stored in S3.  
  - *Input:* Upload Final Photo to S1.  
  - *Output:* URL string.

- **Send Final URL to User via Telegram**  
  - *Type:* Telegram  
  - *Role:* Sends a message back to the Telegram user containing the URL to the final expanded image.  
  - *Input:* Generate Public URL for Final Image.  
  - *Output:* Confirmation of message sent.  
  - *Failure cases:* Telegram API errors, user blocking bot.

---

### 3. Summary Table

| Node Name                         | Node Type          | Functional Role                         | Input Node(s)                            | Output Node(s)                            | Sticky Note                             |
|----------------------------------|--------------------|---------------------------------------|-----------------------------------------|------------------------------------------|---------------------------------------|
| Telegram Trigger1                 | Telegram Trigger   | Receive image input from Telegram     | -                                       | Check If Photo Exists1                    |                                       |
| Check If Photo Exists1            | IF                 | Check if image already exists          | Telegram Trigger1                       | Set Input URL for AI Model, Generate Input File Name (.jpeg) |                                       |
| Generate Input File Name (.jpeg) | Set                | Generate filename for original image  | Check If Photo Exists1 (false)          | Upload Original Photo to S1               |                                       |
| Upload Original Photo to S1       | AWS S3             | Upload original image to S3            | Generate Input File Name (.jpeg)         | Generate Public URL for Original Image   |                                       |
| Generate Public URL for Original Image | Set          | Create public URL for original image  | Upload Original Photo to S1              | Set Input URL for AI Model                |                                       |
| Set Input URL for AI Model        | Set                | Prepare input URL for AI upscale       | Generate Public URL for Original Image  | Generate Output File Name (.png)          |                                       |
| Generate Output File Name (.png)  | Set                | Generate output filename for upscale   | Set Input URL for AI Model               | AI Upscale Image via Replicate            |                                       |
| AI Upscale Image via Replicate    | HTTP Request       | Call Replicate AI for image upscaling | Generate Output File Name (.png)         | Wait After Upscale                        |                                       |
| Wait After Upscale                | Wait               | Wait for AI upscale to complete        | AI Upscale Image via Replicate           | Download Upscaled Image from Replicate1   |                                       |
| Download Upscaled Image from Replicate1 | HTTP Request | Download upscaled image                 | Wait After Upscale                       | Calculate Padding for 3:2 Ratio, Get File Size (in Bytes), Merge Data for Upload |                                       |
| Calculate Padding for 3:2 Ratio   | Code               | Calculate padding to 3:2 aspect ratio  | Download Upscaled Image from Replicate1 | Merge Data for Upload                     |                                       |
| Get File Size (in Bytes)          | Code               | Calculate binary file size              | Download Upscaled Image from Replicate1 | Request Upload Link from LightX           |                                       |
| Request Upload Link from LightX   | HTTP Request       | Request pre-signed upload URL           | Get File Size (in Bytes)                 | Extract Upload Link from LightX Response  |                                       |
| Extract Upload Link from LightX Response | Set          | Extract upload URL from response        | Request Upload Link from LightX          | Merge Data for Upload                     |                                       |
| Merge Data for Upload             | Merge              | Combine data for upload                  | Calculate Padding for 3:2 Ratio, Extract Upload Link from LightX Response, Download Upscaled Image from Replicate1 | Upload File to LightX via PUT             |                                       |
| Upload File to LightX via PUT     | HTTP Request       | Upload image to LightX                   | Merge Data for Upload                    | Request AI Expansion from LightX          |                                       |
| Request AI Expansion from LightX  | HTTP Request       | Request AI expansion processing         | Upload File to LightX via PUT            | Iterate Until Expansion is Ready          |                                       |
| Iterate Until Expansion is Ready  | Split In Batches   | Loop to check expansion status           | Request AI Expansion from LightX         | (empty), Wait Before Checking Status Again |                                       |
| Wait Before Checking Status Again | Wait               | Delay between status polls               | Iterate Until Expansion is Ready         | Check LightX Expansion Status             |                                       |
| Check LightX Expansion Status     | HTTP Request       | Poll status of AI expansion              | Wait Before Checking Status Again         | Check If Expansion Is Complete            |                                       |
| Check If Expansion Is Complete    | IF                 | Determine if expansion finished          | Check LightX Expansion Status             | Generate Output Filename (.jpeg) Final, Iterate Until Expansion is Ready |                                       |
| Generate Output Filename (.jpeg) Final | Set            | Generate filename for final image        | Check If Expansion Is Complete (true)    | Download Final Photo from LightX          |                                       |
| Download Final Photo from LightX  | HTTP Request       | Download final expanded image            | Generate Output Filename (.jpeg) Final   | Upload Final Photo to S1                   |                                       |
| Upload Final Photo to S1          | AWS S3             | Upload final image to S3                  | Download Final Photo from LightX          | Generate Public URL for Final Image       |                                       |
| Generate Public URL for Final Image | Set               | Create public URL for final image         | Upload Final Photo to S1                   | Send Final URL to User via Telegram        |                                       |
| Send Final URL to User via Telegram | Telegram          | Send final image URL to Telegram user    | Generate Public URL for Final Image       | -                                        |                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Telegram Trigger node**  
   - Type: Telegram Trigger  
   - Configure credentials for Telegram Bot API.  
   - Set to trigger on incoming photo messages.

2. **Add an IF node to check if the photo already exists**  
   - Type: IF  
   - Condition: Check presence in AWS S3 or metadata.  
   - Connect Telegram Trigger output to this node.

3. **Create a Set node to generate input filename (.jpeg)**  
   - Type: Set  
   - Set a filename variable using Telegram message ID or timestamp.  
   - Connect IF node’s false output to this node.

4. **Create AWS S3 node to upload original photo**  
   - Type: AWS S3  
   - Configure AWS credentials and target bucket.  
   - Use filename from previous Set node as key.  
   - Connect Set node output to this node.

5. **Create Set node to generate public URL for original image**  
   - Type: Set  
   - Combine S3 bucket URL and uploaded key to form public URL.  
   - Connect AWS S3 output to this node.

6. **Create Set node to set input URL for AI model**  
   - Type: Set  
   - Pass public URL as input for AI upscaling.  
   - Connect previous Set node output here.

7. **Create Set node to generate output filename (.png) for upscaled image**  
   - Type: Set  
   - Generate filename with `.png` extension.  
   - Connect AI model input URL Set node output here.

8. **Create HTTP Request node to call Replicate AI upscale API**  
   - Type: HTTP Request  
   - Configure with Replicate API credentials and endpoint.  
   - Pass input URL and other parameters as JSON body.  
   - Connect output from output filename Set node here.

9. **Create a Wait node to pause for processing**  
   - Type: Wait  
   - Set appropriate delay (e.g., 30 seconds).  
   - Connect output of Replicate HTTP Request node.

10. **Create HTTP Request node to download upscaled image**  
    - Type: HTTP Request  
    - Use GET to download upscaled image from Replicate.  
    - Implement retry with delay if needed.  
    - Connect Wait node output here.

11. **Create Code node to calculate padding for 3:2 aspect ratio**  
    - Type: Code  
    - Input: Image metadata/dimensions.  
    - Output: Padding dimensions.  
    - Connect output from download upscaled image node.

12. **Create Code node to get file size in bytes**  
    - Type: Code  
    - Calculate size from binary data.  
    - Connect download upscaled image node output here.

13. **Create HTTP Request node to request upload link from LightX**  
    - Type: HTTP Request  
    - Configure LightX API credentials and endpoint.  
    - Pass file size and metadata.  
    - Connect output from file size Code node.

14. **Create Set node to extract upload URL from LightX response**  
    - Type: Set  
    - Extract pre-signed URL string.  
    - Connect LightX upload link request output here.

15. **Create Merge node to combine image data, padding info, and upload URL**  
    - Type: Merge  
    - Combine outputs from padding calculation, upload link extraction, and upscaled image download.  
    - Connect respective outputs here.

16. **Create HTTP Request node to upload file to LightX via PUT**  
    - Type: HTTP Request  
    - Use PUT method with pre-signed URL.  
    - Upload image binary data.  
    - Connect Merge node output here.

17. **Create HTTP Request node to request AI expansion on LightX**  
    - Type: HTTP Request  
    - Start expansion job with LightX API.  
    - Connect upload file node output here.

18. **Create Split In Batches node to control polling iterations**  
    - Type: Split In Batches  
    - Manage polling loop for expansion status.  
    - Connect expansion request output here.

19. **Create Wait node for polling delay**  
    - Type: Wait  
    - Delay between status checks (e.g., 10 seconds).  
    - Connect false output of Split In Batches node.

20. **Create HTTP Request node to check LightX expansion status**  
    - Type: HTTP Request  
    - Poll status endpoint with expansion job ID.  
    - Connect Wait node output.

21. **Create IF node to check if expansion is complete**  
    - Type: IF  
    - Condition: Expansion status equals completed.  
    - Connect LightX status check output.

22. **Create Set node to generate final output filename (.jpeg)**  
    - Type: Set  
    - Generate filename for expanded image.  
    - Connect IF node true output.

23. **Create HTTP Request node to download final expanded image**  
    - Type: HTTP Request  
    - Download binary image data from LightX.  
    - Connect final filename Set node output.

24. **Create AWS S3 node to upload final photo**  
    - Type: AWS S3  
    - Upload final expanded image to S3 bucket.  
    - Connect download final photo node output.

25. **Create Set node to generate public URL for final image**  
    - Type: Set  
    - Construct public URL for final image.  
    - Connect AWS S3 upload output.

26. **Create Telegram node to send final URL to user**  
    - Type: Telegram  
    - Configure with Telegram credentials.  
    - Send message with final image URL.  
    - Connect public URL Set node output.

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                    |
|------------------------------------------------------------------------------|---------------------------------------------------|
| The workflow uses OpenAI and Outlook OAuth2 credentials (if needed) for AI and email integrations. | Credential configuration in n8n settings.         |
| For Telegram bot setup and webhook configuration, see https://core.telegram.org/bots/api | Telegram Bot API official documentation.           |
| LightX AI API requires authentication and pre-signed upload URL handling.    | LightX API docs (private or public depending on provider). |
| Replicate API usage requires a valid API token and adherence to rate limits. | https://replicate.com/docs/api-reference           |
| The aspect ratio padding calculation ensures images conform to 3:2 ratio for consistent display or printing. | Custom JavaScript code node.                        |
| Retry logic on HTTP requests with waits helps handle asynchronous AI processing delays. | n8n Wait node and HTTP Request retry parameters.  |

---

This document fully describes the structure, logic, and configuration of the "Enhance & Expand Telegram Images to 3:2 with CodeFormer & LightX AI" workflow, enabling reproduction, modification, and troubleshooting by users or automation agents.