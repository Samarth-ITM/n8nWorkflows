Transform RSS Feeds into Platform-Optimized Content with GPT-4o for Medium, LinkedIn & Instagram

https://n8nworkflows.xyz/workflows/transform-rss-feeds-into-platform-optimized-content-with-gpt-4o-for-medium--linkedin---instagram-8662


# Transform RSS Feeds into Platform-Optimized Content with GPT-4o for Medium, LinkedIn & Instagram

### 1. Workflow Overview

This workflow automates the transformation of RSS feed news articles into platform-optimized content for Medium, LinkedIn, and Instagram using OpenAI's GPT-4o language model. It is designed primarily for marketers, social media managers, and content creators who want to leverage AI to generate professional, engaging posts consistently.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled RSS Feed Retrieval**: Periodically fetches news articles from a specified RSS feed.
- **1.2 News Aggregation & Structuring**: Aggregates multiple RSS items and structures them into a concise, AI-readable format.
- **1.3 Article Selection by AI**: Uses an AI agent to analyze the aggregated news and select the top articles relevant to the company’s domain.
- **1.4 Content Generation for Medium**: Generates a Medium post draft using the selected articles with a specified “Codint” tone of voice.
- **1.5 Content Generation for LinkedIn & Instagram**: Converts the Medium post into tailored content for LinkedIn and Instagram platforms.
- **1.6 Email Notifications**: Sends generated drafts via Gmail SMTP for manual review before publishing.

Supporting the above logic are multiple code nodes for data transformation and sticky notes for documentation and instructions.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Scheduled RSS Feed Retrieval

- **Overview:** Automatically triggers the workflow every 3 days at 08:30 to poll an RSS feed for new articles.
- **Nodes Involved:** `Schedule Trigger`, `RSS Read`
- **Node Details:**

  - **Schedule Trigger**
    - Type: Schedule Trigger
    - Configuration: Runs every 3 days at 08:30 AM.
    - Inputs: None (start node)
    - Outputs: `RSS Read`
    - Edge Cases: Workflow not triggered if n8n instance is offline or suspended.

  - **RSS Read**
    - Type: RSS Feed Read
    - Configuration: Reads from "https://aibusiness.com/rss.xml"
    - Inputs: Trigger from `Schedule Trigger`
    - Outputs: `Aggregate`
    - Edge Cases: RSS feed unreachable or malformed XML causing read failures or empty data.

#### Block 1.2: News Aggregation & Structuring

- **Overview:** Aggregates fields (title, content, link) from RSS items and restructures them into a format suitable for AI analysis.
- **Nodes Involved:** `Aggregate`, `group the news into 1 item`, `Sticky Note`
- **Node Details:**

  - **Aggregate**
    - Type: Aggregate
    - Configuration: Aggregates `title`, `content`, and `link` fields from RSS items into arrays.
    - Inputs: `RSS Read`
    - Outputs: `group the news into 1 item`
    - Edge Cases: Empty RSS feed results in empty arrays.

  - **group the news into 1 item**
    - Type: Code Node (JavaScript)
    - Configuration: Converts aggregated arrays into an array of article objects with stable IDs and generates a short preview for each article.
    - Key Variables: `articles` (array of objects with id, title, content, link), `forAI` (string list for AI input)
    - Inputs: `Aggregate`
    - Outputs: `Best Article finder` and `Find the urls the ai outputted`
    - Edge Cases: Missing fields default to empty strings; no articles results in empty outputs.

  - **Sticky Note ("# Collecting the news")**
    - Functional Role: Documentation and contextual note for the block.

#### Block 1.3: Article Selection by AI

- **Overview:** An AI agent reads the aggregated news summaries and selects up to 3 best articles relevant to the company’s domain.
- **Nodes Involved:** `Best Article finder`, `Find the urls the ai outputted`
- **Node Details:**

  - **Best Article finder**
    - Type: Langchain Agent node (AI agent)
    - Configuration: Prompt instructs AI to pick up to 3 best articles related to "AI Agents & Automation" from the provided list.
    - Input Expression: `{{ $json.forAI }}` — the human-readable article previews.
    - Inputs: `group the news into 1 item`
    - Outputs: `Find the urls the ai outputted`
    - Edge Cases: AI output format errors or empty response can disrupt downstream logic.

  - **Find the urls the ai outputted**
    - Type: Code Node (JavaScript)
    - Configuration: Parses AI output to extract article IDs, maps them back to full article objects, and returns chosen articles and their links.
    - Inputs: `Best Article finder`, fallback from `group the news into 1 item`
    - Outputs: `All url's to 1 string`
    - Edge Cases: Empty or malformed AI output throws explicit error; missing articles array throws error.

#### Block 1.4: Content Generation for Medium

- **Overview:** Generates a Medium blog post draft based on the selected articles, using a defined “Codint” tone of voice.
- **Nodes Involved:** `All url's to 1 string`, `Tone of voice content writer`, `Send Content Confirmation1`
- **Node Details:**

  - **All url's to 1 string**
    - Type: Code Node (JavaScript)
    - Configuration: Collects all article links into an array and a single newline-separated string.
    - Inputs: `Find the urls the ai outputted`
    - Outputs: `Tone of voice content writer`
    - Edge Cases: Handles links inside nested arrays or directly inside items.

  - **Tone of voice content writer**
    - Type: Langchain Agent node (AI agent)
    - Configuration: Prompt to create a Medium post based on article links, using the Codint tone of voice:
      - Professional, innovative, human
      - Short, clear, active sentences
      - Direct reader address
      - Simple explanations with examples/metaphors
      - Positive and future-oriented
      - Empathetic and result-focused
      - Ends with a question or call-to-action
      - Links back to Codint with a free AI consult offer
    - Inputs: `All url's to 1 string`
    - Outputs: `Send Content Confirmation1` and `Instagram & Linkedin Writer`
    - Edge Cases: AI failures or prompt misinterpretation could generate irrelevant or off-tone content.

  - **Send Content Confirmation1**
    - Type: Gmail node (OAuth2)
    - Configuration: Sends the Medium post draft to a specified email (`alperaksakal2006@gmail.com`)
    - Inputs: `Tone of voice content writer`
    - Outputs: None
    - Edge Cases: Gmail authentication errors, mail sending failures.

  - **Sticky Note ("# Generate content based on the news for Medium, Linkedin & Instagram")**
    - Functional Role: Documentation for this content generation block.

#### Block 1.5: Content Generation for LinkedIn & Instagram

- **Overview:** Generates platform-specific posts for LinkedIn and Instagram from the Medium draft, adapting tone and style per platform.
- **Nodes Involved:** `Instagram & Linkedin Writer`, `Send Content Confirmation`
- **Node Details:**

  - **Instagram & Linkedin Writer**
    - Type: Langchain Agent node (AI agent)
    - Configuration: Receives Medium post content as input and produces formatted posts for LinkedIn and Instagram with platform-specific tone:
      - Instagram: light, visual, short
      - LinkedIn: business, thought leadership, case studies
      - Same Codint tone of voice guidelines as previous node apply.
    - Inputs: `Tone of voice content writer`
    - Outputs: `Send Content Confirmation`
    - Edge Cases: AI may not perfectly distinguish platforms if input is ambiguous.

  - **Send Content Confirmation**
    - Type: Gmail node (OAuth2)
    - Configuration: Sends the LinkedIn and Instagram drafts to the same email as above.
    - Inputs: `Instagram & Linkedin Writer`
    - Outputs: None
    - Edge Cases: Same as above for email sending.

#### Block 1.6: Workflow Trigger & Meta Information

- **Sticky Notes:**

  - **Sticky Note2**: Provides detailed usage instructions, tips for customization, and potential improvements such as adding user approval loops or direct posting.
  - **Sticky Note3**: Provides project overview, quick setup steps, and target audience description.

---

### 3. Summary Table

| Node Name                | Node Type                       | Functional Role                                  | Input Node(s)                    | Output Node(s)                   | Sticky Note                                                                                                                             |
|--------------------------|--------------------------------|-------------------------------------------------|---------------------------------|---------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger         | Schedule Trigger                | Periodic workflow trigger every 3 days at 08:30 | None                            | RSS Read                       |                                                                                                                                        |
| RSS Read                 | RSS Feed Read                  | Fetch RSS feed articles                          | Schedule Trigger                 | Aggregate                      |                                                                                                                                        |
| Aggregate                | Aggregate                      | Aggregate title, content, link fields           | RSS Read                       | group the news into 1 item     |                                                                                                                                        |
| group the news into 1 item| Code Node (JS)                 | Structure aggregated data into article objects  | Aggregate                      | Best Article finder, Find the urls the ai outputted |                                                                                                                                        |
| Best Article finder      | Langchain AI Agent             | Select top 3 relevant articles from news        | group the news into 1 item      | Find the urls the ai outputted |                                                                                                                                        |
| Find the urls the ai outputted | Code Node (JS)             | Parse AI output IDs, map to full articles        | Best Article finder, group the news into 1 item | All url's to 1 string       |                                                                                                                                        |
| All url's to 1 string    | Code Node (JS)                 | Collect all article links into array/string     | Find the urls the ai outputted  | Tone of voice content writer   |                                                                                                                                        |
| Tone of voice content writer | Langchain AI Agent          | Generate Medium post draft in Codint tone        | All url's to 1 string           | Send Content Confirmation1, Instagram & Linkedin Writer |                                                                                                                                        |
| Send Content Confirmation1 | Gmail (OAuth2)                | Email Medium draft for review                    | Tone of voice content writer    | None                          |                                                                                                                                        |
| Instagram & Linkedin Writer | Langchain AI Agent           | Generate LinkedIn & Instagram posts from Medium draft | Tone of voice content writer    | Send Content Confirmation      |                                                                                                                                        |
| Send Content Confirmation | Gmail (OAuth2)                 | Email LinkedIn & Instagram drafts for review     | Instagram & Linkedin Writer     | None                          |                                                                                                                                        |
| Sticky Note              | Sticky Note                   | "# Collecting the news"                           | None                           | None                          |                                                                                                                                        |
| Sticky Note1             | Sticky Note                   | "# Generate content based on the news for Medium, Linkedin & Instagram" | None                           | None                          |                                                                                                                                        |
| Sticky Note2             | Sticky Note                   | Instructions and potential improvements          | None                           | None                          | This automation is made by Codint. See detailed usage and improvement notes inside the workflow.                                        |
| Sticky Note3             | Sticky Note                   | Project overview and quick setup guide           | None                           | None                          | Related news for content marketing; perfect for marketers and content creators. Setup steps included.                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**
   - Type: Schedule Trigger
   - Set to trigger every 3 days at 08:30 AM.

2. **Create RSS Read Node**
   - Type: RSS Feed Read
   - Set URL: `https://aibusiness.com/rss.xml`
   - Connect Schedule Trigger output to this node’s input.

3. **Create Aggregate Node**
   - Type: Aggregate
   - Configure to aggregate fields: `title`, `content`, `link`
   - Connect RSS Read output to Aggregate input.

4. **Create Code Node "group the news into 1 item"**
   - Paste provided JavaScript code:
     - Aggregates arrays into array of article objects with IDs, titles, previews.
   - Connect Aggregate output to this node.

5. **Create Langchain Agent Node "Best Article finder"**
   - Model: GPT-4o-mini (or your OpenAI GPT-4o model)
   - Prompt: "I want you to analyze the following for my company. Find me the best articles to use in my medium post. My company is about AI Agents & Automation. Pick up to 3. Reply ONLY with a comma-separated list of the IDs (e.g., 3,7,12).\n\n{{ $json.forAI }}"
   - Connect "group the news into 1 item" output to this node.
   - Set OpenAI credentials.

6. **Create Code Node "Find the urls the ai outputted"**
   - Paste provided JavaScript code:
     - Parses AI output, extracts numeric IDs, maps back to full articles.
   - Connect Best Article finder output and also connect "group the news into 1 item" as fallback.

7. **Create Code Node "All url's to 1 string"**
   - Paste provided JavaScript code:
     - Collects all article links into array and single string.
   - Connect "Find the urls the ai outputted" output to this node.

8. **Create Langchain Agent Node "Tone of voice content writer"**
   - Model: GPT-4o-mini
   - Prompt: "Maak een medium post gebaseerd op de volgende artiekelen.\n\n{{ $json.links }}"
   - System Message: Codint tone of voice instructions (professional, innovative, human, etc.)
   - Connect "All url's to 1 string" output to this node.
   - Set OpenAI credentials.

9. **Create Gmail Node "Send Content Confirmation1"**
   - Configure to send email to your email (e.g., `alperaksakal2006@gmail.com`)
   - Subject: "Je nieuwe medium post is hier!"
   - Message body: `={{ $json.output }}`
   - Connect "Tone of voice content writer" output to this node.
   - Set Gmail OAuth2 credentials.

10. **Create Langchain Agent Node "Instagram & Linkedin Writer"**
    - Model: GPT-4o-mini
    - Prompt: "Het volgende is gepost op ons medium platform:\n{{ $json.output }}\n\nIk wil dat je dit als basis gebruikt, voor een post op zowel LinkedIn als Instagram.\nOutput alleen de content. Gebruik het volgende structuur:\n\nLinkedin:\n[content]\n\nInstagram:\n[content]"
    - System Message: Codint tone of voice + platform style instructions
    - Connect "Tone of voice content writer" output to this node.
    - Set OpenAI credentials.

11. **Create Gmail Node "Send Content Confirmation"**
    - Configure to send email to your email (same as above)
    - Subject: "Linkedn & Instagram content!"
    - Message body: `={{ $json.output }}`
    - Connect "Instagram & Linkedin Writer" output to this node.
    - Set Gmail OAuth2 credentials.

12. **Set Node Connections**
    - Schedule Trigger → RSS Read → Aggregate → group the news into 1 item → Best Article finder → Find the urls the ai outputted → All url's to 1 string → Tone of voice content writer → Send Content Confirmation1
    - Tone of voice content writer → Instagram & Linkedin Writer → Send Content Confirmation

13. **Add Sticky Notes as desired**
    - Add descriptive sticky notes for documentation and instructions.

14. **Ensure Credentials Setup**
    - OpenAI API credentials configured and tested in all Langchain nodes.
    - Gmail OAuth2 credentials configured for email nodes.

15. **Test Workflow**
    - Trigger manually and verify outputs at each step.
    - Adjust prompts or RSS feed URLs as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                                                            |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------|
| Workflow created by Codint, an AI-automation & agents studio.                                                 | Workflow sticky notes describe usage, improvements, and contact points.                                                                   |
| Usage instructions include updating RSS feed URL, tailoring AI prompts, and email recipient configuration.   | Sticky Note2 inside workflow.                                                                                                             |
| Potential improvements: user approval loop, automatic posting, multi-platform support.                        | Mentioned in Sticky Note2.                                                                                                                |
| Perfect for marketers, social media managers & creators seeking AI-powered content.                           | Sticky Note3 overview.                                                                                                                    |
| Tone of voice guidelines emphasize professionalism, clarity, empathy, and direct reader engagement.          | Applied consistently in AI prompt system messages for content generation nodes.                                                           |
| OpenAI GPT-4o-mini model is used for all AI processing nodes, requires valid OpenAI API key credential setup. | All Langchain nodes are configured with this model and credentials.                                                                       |
| Gmail OAuth2 credential required for sending emails, ensure proper OAuth2 setup and token refresh mechanisms. | Gmail nodes sending drafts for review use OAuth2 credentials named "Gmail account" in this workflow.                                        |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created using n8n, an integration and automation tool. This process strictly adheres to content policies and contains no illegal, offensive, or protected material. All manipulated data is legal and public.