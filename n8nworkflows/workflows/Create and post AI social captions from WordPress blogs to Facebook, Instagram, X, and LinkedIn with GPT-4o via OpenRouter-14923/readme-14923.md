Create and post AI social captions from WordPress blogs to Facebook, Instagram, X, and LinkedIn with GPT-4o via OpenRouter

https://n8nworkflows.xyz/workflows/create-and-post-ai-social-captions-from-wordpress-blogs-to-facebook--instagram--x--and-linkedin-with-gpt-4o-via-openrouter-14923


# Create and post AI social captions from WordPress blogs to Facebook, Instagram, X, and LinkedIn with GPT-4o via OpenRouter

### 1. Workflow Overview

The **"Create and post AI social captions from WordPress blogs to Facebook, Instagram, X, and LinkedIn"** workflow is an automated content distribution system. It triggers upon the publication of a WordPress post via a webhook, cleanses the blog content, uses an AI Agent (powered by GPT-4o via OpenRouter) to generate platform-optimized captions, and automatically publishes them across four major social networks.

The workflow logic is divided into the following functional blocks:
- **1.1 Input Reception & Validation:** Handles the incoming WordPress webhook and filters for published posts.
- **1.2 Data Pre-processing:** Converts HTML content to Markdown and organizes metadata into a clean structure.
- **1.3 AI Content Generation:** Uses a Large Language Model (LLM) to generate specific copy for Twitter, Facebook, Instagram, and LinkedIn based on strict stylistic guidelines.
- **1.4 Multi-Channel Distribution:** Executes the API calls to post the generated content and images to the respective social platforms.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Validation
**Overview:** This block acts as the gateway, ensuring only valid "published posts" from WordPress trigger the subsequent automation.
- **Nodes Involved:** `Webhook`, `If`
- **Node Details:**
    - **Webhook:** 
        - *Role:* Trigger.
        - *Configuration:* Listens for `POST` requests on a specific unique path.
        - *Input/Output:* Receives raw JSON from WordPress $\rightarrow$ passes to `If`.
    - **If:** 
        - *Role:* Logic Gate.
        - *Configuration:* Checks two conditions: `post_type` must equal `post` AND `post_status` must equal `publish`.
        - *Failure Case:* If conditions aren't met, the workflow stops here to avoid posting drafts or pages.

#### 2.2 Data Pre-processing
**Overview:** Transforms raw WordPress data into a format that is readable for an AI and structured for the final API requests.
- **Nodes Involved:** `HTML To Markdown`, `Clean Data`
- **Node Details:**
    - **HTML To Markdown:** 
        - *Role:* Text Converter.
        - *Configuration:* Takes `post_content` from the webhook and converts HTML tags to Markdown for better LLM token efficiency.
    - **Clean Data (Set Node):** 
        - *Role:* Data Normalization.
        - *Configuration:* Maps various fields to fixed keys: `post_title`, `post_content`, `post_link`, `post_thumbnail`, `facebook_page_id`, and `instagram_id`.
        - *Key Variables:* Hardcoded Page IDs for FB and IG are stored here for easy modification.

#### 2.3 AI Processing
**Overview:** The "brain" of the workflow. It interprets the blog content and creates tailored marketing copy for different audiences.
- **Nodes Involved:** `AI Agent`, `OpenRouter Chat Model`, `Structured Output Parser`
- **Node Details:**
    - **AI Agent:** 
        - *Role:* Orchestrator.
        - *Configuration:* Uses a detailed system prompt defining roles (Professional Social Media Writer) and strict constraints for each platform (e.g., X: <220 chars, LinkedIn: professional tone 800-1200 chars).
        - *Input:* Receives the cleaned post title, link, and a slice of the content (first 800 characters).
    - **OpenRouter Chat Model:** 
        - *Role:* LLM Provider.
        - *Configuration:* Model set to `openai/gpt-4o`.
    - **Structured Output Parser:** 
        - *Role:* Data Formatter.
        - *Configuration:* Ensures the AI returns a valid JSON object with keys: `twitter`, `facebook`, `instagram`, and `linkedin`.

#### 2.4 Social Media Publishing
**Overview:** Distributes the AI-generated captions to the social APIs.
- **Nodes Involved:** `Create Tweet`, `Post on FB`, `Create Media Container`, `Post On IG`, `HTTP Request`, `Create a post`
- **Node Details:**
    - **Create Tweet:** Sends `output.twitter` to X via OAuth2.
    - **Post on FB:** Uses `httpRequest` to the Graph API (`/photos` endpoint) combining the caption and the thumbnail URL.
    - **Create Media Container $\rightarrow$ Post On IG:** A two-step process. First, it uploads the image to the `/media` endpoint to get a `creation_id`, then it calls `/media_publish` to make it live.
    - **HTTP Request $\rightarrow$ Create a post (LinkedIn):** Fetches the thumbnail image and then posts the professional caption to a LinkedIn organization page.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Webhook | Webhook | Entry point | None | If | Webhook Trigger and Condition |
| If | If | Filter for published posts | Webhook | HTML To Markdown | Webhook Trigger and Condition |
| HTML To Markdown | Markdown | Convert HTML to text | If | Clean Data | Markdown Conversion and Clean Data |
| Clean Data | Set | Structure post metadata | HTML To Markdown | AI Agent | Markdown Conversion and Clean Data |
| AI Agent | AI Agent | Generate social captions | Clean Data | Tweet, FB, IG, LinkedIn | AI Processing |
| OpenRouter Chat Model | LM Chat | Provide GPT-4o intelligence | None | AI Agent | AI Processing |
| Structured Output Parser | Output Parser | Force JSON format | None | AI Agent | AI Processing |
| Create Tweet | Twitter | Post to X | AI Agent | None | Social Media Publishing |
| Post on FB | HTTP Request | Post image + text to FB | AI Agent | None | Social Media Publishing |
| Create Media Container | HTTP Request | Upload image to IG | AI Agent | Post On IG | Social Media Publishing |
| Post On IG | HTTP Request | Publish IG media | Create Media Container | None | Social Media Publishing |
| HTTP Request | HTTP Request | Prepare image for LinkedIn | AI Agent | Create a post | LinkedIn Posting |
| Create a post | LinkedIn | Post to LinkedIn Org | HTTP Request | None | LinkedIn Posting |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Trigger & Filtering
1. Create a **Webhook** node. Set HTTP Method to `POST`. Note the production URL to plug into WordPress.
2. Connect an **If** node. Add two string conditions:
   - `{{ $json.body.post.post_type }}` equals `post`
   - `{{ $json.body.post.post_status }}` equals `publish`

#### Step 2: Data Cleaning
3. Create a **Markdown** node. Set the HTML input to `{{ $('Webhook').item.json.body.post.post_content }}`.
4. Create a **Set** node (`Clean Data`). Define the following string variables:
   - `post_title`: From Webhook body.
   - `post_content`: From Markdown node.
   - `post_link`: From Webhook body (`post_permalink`).
   - `post_thumbnail`: From Webhook body.
   - `facebook_page_id`: (Your actual FB Page ID).
   - `instagram_id`: (Your actual IG Business ID).

#### Step 3: AI Configuration
5. Create an **AI Agent** node.
6. Attach an **OpenRouter Chat Model** node; configure credentials and select `openai/gpt-4o`.
7. Attach a **Structured Output Parser**; set the JSON schema to include keys for `twitter`, `facebook`, `instagram`, and `linkedin`.
8. In the Agent's **System Message**, paste the detailed role and platform guidelines (character limits and tone) provided in the original JSON.
9. Set the Agent's **Prompt** to: `Post Title: {{ $json.post_title }} | Link: {{ $json.post_link }} | Content: {{ $json.post_content.slice(0, 800) }}`.

#### Step 4: Social Distribution
10. **X (Twitter):** Add a **Twitter** node. Map the text to `{{ $json.output.twitter }}`.
11. **Facebook:** Add an **HTTP Request** node. 
    - URL: `https://graph.facebook.com/v25.0/{{ $('Clean Data').item.json.facebook_page_id }}/photos`
    - Method: `POST`. 
    - Query Params: `message` ($\text{AI output}$) and `url` ($\text{post\_thumbnail}$).
12. **Instagram:** 
    - Add an **HTTP Request** node to `/media` endpoint with `caption` and `image_url`.
    - Connect this to a second **HTTP Request** node to `/media_publish` using `creation_id` from the previous node.
13. **LinkedIn:** 
    - Add an **HTTP Request** to fetch the thumbnail image.
    - Connect to a **LinkedIn** node. Set `Post As` to `organization`, provide the Org URN, and map the text to `{{ $json.output.linkedin }}`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Setup requires Webhook URL configuration in WordPress | Integration Setup |
| API credentials required for: OpenRouter, X, LinkedIn, and Facebook Graph API | Authentication |
| Content is sliced to 800 characters to avoid LLM context window overflow/cost | AI Optimization |
| Ensure Instagram account is a Business Account linked to a Facebook Page | Instagram API requirement |