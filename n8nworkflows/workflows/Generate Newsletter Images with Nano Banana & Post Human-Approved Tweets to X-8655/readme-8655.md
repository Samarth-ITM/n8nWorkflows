Generate Newsletter Images with Nano Banana & Post Human-Approved Tweets to X

https://n8nworkflows.xyz/workflows/generate-newsletter-images-with-nano-banana---post-human-approved-tweets-to-x-8655


# Generate Newsletter Images with Nano Banana & Post Human-Approved Tweets to X

### 1. Workflow Overview

This n8n workflow automates the generation of newsletter images using the Nano Banana API, crafts tweet text with OpenAI language models, and manages human approval before posting tweets with images to X (formerly Twitter). It targets content creators and marketers who want to streamline content creation, approval, and social media posting with AI assistance integrated with Google Sheets and Telegram for approval workflows.

The workflow is logically divided into these key functional blocks:

- **1.1 Input Reception and Triggering**: Listens for new input data from Google Sheets to start the process.
- **1.2 AI Prompt and Image Generation**: Uses OpenAI models to generate prompts, then sends them to the Nano Banana image generation API.
- **1.3 Image Retrieval and Validation**: Waits for image generation completion, fetches the image, and checks if it was created successfully.
- **1.4 Tweet Generation and Approval Workflow**: Generates tweet text, sends both image and tweet text to Telegram for human approval, and branches based on approval status.
- **1.5 Posting to X and Logging**: Downloads approved images, posts tweets with media to X, and logs results back to Google Sheets.
- **1.6 Error Handling and Re-processing**: Handles failures in image download or tweet posting gracefully, continuing workflow without blocking.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Triggering

**Overview:**  
This block initiates the workflow when new data is added or updated in Google Sheets, signaling new newsletter content requiring image and tweet generation.

**Nodes Involved:**  
- Google Sheets Trigger

**Node Details:**

- **Google Sheets Trigger**  
  - Type: Trigger node for Google Sheets  
  - Role: Watches a specified Google Sheet for new or updated rows to trigger the workflow.  
  - Configuration: Uses OAuth2 credentials for Google Sheets; monitors a specific spreadsheet and sheet (details set in parameters).  
  - Input: External event from Google Sheets.  
  - Output: Data row triggering subsequent prompt generation.  
  - Edge cases: Authorization failures on expired tokens, sheet access permissions, empty or malformed rows.  
  - Notes: This is an entry point; no input connections.

---

#### 2.2 AI Prompt and Image Generation

**Overview:**  
Generates creative prompts using OpenAI chat models, then requests the Nano Banana API to generate corresponding newsletter images.

**Nodes Involved:**  
- OpenAI Chat Model  
- Prompt Generator  
- Image Gen (Nano Banana)  
- Wait1

**Node Details:**

- **OpenAI Chat Model**  
  - Type: LangChain OpenAI Chat Model  
  - Role: Generates text prompts for image creation based on input data.  
  - Configuration: Uses OpenAI API credentials; configured with prompt templates and relevant parameters.  
  - Input: Trigger data from Google Sheets.  
  - Output: Text prompt for image generation.  
  - Edge cases: API rate limits, invalid prompts, network timeouts.

- **Prompt Generator**  
  - Type: LangChain Agent  
  - Role: Processes OpenAI output and prepares final prompt for image generation.  
  - Configuration: Contains instructions or agent logic for prompt refinement.  
  - Input: Output from OpenAI Chat Model.  
  - Output: Final prompt string.  

- **Image Gen (Nano Banana)**  
  - Type: HTTP Request  
  - Role: Sends prompt to Nano Banana image generation API to create an image.  
  - Configuration: HTTP POST with authentication to Nano Banana API; payload includes prompt from Prompt Generator.  
  - Input: Prompt string.  
  - Output: Image generation job ID or immediate response.  
  - Edge cases: API errors, network failures, malformed requests.

- **Wait1**  
  - Type: Wait  
  - Role: Delays next steps allowing time for image generation to complete asynchronously.  
  - Configuration: Fixed or dynamic wait time (exact duration unspecified).  
  - Input: Request sent to Nano Banana.  
  - Output: Triggers next node after wait.  
  - Edge cases: Too short wait times resulting in incomplete image, too long causing delays.

---

#### 2.3 Image Retrieval and Validation

**Overview:**  
After waiting, this block retrieves the generated image, verifies its existence, and proceeds based on success or failure.

**Nodes Involved:**  
- Get Image  
- Image created (IF node)  
- Add Image URL

**Node Details:**

- **Get Image**  
  - Type: HTTP Request  
  - Role: Polls or fetches the generated image from Nano Banana API after waiting.  
  - Configuration: HTTP GET request with job ID or image URL; uses appropriate headers/authentication.  
  - Input: Trigger from Wait1.  
  - Output: Image data or error info.  
  - Edge cases: Image not ready, 404 errors, API timeouts.

- **Image created (If node)**  
  - Type: If condition  
  - Role: Checks if the image was successfully created and is valid.  
  - Configuration: Condition based on HTTP status, presence of image URL, or response data.  
  - Input: Output from Get Image.  
  - Output: Routes workflow on success or failure.  
  - Edge cases: False negatives if API returns unexpected format.

- **Add Image URL**  
  - Type: Google Sheets  
  - Role: Updates the Google Sheet row with the URL of the generated image.  
  - Configuration: Uses Google Sheets credentials; selects spreadsheet and sheet; updates specific row and cell with image URL.  
  - Input: Validated image URL.  
  - Output: Confirmation of update.

---

#### 2.4 Tweet Generation and Approval Workflow

**Overview:**  
Generates tweet content using AI, sends both tweet text and image to Telegram for human approval, and evaluates approval decision to proceed or log rejection.

**Nodes Involved:**  
- Tweet Generator  
- Download Image  
- Send Tweet image for approval  
- Send Tweet text for approval  
- Approved (If node)  
- Tweet not approved

**Node Details:**

- **Tweet Generator**  
  - Type: LangChain Agent  
  - Role: Generates tweet text content based on image and prompt data.  
  - Configuration: Uses OpenAI API; configured with tweet style instructions.  
  - Input: Data from Image created node.  
  - Output: Tweet text.

- **Download Image**  
  - Type: HTTP Request  
  - Role: Downloads the generated image file from the provided URL for inclusion in tweet approval and posting.  
  - Configuration: HTTP GET with image URL; continues on error to prevent blockage.  
  - Input: Image URL from previous block.  
  - Output: Image binary data.

- **Send Tweet image for approval**  
  - Type: Telegram node  
  - Role: Sends the image via Telegram bot for human approval.  
  - Configuration: Telegram credentials and chat ID; sends photo with caption or context.  
  - Input: Downloaded image binary data.  
  - Output: Confirmation of message sent.

- **Send Tweet text for approval**  
  - Type: Telegram node  
  - Role: Sends the tweet text via Telegram bot for human approval.  
  - Configuration: Same Telegram credentials; sends message text.  
  - Input: Tweet text from Tweet Generator.  
  - Output: Confirmation of message sent.

- **Approved (If node)**  
  - Type: If condition node  
  - Role: Evaluates approval response from Telegram to determine if tweet should be posted.  
  - Configuration: Condition based on Telegram webhook response or user input.  
  - Input: Approval response.  
  - Output: Routes to posting or logging rejection.

- **Tweet not approved**  
  - Type: Google Sheets  
  - Role: Logs rejected tweets in Google Sheets for record keeping.  
  - Configuration: Updates spreadsheet with rejection details.  
  - Input: Data from Approved node when false.  
  - Output: Confirmation of logging.

---

#### 2.5 Posting to X and Logging

**Overview:**  
Downloads the image again if approved, posts the tweet with media to X via Twitter API, and records successful postings in Google Sheets.

**Nodes Involved:**  
- Downlaod Image again  
- post media  
- Create Tweet  
- Tweet posted

**Node Details:**

- **Downlaod Image again**  
  - Type: HTTP Request  
  - Role: Downloads image again before posting, ensuring media is accessible.  
  - Configuration: HTTP GET; continues on error to not block posting.  
  - Input: Approved branch from "Approved" node.  
  - Output: Image binary data for upload.

- **post media**  
  - Type: HTTP Request  
  - Role: Uploads media image to X (Twitter) API, obtaining media ID for tweet.  
  - Configuration: Uses Twitter OAuth credentials; multipart/form-data POST; handles media upload endpoint.  
  - Input: Image binary data from download.  
  - Output: Media ID on success.

- **Create Tweet**  
  - Type: Twitter node  
  - Role: Posts tweet text with attached media ID to X using Twitter API.  
  - Configuration: Twitter OAuth2 credentials configured; text and media ID parameters.  
  - Input: Tweet text and media ID.  
  - Output: Confirmation of posted tweet.

- **Tweet posted**  
  - Type: Google Sheets  
  - Role: Logs successful tweets into Google Sheets, updating status and metadata.  
  - Configuration: Spreadsheet update with tweet ID, timestamp, or status.  
  - Input: Confirmation from Create Tweet node.  
  - Output: Confirmation of logging.

---

#### 2.6 Error Handling and Notes

**Overview:**  
Certain HTTP Request nodes like image downloads and media uploads have error handling configured to continue workflow without failing, allowing for retries or logging downstream.

**Nodes Involved:**  
- Download Image  
- Downlaod Image again  
- post media

**Node Details:**

- These nodes use "continue on error" or "onError: continueRegularOutput" to ensure workflow resilience.

---

### 3. Summary Table

| Node Name                  | Node Type                    | Functional Role                           | Input Node(s)                  | Output Node(s)                    | Sticky Note                                                                                              |
|----------------------------|------------------------------|-----------------------------------------|-------------------------------|----------------------------------|---------------------------------------------------------------------------------------------------------|
| Google Sheets Trigger       | Google Sheets Trigger         | Workflow entry on new sheet data        | -                             | Prompt Generator                 |                                                                                                         |
| OpenAI Chat Model           | LangChain OpenAI Chat Model  | Generate initial AI prompt               | Google Sheets Trigger          | Prompt Generator                 |                                                                                                         |
| Prompt Generator            | LangChain Agent              | Prepare prompt for image generation      | OpenAI Chat Model              | Image Gen (Nano Banana)          |                                                                                                         |
| Image Gen (Nano Banana)     | HTTP Request                 | Send prompt to Nano Banana API           | Prompt Generator              | Wait1                           |                                                                                                         |
| Wait1                      | Wait                         | Pause workflow for image generation      | Image Gen (Nano Banana)        | Get Image                       |                                                                                                         |
| Get Image                  | HTTP Request                 | Retrieve generated image                  | Wait1                         | Image created                   |                                                                                                         |
| Image created              | If                           | Verify image creation success             | Get Image                     | Add Image URL, Tweet Generator, Wait1 |                                                                                                         |
| Add Image URL              | Google Sheets                | Update sheet with image URL                | Image created                 | -                              |                                                                                                         |
| Tweet Generator            | LangChain Agent              | Generate tweet text                        | Image created                 | Download Image                  |                                                                                                         |
| Download Image             | HTTP Request                 | Download image for approval                | Tweet Generator               | Send Tweet image for approval   |                                                                                                         |
| Send Tweet image for approval | Telegram                    | Send image via Telegram for approval       | Download Image                | Send Tweet text for approval    |                                                                                                         |
| Send Tweet text for approval | Telegram                    | Send tweet text via Telegram for approval | Send Tweet image for approval | Approved                       |                                                                                                         |
| Approved                   | If                           | Check if tweet approved                    | Send Tweet text for approval  | Downlaod Image again, Tweet not approved |                                                                                                         |
| Tweet not approved         | Google Sheets                | Log rejected tweets                        | Approved                     | -                              |                                                                                                         |
| Downlaod Image again       | HTTP Request                 | Download image again for posting           | Approved (approved path)      | post media                     |                                                                                                         |
| post media                 | HTTP Request                 | Upload media to X (Twitter)                 | Downlaod Image again          | Create Tweet                   |                                                                                                         |
| Create Tweet               | Twitter                      | Post tweet with media                       | post media                   | Tweet posted                   |                                                                                                         |
| Tweet posted               | Google Sheets                | Log successful tweet posting                | Create Tweet                 | -                              |                                                                                                         |
| Sticky Note                | Sticky Note                  | Comments or instructions                    | -                             | -                              |                                                                                                         |
| Sticky Note2               | Sticky Note                  | Comments or instructions                    | -                             | -                              |                                                                                                         |
| Sticky Note3               | Sticky Note                  | Comments or instructions                    | -                             | -                              |                                                                                                         |
| Sticky Note4               | Sticky Note                  | Comments or instructions                    | -                             | -                              |                                                                                                         |
| Sticky Note5               | Sticky Note                  | Comments or instructions                    | -                             | -                              |                                                                                                         |
| Sticky Note6               | Sticky Note                  | Comments or instructions                    | -                             | -                              |                                                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets Trigger node:**  
   - Type: Google Sheets Trigger  
   - Configure OAuth2 credentials for Google Sheets.  
   - Set spreadsheet ID and sheet name to watch for new rows.

2. **Add OpenAI Chat Model node:**  
   - Type: LangChain OpenAI Chat Model  
   - Set OpenAI API credentials.  
   - Configure prompt template to generate image prompts based on sheet data.  
   - Connect Google Sheets Trigger output to this node.

3. **Add Prompt Generator node:**  
   - Type: LangChain Agent  
   - Configure agent with instructions refining OpenAI output into final image prompt.  
   - Connect OpenAI Chat Model output here.

4. **Add Image Gen (Nano Banana) node:**  
   - Type: HTTP Request  
   - Set HTTP POST to Nano Banana API endpoint.  
   - Include authentication headers (API key).  
   - Pass prompt from Prompt Generator as request body.  
   - Connect Prompt Generator output here.

5. **Add Wait1 node:**  
   - Type: Wait  
   - Configure wait time (e.g., 30 seconds) to allow image generation.  
   - Connect Image Gen output here.

6. **Add Get Image node:**  
   - Type: HTTP Request  
   - Configure HTTP GET to Nano Banana API to retrieve image by job ID or URL.  
   - Connect Wait1 output here.

7. **Add Image created If node:**  
   - Type: If condition  
   - Condition checking HTTP status code or response data indicates image success.  
   - Connect Get Image output here.

8. **Add Add Image URL node:**  
   - Type: Google Sheets  
   - Configure to update original row with image URL field.  
   - Connect Image created True output here.

9. **Add Tweet Generator node:**  
   - Type: LangChain Agent  
   - Configure to generate tweet text based on image and prompt data.  
   - Connect Image created True output here.

10. **Add Download Image node:**  
    - Type: HTTP Request  
    - Configure HTTP GET for image URL to download image binary.  
    - Set "Continue on Error" enabled.  
    - Connect Tweet Generator output here.

11. **Add Send Tweet image for approval node:**  
    - Type: Telegram  
    - Configure Telegram bot credentials and chat ID.  
    - Send photo with caption or context.  
    - Connect Download Image output here.

12. **Add Send Tweet text for approval node:**  
    - Type: Telegram  
    - Same Telegram credentials.  
    - Send message with tweet text.  
    - Connect Send Tweet image for approval output here.

13. **Add Approved If node:**  
    - Type: If condition  
    - Condition checks Telegram approval response (e.g., reply or button click).  
    - Connect Send Tweet text for approval output here.

14. **Add Tweet not approved node:**  
    - Type: Google Sheets  
    - Configure to log rejected tweets with details.  
    - Connect Approved False output here.

15. **Add Downlaod Image again node:**  
    - Type: HTTP Request  
    - HTTP GET to image URL to re-download binary for posting.  
    - Set "Continue on Error" enabled.  
    - Connect Approved True output here.

16. **Add post media node:**  
    - Type: HTTP Request  
    - Configure Twitter API media upload endpoint with OAuth2 credentials.  
    - Send image binary data as multipart/form-data.  
    - Connect Downlaod Image again output here.

17. **Add Create Tweet node:**  
    - Type: Twitter  
    - Configure with Twitter OAuth2 credentials.  
    - Pass tweet text and media ID from post media.  
    - Connect post media output here.

18. **Add Tweet posted node:**  
    - Type: Google Sheets  
    - Configure to log tweet posting success with metadata.  
    - Connect Create Tweet output here.

---

### 5. General Notes & Resources

| Note Content                                                                                                        | Context or Link                                                                                 |
|---------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow integrates OpenAI models with Nano Banana API for automated creative content generation.                   | Workflow description                                                                            |
| Uses Telegram bot for human-in-the-loop approval of generated tweets before posting to X.                           | Approval mechanism                                                                              |
| Google Sheets is used both as input trigger source and logging destination for audit trail of tweets and images.    | Input and logging strategy                                                                      |
| Twitter node uses OAuth2 credentials; ensure proper app permissions and token refresh setup to avoid auth failures. | Twitter API integration                                                                         |
| Error handling on image download and media upload nodes prevents workflow failure and supports retry logic.        | Robustness consideration                                                                        |
| Official n8n documentation for Telegram node usage: https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.telegram/ | Relevant for configuring Telegram nodes                                                        |
| OpenAI API rate limits and costs should be monitored when scaling this workflow.                                    | Operational consideration                                                                      |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This treatment strictly respects current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.