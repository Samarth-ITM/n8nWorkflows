Classify YouTube Videos & Generate Email Summaries with GPT-4 and Gmail

https://n8nworkflows.xyz/workflows/classify-youtube-videos---generate-email-summaries-with-gpt-4-and-gmail-9616


# Classify YouTube Videos & Generate Email Summaries with GPT-4 and Gmail

### 1. Workflow Overview

This workflow automates the monitoring of selected YouTube channels to identify trending videos and generate professional weekly email briefings. It targets content curators, social media strategists, AI engineers, and system integrators who want to track YouTube video trends, classify videos by popularity, and communicate insights efficiently via email.

The workflow consists of the following logical blocks:

**1.1 Setup & Input Reception**  
Manual trigger, channel ID input, and initial splitting to handle multiple YouTube channels.

**1.2 Data Acquisition**  
Fetching latest videos via YouTube RSS feeds, filtering recent uploads (last 72 hours), and retrieving detailed video statistics from the YouTube Data API.

**1.3 Video Classification**  
Analyzing videos by like count to determine viral status using a threshold, branching into viral vs. normal video flows.

**1.4 AI-Based Content Generation**  
Generating LinkedIn-style social media posts for each video classification using OpenAI GPT-4 prompts tailored for viral or normal videos.

**1.5 Aggregation & Weekly Briefing Assembly**  
Compiling all generated posts into a consolidated text and producing a detailed HTML weekly briefing email with GPT-5, designed to be visually robust for Gmail.

**1.6 Email Delivery**  
Sending the finalized briefing via Gmail OAuth2.

---

### 2. Block-by-Block Analysis

#### 2.1 Setup & Input Reception

- **Overview:**  
  This block starts the workflow manually, sets the YouTube channel IDs to monitor, and splits them for individual processing.

- **Nodes Involved:**  
  Manual Run, Set Channel IDs, Split Channels, Sticky Notes related to setup and channel IDs.

- **Node Details:**

  - **Manual Run**  
    - Type: Manual Trigger  
    - Role: Initiates the workflow on demand.  
    - Configuration: Default, no parameters.  
    - Input: None  
    - Output: Triggers "Set Channel IDs" node.  
    - Edge Cases: None significant.

  - **Set Channel IDs**  
    - Type: Set  
    - Role: Defines an array of YouTube Channel IDs to process.  
    - Configuration: Assigns a field `ChannelID` as an array of four channel IDs (strings starting with "UC...").  
    - Key variables: `ChannelID` array.  
    - Input: Trigger from Manual Run.  
    - Output: Sends to "Split Channels".  
    - Edge Cases: Empty or invalid channel IDs will cause no videos to be fetched.

  - **Split Channels**  
    - Type: SplitOut  
    - Role: Splits the array of Channel IDs into individual workflow items for parallel processing.  
    - Configuration: Splits on field `ChannelID`.  
    - Input: From "Set Channel IDs".  
    - Output: One item per channel to "Fetch Latest Videos (RSS)".  
    - Edge Cases: If ChannelID is empty or malformed, no output items.

  - **Sticky Notes**  
    - Provide critical instructions such as:  
      - "⚠️ IMPORTANT: Provide the YouTube Channel IDs in the Set node"  
      - "Title: Setup — Your YouTube Channels"  
      - "Enter the Channel IDs (UC…) you want to monitor"  
    - These do not affect runtime but guide users for proper configuration.

---

#### 2.2 Data Acquisition

- **Overview:**  
  Fetches the latest videos from each YouTube channel via RSS feed, then filters to keep only those published within the last 72 hours. For filtered videos, it fetches detailed video statistics (title, description, like counts) via YouTube Data API.

- **Nodes Involved:**  
  Fetch Latest Videos (RSS), Filter: Published in Last 72h, Get Video Stats (YouTube API), Sticky Notes about API key and YouTube API configuration.

- **Node Details:**

  - **Fetch Latest Videos (RSS)**  
    - Type: RSS Feed Read  
    - Role: Retrieves the latest videos feed for a given YouTube channel.  
    - Configuration: URL dynamically constructed as `https://www.youtube.com/feeds/videos.xml?channel_id={{ $json.ChannelID }}`.  
    - Input: From "Split Channels" (one per channel).  
    - Output: Sends to "Filter: Published in Last 72h".  
    - Edge Cases: Invalid ChannelID leads to empty feed; network errors or rate limits possible.

  - **Filter: Published in Last 72h**  
    - Type: If  
    - Role: Filters videos published within the past 72 hours using a date comparison.  
    - Configuration: Checks if `pubDate` is after `now - 72 hours`.  
    - Input: From "Fetch Latest Videos (RSS)".  
    - Output: To "Get Video Stats (YouTube API)".  
    - Edge Cases: Videos without `pubDate` or malformed dates will be filtered out.

  - **Get Video Stats (YouTube API)**  
    - Type: HTTP Request  
    - Role: Retrieves detailed video metadata and statistics via YouTube Data API v3.  
    - Configuration:  
      - URL: `https://www.googleapis.com/youtube/v3/videos`  
      - Query parameters:  
        - `id`: Extracted video ID from the video link in RSS (handling shorts and standard URLs)  
        - `part`: `snippet,statistics` to fetch title, description, like counts  
      - Authentication: Uses a credential named `YouTube_API_Key` via HTTP Query Authentication.  
    - Input: From "Filter: Published in Last 72h".  
    - Output: To "Classify by Likes (Code)".  
    - Edge Cases:  
      - API quota limits or invalid API key cause failures.  
      - Video ID extraction might fail if URL format unexpected.  
      - Videos without statistics data cause classification fallback.

  - **Sticky Notes**  
    - Emphasize:  
      - "🔐 IMPORTANT: Do not hardcode API keys. Use the credential named 'YouTube_API_Key'."  
      - "Title: API-Key Configuration"  
      - "Classification fails without statistics."  
    - These notes ensure proper and secure API usage.

---

#### 2.3 Video Classification

- **Overview:**  
  Classifies each video as "viral" or "normal" based on like count threshold (default 1000 likes). If statistics are missing, marks for additional stats fetch. Then routes videos accordingly.

- **Nodes Involved:**  
  Classify by Likes (Code), Branch: Viral, Branch: Normal, Sticky Note about classification logic and threshold.

- **Node Details:**

  - **Classify by Likes (Code)**  
    - Type: Code  
    - Role: Processes video statistics to classify videos.  
    - Configuration:  
      - Threshold constant set to 1000 likes.  
      - Extracts like count robustly from statistics (handles numbers or strings).  
      - Classifies as:  
        - `"viral"` if likes ≥ threshold  
        - `"normal"` if likes < threshold  
        - `"unknown"` if no likes data  
      - Adds `needsStatsFetch` flag if statistics missing.  
      - Outputs videoId, title, likeCount, classification, original video data.  
    - Input: From "Get Video Stats (YouTube API)".  
    - Output: To both "Branch: Viral" and "Branch: Normal" switch nodes.  
    - Edge Cases:  
      - Videos missing like counts.  
      - Non-numeric like counts.  
      - Unexpected data structure.

  - **Branch: Viral**  
    - Type: Switch  
    - Role: Routes items with classification "viral" or "normal" to different outputs.  
    - Configuration: Two outputs named "Viral Video" and "Normal Video".  
    - Input: From "Classify by Likes (Code)".  
    - Output:  
      - "Viral Video" to "Write Post (Viral)".  
      - "Normal Video" unused on this branch (empty).  
    - Edge Cases: If classification missing or unexpected, no output.

  - **Branch: Normal**  
    - Type: Switch  
    - Role: Similar routing as "Branch: Viral" but for the other path.  
    - Configuration: Two outputs, "Normal Video" sends to "Write Post (Normal)".  
    - Input: From "Classify by Likes (Code)".  
    - Output: "Normal Video" to "Write Post (Normal)".  
    - Edge Cases: Same as above.

  - **Sticky Note**  
    - "📊 Classification: Videos with >= 1000 likes = \"viral\". Adjust by changing constant THRESHOLD in the classification Function node."  
    - "Title: Logic — What counts as \"viral\"?"  
    - "Routing is based on the classification."  
    - This guides users on tuning classification.

---

#### 2.4 AI-Based Content Generation

- **Overview:**  
  Generates LinkedIn-style posts for each video according to its classification using OpenAI GPT-4. Viral videos get prompts emphasizing urgency and relevance; normal videos receive informative, analytical posts.

- **Nodes Involved:**  
  Write Post (Viral), Write Post (Normal).

- **Node Details:**

  - **Write Post (Viral)**  
    - Type: OpenAI (LangChain)  
    - Role: Generates concise LinkedIn post emphasizing viral video urgency.  
    - Configuration:  
      - Model: GPT-4.1-mini  
      - System role: Social media strategist  
      - Prompt includes video title, classification, description, with instructions to produce:  
        - Strong hook question, 3 takeaways, short professional angle, CTA question, hashtags.  
    - Input: From "Branch: Viral" switch node.  
    - Output: To "Aggregate Posts for Briefing".  
    - Edge Cases:  
      - API quota or network errors.  
      - Input missing expected fields causes incomplete output.

  - **Write Post (Normal)**  
    - Type: OpenAI (LangChain)  
    - Role: Generates LinkedIn post for normal videos, more informative and discussion-oriented.  
    - Configuration: Similar to viral, but prompt tailored for normal content with 2–3 facts and discussion tone.  
    - Input: From "Branch: Normal" switch node.  
    - Output: To "Aggregate Posts for Briefing".  
    - Edge Cases: Same as above.

  - **Sticky Note**  
    - "🤖 OpenAI: Ensure credential \"OpenAi account\" exists. You can tweak tone/length of the prompts for LinkedIn/email."  
    - Guides users on OpenAI credential and prompt customization.

---

#### 2.5 Aggregation & Weekly Briefing Assembly

- **Overview:**  
  Aggregates all generated posts into a single input text, then uses GPT-5 to create a compact, structured HTML email briefing optimized for Gmail. The output includes trend overviews, deep dives, strategy, and audit logs.

- **Nodes Involved:**  
  Aggregate Posts for Briefing, Generate Weekly Briefing (HTML).

- **Node Details:**

  - **Aggregate Posts for Briefing**  
    - Type: Code  
    - Role: Concatenates all post texts from prior OpenAI responses into one string separated by delimiters.  
    - Configuration: Extracts `.json.choices[0].message.content` from each item, joins with `\n\n---\n\n`.  
    - Input: From both "Write Post" nodes.  
    - Output: To "Generate Weekly Briefing (HTML)".  
    - Edge Cases: Missing or empty post content results in empty briefing input.

  - **Generate Weekly Briefing (HTML)**  
    - Type: OpenAI (LangChain)  
    - Role: Produces a structured HTML email newsletter using GPT-5, designed for internal AI intelligence teams.  
    - Configuration:  
      - Model: GPT-5  
      - System prompt details strict HTML requirements:  
        - Gmail-robust HTML with centered 600px table, system fonts, no external assets, one <h1>, several <h2>s.  
        - Includes audit info: entity index, evidence snippets, SHA-256 hash of input.  
        - Regenerates if output not starting with `<!DOCTYPE html>`.  
      - User prompt feeds the aggregated raw post text.  
    - Input: From "Aggregate Posts for Briefing".  
    - Output: To "Send Weekly Briefing (Gmail)".  
    - Edge Cases: Failed HTML validation or empty input may cause regeneration loops or blank output.

---

#### 2.6 Email Delivery

- **Overview:**  
  Sends the generated HTML weekly briefing via Gmail using OAuth2 authentication.

- **Nodes Involved:**  
  Send Weekly Briefing (Gmail), Sticky Note about Email delivery.

- **Node Details:**

  - **Send Weekly Briefing (Gmail)**  
    - Type: Gmail  
    - Role: Sends the email with subject "Weekly AI Briefing" and content from the generated HTML.  
    - Configuration:  
      - Recipient email address set via parameter (empty string by default; must be configured).  
      - Message body uses the OpenAI HTML content.  
      - Uses Gmail OAuth2 credential named "Gmail account".  
    - Input: From "Generate Weekly Briefing (HTML)".  
    - Output: None (end node).  
    - Edge Cases:  
      - Missing recipient address causes send failure.  
      - OAuth token expiration or permission errors can block sending.

  - **Sticky Note**  
    - "✉️ Email delivery: Use Gmail OAuth (credential: \"Gmail account\") or any SMTP (credential: \"SMTP_Default\"). Always send a test first!"  
    - Advises on credential setup and testing.

---

### 3. Summary Table

| Node Name                 | Node Type                     | Functional Role                                            | Input Node(s)                 | Output Node(s)               | Sticky Note                                                                                     |
|---------------------------|-------------------------------|------------------------------------------------------------|------------------------------|-----------------------------|------------------------------------------------------------------------------------------------|
| Manual Run                | Manual Trigger                | Workflow manual start                                       | None                         | Set Channel IDs              | Title: Setup — Your YouTube Channels; Enter the Channel IDs (UC…) in the Set node              |
| Note: Channel IDs         | Sticky Note                  | Instruction on providing Channel IDs                        | None                         | None                        | ⚠️ IMPORTANT: Provide the YouTube Channel IDs in the Set node (field: ChannelID). After changes, run a manual test (Execute). |
| Set Channel IDs           | Set                          | Defines list of YouTube Channel IDs                         | Manual Run                   | Split Channels              | Title: Setup — Your YouTube Channels; Enter the Channel IDs (UC…) you want to monitor          |
| Split Channels            | SplitOut                     | Splits ChannelID array into individual items                | Set Channel IDs              | Fetch Latest Videos (RSS)   |                                                                                                |
| Fetch Latest Videos (RSS) | RSS Feed Read                | Fetch latest videos per channel via YouTube RSS feed        | Split Channels               | Filter: Published in Last 72h |                                                                                                |
| Filter: Published in Last 72h | If                         | Filters videos published in last 72 hours                   | Fetch Latest Videos (RSS)    | Get Video Stats (YouTube API) |                                                                                                |
| Get Video Stats (YouTube API) | HTTP Request               | Retrieves detailed video stats via YouTube Data API         | Filter: Published in Last 72h | Classify by Likes (Code)     | 🔐 IMPORTANT: Do not hardcode API keys. Use the credential named "YouTube_API_Key"              |
| Classify by Likes (Code)  | Code                         | Classifies videos as viral/normal based on like count       | Get Video Stats (YouTube API) | Branch: Viral, Branch: Normal | 📊 Classification: Videos with >= 1000 likes = "viral". Adjust by changing constant THRESHOLD in the classification Function node. |
| Branch: Viral             | Switch                       | Routes viral videos to viral post generation                 | Classify by Likes (Code)     | Write Post (Viral)          | Title: Logic — What counts as "viral"? Routing based on classification threshold                |
| Branch: Normal            | Switch                       | Routes normal videos to normal post generation               | Classify by Likes (Code)     | Write Post (Normal)         | Title: Logic — What counts as "viral"? Routing based on classification threshold                |
| Write Post (Viral)        | OpenAI (LangChain)           | Generates LinkedIn post for viral videos                     | Branch: Viral                | Aggregate Posts for Briefing | 🤖 OpenAI: Ensure credential "OpenAi account" exists. You can tweak tone/length of the prompts. |
| Write Post (Normal)       | OpenAI (LangChain)           | Generates LinkedIn post for normal videos                    | Branch: Normal               | Aggregate Posts for Briefing | 🤖 OpenAI: Ensure credential "OpenAi account" exists. You can tweak tone/length of the prompts. |
| Aggregate Posts for Briefing | Code                       | Aggregates all LinkedIn posts into one briefing input       | Write Post (Viral), Write Post (Normal) | Generate Weekly Briefing (HTML) |                                                                                                |
| Generate Weekly Briefing (HTML) | OpenAI (LangChain)       | Creates structured HTML email briefing from aggregated posts | Aggregate Posts for Briefing  | Send Weekly Briefing (Gmail) |                                                                                                |
| Send Weekly Briefing (Gmail) | Gmail                      | Sends final weekly briefing email                            | Generate Weekly Briefing (HTML) | None                        | ✉️ Email delivery: Use Gmail OAuth (credential: "Gmail account") or any SMTP (credential: "SMTP_Default"). Always send a test first! |
| Note: API Key             | Sticky Note                  | Reminder about API key usage and credential                  | None                         | None                        | 🔐 IMPORTANT: Do not hardcode API keys. Use the credential named "YouTube_API_Key" (field: apiKey). |
| Note: Threshold           | Sticky Note                  | Explains viral classification threshold                      | None                         | None                        | 📊 Classification: Videos with >= 1000 likes = "viral". Adjust by changing constant THRESHOLD in the classification Function node. |
| Note: OpenAI              | Sticky Note                  | Reminder about OpenAI credential and prompt customization    | None                         | None                        | 🤖 OpenAI: Ensure credential "OpenAi account" exists. You can tweak tone/length of the prompts for LinkedIn/email. |
| Note: Email               | Sticky Note                  | Instructions for email sending credentials and testing      | None                         | None                        | ✉️ Email delivery: Use Gmail OAuth (credential: "Gmail account") or any SMTP (credential: "SMTP_Default"). Always send a test first! |
| Note: Setup               | Sticky Note                  | Instruction about entering channel IDs                       | None                         | None                        | Title: Setup — Your YouTube Channels; Enter the Channel IDs (UC…) you want to monitor           |
| Note: YouTube API         | Sticky Note                  | API key configuration instructions                           | None                         | None                        | Title: API-Key Configuration; Ensure a valid YouTube Data API v3 credential exists.              |
| Note: Logic               | Sticky Note                  | Explanation of viral classification logic                    | None                         | None                        | Title: Logic — What counts as "viral"? Routing is based on the classification from the previous step. Threshold currently at 1000 likes (see Classify by Likes). |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node:**  
   - Name: Manual Run  
   - Default configuration.

2. **Add a Set node:**  
   - Name: Set Channel IDs  
   - Add field `ChannelID` of type Array.  
   - Assign Channel IDs to monitor, e.g., `["UC8T5gQ4U4GbI2h8kYCkEcvg","UCkZ3fSYruC0IXv6p34BHciQ","UCP0SLk271Ov781xSxG__7mA","UCM2u6Uvi5XBBlh5GDv4otsg"]`.  
   - Connect Manual Run → Set Channel IDs.

3. **Add a SplitOut node:**  
   - Name: Split Channels  
   - Split on field `ChannelID`.  
   - Connect Set Channel IDs → Split Channels.

4. **Add an RSS Feed Read node:**  
   - Name: Fetch Latest Videos (RSS)  
   - URL: `https://www.youtube.com/feeds/videos.xml?channel_id={{ $json.ChannelID }}`  
   - No additional custom fields.  
   - Connect Split Channels → Fetch Latest Videos (RSS).

5. **Add an If node:**  
   - Name: Filter: Published in Last 72h  
   - Condition: Check if `pubDate` (converted to Date) is after `now - 72 hours`.  
   - Use expression: `={{ new Date($json.pubDate) > new Date(Date.now() - 72*60*60*1000) }}`  
   - Connect Fetch Latest Videos (RSS) → Filter: Published in Last 72h.

6. **Add an HTTP Request node:**  
   - Name: Get Video Stats (YouTube API)  
   - Method: GET  
   - URL: `https://www.googleapis.com/youtube/v3/videos`  
   - Query parameters:  
     - `id`: Extract video ID from `$json.link` using expression to handle shorts or standard URLs.  
     - `part`: `snippet,statistics`  
   - Authentication: HTTP Query Authentication with credential named `YouTube_API_Key`.  
   - Connect both outputs of Filter node (true and false) to this node (to handle all filtered videos).  
   - Connect Filter: Published in Last 72h → Get Video Stats (YouTube API).

7. **Add a Code node:**  
   - Name: Classify by Likes (Code)  
   - Paste the JavaScript code provided, which:  
     - Extracts like count, classifies as "viral" or "normal" using threshold=1000.  
     - Flags videos needing stats fetch.  
   - Connect Get Video Stats (YouTube API) → Classify by Likes (Code).

8. **Add two Switch nodes:**  
   - Names: Branch: Viral and Branch: Normal  
   - Each configured with two outputs: "Viral Video" and "Normal Video", routing based on `$json.classification`.  
   - Connect Classify by Likes (Code) → Branch: Viral & Branch: Normal.

9. **Add two OpenAI (LangChain) nodes:**  
   - Names: Write Post (Viral) and Write Post (Normal).  
   - Model: GPT-4.1-mini  
   - Prompts: Use the detailed prompts provided for viral and normal videos respectively.  
   - Credentials: OpenAI API credential named `OpenAi account`.  
   - Connect Branch: Viral "Viral Video" → Write Post (Viral).  
   - Connect Branch: Normal "Normal Video" → Write Post (Normal).

10. **Add a Code node:**  
    - Name: Aggregate Posts for Briefing  
    - JavaScript code to concatenate all post texts from `.json.choices[0].message.content` with delimiter `\n\n---\n\n`.  
    - Connect both Write Post nodes → Aggregate Posts for Briefing.

11. **Add an OpenAI (LangChain) node:**  
    - Name: Generate Weekly Briefing (HTML)  
    - Model: GPT-5  
    - System prompt: Detailed instructions for producing a Gmail-compatible HTML briefing with audit and structure.  
    - User prompt: Feed aggregated raw post text.  
    - Credentials: OpenAI API credential `OpenAi account`.  
    - Connect Aggregate Posts for Briefing → Generate Weekly Briefing (HTML).

12. **Add a Gmail node:**  
    - Name: Send Weekly Briefing (Gmail)  
    - Set recipient email in parameter `sendTo`.  
    - Subject: "Weekly AI Briefing".  
    - Message body: Use generated HTML content from previous node.  
    - Credentials: Gmail OAuth2 credential named `Gmail account`.  
    - Connect Generate Weekly Briefing (HTML) → Send Weekly Briefing (Gmail).

13. **Add Sticky Notes throughout the canvas as per the original workflow:**  
    - Notes for API key usage, channel ID setup, classification threshold, OpenAI credential reminders, and email sending instructions.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                              |
|-----------------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| ⚠️ IMPORTANT: Provide the YouTube Channel IDs in the Set node (field: ChannelID). After changes, run a manual test (Execute). | Setup instructions for monitored channels                    |
| 🔐 IMPORTANT: Do not hardcode API keys. Use the credential named "YouTube_API_Key".                        | API key configuration and security best practice            |
| 📊 Classification: Videos with >= 1000 likes = "viral". Adjust threshold in the classification Function node. | Threshold tuning for viral detection                          |
| 🤖 OpenAI: Ensure credential "OpenAi account" exists. You can tweak tone/length of the prompts.            | OpenAI credential and prompt customization                    |
| ✉️ Email delivery: Use Gmail OAuth (credential: "Gmail account") or any SMTP (credential: "SMTP_Default"). Always send a test first! | Email sending configuration and testing recommendation       |
| Title: Setup — Your YouTube Channels. Enter the Channel IDs (UC…) you want to monitor.                     | User instruction on channel ID entry                          |
| Title: API-Key Configuration. Ensure a valid YouTube Data API v3 credential exists.                         | API key and permission reminder                               |
| Title: Logic — What counts as "viral"? Routing is based on classification threshold.                       | Explanation of classification and routing logic              |

---

**Disclaimer:** The provided text is exclusively derived from an automated n8n workflow designed for lawful and public data processing. It adheres strictly to content policies and contains no illegal, offensive or protected content. Data manipulated is legal and public.