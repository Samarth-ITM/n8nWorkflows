Generate AI Promo Videos for Products with GPT-4o, Fal.ai & Human Supervision

https://n8nworkflows.xyz/workflows/generate-ai-promo-videos-for-products-with-gpt-4o--fal-ai---human-supervision-7196


# Generate AI Promo Videos for Products with GPT-4o, Fal.ai & Human Supervision

### 1. Workflow Overview

This workflow automates the generation of AI-powered promotional videos for products, specifically targeting an online ceramics store. It leverages GPT-4o for AI content generation, Fal.ai for image and video creation, and gotoHuman for human supervision and review at key steps. The workflow is designed to run on a schedule or via manual webhook trigger, producing weekly social media-ready video clips with human-in-the-loop approval and iterative refinement.

Logical blocks:

- **1.1 Trigger & Campaign Idea Generation**: Schedule or manual trigger starts the process; GPT-4o generates topic ideas for marketing posts adapted to season and audience.
- **1.2 Topic Selection & Tagline Generation**: Human reviews topic ideas in gotoHuman; approved topics feed AI generation of taglines, enhanced by examples from previous approvals.
- **1.3 Reference Image Generation**: Using approved topics and taglines, Fal.ai generates product images; Cloudinary overlays taglines on images; human review and retry possible.
- **1.4 Video Clip Generation**: Fal.ai creates short video clips from approved images with natural movement; Cloudinary overlays taglines on videos; human review and retry possible.
- **1.5 Final Posting Preparation**: Approved videos are ready for publication; URLs are provided for manual or automated social media posting.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Campaign Idea Generation

**Overview:**  
This block initiates the workflow either on a scheduled weekly basis or via a manual webhook trigger. It then uses GPT-4o mini to generate five topic ideas tailored to the current season and northern hemisphere customers.

**Nodes Involved:**  
- Schedule Trigger  
- Remote manual trigger  
- LLM (gpt-4o-mini)  
- Thinking (tool)  
- AI Agent: Generate Topic ideas (Langchain agent)  
- Extract JSON (structured output parser)  
- Data to send (Set)  
- gotoHuman - Topics

**Node Details:**  

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Configured to run weekly on Thursdays at 12:00 PM.  
  - Output triggers topic idea generation.  
  - Edge cases: Time zone considerations, missed triggers.

- **Remote manual trigger**  
  - Type: Webhook (POST)  
  - Allows external manual start with campaign name input.  
  - Input expects JSON with campaignName and metadata.  
  - Edge cases: Invalid payload, authentication.

- **LLM**  
  - Type: Langchain OpenAI chat model node  
  - Model: gpt-4o-mini (OpenAI GPT-4 variant)  
  - Role: Provides AI language model backend for topic generation.  
  - Edge cases: API rate limits, token limits, timeouts.

- **Thinking**  
  - Type: Langchain toolThink  
  - Used as an internal reasoning step feeding AI Agent.  
  - No parameters.  
  - Edge cases: N/A.

- **AI Agent: Generate Topic ideas**  
  - Type: Langchain agent node  
  - Prompt: Generates 5 marketing topic ideas with background, situation, and target audience, adjusted for date and season.  
  - Outputs JSON-parsed structured data.  
  - Edge cases: Incorrect JSON output, prompt failures.

- **Extract JSON**  
  - Type: Langchain outputParserStructured  
  - Parses AI agent output into structured JSON with fields: scenery, activity, audience.  
  - Edge cases: Parsing errors, schema mismatch.

- **Data to send**  
  - Type: Set  
  - Prepares data payload for gotoHuman with topic ideas and metadata.  
  - Variables: topics (stringified), metaJson (workflow metadata including campaign name).  
  - Edge cases: Missing metadata fields.

- **gotoHuman - Topics**  
  - Type: gotoHuman node  
  - Purpose: Sends topic ideas for human review and selection.  
  - Inputs: topics, metaJson.  
  - Outputs: reviewed topics, review IDs for tracking.  
  - Edge cases: API auth issues, network errors.

---

#### 1.2 Topic Selection & Tagline Generation

**Overview:**  
Once topics are approved by humans, this block fetches previous approved taglines to include as positive examples, then generates new taglines for the selected topic using GPT-4o mini with a self-improving prompt. Results are sent back to gotoHuman for review.

**Nodes Involved:**  
- Approved? (If node)  
- Fetch approved examples (HTTP Request to gotoHuman)  
- Tagline Prompt (Set)  
- AI: Generate Tag Lines (Langchain agent)  
- Extract JSON1 (Output parser)  
- Data to send2 (Set)  
- gotoHuman - Content  
- Approved or Retry? (Switch node)

**Node Details:**  

- **Approved?**  
  - Type: If  
  - Checks if topic review response equals "approved".  
  - Controls flow to fetch approved tagline examples or end.  
  - Edge cases: Unexpected response values.

- **Fetch approved examples**  
  - Type: HTTP Request  
  - Calls gotoHuman API to retrieve only approved taglines grouped by field.  
  - Credentials: gotoHuman API credentials required.  
  - Edge cases: Auth failure, request timeouts.

- **Tagline Prompt**  
  - Type: Set  
  - Builds prompt string for tagline generation, including examples of approved taglines and topic context (audience, activity, scenery).  
  - Key expressions use gotoHuman node outputs for dynamic prompt data.  
  - Edge cases: Missing topic context, empty examples.

- **AI: Generate Tag Lines**  
  - Type: Langchain agent  
  - Generates short taglines suitable for social media post overlays; must not include commas.  
  - Temperature set to 0.9 for creativity.  
  - Edge cases: Generation failures, output parsing errors.

- **Extract JSON1**  
  - Type: Output parser structured  
  - Parses tagline output JSON with schema requiring "tagLine" string property.  
  - Edge cases: Parsing errors on malformed AI output.

- **Data to send2**  
  - Type: Set  
  - Aggregates approved topic data and generated tagline into a payload for gotoHuman; includes products array (static URLs of ceramic products).  
  - Meta information for workflow tracing included.  
  - Edge cases: Product URLs missing or invalid.

- **gotoHuman - Content**  
  - Type: gotoHuman node  
  - Sends tagline and topic content for human review and editing, including scenery and activity fields.  
  - Supports manual edits or retry requests.  
  - Edge cases: API failures, review update conflicts.

- **Approved or Retry?**  
  - Type: Switch  
  - Routes output based on human response type: "approved" proceeds, "chat" triggers rerun of tagline generation.  
  - Edge cases: Unexpected response types.

---

#### 1.3 Reference Image Generation

**Overview:**  
Using approved content, this block formats an image generation prompt with a style selected by humans, sends a request to Fal.ai to generate a product image fitting the scenario, waits for completion, then uploads the resulting image to Cloudinary with the tagline overlay. The result is sent to gotoHuman for review.

**Nodes Involved:**  
- Set Initial Image Prompt (Code)  
- Set Image Request (Code)  
- Fal.ai Image Gen (HTTP Request)  
- Fetch Status1 (HTTP Request)  
- Is Ready? (If)  
- Wait1 (Wait)  
- Fetch Result1 (HTTP Request)  
- Set Cloudinary options (Set)  
- Cloudinary upload (HTTP Request)  
- Image URL with Tag Line (Set)  
- Data to send3 (Set)  
- gotoHuman - Image  
- Approved or Retry ? (Switch)  
- Rerun input (Set)

**Node Details:**  

- **Set Initial Image Prompt**  
  - Type: Code  
  - Based on style selected by human (interior, garden, default), produces a detailed descriptive prompt for Fal.ai image generation.  
  - Uses fields from gotoHuman - Content node (scenery, activity).  
  - Edge cases: Unknown style values.

- **Set Image Request**  
  - Type: Code  
  - Creates JSON payload with prompt, product image URL, and aspect ratio ("3:4") for Fal.ai.  
  - Pulls product URL from gotoHuman selected products.  
  - Edge cases: Missing product URLs.

- **Fal.ai Image Gen**  
  - Type: HTTP Request  
  - Posts image generation request to Fal.ai API with authorization header.  
  - Authentication: Generic HTTP header Auth with API key.  
  - Edge cases: API errors, rate limits.

- **Fetch Status1**  
  - Type: HTTP Request  
  - Polls Fal.ai API for generation status of image request using request_id.  
  - Edge cases: Network errors, delayed responses.

- **Is Ready?**  
  - Type: If  
  - Checks if Fal.ai response status equals "COMPLETED".  
  - If not ready, loops back to Wait1.  
  - Edge cases: Unexpected status values.

- **Wait1**  
  - Type: Wait  
  - Waits 1 second between status polls.  
  - Edge cases: Long delays.

- **Fetch Result1**  
  - Type: HTTP Request  
  - Fetches the completed image generation result from Fal.ai response_url.  
  - Edge cases: Broken URLs.

- **Set Cloudinary options**  
  - Type: Set  
  - Contains Cloudinary environment and upload preset names (user must fill).  
  - Edge cases: Missing or incorrect Cloudinary credentials.

- **Cloudinary upload**  
  - Type: HTTP Request  
  - Uploads the generated image to Cloudinary with multipart form-data, including overlay of tagLine as text layer.  
  - Uses Cloudinary unsigned upload preset.  
  - Edge cases: Upload failures, preset misconfiguration.

- **Image URL with Tag Line**  
  - Type: Set  
  - Constructs Cloudinary URL for the image with text overlay parameters (font, color, size, positioning).  
  - Edge cases: URL encoding issues.

- **Data to send3**  
  - Type: Set  
  - Prepares payload with AI prompt and Cloudinary image URL for gotoHuman review.  
  - Includes metaJson for workflow traceability.  
  - Manages reviewToUpdate field depending on rerun state.  
  - Edge cases: Missing prior review IDs.

- **gotoHuman - Image**  
  - Type: gotoHuman node  
  - Sends reference image with tag line for human review and approval.  
  - Edge cases: API errors.

- **Approved or Retry ?**  
  - Type: Switch  
  - Routes based on approval: approved proceeds to video generation, chat triggers rerun of image generation.  
  - Edge cases: Unexpected types.

- **Rerun input**  
  - Type: Set  
  - Sets prompt and reviewToUpdate variables for retry of tagline generation on rerun.  
  - Edge cases: Missing prior data.

---

#### 1.4 Video Clip Generation

**Overview:**  
Using the approved reference image, this block creates a prompt to generate a short video clip with natural movements via Fal.ai, polls for completion, uploads the video to Cloudinary with text overlay, and sends the video for human review with retry capability.

**Nodes Involved:**  
- Set Initial Video Prompt (Code)  
- Set Video Request (Code)  
- Fal.ai Video Gen (HTTP Request)  
- Fetch Status (HTTP Request)  
- Is Ready?1 (If)  
- Wait (Wait)  
- Fetch Result (HTTP Request)  
- Cloudinary upload1 (HTTP Request)  
- Clip URL with Tag Line (Set)  
- Data to send4 (Set)  
- gotoHuman - Video  
- Approved or Retry  ? (Switch)  
- Rerun Video (Set)  
- Homework: Post it (NoOp)

**Node Details:**  

- **Set Initial Video Prompt**  
  - Type: Code  
  - Produces a prompt requesting natural movement effects (e.g. plants, shadows, insects) for video generation.  
  - Edge cases: Static prompt only.

- **Set Video Request**  
  - Type: Code  
  - Creates JSON payload with prompt and URL of approved reference image.  
  - Edge cases: Missing reference image URL.

- **Fal.ai Video Gen**  
  - Type: HTTP Request  
  - Sends video generation request to Fal.ai API with authentication header.  
  - Edge cases: API failures.

- **Fetch Status**  
  - Type: HTTP Request  
  - Polls Fal.ai API for video generation status.  
  - Edge cases: Network timeouts.

- **Is Ready?1**  
  - Type: If  
  - Checks if video generation status is "COMPLETED".  
  - Loops with Wait if not ready.  
  - Edge cases: Unexpected status.

- **Wait**  
  - Type: Wait  
  - Waits 1 second between status polling.  
  - Edge cases: Long delay.

- **Fetch Result**  
  - Type: HTTP Request  
  - Retrieves the generated video from Fal.ai.  
  - Edge cases: Broken link.

- **Cloudinary upload1**  
  - Type: HTTP Request  
  - Uploads video to Cloudinary with text overlay similar to image step.  
  - Uses environment and upload preset from Set Cloudinary options node.  
  - Edge cases: Upload errors.

- **Clip URL with Tag Line**  
  - Type: Set  
  - Constructs Cloudinary URL for the video with tag line overlay parameters.  
  - Edge cases: Encoding issues.

- **Data to send4**  
  - Type: Set  
  - Prepares payload with AI prompt, video URL, metaJson, and reviewToUpdate for gotoHuman.  
  - Edge cases: Missing review data.

- **gotoHuman - Video**  
  - Type: gotoHuman node  
  - For human review and approval of final video clip.  
  - Edge cases: API errors.

- **Approved or Retry  ?**  
  - Type: Switch  
  - Routes output based on human response: approved proceeds to posting, chat triggers rerun of video generation.  
  - Edge cases: Unexpected response.

- **Rerun Video**  
  - Type: Set  
  - Sets prompt and reviewToUpdate fields for retrying video generation.  
  - Edge cases: Missing prior data.

- **Homework: Post it**  
  - Type: NoOp  
  - Placeholder for manual or automated posting after final approval.  
  - Edge cases: None.

---

#### 1.5 Final Posting Preparation

**Overview:**  
This minimal block represents manual or external steps to publish the final approved video clip to social media platforms, receiving the video URL from gotoHuman.

**Nodes Involved:**  
- Homework: Post it (NoOp)

**Node Details:**  

- **Homework: Post it**  
  - Type: NoOp node  
  - Acts as an endpoint placeholder where the final video URL is available for posting.  
  - Users can add nodes here for social media API posting.  
  - Edge cases: None.

---

### 3. Summary Table

| Node Name                  | Node Type                              | Functional Role                                | Input Node(s)                    | Output Node(s)                   | Sticky Note Content                                                                                     |
|----------------------------|--------------------------------------|-----------------------------------------------|---------------------------------|---------------------------------|-------------------------------------------------------------------------------------------------------|
| Schedule Trigger           | Schedule Trigger                     | Weekly trigger to start workflow              |                                 | AI Agent: Generate Topic ideas   |                                                                                                       |
| Remote manual trigger      | Webhook                             | Manual trigger for workflow start             |                                 | AI Agent: Generate Topic ideas   |                                                                                                       |
| AI Agent: Generate Topic ideas | Langchain Agent                   | Generate topic ideas with context             | Schedule Trigger, Remote manual trigger | Data to send                   | ## 1. Generate campaign ideas We ask the LLM to generate ideas for our upcoming marketing campaign based on the current date/season. In gotoHuman we choose our favorite and approve. |
| Extract JSON               | Output Parser Structured            | Parse topic ideas JSON output                   | AI Agent: Generate Topic ideas  | Data to send                    |                                                                                                       |
| Data to send               | Set                                | Prepare payload with topics and metadata       | Extract JSON                    | gotoHuman - Topics              |                                                                                                       |
| gotoHuman - Topics         | gotoHuman                          | Human review and approval of topic ideas       | Data to send                    | Approved?                      |                                                                                                       |
| Approved?                  | If                                 | Check if topics are approved                    | gotoHuman - Topics              | Fetch approved examples          |                                                                                                       |
| Fetch approved examples    | HTTP Request                       | Get previous approved taglines from gotoHuman | Approved?                      | Tagline Prompt                  |                                                                                                       |
| Tagline Prompt             | Set                                | Compose prompt for tagline generation           | Fetch approved examples          | AI: Generate Tag Lines          | ## 2. Generate tag line (self-improving) & pick settings for image generation We are going to place a text label (tag line) on top of our video and we ask AI to generate it for us. To **allow our workflow to learn over time**, we are fetching previously approved tag lines from gotoHuman to incl. in our prompt as positive examples. In gotoHuman we edit the scenery of the image, pick the desired style and product to show. We can also manually edit the tag line OR ask AI to retry (we can change the prompt as well) |
| AI: Generate Tag Lines     | Langchain Agent                   | Generate marketing taglines                     | Tagline Prompt                 | Extract JSON1                   |                                                                                                       |
| Extract JSON1              | Output Parser Structured            | Parse tagline JSON output                        | AI: Generate Tag Lines         | Data to send2                   |                                                                                                       |
| Data to send2              | Set                                | Prepare payload with tagline and topic info     | Extract JSON1                  | gotoHuman - Content            |                                                                                                       |
| gotoHuman - Content        | gotoHuman                          | Human review of tagline and content settings    | Data to send2                  | Approved or Retry?             |                                                                                                       |
| Approved or Retry?         | Switch                            | Route based on human approval or retry request  | gotoHuman - Content            | Set Initial Image Prompt, Rerun input |                                                                                                    |
| Set Initial Image Prompt   | Code                              | Create detailed prompt for Fal.ai image gen     | Approved or Retry?             | Set Image Request              | ## 3. Generate reference product image Using the approved settings we ask Fal.ai to generate the image that our short video clip will be based on. We then use Cloudinary to put the tag line as a boxed label on top of it. In gotoHuman, we check the image and can ask for a retry. Clicking the chat button allows us to edit the prompt before retrying which is very useful with images. |
| Set Image Request          | Code                              | Format Fal.ai image generation request payload  | Set Initial Image Prompt       | Fal.ai Image Gen              |                                                                                                       |
| Fal.ai Image Gen           | HTTP Request                      | Request Fal.ai to generate product image        | Set Image Request              | Fetch Status1                 |                                                                                                       |
| Fetch Status1              | HTTP Request                      | Poll Fal.ai for image generation status          | Fal.ai Image Gen              | Is Ready?                    |                                                                                                       |
| Is Ready?                  | If                                 | Determine if image generation is completed       | Fetch Status1                 | Fetch Result1, Wait1          |                                                                                                       |
| Wait1                     | Wait                              | Wait 1 second before next status poll            | Is Ready? (no)                | Fetch Status1                 |                                                                                                       |
| Fetch Result1             | HTTP Request                      | Retrieve generated image from Fal.ai              | Is Ready? (yes)               | Set Cloudinary options        |                                                                                                       |
| Set Cloudinary options    | Set                                | Set Cloudinary environment and upload preset     | Fetch Result1                 | Cloudinary upload             | ## ⚠️ Instructions [ ] You need a cloudinary.com account [ ] In Cloudinary, find your environment/cloud name [ ] In Cloudinary, [add an upload preset](https://console.cloudinary.com/settings/upload/presets). Enter a name, set signing mode to **Unsigned** and enter asset folder name. Leave rest as is. Save. [ ] In this node, enter names of environment and upload preset (no credential req.) |
| Cloudinary upload         | HTTP Request                      | Upload generated image with tag line overlay      | Set Cloudinary options        | Image URL with Tag Line       |                                                                                                       |
| Image URL with Tag Line   | Set                                | Construct Cloudinary URL with text overlay        | Cloudinary upload             | Data to send3                 |                                                                                                       |
| Data to send3             | Set                                | Prepare payload for sending image to gotoHuman   | Image URL with Tag Line       | gotoHuman - Image            |                                                                                                       |
| gotoHuman - Image         | gotoHuman                          | Human review of generated image                    | Data to send3                 | Approved or Retry ?           |                                                                                                       |
| Approved or Retry ?       | Switch                            | Route based on image approval or retry             | gotoHuman - Image             | Set Initial Video Prompt, Rerun Image |                                                                                                    |
| R