Generate Complete Social Media Campaigns with DALL·E, GPT-4o and Slack Approval

https://n8nworkflows.xyz/workflows/generate-complete-social-media-campaigns-with-dall-e--gpt-4o-and-slack-approval-11734


# Generate Complete Social Media Campaigns with DALL·E, GPT-4o and Slack Approval

---

### 1. Workflow Overview

This workflow automates the creation of a complete, production-ready social media campaign package using AI tools including GPT-4o and DALL·E 3, and delivers the final assets to a Slack channel for team approval and publishing.  
It is designed for marketing teams, social media managers, and content creators who want to rapidly generate multi-platform campaigns from minimal product information input.

The workflow is logically divided into these blocks:

- **1.1 Product Input Reception and Normalization:** Receives raw product details via a webhook and cleans the data for AI processing.  
- **1.2 AI Campaign Blueprint Generation:** Uses GPT-4o to create a structured campaign blueprint including article summary, platform posts, and insights.  
- **1.3 Instagram Caption Generation:** Generates 5 concise Instagram captions with CTAs based on the campaign blueprint.  
- **1.4 Hashtag Generation:** Produces a set of 12–18 optimized Instagram hashtags tailored to the campaign.  
- **1.5 AI Image Generation and Hosting:** Breaks down posts, generates ultra-realistic social media images per post using DALL·E 3, and uploads them to Cloudinary for hosting.  
- **1.6 Posting Schedule Generation:** Determines the optimal posting times per platform in the Asia/Kolkata timezone using AI.  
- **1.7 Campaign Package Assembly and Delivery:** Merges all generated campaign assets into a unified JSON and sends a formatted message with all assets to Slack.  
- **1.8 Error Handling:** Captures any workflow errors and posts alerts to Slack for immediate awareness.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Product Input Reception and Normalization

**Overview:**  
This block captures incoming product details via HTTP POST webhook and cleans the text fields by removing escape characters and trimming whitespace to prepare consistent input for AI processing.

**Nodes Involved:**  
- Receive Product Details via Webhook  
- Clean & Normalize Product Input Fields

**Node Details:**

- **Receive Product Details via Webhook**  
  - Type: Webhook  
  - Role: Entry point to receive product data from external systems (e.g., form submissions or API calls).  
  - Configuration: POST method, specific webhook path defined for secure capture.  
  - Inputs: External HTTP request with JSON payload containing product_name, product_description, target_audience, brand_voice, benefits, reference_image.  
  - Outputs: Raw JSON product data forwarded downstream.  
  - Failure Modes: Missing fields, malformed JSON, unauthorized calls (not handled explicitly).  
  - Version: 2.1  

- **Clean & Normalize Product Input Fields**  
  - Type: Set  
  - Role: Cleans and normalizes string inputs to remove unwanted escape characters and trailing commas, trims whitespace for consistent AI input.  
  - Configuration: Custom expressions using JavaScript string replace and trim functions on each product field.  
  - Inputs: Raw webhook JSON.  
  - Outputs: Cleaned JSON fields ready for AI consumption.  
  - Edge Cases: Unexpected special characters may not be fully sanitized; empty fields become empty strings.  
  - Version: 3.4  

---

#### 2.2 AI Campaign Blueprint Generation

**Overview:**  
Generates a structured campaign blueprint JSON object outlining the marketing strategy, article summary, and recommended posts for multiple platforms using GPT-4o via LangChain.

**Nodes Involved:**  
- Generate Campaign Blueprint Using AI  
- LLM Engine for Campaign Blueprint  
- Parse Campaign Blueprint JSON

**Node Details:**

- **Generate Campaign Blueprint Using AI**  
  - Type: LangChain Agent  
  - Role: Sends a prompt with cleaned product details to GPT-4o to generate a JSON campaign blueprint.  
  - Configuration:  
    - System message sets marketing strategist persona and instructs strict JSON output with no explanations.  
    - Input prompt includes product details injected dynamically.  
    - Output parser enabled to enforce JSON schema compliance.  
  - Inputs: Cleaned product JSON from previous block.  
  - Outputs: Raw AI JSON output.  
  - Edge Cases: Model could produce invalid JSON if prompt misunderstood; handled by output parser.  
  - Version: 2  

- **LLM Engine for Campaign Blueprint**  
  - Type: LangChain OpenAI Chat Model  
  - Role: Provides GPT-4o model instance for blueprint generation.  
  - Configuration: Model set to gpt-4o, OpenAI credentials specified.  
  - Inputs: Triggered by agent node.  
  - Outputs: AI text response.  
  - Failure Modes: API rate limits, network issues, invalid credentials.  
  - Version: 1.3  

- **Parse Campaign Blueprint JSON**  
  - Type: LangChain Output Parser (Structured)  
  - Role: Validates and extracts AI JSON output strictly according to defined JSON schema for downstream use.  
  - Configuration: JSON schema example defining required fields such as article_summary and posts.  
  - Inputs: Raw AI output from agent node.  
  - Outputs: Parsed JSON data structure.  
  - Failure Modes: Parsing errors if AI output is malformed.  
  - Version: 1.3  

---

#### 2.3 Instagram Caption Generation

**Overview:**  
Generates 5 unique, Instagram-optimized captions with strong hooks, concise benefit highlights, and CTAs, all aligned with the campaign blueprint.

**Nodes Involved:**  
- Generate Instagram Captions Using AI  
- LLM Engine for Caption Writing  
- Parse Caption Output JSON

**Node Details:**

- **Generate Instagram Captions Using AI**  
  - Type: LangChain Agent  
  - Role: Uses GPT-4o to generate short, punchy captions based on the campaign blueprint’s article summary.  
  - Configuration: System prompt instructs caption style rules (2–3 lines, 0–2 emojis, no hashtags, separate CTA field, no markdown).  
  - Inputs: Campaign blueprint JSON from previous parsing node.  
  - Outputs: Raw AI captions JSON.  
  - Edge Cases: Model might generate captions exceeding length or including disallowed elements; output parser enforces schema.  
  - Version: 2  

- **LLM Engine for Caption Writing**  
  - Type: LangChain OpenAI Chat Model  
  - Role: Provides GPT-4o model instance for caption generation.  
  - Configuration: Model set to gpt-4o, uses same OpenAI credentials as blueprint model.  
  - Inputs: Triggered by agent node.  
  - Outputs: AI text response.  
  - Failure Modes: API limits, network errors.  
  - Version: 1.3  

- **Parse Caption Output JSON**  
  - Type: LangChain Output Parser (Structured)  
  - Role: Validates AI captions JSON to ensure all captions and CTAs conform to expected schema.  
  - Configuration: JSON schema example with array of caption objects containing caption and cta strings.  
  - Inputs: Raw captions JSON.  
  - Outputs: Parsed structured captions.  
  - Failure Modes: Parsing errors if AI output invalid.  
  - Version: 1.3  

---

#### 2.4 Hashtag Generation

**Overview:**  
Creates a set of 12 to 18 optimized Instagram hashtags mixing broad and niche tags to maximize reach and engagement without banned or spammy tags.

**Nodes Involved:**  
- Generate Hashtag Set Using AI  
- LLM Engine for Hashtag Generation  
- Parse Hashtag Output JSON

**Node Details:**

- **Generate Hashtag Set Using AI**  
  - Type: LangChain Agent  
  - Role: Uses GPT-4o to generate a curated list of hashtags based on campaign blueprint and captions.  
  - Configuration: System prompt emphasizes no duplicates, lowercase only, no spaces, no explanations, and JSON-only output.  
  - Inputs: Campaign blueprint article summary and parsed Instagram captions.  
  - Outputs: Raw AI-generated hashtags JSON.  
  - Edge Cases: Potential for banned or irrelevant tags if model misunderstood; prompt tries to prevent this.  
  - Version: 2  

- **LLM Engine for Hashtag Generation**  
  - Type: LangChain OpenAI Chat Model  
  - Role: GPT-4o model instance dedicated to hashtag generation tasks.  
  - Configuration: Same GPT-4o model and credentials as prior AI nodes.  
  - Inputs: Triggered by agent node.  
  - Outputs: AI text response.  
  - Failure Modes: API limitations.  
  - Version:1.3  

- **Parse Hashtag Output JSON**  
  - Type: LangChain Output Parser (Structured)  
  - Role: Ensures hashtags list matches expected JSON schema (array of strings).  
  - Configuration: JSON schema example with hashtags array.  
  - Inputs: Raw hashtags JSON.  
  - Outputs: Parsed hashtag list.  
  - Failure Modes: Parsing errors.  
  - Version: 1.3  

---

#### 2.5 AI Image Generation and Hosting

**Overview:**  
Splits campaign posts to generate individual images using DALL·E 3 with ultra-realistic style, then uploads each generated image to Cloudinary for hosting and public URL retrieval.

**Nodes Involved:**  
- Split Campaign Posts for Image Generation  
- Generate Social Media Image Using AI  
- Upload Generated Image to Cloudinary

**Node Details:**

- **Split Campaign Posts for Image Generation**  
  - Type: SplitOut  
  - Role: Splits the array of posts from the campaign blueprint into individual items for parallel image generation.  
  - Configuration: Splits on the field `output.posts`.  
  - Inputs: Campaign blueprint JSON.  
  - Outputs: One item per post with its data.  
  - Edge Cases: Empty posts array results in no downstream images.  
  - Version: 1  

- **Generate Social Media Image Using AI**  
  - Type: OpenAI Node  
  - Role: Uses DALL·E 3 to generate one high-quality commercial social media image per post.  
  - Configuration:  
    - Model: dall-e-3  
    - Prompt dynamically composed using platform, brand theme, audience, tone, and image prompt from post item.  
    - Style instructions for ultra-realistic, cinematic lighting, 8K quality.  
    - Returns only the final image description (not the image file).  
  - Inputs: Split post items.  
  - Outputs: Image generation response (binary image data).  
  - Failure Modes: API limits, generation timeouts, malformed prompts.  
  - Version: 1  

- **Upload Generated Image to Cloudinary**  
  - Type: HTTP Request  
  - Role: Uploads the generated image binary data to Cloudinary using multipart/form-data POST request to receive a public URL.  
  - Configuration:  
    - URL: Cloudinary image upload endpoint.  
    - Authentication: HTTP Basic with Cloudinary credential.  
    - Form fields: `file` (binary image), `upload_preset` set to "upload".  
    - Retries enabled on failure (up to 5 times).  
  - Inputs: Image binary data from previous node.  
  - Outputs: JSON response containing secure URL of hosted image.  
  - Failure Modes: Network errors, Cloudinary quota limits, invalid credentials.  
  - Version: 4.2  

---

#### 2.6 Posting Schedule Generation

**Overview:**  
Generates optimal posting times per platform tailored to the target audience behavior and platform engagement trends, using Asia/Kolkata timezone.

**Nodes Involved:**  
- Generate Optimal Posting Schedule Using AI  
- LLM Engine for Posting Schedule  
- Parse Posting Schedule Output JSON

**Node Details:**

- **Generate Optimal Posting Schedule Using AI**  
  - Type: LangChain Agent  
  - Role: Instructs GPT-4o to produce a JSON schedule recommending the best posting time per platform with reasoning.  
  - Configuration: System message frames strategist persona with rules for timezone, audience habits, and no caption repetition.  
  - Inputs: Campaign blueprint target audience and posts JSON.  
  - Outputs: Raw scheduling JSON.  
  - Edge Cases: Model might output invalid JSON; output parser validation mitigates risk.  
  - Version: 2  

- **LLM Engine for Posting Schedule**  
  - Type: LangChain OpenAI Chat Model  
  - Role: GPT-4o engine used exclusively for scheduling generation.  
  - Configuration: Same as other AI nodes.  
  - Inputs: Triggered by the scheduling agent.  
  - Outputs: AI response text.  
  - Failure Modes: API limits, network errors.  
  - Version: 1.3  

- **Parse Posting Schedule Output JSON**  
  - Type: LangChain Output Parser (Structured)  
  - Role: Validates schedule JSON against schema requiring platform, recommended_time, timezone, and reasoning fields.  
  - Configuration: JSON schema example provided.  
  - Inputs: Raw AI scheduling JSON.  
  - Outputs: Parsed schedule data.  
  - Failure Modes: Parsing errors if AI output malformed.  
  - Version: 1.3  

---

#### 2.7 Campaign Package Assembly and Delivery

**Overview:**  
Merges all generated campaign data—images, captions, hashtags, posting schedule—into one final JSON object then sends a richly formatted campaign summary to a Slack channel.

**Nodes Involved:**  
- Combine All Campaign Assets  
- Prepare Final Campaign Package JSON  
- Send Final Campaign Package to Slack

**Node Details:**

- **Combine All Campaign Assets**  
  - Type: Merge  
  - Role: Combines outputs from images, captions, hashtags, and schedule nodes into a single stream for final formatting.  
  - Configuration: Accepts four inputs (images, captions, hashtags, schedule).  
  - Inputs: Outputs from Upload Generated Image to Cloudinary, Generate Instagram Captions Using AI, Generate Hashtag Set Using AI, Generate Optimal Posting Schedule Using AI.  
  - Outputs: Merged items array.  
  - Edge Cases: Missing data on any input leads to incomplete package.  
  - Version: 3.2  

- **Prepare Final Campaign Package JSON**  
  - Type: Code  
  - Role: Custom JavaScript node that aggregates images (extracts secure URLs), captions, hashtags, and schedule arrays into a unified JSON object.  
  - Configuration: Loops over all input items, conditionally collects relevant data fields into arrays, and returns a single JSON with all campaign assets.  
  - Inputs: Combined campaign assets from Merge node.  
  - Outputs: Final campaign JSON object.  
  - Edge Cases: Missing or malformed data in any input can cause empty fields.  
  - Version: 2  

- **Send Final Campaign Package to Slack**  
  - Type: Slack Node  
  - Role: Sends a formatted Slack message to a predefined channel with images, captions + CTAs, hashtags, and posting schedule details.  
  - Configuration:  
    - Text uses Slack markdown formatting, dynamically injecting all campaign data.  
    - Channel ID set to a specific Slack channel (configurable).  
    - Uses Slack API OAuth2 credentials.  
  - Inputs: Final campaign JSON.  
  - Outputs: Slack message post response.  
  - Failure Modes: Slack API limits, invalid channel ID or credentials.  
  - Version: 2.1  

---

#### 2.8 Error Handling

**Overview:**  
Monitors the entire workflow for failures and sends an immediate error alert message to Slack with details for fast troubleshooting.

**Nodes Involved:**  
- Error Handler Trigger  
- Slack: Send Error Alert

**Node Details:**

- **Error Handler Trigger**  
  - Type: Error Trigger  
  - Role: Listens to workflow execution errors globally.  
  - Inputs: Workflow error events.  
  - Outputs: Error data forwarded downstream.  
  - Version: 1  

- **Slack: Send Error Alert**  
  - Type: Slack Node  
  - Role: Posts an error alert message to Slack including node name, error message, and timestamp.  
  - Configuration: Sends to the same Slack channel as final package node, uses OAuth2 credentials.  
  - Inputs: Error data from error trigger node.  
  - Outputs: Slack API response.  
  - Version: 2.3  

---

### 3. Summary Table

| Node Name                         | Node Type                          | Functional Role                         | Input Node(s)                         | Output Node(s)                                         | Sticky Note                                                                                                      |
|----------------------------------|----------------------------------|---------------------------------------|-------------------------------------|-------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Receive Product Details via Webhook | Webhook                         | Captures raw product details           | -                                   | Clean & Normalize Product Input Fields                 | 🧼 Product Intake & Normalization: Receives product details and cleans escape characters.                        |
| Clean & Normalize Product Input Fields | Set                            | Cleans and normalizes product fields   | Receive Product Details via Webhook | Generate Campaign Blueprint Using AI                   | 🧼 Product Intake & Normalization: Cleans text before AI processing.                                            |
| Generate Campaign Blueprint Using AI | LangChain Agent                | Generates structured campaign blueprint | Clean & Normalize Product Input Fields | Generate Instagram Captions Using AI, Split Campaign Posts for Image Generation | 🧠 AI Campaign Blueprint Creation: Builds marketing blueprint JSON using GPT-4o.                                |
| LLM Engine for Campaign Blueprint | LangChain OpenAI Chat Model     | Provides GPT-4o model for blueprint    | Generate Campaign Blueprint Using AI | Generate Campaign Blueprint Using AI                   | 🧠 AI Campaign Blueprint Creation: GPT-4o engine for blueprint generation.                                      |
| Parse Campaign Blueprint JSON     | LangChain Output Parser (Structured) | Validates campaign blueprint JSON      | Generate Campaign Blueprint Using AI | Generate Campaign Blueprint Using AI                   | 🧠 AI Campaign Blueprint Creation: Parses and validates blueprint JSON.                                        |
| Generate Instagram Captions Using AI | LangChain Agent                | Creates 5 Instagram captions with CTAs | Parse Campaign Blueprint JSON       | Generate Hashtag Set Using AI, Combine All Campaign Assets | ✍️ Instagram Caption Generation: Produces concise captions with CTAs.                                          |
| LLM Engine for Caption Writing    | LangChain OpenAI Chat Model     | Provides GPT-4o model for captions     | Generate Instagram Captions Using AI | Generate Instagram Captions Using AI                   | ✍️ Instagram Caption Generation: GPT-4o engine for caption writing.                                            |
| Parse Caption Output JSON          | LangChain Output Parser (Structured) | Validates captions JSON                 | Generate Instagram Captions Using AI | Generate Instagram Captions Using AI                   | ✍️ Instagram Caption Generation: Parses and validates caption JSON.                                            |
| Generate Hashtag Set Using AI      | LangChain Agent                | Generates optimized hashtag set         | Parse Caption Output JSON, Parse Campaign Blueprint JSON | Generate Optimal Posting Schedule Using AI, Combine All Campaign Assets | #️⃣ High-Performing Hashtag Generation: Creates discovery-optimized hashtags.                                  |
| LLM Engine for Hashtag Generation  | LangChain OpenAI Chat Model     | GPT-4o model for hashtag generation    | Generate Hashtag Set Using AI       | Generate Hashtag Set Using AI                           | #️⃣ High-Performing Hashtag Generation: GPT-4o engine for hashtags.                                            |
| Parse Hashtag Output JSON          | LangChain Output Parser (Structured) | Validates hashtags JSON                 | Generate Hashtag Set Using AI       | Generate Hashtag Set Using AI                           | #️⃣ High-Performing Hashtag Generation: Parses and validates hashtag JSON.                                    |
| Split Campaign Posts for Image Generation | SplitOut                     | Splits posts array for image generation | Generate Campaign Blueprint Using AI | Generate Social Media Image Using AI                    | 🎨 AI Image Generation for Campaign Posts: Splits posts for individual image creation.                          |
| Generate Social Media Image Using AI | OpenAI (DALL·E 3)             | Generates ultra-realistic images per post | Split Campaign Posts for Image Generation | Upload Generated Image to Cloudinary                   | 🎨 AI Image Generation for Campaign Posts: Creates commercial-quality images with DALL·E 3.                     |
| Upload Generated Image to Cloudinary | HTTP Request                  | Uploads images and retrieves public URLs | Generate Social Media Image Using AI | Combine All Campaign Assets                             | 🎨 AI Image Generation for Campaign Posts: Uploads images to Cloudinary and provides URLs.                     |
| Generate Optimal Posting Schedule Using AI | LangChain Agent              | Recommends posting times per platform  | Parse Hashtag Output JSON           | Combine All Campaign Assets                             | 🕒 Optimal Posting Schedule Generation: Creates best posting schedule per platform in IST timezone.             |
| LLM Engine for Posting Schedule    | LangChain OpenAI Chat Model     | GPT-4o model for scheduling            | Generate Optimal Posting Schedule Using AI | Generate Optimal Posting Schedule Using AI           | 🕒 Optimal Posting Schedule Generation: GPT-4o engine for schedule generation.                                 |
| Parse Posting Schedule Output JSON | LangChain Output Parser (Structured) | Validates posting schedule JSON        | Generate Optimal Posting Schedule Using AI | Generate Optimal Posting Schedule Using AI           | 🕒 Optimal Posting Schedule Generation: Parses and validates schedule JSON.                                   |
| Combine All Campaign Assets        | Merge                          | Merges images, captions, hashtags, schedule | Upload Generated Image to Cloudinary, Generate Instagram Captions Using AI, Generate Hashtag Set Using AI, Generate Optimal Posting Schedule Using AI | Prepare Final Campaign Package JSON                     | 📦 Assemble Final Campaign Package: Combines all assets into unified data.                                    |
| Prepare Final Campaign Package JSON | Code                          | Aggregates all campaign elements into final JSON | Combine All Campaign Assets         | Send Final Campaign Package to Slack                   | 📦 Assemble Final Campaign Package: Formats final JSON package for Slack delivery.                            |
| Send Final Campaign Package to Slack | Slack                         | Sends campaign package message to Slack channel | Prepare Final Campaign Package JSON | -                                                     | 📦 Assemble Final Campaign Package: Posts rich campaign summary to Slack channel.                             |
| Error Handler Trigger              | Error Trigger                  | Listens for workflow errors globally  | -                                   | Slack: Send Error Alert                                | 🚨 Error Handling: Catches workflow failures and triggers Slack alerts.                                       |
| Slack: Send Error Alert            | Slack                         | Sends error alert message to Slack    | Error Handler Trigger               | -                                                     | 🚨 Error Handling: Posts error details including node name and message to Slack channel.                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Name: Receive Product Details via Webhook  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: Unique webhook path (e.g., "691cefdb-6770-40db-9b6a-1da2350088fe")  
   - Purpose: Receive JSON product details payload  

2. **Add Set Node**  
   - Name: Clean & Normalize Product Input Fields  
   - Type: Set  
   - Configure assignments with JavaScript expressions to:  
     - Remove escaped quotes from `body.product_name`, `body.product_description`, `body.target_audience`, `body.brand_voice`, `body.benefits`  
     - Trim whitespace and trailing commas  
     - Pass `body.reference_image` as is  
   - Connect output of Webhook node to this node  

3. **Add LangChain Agent Node**  
   - Name: Generate Campaign Blueprint Using AI  
   - Type: LangChain Agent  
   - Prompt: Include all cleaned product fields; instruct to output strictly valid JSON matching a schema with article_summary and posts fields  
   - Enable Output Parser with JSON schema example  
   - Connect output of Set node here  

4. **Add LangChain Chat Model Node**  
   - Name: LLM Engine for Campaign Blueprint  
   - Type: LangChain OpenAI Chat  
   - Model: gpt-4o  
   - Credentials: OpenAI API key  
   - Connect as AI language model to Agent node  

5. **Add LangChain Output Parser Node**  
   - Name: Parse Campaign Blueprint JSON  
   - Type: LangChain Output Parser (Structured)  
   - JSON Schema: Matches campaign blueprint structure (article_summary, posts array)  
   - Connect AI output of Agent node here  

6. **Add LangChain Agent Node**  
   - Name: Generate Instagram Captions Using AI  
   - Type: LangChain Agent  
   - Prompt: Use campaign blueprint article_summary; generate 5 captions with CTAs, no hashtags, short length, 0-2 emojis, JSON-only output  
   - Enable Output Parser with captions JSON schema  
   - Connect parsed blueprint JSON output here  

7. **Add LangChain Chat Model Node**  
   - Name: LLM Engine for Caption Writing  
   - Type: LangChain OpenAI Chat  
   - Model: gpt-4o  
   - Credentials: OpenAI API key  
   - Connect as language model to Captions Agent  

8. **Add LangChain Output Parser Node**  
   - Name: Parse Caption Output JSON  
   - Type: LangChain Output Parser (Structured)  
   - JSON Schema: Array of captions with caption and cta fields  
   - Connect AI output of Captions Agent here  

9. **Add LangChain Agent Node**  
   - Name: Generate Hashtag Set Using AI  
   - Type: LangChain Agent  
   - Prompt: Use campaign blueprint article_summary and captions; generate 12-18 lowercase hashtags, no duplicates or banned tags, JSON-only output  
   - Enable Output Parser with hashtags schema  
   - Connect output of Parse Caption Output JSON and Parse Campaign Blueprint JSON (merge or select) here  

10. **Add LangChain Chat Model Node**  
    - Name: LLM Engine for Hashtag Generation  
    - Type: LangChain OpenAI Chat  
    - Model: gpt-4o  
    - Credentials: OpenAI API key  
    - Connect as language model to Hashtag Agent  

11. **Add LangChain Output Parser Node**  
    - Name: Parse Hashtag Output JSON  
    - Type: LangChain Output Parser (Structured)  
    - JSON Schema: Array of strings for hashtags  
    - Connect AI output of Hashtag Agent here  

12. **Add SplitOut Node**  
    - Name: Split Campaign Posts for Image Generation  
    - Type: SplitOut  
    - Field to split: output.posts from campaign blueprint JSON  
    - Connect parsed campaign blueprint JSON here  

13. **Add OpenAI Node**  
    - Name: Generate Social Media Image Using AI  
    - Type: OpenAI (Image generation)  
    - Model: dall-e-3  
    - Prompt: Compose prompt using platform, brand theme (article_summary.title), audience, tone, and image prompt from split post item  
    - Image parameters: 1 image, 1792x1024, vivid style, high quality  
    - Credentials: OpenAI API key  
    - Connect output from SplitOut node here  

14. **Add HTTP Request Node**  
    - Name: Upload Generated Image to Cloudinary  
    - Type: HTTP Request  
    - Method: POST  
    - URL: https://api.cloudinary.com/v1_1/{your_cloud_name}/image/upload  
    - Authentication: HTTP Basic Auth with Cloudinary credentials  
    - Body: multipart/form-data, field "file" with binary image data, "upload_preset" set to "upload"  
    - Enable retry on failure (max 5 tries)  
    - Connect output from Image Generation node here  

15. **Add LangChain Agent Node**  
    - Name: Generate Optimal Posting Schedule Using AI  
    - Type: LangChain Agent  
    - Prompt: Use campaign blueprint target audience and posts; generate posting schedule per platform in Asia/Kolkata timezone with reasoning, JSON-only output  
    - Enable Output Parser with schedule JSON schema  
    - Connect output of Parse Hashtag Output JSON here  

16. **Add LangChain Chat Model Node**  
    - Name: LLM Engine for Posting Schedule  
    - Type: LangChain OpenAI Chat  
    - Model: gpt-4o  
    - Credentials: OpenAI API key  
    - Connect as language model to Schedule Agent  

17. **Add LangChain Output Parser Node**  
    - Name: Parse Posting Schedule Output JSON  
    - Type: LangChain Output Parser (Structured)  
    - JSON Schema: Schedule array with platform, recommended_time, timezone, reasoning fields  
    - Connect AI output of Schedule Agent here  

18. **Add Merge Node**  
    - Name: Combine All Campaign Assets  
    - Type: Merge  
    - Number of Inputs: 4  
    - Connect outputs from:  
      - Upload Generated Image to Cloudinary (images)  
      - Generate Instagram Captions Using AI (captions)  
      - Generate Hashtag Set Using AI (hashtags)  
      - Generate Optimal Posting Schedule Using AI (schedule)  

19. **Add Code Node**  
    - Name: Prepare Final Campaign Package JSON  
    - Type: Code (JavaScript)  
    - Script: Aggregate all images (secure_url), captions, hashtags, and schedule arrays into a single JSON object and output one item  
    - Connect output of Merge node here  

20. **Add Slack Node**  
    - Name: Send Final Campaign Package to Slack  
    - Type: Slack  
    - Channel: Set to your team channel (e.g., #general)  
    - Text: Compose using Slack markdown with sections for images, captions + CTAs, hashtags, and posting schedule, dynamically injected from final JSON  
    - Credentials: OAuth2 Slack account  
    - Connect output from Code node here  

21. **Add Error Trigger Node**  
    - Name: Error Handler Trigger  
    - Type: Error Trigger  
    - No inputs; listens globally  

22. **Add Slack Node**  
    - Name: Slack: Send Error Alert  
    - Type: Slack  
    - Channel: Same as final package node  
    - Text: Include node name, error message, timestamp from error trigger  
    - Credentials: Slack OAuth2  
    - Connect output from Error Trigger node here  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                              | Context or Link                                                                                           |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| This workflow replaces hours of manual creative work by automating social media campaign production end-to-end using AI-powered generation and integration with Slack for team collaboration.                                            | Sticky Note at workflow start                                                                             |
| The campaign blueprint strictly uses only provided product benefits and brand voice; no invented content to ensure brand consistency.                                                                                                  | Sticky Note2: AI Campaign Blueprint Creation                                                             |
| Instagram captions exclude hashtags (generated separately) and limit emoji use to maintain brand professionalism and engagement.                                                                                                       | Sticky Note3: Instagram Caption Generation                                                               |
| Hashtag generation avoids banned or spammy tags and mixes broad and niche tags for optimal discovery on Instagram.                                                                                                                     | Sticky Note4: High-Performing Hashtag Generation                                                         |
| Image generation uses DALL·E 3 with ultra-realistic, cinematic style guidelines for professional quality social content; images exclude text and logos for compliance.                                                                     | Sticky Note5: AI Image Generation for Campaign Posts                                                     |
| Posting schedule is optimized for Indian audience timezone (Asia/Kolkata) with insights on audience behavior and platform trends for maximum engagement.                                                                               | Sticky Note6: Optimal Posting Schedule Generation                                                        |
| Final Slack message uses rich formatting, showing images, captions with CTAs, hashtags, and posting times, providing a comprehensive campaign package ready for publishing.                                                             | Sticky Note7: Assemble Final Campaign Package                                                            |
| Robust error handling sends alerts to Slack with time-stamped error information and node identification for quick debugging and operational resilience.                                                                                   | Sticky Note8: Error Handling                                                                              |
| [Slack API Documentation](https://api.slack.com/messaging/sending) - Useful for customizing Slack messages.                                                                                                                             | Slack integration reference                                                                               |
| [OpenAI API Documentation](https://platform.openai.com/docs) - Reference for GPT-4o and DALL·E 3 usage and parameters.                                                                                                                 | AI model integration reference                                                                            |
| [Cloudinary API Documentation](https://cloudinary.com/documentation/image_upload_api_reference) - For image upload and management details.                                                                                             | Image hosting integration reference                                                                       |

---

*Disclaimer: The text provided originates exclusively from an n8n automated workflow. It complies strictly with content policies and contains no illegal, offensive, or protected material. All data processed is legal and public.*