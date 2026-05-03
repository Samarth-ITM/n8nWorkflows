Generate social media posts with GPT-4 for LinkedIn, X, and Facebook

https://n8nworkflows.xyz/workflows/generate-social-media-posts-with-gpt-4-for-linkedin--x--and-facebook-13648


# Generate social media posts with GPT-4 for LinkedIn, X, and Facebook

# AI Social Media Content Generator Reference Document

### 1. Workflow Overview
The **AI Social Media Content Generator** is an automated pipeline designed to create, optimize, and publish marketing content across LinkedIn, Twitter (X), and Facebook. It uses GPT-4 to generate high-quality posts based on rotating industry topics and then applies platform-specific logic (character limits, hashtag density, and tone adjustments) to ensure maximum engagement on each social network.

**Logical Blocks:**
- **1.1 Input & Strategy Selection:** Triggers the workflow and randomly selects a content category (e.g., Product Launch, Thought Leadership) to ensure variety.
- **1.2 AI Content Generation:** Interfaces with OpenAI to generate the raw marketing copy.
- **1.3 Platform Optimization:** Tailors the AI output into three distinct versions (LinkedIn, X, and Facebook).
- **1.4 Multi-Platform Publishing:** Disseminates the content to the respective social APIs.
- **1.5 Data Persistence & Notification:** Logs the execution results into a PostgreSQL database and alerts the team via Slack with live links.

---

### 2. Block-by-Block Analysis

#### 1.1 Input & Strategy Selection
- **Overview:** Defines when the workflow runs and selects the creative direction for the day.
- **Nodes Involved:** `Daily content generation at 10 AM`, `Select content topic and type`.
- **Node Details:**
    - **Daily content generation at 10 AM:** A Schedule Trigger set to run daily at 10:00 AM via cron expression `0 10 * * *`.
    - **Select content topic and type:** A Code node that picks a random topic (Product Launch, Industry Insights, etc.) and a content type (Educational, Promotional, etc.). It also generates a unique `postId` and formatted timestamps.

#### 1.2 AI Content Generation
- **Overview:** Sends the selected topic and tone to OpenAI to produce the base marketing copy.
- **Nodes Involved:** `Generate content with OpenAI GPT-4`, `Parse AI response and extract content`.
- **Node Details:**
    - **Generate content with OpenAI GPT-4:** An HTTP Request node using `gpt-4`. It uses a system prompt defining the AI as a marketing expert and a user prompt injecting variables from the previous node (`$json.contentType`, `$json.category`, etc.).
    - **Parse AI response and extract content:** A Code node that cleans the AI output, extracts hashtags using regex, and calculates word counts.

#### 1.3 Platform Optimization
- **Overview:** Transforms one piece of content into three platform-ready versions.
- **Nodes Involved:** `Optimize content for each platform`, `Split content for each platform`.
- **Node Details:**
    - **Optimize content for each platform:** Applies logic: LinkedIn (up to 1200 chars, 5 hashtags), X (280 chars limit, 3 hashtags, sentence truncation), and Facebook (1500 chars, emojis based on tone).
    - **Split content for each platform:** Uses a Split Out node to turn the single item containing three platform objects into three separate items for parallel processing.

#### 1.4 Multi-Platform Publishing
- **Overview:** Routes the optimized content to the specific social media APIs.
- **Nodes Involved:** `Route to appropriate platform`, `Post to LinkedIn`, `Post to Twitter/X`, `Post to Facebook`.
- **Node Details:**
    - **Route to appropriate platform:** A Switch node that directs items based on the value of `{{ $json.platform }}`.
    - **Post to LinkedIn:** Uses the `ugcPosts` API. Requires a valid LinkedIn URN (`urn:li:person:YOUR_LINKEDIN_ID`).
    - **Post to Twitter/X:** Uses the Twitter v2 `tweets` endpoint.
    - **Post to Facebook:** Uses the Facebook Graph API `me/feed` endpoint.

#### 1.5 Data Persistence & Notification
- **Overview:** Collects results from all posts, saves them to a database, and notifies the user.
- **Nodes Involved:** `Aggregate all platform results`, `Prepare database record`, `Store post record in PostgreSQL`, `Prepare Slack notification`, `Send confirmation to Slack`, `Log success and update statistics`.
- **Node Details:**
    - **Aggregate all platform results:** Merges the individual responses from the three social media nodes back into a single array.
    - **Prepare database record:** Consolidates success/failure statuses and generates live URLs for the published posts.
    - **Store post record in PostgreSQL:** Inserts the final payload into a table named `social_posts`.
    - **Send confirmation to Slack:** Sends a formatted Markdown block with post links and performance stats to a Slack Webhook.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Daily content generation at 10 AM | Schedule Trigger | Workflow Entry | (None) | Select content topic... | 1. Trigger content generation |
| Select content topic and type | Code | Strategy Selection | Daily content generation... | Generate content... | 1. Trigger content generation |
| Generate content with OpenAI GPT-4 | HTTP Request | Content Creation | Select content topic... | Parse AI response... | 2. Generate & optimize content |
| Parse AI response and extract content | Code | Data Cleaning | Generate content... | Optimize content... | 2. Generate & optimize content |
| Optimize content for each platform | Code | Content Adaptation | Parse AI response... | Split content... | 2. Generate & optimize content |
| Split content for each platform | Split Out | Data Routing | Optimize content... | Route to appropriate... | 3. Publish to platforms |
| Route to appropriate platform | Switch | Path Logic | Split content... | Post to [Platforms] | 3. Publish to platforms |
| Post to LinkedIn | HTTP Request | Publishing | Route to appropriate... | Aggregate all... | 3. Publish to platforms |
| Post to Twitter/X | HTTP Request | Publishing | Route to appropriate... | Aggregate all... | 3. Publish to platforms |
| Post to Facebook | HTTP Request | Publishing | Route to appropriate... | Aggregate all... | 3. Publish to platforms |
| Aggregate all platform results | Aggregate | Result Gathering | Post to [Platforms] | Prepare database... | 4. Track & notify |
| Prepare database record | Code | Data Mapping | Aggregate all... | Store post... / Prepare Slack... | 4. Track & notify |
| Store post record in PostgreSQL | Postgres | Persistence | Prepare database... | Log success... | 4. Track & notify |
| Prepare Slack notification | Code | Notification Formatting | Prepare database... | Send confirmation... | 4. Track & notify |
| Send confirmation to Slack | HTTP Request | Notification | Prepare Slack... | (None) | 4. Track & notify |
| Log success and update statistics | Code | Internal Logging | Store post... | (None) | 4. Track & notify |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:** Create a **Schedule Trigger** set to `0 10 * * *`.
2.  **Strategy Node:** Add a **Code Node** with an array of objects containing `category`, `focus`, and `tone`. Use `Math.random()` to output one item.
3.  **AI Node:** Add an **HTTP Request** node.
    - Method: `POST`, URL: `https://api.openai.com/v1/chat/completions`.
    - Authentication: OpenAI API Key.
    - Body: Send `model: gpt-4` and a system/user prompt using expressions from the Strategy Node.
4.  **Parsing Node:** Add a **Code Node** to extract `choices[0].message.content` and use regex `/ #\w+/g` to pull hashtags into an array.
5.  **Optimization Node:** Add a **Code Node**. Use `if/else` or `switch` logic to create three new properties in the JSON: `linkedin`, `twitter`, and `facebook`. Apply `.substring()` limits for X (280) and LinkedIn (1200).
6.  **Splitting:** Add a **Split Out** node targeting the platform objects created in the previous step.
7.  **Routing:** Add a **Switch** node checking if `platform` equals "LinkedIn", "Twitter/X", or "Facebook".
8.  **API Connectors:**
    - **LinkedIn:** HTTP Request to `api.linkedin.com/v2/ugcPosts`. Requires your LinkedIn URN.
    - **X (Twitter):** HTTP Request to `api.twitter.com/2/tweets`.
    - **Facebook:** HTTP Request to `graph.facebook.com/v18.0/me/feed`.
9.  **Aggregation:** Add an **Aggregate** node to wait for all three social nodes to complete.
10. **Storage & Notification:**
    - Create a **PostgreSQL** node to insert the summary into a `social_posts` table.
    - Create an **HTTP Request** node for Slack, passing a formatted `text` string containing the post links.
11. **Final Stats:** Use a **Code Node** with `$getWorkflowStaticData('global')` to increment counters for total posts and successful platforms.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Prerequisite:** Create `social_posts` table in Postgres | Required for the "Store post record" node to function. |
| **Manual Review:** Optional step | You can insert an "Edit" or "Wait" node before publishing for human approval. |
| **API Limits:** Twitter/X Rate Limits | Ensure the workflow doesn't trigger more than your tier allows (Free vs Basic). |
| **Credential Setup:** LinkedIn OAuth2 | Requires `w_member_social` permissions to post content. |