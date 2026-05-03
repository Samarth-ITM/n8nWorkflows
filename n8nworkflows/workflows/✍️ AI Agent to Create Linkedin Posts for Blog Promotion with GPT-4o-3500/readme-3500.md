✍️ AI Agent to Create Linkedin Posts for Blog Promotion with GPT-4o

https://n8nworkflows.xyz/workflows/---ai-agent-to-create-linkedin-posts-for-blog-promotion-with-gpt-4o-3500


# ✍️ AI Agent to Create Linkedin Posts for Blog Promotion with GPT-4o

### 1. Workflow Overview

This workflow automates the creation of professional LinkedIn posts to promote blog articles hosted on a Ghost CMS platform. It is designed for content creators, marketers, and bloggers who want to save time by automatically generating engaging LinkedIn content using AI (GPT-4o) based on their blog posts.

The workflow is logically divided into the following blocks:

- **1.1 Workflow Trigger**: Manual initiation of the workflow (can be replaced or supplemented by a scheduler).
- **1.2 Blog Post Extraction**: Fetches recent blog posts from Ghost CMS and extracts relevant metadata and content.
- **1.3 Content Cleaning**: Cleans the raw HTML content of blog posts to prepare it for AI processing.
- **1.4 AI Processing (LinkedIn Post Generation)**: Uses an AI Agent powered by GPT-4o to generate a polished LinkedIn post for each blog article.
- **1.5 Data Recording**: Appends the original blog data and generated LinkedIn posts into a Google Sheet for record-keeping and further use.

---

### 2. Block-by-Block Analysis

#### 2.1 Workflow Trigger

- **Overview:**  
  This block initiates the workflow manually. It can be replaced or supplemented by a scheduled trigger for automation.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’

- **Node Details:**  
  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Configuration: No parameters; triggers workflow on manual test click  
    - Input: None  
    - Output: Triggers the next node to start the workflow  
    - Edge Cases: None (manual trigger)  
    - Notes: Can be replaced by a Cron node for scheduled runs  

---

#### 2.2 Blog Post Extraction

- **Overview:**  
  This block pulls recent blog posts from the Ghost CMS API and extracts key metadata and content fields for further processing.

- **Nodes Involved:**  
  - Extract Blog Posts  
  - Extract Post Content

- **Node Details:**  
  - **Extract Blog Posts**  
    - Type: Ghost CMS node  
    - Configuration:  
      - Operation: getAll  
      - Limit: 3 posts (configurable)  
      - Credentials: Ghost API credentials required  
    - Input: Trigger node output  
    - Output: Array of blog posts with full metadata and content  
    - Edge Cases: API authentication failure, network errors, empty blog post list  
    - Notes: Requires Ghost API credentials setup  

  - **Extract Post Content**  
    - Type: Set node  
    - Configuration: Extracts and assigns specific fields from the Ghost node output:  
      - id, title, featured_image, excerpt, content (HTML), link (URL)  
    - Input: Output from Extract Blog Posts  
    - Output: Structured JSON with selected fields for each post  
    - Edge Cases: Missing fields in blog post data, malformed content  
    - Notes: Prepares data for batch processing  

---

#### 2.3 Content Cleaning

- **Overview:**  
  Cleans the HTML content of each blog post by stripping tags and normalizing whitespace to prepare clean text for AI input.

- **Nodes Involved:**  
  - Loop Over Posts  
  - Clean HTML  
  - Add Clean HTML

- **Node Details:**  
  - **Loop Over Posts**  
    - Type: SplitInBatches  
    - Configuration: Default batch size (processes posts one by one)  
    - Input: Output from Extract Post Content  
    - Output: Single post per batch for processing  
    - Edge Cases: Large number of posts may slow processing; batch size can be adjusted  

  - **Clean HTML**  
    - Type: Code node (JavaScript)  
    - Configuration:  
      - Removes all HTML tags using regex  
      - Normalizes spaces and decodes common HTML entities (e.g., &nbsp;)  
      - Trims leading/trailing whitespace  
    - Input: Single post content (HTML) from Loop Over Posts  
    - Output: JSON with `clean_content` field containing plain text  
    - Edge Cases: Complex HTML structures may not be fully cleaned; potential for regex failures if content is malformed  

  - **Add Clean HTML**  
    - Type: Merge node  
    - Configuration: Combine mode "combineBySql" to merge original post data with cleaned content  
    - Input: Original post data and cleaned content from Clean HTML  
    - Output: Enriched post data including `clean_content`  
    - Edge Cases: Merge failures if input data mismatches  

---

#### 2.4 AI Processing (LinkedIn Post Generation)

- **Overview:**  
  Sends the cleaned blog post content and metadata to an AI Agent powered by GPT-4o to generate a professional LinkedIn promotional post.

- **Nodes Involved:**  
  - OpenAI Chat Model  
  - AI Agent  
  - Merge Linkedin

- **Node Details:**  
  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI Chat Model node  
    - Configuration:  
      - Model: gpt-4o-mini  
      - Credentials: OpenAI API key required  
    - Input: None directly; linked as language model for AI Agent  
    - Output: Language model interface for AI Agent  
    - Edge Cases: API rate limits, authentication errors, model unavailability  

  - **AI Agent**  
    - Type: LangChain Agent node  
    - Configuration:  
      - Prompt includes article title, link, and cleaned content  
      - System message instructs AI to generate a LinkedIn post with:  
        - Attention-grabbing hook  
        - Summary of article value  
        - Call-to-action  
        - Author signature with contact info  
      - Output: Generated LinkedIn post text (no hashtags or markdown)  
      - Uses OpenAI Chat Model as language model  
    - Input: Merged post data with `clean_content`  
    - Output: AI-generated LinkedIn post text in `output` field  
    - Edge Cases: AI response errors, prompt failures, incomplete generation  

  - **Merge Linkedin**  
    - Type: Merge node  
    - Configuration: Combine mode "combineBySql" to merge AI output with post data  
    - Input: AI Agent output and original post data  
    - Output: Complete data including LinkedIn post text  
    - Edge Cases: Merge mismatches  

---

#### 2.5 Data Recording

- **Overview:**  
  Appends the blog post metadata, cleaned content, and generated LinkedIn post into a Google Sheet for record-keeping and further use.

- **Nodes Involved:**  
  - Record the posts

- **Node Details:**  
  - **Record the posts**  
    - Type: Google Sheets node  
    - Configuration:  
      - Operation: Append  
      - Document ID: Configured to target specific Google Sheet  
      - Sheet Name: Configured (gid=0)  
      - Columns mapped: id, title, featured_image, excerpt, content, clean_content, linkedin_post  
      - Credentials: Google Sheets API OAuth2 required  
    - Input: Merged post data with LinkedIn post text  
    - Output: Confirmation of append operation  
    - Edge Cases: Authentication failure, quota limits, invalid sheet ID, network errors  

  - **Loop Over Posts (retrigger)**  
    - After recording, the workflow loops back to process next batch of posts until all are processed  

---

### 3. Summary Table

| Node Name               | Node Type                     | Functional Role                          | Input Node(s)               | Output Node(s)           | Sticky Note                                                                                                  |
|-------------------------|-------------------------------|----------------------------------------|-----------------------------|--------------------------|--------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                | Initiates the workflow manually        | None                        | Extract Blog Posts        | ### 1. Workflow Trigger<br>This workflow uses simple trigger.<br>#### How to setup?<br>*Nothing to do.*      |
| Extract Blog Posts       | Ghost CMS node                | Fetches recent blog posts from Ghost   | When clicking ‘Test workflow’ | Extract Post Content      | ### 2. Extract Blog Posts Content<br>The Ghost node extracts all the posts of your blog with content and metadata.<br>#### How to setup?<br>- **Ghost Account API**:<br>   1. Add your Ghost Blog Account Credentials<br>   2. Select the number of Blog Posts you want to collect<br>[Learn more about the Ghost Node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.ghost) |
| Extract Post Content     | Set node                     | Extracts and structures blog post data | Extract Blog Posts          | Loop Over Posts           | See above                                                                                                    |
| Loop Over Posts          | SplitInBatches               | Processes posts one by one in batches  | Extract Post Content        | Clean HTML, Add Clean HTML | See Sticky Note 3 below                                                                                      |
| Clean HTML              | Code node                    | Cleans HTML content to plain text      | Loop Over Posts             | Add Clean HTML            | See Sticky Note 3 below                                                                                      |
| Add Clean HTML           | Merge node                   | Merges cleaned content with post data  | Loop Over Posts, Clean HTML | AI Agent, Merge Linkedin  | See Sticky Note 3 below                                                                                      |
| OpenAI Chat Model        | LangChain OpenAI Chat Model  | Provides GPT-4o language model         | None (linked to AI Agent)   | AI Agent (language model) | See Sticky Note 3 below                                                                                      |
| AI Agent                 | LangChain Agent              | Generates LinkedIn post from content   | Add Clean HTML              | Merge Linkedin            | ### 3. Generate a Linkedin Post for each Post with an AI Agent<br>This block loops through all the posts pulled by the Ghost Node, send the content to the AI agent that generates a Linkedin post. The results are combined and pulled in a Google Sheet.<br>#### How to setup?<br>- **AI Agent with the Chat Model**:<br>   1. Add a **chat model** with the required credentials *(Example: Open AI 4o-mini)*<br>   2. Adapt the system prompt with your **post signature** and additional points you want to add in your posts<br>[Learn more about the AI Agent Node](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent) |
| Merge Linkedin           | Merge node                   | Combines AI output with post data      | AI Agent, Add Clean HTML    | Record the posts          | See above                                                                                                    |
| Record the posts         | Google Sheets node           | Appends data to Google Sheet           | Merge Linkedin              | Loop Over Posts           | See Sticky Note 3 above                                                                                      |
| Sticky Note1             | Sticky Note                  | Workflow trigger explanation           | None                       | None                     | ### 1. Workflow Trigger<br>This workflow uses simple trigger.<br>#### How to setup?<br>*Nothing to do.*      |
| Sticky Note2             | Sticky Note                  | Ghost blog extraction explanation      | None                       | None                     | ### 2. Extract Blog Posts Content<br>The Ghost node extracts all the posts of your blog with content and metadata.<br>#### How to setup?<br>- **Ghost Account API**:<br>   1. Add your Ghost Blog Account Credentials<br>   2. Select the number of Blog Posts you want to collect<br>[Learn more about the Ghost Node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.ghost) |
| Sticky Note              | Sticky Note                  | AI Agent and Google Sheets explanation | None                       | None                     | ### 3. Generate a Linkedin Post for each Post with an AI Agent<br>This block loops through all the posts pulled by the Ghost Node, send the content to the AI agent that generates a Linkedin post. The results are combined and pulled in a Google Sheet.<br>#### How to setup?<br>- **AI Agent with the Chat Model**:<br>   1. Add a **chat model** with the required credentials *(Example: Open AI 4o-mini)*<br>   2. Adapt the system prompt with your **post signature** and additional points you want to add in your posts<br>[Learn more about the AI Agent Node](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent)<br>- **Record Long Break in the Google Sheet Node**:<br>   1. Add your Google Sheet API credentials to access the Google Sheet file<br>   2. Select the file using the list, an URL or an ID<br>   3. Select the sheet in which you want to record your working sessions<br>   4. Map the fields<br>[Learn more about the Google Sheet Node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets) |
| Sticky Note3             | Sticky Note                  | Tutorial video and example post image  | None                       | None                     | ### [📺Complete Tutorial](https://www.youtube.com/watch?v=Lhi6hV6rWEo)<br>![Thumbnail](https://www.samirsaci.com/content/images/2025/04/temp-4.png) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To start the workflow manually  
   - No parameters needed  

2. **Add a Ghost CMS Node**  
   - Type: Ghost  
   - Operation: getAll  
   - Limit: Set to desired number of blog posts (e.g., 3)  
   - Credentials: Configure Ghost API credentials (API URL and key)  
   - Connect output from Manual Trigger  

3. **Add a Set Node to Extract Post Content**  
   - Type: Set  
   - Assign the following fields from Ghost node output:  
     - id = {{$json.id}}  
     - title = {{$json.title}}  
     - featured_image = {{$json.feature_image}}  
     - excerpt = {{$json.excerpt}}  
     - content = {{$json.html}} (raw HTML)  
     - link = {{$json.url}}  
   - Connect output from Ghost node  

4. **Add a SplitInBatches Node**  
   - Type: SplitInBatches  
   - Purpose: Process each blog post individually  
   - Connect output from Set node  

5. **Add a Code Node to Clean HTML Content**  
   - Type: Code (JavaScript)  
   - Code snippet:  
     ```javascript
     const htmlContent = $input.first().json.content;
     const cleanText = htmlContent
       .replace(/<[^>]*>/g, '') // remove HTML tags
       .replace(/\s+/g, ' ')    // normalize spaces
       .replace(/&nbsp;/g, ' ') // decode common entity
       .trim();
     return [{ json: { clean_content: cleanText } }];
     ```  
   - Connect output from SplitInBatches node  

6. **Add a Merge Node to Combine Cleaned Content with Original Data**  
   - Type: Merge  
   - Mode: combineBySql  
   - Connect inputs:  
     - First input: output from SplitInBatches (original post data)  
     - Second input: output from Code node (clean_content)  

7. **Add an OpenAI Chat Model Node**  
   - Type: LangChain OpenAI Chat Model  
   - Model: gpt-4o-mini  
   - Credentials: Configure OpenAI API key  
   - No direct input connection; will be linked to AI Agent node  

8. **Add an AI Agent Node**  
   - Type: LangChain Agent  
   - Parameters:  
     - Text input:  
       ```
       Article Title: {{ $json.title }}
       Article Link: {{ $json.link }}
       Article Content: {{ $json.clean_content }}
       ```  
     - System message:  
       ```
       You are a content marketing assistant. Based on the article metadata (ID, title) and cleaned content, generate a short LinkedIn promotional message for a professional audience.

       Follow this structure:

       Start with a hook that grabs attention (a bold insight, surprising fact, or thought-provoking question).

       Briefly summarize the article’s value — what readers will learn or gain from it.

       Include a clear call-to-action encouraging readers to read the article.

       End with this author signature and invitation:
       “—
       Samir Saci
       Supply Chain Data Scientist & Founder of LogiGreen
       📩 Contact me: https://logi-green.com/contactus”

       Use a professional and engaging tone. Do not include hashtags or Markdown formatting.
       ```  
     - Prompt type: define  
   - Connect input from Merge node (cleaned post data)  
   - Link OpenAI Chat Model node as language model  

9. **Add a Merge Node to Combine AI Output with Post Data**  
   - Type: Merge  
   - Mode: combineBySql  
   - Connect inputs:  
     - First input: output from AI Agent (LinkedIn post text)  
     - Second input: output from previous Merge node (post data with clean content)  

10. **Add a Google Sheets Node to Record Data**  
    - Type: Google Sheets  
    - Operation: Append  
    - Document ID: Select or enter your Google Sheet ID  
    - Sheet Name: Select the sheet (e.g., gid=0)  
    - Map columns:  
      - id = {{$json.id}}  
      - title = {{$json.title}}  
      - featured_image = {{$json.featured_image}}  
      - excerpt = {{$json.excerpt}}  
      - content = {{$json.content}}  
      - clean_content = {{$json.clean_content}}  
      - linkedin_post = {{$json.output}} (AI-generated post)  
    - Credentials: Configure Google Sheets OAuth2 credentials  
    - Connect input from Merge node (AI output + post data)  

11. **Connect Google Sheets Node Output Back to SplitInBatches Node**  
    - This loops the workflow to process the next batch until all posts are processed  

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow was built using n8n version 1.85.4                                                               | Version requirement                                                                              |
| You can adapt this template for Twitter, Facebook, or email newsletters by adjusting the AI prompt and output channel | Use case flexibility                                                                             |
| Tutorial video explaining the workflow and usage                                                                | [YouTube Tutorial](https://www.youtube.com/watch?v=Lhi6hV6rWEo)                                  |
| Ghost Node documentation                                                                                        | [Ghost Node Docs](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.ghost)       |
| AI Agent Node documentation                                                                                     | [AI Agent Node Docs](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent) |
| Google Sheets Node documentation                                                                                 | [Google Sheets Node Docs](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets) |
| Author and contact for further help                                                                              | Samir Saci - [LinkedIn](https://www.linkedin.com/in/samir-saci)                                  |

---

This document provides a comprehensive, stepwise understanding of the workflow, enabling users and AI agents to reproduce, modify, and troubleshoot the automation effectively.