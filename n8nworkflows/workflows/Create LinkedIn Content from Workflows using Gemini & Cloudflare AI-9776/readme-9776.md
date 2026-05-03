Create LinkedIn Content from Workflows using Gemini & Cloudflare AI

https://n8nworkflows.xyz/workflows/create-linkedin-content-from-workflows-using-gemini---cloudflare-ai-9776


# Create LinkedIn Content from Workflows using Gemini & Cloudflare AI

### 1. Workflow Overview

This n8n workflow automates the generation and publication of LinkedIn content based on n8n workflow JSON inputs, leveraging Google Gemini AI for text and image prompt creation and Cloudflare AI for image generation. Targeted at content creators and automation professionals, the workflow transforms exported n8n workflow JSON files into engaging LinkedIn posts with AI-generated images, streamlining the content creation process.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives n8n workflow JSON files via Telegram and extracts the JSON content.
- **1.2 AI Processing:** Sends extracted JSON to Google Gemini to generate LinkedIn post text, an image prompt, and metadata.
- **1.3 Data Parsing & Logging:** Parses AI output JSON and logs key data to Google Sheets.
- **1.4 Image Generation:** Uses Cloudflare AI to generate a post image based on the AI-generated prompt, with size validation.
- **1.5 User Notification:** Sends the generated post text and image prompt back to the user on Telegram for verification.
- **1.6 LinkedIn Posting:** Listens for user replies with final post text and image, retrieves the image, and publishes the post on LinkedIn.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Listens for Telegram messages containing n8n workflow JSON files, downloads the file, and extracts the JSON content for processing.
- **Nodes Involved:**
  - Receive n8n JSON (Telegram Trigger)
  - Get JSON File from Telegram (Telegram node)
  - Parse Workflow JSON (Extract from File node)

- **Node Details:**

  - **Receive n8n JSON**
    - Type: Telegram Trigger
    - Role: Webhook endpoint to receive Telegram messages
    - Configuration: Listens for `message` updates, uses Telegram API credentials for "Linked In (JSON) Bot"
    - Inputs: Telegram webhook
    - Outputs: Message JSON with file metadata
    - Failure modes: Invalid webhook setup, credential auth errors, Telegram API rate limits
    
  - **Get JSON File from Telegram**
    - Type: Telegram
    - Role: Downloads the JSON file sent in the Telegram message
    - Configuration: Uses `file_id` from incoming message document, same Telegram credentials as above
    - Inputs: Output of Telegram Trigger node
    - Outputs: File content binary
    - Failure modes: File not found, API errors, invalid file type
    
  - **Parse Workflow JSON**
    - Type: Extract From File
    - Role: Parses the downloaded file binary into JSON object
    - Configuration: Operation set to `fromJson`
    - Inputs: File binary from previous node
    - Outputs: Parsed JSON object representing the n8n workflow
    - Failure modes: Malformed JSON file, parse errors

#### 2.2 AI Processing

- **Overview:** Sends the parsed workflow JSON to Google Gemini AI to generate a LinkedIn post, an image prompt, and a unique image filename in a structured JSON response.
- **Nodes Involved:**
  - Gemini: Generate Post & Image Prompt (Google Gemini node)
  - Parse Gemini Output (Code node)

- **Node Details:**

  - **Gemini: Generate Post & Image Prompt**
    - Type: Google Gemini AI node (Langchain integration)
    - Role: AI text generation with a detailed prompt instructing Gemini to create LinkedIn post content plus image generation prompt and metadata in JSON format.
    - Configuration:
      - Model: `models/gemini-2.5-flash`
      - Custom system message defining persona "FlowArchitect"
      - Instructions for two LinkedIn post styles, example input and output, and final task to generate JSON only
      - Input includes the workflow JSON stringified and passed as a variable
      - Uses Google Gemini API credentials
    - Inputs: Parsed workflow JSON from previous block
    - Outputs: Raw AI response text with embedded JSON inside markdown code block
    - Failure modes: API quota exceeded, network issues, malformed prompt response, JSON syntax errors

  - **Parse Gemini Output**
    - Type: Code (JavaScript)
    - Role: Extracts valid JSON from Gemini's markdown output, parses it into a JSON object with keys: post_text, image_prompt, style_used, image_filename, dm_message.
    - Configuration: Custom JS code removing code block markdown and parsing JSON
    - Inputs: Gemini node output
    - Outputs: Clean JSON object with AI-generated content
    - Failure modes: Parsing errors if AI output is malformed or missing expected format

#### 2.3 Data Parsing & Logging

- **Overview:** Logs key AI-generated metadata (keyword, style, image prompt) into a Google Sheets document for tracking and analytics.
- **Nodes Involved:**
  - Log Content to Google Sheet (Google Sheets node)

- **Node Details:**

  - **Log Content to Google Sheet**
    - Type: Google Sheets
    - Role: Appends a new row to a Google Sheet with columns: Keyword, Style Used, Image Prompt
    - Configuration:
      - Sheet ID and name preconfigured
      - Credentials use Google OAuth2 account
      - Mapping dynamically extracts from parsed Gemini JSON output
    - Inputs: Parsed Gemini output JSON object
    - Outputs: Confirmation of append operation
    - Failure modes: OAuth token expiration, permission errors, quota limits

#### 2.4 Image Generation

- **Overview:** Prepares parameters and requests Cloudflare AI to generate an image according to the AI-generated prompt and validates image size constraints.
- **Nodes Involved:**
  - Set Image Generation Parameters (Set node)
  - Validate Image Dimensions (If node)
  - Image > 1 megapixel (Stop and Error node)
  - Cloudflare: Get Account ID (HTTP Request node)
  - Cloudflare: Generate Image (HTTP Request node)
  - Convert Image to Binary (Convert To File node)

- **Node Details:**

  - **Set Image Generation Parameters**
    - Type: Set
    - Role: Sets properties like width, height, prompt, seed, steps for image generation based on AI output or defaults
    - Configuration: Uses expressions to set defaults or values from AI JSON
    - Inputs: Parsed Gemini output JSON
    - Outputs: Parameter object for image generation
    - Failure modes: Missing or invalid input values

  - **Validate Image Dimensions**
    - Type: If
    - Role: Checks if image width * height exceeds 1 megapixel (1,048,576 pixels)
    - Configuration: Boolean expression for size check
    - Inputs: Parameters from Set node
    - Outputs:
      - True branch triggers Error node (Image > 1 megapixel)
      - False branch proceeds to next step
    - Failure modes: Incorrect dimension fields

  - **Image > 1 megapixel**
    - Type: Stop and Error
    - Role: Stops workflow with error message if image size is too large
    - Configuration: Custom error message
    - Inputs: If node True branch
    - Outputs: Workflow stops

  - **Cloudflare: Get Account ID**
    - Type: HTTP Request
    - Role: Retrieves Cloudflare account ID required for image generation API endpoint
    - Configuration:
      - URL: `https://api.cloudflare.com/client/v4/accounts`
      - Authentication: HTTP Bearer and Header Auth with Cloudflare credentials
    - Inputs: If node False branch
    - Outputs: JSON with account IDs

  - **Cloudflare: Generate Image**
    - Type: HTTP Request
    - Role: Sends POST request to Cloudflare AI image generation endpoint with parameters
    - Configuration:
      - URL constructed dynamically using Cloudflare account ID
      - JSON body includes prompt, seed, width, height, steps from previous nodes
      - Auth as above
    - Inputs: Account ID from previous node
    - Outputs: Image data in JSON response

  - **Convert Image to Binary**
    - Type: Convert To File
    - Role: Converts base64 or JSON image data into binary format for Telegram upload
    - Configuration: Reads image from `result.image` property
    - Inputs: Cloudflare image generation output
    - Outputs: Binary image file ready for sending
    - Failure modes: Invalid image data, conversion errors

#### 2.5 User Notification

- **Overview:** Sends the generated LinkedIn post text to the user for verification and sends the generated image and its prompt back via Telegram.
- **Nodes Involved:**
  - Send Post Content For Verification (Telegram node)
  - Send Image & Prompt to User (Telegram node)

- **Node Details:**

  - **Send Post Content For Verification**
    - Type: Telegram
    - Role: Sends AI-generated LinkedIn post text as a message to the user who uploaded the JSON
    - Configuration: Uses chat ID from initial Telegram message, text from parsed Gemini JSON
    - Inputs: Parsed Gemini JSON
    - Outputs: Confirmation of message sent
    - Failure modes: Telegram API errors, invalid chat ID

  - **Send Image & Prompt to User**
    - Type: Telegram
    - Role: Sends the generated AI image as a photo with a caption showing the image prompt
    - Configuration: Uses binary image data, chat ID from initial Telegram message, caption from AI JSON `image_prompt`
    - Inputs: Converted binary image
    - Outputs: Confirmation of photo sent
    - Failure modes: Telegram API limits, image size restrictions

#### 2.6 LinkedIn Posting

- **Overview:** Listens for Telegram replies to the image message containing final post text, downloads the associated image, and publishes the post with image on LinkedIn.
- **Nodes Involved:**
  - Trigger: Reply to Post Image (Telegram Trigger)
  - Get Image from Reply (Telegram node)
  - Publish Post to LinkedIn (LinkedIn node)

- **Node Details:**

  - **Trigger: Reply to Post Image**
    - Type: Telegram Trigger
    - Role: Listens for user reply messages to the previously sent image message
    - Configuration: Listens for `message` updates, uses Linked In Post Bot credentials
    - Inputs: Telegram webhook
    - Outputs: Reply message JSON containing text and reference to image message
    - Failure modes: Timing issues, webhook misconfiguration, bot permissions

  - **Get Image from Reply**
    - Type: Telegram
    - Role: Retrieves the original image file from the replied message to use for LinkedIn post
    - Configuration: Extracts file_id of largest photo from replied message
    - Inputs: Reply message JSON
    - Outputs: Image file binary
    - Failure modes: Missing image, Telegram API errors

  - **Publish Post to LinkedIn**
    - Type: LinkedIn
    - Role: Publishes the final LinkedIn post with the image and text provided by user reply
    - Configuration:
      - Text from reply message
      - Static person ID (LinkedIn profile)
      - Visibility set to "CONNECTIONS"
      - Media category: IMAGE
      - Uses OAuth2 LinkedIn credentials
    - Inputs: Text from reply, image file from previous node
    - Outputs: Post publication confirmation
    - Failure modes: OAuth token expiration, API limits, invalid content, media upload errors

---

### 3. Summary Table

| Node Name                     | Node Type                   | Functional Role                                     | Input Node(s)                 | Output Node(s)                      | Sticky Note                                                                                                           |
|-------------------------------|-----------------------------|----------------------------------------------------|------------------------------|-----------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| Sticky Note                   | Sticky Note                 | Title/branding of workflow                          | —                            | —                                 | ## LinkedIn Post Content and Image Generator                                                                          |
| Sticky Note1                  | Sticky Note                 | Indicates final posting stage                       | —                            | —                                 | ## Finalized Post Content and Image is Posted in LinkedIn                                                              |
| Image > 1 megapixel           | Stop and Error              | Stops workflow if image size exceeds 1 megapixel  | Validate Image Dimensions     | —                                 |                                                                                                                       |
| Sticky Note4                  | Sticky Note                 | Branding, contact info, and tips                    | —                            | —                                 | Turn Your n8n Workflows to LinkedIn Post with a suitable Image. For questions contact: anirudh.n.aeran@gmail.com, etc.  |
| Sticky Note2                  | Sticky Note                 | Detailed user instructions and setup guide         | —                            | —                                 | Try It Out! Detailed instructions to configure credentials and usage steps for the workflow                           |
| Receive n8n JSON             | Telegram Trigger            | Receives Telegram messages with n8n JSON files     | —                            | Get JSON File from Telegram       |                                                                                                                       |
| Get JSON File from Telegram  | Telegram                   | Downloads workflow JSON file from Telegram          | Receive n8n JSON             | Parse Workflow JSON               |                                                                                                                       |
| Parse Workflow JSON          | Extract From File           | Parses downloaded JSON file to JSON object          | Get JSON File from Telegram  | Gemini: Generate Post & Image Prompt |                                                                                                                       |
| Gemini: Generate Post & Image Prompt | Google Gemini AI          | Generates LinkedIn post text and image prompt via AI | Parse Workflow JSON          | Parse Gemini Output               |                                                                                                                       |
| Parse Gemini Output          | Code                       | Parses AI response JSON from Gemini's markdown      | Gemini: Generate Post & Image Prompt | Log Content to Google Sheet, Set Image Generation Parameters, Send Post Content For Verification |                                                                                                                       |
| Log Content to Google Sheet  | Google Sheets              | Logs keywords, style, and image prompt metadata     | Parse Gemini Output          | —                                 |                                                                                                                       |
| Set Image Generation Parameters | Set                       | Sets image generation parameters for Cloudflare AI | Parse Gemini Output          | Validate Image Dimensions         |                                                                                                                       |
| Validate Image Dimensions    | If                         | Checks if image size exceeds 1 megapixel            | Set Image Generation Parameters | Image > 1 megapixel (on true), Cloudflare: Get Account ID (on false) |                                                                                                                       |
| Cloudflare: Get Account ID   | HTTP Request               | Retrieves Cloudflare account ID                      | Validate Image Dimensions    | Cloudflare: Generate Image        |                                                                                                                       |
| Cloudflare: Generate Image   | HTTP Request               | Calls Cloudflare AI image generation endpoint       | Cloudflare: Get Account ID   | Convert Image to Binary           |                                                                                                                       |
| Convert Image to Binary      | Convert To File             | Converts Cloudflare image response to binary file   | Cloudflare: Generate Image   | Send Image & Prompt to User       |                                                                                                                       |
| Send Post Content For Verification | Telegram                   | Sends generated post text to Telegram user          | Parse Gemini Output          | —                                 |                                                                                                                       |
| Send Image & Prompt to User | Telegram                   | Sends generated image and prompt as photo to user   | Convert Image to Binary      | —                                 |                                                                                                                       |
| Trigger: Reply to Post Image | Telegram Trigger            | Listens for user replies to posted images            | —                            | Get Image from Reply              |                                                                                                                       |
| Get Image from Reply         | Telegram                   | Downloads the image replied to by user               | Trigger: Reply to Post Image | Publish Post to LinkedIn          |                                                                                                                       |
| Publish Post to LinkedIn     | LinkedIn                   | Publishes post and image on LinkedIn profile         | Get Image from Reply         | —                                 |                                                                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node ("Receive n8n JSON")**
   - Type: Telegram Trigger
   - Parameters: Listen for `message` updates
   - Credentials: Telegram API for "Linked In (JSON) Bot"
   - Connects to: Get JSON File from Telegram

2. **Create Telegram Node ("Get JSON File from Telegram")**
   - Type: Telegram
   - Parameters: Set `fileId` with expression from incoming message document file_id
   - Credentials: Same Telegram API as above
   - Connects to: Parse Workflow JSON

3. **Create Extract From File Node ("Parse Workflow JSON")**
   - Type: Extract From File
   - Parameters: Operation: fromJson
   - Connects to: Gemini: Generate Post & Image Prompt

4. **Create Google Gemini Node ("Gemini: Generate Post & Image Prompt")**
   - Type: Google Gemini (Langchain)
   - Parameters:
     - Model: `models/gemini-2.5-flash`
     - Messages: Set system prompt including persona, instructions, examples, and input workflow JSON
   - Credentials: Google Gemini API credentials
   - Connects to: Parse Gemini Output

5. **Create Code Node ("Parse Gemini Output")**
   - Type: Code (JavaScript)
   - Parameters: JS code to strip markdown and parse JSON from Gemini output
   - Connects to three nodes: Log Content to Google Sheet, Set Image Generation Parameters, Send Post Content For Verification

6. **Create Google Sheets Node ("Log Content to Google Sheet")**
   - Type: Google Sheets
   - Parameters:
     - Operation: Append
     - Sheet ID and name configured
     - Columns mapped from parsed Gemini output (Keyword, Style Used, Image Prompt)
   - Credentials: Google Sheets OAuth2
   - No further connections needed

7. **Create Set Node ("Set Image Generation Parameters")**
   - Type: Set
   - Parameters:
     - Assign width, height (default 1024 if missing), prompt, seed (random), steps (4)
   - Connects to: Validate Image Dimensions

8. **Create If Node ("Validate Image Dimensions")**
   - Type: If
   - Parameters:
     - Condition: Check if width * height > 1,048,576 pixels (1 megapixel)
   - True branch connects to Stop and Error node
   - False branch connects to Cloudflare: Get Account ID

9. **Create Stop and Error Node ("Image > 1 megapixel")**
   - Type: Stop and Error
   - Parameters: Error message "The size of the image (width x height) is larger than 1 megapixel"
   - No outputs (workflow stops here if triggered)

10. **Create HTTP Request Node ("Cloudflare: Get Account ID")**
    - Type: HTTP Request
    - Parameters:
      - URL: `https://api.cloudflare.com/client/v4/accounts`
      - Authentication: HTTP Bearer and Header Auth with Cloudflare credentials
    - Connects to: Cloudflare: Generate Image

11. **Create HTTP Request Node ("Cloudflare: Generate Image")**
    - Type: HTTP Request
    - Parameters:
      - URL constructed with account ID from previous node
      - Method: POST
      - JSON Body: includes prompt, seed, width, height, steps from Set node
      - Authentication: same as above
    - Connects to: Convert Image to Binary

12. **Create Convert To File Node ("Convert Image to Binary")**
    - Type: Convert To File
    - Parameters: Source property `result.image` (Cloudflare response)
    - Connects to: Send Image & Prompt to User

13. **Create Telegram Node ("Send Post Content For Verification")**
    - Type: Telegram
    - Parameters:
      - Chat ID: from initial Telegram message chat.id
      - Text: from parsed Gemini output post_text
    - Credentials: Telegram API for "Linked In Post Bot"
    - No further connections

14. **Create Telegram Node ("Send Image & Prompt to User")**
    - Type: Telegram
    - Parameters:
      - Chat ID: from initial Telegram message chat.id
      - Operation: sendPhoto with binary image
      - Caption: parsed Gemini output image_prompt
    - Credentials: Telegram API for "Linked In (JSON) Bot"
    - No further connections

15. **Create Telegram Trigger Node ("Trigger: Reply to Post Image")**
    - Type: Telegram Trigger
    - Parameters: Listen for `message` updates
    - Credentials: Telegram API for "Linked In Post Bot"
    - Connects to: Get Image from Reply

16. **Create Telegram Node ("Get Image from Reply")**
    - Type: Telegram
    - Parameters: fileId from replied-to photo message (largest photo)
    - Credentials: Telegram API for "Linked In Post Bot"
    - Connects to: Publish Post to LinkedIn

17. **Create LinkedIn Node ("Publish Post to LinkedIn")**
    - Type: LinkedIn
    - Parameters:
      - Text: from reply message text
      - Person: LinkedIn profile ID
      - Visibility: CONNECTIONS
      - Share media category: IMAGE
    - Credentials: LinkedIn OAuth2
    - No further connections

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Context or Link                                                                                                                     |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| Turn Your n8n Workflows to LinkedIn Post with a suitable Image. For any questions or support, contact: anirudh.n.aeran@gmail.com. Explore more tips on LinkedIn: https://www.linkedin.com/in/anirudh-narayan-a-/                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Sticky Note4 content - author contact and social media                                                                          |
| This workflow is your personal AI ghostwriter and designer for LinkedIn. It uses multiple AI services and Telegram bots. Instructions include credential setup for Google Gemini, Cloudflare, LinkedIn, Google Sheets, and Telegram bots. Detailed usage steps are provided to generate posts from exported n8n workflows and publish them on LinkedIn via Telegram interactions. Watch Cloudflare credential setup video: https://youtu.be/2tycZNP5_IA?t=75                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Sticky Note2 content - comprehensive user instructions                                                                           |
| Google Gemini node uses a sophisticated prompt with two LinkedIn post styles ("Builder" and "Strategist") and example inputs/outputs to generate post texts and image prompts in JSON format. The final post style alternates for testing performance.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Internal prompt design for AI generation                                                                                         |
| When validating image dimensions, the workflow enforces a maximum of 1 megapixel (e.g., 1024x1024) to prevent overly large images that may cause issues in Telegram or LinkedIn uploads.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Validation constraint in Validate Image Dimensions node                                                                         |
| The LinkedIn posting node uses a static person ID and posts content with visibility restricted to connections, ensuring controlled sharing.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | LinkedIn node configuration                                                                                                     |

---

**Disclaimer:** The content processed and generated by this workflow complies strictly with all current content policies and contains no illegal or protected data. All data handled is legal and public.