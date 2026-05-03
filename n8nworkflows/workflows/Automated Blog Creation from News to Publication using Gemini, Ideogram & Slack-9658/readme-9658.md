Automated Blog Creation from News to Publication using Gemini, Ideogram & Slack

https://n8nworkflows.xyz/workflows/automated-blog-creation-from-news-to-publication-using-gemini--ideogram---slack-9658


# Automated Blog Creation from News to Publication using Gemini, Ideogram & Slack

---

### 1. Workflow Overview

**Purpose:**  
This workflow automates the creation and publication of SEO-optimized blog posts by analyzing current industry news, generating relevant blog topics and full content via Google Gemini AI, creating a featured image using Replicate’s Ideogram model, publishing to Supabase CMS, and notifying the team on Slack. It runs on a scheduled daily basis, providing a hands-free content pipeline for marketing or SaaS teams.

**Target Use Cases:**  
- Automated content marketing for SaaS, AI, automation, and related industries  
- Generating trending blog posts from live news data  
- Streamlining blog publishing with AI-generated images and Slack notifications  
- Teams using Supabase or custom CMS APIs with integrations to Slack  

**Logical Blocks:**  

- **1.1 Scheduled Trigger & News Fetching:** Starts daily, fetches latest news articles relevant to AI, automation, and SaaS from NewsAPI.  
- **1.2 AI Topic Generation (Google Gemini):** Sends trending articles to Gemini for extracting a trending blog topic idea in JSON format.  
- **1.3 AI Blog Content Creation (Google Gemini):** Uses the selected topic to generate a full-length SEO-optimized blog post in Markdown with metadata.  
- **1.4 Output Cleaning & Parsing (Code Node):** Cleans and normalizes Gemini’s sometimes malformed or noisy JSON output to structured fields.  
- **1.5 Image Generation (Replicate Ideogram):** Uses the blog’s image prompt to generate a 1024×1024 featured image, polling until the image is ready.  
- **1.6 Final Assembly & Publishing:** Combines cleaned blog content and image URL, publishes to Supabase CMS, then posts a notification to Slack.  

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & News Fetching

**Overview:**  
Triggers the workflow daily at 10 AM IST and fetches the latest 10 news articles about “AI automation SaaS” from NewsAPI for use as contextual input.

**Nodes Involved:**  
- Schedule Trigger  
- Fetch Industry Trends (HTTP Request)

**Node Details:**  

- **Schedule Trigger**  
  - Type: Cron Trigger  
  - Configuration: Runs daily at hour 10 (10 AM IST)  
  - Input: None (trigger node)  
  - Output: Single trigger event to start workflow  
  - Edge Cases: Workflow not triggered if n8n server offline or misconfigured timezone  

- **Fetch Industry Trends**  
  - Type: HTTP Request  
  - Purpose: GET request to NewsAPI endpoint with query parameters: `q=AI automation SaaS`, `sortBy=publishedAt`, `pageSize=10`  
  - Authentication: Generic HTTP Query Auth with NewsAPI key  
  - Output: JSON containing latest news articles  
  - Edge Cases: API key limit exceeded, network errors, empty or malformed response  
  - Notes: Adjust `q` and `pageSize` to customize for other industries or volume  

---

#### 2.2 AI Topic Generation (Google Gemini)

**Overview:**  
Sends the fetched news articles to Google Gemini AI to generate one trending blog topic idea relevant to the company’s domain, returning a strict JSON with title, description, and keywords.

**Nodes Involved:**  
- Message a model (Google Gemini)

**Node Details:**  

- **Message a model**  
  - Type: Google Gemini (LangChain node)  
  - Configuration: Model `gemini-2.5-flash`, prompt instructs AI to analyze input articles and generate one trending blog topic idea as strict JSON  
  - Input: JSON from NewsAPI’s articles field  
  - Output: JSON with blog topic fields: title, description, keywords  
  - Credentials: Google Gemini (PaLM) API key  
  - Edge Cases: API quota exceeded, malformed or unexpected JSON output (handled downstream)  
  - Notes: Prompt can be customized for other industries or output formats  

---

#### 2.3 AI Blog Content Creation (Google Gemini)

**Overview:**  
Generates a detailed 1200–1500 word SEO-optimized blog post in Markdown on the selected topic, including SEO title, meta description, excerpt, keywords, and an image prompt, outputting structured JSON.

**Nodes Involved:**  
- Message a model1 (Google Gemini)

**Node Details:**  

- **Message a model1**  
  - Type: Google Gemini (LangChain node)  
  - Configuration: Same model as above, prompt instructs AI to write a full blog post with SEO elements in JSON format  
  - Input: Topic title from previous node’s JSON output  
  - Output: JSON with fields: title, slug, excerpt, content (Markdown), keywords, image_prompt  
  - Credentials: Google Gemini (PaLM) API key  
  - Edge Cases: Inconsistent JSON or extra Markdown noise (handled by next node)  
  - Notes: You can modify prompt to shorten or adapt content style  

---

#### 2.4 Output Cleaning & Parsing (Code Node)

**Overview:**  
Processes Gemini’s output to clean Markdown fences, repair malformed JSON, extract required fields, normalize slugs, and ensure consistent structured data for downstream nodes.

**Nodes Involved:**  
- Code in JavaScript

**Node Details:**  

- **Code in JavaScript**  
  - Type: Code (JavaScript)  
  - Purpose:  
    - Strips Markdown code fences and language tags  
    - Extracts innermost JSON blocks, attempts parsing and repair  
    - Uses regex fallback to extract missing fields  
    - Normalizes slug (lowercase, hyphens), trims quotes, and provides default titles if empty  
  - Input: Gemini’s raw JSON text output  
  - Output: Cleaned JSON with fields: title, slug, excerpt, content, keywords, image_prompt  
  - Edge Cases: Empty AI output, malformed JSON, unexpected text noise  
  - Notes: Robust and safe to modify if AI prompt output schema changes  

---

#### 2.5 Image Generation (Replicate Ideogram)

**Overview:**  
Uses the `image_prompt` from the cleaned blog content to generate a 1024×1024 featured image via Replicate’s Ideogram v3 Turbo model, with polling and wait to handle asynchronous generation.

**Nodes Involved:**  
- HTTP Request (Replicate – Generate Image)  
- HTTP Request1 (Replicate Polling)  
- Wait  
- If

**Node Details:**  

- **HTTP Request (Replicate – Generate Image)**  
  - Type: HTTP Request (POST)  
  - Purpose: Calls Replicate API to start image generation with prompt, width, height  
  - Authentication: Bearer Token (Replicate API key)  
  - Output: Prediction object including `id` and URLs for polling  
  - Edge Cases: API rate limits, network errors  

- **HTTP Request1 (Replicate Polling)**  
  - Type: HTTP Request (GET)  
  - Purpose: Polls Replicate API using prediction URL to get current status and image URLs  
  - Authentication: Same Bearer Token  
  - Output: Status field (`succeeded` or other) and image URLs  
  - Edge Cases: Polling too fast causing rate limits, job failure states  

- **Wait**  
  - Type: Wait node  
  - Purpose: Delays 20 seconds before next poll to avoid throttling and allow image generation time  
  - Configurable wait time depending on model speed  

- **If**  
  - Type: Conditional node  
  - Purpose: Checks if Replicate image status is `succeeded` to continue workflow  
  - If not succeeded, loops back to poll again  

---

#### 2.6 Final Assembly & Publishing

**Overview:**  
Combines the cleaned blog content fields with the generated image URL, publishes the blog post to Supabase CMS via a custom API endpoint, and sends a notification message to a Slack channel with blog details.

**Nodes Involved:**  
- Edit Fields (Set node)  
- Publish to Supabase (HTTP Request)  
- Slack Notification (Slack node)

**Node Details:**  

- **Edit Fields**  
  - Type: Set node  
  - Purpose: Constructs final JSON payload with `title`, `slug`, `excerpt`, `content`, `image_url` (from generated image), and `published: true` flag  
  - Input: Cleaned JSON + image URL from polling response  
  - Output: Final blog post JSON ready for publishing  
  - Edge Cases: Missing image URL or incomplete fields (should be handled by upstream nodes)  

- **Publish to Supabase**  
  - Type: HTTP Request (POST)  
  - Purpose: Sends final blog JSON to Supabase Edge Function at `/functions/v1/blog-api` for publishing  
  - Authentication: Header Auth with Supabase API key  
  - Edge Cases: API failures, authorization errors, data validation failures by backend  

- **Slack Notification**  
  - Type: Slack node  
  - Purpose: Sends confirmation message to Slack channel (default `#blog-automation`) including blog title, slug, and URL  
  - Credentials: Slack API token with `chat:write` scope  
  - Edge Cases: Token invalid, channel permission issues, message formatting errors  

---

### 3. Summary Table

| Node Name                        | Node Type                     | Functional Role                               | Input Node(s)                   | Output Node(s)                 | Sticky Note                                                                                                 |
|---------------------------------|-------------------------------|----------------------------------------------|--------------------------------|------------------------------|-------------------------------------------------------------------------------------------------------------|
| Schedule Trigger                | Cron Trigger                  | Triggers workflow daily at 10 AM IST          | None                           | Fetch Industry Trends          | See Sticky Note11: General notes on credentials, error handling, customization, runtime, timezone settings. |
| Fetch Industry Trends           | HTTP Request                  | Fetches latest 10 news articles from NewsAPI | Schedule Trigger               | Message a model               | Sticky Note: Detailed info on NewsAPI usage and credentials.                                                |
| Message a model                | Google Gemini (LangChain)     | Generates 1 trending blog topic in JSON       | Fetch Industry Trends          | Message a model1              | Sticky Note1: Purpose and credential info for Gemini topic generation.                                      |
| Message a model1               | Google Gemini (LangChain)     | Generates full SEO blog content in JSON       | Message a model               | Code in JavaScript            | Sticky Note2: Purpose and prompt details for SEO blog content generation.                                   |
| Code in JavaScript             | Code Node (JavaScript)        | Cleans and normalizes Gemini JSON output      | Message a model1              | HTTP Request (Replicate)      | Sticky Note3: Details on the resilient JSON cleaning and parsing approach.                                 |
| HTTP Request (Replicate – Generate Image) | HTTP Request                  | Starts image generation with Ideogram model  | Code in JavaScript            | HTTP Request1 (Replicate Polling) | Sticky Note4: Info on Replicate API call and credentials.                                                  |
| HTTP Request1 (Replicate Polling) | HTTP Request                  | Polls image generation status from Replicate | HTTP Request (Replicate)      | Wait / If                    | Sticky Note6: Polling purpose and credential notes for Replicate.                                         |
| Wait                          | Wait Node                     | Waits 20 seconds before polling again         | HTTP Request1 (Replicate Polling) | If                          | Sticky Note5: Reasoning for wait time and rate limit avoidance.                                           |
| If                            | If Node                       | Checks if image generation succeeded          | Wait                         | Edit Fields / HTTP Request1 (Replicate Polling) | Sticky Note7: Logic for looping or continuing based on success.                                            |
| Edit Fields                   | Set Node                     | Combines blog fields and image URL for publish| If                          | Publish to Supabase          | Sticky Note8: Merging fields for final payload and adding published flag.                                 |
| Publish to Supabase           | HTTP Request                  | Publishes blog post to Supabase CMS            | Edit Fields                  | Slack Notification           | Sticky Note9: Publishing endpoint and credential setup details.                                           |
| Slack Notification            | Slack Node                   | Sends Slack message notifying blog publication | Publish to Supabase          | None                        | Sticky Note10: Slack message formatting and credential requirements.                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node**  
   - Type: Cron Trigger  
   - Set to run daily at 10 AM (hour: 10) in your timezone (adjust if needed).  

2. **Add HTTP Request Node: Fetch Industry Trends**  
   - Method: GET  
   - URL: `https://newsapi.org/v2/everything?q=AI automation SaaS&sortBy=publishedAt&pageSize=10`  
   - Authentication: Generic HTTP Query Auth with NewsAPI API key  
   - Save credentials with your NewsAPI key  
   - Connect output of Schedule Trigger to this node  

3. **Add Google Gemini Node: Message a model**  
   - Use LangChain Google Gemini node  
   - Model: `models/gemini-2.5-flash`  
   - Prompt: Instruct AI to analyze news articles (`{{$json.articles}}`) and generate exactly one trending blog topic as JSON with title, description, keywords  
   - Credentials: Google Gemini API key  
   - Connect output of Fetch Industry Trends to this node  

4. **Add Google Gemini Node: Message a model1**  
   - Same model as above  
   - Prompt: Write a 1200–1500 word SEO blog in Markdown on the topic title from previous node  
   - Output JSON fields: title, slug, excerpt, content, keywords, image_prompt  
   - Credentials: Same Google Gemini API key  
   - Connect output of Message a model to this node  

5. **Add Code Node: JavaScript for Cleaning**  
   - Paste provided JavaScript code that:  
     - Strips markdown fences  
     - Parses nested JSON or repairs malformed JSON  
     - Extracts and normalizes fields: title, slug, excerpt, content, keywords, image_prompt  
   - Connect output of Message a model1 to this node  

6. **Add HTTP Request Node: Replicate Image Generation**  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/models/ideogram-ai/ideogram-v3-turbo/predictions`  
   - Body (JSON):  
     ```json
     {
       "input": {
         "prompt": "{{ $json.image_prompt }}",
         "width": 1024,
         "height": 1024
       }
     }
     ```  
   - Authentication: Bearer Token with Replicate API key  
   - Connect output of Code Node to this node  

7. **Add HTTP Request Node: Replicate Polling**  
   - Method: GET  
   - URL: `={{ $json.urls.get }}` (dynamic URL from previous response)  
   - Authentication: Same Bearer Token  
   - Connect output of Replicate Image Generation node to this node  

8. **Add Wait Node**  
   - Wait for 20 seconds  
   - Connect output of Replicate Polling node to this node  

9. **Add If Node**  
   - Condition: Check if `{{$json.status}}` equals `succeeded`  
   - True branch: Connect to Edit Fields node  
   - False branch: Loop back to Replicate Polling node (to continue polling)  

10. **Add Set Node: Edit Fields**  
    - Mode: Raw JSON  
    - Set fields:  
      - `title` from Code Node output  
      - `slug` from Code Node output  
      - `excerpt` from Code Node output  
      - `content` from Code Node output  
      - `image_url` from Replicate Polling output (`$json.output`)  
      - `published` set to `true`  
    - Connect True output of If node to this node  

11. **Add HTTP Request Node: Publish to Supabase**  
    - Method: POST  
    - URL: Your Supabase function endpoint, e.g., `https://your.supabase.co/functions/v1/blog-api`  
    - Send body: JSON from Edit Fields node  
    - Authentication: Header Auth with Supabase API key (Bearer token)  
    - Connect output of Edit Fields node  

12. **Add Slack Node: Slack Notification**  
    - Send message to a Slack channel (e.g., `#blog-automation`)  
    - Message text example:  
      ```
      ✅ *New Blog Published!*
      *Title:* {{$json.title}}
      *Slug:* {{$json.slug}}
      *URL:* https://yoururl.com/blog/{{$json.slug}}
      ```  
    - Credentials: Slack API token with `chat:write` permission  
    - Connect output of Publish to Supabase node  

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                |
|----------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| All credentials (NewsAPI, Google Gemini, Replicate, Supabase, Slack) must be created before running this workflow.  | See Sticky Note11 for details                                 |
| Workflow runtime typically 1–2 minutes, mostly waiting on image generation latency.                                  | Timing depends on Replicate model speed                        |
| You can customize the NewsAPI query to fit your industry keywords.                                                  | Change `q` parameter in URL                                    |
| The Code Node JavaScript is highly resilient to malformed AI output and can be safely modified for custom fields.   | See Sticky Note3                                              |
| Supabase endpoint can be replaced by any CMS or API accepting structured blog post JSON.                             | See Sticky Note9                                              |
| Slack messages can be customized with Block Kit or different channels.                                              | See Sticky Note10                                             |
| The Replicate model `ideogram-ai/ideogram-v3-turbo` can be swapped with other text-to-image models like Stable Diffusion. | See Sticky Note4                                              |
| Workflow timezone is set to Asia/Kolkata by default; adjust as needed in workflow settings.                         | Workflow settings                                              |
| For API quotas or limits, monitor NewsAPI, Gemini, and Replicate usage to avoid failures.                            | Credential providers’ dashboards                              |
| The workflow can be extended to post to LinkedIn, Twitter, or email subscribers by adding nodes after Slack.        | Customization option                                          |

---

**Disclaimer:**  
The text provided is exclusively generated from an automated workflow created with n8n, a workflow automation tool. The processing strictly follows content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.

---