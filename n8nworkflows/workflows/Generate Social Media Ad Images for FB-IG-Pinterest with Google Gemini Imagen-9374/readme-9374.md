Generate Social Media Ad Images for FB/IG/Pinterest with Google Gemini Imagen

https://n8nworkflows.xyz/workflows/generate-social-media-ad-images-for-fb-ig-pinterest-with-google-gemini-imagen-9374


# Generate Social Media Ad Images for FB/IG/Pinterest with Google Gemini Imagen

### 1. Workflow Overview

This workflow automates the generation and distribution of social media advertisement images tailored to specific platform dimensions using Google Gemini Imagen AI. It targets marketing teams or social media managers who want to create platform-optimized visuals (Facebook, Instagram, Pinterest) from user-submitted prompts without manual image editing.

The workflow is structured into these logical blocks:

- **1.1 Input Reception:** Captures user inputs for ad generation via a form trigger.
- **1.2 Dimension Routing:** Routes the input data based on selected image dimensions into separate generation paths.
- **1.3 AI Image Generation:** Calls Google Gemini Imagen nodes configured for each platform’s image dimension and format requirements.
- **1.4 Image Distribution:** Sends the generated images with appropriate captions to a Telegram chat via a Telegram bot, separately by platform and format.


---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This initial block captures all user inputs through a configured form. The form fields define the ad parameters (e.g., description, style), which serve as prompts for image generation.

**Nodes Involved:**  
- 📝 Form Trigger

**Node Details:**

- **📝 Form Trigger**  
  - *Type:* Form Trigger node (webhook-based entry)  
  - *Configuration:* Contains form fields set up to capture user input relevant for ad creation (exact fields not detailed). The webhook URL is generated and shared with users to submit requests.  
  - *Key variables:* User inputs are available as node output JSON to the next nodes.  
  - *Connections:* Output connected to the dimension routing node.  
  - *Edge cases:* Potential failures include form misconfiguration, webhook URL not shared correctly, or malformed user input.  
  - *Notes:* Users must configure the form fields as per instructions and distribute the webhook URL.

#### 1.2 Dimension Routing

**Overview:**  
This block directs the workflow to the correct image generation path based on the requested image dimension selected in the form inputs.

**Nodes Involved:**  
- 🔀 Route by Dimensions

**Node Details:**

- **🔀 Route by Dimensions**  
  - *Type:* Switch node  
  - *Configuration:* Configured with 7 outputs corresponding to different image dimensions/formats: Facebook Feed, Facebook Story, Instagram Feed, Instagram Story, Instagram Reel, Pinterest Pin, Pinterest Story.  
  - *Options:* "Output All Matching" enabled to allow multiple simultaneous routes if multiple dimensions are selected.  
  - *Input:* Receives form trigger outputs.  
  - *Output:* Each output connected to a dedicated Google Gemini node for image generation.  
  - *Edge cases:* If no dimension matches, no output will be triggered; malformed input may cause routing failure.  
  - *Notes:* Enables parallel generation of multiple image variants.

#### 1.3 AI Image Generation

**Overview:**  
This block invokes Google Gemini Imagen AI nodes to generate images with platform-specific aspect ratios and styles, based on the prompt data from the user form.

**Nodes Involved:**  
- 🎨 FB Feed (1200x630) -gemini  
- 🎨 FB Story (1080x1920)  
- 🎨 IG Feed (1080x1080)  
- 🎨 IG Story (1080x1920)  
- 🎨 IG Reel (1080x1920)  
- 🎨 Pinterest Pin (1000x1500)  
- 🎨 Pinterest Story (1080x1920)  

**Node Details (common attributes):**

- *Type:* Google Gemini Imagen node (Langchain integration)  
- *Credentials:* All nodes use the same Google Gemini API credentials.  
- *Configuration:* Each node is configured with a specific aspect ratio and style optimized for the target platform and use case (e.g., 9:16 vertical for stories, 1:1 square for IG feed, 2:3 for Pinterest pins).  
- *Prompt:* Automatically populated from the form input (expression referencing form data).  
- *Output:* Generated image data passed to corresponding Telegram send node.  
- *Edge cases:* API auth failures, request timeouts, invalid prompt syntax, or exceeding quota limits.  
- *Notes:* Each node is tuned to generate images suitable for a particular social media format.

#### 1.4 Image Distribution

**Overview:**  
Sends the generated images to a Telegram chat via a bot, with captions customized per platform and format.

**Nodes Involved:**  
- 📤 Send FB Feed  
- 📤 Send FB Story  
- 📤 Send IG Feed  
- 📤 Send IG Story  
- 📤 Send IG Reel  
- 📤 Send Pinterest Pin  
- 📤 Send Pinterest Story  

**Node Details (common attributes):**

- *Type:* Telegram node (sending messages with media)  
- *Credentials:* Uses Telegram bot token credential created via @BotFather and configured with the chat ID of the recipient.  
- *Configuration:* Each node is set up to send the image received from the Gemini node along with a caption specifying the format and any descriptive text.  
- *Input:* Connects directly from each corresponding image generation node.  
- *Edge cases:* Invalid bot token, revoked permissions, incorrect chat ID, message size limits.  
- *Notes:* All Telegram nodes share the same credentials and chat ID for consistent delivery.

---

### 3. Summary Table

| Node Name                      | Node Type                         | Functional Role                  | Input Node(s)           | Output Node(s)                   | Sticky Note                                                                                                  |
|-------------------------------|----------------------------------|--------------------------------|------------------------|---------------------------------|--------------------------------------------------------------------------------------------------------------|
| 📝 Form Trigger                | Form Trigger                     | Captures user input via form   | -                      | 🔀 Route by Dimensions           | SETUP: Configure form fields, copy webhook URL, share with users. Captures inputs for ad generation.          |
| 🔀 Route by Dimensions         | Switch                          | Routes by requested dimension  | 📝 Form Trigger         | All Gemini nodes (7 outputs)     | SETUP: Configured for 7 dimension routes, output all matching enabled. Creates separate paths per dimension.  |
| 🎨 FB Feed (1200x630) -gemini | Google Gemini Imagen Node        | Generate FB Feed image          | 🔀 Route by Dimensions  | 📤 Send FB Feed                 | SETUP: Use Google Gemini API, model imagen-3.0-generate-001, aspect ratio 16:9. Generates FB feed landscape.   |
| 🎨 FB Story (1080x1920)        | Google Gemini Imagen Node        | Generate FB Story image         | 🔀 Route by Dimensions  | 📤 Send FB Story                | SETUP: Same API credentials, aspect ratio 9:16, optimized vertical story format. Creates vertical story image.|
| 🎨 IG Feed (1080x1080)         | Google Gemini Imagen Node        | Generate IG Feed image          | 🔀 Route by Dimensions  | 📤 Send IG Feed                 | SETUP: Same credentials, aspect ratio 1:1, Instagram feed optimized prompt. Perfect square for IG posts.      |
| 🎨 IG Story (1080x1920)        | Google Gemini Imagen Node        | Generate IG Story image         | 🔀 Route by Dimensions  | 📤 Send IG Story                | SETUP: Same credentials, aspect ratio 9:16, story-optimized vertical design. Vertical format for IG Stories.  |
| 🎨 IG Reel (1080x1920)         | Google Gemini Imagen Node        | Generate IG Reel thumbnail      | 🔀 Route by Dimensions  | 📤 Send IG Reel                 | SETUP: Same credentials, aspect ratio 9:16, designed for reels thumbnails. Optimized for video covers.        |
| 🎨 Pinterest Pin (1000x1500)   | Google Gemini Imagen Node        | Generate Pinterest Pin image    | 🔀 Route by Dimensions  | 📤 Send Pinterest Pin           | SETUP: Same credentials, aspect ratio 2:3, Pinterest-style tall pin format. Standard Pinterest pin size.      |
| 🎨 Pinterest Story (1080x1920) | Google Gemini Imagen Node        | Generate Pinterest Story image  | 🔀 Route by Dimensions  | 📤 Send Pinterest Story         | SETUP: Same credentials, aspect ratio 9:16, Pinterest Story format. Story-style vertical pin.                 |
| 📤 Send FB Feed                | Telegram Node                   | Send FB Feed image to Telegram | 🎨 FB Feed (1200x630)   | -                               | SETUP: Telegram bot credentials, chat ID, sends image with caption for FB feed.                              |
| 📤 Send FB Story               | Telegram Node                   | Send FB Story image to Telegram| 🎨 FB Story (1080x1920) | -                               | SETUP: Same Telegram credentials/chat ID, caption includes format details.                                   |
| 📤 Send IG Feed                | Telegram Node                   | Send IG Feed image to Telegram | 🎨 IG Feed (1080x1080)  | -                               | SETUP: Same credentials, optimized caption for IG feed square format.                                        |
| 📤 Send IG Story               | Telegram Node                   | Send IG Story image to Telegram| 🎨 IG Story (1080x1920) | -                               | SETUP: Same credentials, story format caption.                                                              |
| 📤 Send IG Reel                | Telegram Node                   | Send IG Reel thumbnail to Telegram| 🎨 IG Reel (1080x1920)| -                               | SETUP: Same credentials, reel-optimized caption.                                                             |
| 📤 Send Pinterest Pin          | Telegram Node                   | Send Pinterest Pin image       | 🎨 Pinterest Pin (1000x1500) | -                           | SETUP: Same credentials, Pinterest-specific caption.                                                         |
| 📤 Send Pinterest Story        | Telegram Node                   | Send Pinterest Story image     | 🎨 Pinterest Story (1080x1920) | -                           | SETUP: Same credentials, Pinterest story pin caption.                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Type: Form Trigger  
   - Configure form fields to capture user inputs necessary for ad image generation (e.g., ad description, style, dimensions).  
   - Save and copy the webhook URL for sharing with users.

2. **Create Switch Node for Dimension Routing**  
   - Type: Switch  
   - Configure 7 outputs, each corresponding to one image dimension/format:  
     1. Facebook Feed (16:9)  
     2. Facebook Story (9:16)  
     3. Instagram Feed (1:1)  
     4. Instagram Story (9:16)  
     5. Instagram Reel (9:16)  
     6. Pinterest Pin (2:3)  
     7. Pinterest Story (9:16)  
   - Enable "Output All Matching" to allow multiple selections.  
   - Connect the output of the Form Trigger node to the input of this Switch node.

3. **Create Google Gemini Imagen Nodes for Each Dimension**  
   - For each output from the Switch node, create a Google Gemini Imagen node:  
     - Use the same Google Gemini API credentials for all nodes.  
     - Set the model to `imagen-3.0-generate-001` (if applicable).  
     - Configure the aspect ratio and prompt appropriately:  
       - FB Feed: 16:9 (1200x630)  
       - FB Story: 9:16 (1080x1920)  
       - IG Feed: 1:1 (1080x1080)  
       - IG Story: 9:16 (1080x1920)  
       - IG Reel: 9:16 (1080x1920)  
       - Pinterest Pin: 2:3 (1000x1500)  
       - Pinterest Story: 9:16 (1080x1920)  
     - Configure the prompt input to dynamically use the form data from the Form Trigger node (e.g., an expression referencing the relevant form field).  
   - Connect each output from the Switch node to its corresponding Gemini node input.

4. **Create Telegram Send Nodes for Image Delivery**  
   - For each Gemini node, create a Telegram node to send the generated image:  
     - Configure Telegram credentials using a bot token obtained from @BotFather.  
     - Obtain and set the target chat ID by messaging the bot and using `/getUpdates` or similar.  
     - Configure the message to include the image and a caption specifying the format and context (e.g., "Facebook Feed image generated").  
   - Connect each Gemini node output to its respective Telegram node input.

5. **Finalize Connections and Test**  
   - Verify that each node connection follows the path: Form Trigger → Switch → Gemini Imagen → Telegram Send.  
   - Test the form endpoint by submitting sample data and confirm images are generated and delivered correctly for each selected dimension.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                 | Context or Link                                  |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| To create a Telegram bot and obtain the bot token, use @BotFather on Telegram.                                                                               | Telegram Bot creation instructions               |
| To get your Telegram chat ID, message the bot and then use `/getUpdates` endpoint or bot tools.                                                              | Telegram chat ID retrieval                        |
| Google Gemini Imagen nodes require valid API credentials with access to the `imagen-3.0-generate-001` model or equivalent.                                   | Google Gemini API documentation                   |
| "Output All Matching" option in the Switch node allows simultaneous multi-dimension processing.                                                              | n8n Switch node documentation                     |
| Form Trigger webhook URL must be securely shared with users to receive inputs.                                                                                | n8n Form Trigger node documentation               |
| Generated images are tailored to social media platform specifications for optimal display and engagement.                                                    | Platform dimension reference guides (FB, IG, Pinterest) |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.